# Query 语法与能力说明

本文档基于《Query 查询应用》使用手册整理，用于单独描述当前 Query 系统的查询语法、能力边界、使用方式和限制。它补充主白皮书中的通用数据库能力清单，重点回答两个问题：

1. 标准数据库系统通常包含哪些查询相关能力，当前 Query 已经支持哪些。
2. Query 已支持能力的具体体现方式、使用方式、必填条件和局限性是什么。

## 1. Query 系统定位

当前 Query 系统不应对外表述为完整通用数据库，也不支持事务。更准确的定位是：

- 统一查询服务。
- MySQL 协议接入层。
- PQL 与 BQL(SQL) 双语义查询层。
- 指标、记录、会话、事件、日志、实体、配置库和第三方数据源的查询适配层。
- 面向业务侧屏蔽底层存储、指标计算类型、慢查询优化、分库分表和部分存储切换差异的查询网关。

对外应强调“支持 MySQL 驱动接入和 SQL 子集查询”，而不是“完整兼容 MySQL 数据库”。

## 2. 标准数据库查询能力与当前支持情况

| 标准数据库能力 | 标准数据库通常包含 | Query 当前状态 | 当前体现方式 | 主要限制 |
| --- | --- | --- | --- | --- |
| 连接协议 | 原生协议、认证、加密、连接管理 | 已支持部分 | MySQL 驱动、TCP 通信、账号鉴权、TLS 双向证书认证 | 需明确支持的 MySQL capability、驱动版本和连接池兼容矩阵 |
| SQL 查询 | SELECT、WHERE、GROUP BY、HAVING、ORDER BY、LIMIT 等 | 已支持子集 | BQL(SQL) 查询模型数据、实体、配置、日志、事件等 | 不应承诺完整 MySQL SQL；子查询、别名、JOIN、日志查询等存在限制 |
| PQL 查询 | 非标准数据库能力，属于时序/监控查询生态 | 已支持 | Prometheus 风格接口和语义 | V1/V5 支持，V2/V3/V4 暂不支持或优先级低；粒度查询需额外参数 |
| DDL | CREATE/ALTER/DROP 等对象定义 | 未在手册中体现 | 不建议对外承诺 | Query 以查询为主，是否支持 DDL 需另行确认 |
| DML | INSERT/UPDATE/DELETE 等数据变更 | 未在手册中体现 | 不建议对外承诺 | 当前文档只覆盖查询应用 |
| 事务 | BEGIN/COMMIT/ROLLBACK、隔离级别、锁 | 不支持 | 不提供事务语义 | 不应暴露事务语法、隔离级别和锁能力承诺 |
| JOIN | 多表关联、跨表查询 | 有限制支持 | 实体联查、配置库联查、受限子查询、Nebula 关系联查 | 全库联查废弃；跨源和关系联查必须受控；日志不支持 join |
| 子查询 | 嵌套查询、派生表、相关子查询 | 有限制支持 | 分组维度一致时可做受限子查询，效果类似 FULL JOIN | 手册存在“不支持嵌套”和“支持子查询”的口径差异，应声明为受限支持 |
| 聚合 | SUM/AVG/MIN/MAX/COUNT/分位数等 | 已支持业务聚合函数 | `sum`、`avg`、`min`、`max`、`last`、`uniqTheta`、`quantile`、`frm` 等 | 受指标模型、粒度表、维度一致性和函数变更影响 |
| 窗口函数 | OVER/PARTITION/ORDER 窗口计算 | 以 Query 特有方式支持 | 聚合函数增加 `cycle` 参数 | 非标准 SQL 窗口语法；需 `monitorTime` 时间对齐；多个窗口大小必须一致 |
| 元数据查询 | information_schema、系统表、对象元数据 | 口径需拆分确认 | 手册后续章节有 br_one/meta 联查说明 | 总览写“未具备元数据查询支持”，后文写“支持 br_one 和 meta 联查”，需区分系统元数据和业务元数据 |
| 权限控制 | 用户、角色、库表列权限、行级权限 | 已支持业务权限过滤 | 业务侧传账号、用户、环境、资源域，Query 统一处理权限 | 查询必须携带权限条件；权限条件禁止加表别名 |
| 多租户隔离 | 租户级数据和资源隔离 | 业务维度支持 | `accountId`、`userId`、`envId`、`resourceZoneId` | 资源隔离、限流、配额需进一步白皮书化 |
| 查询优化 | 优化器、执行计划、统计信息、下推 | 部分能力由 Query 内部承担 | 业务侧无感切换指标计算类型、慢查询优化、分库分表 | 需要补充对外可观测的 explain、慢查询、资源限制和错误行为 |
| 跨源查询 | 联邦查询、外部表、跨引擎适配 | 有限制支持 | Nebula 已支持；ELK、日志易规划中 | 第三方数据源延迟、限流、错误码、权限和能力差异需单独声明 |
| 日志查询 | 日志详情、日志检索、live tail | 有限制支持 | `data.logdetails`、`data.loglive` | 不支持 join、子查询、取样查询、第三方日志 |
| 事件查询 | 事件属性、关联实体、扩展字段 | 有限制支持 | `data.\`event\`` | `event` 是关键字需反引号；部分内置字段不可用于 where |
| 备份恢复 | 备份、恢复、PITR | 未在手册中体现 | 不建议在 Query 语法文档中承诺 | 属于底层存储或平台能力 |

### 2.1 SQL 语法细粒度能力对照

上表只适合做能力总览。正式对外或需求评审时，应进一步拆到 SQL 语法颗粒度，避免“支持 JOIN”“支持 HAVING”“支持子查询”被理解为完整 MySQL 能力。

