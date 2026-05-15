---
name: export_rent_order_fiscal_year
description: 按店铺 + 财年口径从生产 SQL Server 导出年度租赁订单到单 sheet 宽表 xlsx（财务/业务视角）。5 段拼接：固定前缀(财年/营非/运营日序号/支付退款汇总) + 动态支付区(每笔5列) + 动态退款区(每笔4列) + 固定中段(分账/减免/会员) + 固定后缀(订单号/结算/测试/临时单/正闭)。支付/退款明细列数据驱动（= 区间内单订单最大笔数）。触发场景：「按财年导一份 xx 店铺年度租赁订单」「导 2025-05~2026-04 万龙租赁宽表」「财务要的那个年度租赁大表」「带分账/营非/财年序号的租赁导出」等。与对账版 export_rent_order（3 sheet）互补，本 skill 是单 sheet 财务宽表。
---

# Export Rent Order (Fiscal Year) Skill

按店铺 + 财年从生产 SQL Server (`100.28.143.19/snowmeet_new`) 导出**年度租赁订单**到**单 sheet 宽表** xlsx。面向财务/业务汇报，非对账。设计目标换机即跑。

## 与 `../export_rent_order`（对账版）的区别

| 维度 | export_rent_order（对账版） | **export_rent_order_fiscal_year（本 skill）** |
|---|---|---|
| sheet 数 | 3（订单汇总/订单明细/支付明细） | **1（年度租赁）** |
| 形态 | 三张窄表 | **一张宽表，5 段拼接** |
| 日期过滤 | `order.create_date`（下单时间） | **`order.biz_date`（业务日期）** |
| 支付/退款列 | 固定（支付明细按行） | **动态展开**（每笔支付 5 列、每笔退款 4 列，列数 = 区间内单订单最大笔数） |
| 特色列 | 测试/临时订单/对账标红 | 财年/营非/运营日序号/分账三列/减免合计(订单级)/正闭/收款方式 |
| 用途 | 内部对账、异常排查 | 财务年度报表、业务汇总 |
| 默认区间 | 必填 --start/--end | **默认 2025-05-01~2026-04-30（可改）** |

> ⚠️ 两者**日期过滤口径不同**（biz_date vs create_date），同店同区间产物**不可 1:1 交叉对账**。金额单列口径（支付/退款/超时费/赔偿/分账）仍同源，可单订单按订单号比对。

## 何时触发

- 「按财年/年度导一份 xx 店铺租赁订单」「财务要的年度租赁大表」「2025-05~2026-04 万龙租赁宽表」
- 用户提到 **财年 / 营非 / 运营日序号 / 分账(应分账/实分账/待分账) / 收款方式 / 正闭** 等本 skill 特有列
- 用户要「一张表搞定」「每笔支付每笔退款都摊开成列」的租赁导出

## 一次性环境准备

```bash
# Windows（本 skill 落地环境）
pip install pyodbc openpyxl
# 需已安装 "ODBC Driver 18 for SQL Server"（验证：python -c "import pyodbc;print(pyodbc.drivers())"）

# macOS（如换机）
brew install unixodbc msodbcsql18
pip3 install pyodbc openpyxl
export ODBCSYSINI=/opt/homebrew/etc
```

依赖 sibling 目录 `../export_rent_order/export_rent_orders.py`（import 复用 `SHOP_PREFIX / REFUND_COND / DEFAULT_CONN / write_sheet`，单点真理）。**两个 skill 目录的 sibling 关系不可破坏**，否则 ImportError。

## 调用方式

```bash
cd /path/to/snowmeet_ai_doc/skills/export_rent_order_fiscal_year

# 默认 25-26 财年（2025-05-01 ~ 2026-04-30）
python export_rent_orders_fy.py --shop 万龙体验中心

# 指定店铺 + 区间
python export_rent_orders_fy.py --shop 渔阳 --start 2025-05-01 --end 2026-04-30 --out yuyang_fy.xlsx
```

参数：

