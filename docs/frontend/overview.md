# 前端架构

> React 19 + Vite 6 + **TanStack Query v5**（数据轮询）+ **Jotai**（per-run UI 状态）+ Tailwind 4 + shadcn/radix，图表用 uplot 与 three.js。构建产物直接落到后端静态目录，一条命令 `npm run build` 即可打包进 Python 包。

## 1. 技术栈（ui/package.json）

| 类别 | 选型 |
|------|------|
| 框架 | react 19.2, react-router-dom v7 |
| 数据 | TanStack Query v5、jotai（+atomWithStorage） |
| 样式 | Tailwind 4、tailwind-merge、clsx、cva |
| UI | shadcn/radix-ui（dialog/dropdown/tooltip/toggle…） |
| 拖拽 | dnd-kit |
| 图表 | uplot（指标/时间线）、three + @react-three/fiber + @react-three/drei（3D 拓扑） |
| 其他 | lucide-react、react-markdown、kaTeX |

## 2. 构建管线（ui/vite.config.ts）

- alias `@` → `ui/`（vite.config.ts:8-12）——import 一律 `@/components/...`。
- **`outDir` → `../src/qorix_ui/static`**（vite.config.ts:16-19）：构建后 FastAPI 直接挂载（main.py:179-194），SPA catch-all 回退 index.html。
- dev server 端口 3000；生产走 Python 静态服务。
- 入口 `ui/src/main.tsx`(13)：StrictMode + BrowserRouter + `<App/>`；`ui/src/App.tsx`(33) 注册全部路由。

## 3. 路由（App.tsx:19-28）

| 路径 | 页面 | 文件 | 行数 |
|------|------|------|------|
| `/` | HomePage | ui/app/page.tsx | 1004 |
| `/metrics` | MetricsPage | ui/app/metrics/page.tsx | 616 |
| `/timeline` | TimelinePage | ui/app/timeline/page.tsx | 1107 |
| `/rollouts` | RolloutsPage | ui/app/rollouts/page.tsx | 390 |
| `/rollouts-discarded` | RolloutsDiscardedPage | ui/app/rollouts-discarded/page.tsx | 581 |
| `/topology` | TopologyPage | ui/app/topology/page.tsx | 75 |
| `/infra` | InfraPage | ui/app/infra/page.tsx | 1181 |
| `/evals` | EvalsPage | ui/app/evals/page.tsx | 632 |
| `/about` | AboutPage | ui/app/about/page.tsx | 97 |

## 4. Hooks（数据层）

- `ui/hooks/use-run-data.ts`(1358)：核心轮询 hook 组：
  - `useRuns`（run 列表 + DnD 排序）、`useRunSummary`/`useRunSummaries`（GET `/run-summary/{run}`）；
  - `useRollouts` / `useEvals`（POST `/rollouts`、`/evals`）；
  - `useStepMetricSingle`/`useEvalStepMetricsMultiRun`(:1161) 多 run 聚合——用 `useRef(prevRunPaths)` + `setQueryData` 预置 cache（“移除变化时复用旧数据”模式）；
  - Query key 按 run 维度缓存；轮询间隔 `POLL_INTERVAL` + `invalidateQueries`。
- 其余 hooks 同构：`use-step-metrics.ts`(2654)、`use-system-metrics.ts`(3249)、`use-run-code-data.ts`(1255)、`use-custom-metrics-layout.ts`(2130)、`use-run-summary.ts`——统一走对应端点 + `useQuery`。

## 5. Jotai 状态（ui/lib/atoms.ts，331 行）

- `darkModeAtom`（localStorage）；`selectedRunPathAtom`（localStorage 持久化、per-tab 独立、跨 tab 不同步以避免竞态——代码注释明确）。
- `visibleRunsAtom`、`isTrackingAtom`/`isSyncingAtom`、各页 UI 状态（overview/metrics/timeline/rollouts/discarded/inference 显示开关、EMA、x 轴模式、分页、lane 数等）。
- `createPerRunAtom`(:57-82)：生成「per-run」状态，随 `selectedRunPathAtom` 切换。

## 6. lib 工具

| 文件 | 内容 |
|------|------|
| `types.ts`(980) | RunData、EventData、StepMetric、SystemMetric、RolloutEvent、EvalData、SampleDetails、CustomPlotItem、API 响应类型 |
| `format.ts`(162) | `formatClockTimeAdaptive`（自适应时钟精度）、`formatValueSmart`、`formatSecondsSmart` |
| `run-name.ts`(41) | `SIDEBAR_MAX_RUN_NAME_CHARS=16` + `getSidebarRunNameParts` 后缀截断 |
| `utils.ts`(6) | `cn()` = tailwind-merge + clsx |
| `parse-model-architecture.ts` | HF 模型分数解析（model-architecture-viewer 用） |
| `custom-metrics.ts` / `custom-metrics-templates.ts` | 自定义指标面板 |

## 7. 关键组件

| 组件 | 行数 | 功能 |
|------|------|------|
| `app-sidebar.tsx` | 2113 | 主导航、run 列表、W&B 配置、暗色切换 |
| `combined-timeline-chart.tsx` | 4651 | 时间线泳道图（SVG + dense canvas 双渲染）、事件下钻 |
| `custom-metrics-view.tsx` | 2107 | 可拖拽自定义指标面板（dnd-kit，持久化到 `/custom-metrics-layout`） |
| `step-metrics-charts.tsx` / `system-metrics-charts.tsx` | — | uplot 折线/柱状图 |
| `topology-viewer.tsx` | — | three.js 3D 拓扑（react-three-fiber，悬停高亮） |
| `rollouts-view.tsx` / `sample-details-dialog.tsx` | — | rollouts 详情下钻（`/sample-details`） |
| `run-config-panel.tsx` / `run-config-compare-dialog.tsx` | — | 配置查看/对比 |
| `run-code-visualizer.tsx` / `run-code-compare-dialog.tsx` | — | 源码树 + 跨 run diff |
| `logs-viewer.tsx` | 264 | 日志查看（虚拟滚动 + `/logs` 分页） |
| `database-dialog.tsx` | 272 | DB 大小显示 + compact 按钮 |

## 8. 数据轮询范式

```
页面挂载 → useXxxQuery(endpoint) → fetch → setData → render
   ↑                                          │
   └────── POLL_INTERVAL 轮询 + invalidate ────┘
```

- 多 run 聚合用 optimistic cache（prevRunPaths → setQueryData）减少闪烁。
- per-run 原子保证切换到不同 run 时 UI 状态（selectedStep/EMA/lane 等）独立。