| 语法能力 | 标准数据库通常支持 | Query 当前支持情况 | 当前使用方式 | 当前边界和限制 |
| --- | --- | --- | --- | --- |
| SELECT 投影 | 普通列、表达式、函数、别名、通配符 | 支持子集 | 查询指标、实体属性、配置字段、日志字段、事件字段 | 实体实例查询列必须带表别名；部分内置字段仅可出现在特定子句 |
| WHERE 过滤 | 原始列、表达式、函数、列别名、子查询过滤 | 支持子集 | 使用原始字段过滤，如 `appId in (...)`、`monitorTime >= ...` | `where` 暂不支持列别名过滤；权限条件禁止表别名；必须携带账号、用户、环境、资源域和时间范围 |
| 权限过滤 | 可通过行级权限、视图、策略或业务字段过滤 | 支持业务字段过滤 | `accountId`、`userId`、`envId`、`resourceZoneId` | 权限条件原则上放在最外层；实体、标签等内置关键字不需要重复添加权限过滤 |
| GROUP BY | 单列、多列、表达式、别名、位置序号 | 支持子集 | 指标维度、实体属性、标签、时间对齐字段分组 | 多指标列式查询要求维度一致；开窗查询必须包含时间对齐字段 |
| HAVING | 聚合后过滤，通常也可引用分组列或部分表达式 | 有限制支持 | 用于聚合结果过滤，如 `having num > 0` | 不支持对非聚合列做 HAVING 过滤；非聚合字段过滤应放到 `where`，并使用原始列名 |
| ORDER BY | 原始列、别名、表达式、聚合结果排序 | 支持子集 | 日志、事件、模型查询中按字段或聚合结果排序 | 需与查询域能力联动确认；大排序应纳入资源限制 |
| LIMIT | 限制返回行数、分页 | 支持 | `limit 10`、`limit 100` | 应声明默认上限和最大上限；日志、关系查询、大结果集必须限制返回规模 |
| INNER JOIN | 任意表内连接 | 有限制支持 | 受限子查询 JOIN、部分实体/配置联查 | 不支持任意库表联查；JOIN 两侧需要满足 Query 查询域规则 |
| 逗号表列表 JOIN 写法 | `from a, b where a.id = b.id` 通常等价于 INNER JOIN | 有限制支持 | 可作为 INNER JOIN 的等价写法单独验证 | 必须在 `where` 中提供明确关联条件；没有关联条件时会退化为笛卡尔 JOIN，当前不支持 |
| LEFT JOIN | 左连接 | 有限制支持 | `entity` 与 `br_one` 配置表联查；实体与关系边子查询联查 | 与 Nebula 关系边联查时，关系边建议先放入子查询；权限条件只在外层添加 |
| RIGHT JOIN | 右连接 | 未在手册中体现 | 不建议对外承诺 | 如业务需要需单独验证解析、执行和结果语义 |
| FULL JOIN | 全连接 | 不直接按标准 SQL 承诺 | 受限子查询组合的效果类似 FULL JOIN | 仅在各子查询分组维度一致的场景下支持组合查询；不代表支持任意 `FULL JOIN` 语法 |
| CROSS JOIN / 笛卡尔 JOIN | 无条件 JOIN、笛卡尔积 | 不支持 | 不应使用 | 当前不支持笛卡尔 JOIN；需求评审中应拒绝无关联条件的 JOIN，避免结果爆炸和资源风险 |
| JOIN ON 条件 | 等值、非等值、多条件、表达式、函数 | 支持子集 | 实体实例、配置表、关系边子查询使用明确关联条件 | 关系查询必须尽量命中精准条件；Nebula 边查询需 `go_from()` 等强过滤 |
| 跨库/跨源 JOIN | 跨库、跨 schema、外部源联邦 JOIN | 有限制支持 | 当前主要体现为实体、配置库和 Nebula 关系查询 | 全库联查废弃；第三方数据源能力差异、延迟、限流和错误码需单独声明 |
| 子查询 | 派生表、嵌套查询、相关子查询、标量子查询、EXISTS | 有限制支持 | 分组维度一致的派生表组合查询 | 不支持任意嵌套 SQL；应限制层数、数量、中间结果规模和扫描范围 |
| UNION | `union all`、`union distinct`、类型对齐 | 场景化支持 | 记录查询、新老表兼容、函数切换兼容 | 类型、函数中间态和时间窗口需按迁移策略校验 |
| 聚合函数 | 标准聚合和扩展聚合 | 支持业务聚合函数 | `sum`、`avg`、`min`、`max`、`last`、`uniqTheta`、`quantile`、`frm` | 受指标模型、粒度、维度一致性和函数演进影响 |
| 窗口函数 | `over(partition by ... order by ...)` 标准窗口语法 | 以 Query 特有方式支持 | 聚合函数增加 `cycle` 参数 | 不支持标准 SQL `over` 语法；多个窗口列要求窗口大小一致 |
| 列别名 | SELECT 别名可在部分子句引用 | 支持子集 | SELECT 中可设置别名 | `where` 不支持列别名过滤；同一指标同时查开窗和不开窗时必须指定别名 |
| 关键字转义 | 反引号、双引号、保留字处理 | 部分需要 | 事件表使用 `data.\`event\`` | 关键字表名必须转义；字段和库表命名规范需继续固化 |

### 2.2 等价语义的不同写法也要单独声明

标准数据库中，同一个查询语义经常存在多种 SQL 写法。Query 能力说明和测试矩阵不能只写“支持某能力”，还应列出该能力支持哪些写法、哪些写法不支持、哪些写法虽然语义相近但会触发不同执行路径。

以 INNER JOIN 为例，至少应拆分为以下场景：

| 场景 | 示例 | Query 口径 |
| --- | --- | --- |
| 显式 INNER JOIN | `from a join b on a.id = b.id` | 作为 JOIN 能力单独验证，要求明确 `on` 条件 |
| 隐式 INNER JOIN | `from a, b where a.id = b.id` | 可作为 INNER JOIN 的等价写法单独验证，要求 `where` 中存在明确关联条件 |
| 无关联条件逗号写法 | `from a, b` | 属于笛卡尔 JOIN / CROSS JOIN，当前不支持 |
| 显式 CROSS JOIN | `from a cross join b` | 当前不支持 |

同样的拆分原则也适用于：

- 聚合过滤：`having count(*) > 0` 与 `where status = 'error'` 不是同类过滤，Query 不支持用 `having` 过滤非聚合列。
- 子查询：派生表 JOIN、标量子查询、相关子查询、`exists` 不应合并成一个“支持子查询”口径。
- UNION：新老表兼容的 `union all`、记录查询的 `union distinct`、普通业务 SQL 的集合运算应分别声明。
- 时间聚合：`toStartOfInterval(monitorTime, INTERVAL 1 HOUR)` 与普通时间字段分组应分别测试。

### 2.3 MySQL 语法结构细分能力矩阵

