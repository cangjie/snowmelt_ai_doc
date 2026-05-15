# 2026-05-15 财年版租赁导出 skill：从截图逐列定义到去重+缺号分析

接续 5-13/5-14 的万龙租赁导出工作。本场会话新建 `export_rent_order_fiscal_year` skill（单 sheet 财务宽表，区别于已有 3-sheet 对账版 `export_rent_order`），并按用户多轮要求迭代：valid 放宽 → code 非空 → 重复 code 去重 → 缺号连续性分析。所有改动落在 `snowmeet_ai_doc/skills/export_rent_order_fiscal_year/` 与产物 `snowmeet_ai_doc/wanlong_rent_orders_fy_2025-05-01_2026-04-30.xlsx`。

## 1. 需求澄清与表结构（plan mode）

### 1.1 起点与分叉

- 用户最初问「获取导出 wanlong_rent_orders_api…xlsx 的 skill」→ 查明无该 skill，只有一次性脚本 + 对账版 skill
- 用户改口：要新建 skill，DB 直查，2025-05-01~2026-04-30，按店铺参数化，**表头逐条口述**
- 关键转折：用户「设计这个表格的人脑子有包，字段多还重复，得一个一个说」→ 改用截图给表头

### 1.2 截图识别 → 5 段动态结构（重大修正）

- 3 张截图覆盖 A-BJ 62 列；U-X 截断、AF 笔误（应为【退款2】退款方式）、AJ/AO/AW 识别不全
- 用户澄清 U-X 是「类似【支付1】日期」的**动态字段**，列数 = 区间内单订单最大支付/退款笔数
- 表结构最终定为 5 段拼接：固定前缀(17) + 动态支付区(maxPay×5) + 动态退款区(maxRefund×4) + 固定中段(14) + 固定后缀(13)

### 1.3 逐列口径确认（按段过）

- 支付每笔 5 列（日期/时间/金额/支付方式/支付账号）；退款每笔 4 列（无退款账号）
- 财年 = biz_date 落 5/1~次年4/30，`YY-YY`；营/非按逐财年 SEASON dict（25-26 = 2025-10-21~2026-04-09）
- 财年序号留空；运营日序号 = 本雪季第 N 天；日序号 = 当日该店第 N 单
- 分账三列：order_share/payment_share；减免合计 = discount 三类(order_id / biz_type=租赁 biz_id / sub_biz_type∈日租金,租赁项 sub_biz_id) **去重** SUM（订单级，区别于对账版 rental 级）
- 客户名称/电话 = 快照优先 fallback member；union id = member_social_account（快照无此字段）
- 收款方式/支付账号 = 金额最大笔；正/闭 = 未支付 且 非招待(haveEntertain) → 关闭（质保条件用户最后去掉）
- 订单状态 = `rentProperties.rentStatus`，纯 SQL 无法精确复现 → 标注「近似需验收」

## 2. 实现与首次导出

### 2.1 关键探索（落地前实探）

- Python：`C:\Users\test\AppData\Local\Python\bin\python.exe`（WindowsApps 那个是 Store stub）；pyodbc 5.3.0 + ODBC Driver 18 OK
- 实探 schema：`order_payment.paid_date` 才是支付成功时间（待支付行该列 null）；`payment_refund` **无退款方式列** → 退款方式经 payment_id→order_payment.pay_method；`discount.valid` 确存在
- 复用对账版 `SHOP_PREFIX/REFUND_COND/DEFAULT_CONN/write_sheet`（sys.path import 单点真理）

### 2.2 实现要点

- 预查询 maxPay/maxRefund → 主查询订单级（聚合/标量子查询/OUTER APPLY 保粒度）→ 支付/退款明细各一条 → Python 按 order_id 拼动态列 + 财年体系，headers 与 row 同处生成防错位
- 财年缺失 raise 提示补 SEASON

### 2.3 第一次跑出的分叉：create_date vs biz_date

- 按 create_date 过滤 2025-05~2026-04 → 8066 单，biz_date 跨 22-23~25-26 四财年 → SEASON 缺财年 raise
- 用户决策：**按 biz_date 过滤**（年度报表语义）。改后默认区间订单财年恒 25-26
- 代价（已文档化）：与对账版（create_date 口径）不可 1:1 交叉对账

### 2.4 首版验证

- 98 列 × 2325 行；2325 行段2/段3 逐笔金额加总 == 支付/退款合计（0 偏差，含 53 多笔支付单）
- 测试 333 / 临时订单 135 与对账版同期记录吻合；快照优先 fallback member 验证通过

## 3. 三轮迭代（用户逐步收紧口径）

### 3.1 order 不论 valid 都导

