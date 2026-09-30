# LensMax 字段校验规则（提案 02）

日期：2026-09-30
状态：**提案，未实现。不改任何现有文档。**
范围：`app_user` / `invite` / `category` / `equipment` / `project` / `request` / `checkout` / `maintenance` / `photo` / CSV 导入行的字段级与跨字段校验规则；`garde` 的用法边界；应用层与数据库约束的分工。
依据：[主稿](../20260930_BACKEND_v1.md) 第 3.3 节完整 DDL、第 5 节写路径、第 7.2 节类型约定、第 7.6 节 PATCH 三态、第 8 节错误处理、第 13 节未决项；[运行时篇](../20260930_BACKEND_RUNTIME_v1.md) R5.1/R5.2/R5.4；[Rust 实现稿](../20260930_RUST_IMPL_v1.md) 第 2 节三套类型、第 5 节错误三层翻译；[日志设计](../20260930_LOGGING_v1.md) 第 3 节字段模式、第 4.4 节 `import.reject`、第 6 节事件清单。

---

## 该插到主稿的哪个位置

| 本文件的内容 | 落点 | 为什么是这里 |
|---|---|---|
| 第 1、2 节（规则总表 + 跨字段规则） | 主稿**新增「字段校验规则」一节，放在 §7.9「传输层」之后、§8「错误处理」之前** | 它管的是线上契约（DTO 的可空、长度、格式），与 §7 同类；而且它必须紧跟 §7.6（PATCH 三态）与 §7.7（可见性）才能读懂。**不要新增顶级 §8**——「主稿 7.3」「主稿第 8 节」这两个引用散落在另外三份文档里，插一个顶级节会把它们全部指错 |
| 第 4 节（`garde` 用法边界） | 上一行那一节的末两个小节；同时**修订 [Rust 实现稿](../20260930_RUST_IMPL_v1.md) §5** 的 `AppError::Validation` 形状（见 §4.5，本文给出理由） | 它是实现约定，主稿只需要指到 Rust 实现稿；但 `AppError::Validation(#[from] garde::Report)` 那个形状会让英文文案直接进用户表单，必须一并改 |
| 第 3 节（四个产品级待定项） | 主稿第 13 节新增四条未决项（编号见下方「编号冲突」） | 与 §13.1–13.10 同类：本稿给了默认值但需要产品拍板，实现前必须收敛 |
| 第 5.3 节（DDL 缺口清单 G1–G16） | 主稿第 13 节新增一条「迁移前要补的 DDL 缺口」（编号见下方），或作为 §3.3 之后的注 | 每一条都是"主稿第 3 节缺了一个约束"或"某个 CHECK 与应用层规则的关系没写清"。**本文件不代改 DDL**，只点名 |
| 第 0.1 节的长度取值依据 A1–A6 | 随规则总表一起进主稿（它是长度上限可被复核的唯一依据，不能只留一张数字表） | 没有它，下一个人会把 60 改成 200 |

**一句话**：新文件，零改动。除了上面这五处落点，主稿、运行时篇、Rust 实现稿、日志设计都不动。

### 本文件的验证边界

**这台机器上没有 `cargo`，也没有 `psql`。** 所以：

| 结论 | 强度 |
|---|---|
| 表格里每一条"与数据库约束的关系" | **来自对主稿 §3.3 DDL 的逐列人工核对**（`NOT NULL` / `CHECK` / `UNIQUE` / 部分唯一索引 / 外键），不是实测 |
| `garde 0.20` 的具体 API 面（自定义校验函数的签名、`Report::iter()` 的产出类型、规则级 `message` 是否存在） | **未编译验证**。第 4 节对此的处理是：把版本相关的写法全部标出来，并给出一个**不依赖这些细节**的兜底方案（正则类规则一律 `custom`；中文文案优先与规则同行，兜底走映射表 + 元测试） |
| §5.3 的 G13 / G14（`23514` / `23503` 未翻译、`DomainError::Conflict` 不存在） | **grep 级别的确认**（主稿 §8 与 Rust 实现稿 §5.1 的 `match` 分支里确实只有 `23P01` 和 `23505`；主稿 §8 的枚举确实没有 `Conflict`），不是运行时实测。实现第一步应当在 `sqlx::test` 里各写一条断言（与运行时篇 R3.2 #6 同法） |
| G15（`time::Date` 的默认 serde 形状） | **来自 `time` 的文档行为**，未实测。验证方式：一条契约测试断言日期字段在 JSON 里是字符串 |

**这个边界本身要写进文档**：上面三类结论若被当成实测结果引用，会在实现第一步就被推翻。

### 编号冲突（写作期间发现，**需要合并时统一裁决**）

本提案与 `docs/proposals/` 下并行产出的其它提案都选择"插在第 7 节末尾"，因此编号已经撞车。提出一个可执行的裁决规则，但**这个分配不该由单份提案自行决定**：

| 想要的位置 | 谁在争 | 建议裁决 |
|---|---|---|
| **§7.10** | `01_api_inventory.md`（「7.10 接口清单」）与本提案（「字段校验规则」） | 按提案编号顺序：**01 → §7.10 接口清单；本提案（02）→ §7.11 字段校验规则**。两者都主张"第 7 节的最后一个子节"，合并时只能有一个在后；顺序上接口清单在前、字段校验在后即可（规则表的"字段"列与接口清单的"请求体"列互为索引，任一顺序都能读，但**要固定一个**，否则后续提案会继续撞） |
| **§13.11–§13.13** | `05_owner_toggles.md`（要 13.11）与 `07_cascade_rules.md`（要 13.11、13.12、13.13）**这两份自己就撞了** | 按提案编号顺序顺延：05 → 13.11；07 → 13.12–13.14；**本提案（02）→ 13.15–13.18**（四个待定项）+ **13.19**（DDL 缺口清单） |

**本文件正文里不写死这些编号**，只用「第 3 节第 1 条」「§5.3 G7」这类稳定标识自引用，所以编号一旦裁决下来，只需改上表。

**还有一处跨提案的交汇点，需要合并时一起看**：本提案 §5.3 的 **G13（SQLSTATE 翻译表缺 `23514` / `23503`）** 与 `03_retry_policy.md`（扩同一张表：`40P01` 与连接类错误）、`07_cascade_rules.md`（同一条 `23503`，从 `ON DELETE` 角度）指向**同一段代码**。三份提案都主张改它，合并时应当**一次改完**，而不是三次分别改。

---

## 0. 这份文档补的是哪个空白

`garde` 在主稿 §2.1 被选为校验库，但**四份文档加起来没有一处写了具体规则**。现在文档里能查到的只有两句：主稿 §7.3 状态码表里的「校验失败（**缺项目名称**、日期区间反了）」、主稿 §3.3 的 `fault text NOT NULL -- 故障说明，必填`。

结果是实现者必须自己决定：器材名称多长算超？内部备注能不能到 1 MB？出库日期能不能填昨天？手工编号能不能带空格？**这四个决定没有一个有文档依据，而它们每一个都会在真实使用中被撞到。** 尤其第三个：主稿 §5.5 定了字段编辑**每次提交写一条历史、且 `detail.changed` 同时存 `from` 和 `to`**（主稿 §10.2 明确"不要为了省空间只存新值"），所以一个没有上限的备注会被**每次编辑复制两份**进 `equipment_history`。

这份提案把"字段 → 上限 → 依据 → 文案"固定下来，并把需要产品拍板的部分显式标出来。

### 0.1 长度上限的取值依据（A1–A6）

**上限不是数据库要求的。** Postgres 的 `text` 上限约 1 GB，所有长度限制都只在应用层。上限真正的作用是：**不让一个错误输入变成一次昂贵的写入、一条巨大的历史记录、一行超长的日志。** 这是手机端现场录入的系统，中文为主（UTF-8 下 1 个中文字 = 3 字节），所以长度一律按 **`chars().count()`（Unicode 码点）** 计，**不是字节**，实现里要用一个明确的 `char_len()` 而不是 `len()`。

| 代号 | 锚点 | 数字与来源 |
|---|---|---|
| **A1** | 手机列表一行 | 竖屏 360–430 px 逻辑宽、14–15 px 字号 → 一行约 **12–14 个中文字**。列表字段的上限取一行的 2 倍（24–28 字）以内，否则用户无法在列表里核对，只能点进详情 |
| **A2** | 手机表单里能核对完的长文本 | 一屏 15–20 行 × 每行 15 字 ≈ **200–300 中文字**。自由长文本取它的 6–8 倍（≈ 2000 字）：容得下"粘一段报价/沟通记录"，又不会产生一个没人读得完的字段 |
| **A3** | A4 打印一行 | 出库清单 / 维修单 A4 竖版 10.5 pt 中文 → 一行约 **30 个中文字**。会出现在打印件上的字段（编号、名称、序列号、使用人、存放位置）上限取 2 行（≈ 60 字）以内，否则清单会折行串行 |
| **A4** | 历史放大 **×2** | 主稿 §5.5 + §10.2：`equipment_history.detail.changed` 对每个改动的字段**同时存 `from` 和 `to`**。字段上限在历史表里是 **×2**，而且每次编辑累积一条。`internal_note` 取 2000 → 单条 `UPDATE` 历史的 `detail` 最大约 4 KB。**这是"备注上限不能取 1 MB"的硬理由** |
| **A5** | 日志单行上限 | 日志设计 §9.5：采集器常见在 16 KB 截断，`import.reject` 的 `rejected[]` 因此限 20 条（§4.4）。任何字段上限都要保证"20 条错误描述 + 上下文"离 16 KB 还有一个数量级 |
| **A6** | 一次现场沟通的输入量 | 长文本（内部备注、项目备注、故障说明）的上限应落在"人一次愿意打完"的量级；超过它，用户不会在手机上填，字段就形同不存在 |

**取值方法**：先按上表定位该字段出现在哪几个面上（手机列表 / 手机表单 / A4 打印 / 历史 / 日志），取**最紧的那个锚点 ×建议倍数**，再向上取整到易记的整数。每个字段的依据在下面表格的「上限依据」列给出。

### 0.2 表头约定

| 列 | 含义 |
|---|---|
| **可空** | `否` = DDL 里是 `NOT NULL`；`是` = DDL 里可空。取自[主稿 §3.3](../20260930_BACKEND_v1.md)的逐列定义 |
| **类型** | 线上类型（主稿 §7.2 类型约定）：`string` / `date`（`YYYY-MM-DD`）/ `cents`（整数分）/ `bool` / `enum` / `id`（字符串形式的 ID）/ `datetime`（RFC 3339 带偏移） |
| **长度/范围上限** | `string` 是字符数上下限；`cents`/整数是闭区间；`date` 是日期区间 |
| **格式** | 正则或枚举取值。**注意**：`garde` 版本不保证内建正则校验器，所有正则类规则按 §4.2 写成 `custom` |
| **校验时机** | `C` = 创建（POST/PUT 首次写入）；`U` = 更新（PATCH 设为值）；`∅` = 更新时允许 `null` 清空；`—` = 不经 API 写入（服务端生成，或永不接受客户端输入）。 **`C / U∅`** = 创建必填、更新可改但**不可清空**（`null` → 400）；**`C / U`** = 创建必填、更新可改不可空；**`C / —`** = 创建时可写，之后不可改（字段**不出现在 PATCH DTO 里**，用类型而不是运行时报错来表达） |
| **与数据库约束的关系** | `一致` = 应用层规则是 DDL 约束的镜像；`仅应用` = DDL 没有这条；`DB 兜底` = 应用层预检查给人话、DB 约束保正确性。缺口编号见 §5.3 |
| **失败时的中文文案** | 直接可用的句子。除显式标注外，**一律 400**（`AppError::Validation`，Rust 实现稿 §5.2）。例外：状态流转非法 **422**（`ILLEGAL_TRANSITION`）、档期冲突 **409**（`RESERVED`）、唯一性冲突 **409**（建议新增，见 §5.3 G14） |

