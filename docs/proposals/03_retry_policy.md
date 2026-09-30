# 数据库错误的重试策略

状态：**提案**（未并入任何现有文档。本文只提建议，不修改主稿、运行时篇、Rust 实现设计、日志设计。）

## 0. 这份提案插在哪里

| # | 目标文档 | 位置 | 加什么 |
|---|---|---|---|
| 1 | [20260930_BACKEND_v1.md](../20260930_BACKEND_v1.md) 第 5 节「写路径」 | 新增 **5.6 数据库错误的重试** | 本文的第 1、2、3、4 节（分类总表 + 包装器形状 + 幂等性 + 退避参数）。主稿第 5 节定"写路径长什么样"，重试是写路径的一部分 |
| 2 | 主稿第 8 节「错误处理」 | **8.1 `DomainError` 两个变体改动**（加 `TemporaryUnavailable`、补上 `Conflict`）+ **`AppError::Sqlx` 带上原始 SQLSTATE**（见 6.2 的取舍说明）+ 重写 `From<sqlx::Error>` + 新增**8.2 状态码矩阵**（现 7.3 的表要改） | 本文第 6 节 |
| 3 | 主稿第 7.3 节「状态码矩阵」 | 改最后一行 `限流 / 事务重试耗尽 → 429 / 503` | 本文第 6.3 节。拆成三行，并补 `Retry-After` |
| 3b | [04_pool_and_timeouts.md](04_pool_and_timeouts.md) | **只读，不改**。它的 6.2 已按本文的出口定案，它的 6.2 末尾给本文的四条反馈已在 2.2 / 2.4.1 / 5.2 回应 | 见本文第 8 节末的"与提案 04 的关系" |
| 4 | [20260930_BACKEND_RUNTIME_v1.md](../20260930_BACKEND_RUNTIME_v1.md) | 新增 **R4.6 重试策略**（紧接 R4.5，与 R4.3 隔离级别的决定相邻，因为这两条必须一起读）；R1.4 级别表加一行（`biz.retry` = WARN）；R2.1 配置表加**三个重试参数**（取连接超时由提案 04 负责，**不要写两遍**）；R7.1 故障表补"连接池耗尽 → 503 + `Retry-After`"；R7.3 每周检查加 `biz.retry`；R7.5 验收清单加两条 | 本文第 4、5.3、7 节 |
| 5 | [20260930_RUST_IMPL_v1.md](../20260930_RUST_IMPL_v1.md) 第 3 节「事务模式」 | 新增 **3.4 重试包装器**（紧跟 3.1 签名约定与 3.3 批量取锁）；第 5.1 节 SQLSTATE 翻译扩几行；第 9.3 节测试汇总加 V3–V7 | 本文第 3、6、8 节 |
| 6 | [20260930_LOGGING_v1.md](../20260930_LOGGING_v1.md) 第 6 节「事件清单」 | 新增 `kind = biz.retry` 一行 + 第 8.3 节白名单测试的覆盖面说明。**`tx.idle` / `pool.stat` 由提案 04 负责，本文不动它们** | 本文第 7 节 |
| 7 | 主稿附录「实现顺序建议」 | 第 3 步与第 4 步之间 | 重试包装器必须在第一个写路径**之前**就位，否则每个写路径各自发明一套重试 |

**为什么不并入主稿第 5 节正文就完了**：因为重试同时改动了四份文档（分类规则在主稿、配置在运行时篇、代码形状在 Rust 篇、日志在日志篇），一次性写在一处、逐条指明落点，比分散改四份更容易评审。

**本文对主稿只做两处、且都是"必须"的改动**（其余全是新增章节）：

1. **加一个 `DomainError` 变体**（`TemporaryUnavailable`）——没有它，重试耗尽只能退化成 500。
2. **`AppError::Sqlx` 带上原始 SQLSTATE**——没有它，"`08006` 可重试 / `57014` 不可重试"这个区分在翻译层之后就消失了（详见 6.2）。**第二处比第一处代价大，6.2 给了它的退路方案**，评审时请连那一段一起看。

其余内容都是**新增**（主稿 5.6、运行时篇 R4.6、Rust 篇 3.4、日志设计的一个 `kind`），不改动现有决定。

**本文全篇的前提**：运行时篇 R4.3 已经决定 `READ COMMITTED` 并**明确拒绝 `SERIALIZABLE`**。本文不提任何形式的 `SERIALIZABLE`、快照隔离、乐观并发（`version` + `If-Match`）。本文要重试的只有两类东西：**数据库自己声明"这一笔因基础设施原因没做成"的错误**（死锁、连接失效），以及**极少数客户端可重试的资源竞争**。第 1 节那张表的立场是：**能重试的比大多数人以为的少得多。**

---

## 1. 一句话结论

**只重试"整笔事务确实没有生效"的错误——`40P01` 死锁与连接类错误（`08000`/`08006`/`57P01`），把 `23P01`（排他冲突）和 `23505`（唯一冲突）显式钉死为不可重试并在类型上让它们永远不能被归类为可重试，重试跑在 handler 层包装整个事务、位置是 `with_retry(|conn| async { … })` 而不是任何一条语句上，最多 3 次尝试、退避 = `min(50ms × 2^(n-1), 400ms)` 的 full jitter，耗尽后返回 `503` + `DomainError::TemporaryUnavailable`（"系统繁忙，请稍后重试"），每次重试记一条 `kind = "biz.retry"`（WARN）、并且只记一次最终的业务日志。**

---

## 2. 可重试性分类总表

### 2.1 判定规则（先读这四条，再看表）

1. **一条错误可重试，当且仅当"数据库或连接状态保证这一笔事务没有生效"。** 不是"这个操作看起来无害"。
2. **重试的收益来自"再试一次会走到不同的结果"。** 如果第二次执行面对的是同一份数据、同一个约束，重试只是把同一个错误变成第三个错误——或者把状态码从 409 变成 503。
3. **表里没列到的一律不可重试**（fail-closed）。新出现一个 SQLSTATE 时，默认是"不重试 + 500"，不是"先试试"。
4. **约束违例是单向的：`23P01` / `23505` 永远不能被归类为可重试**（见 2.3 的不变量）。

### 2.2 总表

**A. SQLSTATE（`sqlx::Error::Database` 里 `db.code()`）**

| SQLSTATE | 条件名 | 可重试 | 判据 | 耗尽 / 不重试后的出口 |
|---|---|---|---|---|
| `23P01` | `exclusion_violation`（`occupancy_no_overlap`） | **绝对不可** | 确定性的：同一区间、同一器材，第二次执行会读到同样的冲突行，约束会再次拒绝。重试只是把 409 浪费成 503 | 409 + `RESERVED`（主稿 3.2 的人话）+ `biz.conflict`（WARN）。**这是全表最容易做错的一格** |
| `23505` | `unique_violation` | **绝对不可** | 同上。`equipment_code_uniq` / `app_user_one_owner` 在重试时仍然是同一条已提交的行 | 409 + `DuplicateCode`；409 + `OwnerImmutable` |
| `40001` | `serialization_failure` | **不可**（本方案下） | `READ COMMITTED` 下 Postgres **不会**产生它。它出现 = 有人把隔离级别设成了 `SERIALIZABLE`（违反 R4.3）。**重试它会掩盖这个 bug**，正确动作是修配置 | 500 + `INTERNAL`，同时 ERROR 日志点名"隔离级别不对"。见 2.4 |
| `40P01` | `deadlock_detected` | **可** | 数据库**已经**回滚了整笔事务，且死锁方是被随机选中的——重试时锁的竞争关系已经变了，第二次极大概率成功。这是唯一天然可重试的数据库错误 | 若 3 次都死锁：503 + `TemporaryUnavailable` + ERROR 日志（这是真缺陷信号，见第 7.3 节） |
| `57014` | `query_canceled`（`statement_timeout` 或 `lock_timeout`） | **不可** | **两种成因共用这个码**：慢查询（`statement_timeout`）重试会在同一个预算里再烧一次；等锁（`lock_timeout`）重试**在安全上是可以的**（见 2.4.1），但两者在 SQLSTATE 上无法区分，所以只能在**能区分之后**才放宽，现在取保守值 | 503 + `TemporaryUnavailable`。**注意这不是防御性的格子**：提案 04 已经配置角色级 `statement_timeout = 15s` / `lock_timeout = 5s`，所以这条分支一定会被走到 |
| `25P03` | `idle_in_transaction_session_timeout` | **不可** | 服务端**主动关闭**了该连接并回滚事务，所以重试在安全上没问题；**但绝不能自动重试**——它意味着"有代码在事务里做了 I/O"（提案 04 第 4 节），重试只会再犯一次，而且重试会把这条"违反设计规则"的信号从日志里洗掉 | 503 + `TemporaryUnavailable`（提案 04 的 6.2 要求它落进同一个出口）+ ERROR 日志点名"事务空闲超时"，同时进 R7.3 每周检查（这条比 `40P01` 更需要人看） |
| `08000` | `connection_exception` | **可**（取连接阶段 / 事务体中；**提交阶段不可**） | 事务从未提交成功：连接都没建立，或事务中途连接失效（后端进程一死，事务必然回滚）。见下方说明与 3.2 阶段表 | 3 次都失败：503 + `TemporaryUnavailable` |
| `08006` | `connection_failure` | **可**（同上） | 典型是 Postgres 侧 `pg_terminate_backend`、网络抖动、failover。语义同 `08000` | 同上 |
| `57P01` | `admin_shutdown` | **可**（同上） | 数据库/实例正在关闭（部署、重启）。事务未生效 | 同上 |
| `3D000` `28P01` `28P02` `3F000` `42P01` `42703` | 库不存在 / 口令错 / schema 或表或列不存在 | **不可** | 配置或迁移问题，重试 3 次只是把启动故障变成三个慢请求 | 500 + `INTERNAL`（配置/迁移问题由 `/readyz` 与启动日志暴露，不在这里兜） |
| `53300` | `too_many_connections` | **不可** | 重试只会更满。**它最可能的原因是池预算算错了**（提案 04 的 3.5：app 10 + jobs 3 + 运维工具必须 ≤ Postgres 的 `max_connections`），而正确反应是查另一个进程的池大小，**不是**回头调重试次数 | 500 + `INTERNAL`，但要有一条指向池预算的说明 |
| `0A000` | `feature_not_supported` | **不可** | 典型是缺 `btree_gist`（主稿 12.3 已列为选型门槛） | 500 + `INTERNAL`。这条要在部署前由迁移验证挡住 |

