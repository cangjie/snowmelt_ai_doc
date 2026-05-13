---
name: export_rent_order
description: 按店铺+日期范围从生产 SQL Server 导出租赁订单 xlsx（订单汇总 / 订单明细 / 支付明细 3 个 sheet），含测试单标记、临时订单分类、订单结余 vs rental 明细实付对账标红。触发场景：「导出 xxx 店铺租赁订单」「按之前做法重导一份」「导一份 xx 店铺 xx 时间段的租赁数据」等。
---

# Export Rent Order Skill

按店铺名 + 日期范围从生产 SQL Server (`100.28.143.19/snowmeet_new`) 导出租赁订单到 xlsx，三个 sheet 一次生成。设计目标是**换机器后能立刻跑通**，因此环境依赖、SQL、后处理逻辑都在这个目录内。

## 何时触发

- 用户说"导出 [店铺] 租赁订单"、"按上次做法重新导一份"、"导一份 xx 时间段的租赁数据"
- 用户提到具体店铺（如"万龙体验中心 / 渔阳 / 南山 / 怀北 / 崇礼旗舰店 / 万龙服务中心"）+ 时间范围且要 xlsx
- 用户说"对账"、"租赁报表 xlsx"、"租赁明细+支付明细"等

## 一次性环境准备（每台新机器）

```bash
# macOS
brew install unixodbc msodbcsql18
pip3 install pyodbc openpyxl

# 让 pyodbc 看到驱动（brew 的 odbcinst.ini 在 /opt/homebrew/etc，但 unixODBC 默认查 /etc）
export ODBCSYSINI=/opt/homebrew/etc
```

验证：
```bash
ODBCSYSINI=/opt/homebrew/etc python3 -c "import pyodbc; print(pyodbc.drivers())"
# 期望输出包含 'ODBC Driver 18 for SQL Server'
```

