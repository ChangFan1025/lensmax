# LensMax 后端实现契约

日期：2026-09-30
版本：v1
状态：设计稿。**未实现**。
范围：**接口清单**（全部端点与权限档）、**校验规则**（字段级 + 跨字段 + 产品待定项）、**连接池与超时**、**重试策略**
关系：

| 文档 | 它定什么 | 本稿与它的关系 |
|---|---|---|
| [20260930_BACKEND_v1.md](20260930_BACKEND_v1.md)（主稿） | 数据模型、状态机、写路径、序列化形状、错误处理 | 本稿是**接口层的展开**。读本稿前必须先读主稿第 7 节（共用形状）与第 8 节（错误） |
| [20260930_BACKEND_RUNTIME_v1.md](20260930_BACKEND_RUNTIME_v1.md)（运行时篇） | 配置、测试、并发、故障处理 | 本稿的池与超时是它 R2.1 配置结构体的展开 |
| [20260930_LOGGING_v1.md](20260930_LOGGING_v1.md)（日志篇） | 日志字段契约与事件清单 | 本稿每个端点标注它对应哪个日志事件 |
| [20260930_RUST_IMPL_v1.md](20260930_RUST_IMPL_v1.md)（实现篇） | crate 划分、类型放置、事务模式、extractor | 本稿的「权限档」列直接对应它的三种 extractor |
| [20260930_OPEN_DECISIONS_v1.md](20260930_OPEN_DECISIONS_v1.md) | 需要产品拍板的问题 | 本稿 §2.4 的待定项在那里汇总 |

> **本稿是四份文档的合并产物。** 它起源于七份并行提案（`docs/proposals/`），那些提案现已标注为已并入。**实现时以本稿为准**，提案目录只作审计留底。

---

## 1. 接口清单

### 1.0 读法与命名风格

**总表的「请求体类型」与「响应类型」列给的是类型的职责，不是完整字段表。** `Req*` / `View*` 是指代名，落到代码里遵循实现篇第 2 节的三套类型（Row / Model / View）。主稿已定义过的结构直接指向主稿节号，不重复。

**命名风格（已裁定）**：资源嵌套 + 显式动作段。

```
# ① 读与创建：复数资源集合
GET    /api/v1/equipment                    # 列表
GET    /api/v1/equipment/{id}               # 详情
POST   /api/v1/equipment                    # 建档
PATCH  /api/v1/equipment/{id}               # 改描述性字段（主稿 7.6 三态）
DELETE /api/v1/equipment/{id}               # 归档，不是物理删除（主稿 3.4）

# ② 状态流转与一次性业务动作：POST /资源/{id}/{动作名词}
POST   /api/v1/equipment/{id}/maintenance
POST   /api/v1/requests/{id}/approve
POST   /api/v1/requests/{id}/deliver

# ③ 作用于集合的批量动作：POST /资源/batch
POST   /api/v1/equipment/batch              # body 里的 action 是封闭枚举
POST   /api/v1/checkouts                    # 批量出库（天然是集合操作，见下）
```

**为什么单件用嵌套、批量用集合**：两者的区别是真实的区别——「对一件做一件事」和「对一批做同一件事」。嵌套路径让**被操作对象的 ID 写在路径里**，于是权限裁剪（主稿 7.7 按器材）、审计、日志的单值 `ids.equipment_id` 都天然对齐；批量则不需要这个 ID。

**出库是唯一的例外**：`POST /api/v1/checkouts` 是集合式，因为一次出库天生可以带多件器材（交接稿 5.2「勾选多件后走批量出库」），而主稿 5.1 的 `CheckoutCmd { equipment_ids }` 就是复数。**同一批器材产生的是一张出库单**，所以资源是 `checkout` 而不是"某件器材的出库动作"。

**动作名词一律单段、英文、小写，且一个概念只有一个名词**：出库永远是 `checkout`，绝不出现 `lend`／`borrow`／`check-out`。这与主稿 13.1 的术语立场一致（代码层 `checkout`/`return`，界面层「出库/归还」）。

**动作名词是编译期常量，这是安全属性。** 主稿 7.5 与实现篇 7.1 都把「拼 SQL 只能用编译期常量、用户输入只走 `.bind()`」列为硬规则；路由同理——动作名词写在路径里意味着它是代码里的字面量，放在 body 里就是一个会随前端需求长出 `"checkout2"` 的自由字符串。

### 1.1 权限档

四档，取自实现篇 6.1–6.2 的三个 extractor，加一个不需要会话的档：

| 档 | 对应 extractor | 含义 |
|---|---|---|
| `PUBLIC` | 无 | 不需要会话。仅登录、bootstrap、健康检查、接受邀请。 |
| `REQUESTER` | `Actor` | **需要登录**，三种角色都能调。返回体按角色裁剪（主稿 7.7）。 |
| `ADMIN` | `Admin(pub Actor)` | 需要 `OWNER` 或 `ADMIN`。申请人调用返 403。 |
| `OWNER` | `Owner(pub Actor)` | 只允许 `OWNER`。 |

**`REQUESTER` 这个名字容易误读**，它的语义是「需登录、不按角色拒绝」：器材列表/详情的路由档位是 `REQUESTER`，但**是否放行以及返回什么由运行时的两个开关 + 角色裁剪决定，不由路由决定**。

> **`Admin` 档含 OWNER，没有「仅 ADMIN 不含 OWNER」这一档。** 所以凡是产品说「OWNER 和 ADMIN 都能做」的，落成 `Admin`；说「只有 OWNER」的，落成 `Owner`。**目前没有反例**，但这个不对称要注意。

**申请人拿不到的东西是四类，不是一类**，实现时逐条守住：

1. **字段**：`purchasePriceCents`、`purchaseDate`、`serialNumber`、`internalNote`、`createdAt`、`updatedAt` —— 由 `EquipmentRequester` 类型保证（主稿 7.7、实现篇 2.1）。
2. **整个入口**：`GET /api/v1/activity` 一律 `ADMIN`。主稿 10.4 的原文是「**决定：申请人完全不给历史入口**」，不是「过滤条目」。
3. **两个开关**：`requester_can_browse = false` → 器材读接口 403 `BROWSING_DISABLED`；`requester_can_request = false` → 提交申请 422 `REQUESTING_DISABLED`。
4. **出库清单**：`GET /api/v1/checkouts/{id}/manifest` 是 `ADMIN`，因为清单含序列号而签字人可能是申请人（主稿 7.8）。**这正是 7.8 禁止「前端拿器材列表 DTO 拼清单」的原因。**

### 1.2 分页与幂等的统一约定

**分页**：所有列表端点一律游标分页，**不用 offset**（主稿 7.5 的理由是正确性不是性能）。三件事必须一致：

| 项 | 约定 |
|---|---|
| 请求参数 | `cursor`（不透明字符串，服务端 base64url，客户端不解析）+ `limit`（默认 50，**上限 100**） |
| 响应 | `{ "items": […], "nextCursor": "…" \| null }`。`nextCursor` 为 `null` 表示到底，**不用空 items 表示到底** |
| 游标键 | 统一取 **`(created_at, id)` 降序** |

**为什么是 `(created_at, id)` 而不是 `(code)`**：主稿 3.3 已经为它建好了部分索引 `equipment_list_idx ON equipment (workspace_id, created_at DESC, id DESC) WHERE archived_at IS NULL`。按 `code` 排序没有对应索引，而且 `code` 有字符串序问题——主稿 3.5 允许序号超过三位，于是 `CAM-1000` 会排在 `CAM-999` 前面。

**幂等**：主稿运行时篇 R4.4 说「只加在三个地方」，但后半句又说「照片上传本来就已经幂等，所以实际上只有两处需要」。

**裁定：四个端点都收 `Idempotency-Key`**：

- `POST /api/v1/checkouts`
- `POST /api/v1/photos`
- `POST /api/v1/equipment/import/commit`
- **`POST /api/v1/equipment`（建档）** ← 第 4 个

**为什么建档也要**：提案 03 发现一个真实缺口——**建档在编号留空时会自动生成编号，所以客户端超时重发会静默建出两条，且不撞任何唯一约束**。这是最容易发生的一种"网络抖动产生脏数据"。照片上传的幂等性来自内容寻址（主稿 6.2），那只保证「不产生重复 blob」，**不保证「不产生重复的 `photo` 行、不重复记 `upload.ok`」**，所以它也该带 key。

带上 key 之后这四处是同一个机制，运维排查时不必记「哪三个要带头、哪一个只是碰巧安全」。

> **承载幂等键的表尚未定义**（存哪、TTL 多久、同 key 不同 body 怎么办）。这是主稿需要补的一项，见 §2.4 待定项 D-12。本稿**不自行设计该表**。