**长度上限一律以字符计，且 `trim` 之后计。** 归一化（§4.1）必须发生在校验**之前**，否则 `"   "` 能通过 `min = 1`。

⚠️ 标记 = 该上限需要产品拍板，见第 3 节。

---

## 1. 规则总表

### 1.1 `workspace` / `app_user` / `app_user_session` / `invite` / `workspace_setting`

| 字段 | 可空 | 类型 | 长度/范围上限 | 格式 | 校验时机 | 与数据库约束的关系 | 失败时的中文文案 |
|---|---|---|---|---|---|---|---|
| `workspace.name` | 否 | string | 1–50 | 单行文本，无控制字符 | `C`（工作区由 bootstrap 建立，无改接口） | 仅应用（`text`，DDL 无长度约束） | 工作室名称不能为空 / 工作室名称最多 50 个字符 |
| `app_user.username` | 否 | string | 3–32 | `^[A-Za-z0-9._-]+$`，归一化小写 | `C / —`（用户名不提供改接口） | **一致 + DB 兜底**：`UNIQUE (workspace_id, username)`。注意该唯一约束**区分大小写**，见 §5.3 G9 | 登录名需为 3–32 位字母、数字、点、下划线或短横线 / 登录名最多 32 个字符 / 已存在这个登录名 |
| `app_user.display_name` | 否 | string | 1–24 | 单行文本 | `C / U`（`null` 或空串 → 400） | 仅应用 | 姓名不能为空 / 姓名最多 24 个字符 |
| `app_user.password` | 否 | string | 8–72 字符 **且 ≤ 128 字节** | 不以 `null` 出现；不写入日志（日志设计 §8.2 L1） | `C / U`（改密码是独立接口，需 `currentPassword`） | 仅应用。**上界不是安全要求**，是给 `argon2` 的输入定界——一个 5 MiB 的"口令"等于一次免费 DoS | 密码至少 8 位 / 密码最多 72 位 |
| `app_user.password_confirm` | 否 | string | 同 `password` | 必须与 `password` 逐字节相等 | `C / U` | 仅应用（跨字段，见 §2） | 两次输入的密码不一致 |
| `app_user.role` | 否 | enum | `OWNER` / `ADMIN` / `REQUESTER` | `SCREAMING_SNAKE` | `C / U`（改角色走独立接口，**OWNER 不可被改**） | **一致 + DB 兜底**：DDL `CHECK (role IN …)`；`app_user_one_owner` 部分唯一索引保证唯一主理人 | 角色取值不合法 / 主理人的权限不能被修改 |
| `app_user.disabled_at` | 是 | datetime | — | RFC 3339 | `U`（停用/恢复，非 PATCH 字段） | **一致**：DDL `CHECK (role <> 'OWNER' OR disabled_at IS NULL)`，但撞上它是 `23514` → 见 §5.3 G13 | 主理人不能被停用 |
| `app_user.workspace_id` | 否 | id | — | — | `—` 服务端从 `Actor` 取 | **仅应用**：DDL 是单列外键，无法阻止跨工作区，见 §5.3 G7 | （内部错误） |
| `invite.role` | 否 | enum | `ADMIN` / `REQUESTER`（**不含 `OWNER`**） | — | `C` | **一致 + DB 兜底**：DDL `CHECK (role IN ('ADMIN','REQUESTER'))`。**这正是"主理人只能有一个"的第二道保证** | 邀请只能是管理员或申请人 |
| `invite.expires_at` | 否 | datetime | `now() < x ≤ now() + 30 天` | RFC 3339 | `C`（生成时） | 仅应用（DDL 无 CHECK；值随 `now()` 变，静态 CHECK 表达不了） | 邀请有效期必须在 30 天以内 / 邀请有效期必须晚于当前时间 |
| `invite.token_hash` / 消费条件 | 是 | — | 消费时须 `consumed_at IS NULL` 且 `expires_at > now()` 且 `workspace_id` 一致 | — | `U`（消费） | `token_hash UNIQUE` 由 DB 保证；**有效期与一次性只在应用层** | 邀请链接已失效，请让管理员重新生成 |
| `app_user_session.expires_at` | 否 | datetime | `> created_at` | — | `—` 服务端 | 仅应用 | （内部） |
| `workspace_setting.requester_can_*` | 否 | bool | `true` / `false` | 布尔，不写 `"yes"` | `C / U` | 一致（`NOT NULL DEFAULT true`） | （内部） |

### 1.2 `category`

| 字段 | 可空 | 类型 | 长度/范围上限 | 格式 | 校验时机 | 与数据库约束的关系 | 失败时的中文文案 |
|---|---|---|---|---|---|---|---|
| `name` | 否 | string | 1–20 | 单行文本 | `C / U∅`？**否**：`U`（不可清空） | **一致 + DB 兜底**：`UNIQUE (workspace_id, name)` | 分类名称不能为空 / 分类名称最多 20 个字符 / 已存在同名分类 |
| `code_prefix` | 否 | string | 1–5 | `^[A-Z0-9]{1,5}$`，**归一化时先转大写** | `C / U` | **一致**：DDL `CHECK (code_prefix ~ '^[A-Z0-9]{1,5}$')` + `UNIQUE (workspace_id, code_prefix)`。大写归一化是必需的：不转的话用户输入 `cam` 会撞 `23514` → 500（§5.3 G13） | 分类前缀只能是 1–5 位大写字母或数字 / 分类前缀最多 5 位 / 已有分类使用这个前缀 |
| `next_seq` | 否 | 整数 | ≥ 1，**不经 API** | — | `—` 服务端 | 仅应用（DDL 无 `CHECK (next_seq >= 1)`，§5.3 G5） | （内部） |

**`name` 依据**：A1（手机上一行 12–14 字）×1.5 → 20。分类在手机上是一个 chip，20 字已经超出单行。
**`code_prefix` 依据**：DDL 给定 1–5。补充理由：前缀要拼进出库清单上的编号（`CAM-001` 共 7 字符），5 是 `LIGHT` / `CAMRA` 这类可读词的上限。

**⚠️ 改前缀的语义（主稿 §3.5）**：编号一旦生成永不改变，所以改前缀**不影响任何历史编号**。但 UI 必须提示这一点——否则管理员会以为改前缀是在"重命名"已有器材。见 §2 第 4 条。

### 1.3 `photo`