> **关于 `08000`/`08006`/`57P01` 的一句必须说清的话（这一格与 `40P01` 不同）**：它们**在提交阶段也会出现**，而提交阶段**一律不重试**。
> 理由是："连接已经断了"看起来足以说明服务端不可能提交成功，但**"响应丢包"和"连接断了"在客户端是同一种错误**——如果服务端已经提交、而 `CommandComplete` 丢在网络上，重放就会写出第二份出库单。
> **立场：取连接阶段与事务体阶段可重试，提交阶段不重试。** 代价是极少数"提交时数据库重启"的场景给用户一个 500（他可再点一次，而重复点由 `occupancy` 排他约束或幂等键挡住）；收益是**绝不出现框架替用户写出重复数据**。与第 3.2 节第 3 点是同一个取舍。

**B. `sqlx::Error` 变体**

> ⚠️ **待验证**：以下变体清单按 `sqlx` 0.8 的常见形态整理，我**没有**可编译环境核对（本机无 `cargo`）。实现第一步应把 `sqlx::Error` 的枚举逐项对照一份，确认没有遗漏、没有改名。**风险点**：这是 `match` 穷尽性会被编译器立刻指出的一次性成本，不会静默出错。

| 变体 | 可重试 | 判据 |
|---|---|---|
| `Error::Io` | **仅建立连接阶段可** | 交易型的重连-重放最危险的地方是**提交阶段**：如果服务端已提交而我们没读到响应，重放会写出第二份。因此按**阶段**判断，而不是按变体：`begin()`/取连接阶段的 `Io` 可重试（一个字节的业务 SQL 都还没发出去）；已发出 SQL 之后的 `Io` 不重试。见第 3.2 节的分阶段实现 |
| `Error::PoolTimedOut` | **不可** | 直觉想重试，实际应该**绝对不**：池已经空了，重试会让每个请求多占一个等待位置，把"慢"放大成"雪崩"。这也正是运行时篇 R1.8 要把池饱和单列成一个信号的原因——信号要有人看，不要靠请求路径自救 |
| `Error::PoolClosed` | **不可** | 池已关闭 = 进程正在停机（R1.10）。重试只会拖过 drain 超时 |
| `Error::PoolAcquire`（若存在，**待验证**） | 视 `source` 而定 | 若来源是 `PoolTimedOut` → 不可；若是取连接时的 `Io` → 按 `Error::Io` 规则 |
| `Error::RowNotFound` | **不可** | 它**不是数据库错误**，是"我按主键取一行但没取到"。在写路径里这是应用层发现了不一致（锁住的行在事务里消失了），重试会再发现一次同样的事 | 
| `Error::WorkerCrashed`（**待验证**） | **可**（仅取连接阶段） | sqlx 的池把连接错误包装成它（语义接近连接失效），行为按上面 `Error::Io` 的规则处理 |
| `Error::Tls`（**待验证**） | **可**（仅取连接阶段） | 握手失败，事务未开始 |
| `Error::Protocol` | **不可** | 协议级不一致通常是 sqlx 与库版本/驱动的问题，重试无意义 |
| `Error::Configuration` / `Migrate` / `Encode` / `Decode` / `ColumnDecode` / `ColumnNotFound` / `ColumnIndexOutOfBounds` / `TypeNotFound` | **不可** | 全是代码/迁移/SQL 文本的问题。**重试是纯浪费**，而且会掩盖编译期该发现的东西 |
| `Error::Database` 但 `code()` 为 `None` | **不可** | 拿不到 SQLSTATE 就无法判定，fail-closed |

### 2.3 要写成显式规则的"最容易做错的地方"

> **规则 R-CONSTRAINT（不可重试单向锁）**
>
> **`23P01` 与 `23505` 永远不可重试，且不允许被任何"通用瞬时错误"分支（`_ => Retry`）捕获。**
>
> 原因是双重的：
> 1. **重试会再失败**——约束是确定性的，同一事务内容第二次执行面对同一条已提交的行。
> 2. **重试会把 409 变成 503**——这是用户可见的语义倒退：409 表示"换个日期或换件器材"（前端据此提示），503 表示"系统坏了稍后再试"（前端据此让用户重试）。**把一个本来能自助解决的问题变成一次无意义的等待，是这条重试策略唯一会造成实际伤害的错误。**

实现上不许靠"写对了 match 顺序"来维持它，而要靠 **`retry_class()` 的类型是封闭 enum、`_` 分支返回 `Fatal`**（第 3.2 节）。测试 V3（8.2）钉住它。

同一条规则的第二个面：**重试耗尽的错误码必须是 503，不是 500。** 用户/前端对 503 的动作是"稍后重试"，对 500 的动作是"报 bug"。整笔重试耗尽说明的是"系统暂时忙"，这是 503。

### 2.4 `40001` 的特例要说清楚

因为运行时篇 R4.3 选了 `READ COMMITTED`，`40001` 在本系统里**不可达**。收到它只有两种可能：

1. 有人在某个 `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE`、连接参数、或池的 `after_connect` 里改了隔离级别；
2. 将来有人"顺手"加了重试逻辑，同时又加了 `SERIALIZABLE`。

**处理：不重试，返回 500，并记一条 ERROR 日志点名这一点。** 理由：**一段为了修不存在的问题而写的重试逻辑，恰恰会掩盖"有人把不存在的问题引进来"这件事。** 让它在第一次出现时就以 500 的形状炸出来，比让它被重试悄悄吞掉好。

> 建议同时加一条 CI/启动检查（`SHOW transaction_isolation` 或池的 `after_connect` 断言），把"隔离级别必须是 `read committed`"变成启动即失败，而不是某天的一个 500。这条与 R4.3 是同一个决定的两种落地方式。

### 2.4.1 `lock_timeout` 的情况要单独说：它是"安全但暂不放宽"

**提案 04 配置了 `lock_timeout = 5s`，而它抛的 SQLSTATE 与 `statement_timeout` 完全相同（都是 `57014`）。** 这两件事的性质相反：

| 成因 | 数据库做了什么 | 重试**安全**吗 | 重试**值得**吗 |
|---|---|---|---|
| `statement_timeout`（慢查询） | 回滚语句，事务进入 aborted | 安全（没有写发生） | **不值得**——再跑一次大概率还是那么慢 |
| `lock_timeout`（等锁太久） | 同上 | 安全 | **往往值得**——锁的持有者通常正在提交，几十毫秒后就没了 |

**本提案的立场（明确写下来，以免被当成疏漏）**：

1. **现在两者都不重试**，因为 `retry_class` 只有 SQLSTATE 可用，无法区分它们。把 `57014` 放宽成可重试，等于**同时**放宽了慢查询——那会把"加索引"这个正确动作掩盖成"重试两次就好了"。**"宁可少重试"在这个不确定的场景下是更便宜的一侧。**
2. **要放宽，前提是先有区分能力。** 提案 04 的 6.3 方案 A（按错误消息文本区分）是那个前提。**在它做出来之前不要单独放宽 `57014`。**
3. **抖动是必需品，不是优化项**——这一条与判 57014 可不可重试无关。`lock_timeout` 的失败模式天然是**同步的**：一批请求同时等同一个锁、同时在 5 s 后超时。没有 full jitter，重试会造出下一轮同样同步的拥塞（第 5.1 节已把 full jitter 定为默认）。
4. **区分之后，日志要分开**（提案 04 的 6.2 实现约定第 3 条：`data.db_timeout_kind = "statement" | "lock"`），**用户可见的 `code` 不分开**。这一条本提案完全同意：用户对两者的正确动作都是"等几秒再点"。

> **给提案 04 的回应（两条反馈 #2 / #4，见提案 04 的 6.2）**：① `lock_timeout` 的瞬时性我已采纳为`57014` 判据里的显式说明，但**结论仍是不重试**，理由如上第 1 点；② `53300`（连接槽满）那一格我按它的建议补上了"先查池预算"的方向（2.2 A 表末行）。

### 2.5 不与本表冲突的两件事

- **主稿 7.3 的 `429`** 保持原义：**限流**（限流器触发时用）。重试耗尽**不用 429**（见 6.3）。
- **`RowNotFound` 与 `23P01` 的出口不是 5xx**，因此它们不在"重试"话题里——它们走主稿第 8 节既有的翻译路径，一步都不改。

---

## 3. 重试包装器的 Rust 形状

### 3.1 为什么必须在 handler 层、必须包住整个事务

三个约束把这个位置唯一确定了：

1. **主稿 5.1 / Rust 篇 3.1：事务边界在 handler**（`let mut tx = pool.begin()` … `tx.commit()`），服务函数收 `&mut PgConnection`。因此"重跑整个事务"这件事，**只有 handler 拿得到 `pool`**——服务层拿不到 pool（这正是 Rust 篇 3.1 想要的类型级保证）。**所以包装器只能放在 handler 层。**
2. **`sqlx::Transaction` 不能被复用。** 事务一旦回滚（死锁、连接失效都会导致回滚），这个 `Transaction` 就不能再 `commit`、也不能继续用；`begin()` 必须重新执行。因此包装器的循环必须在 `begin()` **外面**，闭包内不出现 `begin`/`commit`。
3. **重试的最小单位是事务，不是语句。** 主稿 5.1 的写路径里，一次出库会依次插入 `checkout` / `checkout_item` / 多条 `occupancy` / 改多件 `equipment.status` / 写多条 `history`。只重跑其中一条语句会得到一个半成品事务——比不重试更糟。

因此形状是：

```text
with_retry(pool, 策略) { |conn: &mut PgConnection| async {
    … 与主稿 5.1 一模一样的整个事务体 …
} }
```

**与 `&mut PgConnection` 签名约定的关系（重要）**：这条约定**原封不动**，而且正是它让重试可以写对。

- 服务函数与 db 函数**继续**收 `&mut PgConnection`，签名一个字都不改；
- 包装器**不给**闭包 pool，闭包内部**看不到** pool——所以"新开一个事务"这件事在闭包内**做不到**，与 Rust 篇 V2（服务层不接受 pool）是同一个方向的强制；
- 包装器自己在每次尝试时 `pool.begin()` 并 `commit()`，闭包收到的 `conn` 就是 `&mut *tx` 的一次重借用；
- 于是"每次尝试都是一个全新事务"不是纪律，而是**签名决定的**：闭包没有能力复用上一个 `Transaction`。

