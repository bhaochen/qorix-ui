# 同步引擎 — 后台循环与队列

> 补充 `backend/ingest.md` 的深层:三个后台循环怎么起、怎么停、怎么与队列协同,以及「发现 → 跟踪 → 同步」的完整状态机。

## 1. 三个循环(全在 ingest.py)

| 循环 | 周期 | 职责 | 位置 |
|------|------|------|------|
| `ingestion_loop` | 5s | 对 `_active_runs` 逐个 `ingest_rollouts` | ingest.py:3391 |
| `tagged_runs_poll_loop` | 5s | 扫描 known projects,发现新 run 就 `enqueue_sync(force_sync=True)` | ingest.py:4153 |
| `ingest_state_reconcile_loop` | 60s | 把 DB 里还在训练但没在跟踪的 run 重新 enqueue;补 mismatch | ingest.py:4182 |

三者开场都要检查 `_compaction_paused`(DB 压缩期间全部暂停,ingest.py:581 置位/:594 复位)。

## 2. 活跃 run 的状态机

`_active_runs`(ingest.py:191)+ `set_active_run`:505 / `get_active_runs`:511 / `clear_active_run`:516:

```
add-run(POST /add-run main.py:786)
  └─ set_active_run(run_path)   → 进 _active_runs
      └─ ingest_rollouts(run_path):3298
           ├─ 首次:保存 metadata → sync_events → sync_rollouts → 写 summary_id
           └─ 后续:比对 mismatch 决定跳过
POST /stop-tracking(main.py:601)
  └─ clear_active_run  → ingest 循环不再轮询
```

- 前端追问的状态(`is_tracking`/`is_syncing`)就是 `_active_runs ⊇ _sync_status` 的投影;
- 重启恢复:`restore_active_runs_from_db`(ingest.py:601)从 `runs` 表把仍为 active 的 run 重新挂上(不丢中断)。

## 3. 同步队列与并发

```
enqueue_sync(run_path, api_key, force_sync) :3901
  └─ _ensure_sync_workers :3891(至多 SYNC_QUEUE_WORKERS=4 个 worker :186)
      └─ _sync_worker :3962   ← 队列里的 jobs 逐个执行
POST /stop-all-syncs(main.py:285)
  └─ stop_all_syncs :3942(_cancelled_syncs 标记 + 队列清空)
GET  /sync-queue-progress(main.py:280)→ get_sync_queue_progress :3931
```

关键常数:下载并发 `PARALLEL_DOWNLOADS=20`(:182)、项目扫描 worker `DISCOVERY_PROJECT_SCAN_WORKERS=8`(:184)。

## 4. 发现新 run(tagged_runs_poll_loop)

`poll_known_projects_for_new_runs`(ingest.py:3797)每 5s:
1. 列出 `known_projects`(db.py:3367);
2. 对每个项目调 `_discover_tagged_runs_for_project`:3445 → `fetch_tagged_runs`:3504 用 GraphQL 查 `tag:"qorix"` 的运行(`_POLL_RUNS_QUERY`:3630);
3. `_build_run_data_from_node`:3654 组装;
4. 逐 run:若 schema 版本匹配且未在跟踪 → `enqueue_sync`。
5. `EMPTY_KNOWN_PROJECTS_DISCOVERY_SECONDS=60`(:111)当项目列表为空时降频。

## 5. 版本/一致性闸门

- 每个 run 必须带 W&B tag `schema_version:0.3.0`(与 `SCHEMA_VERSION` ingest.py:117 一致),否则 `_schema_version_mismatch`:403 跳过;
- `schema_version_<table>:<ver>` 允许逐表版本差异(`TABLE_SCHEMA_VERSIONS`:124-179);
- 运行节点有 `commit:<hash>` → `runs.trainer_commit`;
- **比当前新的 run 会被 enqueue(等待兼容升级重建)**,旧的被跳过,:3474-3479 是判据。

## 6. 故障排查表

| 现象 | 看哪 |
|------|------|
| run 一直 "syncing" | `/sync-queue-progress`;`_sync_status` reasons(`get_sync_mismatch_reasons`:2425) |
| run 不被发现 | W&B tag 缺失/版本不符(:403);known_projects 为空 |
| 下载慢 | `PARALLEL_DOWNLOADS`;网络;zip 巨大 |
| 同步中途停 | `_compaction_paused`(压缩进行中)或 `_cancelled_syncs` |
| 重启后丢跟踪 | `restore_active_runs_from_db` 需要 W&B key(未配置则无法恢复) |