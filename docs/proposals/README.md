# ⚠️ 已并入 —— 本目录只作审计留底

**这七份提案的内容已经合并进 `docs/` 下的正式文档。实现时不要读这里。**

保留它们的目的是**可追溯**：合并过程中做过的判断、被否决的选项、以及每条结论的来源，都能在这里查到。

## 映射：提案 → 现在在哪

| 提案 | 内容现于 |
|---|---|
| [01_api_inventory.md](01_api_inventory.md) | [实现契约](../20260930_IMPL_CONTRACT_v1.md) §1 |
| [02_validation_rules.md](02_validation_rules.md) | [实现契约](../20260930_IMPL_CONTRACT_v1.md) §2；DDL 缺口进了[主稿](../20260930_BACKEND_v1.md) 3.6–3.8 与第 8 节 |
| [03_retry_policy.md](03_retry_policy.md) | [实现契约](../20260930_IMPL_CONTRACT_v1.md) §4 |
| [04_pool_and_timeouts.md](04_pool_and_timeouts.md) | [实现契约](../20260930_IMPL_CONTRACT_v1.md) §3 |
| [05_owner_toggles.md](05_owner_toggles.md) | [实现契约](../20260930_IMPL_CONTRACT_v1.md) §2.3.7；决策进[待拍板清单](../20260930_OPEN_DECISIONS_v1.md) D-06…D-09 |
| [06_request_rejection.md](06_request_rejection.md) | [实现契约](../20260930_IMPL_CONTRACT_v1.md) §2.3.4；决策进[待拍板清单](../20260930_OPEN_DECISIONS_v1.md) D-10…D-14 |
| [07_cascade_rules.md](07_cascade_rules.md) | [主稿](../20260930_BACKEND_v1.md) 3.6–3.8 的 DDL；技术决定 T-01/T-02 进[待拍板清单](../20260930_OPEN_DECISIONS_v1.md) |

## 合并时的两处改动（提案里写的是旧值）

1. **端点命名风格**：提案 01 §0.1 论证并采用嵌套式。**已裁定为「A 方案」**——单件用嵌套（`/equipment/{id}/maintenance`），批量用集合（`/equipment/batch`），**出库是集合式 `POST /api/v1/checkouts`**（一次出库天生可带多件，资源是 `checkout` 而不是"某件器材的出库动作"）。实现契约 §1.0 是准的，提案 01 里凡把出库写成 `/api/v1/checkouts` 的地方与之一致。
2. **`Idempotency-Key` 从三处扩到四处**：加了建档（提案 03 §4.3 的发现——编号留空时客户端重发会静默建出两条且不撞唯一约束）。

## 提案里明确"待验证、未实测"的项

七份提案都诚实标注了验证边界（写作时本机没有 `cargo`、`psql`、Docker）。那些项**没有**被合并进正式文档当作结论，而是汇总在[实现契约](../20260930_IMPL_CONTRACT_v1.md) §6「待验证」里，共 11 条。实现第一步应逐条跑掉。
