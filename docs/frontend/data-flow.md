# 前端数据流 — 页面 → Hook → 端点 → 图表

> 前端几乎全部由 `use-run-data.ts`(1358 行)的 TanStack Query hooks 驱动:每个 hook = 一个端点 = 一个 queryKey。本篇把「页面如何获得数据」拆成可复制的套路。

## 1. 全局基础设施

- 入口:`src/main.tsx:7-13`(BrowserRouter)→ `src/App.tsx:14` 外壳 + `:19-29` 路由表;
- `providers.tsx:16-37`:`QueryClient`(staleTime 60s、refetchOnWindowFocus off)+ Jotai + TooltipProvider;
- `API_BASE` 空串 = 同源;轮询 `POLL_INTERVAL=5000`(constants.ts:4)。

## 2. Hook → 端点对照(use-run-data.ts)

| Hook | 端点 | 位置 |
|------|------|------|
| `useRuns` | GET /runs(2.5s 发现期加速 1.5s) | :48 |
| `useRemovedRuns` / `useKnownProjects` | GET /removed-runs / /known-projects | :66/:90 |
| `useRollouts` / `useRolloutsDiscarded` | POST /rollouts / /rollouts-discarded | :108/:135 |
| `useEvals` | POST /evals | :162 |
| `useSampleDetails` / `useSampleStatuses` | POST /sample-details / /sample-statuses | :191/:220 |
| `useRunSummary` | GET /run-summary/{run} | :267 |
| `useRunSummaries` | useQueries 批量合并 | :292 |
| `useLogs` / `useLogsSummary` | POST /api/logs / GET /api/logs/summary | :413/:446 |
| `useRunCodeTree` / `useRunCodeFile` / `useRunCodeDiffSummary` | GET /run-code/{tree,file,diff-summary} | :465/:482/:504 |
| `useTimelinePaginated` | POST /events/timeline-paginated(page+interval) | :526 |
| `useInflightGenerations` / `useRolloutEventsByGroup` / `useTrainerBreakdownEvents` | 对应 /events/* 端点 | :557/:580/:605 |
| `useInferencePerformance` / `useTrainerPerformance` | POST /inference-performance / trainer-performance | :649/:676 |
| `useGpuMetricsForTrainerRanks` | POST /system-metrics/gpu(limit 50k) | :710 |
| `usePaginatedGpu/Cpu/ThreadPool/VllmMetrics` | 分页 infra 四端点 | :782/:829/:887/:945 |
| `useStepMetricsMultiRun` | POST /step-metrics/multi(limit 100k, tag/env 过滤) | :991 |
| `useStepTimes` / `useStepMetricSingle` / eval 变体 | POST /step-times / step-metrics | :1058/:1083/:1112/:1161 |
| `useStepHistogram` / `useStepDistributionOverTime` | POST /step-histogram / step-distribution-over-time | :1248/:1281 |
| `useCustomMetricsLayout` / `useCustomMetricsTemplates`(|`useCustomMetricsTemplate`) | 自定义指标三端点 | :1315/:1329/:1343 |

## 3. 三层状态(运行快照)

| 层 | 状态 | 存哪 |
|----|------|------|
| 服务端 | DB 里已入库的数据 | DuckDB(无会话) |
| Query 缓存 | 每个 endpoint 的响应(60s stale) | React Query |
| 本地偏好 | page/选中的 run/可见 runs/EMA 开关… | Jotai atoms(lib/atoms.ts) |

页面不自主决定「展示什么 run」:它读 `selectedRunPathAtom`(每 tab 一个)与 `visibleRunsAtom`,拼出 `allRunPaths` 后批量 `useRunSummaries`(首页 page.tsx:396-414 是标准样板)。

## 4. 页面级装配的三个样板

**图表页(metrics)**:`StepMetricsCharts`(step-metrics-charts.tsx:501)→ 每 run 至多 20 个 `useStepTimes`(:540-639)→ 每个图卡 `MetricChart`:3855 喂 `useStepMetricsMultiRun`:3797;可选 EMA / x 轴 step|time / 直方图 `DistributionOverTimeChart`:2076。

**时间线页(timeline)**:`useTimelinePaginated` → `CombinedTimelineChart`(combined-timeline-chart.tsx:396);右侧 `GroupSampleTimeline`(:2068)/ `TrainerBreakdownContent`(:4209)由 `useRolloutEventsByGroup` + `useTrainerBreakdownEvents` 供稿。

**Rollouts 页**:三件套 `RolloutsView` + `RolloutsSamplePickerSidebar` + `RolloutsMetricsPanel`;deep-link 靠 goToBasePath/goToStepParam(rollouts-view.tsx:236);逐样本详情弹窗 `SampleDetailsDialog`(sample-details-dialog.tsx:178)拉 `useSampleDetails`。

## 5. 刷新节奏与心智

- 全局 5s 轮询;发现 run 时 `/runs` 提到 1.5s;
- `staleTime 60s` → 切 tab 不闪、不重打;显式 invalidate 只在变更操作后(如删除 run 时预清 `/step-metrics/multi` 缓存:use-run-data.ts:1002-1026);
- 全站无 WebSocket:所有"实时"都是 Poll;训练侧 5s 一帧的 tail 决定了你最多慢一个轮询周期看到事件。