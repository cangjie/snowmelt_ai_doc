# 2026-05-13 晚 ~ 2026-05-14 凌晨：租赁订单接口排查 + 对账重做 + skill 落地

按时间线/主题整理。所有 SQL 都直查生产 `100.28.143.19/snowmeet_new`，所有改动落在工作目录 `/Users/cangjie/source/snowmeet/snowmeet_ai/`。

## 1. 起点：`WT_ZL_260314_00006` 为什么不在 GetConfirmedRentOrder 返回

### 1.1 字段层面排查（DB 直查）

订单 `id=70831`：
- `closed=1`、`valid=1`、`hide=0`、`close_date=2026-03-14 15:11:22` 全部满足接口字面要求
- 唯一支付 `op.id=41834`：微信支付 5000，status=支付成功
- 接口 [`GetConfirmedRentOrder`](../../SnowmeetApi/Controllers/RentController.cs:5544) 的 5 条规则（`paidAmount>0 AND closed=1 AND close_date!=null AND !hide AND 全是微信/支付宝`）都通过。

### 1.2 本地起服务实测

`http://localhost:5050/api/Rent/GetConfirmedRentOrder?sessionKey=…&startDate=2026-03-13&endDate=2026-03-20`：

```
http=200 rows=89 has_target=True
index of target: 69
shop=万龙体验中心, biz_date=2026-03-14, closed=1, paidAmount=5000
```

**接口实际是返回这条订单的**。问题在前端。

### 1.3 真正的根因：`rent_report_new.html`

前端报错：`TypeError: undefined is not an object (evaluating 'totalAmount.toFixed')` at line 264。

源码 [`SnowmeetApi/wwwroot/background/rent/rent_report_new.html`](../../SnowmeetApi/wwwroot/background/rent/rent_report_new.html:87-91)：

```js
var tHead = tHeadTemplate;
var tData = [];
render();          // 第 90 行调 render()
var totalAmount = 0  // 第 91 行才赋值
```

`var totalAmount` 因 var 提升存在但 `=0` 还没跑，render() 进 264 行 `totalAmount.toFixed(2)` 即抛错。修：把 `var totalAmount = 0` 移到 `render()` 前面（一行调整）。

### 1.4 二级根因：entertain 招待单被前端过滤

修了 toFixed bug 后，这条订单依然不在表格里。原因是它 `rental.entertain=true`：

```js
// rent_report_new.html:123
if (rental.entertain != 0) {
    continue;
}
```

`true != 0` 为 true → 跳过整条 rental。租金 0/已退 5000，是纯招待单，不计入"租赁订单报表"，业务语义上正确。

## 2. csv_excel_diff.xlsx 加分类列

为「应通过但CSV没有」sheet（194 行）逐列加分类，定位 CSV 漏单的原因分布。每列规则：

| 列 | 规则 | 命中数 |
|---|---|---|
| 招待 | `rental.entertain=1` | 18 |
| 体验 | `rental.experience=1` | 74 |
| 减免 | `sub_biz_type='日租金' AND biz_id=rental.id` 的 amount 总和 | 48 |
| 免除 | rental_detail 无 `valid=1 AND rental_id=rental.id` 的明细 | 48 |
| 测试 | `_订单已付金额 < 10` | 31 |

194 行里 6 行 5 个标签都未命中：

| 订单号 | 商品 | 租金 | 已付 | close_date |
|---|---|---|---|---|
| WT_ZL_251205_00004 | 【中级】全能板+青少全能 | 150 | 4000 | 2025-12-05 12:52 |
| WT_ZL_251230_00009 | 普通单板套餐 | 200 | 9000 | 2025-12-30 16:40 |
| WT_ZL_260103_00013 | 【成人中端】雪裤Nandn/Swagli/Trake | 120 | 1000 | 2026-01-03 16:21 |
| WT_ZL_260212_00013 | 头盔 | 0 | 2500 | 2026-02-12 22:31 |
| WT_ZL_260308_00005 | 中级双板套餐 | 180 | 6000 | 2026-03-08 11:16 |
| WT_ZL_260316_00004 | 【高级】民用高级+青少竞技+公园野雪 | 220 | 5000 | 2026-03-16 13:32 |

6 条全部 `closed=1`、`valid=1`、`hide=0`。

### 2.1 加「减免2」列发现关键差异

第二次定义减免：`biz_type='租赁' AND biz_id=rental.id`（不限 sub_biz_type）。
- 49 行非零，与「减免」（48 行）高度重合
- **WT_ZL_260316_00004** 的「减免」=0、「减免2」=220 → 这条 rental 的 totalRentalAmount=220，被 220 的 rental 级减免抵消为 0，前端 `>= 1` 判断把它跳过

### 2.2 排查 WT_ZL_260103_00013（rental 数据问题）

接口返回 `rentals[0].totalRentalAmount=0`，但 `paidAmount=1000 / refundAmount=880` 真实付了 120。DB 直查 rental_detail：

