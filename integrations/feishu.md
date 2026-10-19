---
id: INT-FEISHU-001
layer: integrations
system: 飞书（Feishu / Lark）
force: soft
scope: 用飞书 CLI 或 OpenAPI 读写多维表格时
status: VERIFIED
verified: 2026-10-19
source: 实测（lark-cli 1.0.97）
---

# 飞书 · 最小必知

> **只写"不看就会做错"的。**飞书自己的 API 文档不在这里复制——**不抄就不会过期。**
> 本文件 2026-10-19 实测，lark-cli 版本 1.0.97。

---

## 怎么进去

```bash
# 解析 Base 链接 → base_token
lark-cli base +url-resolve --url '<url>' --as user

# 高层快捷命令（推荐）
lark-cli base +record-list  --base-token <b> --table-id <t> --as user
lark-cli base +record-get   --base-token <b> --table-id <t> --record-id <r> --as user
lark-cli base +field-list   --base-token <b> --table-id <t> --as user

# 原始 API 逃生舱（同一条数据，形态不同——见下）
lark-cli api GET "/open-apis/bitable/v1/apps/<b>/tables/<t>/records/<r>" --as user
```

---

## ⚠️ 核心发现：同一个字段，CLI 和 OpenAPI 是**两套形态**

**这两个不是"语法糖"关系，是数据形态不同。混用会出错。**

实测对比（同一条记录、同一个字段，2026-10-19）：

| 字段类型 | 飞书 CLI（`+record-get`） | 原始 OpenAPI（`api GET`） |
|---|---|---|
| **URL** | `"[agentsmd/agents.md](https://github.com/…)"`<br>（已渲染成 Markdown 字符串） | `{"link": "https://…", "text": "agentsmd/agents.md"}`<br>**（对象）** |
| **数字** | `24740`（number） | `"24740"`**（字符串）** |
| **日期** | `"2026-10-03T00:00:00.000+08:00"`（ISO 字符串） | `1790956800000`**（Unix 毫秒）** |
| **单选** | `["参考中"]`**（数组）** | `"参考中"`**（字符串）** |
| **多选** | `["Discovery"]`（数组） | `["Discovery"]`（数组，一致） |
| **关联（link）** | `[{"id":"rec…"}]`<br>**只有 id，丢了 table_id 和显示名** | `[{"record_ids":[…],"table_id":"tbl…","text":"飞书助理","text_arr":[…],"type":"text"}]` |

### 一句话结论

> **CLI 给的是"人能看"的平铺形态；OpenAPI 给的是"程序能算"的带类型形态。**
> **读取口径要和写入口径一致**——不要用 CLI 读到的形态直接喂给 OpenAPI 写，反之亦然。

---

## 已踩过的坑（只追加，不重写）

### P001 · 飞书 CLI 读关联（link）字段是**有损**的

- **状态**：VERIFIED · 2026-10-19 · 本机实测
- **现象**：`+record-get` 返回关联字段为 `[{"id":"rec…"}]`，
  **只保留 id，丢掉了 `table_id`、显示文本 `text`、`record_ids`**
- **对比**：原始 OpenAPI 返回完整信息
  ```json
  [{"record_ids":["rec…"], "table_id":"tbl…", "text":"飞书助理",
    "text_arr":["飞书助理"], "type":"text"}]
  ```
- **为什么算坑**：**不报错。**它会安静地给你一个"看起来对"的数组，
  直到你拿它去解析或写回时才失败——而失败信息指向的是"无效引用"，
  不指向"CLI 丢了 table_id"。
- **正确做法**：
  - 需要解析关联 → 用 `lark-cli api GET /open-apis/bitable/v1/...`
  - 或者拿到 id 后**再查一次目标表**补出 `table_id` 和文本
  - **不要假设 CLI 返回的 `{"id"}` 就是完整的关联引用**

### P002 · 用户报告："CLI 读关联字段会提示无效引用"

- **状态**：OBSERVED（用户报告）· 2026-10-19 · **本机未复现**
- **说明**：用户在实际项目中遇到该报错。
  本机用 dry-run 验证时，CLI 的 `[{"id":…}]` 形态与完整形态**都能通过请求构造**，
  所以**报错发生在服务端校验或后续解析阶段，未定位到确切触发条件**。
- **关联怀疑**（**未证实**）：与 P001 同源——缺 `table_id` 导致引用无法解析。
- **待办**：下次遇到该报错时，记录完整请求与响应，把它从 OBSERVED 升级为 VERIFIED。

### P003 · `--output` 目标已存在时会拒绝写入

- **状态**：VERIFIED · 2026-10-19
- **现象**：`+record-list --output ./x.ndjson` 第二次执行报
  `failed_precondition: output file already exists`
- **正确做法**：加 `--overwrite`
- **为什么算坑**：报错类型是 `validation/failed_precondition`，
  容易被误读成"参数错了"或"表有问题"，实际只是文件已存在。

### P004 · Base 写操作是异步的

- **状态**：OBSERVED · 来源 `lark-base` skill 的执行契约
- **现象**：写入返回成功，**立即读回可能读到旧值**
- **为什么算坑**：不报错，会被误判成"写失败了"然后重复写
- **正确做法**：本轮变更全部提交完，**最后统一读一次**验收

---

## 和本机环境的边界

「`lark-cli` 在 DSH 沙箱里刷新 token 失败」这件事**不是飞书的问题**，
是**这台机器**的权限问题——所以不在本文件，属于 `memory.md`（环境事实）。
症状特征是 `keychain Set failed ... operation not permitted`。

---

## 待补

- [ ] 关联字段**写入**时的正确形态（目前只用 dry-run 验证了请求构造，未做真实写入对比）
- [ ] 双向关联（Two-way link）的形态是否与单向一致
- [ ] 人员 / 群聊 / 位置 / 附件字段的两种形态对比
- [ ] P002 的确切触发条件
