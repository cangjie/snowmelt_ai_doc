# 2026-05-16 财年导出 xlsx 增加「支付明细」+「支付流水」2 sheet：13 轮迭代 / 5 类对账 / 10 条关键发现

接续 5-15 的财年导出工作（财年版 xlsx `wanlong_rent_orders_fy_2025-05-01_2026-04-30.xlsx`，主 sheet「年度租赁」98 列 × 2428 行）。本次新增 2 个 sheet 从支付维度补全财务视角，与年度租赁三个分账口径完整对账。所有改动落在 `snowmeet_ai_doc/add_payment_detail_sheet_to_fy_xlsx.py`（新建，~290 行）+ 原 xlsx（追加 sheet）。本次走 plan mode（`/Users/cangjie/.claude/plans/start-work-graceful-pine.md`），plan 文件多轮 edit 跟随用户口径演进。

## 1. 第一轮：「支付明细」sheet 初版

### 1.1 列定义协商

用户给出列结构：每笔成功支付一行，固定列 + 动态退款列。最终敲定：
- 固定 8：订单号 / 支付订单号 / 支付方式 / 支付账户 / 支付日期 / 支付时间 / 支付金额 / 退款金额
- 动态：退款k 日期/时间/金额/方式（maxRefund 数据驱动）

订单号集合来自主 sheet「年度租赁」的「订单号」列（不重做 DB dedup），保证两 sheet 1:1 可交叉对账。

### 1.2 实现关键点

- 读主 sheet「年度租赁」抓订单号集合 2428（用 `header.index('订单号')` 不硬编位置）
- 分批 IN（每批 2000）查 order_payment（status=支付成功 AND valid=1）：2141 笔
- 分批 IN 查 payment_refund（命中 REFUND_COND `state=1 OR refund_id<>''`）：2088 笔
- maxRefund 数据驱动 = 3
- 复用 sibling skill `snowmeet_ai_doc/skills/export_rent_order/export_rent_orders.py` 的 `write_sheet` + `DEFAULT_CONN` + `REFUND_COND`（sys.path import 单点真理）

### 1.3 第一次跑踩坑：`payment_refund.valid` 列不存在

加了 `AND pr.valid = 1` → `pyodbc.ProgrammingError [42S22] 列名 'valid' 无效`。参考 skill 的 PAYMENT_SQL 也没用 `pr.valid`，证明 REFUND_COND 已是规范判定标准，删掉即可。

### 1.4 校验全部 PASS

- 行数自洽：主 sheet 【支付k日期】非空格子总数 2141 == 新 sheet 行数 2141 ✓
- 金额按订单号交叉：新 sheet 支付金额 SUM vs 主 sheet 【支付k】金额 SUM = 0 差异 ✓
- 退款金额按订单号交叉 = 0 差异 ✓
- 349 单仅在主 sheet 有 → DB 直查全部 0 笔成功支付（符合预期）
- 0 单仅在新 sheet 有（闭环）
- 3 类样本 spot-check：A 单笔无退款 / B 多笔支付 / C 含退款 全 PASS

## 2. 第二轮：加「支付结余」列

用户：「再增加一个"支付结余"列」。逻辑：`支付结余 = 支付金额 − 退款金额`。
- 加在退款金额之后（第 9 列）
- 全 2141 行校验 `支付结余 == round(支付金额 − 退款金额, 2)` 0 差异

## 3. 第三轮：澄清支付账户 ≠ 顾客 ID

**用户原话**：「支付账户不是顾客的openid 或者 payerid，而是如果是微信支付，支付账户就是mch_id，如果是支付宝则为空字符串；不过现有的支付账户的数据保留，列改名为顾客ID。」

### 3.1 拆列

- 现有「支付账户」列 → 改名「顾客ID」（保留 `COALESCE(NULLIF(op.open_id,''), op.ali_buyer_id)` 数据）
- 新增「支付账户」列：
  - 微信支付：`LEFT JOIN wepay_key wk ON wk.id = op.mch_id` 取 `wk.mch_id`（真实商户号）
  - 支付宝：空字符串
  - 其他（微信转账/储值支付/现金等）：空字符串

### 3.2 数据观察 — 万龙 25-26 财年 1690 笔微信支付分 3 商户

| 真实商户号 | 笔数 | 占比 | 说明 |
|---|---|---|---|
| `1604236346` | 1349 | 79.8% | 万龙租赁主力 |
| `1636313350` | 332 | 19.6% | 旗舰租赁历史遗留 |
| `1636404775` | 9 | 0.5% | 万龙零售 |

