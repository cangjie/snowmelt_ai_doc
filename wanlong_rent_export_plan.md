# 万龙租赁订单导出 v2（按你的新字段需求）

> 保存日期：2026-05-13
> 状态：待执行

## Context
之前导出的 `rent_orders_2025-10-15_2026-04-15.xlsx` 是全店铺全字段，本轮要按你新需求重做：只导**万龙体验中心**、**精简字段**、**含退款/超时/赔偿**。

## 范围
- 店铺：`shop = N'万龙体验中心'`（其他店铺不导）
- 时间：`create_date >= '2025-10-15' AND create_date < '2026-04-16'`
- 类型：`type = N'租赁' AND valid = 1`（不含接待中草稿）
- 实测命中 **2325 个订单**，rental_detail 17287 行租金 / 48 行超时 / 4 行赔偿

## 关键查询规则（已 readonly 验证）
- **支付总额** = `SUM(order_payment.amount)` WHERE `status=N'支付成功' AND valid=1`
- **退款总额** = `SUM(payment_refund.amount)` WHERE `state=1 OR refund_id 非空非空字符串`
  - 仅 `state=1` 漏掉 538 万（多数已发起退款只回写了 refund_id，state 未及时回调）
  - 旧代码 `Models/Rent/RentOrder.cs:519` 也用此条件，与之一致
  - 万龙范围实测此条件下退款总额 **¥6,604,799.33**
- **结算日期** = `MAX(payment_refund.create_date)` 同条件，按订单聚合（无退款则 NULL）
- **租金/超时费/损坏赔偿** = `rental_detail.charge_type` 三种值（中文 `租金` / `超时费` / `赔偿金`），按 rental 分组求和

## 主表字段（每订单一行）
| 列 | SQL 来源 |
|---|---|
| 订单号 | `[order].code` |
| 业务日期 | `[order].biz_date` |
| 结算日期 | `MAX(payment_refund.create_date)` 同退款条件 |
| 支付总金额 | 见上 |
| 退款总金额 | 见上 |
| 订单结余 | 支付总金额 − 退款总金额 |
| 店员姓名 | `staff.name` LEFT JOIN `[order].staff_id` |

## 明细表字段（每 rental 一行）
| 列 | SQL 来源 |
|---|---|
| 订单号 | `[order].code` |
| 租赁商品名称 | `rental.name` |
| 起租日期 | `rental.start_date` |
| 退租日期 | `rental.end_date` |
| 租金总额 | `SUM(rental_detail.amount WHERE charge_type=N'租金' AND valid=1)` |
| 超时费 | `SUM(rental_detail.amount WHERE charge_type=N'超时费' AND valid=1)` |
| 损坏赔偿 | `SUM(rental_detail.amount WHERE charge_type=N'赔偿金' AND valid=1)` |

## 实施
改写 `D:\snowmeet\snowmeet_ai_doc\export_rent_orders.py`（沿用现有结构，替换两个 SQL + 输出文件名）：
- 输出：`D:\snowmeet\snowmeet_ai_doc\wanlong_rent_orders_2025-10-15_2026-04-15.xlsx`
- 两个 sheet：`订单汇总` + `订单明细`
- 数据库连接 / openpyxl 写入逻辑保留不动

## 验证
跑完后用 readonly Python 打开 xlsx 校对：
1. 主表行数 ≈ 2325（万龙 valid=1 订单数）
2. 主表「支付总金额」列求和应 ≈ ¥7,204,721.30（已 readonly 验证）
3. 主表「退款总金额」列求和应 ≈ ¥6,604,799.33（已 readonly 验证）
4. 明细「租金总额」全列求和应 ≈ ¥2,503,585.18
5. 明细「超时费」全列求和 ≈ ¥4,630.00
6. 明细「损坏赔偿」全列求和 ≈ ¥300.00
7. 任取 1-2 个有退款的订单（如 71518/71593）人工对比 SQL 直查结果

## 相关文件
- 数据库：`100.28.143.19:1433` `snowmeet_new`（账号见环境）
- 旧脚本（可参考）：`D:\snowmeet\snowmeet_ai_doc\export_rent_orders.py`
- 旧产出（可对照）：`D:\snowmeet\snowmeet_ai_doc\rent_orders_2025-10-15_2026-04-15.xlsx`（全店全字段）
- 关联 PRD：`D:\snowmeet\snowmeet_ai_doc\PRD.docx`
- 身份验证 plan（独立任务）：`D:\snowmeet\snowmeet_ai_doc\payment_identity_verification_plan.md`
