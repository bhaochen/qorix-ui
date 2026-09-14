# 模块地图与符号索引

> qorix-ui 以「后端 18K 行(4 个文件)+ 前端 78 个组件」的紧凑形态存在。本表给出「功能 → 文件:行号」直达索引。后端根 = `src/qorix_ui/`,前端根 = `ui/`。

## 1. 后端速查(server/)

### FastAPI 路由→处理器
```
POST /rollouts                              get_rollouts                main.py:1255
POST /rollouts-discarded                    get_rollouts_discarded      main.py:1583
POST /evals                                 get_evals                   main.py:1936
POST /eval-step-metrics                     get_eval_step_metrics       main.py:2288
POST /sample-details                        get_sample_details          main.py:2532
POST /sample-statuses                       get_sample_statuses         main.py:3315
POST /events/inference-by-group             get_inference_events_by_group   main.py:3377
POST /events/trainer-breakdown              get_trainer_breakdown_events    main.py:3584
GET  /run-summary/{run_path}                get_run_summary             main.py:3642
GET  /run-code/tree/{run_path}              get_run_code_tree           main.py:3947
GET  /run-code/file/{run_path}              get_run_code_file           main.py:3971
GET  /run-code/diff-summary/{left}/{right}  get_run_code_diff_summary   main.py:4013
POST /events/timeline-paginated             get_timeline_paginated      main.py:4054
GET  /events/inflight/{run_path}            get_inflight_generations    main.py:4322
POST /system-metrics/gpu                    get_system_metrics_gpu      main.py:4339
POST /system-metrics/gpu-paginated          get_system_metrics_gpu_paginated main.py:4542
POST /system-metrics/cpu-paginated          get_system_metrics_cpu_paginated main.py:4689
POST /system-metrics/thread-pools-paginated get_thread_pool_metrics_paginated main.py:4830
POST /vllm-metrics/paginated                get_vllm_metrics_paginated  main.py:4959
POST /step-metrics                          get_step_metrics            main.py:5107
POST /step-metrics/multi                    get_step_metrics_multi      main.py:7002
POST /step-times                            get_step_times              main.py:8396
POST /inference-performance                 get_inference_performance   main.py:8430
POST /trainer-performance                   get_trainer_performance     main.py:8992
POST /step-histogram                        get_step_histogram          main.py:9106
POST /step-distribution-over-time           get_step_distribution_over_time  main.py:9286
POST /api/logs                              get_logs                    main.py:9551
GET  /api/logs/summary/{run_path}           get_logs_summary            main.py:9613
GET  /database-info                         database_info               main.py:9644
POST /compact-database(+/status)            start_compact_database      main.py:9656/9650
```
管理类:`/health`:275、`/sync-queue-progress`:280、`/stop-all-syncs`:285、`/version-check`:291(PyPI 巡检)、`/update`:307(pip 自升级)、`/restart`:338(os.execv)、`/sync`:562、`/sync-evals-after-training`:584、`/stop-tracking`:601、`/wandb-config`:609、`/custom-metrics-*`:635-733、`/known-projects`+/add-project:735/742、`/add|remove|drain|set-run-color|rename-run|update-notes|delete-run-data`:786/815/851/887/914/954/992、`/runs`:1052、`/removed-runs`:1116、`/search-runs-by-config`:1155。

### 状态与全局
| 你要找的全局 | 位置 |
|--------------|------|
| FastAPI 应用 | main.py:93 |
| 41 张带 run_id 的表(删除用) | main.py:98-141 |
| 启动三循环启动 | main.py:214-272 |
| 活跃 run 集 / 同步状态 / 同步队列 | ingest.py:191/194/205 `get_active_runs`:511 `enqueue_sync`:3901 |
| 进度上报 | ingest.py `get_sync_queue_progress`:3931 |
| 存储目录 / DB 路径 | db.py:17-28 |

