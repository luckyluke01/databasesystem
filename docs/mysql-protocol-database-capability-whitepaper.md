# MySQL 协议数据库服务对外能力清单与白皮书草案

本文档用于梳理一个“对外按 MySQL 协议提供服务”的数据库系统应公开声明、内部评估和测试验证的能力范围。目标是在需求评审、产品口径、研发规划、测试准入和客户接入前，将能力边界、性能边界、资源边界和兼容性边界明确下来，降低上线后才暴露 CPU 吃紧、内存占用过高、语义不兼容或底层无法支撑上层承诺的风险。

本文档是一份“大而全”的基础清单。实际落地时，应结合项目真实架构、底层引擎能力、客户场景和研发规划，对每一项标注“支持 / 部分支持 / 有限制支持 / 实验性支持 / 规划中 / 不支持 / 废弃中”。

## 1. 产品定位与边界

### 1.1 产品定位

需要先明确系统本质，避免外部天然按完整 MySQL 预期使用：

- 完整自研数据库
- MySQL 兼容数据库
- MySQL 协议网关
- 查询加速服务
- 分析型数据库
- 事务型数据库
- 多租户数据服务
- 内部数据中台的 MySQL 协议访问层
- 跨源或联邦查询服务

### 1.2 核心适用场景

- 在线查询 OLTP
- 离线分析 OLAP
- 混合负载 HTAP
- 报表查询
- 数据服务 API 化
- 轻量数据写入
- 大批量导入
- 多租户隔离
- 跨源查询
- 数据联邦查询
- BI 工具接入
- 只读查询服务
- 读写一体服务

### 1.3 明确不适用场景

建议在白皮书中清晰声明，例如：

- 不适合高频小事务写入
- 不适合强一致高并发转账类业务
- 不适合超大结果集无分页导出
- 不适合复杂存储过程迁移
- 不适合依赖 MySQL 全量语法兼容的业务
- 不适合无索引的大表高频扫描
- 不适合长事务
- 不适合把服务当作原生 MySQL 的无差别替代品直接迁移

## 2. 对外能力矩阵模板

建议所有能力使用统一格式维护：

| 能力项 | 状态 | 兼容范围 | 限制说明 | 性能影响 | 测试锚点 | 备注 |
| --- | --- | --- | --- | --- | --- | --- |
| SELECT 查询 | 支持 | MySQL 5.7/8.0 部分语法 | 不支持部分函数 | 中 | SQL 兼容测试 |  |
| 事务 | 有限制支持 | BEGIN/COMMIT/ROLLBACK | 不保证完整隔离级别 | 高 | 并发一致性测试 |  |
| JOIN | 支持 | INNER/LEFT/RIGHT | 大表 JOIN 需限制 | 高 | 执行计划测试 |  |
| 存储过程 | 不支持 | - | 客户端需改造 | - | 兼容性测试 |  |

状态建议统一为：

- 支持
- 部分支持
- 有限制支持
- 实验性支持
- 规划中
- 不支持
- 废弃中

## 3. MySQL 协议兼容能力

这是最容易和客户产生误解的部分，应单独成章说明“兼容 MySQL 协议”不等于“完整兼容 MySQL 数据库”。

### 3.1 连接协议

- MySQL Handshake 支持
- 协议版本声明
- 客户端认证流程
- SSL/TLS 连接
- 明文密码认证
- Native Password
- Caching SHA2 Password
- 连接属性 capabilities
- 客户端字符集协商
- 连接超时
- 空闲连接超时
- 最大连接数
- 单用户最大连接数
- 单租户最大连接数
- 连接池兼容性

### 3.2 客户端兼容

需明确测试过哪些客户端和版本：

- mysql CLI
- MySQL Workbench
- Navicat
- DBeaver
- DataGrip
- JDBC Driver
- Go MySQL Driver
- Python PyMySQL
- Python mysqlclient
- Node.js mysql2
- PHP PDO MySQL
- BI 工具：Tableau、Power BI、FineBI、Superset、Metabase
- ORM：MyBatis、Hibernate、SQLAlchemy、GORM、Prisma

### 3.3 协议命令支持

- COM_QUERY
- COM_INIT_DB
- COM_FIELD_LIST
- COM_PING
- COM_QUIT
- COM_STMT_PREPARE
- COM_STMT_EXECUTE
- COM_STMT_CLOSE
- COM_STMT_RESET
- COM_STMT_SEND_LONG_DATA
- COM_SET_OPTION
- COM_CHANGE_USER
- COM_RESET_CONNECTION

