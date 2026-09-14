# 数据生命周期 — 一条训练事件如何走到图表

> 完整链路:`W&B artifact → zip → parquet → DuckDB → FastAPI → React Query → 图表`。所有「数据看不到」「缺尾巴」「一直 syncing」的问题都发生在这条链的手机处,本篇按停靠点逐帧讲清。

## 1. 一图流

```
训练进程(qorix)
  └─ events/tail.zip(每 5s 一帧)+ rollouts/block 归档
        │  W&B 托管
        ▼
qorix-ui 启动:FastAPI startup(main.py:214)
  ├─ recover_from_failed_compaction()
  ├─ create_task(ingestion_loop)            ← 每 5s
  ├─ create_task(tagged_runs_poll_loop)     ← 每 5s(发现新 run)
  ├─ create_task(ingest_state_reconcile_loop) ← 每 60s
  └─ restore_active_runs_from_db(api_key)    ← 重连上次跟踪的 run
        │
        ▼  ingest.py
  sync_events_background:2089   ← 先 tail.zip,再补齐 block
  sync_rollouts_blocks:2652     ← steps/tail.zip + block_live.zip
        │
        ▼  db.py insert_*  族(幂等,tail_idx/step 去重)
  DuckDB(~/.qorix/qorix.duckdb,40+ 表)
        │
        ▼  main.py 只读查询(stateless)
  FastAPI JSON
        │
        ▼  ui/hooks/use-run-data.ts(TanStack Query,5s 轮询)
  React 组件 → 图表
```

## 2. 按帧同步:tail 是最大的幂等键

- **帧 = 一次上传周期**:qorix 训练端默认每 `event_upload_interval_seconds=5s` 把增量事件打一个帧(tail 帧,tail.zip 即最新帧),每帧带 `metadata.json` 给出该帧覆盖的 `[min_tail_idx, max_tail_idx]`;
- qorix 侧 `event_tail_window_seconds=60` 决定 tail.zip 会话内可回溯的历史(回补用的依据),**不是帧长**;
- 已归档帧按 `blocks` 分:`TAILS_PER_BLOCK=360`(ingest.py:2079,注释按 5s/帧 ≈ **30 分钟/块**);
- `sync_events_background`(ingest.py:2089)流程:
  1. 下载 `tail.zip`;
  2. 对 `get_ingested_tails`(db.py:3431)求差,找出缺失帧;
  3. 若缺失帧已超出 tail 可回溯范围,`_calculate_blocks_for_tails`(:2082,用 `// TAILS_PER_BLOCK` 定位)算出对应 block,逐 block 下载补齐。
- `EventZipData`(ingest.py:724)一次性读 zip 内全部 parquet:`orchestrator/trainer/events_rollout/events_infra/gpu/cpu/vllm/thread_pools/logs` 及 `*_discarded`(ingest.py:878-1151),`inflight.json` 快照(:779)。

## 3. 按步同步:rollout 载荷

- rollout 数据(qorix 的 `rollouts/*.parquet`)也要按步回填:`steps/tail.zip` 给 `[min_step, max_step]`;
- 归档是 `rollouts/block_live.zip` + 历史 block(`STEPS_PER_ROLLOUT_BLOCK=500`,ingest.py:2388);
- `sync_rollouts_background`(ingest.py:2893)与 events 回填**互不阻塞**,但共享同一个串行写连接;
- 一次同步结束把 `summary_id` 写回 `ingest_state`——下次轮询发现 summary 版本没变就跳过(`run_needs_sync`:2481)。

## 4. 幂等写入的判决依据

`_insert_event_zip_data`(ingest.py:1560)→ `_filter_events_by_tails`:1450 先用 `ingested_tails` 过滤掉已经见过的帧,再批量 `insert_*`。判决链:

```
tail_idx ∈ ingested_tails ?  跳过插入
step  ∈ ingested_steps   ?  跳过该 step 的 rollout 表
summary_id 未变化       ?  跳过本次整个同步
```

所以「重复拉取同一 zip」是无害的——DB 里不会重复。**查询侧**:`decompress_blob`(db.py:30)在读取时 zstd 解压。

## 5. 回看数据缺了,按哪里查

| 症状 | 第一排查点 |
|------|-----------|
| Timeline 少了某段时间 | `get_ingested_tails` 是否覆盖该帧;block 下载是否中断(`_compaction_paused` 期间暂停) |
| 某 step 的 rollouts 空 | `get_ingested_steps`(db.py:3485)是否含该 step;`ingested_steps` 写入时机 |
| 一直 syncing 不收敛 | `/sync-queue-progress`(:280);`_sync_status` 的 mismatch reasons(`get_sync_mismatch_reasons`:2425) |
| run 直接被跳过 | `_schema_version_mismatch`:403 —— tag `schema_version:` 与 `SCHEMA_VERSION="0.3.0"`(ingest.py:117)不一致 |
| 数据只到旧时刻 | qorix 侧没再产出 tail.zip(训练结束后不再有新帧) |

## 6. 数据库压缩不影响链路

`compact_database`(db.py:3668)把整个 DuckDB 导出再导入回收空间;期间 `_compaction_paused` 会让三个循环暂停(ingest.py:581/594),`/compact-database/status` 轮询进度。**压缩失败会留下待恢复标志**,启动时 `recover_from_failed_compaction`(:3645)清理。