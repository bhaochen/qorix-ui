# 同步引擎（ingest.py）

> `server/ingest.py`（4265 行）是本地到 W&B 的数据搬运工：**发现 → 轮询 → 下载 zip → zstd 解压 → 按 tail/step 幂等去重 → 写 DuckDB**。全程 asyncio + 队列 + 动态 worker。

## 1. 常量与调度节奏

| 常量 | 值 | 用途 |
|------|-----|------|
| `POLL_SECONDS` | 5（:105） | 常规轮询 |
| `TAGGED_RUNS_POLL_SECONDS` | 5（:106） | tagged runs 轮询 |
| `INGEST_STATE_RECONCILE_SECONDS` | 60（:108） | 状态对账 |
| `EMPTY_KNOWN_PROJECTS_DISCOVERY_SECONDS` | 60（:111） | 空项目兜底发现 |
| `SCHEMA_VERSION` | "0.3.0"（:117） | 数据格式版本 |
| `PARALLEL_DOWNLOADS` | 20（:182） | 并行下载数 |
| `DISCOVERY_PROJECT_SCAN_WORKERS` | 8（:184） | 发现扫描 worker |
| `SYNC_QUEUE_WORKERS` | 4（:186） | 同步 worker 数 |
| `TAGGED_RUNS_TAG` | "qorix"（:107） | 发现标签 |
| 远程产物 | `code/source.zip` / `code/metadata.json`（:187-188） | 源码快照 |

## 2. 发现（Discovery）

- `discover_and_sync_project`(:4057)：对 `known_projects` 逐个跑 GraphQL（`_POLL_RUNS_QUERY` :3630 —— `PollRuns`：`project(name, entityName){ runs(filters, after, first, order) }`）。
- 过滤：tag 匹配 + 排除 `qorix-ignore`（:3758-3760）+ schema_version 匹配（:3761-3762）。
- `_build_run_data_from_node`(:3654) 构造 run_data（commit/schema_version 经 `_extract_commit_and_schema_version` :3683-3689 解析）。
- `poll_project_for_new_runs` → `discover_new_runs` 反序列化、去重；`poll_known_projects_for_new_runs`(:3797) 每 5s、`per_page=500`，只处理本地 DB 尚未存在的 run。

## 3. 同步（Sync）

主流程 `_sync_events_for_run`(:2909-2940)：
1. acquire per-run `asyncio.Lock`（:2909-2912，防止并发写同一 run）；
2. 拿 wandb api/run（:2914-2917）；
3. fetch summary/config JSON（:2921-2928）→ `save_run_metadata`(:2932)；
4. `_sync_source_artifacts`(:300)：下载 `code/source.zip` + `metadata.json`。

后台常驻 `sync_events_background`(≈:2089)：
- 每族 counts 字典 + `inserted_tails` 去重集合；
- `_download_rollout_zips_parallel`(:1414) 用 ThreadPoolExecutor + `PARALLEL_DOWNLOADS=20`；
- 事件数据类型 `EventZipData`(:724) 承载 orchestrator/trainer/rollout/infra/cpu/gpu/vllm/thread_pools + `_discarded/_cancelled/_eval` 变体 + start/end/min/max step 元数据；`RolloutZipData`(:1200) 解 rollout parquet。

**尾部去重**：`_filter_events_by_tails` + `missing_tails`(:1629-1639) 用 `ingested_tails`/`ingested_steps` 记录已入库 tail/step，只插新 tail；`tail_idx`/`step` 是幂等键。

**eval 家族**：训练后 `_download_prompts_eval`/`_download_generations_eval`/`_download_env_responses_eval`/`_download_tool_calls_eval`/`_download_samples_data_eval`/`_download_rollouts_metrics_eval`（:2000-2030 区）→ 写 `*_eval` 表；`sync_evals_after_training_background`(:3091)。

## 4. 调度与对账

| 任务 | 位置 | 说明 |
|------|------|------|
| `tagged_runs_poll` | :4153 | 每 5s 轮询 |
| `ingest_state_reconcile_loop` | :4182 | 每 60s 用 `run_needs_sync`/`get_sync_mismatch_reasons` 比对 DB 与 W&B summary（last_gen_step/last_event_zip_idx/last_block_idx/last_rollout_block_idx/summary_id 等），不匹配重新入队 |
| `_ensure_sync_workers` | :3891 | 按队列水位动态起 worker |
| `stop_all_syncs` | :3942 | 全部停止 |

- 全局状态（:190-230）：`_active_runs`、`_sync_status`、`_sync_queue`（asyncio.Queue）+ `_sync_queue_pending`（去重集合）、`_known_projects`、W&B API client 缓存 `_cached_api`。
- W&B API：`_get_wandb_api`(:232) 缓存；`_get_fresh_run`(:247) 按 run 创建；`WANDB_SILENT=true`（:19）。
- 元数据持久化：`build_run_metadata`(:422)/`save_run_metadata`(:485)。

## 5. W&B 事件到表映射（契约）

- 事件族经 zip artifact 传输，正文列为 **zstd 压缩**（`decompress_blob`，ingest.py:30 / db.py:30）。
- 列中含 `tail_idx`/`step`/`sample_idx`/`completion_idx` 分片键 → 落到 `ingested_tails`/`ingested_steps` 幂等键。
- zip 内 orchestrator/trainer/infra 事件 → `events_orchestrator/testing…`；rollouts → parquet。
- eval 样本走 `_eval` 分表；丢弃/取消走 `_discarded`/`_cancelled`。

## 6. 调试入口

- UI 侧 `/sync-queue-progress`（main.py:280）、`/stop-all-syncs`、`/drain-run`（main.py:851）可干预同步行为。
- `wandb login` 后 `/wandb-config`（main.py:609）写入 API key；`configure_wandb_and_sync`(:4041) 存 key 后对 known projects 轮询并 enqueue。