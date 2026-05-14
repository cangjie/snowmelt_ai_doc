# 小程序页面可达性分析

> 生成日期：2026-05-14
> 入口：`pages/index/index` + `pages/mine/mine`
> 方法：静态分析 — 解析 app.json 拿到 117 个 pages，递归扫描每个 page + 其依赖组件的 `.js/.wxml/.json/.wxss` 中 `/pages/xxx` 字符串引用，从两个入口做 BFS

## 总览

| 类别 | 个数 | 含义 |
|---|---|---|
| A. 完全可达 | 66 | 从 index/mine 经 BFS 能到 |
| B. BFS 不可达但全局有引用 | 13 | 引用方可能是其它非可达页面、组件外字符串、动态拼接等。需人工 review |
| C. 完全孤立 | 62 | 整个项目里除了自己目录，零引用。**高概率死代码**，但仍需考虑 QR 扫码 / 外部链接入口 |

## A. 完全可达（不用看）

66 个，略。

## B. BFS 不可达但全局有引用（13 个，先看）

引用数仅统计目录外的字符串匹配。

| 页面 | 引用数 | 备注 |
|---|---|---|
| `pages/admin/care/order_detail` | 3 | |
| `pages/admin/fire/fire_order_detail` | 2 | |
| `pages/admin/recept/recept_member_info` | 1 | 新版 reception/recept_new.js:111 临时复用旧页（待新版会员详情页完成） |
| `pages/admin/reception/recept_entry` | 1 | 新版接待流程第一步 — 入口应是 index/mine 的某个按钮，但似乎只在内部链路被引用 |
| `pages/admin/reception/recept_new` | 1 | 新版接待 — 由 reception/recept_entry 跳入 |
| `pages/admin/reception/recept_package` | 1 | 新版接待 — 由 reception/recept_new 跳入 |
| `pages/admin/rent/rent_details` | 2 | |
| `pages/admin/retail/retail_order_detail` | 2 | |
| `pages/mine/ticket/ticket_share` | 3 | 由 mine/ticket/ticket_detail 跳入 |
| `pages/payment/pay_recept` | 1 | |
| `pages/payment/settle/index` | 1 | 通用结算页 — 由新版 reception/recept_new 跳入 |
| `pages/ski_pass/ski_pass_selector` | 1 | |
| `pages/template/stitch/_5/index` | 3 | stitch 模板原型页，本身不是生产用页面 |

**B 类共同特点：** 这条链路本身是活的（新版接待 + 通用结算 = 现在主力开发的流程），只是 BFS 起点是 index/mine 还没接通新流程的入口。你需要看的不是该不该删，而是 **该不该补 index/mine 上对应入口**。

## C. 完全孤立（62 个，高概率死代码）

下面这些页面在整个项目中除自己目录外，**零引用**。但其中一部分是 **QR 扫码 / 外部链接** 的落地页（无法被静态分析覆盖），需要人工区分：

### C-1：已知 / 怀疑是 QR 扫码 / 外部链接落地页（**慎删**）

| 页面 | 我的判断 |
|---|---|
| `pages/order/payment_entry` | **顾客扫码支付落地页**，员工生成的二维码 URL 指向这里。你刚改造过的页面 — **不能删** |
| `pages/tickets/get_ticket_from_channel` | 渠道领票落地页（带 `getPhoneNumber` 入口），多半是 H5 / 推广链接进来 |
| `pages/tickets/me_pick` | 票务自取页 |
| `pages/tickets/tickets_get` | 票务领取页 |
| `pages/register/staff_check_in` | 员工签到，可能是 staff_reg_qrcode 生成的二维码落地点 |
| `pages/register/out_reg` | 外部注册页 |
| `pages/register/reg` | 注册页 |
| `pages/admin/staff_reg` | 员工注册 — 跟 staff_reg_qrcode 配套 |
| `pages/logs/logs` | 微信小程序模板默认调试日志页，**留着无妨** |

### C-2：业务子页（疑似从某 list 页动态跳入但被 BFS 漏掉，需查父页 JS）