跟 CLAUDE.md 之前记录的"万龙微信支付分 3 商户"完全对得上（笔数有微小差异因为时段不同：本财年 vs 之前 25-10-15~26-04-15 区间）。

## 4. 第四轮：两 sheet 订单号集合一致性

**用户**：「两个sheet当中的订单号是否完全相同。」

结果：**支付明细 ⊊ 年度租赁**（严格真子集）
- 年度租赁 2428 唯一订单号
- 支付明细 2079 唯一订单号
- 仅年度租赁有 349 单（DB 抽查全部 0 笔成功支付，符合 status=支付成功 AND valid=1 过滤）
- 仅支付明细有 0 单（订单号溯源闭环）

### 4.1 进一步排除储值支付

**用户继续**：「年度租赁的订单，如果只看有实际支付，并且非储值支付的订单，订单号是否和支付明细相同？」

结果：**两集合完全相等**（双向零差，都是 2079）
- 年度租赁有支付的 2079 单中，**每一单都至少有 1 笔非储值支付**
- 支付明细里 35 笔储值支付都发生在"还有别的通道补差额"的订单上（混合支付：储值抵一部分 + 微信付剩余）
- 加不加"非储值"过滤，订单号集合都一样

## 5. 第五轮：增加分账列（payment_share / order_share / order_share_relation）

**用户**：「支付明细，参考payment_share order_share order_share_relation 增加 分账金额/成功/分账对象」

### 5.1 JOIN 链探明（Explore agent 报告）

```
payment_share.id  (one share execution per payment)
  ├── share_id → order_share.id (planned share at order level)
  │                  └── relation_id → order_share_relation.id
  │                                        └── name (分账对象显示名)
  ├── payment_id → order_payment.id
  ├── amount (this share's amount)
  ├── success (bool? true/false/null)
  └── valid (bool — 软删标记)
```

参考 [`OrderShareController.cs:244-248`](../SnowmeetApi/Controllers/Order/OrderShareController.cs) 的 Include 链：
```csharp
.Include(s => s.orderShare)
    .ThenInclude(s => s.relation)  // ← 分账对象 (relation.name)
```

### 5.2 实现

- 加 `fetch_shares(conn, payment_ids)` 分批查（初版仅过滤 `ps.valid=1`，success 状态留作展示）
- 动态 maxShare × 3 列：分账k 金额 / 成功（是/否/空） / 对象
- success bool? → 标签：True→"是"，False→"否"，None→""
- 分账方式仿照退款逻辑：分账k方式 = 原支付通道（payment_share 自身无 pay_method）

### 5.3 初版校验

- 1607 笔 payment 有分账（1608 share records，1 笔 payment 拆 2 行）
- 成功分布：1554 是 / 12 否 / 42 空（NULL 待回调） — 与 DB 直查完全匹配
- 唯一拆 2 行的 payment：`WT_ZL_251127_00009` pid=34208，分账 1: ¥0.02 + 分账 2: ¥0.02
- 全部分账对象 = 「万龙租赁分账」（单一商户号）
- 总分账金额 ¥237,895.39
- xlsx 按 payment_id 分账金额 SUM vs DB 直查 = 0 不一致

## 6. 第六轮：分账失败明细分析

**用户**：「看看分账失败的有哪些」+「列出订单号」

### 6.1 失败 12 笔全部支付宝（0 笔微信）

| 错误码 | 笔数 | 含义 | 业务根因 |
|---|---|---|---|
| `ACQ.ILLEGAL_SETTLE_STATE` | 8 | 结算单状态不允许分账，前置动作未完成 | 订单已退款 → 再发分账被驳 |
| `ACQ.ALLOC_AMOUNT_VALIDATE_ERROR` | 1 | 分账金额超过最大可分账金额 | 付¥3000→退¥2800→剩¥200，但发起 ¥260 分账 |
| `ACQ.TXN_RESULT_ACCOUNT_BALANCE_NOT_ENOUGH` | 1 | 支付宝账户余额不足 | 付¥5000→退¥4780→剩¥220，分账¥110 时余额不够 |
| `ACQ.DISCORDANT_REPEAT_REQUEST` | 1 | 流水号重复但明细不同 | 同 out_trade_no 第二次发请求换了金额 |

### 6.2 12 单订单号

