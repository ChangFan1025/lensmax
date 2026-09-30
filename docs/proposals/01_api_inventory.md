# LensMax 接口清单（API inventory）

日期：2026-09-30
状态：**提案**。未并入任何现有文档。本文件是新增文件，不改动 `docs/` 下任何既有文档。
依据：[20260930_BACKEND_v1.md](../20260930_BACKEND_v1.md)（下称「主稿」）、[20260930_BACKEND_RUNTIME_v1.md](../20260930_BACKEND_RUNTIME_v1.md)（下称「运行时篇」，节号写作 `R-n`）、[20260930_LOGGING_v1.md](../20260930_LOGGING_v1.md)（下称「日志篇」）、[20260930_RUST_IMPL_v1.md](../20260930_RUST_IMPL_v1.md)（下称「实现篇」）。

---

## 0. 这一节该插进主稿的哪里

**插入位置：主稿第 7 节之后、第 8 节之前，作为新的「7.10 接口清单」。**
第 7 节定的是**所有接口共用的形状**（类型约定 7.2、错误信封 7.3、`actions` 7.4、游标分页 7.5、PATCH 三态 7.6、可见性双结构体 7.7、两条越权路径 7.8、传输层 7.9）。本节是那份约定在**具体端点上的展开**——读法上它必须排在 7.2–7.9 之后，编号上它是第 7 节的最后一个子节。第 8 节（错误处理）与第 9 节（认证授权）反过来要靠本节来落定「哪个端点返哪个码」，所以放在它们之前。

**它替代或汇总的现有零散表述**（这些地方目前各自说了一半，没有一处是完整清单）：

| 现有位置 | 说了什么 | 本节的处理 |
|---|---|---|
| 主稿 6.3 | `POST /api/photos`（**缺 `/v1`**） | 收录为 `POST /api/v1/photos`，见 §5-G1 |
| 主稿 7.5 | `GET /api/equipment?…`（**缺 `/v1`**） | 收录为 `GET /api/v1/equipment`，见 §5-G1 |
| 主稿 7.8 | `GET /api/checkouts/{id}/manifest`（**缺 `/v1`**） | 收录为 `GET /api/v1/checkouts/{id}/manifest`，见 §5-G1 |
| 主稿 10.5 | 活动流四个 `scope`，已带 `/api/v1` | 收录为 `GET /api/v1/activity`，见 §5-G4、§5-I3 |
| 日志篇 4.1 | `PATCH /api/v1/equipment/checkout`（**与出库是 POST 这件事矛盾**，且 `PATCH` 与「出库是状态流转、不是字段编辑」冲突） | 收录为 `POST /api/v1/checkouts`，见 §5-G2 |
| 日志篇 4.3 | `POST /api/v1/photos` | 一致，直接收录 |
| 日志篇 4.4 | `POST /api/v1/equipment/import` | 拆成 `preview` 与 `commit` 两个端点，见 §5-G5 |
| 日志篇 6 节事件清单 | 隐含 `auth.login`/`auth.logout`/`biz.approve`/`biz.deliver`/`biz.archive`/`export` 等操作 | 作为端点齐全性的交叉核对表，见 §5-I1 |
| 实现篇 6.2 | `Actor` / `Admin` / `Owner` 三种 extractor | 落成本节的「权限档」列，见 §0.2 |

**同时它应该被主稿第 14 节（与交接稿的差异汇总）引用一次**，因为本节新增了几条主稿目前没有的约定（幂等的落地方式、错误码的补齐、`equipment_history.action` 缺 `UPDATE`），那些属于「需要同步回交接稿」的类别。

### 0.1 命名风格：选定一种，全部按它写

**选定：资源嵌套 + 显式动作段（nested resource + explicit action segment）。**

形状固定为两种，不允许出现第三种：

```
# ① 读与创建：复数资源集合
GET    /api/v1/equipment                    # 列表
GET    /api/v1/equipment/{id}               # 详情
POST   /api/v1/equipment                    # 建档
PATCH  /api/v1/equipment/{id}               # 改描述性字段（主稿 7.6 三态）
DELETE /api/v1/equipment/{id}               # 归档，不是物理删除（主稿 3.4）

# ② 状态流转与一次性业务动作：POST /资源/{id}/{动作名词}
POST   /api/v1/equipment/{id}/checkout
POST   /api/v1/equipment/{id}/maintenance
POST   /api/v1/requests/{id}/approve
POST   /api/v1/requests/{id}/deliver
```

**为什么不选另一种（动作端点集合，如 `POST /api/v1/checkouts` 表示「出库若干件」）：**

1. **主稿自己已经选了嵌套式。** 主稿 7.8 那条「出库交接清单」写的是 `GET /api/checkouts/{id}/manifest` —— 这是全文档里唯一一处为了安全性（角色守卫收在单个 handler 里）而专门设计的接口，它用的是嵌套式。清单必须与出库资源同源，风格没法另立。
2. **被操作的对象有 ID，而权限检查挂在对象上。** 主稿 7.7 的可见性裁剪是**按器材**做的（`EquipmentView::for_actor`），实现篇 2.1 把它收成唯一分支点。嵌套路径让「动作作用在哪件器材上」写在路径里，handler 签名里天然带得进这个 ID；`POST /api/v1/checkouts` + body 里一坨 `equipmentIds` 会让权限、审计、日志三处都要在 body 解析之后再判断（日志篇 `ids.equipment_id` 是单值字段，批量出库的日志按主稿 §4.1 只记 `checkout_id` 和件数，嵌套式与它一致）。
3. **路径段是编译期常量，这是安全属性。** 主稿 7.5 与实现篇 7.1 都把「拼 SQL 只能用编译期常量、用户输入只走 `.bind()`」列为硬规则。路由风格同理：动作名词是有限封闭集合（本节 §3 是它的完整枚举），写在路径里意味着它是代码里字面量，而放在 body 里就是一个自由字符串，会随前端需求长出 `"checkout2"` 这种值。

**嵌套式的两个代价，以及本节的对待方式：**

| 代价 | 处理 |
|---|---|
| 路径变长（`/equipment/{id}/maintenance`） | 接受。动作名词一律**单段、英文、小写**，且**一个概念只有一个名词**（出库永远是 `checkout`，绝不出现 `lend`/`borrow`/`check-out`）。这与主稿 13.1 的术语立场一致（代码层 `checkout`/`return`，界面层「出库/归还」）。 |
| 同一业务事件出现在多个路径下（出库既可能是「直接出库」也可能是「申请交付」） | 用「哪个资源被创建」来判定归属，不用「前端从哪个页面点的」。见 §5-G3——这是本节发现的最大的风格裂缝。 |

### 0.2 权限档的定义

四档，取自实现篇 6.1–6.2 的三个 extractor（`Actor` / `Admin` / `Owner`），加一个不需要会话的档：

| 档 | 对应的 extractor（实现篇 6.2） | 含义 |
|---|---|---|
| `PUBLIC` | 无 | 不需要会话。仅登录、bootstrap、健康检查、静态照片。 |
| `REQUESTER` | `Actor` | **需要登录**，三种角色都能调。返回体按角色裁剪（主稿 7.7：`EquipmentRequester` 或 `EquipmentAdmin`）。 |
| `ADMIN` | `Admin(pub Actor)` | 需要 `OWNER` 或 `ADMIN`。申请人调用返回 403。 |
| `OWNER` | `Owner(pub Actor)` | 只允许 `OWNER`。 |

**注意 `REQUESTER` 这一档的名字容易误读，这里写清它的语义：**

- §5-I2 指出，主稿 3.3 的 `workspace_setting.requester_can_browse` 默认 `true`，也就是说**申请人默认能看器材库**。所以「器材列表/详情」的路由档位是 `REQUESTER`（需登录、不按角色拒绝），但是否放行以及返回什么，由**运行时的两个开关 + 角色裁剪**决定，不由路由决定。
- **申请人拿不到的东西是四类，不是一类**，必须在实现时逐条守住：
  1. **字段**：`purchasePriceCents`、`purchaseDate`、`serialNumber`、`internalNote`、`createdAt`、`updatedAt` —— 由 `EquipmentRequester` 类型保证（主稿 7.7、实现篇 2.1）。
  2. **整个入口**：`GET /api/v1/activity` 一律 `ADMIN`。主稿 10.4 的原文是「**决定：申请人完全不给历史入口**」，不是「过滤条目」。实现篇 V1 与运行时篇 R3.2 #3 的契约测试要覆盖这一条。
  3. **两个开关**：`requester_can_browse = false` 时列表与详情对 `REQUESTER` 返 403；`requester_can_request = false` 时提交申请返 `REQUESTING_DISABLED`（主稿第 8 节 `DomainError::RequestingDisabled`）。
  4. **清单**：`GET /api/v1/checkouts/{id}/manifest` 是 `ADMIN`，因为清单含序列号而签字人可能是申请人（主稿 7.8）。**这是 7.8 明确禁止「前端拿器材列表 DTO 拼清单」的原因**，因此它不能是 `REQUESTER`。

### 0.3 分页与幂等的统一约定

**分页（主稿 7.5）：** 所有列表端点一律游标分页，**不用 offset**。三件事必须在每个端点上一致：

| 项 | 约定 |
|---|---|
| 请求参数 | `cursor`（不透明字符串，服务端 base64url，客户端不解析）+ `limit`（默认 50，**上限 100**） |
| 响应 | `{ "items": [ … ], "nextCursor": "…" \| null }`。`nextCursor` 为 `null` 表示到底，**不用空 items 表示到底** |
| 游标键 | 写在本节总表的「分页」列。主稿 7.5 允许 `(created_at, id)` 或 `(code)`，本节统一取 **`(created_at, id)` 降序**，理由见下 |

**为什么 `(created_at, id)` 而不是 `(code)`：** 主稿 3.3 已经为它建好了部分索引 `equipment_list_idx ON equipment (workspace_id, created_at DESC, id DESC) WHERE archived_at IS NULL`。按 `code` 排序则没有对应索引（`equipment(workspace_id, code)` 是唯一约束索引，方向是 ASC，且不排除已归档），而 `code` 还有 `CAM-1000` 与 `CAM-999` 的**字符串序与意图不符**问题（主稿 3.5 允许序号超过三位）。

**一个必须在实现时确认的既有故障：** 主稿 3.3 的 `equipment_history` 只有 `(equipment_id, created_at DESC)` 与 `(workspace_id, created_at DESC)` 两个索引，**都不含 `id`**。活动流若按 `(created_at, id)` 做游标，同一毫秒内的多条记录（一次批量出库写 3 条 `CHECKOUT`）会需要按 `id` 打破平局，而索引里没有 `id`；这不会出错，但会退化成 filter。见 §5-I6，**需要主稿补充**（索引加 `id`，或明确接受退化成 filter）。

**幂等（运行时篇 R4.4）：** R4.4 的原文是「只加在三个地方：出库、照片上传、CSV 导入提交」，且后半句说「照片上传本来就已经幂等（内容寻址），所以实际上只有出库和导入两处需要这个头」。本节的立场是**按 R4.4 字面执行，三个端点都收 `Idempotency-Key`**：

- `POST /api/v1/checkouts`
- `POST /api/v1/photos`
- `POST /api/v1/equipment/import/commit`

理由是照片上传的幂等性来自**内容寻址**（主稿 6.2：路径相同就跳过写入），那只保证「不会产生重复的 blob」，**不保证「不会产生重复的 `photo` 行、不会重复记 `upload.ok` 日志」**。带上 key 之后这三处是同一个机制，运维排查时不必记「哪两个接口要带头、哪一个只是碰巧安全」。

**R4.4 没有定义承载幂等键的表，也没有定义冲突时的行为（同 key 不同 body 怎么办）。**这是本节要求主稿补充的第一项，见 §5-I5。本节**不自行设计该表**，只标注三个端点需要它。总表的「幂等」列因此只写 `Idempotency-Key`，不写实现细节。

---

## 1. 总表

**读法：** 「请求体类型」与「响应类型」列给的是**类型的职责**，不是完整字段表。`Req*` / `View*` 是本提案为了指代而起的名字，落到代码里应遵循实现篇 2 节的三套类型（Row / Model / View）。凡是主稿已定义过的结构，本列直接指向主稿的节号，§2 不重复。

**`—` 表示该端点没有请求体**（GET/DELETE）或没有分页/幂等要求。

### 1.1 健康检查与照片静态资源

| 方法 | 路径 | 权限档 | 用途 | 请求体类型 | 响应类型 | 分页 | 幂等 | 备注 |
|---|---|---|---|---|---|---|---|---|
| GET | `/healthz` | PUBLIC | 容器 liveness。**不碰数据库** | — | 纯文本 `ok` | — | — | 运行时篇 R1.9。**不带 `/api/v1`**，它是运维端点不是业务 API |
| GET | `/readyz` | PUBLIC | readiness：数据库可连 + 迁移已到最新 | — | `{status, migrationVersion}`；不就绪时 503 | — | — | 运行时篇 R1.9。**不带 `/api/v1`** |

> **照片的读取不在本节**：主稿 6.6 决定 `GET {PHOTO_BASE_URL}/…`（生产为 `/photos/*`）由 Caddy 静态直发、**不经过 Axum、不做鉴权**，理由是照片是内容寻址且三种角色都可见。所以本节不列「下载照片」端点。**推论**：照片 URL 一旦下发就永久有效（主稿 6.2 的 `immutable`），运行时篇 R2.2 已记下「改 `PHOTO_BASE_URL` 不会让旧 URL 失效」这个后果。**这条推论对下面的 404 处理有影响**：器材详情里 `photo` 字段可能指向一个已 404 的 URL（R7.2 的文件丢失），而接口层对此无能为力——**接口层不许把 `photo` 置空来掩盖它**。

### 1.2 账号：登录、会话、bootstrap

| 方法 | 路径 | 权限档 | 用途 | 请求体类型 | 响应类型 | 分页 | 幂等 | 备注 |
|---|---|---|---|---|---|---|---|---|
| POST | `/api/v1/auth/login` | PUBLIC | 登录，建会话，下发 cookie | `ReqLogin{ username, password }` | `ViewMe` | — | — | 日志篇 `auth.login` / `auth.fail`。**`auth.fail` 的 `data.reason` 不含用户名原文**（日志篇 6 节、8.2 L2） |
| POST | `/api/v1/auth/logout` | REQUESTER | 登出，撤销当前会话 | — | 204 | — | — | 日志篇 `auth.logout`。**幂等性是天然的**（已登出再登出同样 204） |
| GET | `/api/v1/auth/me` | REQUESTER | 当前账号 + 角色 + 工作区（前端决定导航用） | — | `ViewMe` | — | — | 前端启动时第一个调用。**不含权限矩阵**——`actions` 是逐资源的（主稿 D8），不在账号级下发 |
| POST | `/api/v1/bootstrap/owner` | PUBLIC | 建**第一个**主理人。仅在 workspace 无 OWNER 时可用 | `ReqBootstrap{ workspaceName, username, displayName, password }` | `ViewMe` | — | — | 主稿第 9 节。**用过即永久失效**。主稿 9 节与 §5-I4 指出 **workspace 的创建路径没有定义**——本端点顺带创建 workspace 是一个提案，需要主稿确认 |

> **会话 cookie 不是端点。** 主稿第 9 节：`httpOnly; Secure; SameSite=Lax`，每请求查一次 `app_user_session`（实现篇 6.1）。**「续期」也没有端点**——`expires_at` 固定（运行时篇 R2.1 的 `session_ttl_days` 默认 14 天），到期即 401，前端跳登录页。
>
> **缺端点：改密码 / 重置密码。** 运行时篇 R7.4 第 2 条把「重置某人的密码」列为**一定会被做、没有文档一定会做错**的操作，但主稿与运行时篇都没有给它接口。见 §5-I4（需要主稿补充）。

### 1.3 账号管理（邀请 / 成员 / 角色）

| 方法 | 路径 | 权限档 | 用途 | 请求体类型 | 响应类型 | 分页 | 幂等 | 备注 |
|---|---|---|---|---|---|---|---|---|
| GET | `/api/v1/invites` | ADMIN | 邀请列表（有效/已用/已过期） | — | `{items:[ViewInvite]}` | 游标 `(created_at,id)`，`limit ≤ 100` | — | **不返回明文 token**（主稿第 9 节：明文只在生成时返回一次） |
| POST | `/api/v1/invites` | ADMIN | 生成邀请链接 | `ReqInviteCreate{ role: ADMIN\|REQUESTER }` | `ViewInviteWithToken` | — | — | **`token` 只在这个响应里出现一次**。主稿第 9 节：顺手解决「邀请邮件没有邮件服务」——链接手动复制 |
| DELETE | `/api/v1/invites/{id}` | ADMIN | 撤销一个未使用的邀请 | — | 204 | — | — | 撤销 = 置 `consumed_at` 还是新列，主稿 3.3 的 `invite` 表**没有 `revoked_at` 列**。见 §5-I6 |
| POST | `/api/v1/invites/{token}/accept` | PUBLIC | 用邀请码建号 | `ReqAcceptInvite{ username, displayName, password }` | `ViewMe` | — | — | 一次性（`consumed_at`）+ `expires_at`（主稿 3.3）。**token 在路径里**：它是一次性凭据，日志不许记它（运行时篇 R1.7、日志篇 L2） |
| GET | `/api/v1/users` | ADMIN | 成员列表 | — | `{items:[ViewUser]}` | 游标 `(created_at,id)` | — | 含 `disabledAt`，否则界面无法显示「(已停用)」（主稿 10.6） |
| GET | `/api/v1/users/{id}` | ADMIN | 成员详情 | — | `ViewUser` | — | — | 主稿未要求，但 `actor` scope 的活动流（主稿 10.5）需要一个落点显示「这是谁」 |
| PATCH | `/api/v1/users/{id}` | ADMIN | 改 `displayName` | `ReqUserPatch{ displayName?: Option<String> }` | `ViewUser` | — | — | 主稿 3.3 的 `app_user` 只有 `display_name` 是可改的描述性字段 |
| POST | `/api/v1/users/{id}/disable` | ADMIN | 停用账号（**同时删除其全部会话**） | — | `ViewUser` | — | — | 主稿第 9 节：「停用账号时删除其全部会话——这是选有状态会话的唯一理由」。运行时篇 R7.4 第 2 条警告：只改 `disabled_at` 会让人以为「停用没生效」 |
| POST | `/api/v1/users/{id}/enable` | ADMIN | 恢复账号 | — | `ViewUser` | — | — | 「停用不是删除」（主稿 3.4 的同一条纪律） |
| POST | `/api/v1/users/{id}/role` | ADMIN | 改角色 | `ReqChangeRole{ role: ADMIN\|REQUESTER }` | `ViewUser` | — | — | **OWNER 不可被改**：主稿第 9 节明确「不依赖『界面上不显示开关』——那是 UI 事实，不是安全边界」。服务层显式判断并返 `OWNER_IMMUTABLE`（实现篇 6.2）。**`role` 的取值里没有 `OWNER`**——转移主理人目前无设计（主稿 13.7） |
| DELETE | `/api/v1/users/{id}/sessions` | ADMIN | 踢掉某人所有会话 | — | 204 | — | — | 「重置密码 / 停用」之外运维手册要用的第三只手（运行时篇 R7.4） |

