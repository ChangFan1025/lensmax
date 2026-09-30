# 07 · 删除与级联规则

日期：2026-09-30
版本：v1
状态：提案。**未实现**。本文只提建议，不改任何现有文档。
上位文档：[20260930_BACKEND_v1.md](../20260930_BACKEND_v1.md)（下称「主稿」）、[20260930_BACKEND_RUNTIME_v1.md](../20260930_BACKEND_RUNTIME_v1.md)（下称「运行时篇」）、[20260930_RUST_IMPL_v1.md](../20260930_RUST_IMPL_v1.md)（下称「Rust 篇」）。

**核实口径**：本文「现状」一列的每一个字都来自对主稿第 3.3 节 DDL 的逐行 grep + 人工核对（`grep -n "REFERENCES\|ON DELETE" docs/20260930_BACKEND_v1.md`，共 30 处 `REFERENCES`、3 处 `ON DELETE`，第 2 节的表逐行对应这 30 处）。凡本文断言「文档没定义」的地方，均已检索全部四份文档确认无相反规定。

**本文的验证边界**：本机没有 Postgres 也没有 Docker（`psql` / `pg_ctl` / `postgres` / `docker` 均不存在），所以以下两类结论属于 **PostgreSQL 文档行为 + 推理**，不是本机实测：①`NO ACTION` 与 `RESTRICT` 在事务内的语义差别；②部分唯一索引 `WHERE archived_at IS NULL` 让「归档行让出编号」成立。实现第一步应在 `sqlx::test` 里各写一条断言（见第 5.4 节）。

**核实范围的边界**：`docs/20260930_LensMax设计交接_v1`（「交接稿」）**不在仓库里**——`docs/` 下只有主稿、运行时篇、日志设计、Rust 篇四份文档。所以本文凡引用「交接稿第 N 节」之处，都不是直接读到原文，而是**转引自主稿的陈述**（主稿第 7 行称其为依据），其中"删除采用软删除：器材从库里消失，历史记录保留"一句转引自主稿 3.3 正文（第 413 行）与第 14 节差异汇总第 1 行（第 1180 行）。**凡本文断言"主稿没有定义"的地方，核实范围是那四份文档**——这是完整且足够的，因为"主稿"这个身份就是它们。

---

## 1. 缺口描述

### 1.1 已经定得很清楚的部分（先说清楚，避免重复讨论）

| # | 既有决定 | 出处 |
|---|---|---|
| 1 | 归档（`archived_at`）与「已出售」（`status='SOLD'`）是两件事，**不做软删除** | 主稿 D6、3.4 |
| 2 | 出库中禁止归档；列表默认排除已归档与已出售；归档**可恢复** | 主稿 3.4 |
| 3 | `equipment_history` 必须有指向 `equipment(id)` 的外键**且不可级联删** | 主稿 3.3 正文 |
| 4 | 编号生成后永不改变；分类改名/改前缀不影响历史编号 | 主稿 3.5 |
| 5 | 批量操作整批拒绝、点名违规件，**不做部分成功** | 主稿 C4、5.3；运行时篇 R3.2 #5 |
| 6 | 账号只停用不删除（`disabled_at`），停用即删会话 | 主稿 9、10.6；运行时篇 R7.4-2 |
| 7 | 照片只清孤儿，**绝不在请求路径上删文件** | 主稿 6.5、11 |
| 8 | 项目「有器材没归还」时不能标记完成或归档 | 主稿 5.6（既有规则） |

### 1.2 真正空白的部分

主稿第 3.3 节的 DDL 是一份**完整的骨架**：30 处外键它写了 30 处，其中 27 处**没有写 `ON DELETE`**。而 PostgreSQL 的默认行为是 `NO ACTION`。于是当前文档的实际承诺是：

> 系统里几乎所有「目标行被删除」的操作都会撞上一个 `23503`（foreign_key_violation）——而主稿第 8 节的 SQLSTATE 翻译只处理了 `23P01` 与 `23505`，**`23503` 会被翻译成 500**。

这不是"删不掉"，而是"删的时候返回 500，用户不知道发生了什么"。空白具体是五处：

| 缺口 | 现状（已核实） | 后果 |
|---|---|---|
| **分类下有器材时怎么办** | `equipment.category_id bigint **NOT NULL** REFERENCES category(id)`，无 `ON DELETE` | 删分类 → `23503` → 500。且因为是 `NOT NULL`，「置空器材的分类」这个选项**在 DDL 层面根本不成立** |
| **项目有出库记录时能不能删** | `checkout.project_id`、`request.project_id` 均 `bigint`（**可空**）`REFERENCES project(id)`，无 `ON DELETE` | 删项目 → `23503` → 500。而 5.6 已经有一条更严格的既有规则管着"归档" |
| **归档器材时照片怎么办** | `equipment.photo_id bigint`（**可空**）`REFERENCES photo(id)`，无 `ON DELETE`；**全库没有任何其他表引用 `photo`** | 规则未定。它直接决定 6.5 那条"数据库中被引用的 hash 集合"的边界 |
| **账号能不能删** | 主稿 10.6 明文决定**不删**。但 `equipment_history.actor_id` 是 `NOT NULL`，`request.requester_id` / `reviewed_by` / `maintenance.created_by` / `checkout.created_by` / `invite.created_by` 全部无 `ON DELETE` | 决定是清楚的，**实现路径**（如果产品坚持要删）是空白的 |
| **`equipment` 本身能不能删** | 没有任何一处定义删除入口。主稿 3.3 正文用外键堵死了它；`equipment_history.action` 的 CHECK 枚举里有 `ARCHIVE`/`RESTORE`，**没有 `DELETE`** | 归档代替删除的立场清楚，但**编号回收**这个连带问题没定义（见第 3.5 节） |

### 1.3 一处必须先澄清的措辞漂移（不改代码也要先统一）

主稿 5.3 的标题是「批量的全有或全无」，正文写"批量**删除**时若混了出库中的器材"，错误体文案是「以下器材不能**删除**（出库中）：CAM-003、CAM-011」，`DomainError::BatchRejected` 的 `#[error]` 也是「不能删除」。但 D6 与 3.4 同时宣布**不做软删除**。

**结论：删除这个概念在系统里已经不存在了，5.3 与第 8 节的"删除"是措辞残留，指的是归档。** 建议统一为「归档」，且错误文案要把**理由**逐件带上（见第 3.5.3 节）——现在的文案只有一个理由（出库中），而归档的拒绝理由至少有三种。

---

## 2. 级联行为对照表（核心产出）

### 2.1 阅读约定

- **现状**一列 = 主稿第 3.3 节 DDL 里**实际写了什么**，逐字核对，不是印象。
- **「不适用」** = 该引用指向的行按设计永远不会被删除（它只有归档/停用/状态流转），所以级联规则是防御性的，不是功能性的。
- 「期望」一列中，**RESTRICT** 与 **NO ACTION** 的区别很重要，见 2.3。
- 表名顺序按主稿 DDL 出现顺序，共 30 行，与 `grep` 结果一一对应。

### 2.2 对照表