### 1.3 端点总表

#### 1.3.1 健康检查

| 方法 | 路径 | 权限档 | 用途 | 备注 |
|---|---|---|---|---|
| GET | `/healthz` | PUBLIC | 容器 liveness。**不碰数据库** | 运行时篇 R1.9。**不带 `/api/v1`**，它是运维端点不是业务 API |
| GET | `/readyz` | PUBLIC | readiness：数据库可连 + 迁移已到最新 | 运行时篇 R1.9。不就绪时 503。**必须用自己的 1s `acquire_timeout`**（见 §3.6） |

> **照片读取不在这里。** 主稿 6.6 决定 `GET {PHOTO_BASE_URL}/…`（生产为 `/photos/*`）由 Caddy 静态直发、**不经过 Axum、不做鉴权**（照片是内容寻址且三种角色都可见）。**推论**：照片 URL 一旦下发就永久有效，所以器材详情里的 `photo` 字段**可能指向一个已 404 的 URL**（运行时篇 R7.2 的文件丢失），而接口层对此无能为力——**接口层不许把 `photo` 置空来掩盖它**。

#### 1.3.2 账号：登录、会话、bootstrap

| 方法 | 路径 | 权限档 | 用途 | 请求体 | 响应 | 备注 |
|---|---|---|---|---|---|---|
| POST | `/api/v1/auth/login` | PUBLIC | 登录，建会话，下发 cookie | `ReqLogin{ username, password }` | `ViewMe` | 日志 `auth.login`/`auth.fail`。**`auth.fail` 不记用户名原文**（日志篇 8.2 L2） |
| POST | `/api/v1/auth/logout` | REQUESTER | 登出，撤销当前会话 | — | 204 | 天然幂等（已登出再登出同样 204） |
| GET | `/api/v1/auth/me` | REQUESTER | 当前账号 + 角色 + 工作区 | — | `ViewMe` | 前端启动时第一个调用。**不含权限矩阵**——`actions` 是逐资源的（主稿 D8） |
| POST | `/api/v1/bootstrap/owner` | PUBLIC | 建**第一个**主理人，并创建 workspace | `ReqBootstrap{ workspaceName, username, displayName, password }` | `ViewMe` | 主稿第 9 节。**用过即永久失效**。同时回答「workspace 行从哪来」 |
| POST | `/api/v1/auth/password` | REQUESTER | 自助改密码（需旧密码） | `ReqPasswordChange{ oldPassword, newPassword }` | 204 | 运行时篇 R7.4 第 2 条点名「一定会做、没文档一定会做错」。**成功后撤销其它会话** |
| POST | `/api/v1/users/{id}/password` | ADMIN | 管理员重置密码 | — | `{ temporaryPassword }` | 返回一次性密码，**只在响应里出现一次**（同邀请 token 的处理） |

> **会话 cookie 不是端点。** 主稿第 9 节：`httpOnly; Secure; SameSite=Lax`，每请求查一次 `app_user_session`。**「续期」也没有端点**——`expires_at` 固定（运行时篇 R2.1 的 `session_ttl_days` 默认 14 天），到期即 401。

#### 1.3.3 账号管理（邀请 / 成员 / 角色）

| 方法 | 路径 | 权限档 | 用途 | 请求体 | 响应 | 备注 |
|---|---|---|---|---|---|---|
| GET | `/api/v1/invites` | ADMIN | 邀请列表 | — | `{items:[ViewInvite]}` | 游标 `(created_at,id)`。**不返回明文 token** |
| POST | `/api/v1/invites` | ADMIN | 生成邀请链接 | `ReqInviteCreate{ role }` | `ViewInviteWithToken` | **`token` 只在这个响应里出现一次**。解决「邀请邮件没有邮件服务」 |
| DELETE | `/api/v1/invites/{id}` | ADMIN | 撤销未使用的邀请 | — | 204 | 需要 `invite.revoked_at` 列（本稿 §2.5 DDL 已补） |
| POST | `/api/v1/invites/{token}/accept` | PUBLIC | 用邀请码建号 | `ReqAcceptInvite{ username, displayName, password }` | `ViewMe` | 一次性 + 有 `expires_at`。**token 在路径里，日志不许记它** |
| GET | `/api/v1/users` | ADMIN | 成员列表 | — | `{items:[ViewUser]}` | 含 `disabledAt`，否则界面无法显示「(已停用)」（主稿 10.6） |
| GET | `/api/v1/users/{id}` | ADMIN | 成员详情 | — | `ViewUser` | `actor` scope 的活动流需要一个落点显示「这是谁」 |
| PATCH | `/api/v1/users/{id}` | ADMIN | 改 `displayName` | `ReqUserPatch{ displayName? }` | `ViewUser` | `app_user` 只有 `display_name` 是可改的描述性字段 |
| POST | `/api/v1/users/{id}/disable` | ADMIN | 停用账号（**同时删除其全部会话**） | — | `ViewUser` | 主稿第 9 节：「停用账号时删除其全部会话——这是选有状态会话的唯一理由」。**主理人不可停用** → `OWNER_NOT_DISABLEABLE` |
| POST | `/api/v1/users/{id}/enable` | ADMIN | 恢复账号 | — | `ViewUser` | 「停用不是删除」 |
| POST | `/api/v1/users/{id}/role` | ADMIN | 改角色 | `ReqChangeRole{ role: ADMIN\|REQUESTER }` | `ViewUser` | **OWNER 不可被改**。`role` 取值里**没有 `OWNER`**——转移主理人无设计（主稿 13.7） |
| DELETE | `/api/v1/users/{id}/sessions` | ADMIN | 踢掉某人所有会话 | — | 204 | 运维要用的第三只手（运行时篇 R7.4） |

> **没有 `DELETE /api/v1/users/{id}`。** 主稿 10.6：`equipment_history.actor_id` 指向 `app_user`，删行会让历史失去作者。**这不是遗漏，是设计。**

#### 1.3.4 器材：建档、列表、详情、编辑、归档

| 方法 | 路径 | 权限档 | 用途 | 请求体 | 响应 | 分页 | 幂等 | 备注 |
|---|---|---|---|---|---|---|---|---|
| GET | `/api/v1/equipment` | REQUESTER | 列表：搜索 + 筛选 + 排序 | — | `{items:[EquipmentView], nextCursor}` | `(created_at,id)` 降序，`limit` 默认 50 上限 100 | — | 参数 `q`/`category`/`status`/`brand`/`sort`/`cursor`/`limit`/`includeSold`。`QueryBuilder` 每段 SQL 是编译期常量，用户输入只走 `.bind()`（实现篇 7.1）。**`requester_can_browse=false` 时 403** |
| POST | `/api/v1/equipment` | ADMIN | 建档。`code` 留空则自动生成 | `ReqEquipmentCreate` | `EquipmentView` | — | **`Idempotency-Key`** | 主稿 3.5 的 `category.next_seq` 分配。**编号格式不做校验**（手填 `CAM-001-备用机` 合法）。返回新对象 |
| GET | `/api/v1/equipment/{id}` | REQUESTER | 详情 | — | `EquipmentView` | — | — | 角色裁剪在此生效。**已归档返 404** |
| PATCH | `/api/v1/equipment/{id}` | ADMIN | 改描述性字段（三态） | `ReqEquipmentPatch` | `EquipmentView` | — | — | **一次提交写一条 `UPDATE` 历史，只记真正变了的字段**（主稿 5.5）。日志 `biz.update` 记 `changed_count` 与 `fields[]`，**不记值** |
| DELETE | `/api/v1/equipment/{id}` | ADMIN | **归档**（`archived_at`），非物理删除 | — | 204 | — | — | **出库中禁止归档** → `CHECKED_OUT_NOT_ARCHIVABLE`。这个动词是 DELETE 但语义是归档 |
| POST | `/api/v1/equipment/{id}/restore` | ADMIN | 恢复归档 | — | `EquipmentView` | — | — | `equipment_history.action = 'RESTORE'` |
| POST | `/api/v1/equipment/batch` | ADMIN | 批量归档/停用/恢复/出售 | `ReqBatch{ action, equipmentIds[] }` | `ViewBatchResult` | — | — | **整批拒绝、点名违规件、不做部分成功**（主稿 C4）。`action` 是封闭枚举 `ARCHIVE`/`RESTORE`/`DISABLE`/`ENABLE`/`SOLD`。**取锁按 `equipment.id` 升序**（主稿 5.2） |

> **交接稿的「批量删除」在这里落成 `action=ARCHIVE`。** 主稿 3.4 与第 14 节差异汇总第 1 条已把「软删除」改写为「归档 + 已出售」，所以没有独立的删除动作。主稿 5.3 仍在使用「批量删除」这个词，**措辞残留，应统一为归档**（已在本稿 §4 记录）。