> **账号不删除。** 主稿 10.6：`equipment_history.actor_id` 指向 `app_user`，**不要为了清理账号去删行**。所以本节**没有 `DELETE /api/v1/users/{id}`**。这不是遗漏，是设计。

### 1.4 器材：建档、列表、详情、编辑、归档

| 方法 | 路径 | 权限档 | 用途 | 请求体类型 | 响应类型 | 分页 | 幂等 | 备注 |
|---|---|---|---|---|---|---|---|---|
| GET | `/api/v1/equipment` | REQUESTER | 列表：搜索 + 四个筛选器 + 排序 | — | `{items:[EquipmentView], nextCursor}` | 游标 `(created_at,id)` 降序；`limit` 默认 50 上限 100 | — | 主稿 7.5 的形状。查询参数：`q` `category` `status` `brand` `sort` `cursor` `limit` **+ `includeSold`**（主稿 3.4：列表默认同时排除已归档与已出售，用一个「显示已出售」开关看回）。**实现篇 7.1 硬规则**：`QueryBuilder` 的每段 SQL 都是编译期常量，用户输入只走 `.bind()`。**申请人被 `requester_can_browse=false` 挡住时返 403** |
| POST | `/api/v1/equipment` | ADMIN | 建档。`code` 留空则自动生成 | `ReqEquipmentCreate` | `EquipmentView` | — | — | 主稿 3.5 的 `category.next_seq` 分配（`UPDATE` 自带行锁）。**编号格式不做校验**（主稿 3.5：手填 `CAM-001-备用机` 合法）。日志篇 `biz.create`。**返回新对象**（主稿 7.6：变更返回更新后的资源，不返回 `{"ok": true}`） |
| GET | `/api/v1/equipment/{id}` | REQUESTER | 详情 | — | `EquipmentView` | — | — | 主稿 7.7 的角色裁剪在此生效。**已归档的返 404**（主稿 7.3 的 404 行原文是「不存在或已归档」） |
| PATCH | `/api/v1/equipment/{id}` | ADMIN | 改描述性字段（`purchaseDate` 等走 **PATCH 三态**） | `ReqEquipmentPatch` | `EquipmentView` | — | — | 主稿 7.6 的 `Option<Option<T>>`。**一次提交写一条 `UPDATE` 历史，只记真正变了的字段**（主稿 5.5、运行时篇 R4.2、R3.2 #12）。日志篇 `biz.update` 记 `changed_count` 与 `fields[]`，**不记值**（`detail` 在库里） |
| DELETE | `/api/v1/equipment/{id}` | ADMIN | **归档**（`archived_at`），不是物理删除 | — | 204 | — | — | 主稿 3.4。**出库中禁止归档** → `CHECKED_OUT_NOT_ARCHIVABLE`。**这个动词是 DELETE 但语义是归档**，主稿 3.4 明确「不做软删除」指的是不另建一套软删列，归档就长在 `archived_at` 上 |
| POST | `/api/v1/equipment/{id}/restore` | ADMIN | 恢复归档 | — | `EquipmentView` | — | — | 主稿 3.4「归档可恢复」。`equipment_history.action = 'RESTORE'`（主稿 3.3 的 CHECK 里有）。日志篇 `biz.archive` 的 `restore=true` |
| POST | `/api/v1/equipment/batch` | ADMIN | 批量操作：归档/停用/恢复/出售 | `ReqBatch{ action, equipmentIds[] }` | `ViewBatchResult{ updated: EquipmentView[] }` | — | — | **整批拒绝、点名违规件、不做部分成功**（主稿 C4、5.3）。`message` 是「以下器材不能删除（出库中）：CAM-003、CAM-011」。**`action` 的封闭集合与 `equipment_history.action` 对齐**：`ARCHIVE`/`RESTORE`/`DISABLE`/`ENABLE`/`SOLD`。**批量取锁按 `equipment.id` 升序**（主稿 5.2、实现篇 3.3） |

> **`/api/v1/equipment/batch` 是提案，不是主稿既有。** 交接稿要求「批量操作」，主稿 5.3 只定义了它的**语义**（一个事务、不设 savepoint、先收集全部违规再返回），没定义它的**路径与请求形状**。本节按 §0.1 的风格补上：批量是「对集合的操作」，所以动作名词放在集合之后而不是某个 ID 之后。**如果主稿更希望每个状态各有批量端点（`POST /api/v1/equipment/batch/archive` 等），本节的通知是：那会让 `action` 从封闭枚举变成路径段，两者都行，但只能选一个**——见 §5-G6。
>
> **交接稿的「批量删除」在本节落成 `action=ARCHIVE`。** 主稿 3.4 与 14 节差异汇总第 1 条已经把「软删除」改写为「归档 + 已出售」，所以没有独立的删除动作。**这一点需要主稿确认措辞**（主稿 5.3 的原文仍在说「批量删除」与 `DomainError::BatchRejected` 的文案「不能删除」）。

### 1.5 器材：状态流转（出库 / 归还 / 送修 / 维修完成 / 停用 / 恢复 / 出售）

| 方法 | 路径 | 权限档 | 用途 | 请求体类型 | 响应类型 | 分页 | 幂等 | 备注 |
|---|---|---|---|---|---|---|---|---|
| POST | `/api/v1/checkouts` | ADMIN | **出库**（直接出库，非申请路径） | `ReqCheckout{ equipmentIds[], projectId?, projectName, userName, checkoutDate, expectedDate?, note? }` | `ViewCheckout` | — | **`Idempotency-Key`** | 运行时篇 R4.4 三处之一。`expectedDate` 可空 = 占用到 `infinity`（主稿 3.1、运行时篇 R3.2 #2）。**档期冲突返 409 `RESERVED`，不是 422**（主稿 7.3）。日志篇 `biz.checkout` |
| GET | `/api/v1/checkouts` | ADMIN | 出入库记录列表（`checkout` 表视角） | — | `{items:[ViewCheckout], nextCursor}` | 游标 `(created_at,id)` | — | **注意与 §5-G4 的关系**：主稿 10.1 定「界面只读一处」= `equipment_history`，所以出入库记录页读的是活动流（`scope=workspace`），**这个端点回答的是另一个问题**：「哪些出库单还开着、按项目/使用人筛选」。主稿没有定义它，见 §5-G4 |
| GET | `/api/v1/checkouts/{id}` | ADMIN | 出库单详情（含 items 与逐件归还状态） | — | `ViewCheckoutDetail` | — | — | 主稿 5.4 的读模型：部分归还是「`request_item` 里还有 `returned_at IS NULL`」，**不建状态字段**（主稿 14 节差异 #3）。出库单自身有 `OPEN`/`CLOSED` |
| GET | `/api/v1/checkouts/{id}/manifest` | ADMIN | **出库交接清单**（打印专用 DTO） | — | `ViewManifest`（打印专用） | — | — | 主稿 7.8 原文路径，本节只补 `/v1`。**角色守卫收在这一个 handler 里**；**不允许前端拿器材列表 DTO 拼清单**（清单含序列号，签字人可能是申请人） |
| POST | `/api/v1/checkouts/{id}/items/{itemId}/return` | ADMIN | **逐件归还**（一件一条记录） | `ReqReturn{ returnNote? }` | `ViewCheckoutDetail` | — | — | 主稿 5.4 七步；交接稿 5.3「一件一条记录」。**部分归还不需要建状态**（主稿 5.4）。日志篇 `biz.return` 的 `closed` 表示该出库单是否已全还 |

> **`DELETE` 不在状态流转里。** 交接稿要求「出库/归还/送修/维修完成/停用/恢复/出售」七种流转，本节给它们全部落了端点（见本表与下面三张表），**没有任何一个用 DELETE**——DELETE 只用于「归档」这一种纠错动作（§1.4）。这与主稿 3.4 的分工一致：**状态机改状态，归档管纠错，两者不混**。
>
> **`POST /api/v1/checkouts` 的路径需要主稿确认**，因为它与主稿 7.8 那条 `GET /api/checkouts/{id}/manifest` 是同一个资源的两半（列表侧与单件侧），但主稿从未写过 `POST /api/v1/checkouts`。**这是本节与日志篇 4.1 冲突的落点**：日志篇写的是 `PATCH /api/v1/equipment/checkout`。见 §5-G2。

### 1.6 器材：送修与维修完成、停用、恢复、出售、上传、导入、导出

| 方法 | 路径 | 权限档 | 用途 | 请求体类型 | 响应类型 | 分页 | 幂等 | 备注 |
|---|---|---|---|---|---|---|---|---|
| POST | `/api/v1/equipment/{id}/maintenance` | ADMIN | **送修**（`IN_STOCK` → `MAINTENANCE`） | `ReqMaintenanceOpen{ fault, vendor?, sentDate, costCents?, note? }` | `ViewMaintenance` | — | — | `fault` **必填**（主稿 3.3）。**同一件同时只能有一条未关闭的维修记录**（`maintenance_one_open` 部分唯一索引，主稿 3.3）。日志篇 `biz.maintenance` |
| POST | `/api/v1/maintenance/{id}/close` | ADMIN | **维修完成** | `ReqMaintenanceClose{ outcome: BACK_TO_STOCK\|DISABLED, note? }` | `ViewMaintenance` | — | — | 主稿 3.3 `maintenance.outcome` 的 CHECK 是这两个值。**`DISABLED` 与 `BACK_TO_STOCK` 是两个不同的目标状态**，状态机允许 `MAINTENANCE→IN_STOCK` 与 `MAINTENANCE→DISABLED`（实现篇第 4 节 ALLOWED 表）。日志篇 `biz.maintenance` |
| GET | `/api/v1/maintenance/{id}` | ADMIN | 维修记录详情 | — | `ViewMaintenance` | — | — | **提案，非主稿既有**。主稿 10.1 的映射表写「送修 / 维修完成 → 点开后显示：跳 `maintenance`：故障说明、维修单位、费用」——**而没有任何端点能取它**，活动流的 `ref: {kind:"maintenance", id:"55"}`（主稿 10.5）会点不开。缺口见 §5-I33 |
| POST | `/api/v1/equipment/{id}/disable` | ADMIN | **停用**（`IN_STOCK` → `DISABLED`） | `ReqNote{ note? }` | `EquipmentView` | — | — | 主稿 3.4：「东西还在，不出借」。「停用不是删除」 |
| POST | `/api/v1/equipment/{id}/enable` | ADMIN | **恢复**（`DISABLED` → `IN_STOCK`） | — | `EquipmentView` | — | — | 实现篇 ALLOWED 表里 `DISABLED→IN_STOCK` 合法。**注意 `DISABLED→SOLD` 也合法**（同一张表） |
| POST | `/api/v1/equipment/{id}/sell` | ADMIN | **出售**（`IN_STOCK`/`DISABLED` → `SOLD`） | `ReqNote{ note? }` | `EquipmentView` | — | — | **`SOLD` 是状态机终点，没有出边**（实现篇 ALLOWED 表）。**「已出售」不等于归档**（主稿 3.4，两者是两件事） |
| POST | `/api/v1/photos` | ADMIN | 照片上传（multipart），返回结构化引用 | `multipart/form-data`（字段名待定，见 §5-I7） | `PhotoRef`（主稿 6.4 的形状） | — | **`Idempotency-Key`** | **8 MiB 上限、只收 JPEG、校验魔数不信任 Content-Type**（主稿 6.3/6.7）。**内容寻址 ⇒ 同图重传直接返回引用不做任何写**（主稿 6.2）。**返回 `sha256`** 让前端能判断是否复用。日志篇 `upload.ok` 的 `deduped` 表达是否命中已有内容。**上传与器材保存是两个请求**（主稿 6.3），所以本端点不改 `equipment.photo_id` |
| POST | `/api/v1/equipment/import/preview` | ADMIN | CSV 导入**第一阶段**：解析 + 校验，不落库 | `multipart/form-data`（CSV 文件） | `ViewImportPreview` | — | — | 运行时篇 R5：**检测编码并回显**（`encodingDetected`）、**金额规范化成整数分、四舍五入不截断**、失败行标红**不静默当 0**。主稿 7.9：**预览只回前 12 行**，不要把 1000 行推给前端。日志篇 `import.reject`：`rejected[]` **最多 20 条**、`value` 是**规范化前的原文** |
| POST | `/api/v1/equipment/import/commit` | ADMIN | CSV 导入**第二阶段**：提交。全有或全无 | `ReqImportCommit`（含服务端已解析的行，或重传文件——见备注） | `ViewImportResult` | — | **`Idempotency-Key`** | 运行时篇 R4.4 三处之一。**建议重校验**（运行时篇 R5.3）：提交时重跑校验，与预览不一致则拒绝并要求重新预览。**去重策略未定**（主稿 13.9）→ 见 §5-I8。**1000 行上限、请求体 5 MiB 上限（在读流时生效）**（主稿 5.1、运行时篇 R5.4） |
| GET | `/api/v1/equipment/import/template` | ADMIN | 下载 CSV 模板 | — | `text/csv`（带 BOM，见备注） | — | — | **提案，非主稿既有**。交接稿 5.1 的模板列固定为 `器材编号,器材名称,分类,品牌,型号,序列号,存放位置,购买日期,购买价格,备注`。日志篇 6 节的 `export` 事件有 `kind`（数据/模板）两种，**说明模板下载是设计里预期的动作**，但没有任何文档给它路径。见 §5-I1 |
| GET | `/api/v1/export` | OWNER 或 ADMIN | **数据导出备份** | — | 文件流（`Content-Disposition: attachment`） | — | — | **主稿 13.2 是阻塞项**：默认方案是「仅 OWNER/ADMIN 可导出、导出**原始数据**（含价格/序列号）、文件头写入导出人/时间/范围」。**范围是「导出什么」这件事，主稿未定** → 见 §5-I9（3 个选项，本节不猜）。日志篇 `export` 的 `masked` 布尔说明「脱敏导出」是预期中的第二个形态（主稿 13.2 末句） |

> **`GET /api/v1/equipment/import/template` 与 `GET /api/v1/export` 都不在 `/equipment` 之下**，因为它们导出的不是「某一个器材资源」，而是一份文件。**这是 §0.1 风格的一次应用**：动作名词的单段规则没错，但资源归属要按「被创建/被产出的是什么」判定，不能按「从哪个页面点的」。模板是导入的输入，所以留在 `/equipment/import/` 下；全库导出产出的是一个与资源无关的文件，所以放 `/export`。**如果主稿倾向于把导出也放进 `/equipment/`，那它必须同时解释「为什么导出的是器材而不是项目/申请/全部」**——主稿 13.2 没回答这个问题。
>
> **`GET /api/v1/export` 的权限档写「OWNER 或 ADMIN」而不是单档**，因为主稿 13.2 的默认方案同时点名了这两个角色，而实现篇 6.2 的 extractor 只有 `Admin`（含 OWNER）和 `Owner`（仅 OWNER）两个档，**没有「仅 ADMIN 不含 OWNER」的档**。所以它落成 `Admin`。这是一个**必须由主稿确认的细节**：若产品的意思是「OWNER 和 ADMIN 都能导」，用 `Admin` 即可；若「只有 OWNER 能导」，改 `Owner`。见 §5-G7。

### 1.7 申请：申请人侧与管理员侧

