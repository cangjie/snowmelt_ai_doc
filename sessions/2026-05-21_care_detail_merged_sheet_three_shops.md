# 2026-05-21 养护财年明细合并 sheet（三店）：年度养护 × care/care_task 一对多 + 7 staff 列

接续 5-20 雪票财年的「年度雪票明细」一对多合并 sheet 模式，把同款方案推到养护业务。三店（万龙服务中心 / 南山 / 崇礼旗舰店）各自的 `*_care_orders_fy_2025-05-01_2026-04-30.xlsx` 都新增 sheet `年度养护明细`，同时按 `care_task.task_name + staff_id` 派生 7 个员工列。脚本 [`add_care_detail_merged_sheet.py`](../add_care_detail_merged_sheet.py) 全程参数化（`--xlsx --shop --start --end`），跨店复用零代码改动。

## 1. 需求澄清与 task_name 映射拍板

### 1.1 起始口径调整：6 列 → 7 列

- 用户首次提的列：`安全检查人 / 修刃人 / 打蜡人 / 刮蜡人 / 维修人 / 发板人`（6 列）
- 我先并行调研了 xlsx 现有结构、`SnowmeetApi/Models/Care/Care.cs`、`CareTask.cs`、雪票明细 sibling 脚本，然后查 DB `care_task.task_name` 取值发现「打蜡」歧义
- 第一次正在追问打蜡映射时用户中断，重新发了 7 列版本：`安全检查人 / 修刃人 / 机打蜡人 / 热打蜡人 / 刮蜡人 / 维修人 / 发板人`（拆「打蜡」为「机打蜡」「热打蜡」两列）

### 1.2 DB 里 task_name 的实际取值

- `care_task.task_name` 与打蜡相关的有三个值：`打蜡` (554)、`热蜡` (2424)、`机打蜡` (32)
- 用户拍板的映射：
  - **机打蜡人 = 仅 `机打蜡`**（32 条）
  - **热打蜡人 = `热蜡` ∪ `打蜡`**（2424 + 554，按 `care_id` 去重后 2821）
  - 即「打蜡」归到热打蜡侧，而非机打蜡侧
- 其余 5 列单一映射：`安全检查 / 修刃 / 刮蜡 / 维修 / 发板`

## 2. 脚本实现 [`add_care_detail_merged_sheet.py`](../add_care_detail_merged_sheet.py)

### 2.1 总体结构（仿 `add_skipass_detail_merged_sheet.py`）

- 参数：`--xlsx <文件名> --shop <店铺名> [--start YYYY-MM-DD] [--end YYYY-MM-DD]`，默认财年 `2025-05-01 ~ 2026-04-30`
- 数据流：原 sheet `年度养护` 订单级数据 + DB `care` 表（按 `shop + start_date 区间 + order_id` 拉） + `care_task` 派生 7 staff 列 → 一对多展开 → 多 care 单整行浅蓝 + 订单级列合并单元格
- 数据库连接走老路：`DRIVER={ODBC Driver 18 for SQL Server}`，`export ODBCSYSINI=/opt/homebrew/etc` 是 macOS 必备
- 新 sheet 名 `年度养护明细`；视觉/合并/上色全部沿用雪票明细的风格

### 2.2 字段构成

订单级列（来自 `年度养护`，合并单元格） + care 字段（15 列：`care_id / order_id / member_id / member_name / phone / staff_id / staff_name / start_date / end_date / total_amount / closed / valid / shop_id / hidden / memo` 等核心字段） + 7 staff 列

7 staff 列派生逻辑（每个 care 取所有 task 中匹配 task_name 的 `staff_id` JOIN `staff.name`）：

```python
TASK_NAME_MAP = {
    '安全检查人': ['安全检查'],
    '修刃人':     ['修刃'],
    '机打蜡人':   ['机打蜡'],
    '热打蜡人':   ['热蜡', '打蜡'],
    '刮蜡人':     ['刮蜡'],
    '维修人':     ['维修'],
    '发板人':     ['发板'],
}
```

同一 care 同一列若多个员工 → 分号 `; ` 连接去重。

### 2.3 上色 / 合并细节

- 多 care 订单：订单级列上色 `EAF2FB` 浅蓝（与 5-20 雪票明细同款），明细列**含 7 staff 列也一起上色**（5-20 的教训）
- 单 care 订单：保留单行白底
- 无 care 订单：保留单行白底（不丢订单）
- 订单级列做垂直合并单元格 → 减少视觉重复

