# 前端状态 — Jotai atoms 与多 run 比较模型

> 前端几乎所有「要不要显示」「显示谁」「怎么显示」都是**客户端偏好**,不是服务器状态。本篇把 `lib/atoms.ts`(331 行)的原子全拆开,并解释「比较」为何只发生在局部 state。

## 1. 原子四大家族

### 全局偏好
| 原子 | 内容 | 位置 |
|------|------|------|
| `darkModeAtom` | 暗色模式 | atoms.ts:11 |
| `wandbApiKeyAtom` | (存输入框,不触后端明文持久化) | :15 |
| `wandbConfigDialogOpenAtom` / `knownProjectsDialogOpenAtom` | 对话框开关 | :16/:19 |

### Run 选择与可见性(最重要)
| 原子 | 语义 | 位置 |
|------|------|------|
| `selectedRunPathAtom` | **每个 tab 一个 active run**,localStorage 持久化(`selected-run-path`),**刻意不跨 tab 同步** | :33-45 |
| `visibleRunsAtom` | 勾选显示在图表上的 runs 集合,localStorage | :48 |
| `isTrackingAtom` / `isSyncingAtom` | 从 /runs 投影的布尔(不单独存储) | :54-55 |
| `createPerRunAtom` | 把任意 atom 变成「按 selectedRunPath 分键」的派生原子 | :57-82 |

> 为什么每 tab 独立选中 run?因为你在 metrics 看 runA、在 rollouts 看 runB 是常见场景,被选中是**页面局部心智**,不是全局事实。想全局比较时用 `visibleRunsAtom` 勾选多个。

### 页面级(按页)
- 首页:`overviewShowCodeViewAtom/:89`、`overviewShowLogsViewAtom/:90`、`overviewShowEmaAtom/:91`、`overviewEmaSpanAtom/:92`、`overviewShowAllRunsAtom/:93`(all vs selected)、`overviewPlotsAtom/:99`(自定义布局默认集 `DEFAULT_OVERVIEW_PLOTS`:94-98);
- 指标:`metricsShowEmaAtom/:105`、`metricsXAxisModeAtom/:108`(step|time)、`metricsMaxStepAtom/:110`、`metricsChartFiltersAtom/:125`、`metricsIntervalAtom/:128`;
- 时间线:`timelinePageAtom/:131`、`inferenceHighlightDiscardedAtom/:145`(高亮被丢弃请求)、`selectedTrainerEventAtom/:161`、`selectedInferenceRequestAtom/:169`(点开详情用);
- Rollouts:`rolloutsSelectedStepAtom/:135`、`rolloutsSelectedMetricsAtom/:172`、`rolloutsSelectedGroupIdAtom/:210`、`rolloutsSelectedSampleIdxAtom/:218`、`rolloutsSamplePickerViewModeAtom/:238`(group|sample)、`rolloutsRenderOptionsAtom/:316`、`rolloutsFormatThinkAtom/:320`(展示 think 块);evals 同族 :264-282;
- Infra:`infraViewTabAtom/:286`(metrics|topology|model)、`infraPageAtom/:287`、`infraRoleModeAtom/:295`、`hoveredRunIdAtom/:312`、`syncedCursorAtom/:331`(跨图十字光标)。

## 2. 比较模型:两种比较,两种存储

| 比较 | 怎么存 | 为什么 |
|------|--------|--------|
| **图表叠加比较**(多 run) | `visibleRunsAtom`,ATOM 级 | 无方向性,任何 run 都能「也在图上」;每 run 有自己的颜色 |
| **文件/配置 diff 比较**(当前 vs 选定) | `selectedCompareRun` **组件局部 state** | 有方向(compare→current),且一次只比一对;放 atom 会污染其它页 |

所以:
- `RunConfigCompareDialog`(run-config-compare-dialog.tsx:988)与 `RunCodeCompareDialog`(run-code-compare-dialog.tsx:102)各自持有 `selectedCompareRunId` local state;
- `RunConfigPanel`(run-config-panel.tsx:663)单 run 浏览,把 config 按 `categorizeConfigs` 分类展示(model/environments 默认展开);
- 代码 diff 的「方向」由 compare-run → current-run 定义(RunCodeVisualizer 注释:703-706)。

## 3. 状态的"生命线":poll → query → atom

```
服务端(5s)─→ React Query 缓存 ─→ 组件读缓存渲染
本地输入(页面交互)─→ setAtom ─→ query 依赖 atom 重新拉取(如 timelinePageAtom 入 queryKey)
```

**没有"全局 store 同步后端"模式**:run 的元数据变化(改名/标色/remove)走后端 REST,再由轮询刷新;前端原子只是视图偏好。

## 4. 深链接(可分享 URL)

`RolloutsView` 支持 `step` 参数 deep-link:`goToBasePath/goToStepParam`(rollouts-view.tsx:236)——把 selectedRunPath+step 编码进 URL,刷新/分享保持位置。颜色 `run-color-picker.tsx:157` 与命名 `rename-run`(后端)都属于「元数据变更会落 `runs` 表并广播到下次 /runs 轮询」。

## 5. 常见坑

- 想让两个 tab 看同一个 run:改 `selectedRunPathAtom` 的跨 tab 同步。默认不这么做是为了不互相打断;
- `visibleRunsAtom` 失效时检查 localStorage 旧数据结构(版本升级);
- chart 叠太多 run:每个 run 一条 series,颜色易混淆——用 `RunColorPicker` 换色;
- EMA/interval 是 atom 不算数据,导出图表需先固化参数。