| 方法 | 路径 | 权限档 | 用途 | 请求体类型 | 响应类型 | 分页 | 幂等 | 备注 |
|---|---|---|---|---|---|---|---|---|
| POST | `/api/v1/requests` | REQUESTER | 申请人提交申请 | `ReqRequestCreate{ equipmentIds[], projectName, useStart, useEnd, note? }` | `ViewRequestRequester` | — | — | 交接稿 5.5：**未审的申请排前面**（`request_queue_idx`）。**`requester_can_request=false` 时返 `REQUESTING_DISABLED`**。`use_end >= use_start` 由 CHECK 兜底（主稿 3.3） |
| GET | `/api/v1/requests/mine` | REQUESTER | **我的申请**（只看自己） | — | `{items:[ViewRequestRequester], nextCursor}` | 游标 `(created_at,id)` | — | **路径用 `/mine` 而不是 `?requesterId=me`**：让「这件事的授权来源是会话本身」写在路径里，避免出现一个能被改参数绕过的端点。服务端**不接受**客户端指定 `requesterId` |
| GET | `/api/v1/requests/{id}` | REQUESTER | 申请详情 | — | `ViewRequestRequester` 或 `ViewRequestAdmin` | — | — | **同一个路径、两种响应类型**：申请人只能看自己的（否则 404，不是 403——见备注），管理员看全部。申请人视图里**每件器材只有当前状态**（主稿 10.4 末句） |
| POST | `/api/v1/requests/{id}/withdraw` | REQUESTER | **撤回申请** | — | `ViewRequestRequester` | — | — | `status = WITHDRAWN`（主稿 3.3）。**同时 DELETE 相关 `occupancy` 行**（主稿 3.1 映射表：「撤回申请 / 申请作废」）。**只有自己的、且未交付的能撤回**——状态机之外的这条业务规则**主稿没写**，见 §5-I10 |
| GET | `/api/v1/requests` | ADMIN | **管理员申请列表**（待审队列） | — | `{items:[ViewRequestAdmin], nextCursor}` | 游标 `(created_at,id)`；默认按未审优先 | — | 筛选参数：`status`、`requesterId`、`projectId`。主稿 3.3 的 `request_queue_idx` 是 `(workspace_id, status, created_at)` |
| POST | `/api/v1/requests/{id}/approve` | ADMIN | **审批**（含部分通过） | `ReqApprove{ items: [{equipmentId, approved: bool}], reviewNote? }` | `ViewRequestAdmin` | — | — | **部分通过不进状态枚举**：主稿 3.1 映射表原文「批准时取消勾选某件（= 拒绝）→ 不 INSERT。**这就是『部分通过』的实现方式**」。`result` 取 `APPROVED`/`PARTIAL`/`REJECTED`（日志篇 `biz.approve`）。**批准即写 `occupancy` 的 `RESERVATION`，区间 `[use_start, use_end + 1天)`**（主稿 3.1 映射表第一行——注意 `+1天` 与半开区间的配合） |
| POST | `/api/v1/requests/{id}/deliver` | ADMIN | **办理交付**（`APPROVED`/`PARTIAL` → `DELIVERED`） | `ReqDeliver{ equipmentIds[], checkoutDate?, expectedDate?, userName }` | `ViewCheckout` | — | — | 主稿 3.1 映射表：「办理交付 → 该行 UPDATE：`kind='CHECKOUT'`，区间起点改为实际出库日期，写 `source_checkout_id`」。日志篇 `biz.deliver`。**注意它与 `biz.checkout` 是两个不同的日志 kind**，所以本节把它们做成两个端点（见 §5-G3） |
| POST | `/api/v1/requests/{id}/reject` | ADMIN | **整体拒绝**（一件都不批） | `ReqReject{ reviewNote }` | `ViewRequestAdmin` | — | — | **提案**：主稿 3.1 只描述「批准时取消勾选某件」，没说「整单拒绝」走哪个端点。语义上它等于 `approve` 且全部 `approved=false`，但那样 `status` 是 `PARTIAL` 还是 `REJECTED` 就有歧义（主稿 3.3 的 `request.status` CHECK 里 `PARTIAL` 与 `REJECTED` 都在）。见 §5-I11 |

> **申请人看不到的东西，在这个模块里是最密集的**：他看不到器材价格/序列号/内部备注（§0.2 第 1 条），看不到全库历史（§0.2 第 2 条），也看不到**时间**——主稿 7.7 把 `createdAt`/`updatedAt` 划给管理员视图。所以 `ViewRequestRequester` **不能带 `reviewedAt`**（主稿没有明说，但按 7.7 的字段清单推论成立）。**这条推论要主稿确认**，见 §5-I12。
>
> **申请人访问别人的申请返 404 而不是 403。** 理由：403 会泄漏「这个 ID 存在」。主稿 7.3 的 403 行原文是「角色不允许（申请人访问全库历史）」——那是**入口级**的拒绝，而「访问别人的资源」是**存在性**的问题。**这是本节的推断，主稿没有明说**，见 §5-I13。

### 1.8 项目

| 方法 | 路径 | 权限档 | 用途 | 请求体类型 | 响应类型 | 分页 | 幂等 | 备注 |
|---|---|---|---|---|---|---|---|---|
| GET | `/api/v1/projects` | ADMIN | 项目列表 | — | `{items:[ViewProject], nextCursor}` | 游标 `(created_at,id)` | — | 筛选：`status`、`projectType`。**为什么是 ADMIN**：`project.client` 是客户名，`project.note` 是备注——按主稿 7.7 的分工（内部备注不下发给申请人）推论，项目列表不该给申请人。这是**本节的推断**，见 §5-I14 |
| POST | `/api/v1/projects` | ADMIN | 新建项目 | `ReqProjectCreate{ name, client?, projectType, startDate?, endDate?, note? }` | `ViewProject` | — | — | `project_type` 是受约束枚举 `PROMO/SHORT_FILM/TVC/EVENT/CAMPUS/OTHER`（主稿 3.3，13.4 待定是否改表）。`(workspace_id, name)` 唯一 |
| GET | `/api/v1/projects/{id}` | ADMIN | 项目详情（含历史出库记录数） | — | `ViewProjectDetail` | — | — | 交接稿：项目卡片显示「历史出库记录数」。主稿 10.5 把这个数**升级成可点进去**的活动流（`scope=project`），所以详情里的这个数字要么由本端点算，要么由活动流的 `nextCursor` 是否存在来推断——**主稿没定**，见 §5-I15 |
| PATCH | `/api/v1/projects/{id}` | ADMIN | 改项目字段（三态） | `ReqProjectPatch` | `ViewProject` | — | — | 与器材编辑同形（主稿 7.6） |
| POST | `/api/v1/projects/{id}/status` | ADMIN | 项目状态流转 | `ReqProjectStatus{ status: ACTIVE\|COMPLETED\|ARCHIVED }` | `ViewProject` | — | — | 主稿 3.3 的 CHECK 是这三个值。**「状态流转」的合法性表主稿没给**——器材有五态单向流转表（实现篇第 4 节），项目没有。见 §5-I16 |
| DELETE | `/api/v1/projects/{id}` | ADMIN | 删除项目 | — | 204 | — | — | **语义未定**：`request.project_id` 与 `checkout.project_id` 都引用 `project(id)` 且**没有 `ON DELETE` 子句**（主稿 3.3），所以有引用的项目物理删除会因外键失败。主稿 3.3 的 `project` 表**没有 `archived_at` 列**，而 `status` 里有 `ARCHIVED`。见 §5-I16 |

### 1.9 分类

| 方法 | 路径 | 权限档 | 用途 | 请求体类型 | 响应类型 | 分页 | 幂等 | 备注 |
|---|---|---|---|---|---|---|---|---|
| GET | `/api/v1/categories` | REQUESTER | 分类列表 | — | `{items:[ViewCategory]}` | **无分页**（见备注） | — | 分类是十量级的小集合，且器材建档/筛选的下拉要一次性拿全 → **不分页**。`ViewCategory` 给申请人**不含 `nextSeq`**（它是内部计数器，泄漏了「你们有多少件器材」且与申请人无关） |
| POST | `/api/v1/categories` | ADMIN | 新建分类 | `ReqCategoryCreate{ name, codePrefix }` | `ViewCategory` | — | — | `code_prefix` 受 CHECK 约束 `^[A-Z0-9]{1,5}$`（主稿 3.3），`(workspace_id, name)` 与 `(workspace_id, code_prefix)` 双唯一 |
| PATCH | `/api/v1/categories/{id}` | ADMIN | 改分类名/前缀 | `ReqCategoryPatch{ name?, codePrefix? }` | `ViewCategory` | — | — | **改前缀不影响已有器材编号**（主稿 3.5 第 2 条：「生成后永不改变」）。这条最容易实现错——`code` 是在建档时**物化**到 `equipment.code` 的，不是运行时拼接的 |
| DELETE | `/api/v1/categories/{id}` | ADMIN | 删除分类 | — | 204 | — | — | **有器材引用时会因外键失败**（`equipment.category_id` 无 `ON DELETE`）。**主稿没定义这个失败翻译成哪个领域错误**（会是 500，除非补）→ 见 §5-I17 |

> **没有「重置 `next_seq`」端点。** 主稿 3.5 三条规则里没有任何一条要求重置，而且重置会让已发出的编号被复用（第 1 条明确说「删除后序号会被回收复用」是它**要避免**的行为）。**这是有意的缺席。**

### 1.10 活动流与总览与设置

| 方法 | 路径 | 权限档 | 用途 | 请求体类型 | 响应类型 | 分页 | 幂等 | 备注 |
|---|---|---|---|---|---|---|---|---|
| GET | `/api/v1/activity` | ADMIN | **一个接口，四个界面**（主稿 10.5） | — | `{items:[ViewActivityItem], nextCursor}` | 游标 `(created_at,id)` 降序；`limit` 默认 20 | — | 参数：`scope`（`equipment`\|`workspace`\|`project`\|`actor`）、`id`（`scope != workspace` 时必填）、`includeUpdates`、`cursor`、`limit`。**`scope=workspace` 默认不含 `UPDATE`**（主稿 10.5 末句）。**申请人一律 403**（主稿 10.4「完全不给入口」）。**`ids.equipment_id` 在日志里是单值**，所以本端点的 `id` 也是单值 |
| GET | `/api/v1/overview` | ADMIN | **总览页聚合数据** | — | `ViewOverview` | — | — | 主稿 11 节：逾期扫描「当前只写日志并在总览页查询里体现」；运行时篇 R1.8：超期未还件数「总览页本身就查」。**字段完全未定义** → 见 §5-I18（需要主稿补充，本节给候选字段但不当结论） |
| GET | `/api/v1/settings` | OWNER | 读工作区设置 | — | `ViewSettings` | — | — | 主稿 3.3 的 `workspace_setting` 两个开关。**默认都开**。交接稿：「主理人的两个开关」→ 所以是 OWNER 档，不是 ADMIN（ADMIN 能看到设置的效果但改不了） |
| PATCH | `/api/v1/settings` | OWNER | 改两个开关 | `ReqSettingsPatch{ requesterCanBrowse?: bool, requesterCanRequest?: bool }` | `ViewSettings` | — | — | 三态对布尔是无意义的（布尔只有真/假/不变三态，`Option<bool>` 已足够表达），**所以这里不用主稿 7.6 的 `Option<Option<T>>`**——7.6 那个形状是为了区分「清空」与「不改」，布尔没有「清空」这个状态。**这一条要写进主稿**，否则实现者会照 7.6 一律套两层 Option |

> **`GET /api/v1/settings` 的 `ViewSettings` 要不要下发给 REQUESTER？** 主稿 3.3 说这两个开关是「主理人的两个开关」，但**申请人是否需要知道「能不能申请」**决定了前端要不要提前禁用提交按钮。两种做法：(a) 申请人也读得到（`REQUESTER` 档，只读），靠 403 在提交时兜底；(b) 只有 OWNER/ADMIN 读得到，申请人靠申请端点返回的 `REQUESTING_DISABLED` 学。**本节按 (b) 写**（与「主理人的开关」这句一致），但见 §5-I19。

---

## 2. 响应形状说明

**不重复主稿已定义的结构。** 下面这张对照表先把「去主稿哪里看」钉死：

| 结构 | 定义位置 |
|---|---|
| `EquipmentRequester` / `EquipmentAdmin`（两个结构体、`flatten`、不得出现 `null`） | 主稿 7.7；实现篇 2 节（含「`Requester` 里一个字段都不许多」的坑） |
| `actions: [{key, enabled, reason}]` | 主稿 7.4 |
| 错误信封 `{error:{code,message,field,requestId}}` | 主稿 7.3 |
| 列表信封 `{items, nextCursor}` | 主稿 7.5 |
| `PhotoRef`（`{id,url,thumbUrl,width,height,sha256,originalName}`） | 主稿 6.4 |
| `ViewActivityItem`（`{id,at,actor,action,fromStatus,toStatus,changed,ref}`） | 主稿 10.5 |
| PATCH 三态（`Option<Option<T>>` 与 JSON 三种形态） | 主稿 7.6 |
| 枚举全部 `SCREAMING_SNAKE`、ID 全部字符串、金额 `*Cents` 整数、业务日期 `"YYYY-MM-DD"`、审计时间 RFC 3339 | 主稿 7.2 |

### 2.1 `ViewMe`（登录 / me / bootstrap / accept invite 共用）

```json
{
  "id": "7",
  "username": "zhangsan",
  "displayName": "张三",
  "role": "OWNER",
  "workspace": { "id": "3f1c…", "name": "LensMax 工作室" }
}
```

**不含权限矩阵。** 逐资源能做什么是 `actions`（主稿 D8、7.4）。**不含 `createdAt`** —— 主稿 7.7 把审计时间划给管理员视图，而 `ViewMe` 是账号自己的信息，不是器材，两种角色的 `/auth/me` 返回**同一个形状**（这是「两个结构体只用于器材」的一个边界，见 §2.10）。

### 2.2 `ViewUser`（成员列表 / 详情 / 停用 / 改角色共用）

```json
{
  "id": "9",
  "username": "lisi",
  "displayName": "李四",
  "role": "ADMIN",
  "disabledAt": null,
  "createdAt": "2026-09-01T02:11:00+08:00",
  "actions": [
    { "key": "changeRole", "enabled": true,  "reason": null },
    { "key": "disable",    "enabled": false, "reason": "主理人的权限不能被修改" }
  ]
}
```

**`actions` 里必须显式表达「OWNER 不可被改、不可被停用」**（主稿 7.4：状态机判断推到服务端；主稿第 9 节：不依赖「界面上不显示开关」）。`reason` 直接用 `DomainError::OwnerImmutable` 的中文文案「主理人的权限不能被修改」，这样按钮的 title 与服务端返回的 `message` 是同一句话。

**`disabledAt: null` 是允许的**——它不是权限字段（主稿 7.7 的禁令针对「看不见的字段用 null 表达」），而是「这个人当前是否被停用」这个事实。

### 2.3 `ViewInvite` / `ViewInviteWithToken`

```json
// GET /api/v1/invites → items[]
{ "id": "3", "role": "REQUESTER", "createdBy": { "id": "7", "name": "张三" },
  "expiresAt": "2026-10-08T12:00:00+08:00", "consumedAt": null, "consumedBy": null }

// POST /api/v1/invites → 唯一一次带明文 token
{ "id": "3", "role": "REQUESTER", "expiresAt": "2026-10-08T12:00:00+08:00",
  "token": "…", "acceptUrl": "https://…/invite/…" }
```

**`token` 字段在整个 API 里只出现这一次。** 主稿第 9 节：「明文 token 只在生成时返回一次，库里存 hash」。**日志里永远不许出现它**（运行时篇 R1.7、日志篇 L2）。`acceptUrl` 是服务端拼的完整链接——因为主稿第 9 节的用途是「生成链接手动复制」，让前端拼链接等于把「链接指向哪个环境」这个事实搬到了前端。

### 2.4 `ViewCheckout` / `ViewCheckoutDetail`

```json
// GET /api/v1/checkouts → items[]
{
  "id": "884",
  "project": { "id": "31", "name": "宣传片A" },
  "userName": "王五",
  "checkoutDate": "2026-09-30",
  "expectedDate": "2026-10-05",
  "status": "OPEN",
  "itemCount": 3,
  "returnedCount": 1,
  "isOverdue": false,
  "photo": null,
  "createdBy": { "id": "7", "name": "张三" },
  "createdAt": "2026-09-30T14:22:31+08:00"
}

// GET /api/v1/checkouts/{id}  → 上面全部 + items[]
{
  "…": "同上",
  "items": [
    { "id": "2201", "equipment": { "id": "1284", "code": "CAM-003", "name": "Sony FX3" },
      "returnedAt": null, "returnNote": null,
      "actions": [{ "key": "return", "enabled": true, "reason": null }] },
    { "id": "2202", "equipment": { "id": "1285", "code": "CAM-004", "name": "Sony FX6" },
      "returnedAt": "2026-10-03T09:10:00+08:00", "returnNote": "镜头盖遗失",
      "actions": [{ "key": "return", "enabled": false, "reason": "已归还" }] }
  ],
  "sourceRequest": { "id": "12" }
}
```

**两个必须做对的地方：**

1. **`isOverdue` 由服务端算**（主稿 7.4 末段：「超期是『今天 > 预计归还 且状态仍为出库』」）。**「今天」必须走 `business_today()`**，不是 `OffsetDateTime::now_utc().date()`——运行时篇 R2.3 说这个 bug 会让东八区每天 08:00 前的逾期判断**少算一天**，CI 里有一条 grep 检查守它。
2. **`expectedDate: null` 表示「无归还期限」**，不是「未填写所以不知道」。主稿 3.1：`end = infinity` 的语义是「在归还之前谁也别想预约它」。前端必须把 `null` 渲染成「无期限」，**不能渲染成空白**。

**`ViewCheckout` 里没有序列号。** 这是主稿 7.8 的推论：清单含序列号，所以清单走**专用 DTO**（`ViewManifest`），其余出库相关视图不带。**这条要主稿确认**，见 §5-I20。

### 2.5 `ViewManifest`（打印专用，主稿 7.8）

```json
{
  "checkoutId": "884",
  "title": "器材出库交接清单",
  "projectName": "宣传片A",
  "userName": "王五",
  "checkoutDate": "2026-09-30",
  "expectedDate": "2026-10-05",
  "items": [
    { "code": "CAM-003", "name": "Sony FX3", "brand": "Sony", "model": "ILME-FX3",
      "serialNumber": "SN-4471" }
  ],
  "signatureLines": ["经办人", "领用人"],
  "generatedAt": "2026-09-30T14:30:00+08:00"
}
```

**这是本节里唯一一个「故意含序列号且可能被申请人相关的人看到」的结构。** 主稿 7.8 的立场是：清单从专用接口取、**角色守卫收在这一个 handler 里**、**不允许前端拿器材列表 DTO 拼清单**。所以本端点档位是 `ADMIN`，而且**它不能复用 `EquipmentAdmin`**——复用的那一天就有人在申请人界面上拼出了清单。

**不含 `priceCents`。** 打印清单上不该出现采购价，而主稿 7.8 只点名了序列号。**这是本节的判断**，见 §5-I20。

### 2.6 `ViewRequestRequester` / `ViewRequestAdmin`

```json
// 申请人视图（GET /api/v1/requests/mine、GET /api/v1/requests/{id} 的自己那侧）
{
  "id": "12",
  "projectName": "宣传片A",
  "useStart": "2026-10-01",
  "useEnd": "2026-10-05",
  "status": "APPROVED",
  "note": "需要一个广角",
  "items": [
    { "equipment": { "id": "1284", "code": "CAM-003", "name": "Sony FX3",
                     "status": "IN_STOCK", "photo": null },
      "approved": true }
  ],
  "actions": [
    { "key": "withdraw", "enabled": true, "reason": null }
  ]
}
```

