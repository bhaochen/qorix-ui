# FastAPI 路由（main.py）

> `server/main.py`（9699 行）承载全部后端：静态 UI 挂载、REST 数据端点、sync 控制、自定义指标 CRUD。前端每个页面的数据都来自下面这张表。

## 1. 应用组装

- 静态 UI：挂载 `src/qorix_ui/static`（main.py:179-194），SPA catch-all 回退 index.html（:194）。
- `log_requests` 中间件(:204)；`_startup`(:215)；健康检查 `/health`(:275)。
- 请求模型（:354-416）：`SyncRequest`、`StopTrackingRequest`、`DeleteRunDataRequest`、`WandbConfigRequest`(:366)、`AddRunRequest`、`RemoveRunRequest`、`DrainRunRequest`、`AddProjectRequest`、`RemoveProjectRequest`、`SetRunColorRequest`、`RenameRunRequest`、`UpdateNotesRequest`、`RolloutsRequest`(:406)、`RolloutsDiscardedRequest`(:411)、`EvalsRequest`(:416)。

## 2. 路由全表（63+）

### 健康 / 版本 / 控制
| 路由 | 方法 | 行号 | 用途 |
|------|------|------|------|
| `/health` | GET | 275 | 健康检查 |
| `/version-check` | GET | 307 | 版本检查 |
| `/update` | POST | 338 | 应用自更新 |
| `/restart` | POST | 351 | 重启应用 |
| `/sync-queue-progress` | GET | 280 | 前台轮询 sync 队列进度 |
| `/stop-all-syncs` | POST | 285 | 停止全部 sync |

### 同步 / 配置
| 路由 | 方法 | 行号 | 用途 |
|------|------|------|------|
| `/sync` | POST | 562 | 手动完整 sync 某 run |
| `/sync-evals-after-training` | POST | 584 | training 后 eval 消费者 |
| `/stop-tracking` | POST | 601 | 停止追踪某 run |
| `/wandb-config` | POST | 609 | 设置 W&B API key |
| `/known-projects` | GET | 735 | 已知项目表 |
| `/add-project` | POST | 742 | 添加项目（触发发现） |
| `/remove-project` | POST | 771 | 移除项目 |
| `/add-run` | POST | 786 | 新增 run 到监控 |
| `/remove-run` | POST | 815 | 移除 run |
| `/drain-run` | POST | 851 | 排空已移除 run 的数据 |
| `/set-run-color` | POST | 887 | 设置 run 颜色 |
| `/rename-run` | POST | 914 | 重命名 run |
| `/update-notes` | POST | 954 | 更新备注 |
| `/delete-run-data` | POST | 992 | 删除 run 全部数据 |

### 数据查询
| 路由 | 方法 | 行号 | 用途 |
|------|------|------|------|
| `/runs` | GET | 1052 | 全部 run + 状态 |
| `/removed-runs` | GET | 1116 | 已移除 run 列表 |
| `/search-runs-by-config` | POST | 1155 | 按 config JSON 搜索 |
| `/rollouts` | POST | 1255 | rollouts（step 分页） |
| `/rollouts-discarded` | POST | 1583 | 被丢弃 rollouts |
| `/evals` | POST | 1936 | eval 数据（eval_name 过滤、sample 过滤） |
| `/eval-step-metrics` | POST | 2288 | eval 的 step-wise 指标 |
| `/sample-details` | POST | 2532 | 单样本详情（completion 维度） |
| `/sample-statuses` | POST | 3315 | 批量 sample 状态 |
| `/events/inference-by-group` | POST | 3377 | 按 group 的 inference 事件 |
| `/events/trainer-breakdown` | POST | 3584 | trainer 事件分步 breakdown |
| `/run-summary/{run_path}` | GET | 3642 | run 摘要（tables/config/最近 tail） |
| `/run-code/tree/{run_path}` | GET | 3947 | 源码树 |
| `/run-code/file/{run_path}` | GET | 3971 | 源码文件内容 |
| `/run-code/diff-summary/{left_run_path}` | GET | 4013 | 双 run 源码差异 |
| `/events/timeline-paginated` | POST | 4054 | 时间线分页（trainer/orchestrator/infra 混合） |
| `/events/inflight/{run_path}` | GET | 4322 | 在途（非持久）事件 |
| `/system-metrics/gpu` | POST | 4339 | GPU 最新聚合 |
| `/system-metrics/gpu-paginated` | POST | 4542 | GPU 分页 |
| `/system-metrics/cpu-paginated` | POST | 4689 | CPU 分页 |
| `/system-metrics/thread-pools-paginated` | POST | 4830 | 线程池分页 |
| `/vllm-metrics/paginated` | POST | 4959 | vLLM 指标 |
| `/step-metrics` | POST | 5107 | step 训练指标（sum/mean/min/max） |
| `/step-metrics-multi` | POST | — | 多 run step 指标 |
| `/step-histogram` / `/step-distribution-over-time` | POST | — | 直方图 / 分布随时间 |
| `/eval-step-metrics-multi` | POST | — | 多 run eval step 指标 |
| `/logs` | POST | — | run 日志分页 |

### 自定义指标
| 路由 | 方法 | 行号 |
|------|------|------|
| `/custom-metrics-layout` | GET/PUT | 635/648 |
| `/custom-metrics-templates` | GET/POST/PUT/GET{id}/PATCH/DELETE | 657/671/685/698/713/726 |

## 3. 关键端点实现要点

- **`/evals` 与 `/eval-step-metrics`**：对 `generations_eval` 用 `COUNT(DISTINCT sample_idx)` 与 `COUNT(DISTINCT sample_idx*10000 + completion_idx)` 统计样本/completion 数；`rollouts_metrics_eval` 按 step 聚合 AVG/STDDEV_SAMP/MIN/MAX。sample 过滤支持 `sample_idx`/`start_step`/`end_step`/env 过滤。
- **`/step-metrics`**（:5107 区）：按 step 聚合 `events_trainer` 中 `start_time/end_time` → 计算 `timing_*` 每个 microbat mean/min/max（`timing_step_total` 等）；step boundaries 由各 step 的 MIN(start_time)/MAX(end_time)（:6437-6448）。
- **`/events/timeline-paginated`**（:4054）：跨 trainer/orchestrator/infra 事件合并分页，供 Timeline 泳道图。
- **`/system-metrics/*-paginated`**：GPU/CPU/线程池采样分页 + 时间窗。
- 静态 SPA catch-all：前端 BrowserRouter 的任意 URL 由 FastAPI 回退到 index.html。

## 4. 前端调用链路

```
useRuns / useRunSummary   → GET  /runs, /run-summary/{path}
useRollouts / useEvals    → POST /rollouts, /evals
useStepMetrics            → POST /step-metrics, /step-metrics-multi
useSystemMetrics          → POST /system-metrics/{gpu,cpu,...}-paginated
useRunCodeData            → GET  /run-code/{tree,file,diff-summary}/{path}
useCustomMetricsLayout    → GET/PUT /custom-metrics-layout
```

全部前端 hooks 以 TanStack Query 轮询消费（见 frontend/overview.md）。