# 可视化组件 — 图表、时间线、拓扑、模型架构

> 前端"会画"的四个核心引擎:`step-metrics`(指标图)、`combined-timeline`(时间线)、`topology-viewer`(基础设施拓扑)、model 解析。全都吃同一个数据哲学:**区间数据 → span,采样数据 → 序列,字符串 → 解析树**。

## 1. StepMetricsCharts(7900 行的图床引擎)

入口 `StepMetricsCharts`(step-metrics-charts.tsx:501),Props 见 :479-499。

### 图卡类型
| 图卡 | 数据源 | 位置 |
|------|--------|------|
| `MetricChart` | `useStepMetricsMultiRun` | :3855(curves per run) |
| `EvalMetricChart` | `useEvalStepMetricsMultiRun` | :2629 |
| `DistributionOverTimeChart` | `useStepDistributionOverTime` | :2076 |
| `InferencePerformanceChartCard` / `AreaChartCard` | `useInferencePerformance` | :5761/:5804 |
| `TrainerPerformanceChartCard` / `AreaChartCard` | `useTrainerPerformance` | :7813/:7858 |

### 三个通用技巧
- **IQR 离群压扁**:`computeIQRBounds`:187 拿数据的 IQR 界钳制 y 轴,噪点不炸图;
- **EMA**:`overviewShowEmaAtom` / `metricsShowEmaAtom` 开 EMA 平滑;
- **escapeHtml**:`:235`,run 标签等外来文本插 HTML 时转义(防注入 + 显示稳定)。

## 2. CombinedTimelineChart(时间线的透视画家)

`CombinedTimelineChart`(combined-timeline-chart.tsx:396)Props :367-394。核心两个纯函数:
- `rolloutEventsToSpans`:184——把后端给的 `RolloutEvent start/end` 对 **pivot 成 span**(一次生成的起止时间条),并按 generation/tool/response 分流;
- `pivotInfraEvents`:217——`weight_sync`/`sandbox` 生命周期事件也变 span;
- 辅助:`buildGenerationDetails`:242(把 generations 时序并成 tooltip)、`buildCloseEventWindows`:315(合并过密事件,防 2000 并发把图挤爆);
- 渲染:uPlot 轴 + 自定义 lane 行,`RolloutSpan`:159 是每个样本条第单位。

## 3. SystemMetricsCharts(infra)

`SystemMetricsCharts`(system-metrics-charts.tsx:2613)Props :109(跨 run 的 gpu/cpu/vllm/threadPool 数据数组 + 角色/节点模式):
- `GpuMetricChart`(gpu-metric-chart.tsx:307,uPlot + IQR 离群过滤,`variant="default"|"timeline"`,x 轴 `elapsed|clock`);
- 分组:`NodeGroupCollapsible`:2304 / `RoleSection`:2347(按 trainer/inference 角色折叠);
- `SeriesRoleMap`(gpu-metric-chart.tsx:301)把 run→角色→名称映射成确定颜色。

## 4. TopologyViewer(3D 拓扑)

`TopologyViewer`(topology-viewer.tsx:1961)的数据是 `summary.setup` 的字符串:
- `parseSetupJson`:131(`JSON.parse` 两次,防嵌套字符串);
- `parseTopology`:148 → 集群节点 + trainer/inference GPU 角色映射;
- 渲染:`FloorOverviewScene`:1420 / `NodeBox`:1270 / `GpuCard`:403(每张卡显示利用率/显存),`CameraController`:1525 让用户旋转缩放;
- `InfoPanel`:1621 悬停信息。**没有真实图数据库**——它就是 3D 卡片的 `nn` 布局。

## 5. ModelArchitectureViewer(模型结构图)

`ModelArchitectureViewer`(model-architecture-viewer.tsx:479)输入是任意 `nn.Module.__repr__` 字符串:
- `parseModelArchitecture`(lib/parse-model-architecture.ts:53):**栈式解析器**读 repr 的缩进层级 → `ModelNode[]`;`collapse repeats`:198 把 "xN" 折叠;`estimateParamsFromArgs`:266 按参数形态估算参数;
- `LayerCategory`(:5)+ `getLayerCategory`:415 给模块分类(attention/ffn/embed/norm...),`CATEGORY_COLORS`:504 上色;
- 渲染:`FlowDiagram`:294(横向流) + `SummarySidebar`:325(参数统计)。

## 6. CustomMetricsView(用户自制仪表盘)

`CustomMetricsView`(custom-metrics-view.tsx:1657):
- `buildPlotCatalog`:106 生成可拖入的图卡目录(step/reward/samples/advantage/evals/dist-over-time);
- dnd-kit 排序(`SortablePlotCard`:991 / `SortableSection`:1361);布局改动 debounced POST 到 `/custom-metrics-layout`(:677-685);
- 模板存取:`useCustomMetricsTemplates`/`useCustomMetricsTemplate`(use-run-data.ts:1329/:1343)加载/另存布局。

## 7. 跨图组件共用协议

- **每张图都能多 run 叠加**:`visibleRunsAtom` 决定画几个 series,颜色取 `RunInfo.color`;
- **悬停联动**:`syncedCursorAtom`(atoms.ts:331)在首页多图间同步十字光标;
- 所有数字格式化统一走 lib/format.ts(HH:MM:SS / 自适应精度 / 人类可读时长)。