```
--shop   必填，店铺中文名（DB order.shop）
--start  业务日期 biz_date 起（inclusive），默认 2025-05-01
--end    业务日期 biz_date 止（inclusive），默认 2026-04-30
--out    输出 xlsx 路径，默认 {prefix}_rent_orders_fy_{start}_{end}.xlsx（写当前目录）
--conn   ODBC 连接串，默认连生产（复用 ../export_rent_order/DEFAULT_CONN）
--include-invalid  导出 order 不论 valid 是否=1（默认仅 valid=1）。
         仅放宽 order 表过滤；rental/支付/退款/分账/会员等 valid 过滤不变。
         开启后包含作废/废弃单（多为未支付测试单），与对账版口径不一致、不可交叉对账
```

```bash
# 不论 valid 全导（含作废单），结果放 snowmeet_ai_doc/
python export_rent_orders_fy.py --shop 万龙体验中心 --include-invalid \
    --out /path/to/snowmeet_ai_doc/wanlong_rent_orders_fy_2025-05-01_2026-04-30.xlsx
```

`{prefix}` 复用对账版 `SHOP_PREFIX`（万龙体验中心→wanlong / 万龙服务中心→wanlong_service / 渔阳→yuyang / 南山→nanshan / 怀北→huaibei / 崇礼旗舰店→chongli）；未知店铺用中文名。

## 输出结构（单 sheet「年度租赁」，5 段）

设固定列 44 个，导出区间内单订单最大成功支付笔数 = P、最大有效退款笔数 = R，则总列数 = 44 + P×5 + R×4。

### 段 1：固定前缀（17 列）

| 列 | 口径 |
|---|---|
| 业务 | 固定 `租赁` |
| 财年 | biz_date 落在 5/1~次年4/30 区间 → `YY-YY`（如 2025-12 → `25-26`） |
| 营/非 | biz_date 在该财年雪季营业区间内 → `营业` 否则 `非营业`（区间见 SEASON dict） |
| 财年序号 | **空列**（表头保留，无值，按设计） |
| 运营日序号 | 本雪季第 N 天 = `biz_date − 营业起始日 + 1`；营业区间外留空 |
| 日序号 | 当日该店第 N 单（同 biz_date 同 shop 按 create_date 升序，限本导出集=租赁有效单） |
| 月份 | biz_date 自然月 1-12 |
| 创建日期/时间 | `order.create_date` 拆分 |
| 支付次数 | `COUNT(order_payment WHERE status='支付成功' AND valid=1)` |
| 支付合计 | `SUM` 同上 |
| 退款次数 | `COUNT(payment_refund WHERE state=1 OR refund_id 非空)` |
| 退款合计 | `SUM` 同上 |
| 订单结余 | 支付合计 − 退款合计 |
| 订单状态 | ⚠️ **近似值**，见下方「订单状态」说明 |
| 最后退款日期/时间 | `MAX(payment_refund.create_date)` 拆分（= 段5 结算日期/时间） |

### 段 2：动态支付区（重复 P 次，每笔 5 列）

`【支付k】日期 / 时间 / 金额 / 支付方式 / 支付账号`，第 k 笔成功有效支付按支付时间（`COALESCE(paid_date, create_date)`）升序。支付账号：微信支付 → `wepay_key.mch_id`（真实商户号），其他为空。不足 P 笔补空。

### 段 3：动态退款区（重复 R 次，每笔 4 列）

`【退款k】日期 / 时间 / 金额 / 退款方式`，第 k 笔有效退款按 `payment_refund.create_date` 升序。**退款方式取原支付通道**：`payment_refund.payment_id → order_payment.pay_method`（payment_refund 表本身无方式列）。不足 R 笔补空。

### 段 4：固定中段（14 列）

| 列 | 口径 |
|---|---|
| 超时费合计 | `SUM(rental_detail.amount WHERE charge_type='超时费' AND valid=1)`（订单下有效 rental） |
| 赔偿合计 | `SUM(... charge_type IN ('赔偿金','损坏赔偿') AND valid=1)` |
| 减免合计 | `discount` 三类记录去重后 SUM，见下方「减免合计」 |
| 隐藏订单 | `order.hide=1 → 是` |
| 应分账金额 | `SUM(order_share.amount WHERE order_id=? AND valid=1)` |
| 实分账金额 | `SUM(payment_share.amount WHERE share_id∈该订单 order_share.id AND valid=1 AND success=1)` |
| 待分账金额 | 应分账 − 实分账 |
| 业务 | 固定 `租赁`（与段1重复，原样保留） |
| 门店 | `order.shop` |
| 客户名称 | `COALESCE(NULLIF(order.contact_name,''), member.real_name)`（快照优先 fallback 会员） |
| 电话 | `COALESCE(NULLIF(order.contact_num,''), member_social_account[type=cell].num)` |
| union id | `member_social_account[type=wechat_unionid].num`（快照无此字段，直接取会员） |
| 收款方式 | **金额最大笔**成功支付的 `pay_method`（并列取 id 小者） |
| 支付账号 | 同上那笔的账号（微信→`wepay_key.mch_id`，否则空） |