| 字段 | 可空 | 类型 | 长度/范围上限 | 格式 | 校验时机 | 与数据库约束的关系 | 失败时的中文文案 |
|---|---|---|---|---|---|---|---|
| 上传体（multipart `file`） | 否 | binary | ≤ `max_upload_bytes`（默认 8 MiB，主稿 §6.3）**在读流时生效** | 魔数 `FF D8 FF`（JPEG）。**不信任 `Content-Type`** | `C` | 仅应用 | 照片太大，请压缩后重试 / 只支持 JPEG 照片 |
| 解码尺寸 | — | — | ≤ 8000 × 8000 像素 | — | `C`（生成缩图那一步） | 仅应用。主稿 §6.3 点名这是流水线上唯一的真实攻击面（解压炸弹） | 照片尺寸过大，无法处理 |
| `sha256` | 否 | string | 64 个字符 | `^[0-9a-f]{64}$` | `—` 服务端边收边算 | `UNIQUE (workspace_id, sha256)` → 命中即幂等返回，不是错误（主稿 §6.2） | （内部） |
| `rel_path` | 否 | string | 固定形状 | `^[0-9a-f]{2}/[0-9a-f]{2}/[0-9a-f]{64}(\.thumb)?\.jpg$` | `—` 服务端按 hash 生成 | **仅应用，且是硬规则**：路径里绝不能出现任何用户输入（主稿 §6.2 的"路径穿越"一栏） | （内部） |
| `width` / `height` | 否 | 整数 | 1–8000 | — | `—` 服务端解码得出 | 仅应用（DDL 无 `> 0`，§5.3 G6） | （内部） |
| `bytes` | 否 | 整数 | 1 – `max_upload_bytes` | — | `—` 服务端 | 仅应用 | （内部） |
| `original_name` | 是 | string | 0–255 | 剥离路径分隔符（`/` `\`）、控制字符；**超长截断并保留扩展名** | `C` 上传时 | 仅应用 | 不报错：文件名仅用于展示，截断比拒绝好（拒绝会让用户在手机上无法自己解决） |

### 1.4 `equipment`

| 字段 | 可空 | 类型 | 长度/范围上限 | 格式 | 校验时机 | 与数据库约束的关系 | 失败时的中文文案 |
|---|---|---|---|---|---|---|---|
| `code` | 否 | string | **1–32** ⚠️（§3.4） | **无格式校验**（主稿 §3.5：`CAM-001-备用机` 合法）。仅拒绝控制字符、`trim` 首尾空白 | `C / —`（生成后永不改变，**字段不出现在 PATCH DTO 里**） | **DB 兜底**：`UNIQUE (workspace_id, code)` → `23505`。**不做格式校验是有意的** | 器材编号不能为空 / 器材编号最多 32 个字符 / 编号已存在：CAM-003 |
| `name` | 否 | string | **1–60** ⚠️（§3.3） | 单行文本 | `C / U` | 仅应用 | 器材名称不能为空 / 器材名称最多 60 个字符 |
| `category_id` | 否 | id | 必须存在、同一工作区 | 字符串 ID | `C / U` | 存在性由 FK 保证（`23503` 未翻译 → 500，§5.3 G13）；**同工作区只在应用层**（G7） | 请选择器材分类 / 所选分类不存在 |
| `brand` | 是 | string | 0–50 | 单行文本 | `C / U∅` | 仅应用 | 品牌最多 50 个字符 |
| `model` | 是 | string | 0–50 | 单行文本 | `C / U∅` | 仅应用 | 型号最多 50 个字符 |
| `serial_number` | 是 | string | 0–64 | 单行文本 | `C / U∅` | 仅应用。**申请人不可见**（主稿 §7.7），但它出现在出库清单上 → 打印件走专用接口 | 序列号最多 64 个字符 |
| `location` | 是 | string | 0–40 | 单行文本 | `C / U∅` | 仅应用 | 存放位置最多 40 个字符 |
| `purchase_date` | 是 | date | `1970-01-01` ≤ x ≤ `business_today()` | `YYYY-MM-DD` | `C / U∅` | 仅应用 | 购买日期不能晚于今天 / 购买日期格式应为 2024-03-01 |
| `price_cents` | 是 | cents | `0 – 10_000_000_000`（1 亿元） | 整数 JSON number，**不接受小数** | `C / U∅` | 仅应用（DDL 无 `>= 0`，§5.3 G2） | 购买价格不能为负数 / 购买价格不能超过 1 亿元 |
| `internal_note` | 是 | string | **0–2000** ⚠️（§3.3） | 多行文本，`\r\n` → `\n` | `C / U∅` | 仅应用。**受 A4 约束** | 内部备注最多 2000 个字符 |
| `photo_id` | 是 | id | 必须存在、同一工作区 | — | `C / U∅` | 存在性 FK；同工作区仅在应用层 | 照片不存在，请重新上传 |
| `status` | 否 | enum | `IN_STOCK`/`CHECKED_OUT`/`MAINTENANCE`/`DISABLED`/`SOLD` | — | `—` **只能经状态机接口变更**，不在 PATCH DTO 里 | **一致**：DDL `CHECK`。变更非法 → **422** | 不能从"维修"改为"出库" |
| `archived_at` | 是 | datetime | — | — | `U`（归档/恢复接口） | 仅应用：**"出库中禁止归档"DDL 没有**（§5.3 G8）。撞 `CHECK` 也不会拦（没有这个 CHECK） | 已出库的器材不能归档，请先归还 |
| `created_at` / `updated_at` | 否 | datetime | — | — | `—` 服务端 | `DEFAULT now()`，**Postgres 没有 `ON UPDATE`**：任何 UPDATE 必须显式 `SET updated_at = now()`（§5.3 G11） | （内部） |

**`code` 依据**：A3（A4 打印一行 ≈ 30 字）×1 → 32。自动生成最长约 `LIGHT-1000000`（13 字符，主稿 §3.5 允许序号加宽），手工 `CAM-001-备用机` 是 10 字符。32 给出约 2.5 倍余量，同时保证出库清单上"编号"这一列不被撑开。
**`name` 依据**：A3 × 2 = 60。器材名要同时出现在手机列表（截断显示）、出库清单（完整打印）和搜索里。
**`serial_number` 依据**：跨品牌差异大（常见 8–24 位），64 是 A3 两行并给扫描枪采集留余量。
**`price_cents` 依据**：最贵的单件（ARRI 机身 + 镜头组）在百万元量级；1 亿元是三个数量级的余量。上限的作用是拦住"把 1280 打成 128000000"这类多打一位的输入，**不是业务上限**。文案用元、字段用分（主稿 §7.2）。

### 1.5 `project`

| 字段 | 可空 | 类型 | 长度/范围上限 | 格式 | 校验时机 | 与数据库约束的关系 | 失败时的中文文案 |
|---|---|---|---|---|---|---|---|
| `name` | 否 | string | 1–60 | 单行文本 | `C / U` | **一致 + DB 兜底**：`UNIQUE (workspace_id, name)` | 项目名称不能为空 / 项目名称最多 60 个字符 / 已存在同名项目 |
| `client` | 是 | string | **0–40** ⚠️（§3.3） | 单行文本 | `C / U∅` | 仅应用 | 客户名称最多 40 个字符 |
| `project_type` | 否 | enum | `PROMO`/`SHORT_FILM`/`TVC`/`EVENT`/`CAMPUS`/`OTHER` | — | `C / U` | **一致**：DDL `CHECK`。若产品要自定义类型 → 主稿 §13.4（改表 + 外键） | 请选择项目类型 |
| `start_date` | 是 | date | `1970-01-01` ≤ x ≤ `business_today() + 5 年` | `YYYY-MM-DD` | `C / U∅` | 仅应用 | 开始日期格式应为 2024-03-01 |
| `end_date` | 是 | date | 同上，且 ≥ `start_date`（§2） | `YYYY-MM-DD` | `C / U∅` | **仅应用 —— DDL 缺这条 CHECK**（§5.3 G1）。`request` 和 `checkout` 都有同类 CHECK，只有 `project` 漏了 | 结束日期不能早于开始日期 |
| `status` | 否 | enum | `ACTIVE`/`COMPLETED`/`ARCHIVED` | — | `C / U` | **一致**：DDL `CHECK` | 项目状态取值不合法 |
| `note` | 是 | string | **0–2000** ⚠️（§3.3） | 多行文本 | `C / U∅` | 仅应用 | 项目备注最多 2000 个字符 |

### 1.6 `request` / `request_item`

| 字段 | 可空 | 类型 | 长度/范围上限 | 格式 | 校验时机 | 与数据库约束的关系 | 失败时的中文文案 |
|---|---|---|---|---|---|---|---|
| `project_name` | 否 | string | 1–60 | 单行文本。**`project_id` 非空时由服务端用 `project.name` 覆盖**（§2 第 5 条） | `C / U` | 仅应用（DDL 是自由文本 `NOT NULL`，它的存在就是为了让申请人打不出项目时也能提交） | 项目名称不能为空 / 项目名称最多 60 个字符 |
| `project_id` | 是 | id | 必须存在、同一工作区 | 字符串 ID | `C / U∅` | 存在性 FK；同工作区仅应用（G7） | 所选项目不存在 |
| `use_start` | 否 | date | 见 §3.1；`1970-01-01` ≤ x ≤ `business_today() + 5 年` | `YYYY-MM-DD` | `C / U` | 仅应用 | 使用开始日期不能为空 / 使用开始日期格式应为 2024-03-01 |
| `use_end` | 否 | date | 同 `use_start`，且 ≥ `use_start` | `YYYY-MM-DD` | `C / U` | **一致 + DB 兜底**：DDL `CHECK (use_end >= use_start)`。撞上它是 `23514` → 500（G13） | 使用结束日期不能早于开始日期 |
| `note` | 是 | string | 0–2000 | 多行文本 | `C / U∅` | 仅应用 | 申请备注最多 2000 个字符 |
| `items[]` | 否 | id 数组 | **1–100** 件，**不得重复** | — | `C / U`（改申请是撤回后重提） | `UNIQUE (request_id, equipment_id)` 兜底 | 请至少选择一件器材 / 一次最多申请 100 件 / 申请里有重复的器材 |
| `items[].equipment_id` | 否 | id | 必须存在、同一工作区、未归档 | — | `C` | 存在性 FK；**同工作区与"未归档"只在应用层** | 以下器材不可申请：CAM-003（已出售） |
| `review_note` | 是 | string | 0–500 | 多行文本 | `U`（审批接口） | 仅应用 | 审批备注最多 500 个字符 |
| `items[].approved` | 是 | bool | `null`=未审 / `true`=批 / `false`=拒 | — | `U` | 一致（DDL 是三态 `boolean`） | 请至少通过一件器材才能批准（§2 第 6 条） |
| `status` | 否 | enum | `PENDING`/`APPROVED`/`PARTIAL`/`REJECTED`/`DELIVERED`/`RETURNED`/`WITHDRAWN` | — | `—` 只能经状态机接口 | **一致**：DDL `CHECK` | 不能从"已归还"改为"审批中" |
| `reviewed_by` / `reviewed_at` / `delivered_at` / `returned_at` | 是 | datetime | — | — | `—` 服务端 | 仅应用 | （内部） |

**`review_note` 依据**：A2 ÷ 4。审批备注是一句"档期排不开，改到 10-08 再提"。

### 1.7 `checkout` / `checkout_item`

| 字段 | 可空 | 类型 | 长度/范围上限 | 格式 | 校验时机 | 与数据库约束的关系 | 失败时的中文文案 |
|---|---|---|---|---|---|---|---|
| `project_id` | 是 | id | 必须存在、同一工作区 | — | `C / U∅` | 存在性 FK；同工作区仅应用 | 所选项目不存在 |
| `project_name` | 否 | string | 1–60 | 单行文本；`project_id` 非空时服务端回填 | `C / U` | 仅应用 | 项目名称不能为空 / 项目名称最多 60 个字符 |
| `user_name` | 否 | string | 1–24 | 单行文本（**自由文本，不是账号引用**，DDL 是 `text NOT NULL`） | `C / U` | **仅应用**：无 FK。所以"使用人"可以和任何账号无关——这是有意的（主稿 §5.2 的「使用人」） | 使用人不能为空 / 使用人最多 24 个字符 |
| `checkout_date` | 否 | date | 见 §3.1 | `YYYY-MM-DD` | `C / U` | 仅应用 | 出库日期不能为空 / 出库日期格式应为 2026-10-01 |
| `expected_date` | 是 | date | ≥ `checkout_date`，且 ≤ `business_today() + 730 天`（§2、§3.2） | `YYYY-MM-DD`；**`null` = 无期限**，占用到 `infinity`（主稿 §3.1） | `C / U∅` | **一致 + DB 兜底**：DDL `CHECK (expected_date IS NULL OR expected_date >= checkout_date)`。撞上它是 `23514` → 500（G13） | 预计归还日期不能早于出库日期 / 预计归还日期不能超过今天起 730 天 |
| `note` | 是 | string | 0–2000 | 多行文本 | `C / U∅` | 仅应用 | 出库备注最多 2000 个字符 |
| `equipment_ids[]` | 否 | id 数组 | **1–100** 件，不得重复 | — | `C` | `UNIQUE (checkout_id, equipment_id)` 兜底 | 请至少选择一件器材 / 一次最多出库 100 件 |
| 每件器材的当前状态 | — | enum | 必须是 `IN_STOCK` | — | `C` | 状态机在 `domain`，不在 garde（§4.3） | **422**：不能从"维修"改为"出库" |
| `items[].return_note` | 是 | string | 0–500 | 多行文本 | `U`（归还接口） | 仅应用 | 归还备注最多 500 个字符 |
| `status` | 否 | enum | `OPEN` / `CLOSED` | — | `—` 服务端（全部归还时自动 `CLOSED`） | **一致**：DDL `CHECK` | （内部） |
| `source_request_id` / `delivered_at` / `created_by` | 是/否 | — | — | — | `—` 服务端 | 仅应用 | （内部） |

**`equipment_ids` 上界 100 依据**：一次出库的清单要打印并手写签字（主稿 §13.3），几十件已是上限；上限的作用是拦住"全选 2000 件"把长事务和清单撑爆，而 `lock_many` 是 `WHERE id = ANY($1)`（Rust 实现稿 §3.3），没有上限时一个 5 MiB 请求体能塞进十几万个 ID。数字与列表接口的 `limit` 上限 100（主稿 §7.5）保持一致。
**`user_name` 依据**：A1 × 2 = 24。它出现在出库清单上，紧挨签名栏。中文姓名 2–4 字，24 容纳「张三（摄影组）」。

> **与接口清单提案（`01_api_inventory.md`）的一处待统一**：本行假设**存在一个批量出库请求体**（`equipment_ids[]`），依据是主稿 C4「批量操作整批拒绝、点名违规件，不做部分成功」、主稿 §5.3 的批量原子性写法、以及主稿 §5.1 的 `CheckoutCmd { equipment_ids }`。而 `01_api_inventory.md` 为出库选择了**逐件的嵌套式端点**（`POST /api/v1/equipment/{id}/checkout`）。这两者不必然冲突（*逐件端点* 与 *批量请求体* 可以并存），但**必须明确**：
> - 若只有逐件端点 → 本表的 `equipment_ids[]` 规则（1–100、不得重复）届时只对"批量操作接口"（批量删除/归档/停用，主稿 §5.3）生效，出库这一行要删掉；**而主稿 §5.1 的 `CheckoutCmd` 与"批量出库后生成清单"（主稿 §7.6）需要一并改写**。
> - 若两者并存 → 逐件端点也要复用它那一件的校验，**不能只校验"有一个 `{id}` 路径参数"**（路径里的 `id` 同样要过存在性、同工作区、未归档、状态机与占用检查）。
>
> 无论哪个结论，**"至少一件 + 上限 100 + 不得重复"这三条在批量请求体上都不变**。

### 1.8 `maintenance`

| 字段 | 可空 | 类型 | 长度/范围上限 | 格式 | 校验时机 | 与数据库约束的关系 | 失败时的中文文案 |
|---|---|---|---|---|---|---|---|
| `equipment_id` | 否 | id | 必须存在、同一工作区；且**当前没有未关闭的维修记录** | — | `C` | **DB 兜底**：`maintenance_one_open` 部分唯一索引。应用层预检查负责给人话 | 这件器材已有一条未完成的维修记录 / 已出库的器材不能送修，请先归还 |
| `fault` | 否 | string | 1–1000 | 多行文本 | `C / U` | 仅应用。日志只记 `fault_len` 不记原文（日志设计 §6：`biz.maintenance` 的 `data` 里是 `fault_len`） | 故障说明不能为空 / 故障说明最多 1000 个字符 |
| `vendor` | 是 | string | 0–50 | 单行文本 | `C / U∅` | 仅应用 | 维修单位最多 50 个字符 |
| `sent_date` | 否 | date | `1970-01-01` ≤ x ≤ `business_today()` | `YYYY-MM-DD` | `C / U` | 仅应用 | 送修日期不能为空 / 送修日期不能晚于今天 |
| `cost_cents` | 是 | cents | `0 – 10_000_000_000` | 整数 JSON number | `C / U∅` | 仅应用（DDL 无 `>= 0`，G2） | 维修费用不能为负数 |
| `note` | 是 | string | 0–2000 | 多行文本 | `C / U∅` | 仅应用 | 维修备注最多 2000 个字符 |
| `outcome` | 是 | enum | `BACK_TO_STOCK` / `DISABLED`。**关闭时必填**（`closed_at` 非空 ⟹ `outcome` 非空，§2 第 7 条） | — | `U`（关闭接口） | **仅应用 —— DDL 缺这条联动**（§5.3 G3）。DDL 的 `CHECK (outcome IN …)` 只约束取值 | 请选择维修结果 |
| `closed_at` | 是 | datetime | ≥ `sent_date`（按业务时区取日期比较，§2 第 7 条） | RFC 3339 | `U` | 仅应用（G4）。跨 `timestamptz` × `date`，DB 表达别扭，**建议留在应用层** | 维修完成日期不能早于送修日期 |
| `opened_at` / `created_by` | 否 | — | — | — | `—` 服务端 | 一致 | （内部） |

**`fault` 依据**：A2 ÷ 2 = 1000。送修单要写清症状与复现条件，但不需要 2000；同时它是会出现在 A4 打印件上的字段（A3 × 2 = 60 只够首行，多行文本按 A2 走）。

### 1.9 无 HTTP 写入口的表

主稿 §3.3 里有三张表**没有任何 handler 直接写它们**。它们不出现在规则总表的字段行里，但各有一条**唯一**的约束需要说明：

| 表 | 谁写 | 校验规则 |
|---|---|---|
| `equipment_history` | `services` 在写事务内 | `action` ∈ DDL 的 11 个取值（`CHECK`）；`detail.changed[].field` **只能是 `equipment` 的真实列名**（主稿 §10.2：字段名→中文标签的映射在前端）；`detail` 体积受"字段上限 × 2"约束（A4）。**无 `notes` 里的自由文本原文** |
| `occupancy` | `services`（出库/审批/归还/撤回） | `during` 必须是半开区间 `[start, end)`，`end = infinity` 表示无期限；`NOT isempty(during)`；两个占用区间不重叠由排他约束保证（应用层预检查给人话，§2 第 8 条）。DDL 无"`kind='RESERVATION'` ⟹ `source_request_id` 非空"，这是**有意留白**（占用对账任务就是查它，主稿第 11 节） |
| `pending_upload` | 上传流水线 | `sha256` / `rel_path` 同 `photo`；由孤儿清理任务按 24 小时窗口处理（主稿 §6.5） |

### 1.10 CSV 导入行

模板列固定为主稿 §13.9 的十列。**导入走的是"预览 → 确认 → 提交"，两次都跑同一套校验函数**（运行时篇 R5.3 的重校验）。

| CSV 列 | 映射到 | 可空 | 上限/格式 | 与 `garde` 的关系 |
|---|---|---|---|---|
| `器材编号` | `equipment.code` | 是（空 → 走主稿 §3.5 自动生成） | 1–32，无格式校验 | **不用 DTO 的 derive**：导入是一次性解析 + 一次性预取（R5.4），逐行用同一批纯函数 |
| `器材名称` | `equipment.name` | 否 | 1–60 | 同上 |
| `分类` | `category.name` → `category_id` | 否 | 必须命中**已预取**的分类名集合（R5.4：导入开始时一次性读进内存，不再逐行查库） | **需要查库的规则在导入里降级成"预取集合 + 内存比对"**——这是唯一一处不能复用 `db::*` 预检查的地方 |
| `品牌` / `型号` | `equipment.brand` / `model` | 是 | 0–50 | 同上 |
| `序列号` | `equipment.serial_number` | 是 | 0–64 | 同上 |
| `存放位置` | `equipment.location` | 是 | 0–40 | 同上 |
| `购买日期` | `equipment.purchase_date` | 是 | `YYYY-MM-DD` **或** `YYYY/M/D`；其余格式标红（运行时篇 R5.2） | 解析函数 `parse_biz_date(s) -> Result<Date, Reject>` |
| `购买价格` | `equipment.price_cents` | 是 | 接受 `1280` / `1280.00` / `¥1,280.00` / `1280元` / `1,280`；去货币符号与千分位 → **四舍五入到分**；范围 `0 – 1 亿元` | `parse_cents(s)`。**解析失败标红，绝不静默当 0**（R5.2 明确：把 1280 静默存成 0 比导入失败严重得多） |
| `备注` | **`equipment.internal_note`** | 是 | 0–2000 | **注意**：`equipment` 表里没有 `note` 列（只有 `internal_note`）。CSV 的「备注」写进 `internal_note` 这个映射要固定下来并写进模板说明 |

**请求级上限（运行时篇 R5.4，在读流时生效）**

| 项 | 上限 | 失败文案 |
|---|---|---|
| 请求体 | 5 MiB | 文件太大，请拆分后导入 |
| 行数 | 1–1000（**交接稿** §5.1，由主稿 §7.9 转引——注意主稿自己的 §5.1 是「唯一形状」，不是这里） | 一次最多导入 1000 行 |
| 单字段长度 | 同 §1.10 各列上限，**在解析阶段生效** | 第 7 行「器材名称」：最多 60 个字符 |
| 预览行数 | 只回前 12 行（交接稿 §5.1，主稿 §7.9 转引） | — |
| 编码 | UTF-8 BOM → 严格 UTF-8 → GBK 回退（R5.1）；BOM 必须从表头剥掉 | 文件编码无法识别，请另存为 UTF-8 或 GBK 的 CSV |
| 列名 | `trim` + 全角转半角 + 忽略大小写后匹配；**缺列 → 整份拒绝**；多余列 → 忽略 | 模板缺少列：器材名称 |
| 错误行数 | `rejected[]` ≤ 20，超出加 `"truncated": true`（日志设计 §4.4） | 共 412 行有问题，只显示前 20 行 |

**错误行的文案形状**（与日志设计 §4.4 的 `rejected[]` 三字段同构，可直接拼）：`第 {row} 行「{column}」：{reason}`，例如 `第 19 行「购买价格」：金额无法解析`。`value` 记**规范化前的原文**（`¥1,28O.00` 里那个字母 O 就是原因）。

### 1.11 永不接受客户端输入的字段

用**类型**表达"不可写"：这些字段根本不出现在请求 DTO 里，而不是"接受后校验拒绝"。

| 类别 | 字段 |
|---|---|
| 主键与归属 | `id`、`workspace_id`、`created_by`、`actor_id`、`requester_id`、`reviewed_by` |
| 服务端算出来的 | `created_at`、`updated_at`、`archived_at`、`opened_at`、`closed_at`、`delivered_at`、`returned_at`、`reviewed_at`、`uploaded_at` |
| 状态与计数器 | `status`、`next_seq`、`approved`（只经审批接口） |
| 路径与摘要 | `sha256`、`rel_path`、`token_hash`、`password_hash`、`id`（`app_user_session`） |
| **永不改变** | `equipment.code`（主稿 §3.5：生成后永不改变） |

**注意**：不建议给请求 DTO 加 `#[serde(deny_unknown_fields)]`。它会破坏运行时篇 R6.2 第 1 条（"加可选字段：安全"）——旧后端 + 新前端会 400，而那条规则要的恰恰是这个组合能跑。代价是"客户端发了 `status`"会被静默忽略；如果需要可观测性，记一条 WARN 而不是 400。