**申请人视图的形状纪律（§0.2 第 1–2 条）：**

- **`items[].equipment` 是 `EquipmentRequester`，不是 `EquipmentAdmin`。** 主稿 7.7 与实现篇 2.1 的唯一分支点在这里生效。
- **没有 `reviewedAt`、`reviewedBy`、`reviewNote`、`createdAt`、`updatedAt`。** 前三个是审批痕迹（含管理员内部评语），后两个是审计时间。
- **没有 `projectId`**，只有 `projectName`（`request.project_name` 是提交时的自由文本，`project_id` 是批准/交付时关联的，主稿 3.3）。申请人看到内部项目 ID 没有用途。
- **`approved: null|true|false` 要保留三态**：`null` = 未审。这对「我的申请还没人看」很重要。主稿 3.3 的 `request_item.approved` 是三值列。
- **`actions` 里 `withdraw` 的可用性由服务端算**（主稿 D8）。它必须在以下情况 `enabled: false`：不是自己的、状态已是 `WITHDRAWN`/`RETURNED`/`DELIVERED`。

```json
// 管理员视图（GET /api/v1/requests）
{
  "id": "12",
  "requester": { "id": "9", "name": "李四" },
  "projectName": "宣传片A",
  "project": { "id": "31", "name": "宣传片A" },
  "useStart": "2026-10-01",
  "useEnd": "2026-10-05",
  "status": "PENDING",
  "note": "需要一个广角",
  "reviewNote": null,
  "reviewedBy": null,
  "reviewedAt": null,
  "createdAt": "2026-09-29T10:02:00+08:00",
  "items": [
    { "id": "1401",
      "equipment": { "id": "1284", "code": "CAM-003", "name": "Sony FX3",
                     "status": "IN_STOCK", "purchasePriceCents": 1280000,
                     "serialNumber": "SN-4471", "…": "EquipmentAdmin 全部字段" },
      "approved": null, "deliveredAt": null, "returnedAt": null,
      "occupancy": { "during": ["2026-10-01", "2026-10-06"], "kind": "RESERVATION" } }
  ],
  "actions": [
    { "key": "approve", "enabled": true,  "reason": null },
    { "key": "reject",  "enabled": true,  "reason": null },
    { "key": "deliver", "enabled": false, "reason": "尚未审批" }
  ]
}
```

**`occupancy.during` 的 `end` 是半开区间的右端**，所以「申请用到 10-05」在表里是 `[2026-10-01, 2026-10-06)`（主稿 3.1 映射表：`[使用开始, 预计归还 + 1天)`）。**这一处 +1 天是最容易实现错、且错了会静默放行/静默拦截的地方**——接口上把它原样给出（`10-06`）会让前端以为是「用到 6 号」。**这里有一个主稿未定的问题，见 §5-I21**（接口到底给原始区间还是给人读的 `useStart`/`useEnd`）。本节按「给原始区间」写，因为它可核验。

### 2.7 `ViewMaintenance`

```json
{
  "id": "55",
  "equipment": { "id": "1284", "code": "CAM-003", "name": "Sony FX3" },
  "fault": "对焦环异响",
  "vendor": "XX 维修",
  "sentDate": "2026-09-20",
  "costCents": 48000,
  "note": null,
  "openedAt": "2026-09-20T09:00:00+08:00",
  "closedAt": null,
  "outcome": null,
  "createdBy": { "id": "7", "name": "张三" }
}
```

**`outcome: null` 表示维修未完成**，与 `closedAt: null` 同义。**同一件同时只能有一条未关闭的维修记录**（主稿 3.3 的 `maintenance_one_open` 部分唯一索引），所以列表里同一件器材最多出现一次 `outcome: null`。

### 2.8 `ViewImportPreview` / `ViewImportResult`

```json
// POST /api/v1/equipment/import/preview
{
  "encodingDetected": "GBK",
  "rowCount": 412,
  "previewRows": [
    { "row": 2, "code": "CAM-001", "name": "Sony FX3", "category": "摄影机",
      "brand": "Sony", "model": "ILME-FX3", "serialNumber": "SN-4471",
      "location": "A柜", "purchaseDate": "2024-03-01", "priceCents": 1280000,
      "note": null, "status": "OK" },
    { "row": 7, "status": "REJECTED", "column": "分类", "value": "摄影机",
      "reason": "分类不存在" }
  ],
  "rejected": [
    { "row": 7,  "column": "分类",     "value": "摄影机",     "reason": "分类不存在" },
    { "row": 19, "column": "购买价格", "value": "¥1,28O.00", "reason": "金额无法解析" },
    { "row": 88, "column": "器材名称", "value": "",           "reason": "必填为空" }
  ],
  "truncated": false,
  "willSkip": 3
}
```

**四条与文档绑死的约定：**

1. **`encodingDetected` 必带**（运行时篇 R5.1：「并把实际检测到的编码回显在预览界面」；日志篇 4.4：「`encoding_detected` 必带」）。值是 `UTF-8` / `UTF-8-BOM` / `GBK`。
2. **`previewRows` 最多 12 行**（主稿 7.9：「预览只回前 12 行——不要让『导入预览』把 1000 行全推给前端」）。
3. **`rejected[].value` 是规范化前的原文**（日志篇 4.4：「`¥1,28O.00` 里那个字母 O 就是原因，不要规范化后再记」）。**`rejected[]` 上限 20 条 + `truncated`**（日志篇 4.4）——**接口与日志用同一个上限与同一份数据**，这样日志与响应不会各说一套。
4. **`priceCents` 已经规范化成整数分且四舍五入**（运行时篇 R5.2：「不要静默当 0」——解析失败的行进 `rejected`，不进 `previewRows`）。

**`willSkip` 依赖主稿 13.9 那个未决项**（重复判定：按编号去重还是每次全新建）。**本节把字段留出来，但它的含义要等主稿拍板** → 见 §5-I8。

```json
// POST /api/v1/equipment/import/commit
{ "createdCount": 409, "skippedCount": 3, "encodingDetected": "GBK", "rowCount": 412 }
```

**提交失败的行为是「全有或全无」**（主稿 5.3、C4），失败时返回 `BATCH_REJECTED` 并点名违规行 —— **不是 200 + 部分成功**。**所以本响应只有「全成功」一种形态**，没有 `partial` 字段。这是与「不做部分成功」一致的必要结果。

### 2.9 `ViewOverview`（**字段待定，本节只给候选，不当结论**）

主稿与运行时篇都只说了「总览页查询里体现逾期」，**没有一处定义总览页的字段**。本节**不给结论**，给 2–3 个选项供主稿选择（见 §5-I18）。下面是候选形状，**仅供讨论**：

```json
{
  "overdueCount": 3,
  "statusCounts": { "IN_STOCK": 412, "CHECKED_OUT": 18, "MAINTENANCE": 2,
                    "DISABLED": 5, "SOLD": 31 },
  "openCheckoutCount": 15,
  "pendingRequestCount": 4,
  "overdueItems": [ { "equipmentId": "1284", "code": "CAM-003", "name": "Sony FX3",
                      "expectedDate": "2026-09-25", "checkoutId": "884",
                      "overdueDays": 5 } ],
  "recentActivity": [ /* ViewActivityItem[]，最多 20 条，复用 scope=workspace 的读法 */ ]
}
```

### 2.10 一条关于「两个结构体」的边界说明（本节提出，需要主稿确认）

主稿 7.7 与实现篇 2 节规定「可见性用两个序列化结构体」，但那两个结构体**是 `Equipment` 专属的**。本节在盘点中用到 `ViewCheckout`、`ViewRequestRequester`/`ViewRequestAdmin`、`ViewUser`、`ViewProject`、`ViewActivityItem` 等**同样按角色分叉**的结构，它们在主稿里**没有对应类型**：

| 结构 | 是否按角色分叉 | 主稿有定义吗 |
|---|---|---|
| `EquipmentRequester` / `EquipmentAdmin` | 是 | **有**（主稿 7.7） |
| `ViewRequestRequester` / `ViewRequestAdmin` | 是 | 无。但主稿 10.4 的「申请人的界面里只需要他的申请单里每件器材当前的状态」隐含了它 |
| `ViewCheckout` / `ViewCheckoutDetail` | 否（仅 ADMIN） | 无 |
| `ViewManifest` | 否（仅 ADMIN，**故意含序列号**） | 主稿 7.8 点了名但没给结构 |
| `ViewActivityItem` | 否（仅 ADMIN，**申请人不给入口**） | **有**（主稿 10.5） |
| `ViewMe` / `ViewUser` | 否 | 无 |

**这是本节的判断：主稿 7.7 那条纪律（「被禁字段一律不出现，不用 `null`」）是对所有 DTO 的，不只对器材。** 所以 `ViewRequestRequester` 必须是**独立结构体**，而不是「`ViewRequestAdmin` 去掉几个字段」——实现篇 2 节那个 `flatten` 的坑（往 `base` 里加字段会跟着出现在管理员响应里并造成重复键）在申请上会原样重现。**需要主稿把这条推广写下来**，见 §5-I22。

---

## 3. 状态码矩阵

**核对基准：主稿第 8 节。** 主稿 7.3 给了「情况 → 码」的表，实现篇 5.2 给了 `DomainError` → `(status, code)` 的 `match`，本表把两者合起来并逐端点落位。

### 3.1 全局横切（每个端点都可能返回）

| HTTP | `code` | `DomainError` 变体（主稿第 8 节） | 触发条件 | 依据 |
|---|---|---|---|---|
| 400 | `VALIDATION` | `AppError::Validation(garde::Report)` | 缺必填、日期区间反了、字段超长 | 主稿 7.3 第 1 行；实现篇 5.2 |
| 401 | `UNAUTHENTICATED` | `AppError::Auth(..)` | 未登录 / 会话失效 / cookie 缺失 | 主稿 7.3；实现篇 5.2、6.1 |
| 403 | `FORBIDDEN` | `AppError::Forbidden(..)` | 角色不允许（如申请人访问全库历史） | 主稿 7.3；实现篇 6.2 |
| 503 | `INTERNAL` | `AppError::Sqlx` / `Io` | 数据库不可用 | 运行时篇 R7.1 明确要求「所有请求 503（**不是 500**，这样前端能区分『稍后重试』和『出错了』）」 |
| 500 | `INTERNAL` | `AppError::Sqlx` / `Io` | 其它内部错误。**`Sqlx`/`Io` 的 `Display` 一律不返回给用户**，只给 `requestId` | 主稿 7.3；实现篇 5.2 末段 |

> **500 与 503 的分界主稿没有写，运行时篇 R7.1 写了。** 本表按运行时篇落位，**并标注这是一条需要回写主稿第 8 节的补充**，见 §5-I23。
>
> **429 主稿 7.3 提到（「限流 / 事务重试耗尽 → 429 / 503」）但没有任何文档定义限流中间件或限流错误变体。** 见 §5-I23。

### 3.2 领域错误 → HTTP（逐条对照主稿第 8 节的枚举）

| `DomainError` 变体（主稿第 8 节原文） | HTTP | `code` | `field` | 会出现在哪些端点 |
|---|---|---|---|---|
| `IllegalTransition { from, to }` | **422** | `ILLEGAL_TRANSITION` | — | §1.4 `batch`；§1.5 出库/归还；§1.6 送修/完成/停用/恢复/出售；§1.7 approve/deliver/withdraw。**`PATCH /equipment/{id}` 不返回它**（编辑描述性字段不过状态机） |
| `Reserved { project, start, end }` | **409** | `RESERVED` | — | §1.5 `POST /checkouts`、§1.6 `POST /equipment/{id}/maintenance`？**不**——只有需要占档期的两个写路径：出库（§1.5）与审批（§1.7 approve）。**归还不返回它** |
| `BatchRejected { items }` | **422** ⚠️ | `BATCH_REJECTED` | — | §1.4 `batch`、§1.5 `checkout`、§1.6 `import/commit`。**⚠️ 主稿第 8 节没给 `BatchRejected` 的 HTTP 码，实现篇 5.2 的 `match` 里也没有它**（`match` 只有 `Validation`/`IllegalTransition`/`Reserved`/`OwnerImmutable`/`Auth`/`Forbidden`/`Upload`/`Sqlx`）。**主稿 7.3 的表里没有「批量拒绝」这一行。** 本节按「它是状态流转违规的集合形态」落 422。见 §5-I24 |
| `CheckedOutNotArchivable` | **409** ⚠️ | `CHECKED_OUT_NOT_ARCHIVABLE` | — | §1.4 `DELETE /equipment/{id}`、`batch`（`action=ARCHIVE`）。**⚠️ 同上：主稿第 8 节与实现篇 5.2 都没有它的码。** 本节按「与当前状态冲突」落 409（与 `Reserved` 一致）。见 §5-I24 |
| `DuplicateCode { code }` | **409** ⚠️ | `DUPLICATE_CODE` | `code` | §1.4 建档与编辑（手填编号）、§1.6 `import/commit`。**⚠️ 编码缺失，见 §5-I24。且这里有一个更严重的问题**：实现篇 5.1 把 `23505` 按**约束名**翻译，但 `equipment_code_uniq`、`category` 的两个唯一约束、`app_user(workspace_id, username)`、`photo(workspace_id, sha256)` **全是 `23505`**，而 `DomainError` 里**只有 `DuplicateCode` 没有「分类重名」「用户名已存在」这类变体** → 见 §5-I25 |
| `OwnerImmutable` | **403** | `OWNER_IMMUTABLE` | — | §1.3 `POST /users/{id}/role`、`POST /users/{id}/disable`。**主稿第 9 节明确「显式列出 OWNER 不可被改，不依赖『界面上不显示开关』」**。**403 而不是 422**（主稿第 8 节的枚举顺序与实现篇 5.2 一致） |
| `RequestingDisabled` | **403** ⚠️ | `REQUESTING_DISABLED` | — | §1.7 `POST /requests`。**⚠️ 主稿第 8 节的「情况」表里没有这一行**（`RequestingDisabled` 只在枚举里出现）。本节按「角色/开关不允许」落 403。见 §5-I24 |
| `Conflict`（**枚举里没有，但两处代码在用**） | **409** | `RESERVED`？ | — | 实现篇 5.1 原文：`Some("23P01") => AppError::Domain(DomainError::Conflict)`；主稿第 8 节的示例代码同样写 `DomainError::Conflict`。**但主稿第 8 节的 `DomainError` 枚举里没有 `Conflict` 变体。** 见 §5-I26（这是一处必须补的编译级不一致） |
| —（**枚举里没有**） | **404** | 待定 | — | 「不存在或已归档」（主稿 7.3 原文）。**没有任何 `DomainError` 变体表达它**，实现篇 5.2 的 `match` 里也没有 404 分支。**所有带 `{id}` 的端点都需要它** → §5-I26 |
| —（**枚举里没有**） | **403** | `FORBIDDEN` / `REQUESTING_DISABLED`？ | — | `requester_can_browse = false` 时申请人读器材列表/详情。**主稿 3.3 有这个开关但第 8 节没有它对应的错误** → §5-I2 |

### 3.3 逐端点的非 2xx

只列**该端点特有的**非 2xx；3.1 的四行（400/401/403/500/503）在每一行都成立，不再重复。

