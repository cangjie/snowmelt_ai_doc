# 2026-05-17 财年 xlsx 加「店员openid」+「union id」改「顾客openid」+ 支付对账验证

接续 5-16 的财年导出工作。本场三件事：① 验证 5-16 新建的「支付流水」「支付明细」两 sheet 按订单号能否对账；② 给主 sheet「年度租赁」在「店员姓名」后加「店员openid」列（口径 3 轮迭代）；③ 把「union id」列改名「顾客openid」并换数据源。所有改动落在 `snowmeet_ai_doc/skills/export_rent_order_fiscal_year/export_rent_orders_fy.py` + 同目录 `SKILL.md` + 新建 `snowmeet_ai_doc/verify_payment_reconcile.py`，xlsx 重生成。plan：`/Users/cangjie/.claude/plans/sheet-openid-adaptive-teacup.md`。

## 1. 支付对账验证（支付流水 vs 支付明细）

### 1.1 需求
用户：「支付流水 和 支付明细 按照订单号汇总 看看最终金额是否一致？最终金额 = 支付金额 − 退款金额 − 分账金额」

### 1.2 实现 + 结论
- 新建只读 `verify_payment_reconcile.py`：openpyxl 读两 sheet，按订单号聚合
- 两 sheet 订单号集合完全相同（各 2079，无单边）
- 支付金额、退款金额逐项 0 差
- **分账口径是关键**：支付流水按设计只收 `success=1 AND valid=1`（脚本 line 330）；支付明细分账列含全部状态（是/否/作废/空）
  - 「仅成功分账」口径：最终金额 ¥376,027.23 双方完全一致，2079 单逐单零差异 ✓
  - 「全部分账」口径：支付明细 ¥361,981.85，差 ¥14,045.38 / 64 单 —— 全是失败/作废分账，**设计预期非 bug**（经济意义上失败分账没分出去，不应抵减）

## 2. 「年度租赁」新增「店员openid」列

### 2.1 探明数据路径（计划阶段）
- 现「店员姓名」来自 `[order].staff_id → staff.name`，但 openid 不在这条路径
- 新 `[order]` 表无 `staff_open_id`（那是旧 RentOrder 模型字段）；`staff` 表也无 openid/member_id 列
- 用户指明路径：`staff_id → staff_social_account`（按订单时间落该表 start_date/end_date 窗口）`→ social_account_for_job.wechat_mini_openid`
- 与后端 `SocialAccountForJob.GetStaff(date)`（`SnowmeetApi/Models/Staff/SocialAccountForJob.cs:24-38`）一致：start_date DESC 取第一条 valid=1 且窗口覆盖 date 的
- 表结构（C# 模型核实）：`staff_social_account`(staff_id/social_account_id/start_date/end_date/valid)、`social_account_for_job`(id/cell/wechat_mini_openid/member_id)

### 2.2 实现机制
脚本 MAIN_SQL 的 SELECT 别名 → `cur.description` → `idx={name:i}`（line 327）→ 行内 `g=lambda n: row[idx[n]]`；headers/seg 手工对齐，末尾 `assert len(full)==len(headers)`（line 462）兜底。所以只要加 `AS 店员openid` 别名 + headers/seg5 各插一项。仿现有 `msa_cell`/`big_pay` 的 `OUTER APPLY ... TOP 1` 风格（防行数翻倍）。

### 2.3 口径 3 轮迭代（每轮都重跑生产验证）

| 轮次 | 口径 | 有名无openid |
|---|---|---|
| ① | 窗口覆盖 biz_date + `ssa.valid=1` | 13 |
| ② | 去掉 `ssa.valid=1`（保留窗口） | 4 |
| ③ | 两级偏好：窗口优先，无窗口回退最近曾用 | **0** |

- **①→②**：用户「不追求有效，离职的店员本来就是无效的」。DB 探明 `段春敏` 有 valid=0 旧账号（start 1905→end 2025-11-11，openid ...f8LE）覆盖历史订单，被 valid=1 误杀；`韩冬垚` 是真·无窗口覆盖（账号 2025-09-30 截止）
- **②→③**：用户「空着的员工 open id 查询下他们曾用的 open id 填写上即可」。改 `ORDER BY CASE WHEN 窗口命中 THEN 0 ELSE 1 END, start_date DESC, id DESC` + `WHERE ... AND LTRIM(RTRIM(saj.wechat_mini_openid))<>''`。全量评估 命中 2325 / 仍空 0
- 最终：2319 行全覆盖；`韩冬垚→oHdTn5edMxhoKhq-nwxmAOMwf8LE`、`张亮→oHdTn5aq19gUyZnwBzjXonzzyxqI`（曾用账号回填）