| # | 订单号 | 分账金额 | 失败原因 |
|---|---|---|---|
| 1 | `WT_ZL_251128_00016` | ¥0.01 | DISCORDANT_REPEAT_REQUEST |
| 2 | `WT_ZL_251203_00003` | ¥110.00 | TXN_RESULT_ACCOUNT_BALANCE_NOT_ENOUGH |
| 3 | `WT_ZL_251203_00009` | ¥0.00 | ILLEGAL_SETTLE_STATE |
| 4 | `WT_ZL_251203_00011` | ¥0.01 | ILLEGAL_SETTLE_STATE |
| 5 | `WT_ZL_251213_00021` | ¥220.00 | ILLEGAL_SETTLE_STATE |
| 6 | `WT_ZL_260128_00014` | ¥0.01 | ILLEGAL_SETTLE_STATE |
| 7 | `WT_ZL_260128_00015` | ¥0.01 | ILLEGAL_SETTLE_STATE |
| 8 | `WT_ZL_260209_00013` | ¥0.01 | ILLEGAL_SETTLE_STATE |
| 9 | `WT_ZL_260209_00014` | ¥0.01 | ILLEGAL_SETTLE_STATE |
| 10 | `WT_ZL_260218_00012` | ¥0.01 | ILLEGAL_SETTLE_STATE |
| 11 | `WT_ZL_260218_00013` | ¥0.01 | ILLEGAL_SETTLE_STATE |
| 12 | `WT_ZL_260325_00005` | ¥260.00 | ALLOC_AMOUNT_VALIDATE_ERROR |

### 6.3 实际业务损失

≈ ¥370.32（`260325_00005` ¥260 + `251203_00003` ¥110 + 10 笔合计 ~¥0.32 小额测试单）

## 7. 第七轮：应/实/待 vs 全部/成功/失败 三对比

**用户**：「年度租赁 应分账金额/实分账金额/待分账金额 的总和 是否和 支付明细中 所有的分账金额的总和 / 成功的分账金额的总和 / 失败的分账金额的总和 相等」

### 7.1 三对比结果

| 对比 | 年度租赁 | 支付明细 | 差 | 结果 |
|---|---|---|---|---|
| 应分账 vs 全部分账 | 245,460.37 | 237,895.39 | +7,564.98 | ✗ |
| **实分账 vs 成功(是)** | **228,495.01** | **228,495.01** | **0** | **✓** |
| 待分账 vs 失败(否) | 16,965.36 | 590.08 | +16,375.28 | ✗ |

### 7.2 差额根因

差 7,564.98 = 22 单 `order_share` 已计划但 `payment_share` 未完整生成（17 单完全无 ps 行 ¥6030 + 5 单部分 ¥1534.98）

**完整闭环等式**：
```
应分账 245,460.37
  = 实分账 228,495.01 + 待分账 16,965.36                                  ✓
  = 成功 228,495.01 + 失败 590.08 + 待处理 8,810.30 + 未生成 7,564.98     ✓
```

## 8. 第八轮：WT_ZL_260223_00007 深查（"应分但作废"典型案例）

**用户**：「以 WT_ZL_260223_00007 为例，支付明细中 是没有分账的，但是年度租赁中 为什么 有实际分账金额？」

### 8.1 澄清误读

年度租赁该行实际值：
- 应分账金额：650 ✓
- **实分账金额：0** ← 用户误读为 650
- 待分账金额：650 ✓

用户看的"650"是应分，不是实分。

### 8.2 DB 真相：payment_share 的「作废」状态

- `[order]` id=69649 valid=1 **closed=1** **hide=True** close_date=2026-02-27 11:59
- `order_share` os_id=1519 amount=650 valid=1 **dealed=1**（订单关闭时同步记账）
- `payment_share` ps_id=1413 amount=650 **valid=False** success=NULL submit_time=NULL response_time=NULL
  - 分账请求生成后**立即被软删**（valid=0），根本没发出去
  - 大额退款后系统主动放弃这笔分账

### 8.3 时间线

| 时刻 | 事件 |
|---|---|
| 02-23 09:13 | 顾客微信支付 ¥9000（pid=40753，mch_id=1604236346）|
| 02-27 11:59 | 退款 ¥7700，订单 closed=1 + hide=True |
| 02-27 11:59 | `order_share` 记账：应分 650（dealed=1）|
| 02-27 22:00 | 分账定时任务创建 `payment_share` 行 + 立即置 valid=0（submit_time=NULL）|

## 9. 第九轮：澄清 应/实/待 定义

**用户**：「年度租赁中，应该首先看分账成功了多少，作为实分账金额，应分账金额 - 实分账金额 = 待分账金额」

确认这正是年度租赁现有定义 + 100% 自洽（全 2428 行行级零差异，整表 `应 − 实 − 待` = 0）。fy 主脚本 `pay_agg` 的 actual_share JOIN 条件 = `ps.valid=1 AND ps.success=1`，与"成功"完全一致。

## 10. 第十轮：更新 sheet — 加 valid=0 作废行

