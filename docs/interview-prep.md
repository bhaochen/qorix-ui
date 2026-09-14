# 面试准备 — qorix-ui 深度要点

> 让面试官相信你不只是「看过这个项目」,而是「懂这套数据链路」。每条都是一个能展开聊十分钟的题。先画那张五层图(见 `architecture/data-lifecycle.md`),再逐个回答。

---

## 第一梯队:30 秒内必须答对

### Q1. qorix-ui 是什么?跟 qorix 什么关系?

- qorix-ui 是 **训练监控盘**:把训练框架 qorix 在 W&B 上打 `qorix` tag 的 runs **增量拉到本地 DuckDB**,FastAPI 提供只读 REST,React(Vite 而非 Next.js、无 Dash)呈现 8 个页面。
- **它不训练、不推理、不执行用户代码**——`run_code.py` 只做目录树 + 逐行 diff(唯一 subprocess 是自升级 `pip install -U`,main.py:307)。

### Q2. 数据从哪来,怎么最快看到?

- 训练侧默认每 `event_upload_interval_seconds=5s` 上传一帧 `events/tail.zip`;后端 5s 轮询 + 幂等补档;
- 所以「实时」≈ **最多 10 秒**看到事件(帧 5s + 轮询 5s 的最坏对齐),`event_tail_window_seconds`(60s)只决定 tail.zip 会话内可回溯长度,不是帧长;
- 间隔 60s 的 Timeline 分页 = 一页一个时间窗,不算 offset。

### Q3. 为什么「幂等」这么强调?多拉一次会重复吗?

- 不会。判决链:`ingested_tails` / `ingested_steps` / `summary_id` 三把锁(见 data-lifecycle.md §4),`insert_*` 全部带 `WHERE NOT EXISTS`(db.py:1121 起);
- 好处:断点续传、压缩期间暂停、重启恢复都不会产生重复或空洞。

### Q4. run 是怎么被发现的?

- **W&B tag 是唯一边界**:`tag:"qorix"` 的 runs 每 5s 被 `tagged_runs_poll_loop`(ingest.py:4153)扫到;
- 版本闸门:schema tag 与 `SCHEMA_VERSION="0.3.0"`(ingest.py:117)不符的 run 被跳过或转重建;
- 凭据三态 netrc/custom/unconfigured(见 backend/wandb.md §1)。

### Q5. 前端状态都在哪,服务端存在什么?

- 服务端:**零会话**。所有查询无游标、stateless(FastAPI);
- 前端:React Query 缓存端点响应 + **Jotai atoms 只存视图偏好**;
- run 选择 `selectedRunPathAtom` 每 tab 独立、"刻意不跨 tab"(atoms.ts:33-45)。

---

## 第二梯队:能展开讲

### Q6. Timeline 页背后那个大 SQL 在干嘛?

- 三个 CTE:orchestrator/trainer 事件区间 + **events_rollout 的 start/end pivot 成 span**,再用 generations 的 vLLM 时序(ttft/prefill/decode)与 samples_data 的 off_policy_steps 增强,最后叠 events_infra(weight_sync/sandbox)(main.py:4168-4243 + :4281-4289);
- 前端 `rolloutEventsToSpans`(combined-timeline-chart.tsx:184)把后端行再聚成 span 列表 → lane 行渲染;
- 意义:一页快照一次窗口,不打全库。

### Q7. 「压缩数据库」会不会弄丢数据?

- `compact_database`(db.py:3668)= 导出→重建导入回收空间;
- 期间 `_compaction_paused` 让三个循环暂停(ingest.py:581/594),前端 `/compact-database/status` 轮询;
- 失败留恢复标志,启动时 `recover_from_failed_compaction`(:3645)兜底;**每帧的幂等锁保证压缩后数据完整**。

### Q8. 多 run「比较」两种实现你选哪个?

- **叠加比较**(任意多 run 同图):`visibleRunsAtom`,无方向、每 run 一条 series;
- **diff 比较**(一对,有方向):组件 local state 而非 atom——因为只比一对、有方向,放全局就污染其它页。这是「偏好 vs 操作」分层的例子。

### Q9. Run Code 页面是「沙箱执行」吗?

不是。它是**代码快照只读器**:source.zip 落 `~/.qorix/code/<slug>/source`,`resolve_code_file_path`(run_code.py:51)防目录穿越,`build_code_diff_summary`:176 用 LCS(≤2M 细胞)算 add/remove 行;diff 动画在客户端 `buildGitStyleDiff`。**整个项目唯一的代码执行是 `pip install -U qorix-ui`**。

### Q10. 为什么图表引擎是 7900 行?

- `StepMetricsCharts`(step-metrics-charts.tsx:501)自己实现了:**多 run 叠加、IQR 离群钳制(:187)、EMA、step|time 双轴、直方图/分布图、推理/训练性能区域图**;每种图都配套数据 hook(`useStepMetricsMultiRun`:991、`useStepTimes`:1058、`useInferencePerformance`:649);
- 它不依赖重图表库的 magic,tails 与性能曲线都是自绘(iplot)。

---

## 第三梯队:让面试官点头的进阶

### Q11. 丢了一个尾巴/空了一块,你怎么查?

1. 前端 `useTimelinePaginated` queryKey 含 page+interval,切页即验;
2. 后端:`get_ingested_tails`(db.py:3431)看覆盖;`/sync-queue-progress`(main.py:280)看 sync 状态与 reasons(`get_sync_mismatch_reasons`:2425);
3. 根因三选一:训练侧没产帧 / 版本 tag 失配被跳过(:403)/ 压缩暂停中;
4. 强制补:删除 run 数据重新 add-run,或等 reconcile(60s)重挂。

### Q12. 怎么给「某个 env 的训练」加一张新图?

- step_metrics 已带 section/group/env 维度(step_metrics 表 db.py:427 + `/step-metrics/multi` 的 tag/env 过滤);
- 前端只需:在 `buildPlotCatalog`(custom-metrics-view.tsx:106)登记一个 catalog item → 用户即可拖进自定义布局,无需改后端;
- 想默认就展示:追加到 `DEFAULT_OVERVIEW_PLOTS`(atoms.ts:94-98)。

### Q13. qorix 训出的 ignore 某段,UI 怎么知道?

- 语义全靠 `event_type` / `discard_reason` / 象限表:
  - `_discarded` 表 = 被丢 rollouts;`_cancelled` 表 = 权重更新时被取消的请求;
  - Timeline 可 `inferenceHighlightDiscardedAtom` 高亮被丢的 inference;
  - Rollouts 页三个不同列表(Rollouts / Rollouts Discarded / Evals)是**同一条渲染管线分象限**,不是三个实现。

### Q14. 你们如何管理「前端不知道新表」?

- 后端 41 张 run 表都不是「前端约定的 API」——前端只依赖 ~10 个聚合 REST 端点;
- 加表不一定要动前端;动了就靠 `types.ts` + RQ hook 增量;
- `RunSummary`(types.ts:578)聚合了 counts + metrics 目录,前端从此逆向知道「这个 run 有哪些指标可画」。

### Q15. 开放题:给全景加「跨 run 时间线」,怎么设计?

- 候选答案(展示设计分层):后端显式支持 `run_paths[]` 入参(现在部分端点已多 run),复用 span pivot;前端新增组合 hook + `visibleRunsAtom` 多色叠加;时间轴仍按 tail 窗口分页——因为 UI 的主键是 run+window,不是「统一时间戳全局对齐」。保证「客户端无全局时间线状态、服务端无游标」两条约束不变。