# DuckDB 数据层（db.py）

> `server/db.py`（3726 行）定义全部表结构、连接管理与查询。单文件数据库 `~/.qorix/qorix.duckdb`，**单连接 + 全局锁**（DuckDB 不支持多连接并发写）。所有事件正文以 zstd 压缩 BLOB 存储，幂等表记录已入库进度。

## 1. 存储位置与连接

- 数据目录：`QORIX_DATA_DIR` env 或 `~/.qorix`（db.py:17-21）；DB 文件 `~/.qorix/qorix.duckdb`（:24-25）。
- 单例共享连接 `_get_shared_connection`(:95-109)；`connect()`(:112-114) 返回基于共享 conn 的 cursor。
- `transaction()`(:71-88)：显式 BEGIN/COMMIT 上下文管理器。
- 启动时 `_init_schema`(:117) 建全部表。
- **压缩读取**：`decompress_blob`(:30-46) 用 `zstd.ZstdDecompressor().stream_reader` 还原（避免内容头缺失时 `Destination buffer too small`）。
- 运行颜色：`RUN_COLORS`(:50-67) 16 色调色板；`_next_run_color`(:3102) 依次分配。

## 2. 表结构总览

### 运行与状态（元数据）

| 表 | 位置 | 内容 |
|----|------|------|
| `runs` | :966-991 | wandb 元数据：state/tags/notes/colors/commits/schema_version + removed/drained 标记 |
| `ingest_state` | :953-964 | 每 run 同步进度：`last_block_idx/last_rollout_step/last_summary_json/last_config_json/last_event_zip_idx/last_rollout_block_idx/summary_id` |
| `ingested_tails` | :1016-1021 | 已入库 tail 集合（JSON 数组，幂等键） |
| `ingested_steps` | :1026-1031 | 已入库 step 集合 |
| `ingested_step_metrics` | :1035-1040 | 已入库 step metrics |
| `ingested_evals_after_training` | :1044-1049 | 训练后 eval 进度 |
| `known_projects` | :1052-1057 | 已发现 W&B 项目列表 |

### 事件家族（主表 + 三象限变体）

每个实体家族在 DB 中镜像为 `_eval`（评测）、`_discarded`（丢弃）、`_cancelled`（取消）分表（同列结构）：

| 家族基类 | 位置 | 变体 |
|----------|------|------|
| `events_orchestrator` | :1151 | - |
| `events_trainer` | :1183 | - |
| `events_rollout` | :1223 | - |
| `events_infra` | :1266 | - |
| `prompts` | :1305 | `prompts_eval` :772-787 |
| `generations` | :1338 | `generations_eval` :791-807, `generations_cancelled` :153 |
| `env_responses` | :1403 | `env_responses_eval` :811-828 |
| `tool_calls` | :1455 | `tool_calls_eval` :831-856 |
| `turn_metrics` | :1518 | - |
| `samples_data` | :1556 | `samples_data_eval` :859-874 |
| `rollouts_metrics` | :1610 | `rollouts_metrics_eval` :877-890, `rollouts_metrics_cancelled` :723-732 |
| `golden_answers` | :1652 | `golden_answers_eval` :893-906, `golden_answers_cancelled` :734-743 |
| `sample_tags` | - | `sample_tags_eval` :909-922, `sample_tags_cancelled` :745-754 |
| `info_turns` | - | `info_turns_eval` :925-942, `info_turns_cancelled` :756-769 |
| `step_metrics` | :1866 | - |

### 系统指标

`system_metrics_gpu` / `system_metrics_cpu` / `thread_pools` / `vllm_metrics`（采样表，UI Infra 页分页读取）。

## 3. 写入函数

批量写用 pandas DataFrame bulk insert；事件正文由下游 main.py 负责解压：

| 函数 | 行号 |
|------|------|
| `insert_logs` | :1121 |
| `insert_events_orchestrator` | :1151 |
| `insert_events_trainer` | :1183 |
| `insert_events_rollout` | :1223 |
| `insert_events_infra` | :1266 |
| `insert_prompts` | :1305 |
| `insert_generations` | :1338 |
| `insert_env_responses` | :1403 |
| `insert_tool_calls` | :1455 |
| `insert_turn_metrics` | :1518 |
| `insert_samples_data` | :1556 |
| `insert_rollouts_metrics` | :1610 |
| `insert_golden_answers` | :1652 |
| `insert_step_metrics` | :1866 |

## 4. 状态与生命周期

- `get_ingest_state`(:1061)/`update_ingest_state`(:1091)。
- `set_run_removed`(:3102 区)：run 标记 removed；`/delete-run-data`（main.py:992，先保留 color 再删）真正清理。
- 自定义指标：layout/templates 存储 `custom_metrics_layout` / `custom_metrics_templates`(:3251-3341)。
- `get_database_info`(:3638)：DB 大小等；`compact_database`(:3668)：DB 压缩（UI sidebar 按钮，可减 20-30%）。

## 5. 一致性要点

- **幂等键**：`tail_idx` 唯一标记事件窗口，`ingested_tails` 去重后只插新内容（与 ingest.md §3 闭环）。
- **`_eval` 表与训练表结构镜像**：qorix-ui 的 Evals 页与 rollouts 页共用同一渲染管线。
- 单连接串行写避免锁冲突；查询走只读 cursor。