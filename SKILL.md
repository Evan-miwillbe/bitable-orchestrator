---
name: bitable-orchestrator
description: "飞书多维表智能编排器——用自然语言描述复杂业务逻辑，AI自动规划、验证、执行跨表操作。支持条件分支、批量处理、事务回滚、审计追踪。触发词：'多维表编排'、'bitable自动化'、'跨表操作'、'业务流程自动化'、'表格工作流'、'bitable-orchestrator'"
---

# Bitable Orchestrator v1.0

> 多维表业务逻辑编排引擎：自然语言 → 执行计划 → 验证 → 原子执行 → 审计

---

## 核心问题

飞书多维表的原生自动化仅支持简单的触发-动作模式。当企业业务逻辑涉及：
- 跨表关联查询（订单表 ↔ 库存表 ↔ 供应商表）
- 条件分支（金额>5万走VP审批，否则走经理审批）
- 批量级联更新（一个状态变更触发20个下游字段更新）
- 数据一致性保证（要么全做，要么全不做）

...原生能力就完全不够用了。V2EX、知乎上这是飞书多维表最高频的吐槽。

本skill将多维表升级为**可编程的业务引擎**——用自然语言描述意图，AI负责规划、验证、执行。

---

## 架构

```
用户自然语言描述业务规则
        │
        ▼
┌─────────────────────────────┐
│  Phase 1: 意图解析 + 环境感知  │  ← 读取表结构、字段类型、现有数据样本
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│  Phase 2: 执行计划生成        │  ← 分解为原子操作序列 + 依赖图
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│  Phase 3: Dry-Run 验证       │  ← 模拟执行，预测结果，检测冲突
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│  Phase 4: 原子执行 + 回滚保障  │  ← 逐步执行，失败即回滚到快照
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│  Phase 5: 审计 + 模式学习      │  ← 记录操作链，提取可复用模板
└─────────────────────────────┘
```

---

## Phase 1: 意图解析 + 环境感知

### Step 1.1: 收集业务规则

用户以自然语言描述需求。支持的描述模式：

| 模式 | 示例 |
|------|------|
| 条件更新 | "当'订单表'的状态变为'已发货'时，自动更新'库存表'中对应SKU的数量减去发货数" |
| 跨表汇总 | "把所有'销售记录'按月汇总到'月度报表'中，包含总金额、订单数、平均客单价" |
| 级联审批 | "金额>5万需要VP审批，1-5万经理审批，<1万自动通过，审批结果写回原表" |
| 数据清洗 | "找出'客户表'中手机号格式错误的记录，标记为'待修正'，并在'异常日志'中记录" |
| 定时聚合 | "每周一从'任务表'生成本周待办汇总，按负责人分组，写入'周报表'" |
| 关联创建 | "当'合同表'新增记录时，自动在'回款计划表'中创建对应的分期记录" |

### Step 1.2: 环境感知（自动执行）

```
FOR each 涉及的表:
  1. lark-cli bitable get-fields → 获取字段定义（名称、类型、选项值）
  2. lark-cli bitable list-records --page-size=5 → 采样数据（理解实际格式）
  3. 检测表间关联：
     - 显式关联：关联字段 / 查找引用字段
     - 隐式关联：字段名相同/相似 + 数据值匹配（如两表都有"客户ID"）
  4. 识别约束：
     - 必填字段（创建/更新时不可缺省）
     - 选项字段（值必须在预定义列表内）
     - 数字字段范围（有无最大/最小值限制）
     - 唯一性约束
```

**输出：环境快照**（写入工作目录 `env_snapshot.md`）：
```markdown
## 涉及表
| 表名 | 表ID | 字段数 | 记录数(估) | 关键字段 |
|------|------|--------|-----------|---------|

## 表间关系图
订单表 --[客户ID]--> 客户表
订单表 --[SKU]--> 库存表
库存表 --[供应商ID]--> 供应商表

## 约束清单
- 订单表.金额: number, 必填, ≥0
- 订单表.状态: single_select, options=[待处理,处理中,已完成,已取消]
- 库存表.数量: number, 必填, ≥0（⚠️ 减到负数=业务异常）
```

