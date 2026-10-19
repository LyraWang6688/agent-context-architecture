# Agent Context Architecture

王颖的个人 Agent 上下文架构：把「怎么和我合作 / 我是谁 / 什么必须遵守」沉淀成任何 agent 都能取用的文本。

**入口 → [`INDEX.md`](INDEX.md)**

- `preferences.md` — 用户偏好
- `memory.md` — 长期记忆
- `global.md` — 全局指令（所有项目通用规则 G01–G13）
- `integrations/` — 第三方系统集成（按生态分文件）
- `hooks/` — Hook

设计原则：**信息放在正确的层级**；规则必须可判定；集成知识靠验证定对错、不抄官方文档。