### 3.4 Prepared Statement

- 文本协议查询
- 二进制协议查询
- 服务端 Prepared Statement
- 客户端 Prepared Statement
- 参数绑定
- 类型推断
- NULL 参数
- 批量执行
- Statement 缓存
- 大参数传输
- Prepared Statement 生命周期限制

### 3.5 返回结果协议

- Result Set Metadata
- Column Definition
- OK Packet
- ERR Packet
- EOF Packet
- Warning Count
- Affected Rows
- Last Insert ID
- Server Status Flags
- 多结果集
- 流式返回
- 大结果集返回限制

## 4. SQL 语法能力

### 4.1 查询语句

- SELECT
- SELECT DISTINCT
- WHERE
- GROUP BY
- HAVING
- ORDER BY
- LIMIT
- OFFSET
- UNION
- UNION ALL
- INTERSECT
- EXCEPT
- 子查询
- 相关子查询
- EXISTS
- IN / NOT IN
- ANY / ALL
- CASE WHEN
- CTE / WITH
- 递归 CTE
- 窗口函数
- 聚合函数
- 标量函数
- 表函数

### 4.2 JOIN 能力

- INNER JOIN
- LEFT JOIN
- RIGHT JOIN
- FULL OUTER JOIN
- CROSS JOIN
- SEMI JOIN
- ANTI JOIN
- NATURAL JOIN
- USING
- ON 多条件
- 多表 JOIN
- 大表 JOIN 限制
- 广播 JOIN
- Shuffle JOIN
- Hash JOIN
- Nested Loop JOIN
- Merge JOIN
- JOIN Reorder

### 4.3 DML

- INSERT
- INSERT VALUES
- INSERT SELECT
- 批量 INSERT
- UPDATE
- DELETE
- REPLACE
- UPSERT
- INSERT ON DUPLICATE KEY UPDATE
- LOAD DATA
- TRUNCATE
- MERGE

### 4.4 DDL

- CREATE DATABASE
- DROP DATABASE
- CREATE TABLE
- DROP TABLE
- ALTER TABLE
- RENAME TABLE
- CREATE VIEW
- DROP VIEW
- CREATE INDEX
- DROP INDEX
- CREATE USER
- GRANT
- REVOKE
- ANALYZE TABLE
- OPTIMIZE TABLE
- SHOW CREATE TABLE

### 4.5 MySQL 方言兼容

- 反引号标识符
- `LIMIT offset, count`
- `LIMIT count OFFSET offset`
- `IFNULL`
- `IF`
- `DATE_FORMAT`
- `STR_TO_DATE`
- `GROUP_CONCAT`
- `REGEXP`
- `JSON_EXTRACT`
- `CAST`
- `CONVERT`
- `AUTO_INCREMENT`
- `ON DUPLICATE KEY`
- `SHOW` 系列语句
- `EXPLAIN`
- `DESCRIBE`
- `USE database`

## 5. 数据类型能力

### 5.1 数值类型

- TINYINT
- SMALLINT
- MEDIUMINT
- INT
- BIGINT
- FLOAT
- DOUBLE
- DECIMAL
- UNSIGNED
- BOOL / BOOLEAN

### 5.2 字符串类型

- CHAR
- VARCHAR
- TEXT
- TINYTEXT
- MEDIUMTEXT
- LONGTEXT
- BINARY
- VARBINARY
- BLOB
- ENUM
- SET

### 5.3 日期时间类型

- DATE
- TIME
- DATETIME
- TIMESTAMP
- YEAR
- 时区处理
- 零日期兼容
- 精度支持
- 日期函数兼容

### 5.4 JSON 与半结构化数据

- JSON 类型
- JSON Path
- JSON_EXTRACT
- JSON_SET
- JSON_CONTAINS
- JSON_ARRAY
- JSON_OBJECT
- JSON 索引
- 嵌套字段查询

### 5.5 类型转换

- 显式 CAST
- 隐式类型转换
- 字符串到数字
- 数字到字符串
- 日期到字符串
- NULL 参与计算
- 溢出行为
- 精度截断行为

## 6. 事务与一致性能力

### 6.1 事务语法

- BEGIN
- START TRANSACTION
- COMMIT
- ROLLBACK
- SAVEPOINT
- ROLLBACK TO SAVEPOINT
- SET autocommit
- 隐式提交
- DDL 是否隐式提交