对照 Rust 篇 3.1 的 handler：

```rust
// 现在的形状（Rust 篇 3.1）
let mut tx = st.pool.begin().await?;
let out = services::checkout(&mut tx, &actor, cmd).await?;
tx.commit().await?;

// 加一层之后。services::checkout 的签名不变。
let out = retry::with_retry(&st.pool, retry::Policy::write(), |conn| {
    services::checkout(conn, &actor, cmd)
}).await?;
```

完整的 handler（可直接对照 Rust 篇 3.1 的 `checkout_handler`）：

```rust
async fn checkout_handler(
    State(st): State<AppState>,
    actor: Actor,
    Json(cmd): Json<CheckoutCmd>,
) -> Result<Json<CheckoutView>, AppError> {
    // 事务边界仍然在 handler，只是从"手写 begin/commit"变成"交给 with_retry"。
    let view = retry::with_retry(&st.pool, retry::Policy::write(), &st.log,
        |conn| services::checkout(conn, &actor, cmd))
    .await?;

    // 【改】业务日志在成功之后记（见第 7.2 节：提交前记会被重试记成多条）
    log::checkout(&actor, &view, &cmd);
    Ok(Json(view))
}
```

**注意这里 `Actor` 与 `cmd` 是被闭包捕获的**：`with_retry` 会**多次调用**这个闭包，所以闭包必须是 `FnMut`（不是 `FnOnce`）——`&actor` 与 `&cmd` 的借用能同时活到闭包的所有调用里，这一点在类型上成立。**但 `cmd` 不能是"只能消费一次"的东西**（比如内部有 `take()`），否则第二次尝试会拿到一个被掏空的 `cmd`。规则：**闭包捕获的上下文必须是只读的，或者每次调用都能重新读一份。**

### 3.2 包装器的实现形状

```rust
// crates/app/src/db/retry.rs
//
// ⚠️ 待验证：本机没有 Rust 工具链，本节的 HRTB（for<'a>）写法与 AsyncFnOnce 的
//    实际可用性我无法编译核对。8.4 列了必须验证的三点。若 AsyncFnOnce 的方案
//    在某版 Rust 上不成立，退路是只保留 TxFn 的“函数指针/具名函数”方案
//    （它对 for<'a> 是原生满足的），代价是 handler 里拿到的上下文要通过一个
//    结构体显式传进去。

use sqlx::{PgConnection, PgPool};
use std::future::Future;
use std::time::Duration;

#[derive(Debug, Clone, Copy)]
pub struct Policy {
    pub max_attempts: u32,          // 总尝试次数，含第一次
    pub base_backoff: Duration,
    pub max_backoff: Duration,
    pub jitter: Jitter,             // Full | None（测试用 None 才可断言）
}

impl Policy {
    pub const fn write() -> Self {
        Self { max_attempts: 3, base_backoff: Duration::from_millis(50),
               max_backoff: Duration::from_millis(400), jitter: Jitter::Full }
    }
}

/// 重试判定的唯一出口。`Retry` 只能由“整笔事务确实没生效”的错误产生（第 2 节）。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum RetryClass {
    Retry,
    Fatal,
}
```

主体：

```rust
/// 重试的观测点。**故意做成注入而不是直接调 logging**，有两个原因：
///   ① 测试要能数"重试了几次"（日志设计 8.4 禁止断言日志条数）；
///   ② `db` 模块不依赖日志模块，依赖方向保持单向。
pub trait RetryObserver: Sync {
    /// `error_class` 是 SQLSTATE 原文（`40P01` / `08006` / …），
    /// 连接阶段的 IO/TLS 失败取不到码时传 `"IO"`，其它未知情况传 `None`。
    ///
    /// **刻意只传一个字符串，不传 `sqlx::Error`**：这样 observer 在类型上就没有
    /// 机会去取 `Display`（可能含 SQL 片段与表名），第 7.1 节的字段白名单由签名守住。
    fn retry(&self, attempt: u32, max_attempts: u32, error_class: Option<&str>, started: Instant);
}

/// 每次尝试：begin → 闭包 → commit。闭包只能看到一个 &mut PgConnection，
/// 因此它无法新开事务、无法拿到 pool（与 Rust 篇 3.1 的意图一致）。
pub async fn with_retry<'p, T, F, Fut, L>(
    pool: &'p PgPool,
    policy: Policy,
    log: &L,
    body: F,
) -> Result<T, AppError>
where
    // HRTB：闭包对任意生命周期都可调用（每次都拿到一个新事务的借用）
    F: for<'c> FnMut(&'c mut PgConnection) -> Fut,
    Fut: Future<Output = Result<T, AppError>>,
    L: RetryObserver,
{
    let started = std::time::Instant::now();
    let mut attempt: u32 = 0;

    loop {
        attempt += 1;

        // ① 取连接 + BEGIN
        //    这一段的失败是“事务从未开始”，是连接类错误的合法重试区间。
        let mut tx = match pool.begin().await {
            Ok(tx) => tx,
            Err(e) if attempt < policy.max_attempts && is_retryable_acquire(&e) => {
                log.retry(attempt, policy.max_attempts, acquire_class(&e), started);
                backoff(policy, attempt).await;
                continue;
            }
            Err(e) => return Err(AppError::from(e)), // 含 PoolTimedOut → Unavailable
        };

        // ② 事务体。&mut *tx 得到 &mut PgConnection（Rust 篇 3.1 的写法）
        let out = match body(&mut *tx).await {
            Ok(v) => v,
            Err(e) if attempt < policy.max_attempts && retry_class(&e) == RetryClass::Retry => {
                drop(tx); // ← 显式回滚：绝不把上一个事务带到下一轮
                log.retry(attempt, policy.max_attempts, sqlstate_of(&e), started);
                backoff(policy, attempt).await;
                continue;
            }
            Err(e) => return Err(e),
        };

        // ③ COMMIT。**提交阶段的失败一律不重试**（见上面的阶段表）。
        return match tx.commit().await {
            Ok(()) => Ok(out),
            Err(e) => Err(AppError::from(e)),
        };
    }
}
```

**"阶段"是这段代码的核心概念，必须先定清楚。** 一次尝试有三个阶段，可重试性各不相同：

| 阶段 | 出现哪些错误 | 可重试 | 原因 |
|---|---|---|---|
| **① 取连接 + `BEGIN`** | `08000` / `08006` / `57P01` / `Io` / `Tls` / `WorkerCrashed`（**都不经过 `retry_class`**，它们是 `sqlx::Error` 而非 `AppError`） | **可** | 一个字节的业务 SQL 都没发出去，服务端不可能留下任何东西。这是**唯一**允许"重连后重放"的区间 |
| **② 事务体** | `40P01` 死锁 | **可** | 数据库已经回滚了整笔（见 2.2） |
| **② 事务体** | `08000` / `08006` / `57P01`（连接在事务中途失效） | **可** | 连接都没了，事务不可能提交成功——postgres 的后端进程一死，它持有的事务必然回滚。所以"重放"是安全的。前提是 `AppError` 保留了原始 SQLSTATE（6.2） |
| **② 事务体** | `57014`（`statement_timeout` / `lock_timeout`）、`25P03`（事务空闲超时） | **不可** | `57014` 意味着"这次执行烧掉的时间已经超过预算"，重试会在同一个预算里再烧一次；`25P03` 意味着"有代码在事务里做了 I/O"，重试会再犯一次（2.4.1、提案 04 第 4 节） |
| **③ `COMMIT`** | 全部（含 `08006`） | **绝对不可** | 我们**不知道**服务端是否已提交：`CommandComplete` 丢在网络上和连接断掉在客户端看起来是同一种失败。重放会写出第二份数据。宁可 500，也不替用户写重复行 |

`is_retryable_acquire(&sqlx::Error)` 是阶段①的判定，它只认"连接还没建立/已失效"这一类（`Io`、`WorkerCrashed`、`Tls`、以及连不上的 class 08），**不认 `PoolTimedOut` / `PoolClosed`**（2.2 已定：这两个不重试，直接 503）。实现时它可能需要解一层 `sqlx::Error::PoolAcquire` 之类的包装才能看到底层的 `Io`/`Database`——**具体形态见第 8 节末的待验证清单第 3 条**。

**`RetryObserver` 的两个实现**：生产上是一个薄结构，内部调 `log::retry(…)`（第 7 节，产出 `kind = "biz.retry"`）；测试里就是 `impl RetryObserver for AtomicU32`（`fetch_add(1)`）。**测试断言重试次数只能走这个注入点，不能去数日志行**（第 7.2 节，日志设计 8.4）。

**四个必须做对的细节**：

1. **`drop(tx)` 而不是继续用 `tx`。** 上一轮的事务必须显式回滚（`Transaction` 的 `Drop` 会 `ROLLBACK`）。忘记这一句、或者试图把 `tx` 提到循环外复用，是这段代码唯一的真实 bug 形态。
2. **`body(&mut *tx)`。** 与 Rust 篇 3.1 的 `&mut *tx` 一致；直接写 `&mut tx` 会因为类型是 `&mut Transaction` 而不匹配 `&mut PgConnection`。
3. **`commit()` 的失败不重试。** 这是全篇最需要克制的一处：提交失败时我们**不知道**服务端是否已经提交（经典的"响应丢包"窗口），重放会产生**第二份**业务数据（第二次出库、第二条 `equipment_history`、编号跳号）。宁可给用户一个 500（他可能再来点一次，而重复点由业务层的约束/幂等键挡住），也不要由框架替他写出重复数据。
   - 与 `pool.begin()` 的失败形成对照：**取连接阶段失败时，一个字节的业务 SQL 都还没发出去，重试是安全的。** 这就是 2.2 里 `Error::Io` 那条"仅建立连接阶段可"的实现含义。
4. **`AppError::from(e)` 里 `PoolTimedOut` 必须映射成 `Unavailable`（503）而不是 `Sqlx`（500）。** 见第 6.2 节。运行时篇 R7.1 已经定了"数据库不可用 → 503"，这一条是它的实现落点。

`retry_class()`（第 2 节那张表的可执行形式）：