#### 1.3.5 出库与归还

| 方法 | 路径 | 权限档 | 用途 | 请求体 | 响应 | 幂等 | 备注 |
|---|---|---|---|---|---|---|---|
| POST | `/api/v1/checkouts` | ADMIN | **出库**（直接出库，非申请路径） | `ReqCheckout{ equipmentIds[], projectId?, projectName, userName, checkoutDate, expectedDate?, note? }` | `ViewCheckout` | **`Idempotency-Key`** | `expectedDate` 可空 = 占用到 `infinity`（主稿 3.1）。**档期冲突返 409 `RESERVED`，不是 422** |
| GET | `/api/v1/checkouts` | ADMIN | 出库单列表 | — | `{items:[ViewCheckout], nextCursor}` | — | 回答「哪些单还开着、按项目/使用人筛选」。**与出入库记录页不同**，见 §1.6 |
| GET | `/api/v1/checkouts/{id}` | ADMIN | 出库单详情（含逐件归还状态） | — | `ViewCheckoutDetail` | — | 部分归还由「还有 `returned_at IS NULL` 的行」推导，**不建状态字段**（主稿 5.4） |
| GET | `/api/v1/checkouts/{id}/manifest` | ADMIN | **出库交接清单**（打印专用 DTO） | — | `ViewManifest` | — | **角色守卫收在这一个 handler 里**；不允许前端拿器材列表 DTO 拼清单（含序列号，签字人可能是申请人） |
| POST | `/api/v1/checkouts/{id}/items/{itemId}/return` | ADMIN | **逐件归还** | `ReqReturn{ returnNote? }` | `ViewCheckoutDetail` | — | 主稿 5.4 七步。**必须校验 `itemId` 属于该 `checkout`** —— 不校验会越权归还（本稿 §4 记录） |

#### 1.3.6 送修、维修完成、停用、恢复、出售

| 方法 | 路径 | 权限档 | 用途 | 请求体 | 响应 | 备注 |
|---|---|---|---|---|---|---|
| POST | `/api/v1/equipment/{id}/maintenance` | ADMIN | **送修**（`IN_STOCK` → `MAINTENANCE`） | `ReqMaintenanceOpen{ fault, vendor?, sentDate, costCents?, note? }` | `ViewMaintenance` | `fault` 必填。**同一件同时只能有一条未关闭维修** → `OPEN_MAINTENANCE_EXISTS` |
| POST | `/api/v1/maintenance/{id}/close` | ADMIN | **维修完成** | `ReqMaintenanceClose{ outcome: BACK_TO_STOCK\|DISABLED, note? }` | `ViewMaintenance` | 状态机允许 `MAINTENANCE→IN_STOCK` 与 `MAINTENANCE→DISABLED` |
| GET | `/api/v1/maintenance/{id}` | ADMIN | 维修记录详情 | — | `ViewMaintenance` | **新增**。主稿 10.1 要求活动流的维修条目「点开后显示故障说明/维修单位/费用」，而原先没有任何端点能取它 → `ref` 点不开 |
| POST | `/api/v1/equipment/{id}/disable` | ADMIN | **停用**（`IN_STOCK` → `DISABLED`） | `ReqNote{ note? }` | `EquipmentView` | 「东西还在，不出借」 |
| POST | `/api/v1/equipment/{id}/enable` | ADMIN | **恢复**（`DISABLED` → `IN_STOCK`） | — | `EquipmentView` | — |
| POST | `/api/v1/equipment/{id}/sell` | ADMIN | **出售**（`IN_STOCK`/`DISABLED` → `SOLD`） | `ReqNote{ note? }` | `EquipmentView` | **`SOLD` 是状态机终点**。「已出售」不等于归档 |

#### 1.3.7 照片、导入、导出

| 方法 | 路径 | 权限档 | 用途 | 请求体 | 响应 | 幂等 | 备注 |
|---|---|---|---|---|---|---|---|
| POST | `/api/v1/photos` | ADMIN | 照片上传（multipart，**字段名 `file`**，单文件） | `multipart/form-data` | `PhotoRef` | **`Idempotency-Key`** | **8 MiB 上限、只收 JPEG、校验魔数不信任 Content-Type**。内容寻址 ⇒ 同图重传不做任何写。返回 `sha256`。**上传与器材保存是两个请求**，本端点不改 `equipment.photo_id` |
| POST | `/api/v1/equipment/import/preview` | ADMIN | CSV 导入**第一阶段**：解析 + 校验，不落库 | `multipart/form-data` | `ViewImportPreview` | — | 检测编码并**回显**、金额规范化成整数分（四舍五入不截断）、失败行标红**不静默当 0**。**预览只回前 12 行** |
| POST | `/api/v1/equipment/import/commit` | ADMIN | CSV 导入**第二阶段**：提交，全有或全无 | `ReqImportCommit` | `ViewImportResult` | **`Idempotency-Key`** | **建议重校验**（运行时篇 R5.3）。1000 行上限、请求体 5 MiB 上限（**读流时生效**）。去重策略见 §2.4 D-03 |
| GET | `/api/v1/equipment/import/template` | ADMIN | 下载 CSV 模板 | — | `text/csv`（**带 BOM**） | — | 服务端生成而非前端静态文件：**列名必须与解析逻辑同源**。带 BOM 是因为模板里全是中文列名，不带 BOM 的 UTF-8 会被 Excel 按 GBK 打开成乱码（运行时篇 R5.1） |
| GET | `/api/v1/export` | ADMIN | **数据导出备份** | — | 文件流 | — | 主稿 13.2 是阻塞项。**默认：原始数据（含价格/序列号），文件头写入导出人/时间/范围**。见 §2.4 D-02 |

#### 1.3.8 申请：申请人侧与管理员侧

| 方法 | 路径 | 权限档 | 用途 | 请求体 | 响应 | 分页 | 备注 |
|---|---|---|---|---|---|---|---|
| POST | `/api/v1/requests` | REQUESTER | 申请人提交申请 | `ReqRequestCreate{ equipmentIds[], projectName, useStart, useEnd, note? }` | `ViewRequestRequester` | — | **`requester_can_request=false` → 422 `REQUESTING_DISABLED`**。**至少一件器材**（新增约束）。`use_end >= use_start` 由 CHECK 兜底 |
| GET | `/api/v1/requests/mine` | REQUESTER | **我的申请**（只看自己） | — | `{items:[ViewRequestRequester], nextCursor}` | `(created_at,id)` | **路径用 `/mine` 而不是 `?requesterId=me`**：让「授权来源是会话本身」写在路径里，避免一个能被改参数绕过的端点。服务端**不接受**客户端指定 `requesterId`。**不受两个开关影响** |
| GET | `/api/v1/requests/{id}` | REQUESTER | 申请详情 | — | `ViewRequestRequester` 或 `ViewRequestAdmin` | — | 申请人只能看自己的，**否则 404 不是 403**（403 会泄漏「这个 ID 存在」） |
| POST | `/api/v1/requests/{id}/withdraw` | REQUESTER | **撤回申请** | — | `ViewRequestRequester` | — | `status = WITHDRAWN`。**同时 DELETE 相关 `occupancy` 行**（主稿 3.1 映射表）。**撤回入口必须独立于两个开关** —— 关闭浏览不该让档期释放路径断掉 |
| GET | `/api/v1/requests` | ADMIN | **管理员申请列表**（待审队列） | — | `{items:[ViewRequestAdmin], nextCursor}` | 默认未审优先 | 筛选 `status`/`requesterId`/`projectId` |
| POST | `/api/v1/requests/{id}/approve` | ADMIN | **审批**（含部分通过） | `ReqApprove{ items: [{equipmentId, approved, rejectReason?}], reviewNote? }` | `ViewRequestAdmin` | — | **部分通过不进状态枚举**——取消勾选 = 不 INSERT `occupancy`。`status` 由 `approved` 列**推导**，不直接写。**批准即写 `RESERVATION`，区间 `[use_start, use_end + 1天)`** |
| POST | `/api/v1/requests/{id}/reject` | ADMIN | **整体拒绝** | `ReqReject{ reviewNote, items?: [{equipmentId, rejectReason?}] }` | `ViewRequestAdmin` | — | **独立端点而非 `approve` 全 false**：只有独立端点才能无歧义地产生 `REJECTED`（详见 §2.3.4）。服务端仍**只有一个写路径**，两者都是它的入口 |
| POST | `/api/v1/requests/{id}/deliver` | ADMIN | **办理交付** | `ReqDeliver{ equipmentIds[], checkoutDate?, expectedDate?, userName }` | `ViewCheckout` | — | 该 `occupancy` 行 UPDATE 成 `kind='CHECKOUT'`、起点改为实际出库日期、写 `source_checkout_id` |

