---
id: HA-01
layer: hooks
title: 任务完成 → 跑 repo-wrapup 出收尾报告
force: DECLARED
trigger: 任务完成 / 准备交接 / 用户问「干净了吗」
source: 王颖 2026-10-20
---

# HA-01 · 任务完成 → 跑 repo-wrapup 出收尾报告

## 触发点

满足任一即触发：

- 一次任务完成、准备交付
- 准备把工作交接给另一个 agent
- 用户问「本地和远端干净了吗 / 分支清了吗 / worktree 清了吗」
- 一个 PR 已合并或已关闭

## 必须发生

跑 `repo-wrapup` 的体检，把**六个数字**作为「收尾报告」贴进交付物：

```bash
bash ~/.agents/skills/repo-wrapup/scripts/audit_cleanliness.sh <repo-path>
```

报告格式：

| 指标 | 清理前 | 清理后 |
|---|---|---|
| 额外 worktree | | |
| 本地已合并未删分支 | | |
| 远端已合并未删分支 | | |
| 含未推送提交的分支 | | |
| UNKNOWN | | |
| 未提交改动 | | |

## 判据

**交付内容里有没有这张「收尾报告」表。**

没有 → 不算交付完成。

## 失败处理

**警告，不阻断。**

## 强制力

**`DECLARED`**

> ⚠️ **没有任何系统会强制执行这一条。**agent 不跑，也没人知道。
> **不要以为自己被保护了。**

## 为什么它只能是 DECLARED

`repo-wrapup` 的核心产出是**一份给人看的报告**。

**系统没法强制「你去看一份报告」。**所以这一条永远停在"约定"，升不成 Hook。

## 但它的一部分**已经**是 ENFORCED 了

这是这一条最值得注意的地方——**它检查的 6 项里，能强制的已经被强制了**：

| 检查项 | 强制手段 | 状态 |
|---|---|---|
| 远端已合并分支 | `delete_branch_on_merge`（GitHub 服务端执行） | ✅ **已开**（见 `integrations/github.md` B1） |
| 本地跟踪分支（`origin/xxx`） | `git config --global fetch.prune true` | ✅ **已设** |
| 未推送提交 | git 本身会拦 | ✅ 默认行为 |
| **worktree 残留** | 系统看不到 | ❌ 只能这个 DECLARED |
| **本地已合并分支** | 可 git hook，但有误删风险 | ⚠️ 待定 |
| **收尾报告本身** | — | ❌ 永远 DECLARED |

**所以这条 hook 的真实价值不是"强制"，是"让下一个 agent 知道有这件事"。**

在此之前：`repo-wrapup` 装好了、能用，但**没有任何地方写着"什么时候该用"**。

## 相关

- skill：`~/.agents/skills/repo-wrapup/`
- 规则来源：`global.md` G14（收尾责任）→ 已迁至 `preferences.md` P07
- 已强制的部分：`integrations/github.md` B1
- 另一个清理 skill（不冲突）：`repo-hygiene` 管**文件**，本 skill 管 **git 引用**