```rust
/// Fail-closed：`_` 分支返回 Fatal。新 SQLSTATE 默认不可重试。
///
/// 入参是 **AppError** 而不是 sqlx::Error：事务体里的错误会先经过
/// `From<sqlx::Error>`（第 6.2 节），所以这里看到的是翻译后的形状。
pub fn retry_class(e: &AppError) -> RetryClass {
    match sqlstate(e) {
        // 死锁：数据库已经回滚了整笔（唯一"整笔没生效且值得再试"的数据库错误）
        Some("40P01") => RetryClass::Retry,

        // 08000/08006/57P01 已被翻译成 TemporaryUnavailable，但原始码还在 e 里。
        // **这类只在"事务体阶段"能走到这里**——取连接阶段的同类错误在
        // with_retry 的第 ① 步就被处理掉了（见阶段表）。
        Some("08000" | "08006" | "57P01") => RetryClass::Retry,

        // 57014（statement_timeout / lock_timeout）与 25P03（事务空闲超时）：
        // **都不重试**。前者会在同一个时间预算里再烧一次；后者意味着
        // "有代码在事务里做了 I/O"（提案 04 第 4 节），重试只会再犯一次。
        // 两者都已被翻译成 TemporaryUnavailable，但原始码还在，所以这里能判。
        Some("57014" | "25P03") => RetryClass::Fatal,

        // 确定性冲突：**绝不重试**，否则 409 会变成 503
        Some("23P01" | "23505") => RetryClass::Fatal,

        // R4.3 下不可达；出现即说明隔离级别被改了，让它以 500 炸出来
        Some("40001") => RetryClass::Fatal,

        // Fail-closed：没列到的一律不重试（含拿不到 SQLSTATE 的情况）
        _ => RetryClass::Fatal,
    }
}
```

**`sqlstate()` 必须能穿透翻译层。** 这是本节对主稿第 8 节提出的**第二处改动**（第一处是加变体）：`AppError::Sqlx` 现在需要带上原始 SQLSTATE，否则 `retry_class` 在错误被翻译成 `TemporaryUnavailable` 之后就再也分不出"`08006` 可重试"与"`57014`/`25P03` 不可重试"。改动形状见第 6.2 节末尾。

**`retry_class` 与 `is_retryable_acquire` 是两个函数，不要合并**：前者收 `AppError`（已经过翻译层），后者收原始 `sqlx::Error`（来自 `pool.begin()`，还没经过翻译层）。合并它们会要求 `with_retry` 在取连接那一段手写一遍 SQLSTATE 匹配——那正是第 2.3 节要避免的"第二处分类逻辑"。

### 3.3 与 `&mut PgConnection` 约定的三条边界（要写进 Rust 篇 3.4）

| 项 | 约定 |
|---|---|
| 服务函数签名 | **不变**。继续 `conn: &mut PgConnection`。重试不改变它 |
| handler 里的事务边界 | 从"手写 `begin`/`commit`"变成"交给 `with_retry`"，但**仍然是 handler 层**，符合 Rust 篇 3.1"handler 管事务边界" |
| 例外：一个 handler 里需要多个事务时 | `with_retry` 管**一个**事务。若将来出现"两个事务 + 中间有 I/O"的 handler（当前没有），第二个事务用自己的 `with_retry`，不要试图把它们塞进一个闭包 |
| `jobs` | `jobs` 直接复用 `app::db::retry`。job 的重试是**下次 cron**（天然幂等），因此 job 内不启用 `with_retry` 的事务重试——除非某个 job 的写路径与 API 共用函数 |
| 不是所有 handler 都套 | 只套会写库的。**读接口不套**：读不产生"重跑一遍"的语义问题，但也几乎没有重试价值，套上去只是给读路径增加等待（且读接口的 500 不需要重试掩盖） |

---

## 4. 幂等性

### 4.1 先把两个不同的"重复"分开（这一节最容易混淆）

| | 机制 | 会不会重复？ |
|---|---|---|
| **事务重放**（本文第 3 节的 `with_retry`） | 上一笔事务被数据库回滚（死锁/连接失效），**然后**才重跑 | **不会。** 因为重跑的触发条件本身就保证"上一次尝试没有留下任何东西"。`next_seq` 的 `UPDATE`、`equipment_history` 的行、`occupancy` 的行，全都随事务回滚消失 |
| **客户端重发**（弱网下用户手点第二次、或客户端超时自动重试） | 第一次**已经提交成功**，客户端没收到响应 | **会。** 这才是重复数据的来源，靠 `Idempotency-Key`（运行时篇 R4.4）或业务约束挡 |

**结论：本文引入的重试不会新增任何重复数据的风险。** 需要 `Idempotency-Key` 的是另一个问题（第 4.4 节）。

### 4.2 "重放一次事务"的安全性，按操作分类

| 操作 | 事务重放安全？ | 理由 |
|---|---|---|
| 所有读 | 安全且无意义 | 不重试 |
| `equipment` 描述性字段编辑（PATCH） | **安全** | 绝对赋值（`SET price_cents = $1`），重放幂等。主稿 5.5 的后写覆盖在这个前提下不会变 |
| 状态流转（出库、归还、送修、完成、停用、启用、出售、归档、恢复） | **安全** | ① 行锁 + `from == to` 返回 Ok 的幂等状态机（Rust 篇 4 第 2 点）；② 即使状态机放行，`occupancy` 的排他约束会拦住第二次同区间插入 |
| 归还（逐件） | **安全** | 第二笔事务会重新读到 `returned_at` 已非空（若第一笔已提交）或看到第一笔已回滚的行 |
| 编号分配（`category.next_seq`） | **安全** | 计数器更新在事务内，回滚即不跳号。这正是主稿 3.5 选计数器而不是 `max(code)+1` 的另一个副产品 |
| CSV 导入提交（1000 件、单事务、无 savepoint，主稿 C4） | **安全** | 整批回滚；重放从零开始。**而且它是最需要重试的操作**：一个事务里几千次往返，锁竞争窗口最大 |
| 照片上传 | **安全** | 内容寻址 + 路径已存在即跳过（主稿 6.2/6.3） |
| 登录（写 `app_user_session`） | **安全** | 重放会产生第二行会话，但第一行随回滚消失；且会话本身允许多行 |
| 邀请消费（`invite.consumed_at`） | **安全** | 一次性消费在同一事务内；回滚后重放是同一个"未消费"状态 |
| 审批（写多条 `occupancy` + 改 `request_item.approved`） | **安全，但有一个边界**：若重试跨过业务日期的零点，`during` 会按新的"今天"重新计算 | 见 4.3 |

### 4.3 两个诚实的边界（不要假装没有）

1. **依赖"今天"的计算在跨零点重试时会变。** `during` 起点用 `business_today()`（运行时篇 R2.3）或 `now()` 的地方（如「办理交付」把 `during` 起点改为实际出库日、归还写 `returned_at`），一次跨零点的重试会得到不同的结果。**这不是重复数据问题，是极小概率的语义偏移**（需要重试恰好横跨本地午夜，且这段代码恰好用了"今天"）。
   - 立场：**接受**，不为它引入"把时间冻结进重试上下文"的机制——那会让一个 3 行的包装器变成一个有状态的执行框架，代价远大于收益。
   - 必须做的只有一件事：**所有"今天"都走 `business_today(cfg)`**（R2.3 已有此约定），不要把 `now()` 散在写路径里。
2. **`RowNotFound` 与"行在两次尝试之间被删了"。** 重试时数据可能真的变了（比如器材被归档、`request_item` 被级联删掉）。这类情况下第二次会得到另一个错误。这是**正确行为**，不是缺陷——重试永远不能承诺"两次结果相同"，只能承诺"不会留下半成品"。

### 4.4 没有 `Idempotency-Key` 的创建型操作：会不会产生重复行？

**会被客户端重发产生重复，但本文的重试不会。** 逐操作：

| 操作 | `Idempotency-Key`（R4.4 定的三处） | 客户端重发两次的后果 | 评估 |
|---|---|---|---|
| 出库 / 交付 | ✅ 有 | 第二次被**排他约束**拦住 → 409 `RESERVED` | **已经被约束兜住**，幂等键是把它变成 200 的体验优化 |
| 照片上传 | ✅ 有（且本来内容寻址幂等，所以实际不依赖这个头） | 直接返回同一引用 | 安全 |
| CSV 导入提交 | ✅ 有 | 第二次可能产生重复器材（主稿 13.9 的重复判定**尚未拍板**） | **这是三个里唯一"重发真会造成损害"的**，幂等键必需要有 |
| 建档（单件） | ❌ 无 | 产生**两条** `equipment` 行 | 用户可见的重复。缓解：`(workspace_id, code)` 唯一约束在**编号相同**时拦下；但编号留空走自动生成时**不会**撞车，于是真的建出两件。**结论：这是本文发现的一个真实缺口，见 4.5** |
| 申请提交 | ❌ 无 | 产生两条 `request` | 重复申请，可人工拒绝，损害可控 |
| 项目创建 | ❌ 无 | `project.name` 唯一约束（主稿 3.3）拦下第二次 → 409 | 已兜住 |
| 分类创建 | ❌ 无 | `(workspace_id, name)` 唯一约束拦下 | 已兜住 |
| 送修 | ❌ 无 | `maintenance_one_open` 部分唯一索引拦下第二次 | 已兜住 |
| 账号创建（走邀请） | ❌ 无 | 邀请一次性消费（`consumed_at`）拦下第二次 | 已兜住 |

### 4.5 由本文指出的一个缺口（建议，不阻塞实现）

**建档是唯一"重发会静默产生重复行且没有任何约束拦住"的创建型操作**（编号留空时，两次自动生成会得到两个不同编号，因此不会撞唯一约束）。

**建议：把 `Idempotency-Key` 从三处扩到四处，加上建档。** 改动只是在建档 handler 里复用已经为出库实现的那套机制（R4.4 说"不要一开始就给所有接口加"，建档恰好落在"重试会造成重复数据"这个标准里，而项目名/分类名/送修都被唯一约束兜住了，所以它是三处之外唯一符合标准的一个）。

理由写在这里以免被重新讨论：**本文的重试策略不需要它；需要它的是 R4.4 的客户端重发场景。** 这一节之所以要点出建档，是因为"重复建档会撞 `23505` 被拦下"这个直觉只有**编号相同**时才成立——编号留空走自动生成时，两次请求会得到两个不同编号，唯一约束不会响。