> **申请人看不到的字段在申请模块里最密集**：看不到器材价格/序列号/内部备注，看不到全库历史，也**看不到时间**——主稿 7.7 把 `createdAt`/`updatedAt` 划给管理员视图。所以 `ViewRequestRequester` **不带 `reviewedAt`**。

#### 1.3.9 项目

| 方法 | 路径 | 权限档 | 用途 | 请求体 | 响应 | 备注 |
|---|---|---|---|---|---|---|
| GET | `/api/v1/projects` | ADMIN | 项目列表 | — | `{items:[ViewProject], nextCursor}` | 筛选 `status`/`projectType`。**为什么不给申请人**：`project.client` 是客户名、`project.note` 是备注，按主稿 7.7 的分工属内部信息；申请人提交申请时用的是 `projectName` 自由文本，不需要看项目库 |
| POST | `/api/v1/projects` | ADMIN | 新建项目 | `ReqProjectCreate{ name, client?, projectType, startDate?, endDate?, note? }` | `ViewProject` | `(workspace_id, name)` 唯一 |
| GET | `/api/v1/projects/{id}` | ADMIN | 项目详情 | — | `ViewProjectDetail` | 含历史出库记录数 |
| PATCH | `/api/v1/projects/{id}` | ADMIN | 改项目字段（三态） | `ReqProjectPatch` | `ViewProject` | 与器材编辑同形 |
| POST | `/api/v1/projects/{id}/status` | ADMIN | 项目状态流转 | `ReqProjectStatus{ status }` | `ViewProject` | 合法性表见 §2.3.5 |
| DELETE | `/api/v1/projects/{id}` | ADMIN | **归档**（`status='ARCHIVED'`），非物理删除 | — | 204 | **不是物理删除**——与主稿 5.6「有器材没归还不能归档」同一条推理链：删除比归档更彻底，门槛不可能更低 |

#### 1.3.10 分类

| 方法 | 路径 | 权限档 | 用途 | 请求体 | 响应 | 备注 |
|---|---|---|---|---|---|---|
| GET | `/api/v1/categories` | REQUESTER | 分类列表 | — | `{items:[ViewCategory]}` | **无分页**（十量级小集合，下拉要一次拿全）。**两种过滤正交，必须都写在同一个 handler 里**：① `availableOnly=true`（默认）→ 只列"可用"的分类（派生规则见主稿 3.7）；`availableOnly=false` → 列所有出现过器材的分类（器材库筛选要用）② `requester_can_browse = false` 时对 `REQUESTER` 返 403 `BROWSING_DISABLED`。给申请人的视图**不含 `nextSeq`**（内部计数器，泄漏「你们有多少件器材」） |
| POST | `/api/v1/categories` | ADMIN | 新建分类 | `ReqCategoryCreate{ name, codePrefix }` | `ViewCategory` | `pinned` 默认 true，所以新分类立刻可用。`code_prefix` 受 `^[A-Z0-9]{1,5}$` 约束（**填小写返 400 而不是 500**，主稿 8.1） |
| PATCH | `/api/v1/categories/{id}` | ADMIN | 改分类名/前缀/钉住状态 | `ReqCategoryPatch{ name?, codePrefix?, pinned? }` | `ViewCategory` | **改前缀不影响已有器材编号**（主稿 3.5：编号是建档时物化到 `equipment.code` 的，不是运行时拼接） |
| POST | `/api/v1/categories/{id}/disable` | ADMIN | **显式覆盖：禁用** | — | `ViewCategory` | **只用于"分类下明明有器材、但不想让新器材再归到它"**这种少见情况。日常"没人用了"由主稿 3.7 的派生规则自动处理，不需要这个端点 |
| POST | `/api/v1/categories/{id}/restore` | ADMIN | 解除禁用 | — | `ViewCategory` | — |

> **没有 `DELETE /api/v1/categories/{id}`，也没有「重置 `next_seq`」。** 两条都是有意的缺席：分类用派生可用性 + 显式禁用代替删除（主稿 3.7）；重置序号会让已发出的编号被复用，而主稿 3.5 第 1 条明确说那是它**要避免**的行为。

#### 1.3.11 活动流、总览、设置

| 方法 | 路径 | 权限档 | 用途 | 响应 | 分页 | 备注 |
|---|---|---|---|---|---|---|
| GET | `/api/v1/activity` | ADMIN | **一个接口，四个界面**（主稿 10.5） | `{items:[ViewActivityItem], nextCursor}` | `(created_at,id)` 降序，`limit` 默认 20 | 参数 `scope`（`equipment`\|`workspace`\|`project`\|`actor`）、`id`（`scope != workspace` 时必填）、`includeUpdates`、`cursor`、`limit`。`scope=workspace` **默认不含 `UPDATE`**。**申请人一律 403** |
| GET | `/api/v1/overview` | ADMIN | **总览页聚合数据** | `ViewOverview` | — | 字段候选见 §2.3.6；**最终字段是待定项 D-05** |
| GET | `/api/v1/settings` | REQUESTER | 读工作区设置 | `ViewSettings` | — | **含两个布尔，供申请人提前禁用提交按钮**（否则申请人只能靠一次失败的提交学到）。见 §2.3.7 |
| PATCH | `/api/v1/settings` | OWNER | 改两个开关 | `ViewSettings` | — | **只有 OWNER**（主稿 C5 的「主理人的两个开关」）。**布尔不用三态**——`Option<bool>` 已足够表达「真/假/不变」，布尔没有「清空」这个状态 |

---

## 2. 校验规则

### 2.1 原则：五层管线

一次写入要过五道，**每一道的职责不同，不要互相替代**：

```
① 归一化    客户端输入 → 领域类型（去除、trim、GBK 解码、金额转整数分）
② garde     单字段的机械规则（长度、格式、范围）        → AppError::Validation → 400
③ domain    跨字段与业务规则（纯函数，无 I/O）          → DomainError → 400/409/422
④ 预检查     需要查库的规则（如档期冲突）               → DomainError → 409（给人话）
⑤ 约束      数据库的 CHECK/UNIQUE/EXCLUDE/外键          → translate_constraint → 同一个出口
```

**第 ⑤ 层是"正确性"，第 ④ 层是"人话"。** 这个分工是主稿 8.1 的核心，也是它原先漏掉 `23514`/`23503` 的地方。

### 2.2 字段级规则（摘要）

完整表在实现时逐字段展开；**这里只列会影响设计的判断**：

| 字段族 | 规则 | 依据 |
|---|---|---|
| 器材名称 | 必填，**1–80 字** | 手机列表一行 12–14 字，80 字足以覆盖"品牌+型号+后缀" |
| 内部备注 / 项目备注 | 可空，**≤ 2000 字** | 日志单行上限 16 KB，2000 汉字约 6 KB，留出余量；且备注不进日志（日志篇 R1.7） |
| 客户 / 部门 | 可空，**≤ 60 字** | 公司名长度 |
| 项目名称 | 必填，**≤ 80 字** | 同器材名称 |
| 使用人 | 必填，**≤ 40 字** | 人名 |
| 故障说明 | 必填，**≤ 1000 字** | 现象描述，比备注短 |
| 分类名 | 必填，**≤ 20 字** | 下拉里显示 |
| 分类前缀 | 必填，**`^[A-Z0-9]{1,5}$`** | 与 DDL 的 CHECK 一致 |
| 器材编号（手填） | 可空，**≤ 40 字**，不做格式校验 | 主稿 3.5 明确 `CAM-001-备用机` 合法 |
| 用户名 | 必填，**`^[a-zA-Z0-9_]{3,32}$`** | 登录用，不含中文 |
| 显示名 | 必填，**≤ 40 字** | 允许中文 |
| 密码 | 必填，**≥ 10 字** | 无其它复杂度要求——长度比组成更重要 |
| 金额（导入解析） | 去掉货币符号与千分位，**四舍五入到分，不截断** | 运行时篇 R5.2 |
| 存放位置 | 可空，**≤ 60 字** | — |
| 序列号 / 品牌 / 型号 | 可空，**≤ 80 字** | — |

**"永不接受客户端输入"的字段**（即使客户端传了也要忽略）：`id`、`workspace_id`、`created_at`、`updated_at`、`archived_at`、`status`（状态只能由流转端点改）、`next_seq`、`code`（建档时若提供则接受，之后改编号只能走专门的 PATCH，见 D-11）。

### 2.3 跨字段与业务规则

#### 2.3.1 日期区间（三条，与 DDL 的 CHECK 一一对应）