### 6.2 隔离级别

- READ UNCOMMITTED
- READ COMMITTED
- REPEATABLE READ
- SERIALIZABLE
- 默认隔离级别
- 是否支持 MVCC
- 是否支持快照读
- 是否支持当前读

### 6.3 一致性模型

- 强一致
- 最终一致
- 读己之写
- 单会话一致性
- 跨副本一致性
- 跨分片一致性
- 分布式事务
- 两阶段提交
- 幂等写入
- 冲突检测

### 6.4 锁能力

- 行锁
- 表锁
- 元数据锁
- 间隙锁
- 意向锁
- 悲观锁
- 乐观锁
- 死锁检测
- 锁等待超时
- `SELECT FOR UPDATE`
- `SELECT LOCK IN SHARE MODE`

## 7. 存储与表能力

### 7.1 表模型

- 普通表
- 分区表
- 临时表
- 外部表
- 视图
- 物化视图
- 系统表
- 内存表
- 只读表
- 宽表
- 明细表
- 聚合表

### 7.2 主键与约束

- PRIMARY KEY
- UNIQUE KEY
- FOREIGN KEY
- NOT NULL
- DEFAULT
- CHECK
- AUTO_INCREMENT
- 复合主键
- 唯一约束
- 外键约束实际执行情况

### 7.3 分区能力

- Range 分区
- List 分区
- Hash 分区
- Key 分区
- 时间分区
- 分区裁剪
- 自动分区
- 分区上限
- 单分区数据量建议
- 分区变更成本

### 7.4 数据生命周期

- TTL
- 冷热分层
- 归档
- 过期清理
- 数据压缩
- 数据清理任务
- 在线删除
- 批量删除限制

## 8. 索引能力

### 8.1 索引类型

- B-Tree 索引
- Hash 索引
- Bitmap 索引
- 倒排索引
- 全文索引
- 空间索引
- 向量索引
- 前缀索引
- 复合索引
- 函数索引
- 表达式索引
- 局部索引
- 全局索引

### 8.2 索引行为

- 索引选择
- 覆盖索引
- 索引下推
- 多索引合并
- 索引统计信息
- 索引失效场景
- 模糊查询索引使用
- 范围查询索引使用
- 排序是否利用索引
- JOIN 是否利用索引

### 8.3 索引维护

- 在线建索引
- 异步建索引
- 索引重建
- 索引删除
- 索引状态查看
- 索引构建资源限制
- 大表索引构建风险

## 9. 查询优化与执行能力

### 9.1 优化器能力

- 基于规则优化 RBO
- 基于代价优化 CBO
- 谓词下推
- 投影下推
- LIMIT 下推
- 聚合下推
- JOIN Reorder
- 子查询改写
- 常量折叠
- 分区裁剪
- 统计信息采集
- 直方图
- Cardinality 估算
- 执行计划缓存

### 9.2 执行引擎能力

- 向量化执行
- 火山模型执行
- Pipeline 执行
- 并行执行
- 分布式执行
- 批处理执行
- 流式执行
- Runtime Filter
- Spill to Disk
- 代码生成
- 自适应执行

### 9.3 EXPLAIN 能力

- EXPLAIN
- EXPLAIN ANALYZE
- 执行计划展示
- 估算行数
- 实际行数
- 执行耗时
- 扫描数据量
- 内存估算
- 是否命中索引
- 是否发生 Shuffle
- 是否发生 Spill

## 10. 资源治理能力

已经出现 CPU、内存问题的系统，应将资源治理作为白皮书和内部评审重点。

### 10.1 连接级限制

- 最大连接数
- 单用户最大连接数
- 单租户最大连接数
- 空闲连接回收
- 连接建立速率限制
- 连接排队
- 连接拒绝策略

### 10.2 查询级限制

- 单查询最大执行时间
- 单查询最大扫描行数
- 单查询最大扫描字节数
- 单查询最大返回行数
- 单查询最大返回字节数
- 单查询最大内存
- 单查询最大 CPU 使用
- 单查询最大并发度
- 单查询最大临时文件大小
- 查询超时取消
- 客户端断开后的查询取消

### 10.3 租户级限制

- 租户 CPU 配额
- 租户内存配额
- 租户并发查询数
- 租户 QPS 限制
- 租户写入速率限制
- 租户存储容量限制
- 租户慢查询隔离
- 租户优先级
- 租户资源组

