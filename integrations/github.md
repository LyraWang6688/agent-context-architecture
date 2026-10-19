---
id: INT-GITHUB-001
layer: integrations
system: GitHub
force: soft
scope: 所有公开 GitHub 项目
status: VERIFIED
verified: 2026-10-19
source: 实测 LyraWang6688/feishu-retail-ops（样板仓库）
---

# GitHub · 公开项目基线清单

> **用途**：每开一个新项目、或定期体检既有项目时，逐条对一遍。
> **前提**：公开仓库 → GitHub 的付费级安全能力**几乎全免费**（Actions 分钟数无限、CodeQL、Secret scanning、Dependabot 都免费）。
> **样板仓库**：`feishu-retail-ops` 是已经配好的参照，复刻它即可。

---

# A. 安全（Security）

## A1 · Secret scanning 开启

- **价值**：自动扫描仓库里的密钥/Token
- **怎么开**：Settings → Code security → Secret scanning
- **验证**：
  ```bash
  gh api repos/OWNER/REPO --jq '.security_and_analysis.secret_scanning.status'
  # 期望 enabled
  ```
- [ ] 已开启

## A2 · Push protection 开启

- **价值**：**推送时就拦住**含密钥的提交（不是事后告警）
- **为什么重要**：这是**真 Hook**——服务端强制，绕不过去
- **验证**：
  ```bash
  gh api repos/OWNER/REPO --jq '.security_and_analysis.secret_scanning_push_protection.status'
  ```
- [ ] 已开启

## A3 · Dependabot alerts 开启

- **价值**：依赖有已知漏洞时告警
- **验证**：
  ```bash
  gh api repos/OWNER/REPO --jq '.security_and_analysis.dependabot_security_updates.status'
  ```
- [ ] 已开启

## A4 · Dependabot security updates 开启

- **价值**：有漏洞时**自动开 PR** 升级
- **注意**：和 A3 是两个开关，别只开一个
- [ ] 已开启

## A5 · CodeQL / Code scanning 开启（Default setup）

- **价值**：自动扫描注入 / XSS / 路径穿越 / 重定向等
- **为什么容易漏**：**Default setup 不生成 workflow 文件**，只翻 `.github/` 会漏判
- **怎么开**：Settings → Code security → Code scanning → Default setup → Configure
- **验证**：
  ```bash
  gh api repos/OWNER/REPO/code-scanning/default-setup --jq '.state'
  # 期望 configured
  ```
- [ ] 已开启

## A6 · SECURITY.md 存在

- **价值**：告诉外部如何报告漏洞（而不是公开挂 Issue）
- **验证**：`ls SECURITY.md` 或 `.github/SECURITY.md`
- [ ] 已存在

## A7 · Actions 默认权限最小化

- **价值**：防止 workflow 被利用时拥有写权限
- **怎么开**：Settings → Actions → General → Workflow permissions → **Read repository contents**
- **验证**：
  ```bash
  gh api repos/OWNER/REPO/actions/permissions/workflow
  # 期望 default_workflow_permissions=read
  ```
- [ ] 已设为 read

## A8 · 部署类项目用 Environment + required reviewers

- **价值**：部署前强制人工批准（**真 Hook**）
- **适用**：有生产部署的项目；纯库项目可跳过
- **怎么开**：Settings → Environments → New environment → Required reviewers
- [ ] 已配置（或无部署，N/A）

---

# B. 合并与分支（Merge & Branch）

## B1 · 合并后自动删除 head 分支 ★

- **价值**：**直接解决「已合并 PR 的分支一直不删」**
- **为什么列第一**：一个开关，永久生效，服务端执行
- **怎么开**：Settings → General → Pull Requests → ✅ Automatically delete head branches
- **验证**：
  ```bash
  gh api repos/OWNER/REPO --jq '.delete_branch_on_merge'
  # 期望 true
  ```
- [ ] 已开启

## B2 · main 分支保护 / Ruleset

- **必须挡住的四件事**：
  - 直接删除 main
  - force push 到 main
  - 绕过 PR 直接推
  - 未过 CI 就合并
- **怎么开**：Settings → Rules → New ruleset（或 Branch protection）
- **验证**：
  ```bash
  gh api repos/OWNER/REPO/rulesets --jq '.[].name'
  ```
- [ ] 已配置

## B3 · 必须的 CI 状态检查（required status checks）

- **价值**：CI 不过 → 合不了。这是**真 Hook**
- **前提**：仓库必须有 CI，否则会锁死合并
- **验证**：
  ```bash
  gh api repos/OWNER/REPO/branches/main/protection \
    --jq '.required_status_checks.contexts'
  ```
- [ ] 已绑定至少一个检查

## B4 · required review ≥ 1，或配 CODEOWNERS

- **坑**：`required_approving_review_count = 0` 等于**没要求 review**
- **适用**：单人项目至少配 CODEOWNERS，让规则可扩展
- [ ] 已配置

## B5 · allow_auto_merge 开启

