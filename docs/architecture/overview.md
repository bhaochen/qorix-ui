# 系统架构概览

> qorix-ui = 「训练可视化 + 离线同步」的本地 Web 应用：**FastAPI + DuckDB** 后端把 W&B 云上的 tagged run 全量拉取、解压（zstd）、解析成事件表；**React 19 + Vite + TanStack Query + Jotai** 前端提供监控大屏。数据全程单机存储，离线可查。

## 1. 五层架构

```
┌─────────────── W&B Cloud (GraphQL + zip artifact) ───────────────┐
│  tag "qorix" + schema_version tag 0.3.0 的 runs                   │
└──────────────┬───────────────────────────────────────────────────┘
               │ ingest.py 发现/轮询/下载/解压/入库（4265 行）
               ▼
┌────────── DuckDB 单文件 ~/.qorix/qorix.duckdb（db.py 3726 行）───┐
│ 运行/事件/指标/样本/eval 表群（含 _eval/_discarded/_cancelled）   │
└──────────────┬───────────────────────────────────────────────────┘
               │ FastAPI REST（main.py 9699 行，63+ 路由）
               ▼
┌────────── React SPA（Vite 构建 → src/qorix_ui/static）───────────┐
│ TanStack Query hooks 轮询 → 8 个可视化页面                        │
└──────────────────────────────────────────────────────────────────┘
```

## 2. 目录与规模

```
src/qorix_ui/                 # Python 后端包（~18K LOC）
  __init__.py (5)             #   importlib.metadata.version → __version__
  __main__.py                 #   python -m qorix_ui → cli.main()
  cli.py (144)                #   console-script `qorix` 入口
  server/
    main.py (9699)            #   FastAPI app + 全部路由
    db.py (3726)              #   DuckDB schema + 查询 + 压缩列
    ingest.py (4265)          #   W&B 同步引擎（核心）
    run_code.py (282)         #   源码产物存储与 diff
ui/                           # React 前端（独立 npm 工程，78 文件）
  src/App.tsx (33)            #   BrowserRouter + 全部路由
  app/                        #   pages：/ /metrics /timeline /rollouts
                              #          /rollouts-discarded /topology
                              #          /infra /evals /about
  components/                 #   shadcn/ui + 业务组件（图表/时间线/拓扑）
  hooks/                      #   use-run-data 等数据 hooks
  lib/                        #   types / atoms / constants / format
  vite.config.ts              #   outDir → ../src/qorix_ui/static
```

## 3. 关键设计

- **单文件列式存储**：DuckDB 单连接模型（db.py:95-109），事件正文以 zstd 压缩 BLOB 存储（db.py:30），rollout content 解压读取。
- **幂等增量同步**：每 run 一行 `ingest_state`（db.py:953-964）+ `ingested_tails`/`ingested_steps` 幂等键；`tail_idx` 标记事件窗口，只插新内容。
- **事件家族 × 四象限**：每一实体家族（prompts/generations/env_responses/tool_calls/samples_data/rollouts_metrics/golden_answers/sample_tags/info_turns）在 DB 中镜像为 `_eval`/`_discarded`/`_cancelled` 分表。
- **自动发现**：tag `qorix` + schema_version 匹配（排除 `qorix-ignore`），无需手动配置跑名。
- **源码可视化**：run 的 `code/source.zip` 落盘 `~/.qorix/code`，支持目录树 + 跨 run diff。

## 4. 数据流速查（每页面数据源）

| 页面 | 端点 | 数据表 |
|------|------|--------|
| 首页 `/` | `/runs`, `/run-summary`, `/run-code/*`, `/logs`, `/custom-metrics-layout` | runs + 聚合 |
| Metrics | `/step-metrics`, `/step-metrics-multi`, `/custom-metrics-*` | events_trainer / step_metrics |
| Timeline | `/events/timeline-paginated`, `/events/inference-by-group`, `/events/trainer-breakdown` | events_orchestrator/trainer/infra |
| Rollouts | `/rollouts` | generations, rollouts_metrics, golden_answers, sample_tags |
| Rollouts Discarded | `/rollouts-discarded` | _discarded 家族 |
| Topology | `/runs` 数据 | runs 节点拓扑 |
| Infra | `/system-metrics/*-paginated`, `/vllm-metrics/paginated` | system_metrics_gpu/cpu, thread_pools, vllm_metrics |
| Evals | `/evals`, `/eval-step-metrics`, `/sample-details`, `/sample-statuses` | *_eval 家族 |

详见 `backend/server.md` 路由全表与 `frontend/pages.md` 页面明细。