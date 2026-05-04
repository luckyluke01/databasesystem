# MySQL 协议数据库服务对外能力清单与白皮书草案

本文档用于梳理一个“对外按 MySQL 协议提供服务”的数据库系统应公开声明、内部评估和测试验证的能力范围。目标是在需求评审、产品口径、研发规划、测试准入和客户接入前，将能力边界、性能边界、资源边界和兼容性边界明确下来，降低上线后才暴露 CPU 吃紧、内存占用过高、语义不兼容或底层无法支撑上层承诺的风险。

本文档是一份“大而全”的基础清单。实际落地时，应结合项目真实架构、底层引擎能力、客户场景和研发规划，对每一项标注“支持 / 部分支持 / 有限制支持 / 实验性支持 / 规划中 / 不支持 / 废弃中”。

当前版本已结合《Query 查询应用》使用手册补充系统现状映射。手册中的能力、限制、废弃项和迁移策略应作为后续对外口径、内部测试和研发排期的优先参考来源；其中存在口径不一致的地方，本文会单独标注为“需确认”。

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

### 1.4 当前 Query 系统手册映射

根据《Query 查询应用》手册，当前系统更接近“统一查询服务 + MySQL 协议接入层 + 多源查询适配层”，而不是完整通用 MySQL 数据库。对外口径建议表述为：

- 对业务侧提供统一查询端点，屏蔽底层指标、记录、会话、实体、配置库和第三方数据源的差异。
- 支持通过 MySQL 驱动接入 BQL(SQL) 查询，覆盖 Java、Python、C++ 等业务侧驱动场景。
- 支持 PQL 查询，并对外提供类 Prometheus 查询接口。
- 支持账号鉴权、TLS 双向证书认证、TCP 通信和本地预编译执行。
- 支持模型数据查询，包括指标、记录、会话、配置联查。
- 支持实例数据查询、实例关系联查和实体属性查询。
- 支持第三方数据源接入，当前已支持 Nebula；ELK、日志易为规划方向。
- 支持配置库查询适配，目标是让业务侧对底层数据库替换无感。
- 查询优化、慢查询优化、分库分表、指标计算类型切换等底层动作原则上由 Query 侧消化，业务侧不主动干预。

当前手册中明确的未具备或需谨慎声明能力：

- 元数据查询：手册总览中写明“未具备元数据查询支持”，但后续章节又写明“支持 br_one 和 meta 的联查”。该口径存在冲突，应在正式白皮书中拆分为“系统元数据查询”“业务元数据表查询”“配置库/模型元数据联查”三个能力项后重新确认。
- 日志、调用链、事件等：手册总览中列为未具备能力，但后续已有事件查询、日志详情查询、日志 live tail 查询说明。建议正式口径改为“分场景有限支持”，并分别声明限制。
- 全库数据联查：已废弃，不应继续对外承诺。
- br_one 库、meta 库全部表联查：已废弃，不应继续默认开放。
- entity 库按注册关系联查、metric 通过 config 访问注册库联查：属于旧能力或收敛中能力，应按需开放并纳入准入审批。
- 逐步迁移策略：业务侧先切换查询写法，底层存储暂不切表；业务侧全部切换后，再按指标批量切换存储。

## 2. 对外能力矩阵模板

建议所有能力使用统一格式维护：

| 能力项 | 状态 | 兼容范围 | 限制说明 | 性能影响 | 测试锚点 | 备注 |
| --- | --- | --- | --- | --- | --- | --- |
| SELECT 查询 | 支持 | MySQL 5.7/8.0 部分语法 | 不支持部分函数 | 中 | SQL 兼容测试 |  |
| 事务 | 有限制支持 | BEGIN/COMMIT/ROLLBACK | 不保证完整隔离级别 | 高 | 并发一致性测试 |  |
| JOIN | 支持 | INNER/LEFT/RIGHT | 大表 JOIN 需限制 | 高 | 执行计划测试 |  |
| 存储过程 | 不支持 | - | 客户端需改造 | - | 兼容性测试 |  |
| PQL 查询 | 支持 | Prometheus query/query_range/labels/series 子集 | V2/V3/V4 暂不支持，V1/V5 支持 | 中 | PQL 兼容测试 | Query 手册现状 |
| BQL(SQL) 查询 | 部分支持 | MySQL 驱动接入，通用 SQL 语义子集 | 嵌套、子查询、别名、权限条件存在限制 | 高 | BQL 语法回归 | Query 手册现状 |
| 模型数据查询 | 支持 | 指标、记录、会话、配置联查 | 必填账号、环境、资源域、时间等条件 | 高 | 模型查询样例回放 | Query 手册现状 |
| 实体关系查询 | 有限制支持 | entity 与 Nebula 关系边表 | 关系查询需精准条件，默认 1w 限制 | 高 | Nebula 关系联查压测 | Query 手册现状 |
| 日志查询 | 有限制支持 | logdetails/loglive | 不支持 join、子查询、取样、第三方日志 | 高 | 日志查询回归 | Query 手册现状 |

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