---

## 2. 跨字段规则

这些规则**不能**用 `garde` 的 derive 表达（`garde` 的 `custom` 是按字段生效的，拿不到兄弟字段），落点是 `domain` 的纯函数（零 I/O，可被几百个单测覆盖）+ `services` 的事务内预检查。理由见 §4.3。

| # | 规则 | 涉及字段 | 时机 | DB 是否已保证 | 失败文案 | 落点 |
|---|---|---|---|---|---|---|
| 1 | `use_end >= use_start` | `request` | `C / U` | ✅ DDL `CHECK` | 使用结束日期不能早于开始日期 | `domain::validate::request_range` |
| 2 | `expected_date IS NULL OR expected_date >= checkout_date` | `checkout` | `C / U` | ✅ DDL `CHECK` | 预计归还日期不能早于出库日期 | `domain::validate::checkout_range` |
| 3 | `expected_date <= business_today() + 730 天` | `checkout` | `C / U` | ❌ 仅应用 | 预计归还日期不能超过今天起 730 天 | 同上。**730 天是可调常量**，见 §3.2 |
| 4 | `project.end_date >= project.start_date`（任一为 `NULL` 时放行） | `project` | `C / U` | ❌ **DDL 缺**（G1） | 结束日期不能早于开始日期 | `domain::validate::project_range` |
| 5 | `project_id` 非空时，`project_name` 由服务端用 `project.name` **覆盖**，不采信客户端 | `request` / `checkout` | `C / U` | ❌ 仅应用 | （不报错；响应里回填后的值，主稿 §7.6「变更返回更新后的资源」） | `services`（事务内已读到项目行） |
| 6 | 审批：至少要有一件 `approved = true` 才能把申请置为 `APPROVED`/`PARTIAL`；全 `false` → `REJECTED`；全 `null` → 拒绝提交 | `request_item[]` | `U` | ❌ 仅应用 | 请至少通过一件器材 / 请先逐件勾选审批结果 | `services::review` |
| 7 | `closed_at` 非空 ⟹ `outcome` 非空；且 `closed_at` 的业务日期 ≥ `sent_date` | `maintenance` | `U` | ❌ **DDL 缺**（G3、G4） | 请选择维修结果 / 维修完成日期不能早于送修日期 | `domain::validate::maintenance_close` |
| 8 | 出库/审批时该器材的 `[start, end)` 与既有占用不重叠 | `occupancy` | `C` | ✅ 排他约束 `occupancy_no_overlap` | **409**：已被「宣传片A」在 2026-10-01 至 2026-10-05 占用 | 预检查 `db::occupancy::find_conflict`（给人话）+ 约束（正确性），主稿 §3.2 |
| 9 | 批量操作 `equipment_ids.len() >= 1`，且**去重后**不得有重复 | 各批量接口 | `C` | `UNIQUE (…, equipment_id)` 兜底 | 请至少选择一件器材 / 选择里有重复的器材 | `garde` 的 `length(min = 1, max = 100)` + 一个手写 `dedup` 检查 |
| 10 | 批量操作的所有器材必须属于**同一工作区**、且未归档 | 各批量接口 | `C` | ❌ 仅应用（G7） | 以下器材不可操作：CAM-003（已归档） | `services`：`lock_many` 返回的行数必须等于请求的 ID 数（主稿 §5.1 第 1 步已经这么做，`len` 不等即 404/400） |
| 11 | 改 `category.code_prefix` 时，新前缀不得与**其它**分类重复 | `category` | `U` | ✅ `UNIQUE (workspace_id, code_prefix)` | 已有分类使用这个前缀 | 预检查给人话（`db::category::prefix_taken`）+ `23505` 兜底 |
| 12 | **改前缀不影响任何已生成的编号**（主稿 §3.5 第 2 条）。所以这条不是校验规则，是**必须由 UI 提示的产品事实**：改前缀只影响此后自动生成的编号 | `category` | `U` | — | （接口返回 `affectedCodes: 0` 或加载时的提示文案：「已有 12 件器材的编号不会改变」） | API 响应字段 |
| 13 | 手工编号与自动编号走**同一**唯一约束；若某分类的新前缀会让未来自动编号撞上已有手工编号，只在真正撞上时由 `23505` 兜底 | `category` / `equipment` | `C` | ✅ `UNIQUE (workspace_id, code)` | 编号已存在：CAM2-001 | 预检查 + 翻译（主稿 §3.5 末句） |
| 14 | 停用账号必须同时删除其全部会话（否则"停用没生效"，运行时篇 R7.4 第 2 条） | `app_user` | `U` | ❌ 仅应用 | （不报错；同一事务内 `DELETE app_user_session`） | `services::account::disable` |
| 15 | 改角色：`OWNER` 不可被改（主稿第 9 节）；一个工作区只能有一个 `OWNER` | `app_user` | `U` | ✅ 部分唯一索引 `app_user_one_owner` | **403**：主理人的权限不能被修改 | `services::account::change_role`（显式判断，`Actor::can_change_role`） |
| 16 | `password == password_confirm` | 账号创建/改密 | `C / U` | ❌ 仅应用 | 两次输入的密码不一致 | `garde` 手写（结构体级函数） |
| 17 | 归档：`status != CHECKED_OUT`；**并对"有未关闭维修记录时是否允许归档"提出待确认**（主稿 §3.4 只写了出库） | `equipment` | `U` | ❌ 仅应用（G8） | 已出库的器材不能归档，请先归还 | `services::equipment::archive` |
| 18 | 时间区间半开 `[start, end)`，`end = infinity` 表示无期限（主稿 §3.1） | `occupancy` | `C` | ✅ `daterange` | （内部） | `domain::occupancy::checkout_range` 纯函数 |
| 19 | 所有必填文本字段先 `trim` 再判空：`"   "` 必须被当作空 | 全部 `string` 必填字段 | `C / U` | ❌ 仅应用 | 器材名称不能为空 | **归一化阶段**（§4.1），必须在 `garde` 之前 |
| 20 | 同一份 CSV 内部两行给出同一 `器材编号` | 导入 | 提交时 | `UNIQUE` 兜底 | 第 7 行与第 3 行编号重复：CAM-001 | 导入器（内存里已有预取集合） |