为便于快速发现“不支持但标准数据库常见”的能力缺口，建议能力矩阵按 MySQL 查询语法结构继续拆分。下表中的“Query 当前口径”应作为需求准入、测试用例设计和后续补齐排期的起点；未在手册中明确出现的能力，先按“未确认 / 不建议承诺”管理。

| 一级语法位置 | 二级分类 | 写法能力点 | MySQL 标准含义 | Query 当前口径 | 示例 | 限制或补齐方向 |
| --- | --- | --- | --- | --- | --- | --- |
| SELECT | 普通字段投影 | 按列名查询 | 返回指定字段 | 支持子集 | `select appId from data.metrics` | 字段必须属于当前查询域；实体实例查询列需带表别名 |
| SELECT | 表别名字段 | `表别名.列名` | 消除多表字段歧义 | 实体实例查询要求支持 | `select a.instanceId from entity.host a` | 实体实例数据查询所有列必须使用表别名 |
| SELECT | 通配符 | `*`、`a.*` | 返回全部字段或指定表字段 | 有限制支持 / 需确认 | `select a.* from entity.app a` | 大结果集风险高，需限制查询域和返回规模 |
| SELECT | 表达式 | 算术、字符串、函数表达式 | 计算派生列 | 支持子集 | `maxMonitorTime - minMonitorTime as duration` | 支持范围依赖底层函数和查询域 |
| SELECT | 列别名 | `expr as alias` | 给输出列命名 | 支持 | `count(*) as num` | `where` 不支持引用列别名；开窗与非开窗同指标同时查询必须指定别名 |
| SELECT | 聚合函数 | 标准聚合 | 聚合多行数据 | 支持业务函数子集 | `sum(metricId)`、`avg(metricId)` | 受指标模型、粒度表、维度一致性和函数演进影响 |
| SELECT | 业务聚合函数 | 平台自定义聚合 | 指标模型专用计算 | 支持 | `frm(metricId)`、`uniqTheta(metricId)` | 函数变更需声明兼容策略，如 `uniqTheta` 到 `uniqHLL12` |
| SELECT | 窗口类聚合 | 标准 MySQL 使用 `over(...)` | 窗口范围内计算 | Query 特有写法支持 | `sum(metricId, 1)` | 不支持标准 `over` 语法；使用 `cycle` 参数且需时间对齐 |
| FROM | 单表查询 | `from db.table` | 从单表读取 | 支持子集 | `from data.metrics` | 表必须属于开放查询域 |
| FROM | schema/table 命名 | `database.table` | 跨 schema 指定对象 | 支持业务库表 | `data.metrics`、`entity.host` | 不是任意库表开放；全库联查已废弃 |
| FROM | 关键字表名 | 反引号转义 | 查询保留字对象 | 支持需要转义的场景 | `from data.\`event\`` | `event` 是关键字，必须转义 |
| FROM | 表别名 | `from table a` | 后续用别名引用表 | 实体实例查询要求支持 | `from entity.host a` | 实体实例、关系联查必须使用表别名 |
| FROM | 派生表 | `from (select ...) a` | 子查询结果作为表 | 受限支持 | `from (...) a join (...) b` | 各子查询分组维度必须一致；限制层数、数量和中间结果规模 |
| FROM | 逗号表列表 | `from a, b` | 标准上可形成笛卡尔积，配合 `where` 可等价内连接 | 有条件支持 | `from a, b where a.id = b.id` | 必须有明确关联条件；无条件逗号表列表不支持 |
| JOIN | 显式 INNER JOIN | `join ... on ...` | 内连接 | 有限制支持 | `from a join b on a.id = b.id` | 不支持任意库表联查；必须满足查询域规则 |
| JOIN | 隐式 INNER JOIN | `from a, b where a.id = b.id` | 等价内连接写法 | 有限制支持 | `from a, b where a.id = b.id` | 与显式 JOIN 分开测试；必须有明确关联条件 |
| JOIN | LEFT JOIN | `left join ... on ...` | 保留左表记录 | 有限制支持 | `entity` left join `br_one` | Nebula 关系边建议放入子查询；权限条件只在外层添加 |
| JOIN | RIGHT JOIN | `right join ... on ...` | 保留右表记录 | 未确认 / 不建议承诺 | `a right join b on ...` | 需单独验证解析、执行和结果语义 |
| JOIN | FULL JOIN | `full join ...` | 保留两侧记录 | 不按标准语法承诺 | 受限子查询组合效果类似 FULL JOIN | 不代表支持任意标准 `FULL JOIN` |
| JOIN | CROSS JOIN | `cross join` | 笛卡尔积 | 不支持 | `from a cross join b` | 应拒绝或返回明确错误，避免结果爆炸 |
| JOIN | 无条件逗号 JOIN | `from a, b` | 笛卡尔积 | 不支持 | `from a, b` | 与隐式 INNER JOIN 区分；无关联条件必须拒绝 |
| JOIN | ON 等值条件 | `on a.id = b.id` | 等值关联 | 支持子集 | `on t2.instance_id = t1.instanceId` | 关系查询需精准条件 |
| JOIN | ON 多条件 | `on a.id=b.id and ...` | 多条件关联 | 支持子集 | `on a.instanceId = ... and b.xxx = ...` | 需确认表达式、函数、非等值条件支持范围 |
| JOIN | USING / NATURAL | `using(col)`、`natural join` | 简化或自动关联 | 未确认 / 不建议承诺 | `join b using(id)` | 未在手册中体现，建议默认不承诺 |
| WHERE | 比较过滤 | `=`, `!=`, `>`, `<`, `>=`, `<=` | 行级过滤 | 支持子集 | `serviceId > 0` | 字段必须属于当前查询域 |
| WHERE | 集合过滤 | `in`, `not in` | 多值过滤 | 支持子集 | `appId in (1429)` | 大集合需限制参数数量 |
| WHERE | 模糊过滤 | `like` | 字符串匹配 | 支持子集 | `detectedName like ...` | 模糊查询需评估索引和扫描风险 |
| WHERE | NULL 判断 | `is null`, `is not null` | 空值过滤 | 支持子集 | `tags.appId is null` | 标签字段和普通字段需分别测试 |
| WHERE | 列别名过滤 | 在 `where` 中引用 select 别名 | MySQL 通常不允许，部分系统扩展支持 | 不支持 | `where avgValue > 0` | 使用原始列名或把过滤放到聚合后的 `having` |
| WHERE | 权限过滤 | 账号、用户、环境、资源域 | 业务权限约束 | 必填 | `accountId=5 and userId=5 and envId='default'` | 禁止加表别名；原则上放在最外层 |
| WHERE | 时间范围 | 明细或粒度表时间条件 | 控制扫描范围 | 必填 | `monitorTime >= ... and monitorTime < ...` | 不同查询域使用 `monitorTime`、`monitorTimeMs`、`receiveTimeMs` |
| WHERE | 实体关键字过滤 | `entity.modelKey[...]` | 实体关联过滤 | 支持子集 | `entity.app.appType = 1` | 多维度指向同一实体时需显式指定维度 ID |
| WHERE | 标签过滤 | `tags.*` | 数据标签或实例标签过滤 | 支持子集 | `tags.appId is null` | 无关联实体时使用标签关键字会报错 |
| WHERE | 子查询过滤 | `exists`、`in (select ...)` | 以子查询结果过滤 | 未确认 / 不建议承诺 | `where exists (...)` | 当前受限子查询主要是派生表组合，不应泛化承诺 |
| GROUP BY | 原始列分组 | `group by col` | 按字段聚合 | 支持子集 | `group by appId` | 字段需属于查询域和指标维度 |
| GROUP BY | 多列分组 | `group by col1, col2` | 组合维度聚合 | 支持子集 | `group by appVersion, monitorTime` | 多指标要求维度一致 |
| GROUP BY | 表达式分组 | `group by expression` | 按表达式结果分组 | 支持子集 | `group by toStartOfInterval(...)` | 开窗查询要求时间对齐表达式 |
| GROUP BY | 别名分组 | `select expr as x ... group by x` | MySQL 支持按别名分组 | 未确认 / 需单独测试 | `group by monitorTime` | 若别名与原字段同名需确认解析语义 |
| GROUP BY | 列序号分组 | `group by 1` | 按 select 第 1 列分组 | 未确认 / 需单独测试 | `select appId, count(*) ... group by 1` | 用户点名的重点补齐项，建议加入回归矩阵 |
| GROUP BY | 实体关键字分组 | `group by entity.xxx` | 按关联实体属性分组 | 支持子集 | `group by entity.app.appName` | 维度歧义时必须指定维度 ID |
| GROUP BY | 标签分组 | `group by tags.xxx` | 按标签聚合 | 支持子集 | `group by entityTag, DataTag` | 标签返回 map 结构，需限制展开规模 |
| HAVING | 聚合表达式过滤 | `having count(*) > 0` | 聚合后过滤 | 支持子集 | `having num > 0` | 应限制为聚合结果或聚合别名 |
| HAVING | 聚合别名过滤 | `having alias > 0` | 使用 select 聚合别名过滤 | 支持子集 / 需确认 | `having num > 0` | 示例体现该用法，需补充正式测试 |
| HAVING | 非聚合列过滤 | `having status = 'x'` | 标准 MySQL 可能允许特定模式下使用 | 不支持 | `having appId = 1` | 非聚合字段过滤应下推到 `where` |
| ORDER BY | 原始列排序 | `order by col` | 按字段排序 | 支持子集 | `order by eventLevel` | 大排序需资源限制 |
| ORDER BY | 别名排序 | `order by alias` | 按输出别名排序 | 支持子集 / 需确认 | `order by hostId desc` | 示例体现该用法，需形成测试项 |
| ORDER BY | 列序号排序 | `order by 1` | 按 select 第 1 列排序 | 未确认 / 需单独测试 | `order by 1 desc` | 标准 MySQL 常见能力，建议补充支持状态 |
| ORDER BY | 表达式排序 | `order by expression` | 按表达式排序 | 未确认 / 不建议承诺 | `order by count(*) desc` | 需结合聚合和资源限制验证 |
| LIMIT | 限制条数 | `limit n` | 返回前 n 行 | 支持 | `limit 10` | 应声明默认上限和最大上限 |
| LIMIT | 偏移分页 | `limit offset, n`、`limit n offset offset` | 分页 | 未确认 / 需单独测试 | `limit 10 offset 20` | MySQL 两种写法应分别列为能力点 |
| UNION | `union all` | 不去重合并 | 场景化支持 | 新老表兼容、记录查询 | 类型对齐和中间态函数需校验 |
| UNION | `union distinct` | 去重合并 | 场景化支持 | 记录查询 | 去重成本和字段类型需限制 |
| 子查询 | 派生表 JOIN | `from (select ...) a join ...` | 子查询作为表 | 受限支持 | 多指标组合查询 | 要求各子查询分组维度一致 |
| 子查询 | 标量子查询 | `select (select ...)` | 单值子查询 | 未确认 / 不建议承诺 | `select (select count(*))` | 未在手册中体现 |
| 子查询 | 相关子查询 | 子查询引用外层列 | 行相关执行 | 未确认 / 不建议承诺 | `where exists (...)` | 高资源风险，不应默认承诺 |
| 注释 | SQL 头部 JSON 注释 | 注释通常被数据库忽略 | Query 扩展支持 | `/**{"one-traceId":"..."}*/ select ...` | 仅支持约定字段；格式错误行为需声明 |

