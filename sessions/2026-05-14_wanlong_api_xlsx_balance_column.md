# 2026-05-14 万龙 API 版 xlsx 补「订单结余」列 + 清科学计数法

接续 5-13 晚的万龙租赁数据导出工作。本次目标单一：把数据库直查版「订单汇总」sheet 的「订单结余」字段按订单号查表写入 API 拉取版 xlsx 的「订单」sheet 末尾。改动全在 `snowmeet_ai_doc/` 目录。

## 1. 补列脚本设计

### 1.1 起点

- 用户请求："给 `wanlong_rent_orders_api_2025-10-15_2026-04-15.xlsx` 增加一列「订单结余」，数值从 `wanlong_rent_orders_2025-10-15_2026-04-15.xlsx` 的 `订单汇总` sheet 按订单号查找写入"
- 已有文件状态：
  - 目标：API 版 4 列（订单号 / 顾客称呼 / 手机号 / 总计租金），2325 行，sheet 名 `订单`
  - 源：DB 直查版 3 sheet（订单汇总 / 订单明细 / 支付明细），「订单汇总」11 列，含 `订单号 + 订单结余`，2325 行
  - 同目录已有用户手动备份 `wanlong_rent_orders_api_2025-10-15_2026-04-15 copy.xlsx`，不用脚本再 backup

### 1.2 探查阶段

用 Explore agent 并行跑 openpyxl read_only 列出两文件所有 sheet + 表头 + 行数，确认：

- 目标 sheet `订单` 的订单号列名是 `订单号`（A 列）
- 源 sheet `订单汇总` 的相关列名分别是 `订单号` 和 `订单结余`
- 两份数据行数都是 2325，预期未命中 = 0

### 1.3 方案（走 plan 流程）

plan 文件：`~/.claude/plans/snowmeet-ai-doc-wanlong-rent-orders-api-abstract-bonbon.md`

一次性小脚本 `snowmeet_ai_doc/add_balance_to_api_xlsx.py`，不进 skill：

1. 源用 `read_only=True, data_only=True` 打开，按表头位置定位列号（不写死索引），构 `dict[订单号]=订单结余`
2. 目标用普通模式打开（保留样式），在 `max_column+1` 写表头「订单结余」+ 复用现有表头样式
3. 第 2 行起按 A 列订单号查 dict 写入；未命中累计计数 + 打印前 5 条样例
4. 列宽按视觉宽度（汉字 2 / 半角 1）+ 上限 36 自适应
5. 直接覆盖回原文件

### 1.4 首次执行结果

```
源 dict size: 2319
目标行数: 2325
未命中: 0
```

源 dict 2319 < 行数 2325 → DB 直查版「订单汇总」有 6 个订单号重复（dict 覆盖去重）。目标全部 2325 行都命中，结余列 100% 填充。

## 2. 科学计数法修复

### 2.1 用户反馈 + 根因定位

用户开 Excel 看新表，反馈"不要有科学计数法"。

抽查发现：

- 「订单结余」最小值 `-3.63806207381856e-14`（典型浮点零误差），Excel General 格式下自动显示 `-3.64E-14`
- 「总计租金」最大值 `42220.00999999999`、有 15 个浮点尾巴（`0.009999999999990905` 之类），General 格式不会触发科学计数法但视觉上糟糕
- 两列其它 99% 都是 int 或干净小数

### 2.2 修补策略

最小改动：在补列脚本里两列都做 `round(float(v), 2)`，并设 `number_format='0.00'` 锁定显示格式。

代码改动（`add_balance_to_api_xlsx.py` 主循环）：

```python
NUM_FMT = "0.00"
# 总计租金列 (D) 一并清浮点尾巴
rent_cell = ws.cell(row=r, column=4)
if isinstance(rent_cell.value, (int, float)):
    rent_cell.value = round(float(rent_cell.value), 2)
    rent_cell.number_format = NUM_FMT
if code in bal_map:
    v = bal_map[code]
    if isinstance(v, (int, float)):
        v = round(float(v), 2)
    bal_cell = ws.cell(row=r, column=new_col, value=v)
    if isinstance(v, (int, float)):
        bal_cell.number_format = NUM_FMT
```

### 2.3 验证

重跑后抽检：

- 残留极小值（总计租金）: 0
- 残留极小值（订单结余）: 0
- 未 round 干净: 0
- 原 `42220.00999999999` → `42220.01`
- 原 `-3.63806207381856e-14` → 全部归零

## 3. 根因未根治的部分

- API 拉取脚本 `export_wanlong_rent_orders_by_api.py` 的 `compute_displayed_rental` 在浮点累加（`r.get("totalRentalAmount") - r.get("totalDiscountAmount")` 跨多个 rental 求和）会持续产生尾巴
- 本次仅在 xlsx 层做 round 兜底，源脚本未改
- 下次 API 重跑覆盖后，浮点尾巴会回来，但只要再跑一次 `add_balance_to_api_xlsx.py` 就会顺手清掉（脚本对「总计租金」也强制 round）
- 如果以后要彻底治本，在 `compute_displayed_rental` 末尾 `return round(..., 2)`

## 关键改动文件

| 文件 | 改动 |
|---|---|
| `snowmeet_ai_doc/add_balance_to_api_xlsx.py` | 新建 — 一次性补列脚本（订单结余 + round 清浮点）|
| `snowmeet_ai_doc/wanlong_rent_orders_api_2025-10-15_2026-04-15.xlsx` | 4 列 → 5 列、两列金额 round(2) + `0.00` 格式 |
| `snowmeet_ai_doc/CLAUDE.md` | 「当前状态」日期戳 → 2026-05-14 晚；开发日志追加本次条目 |

未改动：`export_wanlong_rent_orders_by_api.py`（API 源脚本，浮点累加根因未触碰）

## 学到的小知识

1. **Excel General 格式 + 浮点零误差 = 科学计数法**：DB 端浮点累加产生 `±1e-14` 级别数值，Excel 默认 `-3.64E-14` 显示。导出脚本写金额到 xlsx 时强制 `round(2)` + `number_format = '0.00'` 一并兜住，比依赖 General 格式可靠
2. **API 版 vs DB 直查版行数对齐但 unique 不对齐**：两份各 2325 单；DB 直查版含 6 个订单号重复（同一订单号多 rental 在「订单汇总」聚合时仍可能产生重复行，dict 化时静默去重不影响本次任务的查表写入）
3. **按表头定位列号 vs 写死索引**：脚本 `header.index(KEY_HEADER)` 比 `header[0]` 安全得多，源表列顺序若调整不影响脚本；本次源表 11 列里两个目标列分别在第 1 和第 8，写死索引会脆
4. **幂等补列**：脚本检测「订单结余」列是否存在，存在则覆盖、不存在则在 `max_column+1` 追加。允许用户重跑（如 API 文件被重新生成后）而不会产生重复列
5. **`read_only=True, data_only=True`** 组合：read_only 释放内存（大表慢遍历无意义构建 in-memory tree）；data_only 拿计算值而非公式字符串。源表无公式但加上无害