**第 12 条值得展开**（任务点名的"分类前缀改动"）：主稿 §3.5 把"编号一旦生成永不改变"定成硬规则，于是改前缀的语义是**只对未来的自动编号生效**。这带来两个后果，都不是校验能解决的，而是要在文档和 UI 里说清：① 管理员会以为改前缀是在"重命名"已有器材 → 必须在接口响应里给出"受影响编号数 = 0"并配提示；② 若新前缀与已有手工编号的字头相同，未来某次自动编号会撞上 `23505` —— 这是**允许发生**的（主稿 §3.5 末句），但不能让它变成 500，所以 `equipment_code_uniq` 的翻译必须在位（§5.3 G14）。

---

## 3. 产品级规则的待定项

**以下四条需要产品拍板，本文档给的是建议与影响分析，不是结论。** 每条都要进主稿第 13 节；**具体编号待合并时统一分配**（见文首「编号冲突」——`05` 与 `07` 已各自占了 13.11–13.13，这两份自己就撞车了）。下面用本文档内的 §3.1–§3.4 自引用，不写死主稿编号。

### 3.1 出库日期能不能早于今天？

（这一条同时决定 `request.use_start`，两者是同一个问题。）

| 选项 | 允许 | 影响 |
|---|---|---|
| **A（建议）允许过去、禁止未来** | `1970-01-01 ≤ checkout_date ≤ business_today()` | 现场没信号、忙到忘了点、纸质单事后补录——**这三种在十人工作室里每周都会发生**。代价：补录一条过去区间的 `CHECKOUT` 占用时，可能与一条**当前有效**的 `RESERVATION` 冲突 → 409，用户需要理解"我补录昨天，为什么被下周的预约挡住"。所以错误文案必须点名冲突项目（日志设计 §4.2 的 `biz.reject` 已经这么设计） |
| B 允许过去也允许未来 | 无上界 | 会出现"2027-03-01 出库"这种记录，逾期扫描、总览页的"当前出库"全部失真。**不建议** |
| C 禁止过去 | 只能 `= business_today()` | 补录需求被否掉。只有在产品明确"不存在事后补录"时才成立 |

**建议 A。** 三条补充：
1. **未来日期必须禁止**——它没有任何真实需求，而它会让"出库中"的语义和逾期判断同时失效。
2. 补录出来的记录**很可能立刻是逾期状态**（`expected_date < today`）。这是正确行为，但 UI 应在保存后立刻把逾期提示显示出来，而不是等第二天的 job。
3. `use_start`（申请）同理：允许过去（补录已开始的拍摄），禁止超过 `today + 5 年`（拦住 `2062` 这类打错的年份）。

**需要产品回答的具体问题**：`2026-09-29` 这种"补录昨天出库的"，是不是真实需求？

### 3.2 预计归还日期能不能是过去？

**先说一个观察，它能省掉一次讨论**：在[主稿 §3.3](../20260930_BACKEND_v1.md) 已有的 `CHECK (expected_date IS NULL OR expected_date >= checkout_date)` 之下，**这条没有独立自由度**——

- 若 §3.1 选 A/C（禁止未来出库），`checkout_date <= today`，而 `expected_date >= checkout_date`，所以 `expected_date` 可以是过去，**当且仅当** `checkout_date` 也是过去。也就是"只有补录的单子才可能有过去的预计归还日"。这正是我们想要的语义。
- 若 §3.1 选 B，"预计归还日期不能早于今天"才需要单独一条规则。

**所以：§3.2 由 §3.1 决定，不需要单独拍板。** 真正独立、需要拍板的是它的**上界**：

| 选项 | 上限 | 影响 |
|---|---|---|
| **A（建议）** | `business_today() + 730 天`（2 年） | 拦住年份打错（`2026` → `2062`）。两年足够覆盖任何长租/外借；真要不定期就用 `null`（DDL 已经支持，占用到 `infinity`） |
| B 一年 | `+ 365 天` | 更紧，但"借出去一年以上"在这里本该用 `null` |
| C 不设上界 | 无 | `expected_date = '2062-10-05'` 会进库，然后逾期扫描和总览页都会把它当作一个正常的远期归还 |

**建议 A。** 需要产品确认：**`预计归还 = null`（无期限）与 `预计归还 = 两年后` 在业务上是不是两种不同的意思？** 如果产品认为"没有归还日期"就该填 `null`，那么上界可以收到 365 天。

### 3.3 器材名称 / 内部备注 / 项目备注 / 客户名的长度上限取多少？

这一条是本节最重要的：**没有上限时，一个 1 MB 的备注会进数据库、进历史、进 CSV 解析、进日志行**——日志设计 §4.4 已经为此把 `import.reject` 的 `rejected[]` 限在 20 条（§9.5 提到采集器常在 16 KB 截断单行）。上游不限，下游的每一个上限都在做无用功。

| 字段 | 选项 | 建议 | 依据 | 超限的具体后果（按建议值的反面） |
|---|---|---|---|---|
| `equipment.name` | 30 / **60** / 120 / 不限 | **60** | A3 × 2：出库清单 A4 一行的 2 倍 | 取 200 时，手机列表每行只能看到名字的前 8 个字，用户靠名字区分器材的能力归零；同时它进 `equipment_search_idx` 的 tsvector |
| `equipment.internal_note` | 500 / **2000** / 10000 / 不限 | **2000** | A2（一屏 200–300 字）× 8；**A4 是决定性约束** | 取 1 MB 时：每次编辑该字段，`equipment_history.detail.changed` 会写 `from` + `to` ≈ **2 MB**（主稿 §10.2 明确要求存原值）。改三次就是 6 MB，而这张表是永久的（日志设计 §2.1） |
| `project.note` | 500 / **2000** / 10000 / 不限 | **2000** | 同 A2 | 项目卡片与项目详情都要渲染它；项目备注不产生历史放大，所以这一条比内部备注松得起，但没有理由不同值 |
| `project.client` | 20 / **40** / 80 / 不限 | **40** | A1 × 3：客户名要出现在项目列表的一行里，还要出现在出库清单抬头 | 取 80 时客户名在列表里会折行，把项目名挤下去 |
| `request.note` | 500 / **2000** / 不限 | **2000** | 与项目备注同值，减少一个要记的数字 | — |
| `maintenance.fault` | 500 / **1000** / 5000 / 不限 | **1000** | A2 ÷ 2；它是会打印在送修单上的字段 | 取 5000 时，一条送修记录在日志里是 `fault_len`（不带原文，日志设计 §6）——所以日志不是约束，约束是**送修单的 A4 版面** |

**取舍的通用判断**：上限的价值不在"防数据库"，而在三件事——① **历史放大 ×2**（主稿 §5.5 + §10.2）；② **打印件 A4 版面**；③ **手机上一屏能不能核对完**。所以 **2000 是长文本的统一值**，只有故障说明按 A4 版面收紧到 1000。**建议这三类长文本用同一个常量**（`MAX_LONG_TEXT = 2000`），而不是各定一个数——少一个要记的数字就少一处漂移。

**需要产品回答的具体问题**：内部备注会不会被用来粘贴一整份报价单或聊天记录？如果是，2000 不够，那我们要讨论的是**它不是备注而是附件**（那要动主稿 §6.1 的存储范围，而不是把上限调大）。

### 3.4 手工填写的器材编号有什么字符和长度限制？

**先确认主稿的既有立场**（[主稿 §3.5](../20260930_BACKEND_v1.md) 末段）：

> 手工填写的编号也走同一个唯一约束。**不做"编号格式校验"**——前缀规则只约束自动生成的，用户手填 `CAM-001-备用机` 是合法的。