### Step 1.3: 歧义消解

对自然语言中可能产生歧义的部分，**主动询问**而非假设：

| 歧义类型 | 询问示例 |
|---------|---------|
| 范围不明 | "更新'所有'记录还是只更新今天的？" |
| 冲突处理 | "如果目标记录已存在，覆盖还是跳过？" |
| 边界条件 | "库存减到0以下时：阻止操作还是标记告警？" |
| 执行时机 | "立即执行一次，还是设为定时规则？" |

---

## Phase 2: 执行计划生成

### Step 2.1: 分解为原子操作

将用户意图分解为不可再分的操作序列。每个原子操作属于以下类型之一：

```
QUERY(table, filter, fields)        → 读取记录
CREATE(table, record_data)          → 创建记录
UPDATE(table, record_id, changes)   → 更新字段
DELETE(table, record_id)            → 删除记录（⚠️ 高危）
AGGREGATE(table, group_by, metrics) → 聚合计算
VALIDATE(condition, on_fail)        → 断言校验
BRANCH(condition, true_ops, false_ops) → 条件分支
LOOP(source_query, per_record_ops)  → 批量遍历
NOTIFY(channel, message)            → 发送通知
```

### Step 2.2: 依赖图构建

```
ops = [op1, op2, ..., opN]
FOR each pair (opi, opj) where i < j:
  IF opj.input DEPENDS ON opi.output:
    add_edge(opi → opj)
  IF opj MODIFIES same record as opi:
    add_edge(opi → opj)  // 顺序约束

topological_sort(ops) → execution_order
identify_parallel_groups(ops) → 可并行的操作组
```

### Step 2.3: 计划可视化（展示给用户确认）

```markdown
## 执行计划

### Step 1 [QUERY] 读取订单表
- 过滤: 状态 = "已发货" AND 发货日期 = 今天
- 预计命中: ~15条记录

### Step 2 [LOOP] 对每条订单执行:
  ├─ 2.1 [QUERY] 查询库存表: SKU = 订单.商品编号
  ├─ 2.2 [VALIDATE] 库存.数量 ≥ 订单.发货数量
  │   └─ on_fail: 标记异常，跳过此订单，继续下一条
  ├─ 2.3 [UPDATE] 库存表: 数量 -= 订单.发货数量
  └─ 2.4 [UPDATE] 订单表: 库存扣减状态 = "已完成"

### Step 3 [AGGREGATE] 汇总今日变更
- 总扣减SKU数 / 总扣减数量 / 异常订单列表

### Step 4 [CREATE] 写入日志表
- 操作时间 / 操作人 / 变更汇总 / 异常列表

---
⚡ 预计影响: 15条订单 + 12条库存记录 + 1条日志
⏱️ 预计耗时: ~30秒
⚠️ 风险点: 库存可能减为负数（已设VALIDATE防护）
```

**用户确认后才进入执行阶段。**

---

## Phase 3: Dry-Run 验证

### 3.1 模拟执行

在不实际修改数据的前提下，模拟整个执行计划：

```
FOR each op in execution_order:
  simulated_state = apply_op_to_virtual_state(op, current_virtual_state)
  
  检查:
  - [ ] 目标记录是否存在？
  - [ ] 字段类型是否匹配（不能把文本写入数字字段）？
  - [ ] 选项值是否在合法范围内？
  - [ ] 数字计算是否会溢出/为负？
  - [ ] 必填字段是否被覆盖为空？
  - [ ] 是否会违反唯一性约束？
  - [ ] 批量操作是否会超过API频率限制？
```

### 3.2 冲突检测

