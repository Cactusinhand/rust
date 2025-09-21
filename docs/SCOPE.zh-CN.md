filter-repo-rs 范围与优先级（Scope & Priorities）
===============================================

目的
----

- 明确“做什么/不做什么”，聚焦高价值能力，避免功能发散与臃肿。
- 为路线图、评审与取舍提供依据，并持续沉淀决策过程。

高价值功能（优先实现）
----------------------

- 敏感信息历史清洗（文件+消息）
  - `--replace-text` 支持跨历史的字节/正则替换；可与路径筛选/重命名/大小阈值联动；配合 `--sensitive` 覆盖全部 refs。
- 大仓库瘦身（历史级别）
  - `--max-blob-size`、`--strip-blobs-with-ids` 批量剔除大对象并删除引用路径；产出报告用于核对与回归。
- 路径重构（monorepo 拆分/目录下沉/批量重命名）
  - `--subdirectory-filter`、`--to-subdirectory-filter`、`--path-rename` 与 `--path`/`--path-glob`/`--path-regex`/`--invert-paths` 组合。
- 提交消息中旧哈希自动重写
  - 使用 `commit-map` 将短/长哈希回写为新哈希，避免“断链”。
- 标签/分支批量改名且一致
  - 处理注解/轻量标签（去重+顺序正确），输出 `ref-map`，降低手动遗漏风险。
- 空提交自动剪枝但保留合并
  - 非合并空提交 `alias` 折叠；合并提交保留，含父去重。
- 原子化更新与可审计映射
  - `git update-ref --stdin` 批量更新；输出 `commit-map`、`ref-map` 作为迁移对照表。
- 敏感模式覆盖全部引用
  - 可抓取所有 refs（非标准命名空间也包含），减少漏改；必要时保留 `origin`。
- 可比对的干跑输出（安全预演）
  - `--dry-run` 同时保存原/滤后流，便于肉眼/脚本 diff。
- Windows 历史路径兼容化
  - 批量清洗非法字符与尾部点/空格，适配跨平台协作。
- 分析报告（人读/JSON）
  - 体积、TOP 对象、热点目录、长路径、重复 blob、父数阈值等，快速定位“膨胀源”。

为何用原生命令难以做到
------------------------

- 需要组合 `fast-export/import`、`rev-list`、`cat-file`、shell/脚本与多次遍历，易错且慢；`filter-branch` 已不推荐。
- “消息中的哈希自动映射”“产出 commit-map/ref-map”“轻/注解标签一致处理”“Windows 历史路径清洗”等缺少现成支撑，手工脚本易碎。
- 覆盖“全部 refs”（含非分支/标签命名空间）与一致性更新，手工列举与校验成本高、风险大。

典型痛点 → 对应能力
--------------------

- 泄漏密钥/令牌清除：`--replace-text`（含正则）+ `--sensitive` + 报告/干跑/映射文件。
- monorepo 拆分/路径重构：子目录滤出、根目录下沉、批量改名一致；避免历史分叉与引用失配。
- 历史瘦身：阈值/白名单双模式删大对象，产出样例与计数便于业务确认。
- 历史整洁化：批量改名标签/分支、剪枝空提交、保持合并与 HEAD 合理指向。

低优先/非核心功能（两边共同）
------------------------------

- 回调框架（filename/refname/blob/commit/tag/reset/message/name/email）
  - 已确认为“非目标”。常见需求以显式 CLI 选项覆盖。
- 增量/替换引用体系
  - `--state-branch`、“already ran”、“stash 重写”、`--replace-refs` 多策略。
- LFS 孤儿检测与 SDR 附加产物
  - LFS orphan、`first-changed-commits`/`changed-refs`、“下一步”长文档。
- 细粒度编码与消息重写开关
  - `--preserve-commit-hashes`、`--preserve-commit-encoding` 等。
- 便利型路径/重命名扩展
  - `--use-base-name`、基于正则的路径重命名（rename‑regex/match）。
- 稀有输入/旗标
  - `--stdin`、`--date-order`、`--no-quotepath`、`--no-mark-tags`、`--no-gc` 等。
- 罕见预检细节
  - 大小写不敏感/Unicode 归一化 ref 冲突、stash 存在、reflog 条目数量等“硬阻断”。

Python 原版特有、可不对齐的项
------------------------------

- 复杂 `--replace-refs` 变体与跨次运行状态管理：学习/运维门槛高，出错代价大。
- fast‑export 字面命令透传与极端输入容错：覆盖面低、代码负担重。
- “次优问题”全集合报告：价值边缘，可留给审计脚本/外部工具。

我们项目里可延后/不做的项
--------------------------

- 消息替换 `regex:`（针对 commit/tag 消息）：先满足常见“字面值替换 + 哈希回写”。
- `--paths-from-file`：易用性增强，优先级低于一致性/正确性。
- Windows 路径策略多模式（sanitize/skip/error）：默认 sanitize 已可用，多模式会增加复杂度。
- mailmap 身份重写：有价值但非 MVP，可在明确需求后推进。

边界建议（收敛范围）
--------------------

- 核心保留：
  - 路径筛选/重命名（前缀/子目录）、Blob 内容替换（含正则，限文件）、大对象剔除、标签/分支前缀改名、
    空提交剪枝（保留合并）、提交消息哈希自动重写、原子化 ref 更新、dry‑run 可对比、
    敏感模式覆盖全部 refs、Windows 路径兼容化、分析报告。
- 明确非目标：
  - 回调框架、增量/状态分支、替换引用高级策略、LFS 孤儿检测/SDR 附加产物、
    编码/哈希保留开关、路径正则重命名、stdin 流处理、过度预检/小旗标。
- 仅在清晰场景/多方反馈时再评估：
 - `--paths-from-file`、消息 `regex:`、mailmap 身份重写。

维护约定
--------

- 本文档是持续更新的“取舍清单”。新增/下线能力、优先级变化、决策背景请在此同步。
- 相关文档：
- PARITY.md（与 Python 版的对齐状态与安全说明）
- STATUS.md（当前状态、限制与 MVP 计划）

可能的“功能发散与臃肿”候选（当前原型）
--------------------------------------

以下选项在实际用户场景中出现频率较低，或更像底层/测试开关，建议“隐藏/合并/简化”，避免表面积扩大：

- fast‑export 直通/底层细节开关
  - `--no-reencode`、`--no-quotepath`、`--mark-tags/--no-mark-tags`、`--date-order`
  - 建议：选用合理默认并隐藏开关（仅用于测试或故障排查）。
- 导入后行为小开关
  - `--no-reset`、`--cleanup [none|standard|aggressive]`
  - 建议：简化为布尔 `--cleanup` 或仅保留 standard；`--no-reset` 仅调试可见（或由 `--dry-run` 隐含）。
- 分析参数“微调旋钮”
  - `--analyze-total-warn`、`--analyze-total-critical`、`--analyze-large-blob`、`--analyze-ref-warn`、`--analyze-object-warn`、`--analyze-tree-entries`、`--analyze-path-length`、`--analyze-duplicate-paths`、`--analyze-commit-msg-warn`、`--analyze-max-parents-warn`
  - 建议：面向 80% 用户仅保留 `--analyze`、`--analyze-json`、`--analyze-top`；其余阈值改用配置文件或环境变量。
- 流覆盖开关（测试专用）
  - `--fe_stream_override`（从文件注入 fast‑export 流）
  - 建议：标注测试专用，不对外文档化或隐藏至开发模式。

说明：上列并非全部“移除”，而是“降低暴露度/合并为更高层语义/采用更安全默认”。核心目标是减少首次使用的认知负担，并把可维护性留在内部实现。