### 10.4 资源隔离

- CPU 隔离
- 内存隔离
- IO 隔离
- 网络隔离
- 查询队列隔离
- 读写隔离
- 大小查询隔离
- 管理任务与用户任务隔离

### 10.5 过载保护

- 熔断
- 降级
- 查询排队
- 快速失败
- Kill Query
- 自动取消大查询
- 慢查询限流
- 内存水位保护
- CPU 水位保护
- 连接风暴保护

## 11. 性能指标与容量边界

建议明确区分“官方推荐边界”和“硬限制”。

### 11.1 数据规模

- 单表最大行数
- 单表最大容量
- 单库最大表数
- 单实例最大库数
- 单租户最大数据量
- 单分区最大数据量
- 单行最大大小
- 单字段最大大小
- 单索引最大字段数
- 单 SQL 最大长度

### 11.2 查询性能

- 点查延迟
- 小范围查询延迟
- 聚合查询延迟
- JOIN 查询延迟
- 排序查询延迟
- 分页查询延迟
- 大结果集导出性能
- 并发查询能力
- QPS
- TPS
- P95 / P99 延迟

### 11.3 写入性能

- 单行写入
- 批量写入
- 并发写入
- 导入吞吐
- 更新吞吐
- 删除吞吐
- 写入延迟
- 写放大
- Compaction 影响

### 11.4 资源消耗模型

- 每连接基础内存
- 每查询基础内存
- 排序内存估算
- JOIN 内存估算
- 聚合内存估算
- 返回结果内存估算
- Prepared Statement 内存占用
- 元数据缓存占用
- 统计信息缓存占用

## 12. 高可用与容灾能力

### 12.1 高可用

- 主备架构
- 多副本
- 自动故障转移
- 手动故障转移
- 读写分离
- 副本延迟
- Leader 选举
- 节点健康检查
- 故障检测时间
- 故障恢复时间

### 12.2 容灾指标

- RTO
- RPO
- 单 AZ 容灾
- 多 AZ 容灾
- 跨地域容灾
- 数据复制延迟
- 灾备切换
- 回切能力
- 容灾演练

### 12.3 数据可靠性

- WAL / Redo Log
- Binlog
- Checkpoint
- 数据校验
- 副本校验
- 校验和
- 坏块检测
- 自动修复
- 数据丢失场景声明

## 13. 备份恢复能力

### 13.1 备份

- 全量备份
- 增量备份
- 逻辑备份
- 物理备份
- 快照备份
- 自动备份
- 手动备份
- 跨地域备份
- 备份加密
- 备份压缩
- 备份保留周期

### 13.2 恢复

- 全量恢复
- 按时间点恢复 PITR
- 单库恢复
- 单表恢复
- 跨实例恢复
- 恢复到新实例
- 原地恢复
- 恢复进度查看
- 恢复期间服务可用性
- 恢复一致性校验

## 14. 安全能力

### 14.1 认证

- 用户名密码
- Token
- IAM 集成
- LDAP
- Kerberos
- OAuth
- mTLS
- 临时凭证
- 密码复杂度
- 密码轮换
- 登录失败锁定

### 14.2 授权

- 用户权限
- 角色权限
- 库级权限
- 表级权限
- 列级权限
- 行级权限
- 视图权限
- 函数权限
- 管理权限
- 最小权限原则

### 14.3 数据安全

- 传输加密
- 静态加密
- 字段加密
- 密钥管理
- 脱敏
- 动态脱敏
- 数据水印
- 数据导出控制
- 敏感字段访问审计

### 14.4 审计

- 登录审计
- 查询审计
- DDL 审计
- DML 审计
- 权限变更审计
- 管理操作审计
- 慢查询审计
- 失败请求审计
- 审计日志保留周期

## 15. 可观测性能力

### 15.1 指标 Metrics

- QPS
- TPS
- 连接数
- 活跃连接数
- 查询并发数
- 查询延迟
- P95 / P99
- CPU 使用率
- 内存使用率
- 磁盘使用率
- 网络 IO
- 扫描行数
- 扫描字节数
- 返回行数
- 错误率
- 慢查询数
- Kill 查询数
- 排队查询数
- Spill 次数
- GC 指标

### 15.2 日志 Logs

- 查询日志
- 慢查询日志
- 错误日志
- 审计日志
- 访问日志
- 资源超限日志
- 后台任务日志
- 节点日志
- 优化器日志
- 执行计划日志