### 段 5：固定后缀（13 列，≈ 对账版 sheet1「订单汇总」基线）

订单号 / 业务日期 / 业务时间 / 结算日期 / 结算时间 / 支付总金额 / 退款总金额 / 订单结余 / 店员姓名 / 测试 / 临时订单 / 客户名称 / 正/闭

| 列 | 口径 |
|---|---|
| 订单号 | `order.code` |
| 业务日期/时间 | `order.biz_date` 拆分 |
| 结算日期/时间 | `MAX(payment_refund.create_date)`（= 段1 最后退款） |
| 支付总金额/退款总金额/订单结余 | 同段1（重复列，原样保留） |
| 店员姓名 | `staff.name` |
| 测试 | 支付合计 < 5 OR 店员姓名含「苍」→ `是` |
| 临时订单 | 非测试 + 订单结余>0 + 该订单无有效 rental → `是` |
| 客户名称 | 同段4（重复列，原样保留） |
| 正/闭 | 未支付(支付合计=0) 且 非招待(`order.entertain≠1` 且 无 entertain rental) → `关闭`；否则 `正常` |

> **重复列**（业务×2、客户名称×2、订单结余×3、支付合计/支付总金额 等）按设计原样照搬，不加后缀区分。openpyxl 表头重名不报错。

## 减免合计（订单级，与对账版 rental 级口径不同）

对每订单，`discount` 表（`valid=1`）以下三类记录**按 discount.id 去重后** `SUM(amount)`：

1. `discount.order_id = 当前订单 id`
2. `discount.biz_type='租赁'` AND `discount.biz_id ∈ {该订单有效 rental 的 id}`
3. `discount.sub_biz_type IN ('日租金','租赁项')` AND `discount.sub_biz_id ∈ {该订单各 rental 的 rental_detail id}`

同一 discount 命中多类只算一次。**注意**：对账版 `export_rent_order` sheet2 的「减免金额」是 rental 级 A∪B 严格归属，口径不同，本 skill 不复用其 SQL 片段。

## 财年与营业区间（SEASON dict，需维护）

财年 = biz_date 落在 `5/1 ~ 次年 4/30`，标签 `YY-YY`。「营/非」「运营日序号」依赖**逐财年雪季营业起止**，内置于脚本 `SEASON` dict：

```python
SEASON = { '25-26': (date(2025,10,21), date(2026,4,9)) }
```

- 按 biz_date 过滤时，默认区间 2025-05-01~2026-04-30 的订单财年必为 `25-26`（按定义），SEASON 够用
- 导**其它财年**（改 --start/--end 跨年）时，若数据出现 SEASON 没有的财年，脚本**报错并列出缺失财年 + 单数**，提示在 `SEASON` 补行后重跑（不会静默算错）
- 滑雪租赁仅雪季发生，实测万龙 25-26 全部订单 biz_date 落在营业区间内（营/非 全 `营业`）属正常

## 「订单状态」是近似值（验收注意）

列「订单状态」目标是后端 `Order.cs:1062 rentProperties.rentStatus`（枚举：未开始/租赁中/部分归还/全部归还/部分退押金/全额退押金/了结关闭）。该值是依赖 `rental.realStartDate`/`totalSummary`/`guaranties.payStatus` 等**计算属性**的状态机，**纯 SQL 无法忠实复现**。本 skill 用可 SQL 化字段（rental 数 / settled 数 / start_date / 租金合计 / paid / refund / closed）按 `Order.cs:1134-1172` 的判定顺序做**近似**：

