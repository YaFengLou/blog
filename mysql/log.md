在 MySQL 的 InnoDB 存储引擎中，**Redo Log**、**Binlog** 和 **Undo Log** 是事务执行过程中关键的三种日志。它们各自作用不同，**写入顺序严格遵循事务持久性和崩溃恢复的一致性设计**。



------



## **✅ 简要结论：写入顺序**



| **操作阶段** | **日志类型**       | **是否先写**     | **说明**         |
| ------------ | ------------------ | ---------------- | ---------------- |
| 1️⃣ 执行前     | Undo Log           | ✅ 最先写入       | 用于回滚与 MVCC  |
| 2️⃣ 执行后     | Redo Log (prepare) | ✅ 写入磁盘       | 保证崩溃后可恢复 |
| 3️⃣ 提交阶段   | Binlog             | ✅ 写入磁盘       | 主从复制/归档    |
| 4️⃣ 最终提交   | Redo Log (commit)  | ✅ 持久化提交标记 | 二阶段提交完成   |

------


## **🧠 详细执行流程分析（以一个典型事务为例）**



```
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;
```



### **1️⃣ 开始事务时：创建 Undo Log**

- 对数据进行变更前，InnoDB 会为被修改的数据生成一份 **Undo Log**（回滚日志）；

- Undo Log 用于：

    - 支持事务回滚；
    - 支持 MVCC：供其他事务生成一致性读视图时回滚得到历史版本。

### **2️⃣ 执行 SQL 时：**

- 更新数据页（Buffer Pool）中的数据；
- 记录 Undo Log（内存中或临时段）；
- 同时构造 Redo Log（内存中的 Redo Log Buffer）；

此时数据已经写到 **Buffer Pool**，但还未落盘。

### **3️⃣ COMMIT 前的准备：Redo Log prepare 阶段**

- 将 Redo Log 的 **prepare 阶段**写入磁盘（ib_logfile），但此时事务还未提交；
- 保证了 **崩溃恢复时可识别事务未完成**；
- Write-Ahead Logging (WAL) 原则：先写日志再写数据。

### **4️⃣ 写入 Binlog**

- Binlog 是 MySQL Server 层的日志，不属于 InnoDB；
- Binlog 记录的是 **逻辑操作语句或事件（ROW/STATEMENT/ MIXED 格式）**；
- 一定是在 Redo Log prepare 之后、事务真正 commit 之前写入。

### **5️⃣ Redo Log commit 阶段（完成二阶段提交）**

- 写入一条标记该事务成功提交的 **Redo Log commit 记录**；

- 表示事务完成提交，写入的数据才对其他事务可见；

- 如果此时系统崩溃，InnoDB 恢复时会判断：

    - 有 prepare、无 commit ⇒ 回滚；
    - 有 prepare、有 commit ⇒ redo；



## **🔄 二阶段提交流程（图示）**

```
事务执行流程：

1. 写入 Undo Log
2. 执行数据更新（Buffer Pool）
3. 写入 Redo Log Buffer（prepare）
4. 刷盘 Redo Log（prepare阶段）
5. 写入 Binlog
6. Redo Log commit
7. 返回 COMMIT 成功
```

------



## **❗ 为什么是这个顺序？**

因为要确保 **崩溃恢复一致性** 与 **主从复制一致性**：

- **保证 Binlog 和 Redo Log 的一致性（双写一致）**
- Binlog 写成功，Redo Log 没 commit ⇒ 崩溃后主从状态不一致；
- 所以：MySQL 使用 **协调的两阶段提交协议**（协调 Binlog + Redo）来保证：

> **“只要 Binlog 写入成功，则一定 redo commit；反之 rollback。”**



------



## **🔍 例外情况补充说明：**

### **❓ Binlog 和 Redo Log 的写入顺序是不是总一致？**

是的：在事务提交的过程中，**必须先写 binlog、后 commit redo**，否则崩溃可能导致复制错误。

### **❓ Undo Log 写入磁盘时机？**

- Undo Log 通常存储在 **回滚段（Rollback Segment）** 中；
- 它写入磁盘的时机**早于 Redo Log commit**；
- 且 Undo 也会有对应的 Redo Log 进行持久化保护。

------


## **✅ 总结图表**



| **日志类型** | **写入阶段**    | **用途**           | **是否刷盘**        |
| ------------ | --------------- | ------------------ | ------------------- |
| Undo Log     | 执行前          | 回滚 & MVCC        | ✅（依赖事务持久性） |
| Redo Log     | prepare 阶段    | 崩溃恢复           | ✅（立即刷盘）       |
| Binlog       | Redo prepare 后 | 主从复制、数据归档 | ✅（立即刷盘）       |
| Redo Log     | commit 阶段     | 标记事务提交完成   | ✅                   |



------



如你需要，我可以提供一段源码分析 InnoDB 两阶段提交流程（如 trx_commit → log_write_up_to）或画出详细执行流程图。是否继续？