| 冲突类型 | 检测方法 | 处理策略 |
|---------|---------|---------|
| 写写冲突 | 同一记录被多步修改同一字段 | 合并为最终值，告警给用户 |
| 读写冲突 | 读取的数据可能在执行期间被他人修改 | 标记为"弱一致性风险" |
| 级联风险 | 修改触发飞书原生自动化，产生二次变更 | 列出可能的连锁反应 |
| 容量风险 | 批量创建/更新超过1000条 | 自动分页执行，加入延迟 |

### 3.3 预测报告

```markdown
## Dry-Run 结果

✅ 操作可行性: 全部通过
⚠️ 风险提示:
  - Step 2.3: 3条记录库存将降至10以下（低库存警告）
  - Step 2 整体: 预计15次API调用，约需25秒

📊 预测变更:
| 表 | 创建 | 更新 | 删除 |
|----|------|------|------|
| 订单表 | 0 | 15 | 0 |
| 库存表 | 0 | 12 | 0 |
| 日志表 | 1 | 0 | 0 |

确认执行？[是/否/调整]
```

---

## Phase 4: 原子执行 + 回滚保障

### 4.1 快照机制

执行前为所有受影响记录创建快照：

```
snapshot = {}
FOR each record_id in affected_records:
  snapshot[record_id] = lark-cli bitable get-record <table_id> <record_id>
  
SAVE snapshot → workspace/snapshots/{timestamp}.json
```

### 4.2 执行引擎

```
results = []
FOR each op in execution_order:
  TRY:
    result = execute_op(op)
    results.append({op, status: "success", result})
    
    // 进度汇报（每完成10个操作或每30秒）
    IF should_report_progress():
      print(f"进度: {completed}/{total} | 成功:{success_count} 失败:{fail_count}")
      
  CATCH error:
    results.append({op, status: "failed", error})
    
    // 错误分类
    IF error.type == "rate_limit":
      wait(error.retry_after)
      retry(op)
    ELIF error.type == "permission_denied":
      ABORT + rollback_all(results)
      报告权限问题
    ELIF error.type == "field_validation":
      IF op.on_fail == "skip":
        log_warning(op, error)
        continue
      ELSE:
        ABORT + rollback_all(results)
    ELSE:
      ABORT + rollback_all(results)
```

### 4.3 回滚协议

```
FUNCTION rollback_all(completed_results):
  // 逆序回滚已执行的操作
  FOR each result in REVERSE(completed_results):
    IF result.op.type == "UPDATE":
      restore_from_snapshot(result.record_id)
    ELIF result.op.type == "CREATE":
      delete_record(result.created_id)
    ELIF result.op.type == "DELETE":
      recreate_from_snapshot(result.record_id)
      
  REPORT:
    "⚠️ 执行在Step {N}失败，已回滚{M}个操作到执行前状态"
    "失败原因: {error}"
    "建议: {fix_suggestion}"
```

### 4.4 批量操作优化

```
IF total_records > 100:
  // 分批执行，避免超时和频率限制
  batches = chunk(records, size=50)
  FOR each batch in batches:
    execute_batch(batch)
    wait(1000ms)  // API友好间隔
    report_batch_progress()
    
IF total_records > 1000:
  // 超大批量警告
  confirm_with_user("即将操作{N}条记录，预计耗时{T}分钟，确认继续？")
```

---

## Phase 5: 审计 + 模式学习

### 5.1 操作审计日志

每次执行完成后，自动记录到指定飞书文档或多维表：

```markdown
## 操作记录 {timestamp}

**操作人**: {user}
**描述**: {原始自然语言指令}
**影响范围**: {表名列表} | {记录数}
**执行结果**: ✅ 成功 / ⚠️ 部分成功 / ❌ 失败回滚
**耗时**: {duration}
**快照位置**: snapshots/{id}.json

### 变更明细
| 表 | 记录ID | 字段 | 旧值 | 新值 |
|----|--------|------|------|------|
```

### 5.2 模式识别与模板提取