| 端点 | 特有的非 2xx |
|---|---|
| `POST /auth/login` | **401** `UNAUTHENTICATED`（用户名或密码错，**故意不区分**，否则枚举账号）。日志 `auth.fail` + 403 拒绝记 WARN（运行时篇 R1.4） |
| `POST /bootstrap/owner` | **409** `OWNER_IMMUTABLE`（已有 OWNER，一次性入口永久失效——主稿第 9 节）。**⚠️ 语义上这是「入口已关闭」而不是「主理人不可改」，复用 `OwnerImmutable` 是本节的选择，见 §5-I27** |
| `POST /invites/{token}/accept` | **404**（token 不存在）；**410 Gone** 或 **409**（已消费/已过期）。**主稿第 8 节没有任何一行对应「邀请失效」** → §5-I28 |
| `POST /equipment` | **409** `DUPLICATE_CODE`（手填编号撞车）；**404**（`categoryId` 不存在）；**400**（编号格式？——**不，主稿 3.5 明确不做编号格式校验**） |
| `PATCH /equipment/{id}` | **404**（不存在或已归档）；**409** `DUPLICATE_CODE`（改成别人已用的编号）；**409** `ILLEGAL_TRANSITION`？**不**——主稿 3.4 与 5.5 的分工是「状态机改状态、字段编辑不改状态」，**所以 PATCH 不允许携带 `status`**。若客户端传了 `status`，应 400 `VALIDATION`（未知字段）而不是 422 |
| `DELETE /equipment/{id}` | **404**；**409** `CHECKED_OUT_NOT_ARCHIVABLE` |
| `POST /equipment/batch` | **404**（任一 ID 不存在）；**422** `BATCH_REJECTED`（**整批拒绝，`message` 里点名全部违规件**——主稿 5.3 要求「先在步骤 2 把所有违规件收集齐再返回，让用户一次看到全部问题」）；**409** `CHECKED_OUT_NOT_ARCHIVABLE`（`action=ARCHIVE` 时） |
| `POST /checkouts` | **409** `RESERVED`（档期被占，**`message` 点名是哪件、被哪个项目占**）；**422** `ILLEGAL_TRANSITION`；**404**；**400**（`expectedDate < checkoutDate` 由 CHECK 兜底，见主稿 3.3）；**409** `DUPLICATE_CODE`？**不**——但**重复提交**由 `Idempotency-Key` 处理（R4.4），**没有 key 时第二次出库会被 `occupancy_no_overlap` 拦住并翻译成 `RESERVED`**（实现篇第 4 节第 2 点明确：「重复出库必须由占用层的排他约束拦住」） |
| `POST /checkouts/{id}/items/{itemId}/return` | **404**；**422** `ILLEGAL_TRANSITION`（该件已归还 / 出库单已 `CLOSED`）；**400**（item 不属于该 checkout——**这个校验主稿没写，见 §5-I29**） |
| `GET /checkouts/{id}/manifest` | **404**（不存在）；**403**（**申请人的正确响应**——清单含序列号） |
| `POST /equipment/{id}/maintenance` | **409**（**该件已有一条未关闭的维修记录**，`maintenance_one_open` 部分唯一索引触发 `23505`）。**⚠️ 这个 409 没有对应的 `DomainError` 变体**（实现篇 5.1 的 `23505` 分支只处理 `equipment_code_uniq` 与 `app_user_one_owner`）→ §5-I25；**422**（该件不是 `IN_STOCK`） |
| `POST /maintenance/{id}/close` | **404**；**422**（已关闭）；**400**（`outcome` 取值不在 `BACK_TO_STOCK`/`DISABLED` 里） |
| `GET /maintenance/{id}` | **404**（不存在）。**⚠️ 该端点本身是提案**（§5-I33） |
| `POST /photos` | **400** `UPLOAD_FAILED`（实现篇 5.2：`AppError::Upload` → 400 `UPLOAD_FAILED`）；**413 Payload Too Large**（超 8 MiB）。**⚠️ 主稿 7.3 没有 413 这一行，运行时篇 R1.3 的 `stage=size_limit` 说明超限是预期路径** → §5-I30。**403** 非管理员 |
| `POST /equipment/import/preview` | **400**（编码无法识别 / 文件为空）；**413**（超 5 MiB——运行时篇 R5.4 要求「在读流的时候生效」）；**400** `VALIDATION`（列名不匹配）。**注意：坏行不是错误**——它们进 `rejected[]`，**预览仍返 200** |
| `POST /equipment/import/commit` | **422** `BATCH_REJECTED`（**整批拒绝**，主稿 5.3）；**409** `RESERVED`？**不**——导入不建占用；**409** `DUPLICATE_CODE`（重校验发现编号已被别处占用，**这正是运行时篇 R5.3「重校验」要拦的场景**）；**409**（与预览结果不一致，要求重新预览——**主稿没定这个码** → §5-I31） |
| `GET /export` | **403**（非 OWNER/ADMIN）；**503**（导出进行中的资源耗尽——主稿未定） |
| `POST /requests` | **403** `REQUESTING_DISABLED`（`requester_can_request = false`）；**404**（`equipmentId` 不存在）；**400**（`useEnd < useStart`，CHECK 兜底）；**403**（`requester_can_browse = false` 时**连申请都不该能提**——**主稿没定义这个组合** → §5-I2） |
| `GET /requests/mine` | 无特有（列表恒为空集而非 404） |
| `GET /requests/{id}` | **404**（不是自己的 / 不存在 —— **同一个码，不区分**，防枚举，见 §5-I13） |
| `POST /requests/{id}/withdraw` | **404**（不是自己的）；**422** `ILLEGAL_TRANSITION`（已交付/已归还/已撤回） |
| `POST /requests/{id}/approve` | **404**；**422**（已审过）；**409** `RESERVED`（**批准的档期与另一条占用冲突**——主稿 3.1 的 `occupancy_no_overlap` 在这里也会触发，`23P01` → 409。**这是很容易漏的一条**：审批也要占档期） |
| `POST /requests/{id}/deliver` | **404**；**422**（未审/已交付）；**409** `RESERVED`（**主稿 3.1 的交付是 UPDATE 已有行而不是 INSERT**，所以理论上不会冲突；但「区间起点改为实际出库日期」若把起点改到与另一条占用重叠，**UPDATE 也会触发排他约束** → 409。这条要写进主稿） |
| `POST /requests/{id}/reject` | **404**；**422**（已审过） |
| `POST /projects` | **409**（`(workspace_id, name)` 重名，**⚠️ 无对应 `DomainError`** → §5-I25） |
| `DELETE /projects/{id}` | **404**；**409**（有 `request`/`checkout` 引用，外键失败，**⚠️ 无对应 `DomainError`** → §5-I32） |
| `POST /projects/{id}/status` | **404**；**422**（非法流转——**项目的状态机表不存在** → §5-I16） |
| `POST /categories` | **409**（name 或 codePrefix 重名，**⚠️ 无对应 `DomainError`** → §5-I25）；**400**（`codePrefix` 不匹配 `^[A-Z0-9]{1,5}$`，CHECK 兜底） |
| `PATCH /categories/{id}` | **409**（同上）；**404** |
| `DELETE /categories/{id}` | **404**；**409**（有器材引用，外键失败 → §5-I17） |
| `GET /activity` | **403**（**申请人一律 403**——主稿 10.4「完全不给入口」）；**400**（`scope != workspace` 但缺 `id`；`scope` 不在四个值里） |
| `PATCH /settings` | **403**（非 OWNER——运行时篇 R7.4 第 3 条：「在它被设计之前，这个操作是不存在的」，所以主理人交接也没有端点） |
| `GET /readyz` | **503**（数据库不可连 / 迁移未到最新——运行时篇 R1.9） |
| `GET /healthz` | **无**。它不碰数据库，所以永远 200（除非进程死了，那就没有响应） |

### 3.4 三条必须在实现时验证的码

| 检查 | 怎么做 | 依据 |
|---|---|---|
| `23P01` 真的翻成 409 + 中文人话 | **手工触发一次**（不经预检查直接插两条重叠占用） | 运行时篇 R7.5 验收清单 |
| `23505` 按**约束名**翻译 | 迁移里每条约束显式命名（`equipment_code_uniq` 等），不依赖 Postgres 默认名 | 主稿第 8 节末段、实现篇 5.1 要点 1 |
| 业务拒绝记 INFO、不记 ERROR | 422/409 走 `biz.reject` 且级别 INFO；只有 `23P01` 实际触发才记 `biz.conflict` 的 **WARN** | 运行时篇 R1.4 明确禁止 4xx 记 ERROR；日志篇 6.1、L7 |

---

## 4. 实现顺序建议（按前置依赖）

**这一节的判据是「谁是谁的前置」，不是「谁重要」。** 主稿附录与实现篇第 11 节给的是**功能顺序**，本节给的是**端点依赖顺序**——两者的差别在最前面几步：主稿的第 1–3 步（迁移、domain、错误出口）在本节里是「没有任何端点，但所有端点都依赖它」。

### 4.0 第 0 层：没有端点，但必须最先存在

| # | 产出 | 为什么它是所有端点的前置 | 依据 |
|---|---|---|---|
| 0.1 | `migrations/0001_init.sql`（含 `occupancy` 与排他约束，**每条约束显式命名**） | 没有表就没有端点。**约束名是 409 能否翻译的前提** | 主稿附录 1；实现篇 5.1、7.3 |
| 0.2 | `domain`：状态机常量表、区间纯函数、`DomainError` | 所有状态流转端点的合法性判断只有一个落点 | 主稿 D2；实现篇第 4 节 |
| 0.3 | `AppError` 的 SQLSTATE 翻译（`23P01`/`23505`） | 决定 409 与 500 的分界。**先于所有写端点** | 主稿附录 3；实现篇 5.1 |
| 0.4 | `Actor` / `Admin` / `Owner` 三个 extractor + 中间件顺序测试 | 所有非 PUBLIC 端点的签名依赖它们 | 实现篇 6.1–6.3 |
| 0.5 | `log` 模块 + HTTP span + `request_id` 贯穿三处 | 错误体里的 `requestId` 依赖它 | 日志篇第 11 节 1–4 |

### 4.1 第一层：让「有身份」成立

| # | 端点 | 依赖 | 为什么在这一层 |
|---|---|---|---|
| 1.1 | `GET /healthz`、`GET /readyz` | 0.1、0.5 | 不依赖鉴权，可以最先跑通部署链路。`readyz` 查迁移版本这条本身需要一个能查版本的连接 |
| 1.2 | `POST /api/v1/bootstrap/owner` | 0.1、0.2、0.4 | **没有它就没有任何账号，其它所有端点都调不通。** 主稿第 9 节：`app_user_one_owner` 唯一索引 + 匿名 bootstrap |
| 1.3 | `POST /api/v1/auth/login` / `logout` / `GET /auth/me` | 1.2、0.4、0.5 | 登录是手工验证其它端点的唯一入口 |
| 1.4 | `POST /api/v1/invites` + `accept` | 1.2 | **没有邀请就没有第二个账号**，而「三档权限」需要三种角色的账号才能测 |
| 1.5 | `GET /api/v1/users`、`role`、`disable`、`enable` | 1.4 | 角色守卫的测试需要能改角色与停用（运行时篇 R3.2 #7） |

### 4.2 第二层：地基（**这一步必须先于所有业务功能**）

| # | 端点 | 依赖 | 为什么 |
|---|---|---|---|
| 2.1 | `GET /api/v1/categories`、`POST /api/v1/categories` | 0.1、1.2 | 器材建档需要 `category_id`，`code` 自动生成需要 `code_prefix` 与 `next_seq` |
| 2.2 | `POST /api/v1/equipment`、`GET /api/v1/equipment`、`GET /api/v1/equipment/{id}`、`PATCH`、`DELETE`、`restore` | 2.1、0.4 | 编号分配（主稿 3.5）、可见性双结构体（主稿 7.7）。**`GET` 是可见性契约测试（实现篇 V1）的唯一载体** |
| 2.3 | **并发互斥测试**（不是端点，是 2.2 的验收） | 2.2 | 主稿附录第 5 步、实现篇第 11 节第 5 步：**「本设计的地基，先于其它功能」**——它证明 `occupancy` 的设计成立。此时 `occupancy` 还没有写路径，所以这条测试直接打 `db::occupancy::insert` |

### 4.3 第三层：写路径（状态机与档期）

| # | 端点 | 依赖 | 为什么在这一层 |
|---|---|---|---|
| 3.1 | `POST /api/v1/checkouts` | 2.2、2.3、0.3 | **`occupancy` 的第一个真实写路径**。主稿 5.1 那个形状在这里成型 |
| 3.2 | `POST /api/v1/checkouts/{id}/items/{itemId}/return` | 3.1 | 主稿附录 4：`services/checkout.rs` + `services/return_item.rs` 是同一对 |
| 3.3 | `GET /api/v1/checkouts`、`GET /checkouts/{id}` | 3.1、3.2 | 读模型：`returnedCount`、`OPEN`/`CLOSED`、部分归还的推导（主稿 5.4） |
| 3.4 | `POST /api/v1/equipment/{id}/maintenance`、`POST /api/v1/maintenance/{id}/close` | 2.2 | 主稿附录 8。**它不碰 `occupancy`**，所以可以与 3.1–3.3 并行 |
| 3.5 | `disable` / `enable` / `sell` / `batch` | 2.2 | 纯状态流转 + 批量原子性（主稿 5.3、C4）。运行时篇 R3.2 #5 的验收在这里 |

### 4.4 第四层：申请（依赖第三层已成型）

| # | 端点 | 依赖 | 为什么在这一层 |
|---|---|---|---|
| 4.1 | `POST /api/v1/requests`、`GET /requests/mine`、`GET /requests/{id}`、`withdraw` | 2.2、0.4、1.5 | 主稿 10.4 的裁剪在这里第一次有真实载荷可裁 |
| 4.2 | `GET /api/v1/requests`（管理员列表）、`approve`、`reject` | 4.1、3.1 | **审批要写 `RESERVATION` 占用**（主稿 3.1 映射表）。**实现篇第 11 节第 10 步的原文：「申请 → 审批 → 交付（`occupancy` 已就位，此处主要是状态流转）」** |
| 4.3 | `POST /api/v1/requests/{id}/deliver` | 4.2、3.1 | 交付要把 `RESERVATION` 行 UPDATE 成 `CHECKOUT` 并写 `source_checkout_id`。**它复用 3.1 的出库落库逻辑** |

> **为什么申请排在出库之后**：主稿附录第 9 步与实现篇第 11 节第 10 步都把申请放在出库之后，理由是「`occupancy` 已就位，此处主要是状态流转」。但**接手的人容易反过来做**（因为申请人侧看起来更简单）。反过来的代价是：审批要写 `occupancy`，而出库还没写过 `occupancy`，于是「+1 天」「`infinity`」这两个最容易错的细节要在审批里第一次实现，**而审批的验证路径比出库长得多**。

### 4.5 第五层：文件与批量

| # | 端点 | 依赖 | 为什么在这一层 |
|---|---|---|---|
| 5.1 | `GET /api/v1/equipment/{id}`（补 `photo` 字段）→ `POST /api/v1/photos` | 2.2、`storage` 模块 | 主稿附录 10。**上传与器材保存是两个请求**（主稿 6.3），所以照片端点可以独立于器材端点实现。**必须先有 `photo` 表 + `pending_upload` 表 + `data_dir` 的同文件系统校验**（运行时篇 R2.1 第 3 条） |
| 5.2 | `import/preview` → `import/commit` → `import/template` | 2.1、2.2 | 主稿附录 11。**两阶段是有序的**：`commit` 复用 `preview` 的校验函数（运行时篇 R5.3「重校验就是『把预览那一步的校验函数再调一次』」），所以 `preview` 必须先落地。**`template` 可以最早做**（它是静态内容），但它的列名必须与 `preview` 的列匹配逻辑同时定，否则会漂移 |
| 5.3 | `GET /api/v1/checkouts/{id}/manifest` | 3.1、3.3 | 主稿附录 12。依赖出库数据已存在。**打印样式与中文字体是它真正的成本**（主稿 13.3） |
| 5.4 | `GET /api/v1/export` | 2.2、`export` 的实现形态 | **依赖主稿 13.2 拍板**（导出什么）。见 §5-I9 |

### 4.6 第六层：汇聚读模型与聚合

| # | 端点 | 依赖 | 为什么在这一层 |
|---|---|---|---|
| 6.1 | `GET /api/v1/activity`（四个 scope） | **所有写端点** | 主稿附录 14 把它放在最后。理由很实在：它的数据来自 `equipment_history`，而**它要能显示的动作种类取决于实现了哪些写路径**。提前做会导致 scope 全是空的，看不出漏了哪种 `action` |
| 6.2 | `GET /api/v1/overview` | 3.1–3.5、4.1–4.3、`jobs` 的逾期扫描 | 总览的每个数字都来自别处。主稿 11 节：逾期扫描「当前只写日志并在总览页查询里体现」——**所以 `overview` 与逾期扫描任务是一对**（谁先都行，但必须一起验证） |
| 6.3 | `GET /api/v1/settings`、`PATCH /api/v1/settings` | 1.2、0.4（`Owner` 档） | 主稿 3.3 的表默认就有行，所以它其实**可以更早做**；放在这里是因为它影响 2.2 与 4.1 的行为（两个开关），**在它落地之前那几个端点必须硬编码「都开」** |
| 6.4 | `GET /api/v1/projects` 全组 | 2.2、3.1 | `project` 表在器材与出库里是可选外键。**`GET /projects` 的「历史出库记录数」需要 3.3 存在**。但 `POST /projects` 可以提前（出库时 `projectName` 是自由文本，`projectId` 可空） |

### 4.7 一条被这个顺序暴露出来的依赖倒置

**`GET /api/v1/settings` 与 `PATCH /api/v1/settings` 影响 `GET /api/v1/equipment` 与 `POST /api/v1/requests` 的行为，但本节把它们排在第六层。** 这不是排序错误，而是一个必须显式处理的工程事实：

- 在 `settings` 端点落地之前，**2.2 与 4.1 必须按 `requester_can_browse = true`、`requester_can_request = true` 的默认行为实现**（与主稿 3.3 的表默认值一致），只是**不读表、硬编码**。
- **不许**为了让顺序更顺而把开关做成配置项（`Config`）——那会让「主理人在界面上改开关」这条永远无法实现，而且运行时篇 R2.1 的 `Config` 是启动时加载一次的（「不是用到时再读」），开关需要立即生效。

---

## 5. 发现的不一致或缺口

**本节只报告，不自行补齐数据结构。** 凡涉及新增列/新增表/新增枚举变体的，一律标为「需要主稿补充」。

分级：**G = 文档之间互相矛盾（必须先统一，否则实现会两种都做一次）**；**I = 缺口（功能没端点，或端点需要主稿里还不存在的字段/表）**。

### 5.1 G 类：现有文档之间的矛盾

