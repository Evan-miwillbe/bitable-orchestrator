# Bitable Orchestrator / 飞书多维表智能编排器

> 用自然语言驱动复杂的多维表业务逻辑——跨表关联、条件分支、批量处理、事务回滚，一句话搞定。

## 为什么需要这个？

飞书多维表（Bitable）的原生自动化只能处理简单的"当X发生时，做Y"。但真实业务逻辑远比这复杂：

| 业务场景 | 原生自动化 | Bitable Orchestrator |
|---------|-----------|---------------------|
| 订单确认→库存扣减→低库存告警→自动补货 | ❌ 无法跨表级联 | ✅ 一句话配置完整链路 |
| 金额分级审批（500/5000/50000三档） | ❌ 不支持条件分支 | ✅ 自动路由+通知 |
| 月底归档：汇总→生成报表→移动历史记录 | ❌ 需手动操作 | ✅ 定时自动执行 |
| 数据质量巡检：格式校验+去重+标记异常 | ❌ 无此能力 | ✅ 批量检测+智能标记 |
| 出错后恢复所有数据到操作前状态 | ❌ 不可能 | ✅ 快照+自动回滚 |

**核心价值**：把多维表从"记录数据的表格"升级为"可编程的业务引擎"。

---

## 这不是我们想象出来的需求

以下痛点来自 Reddit（r/Airtable, r/Notion, r/nocode）和中文社区（V2EX, 知乎）的真实用户反馈。Airtable/Notion 与飞书多维表功能高度同构，痛点直接适用。

### 痛点1：自动化条数限制，花钱也解决不了

> *"I just hit the wall with automation limit with our production system. Make or Zapier is a road through hell."*
> — r/Airtable, 印刷厂管理系统用户（排班/仓库/工资/快递全在表里）

> *"Going from $240/year to $8K+ just to get 50 more automations. I seriously don't understand."*
> — r/Airtable, CRM用户，9 upvotes, 25 comments

Airtable Team计划每个base限50条自动化。要更多？买Enterprise，$8K+/年，涨33倍。**Bitable Orchestrator 用AI驱动操作，不受原生自动化条数限制。**

### 痛点2：出错了没有后悔药

> *"I totally screwed up integrating 2 bases. Is there a way to revert back? Or is the lost data just that, lost?"*
> — r/Airtable, 用户不小心覆盖了数千条关联数据

> *"No native point-in-time data versioning. Users building fragile 3-table workarounds for basic temporal data."*
> — r/Airtable, 金融团队为追踪周度变化被迫建3张辅助表手动做快照

原生工具没有单条记录级别的撤销，只能恢复整个base快照。第三方备份工具（ProBackup, On2Air）因此形成了一个市场。**Bitable Orchestrator 每次操作前自动快照，失败自动回滚到操作前状态。**

### 痛点3：级联自动化被禁止

> *"After I spent the whole day making an automation that other automations require to be triggered, I searched it up and it says Notion intentionally has this limitation."*
> — r/Notion, 5+ upvotes

Notion为防无限循环，一刀切禁止自动化触发自动化。**Bitable Orchestrator 用依赖图+拓扑排序原生支持链式操作，同时内置循环保护。**

### 痛点4：零代码和写代码之间没有中间地带

> *"The solution for self-imposed limit is to take a tool marketed as no-code and write code. Airtable is losing its way."*
> — r/Airtable, 6 upvotes

> *"Non-technical teammates are starting to feel overwhelmed. What started as easy-to-adopt is turning into something that needs onboarding and training."*
> — r/Airtable, 团队10人

**自然语言就是中间地带。** 不用学代码，也不受零代码工具的限制。

### 痛点5：谁改了什么数据，无从得知

没有变更审计追踪是整个 no-code 数据库赛道的共同缺陷。**Bitable Orchestrator 记录每次操作的完整审计链：谁/什么时候/对什么数据/做了什么变更。**

### 中文社区的声音（V2EX / 知乎 / 人人都是产品经理）