| 规则 | DDL 约束 | 应用层文案 |
|---|---|---|
| `request.use_end >= use_start` | `request_use_end_check` | 预计归还不能早于使用开始 |
| `checkout.expected_date IS NULL OR >= checkout_date` | `checkout_expected_date_check` | 预计归还不能早于出库日期 |
| `project.end_date IS NULL OR start_date IS NULL OR end_date >= start_date` | `project_date_range_check`（**本稿新增**） | 结束日期不能早于开始日期 |

#### 2.3.2 批量操作

| 规则 | 文案 |
|---|---|
| 至少 1 件 | 请至少选择一件器材 |
| 最多 100 件 | 一次最多处理 100 件 |
| 不得重复 | 选择里有重复的器材 |
| 全部违规一次报出，不做部分成功 | 以下器材不能归档（出库中）：CAM-003、CAM-011 |

#### 2.3.3 申请单

| 规则 | 来源 |
|---|---|
| 至少一件器材 | **新增**。主稿没有这条约束，而状态推导（§2.3.4）缺它会有歧义 |
| 不得重复同一件器材 | `request_item_uniq` |
| 提交时必须 `requester_can_request = true` | 主稿 3.3 |
| 只能撤回自己的、且未交付的 | 见 §2.3.4 |
| `use_end` 不得早于今天 | 见 D-01（待定） |

#### 2.3.4 申请状态是**推导**出来的，不直接写

主稿 3.1 的映射表说「批准时取消勾选某件（= 拒绝）→ 不 INSERT」。**这个模型有一个必须补的判别位**：

- 若"提交审批"把未勾选项写成 `approved = false`，那么**"全部拒绝"与"还没审"在数据行上无法区分**（都是 false）。
- 若只写被勾选的件、其余留 `NULL`，那么**"全部拒绝"根本落不到 `REJECTED`**（没有任何行是 false）。

**结论：`reviewed_at` 是判别位。** 规则：

```
status = PENDING     当 reviewed_at IS NULL
status = REJECTED    当 reviewed_at IS NOT NULL 且没有任何 approved = true
status = APPROVED    当 reviewed_at IS NOT NULL 且全部 approved = true
status = PARTIAL     当 reviewed_at IS NOT NULL 且部分 approved = true
status = DELIVERED   当有 checkout.source_request_id 指向它
status = RETURNED    当所有 request_item.returned_at IS NOT NULL
status = WITHDRAWN   由撤回端点显式置位
```

**`approve` 与 `reject` 是同一写路径的两个入口**：前者带上逐件的 `approved`，后者等价于全部 `approved = false`，但**必须走独立端点**，因为只有独立端点才能明确写出「这是一次整体拒绝」而不是「审批时一个都没勾」。两者都在同一个事务里写 `reviewed_at` + `reviewed_by` + `status`（推导得出）。

**撤回规则**：

| 当前状态 | 能否撤回 | 后果 |
|---|---|---|
| `PENDING` | ✅ | 不碰 `occupancy`（还没有） |
| `APPROVED` / `PARTIAL` | ✅ | **DELETE 对应的 `RESERVATION` 行**，档期释放 |
| `DELIVERED` | ❌ | 器材在外面，只能逐件归还 |
| `RETURNED` / `REJECTED` / `WITHDRAWN` | ❌ | 终局 |

> **⚠️ 撤回与交付都要锁 `request` 行**（`SELECT … FOR UPDATE`），否则「撤回」与「交付」并发时会一边 DELETE 占用一边 UPDATE 占用，得到一张 `source_checkout_id` 指向已撤回申请的出库单。

#### 2.3.5 项目状态流转

主稿没有给项目的流转表（器材有五态表，项目没有）。**补上**：

| 当前 | 可以变成 | 额外条件 |
|---|---|---|
| `ACTIVE` | `COMPLETED`、`ARCHIVED` | **该项目没有未归还的出库** —— 主稿 5.6「有器材没归还的项目不能标记完成或归档，按钮禁用并提示」 |
| `COMPLETED` | `ACTIVE`、`ARCHIVED` | 同上 |
| `ARCHIVED` | `ACTIVE` | — |

#### 2.3.6 总览页字段（候选，待定 D-05）

```
{ totalCount, inStockCount, checkedOutCount, maintenanceCount,
  statusCounts: { IN_STOCK, CHECKED_OUT, MAINTENANCE, DISABLED, SOLD },
  openCheckoutCount, pendingRequestCount,
  overdueCount, overdueItems: [ … ] }
```

"总览页四张卡片：器材总数、在库可用、拍摄中、维修中"来自交接稿第 6 节；`overdueItems` 来自主稿 11 节与运行时篇 R1.8。

#### 2.3.7 两个开关的边界（**准入开关，不回溯**）

**这是本稿最重要的一条业务规则裁定：**

> 两个开关是**准入开关**——只阻止**开始新动作**，**绝不回溯修改任何已存在的 `request` 状态或 `occupancy` 行**。

| 场景 | 行为 |
|---|---|
| `can_browse = false` | 器材列表/详情/分类对 `REQUESTER` 返 **403 `BROWSING_DISABLED`**（不是空列表、不是 404——404 会让申请人能用「这个 id 存不存在」当探测手段） |
| `can_request = false` | `POST /requests` 返 **422 `REQUESTING_DISABLED`**（"暂未开放申请权限"） |
| 关掉开关时，已有的 `APPROVED` 申请 | **档期保留**。释放不可逆（别人能立刻领走，开关再打开也回不来） |
| 关掉开关时，已有的 `PENDING` 申请 | **保留在队列里**，管理员仍能审 |
| `GET /requests/mine` | **不受任何开关影响，始终 200** |
| `POST /requests/{id}/withdraw` | **不受任何开关影响**——否则关闭浏览会让档期释放路径断掉 |
| 两个开关谁能改 | **只有 OWNER**（`PATCH /settings`） |
| `can_request = true` 且 `can_browse = false` | **保存时拦住**（422）：申请是针对具体器材的，不允许浏览就无法提交。见 D-04 |

**为什么"关掉就全部作废"是错的**（提案论证，采纳）：主稿 3.1 那张「业务事件 → occupancy 操作」映射表是**封闭的**（只有交付/归还/撤回三个事件写入或删除占用），开关不在其中。而且如果开关去释放档期，会**打穿每日占用对账**——对账规则是「每条 `RESERVATION` 必须有对应的 `approved = true` 的 `request_item`」，释放后这条规则天天误报 ERROR，**对账从 bug 探针退化成噪音**。

**没有通知渠道时这个开关的实际效果**：申请人只会在下次打开时才知道。**真正的信息缺口不是申请人不知道，而是主理人可能忘了队列里还有单** —— 所以管理员界面应显示「申请已关闭，队列中还有 N 条待审」。

---

## 3. 连接池与超时

### 3.1 应用池（app 进程）

| 参数 | 取值 | 依据 | 设错时的表现 |
|---|---|---|---|
| `max_connections` | **10** | 见 §3.3 推算 | 太少：正常请求排队；太多：吃掉 Postgres 全局连接预算 |
| `min_connections` | **2** | 避免冷启动时每次都要建连 | 0：第一个请求慢；太高：空转占槽 |
| `acquire_timeout` | **5s** | 十人规模下池几乎不会饱和，5s 是"真的有问题"的分界线 | 太短：偶发误报 503；太长：请求挂住占着连接 |
| `idle_timeout` | **600s** | 与 `max_lifetime` 配合做连接回收 | 太短：频繁重建连接 |
| `max_lifetime` | **1800s** | 让连接定期重建，绕过中间设备的空闲超时 | 无限：中间设备断连后应用不知道 |

### 3.2 服务端设置：设在**角色**上，不设在连接上

```sql
ALTER ROLE lensmax_app SET statement_timeout = '15s';
ALTER ROLE lensmax_app SET lock_timeout = '5s';
ALTER ROLE lensmax_app SET idle_in_transaction_session_timeout = '2s';
```

| 参数 | 取值 | 守的是什么 |
|---|---|---|
| `statement_timeout` | **15s** | 一条查询不许跑超过 15 秒。十人规模下任何正常的列表/详情查询都在毫秒级 |
| `lock_timeout` | **5s** | 等锁超过 5 秒就放弃——**这防的是一次锁等待拖死一个连接** |
| `idle_in_transaction_session_timeout` | **2s** | **见 §3.4，这是三个里最重要的** |

**为什么设在角色上而不是 SQLx 的 `after_connect`**：

> **`psql` 手工连接是这套系统里最可能的锁事故源。** 某个终端 `BEGIN; SELECT … FOR UPDATE;` 之后去开会，CAM-003 从此改不动，**而应用日志里一个字都没有**（它根本不在应用里）。连接级设置对 `psql` 完全无效；角色级设置覆盖所有以该角色连接的客户端。

