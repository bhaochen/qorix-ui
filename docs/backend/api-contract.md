# API 契约 — 每个页面与后端怎么说话

> main.py 9699 行全是"死读者":任何 `POST /xxx` 都带 `run_path`,返回无分页游标的 JSON。本篇拆解**分页语义、请求模型、三宗大 SQL**,以及前端如何消费。

## 1. 请求模型的形状

几乎所有数据端点长这样(Pydantic `BaseModel`,main.py:354-560):

```python
class RolloutsRequest:      # main.py:1151 一带
    run_path: str
    step: int | None        # 不传 = 最新 step
    ...

class TimelinePaginatedRequest:   # main.py:477
    run_path: str
    page: int
    interval_seconds: int = 60
```

设计主轴:**无 session、无游标、无 offset**。前端每次拿「run + page + 窗口参数」来,后端重算 WHERE,因此 server 可以无限实例、横向扩展、restart 无状态损失。

## 2. 分页词汇:page ≠ offset

- Timeline:`total_pages = max(1, int(total_duration // interval) + 1)`(main.py:4108),**每页是一个固定长度时间窗**;翻页 = 换窗,不是往前跳 N 行;
- 系统指标:同样时间窗(page → `interval_start/end` 窗内的行);
- **例外是 `/api/logs`**:纯 offset 分页(`offset = page * page_size`,main.py:9590,默认 500 条);日志不需要时间窗,量大且线性,offset 简单直接。
- 前端 `PaginationControls`(ui/components/pagination-controls.tsx:20)只做 ±1 页和跳页,配合 `timelinePageAtom`(:131)。

含义:如果时间窗很长(比如 interval=60 但整个 run 5 小时=300 页),翻页不慢——因为每页 SQL 都是区间扫描,与总行数无关。

## 3. 三宗大 SQL(读懂它们=读懂时序)

| 端点 | 核心 SQL 发生了什么 | 位置 |
|------|--------------------|------|
| `/events/timeline-paginated` | 一个大 CTE:**把 `events_rollout` 的 start/end 对 pivot 成 span**,再用 `generations`/`generations_discarded`/`generations_eval` 的 vLLM 时序字段(ttft/prefill/decode/inference)与 `samples_data.off_policy_steps` 增强;末尾叠 `events_infra`(weight_sync/sandbox) | main.py:4168-4243(rollout pivot)+ :4281-4289(infra) |
| `/rollouts` | 单 step 六连查:prompts→generations→env_responses→tool_calls→samples_data→metrics/golden/tags/info_turns,`decompress_blob` 解出正文 | main.py:1299-1420 区 |
| `/step-metrics/multi` | 多 run 的 step_metrics 合并;支持 tag/env 过滤(Query 参数 min/max step) | main.py:7002 |

前端侧对应:
- timeline 页把 4 路事件喂给 `CombinedTimelineChart`(combined-timeline-chart.tsx:396);
- `rolloutEventsToSpans`(:184)与 `pivotInfraEvents`(:217)是前端把后端已 pivot 的行再聚成「span 列表」;
- `useTimelinePaginated`(use-run-data.ts:526)queryKey `["timeline-paginated", runPath, page, intervalSeconds]`。

## 4. Run Code 契约(只读,无执行)

| 端点 | 契约 | 安全策略 |
|------|------|---------|
| `GET /run-code/tree/{run}` | 目录树(sha1 哈希节点),上限 20k 节点 | `run_code.py:231` + `:14` |
| `GET /run-code/file/{run}?file_path=` | 文件正文,上限 2MB(`MAX_CODE_FILE_BYTES main.py:94`) | `resolve_code_file_path` run_code.py:51,**路径穿越直接 400** |
| `GET /run-code/diff-summary/{left}?right_run_path=` | `{changed,added,removed_lines}`;LCS(≤2M 矩阵)或 fallback 计数(>4MB 文件截断) | run_code.py:176/:15/:16 |

前端 diff 引擎(`buildGitStyleDiff`/`mergeTreeNodes`/`buildChangeBlocks`,run-code-visualizer.tsx:443/:484/:581)做了 git 风格的逐行 diff + 关键字高亮——**全是客户端算的**,后端只给树和文件。

## 5. 管理与运维端点

- `/version-check`:291(PyPI 元数据)与 `/update`:307(`pip install -U qorix-ui`,120s 超时)——**后端唯一 subprocess**;
- `/restart`:338 `os.execv` 自重启(dev 模式下 uvicorn reload 配合);
- `/search-runs-by-config`:1155:按 `config_json` 的 dot-notation key 过滤 run,key=value、key=regex;
- `/database-info` :9644 / `/compact-database`(+status):9656/9650。

## 6. 服务端与静态 SPA 的边界

- `mount_static_ui()`(main.py:179-200)在非 dev 模式挂 `/assets`(StaticFiles)+ 保底 SPA 路由 `GET /{full_path:path}` 返回 `index.html`;
- dev 模式由 Vite dev server 提供前端(端口 5173),FastAPI 只开 REST,CORS 放行 `localhost:3000`(main.py:165-171);
- 前端 `API_BASE = import.meta.env.VITE_API_BASE ?? ""`(constants.ts:3):同源时走相对路径。