### DuckDB 层
| 功能 | 位置 |
|------|------|
| 连接(单例+只读 cursor) | db.py:91-114 |
| 事务上下文 | db.py:70-88 |
| zstd 解压 | db.py:30 `decompress_blob` |
| 建全部表 | db.py:117-1057 |
| 幂等写入 | `insert_*` 族 db.py:1121-3100 |
| 进度查询 | `get_ingested_tails`:3431 / `get_ingested_steps`:3485 |
| 压缩 DB | `compact_database`:3668 |

### 同步引擎
| 功能 | 位置 |
|------|------|
| W&B 下载 worker(20 并发) | ingest.py `_download_event_zips_parallel`:1166 |
| tail.zip 解析 | ingest.py `EventZipData`:724 / `_download_event_zip_sync`:846 |
| 可回填 block(360 tail) | ingest.py:2079-2082 `sync_events_background`:2089 |
| rollout block(500 step) | ingest.py:2388 `sync_rollouts_blocks`:2652 |
| 差异判定 | ingest.py `run_needs_sync`:2481 / `get_sync_mismatch_reasons`:2425 |
| 版本校验 | ingest.py `_schema_version_mismatch`:403 |
| 源码产物 | ingest.py `_sync_source_artifacts`:300 |

## 2. 前端速查(ui/)

| 功能 | 位置 |
|------|------|
| 路由表 | src/App.tsx:19-29 |
| Provider(QueryClient+Jotai) | app/providers.tsx:16-37 |
| API 基址 / 轮询间隔 | lib/constants.ts:3-4 |
| 查询 hooks(11 组) | hooks/use-run-data.ts(1358 行,入口 :48-:1343) |
| 类型词表 | lib/types.ts(980 行) |
| Jotai atoms | lib/atoms.ts(331 行) |
| 格式化 | lib/format.ts(162 行) |
| 模型 repr 解析 | lib/parse-model-architecture.ts:53 |
| 时间线 span 透视 | components/combined-timeline-chart.tsx `rolloutEventsToSpans`:184 / `pivotInfraEvents`:217 |
| 图表引擎 | components/step-metrics-charts.tsx(7900 行,入口 `StepMetricsCharts`:501) |
| Infra 三子视图 | components/system-metrics-charts.tsx `SystemMetricsCharts`:2613 |
| 拓扑 3D | components/topology-viewer.tsx `TopologyViewer`:1961 |
| Run Code diff | components/run-code-visualizer.tsx `RunCodeVisualizer`:646 |
| 自定义指标拖拽 | components/custom-metrics-view.tsx `CustomMetricsView`:1657 |

## 3. 页面 → 数据源

| 页面 | 主要查询 hooks / 端点 |
|------|----------------------|
| 首页 `/` | `useRunSummaries` + `step-metrics/multi`;`run-code/*`;`api/logs` |
| 指标 `/metrics` | `useStepMetricsMultiRun`:991、`useStepTimes`:1058、`useInferencePerformance`:649、`useTrainerPerformance`:676 |
| 时间线 `/timeline` | `useTimelinePaginated`:526、`useInflightGenerations`:557、`useRolloutEventsByGroup`:580、`useTrainerBreakdownEvents`:605 |
| Rollouts `/rollouts` | `useRollouts`:108、`useSampleDetails`:191 |
| 丢弃 `/rollouts-discarded` | `useRolloutsDiscarded`:135 |
| Topology `/topology` | `parseTopology(summary.setup)` + `useGpuMetricsForTrainerRanks`:710 |
| Infra `/infra` | 分页 infra hooks(`usePaginatedGpuMetrics`:782 等) |
| Evals `/evals` | `useEvals`:162、`useEvalStepMetricsMultiRun`:1161 |
| About `/about` | `useRuns` + `/version-check` |

## 4. 缩略语与词汇

| 词 | 含义 |
|----|------|
| run_path | `entity/project/run_id` 三段(URL 安全编码),DB 主键之一 |
| tail | 事件帧序号(按 5s 上传周期递增),幂等同步单位 |
| block | 写入端的归档块(tail×360 / step×500) |
| summary_id | W&B 摘要 artifact 的 version id,用于跳过未变化的摘要 |
| slug | run 的本地目录名(清洗+sha1) |
| wandb_key_source | `netrc` / `custom` / `unconfigured` |