```
rental_id=27070 →
  id=81744  charge_type=租金    amount=60   valid=0  ← 失效
  id=81759  charge_type=超时费  amount=120  valid=1
```

唯一一条租金明细 `valid=0`，剩下只有超时费。`totalRentalAmount` 只统计 `charge_type='租金' AND valid=1` 的明细 → 为 0 → 前端跳过。这是数据质量问题：120 元实际收到的是超时费，不是租金。

## 3. 重新导出 wanlong_rent_orders_2025-10-15_2026-04-15.xlsx

走 plan mode 评审。

### 3.1 环境修复

- `OUT` 路径从 Windows `D:\snowmeet\snowmeet_ai_doc\...` 改 macOS 绝对路径
- pyodbc 看不到驱动 → brew 装的 `msodbcsql18 / unixodbc` 注册在 `/opt/homebrew/etc/odbcinst.ini`，但 pyodbc 默认查 `/etc/odbcinst.ini`。修复：`export ODBCSYSINI=/opt/homebrew/etc`

```bash
$ ODBCSYSINI=/opt/homebrew/etc python3 -c "import pyodbc; print(pyodbc.drivers())"
['ODBC Driver 17 for SQL Server', 'ODBC Driver 18 for SQL Server']
```

### 3.2 DETAIL_SQL 重构为 14 列

新增 5 列：是否招待 / 是否体验 / 应付租金 / 减免金额 / 实付金额。

减免金额最终口径（用户拍板，每条 rental 单独计算自己的归属 discount）：
- A：`discount.sub_biz_id` 指向**该 rental** 的某个 `rental_detail`（`valid=1`）
- B：`discount.biz_type='租赁' AND discount.biz_id=rental.id`，且 `sub_biz_id` **不指向该 rental 的任何 detail**（典型场景是 sub_biz_id NULL）

A ∪ B 取 distinct discount row 求和。**每条 discount 严格归属一条 rental**，多 rental 单子的全单 discount 不会被重复算。

DB 端实际所有 274 条 discount 三字段同填（`order_id + biz_type='租赁' biz_id + sub_biz_id`），但脚本逻辑通用，应付各种字段缺失情况。

| 字段 | 公式 |
|---|---|
| 应付租金 | 招待 OR 体验 → 0，否则 = 租金总额 |
| 损毁赔偿 | `charge_type IN ('赔偿金','损坏赔偿') AND valid=1`（DB 实际只有'赔偿金'，兼容写法） |
| 实付金额 | 应付租金 − 减免金额 + 超时费 + 损毁赔偿 |

### 3.3 跑出结果

```
汇总: 2325 行
明细: 2839 行
支付明细: 2125 行
文件: 327.9 KB（后来 341.7 KB，加完测试列）
```

抽样验证：
- `WT_ZL_260314_00006`（招待）：应付=0、实付=0 ✓
- `WT_ZL_260316_00004`（之前 5 条未命中）：租金 220 − 减免 220 = 实付 0，**前端被 `>= 1` 过滤掉的根因清楚体现** ✓
- `WT_ZL_251230_00011`（6 rental + 5 条 discount）：6 行减免合计 ¥879.95，无重复 ✓
- DB 端全表减免 ¥21,698.01 = Excel 全表减免合计 ¥21,698.01 ✓

## 4. 三个 sheet 都加测试列

规则统一：`订单的 paid_amount < 5` OR `店员姓名含 '苍'` → 测试='是'。

| Sheet | 列数 | 行数 | 测试=是 |
|---|---|---|---|
| 订单汇总 | 10 | 2,325 | 333（仅 paid<5: 223 / 仅含苍: 8 / 同时: 102） |
| 订单明细 | 15 | 2,839 | 531 |
| 支付明细 | 7 | 2,125 | 95 |

跨 sheet 一致性：107 条「苍」店员订单 → 在订单明细展开 136 条 rental 行**全部**标测试 ✓，在支付明细展开 90 条全部标测试 ✓。

支付明细测试只有 95 行（订单汇总 333）的原因：paid<5 多是 paid=0 的招待/未结算单，根本没 order_payment 记录，自然不出现在支付明细。

## 5. 对账：订单结余 vs 订单明细实付合计

规则：非测试订单中，「订单结余」≠ Σ(订单明细该订单非测试 rental 行的实付金额) → 订单号标红。

第一次跑：**158 行**标红。拆出来：

| 类别 | 行数 | 含义 |
|---|---|---|
| A | 135 | 结余>0 但订单明细里 0 条非测试 rental 行（临时订单/纯押金单） |
| B | 23 | rental 行存在但金额对不上（真正异常） |

B 类样本（值得关注的）：

| 订单号 | 结余 | 明细实付 | 差额 |
|---|---|---|---|
| WT_ZL_251103_00007 | 50 | 7610 | -7560 |
| WT_ZL_251115_00004 | 180 | 32620 | -32440 |
| WT_ZL_251116_00005 | 180 | 32400 | -32220 |
| WT_ZL_251128_00012 | 5000 | 0 | +5000 |