当前 Query 手册映射：

- BQL(SQL) 查询通过 MySQL 驱动接入，手册明确覆盖 Java、Python、C++ 等客户端语言。
- 当前能力包括账号鉴权、TLS、本地预编译执行。
- 是否支持完整服务端 Prepared Statement、二进制协议参数绑定、批量执行、Statement 缓存，需要在能力矩阵中拆项确认，避免业务侧将“本地预编译执行”理解为完整 MySQL Prepared Statement 兼容。
- 业务接入时需要在 SQL 头部通过 `/** ... */` JSON 注释传递扩展信息，例如 `one-traceId`、`business`，用于溯源与排障。

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

### 4.6 当前 Query BQL(SQL) 支持与限制

根据《Query 查询应用》手册，当前 BQL(SQL) 是面向模型化查询的 SQL 子集，不应按完整 MySQL SQL 能力承诺。

当前已明确的支持能力：

- 使用 MySQL 驱动发起查询。
- 支持通用 SQL 语义下的过滤、分组、聚合、排序、分页。
- 支持指标、记录、会话、事件、日志、实体、配置等不同查询域。
- 支持实体关键字：`entity.modelKey[dimessionId/attribute].attribute`。
- 支持配置关键字：`config.tableName[dimessionId/attribute,other condition].column`。
- 支持标签关键字：`tags.tags`、`tags.dimessionId`、`tags.tag`。
- 支持 `union all` 或 `union distinct` 用于记录查询场景。
- 支持特定子查询场景：各子查询分组维度一致时可查询，实现效果等同于 FULL JOIN。
- 支持开窗类聚合写法：在原聚合函数基础上增加 `cycle` 参数，例如 `sum(metricId, cycle)`、`avg(metricId, cycle)`、`quantile(metricId, level, cycle)`。

当前已明确的限制：

- 手册早期描述为“不支持嵌套语句查询”，后续又描述“子查询支持”；正式白皮书应声明为“仅支持受限子查询”，并按场景列出准入条件。
- 权限条件 `accountId`、`userId`、`envId`、`resourceZoneId` 原则上必须出现在 SQL 最外层。
- 权限相关条件禁止增加表别名。
- `where` 条件暂不支持使用列别名过滤，必须使用原始列名。
- 实体实例数据查询中，所有表必须携带表别名，列必须使用 `表别名.列名`。
- 关系数据查询涉及 Nebula 远端调用，必须尽量使用子查询并命中精准数据。
- 多指标列式查询要求所有指标维度一致，即 `dimensionId + dataType` 一致；维度不一致时应使用受限子查询模式。
- 当一个指标多个维度指向同一实体时，必须显式指定维度 ID，否则应报错。
- 存在开窗查询列时，`select` 列中必须包含对 `monitorTime` 使用 `toStartOfInterval` 的时间对齐函数。
- 多个开窗查询列要求窗口大小一致。
- 使用 `quantile` 分位数函数做开窗查询时，必须指定分位值参数。
- 如果同一个指标同时查询开窗和不开窗结果，必须指定别名。
- 事件表名 `event` 是关键字，查询时需要使用反引号，例如 `data.\`event\``。
- `brEventRelationEntityKey` 只支持在 `select`、`group by`、`having` 中使用，不可用于 `where`。

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

### 7.5 当前 Query 查询域与表域映射

结合《Query 查询应用》手册，当前系统应将“库表能力”从通用数据库表模型拆分为业务查询域，并分别声明支持范围、必填条件和限制。