### 15.3 链路追踪 Tracing

- 请求 Trace ID
- SQL 生命周期追踪
- 协议层耗时
- 解析耗时
- 优化耗时
- 调度耗时
- 执行耗时
- 返回耗时
- 下游访问耗时
- 跨节点追踪

### 15.4 运维诊断

- SHOW PROCESSLIST
- KILL QUERY
- KILL CONNECTION
- SHOW STATUS
- SHOW VARIABLES
- SHOW WARNINGS
- 慢查询分析
- Top SQL
- Top 用户
- Top 租户
- Top 表
- 执行计划历史
- 查询画像
- 资源画像

## 16. 运维管理能力

### 16.1 实例管理

- 创建实例
- 删除实例
- 启停实例
- 重启实例
- 扩容
- 缩容
- 升级
- 降级
- 配置变更
- 参数管理
- 版本管理

### 16.2 变更能力

- 在线 DDL
- 滚动升级
- 灰度发布
- 配置热更新
- 参数动态生效
- 变更回滚
- 变更审计
- 变更前检查
- 变更影响评估

### 16.3 任务管理

- 后台任务列表
- 任务进度
- 任务取消
- 任务重试
- 任务失败原因
- 任务并发限制
- 导入任务
- 导出任务
- 建索引任务
- 备份任务
- 恢复任务

## 17. 兼容性与迁移能力

### 17.1 MySQL 迁移

- Schema 迁移
- 数据迁移
- 全量迁移
- 增量迁移
- Binlog 同步
- 双写
- 校验
- 回滚
- 迁移限速
- 迁移兼容性报告

### 17.2 生态兼容

- JDBC
- ODBC
- SQLAlchemy
- MyBatis
- Hibernate
- Spark
- Flink
- Kafka Connect
- Debezium
- Airflow
- dbt
- Superset
- Tableau
- Power BI

### 17.3 兼容性差异说明

建议专门维护：

- MySQL 5.7 差异
- MySQL 8.0 差异
- SQL Mode 差异
- 函数差异
- 数据类型差异
- 事务差异
- 锁行为差异
- 错误码差异
- 字符集差异
- 排序规则差异

## 18. 错误码与异常行为

### 18.1 错误码兼容

- MySQL 标准错误码
- 自定义错误码
- SQLSTATE
- 错误消息格式
- 客户端可识别性
- 错误是否可重试
- 错误分类

### 18.2 异常场景

- 语法错误
- 权限错误
- 连接失败
- 查询超时
- 内存超限
- CPU 超限
- 结果集超限
- 锁等待超时
- 死锁
- 节点故障
- 副本不可用
- 元数据不可用
- 下游存储不可用
- 网络中断

### 18.3 重试语义

- 哪些错误可重试
- 哪些错误不可重试
- 是否幂等
- 客户端重试建议
- 服务端自动重试
- 查询取消后的状态

## 19. 多租户能力

### 19.1 租户模型

- 单租户实例
- 多租户共享实例
- 逻辑租户隔离
- 物理租户隔离
- 租户命名空间
- 租户级元数据
- 租户级权限

### 19.2 租户隔离

- 数据隔离
- 权限隔离
- 资源隔离
- 日志隔离
- 审计隔离
- 备份隔离
- 配置隔离
- 故障隔离

### 19.3 租户治理

- 租户配额
- 租户限流
- 租户账单
- 租户监控
- 租户告警
- 租户级 SLA
- 租户级变更窗口

## 20. 数据导入导出能力

### 20.1 导入

- INSERT 批量导入
- LOAD DATA
- CSV 导入
- Parquet 导入
- JSON 导入
- ORC 导入
- 对象存储导入
- Kafka 导入
- MySQL Binlog 导入
- 导入限速
- 导入失败重试
- 导入错误行处理

### 20.2 导出

- SELECT 导出
- CSV 导出
- Parquet 导出
- JSON 导出
- 对象存储导出
- 分片导出
- 异步导出
- 导出限速
- 最大导出数据量
- 导出权限控制

## 21. 测试能力锚点

### 21.1 协议兼容测试

- 各语言客户端连接测试
- 鉴权测试
- SSL 测试
- Prepared Statement 测试
- 大结果集测试
- 连接池测试
- 异常断连测试

### 21.2 SQL 兼容测试