- 加 `--include-invalid` 开关，用 `__VALID__` 运行期占位替换实现（仅放宽 order 表，其它表 valid 不动）
- 实测 3094 行；DB 同条件 COUNT=3094 全匹配，valid=1 子集=2325（与首版一致）证明超集正确

### 3.2 覆盖核查 + code 为空不导

- 用户问「全表该区间 type=租赁 code非空是否都在表里」→ 查明：全 DB 2965 单分 5 店（万龙2434/南山250/崇礼227/渔阳31/怀北23），脚本 `--shop` 必填天然单店，万龙这部分零差，其它店不在
- 用户：「只万龙 + code 为空不导」→ ORDER_FILTER **恒加 code 非空**（无 code = 未下单废弃单），即便 --include-invalid 也排除。剔 660 空 code 行 → 2434 行

### 3.3 重复订单号去重

- 用户规则：重复 code 留一条，优先级 **有成功支付记录 > valid=1 > id 最大**
- 实现：Python 按 code 分组 `max(key=(支付次数>0, valid==1, id))`；`o.valid` 加进主查询做判据；删原 PREQUERY_SQL，maxPay/maxRefund 改为去重后保留集 Python 取 max
- 实测 2434 → 2428 行；关键校验 `WT_ZL_251129_00016` 正确留带钱条 62709（¥1000/¥880）而非空单孪生 62708 ✓

## 4. 缺号连续性分析（收尾问题）

- 用户问「按天 code 尾号有无不连续」
- 结果：168 天每天都从 00001 起，仅 **3 天缺 6 个号**：251031 缺 7/11/14、251107 缺 11/13、251129 缺 15
- 逐个查 DB 证实这 6 个尾号**从未生成**（非过滤/去重副作用）
- **缺号与重复号是同一竞态的镜像**：两单同时读订单数 N 都写 N+1（1 重复），订单数 +2 只用 N+1，下一单读 N+2 写 N+3 → N+2 永久跳过（1 缺号）。6 重复 ↔ 6 缺号账完全对上
- 结论：导出完整无丢单，缺号是系统没发的序号，脚本无需改

## 关键改动文件

| 文件 | 改动 |
|---|---|
| [skills/export_rent_order_fiscal_year/export_rent_orders_fy.py](skills/export_rent_order_fiscal_year/export_rent_orders_fy.py) | 新建：argparse + 主查询(订单级) + 支付/退款明细 + Python 拼动态列&财年体系 + `--include-invalid` + code非空恒过滤 + 同 code 去重 |
| [skills/export_rent_order_fiscal_year/SKILL.md](skills/export_rent_order_fiscal_year/SKILL.md) | 新建：触发词区分财年版 vs 对账版、5 段结构、口径、SEASON 维护、订单状态近似、变更记录 |
| [wanlong_rent_orders_fy_2025-05-01_2026-04-30.xlsx](wanlong_rent_orders_fy_2025-05-01_2026-04-30.xlsx) | 最终产物 98 列 × 2428 行（669 KB） |
| `CLAUDE.md` | 2026-05-15 devlog：建 skill + biz_date 决策 + include-invalid + code非空 + 去重 + 缺号分析 |

## 学到的小知识

1. **`order_payment.paid_date` 是支付成功时间**：不是 create_date；待支付行 paid_date 为 null。对账版 PAYMENT_SQL 没取支付时间所以没踩到，财年版要支付日期列才发现
2. **`payment_refund` 无退款方式列**：退款方式只能经 `payment_refund.payment_id → order_payment.pay_method` 取原通道
3. **年度/财年报表必须按 biz_date 过滤**：按 create_date 会带出 biz_date 在旧财年的晚结算尾巴，财年列全乱；代价是与对账版 create_date 口径不可交叉
4. **`--shop` 必填 → 导出天然单店**：用户问「是否全包含」要先分清全表 vs 单店口径（该区间 type=租赁 code非空全 DB 2965 单分 5 店）
5. **重复 code = `GenerateOrderCode` 序号竞态**：「序号=同前缀订单数+1」并发下两单同序号。每次碰撞 = 1 重复号 + 1 缺号（下个序号被永久跳过），互为镜像，账可对平
6. **`__VALID__` 运行期占位技巧**：要在 import 期 f-string SQL 里留一个运行期才决定的过滤片段，用非花括号 token（如 `__VALID__`）穿过 f-string，main 里 `.replace()`
7. **本机 Python 路径坑**：`C:\Users\test\AppData\Local\Microsoft\WindowsApps\python.exe` 是 Store stub（exit 49），真解释器在 `C:\Users\test\AppData\Local\Python\bin\python.exe`
8. **交互教训**：用户逐列定义复杂规格时偏好自由口述/截图，连发 AskUserQuestion 多选题会反复挡住表达——已存 memory `feedback_spec_gathering_style.md`
