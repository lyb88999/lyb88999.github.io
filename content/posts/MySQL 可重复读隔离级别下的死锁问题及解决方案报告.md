---
title: "MySQL可重复读隔离级别下的死锁问题及解决方案报告"
date: 2025-03-16
draft: false
tags: ["学习笔记", "MySQL"]
categories: ["学习"]
---

# MySQL 可重复读隔离级别下的死锁问题及解决方案报告



#### **问题背景**
在转账业务场景中，两个事务（A 和 B）按如下顺序操作，导致死锁：
1. **事务A** 向 `transfers` 表插入记录 `(from=1, to=2, amount=10)`。
2. **事务B** 向 `transfers` 表插入记录 `(from=1, to=2, amount=10)`。
3. **事务A** 使用 `SELECT ... FOR UPDATE` 查询 `accounts` 表 id=1 的余额。
4. **事务B** 使用 `SELECT ... FOR UPDATE` 查询 `accounts` 表 id=1 的余额。

**表结构**：
```sql
CREATE TABLE transfers (
    id BIGSERIAL PRIMARY KEY,
    from_account_id BIGINT NOT NULL,
    to_account_id BIGINT NOT NULL,
    amount BIGINT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE accounts (
    id BIGSERIAL PRIMARY KEY,
    balance BIGINT NOT NULL,
    -- 其他字段省略
);
```
- `transfers` 表的 `from_account_id` 和 `to_account_id` 是 `accounts` 表的外键。

---

### **死锁原因分析**
#### **核心问题**
- **外键检查的共享锁（S锁）**：  
  插入 `transfers` 记录时，MySQL 会检查外键引用的 `accounts` 行是否存在，并对这些行加 **S锁**。
- **显式排他锁（X锁）冲突**：  
  后续 `SELECT ... FOR UPDATE` 需要对同一行加 **X锁**，而 S锁 与 X锁 不兼容，导致事务互相等待。

#### **操作时序与锁竞争**
1. **事务A** 插入 `transfers` 记录：
   - 对 `accounts` 的 id=1 和 id=2 加 S锁。
2. **事务B** 插入 `transfers` 记录：
   - 对 `accounts` 的 id=1 和 id=2 加 S锁（S锁可共享，不冲突）。
3. **事务A** 执行 `SELECT ... FOR UPDATE`：
   - 尝试对 id=1 加 X锁，但事务B已持有 S锁，事务A被阻塞。
4. **事务B** 执行 `SELECT ... FOR UPDATE`：
   - 尝试对 id=1 加 X锁，但事务A仍持有 S锁，事务B也被阻塞。
5. **结果**：事务A和事务B形成循环等待，触发死锁。

---

### **解决方案对比**

| **方案**                   | **实现方式**                                                 | **是否解决死锁** | **优点**                                 | **缺点**                                 |
| -------------------------- | ------------------------------------------------------------ | ---------------- | ---------------------------------------- | ---------------------------------------- |
|                            |                                                              |                  |                                          |                                          |
| **去掉外键约束**           | 移除 `transfers` 表的外键约束，由应用层校验数据完整性。      | ✅ 是             | 消除外键锁冲突，提高并发性。             | 需应用层维护数据一致性，增加开发复杂度。 |
| **直接使用 `UPDATE` 语句** | 直接通过 `UPDATE accounts SET balance = balance ± 10` 修改余额，替代显式加锁。 | ✅ 是             | 原子操作，避免显式锁竞争，代码简洁高效。 | 需确保更新顺序一致，避免交叉加锁。       |

---

### **推荐方案：直接使用 `UPDATE` 语句**
#### **实现逻辑**
1. **直接更新余额**：通过原子操作完成转账，无需先查询余额。
   ```sql
   -- 事务操作示例
   BEGIN;
   -- 转出账户扣款
   UPDATE accounts SET balance = balance - 10 WHERE id = 1;
   -- 转入账户加款
   UPDATE accounts SET balance = balance + 10 WHERE id = 2;
   -- 插入转账记录
   INSERT INTO transfers (from_account_id, to_account_id, amount) VALUES (1, 2, 10);
   COMMIT;
   ```

#### **优势**
1. **避免显式锁竞争**：  
   `UPDATE` 语句隐式对目标行加 X锁，覆盖外键检查的 S锁需求，彻底消除锁冲突。
2. **原子性与简洁性**：  
   余额修改和流水插入在同一事务中完成，无需应用层计算，保证数据一致性。
3. **高性能**：  
   锁持有时间短，事务粒度小，并发性能高。

#### **注意事项**
1. **统一更新顺序**：  
   若需更新多行（如转出和转入账户），所有事务需按相同顺序加锁（如先 id=1 后 id=2），避免交叉等待。
2. **外键索引优化**：  
   确保 `transfers` 表的 `from_account_id` 和 `to_account_id` 有索引，加速外键检查。
3. **短事务原则**：  
   尽量缩短事务时间，减少锁持有时长。

---

### **验证示例**
#### **事务操作时序**
1. **事务A**：
   ```sql
   BEGIN;
   UPDATE accounts SET balance = balance - 10 WHERE id = 1; -- 对 id=1 加 X锁
   -- 事务B的 UPDATE 操作在此处被阻塞
   INSERT INTO transfers (from_account_id, to_account_id, amount) VALUES (1, 2, 10);
   COMMIT; -- 释放 X锁，事务B继续执行
   ```

2. **事务B**：
   ```sql
   BEGIN;
   UPDATE accounts SET balance = balance - 10 WHERE id = 1; -- 等待事务A释放 X锁
   -- 事务A提交后，事务B获取 X锁并继续
   INSERT INTO transfers (from_account_id, to_account_id, amount) VALUES (1, 2, 10);
   COMMIT;
   ```

#### **结果**
- 事务A和事务B顺序执行，无死锁。
- 若事务B更新其他账户（如 id=3），可完全并行执行。

---

### **总结**
1. **死锁根源**：外键检查的 S锁 与显式 X锁 冲突。
2. **最佳实践**：  
   - **优先使用 `UPDATE` 语句**：通过原子操作替代显式锁，从根本上消除死锁条件。  
   - **统一加锁顺序**：若需保留复杂逻辑，强制按固定顺序加锁（如按账户ID排序）。  
   - **权衡外键约束**：在高并发场景，可移除外键约束，由应用层保证数据完整性。  