- MySQL 官方语法子集
- 常用函数
- 数据类型
- JOIN
- 子查询
- 聚合
- 窗口函数
- DDL
- DML
- 错误码

### 21.3 性能测试

- 单查询性能
- 并发查询性能
- 大表扫描
- 大表 JOIN
- 大排序
- 大聚合
- 高 QPS
- 高连接数
- 长时间稳定性
- 资源水位测试

### 21.4 稳定性测试

- 长稳压测
- 节点重启
- 网络抖动
- 下游故障
- 磁盘满
- 内存不足
- CPU 打满
- 元数据故障
- 查询取消
- 连接风暴

### 21.5 安全测试

- 权限绕过
- SQL 注入
- 越权访问
- TLS 验证
- 密码策略
- 审计完整性
- 敏感信息泄露
- 多租户隔离

### 21.6 兼容性回归测试

- 客户端版本矩阵
- MySQL 5.7 行为对比
- MySQL 8.0 行为对比
- ORM 框架回归
- BI 工具回归
- 历史 SQL 样本回归
- 线上慢 SQL 回放
- 线上 Top SQL 回放

## 22. 建议重点输出的边界声明

### 22.1 语法边界

- 支持哪些 SQL
- 不支持哪些 SQL
- 哪些 SQL 是实验性支持
- 哪些 SQL 有规模限制

### 22.2 性能边界

- 推荐单查询扫描量
- 推荐单查询返回行数
- 推荐并发数
- 推荐连接数
- 推荐单表规模
- 不推荐查询模式

### 22.3 一致性边界

- 是否支持事务
- 支持到什么隔离级别
- 是否保证读己之写
- 是否可能读到延迟数据
- 故障切换期间语义

### 22.4 资源边界

- 查询超时时间
- 内存上限
- CPU 限制
- 返回结果限制
- 导入导出限制
- 大查询处理策略

### 22.5 兼容性边界

- 兼容 MySQL 协议，不等于兼容完整 MySQL
- 兼容常用客户端，不等于兼容所有 ORM 行为
- 兼容 SQL 子集，不等于支持所有 MySQL 方言
- 错误码尽量兼容，但可能存在扩展错误码

## 23. 需求确认表

后续接需求时，建议要求业务方填写：

| 问题 | 业务方填写 |
| --- | --- |
| 使用场景是 OLTP、OLAP、报表、导出还是混合？ |  |
| 是否需要写入？写入 QPS 多少？ |  |
| 是否需要事务？需要什么隔离级别？ |  |
| 单表预计数据量？ |  |
| 单次查询预计扫描多少数据？ |  |
| 单次查询预计返回多少数据？ |  |
| 并发连接数多少？ |  |
| 并发查询数多少？ |  |
| P95/P99 延迟要求？ |  |
| 是否有大 JOIN、大排序、大聚合？ |  |
| 是否依赖存储过程、触发器、外键？ |  |
| 使用哪些客户端、ORM、BI 工具？ |  |
| 是否需要跨地域容灾？ |  |
| 是否有合规、审计、加密要求？ |  |
| 是否需要导入导出？规模多大？ |  |
| 是否需要多租户隔离？ |  |
| 是否有历史 SQL 样本？ |  |

## 24. 建议内部维护的配套文档

### 24.1 对外白皮书

面向客户、产品、售前、业务方，说明：

- 支持什么
- 不支持什么
- 如何正确使用
- 边界在哪里
- 推荐实践是什么

### 24.2 内部能力矩阵

面向研发、测试、运维，维护：

- 每个能力的真实状态
- 技术负责人
- 测试覆盖
- 已知问题
- 性能边界
- 规划版本

### 24.3 需求准入 Checklist

面向需求评审，确认：

- 新需求是否超出能力边界
- 是否会造成 CPU/内存风险
- 是否需要新增测试锚点
- 是否需要产品口径调整
- 是否需要明确对外限制

## 25. 建议优先级

如果系统已经遇到 CPU、内存问题，第一批建议先梳理：

1. MySQL 协议兼容范围
2. SQL 语法支持范围
3. 查询资源限制
4. 大查询治理策略
5. 连接数与并发限制
6. 单查询返回结果限制
7. JOIN、排序、聚合的规模边界
8. 慢查询与 Top SQL 观测能力
9. 客户端兼容矩阵
10. 测试回归 SQL 集合

这些内容能直接降低“需求上线后才暴露底层撑不住”的风险，并为研发迭代、产品承诺和测试验收提供共同锚点。
