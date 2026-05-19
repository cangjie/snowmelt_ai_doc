# 2026-05-19 总部零售财年导出 + 孤儿核对纳入总部：四店升五店

接续 2026-05-19（续）零售明细/孤儿核对线。本会话两件事：① 用与万龙/南山/崇礼旗舰同样的方法导出**总部**财年零售报表；② 总部既已有报表，把反向核对孤儿口径从「四店」升级到「五店」，重算 `all_销售单列表_孤儿记录.xlsx`。改动落在 `snowmeet_ai_doc/`（脚本 + 报表 + CLAUDE.md）。

## 1. 总部财年零售报表

### 1.1 定位店铺名与环境

- 用户原话："参考 万龙 南山 崇礼旗舰店导出的零售报表，导出 总部 的零售报表"
- 查生产库 distinct `order.shop`（type=零售 / FY）→ 确认 HQ 的 DB 值就是 `总部`，47 单（biz_date 2025-12-10~2026-03-21）
- `总部` 不在 `export_rent_orders.py:SHOP_PREFIX`，`default_out_name` 会 fallback 用中文原文当前缀 → 显式 `--out headquarters_...` 保持与四店英文前缀命名一致
- **环境核查（重要纠偏）**：本机 `brew --prefix` = `/opt/homebrew`（Apple Silicon），非 CLAUDE.md 笔记里的 Intel Mac。`brew --prefix unixodbc` 会卡死（杀掉，别用 brew 探测）；直接看 `/opt/homebrew/etc/odbcinst.ini` 有 Driver 17/18，`pyodbc 4.0.39`。结论：只需 `export ODBCSYSINI=/opt/homebrew/etc`，Driver 18 即脚本 `DEFAULT_CONN` 默认值，**无须 `--conn` 覆盖**（与四店 Intel Mac 那条笔记不同机器）

### 1.2 两脚本工作流（顺序不可换）

```
ODBCSYSINI=/opt/homebrew/etc python3 export_retail_orders_fy.py --shop 总部 \
  --out .../headquarters_retail_orders_fy_2025-05-01_2026-04-30.xlsx
ODBCSYSINI=/opt/homebrew/etc python3 add_payment_detail_sheet_to_fy_xlsx.py \
  --xlsx headquarters_retail_orders_fy_2025-05-01_2026-04-30.xlsx --main-sheet 年度零售
ODBCSYSINI=/opt/homebrew/etc python3 verify_payment_reconcile.py \
  --xlsx headquarters_retail_orders_fy_2025-05-01_2026-04-30.xlsx
```

- 主表：51 列 × 47 行；maxPay=1 / maxRefund=0 / 0 重复 / 0 退款 / 0 分账
- 3 sheet：`年度零售` / `支付明细` / `支付流水`

### 1.3 三表对账（零差异）

- Σ订单结余 = Σ销售额合计 = DB SUM(deal_price) = DB SUM(支付成功 amount) = **¥114,924.00**
- 逐订单口径 A/B 均 0 单不一致
- 总部无退款、无折让、全额支付 → 销售额合计 恰等于 订单结余（符合 SKILL.md 口径）

## 2. 孤儿核对升级到五店

### 2.1 为何要更新

原 `export_all_orphan_records.py` 只读四店 `年度零售明细` 算 `consumed`，总部归在「总部·无财年零售报表(预期)」。总部既出报表，这批应重新匹配 → 从预期转待查/已消费。

### 2.2 链路（三步）

1. **补七色米号列**：总部 `年度零售` 原本无 `七色米订单号`（该列是四店 2026-05-18续2 单独后处理加的，非 fy 导出自带）。一次性脚本按订单号查 `retail.mi7_code`，`STUFF(...FOR XML PATH(''))` 去重多码分号连接（SQL Server 2012 无 `STRING_AGG`），追加为第 52 列，其余 sheet 不动。47 单中 40 单有号。脚本用完即删（与 `_discover_hq.py`/`_verify_hq.py` 同处理）
2. **明细合并**：新建 [`add_headquarters_retail_detail_merged_xlsx.py`](../add_headquarters_retail_detail_merged_xlsx.py)，克隆 `add_chongli_retail_detail_merged_xlsx.py`，仅换输入/输出文件 + `EXCLUDE_CODES=set()`（总部首次合并无用户指定测试单）。统一明细源 `all_销售单列表.xls`。结果：40 单全匹配、0 差额（Σ明细总额=销售额合计）、17 单需合并、7 单无号标红、0 关闭、0 剔除；总部主报表 +`年度零售明细`（63列×67行）+ 备份 `headquarters_retail_orders_fy_with_detail.xlsx`
3. **孤儿脚本改 4 处**：`FILES` 加 `"总部": "headquarters_..."`；`categorize()` 总部分支 `"总部·无财年零售报表"` → `"总部·报表无七色米号(待查)"`；`order` 权重表插入总部待查、删旧总部预期键；docstring/打印 `四店`→`五店`、删硬编码 `210 行级 / 124 单据级` 改动态计数

