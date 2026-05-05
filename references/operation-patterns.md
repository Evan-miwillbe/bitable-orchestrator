# 操作模式参考

## 模式1：跨表级联更新

**场景**：A表状态变更 → 触发B表数据同步更新

**操作序列**：
```
QUERY(A, filter=status_changed) 
→ LOOP(results) {
    QUERY(B, filter=foreign_key_match)
    → VALIDATE(business_rule)
    → UPDATE(B, computed_changes)
    → UPDATE(A, sync_status="已同步")
  }
```

**关键设计点**：
- 外键匹配策略：精确匹配 > 模糊匹配
- 批量优化：收集所有B表变更，batch_update一次性提交
- 幂等性：重复执行不产生副作用（用sync_status防重）

**常见变体**：
- 一对多级联（1条A→N条B）
- 多级级联（A→B→C，需要拓扑排序）
- 条件级联（只有满足条件的B记录才更新）

---

## 模式2：条件路由分发

**场景**：根据记录字段值，将操作路由到不同分支

**操作序列**：
```
QUERY(source, filter=pending)
→ LOOP(results) {
    BRANCH(
      condition_1: amount < 500 → [UPDATE(status="自动通过")],
      condition_2: amount < 5000 → [NOTIFY(manager), UPDATE(status="待经理审批")],
      condition_3: amount >= 5000 → [NOTIFY(manager), NOTIFY(director), UPDATE(status="待总监审批")],
      default: [UPDATE(status="异常"), NOTIFY(admin)]
    )
  }
```

**关键设计点**：
- 条件互斥性检查：确保每条记录只匹配一个分支
- Default兜底：未匹配任何条件时的处理
- 状态机一致性：状态值必须在预定义选项内

---

## 模式3：定时聚合报表

**场景**：按时间窗口汇总明细数据，写入汇总表

**操作序列**：
```
QUERY(detail_table, filter=date_range)
→ AGGREGATE(
    group_by=[department, category],
    metrics={
      count: COUNT(*),
      total: SUM(amount),
      avg: AVG(duration),
      max: MAX(priority)
    }
  )
→ LOOP(aggregated_results) {
    QUERY(summary_table, filter=same_group_same_period)
    → BRANCH(
        exists: UPDATE(summary_record, new_metrics),
        not_exists: CREATE(summary_table, {group + period + metrics})
      )
  }
```

**关键设计点**：
- Upsert逻辑：已存在则更新，不存在则创建
- 时间窗口对齐：月度=自然月，周度=周一到周日
- 增量vs全量：首次全量计算，后续可增量追加

---

## 模式4：数据质量巡检

**场景**：按规则扫描数据异常，标记并汇报

**操作序列**：
```
rules = [
  {name: "手机号格式", table: T, field: "phone", check: matches("^1[3-9]\\d{9}$")},
  {name: "邮箱格式", table: T, field: "email", check: contains("@")},
  {name: "重复检测", table: T, field: "phone", check: unique()},
  {name: "时效检测", table: T, field: "last_contact", check: within_days(180)},
]

FOR each rule:
  QUERY(rule.table, all_records)
  → LOOP(records) {
      VALIDATE(rule.check)
      → on_fail: [
          UPDATE(record, quality_flag="异常", quality_note=rule.name),
          CREATE(exception_log, {record_id, rule_name, current_value, expected})
        ]
    }

AGGREGATE(exception_log, group_by=rule_name, count)
→ CREATE(quality_report, summary)
→ NOTIFY(data_owner, report_link)
```

**关键设计点**：
- 规则可组合：多条规则独立运行，异常标记可叠加
- 不修改业务数据：只标记flag字段，不改原始值
- 去重：同一记录同一规则不重复记录

---

## 模式5：批量关联创建

**场景**：主表新增记录时，自动在子表创建关联记录

**操作序列**：
```
QUERY(master_table, filter=created_today AND no_child_records)
→ LOOP(results) {
    template = get_child_template(record.type)
    FOR each child_config in template:
      CREATE(child_table, {
        parent_id: record.id,
        ...child_config.default_fields,
        due_date: record.start_date + child_config.offset_days
      })
  }
→ UPDATE(master_table, child_created=true)
```

**关键设计点**：
- 模板化：不同类型的主记录对应不同的子记录模板
- 日期计算：子记录截止日相对于主记录起始日偏移
- 防重：检查是否已创建过子记录

---

## 模式6：数据迁移/重构

**场景**：从旧表结构迁移到新表，含字段映射和数据转换

**操作序列**：
```
field_mapping = {
  old_field_1: new_field_A,  // 直接映射
  old_field_2: transform(new_field_B, fn),  // 需要转换
  // old_field_3: 废弃，不迁移
  // new_field_C: 新增，需要默认值
}

QUERY(old_table, all_records, page_size=100)
→ LOOP(pages) {
    batch = []
    FOR each record in page:
      new_record = apply_mapping(record, field_mapping)
      new_record.migration_source = old_table.id
      new_record.migration_date = now()
      batch.append(new_record)
    
    BATCH_CREATE(new_table, batch)
    // 验证：随机抽查3条对比
    VALIDATE(sample_check(old_records, new_records, 3))
  }

AGGREGATE(new_table, count) vs AGGREGATE(old_table, count)
→ VALIDATE(counts_match, tolerance=0)
→ NOTIFY(admin, "迁移完成: {count}条记录, 耗时{duration}")
```

**关键设计点**：
- 分页处理：避免内存溢出
- 抽样验证：每批次随机检查数据正确性
- 计数校验：迁移前后总数必须一致
- 可追溯：新记录标记来源和迁移时间
- 不删旧表：迁移成功后旧表保留备查，不自动删除

---

## 模式组合原则

1. **原子性优先**：每个模式内的操作要么全成功要么全回滚
2. **模式可嵌套**：级联更新的内层可以包含条件路由
3. **失败隔离**：LOOP中单条失败不影响其他记录（除非设置strict模式）
4. **模式识别**：当用户描述匹配已有模式>70%时，推荐使用模板