### 2.4 子查询能力与 MySQL 子查询场景矩阵

当前子查询能力应按“受限支持”对外说明。

支持场景：

- 各子查询分组维度一致。
- 用于多指标维度不一致时的组合计算。
- 实现效果等同于 FULL JOIN。

示例：

```sql
select (m1 + m2) / 100,
       a.appId,
       b.appName
from (
  select avg(one.rum.net.businessRequest.totalTime) as m1,
         appId,
         entity.app.customizedName as appName
  from data.metrics
  where accountId = 5
    and userId = 5
    and envId = 'default'
    and resourceZoneId = 1
    and monitorTime >= '2026-01-01 00:00:00'
    and monitorTime < '2026-01-13 00:00:00'
    and entity.app.appType = 1
  group by appId, appName
) as a
join (
  select avg(one.rum.net.businessRequest.totalTime) as m2,
         appId,
         entity.app.customizedName as appName
  from data.metrics
  where accountId = 5
    and userId = 5
    and envId = 'default'
    and resourceZoneId = 1
    and monitorTime >= '2026-01-13 00:00:00'
    and monitorTime < '2026-01-20 00:00:00'
    and entity.app.appType = 2
  group by appId, appName
) b on a.appId = b.appId and a.appName = b.appName;
```

限制：

- 不支持任意嵌套 SQL。
- 应限制子查询数量、层数、中间结果规模和扫描范围。
- 子查询中的分组维度必须一致。
- 权限条件仍应放在最外层或按 Query 规则校验，避免重复加权限条件导致语义错误。