| # | 矛盾 | 出处 | 影响 | 建议 |
|---|---|---|---|---|
| **G1** | **`/api/v1` 前缀不统一。** 运行时篇 R6.1 定「路径带版本：`/api/v1/...`」，但主稿 6.3 写 `POST /api/photos`、主稿 7.5 写 `GET /api/equipment?…`、主稿 7.8 写 `GET /api/checkouts/{id}/manifest` —— **三处都缺 `/v1`**。日志篇用的是带版本的形式（`/api/v1/photos`、`/api/v1/equipment/{id}`） | 主稿 6.3 / 7.5 / 7.8 vs 运行时篇 R6.1、日志篇 4.1–4.4 | 若照主稿实现，就会有一批端点不带版本，R6.1 的兼容性规则（R6.2）对它们不成立。**这不是风格问题：R6.2 第 3 条说「删除或重命名字段需要新的 `/v2`，且旧版本至少保留一个发布周期」，一个不在 `/v1` 下的端点无法执行这条** | **本节采用 `/api/v1` 全部统一**（唯一有明确规则支持的一侧）。请主稿把三处补上 `/v1` |
| **G2** | **出库的方法与路径矛盾。** 日志篇 4.1 的示例写 `"msg":"PATCH /api/v1/equipment/checkout 200"`、`"method":"POST"`、`"route":"/api/v1/equipment/checkout"` —— **同一行日志里方法自相矛盾（`PATCH` 与 `POST` 两个值）**，而且路径既不是嵌套式（`/equipment/{id}/checkout`）也不是本节采用的集合式（`/checkouts`）。同时 `PATCH` 与「出库是一次状态流转」冲突：主稿 7.6 的 PATCH 三态是为**描述性字段**定义编辑语意的（「不改 / 清空 / 设置」对出库没有意义） | 日志篇 4.1；主稿 5.1、7.6、7.8 | 这是全套文档里**唯一一处硬矛盾**（同一条记录里两个方法）。照它实现会得到一个 `PATCH` 的出库端点，而 PATCH 的语义（部分更新）会让「重复提交」与「不填 `expectedDate`」这两件事变得含混 | 本节采用 `POST /api/v1/checkouts`。**请主稿与日志篇同时修正**——日志篇 4.1 那个示例的 `msg`/`method`/`route` 三个字段要一起改，否则 Q1（「这个 500 是怎么来的」）的 `route` 会与实际路由模板对不上，日志按端点聚合失效（运行时篇 R1.2 第一条） |
| **G3** | **「出库」与「交付」是一个端点还是两个？** 主稿 5.1 的服务函数叫 `checkout`，主稿 3.1 的映射表把「直接出库」与「办理交付」列为两个业务事件（一个 INSERT、一个 UPDATE）。日志篇给它们**两个不同的 `kind`**（`biz.checkout` 与 `biz.deliver`），但主稿 5.1 的形状只有一个 `checkout` 函数 | 主稿 3.1、5.1；日志篇 6 节 | 如果做成一个端点（`POST /checkouts` 带可选 `requestId`），那 `biz.checkout` 与 `biz.deliver` 的选择就得在服务里按 `request_id` 是否为空分支——**日志的 kind 成了业务分支的副产物**。如果做成两个端点，则 §0.1 的「同一业务事件出现在多个路径下」这条裂缝需要一条规则 | 本节做成**两个端点**（`POST /checkouts` 与 `POST /requests/{id}/deliver`），理由是**被创建/被改变的资源不同**：前者新建出库单，后者把一个已存在的申请从 `APPROVED` 推到 `DELIVERED` 并改写占用行的 `kind`。**请主稿确认**，因为 5.1 的形状暗示它们共用一个服务函数（那仍然成立——两个 handler 可以调同一个 `services::checkout`，只是 `cmd` 的来源不同） |
| **G4** | **出入库记录页读哪个接口？** 主稿 10.1 定「**`equipment_history` 是唯一的叙事主线**」，10.5 又把「出入库记录页」映射到 `GET /api/v1/activity?scope=workspace`。但 §1.5 的 `GET /api/v1/checkouts`（出库单列表）回答的是另一个问题（哪些单还开着、按项目/使用人筛选），**主稿从未定义它** | 主稿 10.1、10.5 | 若出入库记录页按 10.5 读活动流，那「按项目筛选出库单」「看某个使用人的未还清单」这两件事就没有端点。若读 `checkout` 表，则 10.1「界面只读一处」被打破，同一次出库会显示两次（10.1 明确说这个后果不可接受） | **需要主稿补充**：(a) 明确出入库记录页读活动流，且「未还清单/按项目筛选」是**活动流之外的另一个界面**（需要一个 `checkout` 视角的读接口）；或 (b) 明确 `GET /checkouts` 就是这个接口，并从 10.1 的「唯一主线」里给它的读法开一个口子 |
| **G5** | **CSV 导入的路径数量。** 日志篇 4.4 的示例只有一个 `route: "/api/v1/equipment/import"`，但运行时篇 R5 与主稿 5.1 都定的是**两阶段**（预览 + 提交） | 日志篇 4.4 vs 运行时篇 R5.3、主稿 5.1 | 一个 route 无法区分「预览被拒绝」与「提交被拒绝」，而 `import.reject` 的 `data` 里没有阶段字段 | 本节拆成 `/import/preview` 与 `/import/commit`。**请日志篇把 4.4 的 `route` 改成两个**（或给 `data` 加阶段），否则「这份 CSV 为什么导入失败」（Q4）会无法判断是预览期还是提交期 |
| **G6** | **批量的形状未定。** 主稿 5.3 只定义语义（一个事务、不设 savepoint、先收集全部违规），没定义路径与请求体 | 主稿 5.3、C4 | 每个实现者会自己发明一个 | 本节提案 `POST /api/v1/equipment/batch` + `action` 封闭枚举。**请主稿确认**（另一种是每个动作一个批量路径，两者都行，但不能混） |
| **G7** | **`GET /export` 的权限档无法用现有 extractor 表达。** 主稿 13.2 的默认方案写「仅 OWNER / ADMIN 可导出」，但实现篇 6.2 只有 `Admin`（OWNER+ADMIN）和 `Owner`（仅 OWNER），**没有「ADMIN 且非 OWNER」这一档** | 主稿 13.2 vs 实现篇 6.2 | 若产品真的要「ADMIN 能导、OWNER 不能导」这种反直觉的档位，现有 extractor 需要新增 | **需要主稿确认**：本节按字面取「OWNER 或 ADMIN」= `Admin` 档。请确认这不是「仅 OWNER」的笔误 |
| **G8** | **「`UPDATE` 历史必须有 `detail.changed`」这条规则只写在正文里，DDL 允许它为空。** 主稿 3.3 的 DDL 是 `detail jsonb`（可空），而 5.5 要求 `UPDATE` 动作的 `detail` 里带 `changed` 数组、运行时篇 R4.2 又给这个数组加了两条工程约束（一次提交一条、只记真正变了的字段） | 主稿 3.3 vs 5.5、14 节差异 #8；运行时篇 R4.2 | 一条 `action='UPDATE'` 且 `detail IS NULL` 或 `detail.changed` 缺失的历史行，**在写路径 bug 或手工改库之后会静默存在**，而界面（主稿 10.2 的渲染示例）会渲染出一个空的「修改了」条目——`changed` 数组是这个机制唯一的产出（运行时篇 R4.3 末段原话） | **需要主稿补充**：明确这条不变量由谁保证。(a) 服务层保证（写入前构造 `detail`，不给出空数组的入口）——**推荐**，主稿 3.2 的双层结构里这一层属于「给人话」的应用层；(b) 加 CHECK：`CHECK (action <> 'UPDATE' OR (detail ? 'changed' AND jsonb_array_length(detail->'changed') > 0))` —— 数据库保证，但 JSONB 的 CHECK 会让迁移与调试变复杂；(c) 在活动流读路径上过滤掉 `changed` 为空的行（**不推荐**：那是让读取端弥补写入端的缺陷） |

> **一条被排除的怀疑（记录在此以免下一个人重查）：** 我一度怀疑主稿 3.3 的 `equipment_history.action` CHECK 列表漏了 `'UPDATE'`，因为 5.5 与日志篇 `biz.update` 都依赖它。**逐字核对后确认它没有漏**——主稿 3.3 的列表是 `('CREATE','UPDATE','CHECKOUT','RETURN','MAINTENANCE_SEND','MAINTENANCE_CLOSE','DISABLE','ENABLE','SOLD','ARCHIVE','RESTORE')`，`'UPDATE'` 在第二位。真正的问题是上面 G8 写的 `detail` 可空，不是枚举缺失。

### 5.2 I 类：缺口

