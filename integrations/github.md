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

# GitHub · 仓库规范与基线清单

> **本文件两部分，都属于「怎么用对 GitHub」**：
> - **A–F 基线清单**：配置层面（安全 / 合并 / CI / 可维护 / 可扩展 / 告警）
> - **G 仓库呈现与文档规范**：README / topics / llms.txt / AGENTS.md 该长什么样
>
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

> **硬要求：所有未归档仓库必须开启。新建仓库时第一件事就是开它。**

- **价值**：**直接解决「已合并 PR 的分支一直不删」**
- **为什么列第一**：一个开关，永久生效，服务端执行。
  **这是唯一能让「分支不堆积」从「靠自觉」变成「自动」的开关。**
- **怎么开**：Settings → General → Pull Requests → ✅ Automatically delete head branches
- **批量开**：
  ```bash
  gh api --method PATCH repos/OWNER/REPO -F delete_branch_on_merge=true
  ```
- **验证**：
  ```bash
  gh api repos/OWNER/REPO --jq '.delete_branch_on_merge'   # 期望 true
  ```

### ⚠️ 它只解决「未来」，不解决「过去」

```
开之前：每合一个 PR → 多一个残留分支（永远清不完）
开之后：合 PR 时自动删  → 不再新增
但：    已经堆着的，不会因为开了开关而消失 → 要手工清（用 repo-wrapup）
```

**实测（2026-10-20）**：一次性为 **7 个未归档仓库**开启（5 公开 + 2 私有）。
开启前 `ai-capability-store` 已累积 **17 个本地 + 22 个远端**已合并分支——
这些是开关开启前的历史残留，需要另外处理。

- [x] 未归档仓库已全部开启（2026-10-20）

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

# G. 仓库呈现与文档规范

> **原在 `global.md` 的 G07–G11，2026-10-20 迁来。**
> 原因：这五条全是 **GitHub 平台特定**的规范（README 索引、topics 搜索、llms.txt、AGENTS.md、git/gh）。
> 平台无关的工程约定留在 `global.md`。
>
> ⚠️ **本节与 A–F 的区别**：
> A–F 是「**去把开关打开**」；G 是「**产出物该写成什么样**」。
> 两者都是我们定的规范，不是 GitHub 的事实。

## R01 · GitHub：优先 `git`+`gh`，优先 SSH

**规则**

1. 涉及仓库、分支、PR、Issue、Actions 的操作，**优先用 `git` + `gh`**。
   **不要优先走浏览器 GUI。**
2. 仓库的 clone / fetch / pull / push **优先用 SSH**。**不要主动改回 HTTPS。**

**判据**

- 用的是 `gh` / `git`，而不是浏览器
- 新 clone 的仓库 remote 是 `git@github.com:…`

**原因（依据，不是独立条目）**

当前环境下 HTTPS 曾出现网络不稳定、重复连接失败和耗时较长的问题，SSH 实际更稳定。

> ⚠️ 这句是**依据**——**约束本身不因它过期而失效**。

**注意**

- 执行前如有必要，先 `gh auth status` 检查登录，`git remote -v` 检查协议
- **不要擅自批量改已有仓库的 remote**——那是带副作用的操作，先问王颖

**效力**：hard
**来源**：王颖原话 #18

---

---

---

## R02 · 仓库的对外呈现也是交付物

**规则**

README / description / topics 不是说明文档，是**别人能不能找到你、能不能用起来的唯一入口**。

**读它的是谁（实测，2026-10-20 traffic 基线）**

| 仓库 | 独立观看 | 独立克隆 |
|---|---|---|
| `ai-capability-store` | 1 | 331 |
| `feishu-retail-ops` | 2 | **1,917** |

→ **读者几乎全是 agent，不是人。**所以按「给 agent 读」写，不是按「给人看」写。

**必须写什么**

1. **前三行回答三问**：给谁用 / 它做什么（≤3 条，动词开头）/ 怎么开始（一行可复制命令）
2. 一句话写「**痛**」，不写「定义」
   - ❌「以 GitHub 为 Canonical Store，让不同 AI Agent 能发现…」
   - ✅「同一个 skill，在 ChatGPT 能装、在 Claude 也能装」
3. **安装命令必须能直接复制运行**（带完整包名，不是裸 `npm i`）
4. **写「不做什么」边界节**——防止 agent 误以为你能做它没有的能力
5. **术语表**：定义仓库里的专有名词（同一个词在不同项目意思可能完全不同）

**判据**

一个没听过这个项目的 agent，只读 README 前三行，
能不能说出「它给谁、解决什么痛」并跑起来？

**标准出处**：`ai-capability-store/automations/repo-discoverability/MAINTAINER-GUIDE.md`

> ⚠️ **本规则只写「执行时该做什么」。**D1–D6 的审计清单在 guide 里，**不要复制过来**——
> 否则两份真相迟早漂移。**审计按 guide 跑，执行按本规则做。**

**效力**：hard
**来源**：MAINTAINER-GUIDE（2026-10-20）+ traffic 实测

---

---

---

## R03 · topics 是布尔门，错了比没有更糟

**规则**