> **与"不可重试"的相互作用（要写进 R4.4 的实现备注）**：建档一旦有了幂等键，重复请求会在**进入事务之前**被幂等表拦下。这一步是必要的，因为**不能指望 `23505` 或重试来兜住重复建档**——重试已经被本文明确禁止处理 `23505`。

---

## 5. 退避参数

### 5.1 具体值

| 参数 | 值 | 依据 |
|---|---|---|
| `max_attempts`（总尝试次数，含第一次） | **3** | 死锁的重试收益衰减极快：第一次重试成功率最高，第二次之后基本说明这不是偶发竞争。3 次以上对真实故障率没有帮助，只增加长尾延迟 |
| 首次退避基数 `base_backoff` | **50 ms** | 死锁的持续时间是"另一个事务剩余的持锁时间"——在本系统的写路径里是几十毫秒量级。50 ms 起步足够让对手提交 |
| 退避上限 `max_backoff` | **400 ms** | 与 `max_attempts=3` 配合，第 2 次尝试前最多等 100 ms、第 3 次前最多等 200 ms。**上限 400 ms 在当前参数下不会被触及**，它是给"将来把 `max_attempts` 调大"留的刹车 |
| 抖动 | **Full jitter**（`sleep = uniform(0, 计算值)`），**生产默认开** | 没有抖动时，一次 `pg_terminate_backend`/failover 会让同时在途的 N 个请求在同一毫秒重试，形成对刚恢复的数据库的尖峰。Full jitter 是 AWS 那篇经典结论里效果最好的一种，实现只有一行 |
| 测试用 | `Jitter::None`（固定延迟） | 让第 8 节的测试可以断言次数而不用等随机时间 |
| 重试总睡眠（`max_attempts = 3`） | **≤ 300 ms**（100 + 200；开 full jitter 后期望约 150 ms） | 见 5.2 |

```text
sleep(n) = min(base_backoff × 2^(n-1), max_backoff)
          然后（若开启抖动）sleep = uniform(0, sleep)
```

### 5.2 为什么不用无上限的指数退避

1. **在请求路径上，"重试多久"和"用户等多久"是同一笔预算。** 运行时篇 R1.10 的 grace drain 上限是 **15 秒**。一次部署应该在前端还没超时前完成 drain；无上限重试会让"部署"表现为一批用户各自等 15 秒然后收到一个 503。
2. **无上限退避的失败形状是"最坏情况下才出现的长尾"**，本地测不出来、压测也难复现，只会在真实故障那天同时出现在所有请求上。
3. **它把一个确定性的错误变成不确定的等待。** 如果 3 次都没成功，第 4 次大概率也不会成功；此时**快速失败、把状态码交给客户端**（503 + `Retry-After`）比服务端自己耗在里面更有用——客户端的重试可以发生在几秒后，而不是占用一个连接。
4. **无上限重试和"雪崩"是同一件事。** 数据库慢 → 请求都卡在重试 → 连接池被占满 → 新请求 `PoolTimedOut` → 更要重试。有上限 + 不重试池超时（2.2）是防止这条闭环的两个必要条件。

5. **`lock_timeout` 的失败模式要求抖动**（提案 04 的 6.6）：一批请求同时等同一个锁、**同时在 5 秒后超时**。这是**同步**的失败模式，所以它们的重试也会同步——没有 full jitter，就会造出下一轮同样同步的拥塞。**这一条与"`57014` 判不判可重试"无关**：只要有任何一类错误会被重试，抖动就是必需品。

**换句话说：本文的重试是为了吸收"毫秒级的基础设施抖动"，不是为了代替"客户端稍后重试"。** 前者服务端做，后者交给 503 + `Retry-After`。

### 5.3 配置位置（建议并入运行时篇 R2.1）

```rust
// Config 增加（默认值即上面的参数；正常情况下不需要调）
//
// ⚠️ 前三个字段与 [提案 04 · 连接池与超时](04_pool_and_timeouts.md) 的 5.3 是**同一套**，
//    不要出现两个取连接超时配置（04 已经用了这个名字）。这里只列一次。
pub pool_acquire_timeout_ms: u64,     // 默认 5000。与 04 共用，**不要重复定义**
pub tx_retry_max_attempts: u32,       // 默认 3
pub tx_retry_base_backoff_ms: u64,    // 默认 50
pub tx_retry_max_backoff_ms: u64,     // 默认 400
```

**为什么给配置项但仍给死默认值**：这三个值在真实故障面前是要调的（"死锁太频繁"应该先修取锁顺序，而不是调大次数）。把它们放在配置里，是为了让"调它"成为一个显式动作、能被写进运维记录，而不是改一行代码重新发布。

**取连接超时必须是 5 秒，而且它与本提案有两个交界**（这一节原来标为"待验证"，提案 04 已给出结论）：

1. **它决定 `PoolTimedOut` 多快变成 503。** sqlx 的默认值（常见说法 30 s）意味着池耗尽的请求先卡 30 秒——那不是"稍后重试"，是"用户已经放弃"。提案 04 的 1 节把它定为 5 s。
2. **它必须 < R1.10 的 15 s drain 窗口**，否则停机时排队的请求还在等连接、进程就要退了，于是**每个在途请求都变成"停机时被砍掉的 500"**——正是 R1.10 要避免的东西。本提案 5.2 的第 1 点与它是同一条约束的两个面：**重试总睡眠（≤300 ms）也必须 < 15 s，而且要和 `acquire_timeout` 一起算进同一个请求预算**（5 s + 300 ms + 事务时间 ≪ 15 s，余量充足）。

**边界判断**：无论 `max_attempts` 调成多少，**"取连接超时 + 重试总睡眠 + 最坏合法事务时间" 必须 < R1.10 的 drain 超时**（15 s）。这条要写成配置校验里的一个断言，否则某天有人把它调到 50 次，症状会是"部署时必有一批请求 500"。

---

## 6. 重试耗尽之后的响应

### 6.1 返回什么

| 项 | 值 |
|---|---|
| HTTP 状态码 | **503 Service Unavailable** |
| `DomainError` | 新增变体 `DomainError::TemporaryUnavailable` |
| 中文文案（`message`，直接进表单内红条） | **"系统繁忙，请稍后重试"** |
| 错误体 `code` | `TEMPORARILY_UNAVAILABLE` |
| 响应头 | **`Retry-After: 1`**（秒）。这是 503 的既有约定，客户端/代理不需要额外契约就能理解 |
| 日志 | 一条 `http.end`（ERROR，`status=503`）；重试过程本身由 `biz.retry` 记录（第 7 节） |

**文案的分寸**：主稿第 8 节的文案风格是"直接给用户看的人话"（`"不能从“维修”改为“出库”"`）。但这条有个特殊约束——**它是由基础设施原因产生的，而且要求用户做一个动作（稍后重试）**。所以它应该是祈使句、承诺"值不值得再试"，而不是陈述句。

一句话表：**文案要能让用户自己决定"再点一次"，而不是让他截图去问"这是不是坏了"。**

### 6.2 需要给 `AppError` / `DomainError` 加什么（以及主稿第 8 节怎么改）

**建议：加一个 `DomainError` 变体，改一句话，不改结构。**

```rust
// domain/error.rs —— 【改】主稿第 8 节的 DomainError 增加一个变体
#[derive(Debug, thiserror::Error)]
pub enum DomainError {
    // … 现有七个变体全部不动 …
    /// 基础设施暂时不可用（重试耗尽 / 连接池耗尽 / 取连接失败）。
    /// 与其它 DomainError 的区别：它不是"这个请求的业务前提不成立"，
    /// 而是"现在别试，稍后可以"。文案必须让用户知道可以重试。
    #[error("系统繁忙，请稍后重试")]
    TemporaryUnavailable,
}
```

**为什么放在 `DomainError` 而不是只放 `AppError`**：主稿第 8 节的硬规则是**`Display` 输出直接是给用户看的中文**，`DomainError` 是这条规则的唯一落点。放这里可以让"文案只有一处"这条保持成立，也能被领域测试直接断言这句话。它确实不是"领域规则"，它是一个**可知的、非 5xx-默认的出口**——这也是本文第 6.3 节的 500/503 分流所依赖的东西。