| # | 表.列 → 引用表 | 目标行被删除时的期望行为 | 现状（主稿实际写的） | 需要改吗 |
|---|---|---|---|---|
| 1 | `occupancy.equipment_id → equipment(id)` | **RESTRICT**（有占用就不许删器材，占用本身就是"器材还有事") | `bigint NOT NULL REFERENCES equipment(id)`，无 `ON DELETE` → 默认 `NO ACTION` | 改：显式 `ON DELETE RESTRICT` + 命名 |
| 2 | `occupancy.source_request_id → request(id)` | **RESTRICT**（申请单不该被删） | `bigint REFERENCES request(id)`，可空，无 `ON DELETE` | 改：显式 `RESTRICT` + 命名 |
| 3 | `occupancy.source_checkout_id → checkout(id)` | **RESTRICT** | `bigint REFERENCES checkout(id)`，可空，无 `ON DELETE` | 改：显式 `RESTRICT` + 命名 |
| 4 | `app_user.workspace_id → workspace(id)` | **RESTRICT**（workspace 永不删除） | `uuid NOT NULL REFERENCES workspace(id)`，无 `ON DELETE` | 改：显式 `RESTRICT` + 命名 |
| 5 | `app_user_session.user_id → app_user(id)` | **CASCADE**（唯一一处 CASCADE 是**对的**：会话是**可丢的派生数据**，不是事实。停用/删除账号必须立刻踢下线，主稿第 9 节把这条列为"有状态会话的唯一理由"） | `bigint NOT NULL REFERENCES app_user(id) **ON DELETE CASCADE**` | 不改语义，只改名（`fk_app_user_session_user`） |
| 6 | `invite.workspace_id → workspace(id)` | **RESTRICT** | `uuid NOT NULL REFERENCES workspace(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 7 | `invite.created_by → app_user(id)` | **RESTRICT**（历史/审计作者，见 10.6） | `bigint NOT NULL REFERENCES app_user(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 8 | `invite.consumed_by → app_user(id)` | **RESTRICT** | `bigint REFERENCES app_user(id)`，可空，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 9 | `workspace_setting.workspace_id → workspace(id)` | **RESTRICT** | `uuid PRIMARY KEY REFERENCES workspace(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 10 | `category.workspace_id → workspace(id)` | **RESTRICT** | `uuid NOT NULL REFERENCES workspace(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 11 | `photo.workspace_id → workspace(id)` | **RESTRICT** | `uuid NOT NULL REFERENCES workspace(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 12 | `equipment.workspace_id → workspace(id)` | **RESTRICT** | `uuid NOT NULL REFERENCES workspace(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 13 | **`equipment.category_id → category(id)`** | **RESTRICT**（主稿 3.3 正文从未说要删分类；本文第 3.1 节的建议是**取消"删除分类"这个功能**，只保留停用。RESTRICT 是这条产品决定的数据库落点） | `bigint **NOT NULL** REFERENCES category(id)`，无 `ON DELETE`。**`NOT NULL` 已核实**：主稿第 255 行 | 改：`RESTRICT` + 命名。**注意：因为 `NOT NULL`，"删除时把器材分类置空"（`SET NULL`）在 DDL 层面不成立，除非同时把列改成可空——而器具必须有分类是主稿 5.1 的既有限制（除名称分类外其余可留空），所以不建议动 `NOT NULL`** |
| 14 | **`equipment.photo_id → photo(id)`** | **RESTRICT**（本文建议：归档**不释放**照片引用，见第 3.3 节。因此 `photo` 行只应由一次**显式的两步运维操作**删除——先清 `photo_id`，再删 `photo` 行。RESTRICT 让"直接删 `photo` 行"这件事失败在 500 之前的领域错误上，而不是留下悬空引用） | `bigint REFERENCES photo(id)`（**可空**），无 `ON DELETE` | 改：显式 `RESTRICT` + 命名。**明确不选 `SET NULL`**，理由见 3.3.4 |
| 15 | `project.workspace_id → workspace(id)` | **RESTRICT** | `uuid NOT NULL REFERENCES workspace(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 16 | `request.workspace_id → workspace(id)` | **RESTRICT** | `uuid NOT NULL REFERENCES workspace(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 17 | `request.requester_id → app_user(id)` | **RESTRICT** | `bigint NOT NULL REFERENCES app_user(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 18 | `request.project_id → project(id)` | **RESTRICT**（与 #22 一起，构成"有出库/有申请的器材项目不能删"） | `bigint REFERENCES project(id)`，可空，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 19 | `request.reviewed_by → app_user(id)` | **RESTRICT** | `bigint REFERENCES app_user(id)`，可空，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 20 | `request_item.request_id → request(id)` | **RESTRICT**（**建议推翻现状**：见 2.4） | `bigint NOT NULL REFERENCES request(id) **ON DELETE CASCADE**` | **改：删掉 CASCADE，换成 RESTRICT** |
| 21 | `request_item.equipment_id → equipment(id)` | **RESTRICT**（器材不该被删） | `bigint NOT NULL REFERENCES equipment(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 22 | `checkout.workspace_id → workspace(id)` | **RESTRICT** | `uuid NOT NULL REFERENCES workspace(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 23 | **`checkout.project_id → project(id)`** | **RESTRICT**（项目**只归档不删除**，见第 3.2 节） | `bigint REFERENCES project(id)`（**可空**），无 `ON DELETE`。**可空已核实**：主稿第 342 行 | 改：`RESTRICT` + 命名 |
| 24 | `checkout.source_request_id → request(id)` | **RESTRICT** | `bigint REFERENCES request(id)`，可空，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 25 | `checkout.created_by → app_user(id)` | **RESTRICT** | `bigint NOT NULL REFERENCES app_user(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 26 | `checkout_item.checkout_id → checkout(id)` | **RESTRICT**（**建议推翻现状**：见 2.4） | `bigint NOT NULL REFERENCES checkout(id) **ON DELETE CASCADE**` | **改：删掉 CASCADE，换成 RESTRICT** |
| 27 | **`checkout_item.equipment_id → equipment(id)`** | **RESTRICT**（器材从不删除，只归档） | `bigint NOT NULL REFERENCES equipment(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 28 | `maintenance.workspace_id → workspace(id)` | **RESTRICT** | `uuid NOT NULL REFERENCES workspace(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 29 | **`maintenance.equipment_id → equipment(id)`** | **RESTRICT**（另需一条应用层规则，见 2.5） | `bigint NOT NULL REFERENCES equipment(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 30 | `maintenance.created_by → app_user(id)` | **RESTRICT** | `bigint NOT NULL REFERENCES app_user(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 31 | `equipment_history.workspace_id → workspace(id)` | **RESTRICT** | `uuid NOT NULL REFERENCES workspace(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 32 | **`equipment_history.equipment_id → equipment(id)`** | **RESTRICT**——这正是主稿 3.3 正文那句"**不可级联删**"要的东西。**现状其实已经满足**（无 `ON DELETE` ⇒ `NO ACTION`），但"不可级联删"有两种写法，见 2.3 | `bigint NOT NULL REFERENCES equipment(id)`，无 `ON DELETE` | 改：显式写成 `RESTRICT` + 命名。**这是全文最重要的一行**：它让"器材永远删不掉"成为数据库事实而不是代码纪律 |
| 33 | `equipment_history.actor_id → app_user(id)` | **RESTRICT**（10.6 的落点：删账号会让历史失去作者） | `bigint NOT NULL REFERENCES app_user(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |
| 34 | `pending_upload.workspace_id → workspace(id)` | **RESTRICT** | `uuid NOT NULL REFERENCES workspace(id)`，无 `ON DELETE` | 改：`RESTRICT` + 命名 |

> 说明：表里 34 行 > 30 处 `REFERENCES` 的唯一原因是我把 `occupancy` 的三处（主稿第 3.1 节的 DDL）也算进来了——`grep` 找到的 30 处包含它们。行号对应主稿 3.3 节 DDL 与 3.1 节 DDL。

### 2.3 为什么建议全部写 `RESTRICT` 而不是留默认的 `NO ACTION`

两者在"目标行被删、子行还在"时都报 `23503`，**差别在事务内**：`NO ACTION` 的检查可以推迟到语句/事务末尾，`RESTRICT` 立即检查。对本文的场景，`RESTRICT` 更好，理由有两条：

1. **`NO ACTION` 在同一个事务里"先删父行、后删子行"是合法的**——这意味着一个"删设备再删它的历史"的脚本会**静默成功**，而 `RESTRICT` 会在第一步就拦下来。主稿 3.3 正文要的是"这条靠外键而不是靠代码自觉"，`RESTRICT` 才是"自觉不了"的那个版本。
2. `RESTRICT` 在迁移里是**显式记录的产品决定**；`NO ACTION` 是"忘了写"。下一个读 DDL 的人不该靠 PostgreSQL 默认值的知识去反推本系统的删除立场。

`RESTRICT` 的成本：将来真要做 workspace 级联清理（比如清空测试数据），必须按依赖顺序一条条删。单工作室、单 workspace 的场景下这不是问题，而且**这正是我们想要的行为**。

### 2.4 建议推翻的两处 `ON DELETE CASCADE`

主稿只写了 3 处 `ON DELETE`，其中 #20 `request_item.request_id` 与 #26 `checkout_item.checkout_id` 建议改掉：

| 现状 | 问题 | 建议 |
|---|---|---|
| `request_item.request_id ... ON DELETE CASCADE` | 一次 `DELETE FROM request` 会**静默抹掉审批事实**（`approved`、`delivered_at`、`returned_at`）。而 `occupancy` 里那条 `RESERVATION` 不会被删（#2 只是 `RESTRICT`，但删 request 本身如果先删了 request_item 就绕过了）——于是"占用对账"任务（主稿 11 节）会在事后报出偏差，而偏差的成因已经不可考 | 改为 `RESTRICT`。删除申请单本来就不是一个存在的功能（申请只有 `WITHDRAWN` 状态） |
| `checkout_item.checkout_id ... ON DELETE CASCADE` | 一次 `DELETE FROM checkout` 会**静默抹掉出库事实**，而 `equipment_history` 里那条 `CHECKOUT` 还在。历史说"出过库"，出库表里查不到——正是主稿 10.1"唯一的叙事主线"要避免的两份互相漂移的真相 | 改为 `RESTRICT` |

**保留 `app_user_session` 的 CASCADE**：会话是可丢的派生数据，删账号必须连带踢下线。这是唯一一处 CASCADE 语义正确的地方。

### 2.5 表里看不出来的三条应用层规则（必须一起做，否则外键不够）

| 规则 | 为什么外键做不到 |
|---|---|
| **归档前检查"没有未归还的出库"** | 主稿 3.4 只写了"出库中禁止归档"（`status = CHECKED_OUT`）。但一件器材可能 `status = IN_STOCK` 却仍有一条 `checkout_item.returned_at IS NULL`（数据漂移、或逐件归还的中间态）。正确判据是 `NOT EXISTS (SELECT 1 FROM checkout_item WHERE equipment_id = $1 AND returned_at IS NULL)`，`status` 只是它的一个冗余投影。**建议以 `checkout_item` 为准，`status` 作为预检查的第一道** |
| **归档前检查"没有未关闭的维修"** | `maintenance_one_open` 部分唯一索引保证"同时只有一条未关闭维修"，但它不阻止归档一件正在维修的器材。建议与出库同口径拒：`DomainError::InMaintenanceNotArchivable`（人话"维修中的器材不能归档，请先完成维修"） |
| **分类停用前检查"这是不是最后一个启用的分类"** | 主稿 5.1 说除名称分类外其余可留空 ⇒ 建档必须有分类。若把最后一个启用分类停用，建档流程会无分类可选。建议拒绝并给 `DomainError::LastActiveCategory`（人话"这是最后一个启用的分类，不能停用"）。**注意这是应用层检查**：数据库层面这个约束要写成"workspace 至少有一个 category 满足 `disabled_at IS NULL`"，属于跨行断言，只能用触发器或物化一个计数列——**不值得**，应用层足够 |

---

## 3. 逐问的选项与建议

### 3.1 分类删除

#### 已核实的事实

| 事实 | 核实位置 |
|---|---|
| `equipment.category_id` 是 **`NOT NULL`** | 主稿第 255 行：`category_id bigint NOT NULL REFERENCES category(id)` |
| 因此「删除分类时把器材的分类置空」（`ON DELETE SET NULL`）**现在不成立**——列不可空，`SET NULL` 会在删除时直接报错 | 同上。要让它成立必须先 `ALTER COLUMN category_id DROP NOT NULL` |
| 「器材必须有分类」是既有限制，不是顺手加的 | 主稿 7.6 / 交接稿第 5.1 节：除名称分类外其余字段可留空 ⇒ 分类是必填项 |
| 分类承担编号前缀 `code_prefix`，且 `UNIQUE (workspace_id, code_prefix)` | 主稿 3.3（第 230、234 行） |
| **编号本身不再依赖分类** | 主稿 3.5 规则 2「生成后永不改变」，3.5 规则 1「数值来自 `category.next_seq`，不是 `max(code)+1`」。**关键：`equipment.code` 是一列已落库的 `text`（主稿第 253 行），`category` 表里没有任何一列参与编号的查询或拼接。** 换句话说：前缀只是**生成时刻**的输入，不是**运行时**的依赖 |
| 全库没有任何表引用 `category` 除 `equipment` 之外 | 对四份文档 grep `category_id` 只命中 2 处：`equipment.category_id`（主稿 255）与 Rust 篇的 `EquipmentRow.category_id`（Rust 篇 94） |
| **主稿从未定义分类的删除或停用** | grep `停用`/`disabled_at` 全部命中只涉及 `app_user`。3.4 的 `DISABLED` 是**器材状态**，不是分类 |

所以「删掉分类会让前缀映射消失、从而破坏已有编号」这个担心**不成立**：已有编号是 `equipment.code` 里的字符串，`CAM-003` 不会因为 `category` 表少一行而变成别的。**但有一个真实后果**：分类删除后，`equipment.category_id` 指向的行没了（如果 DDL 允许的话），而 `equipment_history` 的 `detail.changed` 数组里可能会出现 `category_id: 12 → 15` 这种**无法翻译成分类名的裸 ID**——历史要能读，分类名就必须能查回来。这一条比编号更硬。

#### 选项

| 选项 | 做法 | 代价 |
|---|---|---|
| **A. 禁止删除**（本文建议） | `ON DELETE RESTRICT` + 应用层给领域错误；同时**新增「停用分类」**（`category.disabled_at`）表达"这个分类我不再用了" | 用户没有"彻底删掉"的能力，需要一个能接受的替代动作 |
| B. 允许删除并要求先迁移器材 | 删除前必须把所有器材（**含已归档**）`category_id` 改到别的分类 | 已归档器材在列表里看不见，"迁移已归档器材"这件事在界面上无处可做；用户会卡死 |
| C. 允许删除并置空分类 | 先 `DROP NOT NULL`，再 `ON DELETE SET NULL` | **推翻"器材必须有分类"**（主稿 5.1 的既有限制）；且历史里的 `category_id` 变成不可翻译的裸 ID；且归档器材会被静默改成"无分类" |

#### 建议（纯技术决定 + 一个待拍板项）

1. **`ON DELETE RESTRICT`，且不提供分类删除接口。** 分类是**低频创建**的对象（十人工作室大概 10～30 个），删除它没有收益，损失是"历史可读性"和"编号前缀映射"。这与主稿对账号的立场（10.6 只停用）完全同构。
2. **新增「停用分类」**，作为"这个分类我不再用了"的唯一表达：
   - DDL：`category.disabled_at timestamptz`（可空，NULL = 启用）；
   - 语义三件套：**停用后不能再被新器材选用**；**已有器材（含已归档）照常显示其分类名**；**可以恢复**；
   - 列表/筛选的分类下拉默认过滤 `disabled_at IS NULL`，但**已停用分类下的器材照常出现在器材列表里**（否则等于隐藏了器材）；
   - 拒绝最后一个启用分类被停用（见 2.5）。
3. 错误文案（`DomainError` 变体，`Display` 直接是中文，Rust 篇 5.2）：

   | 场景 | 错误体 `message` | 建议状态码 |
   |---|---|---|
   | 删除一个有器材的分类（外键兜底 / 有意调用） | 「分类「摄影机」下还有 12 件器材，不能删除。若不再使用，请把它停用」 | 409 |
   | 停用最后一个启用分类 | 「这是最后一个启用的分类，不能停用——建档时必须选择分类」 | 409 |
   | 给新器材选了已停用的分类 | 「分类「摄影机」已停用，不能用于新建器材」 | 422 |

4. **待拍板（产品决定）**：

   | # | 要拍的事 | 选项 | 建议 |
   |---|---|---|---|
   | P1 | 分类要不要"彻底删除"这个能力？ | (a) 只停用，永不删除（本文建议）／(b) 允许删除空分类（名下 0 件器材，含已归档） | **(a)**。理由是"停用不是删除"已在账号上立过一次（10.6），分类没有更强的不删理由。(b) 也可以接受，但收益只有"下拉少一条"，而它让"分类永不删除"这个统一口径破了一个口子 |
   | P2 | 「删除这个概念在本系统里还需不需要存在？」 | (a) 不需要——三种语义已经覆盖：东西还在不出借=`DISABLED`，东西没了正常终局=`SOLD`，录错了=`archived_at`（主稿 3.4 的三行表）／(b) 保留一个窄口径的真删除（如"零引用对象"的清理） | **(a)**。删除的每一种用例都已有对应语义，见第 3.5 节。**"这个分类我不再用了"用停用表达（本文新增），"这个账号不在了"用 `disabled_at` 表达（10.6），"这件器材录错了"用归档表达（3.4）** |

---

### 3.2 项目删除

#### 已核实的事实

| 事实 | 核实位置 |
|---|---|
| `checkout.project_id` **可空**，无 `ON DELETE` | 主稿第 342 行：`project_id bigint REFERENCES project(id)` |
| `request.project_id` **可空**，无 `ON DELETE` | 主稿第 312 行 |
| `project` 有 `UNIQUE (workspace_id, name)` | 主稿第 299 行 |
| `project.status` 已经是 `('ACTIVE','COMPLETED','ARCHIVED')`——**项目已经有"归档"这个状态** | 主稿第 294–295 行 |
| 既有规则：有器材没归还的项目不能标记完成或归档 | 主稿 5.6（本文第 1.1 节 #8） |
| **主稿没有定义项目删除** | grep 无任何项目删除的接口或规则 |

#### 建议（纯技术决定）

1. **项目不提供删除，只提供状态流转 `ACTIVE → COMPLETED → ARCHIVED`**，DB 层 `ON DELETE RESTRICT`（#18、#23）。
2. **与 5.6 保持一致的推理链**：5.6 说"有器材没归还的项目不能标记完成或归档"。删除是**比归档更彻底的消失**，所以适用同一条件、且更严：删除的门槛不可能低于归档。既然归档已经够了（`status='ARCHIVED'` 让项目从默认列表消失，且 `checkout.project_id` 的历史照常可查），删除的边际收益是零，而它的代价是 `checkout`/`request` 的历史失去项目引用。
3. **`UNIQUE (workspace_id, name)` 与归档的冲突**：归档一行仍占着名字，所以"删掉再建同名项目"这个动作在归档路径下会撞 `23505`。这**不是 bug，是必须显式回答的问题**（因为现实里"去年那个《秋日》拍完了，今年又有一个《秋日》"会发生）。
   - **建议（纯技术，v1）**：保持 `UNIQUE (workspace_id, name)` 不变，把错误说清楚——「已存在同名项目「秋日」（含已归档）。归档项目改个名，或给新项目换一个名字」。
   - **备选（产品决定）**：改成部分唯一索引 `CREATE UNIQUE INDEX project_name_active_uniq ON project (workspace_id, name) WHERE status <> 'ARCHIVED'`，允许"已归档的同名项目 + 新的同名项目"共存。代价是**项目名不再是稳定标识**——`GET /api/v1/activity?scope=project` 是按 `id` 查的（没问题），但任何"按名字找项目"的路径（含前端缓存、导出 CSV 的人工核对）都可能返回多行。

#### 待拍板（产品决定）

| # | 要拍的事 | 选项 | 建议 |
|---|---|---|---|
| P3 | 归档项目占着名字怎么办？ | (a) 保持现状：同名 → 报错，让用户改名（v1）／(b) 部分唯一索引，允许同名共存 | **(a) v1**。理由是它零改动、且"归档项目改个名"的成本极低。(b) 留到真的被抱怨再做，那时也已有一年的真实数据可参考 |
| P4 | 项目要不要真正的删除？ | (a) 不要（本文建议）／(b) 只要"零出库记录"的项目可删 | **(a)**。项目卡片要显示"历史出库记录数"（主稿 10.5），一个从没出过库的项目归档后正是那条记录的来源；删掉它，界面上的"0"和"项目不存在"就没法区分 |

---

### 3.3 归档器材时照片怎么办

#### 已核实的事实

| 事实 | 核实位置 |
|---|---|
| `equipment.photo_id` **可空**，无 `ON DELETE` | 主稿第 263 行 |
| **全库只有 `equipment.photo_id` 引用 `photo`** | 对四份文档 grep `photo_id`：命中 `equipment` DDL（263）、Rust 篇 `EquipmentRow.photo_id`（102）、日志事件 `file.missing` 的字段（日志设计 246）。**没有第二张表引用 `photo`** |
| 照片是**内容寻址**、不可变、`UNIQUE (workspace_id, sha256)` ⇒ **相同图片在库里只有一行 `photo`，可能被多件器材引用** | 主稿 6.2、3.3 第 247 行 |
| 孤儿清理是**标记清除**，第 2 步是"减去数据库中被引用的 hash 集合（`photo` 表）" | 主稿 6.5 |
| 「换图时不删旧文件（可能被历史引用）」 | 主稿 6.5 正文第一句 |
| 归档**可恢复** | 主稿 3.4 |
| 已出售是**状态机终点**（`SOLD` 没有出边） | Rust 篇第 4 节 `ALLOWED` 表：`// Sold 是终点，没有出边` |
| **主稿没有定义归档时照片的处理** | grep 无。6.5 只说"被引用的 hash 集合"，没说"被谁引用" |

#### 三个问题的直接回答

**① 归档器材是否应该释放 `photo_id`？**

不建议。关键是把"文件能不能被删"这件事绑定到**引用计数的唯一实现者**（`photo` 表的行数），而不是绑定到器材的可见性：

- 只要还有**任何一行 `equipment`**（无论 `archived_at` 是否为空、无论 `status` 是 `SOLD` 还是 `DISABLED`）引用那个 `photo_id`，这一行 `photo` 就存在，它的文件就**永远不会**进入 6.5 的删除集合。这是 6.5 原文的直接推论，不用改一个字。
- 归档是可恢复的（3.4）。归档时保留引用 ⇒ **恢复是完美的**（照片立刻回来）。归档时释放引用 ⇒ 恢复后是一张**没有照片的器材**，而且没有任何地方记录"原来是哪张"（除非写进 `equipment_history.detail`，见下）。
- 释放引用会**立刻把文件推进待删集合**，而 6.5 的清理是每周一次的全盘标记清除——归档后第 3 天"我点错了"要恢复，照片可能已经不在了。这违反 6.5 的立场（"一次误删是永久数据损失，一次没删只是占几百 KB"）。

**代价与判断**：归档 + 已出售的器材会**永久持有**它们的照片，磁盘只增不减。量级是可控的：主稿 6.1 说"几百到几千张，每张 300–800 KB，合计几百 MB"。一台三年期的单工作室台账，归档+已出售照片累计大概在 GB 级，而**这个量级不值得为它引入一套引用释放 + 宽限期 + 恢复后重挂的机制**。

**② 同一张照片被多件器材引用时，释放一个引用会不会影响别人？**

不会——**前提是"释放"只写成 `UPDATE equipment SET photo_id = NULL WHERE id = $1`，绝不写成 `DELETE FROM photo`**。删除引用的语义是"这一行不再引用"，`photo` 行的存活由**所有**引用者共同决定。这也正是 `SET NULL` 与"应用层显式清空"的关键差别：

- 应用层显式清空 ⇒ 引用计数真实反映剩余引用者；
- `ON DELETE SET NULL` ⇒ 只有"删 `photo` 行"这一个动作会触发。它不解决"释放引用"这个需求，只会让"直接删 `photo` 行"变得静默。**所以不选 `SET NULL`。**

**③ 释放后 `equipment.photo_id` 变成什么？**

可空——**这一列已经是可空的**，不需要 DDL 改动。本文的建议是**归档时不写它**（保持原值），所以这一问在推荐路径下不触发。

**但如果产品选了"归档即释放照片"这个备选**，那么必须补三件事（本文不建议做，列出来是为了让选项完整）：
1. 释放前把 `photo_id` 与 `sha256` 写进 `equipment_history.detail`（`action='ARCHIVE'` 那一条），否则恢复后不知道该挂哪张；
2. 引入宽限期，且**不能靠 `pending_upload`**（它只保护 24 小时，主稿 6.3/6.5）：需要一个 `photo_hold(workspace_id, sha256, until)` 之类的例外表，让 6.5 的第 3 步多减去一个集合——**这是对 6.5 的机制扩展，不是参数调整**；
3. 接受"归档 30 天后照片被删，此后恢复只能是无照片器材"这个产品事实。

#### 建议（纯技术决定）

**归档不释放照片引用。** 于是：

| 场景 | 照片行为 |
|---|---|
| 换图（PATCH `photoId`） | 新旧 `photo` 行都留存；旧行在新的 `photo_id` 写完后不再被引用，**进入下一轮孤儿清理的待删集合**（主稿 6.5 原文已如此） |
| 归档器材 | `photo_id` **不动**。文件被引用 ⇒ 永不被删。恢复后照片完好 |
| 恢复器材 | 无需任何照片处理 |
| 出售器材（`status='SOLD'`） | 同上，**引用保留**。已出售是终局，但"这件器材曾经长什么样"对台账有价值，且文件量级可接受 |
| 用户主动"清除照片"（PATCH `photoId: null`，主稿 7.6 的三态写入已经支持） | 清空引用 ⇒ 若无人再引用，下一轮清理回收文件。**这是唯一一条照片被回收的正常路径，且它是用户显式发起的** |
| 删除 `photo` 行 | **不允许作为请求路径上的操作。** 若运维必须做，顺序是：`UPDATE equipment SET photo_id = NULL WHERE photo_id = $1` → `DELETE FROM photo WHERE id = $1` → 让下周的孤儿清理回收文件。`ON DELETE RESTRICT` 保证顺序不能被跳过 |

**唯一的代价**：磁盘只增不减，直到有人手动清理已出售器材的照片。**这一条要写进运维手册**（运行时篇 R7.4 的清单里加一项），作为"如果你真的缺磁盘"的应对，而不是一个自动机制。

#### 待拍板（产品决定）

| # | 要拍的事 | 选项 | 建议 |
|---|---|---|---|
| P5 | 已出售（`SOLD`，状态机终点）器材的照片要不要在 N 年后放开？ | (a) 永不放开（本文建议）／(b) 出售满 N 年（如 5 年）后允许一个显式的运维任务清理 | **(a) v1**。理由是"台账"的价值就在能查回历史，而几百 MB 不是理由。若将来磁盘真的紧张，(b) 是一个纯运维脚本，不需要动数据模型 |
| P6 | 已归档/已出售器材的照片要不要在界面上隐藏？ | (a) 照常显示／(b) 列表缩图隐藏、详情显示／(c) 一律隐藏 | 需要产品定。**技术侧无差别**：无论选哪个，`photo_id` 都不动。注意 3.4 已经定了"列表默认排除已归档与已出售"，所以这条的实际影响面只有"显示已出售"筛选和直接打开详情/恢复的场景 |

---

### 3.4 账号停用 vs 删除

#### 已核实的事实（问题里要求"核实确实决定不删除"）

| 事实 | 核实位置 |
|---|---|
| **"账号只停用（`disabled_at`），不删除"** | 主稿 10.6 标题就是「账号不删除」，正文第 1042 行逐字如此 |
| 理由：**"不要为了清理账号去删行——那会让历史失去作者，而'谁改的'正是这个界面存在的理由"** | 主稿 10.6 |
| 机制已在第 9 节落地：`disabled_at` 列 + "停用账号时删除其全部会话" | 主稿第 184 行（DDL）、第 941 行 |
| 数据库层面的额外约束：**`CHECK (role <> 'OWNER' OR disabled_at IS NULL)`**（主理人不可被停用）+ `app_user_one_owner` 部分唯一索引 | 主稿第 191–196 行 |
| 停用账号在历史里"照常显示名字并标注(已停用)" | 主稿 10.6 |
| 指向 `app_user` 的外键有 7 处，**全部无 `ON DELETE`**（= `NO ACTION`） | 主稿：`app_user_session.user_id`（有 CASCADE）、`invite.created_by`、`invite.consumed_by`、`request.requester_id`、`request.reviewed_by`、`checkout.created_by`、`maintenance.created_by`、`equipment_history.actor_id`（`NOT NULL`） |
| **主稿没有任何账号删除接口** | grep 无 |

**结论：确实已经决定不删除，且理由写在 10.6 里。** 本文要补的是"如果产品坚持要删"的可行路径。

#### 如果产品坚持要删（GDPR 类诉求 / 人员离职）

正确的做法是**匿名化 `app_user` 行，不真删**。理由是硬的、不是偏好：

- `equipment_history.actor_id` 是 **`NOT NULL`**，没有 `SET NULL` 可选。`DELETE FROM app_user` 会被外键拦下（`23503`），除非同时删掉这个人的全部历史——而那正是 10.6 拒绝的事，也是"每一步都留痕，谁也赖不掉"（交接稿第 4 节）的直接对立面。
- 而且**匿名化恰好满足 GDPR 类诉求的本意**：被要求删除的是"能识别到具体自然人的信息"，不是"这条业务事实"。`app_user` 行里能识别人的只有 `username` 与 `display_name`；`equipment_history` 里除了 `actor_id` 这个内部代理键，没有任何个人信息（`detail.changed` 是字段值，不含人名）。

**可行路径（一个 `POST /api/v1/users/{id}/anonymize`，OWNER only，单个事务）**：

```sql
-- 1. 匿名化身份字段，保留主键与历史引用
UPDATE app_user
   SET username     = 'deleted:' || id,      -- 释放用户名；见下方唯一约束说明
       display_name = '已删除账号',
       disabled_at  = COALESCE(disabled_at, now()),
       updated_at   = now()
 WHERE id = $1
   AND role <> 'OWNER';                        -- 主理人不可匿名化（见下）
-- 2. 杀会话（数据库层面由 app_user_session 的 CASCADE 兜底，
--    但在 DISABLE 路径上必须显式删，见主稿 9 / 运行时篇 R7.4-2）
DELETE FROM app_user_session WHERE user_id = $1;
```

四条实现要点：

1. **应用层必须显式拒绝 `role = 'OWNER'`**，且这**不是**过度防御：`CHECK (role <> 'OWNER' OR disabled_at IS NULL)`（主稿 196 行）会让这条 UPDATE 直接报 `23514`。真正的 OWNER 删除路径是"先做**主理人交接**"——而主稿 13.7 说这件事**当前无设计**。所以运营口径是：**主理人的离职在系统里目前无法完成，这是 R7.4-3 已经记录的既有限制，匿名化不改变它。**
2. **`username` 是 `NOT NULL`（主稿第 180 行），所以匿名化不能置空，只能填占位值。** `'deleted:' || id` 是最简单且天然唯一的选择（`id` 是 `bigserial`），并且它与原唯一约束 `UNIQUE (workspace_id, username)` 相容的同时**顺手把原用户名释放出来**——同名新账号可以再建。这正是"人员离职后新人接岗、沿用同一个登录名"的场景。若产品希望**禁止**登录名被复用，改成 `'deleted:' || id || ':' || gen_random_uuid()` 即可（位数更长，但保证不复用）。**这一条要产品定**：它决定"一个新账号能不能用离职者的登录名"。
3. **历史里的呈现不变**：因为 `actor_id` 没动，活动流接口（主稿 10.5）仍然能返回 `actor.name`。差别只有一处：名字从"李四"变成"已删除账号"。**建议在活动流的视图层同时输出一个 `actor.deleted: true`**，前端据此渲染"已删除账号"的样式——和 10.6 已经定的 `(已停用)` 标注是同一套机制，不要新造一套。
4. **`display_name` 里不放任何可识别信息**，也不要在 `detail` 或日志里留下原名。若产品要求"留一个内部可查的原始姓名的映射"，那就用一个**只有 OWNER 能读**的独立表（`user_erasure_log(deleted_user_id, original_username, requested_by, requested_at)`）——但这会让匿名化变成名义上的，需产品明确权衡。**默认不做。**

#### 待拍板（产品决定）

| # | 要拍的事 | 选项 | 建议 |
|---|---|---|---|
| P7 | 要不要匿名化入口？ | (a) 不做，只停用（现状）／(b) 做 `POST .../anonymize`（本文给的路径） | **(b) 但排在最后**。它解决的是一个**尚未发生**的诉求（GDPR 不适用于十人内部工作室；"离职"用停用完全够用）。**建议写进文档、不写进 v1 实现**——这样真出现诉求时不必重新论证，也不必冒然在 v1 里加一个会毁掉审计可读性的接口 |
| P8 | 匿名化后审计记录保留多久？ | (a) 永久保留（`actor_id` 指向匿名行）／(b) N 年后清除该 actor 的 `equipment_history.detail.changed` | **(a)**。删除字段变更详情会让"谁把价格改了"重新变成不可查，而这正是 10.6 和 5.5 存在的理由 |
| P11 | 匿名化后，离职者的登录名能不能被新账号复用？ | (a) 可以复用（`username = 'deleted:' \|\| id`）／(b) 禁止复用（`'deleted:' \|\| id \|\| ':' \|\| gen_random_uuid()`） | **(a)**。十人工作室里"离职后新人接岗、沿用同一个登录名"是常态；而 `app_user` 的 `id` 才是历史的锚点，登录名从来不是。选 (b) 的唯一理由是"担心两个自然人共用一个登录名造成误判"，但那在停用/匿名化之后已经不可能（该行已 `disabled_at`） |

---

### 3.5 `equipment` 本身的删除

#### 已核实的事实（问题要求"核实主稿的立场"）

| 事实 | 核实位置 |
|---|---|
| **主稿的立场是归档代替删除。** D6：`归档（archived_at）与「已出售」是两件事，**不做软删除**`；3.4 的三行表把"录错了、重复建档"归到 `archived_at` | 主稿 73、417–425 |
| **数据库层面已经堵死**：3.3 正文——"**历史表必须有外键指向 `equipment(id)` 且不可级联删**……这条靠外键而不是靠代码自觉" | 主稿 413 |
| `equipment_history.action` 的 CHECK 枚举**没有 `DELETE`**（有 `CREATE`/`UPDATE`/`ARCHIVE`/`RESTORE`/`SOLD`/…） | 主稿 389–391 |
| 状态机里**没有"移除"出边**（`SOLD` 是终点） | Rust 篇 `ALLOWED` 表 |
| 5.3 用"删除"这个词，但 3.4/D6 宣布不做软删除 ⇒ **措辞残留** | 主稿 536、549、891；见本文 1.3 |
| `equipment(workspace_id, code)` 是**表级唯一约束**（不是部分唯一索引）⇒ **已归档的行仍然占着编号** | 主稿第 269 行：`UNIQUE (workspace_id, code)` |
| 3.5 规则 2「生成后永不改变」；规则 1 说 `max(code)+1` 的坏处之一是"删除后序号会被回收复用" | 主稿 433–435 |

#### 「重复建档录错了」的正确操作路径

**归档，不真删。** 具体路径（全部用已有机制，无新概念）：

1. 在列表里找到那件（若它已被归档，用"显示已归档"筛回来）；
2. **先确认它没有未归还的出库、没有未关闭的维修**（2.5 的两条规则）；
3. 归档（`POST /api/v1/equipment/{id}/archive`，写一条 `action='ARCHIVE'` 的 history + `biz.archive` 日志）；
4. 若编号也错了，按下面的 P9 处理。

**为什么真删不可取**（三条，从硬到软）：

1. **它会撞外键**——`equipment_history`、`occupancy`、`maintenance`、`checkout_item`、`request_item` 五处 `RESTRICT`。要让真删成立，必须给这些表加 `ON DELETE CASCADE`，那等于**同时推翻 3.3 正文和 10.6**（历史失去器材、也失去作者）。
2. **它会让编号被复用**。"录错了"最常见的一种恰恰是"编号也录错了"，而 `max(code)+1` 的坏处（3.5 规则 1）正是"删除后序号会被回收复用"——真删会把这个被明确否掉的行为重新引进来。
3. **它的收益是零**：归档已经让器材从列表、搜索、可用集合里消失（`equipment_list_idx` / `equipment_status_idx` 都是 `WHERE archived_at IS NULL` 的部分索引），并能恢复。

#### 数据累积问题：会不会有？

**结论：会有，但不是问题；需要产品接受一个具体的、可见的代价。**

| 维度 | 判断 | 依据 |
|---|---|---|
| 行数累积 | **不是问题**。一个十人工作室的年建档量是几百件，`archived_at` 为空的活跃行才是所有索引覆盖的对象（`WHERE archived_at IS NULL`），归档行不进索引、不进默认查询 | 主稿 272–274 |
| 历史累积 | **不是问题，而且是我们想要的**。`ARCHIVE` 会写一条 history；"每一步都留痕"要求它留下 | 主稿 10.1 / 交接稿第 4 节 |
| **编号空隙** | **会留下，且必须接受。** 举例：CAM-001 建错了 → 归档 → 下次建档 `next_seq` 已经是 2，得到 **CAM-003**。用户会问"CAM-002 去哪了"。**这是"编号生成后永不改变 + 不复用"的必然代价**（3.5 规则 1、2），也是纸质档案与系统对得上号的代价 | 主稿 3.5 |
| **错误的编号被永久占用** | **这是真实的、需要产品拍板的一处。** 若录错的是**编号本身**（本该 CAM-010、录成 CAM-001），归档只能让 CAM-001 从列表消失，**它仍然占着 `(workspace_id, code)` 这个唯一约束**——想用 CAM-001 重新建档会撞 `23505`。主稿对此**没有定义**（已 grep 核实） | 主稿 269、3.5 |

**建议（纯技术）：不加"归档时释放编号"。** 让"归档行也让出 `(workspace_id, code)`"成立需要把唯一约束改成部分唯一索引 `WHERE archived_at IS NULL`，代价是**同一 workspace 内出现两行都是 `CAM-003`**（一归档一活跃）。而编号是纸质档案、出库清单、历史记录的共同锚点——3.5 规则 2 的整段理由（"历史记录、出库清单、纸质档案都引用它"）依赖"编号在库里唯一"。**为了修一个录入手误而弱化那句理由，不划算。**

替代方案（成本极低，且不破坏任何原则）：**允许修改 `code`**。主稿 3.5 说"生成后永不改变"，指的是自动生成的编号；手工填错的编号只能通过"改"来纠正，而现在**没有任何一步文档说 `code` 可以被 PATCH**（`equipment_search_idx` 包含 `code`，说明它预期可变；但 3.5 规则 2 又说不改）。建议明确成一句话：

> **自动生成的编号永不改变；手工填写的编号可以修改（仍受 `UNIQUE (workspace_id, code)` 约束）。** 改编号要写进 `equipment_history.detail.changed`（因为它出现在纸质出库单上，属于"谁也不许赖账"的字段）。

若产品连"改编号"也不接受，那么唯一的出路就是归档 + 换一个可用的编号，接受那个被占用的编号永久作废——**这不需要任何 DDL 改动，只是让用户知情。**

#### 待拍板（产品决定）

| # | 要拍的事 | 选项 | 建议 |
|---|---|---|---|
| **P9** | 录错且**从未出过库**的器材，要不要允许"撤销编号"（真删或让出编号）？ | (a) 不允许，编号永久占用（本文建议）／(b) 允许 PATCH `code`（不释放旧编号）／(c) 唯一约束改部分索引，归档即让出编号 | **(b)**。它是修复"编号录错"的正确工具：不需要删任何行、不弱化唯一性、不留两行同号的歧义，而且历史仍能通过 `equipment_id` 精确指认。(c) 的代价见上；它还有一个更隐蔽的坏处：**"CAM-003 是谁"这个问题在被问第二次时会得到两个答案**（老的归档件与新的活跃件），而纸质出库单上只有一个 CAM-003 |
| **P10** | 归档件要不要有一个"永久删除"的运维出口（比如误建的测试数据）？ | (a) 不要，只能归档（本文建议）／(b) 提供一个仅 OWNER、仅对"零引用器材"（无 history、无 checkout_item、无 occupancy、无 maintenance、无 request_item）的真删 | **(a)**。判据听起来很干净，但"零引用的器材"实际上只有"刚建完就发现错、连 CREATE 历史都不想要"这一种，而它刚好是"改一下就行"的场景。(b) 的价值不足以换一个新接口 + 一套引用检查 + 一个新的审计语义 |

---

## 4. 建议拍板清单

按"是否需要产品回答"分组。**技术决定**部分我建议直接采纳并落到迁移里，不需要产品参与；**产品决定**需要在实现前收敛，其中 **P1 与 P9 会影响 DDL，属于阻塞项**（P3 只在产品否决建议方案时才变成阻塞）。

### 4.1 纯技术决定（本文已定，请复核）

| # | 决定 | 理由位置 |
|---|---|---|
| T1 | 34 处外键全部显式命名并写 `ON DELETE RESTRICT`，**唯一例外**：`app_user_session.user_id` 保留 `CASCADE` | 2.2、2.3 |
| T2 | 推翻两处 CASCADE：`request_item.request_id`、`checkout_item.checkout_id` 改 `RESTRICT` | 2.4 |
| T3 | 归档**不释放** `equipment.photo_id`；照片的回收只发生在"用户显式清空引用 + 下一轮孤儿清理"这条路径上 | 3.3 |
| T4 | `equipment.photo_id` 保持可空，**不选 `ON DELETE SET NULL`** | 2.2 #14、3.3② |
| T5 | 系统**不提供**任何 `DELETE` 接口（分类 / 项目 / 器材 / 账号） | 3.1、3.2、3.4、3.5 |
| T6 | 新增"归档前必须无未归还出库、无未关闭维修"两条应用层检查，以 `checkout_item` 为准而非仅看 `status` | 2.5 |
| T7 | 新增 `DomainError` 变体与 `23503` 的 SQLSTATE 翻译（见第 5.5 节） | 5.5 |
| T8 | 把 5.3 与第 8 节的"删除"措辞统一为"归档"，并在 `BatchRejected` 的文案里逐件带上**理由** | 1.3、5.5、5.6 #7 |

### 4.2 产品决定（待拍板）

| # | 问题 | 建议 | 是否阻塞 DDL |
|---|---|---|---|
| **P1** | 分类要不要"彻底删除"？ | 只停用，永不删除 | **是**（决定是否加 `category.disabled_at`） |
| **P2** | 删除这个概念还需不需要存在？ | 不需要——`DISABLED` / `SOLD` / `archived_at` / `disabled_at` 已覆盖全部用例 | 否（是口径决定） |
| **P3** | 归档项目占用同名怎么办？ | v1 保持 `UNIQUE (workspace_id, name)` + 明确文案 | 否（若要改则是部分唯一索引） |
| **P4** | 项目要不要真删除？ | 不要，`status='ARCHIVED'` 足够 | 否 |
| **P5** | 已出售器材的照片要不要 N 年后放开？ | v1 不放开 | 否 |
| **P6** | 已归档/已出售器材的照片在界面上怎么显示？ | 待定；技术侧无差别 | 否 |
| **P7** | 要不要账号匿名化入口？ | 写进文档，不写进 v1 | 否 |
| **P8** | 匿名化后审计保留多久？ | 永久 | 否 |
| **P9** | 录错且从未出库的器材能否"撤销编号"？ | 允许 PATCH `code`，不释放旧编号 | **是**（决定唯一约束是否改部分索引） |
| **P10** | 要不要"零引用器材"的永久删除出口？ | 不要 | 否 |
| **P11** | 匿名化后离职者的登录名能否复用？ | 可复用 | 否 |

### 4.3 与既有决定的相容性声明

本文的**全部建议都与既有决定相容**。有六处是**补充**（把原文的一句话变成可执行的机制），另有**两处是推翻**——就是 2.4 那两处 `ON DELETE CASCADE`。理由都写在对应位置：

| 既有决定 | 本文的关系 |
|---|---|
| 归档代替删除（D6 / 3.4） | **相容并强化**：把"器材永远删不掉"从正文的一句话变成 `RESTRICT` 的数据库事实 |
| 账号只停用（9 / 10.6） | **相容并延伸**：补出"如果坚持要删"的匿名化路径，且不真删 |
| 批量操作不做部分成功（C4 / 5.3） | **相容并延伸**：5.3 的"删除"实为归档，仍整批拒绝；建议把文案改成逐件带理由（现在是单一理由"出库中"） |
| 孤儿清理不在请求路径上删文件（6.5 / 11） | **相容并强化**：归档不释放引用 ⇒ 请求路径上连"引用被释放"这件事都不发生；照片的回收完全由用户显式操作 + 定时任务驱动 |
| `request_item.request_id ... ON DELETE CASCADE`（3.3） | **推翻**。理由：它会静默抹掉审批事实，且让 `occupancy` 对账任务在事后报出无法追溯的偏差。申请单没有删除功能，CASCADE 没有服务对象 |
| `checkout_item.checkout_id ... ON DELETE CASCADE`（3.3） | **推翻**。理由：它会静默抹掉出库事实，制造"历史说出过库、出库表里查不到"的两份真相——正是 10.1"唯一的叙事主线"要避免的 |
| `equipment(workspace_id, code)` 表级唯一约束（3.3） | **不推翻**（P9 建议选 (b)）。但需要补一句话：**归档行仍占编号**，且自动编号与手填编号的可改性不同 |
| 3.3 正文"历史表……不可级联删"（413 行） | **相容**：现状（无 `ON DELETE` ⇒ `NO ACTION`）已经满足，但建议写成 `RESTRICT`，原因是 `NO ACTION` 允许同事务内先删父行后删子行而静默成功（2.3） |

---

## 5. 需要的 DDL 变更

### 5.1 迁移文件

新建 `migrations/0002_cascade_rules.sql`。**全部 forward-only、纯 SQL**（Rust 篇 7.3）。下面给出完整 SQL。

```sql
-- migrations/0002_cascade_rules.sql
-- 删除与级联规则：见 docs/proposals/07_cascade_rules.md
--
-- 本文的决定：
--   ① 除 app_user_session 外，所有外键显式命名并 ON DELETE RESTRICT
--   ② 推翻两处 CASCADE（request_item.request_id / checkout_item.checkout_id）
--   ③ 新增「停用分类」category.disabled_at
--   ④ 归档不释放照片引用（无需 DDL —— equipment.photo_id 已可空）
--
-- 注意：本迁移不创建任何 DELETE 入口。系统不提供删除功能（见提案 T5）。

-- ---------- ① 分类：停用代替删除 ----------
ALTER TABLE category ADD COLUMN disabled_at timestamptz;

COMMENT ON COLUMN category.disabled_at IS
  '分类停用时间，NULL = 启用。停用后不能被新器材选用；已停用的分类名仍要能解析（历史可读）。见 07_cascade_rules 3.1';

-- ---------- ② 推翻两处 CASCADE ----------
-- 默认约束名按 Postgres 命名规则（<table>_<column>_fkey）。
-- 若 0001 里已显式命名，请改成对应名字。这条 DROP 在干净库上是幂等的：
-- 0001 的 DDL 未命名这两个外键，Postgres 会给出下面这两个默认名。
ALTER TABLE request_item
  DROP CONSTRAINT request_item_request_id_fkey;
ALTER TABLE request_item
  ADD CONSTRAINT fk_request_item_request
  FOREIGN KEY (request_id) REFERENCES request(id) ON DELETE RESTRICT;

ALTER TABLE checkout_item
  DROP CONSTRAINT checkout_item_checkout_id_fkey;
ALTER TABLE checkout_item
  ADD CONSTRAINT fk_checkout_item_checkout
  FOREIGN KEY (checkout_id) REFERENCES checkout(id) ON DELETE RESTRICT;

-- 显式命名唯一一处正确的 CASCADE（语义不变，只为可读与稳定）
ALTER TABLE app_user_session
  DROP CONSTRAINT app_user_session_user_id_fkey;
ALTER TABLE app_user_session
  ADD CONSTRAINT fk_app_user_session_user
  FOREIGN KEY (user_id) REFERENCES app_user(id) ON DELETE CASCADE;

-- ---------- ③ 其余 27 处：显式命名 + RESTRICT ----------
-- 说明：ADD CONSTRAINT ... FOREIGN KEY 不会重复校验既有数据之外的东西；
-- 但如果目标是「给已存在的外键换个删除行为」，必须 DROP 再 ADD。
-- 0001 里的这些外键全部没有命名，Postgres 的默认名是 <table>_<column>_fkey。

-- 工作区
ALTER TABLE app_user          DROP CONSTRAINT app_user_workspace_id_fkey;
ALTER TABLE app_user          ADD CONSTRAINT fk_app_user_workspace
  FOREIGN KEY (workspace_id) REFERENCES workspace(id) ON DELETE RESTRICT;
ALTER TABLE invite            DROP CONSTRAINT invite_workspace_id_fkey;
ALTER TABLE invite            ADD CONSTRAINT fk_invite_workspace
  FOREIGN KEY (workspace_id) REFERENCES workspace(id) ON DELETE RESTRICT;
ALTER TABLE workspace_setting DROP CONSTRAINT workspace_setting_workspace_id_fkey;
ALTER TABLE workspace_setting ADD CONSTRAINT fk_workspace_setting_workspace
  FOREIGN KEY (workspace_id) REFERENCES workspace(id) ON DELETE RESTRICT;
ALTER TABLE category          DROP CONSTRAINT category_workspace_id_fkey;
ALTER TABLE category          ADD CONSTRAINT fk_category_workspace
  FOREIGN KEY (workspace_id) REFERENCES workspace(id) ON DELETE RESTRICT;
ALTER TABLE photo             DROP CONSTRAINT photo_workspace_id_fkey;
ALTER TABLE photo             ADD CONSTRAINT fk_photo_workspace
  FOREIGN KEY (workspace_id) REFERENCES workspace(id) ON DELETE RESTRICT;
ALTER TABLE equipment         DROP CONSTRAINT equipment_workspace_id_fkey;
ALTER TABLE equipment         ADD CONSTRAINT fk_equipment_workspace
  FOREIGN KEY (workspace_id) REFERENCES workspace(id) ON DELETE RESTRICT;
ALTER TABLE project           DROP CONSTRAINT project_workspace_id_fkey;
ALTER TABLE project           ADD CONSTRAINT fk_project_workspace
  FOREIGN KEY (workspace_id) REFERENCES workspace(id) ON DELETE RESTRICT;
ALTER TABLE request           DROP CONSTRAINT request_workspace_id_fkey;
ALTER TABLE request           ADD CONSTRAINT fk_request_workspace
  FOREIGN KEY (workspace_id) REFERENCES workspace(id) ON DELETE RESTRICT;
ALTER TABLE checkout          DROP CONSTRAINT checkout_workspace_id_fkey;
ALTER TABLE checkout          ADD CONSTRAINT fk_checkout_workspace
  FOREIGN KEY (workspace_id) REFERENCES workspace(id) ON DELETE RESTRICT;
ALTER TABLE maintenance       DROP CONSTRAINT maintenance_workspace_id_fkey;
ALTER TABLE maintenance       ADD CONSTRAINT fk_maintenance_workspace
  FOREIGN KEY (workspace_id) REFERENCES workspace(id) ON DELETE RESTRICT;
ALTER TABLE equipment_history DROP CONSTRAINT equipment_history_workspace_id_fkey;
ALTER TABLE equipment_history ADD CONSTRAINT fk_equipment_history_workspace
  FOREIGN KEY (workspace_id) REFERENCES workspace(id) ON DELETE RESTRICT;
ALTER TABLE pending_upload    DROP CONSTRAINT pending_upload_workspace_id_fkey;
ALTER TABLE pending_upload    ADD CONSTRAINT fk_pending_upload_workspace
  FOREIGN KEY (workspace_id) REFERENCES workspace(id) ON DELETE RESTRICT;

-- 账号（审计作者：删账号会让历史失去作者 —— 主稿 10.6）
ALTER TABLE invite            DROP CONSTRAINT invite_created_by_fkey;
ALTER TABLE invite            ADD CONSTRAINT fk_invite_created_by
  FOREIGN KEY (created_by) REFERENCES app_user(id) ON DELETE RESTRICT;
ALTER TABLE invite            DROP CONSTRAINT invite_consumed_by_fkey;
ALTER TABLE invite            ADD CONSTRAINT fk_invite_consumed_by
  FOREIGN KEY (consumed_by) REFERENCES app_user(id) ON DELETE RESTRICT;
ALTER TABLE request           DROP CONSTRAINT request_requester_id_fkey;
ALTER TABLE request           ADD CONSTRAINT fk_request_requester
  FOREIGN KEY (requester_id) REFERENCES app_user(id) ON DELETE RESTRICT;
ALTER TABLE request           DROP CONSTRAINT request_reviewed_by_fkey;
ALTER TABLE request           ADD CONSTRAINT fk_request_reviewed_by
  FOREIGN KEY (reviewed_by) REFERENCES app_user(id) ON DELETE RESTRICT;
ALTER TABLE checkout          DROP CONSTRAINT checkout_created_by_fkey;
ALTER TABLE checkout          ADD CONSTRAINT fk_checkout_created_by
  FOREIGN KEY (created_by) REFERENCES app_user(id) ON DELETE RESTRICT;
ALTER TABLE maintenance       DROP CONSTRAINT maintenance_created_by_fkey;
ALTER TABLE maintenance       ADD CONSTRAINT fk_maintenance_created_by
  FOREIGN KEY (created_by) REFERENCES app_user(id) ON DELETE RESTRICT;
ALTER TABLE equipment_history DROP CONSTRAINT equipment_history_actor_id_fkey;
ALTER TABLE equipment_history ADD CONSTRAINT fk_equipment_history_actor
  FOREIGN KEY (actor_id) REFERENCES app_user(id) ON DELETE RESTRICT;

-- 分类 → 器材：【关键】主稿 3.3 正文那句「历史表必须有外键指向 equipment(id) 且不可级联删」
-- 的姊妹条款：器材的分类永不被置空（category_id 是 NOT NULL），分类永不被删除。
ALTER TABLE equipment         DROP CONSTRAINT equipment_category_id_fkey;
ALTER TABLE equipment         ADD CONSTRAINT fk_equipment_category
  FOREIGN KEY (category_id) REFERENCES category(id) ON DELETE RESTRICT;

-- 照片 → 器材：归档不释放引用，所以 photo 行只由显式的两步运维操作删除。
-- 明确不选 SET NULL（见提案 3.3②）。
ALTER TABLE equipment         DROP CONSTRAINT equipment_photo_id_fkey;
ALTER TABLE equipment         ADD CONSTRAINT fk_equipment_photo
  FOREIGN KEY (photo_id) REFERENCES photo(id) ON DELETE RESTRICT;

-- 项目
ALTER TABLE request           DROP CONSTRAINT request_project_id_fkey;
ALTER TABLE request           ADD CONSTRAINT fk_request_project
  FOREIGN KEY (project_id) REFERENCES project(id) ON DELETE RESTRICT;
ALTER TABLE checkout          DROP CONSTRAINT checkout_project_id_fkey;
ALTER TABLE checkout          ADD CONSTRAINT fk_checkout_project
  FOREIGN KEY (project_id) REFERENCES project(id) ON DELETE RESTRICT;

-- 申请与出库
ALTER TABLE checkout          DROP CONSTRAINT checkout_source_request_id_fkey;
ALTER TABLE checkout          ADD CONSTRAINT fk_checkout_source_request
  FOREIGN KEY (source_request_id) REFERENCES request(id) ON DELETE RESTRICT;
ALTER TABLE request_item      DROP CONSTRAINT request_item_equipment_id_fkey;
ALTER TABLE request_item      ADD CONSTRAINT fk_request_item_equipment
  FOREIGN KEY (equipment_id) REFERENCES equipment(id) ON DELETE RESTRICT;
ALTER TABLE checkout_item     DROP CONSTRAINT checkout_item_equipment_id_fkey;
ALTER TABLE checkout_item     ADD CONSTRAINT fk_checkout_item_equipment
  FOREIGN KEY (equipment_id) REFERENCES equipment(id) ON DELETE RESTRICT;

-- 占用（equipment 的引用是「器材永不被删」的第三道闸）
ALTER TABLE occupancy         DROP CONSTRAINT occupancy_equipment_id_fkey;
ALTER TABLE occupancy         ADD CONSTRAINT fk_occupancy_equipment
  FOREIGN KEY (equipment_id) REFERENCES equipment(id) ON DELETE RESTRICT;
ALTER TABLE occupancy         DROP CONSTRAINT occupancy_source_request_id_fkey;
ALTER TABLE occupancy         ADD CONSTRAINT fk_occupancy_source_request
  FOREIGN KEY (source_request_id) REFERENCES request(id) ON DELETE RESTRICT;
ALTER TABLE occupancy         DROP CONSTRAINT occupancy_source_checkout_id_fkey;
ALTER TABLE occupancy         ADD CONSTRAINT fk_occupancy_source_checkout
  FOREIGN KEY (source_checkout_id) REFERENCES checkout(id) ON DELETE RESTRICT;

-- 维修
ALTER TABLE maintenance       DROP CONSTRAINT maintenance_equipment_id_fkey;
ALTER TABLE maintenance       ADD CONSTRAINT fk_maintenance_equipment
  FOREIGN KEY (equipment_id) REFERENCES equipment(id) ON DELETE RESTRICT;

-- 历史：本提案最重要的一行。器材永远删不掉，是数据库事实，不是代码纪律。
ALTER TABLE equipment_history DROP CONSTRAINT equipment_history_equipment_id_fkey;
ALTER TABLE equipment_history ADD CONSTRAINT fk_equipment_history_equipment
  FOREIGN KEY (equipment_id) REFERENCES equipment(id) ON DELETE RESTRICT;

-- ---------- ④ 孤儿清理的支撑索引 ----------
-- 【修正】不要加 photo_workspace_idx：`UNIQUE (workspace_id, sha256)` 已经是一个
-- 最左列 workspace_id 的 B-tree 索引，再建一个同前缀的索引纯属冗余。
-- 真正缺的是反方向：6.5 第 2 步要「照片有没有被引用」，而这要 join 器材，
-- 而 equipment.photo_id 上没有任何索引。
CREATE INDEX equipment_photo_idx ON equipment (photo_id) WHERE photo_id IS NOT NULL;
```

> **上面这条索引是优化，不是正确性前提。** 主稿 6.1 的规模是"几百到几千张"，即使在 `equipment` 上做一次顺序扫描也在一毫秒级。列出来是因为**它是本文这几条建议里唯一一个"让 6.5 的引用集合算得清楚"的 DDL 支撑**；如果不想加，删掉它不影响任何语义。

**关于 `DROP CONSTRAINT <默认名>` 的风险**：主稿第 3.3 节 DDL 里这 27 处外键都是**匿名内联 `REFERENCES`**，PostgreSQL 会按 `<table>_<column>_fkey` 命名。Rust 篇 5.1/7.3 要求"每一条约束都显式命名"，但主稿的 DDL 目前只给唯一约束和 `CHECK` 起了名。**更干净的做法是直接在 `0001_init.sql` 里把这些外键一次写对**（因为代码还没实现、还没有任何库存在，见第 5.6 节）——`0002` 是"已经跑过 0001 的库"才需要的形式。**建议：0001 直接采用本文给的 `fk_*` 命名与 `RESTRICT`，不要额外制造一个 0002。** 第 5.6 节给出替换方式。

### 5.2 应用层改动（不是 DDL，但必须同批做）

| 改动 | 内容 |
|---|---|
| `DomainError` 新增变体 | 见 5.5 |
| `AppError::from(sqlx::Error)` | 新增 `23503` 的翻译（见 5.5） |
| 归档路径的前置检查 | 无未归还出库（查 `checkout_item.returned_at IS NULL`）、无未关闭维修（`maintenance.closed_at IS NULL`）；两者任一命中则整批拒绝并**逐件带理由** |
| 分类停用/恢复 | `POST /api/v1/categories/{id}/disable`、`.../restore`；停用前查是否为最后一个启用分类；建档时校验所选分类 `disabled_at IS NULL` |
| 照片清空路径 | PATCH `photoId: null`（主稿 7.6 的三态写入已支持）是**唯一**会让照片进入待删集合的用户操作；日志/文档要写清这一点 |
| 活动流视图 | `actor` 增加 `deleted: bool`（为 3.4 的匿名化预留；v1 恒为 `false`，与 10.6 的"(已停用)"标注共用一套渲染逻辑） |

### 5.3 运维手册补一条（运行时篇 R7.4 的清单）

> **6. 删除一张照片（只有在磁盘真的紧张时才做）**
> 顺序固定：① `UPDATE equipment SET photo_id = NULL WHERE photo_id = <id>;` ② `DELETE FROM photo WHERE id = <id>;` ③ 等下一轮孤儿清理（每周）回收文件，或手工比对 `photos/` 与 `photo` 表。
> **不要**跳过 ①——`fk_equipment_photo` 会让你失败，这是有意的。
> 归档或出售器材**不需要**这一步，照片会一直被引用（见 07_cascade_rules 3.3）。

### 5.4 上线前必须验证的 DB 行为（本文无法本机实测）

本机没有 Postgres（见文首"验证边界"），所以下面四条要在 `sqlx::test` 里各写一条断言（运行时篇 R3.1 真库、R3.2 #6 的同类）：

| # | 断言 | 为什么值得单独测 |
|---|---|---|
| 1 | `DELETE FROM equipment WHERE id = <有历史的件>` → `23503`，且约束名是 `fk_equipment_history_equipment` | 这是"归档代替删除"的**唯一执行者**（主稿 413 的原话）。它一旦失效，整套语义靠纪律维持 |
| 2 | `DELETE FROM category WHERE id = <有器材的分类>` → `23503`，且**已归档的器材也算** | 归档行仍然引用分类，这正是 3.1 里"迁移已归档器材在界面上无处可做"的技术依据 |
| 3 | `DELETE FROM app_user WHERE id = <有历史的账号>` → `23503`；`DELETE FROM app_user_session WHERE user_id = <同一个人>` → 成功 | 10.6 的落点，以及"唯一合法的 CASCADE"确实只有会话 |
| 4 | `DELETE FROM photo WHERE id = <被器材引用的照片>` → `23503`；先 `SET photo_id = NULL` 后 → 成功 | 3.3 的"RESTRICT 保护引用完整性"与"只有显式两步才能删" |

### 5.5 `DomainError` 与状态码（Rust 篇第 5 节的落点）

新增变体（`Display` 直接是中文，Rust 篇 5.2 的硬规则）：

```rust
// domain/error.rs 追加
#[error("分类「{name}」下还有 {count} 件器材，不能删除。若不再使用，请把它停用")]
CategoryInUse { name: String, count: i64 },

#[error("分类「{name}」已停用，不能用于新建器材")]
CategoryDisabled { name: String },

#[error("这是最后一个启用的分类，不能停用——建档时必须选择分类")]
LastActiveCategory,

#[error("维修中的器材不能归档，请先完成维修")]
InMaintenanceNotArchivable,

#[error("以下器材不能归档：{items}")]           // 5.3 的文案统一（原为「不能删除」）
BatchRejected { items: String },

#[error("分类下还有器材，不能删除")]              // 23503 的通用兜底（预检查漏了时）
ReferenceInUse,
```

`IntoResponse` 映射（追加到 Rust 篇 5.2 那张表）：

| 变体 | HTTP | `error.code` |
|---|---|---|
| `CategoryInUse` | **409** | `CATEGORY_IN_USE` |
| `CategoryDisabled` | **422** | `CATEGORY_DISABLED` |
| `LastActiveCategory` | **409** | `LAST_ACTIVE_CATEGORY` |
| `InMaintenanceNotArchivable` | **422** | `IN_MAINTENANCE_NOT_ARCHIVABLE` |
| `BatchRejected` | **422** | `BATCH_REJECTED` |
| `ReferenceInUse` | **409** | `REFERENCE_IN_USE` |

`23503` 的 SQLSTATE 翻译（补进 Rust 篇 5.1 的 `match`）：

```rust
// 23503 = foreign_key_violation。预检查漏了时的兜底，与 23P01 / 23505 同一个出口。
Some("23503") => AppError::Domain(DomainError::ReferenceInUse),
```

**这一条不加，本文讨论的所有"删不掉"都会以 500 的形式到达用户**——而 500 既没有 `requestId` 之外的线索，也没有"请改用停用"这句人话。

### 5.6 建议的主稿改动位置（本文不改，只列清单）

> **建议的实现顺序**：如果 `0001_init.sql` 还没写（主稿"状态：未实现"，所以几乎肯定没写），**直接按本文的命名与 `RESTRICT` 写 `0001`，不要造 `0002`**。下面"第 3.3 节"的改动就是把 5.1 的等价内容写进 0001。

| # | 位置 | 改什么 |
|---|---|---|
| 1 | **主稿 3.3 节 DDL**（第 177–410 行） | 27 处匿名 `REFERENCES` 全部改成 `<列> ... CONSTRAINT fk_<table>_<col> REFERENCES <t>(id) ON DELETE RESTRICT`；`request_item.request_id` 与 `checkout_item.checkout_id` 的 `ON DELETE CASCADE` 改为 `RESTRICT`；`app_user_session.user_id` 保留 `CASCADE` 但显式命名 `fk_app_user_session_user` |
| 2 | **主稿 3.3 节 DDL · `category` 表**（第 226–235 行） | 加 `disabled_at timestamptz` 一列 |
| 3 | **主稿 3.3 节正文**（第 413 行） | 「历史表必须有外键指向 `equipment(id)` 且不可级联删」后面补一句：**"并且 `ON DELETE RESTRICT` 是显式写出的，不依赖 PostgreSQL 的默认 `NO ACTION`——后者允许同一事务内先删器材再删历史而静默成功。"** 同时把"不可级联删"升级为完整表述：**器材在数据库层面无法被删除（五处 `RESTRICT` + 历史表）** |
| 4 | **主稿 3.4 节**（第 415–425 行） | 补三条：①归档前必须无未归还出库（以 `checkout_item` 为准）、无未关闭维修；②**归档不释放照片引用**（指向 6.5 的"被引用的 hash 集合"）；③归档件**仍然占用** `(workspace_id, code)` 与分类引用 |
| 5 | **主稿 3.4 节 或新增 3.6 节** | 新增「停用分类」：`category.disabled_at` 的语义三件套（不能被新器材选 / 已停用仍可解析分类名 / 可恢复）+ 不能停用最后一个启用分类 |
| 6 | **主稿 3.5 节**（第 427–447 行） | 在规则 2 后面补：**"自动生成的编号不可改；手工填写的编号可以改（仍受 `(workspace_id, code)` 唯一约束）。"** 并明确"归档会让出列表位但**不**让出编号"。另：规则 1 那句"删除后序号会被回收复用"现在更准确的说法是"**真删除**会回收编号"，而本系统不提供真删除 |
| 7 | **主稿 5.3 节**（第 534–549 行） | 标题与正文的"删除"全部改为「归档」；`BatchRejected` 的文案从「以下器材不能删除（出库中）：…」改为**逐件带理由**（如「CAM-003（出库中）、CAM-011（维修中）」）。理由：归档的拒绝理由至少三种，单一句"出库中"在维修中/已归还但状态漂移时会给出错误的解释 |
| 8 | **主稿 6.5 节**（第 704–713 行） | 第 2 步"减去数据库中被引用的 hash 集合（`photo` 表）"要写清**"被引用"不区分器材是否已归档/已出售**——归档件也持有引用。这一句是把 3.3 的建议和 6.5 的机制焊在一起的地方 |
| 9 | **主稿 6.5 节末尾** | 补一句「若需要真正回收照片，唯一路径是用户显式清空 `photoId`（7.6 的三态写入）或运维按 07_cascade_rules 5.3 的两步操作；**没有自动回收归档件照片的机制**」 |
| 10 | **主稿 第 8 节 · 表格与 `DomainError`**（第 777–928 行） | 加 `23503 → DomainError::ReferenceInUse`（409）；加 5.5 的六个新变体；`BatchRejected` 的 `#[error]` 措辞改为"归档"。**另：第 920 行的 `equipment_workspace_id_code_key` 与第 932 行自己要求的"显式命名 `equipment_code_uniq`"互相矛盾**——统一成 `equipment_code_uniq`（这是现存的跨节漂移，顺手修） |
| 11 | **主稿 10.6 节**（第 1040–1044 行） | 补"如果坚持要删"的匿名化路径摘要（完整版指向本提案 3.4）：**不真删，匿名化 `username` / `display_name`**；并注明**主理人无法匿名化**（`app_user_owner_not_disabled` 这个 CHECK 会拦下），其离职路径依赖 13.7 的主理人交接——**而 13.7 当前无设计** |
| 12 | **主稿第 13 节未决项** | 新增 13.11「分类能否删除 / 能否停用」（本文 P1、P2）、13.12「项目归档占用同名」（P3）、13.13「编号可否修改 / 归档是否让出编号」（P9）。三条都是**阻塞项**，因为 P1、P9 影响 DDL |
| 13 | **主稿第 14 节差异汇总**（第 1174–1191 行） | 表里已有第 1 行"删除用软删除 → 归档+已出售"。建议补一行：**"交接稿：删除器材；本稿：不提供任何删除功能，器材在数据库层面无法删除（`RESTRICT` 五处 + 历史表）"** ——把 D6 从"语义选择"升级为"数据库事实" |
| 14 | **运行时篇 R7.4**（第 447–455 行） | 清单加第 6 项：见 5.3。另：**R7.4-3 写的"主稿 12.7"应为"主稿 13.7"**（现存交叉引用漂移，顺手修） |
| 15 | **运行时篇 R7.2**（第 425–434 行） | "反向（文件在、库里没有）是安全的"这句仍然成立，但建议补：**"文件在、`photo` 行在、但没有任何器材引用它"是正常的中间态**（用户清空了 `photoId`，等下一轮清理），不是不一致 |
| 16 | **Rust 篇 5.1 / 5.2**（第 322–373 行） | 加 `23503` 翻译与 5.5 的六个变体 + 状态码 |
| 17 | **日志设计 6 节事件清单**（第 221–246 行） | `biz.archive` 现有的 `restore`(bool) 之外建议加 `reason`（`MANUAL` / `DUPLICATE` / `MISTAKE`）——归档是"东西消失"的入口，事后要能回答"为什么归档"。**若不加，至少确保 `biz.reject` 覆盖"归档被拒"（已有 `error_code`，够用）** |

---

## 6. 与并行提案的交汇

写作期间 `docs/proposals/` 下并行存在 `03_retry_policy.md`、`05_owner_toggles.md`、`06_request_rejection.md`（它们是同期由其他作者新增的，不在主稿范围内）。本文与其中两份有**实质交汇**，必须一起读：

| 交汇点 | 对方的立场 | 本文的影响 |
|---|---|---|
| **03 提案 · 4.2「重放一次事务」的安全性** | 把"状态流转（出库、归还、送修、完成、**停用、启用、出售、归档、恢复**）"整体判为**重试安全** | **与本文相容，无需修改**。理由：本文没有给这些操作增加任何新的不可重放副作用。归档重放两次 → 第二次 `archived_at` 已经非空，`from == to` 的幂等语义（Rust 篇 4 第 2 点）覆盖；分类停用/恢复同构 |
| **03 提案 · 第 4.2 节引注** | 举了"**`request_item` 被级联删掉**"作为 `RowNotFound` 的合法来源之一 | **本文的 T2 让这个例子不再成立**（`request_item.request_id` 由 `CASCADE` 改 `RESTRICT`，`request_item` 不会再被级联删除）。**03 提案没有错**——它描述的是现状；但如果本文被采纳，那句话的举例要换一个（比如"器材行被归档后不再匹配部分索引条件"）。**这是一处需要同步的跨文档改动**，容易漏 |
| **05 提案 · E3「`GET /api/v1/categories` 等选器材的辅助读接口」** | 只定义了"申请人 + `can_browse = false`"时的行为（403 `BROWSING_DISABLED`） | **本文的"停用分类"（3.1 建议 2）落在这个端点上，而 05 没有覆盖**：05 的 Q6-B 那张表只按**角色**分叉，没有按**分类状态**分叉。采纳本文后，E3 的响应需要同时满足两个过滤：①`can_browse` 的角色判定（05 的）②分类下拉默认只列 `disabled_at IS NULL`（本文的）。**两个过滤是正交的，不冲突，但必须都写在同一个 handler 里**，否则会出现"停用分类照常出现在新建器材的下拉里"这种只在实现层暴露的漏。建议 05 的 E3 行补一句指向本文 3.1 |
| **06 提案 · 撤回申请要 `DELETE` 对应的 `RESERVATION` 行**（主稿 3.1 的业务事件已列，但实现未写） | 推荐把"`PENDING` 与 `APPROVED` 都可撤回"作为 v2，并要求撤回与交付都锁 `request` 行 | **与本文相容，且本文不碰 `occupancy` 的 `DELETE`**。本文的 `RESTRICT` 只作用于**删 `request` 行**（一个不存在的操作），不作用于**删 `occupancy` 行**（归还与撤回的正常路径，主稿 3.1 第 133–134 行明确列了两次 `DELETE`）。**这两件事必须在文档里分开写**，否则实现者可能以为"给 `request` 加了 `RESTRICT`，撤回就不能删占用了" |

**一句话**：本文的 DDL 变更不影响 03/05/06 的任何推荐，但有**两处文字需要同步改**（03 的举例、05 的 E3 行），否则四份提案放一起会出现"引用了已经不复存在的现状"。

---

## 7. 一页纸结论

1. **删除这个概念在本系统里不需要存在**（P2）。四种"东西消失"的语义已经齐了：`DISABLED`（东西在、不出借）、`SOLD`（东西没了、正常终局）、`archived_at`（录错了、可恢复）、`disabled_at`（账号/分类不再使用）。
2. **分类**：`NOT NULL` 已核实 ⇒ "置空分类"不是选项。建议 `RESTRICT` + **停用分类**（新增 `category.disabled_at`）。编号不受影响，因为编号已是 `equipment.code` 里的字符串，`category` 表不参与运行时编号。
3. **项目**：只归档不删除，`status='ARCHIVED'` 已经存在；与 5.6 的"有器材没归还不能归档"同一条推理链。同名冲突用明确文案（v1），是否改成部分唯一索引是产品决定。
4. **照片**：**归档不释放引用**；`photo_id` 已可空故无 DDL 改动；相同图片被多件器材引用时，删引用**绝不等于**删 `photo` 行。`RESTRICT` 而不是 `SET NULL`——因为它保护的是"没人能静默删掉一张被引用的照片"。
5. **账号**：10.6 确实已决定不删（已逐字核实）。如果产品坚持，正确做法是**匿名化行**（`username = 'deleted:' || id`、`display_name = '已删除账号'`），因为 `equipment_history.actor_id` 是 `NOT NULL` 且删账号会让历史失去作者。主理人无法匿名化，其离职依赖 13.7（当前无设计）。
6. **器材**：归档代替删除（D6/3.4），且**数据库层面已堵死**（3.3 正文 + 五处 `RESTRICT`）。数据累积不是问题（部分索引 + 几千行量级），但**编号空隙**（CAM-002 消失）和**错误编号被永久占用**是真实代价，需要产品接受（P9）。
7. **最容易被漏掉的一步**：27 处外键现在是 `NO ACTION`、`23503` 会变成 500。**加 `RESTRICT` 的名字 + 加 `23503 → 409` 的翻译，是这一整套语义能被用户看见的前提。**

---

## 附录 A：核实方法

「现状」那一列的每一个字都来自下面这几条命令的输出，不是印象：

```bash
# ① 全部外键声明（本文第 2 节的 34 行逐行对应这 30 条）
grep -n "REFERENCES" docs/20260930_BACKEND_v1.md

# ② 全部 ON DELETE（只有 3 处 —— 这就是缺口的核心）
grep -n "ON DELETE" docs/20260930_BACKEND_v1.md
grep -rn "ON DELETE" docs/          # 四份文档一起查，确认没有漏

# ③ 断言"文档没定义"的地方
grep -n "删除\|级联\|CASCADE\|RESTRICT\|归档\|停用" docs/*.md
grep -n "DELETE\|disabled_at\|23503\|软删除" docs/*.md

# ④ 逐个字段的引用面（用来断言"没有第二张表引用 photo/category"）
grep -n "photo_id\|category_id\|project_id\|actor_id" docs/*.md

# ⑤ 主稿行号对照（本文引用的行号）
#   category_id NOT NULL ....... 255
#   photo_id 可空 .............. 263
#   project UNIQUE(ws,name) .... 299
#   checkout.project_id 可空 ... 342
#   request_item CASCADE ....... 327
#   checkout_item CASCADE ...... 358
#   历史表外键不可级联删 ....... 413
#   账号不删除 ................. 1040-1044
#   差异汇总第 1 行 ............ 1180
#   R7.4-3 写"主稿 12.7" ....... 运行时篇 453（应为 13.7）
```

**读法**：①与②的差集就是本文的全部缺口（30 条外键里 27 条没写 `ON DELETE`）。③的每一处命中都在本文里有对应结论；凡③没有命中的主题（分类的删除/停用、项目的删除、照片的释放、`equipment` 的删除），本文的"主稿没有定义"就是由"检索为空"得出的。

**未能核实的**：本机无 Postgres（见文首），所以 2.3 的事务内语义差别与 5.1 的部分唯一索引替代方案属于待验证项，已列进 5.4 的四条断言。