MySQL 中“子查询”不是单一能力，而是按照出现位置、返回形态和是否引用外层字段分成多类。Query 目前手册只明确了受限派生表组合查询，因此其他子查询形态应先按“未确认 / 不建议承诺”管理，避免业务误以为完整支持。

| 一级位置 | 二级分类 | 写法能力点 | MySQL 标准含义 | Query 当前口径 | 示例 | 限制或补齐方向 |
| --- | --- | --- | --- | --- | --- | --- |
| SELECT | 标量子查询 | `select (select count(*)) as c` | 子查询返回单行单列作为一个值 | 未确认 / 不建议承诺 | `select (select count(*) from t2) as c from t1` | 未在手册中体现；需验证解析、执行次数和资源消耗 |
| SELECT | 相关标量子查询 | `select (select ... where b.id=a.id)` | 子查询引用外层行，每行计算一次 | 不建议承诺 | `select a.id, (select max(x) from b where b.id=a.id) from a` | 高资源风险，容易退化为逐行远端调用 |
| SELECT | 子查询表达式参与计算 | `(select v) + col` | 子查询结果参与表达式计算 | 未确认 / 不建议承诺 | `select value / (select total from t2) from t1` | 应优先改写为受控 JOIN 或预聚合结果 |
| FROM | 派生表 | `from (select ...) a` | 子查询结果作为临时表 | 受限支持 | `from (...) a join (...) b` | 当前支持重点；要求子查询分组维度一致 |
| FROM | 多派生表 JOIN | `from (subq1) a join (subq2) b` | 多个子查询结果关联 | 受限支持 | 多指标时间段对比查询 | 子查询数量、中间结果规模、扫描时间范围需要限制 |
| FROM | 派生表嵌套派生表 | `from (select ... from (select ...)) a` | 多层嵌套 | 不支持 / 不建议承诺 | `from (select * from (select * from t) x) a` | 当前“不支持任意嵌套 SQL”，应拒绝或限制层数为 1 |
| FROM | 派生表无别名 | `from (select ...)` | MySQL 要求派生表有别名 | 不支持 / 应拒绝 | `from (select ... )` | 应返回清晰语法错误，引导补充别名 |
| WHERE | `IN` 子查询 | `where col in (select col from ...)` | 使用多行单列结果过滤 | 未确认 / 不建议承诺 | `where appId in (select appId from ...)` | 当前手册只体现常量 `in (...)`；子查询版需单独验证 |
| WHERE | `NOT IN` 子查询 | `where col not in (select col from ...)` | 排除子查询结果 | 未确认 / 不建议承诺 | `where appId not in (select appId from ...)` | 需明确 NULL 语义和大结果集限制 |
| WHERE | `EXISTS` 子查询 | `where exists (select 1 ...)` | 判断子查询是否存在结果 | 未确认 / 不建议承诺 | `where exists (select 1 from b where b.id=a.id)` | 通常依赖相关子查询，资源风险高 |
| WHERE | `NOT EXISTS` 子查询 | `where not exists (...)` | 反半连接语义 | 未确认 / 不建议承诺 | `where not exists (...)` | 需明确反连接语义、NULL 和性能边界 |
| WHERE | 比较子查询 | `where col > (select max(...))` | 与单值子查询结果比较 | 未确认 / 不建议承诺 | `where value > (select avg(value) from t)` | 需要保证子查询单行单列，否则错误语义需定义 |
| WHERE | `ANY` / `SOME` / `ALL` | `where col > all (select ...)` | 与子查询集合做量词比较 | 未确认 / 不建议承诺 | `where latency > all (select latency from t2)` | MySQL 支持但手册未体现，建议默认不承诺 |
| WHERE | 相关子查询 | 子查询引用外层表字段 | 外层每行驱动子查询 | 不建议承诺 | `where exists (select 1 from b where b.id=a.id)` | 对 Nebula、日志、跨源查询风险极高，应优先改写 |
| WHERE | 子查询在权限过滤中 | 权限字段来自子查询 | 动态权限集合过滤 | 不支持 / 不建议承诺 | `where accountId in (select ...)` | 权限条件必须显式携带且禁止表别名，不应通过子查询绕过 |
| JOIN | 派生表 JOIN | `join (select ...) b on ...` | 子查询结果参与 JOIN | 受限支持 | `a join (select ... group by ...) b on ...` | 要求分组维度一致、明确关联条件 |
| JOIN | 子查询放在 ON 条件中 | `on a.id in (select ...)` | ON 内使用子查询过滤关联 | 未确认 / 不建议承诺 | `on a.id in (select id from b)` | ON 条件当前应保持明确关联条件，避免隐藏扫描 |
| HAVING | 聚合子查询 | `having sum(x) > (select ...)` | 聚合结果与子查询比较 | 未确认 / 不建议承诺 | `having num > (select avg(num) from ...)` | 当前 HAVING 仅建议用于聚合结果/别名过滤 |
| ORDER BY | 排序子查询 | `order by (select ...)` | 按子查询结果排序 | 未确认 / 不建议承诺 | `order by (select weight from ...)` | 高资源风险，手册未体现 |
| UNION | 子查询内部 UNION | `from (select ... union all select ...) a` | 子查询结果由集合运算产生 | 场景化支持 / 需确认 | 新老表 `union all` 兼容 | 仅在迁移兼容、记录查询等场景承诺；类型和函数中间态需校验 |
| DML | DML 子查询 | `insert into ... select ...`、`update ... where in (select ...)` | 写入或更新中使用子查询 | 不支持 | - | Query 不支持 DDL/DML 写入能力 |

对子查询需求，建议在需求评审时按以下顺序判断：