### 2.3 结果对比

| | 之前 | 现在 |
|---|---|---|
| all | 910 单据 / 1268 明细行 | 同 |
| 消费 | 786 单据（四店） | **826 单据（五店，+40）** |
| 孤儿 | 124 单据 / 210 行 | **84 单据 / 150 行** |
| 待查 | 30（崇礼25 + 南山5） | **72（崇礼25 + 南山4 + 总部43）** |
| 预期 | 94（总部82 + 崇礼万龙4 + 关闭7 + 剔除1） | 12（崇礼万龙4 + 关闭7 + 剔除1） |

输出文件 `all_销售单列表_孤儿记录.xlsx`（36.4 KB，2 sheet：孤儿明细 150 行 / 孤儿汇总 84 单；待查行红 FF9999，预期行无色）验证通过：排序待查在前、关闭行无红、类目分布正确。

## 关键改动文件

| 文件 | 改动 |
|---|---|
| `headquarters_retail_orders_fy_2025-05-01_2026-04-30.xlsx` | 新增（主报表 4 sheet：年度零售/支付明细/支付流水/年度零售明细，含七色米订单号列） |
| `headquarters_retail_orders_fy_with_detail.xlsx` | 新增（年度零售明细独立备份） |
| `add_headquarters_retail_detail_merged_xlsx.py` | 新建（克隆崇礼版，retarget 总部，EXCLUDE_CODES 空，all 统一源） |
| `export_all_orphan_records.py` | FILES 加总部 / categorize 总部转待查 / 排序权重 / 四→五店文案 / 动态计数 |
| `all_销售单列表_孤儿记录.xlsx` | 重算覆盖（124/210 → 84/150） |
| `CLAUDE.md` | 新增 2026-05-19（续2）开发日志 + 更新孤儿口径已知遗留 |

## 学到的小知识

1. **本机 ODBC = Apple Silicon 形态**：`/opt/homebrew/etc/odbcinst.ini` + Driver 18 = 脚本 `DEFAULT_CONN` 默认值，跑库脚本只加 `ODBCSYSINI=/opt/homebrew/etc` 即可，无 `--conn`。`brew --prefix unixodbc` 会挂起，探测 ODBC 别走 brew，直接读 odbcinst.ini + `pyodbc.drivers()`。CLAUDE.md 的 Intel Mac/Driver 13 笔记是异机，勿照搬。
2. **fy 导出不自带七色米订单号**：该列是四店事后单独加的（DB `retail.mi7_code` + `STUFF FOR XML PATH`），非 `export_retail_orders_fy.py` 产出。要做明细合并/孤儿核对，必须先补这列。原一次性脚本未留存，本会话按同口径重写后即删。
3. **跨店七色米号命中**：`consumed` 是全局七色米号集合，某店报表消费的号可能对应 `all` 里别店门店的单据。本会话南山待查 5→4 即因 1 个南山门店单据的七色米号被某总部零售单消费——属正确行为，归因务必按「七色米号是否被任一店消费」而非「单据所属门店是否有报表」。
4. **总部 7 单无号是真实异常非测试**：`ZB_LS_260104_00001~00007` 同日 2026-01-04、¥300–¥4600、全额已付。按用户固化口径（微额¥0.0x/¥0 无号才当测试剔除，实额缺号保留标红），`EXCLUDE_CODES` 留空，标红待业务确认为何漏录七色米引用。
5. **孤儿脚本残留硬编码**：原脚本末尾打印写死 `210 行级 / 124 单据级`，改店铺集合后会与实际不符；顺手改成动态计数，避免下次再加店时误导。