- **价值**：PR 过检查自动合并，不用人盯
- **验证**：
  ```bash
  gh api repos/OWNER/REPO --jq '.allow_auto_merge'
  ```
- [ ] 已开启

## B6 · 对话必须解决（required conversation resolution）

- **价值**：未解决的 review comment 不能合并
- [ ] 已开启

---

# C. CI（确定性门禁）

## C1 · 有真实 CI（不是空 workflow）★

- **判据**：`.github/workflows/` 里至少有一个在 PR 时**真的执行命令**的 workflow
- **反例**：目录存在但文件是空的 / 只做 checkout
- **验证**：`gh run list --limit 5`
- [ ] 已有

## C2 · CI 跑测试

- [ ] 已包含测试步骤

## C3 · CI 跑依赖审计

- **价值**：让「测试过了但依赖带洞」不再是可以合并的状态
- **参考实现**（`feishu-retail-ops`）：
  ```yaml
  - run: pnpm audit --prod --audit-level=high
  ```
- **注意**：只审计**生产依赖**（`--prod`），否则会被上游未修复的开发依赖卡死
- [ ] 已包含

## C4 · CI 结果绑定为 required check

同 B3。
- [ ] 已绑定

---

# D. 可维护性

## D1 · PR 模板

- **价值**：统一格式；可以设「必答问题」逼人想清楚
- **样板**：`feishu-retail-ops` 的模板带「⭐必答」
- [ ] 已存在

## D2 · Issue 模板

- [ ] 已存在

## D3 · CODEOWNERS

- **价值**：多 agent / 多人协作时**自动指派 who reviews**
- **位置**：`.github/CODEOWNERS`
- [ ] 已存在

## D4 · 文件规模 / 代码规范门禁

- **价值**：把「文件别太大」变成 CI 检查，而不是靠自觉
- **参考**：`feishu-retail-ops` 的 `file-size` job + 棘轮脚本
- [ ] 已配置（或明确不需要）

## D5 · Labels 规范化

- [ ] 已建立基础标签（bug / enhancement / dependencies …）

---

# E. 可扩展性

## E1 · 公共 CI 抽成 reusable workflow

- **价值**：多个项目共用一套门禁，**改一处全部生效**
- **做法**：单独的 `.github` 仓库放 `workflow_call` 的 workflow，各项目 `uses:`
- **适用**：有 3 个以上项目的场景（你现在 12 个，值得做）
- [ ] 已抽取（或已计划）

## E2 · Releases + Tags 版本化

- **价值**：可回滚、可追溯
- **验证**：
  ```bash
  gh api repos/OWNER/REPO/tags --jq 'length'
  ```
- [ ] 有 tag 且有 Release

## E3 · Dependabot 分组策略

- **价值**：避免「每个包各开一个 PR」的洪水
- **参考**：minor+patch 分组、major 单独、`open-pull-requests-limit`、安全更新不排队
- [ ] 已配置分组

---

# F. 告警处理（最容易被忽略）

> **扫描开了不等于安全。没有 triage 的告警 = 没人看的报警器。**

## F1 · CodeQL 告警有处理

- **验证**：
  ```bash
  gh api "repos/OWNER/REPO/code-scanning/alerts?state=open" --jq 'length'
  ```
- **判据**：未处理数有明确解释（已评估 / 已豁免 / 已排期），不是无限堆积
- [ ] 已 triage

## F2 · Dependabot 告警有处理

- [ ] 已 triage

---

# 快速体检命令

```bash
O=OWNER; R=REPO
gh api "repos/$O/$R" --jq '{delete_branch_on_merge, allow_auto_merge}'
gh api "repos/$O/$R" --jq '.security_and_analysis'
gh api "repos/$O/$R/code-scanning/default-setup" --jq '.state'
gh api "repos/$O/$R/rulesets" --jq '.[].name'
gh api "repos/$O/$R/branches/main/protection" --jq '.required_status_checks.contexts'
gh api "repos/$O/$R/actions/permissions/workflow"
```

---

# 优先级

| 级别 | 项目 |
|---|---|
| **P0** | B1 自动删分支 · A1–A5 安全开关 |
| **P1** | B2/B3 required check · C1–C3 CI · F1/F2 告警 triage |
| **P2** | D1–D5 模板与规范 · E1 reusable workflow |

---

# 已知样板

| 仓库 | 完成度 |
|---|---|
| `feishu-retail-ops` | **最高**（A1–A5、B1、B3、C1–C3、D1、E2、E3 均有；缺 A6/A7、D2/D3/D4、E1） |
| `ai-capability-store` | 中（有 CI + required check；缺 A3/A4/A5、B1、D1、E1） |
| `personal-ip-hub` | 低（有 ci.yml） |
| `teaming-bot` | **最低**（无 CI、无 dependabot、无模板） |

> **注意**：`teaming-bot` 没有 CI，**在补上 CI 之前不能开 required check**，否则会锁死合并。