1. **是否能改写为当前已支持的派生表组合查询。** 如果能，应要求各子查询分组维度一致，并限制时间范围和中间结果规模。
2. **是否涉及相关子查询。** 如果子查询引用外层字段，应默认判定为高风险，不进入默认支持范围。
3. **是否出现在 `where` / `having` / `on` 中。** 这些位置的子查询容易隐藏扫描或改变过滤下推，应单独测试并声明错误行为。
4. **是否影响权限过滤。** 权限字段不应通过子查询间接提供，仍需显式携带 `accountId`、`userId`、`envId`、`resourceZoneId`。
5. **是否跨查询域或跨数据源。** 指标、实体、Nebula、日志、配置库之间的子查询组合应按跨源查询处理，明确超时、限流和错误码。

## 3. PQL 查询能力

### 3.1 支持接口

Query 对外提供类 Prometheus 的 PQL 查询接口：

| 请求方式 | Query 路径 | Prometheus API | 功能 |
| --- | --- | --- | --- |
| POST/GET | `/quer/api/v1/query` | `/api/v1/query` | 瞬时向量查询 |
| POST/GET | `/quer/api/v1/query_range` | `/api/v1/query_range` | 区间向量查询 |
| POST/GET | `/quer/api/v1/labels` | `/api/v1/labels` | 查询所有 label name |
| POST/GET | `/quer/api/v1/label/<label_name>/values` | `/api/v1/label/<label_name>/values` | 查询给定维度的所有维度值 |
| POST/GET | `/quer/api/v1/series` | `/api/v1/series` | 按 label 查询匹配序列 |
| POST/GET | `/quer/meta/v2` | 无 | 语法解析，非标接口 |

### 3.2 PQL 权限参数

查询所有指标数据时，需要携带账号和环境相关信息：

| 参数 | 说明 | 是否必填 |
| --- | --- | --- |
| `accountId` | 主账号 ID | 必填 |
| `userId` | 登录账号 ID | 建议必填 |
| `envId` | 环境 ID | 必填 |
| `resourceZoneId` | 资源域 ID | 建议必填 |

### 3.3 PQL 粒度参数

长周期指标数据查询需要使用粒度参数：

| 参数 | 说明 | 取值或规则 |
| --- | --- | --- |
| `source_granularity_unit` | 粒度单位 | `second`、`minute`、`hour`、`day` |
| `source_granularity` | 粒度值 | second/minute: 1-59；hour: 1-23；day: 1-364 |
| `granularity_function` | 明细数据聚合函数 | `last`、`min`、`max`、`avg`、`sum`、`uniqtheta` |

如果是非 V1 模型且为宽表模型，上述三个参数可不填写，默认规则为：

- `source_granularity_unit = minute`
- `source_granularity = 1`
- `granularity_function` 按 PQL 模式优先选择 `last`，再选择 `avg`，最后使用默认聚合方式。

### 3.4 PQL 支持范围与限制

- 第三方指标支持明细查询和粒度查询。
- 第三方指标明细查询保留 3 天，对标 Prometheus 指标数据查询。
- 粒度查询按 minute、hour、day 保留，用于长周期统计。
- 预设指标，如 RUM、APM、LOG、STM 等，仅支持粒度查询。
- 第三方指标、业务预设指标做 PQL 查询时，需要增加粒度、聚合函数等参数。
- BQL 支持单指标星型模型、多指标星型模型粒度查询。
- 单指标星型模型明细查询暂不支持。
- V1、V5 支持；V2、V3、V4 暂不支持或优先级较低。

## 4. BQL(SQL) 查询通用规则

### 4.1 接入方式

BQL 使用 MySQL 驱动接入，手册明确覆盖 Java、Python、C++ 等业务侧场景。当前支持：

- 账号鉴权。
- TLS。
- 本地预编译执行。
- SQL 头部注释传递扩展信息。

SQL 头部扩展信息示例：

```sql
/**{"one-traceId":"asdfghjkl","business":"abc123"}*/
select frm(one.rum.app.customError.userRatio),
       appVersion,
       toStartOfInterval(monitorTime, INTERVAL 1 HOUR) as monitorTime
from data.metrics
where resourceZoneId = 1
  and userId = 5
  and accountId = 5
  and envId = 'default'
  and monitorTime >= '2025-12-10 00:00:00'
  and monitorTime < '2026-01-10 23:59:59'
group by appVersion, monitorTime;
```

当前可传递扩展字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `one-traceId` | String | 前端页面向后传递的 trace ID，用于溯源和排障 |
| `business` | String | 业务 ID，需产品进一步定义 |

### 4.2 权限与过滤通用规则

所有模型查询原则上必须携带：

- `accountId`
- `userId`
- `envId`
- `resourceZoneId`
- 时间范围，如 `monitorTime`、`monitorTimeMs`、`receiveTimeMs`

重点限制：

- 权限相关条件禁止增加表别名。
- 权限条件原则上放在 SQL 最外层。
- `where` 条件暂不支持使用列别名过滤，必须使用原始列名。
- 业务侧查询数据无需额外添加权限控制逻辑，由 Query 统一处理。
- 实体、标签等内置关键字无需重复添加账号权限过滤。

## 5. BQL(SQL) 查询域与使用方式

### 5.1 指标数据查询

典型表：

- `data.metrics`

能力：

- 指标聚合。
- 维度分组。
- 实体属性关联。
- 配置表关联。
- 标签查询。
- 粒度查询。

典型写法：

```sql
select avg(one.rum.net.totalTime) as totalTime,
       entity.app.appName
from data.metrics
where accountId = 5
  and userId = 5
  and envId = 'default'
  and resourceZoneId = 1
  and monitorTime >= '2025-12-01 00:00:00'
  and monitorTime < '2025-12-02 00:00:00'
group by entity.app.appName;
```

限制：

- 多指标列式查询要求所有指标维度一致，即 `dimensionId + dataType` 一致。
- 多个指标维度不一致时，应使用受限子查询模式。
- 当同一个指标的多个维度指向同一实体时，必须显式指定维度 ID，否则应报错。
- 单指标星型模型明细查询暂不支持。
- `monitorTime` 为必填条件，并决定查询数据源。

### 5.2 记录数据查询

典型表：

- `data.records`

能力：

- 查询记录属性。
- 查询关联实体属性。
- 支持 `union all` 或 `union distinct`。

典型写法：

```sql
select attribute,
       entity.modelKey.attribute
from data.records
where recordKey = 'crash'
  and accountId = 5
  and userId = 5
  and envId = 'default'
  and resourceZoneId = 1
  and monitorTime >= '2026-01-01 00:00:00'
  and monitorTime < '2026-01-02 00:00:00';
```

限制：

