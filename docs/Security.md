安全策略与不可逆操作的防护清单（基于 Python 版 git-filter-repo 的做法）
===========================================================

目的
----
总结 Python 版 git-filter-repo 在“开始前、执行中、结束后”围绕数据安全所做的防护与收尾措施，作为 Rust 版对齐与跟踪实现的参考清单。

一、开始前：安全前置检查（不满足则拒绝改史）
-------------------------------------------
- 新鲜克隆（fresh clone）校验：除非传入 `--force`，否则拒绝在非 fresh clone 上重写历史，并提示原因与建议（改史不可逆，将立即清理 reflog 与旧对象）。
- GIT_DIR 位置约束：
  - 裸库：`GIT_DIR` 必须为 `.`
  - 非裸库：`GIT_DIR` 必须为 `.git`
- 引用名冲突：
  - 大小写不敏感文件系统（`core.ignorecase=true`）下，若存在仅大小写不同的 refs，拒绝改史。
  - 归一化文件系统（`core.precomposeunicode=true`，常见于 macOS）下，若存在仅 Unicode 归一化不同的 refs，拒绝改史。
- 仓库打包形态：要求“像新克隆一样已打包”。通过 `git count-objects -v` 检查 packs 数量、loose objects 数量，判断是否满足 freshly packed。
- 远程配置：只允许存在一个名为 `origin` 的远程（或全新裸库无 pack 且无远程），否则拒绝改史。
- Reflog 状态：要求所有 reflog 至多一条记录（fresh clone 特征），否则拒绝。
- 无 stash：若存在 `refs/stash`，拒绝改史（避免用户未保存的工作状态在改史中“失联”）。
- 工作区干净（非裸库）：
  - 无已暂存变更（`git diff --staged --quiet`）
  - 无未暂存变更（`git diff --quiet`）
  - 无未跟踪文件（忽略工具自身 `__pycache__/git_filter_repo.*`）
- 未推送变更对齐（非裸库）：本地分支 `refs/heads/*` 与 `refs/remotes/origin/*` 必须一一对应且 commit 一致。
- 单一 worktree：`git worktree list` 仅一条记录。

二、执行中：保护与铺垫
-----------------------
- dry-run 模式（`--dry-run`）：只导出与过滤，不触碰仓库；保存原始与过滤后的 fast-export 流供审阅。
- 远程迁移（完整改史、非 `--partial`）：
  - 将 `refs/remotes/origin/*` 迁移至 `refs/heads/*`，并删除 `origin/HEAD`，确保改史在本地分支空间进行。
  - 非敏感模式结束后移除 `origin` 远程（下文“结束后”细述）。
- 敏感数据场景：
  - 默认提示并强制 `fetch` 所有 refs（`git fetch -q --prune --update-head-ok --refmap "" origin +refs/*:refs/*`）以覆盖所有可能引用敏感对象的历史；
  - 若用户拒绝，自动启用 `--no-fetch` 并跳过移除 `origin` 的步骤，降低破坏面。
- 局部改史（`--partial`）的安全降级：
  - 不迁移 remotes/origin，不移除 `origin`，不删除未导出的 refs；
  - 禁止 reflog 过期与自动 gc；
  - 标签到/分支名重写改为“新增而非替换”。

三、结束后：引用更新、元数据、不可逆清理与后续指引
----------------------------------------------
- 引用更新与清理：
  - 通过 `git update-ref --no-deref --stdin` 批量更新重写后的 refs；
  - 删除未导出的旧 refs（剪枝后的引用）；
  - 按 `--replace-refs` 策略处理 replace refs（delete-*/update-*）。
- 元数据产出（便于审计追溯）：写入 `.git/filter-repo/`：
  - `commit-map`：old → new 提交哈希（被删除时 new 用 40 个 0）
  - `ref-map`：old/new 哈希与 ref 名；`changed-refs`；`first-changed-commits`
  - `suboptimal-issues`：例如某些 merge 变为普通提交等
  - 敏感模式：`sensitive_data_removal`、LFS orphan 相关记录
- 不可逆清理（切断旧历史可达性）：
  - `git reflog expire --expire=now --all`
  - `git gc --prune=now`（可加 `--quiet`）
  - 非裸库：`git reset --hard`（工作区对齐至新历史）
  - stash 重写：若此前捕获了 stash，依据重写映射重写 `refs/stash` 的 reflog，并打印 “Rewrote the stash.”
- 移除 `origin` 远程（非敏感模式）：打印 NOTICE 并 `git remote rm origin`，防止意外推送回旧远程。
- 推送与后续指引：
  - 输出耗时与下一步建议；敏感场景建议 `git push --force --mirror origin`；
  - 若用户拒绝 fetch，则根据场景建议 `git push --all --tags origin` 或对变更 refs 定向强推；
  - 明确提示其他副本需要清理/重克隆。

四、关键选项与默认策略
------------------------
- `--force`：跳过 fresh clone 强校验（仍提示不可逆与清理后果）。
- `--dry-run`：不改仓库；导出/过滤并保存原始与过滤流。
- `--partial`：降低破坏面（不移除 origin、不删未导出 refs、不清理 reflog/gc，重命名改为新增）。
- `--no-gc`：跳过结束后 gc。
- `--no-fetch`：敏感场景下禁止自动 fetch。
- `--analyze`：只分析，不改史。
- `--replace-refs`：控制 replace refs 的删除/更新/新增策略（delete-* / update-*）。

五、Rust 版对齐计划（建议优先实现）
---------------------------------
1) 前置校验：完整实现 fresh clone 校验（GIT_DIR、打包状态、reflog 单条、工作区干净、未推送、单 worktree、大小写/归一化冲突）。
2) `--force` / `--dry-run` 语义与 Python 版一致。
3) 完整改史下的 remotes/origin → refs/heads 迁移与（非敏感时）移除 `origin`。
4) 敏感模式：fetch all + 用户确认 + `--no-fetch` 回退；保留/或延迟移除 `origin`。
5) 结束后：`update-ref` 批量更新、删除未导出 refs、replace refs 策略、reflog 过期 + gc + reset --hard + stash 重写。
6) 元数据/报告：`commit-map`、`ref-map`、`changed-refs`、`first-changed-commits`、`suboptimal-issues`、敏感/LFS 记录。

备注
----
以上要点依据 Python 版 `git-filter-repo` 源码梳理（sanity_check、_migrate_origin_to_heads、_ref_update、cleanup、_record_metadata 等）。Rust 版在保障性能与可移植的前提下，优先落地“拒绝在不安全环境改史 + 不可逆清理”两大块，确保用户数据安全与预期一致。

