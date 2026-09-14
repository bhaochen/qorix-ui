# DuckDB 数据层 — 表家族与查询模式

> 深挖 db.py:表家族的镜像规则、迁移、写入与查询模式。与 `backend/database.md`(结构和写函数表)配合,本篇回答「同一个实体为什么有 4 张表」和「怎么高效读」。

## 1. 表家族:主表 × 四象限

每一实体不是一张表,而是一个**家族**:

```
<实体>            ← 训练主路径
<实体>_eval       ← 训练内/standalone 评测
<实体>_discarded  ← 被丢弃的 rollout(off-policy/零优势/超限)
<实体>_cancelled  ← 被取消请求(权重更新驱逐)
```

实例(`db.py` CREATE 位置):

| 实体 | 主 | eval | discarded | cancelled |
|------|-----|------|-----------|----------|
| prompts | :186 | :773 | :439 | :618 |
| generations | :200 | :791 | :457 | :635 |
| env_responses | :225 | :812 | :484 | :661 |
| tool_calls | :241 | :832 | :502 | :678 |
| samples_data | :293 | :860 | :528 | :703 |
| rollouts_metrics | :311 | :878 | :550 | :724 |
| golden_answers | :338 | :894 | :562 | :735 |
| sample_tags | :350 | :910 | :574 | :746 |
| info_turns | :586 | :926 | :602 | :757 |

设计收益(不重要但值得知道):
- UI 的 rollouts / rollouts-discarded / evals 三页 = **同一渲染管线扫三个象限**,共用 `RolloutsView`(frontend/rollouts-view.tsx:236);
- DB 不做笛卡尔合并,前端按 run+step 分开拉,查询各自精简。

## 2. 系统指标与事件表(高频读)

| 表 | 位置 | 说明 |
|-----|------|------|
| `events_orchestrator` | :121 | 瞬时事件(时间戳+event_type+node) |
| `events_trainer` | :137 | 区间事件(rank/local_rank/gpu_index/microbatch/minibatch) |
| `events_rollout` | :154 | 样本生命周期(lane/phase→span 透视) |
| `events_infra` | :172 | weight_sync / sandbox |
| `system_metrics_gpu` | :362 | gpu 采样(→ Infra 页) |
| `system_metrics_cpu` | :377 | cpu 采样 |
| `vllm_metrics` | :389 | vLLM(server/tp_group_id/tp_size) |
| `system_metrics_thread_pools` | :403 | 线程池遥测 |
| `logs` | :414 | 按 tail 分页,`tail_idx` 追踪 |
| `step_metrics` | :427 | section+metric+value(+group) |

## 3. 光一样快的秘密:读取路径是「区间扫描」

分页查询不在 DB 里做 offset,而是把**时间/步数窗口重算进 SQL WHERE**:

- Timeline:`interval_seconds`(默认 60s)求出 `interval_start/end` → 只读该窗的 orchestral/trainer/rollout/infra(main.py:4054 的 CTE);
- 系统指标同理:page → 时间窗口 → WHERE timestamp BETWEEN;
- Rollouts:按 run+step 精查一到两张表;
- 前端把 `page` 和窗口原子化(见 frontend/state.md),查询参数化,**服务端无游标状态**。

## 4. 幂等写模式的统一签名

所有 `insert_*` 都是「DataFrame → `INSERT ... FROM df ... WHERE NOT EXISTS`」(db.py:1121 起)。三件套:
1. 用 `ingested_*` 表先滤一次(main.py/ingest.py 侧);
2. `WHERE NOT EXISTS` 兜住并发/重复帧;
3. 全在一次 `transaction()`(db.py:70)内提交。

## 5. 状态表的角色(调试入口)

| 表 | 何时更新 | 用在哪 |
|----|---------|--------|
| `ingest_state`(:954) | 每轮同步末尾 | `get_ingest_state`:1061,UI 显示进度 |
| `ingested_tails`(:1017) | 每个 tail 帧插入后 | 去重 + 回补判定(:3431/3460) |
| `ingested_steps`(:1027) | 每步 rollout 插入后 | rollout 去重 |
| `runs`(:968) | 发现/增删/改名/标色 | `/runs` 列表 + per-run 属性 |
| `known_projects`(:1053) | 项目扫描 | 发现循环;

## 6. 迁移与兼容

- `runs.schema_version` + 逐表 `schema_version_<table>`(ingest.py:379 解析 tag)驱动;
- `samples_data_eval.sample_id` 迁移在 db.py:945-950(老数据补 signed int);
- 升级路径:qorix-ui 的 `SCHEMA_VERSION` 变了,旧 run 自动被标为需重同步(见 sync-engine.md §5)。

## 7. W&B 密钥与本地配置

| 键 | 位置 |
|----|------|
| `~/.qorix/wandb_key` + `~/.qorix/wandb_key_source` | db.py:3220-3244,`netrc`/`custom`/`unconfigured` |
| `~/.netrc` 读取 | db.py:3182 `get_wandb_key_from_netrc` |
| DB 路径 | db.py:17-28(`QORIX_DATA_DIR` 可覆盖) |