**同时有一个必踩的坑**：迁移**不能用应用角色跑**。`ALTER TABLE` 需要 `ACCESS EXCLUSIVE` 锁，而 `lock_timeout = 5s` 会让它在锁等待中失败在迁移中途。**必须另建一个不设 `lock_timeout` 的迁移角色**。

**应用侧 `after_connect` 不 SET、只断言**：连上后执行 `SELECT 1 WHERE current_setting('statement_timeout') = '15s'`，失败就断开。这把"角色设置漏了/被 RESET 了"变成**第一个连接就失败**，而不是某天某条查询永远不超时。

### 3.3 池大小推算（可复核）

用「到达率 × 持有时间」反推需要多少连接：

```
连接被持有的时间 = 数据库时间（不是请求总时长）
最重的路径：照片上传
  收流 / SHA-256 / 解码 / 缩图 / 落盘  ← 全部在事务外（主稿 6.3）
  只有最后写 photo 行 + pending_upload 行占连接 → 20–50 ms

十人规模的峰值请求率 ≈ 0.17 req/s
要打满 10 个连接需要 ≈ 200 req/s
差距 ≈ 1200 倍
```

**但结论不是"池大小无所谓"。** 10 的真实作用是：**容纳「少数几条卡住 + 其余正常服务」**。按 §3.2 的 `statement_timeout = 15s`，最坏情况是 10 个连接全被慢查询占满 15 秒——**正好等于运行时篇 R1.10 的 drain 窗口**，所以这两种超时的量级是有意对齐的。

**上限依据（写进主稿第 12.1 节）**：Postgres 默认 `max_connections = 100`，预算分配为**所有进程的池之和 ≤ 60，留 5 个给运维工具（`psql`、备份）**。当前 app 10 + jobs 3 = 13，余量充足。

### 3.4 `idle_in_transaction_session_timeout = 2s` 为什么特别重要

主稿已经有一条规则：**事务里绝不做网络或文件 I/O**（照片处理、通知都在事务外）。那条规则意味着**任何合法事务都不会在两条 SQL 之间空闲 2 秒**。所以 2 秒这个值不会误杀正常操作，而它能拦住三类事故：

| 事故 | 应用侧能发现吗 |
|---|---|
| 应用里某个 handler 忘了提交/回滚，事务挂着 | ✅ 有探针（见下） |
| **`psql` 手工开了事务去开会** | ❌ **完全在应用之外，应用日志里零痕迹** |
| 应用代码在事务里做了慢 I/O | ✅ 有探针 |

**第二类是它唯一不可替代的地方。** 配套要一个应用侧 **500ms 的 `tx.idle` WARN 探针**：数据库的 2 秒是兜底，应用的 500ms 是探针——**两个都要**，因为兜底发生时连接已经被杀，你只看到"连接断了"。

它和 `statement_timeout` 的**计时区间不相交**（一个算语句执行时间，一个算两条语句之间的空闲），所以不会误杀 `jobs` 的长查询。

### 3.5 jobs 池

| 参数 | 取值 | 说明 |
|---|---|---|
| `max_connections` | **3** | 它跑的是长查询（逾期扫描、占用对账） |
| `min_connections` | **0** | 它是 cron 触发的短命进程，不常驻 |
| `acquire_timeout` | **30s** | **故意比 app 的 5s 长** |
| `idle_timeout` | **30s** | — |

**`acquire_timeout` 为什么故意更长**：jobs 不对外服务。等一会儿的代价是"晚点跑完"，失败的代价是**等 24 小时**（运行时篇 R1.6）。

**不给 jobs 放宽 `statement_timeout`**：那会掩盖缺索引的问题，而且它占全局连接预算。正确做法是**分批**。

**两条必须写进实现的纪律**：

- **孤儿清理"先取集合、归还连接、再走文件系统"。** 否则几十秒的文件扫描会被 `idle_in_transaction_session_timeout` 杀掉，而失败表象是"数据库连接莫名其妙断了"。
- **占用对账不能用 Rust 在事务里慢慢比对**——同理。用一条 SQL 找出偏差，再离开事务去处理。

### 3.6 `readyz` 必须用自己的 1s `acquire_timeout`

否则池饱和时每次探活要等 5 秒，**容器被判定为不健康并重启，情况变得更糟**（运行时篇 R1.9 已为同类陷阱论证过一次）。

### 3.7 错误映射

| 触发 | SQLSTATE / 错误 | 映射 | 码 |
|---|---|---|---|
| `statement_timeout` / `lock_timeout` | `57014` | `TemporaryUnavailable` | **503** + `Retry-After: 1` |
| `idle_in_transaction_session_timeout` | `25P03` | `TemporaryUnavailable` | **503** |
| `acquire_timeout` 到期 | `sqlx::Error::PoolTimedOut` | `TemporaryUnavailable` | **503**（不是 500） |

**全都不是 500**：这三种都是"稍后重试可能成功"，不是"出错了"。前端据此显示"系统繁忙，请稍后重试"。

### 3.8 测试环境

`sqlx::test` 每个测试一个隔离库，**会打满连接**，而且**失败现象误导**——抛的是 `53300 too many clients`，看起来像"建库失败/迁移没跑"，完全不像连接池问题。

| 项 | 取值 |
|---|---|
| 测试实例 `max_connections` | **200** |
| 每个测试的池 | **2** |
| 测试池 `acquire_timeout` | **10s** |
| 那三条角色级超时 | **测试库不设** |

**测试库不设超时的理由**：运行时篇 R3.2 #1 的并发互斥测试**故意制造锁竞争**，设了 `lock_timeout` 会变成随机失败。"超时生效"这件事改用 `SET LOCAL` 在单个测试里验证翻译逻辑。

---

## 4. 重试策略

### 4.1 一句话原则

> **只重试"整笔事务确实没生效"的错误。**

### 4.2 可重试性分类（封闭枚举 + fail-closed）

| SQLSTATE / 错误 | 可重试 | 理由 |
|---|---|---|
| `40P01` 死锁 | ✅ | Postgres 已回滚整个事务，重放安全。**锁冲突是瞬时的，重试大概率立刻成功** |
| `40001` 序列化失败 | ✅ | 同上（虽然本设计用 `READ COMMITTED`，实际不会出现） |
| `08000` / `08006` 连接异常 | ✅ **仅在取连接或事务体阶段** | 见 §4.3 |
| `57P01` 管理员关闭连接 | ✅ 同上 | 同上 |
| **`23P01` 排他冲突** | ❌ **绝不** | 重试必然再失败，而且**把 409 变成 503**——前端据此把"换日期或换器材"变成"稍后重试"，**用户的下一步动作被改错了** |
| **`23505` 唯一冲突** | ❌ **绝不** | 同上 |
| `23514` CHECK 违例 | ❌ | 用户输入错误，重试无意义 |
| `23503` 外键违例 | ❌ | 同上 |
| `57014` 超时 | ❌ | 重试会在同一个预算里再烧一次 |
| `25P03` 事务空闲被杀 | ❌ | 说明代码有 bug，调重试次数是掩盖它 |
| `53300` 连接槽满 | ❌ | 池预算算错或迁移占位，要修配置不是重试 |
| `PoolTimedOut` | ❌ | 见上 |

**fail-closed 是硬要求**：`retry_class()` 返回**封闭 enum**，`_ => Fatal`。**绝不允许出现 `_ => Retry`** —— 那会让未来新增的某个 SQLSTATE 自动变成可重试，而"把 409 变成 503"正是这样发生的。

### 4.3 `commit()` 失败**不重试**

**这是最容易做错的一条。**

提交阶段"响应丢包"和"连接断了"**在客户端是同一种错误**。重放会写出第二份数据——重复出库、`next_seq` 跳号。**宁可给用户一个 500，也不能制造第二份数据。**

| 阶段 | 可重试吗 |
|---|---|
| 取连接（`pool.acquire()`） | ✅ 一个字节业务 SQL 都没发出 |
| 事务体（业务 SQL 执行中） | ✅ 事务已回滚 |
| **`COMMIT`** | ❌ **绝不** |

### 4.4 重试包装器

重试必须**重跑整个事务**，不能只重跑一条语句。因为 `Transaction` 不能被复用，重试要重新 `begin()`：