- `recordKey` 为必选参数。
- 其他权限和时间参数同指标查询。

### 5.3 会话数据查询

典型表：

- `data.session`

典型写法：

```sql
select eventCount,
       maxMonitorTime - minMonitroTime as duration,
       sessionId
from data.session
where eventCount > 0
  and accountId = 5
  and userId = 5
  and envId = 'default'
  and resourceZoneId = 1;
```

限制：

- 手册中会话查询示例较少，正式白皮书应补充字段清单、必填条件、可用聚合和排序能力。

### 5.4 实体实例查询

典型表：

- `entity.modelKey`
- 示例：`entity.\`host\``、`entity.app`、`entity.service`

能力：

- 实例列表查询。
- 实体属性查询。
- 实体标签查询。
- 高基数实例查询。

典型写法：

```sql
select a.envId as env
from entity.`host` a
where accountId = 52214
  and userId = 52214
  and resourceZoneId = 1
  and envId = 'szwyy';
```

限制：

- 表别名必填。
- select 列必须带表别名。
- 原则上所有查询都应携带主账号、用户、环境、资源域。
- `accountId`、`envId` 不支持作为实例列查询，查询端应已具备这些信息。
- 高基数实例查询需要补充 `metricId`、`monitorTime`、`aggFunction`、`dimensionId` 等条件。
- `dimensionId` 用于处理同一指标维度下多个维度关联到相同实体的场景；未传应报错。

### 5.5 实体关系联查

典型表：

- `entity.entity_relationship_edge`
- `entity.entity_relationship_vertex`

能力：

- 基于 Nebula 的实体点边查询。
- 实体实例与关系边联查。

推荐写法：

```sql
select n.src_vid as vid,
       s.instanceId as id,
       ifnull(s.customizedName, s.detectedName) as name
from entity.service s
left join (
  select t.src_vid
  from entity.entity_relationship_edge t
  where go_from() = 'service_1000_5_default'
) n on concat('service_', s.instanceId, '_', '5_default') = n.src_vid
where accountId = 5
  and userId = 5
  and envId = 'default'
  and resourceZoneId = 10
  and s.detectedName = 'pageName6.0'
limit 10;
```

限制：

- 实例数据联查只需要在最外层添加账号权限信息，left join 或子查询部分无需重复添加。
- 所有列、表、子查询必须添加表别名。
- 关系查询必须尽量命中精准数据。
- `entity_relationship_edge` 查询需要必填 `go_from()` 条件，避免全表扫描。
- 与实例联查时，关系边查询应改为子查询，因为底层不会自动下推 Nebula 查询条件。
- Nebula 远端调用默认有 1w 条数据限制，可通过配置调整。

### 5.6 事件查询

典型表：

- `data.\`event\``

能力：

- 标准事件属性查询。
- 扩展属性查询：`extendedAttribute.xxx`。
- 附加信息查询：`additionalInfo.xxx`。
- 关联实体查询：`relatedEntity.${modelKey}`。
- 事件关联实体 key 查询：`brEventRelationEntityKey`。

典型写法：

```sql
select eventLevel as eventLevel,
       brEventRelationEntityKey as `modelKeys`,
       any_value(relatedEntity.host) as hostId,
       extendedAttribute.alertId as alertId,
       any_value(additionalInfo.originInfo) as originInfo
from data.`event`
where accountId = 5
  and userId = 5
  and envId = 'default'
  and resourceZoneId = 10
  and monitorTime >= '2026-01-10 10:00:00'
  and monitorTime < '2026-02-12 12:00:00'
  and extendedAttribute.pid != '0'
  and eventType = 'one.processStart'
group by eventLevel, modelKeys, alertId
order by eventLevel;
```

限制：

- `event` 是数据库关键字，查询时必须加反引号。
- `accountId`、`userId`、`envId`、`resourceZoneId`、`monitorTime` 必传。
- `eventType` 不传时查询全部事件类型。
- `brEventRelationEntityKey` 只支持在 `select`、`group by`、`having` 中使用，不可用于 `where`。

### 5.7 日志详情查询

典型表：

- `data.logdetails`

能力：

- 查询日志消息。
- 查询日志属性。
- 查询关联实体。
- 查询实体属性。
- 支持 `group by`、`having`、`order by`、`limit`。

典型写法：

```sql
select any_value(logMessage),
       relationEntity.host as hostId,
       count(*) as num
from data.logdetails
where accountId = 6
  and userId = 6
  and envId = 'default'
  and resourceZoneId = 1
  and indexId = 4
  and monitorTimeMs >= '2026-04-17 10:11:32'
  and monitorTimeMs < '2026-04-18 19:11:32'
  and logStatus in ('error', 'warn', 'info')
group by hostId
having num > 0
order by hostId desc
limit 100;
```

限制：

- 不支持 join。
- 不支持子查询。
- 不支持取样查询。
- 暂不支持第三方日志。
- 必传 `accountId`、`userId`、`envId`、`resourceZoneId`、`monitorTimeMs` 时间范围、`indexId`。
- 字段名取日志属性元数据表 `meta.log_config_attribute` 的 `attribute_id`。

### 5.8 日志 live tail 查询

典型表：

- `data.loglive`

典型写法：

```sql
select logMessage as logMessage,
       relationEntity.host as hostId
from data.loglive
where accountId = 6
  and userId = 6
  and envId = 'default'
  and resourceZoneId = 1
  and monitorTimeMs >= '2026-04-13 16:11:32'
  and monitorTimeMs < '2026-04-13 17:11:32'
  and logStatus in ('error', 'warn')
  and entity.host.customizedName = 'aaa'
limit 10;
```

限制：

- 手册说明 live tail 必传 `receiveTimeMs` 时间范围，但示例使用 `monitorTimeMs`。正式口径需要统一。
- 字段名取日志属性表 `meta.log_config_attribute` 的 `attribute_id`。
- 需要限制时间窗口和返回行数，避免实时日志查询拖垮资源。

### 5.9 标签查询

支持标签关键字：

- `tags.tags`：查询全部标签，包括数据标签和实例关联标签。
- `tags.dimessionId`：查询某个实体关联维度对应的所有关联标签。
- `tags.tag`：查询数据标签，仅限指标 V1 和 V5 查询。
- `表别名.tags`：实体实例标签查询。

示例：