| 页面 | 疑似父页 |
|---|---|
| `pages/admin/deposit/deposit_charge` | deposit/deposit_balance? |
| `pages/admin/deposit/deposit_detail` | deposit/deposit_list? |
| `pages/admin/fd/fd_cart` | fd/fd_category_prod_list? |
| `pages/admin/fd/fd_order_confirm` | fd_cart? |
| `pages/admin/fd/fd_order_detail` | fd/fd_order_list? |
| `pages/admin/retail/retail_order_list` | admin/admin 列表项? |
| `pages/admin/sale/order_detail` | sale/shop_sale_entry? |
| `pages/admin/sale/shop_sale` / `shop_sale_entry` / `mod_mi7_order_no` | sale 模块 |
| `pages/admin/scan/scan` | 通用扫码工具页? |
| `pages/admin/ski_pass/common_skipass_detail` | common_skipass_list? |
| `pages/admin/ski_pass/nanshan_refund_detail` | nanshan_refund? |
| `pages/admin/ski_pass/nanshan_reserve_detail` | nanshan_reserve? |
| `pages/admin/ski_pass/nanshan_verify` | nanshan_pick_card_scan? |
| `pages/admin/staff/staff_detail` | staff/staff_list? |
| `pages/admin/ticket/ticket_unuse_list` | admin/admin? |
| `pages/admin/unipay/unipay_detail` | unipay/unipay? |
| `pages/admin/unipay/unipay_list` | unipay 模块 |
| `pages/admin/rent/new_rent_list` | admin/admin? |
| `pages/admin/rent/pay_additional` | rent_details? |
| `pages/admin/rent/rent_item_change` | rent_details? |
| `pages/admin/rent/rent_list_by_cell` | 业务工具页 |
| `pages/admin/rent/rent_report` | 报表 |
| `pages/admin/rent/set_award` | 设置奖励 |
| `pages/admin/rent/settings/rent_product` | rent_product_list? |
| `pages/ski_pass/nanshan_overtime_reserve` | ski_pass_selector? |
| `pages/ski_pass/ski_pass_reserve` | ski_pass_selector? |
| `pages/ski_pass/skipass_detail` / `skipass_detail_new` | my_skipasses? |

### C-3：旧版残留 / stitch 模板（**可优先考虑删**）

| 页面 | 备注 |
|---|---|
| `pages/admin/recept/recept_new` | **旧版接待主页**，新版在 `pages/admin/reception/recept_new`。CLAUDE.md 已写"新版替换旧版" |
| `pages/admin/printer/gprinter/print_task` | 打印任务页 |
| `pages/admin/printer/gprinter/ticket` | 票据打印 |
| `pages/admin/background/set_session_key` | 调试用 |
| `pages/blt/open_lock` | 蓝牙开锁页 — 项目方向已变? |
| `pages/claude/index` / `tickets` | 早期 claude 系列实验页 |
| `pages/experience/pay_temp` | 临时支付页 |
| `pages/mine/maintain/bind_maintain_order` / `order_detail` / `task` | mine/maintain 系列（mine/maintain/order_list 是可达的） |
| `pages/mine/my_maintain/my_maintain` / `my_maintain_detail` | mine 旧版养护 |
| `pages/mine/skipass/my_skipass` | 单数版（`my_skipasses` 复数版可达） |
| `pages/mine/ticket/ticket_bind` / `ticket_detail` | mine 票务子页（list 可达，但 detail/bind 静态无引用 — 可能动态跳） |
| `pages/order/order_entry` | 订单入口 — 跟 payment_entry 类似可能是扫码落地 |
| `pages/payment/pay_hub` / `rent_pay_add` / `uni_pay` | payment 子包多个页面 |
| `pages/rent/bind_rent_order` | 租赁绑定 |
| `pages/shop_sale/order_info` / `shop_landing` | shop_sale 子包 |

## 建议下一步

1. 先把 **C-3 旧版残留** 类目删（已确认无 BFS 路径 + 全局零引用）
2. **C-1 慎删**：人工排查是否还有 QR 扫码 / 外部链接指向（看后端 / 老二维码）
3. **C-2 业务子页**：怀疑还活，需要打开父页 .js 看动态 url 拼接逻辑
4. **B 类**：补 index/mine 入口而不是删