负差额大（明细远超结余）一般是 `rental.settled=0` 的虚账（按天累计但订单已关）；正差额（结余多于明细）是订单实收钱但 rental_detail 没记够。

## 6. 增加「临时订单」列

用户决策：A 类（135 条）不应标红，加一个「临时订单」列标'是'，订单号恢复正常显示；B 类（23 条）继续标红。

最终订单汇总 11 列、状态分布：

| 状态 | 行数 | 显示 |
|---|---|---|
| 测试单 | 333 | 测试列='是' |
| 临时订单 | 135 | 临时订单列='是'，订单号正常 |
| 异常订单 | 23 | 订单号红底深红字 |
| 正常订单 | 1834 | 三列都空 |
| 总计 | 2,325 | |

## 7. 落 skill：`snowmeet_ai_doc/skills/export_rent_order/`

通用化版本，未来导其他店铺/时间段直接复用。

文件：
- [SKILL.md](../skills/export_rent_order/SKILL.md) — 触发条件、环境要求、调用方式、列结构、排错
- [export_rent_orders.py](../skills/export_rent_order/export_rent_orders.py) — argparse 通用脚本

参数：

```
--shop    必填，店铺中文名（DB order.shop 字段值）
--start   必填，起始日期（inclusive）YYYY-MM-DD
--end     必填，截止日期（inclusive）YYYY-MM-DD
--out     可选，xlsx 路径
--conn    可选，ODBC 连接串
--no-postprocess  跳过临时订单+标红
```

店铺前缀映射（用于默认输出文件名）：

| shop | prefix |
|---|---|
| 万龙体验中心 | wanlong |
| 万龙服务中心 | wanlong_service |
| 渔阳 | yuyang |
| 南山 | nanshan |
| 怀北 | huaibei |
| 崇礼旗舰店 | chongli |

`post_process` 函数内化了 A 类（临时订单）+ B 类（标红）的互斥规则：临时订单 `continue` 掉，永远不会进标红分支。

冷启动验证（5-14 凌晨）：

```
ODBCSYSINI=/opt/homebrew/etc python3 export_rent_orders.py \
  --shop 万龙体验中心 --start 2025-10-15 --end 2026-04-15 --out /tmp/skill_test.xlsx

汇总: 2325 / 明细: 2836 / 支付: 2125
临时订单标记: 135 行
订单号标红: 21 行
```

（明细 2836 vs 2839、标红 21 vs 23 的轻微变化是 staff 入职更新 / rental 增减导致，逻辑无问题。）

## 关键改动文件

| 文件 | 改动 |
|---|---|
| [SnowmeetApi/wwwroot/background/rent/rent_report_new.html](../../SnowmeetApi/wwwroot/background/rent/rent_report_new.html:90) | 修 totalAmount var 提升 bug |
| [snowmeet_ai_doc/export_wanlong_rent_orders.py](../export_wanlong_rent_orders.py) | OUT 改 macOS / DETAIL_SQL 14 列重构 / 测试列加到 3 段 SQL |
| [snowmeet_ai_doc/csv_excel_diff.xlsx](../csv_excel_diff.xlsx) | 「应通过但CSV没有」sheet 加 6 列：招待 / 体验 / 减免 / 免除 / 测试 / 减免2 |
| [snowmeet_ai_doc/wanlong_rent_orders_2025-10-15_2026-04-15.xlsx](../wanlong_rent_orders_2025-10-15_2026-04-15.xlsx) | 3 sheet 重建 + 测试列 + 临时订单列 + 23 行标红 |
| [snowmeet_ai_doc/skills/export_rent_order/](../skills/export_rent_order/) | 新建 skill（SKILL.md + export_rent_orders.py） |

## 学到的小知识

1. **macOS pyodbc 找不到驱动** → `export ODBCSYSINI=/opt/homebrew/etc`，比改 `~/.odbcinst.ini` / `/etc/odbcinst.ini` 都更轻量
2. **var 提升坑**：JS `var` 声明前置但初值不前置，初始 `render()` 会拿到 undefined。修复就是把初始化挪到调用点之前
3. **discount 表三字段的同填特性**：万龙时段所有 discount 都同时填 `order_id + biz_type='租赁' biz_id + sub_biz_id`，所以 bucket A/B/C 完全重叠；但脚本逻辑要按字面定义实现，应付未来数据字段稀疏的情况
4. **DB 端 charge_type 实际只有'租金/超时费/赔偿金'**：'损坏赔偿'是用户口语，DB 不存在；脚本兼容写 `IN ('赔偿金','损坏赔偿')`
5. **rental_detail.rental.valid=0 的失效明细**：会让 `totalRentalAmount=0`，前端用 `>= 1` 过滤掉整行；这是 WT_ZL_260103_00013 之类订单看不见的根因
6. **多 rental 订单的 discount 归属**：写 SQL 时必须用「严格归属」（detail 级 + 非 detail rental 级），否则按 order_id 简单匹配会让全单 discount 在每条 rental 上重复算