topic 决定你的仓库会出现在**哪些 topic 页**。
加错了，来看的人是白跑一趟，**长期是负资产**——它不像 README 写歪了还能改，**topic 是布尔门**。

**三问全中才加**

1. 这个仓库里**真的有**它的痕迹吗？（代码 / 配置 / 依赖 / 文档）
2. 一个搜这个词的人，看到这个仓库会觉得「对，我找的就是它」吗？
3. 这个 topic 下的**其他仓库**，和这个仓库是同一类东西吗？

**任何一个是「否」→ 不加。**

**不许做**

- ❌ 只因 README 里出现过这个词就加
  —— 实测踩过：文件树里一行 `AGENTS.md` 就让 `feishu-retail-ops` 被推成 `ai-agents`
- ❌ 加仓库里没有实现的东西（没有 MCP 实现却加 `mcp`）
- ❌ 为蹭流量加热门词（`llm` / `chatgpt` 但和项目无关）

**判据**

- `len(topics) >= 5`（少于这个，topic 搜索覆盖太窄）
- 每个 topic 都能指出它在仓库里的**出处**
- **改完 README 主题后，topics 有没有跟着改**（最容易漏的一步）

**效力**：hard
**来源**：MAINTAINER-GUIDE（2026-10-20）；含一次 dry-run 的真实误判证据

---

---

---

## R04 · 仓库的机器可读产物与交接资产，是我们自己维护的

**规则**

除了 README，每个仓库还有三类产物**必须由仓库负责 AI 自己产出和维护**。

> ⚠️ **审计只检查「有没有」。**内容是否同步、是否过期，**审计查不出来**——
> 那是执行方的责任，不是审计方的责任。

| 产物 | 是什么 | 什么时候必须更新 |
|---|---|---|
| **`llms.txt`** | 给 LLM 的机器可读入口（章节索引） | **README 章节变了 → 重新生成** |
| **DSH 徽章** | 署名 + 生态可追溯（**明确不是排名因素**） | 新建仓库时加；改 README 时确认还在 |
| **`topics.plan.json`** | topics 的出处记录 | **每次改 topics 时同步** |

**为什么这三样特别容易坏**

```
llms.txt 过期   →  审计只查存在，查不出它和 README 已经对不上
徽章缺失        →  软要求，最容易在重构 README 时被冲掉
topics 无出处   →  下一个人必须从头猜（实测：自动化推导 topics 第一次就错两个）
```

**判据**

- `llms.txt` 里列的章节，和 README **当前**的章节对得上吗？
- 徽章在 README 里吗？（可用 `dsh-badge` skill）
- 每个 topic 在 `topics.plan.json` 里都有记录吗？

**交接的硬要求**

> **不要把「我为什么选这个 topic」留在脑子里。**
> 写进 `topics.plan.json`，否则下一个 AI 会重新推导一遍，而且大概率推错。

**效力**：hard
**来源**：王颖 2026-10-20 指出「审计只审有没有，执行是我们」+ MAINTAINER-GUIDE D4/D6/第五节

---

---

---

## R05 · 项目 AGENTS.md 是 README 的补充，不是复制

**规则**

每个项目的 `AGENTS.md` 放的是**「agent 干活需要、但放进 README 会显得杂乱」**的信息。
**它补充 README，不是替代 README，也不是 README 的翻译版。**

**必须覆盖**

| 节 | 回答什么 |
|---|---|
| 项目概述 | 这是什么、给谁用 |
| **构建与测试命令** | **怎么跑起来、怎么测**（最重要，缺了 agent 就得自己猜） |
| 代码风格约定 | 新代码写成什么样 |
| 测试说明 | 测什么、怎么算通过 |
| 安全注意事项 | **什么不能碰** |

**有则补**：commit / PR 规范、安全陷阱、大数据集处理、部署步骤。

**和 `global.md` 的关系**

```
global.md        →  跨项目的 conventions（所有项目共性）
项目 AGENTS.md   →  本项目的操作性信息（build / test / style）
                 +  吸收 global 的共性
```

> **不要把 `global.md` 整份抄进项目 AGENTS.md**——那是两份真相。
> 也不要只写一句「见 global.md」——agent 不会主动跨仓库跳转。

**判据**

一个**不熟悉这个项目**的 agent，只读 `AGENTS.md`，能不能知道：
- 怎么把项目跑起来
- 怎么验证改动是对的
- 什么绝对不能碰

**实测缺口**（2026-10-20）

| 项目 | 构建/测试 | 风格约定 | 安全边界 |
|---|---|---|---|
| `feishu-retail-ops` · `teaming-bot` | ✓ | ✓ | ✓ |
| `InvestDesk` | ✓ | ✗ | ✓ |
| `personal-ip-hub` | ✓ | ✗ | ✗ |
| **`ai-capability-store`** | **✗** | **✗** | **✗** |

**效力**：hard
**来源**：王颖 2026-10-20 提出；依据 [AGENTS.md 官方](https://github.com/agentsmd/agents.md)
（OpenAI Codex 发起，Amp / Google Jules / Cursor / Factory 共同，现由 Linux Foundation 下
Agentic AI Foundation 托管）+ 5 个项目 AGENTS.md 实测

---

---

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