**用户**：「好的，现在把表格更新下」

### 10.1 实施

- 移除 `fetch_shares` 里 `ps.valid=1` 过滤
- `_success_label(success, valid)` 加 valid 参数：valid=False → "作废"，else 走原逻辑
- 重跑：13 笔作废行可见（其中 WT_ZL_260223_00007 出现作废分账 ¥650）

### 10.2 重新对账

新分布：1554 是 + 12 否 + 42 空 + 13 作废 = 1621 笔 ¥242,540.39

剩余差 ¥2,919.98 = 9 单完全在 payment_share 维度看不到的「应分但 ps 全无」+ 部分不齐

## 11. 第十一轮：列出 9 单缺口订单

**用户**：「一共只有8单吗？」

我之前用 `should-got>0.005` 单向过滤少算 1 单**反向差额**：`WT_ZL_251127_00009` 应分 ¥0.02 但生成了 2 笔 ¥0.04，gap = -¥0.02 被滤掉。

修正口径 `abs(diff) > 0.005` 后 = **9 单 ¥2,919.98**：

| 订单号 | 应分 | 已生成 ps | 差额 | 类型 |
|---|---|---|---|---|
| WT_ZL_251130_00004 | 3300 | 2030 | +1270 | 部分 |
| WT_ZL_260205_00001 | 465 | 0 | +465 | 无 ps |
| WT_ZL_260101_00022 | 360 | 0 | +360 | 无 ps |
| WT_ZL_251224_00014 | 290 | 0 | +290 | 无 ps |
| WT_ZL_260208_00001 | 270 | 0 | +270 | 无 ps |
| WT_ZL_251128_00017 | 130 | 0 | +130 | 部分 |
| WT_ZL_251203_00009 | 130 | 0 | +130 | 部分 |
| WT_ZL_260123_00012 | 875 | 870 | +5 | 部分 |
| **WT_ZL_251127_00009** | **0.02** | **0.04** | **−0.02** | **反向多生成** |

拆类：4 单完全无 ps（¥1385）+ 5 单部分（¥1534.98）= 9 单 ¥2919.98 ✓

## 12. 第十二轮：新加「成功交易」sheet

**用户**：「再增加一个sheet，列名: 订单号 / 支付方式 / 支付账户 / 商户订单号 (out_trade_no) / 类型（支付 退款 分账） / 日期 / 时间 此sheet只收录成功的交易」

### 12.1 实现

3 类成功交易合并到一个时间线，按日期+时间升序：
- 支付：`order_payment.out_trade_no`, paid_date(fallback create_date)
- 退款：`payment_refund.out_refund_no`（不是 out_trade_no — payment_refund 表没这字段）, create_date
- 分账：`payment_share.out_trade_no`, response_time(fallback create_date)，**仅含 success=1 AND valid=1**
- 支付方式/支付账户：退款/分账继承自所属 payment（`pid_ctx` map 查父支付）

### 12.2 数据

- 总 5783 行 = 2141 支付 + 2088 退款 + 1554 分账（success=1 且 valid=1）
- 出现 3 种 out_trade_no 命名约定：
  - `{订单号}_ZF_NN` 支付（如 `WT_ZL_251021_00001_ZF_01`）
  - `{订单号}_ZF_NN_TK_MM` 退款（如 `..._ZF_01_TK_01`，第 1 次支付的第 1 次退款）
  - `{订单号}_ZF_NN_FZ_MM` 分账（如 `..._ZF_01_FZ_01`）

## 13. 第十三轮：rename + 加金额列

**用户**：「成功交易这个sheet 改名叫"支付流水"，增加交易金额列，金额 如果是 支付 则为正，否则为负。」

- sheet 名 `成功交易` → `支付流水`（旧名也清理）
- 加「交易金额」列：支付正数 / 退款负数 / 分账负数（站在商家本主体角度，退款+分账都是流出）

### 13.1 三类金额合计与年度租赁对账完整

| 项 | 支付流水 | 年度租赁 | 结果 |
|---|---|---|---|
| 支付总额 | 7,209,321.57 | 【支付k】sum 7,209,321.57 | **✓** |
| 退款 abs | 6,604,799.33 | 【退款k】sum 6,604,799.33 | **✓** |
| 分账 abs | 228,495.01 | 实分账金额 sum 228,495.01 | **✓** |

净流入 ¥376,027.23（5783 行 含符号 SUM = 实际净留存）。