## 3. 万龙服务中心跑通 + 七列 DB 对账

### 3.1 第一次跑

- `python3 -u add_care_detail_merged_sheet.py --xlsx wanlong_service_care_orders_fy_2025-05-01_2026-04-30.xlsx --shop 万龙服务中心`
- 中途几次试图用 zsh heredoc 写校验脚本失败（中文被 mangle），改成把校验写成 `/tmp/verify_care_detail.py` 真文件 + `iter_rows` 读 xlsx 才稳定

### 3.2 对账结果

- 订单 4014 / 多 care 单 336 / care 总数 4394 / 有员工 3817 / 明细行 4981 / 列数 85（63 订单级 + 15 care + 7 staff）
- 7 staff 列填充数 × DB JOIN `staff` 期望数：**全部零差异 ✓**
- 关键验证：热打蜡人列正确合并 `热蜡`(2313) ∪ `打蜡`(508) = **2821**（去重 care_id），与 DB 直查一致

## 4. 南山 + 崇礼旗舰店并行跑通

### 4.1 店铺名查询

- 先查 DB `shop.name` 确认非「南山店」也非「崇礼店」，实际是 `南山` / `崇礼旗舰店`（万龙侧是 `万龙服务中心`）

### 4.2 并行执行 + 对账

- 两店并行 `python3 -u add_care_detail_merged_sheet.py --shop 南山|崇礼旗舰店`，各自跑完后用 `/tmp/verify_care_detail.py` 做对账
- 全部零差异 ✓

### 4.3 三店汇总

| 店铺 | 订单 | 多care单 | care总数 | 有员工 | 明细行 | 列数 |
|------|------|---------|---------|--------|--------|------|
| 万龙服务中心 | 4014 | 336 | 4394 | 3817 | 4981 | 85 |
| 南山 | 86 | 5 | 92 | 58 | 92 | 76 |
| 崇礼旗舰店 | 23 | 5 | 28 | 10 | 31 | 76 |

万龙服务的订单级列比另两店多 9 列（83 vs 76），是历史脚本累积差异，本次不动。三店 7 staff 列 × DB 期望全部 ✓。

## 关键改动文件

| 文件 | 改动 |
|---|---|
| [`snowmeet_ai_doc/add_care_detail_merged_sheet.py`](../add_care_detail_merged_sheet.py) | 新建。参数化（`--xlsx --shop --start --end`），加「年度养护明细」sheet，含 7 staff 列 |
| `snowmeet_ai_doc/wanlong_service_care_orders_fy_2025-05-01_2026-04-30.xlsx` | 加 sheet `年度养护明细`（4981 行 × 85 列） |
| `snowmeet_ai_doc/nanshan_care_orders_fy_2025-05-01_2026-04-30.xlsx` | 加 sheet `年度养护明细`（92 行 × 76 列） |
| `snowmeet_ai_doc/chongli_care_orders_fy_2025-05-01_2026-04-30.xlsx` | 加 sheet `年度养护明细`（31 行 × 76 列） |

## 学到的小知识

1. **`care_task.task_name` 三种「打蜡」相关值**：`打蜡` / `热蜡` / `机打蜡`。业务口径下「打蜡」归到「热打蜡」侧，与「热蜡」合并去重 care_id，机打蜡仅 `机打蜡` 一种。新做养护类报表时这是必须先和业务对齐的歧义点
2. **同一 care 多个 task 同一类型**：可能出现一个 care_id 在 `care_task` 表里有多条同 task_name 但不同 staff_id 的记录，派生 staff 列时必须去重并 `; ` 连接（与雪票明细多票兜底口径同理）
3. **zsh heredoc 处理中文字符串易 mangle**：Python `<<EOF` 内含中文 task_name 时 `pyodbc` 收到的可能是乱码。改成写真 `.py` 文件用 `python3 -u file.py` 跑，稳定可复现
4. **`iter_rows` 比 `cell(r,c)` 快几个数量级**：4981 行 × 85 列校验，前者秒级、后者卡住超时；大 xlsx 校验默认走 `iter_rows`
5. **shop 名要先查 DB**：`shop.name` 中三店是 `万龙服务中心` / `南山` / `崇礼旗舰店`（不带"店"），不能直接拍脑袋拼，会拉到 0 条 care