**这条立场应当保留，我确认它。** 理由：编号在这个系统里是**人给东西起的名字**（`CAM-001-备用机`、`BAT-A7M4-2`），不是机器标识——机器标识有 `equipment.id`（主稿 §7.2：ID 是字符串、语义上不可运算）。而"什么算合法名字"取决于这个工作室怎么说话，规则化只会逼出 `CAM-001-备用机2` 这种绕过。

但"无格式校验"不等于"什么都收"。建议在**不动格式**的前提下加三条**所有文本字段都该有的**约束：

| 项 | 建议 | 理由 |
|---|---|---|
| 长度 | **1–32** | A3 × 1。见 §1.4 的依据 |
| 字符 | **拒绝 Unicode 控制字符**（`Cc`、`Cf`，含 `\n` `\t` `\0` `\u200B`）；`trim` 首尾空白；允许中间空格 | 这个值会出现在：出库清单（打印）、`equipment_history`、CSV 导入导出、日志的 `data.equipment_code`（日志设计 §4.2）。一个带 `\n` 的编号会让打印件串行、让 CSV 回导变形 |
| 唯一性 | 用**现有**的 `UNIQUE (workspace_id, code)`，**区分大小写** | 见下面的"真正的待定项" |

**真正需要产品拍板的只有一个问题：`cam-001` 和 `CAM-001` 是不是同一件器材？**

| 选项 | 做法 | 影响 |
|---|---|---|
| **A（建议）** 只要长度/控制字符合格就收，大小写视为不同 | 零 DDL 变更 | 用户输入 `cam-001` 会建出**第二件**器材，而它的编号在界面上几乎和第一件一样。**这是一个会真的发生的重复建档**（现场用手机打字，键盘自动小写） |
| B 应用层归一化大写 | 零 DDL 变更，但 DB 仍允许 `cam-001` 从直接写库进来 | 挡住了 App 路径的真实事故；堵不住运维 SQL |
| C 加函数唯一索引 `UNIQUE INDEX ON equipment (workspace_id, lower(code))` | **一次 DDL 变更**（本文件不代为决定） | 最彻底，且与 `app_user.username` 的小写归一化（§1.1）语义一致 |

**建议 B（现在）+ 把 C 记进 §5.3 G9。** 如果产品希望 B 和 C 一致，那就是一次 `0002_*.sql` 迁移，要和 G1–G8 一起做。

### 3.5 次级待定项

不属于上面四条，但同样需要一句话确认：

| 项 | 问题 | 建议 |
|---|---|---|
| 申请能否选非 `IN_STOCK` 的器材 | 主稿 §5.2 只说"已被通过预留的器材不能出库"。维修中/停用中的器材能不能**提交**申请？ | 允许提交（申请人不知道器材现状），审批时按状态与档期决定。**但已出售与已归档的必须挡在提交前** |
| 有未关闭维修记录时能否归档 | 主稿 §3.4 只写了"出库中禁止归档" | 建议同时禁止。归档的语义是"录错了要藏起来"，一件还在维修厂的东西被藏起来会让对账任务失去线索 |
| 唯一性冲突的状态码 | 主稿 §7.3 的表里没有"唯一性冲突"这一行 | 建议 **409**（语义与"档期冲突"同类：请求与现有状态冲突，用户要改的是输入值而不是重试）。若产品希望归入"表单填错"，用 **400** 并复用 `VALIDATION` 码 |
| 批量操作的上界 | 主稿没有给 | 建议 100，与列表 `limit` 一致（§1.7 的依据） |

---

## 4. `garde` 的用法边界

### 4.1 管线：归一化必须在校验之前

```
JSON bytes
  → serde 反序列化        （类型错了就在这里 400：字符串字段收到数字、枚举值不认识、日期格式不对）
  → normalize()          ← 【必须在这里】trim / 全角转半角 / 大小写 / \r\n → \n / trim 后判空
  → garde::Validate      （单字段、无 I/O 的规则）
  → domain 纯函数        （跨字段，见 §4.3）
  → services + DB 预检查 （需要查库的，在主稿 §5.1 的事务内）
  → 数据库约束            （最后一道，正确性）
```

**为什么归一化必须在 `garde` 之前**：`garde` 校验的是它拿到的值。`"   "` 有三个字符，所以 `length(min = 1)` 会**通过**它，然后一条三空格的器材名进库。同类问题还有：`cam`（前缀）会撞 DDL 的 `23514` → 500；`２０２４-０３-０１`（中文输入法的全角数字）会解析失败；`¥1,280.00` 直接进 `price_cents` 会反序列化失败。

落地形状（建议新增一个 extractor，Rust 实现稿 §6 的 handler 从 `Json(cmd)` 改成 `ValidatedJson(cmd)`）：

```rust
// crates/app/src/api/extract.rs
pub struct ValidatedJson<T>(pub T);

pub trait Normalize { fn normalize(&mut self); }

impl<T, S> axum::extract::FromRequest<S> for ValidatedJson<T>
where
    T: serde::de::DeserializeOwned + garde::Validate + Normalize,
    S: Send + Sync,
{
    type Rejection = AppError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(mut v) = Json::<T>::from_request(req, state).await
            .map_err(|e| AppError::Validation(ValidationError::from_json(e)))?;
        v.normalize();                                     // ← 必须在 validate 之前
        v.validate(&())                                    // garde 0.20 的签名带 &Self::Context
            .map_err(ValidationError::from_report)?;
        Ok(Self(v))
    }
}
```

`Normalize` 的三条实现约定：

| 约定 | 为什么 |
|---|---|
| 全角 → 半角**只对结构化字段**（编号、前缀、日期、登录名、数字），**不碰自由文本** | NFKC 会把备注里的全角标点也改掉，那是用户的排版 |
| `\r\n` → `\n`，并且去掉首尾空白 | 手机输入法与 Excel 粘贴都会带 `\r`，它会让"打印一行的字段"多出一个不可见字符 |
| **空串清空为 `None`**（对可空字段） | 手机表单里"清空一个输入框"提交的是 `""` 而不是 `null`。不转的话，`brand = ""` 和 `brand = null` 会变成两种不同的库内状态，而 `brand` 的语义只有一个（主稿 §7.6 的三态里"清空"是 `null`） |

### 4.2 用 derive 表达的规则（单字段、无 I/O、只看值本身）

| 规则 | `garde` 属性 | 适用范围 |
|---|---|---|
| 必填且非空 | `#[garde(length(min = 1, max = N))]` | 所有 `NOT NULL` 的 `string` |
| 长度上下限 | `#[garde(length(min = a, max = b))]` | 同上 |
| 数值闭区间 | `#[garde(range(min = a, max = b))]` | `price_cents`、`cost_cents` |
| 可空字段的内层校验 | `#[garde(inner(length(min = 1, max = N)))]` | `brand: Option<String>` |
| 集合非空与长度 | `#[garde(length(min = 1, max = 100))]` | `equipment_ids: Vec<i64>` |
| 嵌套结构的校验 | `#[garde(dive)]` | 请求体里的 `Vec<RequestItemCmd>` |
| 手写校验的字段 | `#[garde(skip)]` | 否则 derive 会要求它套一个内建校验器 |
| 正则/格式 | **一律写成 `#[garde(custom(f))]`** | `code_prefix`、`username`、`sha256`、`rel_path`。**理由**：`garde` 各版本的内建校验器集合不同，写 `custom` 不依赖版本是否内建了 regex 校验器 |
| 枚举取值 | **不用 `garde`** | 用 serde 反序列化 + `sqlx::Type`（Rust 实现稿 §1）。**这是最好的校验：非法值连类型都构造不出来**，比运行时报错早一步 |

**不建议用 `Option<Option<T>>` 上的 `inner(...)`。** 主稿 §7.6 的三态字段（`expected_date`、`purchase_date` 等）是**两层** `Option`，而 `garde` 的 `inner` 只穿透一层容器。三态字段的校验一律走手写函数（下面 §4.4），并且**三种态都要有单测**。

### 4.3 必须手写的规则（以及手写在哪一层）

| 类别 | 为什么 `garde` 不行 | 落点 |
|---|---|---|
| **归一化**（§4.1） | 它不是校验，是校验的前置步骤 | `ValidatedJson` extractor / `impl Normalize` |
| **跨字段**（§2 的 1–7、16） | `garde` 的 `custom` 挂在**字段**上，拿不到兄弟字段 | **`domain` 纯函数**。这不是被迫的退路：跨字段规则在服务层的其它入口也要用（例如归还流程要重算日期区间、交付流程要回填项目名），而 `domain` 零 I/O 才能被几百个单测覆盖（Rust 实现稿第 4 节的理由）。若所选 `garde` 版本支持**结构体级** `custom`，可以在 DTO 上也写一份，但**不能只在 DTO 上写** |
| **需要查库**：唯一性（登录名、分类名、分类前缀、器材编号、项目名）、存在性（分类、项目、照片、器材）、工作区一致、同工作区互斥（一条未关闭维修） | 纯函数拿不到 `PgConnection`；而服务层按 Rust 实现稿 §3.1 **只收 `&mut PgConnection`** | `db::<entity>::<check>` 预检查函数，在 `services` 的事务内调用，返回 `DomainError`。**必须与 DB 约束走同一个错误出口**（主稿 §8：「预检查给人话、约束给正确性」） |
| **三态 PATCH 的"可清空"** | 取决于该列是否 `NOT NULL`，是跨字段性质 | 手写函数：`None` 不动 → 不校验；`Some(None)` → 只看该列可不可空；`Some(Some(v))` → 按创建规则校验 |
| **状态机合法性**（五态流转） | 它不是"字段校验"，是领域规则；而且主稿 §7.3 给它的是 **422** 而不是 400 | `domain::state_machine::check`（Rust 实现稿第 4 节），**不要放进 `garde`**——放进去会把一个领域错误降级成 400 |

### 4.4 完整示例：`checkout.expected_date`

选它是因为它同时踩到四个难点：三态 PATCH、跨字段、日期范围、DDL 的 `CHECK`。

```rust
// ---------- ① 请求 DTO（app/api/dto/checkout.rs） ----------
#[derive(Debug, serde::Deserialize, garde::Validate)]
#[serde(rename_all = "camelCase")]          // 不要用 #[serde(rename)]，见 4.5 的字段名推导
pub struct CheckoutUpdateCmd {
    /// 三态（主稿 7.6）：缺省 = 不动；null = 清空；值 = 设置。
    /// 两层 Option → garde 的 inner 穿不透 → 走手写校验。
    /// 注意：请求 DTO 只需要 #[serde(default)]；
    /// skip_serializing_if 属于响应 View（主稿 7.7 明确"不用它表达权限"）。
    #[serde(default)]
    #[garde(skip)]
    pub expected_date: Option<Option<time::Date>>,

    #[serde(default)]
    #[garde(skip)]
    pub checkout_date: Option<time::Date>,

    #[serde(default)]
    #[garde(length(max = 2000))]
    pub note: Option<String>,
}

// ---------- ② 三态 + 跨字段 手写校验（domain，零 I/O） ----------
// crates/domain/src/validate.rs
pub const MAX_DUE_DAYS: i64 = 730;

pub fn check_expected_date(
    checkout_date: Option<time::Date>,   // None = 本次不改出库日期
    expected: &Option<Option<time::Date>>,
    current: Option<time::Date>,         // 库里当前的 expected_date
    current_checkout_date: time::Date,   // 库里当前的 checkout_date
    today: time::Date,
) -> Result<(), DomainError> {
    // 态 1：不动 —— 不校验
    let patch = match expected {
        None => return Ok(()),
        // 态 2：清空 —— 该列可空（DDL: date，无 NOT NULL），合法
        Some(None) => return Ok(()),
        // 态 3：设置
        Some(Some(d)) => Some(*d),
    };
    let Some(due) = patch else { return Ok(()) };

    let base = checkout_date.unwrap_or(current_checkout_date);
    if due < base {
        return Err(DomainError::DueBeforeCheckout);            // 主稿 3.3 的 DB CHECK 的人话版
    }
    if due > today + time::Duration::days(MAX_DUE_DAYS) {
        return Err(DomainError::DueTooFar { max_days: MAX_DUE_DAYS });
    }
    Ok(())
}

// ---------- ③ 文案（domain::error，thiserror 的 Display 就是给用户看的中文） ----------
#[derive(Debug, thiserror::Error)]
pub enum DomainError {
    #[error("预计归还日期不能早于出库日期")]
    DueBeforeCheckout,
    #[error("预计归还日期不能超过今天起 {max_days} 天")]
    DueTooFar { max_days: i64 },
    // …
}
```