```rust
// app/error.rs —— 【改】主稿第 8 节的 AppError：给 Sqlx 带上原始 SQLSTATE
//
// 为什么需要它：错误一旦被翻译成 `DomainError::TemporaryUnavailable`，
// 就再也没人分得出它原本是 08006（可重试）还是 57014（不可重试）。
// 保留一个 `sqlstate` 字段是这类分类逻辑的唯一无损做法，
// 也让第 7 节的日志能记 SQLSTATE 而不是错误的 Display。
pub enum AppError {
    #[error(transparent)] Domain(#[from] DomainError),
    #[error(transparent)] Validation(#[from] garde::Report),
    /// 从 `sqlx::Error` **翻译构造**。不要用 `#[from]`——
    /// 那会绕过第 6.2 节的 SQLSTATE 翻译，让 23P01 重新变成 500。
    #[error(transparent)]
    Sqlx { source: sqlx::Error, sqlstate: Option<&'static str> },
    #[error(transparent)] Io(#[from] std::io::Error),
    #[error("上传失败：{stage}")]
    Upload { stage: UploadStage, source: anyhow::Error },
    // … 其余变体（Auth / Forbidden 等）不动 …
}
```

> ⚠️ **这是一个结构性改动，值得单独评审。** 它比"只加一个变体"代价大，但它是第 3.2 节
> `retry_class` 能正确区分 08006 与 57014 的前提。**若评审不接受这个改动**，退路是：
> `AppError::Sqlx(sqlx::Error)` 保持不变，`retry_class` 只按 `sqlx::Error` 的变体与
> `db.code()` 判定，并接受"08006 在事务体阶段不可重试"这个更保守的立场
> （即只保留取连接阶段这一条重试通道）。**两条路都成立，选一条并在文档里写明，不要两套并存。**

```rust
// app/error.rs —— 【改】主稿第 8 节的 From<sqlx::Error>
impl From<sqlx::Error> for AppError {
    fn from(e: sqlx::Error) -> Self {
        let code = match &e {
            sqlx::Error::Database(db) => db.code().map(|c| intern_sqlstate(&c)),
            _ => None,
        };

        // 【新增】连接池耗尽 / 关闭：直接是"稍后重试"，不是"出错了"。
        // 这条把运行时篇 R7.1（数据库不可用 → 503）落成实现。
        if matches!(e, sqlx::Error::PoolTimedOut | sqlx::Error::PoolClosed) {
            return AppError::Domain(DomainError::TemporaryUnavailable);
        }

        if let Some(c) = code {
            match c {
                // 【新增】三种"整笔事务没生效"的连接类错误。
                // 它们能被翻译成本变体，正是因为 with_retry 只对这些重试，
                // 而重试耗尽后需要给用户一个"可以再来一次"的出口。
                "08000" | "08006" | "57P01" => {
                    return AppError::Domain(DomainError::TemporaryUnavailable)
                }
                // 【新增】两个超时（提案 04 的 6.2 要求它们落进同一个出口）：
                //   57014 = statement_timeout / lock_timeout（两者共用这个码）
                //   25P03 = idle_in_transaction_session_timeout（服务端会关掉连接）
                // 它们都返回 503，但 retry_class 会把它们判成 Fatal（不重试）。
                "57014" | "25P03" => {
                    return AppError::Domain(DomainError::TemporaryUnavailable)
                }

                // —— 以下两行是主稿原有内容，一字不动 ——
                "23P01" => return AppError::Domain(DomainError::Conflict),
                "23505" => {
                    return match e.as_database_error().and_then(|db| db.constraint()) {
                        Some("equipment_code_uniq") => {
                            AppError::Domain(DomainError::DuplicateCode { code: /* … */ })
                        }
                        Some("app_user_one_owner") => AppError::Domain(DomainError::OwnerImmutable),
                        _ => AppError::Sqlx { source: e, sqlstate: code },
                    }
                }
                // 【建议】40001 不翻译：落到 Sqlx → 500，并记 ERROR，见 2.4
                _ => {}
            }
        }

        AppError::Sqlx { source: e, sqlstate: code }
    }
}
```

`sqlstate()` 就是上面那个字段的读取器（`AppError::Sqlx { sqlstate, .. } => *sqlstate`）。`intern_sqlstate` 把 SQLSTATE 变成 `&'static str`（五个字符的码一共就几十个，一个小 `match` 或 `OnceLock` 表即可）——**不要用 `Box<str>` 存它，那会让 `AppError` 的 `Clone` 与匹配都变麻烦**。

**但这里有一个洞，必须补上**：`08000`/`08006`/`57P01` 与两个超时码（`57014`/`25P03`）被翻译成了 `DomainError::TemporaryUnavailable`，于是 `AppError::Sqlx` 那个字段就不存在了——第 3.2 节的 `retry_class` 需要 `sqlstate()` 在这种情况下仍拿得到原始码（否则它分不出"`08006` 可重试"与"`57014`/`25P03` 不可重试"）。两个补法，**选一个并在文档里写明**：

```rust
// 补法一（推荐）：变体自带一个可选的原始 SQLSTATE。
//   加一个字段就要改所有构造点，所以只给"需要区分"的那一个变体加。
#[error("系统繁忙，请稍后重试")]
TemporaryUnavailable { sqlstate: Option<&'static str> },

// sqlstate() 于是有两个来源：
//   AppError::Sqlx { sqlstate, .. } => *sqlstate,
//   AppError::Domain(DomainError::TemporaryUnavailable { sqlstate }) => *sqlstate,
```

```rust
// 补法二：不加字段，把原始 SQLSTATE 放进 IntoResponse 之前的某个 task-local
// 或 request extension 里。**不推荐**——它把"分类"和"响应"耦合起来，
// 而且 task-local 在非请求上下文（jobs）里没有值，行为会分叉。
```

补法一的代价很小（只影响一个变体的构造点，而且只有 6.2 这一处构造它），收益是**分类逻辑仍然只有一处输入**。本文第 3.2 节的 `retry_class` 按补法一写。

**唯一一处必须新增的分支**：`DomainError::Conflict` 需要有人写 `Display` / 翻译，因为主稿第 8 节的枚举里**没有它**（`From` 里用了它，但枚举定义里漏了）。建议：

```rust
// 【改】主稿第 8 节的 DomainError，补上已在 From 里使用但未定义的变体
#[error("这段时间已被占用，请更换日期或器材")]
Conflict,
```

**注意这两件事的顺序**：`Conflict` 的文案必须与 `Reserved` **区分开**。`Reserved` 是预检查拦下的、能点名"被哪个项目占用"的人话；`Conflict` 是预检查漏了、约束兜底的兜底文案（日志设计 6.1 说这个场景"说明预检查有 bug"，见 8.2 V6）。**它们必须是两句不同的话**，否则"预检查有 bug"这件事在用户看到的界面里被抹平了，只能靠日志发现——那是对的，用户不该看到基础设施细节，但两句话的价值在于：用户对 `Reserved` 的反应是"换日期"，对 `Conflict` 的反应也是"换日期"，所以文案可以近似，但日志必须分开（`biz.reject` vs `biz.conflict`）。

**`Retry-After` 怎么出去**：`IntoResponse` 里 `TemporaryUnavailable` 与 `PoolTimedOut` 这两个分支除了状态码，还要 `headers.insert(RETRY_AFTER, HeaderValue::from_static("1"))`。这是这一节唯一需要在 Rust 篇 5.2 那个 `IntoResponse` 上加的代码（一个 `if matches!(…)` 里的两行）。

> **待验证**：`sqlx::Error::PoolClosed` 与 `PoolTimedOut` 的变体名/存在性，以及是否还有 `PoolAcquire` 之类的包装变体（见第 2.2 B 表上方那句）。

### 6.3 主稿 7.3 状态码矩阵怎么改

**原文**：

| 情况 | 码 |
|---|---|
| 限流 / 事务重试耗尽 | 429 / 503 |

**改成三行**：

| 情况 | 码 | 说明 |
|---|---|---|
| 限流（限流器触发） | **429** + `Retry-After` | 保持原义，与重试无关 |
| **事务重试耗尽** | **503** + `Retry-After: 1` | 整笔事务重跑了 `max_attempts` 次仍未成功（死锁/连接失效） |
| **数据库不可用 / 连接池耗尽** | **503** + `Retry-After: 1` | 运行时篇 R7.1 已定的行为，在这里落成同一行 |

**为什么不用 429 表示重试耗尽**：429 的既有语义是"你（客户端）请求太频繁"，前端/网关对它的标准动作是**降速**（`Retry-After` 之外的退避）。而重试耗尽的语义是"我们（服务端）现在做不了，稍后再来"——标准动作是**原样重试**。两者混淆的后果是前端给用户显示"操作过于频繁"，而用户其实什么都没做快。**一次误报的限流提示会让用户以为是自己操作错误。**

**为什么 503 要和 500 分开**（运行时篇 R7.1 已有一半的答案，这里是补全）：

| | 500 | 503 |
|---|---|---|
| 含义 | 我们出错了，这是 bug | 现在做不了，稍后可以 |
| 前端动作 | 报 bug / 显示 requestId | 提示"稍后重试"，或自动退避重试 |
| 是否进 R1.8 的 5xx 告警 | 是（需要人看） | **是**（也要人看，但判断不同：503 成片出现 = 数据库/连接池问题，不是代码 bug） |

**注意 503 仍然属于 5xx，仍然要进 R1.8 的告警口径。** 它的价值不在"不告警"，而在**让前端知道该重试**。

### 6.4 服务端不自己重试、交给客户端的那几种

| 错误 | 服务端 | 客户端看到 |
|---|---|---|
| `PoolTimedOut`（提案 04 的 `acquire_timeout = 5s`） | 立刻返回，不重试（2.2） | 503 + `Retry-After` |
| `57014`（`statement_timeout` 15s / `lock_timeout` 5s） | 不重试（2.2、2.4.1） | 503 + `Retry-After` |
| `25P03`（`idle_in_transaction_session_timeout` 2s） | 不重试，且记 ERROR（它意味着有人违反"事务里不做 I/O"） | 503 + `Retry-After` |
| 重试 3 次后仍死锁 | 放弃 | 503 + `Retry-After` + ERROR 日志（第 7.3 节） |
| `23P01` / `23505` | **不重试**，业务错误 | 409（不是 503），见 2.3 |

### 6.5 要加进运行时篇 R7.5 验收清单的两条

现有清单里有一条"**手工触发一次 `23P01`，确认返回 409 + 中文人话，不是 500**"。它守的是翻译层，但守不住**重试层**——一个把 `23P01` 误判成可重试的实现，照样会让那条验收通过（只是慢了几百毫秒）。补两条：

```
- [ ] 手工触发一次 `23P01`，断言**只发生一次尝试**（日志里没有 `biz.retry`），
      且最终状态码是 409 而不是 503
- [ ] 手工制造一次死锁（两个按相反顺序取锁的事务），断言其中一个请求
      **成功返回 200**，且日志里能搜到带 `error_class=40P01` 的 `biz.retry`
```

---

## 7. 日志

### 7.1 每次重试都记，一条，`kind = "biz.retry"`

**新增一个 `kind`**（日志设计第 6 节说得很明确：新增 `kind` 必须同时更新那张表和 8.3 的白名单测试，这是有意的摩擦）。它是**一个**新 `kind`，不是每次尝试一条业务日志。

```json
{"ts":"2026-09-30T06:22:31.650Z","level":"WARN","kind":"biz.retry",
 "msg":"数据库死锁，正在重试（第 2/3 次）",
 "request_id":"01J8ZQ7K3M9WX2P4","route":"/api/v1/equipment/checkout",
 "actor_id":"7","role":"ADMIN",
 "data":{"attempt":2,"max_attempts":3,"error_class":"40P01",
         "error_kind":"DEADLOCK","elapsed_ms":52}}
```

| 字段 | 位置 | 类型 | 取值 |
|---|---|---|---|
| `attempt` | `data` | number | 本次失败的尝试序号（1-based） |
| `max_attempts` | `data` | number | 上限，让人一眼看出还剩几次 |
| `error_class` | `data` | string | **SQLSTATE 原文**（`40P01` / `08006` / `57P01`），连接类错误填 `IO` |
| `error_kind` | `data` | string | `DEADLOCK` / `CONNECTION` / `IO`。给不想记 SQLSTATE 的人读 |
| `elapsed_ms` | `data` | number | 从第一次尝试开始到现在 |

