# 源码产物与代码对比（run_code.py）

> `server/run_code.py`（282 行）管理从 W&B 拉取并落盘的源码快照（`~/.qorix/code`），提供**目录树浏览**与**跨 run diff**，供前端 Code Compare / 首页源码面板使用。

## 1. 存储布局

- 存储根目录：`~/.qorix/code`（run_code.py:12-13）；`ensure_run_storage_dirs`(:47)。
- `run_storage_slug`(:27-32)：由 `entity/project/run_id` 生成安全 slug（sanitize + sha1 截断 10 hex）。
- `get_run_code_run_dir`(:35)、`get_run_code_metadata_path`。

## 2. 安全防护

- `resolve_code_file_path`(:51-56)：**路径穿越防护** —— 基于 slug 归一化后开头 relative path 再查 `..`，杜绝 `/../../etc/passwd` 类读取。

## 3. 目录树与指纹

- `build_code_tree`(:231)：递归生成目录树（`_sorted_entries` :59 按目录优先 + 字母排序；文件用 `_file_sha1`(:68) 做内容指纹）。
- 上限常量（:14-16）：
  - `MAX_TREE_NODES = 20000`（目录树节点上限）
  - `MAX_DIFF_MATRIX_CELLS = 2000000`（差异矩阵上限）
  - `MAX_DIFF_SUMMARY_FILE_BYTES = 4MB`（单文件 diff 摘要上限）

## 4. diff 引擎

| 函数 | 行号 | 说明 |
|------|------|------|
| `_content_to_diff_lines` | :103 | 内容 → diff 行 |
| `_count_line_diff_fallback` | :130 | 兜底行数差异 |
| `_count_line_diff_lcs` | :154 | LCS 精确算法 |
| `_count_line_diff` | :169 | 按规模选择策略（fallback vs LCS） |
| `build_code_diff_summary` | :176 | 汇总两个 run 的 diff（最大 4MB + 矩阵上限） |

## 5. 数据来源

- ingest 侧 `_sync_source_artifacts`（ingest.py:300，下载 `code/source.zip` + `metadata.json`，远程路径常量 ingest.py:187-188）。
- 前端端点：`/run-code/tree/{run_path}`、`/run-code/file/{run_path}`、`/run-code/diff-summary/{left_run_path}`（main.py:3947/3971/4013）。
- 前端组件：`run-code-visualizer.tsx`（树浏览）、`run-code-compare-dialog.tsx`（diff 弹窗 + `run-code-compare`）。

## 6. 使用场景

- 复现实验：打开 run 的源码树查看训练实际运行的代码版本。
- 对比实验：选择两个 run 做 LCS diff，快速定位超参/环境/奖励函数差异（`MAX_TREE_NODES` 大项目自动截断保护）。