| 查询域 | 典型库表 | 当前能力 | 必填或关键条件 | 主要限制 |
| --- | --- | --- | --- | --- |
| 指标数据 | `data.metrics` | 支持指标聚合、维度分组、实体属性关联、标签查询、粒度查询 | `accountId`、`userId`、`envId`、`resourceZoneId`、`monitorTime` 时间范围 | 多指标列式查询要求维度一致；单指标星型模型明细查询暂不支持 |
| 记录数据 | `data.records` | 支持记录属性与实体属性查询 | 通用权限条件、`monitorTime`、`recordKey` | `recordKey` 为必选参数；支持 `union all` 或 `union distinct` 的场景需单独回归 |
| 会话数据 | `data.session` | 支持会话字段计算与过滤 | 通用权限条件、时间条件 | 需补充会话字段、聚合和排序的正式支持矩阵 |
| 事件数据 | `data.\`event\`` | 支持事件属性、扩展属性、附加信息、关联实体查询 | `accountId`、`userId`、`envId`、`resourceZoneId`、`monitorTime` | `event` 需反引号；`brEventRelationEntityKey` 不可用于 `where` |
| 日志详情 | `data.logdetails` | 有限制支持日志详情、日志属性、关联实体、实体属性查询 | `accountId`、`userId`、`envId`、`resourceZoneId`、`monitorTimeMs`、`indexId` | 不支持 join、子查询、取样查询、第三方日志 |
| 日志 live tail | `data.loglive` | 有限制支持实时日志查询 | `accountId`、`userId`、`envId`、`resourceZoneId`、`receiveTimeMs` 时间范围 | 字段名依赖日志属性元数据；需限制返回规模 |
| 实体实例 | `entity.modelKey` | 支持实体实例查询、实体属性查询、实体标签查询 | `accountId`、`userId`、`envId`、`resourceZoneId`；高基数实例还需 `metricId`、`monitorTime`、`aggFunction`、`dimensionId` 等 | 表别名必填；列必须带表别名；高基数查询需避免全量扫描 |
| 实体关系 | `entity.entity_relationship_edge`、`entity.entity_relationship_vertex` | 支持基于 Nebula 的实体点边查询和实例联查 | 精准关系条件，例如 `go_from()` | 远端 Nebula 调用默认 1w 条限制，可通过配置调整；与实例联查时关系边建议放入子查询 |
| 配置库 | `br_one.*` | 支持配置库代理查询和部分配置联查 | 业务 SQL 需符合 Query 规范 | 目前实体实例联查主要支持 `br_one` 与 `entity`；全库联查能力已废弃 |
| 元数据 | `meta.*` | 分场景支持模型元数据、维度元数据、记录元数据、日志索引元数据等 | 需按元数据表定义确认 | 手册总览与后续章节口径不一致，需拆分后确认正式能力 |
| 第三方数据源 | `thr_db.*`、Nebula 相关表 | 当前已支持 Nebula；ELK、日志易为规划方向 | 依赖数据源类型 | 外部数据源延迟、限流、权限、错误码需单独声明 |

对于正式白皮书，建议将上表中的每一行扩展为独立能力卡片，明确“支持 SQL 模式、必填过滤条件、禁止语法、默认限制、超限错误、慢查询治理策略、测试样例”。

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

### 10.6 当前 Query 系统需优先固化的资源边界

结合手册和已暴露的 CPU、内存问题，以下边界应优先从“隐含规则”变成“白纸黑字的准入规则”和“可自动化测试的限制”：

- 所有模型数据查询必须携带账号、用户、环境、资源域和时间范围；缺失时应快速失败，避免全量扫描。
- `monitorTime`、`monitorTimeMs`、`receiveTimeMs` 等时间条件应明确最大查询跨度，并区分明细表、分钟粒度、小时粒度、天粒度。
- 第三方指标明细查询手册中标注保留 3 天；超过范围应自动引导到粒度查询或返回明确错误。
- 预设指标仅支持粒度查询，不应落到明细查询路径。
- Nebula 实体关系边查询默认存在 1w 条限制；正式白皮书应声明默认值、可配置范围、超限行为和是否允许业务申请放宽。
- 实体关系联查必须要求精准关系条件，例如 `go_from()`，避免远端关系表全扫。
- 日志详情和 live tail 查询必须要求索引 ID 或时间范围等强过滤条件，并限制 `limit` 上限。
- 大 JOIN、跨源 JOIN、实体关系 JOIN、日志聚合、标签展开、`groupUniqArray`、`uniqTheta`、`quantile` 等高资源查询应纳入慢查询和资源限流策略。
- 子查询仅在受控场景开放；需要限制子查询层数、子查询数量、每个子查询扫描范围和中间结果规模。
- 标签查询返回 map 结构时，应明确单条记录最大标签数量和响应体大小。

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