- 无有效 rental → 空（后端 rentProperties 为 null）
- min(start_date) > 今天 → `未开始`
- 有 rental.end_date 为空 → `租赁中`；settled 部分 → `部分归还`；settled 全部 → `全部归还`
- 有退款：`paid − 租金合计 ≤ refund` 时 closed=1→`了结关闭` 否则 `全额退押金`；否则 `部分退押金`

近似点：`租金合计`（rental_detail charge_type='租金' valid=1）替代后端 `totalSummary`，`start_date` 替代 `realStartDate`。**与小程序 new_rent_list 显示可能有偏差**，使用前请抽样验收；其余 90+ 列均精确。

## 已知问题排查

1. **ImportError: export_rent_orders** → sibling 目录 `../export_rent_order/export_rent_orders.py` 缺失或被移动。两个 skill 必须保持同级。
2. **`pyodbc.drivers()` 空 / Can't open lib** → 未装 ODBC Driver 18（Windows）或 macOS 未 `export ODBCSYSINI=/opt/homebrew/etc`。
3. **SystemExit「财年不在 SEASON」** → 按设计。在脚本 `SEASON` dict 补缺失财年的雪季营业起止后重跑。
4. **跑得慢（>2 min）** → 主查询含多个订单级标量/聚合子查询（减免去重、分账、租金合计），单店一年 ~2-3k 单约 30-90 秒，生产网络抖动会更久。
5. **某订单金额存疑** → `sqlcmd -S 100.28.143.19 -U claude -P 'abcd123!@#' -d snowmeet_new -C -W -Q "SELECT * FROM [order] WHERE code='WT_ZL_xxxx'"` 直查核对；支付/退款单列口径与对账版同源，可按订单号比对。
6. **xlsx 写入 PermissionError** → 目标文件被 Excel/WPS 打开，关掉再跑。
7. **金额科学计数法** → 已在脚本对金额列 `round(2)` + `number_format='0.00'` 兜住，无需手动处理。

## 文件清单

- [`SKILL.md`](SKILL.md)（本文档）
- [`export_rent_orders_fy.py`](export_rent_orders_fy.py) — 导出脚本（argparse + 预查询 maxPay/maxRefund + 主查询订单级 + 支付/退款明细查询 + Python 拼财年体系&动态列 + 复用对账版 write_sheet）

## 变更记录

- 2026-05-15：初版。按用户截图逐列定义（62 列截图 → 实为 5 段动态结构）。日期过滤最终定为 biz_date（年度报表语义；与对账版 create_date 口径不同，不可 1:1 交叉对账）。万龙体验中心 25-26 财年实测：98 列 × 2325 行，全 2325 行段2/段3 逐笔金额加总 = 支付/退款合计（0 偏差），测试 333 / 临时订单 135（与对账版同期记录吻合）。「订单状态」为 SQL 近似，「财年序号」按用户要求留空。
- 2026-05-15（同日）：加 `--include-invalid` 开关（仅放宽 order.valid，其它表 valid 过滤不变；用 `__VALID__` 运行期占位替换实现）。万龙 25-26 带开关实测：98 列 × 3094 行（DB 同条件总单 3094 全匹配，其中 valid=1 子集 2325 与初版一致，证明超集正确）；含作废单后 营/非 出现 218 非营业（淡季废弃单）、测试 1102、正/闭 关闭 849，均符合预期。
- 2026-05-15（同日）：**ORDER_FILTER 恒加 code 非空过滤**（`o.code IS NOT NULL AND LTRIM(RTRIM(o.code))<>''`），即便 `--include-invalid` 也排除无 code 的未下单/废弃单（不是真实业务记录）。
- 2026-05-15（同日）：**同订单号去重**（Python 端，通用）。DB 内 `GenerateOrderCode` 序号竞态会产生重复 code（万龙 25-26 有 6 个）。用户指定优先级保留一条：① 有成功支付记录 > ② valid=1 > ③ id 最大。同时 maxPay/maxRefund 改为去重后实际保留集在 Python 取 max（删原预查询）。万龙 25-26 + `--include-invalid` 最终实测：98 列 × **2428 行**（2434 → 去 6 重复），0 空订单号、0 重复 code；关键校验 `WT_ZL_251129_00016` 正确保留带钱条（¥1000/¥880）而非空单孪生。产物：`snowmeet_ai_doc/wanlong_rent_orders_fy_2025-05-01_2026-04-30.xlsx`（669 KB）。