**为什么这条规则要放在 `domain` 而不是 DTO 上**：归还流程（主稿 §5.4）与交付流程（主稿 §3.1 的"区间起点改为实际出库日期"）也要算同一件事。写在 DTO 上意味着同一个判断在服务层还要再写一遍，两份必然漂移。

### 4.5 中文文案与 `AppError::Validation`（**这里要对 Rust 实现稿 §5 提一处修订**）

[主稿 §8](../20260930_BACKEND_v1.md) 与 [Rust 实现稿 §5](../20260930_RUST_IMPL_v1.md) 都写了：

```rust
#[error(transparent)] Validation(#[from] garde::Report),
```

两个问题：

1. **`transparent` 会把 `garde::Report` 的 `Display` 原样交给用户**，而那是英文（`length is not in range`）。这与主稿 §7.3「`message` 直接进交接稿第 7 节那条表单内红条」和 Rust 实现稿 §5.2「`Display` 输出直接是给用户看的中文」**直接冲突**。
2. **错误信封里的 `field` 没有落点**。主稿 §7.3 的 `{ code, message, field, requestId }` 要求 `field` 供表单定位输入框，而 `garde::Report` 只带路径，没人把它转成 `field`。

**建议的修订形状**：

```rust
// crates/app/src/api/error.rs
#[derive(Debug, thiserror::Error)]
#[error("{message}")]
pub struct ValidationError {
    /// camelCase 的 API 字段名，填主稿 7.3 的 error.field
    pub field: Option<String>,
    /// 主稿 7.3 的 error.code，取值 VALIDATION
    pub code: &'static str,
    /// 中文，直接进表单内红条
    pub message: String,
    /// 全部错误，**只进日志**，不进响应（保持信封形状不变）
    pub all: Vec<(String, String)>,
}

#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error(transparent)] Validation(#[from] ValidationError),   // ← 形状改变
    #[error(transparent)] Domain(#[from] DomainError),
    // …
}

impl From<garde::Report> for ValidationError {
    fn from(r: garde::Report) -> Self {
        // 每一条错误自带中文：优先用规则上的 message 属性；
        // 没写 message 的（内建校验器），走 zh_fallback() 兜底。
        let all: Vec<(String, String)> = r.iter()
            .map(|(path, e)| (snake_to_camel(path), zh_message(path, e)))
            .collect();
        // 取第一条：顺序必须确定（否则同一次提交两次请求可能给不同的红条）
        let (field, message) = all.first().cloned()
            .unwrap_or_else(|| (String::new(), "提交的内容不合法".to_string()));
        Self { field: Some(field), code: "VALIDATION", message, all }
    }
}
```

三条实现约定：

| 约定 | 理由 |
|---|---|
| **中文文案与规则同行**：优先在规则上写 `message`（`#[garde(length(min = 1, max = 60, message = "器材名称不能为空或超过 60 个字符"))]`）。若所选 `garde` 版本不支持规则级 `message`，则**该字段的规则一律改用 `custom`**，函数返回 `garde::Error::new("…")` | 避免维护一张 `(字段 × 规则) → 文案` 的映射表——那种表会和 DTO 漂移，而漂移的表现是"用户看到英文" |
| `field` 由 `snake_to_camel(path)` 推导 | `garde` 的路径是 **Rust 字段名**（`expected_date`），而线上契约是 camelCase（主稿 §7.2）。所以**禁止对参与校验的字段写 `#[serde(rename = "...")]`**——只用 `rename_all = "camelCase"`，两者的换算才是可推导的 |
| **只返回第一条，全部错误进日志** | 主稿 §7.3 的信封只有一个 `message`。返回第一条时必须**顺序确定**（若 `garde` 的顺序不保证，就在 `From` 里按固定字段优先级排序） |

**兜底映射 + 元测试（这一对的组合才是完整的）**：`zh_message` 对认不出来的错误返回 `"提交的内容不合法"`（绝不返回英文）。**但不能靠它兜着**——加一条元测试：

```rust
#[test]
fn every_validation_rule_has_a_chinese_message() {
    // 对每个请求 DTO 的每个字段，用"必然违反该规则"的值构造一条命令，跑一次校验，
    // 断言 error.message 只含中文字符、数字和中文标点，**不含 a-zA-Z 单词**
    // （英文的 field 名允许，但 "length" / "required" / "range" 这类词出现即失败）
}
```

这是本项目"元测试"传统的延续（运行时篇 R3.2 #9、日志设计 §8.3）：**它测的不是功能，是"约定没被破坏"。** 文档会被跳过，测试不会。

### 4.6 修 SQLSTATE 翻译表（与校验直接相关，见 §5.3 G13）

主稿 §8 与 Rust 实现稿 §5.1 只翻译了 `23P01`（排他约束）和 `23505`（唯一约束）。**`23514`（`check_violation`）没有翻译**，于是"应用层忘了校验、数据库的 `CHECK` 拦住"的每一条都返回 **500**：

| 能被用户输入触发的 `CHECK` | 触发方式 | 现在的结果 |
|---|---|---|
| `category_code_prefix_check` | 用户填 `cam`（小写） | 500 |
| `request_use_end_check` | 跳过应用层校验（抓包改请求） | 500 |
| `checkout_expected_date_check` | 同上 | 500 |
| `app_user_owner_not_disabled` | 停用主理人 | 500 |
| `maintenance_outcome_check` | 关闭维修时传非法 outcome | 500 |

这三条要一起改（**建议，不动现有文档**）：

1. **翻译里补 `23514`**：`Some("23514") => AppError::Validation(...)`（或按 `db.constraint()` 映射到对应的领域错误），并附中文文案。
2. **补 `23503`（`foreign_key_violation`）**：分类/项目被删后引用它 → 当前也是 500。文案：`所选分类不存在`。
3. **主稿第 8 节的 `DomainError` 枚举里没有 `Conflict`**，但同一节的翻译代码返回 `DomainError::Conflict`；`DuplicateCode { code }` 在翻译点拿不到编号（Rust 实现稿 §5.1 传了 `String::new()`，文案会变成"编号已存在："半句话）。前者补变体，后者的 `code` 由**预检查**填，翻译点用兜底文案。

**为什么这不属于"悄悄改 DDL"**：它改的是错误映射，不是约束。但它是本提案里**优先级最高的一条**——主稿 §8 那句"约束给正确性"要成立，前提是约束的违例也能翻译成人话；`23514` 不翻译，这句话就有五个反例。

---

## 5. 与数据库约束的分工表

### 5.1 一句话原则

> **应用层的校验负责"用户看得懂的拒绝"（400 + 中文 + 能定位到哪个输入框），数据库的约束负责"不可能存在的事实"（正确性，与谁写的代码无关）。两边都表达同一件事时，它们必须走同一个错误出口**（主稿 §8：预检查给人话、约束给正确性）。

三条推论：

1. **凡是数据库能表达、且表达后不会让改产品决策变成迁移的规则，它必须同时出现在两边**（必填、唯一、日期区间、枚举取值）。少一边就有一种失败形态：只有 DB → 用户看到 500/23502；只有应用层 → 绕过应用的写路径（运维 SQL、`pg_dump` 恢复、将来的第二个服务）能写坏数据。
2. **凡是数据库表达不了或不该表达的（长度上限、金额范围、控制字符、跨字段、需要查库的），一律只在应用层**。特别是**长度上限**：把它落成 `CHECK (char_length(name) <= 60)`，意味着本文档第 3 节的任何一个待定项被产品改掉，都要发一次迁移。
3. **凡是"应用层不可能可靠做到"的，一律只由数据库保证**，应用层只做预检查给人话：`occupancy` 的区间互斥（并发）、`app_user_one_owner`（唯一主理人）、`maintenance_one_open`（一条未关闭维修）。这三条是 Rust 实现稿 §5 里 `23P01`/`23505` 翻译存在的全部理由。

### 5.2 对照表