### 17.4 当前 Query 迁移与数据兼容策略

《Query 查询应用》手册中已经给出较明确的迁移思路，建议纳入正式白皮书和需求评审流程：

- 业务侧先逐步切换查询写法，底层存储暂不切表。
- 业务侧全部切换完成后，再按相关指标批量切换底层存储。
- V3/V4 到 V1、VM-V1 明细、V2 到 V5 等兼容路径需要形成独立迁移矩阵。
- 类型、函数等变更后，优先使用新老表 `union all` 的方式兼容。
- 聚合中间态函数使用 state 进行 union，非中间态直接使用聚合函数后 union。
- 对无法 union 的场景，应通过双写一段时间实现动态切换，并明确双写窗口、回滚策略和业务可接受的数据舍弃范围。
- 对 `uniqTheta` 切换为 `uniqHLL12` 这类聚合算法变更，应在白皮书中声明不同时间范围命中不同算法时的查询行为。

迁移能力不应只作为实现细节存在。每次存储切换、函数切换、粒度表切换、模型版本切换，都应同步更新：

- 对外查询语义
- 兼容范围
- 数据保留范围
- 查询结果差异
- 线上回放 SQL 集
- 回滚条件
- 数据校验方式

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

### 21.7 当前 Query 手册场景回归测试

结合《Query 查询应用》手册，建议将文档中的示例 SQL 和接口场景沉淀为固定回归集：

- PQL 官方接口兼容：`query`、`query_range`、`labels`、`label/<label_name>/values`、`series`。
- PQL 粒度参数：`source_granularity_unit`、`source_granularity`、`granularity_function`。
- 第三方指标明细查询 3 天保留范围与粒度查询切换。
- 预设指标仅粒度查询。
- BQL 权限必填条件校验：`accountId`、`userId`、`envId`、`resourceZoneId`。
- SQL 头部 JSON 注释扩展信息解析：`one-traceId`、`business`。
- 权限条件禁止表别名。
- `where` 禁止列别名过滤。
- 指标实体关键字、配置关键字、标签关键字解析。
- 多指标维度一致时列式查询。
- 多指标维度不一致时报错或走受限子查询。
- 开窗聚合函数 `cycle` 参数和限制。
- 受限子查询分组维度一致校验。
- 记录查询 `recordKey` 必填。
- 实体实例查询表别名、列别名强制校验。
- 高基数实例查询 `metricId`、`monitorTime`、`aggFunction`、`dimensionId` 等参数校验。
- Nebula 关系边查询 `go_from()` 精准条件校验。
- Nebula 关系子查询默认 1w 限制。
- 事件表反引号查询。
- `brEventRelationEntityKey` 使用位置限制。
- 日志详情查询 `indexId`、`monitorTimeMs` 必填。
- 日志 live tail 查询 `receiveTimeMs` 必填。
- 日志查询不支持 join、子查询、取样、第三方日志的错误提示。
- 配置库 `br_one` 与 `entity` 联查。
- 新老表 `union all` 兼容查询。
- 聚合函数变更和双写窗口命中规则。

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
| 查询域是指标、记录、会话、事件、日志、实体、配置库还是第三方数据源？ |  |
| 是否使用 PQL？是否需要 Prometheus 官方接口兼容？ |  |
| 是否使用 BQL(SQL)？是否依赖子查询、JOIN、窗口聚合、标签、实体关键字或配置关键字？ |  |
| 是否能保证每条查询都携带账号、用户、环境、资源域和时间范围？ |  |
| 查询时间跨度是多少？是否命中明细表、分钟粒度、小时粒度或天粒度？ |  |
| 是否需要查询第三方指标明细？是否超过 3 天保留范围？ |  |
| 是否涉及高基数实体？是否能提供 `metricId`、`aggFunction`、`dimensionId` 等精确条件？ |  |
| 是否涉及 Nebula 实体关系查询？是否能提供 `go_from()` 等精准关系条件？ |  |
| 是否涉及日志详情或 live tail？是否能提供 `indexId` 和毫秒级时间范围？ |  |
| 是否需要 br_one、meta、entity 或 metric 的历史全库联查能力？ |  |
| 是否处于新老模型、函数、粒度表或存储切换窗口？ |  |
| 是否要求 Query 层对业务无感切换底层存储或计算函数？ |  |

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