> "商业标准版每月仅200次自动化运行次数，工作流和自动化共用同一额度。5人团队第一周就用完了。"
> — 知乎、[V2EX t/1143342](https://www.v2ex.com/t/1143342)

> "审批流程是飞书多维表格目前比较薄弱的环节，但却是B端产品较常需要使用到的功能。"
> — [人人都是产品经理](https://www.woshipm.com/evaluating/6150755.html)，专业产品分析

> "跨多维表格同步的数据不支持修改或编辑——只读同步。"
> — 博客园，NocoBase 替代方案对比文章

> "触发条件与执行操作为一对一关系，若有多对一的情况设置起来较为繁琐。"
> — [人人都是产品经理](https://www.woshipm.com/evaluating/5935411.html)

> "超过68%的企业面临'未授权访问'、'数据篡改'或'敏感信息泄露'问题。某制造企业因表格分享权限设置错误，导致商业秘密泄露，损失超千万元。"
> — FineReport 报表知识库

V2EX 上活跃讨论 NocoDB、Teable、NocoBase 等开源替代方案这一事实本身，就说明了用户对现有产品的不满。**Bitable Orchestrator 不是要替代多维表，而是补上它最缺的那块：复杂业务逻辑的自动化执行能力。**

---

## 特性

### 五阶段安全执行管线

```
自然语言 → 环境感知 → 执行计划 → Dry-Run验证 → 原子执行 → 审计记录
                                      ↑
                          看到预测结果再决定是否执行
```

### 核心能力

- **跨表关联操作** — 自动发现表间关系（显式关联+隐式匹配），支持任意深度的级联更新
- **条件分支** — IF/ELSE逻辑、多级路由、异常处理分支
- **批量处理** — 智能分页，自动限流，千条记录级别的批量操作
- **事务保障** — 操作前快照，失败自动回滚到操作前状态
- **Dry-Run模式** — 不实际修改数据，先模拟执行并展示预测结果
- **审计追踪** — 每次操作记录：谁/什么时候/对什么数据/做了什么变更
- **模式学习** — 从历史操作中提取可复用模板，同类需求一键复用
- **定时编排** — 将规则设为定时任务（如每天9点检查到期合同并提醒）
- **歧义消解** — 主动询问模糊指令，不做假设

### 安全设计

- DELETE操作强制二次确认
- 单次上限5000条，超过必须分批确认
- 敏感字段（密码/身份证/银行卡）不出现在日志中
- 失败自动回滚，不留半成品状态
- API调用自限频（100次/分钟）

---

## 快速开始

### 前置条件

- [飞书CLI](https://github.com/nicepkg/lark-cli) 已安装并完成授权
- Claude Code 已安装

### 安装

```bash
# 克隆到 Claude Code skills 目录
git clone https://github.com/Evan-miwillbe/bitable-orchestrator.git ~/.claude/skills/bitable-orchestrator
```

### 使用

在 Claude Code 中直接用自然语言描述你的需求：

```
/bitable-orchestrator

当"销售订单"表中新增状态为"已确认"的订单时，
自动从"库存表"扣减对应商品数量。
如果库存低于安全线，在"采购需求"表创建补货申请，
同时通知采购群。
```

Claude Code 会：
1. 自动读取相关表的结构和样本数据
2. 生成可视化的执行计划
3. 模拟执行（Dry-Run），展示预测结果
4. 等你确认后才真正执行
5. 记录完整审计日志

---

## 使用场景

### 场景一：销售→库存→采购 联动

```
用户：当销售订单确认时，扣减库存。库存低于安全线就自动创建采购申请。
```

**编排器处理**：
- 解析3张表（订单/库存/采购）的结构和关联
- 生成 QUERY → LOOP → VALIDATE → UPDATE → BRANCH → CREATE → NOTIFY 操作链
- Dry-Run预测：哪些SKU会低于安全线、会创建几条采购申请
- 执行时逐条校验，库存不足则跳过并告警

### 场景二：月度数据归档

```
用户：每月1号，把上月已关闭的工单按部门汇总写入月报表，原始记录移到历史表。
```

**编排器处理**：
- AGGREGATE（按部门分组，计算平均处理时长/总数/满意度）
- CREATE（写入月报表）
- BATCH_MOVE（原始记录→历史表，分批执行）
- 设为定时任务：每月1日 09:00 自动执行

### 场景三：数据质量巡检

```
用户：检查客户表：手机号格式、邮箱格式、重复记录、长期未联系客户。异常的标记出来。
```

**编排器处理**：
- 并行4项检测（格式/格式/唯一性/时间）
- 标记异常记录 + 汇总到质量报告表
- 输出：X条格式错误、Y条疑似重复、Z条可能流失

### 场景四：多级审批路由

```
用户：报销单按金额分流：<500自动通过，500-5000给经理，>5000经理通过后给总监。
```

**编排器处理**：
- BRANCH三路条件分支
- 自动通过：直接UPDATE状态
- 需审批：NOTIFY审批人 + 设置pending状态
- 双级审批：串联两次通知 + 状态流转

---

## 技术架构

```
┌─────────────────────────────────────────────────────────┐
│                    Bitable Orchestrator                   │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │  Intent   │  │  Plan    │  │  Dry-Run │  │ Execute │ │
│  │  Parser   │→│ Generator │→│  Engine  │→│  Engine │ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │
│       ↕              ↕              ↕             ↕       │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Safety Layer (快照/回滚/限频)          │   │
│  └──────────────────────────────────────────────────┘   │
│       ↕              ↕              ↕             ↕       │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Feishu CLI Layer (lark-cli)           │   │
│  └──────────────────────────────────────────────────┘   │
│                                                           │
├─────────────────────────────────────────────────────────┤
│  Template Library │ Audit Logger │ Schedule Engine        │
└─────────────────────────────────────────────────────────┘
```

### 原子操作类型

| 操作 | 说明 | 可回滚 |
|------|------|--------|
| `QUERY` | 读取记录 | — |
| `CREATE` | 创建记录 | ✅ 删除已创建的 |
| `UPDATE` | 更新字段 | ✅ 从快照恢复 |
| `DELETE` | 删除记录 | ✅ 从快照重建 |
| `AGGREGATE` | 聚合计算 | — |
| `VALIDATE` | 断言校验 | — |
| `BRANCH` | 条件分支 | — |
| `LOOP` | 批量遍历 | ✅ 逐条回滚 |
| `NOTIFY` | 发送通知 | ❌ 不可撤回 |

---

## 与其他 Skill 组合

| 组合 | 效果 |
|------|------|
| + lark-minutes | 会议决策→自动创建多维表追踪记录 |
| + lark-whiteboard | 数据聚合后→生成可视化图表 |
| + lark-im | 操作结果→通知相关人员 |
| + schedule | 编排规则→定时自动执行 |
| + lark-drive | 文件上传→解析→写入多维表 |

---

## 设计哲学

1. **声明式 > 命令式** — 描述目标，不描述步骤
2. **可预测 > 黑箱** — Dry-Run让你执行前就知道结果
3. **可撤销 > 不可逆** — 快照+回滚，永远有后悔药
4. **渐进信任** — 小操作直接执行，大操作需确认，高危操作二次确认
5. **透明审计** — 每次变更有据可查

---

## 适用对象

- **运营/业务人员**：不写代码，用自然语言配置复杂业务规则
- **项目管理者**：自动化项目数据的归档、汇总、报告
- **财务/HR**：审批流程自动化、数据合规检查
- **IT管理员**：数据迁移、质量巡检、系统间同步

---

## 参赛信息

本项目参加 [飞书CLI创作者大赛](https://bytedance.larkoffice.com/docx/HWgKdWfeSoDw36xu7EYctBrUnsg)（GitHub赛道）。

**解决的核心问题**：飞书多维表原生自动化能力不足以处理企业真实业务逻辑的复杂度——跨表级联、条件分支、事务一致性、批量处理这些需求在V2EX和知乎上被反复提及但从未被解决。

**技术亮点**：
- 五阶段安全管线（不是简单的API封装）
- Dry-Run预测执行（执行前可见结果）
- 快照+原子回滚（数据安全保障）
- 模式学习+模板库（越用越智能）
- 定时编排支持（设好规则，自动执行）

---

## License

MIT
