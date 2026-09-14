# W&B 集成 — 凭据、发现与同步编排

> qorix-ui 不自己造数据,只把 W&B 上的 `qorix` run「拉」到本地。本篇回答三个问题:凭据放哪、run 怎么被发现、同步怎么被编排。

## 1. 凭据三态

| 来源 | 值 | 存哪 | 位置 |
|------|----|------|------|
| `~/.netrc` | `netrc` | 系统 netrc | db.py:3182 `get_wandb_key_from_netrc` |
| 自填 key | `custom` | `~/.qorix/wandb_key` | db.py:3220 `set_wandb_api_key` / :3229 `get_wandb_api_key` |
| 未配置 | `unconfigured` | — | `get_wandb_key_source` :3205 |

- `/wandb-config`(main.py:609)设置 key 后即调用 `configure_wandb_and_sync`(ingest.py:4041);
- API 返回 `has_wandb_key` + `wandb_key_source`,前端 `NoRunSelectedState`(no-run-selected-state.tsx:21)据此提示你「去配置 key / 添加项目」;
- UI 读 key 只在**服务端**,前端永不接触明文。

## 2. Run 的发现逻辑(tagged_runs_poll_loop)

```
每 5s:
  known_projects(db.py:3367)→ 逐个 poll_known_projects_for_new_runs(ingest.py:3797)
    └─ _discover_tagged_runs_for_project :3445
         └─ fetch_tagged_runs :3504(GraphQL:_POLL_RUNS_QUERY :3630)
              查 tag="qorix"(TAGGED_RUNS_TAG :107)
    └─ _build_run_data_from_node :3654 → 组装 run 元数据
         └─ schema tag 校验(_schema_version_mismatch :403)
              └─ enqueue_sync(force_sync=True)
```

要点:
- **tag 就是魔法字符串**:`tag:"qorix"` 是发现/过滤的唯一边界(qorix 训练时自动打上);
- 版本 tag:`schema_version:0.3.0`、`schema_version_<table>:*`、`commit:<hash>`——见 sync-engine.md §5;
- 项目列表驱动枚举;为空时降频(`EMPTY_KNOWN_PROJECTS_DISCOVERY_SECONDS=60` ingest.py:111)。

## 3. 项目的增删

| 端点 | 函数 | 位置 |
|------|------|------|
| `GET /known-projects` | `get_known_projects` | main.py:735 |
| `POST /add-project` | `add_project`(写 known_projects) | main.py:742 |
| `POST /remove-project` | `remove_project` | main.py:771 |
| `POST /add-run` | `add_run`(精确指定 entity/project/run_id) | main.py:786 |

`add-run` 走 `discover_and_sync_project`(ingest.py:4057)或直接 `enqueue_sync`,把指定 run 变成 `_active_runs` 的一员。

## 4. 典型操作组合(用户视角)

```
第一次用:
  POST /wandb-config {api_key}         → 配 key + 触发 configure_wandb_and_sync
  GET  /known-projects                 → 显示候选项目
  POST /add-project                    → 开始发现该项目的 qorix runs
  等几秒 → /runs 列表出现新 run → 前端侧边栏亮起可点
持续使用:
  每 5s /runs 轮询;新 run 自动出现并 sync
  停止关注:POST /stop-tracking(run)   → clear_active_run
  永久移除:POST /remove-run → 标记 removed;/delete-run-data 清理本地数据
```

## 5. 重建/升级场景

- `restore_active_runs_from_db`(ingest.py:601):重启后从 `runs` 表恢复仍 active 的 run(依赖已配置的 key);
- 缺 key 时恢复不了 → UI 提示重配;
- `SCHEMA_VERSION` 升级后旧 run 失配:要么被跳过,要么(版本更新)被重新 enqueue 重建(见 sync-engine.md §5);
- 强制重同步:`enqueue_sync(force_sync=True)`(发现循环里的新 run 默认强制一次)。

## 6. 与后端其他模块的边界

- ingest 不直接操作 DB 之外的磁盘(除了 `~/.qorix/code/<slug>/source` 源码产物,ingest.py:300);
- 下载走 wandb 官方 SDK(`wandb.Api(run)`),不手搓 HTTP(仅 `version-check` 用 httpx 打 PyPI,main.py:291);
- 所有并发由 asyncio + 队列控制,无额外线程池(下载并行靠 wandb SDK 内部);thread-pool 遥测只是**展示** qorix 训练端的数据,与自身 IO 无关。