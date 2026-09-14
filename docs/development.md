# 开发指南

> 面向新贡献者的最小可运行知识集：环境准备、前端构建、后端调试、新增页面怎么走通。

## 快速开始

```bash
# 安装并运行后端
uv venv --python 3.11 && source .venv/bin/activate
uv sync
wandb login
uv run qorix            # 或 python -m qorix_ui；→ http://localhost:8005

# 前端开发（独立进程）
cd ui
npm install
npm run dev             # :3000，vite dev server（API 指向后端）
```

## CLI 参数（src/qorix_ui/cli.py）

| 参数 | 默认 | 说明 |
|------|------|------|
| `--port` | 8005 | 端口 |
| `--host` | 127.0.0.1 | 绑定地址 |
| `--no-browser` | false | 不自动打开浏览器 |
| `--debug` | false | 详细日志 |
| `--dev` | false | 开发模式（跳过静态 UI 挂载，允许 reload） |

## 前端构建与打包

- `cd ui && npm run build` → 产物进 **`../src/qorix_ui/static`**（vite.config.ts:16-19）。
- Hatch 构建包含 `src/qorix_ui/static/`（pyproject.toml:48）——`pip install qorix-ui` 即带 UI。
- 生产下 FastAPI 挂载 static 目录 + SPA catch-all（main.py:179-194）。

## 数据目录与调试

- 数据：`~/.qorix/`（`QORIX_DATA_DIR` 可覆盖）；DB `qorix.duckdb`；源码快照 `~/.qorix/code`。
- **查看 DB**：`duckdb ~/.qorix/qorix.duckdb`（列表示）。
- **同步调试**：`/sync-queue-progress`、`/stop-all-syncs`、`/drain-run`。
- **日志**：`--debug` 开启后端详细日志；W&B 侧 `WANDB_SILENT=true` 静默。
- 压缩：sidebar → database dialog → Compress（`compact_database`，可减 20-30%）。

## 新增页面 5 步

1. 后端：main.py 加端点（复用 db.py 的 `connect()` + 查询函数）。
2. 类型：ui/lib/types.ts 定义响应类型。
3. Hook：ui/hooks/use-xxx.ts 用 `useQuery` + `POLL_INTERVAL` 轮询。
4. 页面：ui/app/<name>/page.tsx 注册到 App.tsx 路由 + 组件。
5. sidebar 加导航（app-sidebar.tsx）。

## 测试与检查

- 项目无测试文件；改动后跑 `npm run build`（前端 typecheck/打包）+ `python -m compileall src/qorix_ui`。
- `npm run lint`（ui/eslint.config.mjs）前端规范。

## 架构速览

```
W&B (tag "qorix")  →  ingest.py（发现/轮询/下载/解压/幂等）
                   →  DuckDB（家族 × eval/discarded/cancelled，zstd 列）
                   →  main.py REST（63+ 路由）
                   →  React hooks（TanStack Query 轮询）
                   →  8 页面（metrics/timeline/rollouts/infra/evals/topology…）
```

详见 `architecture/overview.md` 与各 backend/frontend 文档。