```
每次成功执行后:
  1. 提取操作模式的抽象描述
     原始: "当订单状态=已发货时，库存-=发货数"
     抽象: "WHEN table_A.field_X = VALUE, UPDATE table_B.field_Y -= table_A.field_Z 
            WHERE table_B.key = table_A.foreign_key"
     
  2. 检查是否与已有模板相似（>70%结构匹配）
     是 → 强化该模板的置信度
     否 → 新建候选模板
     
  3. 候选模板被成功使用3次以上 → 提升为正式模板
     正式模板可在后续操作中被推荐：
     "检测到你的需求与模板'跨表级联更新'相似，是否使用？"
```

### 5.3 内置模板库

| 模板名 | 适用场景 | 涉及操作 |
|--------|---------|---------|
| 跨表级联更新 | A表状态变更 → B表数据同步 | QUERY → LOOP → VALIDATE → UPDATE |
| 定时汇总报表 | 从明细表按时间聚合到汇总表 | QUERY → AGGREGATE → CREATE/UPDATE |
| 条件审批路由 | 按金额/类型分级审批 | QUERY → BRANCH → UPDATE → NOTIFY |
| 数据清洗标记 | 检测异常数据并标记 | QUERY → LOOP → VALIDATE → UPDATE |
| 批量关联创建 | 主表新增时自动创建子表记录 | QUERY → LOOP → CREATE |
| 库存扣减 | 出库时自动减库存+校验 | QUERY → VALIDATE → UPDATE → NOTIFY |
| 到期提醒 | 检测即将到期的记录并通知 | QUERY(date filter) → LOOP → NOTIFY |
| 数据迁移 | 从旧表结构迁移到新表 | QUERY(all) → TRANSFORM → CREATE |

---

## 安全协议

### 硬性规则（不可覆盖）

1. **DELETE操作必须二次确认** — 即使在dry-run通过后，删除前再次确认
2. **单次操作上限5000条记录** — 超过时强制分批+显式确认
3. **快照不可跳过** — 任何写操作前必须完成快照
4. **权限最小化** — 只请求操作所需的最小权限范围
5. **敏感字段保护** — 包含"密码/token/secret/身份证/银行卡"的字段不出现在日志中
6. **频率自限** — 单次执行最多100次API调用/分钟，避免触发平台限流

### 防误操作设计

| 信号 | 含义 | 响应 |
|------|------|------|
| 影响>100条且包含DELETE | 高危操作 | 强制dry-run + 二次确认 + 输出影响列表前10条 |
| 修改字段含"金额/价格/数量" | 财务敏感 | 展示计算公式 + 抽样验证3条 |
| 目标表>10万条 | 大表风险 | 建议先在过滤后的子集上测试 |
| 用户指令含"全部/所有" | 范围风险 | 确认是否真的是全量，还是有隐含条件 |

---

## 错误恢复策略

### 常见错误及自修复

| 错误类型 | 自动修复 | 需要用户介入 |
|---------|---------|------------|
| Rate limit | 自动重试(指数退避) | 否 |
| 记录不存在 | 跳过+记录日志 | 否（除非是关键记录） |
| 字段值不合法 | 尝试类型转换/截断 | 转换失败时 |
| 网络超时 | 重试3次 | 持续失败时 |
| 权限不足 | — | 是（提示需要开通的权限） |
| 表被锁定 | 等待+重试 | 超过5分钟时 |

### 部分失败处理

当批量操作中部分记录失败：
1. 成功的操作保留（已commit）
2. 失败的记录列出具体原因
3. 生成"重试命令"——只针对失败记录重新执行
4. 用户可选择：接受部分成功 / 全量回滚

---

## 定时编排（Schedule模式）

除了一次性执行，支持将编排规则设为定时任务：

```
用户: "每天早上9点，检查'合同表'中今天到期的合同，给负责人发飞书消息提醒"

→ 生成定时规则:
  schedule: "0 9 * * *"
  operation: [
    QUERY(合同表, 到期日=today),
    LOOP(results, [
      NOTIFY(record.负责人, "合同'{record.合同名}'今日到期，请处理")
    ])
  ]
```