**`level`：WARN（全部）。** 理由：

- 重试**不是** `biz.reject` 那种"系统在正常工作"的业务结果（日志设计 4.2 / 运行时篇 R1.4 明确要求那类是 INFO）。它意味着**基础设施出了一次小故障**。
- 也不该是 ERROR：用户最终付了 200，把它记成 ERROR 会污染"错误率"这个信号（R1.4 明确禁止把正常流量记成 ERROR 的同一逻辑）。
- WARN 的定位正好：**出现一次值得看一眼**（死锁 = 有人按不同的顺序取锁了；连接失效 = 数据库在上一次部署/重启；两者都是运维信息），但**不构成告警**。

**为什么 `error_class` 记 SQLSTATE 而不是错误的 `Display`**：

1. `sqlx::Error` 的 `Display` 可能含 SQL 片段/表名/参数值，**日志设计 3.3 明确不放自由文本**，运行时篇 R1.7 更是禁止整体记请求体；
2. SQLSTATE 是**封闭、可聚合**的值（`jq 'select(.data.error_class == "40P01")'` 能直接数出死锁频率），而且正是运维要看的那个信息（R7.3 每周检查 `23P01` 的同一种用法）；
3. 它**不含任何用户数据**，可以安全地长期保留。

**为什么不记整个 `AppError` 到 `error` 字段**：顶层白名单里 `error` 的契约是 `{ code, kind, message }`（日志设计 3.1），那是给**最终失败**用的。中间的重试不是 `error`，是一个 `data` 事件。

### 7.2 与"不要断言日志条数"（日志设计 8.4）保持一致

日志设计 8.4 说：**"不要断言日志条数。数量会随实现细节变化（比如事务重试），而字段契约才是要守的东西。"**

**本文的做法与这句话完全一致，并且是它的具体来源**：

1. **测试一律不断言日志行数。** 第 8 节的测试改用**可注入的计数器**（`with_retry` 接收一个 `RetryObserver`，测试传一个 `AtomicU32`），来断言"重试发生了 N 次"。**这是本文特意选择注入而非解析日志的唯一理由。**
2. **业务日志的"一次操作一条"要维持住，但它的实现方式变了。** Rust 篇 3.2 第 7 步（`log::checkout(…)` 在提交前调用）在加了重试之后会**每次尝试各记一条**——3 次尝试 = 3 条 `biz.checkout`，而其中前两条对应的事务都回滚了。**这违反"日志不能记不存在的事实"（运行时篇 R1.1：日志不是事实来源，但也不该记假事实）。**
   - **改法**：`biz.*` 业务事件的记录位置从"服务函数末尾（提交前）"移到"**`tx.commit()` 成功之后**"。即 `with_retry` 的闭包只做事务，业务日志由调用方在拿到 `Ok` 之后记。
   - 这与 Rust 篇 3.2 那段"日志在提交前记、接受不一致"的取舍**需要修正**：那段取舍的前提是"提交后服务层已经拿不到 `&mut PgConnection` 了"——但业务日志本来就不需要连接，它需要的是**视图对象**（`CheckoutView`），而视图已经作为闭包的返回值出来了。**重试把"提交前记"这个折衷的理由消掉了。**
   - **建议**：Rust 篇 3.2 第 7 步改为"**在 `with_retry` 返回 `Ok` 之后记**，用返回的 view"。这样 3 次尝试只产生 1 条 `biz.checkout`，与"一次成功的操作一条日志"的契约一致，也让 8.4 的"不要数条数"更容易被遵守。
   - **`biz.reject`（业务拒绝）不需要跟着改位置，而且原因值得记下来**：业务拒绝是**在事务体里**返回 `Err` 时即记录（Rust 篇 3.2 第 3 步的 `log::reject_reserved`）。它天然只记一次——因为**业务错误是 `Fatal`，永不重试**（第 2 节）。所以"3 次尝试 = 3 条日志"这个问题在 `biz.reject` 上不存在。
     - 代价是它仍然有"记了但事务回滚了"的窗口（Rust 篇 3.2 已接受这一点）。**这正是"约束不可重试"这条规则的一个附带好处**：`biz.reject` 的记录次数与冲突次数严格一一对应，不会被重试放大。

3. **`http.end` 仍然每个请求恰好一条**（在 span 关闭时发），不受重试影响。**这一点很重要**：R1.8 的"5xx 请求数"是按 `http.end` 的 `status` 聚合的，如果重试给 `http.end` 加了噪声，那个信号就不可用了。

### 7.3 运维侧：`23P01` 的模式在重试策略下不变

运行时篇 R7.3 每周检查"`23P01` 告警有没有出现过（出现过就是预检查有漏）"，日志设计 6.1 给了 `biz.conflict`（WARN）。

**本文的规则让这个模式继续成立**：`23P01` **不可重试**，所以

- 一次冲突 = 一条 `biz.conflict`（不会被重试放大成 3 条）；
- 也不会出现"重试了 3 次之后才报冲突"这种把 WARN 变成 ERROR 的路径。

如果本文允许重试 `23P01`，这条每周检查会立刻失效（同一缺陷被记 3 次，而且最后一次是 ERROR）——**这是"约束不可重试"这条规则的第二个独立理由**。

**反过来，`biz.retry` 本身也应该进 R7.3 的每周检查清单**：

| 频率 | 加什么 |
|---|---|
| 每周 | 看有没有 `biz.retry`：`40P01` 出现 = 有条代码路径的取锁顺序与主稿 5.2/5.3 不一致；`08006`/`57P01` 出现 = 数据库在上周重启过（结合部署记录看） |

**"死锁应该按周看，不是按次告警"**：主稿 5.2 说得很清楚——"锁的排序规则一旦漏了，得到的死锁比不用锁更难查"。本文的重试**会让死锁从"用户看到的 500"变成"日志里的 WARN"**，这是一个必要的取舍（可用性优先），代价正是它变得更容易被忽略。所以 `biz.retry` + R7.3 的每周一眼，是这条重试的配套义务，不是可选项。

**重试 3 次后仍死锁 = ERROR。** 这一条不需要新 `kind`：`http.end` 的 `status=503` 就是 ERROR（6.3），并且此时日志里已经有 2 条 `biz.retry` 在同一 `request_id` 下可直接串起来。**不需要为它设计第三个事件。**

### 7.4 要改日志设计的两个地方

| # | 位置 | 改什么 |
|---|---|---|
| 1 | 第 6 节事件清单 | 加一行：`biz.retry` / **WARN** / 每次事务重试一次 / `attempt`, `max_attempts`, `error_class`, `error_kind`, `elapsed_ms` |
| 2 | 第 8.3 节白名单测试的覆盖面 | 说明代表性流程里要包含"一次会重试的路径"（用可注入的故障连接/人工制造一次死锁），否则新 `kind` 的字段不会被这条测试覆盖 |

**不需要改的**：第 3.1 节的顶层字段白名单（`kind` 已在表里）、第 3.3 节的 `data` 深度 ≤ 2 的限制（新事件正好 1 层）。

---

## 8. 测试

编号接运行时篇 R3.2 的 #1–#12 与 Rust 篇 V1–V2，这里给 **V3–V7**。全部进 Rust 篇 9.3 的汇总表。

### V3（最重要）：约束违例绝不被重试

这是"409 不能被变成 503"这条规则的唯一守门测试。

```rust
#[sqlx::test(migrations = "../../migrations")]
async fn exclusion_violation_is_never_retried(pool: PgPool) {
    // 造一条已存在的 occupancy，然后让 with_retry 的事务体去插一个重叠区间。
    // 预检查被绕过（直接插），于是必然触发 23P01。
    // body 每次尝试都必然撞 23P01（这里把事务边界交给 with_retry，
    // 所以闭包里不出现 begin/commit —— 见第 3 节）
    let counter = Arc::new(AtomicU32::new(0));
    let mut f = |conn: &mut PgConnection| async {
        db::occupancy::insert(conn, EQ, during, OccupancyKind::Checkout, None, None).await?;
        Ok::<_, AppError>(())
    };
    // ⚠️ 闭包的具体写法（HRTB / 捕获）见第 8 节末的待验证清单第 1–2 条；
    //    这条测试的要点是"每次尝试都必然撞约束"
    let err = with_retry(&pool, Policy::write(), &counter, f).await.unwrap_err();

    assert_eq!(counter.load(SeqCst), 1, "23P01 必须只尝试一次");
    assert!(matches!(err, AppError::Domain(DomainError::Conflict)),
            "必须仍是 409 的形状，不能被 retry-exhausted 吞成 503");
    assert_eq!(status_of(err), StatusCode::CONFLICT);
}
```

**对 `23505` 再写一份**（`equipment_code_uniq`）：断言尝试次数为 1，且结果是 `DuplicateCode`（409），**不是** `TemporaryUnavailable`（503）。

> 这两条测试的断言方向是**单向**的：只断言"约束不被重试"，不断言"哪些别的错误被重试"。这样将来把某类错误**收紧**（从 Retry 改成 Fatal）不会弄红测试，而**放宽**（把约束改成可重试）一定会红。

### V4：两个并发事务触发死锁后，应该有一个成功

这是本文功能本身的地基测试，形状照抄 Rust 篇 9.1 的并发互斥测试（**两个真连接，不包事务回滚**）。