| # | 缺口 | 证据 | 影响 | 建议（2–3 个选项，不猜） |
|---|---|---|---|---|
| **I1** | **「导出备份」到底导出什么，以及「模板下载」的路径都没有。** 交接稿把「导出备份」列为管理员能力（主稿 13.2 引用），日志篇 6 节的 `export` 事件有 `kind`（数据/模板）两种取值——**说明「模板」也是导出的一种**，但两处都没有路径 | 主稿 13.2；日志篇 6 节 `export` | 无端点则功能缺失 | 见 §5.3 的 I9（导出范围）。**模板**部分本节提案 `GET /api/v1/equipment/import/template`（§1.6），**依据是日志篇的 `kind` 只有两个值，所以「模板」必然是一个端点而不是纯前端静态文件**——但更强的一种可能是：模板由前端静态托管（它不需要数据）。**需要主稿确认**：(a) 服务端生成（可保证列名与解析逻辑同源，代价是多一个端点）；(b) 前端静态文件（零成本，代价是列名可能与解析逻辑漂移）；(c) 服务端生成并带 BOM（因为运行时篇 R5.1 说 Excel 会按 GBK 读，而**模板里全是中文列名**，不带 BOM 的 UTF-8 模板会被 Excel 打开成乱码） |
| **I2** | **`requester_can_browse = false` 时，申请人能看到/能做什么，主稿只定义了一半。** 主稿 3.3 有这两个开关并注释「交接稿第 3 节」。`requester_can_request` 有对应的 `DomainError::RequestingDisabled`；`requester_can_browse` **没有任何错误变体，也没有说它管哪些端点** | 主稿 3.3、第 8 节的枚举 | 实现者可能：(a) 只在列表上生效而详情仍可直连（越权）；(b) 连 `GET /requests/mine` 也挡掉（用户看不到自己的申请，这是错的）；(c) 连申请提交也挡掉 | **需要主稿补充**：(a) `requester_can_browse=false` 只关 `GET /api/v1/equipment` 与 `GET /api/v1/equipment/{id}`（`categories` 也关？），返 **403 `FORBIDDEN`**（复用现有码）；或 (b) 新增 `DomainError::BrowsingDisabled` + `BROWSING_DISABLED`（与 `RequestingDisabled` 对称，**本节倾向这个，因为对称性让前端能对两个开关用同一套处理**）；或 (c) 该开关只影响界面可见性，不影响 API（**不推荐**——那是 UI 事实当安全边界，主稿第 9 节明确反对这种做法） |
| **I3** | **`includeUpdates` 的作用域比主稿描述的大。** 主稿 10.5 只在 `scope=workspace` 的例子里写了 `includeUpdates=false`，但正文说「`scope=workspace` 默认**不包含 `UPDATE`**」——没说其它三个 scope 的默认值 | 主稿 10.5 | 实现者会让 `scope=equipment` 默认含 `UPDATE`（因为器材详情弹窗正是要看字段修改的地方，主稿 10.2 的渲染示例就是 `UPDATE`），也可能让四个 scope 一致默认不含 | **需要主稿补充**：明确四个 scope 各自的 `includeUpdates` 默认值。**本节按「`workspace` 默认 false，其余三个默认 true」写**（理由：器材/项目/人的历史里字段修改正是要查的东西，而全库流水需要它来防刷屏——这正是 10.5 末句给的理由）。**请主稿确认** |
| **I4** | **三个账号相关的端点没有定义，而运行时篇把它们列为「一定会被做、没有文档一定会做错」的操作。** 具体是：(1) 改密码（用户自助）；(2) 重置密码（管理员代劳）；(3) **workspace 的创建**——主稿第 9 节的 bootstrap 说「当 workspace 无 OWNER 时允许匿名创建」，但 `workspace` 行本身从哪来没有说 | 运行时篇 R7.4 第 2 条；主稿第 9 节、3.3 的 `workspace` 表 | 「重置密码」在运行时篇 R7.4 里是必须写进运维手册的操作，而**没有端点意味着它只能靠改数据库**——运行时篇 R7.4 第 3 条的警告（「否则有人会直接改数据库」）同样适用于它 | **需要主稿补充**。本节给三个选项：<br>(a) `POST /api/v1/users/me/password`（自助，需旧密码）+ `POST /api/v1/users/{id}/password`（ADMIN 重置，返回一次性新密码或直接置空要求下次改）—— 两个端点，职责清楚<br>(b) 一个端点 + `?self=` 参数 —— 少一个端点但把权限判断塞进了参数<br>(c) 不做自助改密，只做 ADMIN 重置 —— 最省事，但用户改密码要找管理员<br>**并且**：`POST /api/v1/bootstrap/owner` 是否顺带创建 workspace？本节在 §1.2 按「是」写，**需要主稿确认** |
| **I5** | **`Idempotency-Key` 的落地没有任何定义。** 运行时篇 R4.4 只说「对创建型接口接受一个可选的 `Idempotency-Key` 头，同 key 的重复请求返回第一次的结果」，**没有说这个 key 存在哪、存多久、同 key 不同 body 怎么办** | 运行时篇 R4.4 | 没有定义就无法实现（每个实现者会自己发明一张表）。**更危险的一侧**：如果只在前端禁用按钮（R4.4 说「第一种靠前端禁用按钮」），弱网重试仍然会开出库两单 | **需要主稿补充**（这是本清单里最硬的一个实现阻塞）。三个选项：<br>(a) **一张 `idempotency_key` 表**（`key`, `endpoint`, `request_hash`, `response_body jsonb`, `created_at`，唯一约束 `(key, endpoint)`，TTL 随会话清理任务淘汰）—— 最完整，能返回**第一次的完整响应**（R4.4 的字面要求）<br>(b) **不存响应，只存「这个 key 处理过了」**（返回 409 `DUPLICATE_REQUEST`）—— 省一张表与响应体存储，但违反 R4.4「返回第一次的结果」<br>(c) **只做出库的天然幂等**（靠 `occupancy` 排他约束拦第二次，实现篇第 4 节第 2 点已经说了这条），导入靠「重校验 + 去重」—— **零新结构**，但照片上传与导入的「重复提交」会变成错误而不是静默成功 |
| **I6** | **两处需要主稿补充的列/索引：** (1) `invite` 表**没有 `revoked_at`**，而 §1.3 需要一个「撤销邀请」端点——撤销到底改哪个列（`consumed_at`？新增 `revoked_at`？）；(2) `equipment_history` 的两个索引**都不含 `id`**（`(equipment_id, created_at DESC)` 与 `(workspace_id, created_at DESC)`），而活动流的游标键是 `(created_at, id)` | 主稿 3.3 的 `invite` 与 `equipment_history` DDL；主稿 7.5 与 10.5 的游标 | (1) 撤销复用 `consumed_at` 会让「被撤销」与「已使用」不可区分（`consumed_by` 为空但 `consumed_at` 非空——这个组合能被推断，但是隐式的）；(2) 同毫秒多条历史（一次批量出库写 3 条）会需要按 `id` 打破平局，索引里没有 → 退化成 filter | **需要主稿补充**：(1) 给 `invite` 加 `revoked_at timestamptz`，或明确「撤销 = 置 `consumed_at` 且 `consumed_by IS NULL`」；(2) 把两个索引改成 `(equipment_id, created_at DESC, id DESC)` 与 `(workspace_id, created_at DESC, id DESC)`，或明确接受退化成 filter |
| **I7** | **`POST /api/v1/photos` 的 multipart 字段名没有定义。** 主稿 6.3 只写了 `(multipart/form-data)` 与流程，没有写字段叫什么 | 主稿 6.3、7.9 | 前端与后端会各猜一个 | **需要主稿补充**。`file` 是唯一合理的选择（`image` / `photo` 也都可能）。另外要定：**是否允许一次请求传多张**（主稿 6.3 的流程图是单文件：一次收流、算一个 sha256、写一个 `.thumb.jpg`）→ 本节按**单文件**写，与流程图一致 |
| **I8** | **CSV 导入的重复判定未定，直接影响 `ViewImportPreview.willSkip` 与 `commit` 的语义。** 主稿 13.9 自己列了这个问题并给了建议（按 `(workspace_id, code)` 去重，预览里标出「已存在，将跳过」） | 主稿 13.9 | 端点形状依赖它：若「每次全新建」，则没有 `willSkip` 这个字段；若按编号去重，则要区分「有编号已存在」与「无编号每次新建」 | **需要主稿拍板**（13.9 已给建议，本节采用建议即可，但它仍是未决项）：(a) 按 `(workspace_id, code)` 去重 + 跳过 + `willSkip` 计数（**13.9 的建议**）；(b) 按编号去重但**覆盖**而不是跳过（需要版本/审计考虑，风险更高）；(c) 每次全新建（最简单，但同一份文件导两次就有两份器材） |
| **I9** | **「数据与设置」里的「导出」范围未定（主稿 13.2 是阻塞项）。** 主稿 13.2 定了权限（仅 OWNER/ADMIN）、定了「原始数据（含价格/序列号）」、定了「文件头写入导出人、时间、范围」——**但「范围」本身是什么没说** | 主稿 13.2、§1.6 | 端点无法实现（导出的文件名、`Content-Type`、体积、是否需要异步都取决于它） | **需要产品拍板**。三个选项：<br>(a) **全库原始导出**（一个包含 `equipment` 全列的 CSV/XLSX，可能加 `category` 名）—— 与「备份的用途是灾难恢复」这句一致（13.2 的理由），但**它不是完整备份**（不含 `photo`、`checkout`、`request`、`history`），而主稿 12.2 已经用 `pg_dump` 做数据库备份了 → 这个导出更像是「给管理员在 Excel 里看的全景表」<br>(b) **按器材库视图导出**（CSV 模板列的超集 + 状态），最贴近交接稿第 6 节那句「导出备份」在界面上的位置<br>(c) **完整业务数据包**（多表、ZIP）—— 与 `pg_dump` 重复，且会引出「脱敏导出」这个 13.2 末句提到的第二个接口<br>**并且**：是否是「脱敏导出」（日志篇 `export` 的 `masked` 布尔说明设计里预期有它）？这决定是一个端点还是两个 |
| **I10** | **「撤回申请」的业务规则主稿没写。** 只定义了状态 `WITHDRAWN` 与「撤回申请 → DELETE 相关 occupancy 行」（主稿 3.3、3.1） | 主稿 3.1、3.3 | 实现者会自己定「审批通过后还能不能撤回」（会影响 `occupancy` 的 `RESERVATION` 行），而这是一个业务决策 | **需要主稿补充**：(a) **只有 `PENDING` 能撤回**（最保守：一旦管理员投入了审批，就要求走拒绝/取消流程）；(b) `PENDING` 与 `APPROVED`/`PARTIAL` 都能撤回（占用行一并删除，档期立即释放——**这会让管理员的审批白做，但用户确实该有这个权利**）；(c) 交付后不能撤回（`DELIVERED` 之后是归还流程）——**本节倾向 (a) 或 (b) 且都含 (c)** |
| **I11** | **「整单拒绝」是独立端点还是 `approve` 的全 `false` 特例？** 主稿 3.1 的映射表只描述了「批准时取消勾选某件（= 拒绝）」，没有描述「一件都不批」 | 主稿 3.1、3.3 | 若复用 `approve`，则 `status` 在「全 `false`」时取 `PARTIAL` 还是 `REJECTED` 有歧义（两个值都在 CHECK 里）；日志篇 `biz.approve` 的 `result` 有 `REJECTED` 这个取值，**说明「整体拒绝」是被预期的结果** | **需要主稿补充**：(a) 独立端点 `POST /requests/{id}/reject`（本节按此写，`reviewNote` 必填）；(b) `approve` + 全部 `false` → `status = REJECTED`（少一个端点，但 `APPROVE` 这个动词名不副实——它可能什么都不批）。**两个都要明确「`PARTIAL` 只在『至少批一件且至少拒一件』时出现」**，否则 `PARTIAL` 会变成一个「什么都可能」的状态 |
| **I12** | **`ViewRequestRequester` 是否含 `reviewedAt` / `reviewNote`。** 主稿 10.4 只说申请人的界面里需要「他的申请单里每件器材当前的状态」；主稿 7.7 把 `createdAt`/`updatedAt` 划给管理员视图，但 `review_note`/`reviewed_at` **不在 7.7 的字段清单里**（那是 `equipment` 的清单） | 主稿 7.7、10.4、3.3 | 申请人看不到「为什么被拒」——这是**功能**问题，不只是权限问题：交接稿的申请流程里，被拒的申请人应当知道原因 | **需要主稿补充**：本节倾向**给 `reviewNote`（去掉 `reviewedBy` 的内部身份也可以保留名字）但不下发 `reviewedAt`**（时间按 7.7 的纪律划给管理员）。请确认——**这是一个可能影响产品行为的决定**（申请人能不能看到拒绝理由） |
| **I13** | **「访问别人的资源」返 404 还是 403。** 主稿 7.3 的 403 行原文是「角色不允许（申请人访问全库历史）」——那是入口级拒绝；而 `GET /api/v1/requests/{id}` 上申请人访问别人的单子是**存在性**问题 | 主稿 7.3 | 返 403 会泄漏「这个 ID 存在」（可枚举）；返 404 会让「我没有权限」与「它不存在」不可区分（对申请人无所谓，因为他本来就没有权限） | **需要主稿补充**。本节按 **404**（防枚举），符合主稿 7.3「不存在或已归档 → 404」的字面。**请确认这条推断**——它与「403 用于角色不允许」不冲突，因为这里不是角色不允许，是「这个对象不属于你」 |
| **I14** | **`GET /api/v1/projects` 给不给申请人？** 主稿没有说项目的可见性。但 `project.client`（客户名）与 `project.note`（备注）按 7.7 的分工属于内部信息，而申请人提交申请时用的是 `project_name` 自由文本（主稿 3.3），**不需要看项目库** | 主稿 3.3、7.7 | 若给申请人，则项目列表会泄漏客户关系；若不给，则「申请人如何知道该填哪个项目名」需要产品回答（主稿 3.3 说 `project_name` 是自由文本，**暗示申请人本来就不从项目库里选**） | **需要主稿补充**。本节按 **ADMIN 档**写。另一种可能：给申请人一个**精简的项目选择列表**（只有 `id` + `name`，取自 `status='ACTIVE'`），用于「选项目」而不是「看项目」——那它会是一个独立端点（`GET /api/v1/projects/lookup`）。**请主稿确认** |
| **I15** | **「项目卡片显示历史出库记录数」这个数字从哪来。** 主稿 10.5 把它升级成「可点开的活动流」，但没说数字本身由谁算 | 主稿 10.5；交接稿第 6 节 | 两种实现会让**同一个界面上的两个数不一致**（卡片上的数 vs 点开后能滚动的条数），因为活动流是游标分页的 | **需要主稿补充**：(a) 卡片上的数 = `GET /projects` 里的一个 `checkoutCount` 字段（一次查询，快）；(b) 卡片不显示数字，只显示「查看记录」（少一个字段，少一处不一致）；(c) 数字 = 活动流第一页的条数（**会在有更多记录时显示错误的数**，明确不推荐）。**本节倾向 (a) 并在 `ViewProject` 里加 `checkoutCount`**，但这需要主稿确认（它会引入一个聚合列，主稿 3.3 的 `project` 表没有它） |
| **I16** | **项目没有状态机表，`DELETE /projects/{id}` 的语义也未定。** 器材有五态单向流转表（实现篇第 4 节 `ALLOWED`）；项目的 `status` CHECK 有 `ACTIVE`/`COMPLETED`/`ARCHIVED`，但**没有任何文档给合法流转**。另外 `project.archived_at` 列不存在（与 `equipment.archived_at` 不对称），而 `status` 里已经有 `ARCHIVED` | 主稿 3.3；实现篇第 4 节 | 实现者会给项目写一套自己的流转规则（可能允许 `ARCHIVED→ACTIVE`，也可能不允许）；`DELETE` 会因外键失败变成 500（`request.project_id`、`checkout.project_id` 都无 `ON DELETE`） | **需要主稿补充**：(a) 项目流转 = `ACTIVE↔COMPLETED`、两者可 `→ARCHIVED`、`ARCHIVED` 是终点；`DELETE /projects/{id}` **不做物理删除**，改成 `POST /projects/{id}/status {status:"ARCHIVED"}`（与器材的归档分工一致）；或 (b) `DELETE` 只在无引用时允许（返 409），有引用时要求先 `ARCHIVED`；或 (c) 项目也加 `archived_at` 列（与器材对称，但会引入与 `status='ARCHIVED'` 重复的两种「消失」语义——**这正是主稿 3.4 为器材明确避免的事，所以不推荐**） |
| **I17** | **删除被引用的分类的失败没有对应的领域错误。** `equipment.category_id` 引用 `category(id)` 且无 `ON DELETE`，所以删除有器材的分类会外键失败（SQLSTATE `23503`） | 主稿 3.3；实现篇 5.1 的 `From<sqlx::Error>` 只处理 `23P01` 与 `23505` | **会变成 500**（`23503` 落到 `_ => AppError::Sqlx(e)`）。用户看到「出错了」，而这是一个可以给人话的业务拒绝 | **需要主稿补充**：(a) 新增 `DomainError::CategoryInUse { count }` + `CATEGORY_IN_USE` → 409，并在 `From<sqlx::Error>` 里加 `23503` 分支；(b) 在服务层预检查（`SELECT count(*) FROM equipment WHERE category_id = $1`）后返回同一个错误（**与主稿 3.2 的「预检查给人话、约束保正确性」双层结构一致，推荐**）；(c) 明确禁止删除有器材的分类，只允许改名 |
| **I18** | **总览页的字段完全未定义。** 主稿 11 节与运行时篇 R1.8 都只说「逾期在总览页查询里体现」，**没有一处定义总览页有哪些数字** | 主稿 11、运行时篇 R1.8 | 端点无法实现 | **需要主稿补充**。本节在 §2.9 给了候选形状（`overdueCount`、`statusCounts`、`openCheckoutCount`、`pendingRequestCount`、`overdueItems[]`、`recentActivity[]`），但**明确不作为结论**。三个选项：<br>(a) **一个聚合端点** `GET /api/v1/overview`（本节提案）—— 一次请求拿全，代价是总览页的任何改动都要改后端<br>(b) **复用已有列表端点**（`GET /equipment?status=CHECKED_OUT&overdue=true` + `GET /requests?status=PENDING`）—— 零新端点，代价是总览页要发 N 个请求，而 `isOverdue` 的筛选参数**主稿 7.5 的筛选器列表里没有**<br>(c) **只做逾期一件事**（`GET /api/v1/overview` 只返回 `overdueItems`）—— 最小，因为主稿 11 与运行时篇 R1.8 只点名了逾期<br>**注意 (b) 需要给 `GET /api/v1/equipment` 加一个 `overdue` 筛选器**，那是对主稿 7.5「四个筛选器（分类、状态、品牌、排序）」的扩展 |
| **I19** | **`GET /api/v1/settings` 给不给 REQUESTER。** 主稿 3.3 说这两个开关是「主理人的两个开关」 | 主稿 3.3 | 若不给，申请人只能靠「提交时被告知 `REQUESTING_DISABLED`」来学（一次失败的提交）；若给，前端可以提前禁用提交按钮 | **需要主稿补充**：(a) 只读开放给 `REQUESTER`（返回两个布尔，申请人据此禁用按钮）；(b) 只有 OWNER/ADMIN（本节按此写），申请人靠 403 学。**本节倾向 (a)**——因为 `REQUESTING_DISABLED` 是一次用户可见的失败，而预先知道可以避免它。请确认 |
| **I20** | **`ViewCheckout` 与 `ViewManifest` 的字段边界没有定义。** 主稿 7.8 只说清单含序列号、走专用接口、角色守卫收在一个 handler 里 | 主稿 7.8 | 实现者可能把序列号放进 `ViewCheckout`（因为它「反正只有 ADMIN 能看」），而 7.8 的**设计意图是清单专用**；也可能把采购价放进清单 | **需要主稿补充**：明确三个结构的字段边界——(a) `ViewCheckout`（不含序列号、不含价格）；(b) `ViewManifest`（含序列号，**不含价格**）；(c) `ViewCheckoutDetail`（含器材的 `EquipmentAdmin`，即含序列号与价格，因为它是管理员界面）。本节按这个写。**请主稿确认「清单不含价格」这一条**（主稿 7.8 只点名了序列号） |
| **I21** | **`occupancy.during` 的 `+1 天` 如何在接口上表达。** 主稿 3.1 的映射表：批准时 INSERT `[使用开始, 预计归还 + 1天)`；半开区间。所以「用到 10-05」在库里是 `[10-01, 10-06)` | 主稿 3.1 | 接口若直接把 `during` 的右端给前端，前端会显示「用到 10-06」（**差一天**）；若接口给 `useEnd`（10-05），则接口层要做一次 `-1 天`，而这一步很容易与 `infinity` 冲突（**`infinity - 1 day` 仍是 `infinity`**，但代码若不特判就可能 panic 或溢出） | **需要主稿补充**：明确 `GET /requests/{id}` 与 `GET /checkouts/{id}` 是否下发 `occupancy.during` 原值。**本节在 §2.6 按「给原始区间」写**（可核验、无信息损失），但**这会让前端必须知道 `+1 天` 语义**，而那是一个跨层的隐式约定。另一种做法是接口层只给人读的 `useStart`/`useEnd`/`expectedDate`，`during` 完全不下发（**更干净，前端不会算错，代价是接口无法自证占用区间**）。请主稿选一个 |
| **I22** | **「两个结构体」的纪律只对器材写了，对其它按角色分叉的 DTO 没写。** 见 §2.10 的对照表 | 主稿 7.7、实现篇 2 节 | 见 §2.10：`flatten` 的重复键坑会在 `ViewRequestRequester`/`ViewRequestAdmin` 上原样重现 | **需要主稿补充**：把 7.7 的纪律（「被禁字段一律不出现，不同角色用不同类型，不用 `skip_serializing_if` 做权限」）从「器材」推广到所有 DTO，并明确 `ViewRequestRequester` 是独立结构体而不是「Admin 减字段」 |
| **I23** | **500 与 503 的分界、以及 429 的归属，主稿第 8 节没有写。** 主稿 7.3 的表最后一行是「限流 / 事务重试耗尽 → 429 / 503」，但实现篇 5.2 的 `match` 里没有 429/503，运行时篇 R7.1 单独定了「数据库不可用 → 503（不是 500）」 | 主稿 7.3；实现篇 5.2；运行时篇 R7.1 | 实现者会让数据库不可用变成 500（前端无法区分「稍后重试」与「出错了」——这正是 R7.1 要避免的） | **需要主稿补充**：把运行时篇 R7.1 那一条回写进主稿第 8 节的表，并明确 429 是否真的要做（**主稿没有任何限流设计**，所以 429 目前是一个没有实现的码）。(a) 明确不做限流，删掉 7.3 表里的「限流」二字；(b) 只保留「事务重试耗尽 → 503」；(c) 引入限流中间件（**本节不推荐**——主稿 D1 起的整套设计都在避免为不存在的问题引入机制） |
| **I24** | **四个 `DomainError` 变体在主稿第 8 节没有 HTTP 码，实现篇 5.2 的 `match` 里也没有分支。** 它们是：`BatchRejected`、`CheckedOutNotArchivable`、`DuplicateCode`、`RequestingDisabled`。**主稿第 8 节的枚举定义了它们，但 7.3 的「情况 → 码」表与实现篇的 `match` 都漏了** | 主稿第 8 节 vs 7.3；实现篇 5.2 | 这四个错误**会落进 `match` 的兜底分支或编译失败**。实现篇 5.2 的 `match` 是穷尽的（没有 `_` 兜底），所以**加了这四个变体之后那个 `match` 根本编译不过**——这其实是好事（编译器会拦住），但它说明**实现篇 5.2 是照一个更早的枚举写的** | **需要主稿补充**一张完整的「`DomainError` → (HTTP, code)」映射表，覆盖全部变体。本节 §3.2 给了建议值（`BatchRejected` → 422 `BATCH_REJECTED`；`CheckedOutNotArchivable` → 409 `CHECKED_OUT_NOT_ARCHIVABLE`；`DuplicateCode` → 409 `DUPLICATE_CODE`；`RequestingDisabled` → 403 `REQUESTING_DISABLED`），**但这些是我按语义推断的，需要主稿确认** |
| **I25** | **`23505` 的翻译只覆盖两个约束名，而 `23505` 有七个来源，其中五个没有对应的 `DomainError`。** 实现篇 5.1 只处理 `equipment_code_uniq` 与 `app_user_one_owner`。实际会触发 `23505` 的唯一约束有：`equipment_code_uniq`、`category(workspace_id,name)`、`category(workspace_id,code_prefix)`、`app_user(workspace_id,username)`、`photo(workspace_id,sha256)`、`invite.token_hash`、**`maintenance_one_open`（部分唯一索引）** | 主稿 3.3 的 DDL；实现篇 5.1 | 分类重名、用户名已存在、重复送修（该件已有未关闭维修）、照片 sha 冲突**全都变成 500**。其中「重复送修」是一个**正常用户操作**（用户不知道已经送修过），会得到 500 而不是人话 | **需要主稿补充**：为每个唯一约束定义领域错误与文案，并在 `From<sqlx::Error>` 里按约束名逐个匹配。**注意 `photo(workspace_id, sha256)` 是一个例外**：它**不该被当成错误**——主稿 6.2 说「手机重传同一张 → 路径相同，写之前发现存在就跳过。幂等」，所以它应该在服务层被处理成「返回已有引用」而不是让约束触发 |
| **I26** | **`DomainError::Conflict` 不存在，但主稿第 8 节与实现篇 5.1 的代码都在用它**；**「不存在或已归档 → 404」没有任何错误变体承载** | 主稿第 8 节的枚举 vs 同节的 `From<sqlx::Error>` 示例；实现篇 5.1 | (1) 照实现篇 5.1 写会编译失败（或需要自己加一个变体——**那就意味着实现者在发明领域错误**）；(2) 所有带 `{id}` 的端点都需要 404，而**没有任何变体表达它** | **需要主稿补充**：(a) 把 `Conflict` 加进枚举（但它与 `Reserved` 的关系要定：`23P01` 触发时究竟返回「已被 X 占用」还是「排他约束被触发」？**实现篇 5.1 翻译成 `Conflict` 之后拿不到项目名，所以它给不出主稿 7.3 要求的人话**——见下面的注）；(b) 新增 `DomainError::NotFound { kind, id }` 或让 handler 直接构造 404（脱离 `DomainError`）；(c) 明确 404 不经 `DomainError`，由 `AppError` 层单独表达 |
| **I27** | **`POST /bootstrap/owner` 在已有 OWNER 时返回什么码。** 主稿第 9 节说这个接口「只在没有任何 OWNER 时可用，用过即永久失效」，但没说失效时返回什么 | 主稿第 9 节 | 实现者会复用 `OwnerImmutable`（本节的做法）或返 404（隐藏入口的存在），**两者的前端行为完全不同** | **需要主稿补充**：(a) 409 `OWNER_IMMUTABLE`（本节按此写——**但语义不贴切**，这个错误的中文是「主理人的权限不能被修改」）；(b) 404（入口不存在，防枚举）；(c) 新增 `BootstrapClosed`。**本节倾向 (b)**，因为它连「这里有一个 bootstrap 入口」都不泄漏。请主稿定 |
| **I28** | **邀请失效（已消费 / 已过期）没有对应的码。** 主稿第 9 节定义了 `expires_at` 与一次性 `consumed_at`，第 8 节没有对应错误 | 主稿第 9 节、第 8 节 | 会变成 500 或 404（不可区分） | **需要主稿补充**：(a) 统一 404（不区分「不存在」「已过期」「已使用」——防枚举，**本节倾向**）；(b) 404（不存在）+ **410 Gone**（已失效，语义最准，但引入一个新码）；(c) 409 + `INVITE_EXPIRED`/`INVITE_CONSUMED`（信息最全，但对未登录的接口泄漏状态） |
| **I29** | **`return` 端点没有定义「item 不属于该 checkout」这个校验。** 路径是 `/checkouts/{id}/items/{itemId}/return`，两个 ID 都要校验 | 主稿 5.4 | 若不校验，`itemId` 可以属于另一个出库单（**越权归还**：把 A 单的器材在 B 单下归还） | **需要主稿补充**（**这是安全项，不是体验项**）：明确 `WHERE checkout_id = $1 AND id = $2`，不匹配返 404。同理 `POST /requests/{id}/deliver` 的 `equipmentIds` 必须属于该申请 |
| **I30** | **413 这个码主稿没有，但它是预期路径。** 主稿 6.3 定 8 MiB 上限、运行时篇 R5.4 定 5 MiB CSV 上限并要求「在读流的时候生效」 | 主稿 6.3；运行时篇 R5.4；主稿 7.3 的表 | 超限会变成 400（若实现者用 `VALIDATION`）或 axum 的默认 413（若用 `RequestBodyLimitLayer`，见实现篇 6.3 的 `.layer(RequestBodyLimitLayer::new(8 * 1024 * 1024))`）——**两者不可区分** | **需要主稿补充**：明确超限返 **413** 且**响应体仍是错误信封**（`code: "PAYLOAD_TOO_LARGE"`）。**注意实现篇 6.3 用的是 `RequestBodyLimitLayer`，它返回的是 axum 自己的响应，不是 `AppError` 的信封**——这会让错误信封在这一个码上破例，前端必须特判。**这一条要写进主稿**，否则「所有非 2xx 都是同一个形状」（主稿 7.3）会有一个静默的例外 |
| **I31** | **「提交时与预览结果不一致」的码没有定义。** 运行时篇 R5.3 建议重校验，「若与预览结果不一致，拒绝并让用户重新预览」——**但没有说用什么码** | 运行时篇 R5.3 | 实现者会复用 `BATCH_REJECTED`（但那一批可能没有问题行，只是数据变了）或 409（与档期冲突撞码） | **需要主稿补充**：(a) 409 + `IMPORT_STALE`（语义准，但与 `RESERVED` 共用 409，前端要靠 `code` 区分——**这正是主稿 7.3 分 409/422 的理由所在，所以复用 409 是可接受的**）；(b) 422 + `IMPORT_STALE`（把它归到「业务规则拒绝」而不是「当前状态冲突」）；(c) 复用 `BATCH_REJECTED`（**不推荐**：违规件为空会让人话变成「以下器材不能删除（出库中）：」后面空着） |
| **I32** | **`DELETE /projects/{id}` 与 `DELETE /categories/{id}` 的外键失败没有码。** 与 I17 同因（`23503`），但影响两个端点 | 主稿 3.3；实现篇 5.1 | 见 I17 | 与 I17 合并处理：新增一个 `Referenced`/`InUse` 类错误，或走服务层预检查。**需要主稿补充** |
| **I33** | **没有任何端点能读维修记录（`maintenance`）。** 日志篇有 `biz.maintenance`，主稿 3.3 有 `maintenance` 表（含 `fault`/`vendor`/`cost_cents`/`outcome`），主稿 10.1 的映射表写「送修 / 维修完成 → 点开后显示：跳 `maintenance`：故障说明、维修单位、费用」——**但没有任何端点能取它**。活动流的 `ref`（主稿 10.5）会给出 `{"kind":"maintenance","id":"55"}`，而**这个 ref 点不开** | 主稿 3.3、10.1、10.5；日志篇 6 节 `biz.maintenance` | 维修的写路径有两个端点、读路径一个都没有。**这是通过「日志事件 → 端点」反查发现的，从功能清单看不出来**（交接稿的功能清单里「送修/维修完成」是作为状态流转列的，没有列「查维修记录」） | **需要主稿补充**：`GET /api/v1/maintenance/{id}`（ADMIN，本节已在 §1.6 收录）。**并且要系统性地检查一遍**：活动流 `ref` 的每一种 `kind`（`checkout` / `maintenance` / …）是否都有对应的可打开端点——`checkout` 有（§1.5），`maintenance` 没有。这个检查要成为加 `ref` 新种类的**前置条件**，否则每加一种就多一个点不开的历史条目 |
| **I34** | **活动流在「作者已停用」时怎么显示标注。** 主稿 10.6 明确要显示「(已停用)」 | 主稿 10.6 | `ViewActivityItem.actor`（主稿 10.5）只有 `{id, name}`，**没有地方承载「已停用」这个状态** | **需要主稿补充**：给 `actor` 加 `disabled: bool`（**推荐**），或明确由前端另发请求查账号列表交叉比对（**不推荐**：活动流每页 20 条会引出 20 次查询或一次全量拉取）。**注意不要在服务端把「(已停用)」拼进 `name`**——那是展示层的事，拼进去会让按人名搜索/排序失效，而且与主稿 7.2「线上跑的是稳定代码，不是中文」冲突 |