| 规则 | 只在应用层 | 只在数据库 | 两边都有 | 为什么 |
|---|---|---|---|---|
| 字段必填（`NOT NULL` 列） | | | ✅ | DB：`NOT NULL` 是"不可能为空"的事实保证。应用层：`garde` 给中文（`器材名称不能为空`），DB 给的是 `23502` → 500 |
| 长度上限（全部 `string`） | ✅ | | | DB 的 `text` 无长度约束，而**上限是产品会调的**（§3.3）。落成 `CHECK` 会让每次调值都变成一次迁移。真实防线是请求体上限（运行时篇 R5.4，5 MiB）+ 应用层上限 |
| 金额范围（`>= 0`、上限） | ✅ | | | 同长度上限；且 DDL 目前连 `>= 0` 都没有（G2） |
| 日期范围（`purchase_date <= today` 等） | ✅ | | | 依赖 `business_today()`（运行时篇 R2.3），静态 `CHECK` 表达不了 |
| 日期格式 `YYYY-MM-DD` | ✅ | | | 由 serde + `time` 的格式描述符承担（§4 的一个坑：`time::Date` 的默认 serde 是 `[year, ordinal]` 数组，必须用 `time::serde::format_description!` 显式声明） |
| 控制字符（编号、名称、单行文本） | ✅ | | | DB 无法枚举控制字符（可以写正则，但正则进 DDL 就是"改产品决策要迁移"的同一类问题） |
| 枚举取值（`status`、`role`、`project_type`、`outcome`） | | | ✅ | DB：DDL `CHECK`。应用层：serde 反序列化（非法值**连类型都构造不出来**，Rust 实现稿 §1 的 `sqlx::Type` + `SCREAMING_SNAKE` 是同一个真理的第三处）。**两边都要，因为它们的失败形态完全不同**：serde 给的是"字段取值不合法"，绕过应用直接写库的是 DB 拦住 |
| 唯一性（登录名、分类名、分类前缀、器材编号、项目名） | | | ✅ | 主稿 §3.5 末句的模式：预检查给人话（`已存在这个登录名` + 定位到输入框），`23505` 保正确性（并发下两个请求都通过预检查）。**只有预检查会撞车，只有约束会给 500** |
| 跨工作区一致（`category_id` 指向别的 workspace） | ✅ | | | 主稿 D7 把 `workspace_id` 放进了所有表，但**没有一处约束它一致**（G7）。落成复合外键是一次 DDL 变更 |
| `use_end >= use_start`（`request`） | | | ✅ | DDL 已有 `CHECK`；应用层必须也有，因为撞 `CHECK` 是 `23514` → 当前会 500（G13），而且它给不出 `field` |
| `expected_date >= checkout_date`（`checkout`） | | | ✅ | 同上 |
| `project.end_date >= start_date` | ✅ | | | **DDL 没有这条**（G1），而 `request`/`checkout` 都有——`project` 是漏的 |
| `maintenance` 的 `outcome`/`closed_at` 联动 | ✅ | | | **DDL 没有**（G3、G4） |
| 三态 PATCH 的"可清空" | ✅ | | | 它是"列是否 `NOT NULL`"的镜像，不是独立约束；`NOT NULL` 已经在 DB |
| `equipment.code` 大小写唯一 | ✅ | | | §3.4 的选项 B；选项 C 需要 DDL 变更（G9） |
| 区间互斥（`occupancy`） | | | ✅ | **只由 DB 保证正确性**（`EXCLUDE USING gist`），应用层的 `find_conflict` 纯粹为了给人话（主稿 §3.2）。这是全系统唯一一条"应用层只能尽量保证"的规则（主稿 C2），也是选 Postgres 的唯一硬理由 |
| 唯一主理人（`app_user_one_owner`） | | | ✅ | 只由 DB 保证（主稿 C5：`不是"业务上应该只有一个"，是数据库不允许多个`）。应用层的 `Actor::can_change_role` 给人话 |
| 一条未关闭维修（`maintenance_one_open`） | | | ✅ | 只由 DB 保证；应用层预检查给"这件器材已有一条未完成的维修记录" |
| 外键存在性（分类、项目、照片、器材） | | ✅ | | 应用层的存在性检查其实是**为了给人话和同工作区判断**，不是为了防止悬空引用——那由 FK 保证。**但 `23503` 没翻译，现在撞上它是 500**（G13） |
| `created_at` / `updated_at` 的取值 | ✅ | | | DB 只有 `DEFAULT now()`，**Postgres 没有 `ON UPDATE`**：任何 UPDATE 必须显式 `SET updated_at = now()`，否则它永远停在创建时间（G11）。这是一条**只在应用层、且很容易漏**的规则，建议配一条测试 |
| `password_hash` 的格式 | | ✅ | | 应用层不校验、不解析它；它由 `argon2` 写、由 `argon2` 读 |
| 状态流转合法性（五态） | ✅ | | | `domain::state_machine` 是唯一落点（主稿 C1）。DB 的 `CHECK` 只保证"状态值在集合内"，不保证"这个迁移合法"——**两者不是同一件事**，不要因为 DB 有 `CHECK` 就以为状态机在 DB 里 |
| 归档与状态的关系（出库中禁止归档） | ✅ | | | **DDL 没有**（G8）。主稿 §3.4 只把这条写成了 `DomainError::CheckedOutNotArchivable` |

**读这张表的方法**：`只在应用层` 那一列里的每一条，都对应 §5.3 的一个缺口编号或一条"DB 表达不了"的理由；`只在数据库` 那一列里只有四条，它们全部是"应用层做不到"的机制；其余都是"两边都要"——而两边都要的那些，**翻译表（`23P01`/`23505`/`23514`/`23503`）是它们共用同一个出口的物理实现**，翻译表缺一项，这个分工就有一个洞。

### 5.3 DDL 缺口与冲突清单（**本文件不代改 DDL，只点名**）

以下是逐条对照[主稿 §3.3](../20260930_BACKEND_v1.md) 的完整 DDL 得到的。**没有一条建议在本文件里被实施**；标为"建议补"的，若要补，落成 `migrations/0002_validation_gaps.sql`，并且**同时补 SQLSTATE 翻译**（否则补了约束只是把 400 变成 500）。

| # | 位置 | 缺什么 / 冲突 | 后果 | 建议 |
|---|---|---|---|---|
| **G1** | `project` | 无 `CHECK (start_date IS NULL OR end_date IS NULL OR end_date >= start_date)` | 项目可以"结束早于开始"，而 `request`（`CHECK (use_end >= use_start)`）和 `checkout`（`CHECK (expected_date IS NULL OR expected_date >= checkout_date)`）都有同类约束，只有 `project` 漏了 | 应用层必检（§2 第 4 条）；**建议补 DDL** 以与另两张表一致 |
| **G2** | `equipment.price_cents`、`maintenance.cost_cents` | 无 `CHECK (… >= 0)` | 负价格/负费用能进库 | 应用层必检；DDL 可补可不补（补了要一并补 `23514` 翻译） |
| **G3** | `maintenance` | 无「`closed_at` 非空 ⟹ `outcome` 非空」 | 一条已关闭却没有结论的维修记录；占用对账与统计会漏 | 应用层必检（§2 第 7 条）；**建议补** `CHECK (closed_at IS NULL OR outcome IS NOT NULL)` |
| **G4** | `maintenance` | 无「`closed_at::date >= sent_date`」 | 维修完成早于送修 | **建议留在应用层**：`closed_at` 是 `timestamptz`、`sent_date` 是 `date`，比较要经过业务时区（运行时篇 R2.3），塞进 CHECK 会把时区写死在 DDL 里 |
| **G5** | `category.next_seq` | 无 `CHECK (next_seq >= 1)` | 手工改成 0 会生成 `CAM-0` | 非用户输入路径，可不补 |
| **G6** | `photo.width` / `height` / `bytes` | 无 `> 0` | 只在服务端算出来，风险低 | 可不补 |
| **G7** | **所有 `workspace_id` 外键** | 都是**单列**外键。`equipment.category_id` 可以指向**另一个 workspace** 的分类；`request_item.equipment_id`、`checkout_item.equipment_id`、`maintenance.equipment_id` 同理 | 主稿 D7 说 `workspace_id` 从第一天进所有表（为交接稿第 11 节的预留位），但**没有任何一处约束它一致**。今天只有一个 workspace 所以无害，第一个多 workspace 的写路径就是串数据 | 应用层必检。要落成约束需要复合外键 `FOREIGN KEY (workspace_id, category_id) REFERENCES category (workspace_id, id)`，而 `category` 需要 `UNIQUE (workspace_id, id)` —— **这是一次跨所有表的 DDL 变更，本文件不代为决定**。**现在决定成本最低（还没有数据）**。这也是本清单里**唯一一条需要动所有表**的缺口，建议列为该条未决项的第一条 |
| **G8** | `equipment.archived_at` | 无 CHECK 关联 `status` | 「出库中禁止归档」（主稿 §3.4）只在应用层 | 应用层必检（§2 第 17 条）；**建议补** `CHECK (archived_at IS NULL OR status <> 'CHECKED_OUT')` |
| **G9** | `equipment.code` 的 `UNIQUE (workspace_id, code)` | 该唯一约束**区分大小写**（`text` 比较） | 用户输入 `cam-001` 会建出第二件几乎同名的器材（§3.4） | 见 §3.4 的三个选项；选项 C 是一次 DDL 变更 |
| **G10** | `occupancy` | 无「`kind='RESERVATION'` ⟹ `source_request_id IS NOT NULL`」/「`kind='CHECKOUT'` ⟹ `source_checkout_id` 非空」 | 无法从 schema 判断占用的来源 | **有意留白，不改**：占用对账任务（主稿第 11 节）查的正是"该有的行是不是没了、不该有的行是不是还在" |
| **G11** | `app_user` / `equipment` / `project` / `workspace_setting` 的 `updated_at` | 只有 `DEFAULT now()`。**Postgres 没有 MySQL 的 `ON UPDATE CURRENT_TIMESTAMP`** | 任何 UPDATE 忘记 `SET updated_at = now()`，这个字段就永远停在创建时间——而主稿 §5.5 的字段编辑正是走 UPDATE，且 `EquipmentAdmin` 会把它下发（Rust 实现稿 §2） | **写进 Rust 实现稿的 SQL 约定**（每条 UPDATE 显式赋值）+ 一条测试（编辑字段后 `updated_at` 必须变大）。**这条最容易漏，因为它不会报错** |
| **G12** | `request.status` / `checkout.status` / `equipment.status` | 与其它列的联动无 CHECK | 状态机只在应用层 | 有意留白（主稿 C1：状态机有唯一落点） |
| **G13** | **SQLSTATE 翻译表**（主稿 §8、Rust 实现稿 §5.1） | 只翻译 `23P01` 和 `23505`。**`23514`（`check_violation`）与 `23503`（`foreign_key_violation`）没有翻译** | §4.6 列出了**五个能被用户输入触发的 `CHECK`**，它们全部返回 **500**。这与主稿 §8「约束给正确性、两者走同一个出口」直接冲突——出口只修了两条路 | **建议补**，见 §4.6。这是本提案里优先级最高的一条 |
| **G14** | `DomainError`（主稿 §8） | ① 枚举里**没有 `Conflict`**，但同一节的翻译代码 `return AppError::Domain(DomainError::Conflict)`；② `DuplicateCode { code }` 在翻译点拿不到编号，Rust 实现稿 §5.1 传了 `String::new()` | ① 是编译不过的笔误；② 用户看到半句话"编号已存在：" | 补 `Conflict` 变体；`DuplicateCode` 的 `code` 由**预检查**填，翻译点用兜底文案（`编号已存在，请换一个`） |
| **G15** | `time::Date` 的 serde | 主稿 §7.2 要求业务日期是 `"2026-10-01"`。`time` 的 `serde` feature **默认把 `Date` 序列化成 `[year, ordinal]` 数组**（`[2026, 274]`），不是字符串 | 每个日期字段都会以数组形式出现在 JSON 里，而前端按字符串解析 → 静默不一致（`gen-api` 生成的类型也会是数组，但主稿 §7.2 说的是字符串） | 用 `time::serde::format_description!(biz_date, Date, "[year]-[month]-[day]")` 显式声明，每个日期字段 `#[serde(with = "biz_date")]`。**建议加一条契约测试**：断言日期字段在 JSON 里是 `"2026-10-01"` 这样的字符串 |
| **G16** | 校验常量 | 全系统没有一处定义长度/范围的常量位置 | 上限会散落在各 handler 的 `#[garde(...)]` 里，改一个值要 grep | **建议**：所有上限收敛到 `domain::limits`（`MAX_NAME = 60`、`MAX_LONG_TEXT = 2000`、`MAX_PRICE_CENTS = 10_000_000_000`、`MAX_BATCH = 100`…），`garde` 属性里也用这些常量。这是"少一个要记的数字就少一处漂移" |

**关于"要不要把长度上限也写进 DDL"**：**建议不写。** 三个理由：① 长度上限是产品会调的量（本文档第 3 节正是把它们标为待定），落成 `CHECK` 意味着每次调值发一次迁移；② `text` 在 Postgres 里没有实际的溢出风险；③ 真实防线是运行时篇 R5.4 的**请求体上限（5 MiB，在读流时生效）**+ 本文档的应用层上限，两者合起来已经覆盖了"一个 1 MB 的备注"这个场景。**如果将来确实要下沉到 DB**，用 `CHECK (char_length(name) <= 60)`，并且**必须同时补 `23514` 的翻译**（G13），否则用户看到的是 500。