> Linux 在 `/etc/odbcinst.ini` 直接注册即可，无需 `ODBCSYSINI`。Windows 上 brew 不适用，需另装 [Microsoft ODBC Driver 18](https://learn.microsoft.com/en-us/sql/connect/odbc/download-odbc-driver-for-sql-server)。

## 调用方式

通用：
```bash
cd /path/to/snowmeet_ai_doc/skills/export_rent_order
ODBCSYSINI=/opt/homebrew/etc python3 export_rent_orders.py \
    --shop 万龙体验中心 \
    --start 2025-10-15 \
    --end 2026-04-15
```

输出文件名默认：`{shop_prefix}_rent_orders_{start}_{end}.xlsx`，写到**当前工作目录**。已知店铺到 prefix 的映射在脚本顶部 `SHOP_PREFIX`：

| shop（DB order.shop） | prefix |
|---|---|
| 万龙体验中心 | `wanlong` |
| 万龙服务中心 | `wanlong_service` |
| 渔阳 | `yuyang` |
| 南山 | `nanshan` |
| 怀北 | `huaibei` |
| 崇礼旗舰店 | `chongli` |

未在映射表内的店铺会直接用 shop 中文名作为前缀，文件名含中文（macOS/Linux 文件系统支持）。新店铺如要规范英文名，往 `SHOP_PREFIX` 加一项即可。

常用参数：

```
--shop          必填，店铺中文名（DB order.shop 字段值）
--start         必填，起始日期（inclusive），格式 YYYY-MM-DD
--end           必填，截止日期（inclusive），格式 YYYY-MM-DD
--out           可选，输出 xlsx 完整路径；不传走默认命名
--conn          可选，ODBC 连接串；不传连生产 100.28.143.19
--no-postprocess 跳过对账后处理（不加「临时订单」列、不标红）
```

例：
```bash
# 万龙体验中心半年数据，默认命名
python3 export_rent_orders.py --shop 万龙体验中心 --start 2025-10-15 --end 2026-04-15

# 渔阳一季度，指定输出
python3 export_rent_orders.py --shop 渔阳 --start 2026-01-01 --end 2026-03-31 \
    --out ~/Desktop/yuyang_q1.xlsx

# 跳过后处理（只要原始 3 sheet）
python3 export_rent_orders.py --shop 南山 --start 2026-02-01 --end 2026-02-28 \
    --no-postprocess
```

## 输出结构

### Sheet 1：订单汇总（11 列）

| # | 列 | 来源 |
|---|---|---|
| 1 | 订单号 | `order.code` |
| 2 | 业务日期 | `CAST(order.biz_date AS DATE)` |
| 3 | 业务时间 | `CONVERT(VARCHAR(8), order.biz_date, 108)` |
| 4 | 结算日期 | 最后一笔有效退款 `MAX(payment_refund.create_date)` 当天 |
| 5 | 结算时间 | 同上 时分秒 |
| 6 | 支付总金额 | `SUM(order_payment.amount WHERE status='支付成功' AND valid=1)` |
| 7 | 退款总金额 | `SUM(payment_refund.amount WHERE state=1 OR refund_id 非空)` |
| 8 | 订单结余 | 支付 − 退款 |
| 9 | 店员姓名 | `staff.name` JOIN |
| 10 | 测试 | 支付总额 < 5 **OR** 店员姓名含 "苍" → '是' |
| 11 | 临时订单 | 非测试 + 结余>0 + 订单明细无非测试 rental 行 → '是'（后处理写入） |

订单号 cell 若被 **浅红底（FFC7CE）+ 深红字（C00000）** 标记，含义：非测试 + 订单结余 ≠ 订单明细该订单的非测试实付合计（差额 ≥ 0.01）。这是真正需要关注的对账异常。

### Sheet 2：订单明细（15 列，按 rental 一行）

| # | 列 | 取值 |
|---|---|---|
| 1 | 订单号 | `order.code` |
| 2 | 租赁商品名称 | `rental.name` |
| 3-6 | 起租/退租 日期/时间 | `rental.start_date`/`end_date` 拆分 |
| 7 | 租金总额 | `SUM(rental_detail.amount WHERE charge_type='租金' AND valid=1)` |
| 8 | 是否招待 | `rental.entertain=1` → '是' |
| 9 | 是否体验 | `rental.experience=1` → '是' |
| 10 | 应付租金 | 招待或体验 → 0，否则 = 租金总额 |
| 11 | 减免金额 | 见下 |
| 12 | 超时费 | `SUM(rental_detail.amount WHERE charge_type='超时费' AND valid=1)` |
| 13 | 损毁赔偿 | `SUM(rental_detail.amount WHERE charge_type IN ('赔偿金','损坏赔偿') AND valid=1)` |
| 14 | 实付金额 | 应付租金 − 减免金额 + 超时费 + 损毁赔偿 |
| 15 | 测试 | 同订单汇总规则（订单维度判断） |

**减免金额定义**（严格按 rental 归属，无重复无遗漏）：
- A：discount.sub_biz_id 指向该 rental 的某个 rental_detail，且 valid=1
- B：discount.biz_type='租赁' AND biz_id=rental.id AND valid=1，且 sub_biz_id 不指向该 rental 的任何 detail

A ∪ B 取 distinct discount row 求和。

### Sheet 3：支付明细（7 列，按 order_payment 一行）

| # | 列 | 取值 |
|---|---|---|
| 1 | 订单号 | `order.code` |
| 2 | 支付方式 | `order_payment.pay_method` |
| 3 | mch_id | 微信支付时 `wepay_key.mch_id`（真实商户号），其他为 NULL |
| 4 | 支付金额 | `order_payment.amount` |
| 5 | 退款金额 | `SUM(payment_refund.amount WHERE 同退款条件)` 按 payment_id |
| 6 | 结余金额 | 4 − 5 |
| 7 | 测试 | 同订单汇总规则（订单维度判断） |

仅含 `op.status='支付成功' AND op.valid=1` 的支付行，paid=0 的招待/未结算单不会出现在此 sheet。

## 退款判定 / mch_id / 数据库 schema 注意

- **退款条件**：`payment_refund.state = 1 OR refund_id 非空` — 与后端 `Models/Rent/RentOrder.cs:519` 旧逻辑一致。仅 `state=1` 会漏大量已发起但未回调的退款（万龙时段实测漏 538 万）。
- **wepay_key 关联**：`order_payment.mch_id` 存的是 `wepay_key.id`（如 5/10/12），真实微信商户号在 `wepay_key.mch_id`（如 1604236346 是万龙租赁主力账户）。统计必须 JOIN。
- **rental_detail.charge_type**：实际只有 `租金 / 超时费 / 赔偿金` 三种值，DB 不存在 `损坏赔偿`，但脚本兼容写法 `IN ('赔偿金','损坏赔偿')` 防止后续字段名变更。
- **新旧 schema**：用的是新表 `[order]`（对应 `Order.cs`）/ `rental` / `rental_detail`。旧表 `order_online` / `rent_list` 2025-10-15 后无新数据，本脚本不查。
- **rental.settled=0 的虚账**：未归还 rental 会持续累积 `rental_detail` 应收记录，做收入分析时要意识到「订单明细.租金总额」可能远超实际应收。脚本不主动过滤，原样保留供分析。

## 已知问题排查

1. **`pyodbc.drivers()` 返回空** → 没设 `ODBCSYSINI`。`export ODBCSYSINI=/opt/homebrew/etc` 或写到 shell rc。
2. **`Can't open lib 'ODBC Driver 18 for SQL Server'`** → `brew install msodbcsql18 unixodbc` 没装齐。
3. **xlsx 写入失败 `PermissionError`** → 目标文件被 Excel/WPS 打开，关掉再跑。
4. **跑得慢（>2 分钟）** → DETAIL_SQL 里有 discount 相关子查询，几千条 rental 时正常 60-90 秒。生产网络抖动也会拖长。
5. **某条订单数据看起来不对** → 跑完前手动 `sqlcmd -S 100.28.143.19 -U claude -P 'abcd123!@#' -d snowmeet_new -C -W -Q "SELECT * FROM [order] WHERE code='WT_ZL_xxxx'"` 核对。

## 文件清单

- [`SKILL.md`](SKILL.md)（本文档）
- [`export_rent_orders.py`](export_rent_orders.py) — 通用导出脚本（argparse + 3 段 SQL + 对账后处理）

## 变更记录

- 2026-05-13：初版，从万龙单店脚本 `snowmeet_ai_doc/export_wanlong_rent_orders.py` 通用化而来。三个 sheet、测试列、临时订单分类、对账标红全部固化。
