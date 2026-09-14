# Qorix UI 文档

> Qorix UI — 训练监控仪表盘：FastAPI + DuckDB 后端把 W&B 上 tag 为 `qorix` 的 runs 增量同步到本地单文件数据库，React 19 + Vite 前端提供 8 个可视化页面（本地 `localhost:8005`）。

## 快速开始

```bash
pip install qorix-ui
wandb login          # 认证 W&B（一次）
qorix                # 打开 http://localhost:8005
```

数据落在 `~/.qorix/`（`QORIX_DATA_DIR` 可覆盖）；运行自动打 `qorix` tag 即被自动发现。

## 目录

### 架构
| 文档 | 说明 |
|------|------|
| [系统架构概览](architecture/overview.md) | W&B → ingest → DuckDB → FastAPI → React 五层；组件与目录（18K LOC 后端 + 78 文件前端） |
| [模块地图与符号索引](architecture/module-map.md) | 功能 → 源码符号:行号直达表；后端路由总表、页面 → 数据源映射、缩略语 |
| [数据生命周期](architecture/data-lifecycle.md) | 一条事件从 tail.zip → parquet → DuckDB → 图表；幂等键、回填判定、故障排查 |

### 后端
| 文档 | 说明 |
|------|------|
| [同步引擎](backend/ingest.md) | 发现/轮询/下载/解压/入库/幂等同步、ingest_state、reconcile（4265 行） |
| [同步引擎深层](backend/sync-engine.md) | 三后台循环、活跃 run 状态机、同步队列（4 worker）、tag 发现、版本闸门 |
| [DuckDB 数据层](backend/database.md) | 全部表结构（家族 × eval/discarded/cancelled）、zstd 压缩列、连接模型 |
| [表家族与查询模式](backend/schema.md) | 主表四象限镜像、区间扫描分页、幂等写签名、状态表矩阵、迁移 |
| [FastAPI 路由](backend/server.md) | 63+ REST 端点全表、请求模型、每个页面的依赖端点 |
| [API 契约](backend/api-contract.md) | 分页语义（page≠offset）、三宗大 SQL、Run Code 契约、管理端点、SPA 边界 |
| [W&B 集成](backend/wandb.md) | 凭据三态、tag 发现逻辑、项目增删、restore/升级场景 |
| [源码产物与代码对比](backend/run-code.md) | source.zip 落盘、目录树、跨 run diff、路径穿越防护 |

### 前端
| 文档 | 说明 |
|------|------|
| [前端架构](frontend/overview.md) | 技术栈、路由、hooks（TanStack Query）、atoms（Jotai）、lib |
| [页面功能](frontend/pages.md) | 8 个页面的功能与数据源 |
| [前端数据流](frontend/data-flow.md) | Hook → 端点对照表、三层状态、页面级装配样板、刷新节奏 |
| [可视化组件](frontend/charts.md) | StepMetrics 图床、时间线 span 透视、拓扑 3D、模型 repr 解析、自定义仪表盘 |
| [前端状态](frontend/state.md) | Jotai 原子全拆解、比较模型（atom vs local state）、深链接、常见坑 |

### 开发
| 文档 | 说明 |
|------|------|
| [开发指南](development.md) | 本地构建（Vite → `src/qorix_ui/static`）、调试、数据目录 |

### 面试
| 文档 | 说明 |
|------|------|
| [面试准备](interview-prep.md) | 15 道深度问答：幂等同步、tail 帧、象限表、比较模型、Run Code 只读 |

> **学习路径 (30min)**：`architecture/overview.md` → `architecture/data-lifecycle.md` → `backend/sync-engine.md` → `backend/schema.md` → `backend/api-contract.md` → `frontend/data-flow.md` → 用 `qorix` 打开一个 run 逐个页面过一遍 → 时间充裕再读 `interview-prep.md`。