```rust
/// 故意按相反顺序锁两件器材，制造一个确定的死锁。
/// 用 raw SQL（不走 lock_many），因为 lock_many 的 ORDER BY id 本身是防死锁的。
async fn lock_two_in_bad_order(conn: &mut PgConnection, first: i64, second: i64) -> Result<(), AppError> {
    sqlx::query("SELECT id FROM equipment WHERE id = $1 FOR UPDATE")
        .bind(first).fetch_one(&mut *conn).await?;
    tokio::time::sleep(Duration::from_millis(60)).await;   // 让对手拿到第一把锁
    sqlx::query("SELECT id FROM equipment WHERE id = $1 FOR UPDATE")
        .bind(second).fetch_one(&mut *conn).await?;
    Ok(())
}

#[sqlx::test(migrations = "../../migrations")]
async fn one_of_two_deadlocking_transactions_succeeds(pool: PgPool) {
    // fixtures::two_in_stock → EQ_A < EQ_B，两个真连接
    let (attempts_a, attempts_b) = (Arc::new(AtomicU32::new(0)), Arc::new(AtomicU32::new(0)));

    let a = retry::with_retry(&pool, Policy::write().no_jitter(), &*attempts_a, |conn| {
        lock_two_in_bad_order(conn, EQ_A, EQ_B)        // A 先 A 后 B
    });
    let b = retry::with_retry(&pool, Policy::write().no_jitter(), &*attempts_b, |conn| {
        lock_two_in_bad_order(conn, EQ_B, EQ_A)        // B 先 B 后 A
    });
    let (ra, rb) = tokio::join!(a, b);

    assert!(ra.is_ok() && rb.is_ok(), "重试之后两边都必须成功，而不是一方拿到 500");
    assert_eq!(attempts_a.load(SeqCst) + attempts_b.load(SeqCst), 3,
               "恰好一方重试了一次（另一方一次就成）");
    // ⚠️ 刻意不断言“死锁的 SQLSTATE 是 40P01”——Postgres 死锁检测可能报
    //    40P01 也可能让某方超时后以 40001 的形状失败（取决于检测时机与版本）。
    //    要断言精确 SQLSTATE 的做法见 V6 的注入式写法（V5 是连接阶段那条）。
    // 同时断言 biz.retry 的字段契约（不是条数）：见 V6
}
```

**这条测试要抓住的东西**（写清楚，因为它容易被写成恒真）：

1. **必须真的发生死锁**：两边都只跑一次就成功的话，这条测试什么都没证明。用一个 `debug_assert!` 或"至少一方 `attempts > 1`"来防止它退化成恒真——**恒真的并发测试比没有测试更糟**（运行时篇 R3.2 #1 与日志 V1 有同样的告诫）。
2. **`ORDER BY id` 的对照测试**（与下一条成对，一个证明"会死锁"，一个证明"不会"）：同样的两件器材，两条路径都走 `db::equipment::lock_many`（`ORDER BY id`），断言**两边都一次成功、零重试**。这才是 Rust 篇 3.3 那句"`ORDER BY id` 是防死锁的全部机制"的证据——有了它，V4 的死锁测试才有对照面。

### V5：连接类错误在取连接阶段可重试，在提交阶段不可

`pool.begin()` 的失败需要一个可控的注入点。**建议引入一个极薄的抽象**（只在测试中暴露）：

```rust
/// 生产：直接 pool.begin()。测试：可注入"前 N 次失败"
#[async_trait]
pub trait TxBeginner: Send + Sync {
    async fn begin(&self) -> Result<Transaction<'static, Postgres>, sqlx::Error>;
}
```

如果不想为测试引入 `async_trait`，退路是**用真实的死连接**：在 `with_retry` 开始前 `SELECT pg_terminate_backend(pid_of_our_own_connection)`，随后的 `begin()` 或事务体的第一条语句会失败（`08006`/`Io`）——因为是取连接阶段，所以可重试。**这条退路更好：它测的是真的错误，而且不需要为生产代码加抽象。**

```rust
#[sqlx::test(migrations = "../../migrations")]
async fn connection_loss_before_any_sql_is_retried(pool: PgPool) {
    // 让某一条连接在 BEGIN 前后掉线，断言 with_retry 重新取连接并成功
}

#[sqlx::test(migrations = "../../migrations")]
async fn commit_phase_failure_is_not_retried(pool: PgPool) {
    // 在 body 里写数据 → 让 commit 失败（例如关掉底层连接）
    // 断言：attempts == 1，返回错误，且**库里没有半成品行**
    // 这条守的是第 3.2 节第 3 点：宁可 500，也不要冒写出重复数据的风险
}
```

第二条是本文**最需要有人反对也最需要保留**的一条测试：如果有人说"提交失败也该重试"，让他先读第 3.2 节第 3 点，再决定要不要接受重复出库。

### V6：`biz.retry` 的字段契约（不是条数）

用日志设计 8.1 的 `LogCapture`：

```rust
#[sqlx::test(migrations = "../../migrations")]
async fn retry_event_carries_sqlstate_not_error_text(pool: PgPool) {
    let cap = LogCapture::new();
    // 手工制造一次 23P01 → 不会重试 → **不该**出现 biz.retry
    // 制造一次死锁 → 应出现 biz.retry
    let rows = cap.with_kind("biz.retry");
    assert_eq!(rows.len(), 1);                       // ← 这里可以断言
    let d = &rows[0]["data"];
    assert_eq!(rows[0]["level"], "WARN");
    assert!(d["attempt"].is_number() && d["max_attempts"].is_number());
    assert!(matches!(d["error_class"].as_str(),
                     Some("40P01") | Some("40001") | Some("08000")
                   | Some("08006") | Some("57P01") | Some("IO")));
    assert!(!cap.text().contains("SELECT"), "不得把 SQL 文本/表名写进日志（3.3 / R1.7）");
}
```

**为什么这里可以断言 `len() == 1`**：因为这是**单请求、单连接、非并发**的场景下，"恰好重试一次"是这条测试自己构造出来的确定事实，不是"随实现细节变化"的数量。日志设计 8.4 禁止的是**"跑一遍代表性流程然后数总数"**那种断言（V3/V4 都改用计数器了）。**为了不越界，这条测试应当写在测试名或注释里表明"本次重试次数由测试自己制造且确定"。**

### V7（可选，但建议）：隔离级别是 `READ COMMITTED`

因为 2.4 把 `40001` 的立场定为"不可达，出现即配置错误"，那就要有一条测试证明它真的不可达：

```rust
#[sqlx::test(migrations = "../../migrations")]
async fn isolation_level_is_read_committed(pool: PgPool) {
    let (lvl,): (String,) = sqlx::query_as("SHOW transaction_isolation").fetch_one(&pool).await.unwrap();
    assert_eq!(lvl, "read committed");
}
```

十行，守的是运行时篇 R4.3 那个决定不会被悄悄翻转（与 R3.2 #11"证明某个机制不存在"是同一类测试）。

### 待验证清单（本文里我没有验证的几点）

| # | 待验证 | 位置 | 失败时的表象 | 验证方式 |
|---|---|---|---|---|
| 1 | `for<'c> FnMut(&'c mut PgConnection) -> Fut` 的 HRTB 闭包能否被 Rust 推断 | 3.2 | 编译错误说闭包不满足生命周期约束（很可能是"实现者不够通用"这一类） | 实现第一步就把 3.2 的代码粘进去编译；若不成立，改用 `AsyncFnOnce` 或退到"`&TxFn` 具名实现"的形状 |
| 2 | `AsyncFnOnce` 作为闭包约束是否在所用 Rust 版本稳定 | 3.2 注释 | 同上 | 同上。这是 3.2 的**替代**方案，不是叠加方案 |
| 3 | `sqlx::Error` 0.8 的变体清单（`PoolTimedOut`/`PoolClosed`/`Io`/`WorkerCrashed`/`Tls` 的存在性）与 `PgPoolOptions` 取连接超时的方法名、默认值 | 2.2 B 表、5.3 | `match` 编译错误（好事，不会静默） | 对照 `sqlx` 源码/文档逐项核一次，把结论写回 Rust 篇 3.4 |
| 4 | Postgres 死锁在真实并发下报的是 `40P01` 还是 `40001` | 第 8 节 V4 | 测试断言错 SQLSTATE 而变红 | 跑 V4 时先断言"重试发生了"，再打印两次尝试的 SQLSTATE，据实写死 |
| 5 | `SHOW transaction_isolation` 的输出串（`read committed` 的精确写法） | 第 8 节 V7 | 断言字符串不匹配 | 跑一次即知 |
| 6 | 主稿第 8 节的 `DomainError` 里 `Conflict` 的缺失是有意还是笔误 | 6.2 | — | 需要主稿作者确认。本文按"笔误"处理并给了文案 |
| 7 | `25P03` 是否**总**能从 `sqlx::Error::Database` 里取到 `code()`（提案 04 的待验证 #5 只覆盖了 `57014`） | 2.2 A 表、6.2 | 取不到就会掉进 500，而它本来该是 503 | 手工触发一次 `idle_in_transaction_session_timeout`，断言 `db.code() == Some("25P03")` |
| 8 | 若采纳 6.2 的推荐方案，`DomainError::TemporaryUnavailable { sqlstate }` 这个带字段的变体是否会波及 `IntoResponse` 的匹配（需要改成 `{ .. }`） | 6.2 | 编译错误（不会静默） | 编译一次 |

### 与 [提案 04 · 连接池与超时](04_pool_and_timeouts.md) 的关系

两份提案在同一周成文，交界处只有四点，都已对齐。**不要在两份文档里各写一套**：

| 交界 | 谁定 | 对齐结果 |
|---|---|---|
| `acquire_timeout` 的字段名与默认值 | **04** | `pool_acquire_timeout_ms = 5000`，本提案 5.3 只引用不定义 |
| `PoolTimedOut` 落进哪个出口 | **03**（04 明确复用） | `DomainError::TemporaryUnavailable` / 503 / `Retry-After: 1` / "系统繁忙，请稍后重试" |
| `57014` / `25P03` 的翻译与可重试性 | 翻译 **04 提要求**、可重试性 **03** | 两者都翻成 `TemporaryUnavailable`（503），**两者都不可重试**（2.2、2.4.1） |
| `kind = tx.idle` / `pool.stat`（04 新增） | **04** | 与本提案的 `kind = biz.retry` 是三个不同事件，都走日志设计 6 节的"新增 `kind` 必须更新白名单测试"这条规则 |

---

## 9. 一句话总结（给评审）

这份提案要加的东西很少——**一个包装器、一个 `DomainError` 变体（+ `AppError::Sqlx` 带上 SQLSTATE 这一处结构改动）、一个新 `kind`、四个配置项**——它要拒绝的东西很多：**不重试约束违例、不重试池耗尽、不重试查询取消、不重试提交失败、不引进 `SERIALIZABLE`、不做无上限退避。**

它带来的两个**必须一起接受**的后果：

1. **死锁从"用户看到的 500"变成"日志里的 WARN"**（可用性优先）。配套义务是把 `biz.retry` 写进 R7.3 的每周检查——否则这条重试会把一个真缺陷藏起来。
2. **`biz.*` 业务日志的记录时机要从"提交前"挪到"提交后"**（Rust 篇 3.2 第 7 步要改）。否则 3 次尝试会产生 3 条 `biz.checkout`，其中两条对应的是已经回滚的事务。
