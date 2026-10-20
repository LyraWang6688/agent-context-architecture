# Agent Context Architecture · 索引

> **这是本包的唯一入口。**读完这一份，你就知道包里有什么、该去读哪个。
> 不要遍历目录。

---

## 这个包是什么

王颖与大量 agent（Codex、豆包工作、WorkBuddy 等）协作过程中沉淀出的**个人上下文**。
交给任意 agent 取用，让它**不用追问就能按王颖的方式工作**。

使用者可能是**任意 agent、任意运行时**。所以本包与运行时无关：
不依赖特定工具、不依赖本机路径、不依赖某个平台的能力。

---

## 结构

```
Agent Context Architecture/
├── INDEX.md          ← 本文件（唯一入口）
├── preferences.md    用户偏好      —— 单独文件
├── memory.md         长期记忆      —— 单独文件
├── global.md         全局指令      —— 单独文件
├── integrations/     集成          —— 文件夹（分不同生态）
└── hooks/            Hook          —— 文件夹
```

**为什么前三个是单独文件、后两个是文件夹：**

| | 形态 | 原因 |
|---|---|---|
| preferences / memory / global | **单独文件** | 都是关于**王颖本人**的，各自就是一份完整的东西 |
| integrations | **文件夹** | 要**分不同的生态**（飞书 / 豆包 / WorkBuddy…），一个生态一份 |
| hooks | **文件夹** | 内部结构待定 |

**没有别的部分。**项目指令、模块指令、任务指令、Runtime Context、Skill 都**不在这里**——
因为它们必须依附某个仓库 / 模块 / 某一次任务 / 某个 agent 宿主才能成立，
**不能跟着王颖走。**

---

## 该读哪个（派发表）

| 你想知道 | 读这个 |
|---|---|
| **agent 该怎么工作**（怎么下结论、怎么说、怎么开会话、怎么交接） | [`preferences.md`](preferences.md) |
| 王颖喜欢 agent 怎么和她合作（详略、节奏、输出方式） | [`preferences.md`](preferences.md) |
| 王颖是谁、她的背景、有什么环境和工具 | [`memory.md`](memory.md) |
| **产出物该长什么样**（代码结构、可维护性、仓库与文档） | [`global.md`](global.md) |
| 某个第三方系统怎么接、哪里会翻车 | [`integrations/README.md`](integrations/README.md) |
| 飞书 CLI 与 OpenAPI 的形态差异、踩过的坑 | [`integrations/feishu.md`](integrations/feishu.md) |
| **配置 GitHub 仓库 / 安全基线 / CI 门禁** | **[`integrations/github.md`](integrations/github.md)** |
| 什么必须自动发生、什么会被系统拦 | [`hooks/README.md`](hooks/README.md) |

---

## 五个部分怎么区分（最容易混的地方）

```
preferences   约束「agent 行为」 —— 怎么想、怎么说、怎么开会话
              不会进任何项目的 AGENTS.md

global        约束「产出物」     —— 代码、仓库、文档长什么样
              是项目 AGENTS.md 的共性上游

memory        描述「是」         —— 不规定该做什么
integrations  描述「世界」       —— 别人定的，靠验证定对错
hooks         描述「事件」       —— 某某发生时，必须做某某
```

**三条最常混的线：**

- **preferences vs global**：**约束行为** 还是 **约束产出物**
  （行为准则实测在你的项目 AGENTS.md 里出现 **0 次**）
- **memory vs global**：**描述「是」** 还是 **规定「该」**
- **global vs hooks**：**状态** 还是 **事件**

> ⚠️ `preferences` 内部有两种效力：**协作偏好是 `soft`，工作要求是 `hard`**。

---

## 效力标记

读任何一份文件时，注意它的效力：

| 标记 | 含义 | 该怎么做 |
|---|---|---|
| `hard` | 硬约束 | 必须遵守；违反需明确说明理由 |
| `soft` | 默认偏好 | 默认遵守；有明确理由可偏离，但**要说出来** |
| `context` | 背景信息 | **不构成约束**，只用于理解上下文 |

> ⚠️ **不要把 `soft` 当 `hard` 执行**——那会让 agent 僵化，
> 反而违背王颖的偏好。

---

## 当前状态

**架构已搭好，内容待写入。**

各文件里现在只有「这部分是什么 / 回答什么问题 / 收什么 / 不收什么」，
**还没有具体内容。**