```rust
/// handler 层唯一的重试入口。服务层收到的是 &mut PgConnection，
/// 它看不到这个包装器，也无法自己决定重试（实现篇第 3 节的签名约定）。
pub async fn with_retry<T, F, Fut>(pool: &PgPool, f: F) -> Result<T, AppError>
where
    F: for<'c> FnMut(&'c mut PgConnection) -> Fut,
    Fut: Future<Output = Result<T, AppError>>,
{
    const MAX_ATTEMPTS: u32 = 3;
    for attempt in 1..=MAX_ATTEMPTS {
        let mut tx = pool.begin().await?;            // ← 每次尝试重新 begin
        match f(&mut tx).await {
            Ok(v) => { tx.commit().await?; return Ok(v); }   // ← commit 错误直接冒泡，不重试
            Err(e) if attempt < MAX_ATTEMPTS && retry_class(&e).is_retryable() => {
                drop(tx);                             // 回滚
                log::biz_retry(attempt, MAX_ATTEMPTS, &e);
                sleep(backoff(attempt)).await;
            }
            Err(e) => return Err(e),
        }
    }
    unreachable!()
}
```

**注意签名里的 `for<'c>`（HRTB）**：闭包必须对**任意**连接生命周期有效。这是本设计里唯一一处 Rust 生命周期写法会咬人的地方，**待验证**（见 §6）。

### 4.5 退避参数

| 项 | 取值 |
|---|---|
| 最大尝试次数 | **3**（不是"无限重试"） |
| 基数 | **50ms** |
| 公式 | `min(50ms × 2^(n-1), 400ms)` |
| 抖动 | **full jitter**（在 `[0, 计算值]` 里取随机） |

**抖动不是优化项，是必需品**：`lock_timeout` 会让所有失败者**同时**在 5 秒后失败、**同时**重试，于是重新制造一次争抢。没有抖动的话重试本身就是一次同步风暴。

### 4.6 耗尽后的行为

| 项 | 值 |
|---|---|
| 状态码 | **503** |
| `DomainError` | `TemporaryUnavailable` |
| 文案 | "系统繁忙，请稍后重试" |
| 响应头 | `Retry-After: 1` |

主稿 7.3 的状态码矩阵最后一行原文是 `限流 / 事务重试耗尽 → 429 / 503`，**已拆开**：429 保持"限流"原义（本设计目前没有限流），重试耗尽独立为 503。

### 4.7 幂等性：重试**不产生**重复行

| 操作 | 重试安全吗 | 说明 |
|---|---|---|
| 出库 / 归还 / 审批 / 交付 | ✅ | 事务已回滚，重跑等价于第一次 |
| **建档（编号留空）** | ✅ **服务端重试安全** | 编号分配在同一个事务里，回滚会连 `next_seq` 一起回滚 |
| **建档（客户端超时重发）** | ❌ **这是重复数据的来源** | 与服务端重试无关——所以建档也需要 `Idempotency-Key`（§1.2） |

**分清楚两种"重试"**：本节的 `with_retry` 是"上一笔已回滚后才重跑"，**不产生重复行**；重复行只来自**客户端重发**，那是运行时篇 R4.4 的 `Idempotency-Key` 负责的。

### 4.8 日志

**新增一个 `kind = biz.retry`（WARN）**，字段 `data = { attempt, max_attempts, error_class, error_kind, elapsed_ms }`。

**不记 `Display`**（它含 SQL 片段）。与日志篇 8.4 一致：**测试用可注入计数器断言重试次数，不断言日志行数。**

> **⚠️ 需要改一处既有设计**：实现篇 3.2 第 7 步把业务日志写在**提交前**。3 次尝试会产出 3 条 `biz.checkout`，其中两条对应已回滚的事务。**业务日志必须从"提交前"挪到"提交后"**（在 `tx.commit()` 成功之后记）。

---

## 5. 写入吞吐与事务长度上限

§3.2 定了 `statement_timeout = 15s` 和 `lock_timeout = 5s`，§4 定了重试策略。**但没人算过最重的那几条写路径在不在预算内。** 这一节补上——因为"超时"不是一个配置项，它是一个**预算**，而预算是要分配的。

### 5.1 逐条写路径的预算核算

| 路径 | 事务里做什么 | 估算 | 结论 |
|---|---|---|---|
| 建档（单件） | 锁 `category` 行 + 取号 + 1 条 INSERT + 1 条 history | < 10 ms | 充裕 |
| 出库（批量 ≤ 100 件） | 锁行 + 状态机 + 100 条 occupancy + 1 条 checkout + 100 条 item + 100 条 history | ~50–150 ms | 充裕 |
| 归还（单件） | 锁行 + 更新 item + DELETE occupancy + 改状态 + history | < 20 ms | 充裕 |
| 审批（≤ 100 件） | 锁 request 行 + 写 `reviewed_at` + N 条 occupancy + 状态推导 + history | ~50 ms | 充裕 |
| 照片上传 | **收流/哈希/解码/缩图全在事务外**（主稿 6.3）；事务内只有 2 条 INSERT | **< 50 ms** | 见 5.3 |
| **CSV 导入 1000 件** | 一次性预取 + **1 次** `next_seq` 分配 + 分批 INSERT + 1000 行历史 | 见 5.2 | **唯一风险点** |
| 孤儿清理（job） | **必须先取集合、归还连接、再走文件系统** | 见 5.4 | **最容易写错** |
| 逾期扫描（job） | **一条 SQL，不用 Rust 循环**；通知在事务外 | 见 5.5 | 见下 |

**结论：只有三条需要专门处理，其余都远在预算内。**

### 5.2 CSV 导入 1000 件：三条必须做的优化

按直觉写（每行一次 INSERT、每行查一次分类、每行分配一次编号），1000 行会变成**几千次往返**——那才是真正会撞 15 秒的地方。三条优化后它变成一个普通事务：

| # | 优化 | 不做会怎样 |
|---|---|---|
| 1 | **一次性预取**：开始时把该工作区的分类名→id、已有编号全部读进内存 | 1000 次分类查询 + 1000 次编号查重 |
| 2 | **`next_seq` 只 UPDATE 一次**，按本批需要量整体前进（`SET next_seq = next_seq + $n RETURNING next_seq - $n`），再在内存里连续分配 | 1000 次行锁更新，**全部串行在同一行上** |
| 3 | **分批 INSERT**（每批 ~250 行一条语句） | 1000 次往返 |

**第 2 条最关键也最容易漏。** `category.next_seq` 是一行，1000 次 `UPDATE` 全部串行在同一行上——**这是本系统里唯一的真实写热点。**

**多分类注意**：一批导入涉及多个分类时，每个分类各跑一次第 2 条，且**按 `category.id` 升序**处理，避免与其它批量操作交叉等待。

**这也是 CSV 导入需要 `Idempotency-Key` 的第三个理由**（§1.2）：它可能跑几百毫秒到一秒，是最可能被客户端超时重发的操作。而重发要么撞编号唯一约束（有编号的行），要么**静默产生 1000 条重复数据**（编号留空的行）。

### 5.3 照片上传：事务里绝不能有文件 I/O

主稿 6.3 已经这么设计了（解码/缩图在 `spawn_blocking` 内）。这里强调**为什么它是硬规则**：

> 一张 1600 px 的 JPEG 解码 + 缩图约 **100–300 ms**。放在事务里看着还能接受，但它是**在持着连接**的情况下。而 §3.4 的 `idle_in_transaction_session_timeout = 2s` 会把"在事务里等 I/O"杀掉，失败表象是"数据库连接莫名其妙断了"。

**规则：事务里只有 SQL。** 收流、哈希、解码、缩图、落盘全部在事务之前完成，事务只写两行（`photo` + `pending_upload`）。

### 5.4 孤儿清理：先取集合、归还连接、再走文件系统

**最容易写错的一条**，因为直觉写法是"在事务里遍历文件"：

```rust
// ❌ 会撞 idle_in_transaction_session_timeout
let mut tx = pool.begin().await?;
let referenced = db::photo::all_hashes(&mut tx).await?;   // 取集合
for entry in fs::read_dir(photos_dir)? {                   // ← 几十秒的文件扫描
    if !referenced.contains(&hash) { fs::remove_file(...)?; }
}
tx.commit().await?;
```

**正确形状**：

```rust
// ✅ 取集合 → 归还连接 → 再做文件操作 → 需要时开新的短事务
let referenced: HashSet<String> = {
    let mut conn = pool.acquire().await?;
    db::photo::all_hashes(&mut conn).await?
};                                                    // 连接已归还
let on_disk = scan_photos_dir(&photos_dir)?;           // 无连接在手
for h in on_disk.difference(&referenced) {
    fs::remove_file(...)?;                             // 无连接在手
}
// 要记录删除数量时，此时再开一个短事务
```

**为什么这条必须写进文档而不是靠 review**：它的失败表象是"数据库连接莫名其妙断了"（`25P03`），**完全不像"文件扫描太慢"**。而 §3.7 把这个错误映射成 503——用户看到"系统繁忙"，真正原因是一个后台任务持着连接扫文件。

**同一形状适用于占用对账**：用一条 SQL 找出偏差，**再离开事务**去处理。

### 5.5 逾期扫描：一条 SQL，通知在事务外