### 5.3 I9 展开：`GET /api/v1/export` 的三个选项（主稿 13.2 是阻塞项）

主稿 13.2 明确把「导出备份是否按角色裁剪」列为**阻塞项**，并且它的默认方案里有半句是不完整的：默认方案说了「导出**原始数据**（含价格/序列号）」，但**「原始数据」的粒度没说**。这不是「按角色裁剪吗」的问题（那一问 13.2 已经答了：不裁剪），而是**「导出哪些表」**的问题。

| 选项 | 内容 | 与既有设计的关系 | 代价 |
|---|---|---|---|
| **(a) 器材库全字段 CSV** | `equipment` 全列 + 分类名。一个平表，Excel 能直接打开 | 与交接稿第 6 节「导出备份」在界面上的位置一致（它长在器材库页面）。**与主稿 12.2 的 `pg_dump` 备份不重复**（那个是灾备，这个是给人看的） | 它**不是备份**——不含 `checkout`/`request`/`history`/照片。若产品的意图是「灾备的第二条腿」，这个选项不满足 |
| **(b) 全库业务数据包（多表 ZIP）** | `equipment` + `category` + `project` + `request` + `request_item` + `checkout` + `checkout_item` + `maintenance` + `equipment_history` + `photo` 各一个 CSV | 与 13.2「备份的用途是灾难恢复」最贴 | **与 `pg_dump -Fc` 重复**（主稿 12.2 已经每日做它）。而且「恢复时先文件后库」（主稿 12.2）这条流程**不适用于一个 ZIP**，会引出「这个 ZIP 能不能用来恢复」的歧义。体积也可能不小（`import` 上限 1000 行，但 `history` 会长） |
| **(c) 一个可参数化的导出端点** | `GET /api/v1/export?scope=equipment\|all&format=csv` | 最灵活 | **违反主稿 R6.2 第 2 条的精神**（改语义要加新字段）不违反，但**它把「导出什么」这个产品决策变成了一个运行时参数**，而 13.2 说这是一个需要产品拍板的问题——参数化等于没拍板 |

**本节不给结论。** 但给一条判断依据：**主稿 12.2 已经把灾备交给 `pg_dump` + 照片冷同步了**，所以 (b) 是重复的；而 13.2 的理由（「备份的用途是灾难恢复，裁剪过的备份没有恢复价值」）是在论证「不该裁剪」，**不是在论证「导出必须是完整备份」**。**因此本节倾向 (a)**，并且如果产品的真实场景是「管理员导出给别人看」（13.2 末句提到的那个场景），那需要的是**第二个脱敏导出端点**（日志篇 `export.masked` 已经预留了这个布尔）。

### 5.4 I1 展开：`GET /api/v1/equipment/import/template` 的三个选项

| 选项 | 内容 | 代价 |
|---|---|---|
| **(a) 服务端生成** | 端点从**与解析器同一份列名常量**生成 CSV，带 UTF-8 BOM | 多一个端点与一个 handler。**收益是列名不可能与解析逻辑漂移**——而漂移的表现是「用户下载的模板上传后被拒」，很难查 |
| **(b) 前端静态文件** | `public/template.csv` | 零成本。**代价是列名有两份真相**（前端文件 + 后端解析器），而运行时篇 R3.2 与实现篇 V1 的整套思路都在反对「两份真相」 |
| **(c) 服务端生成 + 带 BOM** | 同 (a)，但明确 BOM | **这个细节是必需的**：运行时篇 R5.1 说中文 Windows 的 Excel 按 GBK 读 CSV，**而模板里全是中文列名**。不带 BOM 的 UTF-8 模板在 Excel 里打开会是乱码——**用户在下载模板的第一步就遇到乱码，而 R5.1 讲的乱码是发生在导入那一步的**。这条因果关系要写进主稿 |

---

## 6. 端点计数与覆盖核对

### 6.1 计数

| 分组 | 端点数 |
|---|---|
| §1.1 健康检查 | 2 |
| §1.2 登录 / 会话 / bootstrap | 4 |
| §1.3 账号管理 | 12 |
| §1.4 器材建档 / 列表 / 编辑 / 归档 | 6 |
| §1.5 状态流转：出库 / 归还 | 5 |
| §1.6 送修 / 完成 / 停用 / 恢复 / 出售 / 上传 / 导入 / 导出 | 11 |
| §1.7 申请 | 9 |
| §1.8 项目 | 6 |
| §1.9 分类 | 4 |
| §1.10 活动流 / 总览 / 设置 | 4 |
| **合计** | **62** |

（`GET /readyz` 与 `GET /healthz` 不在 `/api/v1` 下，但计入总数。若只算 `/api/v1` 下的业务端点，是 **60**。）

### 6.2 交接稿功能清单 → 端点覆盖核对

| 交接稿要求的功能 | 覆盖它的端点 | 状态 |
|---|---|---|
| 登录 | `POST /auth/login` | ✅ |
| 登出 | `POST /auth/logout` | ✅ |
| 会话 | `GET /auth/me`、`DELETE /users/{id}/sessions` | ✅（**续期无端点，是有意的**，见 §1.2） |
| 邀请 | `GET/POST /invites`、`DELETE /invites/{id}`、`POST /invites/{token}/accept` | ✅ |
| 账号管理 | `GET /users`、`GET/PATCH /users/{id}` | ✅（**改密码/重置密码缺失** → I4） |
| 改角色 | `POST /users/{id}/role` | ✅ |
| 停用 | `POST /users/{id}/disable`、`enable` | ✅ |
| 器材建档 | `POST /equipment` | ✅ |
| 器材编辑 | `PATCH /equipment/{id}` | ✅ |
| 器材归档 | `DELETE /equipment/{id}`、`POST /equipment/{id}/restore` | ✅ |
| 器材详情 | `GET /equipment/{id}` | ✅ |
| 列表搜索筛选排序 | `GET /equipment` | ✅（**`overdue` 筛选器缺失** → I18 选项 b） |
| 批量操作 | `POST /equipment/batch` | ⚠️ 形状是本节提案（G6） |
| 批量出库 | `POST /checkouts`（`equipmentIds[]`） | ✅ |
| 照片上传 | `POST /photos` | ✅（**multipart 字段名未定** → I7） |
| CSV 导入预览与提交 | `POST /equipment/import/preview`、`commit` | ✅（**路径与日志篇不一致** → G5） |
| 导出备份 | `GET /export` | ⚠️ **范围未定**（I9 / 主稿 13.2 阻塞项） |
| 出库 | `POST /checkouts` | ✅（**路径与日志篇冲突** → G2） |
| 归还 | `POST /checkouts/{id}/items/{itemId}/return` | ✅ |
| 送修 | `POST /equipment/{id}/maintenance` | ✅ |
| 维修完成 | `POST /maintenance/{id}/close` | ✅ |
| 停用 / 恢复（器材） | `POST /equipment/{id}/disable`、`enable` | ✅ |
| 出售 | `POST /equipment/{id}/sell` | ✅ |
| 申请人提交 | `POST /requests` | ✅ |
| 我的申请 | `GET /requests/mine` | ✅ |
| 撤回 | `POST /requests/{id}/withdraw` | ✅（**规则未定** → I10） |
| 管理员列表 | `GET /requests` | ✅ |
| 审批 | `POST /requests/{id}/approve`（+ `reject`） | ⚠️ **整体拒绝的归属未定**（I11） |
| 办理交付 | `POST /requests/{id}/deliver` | ✅（**与出库是否同端点未定** → G3） |
| 项目增删改 | `GET/POST/PATCH/DELETE /projects` | ⚠️ **`DELETE` 语义未定**（I16） |
| 项目状态流转 | `POST /projects/{id}/status` | ⚠️ **流转表缺失**（I16） |
| 分类增删改 | `GET/POST/PATCH/DELETE /categories` | ✅（**删除被引用的失败无码** → I17） |
| 活动流 | `GET /activity`（四个 scope） | ✅（**`includeUpdates` 的作用域未定** → I3） |
| 总览页聚合数据 | `GET /overview` | ⚠️ **字段完全未定义**（I18） |
| 数据与设置 | `GET/PATCH /settings`、`GET /export` | ⚠️ 设置的表结构已有、端点形状是本节提案；导出见 I9 |
| （交接稿未列）查维修记录 | `GET /maintenance/{id}`（**本节提案**） | ⚠️ **主稿 10.1 要求活动流的 `ref` 能点开 `maintenance`，但没有端点** → I33 |
| （交接稿未列）活动流标注已停用作者 | 无新端点，需要 `ViewActivityItem.actor` 加字段 | ⚠️ **需要主稿补充** → I34 |

### 6.3 日志篇事件清单 → 端点覆盖核对（交叉验证用）

日志篇第 6 节的 `kind` 是**封闭集合**，每个业务事件都应当有端点产生它。反查一遍，能发现「有事件没端点」或「有端点没事件」：

| `kind`（日志篇第 6 节） | 产生它的端点 | 核对结果 |
|---|---|---|
| `boot` | 无（进程启动） | ✅ 不需要端点 |
| `http.end` | 所有端点（中间件） | ✅ |
| `biz.create` | `POST /equipment` | ✅ |
| `biz.update` | `PATCH /equipment/{id}` | ✅ |
| `biz.checkout` | `POST /checkouts` | ✅ |
| `biz.return` | `POST /checkouts/{id}/items/{itemId}/return` | ✅ |
| `biz.maintenance` | `POST /equipment/{id}/maintenance`、`POST /maintenance/{id}/close` | ✅（**但没有任何读取维修记录的端点** → 见下） |
| `biz.approve` | `POST /requests/{id}/approve`（+ `reject`） | ⚠️ **`reject` 的归属未定**（I11）。**若 `reject` 不存在，`result: "REJECTED"` 就没有产生者** |
| `biz.deliver` | `POST /requests/{id}/deliver` | ✅ |
| `biz.archive` | `DELETE /equipment/{id}`、`POST /equipment/{id}/restore`、`POST /equipment/batch` | ✅（`restore` 布尔表达方向） |
| `biz.reject` | 所有返回业务拒绝的端点 | ✅ |
| `biz.conflict` | `23P01` 实际触发（无专门端点） | ✅ 在 `IntoResponse`/中间件里记（实现篇 5.1 第 2 点） |
| `auth.login` / `auth.fail` | `POST /auth/login` | ✅ |
| `auth.deny` | 所有 403 | ✅ |
| `auth.logout` | `POST /auth/logout` | ✅ |
| `upload.ok` / `upload.fail` | `POST /photos` | ✅ |
| `import.accept` / `import.reject` | `POST /equipment/import/commit`（+ `preview`） | ⚠️ **`route` 只有一个，无法区分预览与提交**（G5） |
| `export` | `GET /export`、`GET /equipment/import/template` | ✅ **这反证了模板下载是一个端点**（`kind` 只有「数据/模板」两个取值）→ I1 |
| `job.end` | 无（`jobs` binary，无 HTTP） | ✅ |
| `store.inconsistent` | 无（占用对账 job） | ✅ |
| `file.missing` | 无（R7.2 的每日检查 job） | ✅ |

**反查发现的两个缺口**（同 §5.2 的 I33、I34，此处保留是因为它们**只能从这个反查得到**——功能清单里看不出来）：

| # | 缺口 | 说明 |
|---|---|---|
| **→ I33** | **没有任何端点能读维修记录（`maintenance`）。** 日志篇有 `biz.maintenance`，主稿 3.3 有 `maintenance` 表（含 `fault`/`vendor`/`cost_cents`/`outcome`），主稿 10.1 的映射表说「点开后显示：跳 `maintenance`：故障说明、维修单位、费用」——**但没有任何端点能取它**。活动流的 `ref` 字段（主稿 10.5）会给出 `{"kind":"maintenance","id":"55"}`，而**这个 ref 点不开**。缺口与建议见 §5.2 的 I33；`GET /api/v1/maintenance/{id}` 已收录在 §1.6 |
| **→ I34** | **`equipment_history.actor_id` 对应的用户如果被停用，活动流怎么显示「(已停用)」。** 主稿 10.6 明确要显示这个标注，而 `ViewActivityItem.actor`（主稿 10.5）只有 `{id, name}`。缺口与建议见 §5.2 的 I34 |

### 6.4 一条需要主稿确认的读路径缺口

**`requester` 怎么看「他的申请单里每件器材当前的状态」（主稿 10.4 末句）？** 主稿 10.4 说申请人的界面需要这个，但**申请人的可用端点里没有一个能给出「当前状态」的实时值**：`GET /requests/mine` 会返回 `items[].equipment.status`（§2.6 里有），所以这条是满足的。**本节记录这个核对过程，结论是：无缺口，但 §2.6 的 `items[].equipment` 必须包含 `status`。** 如果实现者为了「省一个 join」而不返回它，主稿 10.4 末句那句就会落空——而那是申请人界面**唯一**需要器材信息的地方。

---

## 附：本提案中被主稿采纳时需要同步修改的位置

| 要改的位置 | 改什么 | 依据 |
|---|---|---|
| 主稿 6.3 | `POST /api/photos` → `POST /api/v1/photos` | G1 |
| 主稿 7.5 | `GET /api/equipment` → `GET /api/v1/equipment` | G1 |
| 主稿 7.8 | `GET /api/checkouts/{id}/manifest` → `GET /api/v1/checkouts/{id}/manifest` | G1 |
| 主稿 5.3 | 「批量删除」措辞 → 与 3.4 的「归档」对齐；`BatchRejected` 的文案「不能删除」 | §1.4 末注 |
| 主稿 第 8 节 | 补全 `DomainError` → (HTTP, code) 映射（缺 4 个变体）；处理「`Conflict` 不存在」与「404 无变体」；补 500/503 分界与 413 | I23、I24、I26、I30 |
| 主稿 3.3 | `invite` 加 `revoked_at`（或明确撤销的语义）；`equipment_history` 两个索引加 `id` | I6 |
| 主稿 7.3 | 补 413 一行，并说明它是否仍走错误信封（`RequestBodyLimitLayer` 会破例） | I30 |
| 日志篇 4.1 | `msg`/`method`/`route` 三个字段全部改成 `POST /api/v1/checkouts` | G2 |
| 日志篇 4.4 | `route` 区分 `/import/preview` 与 `/import/commit`（或给 `data` 加阶段字段） | G5 |
| 实现篇 5.1 | `From<sqlx::Error>` 补 `23503` 与其余 `23505` 约束名；`DomainError::Conflict` 需要先在枚举里存在 | I25、I26 |
| 实现篇 5.2 | `match` 补 `BatchRejected`/`CheckedOutNotArchivable`/`DuplicateCode`/`RequestingDisabled`/404/413/503 | I24、I26、I30 |
| 实现篇 2 节 | 把「可见性用两个结构体」的纪律从器材推广到 `ViewRequest*` 等所有按角色分叉的 DTO | I22 |