样本 WT_ZL_260405_00006 全链路：
| 商户订单号 | 类型 | 交易金额 | 时间 |
|---|---|---|---|
| `..._ZF_01` | 支付 | +5000 | 2026-04-05 10:30 |
| `..._ZF_01_TK_01` | 退款 | −4640 | 2026-04-06 14:59 |
| `..._ZF_01_FZ_01` | 分账 | −180 | 2026-04-07 23:00 |
| 该单净流入 | | **+180** | |

## 关键改动文件

| 文件 | 改动 |
|---|---|
| [`snowmeet_ai_doc/add_payment_detail_sheet_to_fy_xlsx.py`](../add_payment_detail_sheet_to_fy_xlsx.py) | 新建 ~290 行：`read_order_codes` / `fetch_payments` / `fetch_refunds` / `fetch_shares` / `build_headers_and_rows` / `build_transaction_rows` / `main`；复用 `skills/export_rent_order` 的 `DEFAULT_CONN` + `REFUND_COND` + `write_sheet` |
| `snowmeet_ai_doc/wanlong_rent_orders_fy_2025-05-01_2026-04-30.xlsx` | 追加「支付明细」(22 列 × 2141 行) +「支付流水」(8 列 × 5783 行) 2 sheet |
| [`~/.claude/plans/start-work-graceful-pine.md`](.) | plan 文件（多轮 edit 跟随用户口径） |

## 学到的小知识

1. **`payment_refund` 表无 `valid` 列**：所有过滤只能走 REFUND_COND（`state=1 OR refund_id<>''`）。盲加 `pr.valid=1` 会 SQL 报错；参考 export_rent_order skill 的 PAYMENT_SQL 同样不写。
2. **`order_payment` 支付账户 vs 顾客ID 容易混淆**：`open_id` / `ali_buyer_id` 是顾客侧 ID（微信 openid / 支付宝 payer_id），不是商家支付账户。"支付账户"语义应取 `JOIN wepay_key wk ON wk.id = op.mch_id` 拿真实商户号。
3. **`payment_share.success` 是 `bool?` 4 态状态机**：`success=1` 成功 / `success=0` 接口驳回 / `success=NULL valid=1` 待回调 / `valid=0` 作废（submit_time 多为 NULL，请求生成后立即软删）
4. **年度租赁的「实分账金额」严格等于 `payment_share` 中 `success=1 AND valid=1` 的 SUM**：与年度租赁 fy_main_sql 的 actual_share JOIN 完全一致（228,495.01）。整表 2428 行 `应分 − 实分 = 待分` 行级零差异。
5. **分账失败的支付宝错误码语义**：
   - `ACQ.ILLEGAL_SETTLE_STATE`：结算单不允许分账（常因退款先于分账，时序问题）
   - `ACQ.ALLOC_AMOUNT_VALIDATE_ERROR`：分账金额>可分余额（付-退后剩余不够）
   - `ACQ.TXN_RESULT_ACCOUNT_BALANCE_NOT_ENOUGH`：账户余额不足
   - `ACQ.DISCORDANT_REPEAT_REQUEST`：同 out_trade_no 二次提交金额不一致
   - 8/12 是退款-分账时序问题
6. **`out_trade_no` 命名编码了业务类型**：`{订单号}_ZF_NN` 支付 / `{订单号}_ZF_NN_TK_MM` 退款 / `{订单号}_ZF_NN_FZ_MM` 分账，可凭字符串判断交易类型。
7. **SQL Server `IN` CTE 双使用时参数数翻倍**：CTE 里两处 IN 同一批次 → SQL Server 把所有占位符都重复一遍，每批 ≤1000 codes 才不超 2100 上限。
8. **`should - got > tolerance` 单向比较会漏反向差额**：用 `abs(diff) > tol` 才完整；本会话曾因此漏 1 单 ¥0.02 多生成的反向 case（WT_ZL_251127_00009）。
9. **`payment_refund` 的"商户订单号"在 `out_refund_no` 字段**：不是 `out_trade_no`（payment_refund 表没这字段）。命名跟支付宝/微信的"退款单号"对齐。
10. **退款方式只能 JOIN order_payment 拿**：payment_refund 表无 pay_method 列（参见 CLAUDE.md 早记录的「退款方式 = 原支付通道」约定）。

## 明日待验证

- Excel 打开 xlsx 肉眼检查 3 sheet 列结构 + 样本数据
- 9 单 ¥2,919.98 应分缺口订单是否需要人工补分账（特别是 `WT_ZL_251130_00004` ¥1270 / `WT_ZL_260205_00001` ¥465 等大额）
- 分账失败 12 笔归因是否准确（按错误码归类后告知运营）
- 是否需要在「支付明细」加「应分账金额」列（合并 order_share + payment_share 维度，让 ¥2919.98 在该 sheet 也可见）