## 3. 「union id」→「顾客openid」

- 用户：「哪个顾客的 unionid 改为顾客的 openid，列名变，数据也重新读取」
- AskUserQuestion 确认新列名 = `顾客openid`（数据源固定 wechat_mini_openid，与店员openid 同类型）
- 改 4 处：SELECT 别名 `msa_uid.num AS [union id]` → `msa_oid.num AS 顾客openid`；OUTER APPLY `type=N'wechat_unionid'` → `N'wechat_mini_openid'`、alias `msa_uid→msa_oid`；headers seg4；seg4 行 `g('union id')→g('顾客openid')`；SKILL.md 口径行
- 结果：非空 2274/2319（98.1%），其余空 = 会员无 wechat_mini_openid（纯线下/未授权小程序顾客）

## 4. 环境关键发现：本机 Intel Mac ODBC

- 首次按 CLAUDE.md「已知遗留」设 `ODBCSYSINI=/opt/homebrew/etc` → `pyodbc.drivers()` 空，连不上
- 根因：那条笔记是给 **Apple Silicon 同步机**；本机是 **Intel Mac**（brew 在 `/usr/local`）
- 正确：`ODBCSYSINI=/usr/local/Cellar/unixodbc/2.3.4/etc`，注册驱动名 `ODBC Driver 13 for SQL Server`（脚本 DEFAULT_CONN 写死 Driver 18，需 `--conn` 覆盖成 Driver 13）
- `add_payment_detail_sheet_to_fy_xlsx.py` 内 `os.environ.setdefault("ODBCSYSINI","/opt/homebrew/etc")` 会污染 → 必须显式预设正确 ODBCSYSINI 让 setdefault 变 no-op
- 已存 auto-memory + CLAUDE.md「已知遗留」追加一条

## 关键改动文件

| 文件 | 改动 |
|---|---|
| [`skills/export_rent_order_fiscal_year/export_rent_orders_fy.py`](../skills/export_rent_order_fiscal_year/export_rent_orders_fy.py) | MAIN_SQL 加 `staff_oid` OUTER APPLY + `店员openid` 别名；`msa_uid→msa_oid` 改 wechat_mini_openid + 别名 `顾客openid`；headers/seg4/seg5 各插列 |
| [`skills/export_rent_order_fiscal_year/SKILL.md`](../skills/export_rent_order_fiscal_year/SKILL.md) | 段5 13→14 列、固定列 44→45、店员openid 两级偏好口径、union id→顾客openid 口径行 |
| [`verify_payment_reconcile.py`](../verify_payment_reconcile.py) | 新建只读对账脚本 |
| `wanlong_rent_orders_fy_2025-05-01_2026-04-30.xlsx` | 重生成：年度租赁 99 列×2319 行 + 重跑 add_payment 补回 支付明细/支付流水 |

## 学到的小知识

1. **跨设备同步项目的环境笔记会失真**：CLAUDE.md 的 ODBC 笔记只对 Apple Silicon 成立，Intel Mac 路径/驱动名全不同。环境类记忆要标明适用机型
2. **财年脚本整本 `Workbook()` 重建**：每次重跑必须紧接重跑 `add_payment_detail_sheet_to_fy_xlsx.py` 补回 2 sheet。曾因 `cd` 进 skill 子目录后链式跑第二脚本导致路径找不到、漏 sheet → 两脚本要在各自正确 CWD 跑
3. **`staff_social_account.valid` 是「当前是否在职该账号」语义**，不是「历史是否有效」。历史归集报表过滤 valid 会丢离职店员的历史订单 openid
4. **SQL 单 OUTER APPLY 做两级偏好**：`ORDER BY CASE WHEN <优先条件> THEN 0 ELSE 1 END, <次序>` + `TOP 1`，一次查询实现「优先 A，否则回退 B」，不用 UNION/二次 APPLY
5. **同一份对账「不一致」可能是设计预期**：支付流水 vs 支付明细差 ¥14045 全是失败/作废分账，口径不同非数据错——对账先问「两边口径是否同源」