```sql
select sum(one.rum.app.log.eventCount) as tt,
       dataCenterId as dataCenterId,
       tags.tag as DataTag,
       tags.tags as AllTag,
       tags.appId as entityTag
from data.metrics
where accountId = 5
  and userId = 5
  and envId = 'default'
  and resourceZoneId = 1
  and monitorTime >= '2026-02-05 10:00:00'
  and monitorTime < '2026-02-06 11:00:00'
  and tags.appId is null
group by dataCenterId, entityTag, DataTag, AllTag;
```

限制：

- 如果对应指标无关联实体，使用 `tags` 关键字会提示错误。
- 响应数据以存储的 map 形式返回，key 为 int64，value 为 string。
- `hasAny`、`hasAll` 等标签过滤需要单独形成支持矩阵。

### 5.10 配置库查询

典型能力：

- 配置库代理。
- 指标查询中通过 `config` 关键字关联配置。
- 实体实例与 `br_one` 配置表 left join。

配置关键字示例：

```sql
select detectedName,
       config.t_apm_agent_config[serviceId].instance_id as instanceId
from data.metrics;
```

实体与配置表联查示例：

```sql
select t1.detectedName as detectedName,
       t2.state
from entity."service" t1
left join br_one.t_apm_agent_config t2
  on t2.instance_id = t1.instanceId
where accountId = 5
  and userId = 5
  and resourceZoneId = 1
  and envId = 'default';
```

限制：

- 配置表注册需要向 meta 提需求，由 meta 维护表名和关联主键。
- 多关联列场景中，手册标注“暂不支持配置其他列和维度的关联”。
- 当前实体实例联查主要支持 `br_one` 和 `entity`。
- 历史全库联查能力已废弃，不应默认承诺。

## 6. 开窗聚合能力

Query 使用聚合函数增加 `cycle` 参数的方式实现窗口类查询。

支持形式：

```sql
sum(metricId [,cycle])
avg(metricId [,cycle])
min(metricId [,cycle])
max(metricId [,cycle])
last(metricId [,cycle])
uniqTheta(metricId [,cycle])
quantile(metricId, level [,cycle])
frm(metricId [,cycle])
```

使用限制：

- 存在开窗查询列时，select 列中必须包含对 `monitorTime` 使用 `toStartOfInterval` 的时间对齐函数。
- 多个开窗查询列要求窗口大小一致。
- 使用 `quantile` 分位数函数进行开窗查询时，必须指定分位值参数。
- 如果同一个指标同时查询开窗和不开窗结果，必须指定别名。

## 7. 明确废弃和不应承诺的能力

以下能力手册已标记为废弃或不应继续默认承诺：

- 全库数据联查支持。
- `br_one` 库全部表联查。
- `meta` 库全部表联查。
- `entity` 库按注册关系全量联查。
- `metric` 通过 config 方式访问注册库进行任意联查。
- 将 Query 当作完整 MySQL 替代品使用。
- 事务、隔离级别、锁能力。
- DDL/DML 写入能力，除非后续有专门文档确认。

## 8. 需进一步确认的口径

以下内容在手册中存在口径冲突或示例不足，建议产品、研发、测试共同确认：

- 元数据查询到底是不支持，还是仅支持业务元数据表查询。
- 日志、调用链、事件总览中写未具备，但后续已有日志和事件查询说明，正式口径应改为分场景有限支持。
- live tail 必填时间字段到底是 `receiveTimeMs` 还是 `monitorTimeMs`。
- PQL V2/V3/V4 暂不支持的准确范围和错误提示。
- `userId`、`resourceZoneId` 在 PQL 与 BQL 中是否都必须强制必填。
- 子查询权限条件到底应全部外置，还是允许子查询内重复携带。
- 实体 `modelKey` 是否会调整为更可读的识别 code。
- 标签过滤函数，如 `hasAny`、`hasAll`，是否正式对外支持。

## 9. 建议测试锚点

应将以下场景沉淀为固定自动化或回归测试：

- MySQL 驱动连接、鉴权、TLS。
- SQL 头部 JSON 注释解析。
- PQL 官方接口兼容。
- PQL 粒度参数与默认粒度选择。
- 第三方指标明细查询 3 天边界。
- 预设指标只走粒度查询。
- BQL 权限必填条件和权限条件禁止表别名。
- `where` 禁止列别名过滤。
- `having` 禁止非聚合列过滤，非聚合字段过滤应下推到 `where`。
- 笛卡尔 JOIN / CROSS JOIN 应拒绝或返回明确不支持错误。
- 显式 `join ... on ...` 和隐式 `from a, b where ...` 应作为两个独立 JOIN 写法测试。
- `from a, b` 无关联条件时应按笛卡尔 JOIN 拒绝，不应误判为普通 INNER JOIN。
- JOIN 必须携带明确关联条件，Nebula 关系查询还需强过滤条件。
- RIGHT JOIN、标准 FULL JOIN、任意跨库跨源 JOIN 不应被误判为已支持。
- `group by` 列名、别名、表达式、列序号应分别测试并标注支持状态。
- `order by` 列名、别名、表达式、列序号应分别测试并标注支持状态。
- `where`、`on`、`having` 中表达相近过滤语义的写法应分别测试，不能只按最终结果相似判断支持。
- `limit n`、`limit offset, n`、`limit n offset offset` 应分别测试并标注分页支持状态。
- `in` 列表、`in (subquery)`、`exists`、标量子查询应分别测试并标注支持状态。
- 指标、记录、会话、事件、日志、实体、配置库各查询域样例 SQL。
- 多指标维度一致和不一致场景。
- 受限子查询分组维度一致校验。
- FROM 派生表子查询作为当前主要支持子查询形态单独测试。
- SELECT 标量子查询、WHERE IN 子查询、WHERE EXISTS 子查询、相关子查询、HAVING 子查询等未确认能力应返回明确不支持或不承诺口径。
- 子查询层数、数量、中间结果规模和每个子查询扫描范围应纳入资源边界测试。
- 开窗聚合 `cycle` 参数校验。
- 高基数实体必填条件。
- Nebula `go_from()` 精准条件和 1w 限制。
- 日志查询不支持 join、子查询、取样和第三方日志。
- 事件表反引号和 `brEventRelationEntityKey` 位置限制。
- 配置表注册和 `br_one` 与 `entity` 联查。
- 废弃全库联查能力的拒绝或降级行为。
