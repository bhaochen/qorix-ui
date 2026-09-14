# 页面功能

> 8 个可视化页面 + 首页。每个页面由「页面组件（app/*/page.tsx）→ hooks → FastAPI 端点 → DuckDB 表」链路支撑。

## 1. 首页 `/`（ui/app/page.tsx:1004）

- 概览大屏：选择 run（sidebar）、自定义 plot 卡片**拖拽排序**（DnD，`PlotSelectPopover`/`SortablePlotCard`/`buildPlotCatalog`）、EMA 叠加、run 对比。
- 数据：`/runs`、`/run-summary/{run}`（tables/config/最近 tail）、`/run-code/*`（源码树）、`/logs`（日志面板）、`/custom-metrics-layout`（布局持久化）。
- 控件：RunConfigPanel（配置面板）、RunCodeVisualizer、LogsViewer、NoRunSelectedState（空态）、备注编辑、切换全部/可见 run。

## 2. Metrics `/metrics`（ui/app/metrics/page.tsx:616）

- step 指标折线/柱状图（uplot）：x 轴 step/time 切换、EMA 叠加、max step/time 裁剪、outlier 过滤、`custom-metrics` 看板布局。
- `formatDurationHms` + `step-metrics-charts`。
- 端点：`/step-metrics`、`/step-metrics-multi`、`/custom-metrics-*`。
- 图表形态：`timing_step_total`、loss/entropy/kl、`rollout/*` 等（键 = section/group/metric）。

## 3. Timeline `/timeline`（ui/app/timeline/page.tsx:1107）

- 训练时间线合并图：trainer/orchestrator/infra 事件**泳道**（`combined-timeline-chart`，SVG + dense canvas 双模式），按 run/step 导航。
- 事件下钻：点击事件 → `selectedTrainerEvent`/`selectedInferenceRequest` atom → 详情。
- 端点：`/events/timeline-paginated`、`/events/inference-by-group`、`/events/trainer-breakdown`、`/events/inflight/{run}`。

## 4. Rollouts `/rollouts`（ui/app/rollouts/page.tsx:390）

- 各训练 step 的 rollout 样本：prompt/generation 文本、reward/advantage、golden_answers（参考答案）、sample_tags。
- 详情下钻：`rollouts-view` + `sample-details-dialog`（调 `/sample-details`，completion 维度）。
- 端点：`/rollouts`（step 分页）、`/sample-details`。

## 5. Rollouts Discarded `/rollouts-discarded`（ui/app/rollouts-discarded/page.tsx:581）

- 被丢弃样本回看（`_discarded` 家族表）——训练中 `discard_group_zero_advantage` 丢弃的样本。
- 端点：`/rollouts-discarded`。

## 6. Topology `/topology`（ui/app/topology/page.tsx:75）

- 3D 拓扑图（react-three-fiber）：run/节点连线、悬停高亮（`topology-viewer`）。
- 数据来源：`/runs` 数据（节点拓扑）。

## 7. Infra `/infra`（ui/app/infra/page.tsx:1181）

- GPU/CPU/线程池/vLLM 使用率时间序列 + 分页 + 时间窗（`system-metrics-charts`、`gpu-metric-chart`）。
- 端点：`/system-metrics/gpu`、`/system-metrics/gpu-paginated`、`/system-metrics/cpu-paginated`、`/system-metrics/thread-pools-paginated`、`/vllm-metrics/paginated`。

## 8. Evals `/evals`（ui/app/evals/page.tsx:632）

- eval 指标：reward 分布、per-eval step 汇总、样本 filtering（`sample_idx`/step/env）、completion 详情、状态。
- 端点：`/evals`、`/eval-step-metrics`、`/sample-details`、`/sample-statuses`。
- 数据：`*_eval` 家族表（`COUNT(DISTINCT sample_idx)` 统计维度）。

## 9. About `/about`（ui/app/about/page.tsx:97）

- 版本信息/使用说明；`/version-check`。

## 10. 共同交互

- **sidebar**（app-sidebar.tsx:2113）：8 页面入口 + run 列表（`SIDEBAR_MAX_RUN_NAME_CHARS=16` 后缀截断）+ 设置对话框（W&B 配置、`database-dialog` DB 压缩、暗色模式）。
- **空态**：NoRunSelectedState —— 未选 run 时引导选择。
- **多 run**：`visibleRunsAtom` 控制对比集合，所有查询 run 维度缓存。