定时编排的额外约束：
- 必须指定"静默模式"（成功时不打扰）vs"汇报模式"（每次都汇报）
- 必须指定错误通知渠道（失败时通知谁）
- 首次创建时强制执行一次dry-run验证
- 每次定时执行自动记录审计日志

---

## 使用示例

### 示例1：销售订单→库存扣减→低库存告警

```
用户：当销售订单表中新增一条状态为"已确认"的订单时，
     自动从库存表中扣减对应商品的数量。
     如果库存扣减后低于安全库存线，在"采购需求"表中自动创建一条补货申请，
     同时给采购群发消息通知。
```

### 示例2：月度数据归档

```
用户：每月1号，把上个月"工单表"中状态为"已关闭"的工单，
     汇总统计后写入"月度工单报表"（按部门分组，计算平均处理时长、总工单数、
     满意度均值），然后把原始记录移动到"历史工单"表中。
```

### 示例3：多级审批流程

```
用户：报销单提交后，根据金额自动分流：
     - 500元以下：自动审批通过，更新状态
     - 500-5000元：发消息给直属经理，等待审批
     - 5000元以上：先发经理，经理通过后再发给财务总监
     审批结果写回报销单表的"审批状态"字段
```

### 示例4：数据质量巡检

```
用户：检查"客户信息表"的数据质量：
     - 手机号不是11位数字的 → 标记为"格式错误"
     - 邮箱不含@的 → 标记为"格式错误"
     - "最后联系日期"超过180天的 → 标记为"可能流失"
     - 同一手机号出现多次的 → 标记为"疑似重复"
     把所有异常记录汇总到"数据质量报告"表中
```

---

## CLI命令参考（skill内部使用）

本skill依赖 `lark-cli` 的以下命令（通过 lark-base / lark-sheets skill调用）：

```bash
# 表结构
lark-cli bitable list-tables <app_token>
lark-cli bitable get-fields <app_token> <table_id>

# 记录操作
lark-cli bitable list-records <app_token> <table_id> [--filter] [--page-size] [--page-token]
lark-cli bitable get-record <app_token> <table_id> <record_id>
lark-cli bitable create-record <app_token> <table_id> --fields '{}'
lark-cli bitable update-record <app_token> <table_id> <record_id> --fields '{}'
lark-cli bitable delete-record <app_token> <table_id> <record_id>
lark-cli bitable batch-create-records <app_token> <table_id> --records '[...]'
lark-cli bitable batch-update-records <app_token> <table_id> --records '[...]'

# 通知
lark-cli im send --to <user/group> --content '...'
```

---

## 与其他skill的协作

| 场景 | 组合方式 |
|------|---------|
| 定时编排 | bitable-orchestrator + /schedule → 定期执行规则 |
| 会议→任务→表 | /lark-minutes → 提取action → bitable-orchestrator → 写入追踪表 |
| 报表可视化 | bitable-orchestrator聚合数据 → /lark-whiteboard → 生成图表 |
| 数据导入 | /lark-drive 获取文件 → 解析 → bitable-orchestrator → 写入表 |

---

## 设计哲学

1. **声明式优于命令式** — 用户说"要什么结果"，不用说"怎么做"
2. **可预测优于可惊喜** — dry-run让用户在执行前就知道会发生什么
3. **可回滚优于不可逆** — 任何操作都应该能撤销
4. **渐进信任** — 小操作自动执行，大操作需要确认，危险操作需要二次确认
5. **模式复用** — 同类操作只描述一次，后续自动匹配模板
6. **审计透明** — 谁在什么时候对什么数据做了什么变更，永远有据可查

---

## 版本记录

| 版本 | 日期 | 变化 |
|------|------|------|
| v1.0 | 2026-05-05 | 初始版本：五阶段架构 + 安全协议 + 模板库 + 定时编排 |