```sql
SELECT e.id, e.code, c.id AS checkout_id, c.expected_date, c.user_name
  FROM equipment e
  JOIN checkout_item ci ON ci.equipment_id = e.id AND ci.returned_at IS NULL
  JOIN checkout      c  ON c.id = ci.checkout_id
 WHERE e.workspace_id = $1
   AND e.status = 'CHECKED_OUT'
   AND c.expected_date < $2;      -- $2 = business_today()，见运行时篇 R2.3
```

**"今天"必须取业务时区**（运行时篇 R2.3）——用 UTC 日期会让东八区每天 08:00 前的判断少算一天。

扫描本身远在预算内（几千行，毫秒级）。**风险在"发现超期之后做什么"**：

> ⚠️ **通知绝对不能放在事务里。** 一次到聊天工具的 webhook 请求可能耗时几秒；20 件超期、每件一条就是几十秒——**连接会被超时杀掉，而且是在持有锁的情况下**。

**正确顺序**：

```
1. 开事务：只做数据库的事（查询、写审计、标记"已提醒"）
2. 提交
3. 提交之后，才发通知
4. 通知失败只记 WARN，不回滚任何东西
```

**第 4 条是有意的**：通知失败不该让"扫描已完成"这个事实消失。超期是既成事实，通知只是它的副作用。这与主稿 8.1"日志不是事实来源"是同一类判断。

### 5.6 一条可以推广的规则

> **事务里只有 SQL。网络、文件系统、进程间通信——全部在事务之外，且通常在提交之后。**

它同时解决三件事：`idle_in_transaction_session_timeout` 不会误杀、锁持有时间最短、以及**重试是安全的**（§4.3 说提交阶段失败不重试——而事务里只有 SQL 时，提交阶段极短，那个窗口也就极小）。

**推论**：这条规则让 §3.4 那个 `tx.idle` 的 500ms 探针**几乎不会触发**。它一旦触发，就说明有人违反了这条规则——**探针的告警就是在抓这个**。

---

## 6. 本稿对既有四份文档的修订

合并过程中发现并修正的问题，按"是否已改"分类：

### 6.1 已直接修正（主稿）

| # | 问题 | 修正 |
|---|---|---|
| 1 | `DomainError` 枚举**缺 `Conflict`**，而第 8 节的翻译代码已在用它 | 补上，并补 `BrowsingDisabled`/`TemporaryUnavailable`/`DuplicateCodeUnknown`/`DuplicateUserName`/`InvalidField`/`OpenMaintenanceExists`/`OwnerNotDisableable`/`ReferenceInUse` |
| 2 | SQLSTATE 翻译**只处理 `23P01` 与 `23505`** → 五条用户可触发的 CHECK 返 500 | 补 `23514`（→ **400**）与 `23503`，改为按约束名 discriminate（主稿 8.1） |
| 3 | `DuplicateCode { code }` 在翻译点拿不到编号 → 用户看到半句话"编号已存在：" | 拆出 `DuplicateCodeUnknown`（文案完整），由服务层补查后升级 |
| 4 | `equipment_history` 两个索引**都不含 `id`**，而游标键是 `(created_at, id)` | 索引加 `id` |
| 5 | `request_item.request_id` 与 `checkout_item.checkout_id` 是 `ON DELETE CASCADE` | 改成 `RESTRICT`（级联删会静默抹掉审批/出库事实，与 10.1 冲突） |
| 6 | 其余 27 处 `REFERENCES` 未写 `ON DELETE` | 全部显式 `RESTRICT`（可读性：写了的和没写的在 review 时无法区分） |
| 7 | `project` 是唯一缺日期区间 CHECK 的表 | 补 `project_date_range_check` |
| 8 | `equipment.category_id` 是单列外键 → **可跨工作区引用分类** | 改复合外键 + `category(workspace_id, id)` 唯一键（主稿 3.6） |
| 9 | 分类没有"不再使用"的表达 | **可用性改为派生**：新增 `category.pinned`，`disabled_at` 降级为显式覆盖（主稿 3.7） |
| 10 | `workspace_setting` 只有 `updated_at`，**没有 `updated_by`** | 补 `updated_by`（已确认的决策） |
| 11 | 第 8 节要求显式命名约束，同节代码却用默认名 `equipment_workspace_id_code_key` | 统一为显式命名 |

### 6.2 需要改但涉及既有决定（列在这里供审）

| 位置 | 要改什么 | 来源 |
|---|---|---|
| 实现篇 3.2 第 7 步 | 业务日志从**提交前**挪到**提交后**（否则 3 次重试产出 3 条已回滚的日志） | §4.8 |
| 实现篇 5 | `Validation(#[from] garde::Report)` 配 `#[error(transparent)]` 会把 **garde 的英文 `Display`** 直接给用户；且主稿 7.3 错误信封的 `field` **没有任何生产者** | 见 §6.3 |
| 主稿 5.3 与第 8 节 | 仍在使用「批量**删除**」措辞，而 D6 已宣布不做软删除；且 `BatchRejected` 文案只给了"出库中"一个理由，实际归档的拒绝理由至少三种 | §1.3.4 |
| 主稿 13.2 / 13.9 | 导出范围、CSV 去重策略仍是阻塞项 | §2.4 |
| 主稿 7.3 | `限流 / 事务重试耗尽 → 429 / 503` 一行拆成三行 | §4.6 |
| 主稿第 9 节 | 需补 §2.3.7 的两个开关边界（准入开关、不回溯、撤回入口独立于开关） | §2.3.7 |

### 6.3 `Validation` 错误体的重构（建议，需要确认）

现状：

```rust
#[error(transparent)] Validation(#[from] garde::Report),
```

两个问题：① `transparent` 把 garde 的**英文** `Display`（`length is not in range`）直接交给用户，与主稿 7.3「message 直接进表单内红条」和「Display 直接是给用户看的中文」冲突；② 主稿 7.3 信封里的 `field`（供表单定位输入框）没有生产者。

建议改成：

```rust
pub struct ValidationError {
    pub field: Option<String>,      // snake_case 路径 → camelCase，如 "purchaseDate"
    pub code: &'static str,         // "TOO_LONG" / "REQUIRED" / "INVALID_FORMAT"
    pub message: String,            // 中文，直接给用户
    pub all: Vec<(String, &'static str)>,   // 其余问题，供一次性列出
}
```

配套一条**元测试**：断言任何校验错误的 `message` 不含 `length`/`required`/`range`/`invalid` 这类英文词。

---

## 7. 待验证（未实测，不许当结论引用）

写本稿时**这台机器上没有 `cargo`、没有 `psql`、没有 Docker**，所以以下均为文档行为 + 推理，**不是实测**：

| # | 待验证 | 失败时的表象 |
|---|---|---|
| 1 | `with_retry` 的 HRTB 闭包写法（`for<'c> FnMut(&'c mut PgConnection)`）能否通过借用检查 | 编译错误指向生命周期，而不是逻辑 |
| 2 | `sqlx::Error` 0.8 的变体名（`PoolTimedOut`/`PoolClosed`） | 编译错误 |
| 3 | `57014` 能否稳定取到 `db.code()` | 取不到会掉进 500，而"查询超时"变成 500 是本稿明确要避免的 |
| 4 | 两种 `57014`（`statement_timeout` vs `lock_timeout`）的**消息文本能否可靠区分** | 不能区分时，`lock_timeout` 的失败只能按保守值处理（不重试）——这正是本稿 §4.2 的取值 |
| 5 | 死锁实际报 `40P01` 还是 `40001` | 分类表写错，重试不生效 |
| 6 | `#[sqlx::test]` 能否指定池大小 | 测试可能打满连接（§3.8） |
| 7 | SQLx 归还脏连接时是否自动回滚 | 事务残留 |
| 8 | 托管 Postgres 是否允许对非超级用户执行 `ALTER ROLE` | 不允许时 §3.2 的角色级方案在托管环境不成立，只能退回连接级 |
| 9 | `garde 0.20` 的 API 面（自定义校验函数签名、`Report::iter()` 的产出类型、规则级 `message` 是否存在） | 需要改用兜底方案（正则类一律 `custom`、文案走映射表 + 元测试） |
| 10 | `NO ACTION` 与 `RESTRICT` 在本系统用法下的事务内语义差别 | 二者对本系统等价，选 `RESTRICT` 是可读性考虑 |
| 11 | 部分唯一索引能否让「归档行让出编号」（D-11 的前提） | D-11 不可行 |

---

## 8. 待定项索引

需要产品/负责人拍板的条目汇总在 [20260930_OPEN_DECISIONS_v1.md](20260930_OPEN_DECISIONS_v1.md)。本稿中引用的编号（`D-01`、`D-02`…）指那份文档。
