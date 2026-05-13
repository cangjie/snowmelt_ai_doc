# 支付前身份验证（按你的 4 条修正版）

> 保存日期：2026-05-13
> 来源：start-work 探索 + PRD V0.13 + 你的 4 条修正
> 状态：待审批，未开工

## Context
按 PRD V0.13（`D:\snowmeet\snowmeet_ai_doc\PRD.docx`）+ 你刚给的 4 条修正实施支付前账户匹配。当前 `pages/order/payment_entry` 顾客扫码后**直接发起微信支付**，无任何会员匹配/手机号验证 — 不符合 PRD「会员匹配仅在支付时进行 + 代付订单必须验证手机号授权」要求。手机号验证用微信 `getPhoneNumber` 一键授权，**不需短信**。

## 你的 4 条修正（已对齐）
1. **订单表是 `order`，不是 `order_online`** — 在 `origin/ai` 分支的 `Models/Order/Order.cs`，本地 SnowmeetApi 当前在 master 必须先切 ai 分支
2. 扫码方 openid **未验证手机号** → 必须先验证（前置步骤）
3. 开单已匹配会员 + 扫码方不是同一会员 → **询问支付身份**（归我 / 代付 二选一）
4. 开单时未匹配会员 → 订单**直接归扫码方**

## 关键事实（已核实）

### `origin/ai` 分支后端
- `Models/Order/Order.cs` `[Table("order")]`：**已有** `id, code, shop, type, contact_num, contact_name, contact_gender, member_id (int?), name, gender, cell, total_amount, staff_id, valid` 等字段
- 缺：`pay_member_id (int?)` 和 `is_proxy_pay (bool)`
- `Controllers/OrderController.cs` route `api/[controller]/[action]` → 对应前端 `Order/PlaceRentOrder` 等
- `Models/Users/Member.cs` + `MemberSocialAccount.cs`（type: `wechat_mini_openid` / `wechat_unionid` / `cell`，**缺 `alipay_payerid` 常量**）
- `MemberController.cs`：`GetMember(num,type)` / `GetMemberByCell()` / `UpdateDetailInfo()` 三件套
- `MiniAppUserController.UpdateUserInfo`：已有 `AES_decrypt(encData,sessionKey,iv)` 解 phoneNumber → 直接复用
- 项目无 `Migrations/` 目录无 `.sql` 文件，DB schema 手动 ALTER

### 前端 (`snowmeet_wechat_mini` ai 分支)
- `pages/order/payment_entry.{js,wxml,wxss}` 当前**完全无身份验证**，pay() 直接调 `Order/WechatPayByOrderPayment`
- `components/auth/auth.js:151-166` + `components/user_info/auth_cell.js` + `pages/payment/pay_hub.js:100-119` 有 `getPhoneNumber` 现成实现可复用
- `pages/admin/reception/recept_entry` 已收 `customerCell`（L45-149），但未持久化到 `order.contact_num` — 需在 `Rent/SaveRentRecept` payload 加这字段（确认下后端 SaveRentRecept 是否已写 order.contact_num）

## 决策树（4 状态 + 1 错误）

### 输入
扫码方 `openid` + 订单 `orderId` + 当前 `sessionKey`

### 步骤 P：扫码方 openid 是否已绑认证手机号？
- 否 → status = `phone_required` — 需一键授权
- 是 → 进入步骤 D

### 步骤 D：根据 order.member_id 判断
| 条件 | status | 含义 |
|---|---|---|
| order.member_id 为空 | `direct_to_scanner` | 订单归扫码方（你的第 4 条）|
| order.member_id == scanner.member_id | `direct` | 同一会员，直接付 |
| order.member_id != scanner.member_id | `choose_identity` | 询问归我/代付（你的第 3 条）|

### 异常
- 订单不存在 / 已支付完成 / openid 缺失 → `error`

## 实施方案

### A. 后端（先 `git checkout ai`）

**A1. `Models/Order/Order.cs`** — 加 2 字段
```csharp
public int? pay_member_id { get; set; }
public bool is_proxy_pay { get; set; } = false;
```

**A2. 数据库**（你执行）：
```sql
ALTER TABLE [order] ADD pay_member_id INT NULL;
ALTER TABLE [order] ADD is_proxy_pay BIT NOT NULL DEFAULT 0;
```

**A3. `Models/Users/MemberSocialAccount.cs`** — 加 type 常量 `alipay_payerid`（仅常量声明 + 注释，本期不实施支付宝小程序流程）

**A4. 新建 `Controllers/Order/PaymentIdentityController.cs`** route `api/PaymentIdentity/[action]`

**Endpoint 1：`GET CheckPayerIdentity(orderId, payerType="wechat", openid, sessionKey)`**

返回 DTO：
```
{
  status: "phone_required" | "direct" | "direct_to_scanner" | "choose_identity" | "error",
  orderId: int,
  orderMemberId: int?,
  orderMemberMaskedCell: string?,    // "138****1234"
  orderMemberName: string?,
  scannerMemberId: int?,
  scannerHasCell: bool,
  scannerMaskedCell: string?,
  errorMessage: string?
}
```

逻辑：
1. 拉 order；不存在/已付 → error
2. 按 (openid, wechat_mini_openid) 查 scanner member；无则建临时态（status 仍按手机号判断）
3. 查 scanner.cell（按 type=cell 的 MSA）
4. 若无 cell → `phone_required`
5. 若 order.member_id 为空 → `direct_to_scanner`
6. 若 scannerMemberId == orderMemberId → `direct`
7. 否则 → `choose_identity`

**Endpoint 2：`POST ConfirmPayIdentity`**

Body：
```
{
  orderId, payerType, openid,
  action: "submit_phone" | "choose",   // 来自 phone_required / choose_identity
  choice: "self" | "proxy",            // action=choose 时
  encData: string?,                    // action=submit_phone 时（getPhoneNumber 返回）
  iv: string?,
  sessionKey
}
```

逻辑：
- `action=submit_phone`：复用 `MiniAppUserController.UpdateUserInfo` 解密 → 解出 phone
  - 若 scanner 还无 member → `MemberController.GetMemberByCell(phone)` 找已有；找不到 → `MemberController` 创建新 member 并加 cell + openid 两个 MSA → scannerMember = newMember
  - 若 scanner 已有 member 但无 cell：
    - 该 phone 未被任何会员认证 → 加 cell MSA 到 scannerMember + 更新 member.cell_number
    - 该 phone 已被另一会员认证（PRD 1.4.1 规则）：
      - 当前是微信ID + 该会员未绑微信 → 把 openid 链到该会员 → scannerMember = 该会员
      - 当前是微信ID + 该会员已绑微信 → return `{ success:false, errorCode:"wechat_conflict", message:"该手机号已绑其他账户，请换号或换微信" }`
      - 当前是支付宝ID → 直接绑（本期不做）
  - 完成后递归调 CheckPayerIdentity 返回新 status
- `action=choose, choice=self` (归我)：
  - order.member_id = scannerMemberId
  - order.pay_member_id = scannerMemberId
  - is_proxy_pay = false
- `action=choose, choice=proxy` (代付)：
  - order.member_id 不变
  - order.pay_member_id = scannerMemberId
  - is_proxy_pay = true
- 其他自动分支（direct / direct_to_scanner）也走 ConfirmPayIdentity 写入对应 member_id/pay_member_id

幂等：CheckPayerIdentity 必须只读；ConfirmPayIdentity 进入时若 order.pay_member_id 已非空 → 直接返既有结果

### B. 小程序（已在 ai 分支）

**B1. `pages/order/payment_entry.{js,wxml,wxss}` 改造**：
- onShow 拉单后立即调 `data.checkPayerIdentity(orderId, 'wechat', sessionKey)`
- pay() 前必须 `identityConfirmed === true`，否则按钮 disabled
- 5 种 status 对应 UI

**B2. 新建 `components/pay-identity-confirm/`**（4 文件）：
- 手机号一键授权 button（`open-type="getPhoneNumber"` + `bindgetphonenumber`）
- 身份选择 modal（归我/代付 二选一 + 取消）
- 错误提示

**B3. `utils/data.js`** 加 `checkPayerIdentity` / `confirmPayIdentity` 两个 promise 包装

**B4. `components/order-summary-card/index.wxml`** 加 `order.contact_name` + `order.contact_num` 显示

**B5. （前置）`pages/admin/reception/recept_new.js`** — `saveRentReceptOrder` payload 必须带 `contact_num` / `contact_name` / `contact_gender`（已收在 customerCell/customerName/customerGender），否则后端 order 永远没 contact_num，决策树无意义

### C. payment_entry 状态机

| status | UI | 用户动作 | 触发 |
|---|---|---|---|
| `phone_required` | "请先验证手机号" + getPhoneNumber 按钮 | 点授权 → ConfirmPayIdentity(submit_phone, encData, iv) | 转步骤 D 重新 check |
| `direct` | 原"敬请支付"按钮 | 点击 → pay() | WechatPayByOrderPayment |
| `direct_to_scanner` | "订单将归您 138****" + 确认 | 点确认 → ConfirmPayIdentity 写 member_id | 转 direct |
| `choose_identity` | modal: ①归我（订单转到我账户）②代付（订单仍归 138****1234）+ 取消 | 选①/②→ ConfirmPayIdentity | 转 direct |
| `error` | 红字错误提示，无支付按钮 | / | / |

## 关键文件清单
- 后端（origin/ai）：
  - `D:\snowmeet\SnowmeetApi\Models\Order\Order.cs`（加 2 字段）
  - `D:\snowmeet\SnowmeetApi\Models\Users\MemberSocialAccount.cs`（加 alipay_payerid 常量）
  - `D:\snowmeet\SnowmeetApi\Controllers\Order\PaymentIdentityController.cs`（新建）
  - `D:\snowmeet\SnowmeetApi\Controllers\User\MemberController.cs`（复用）
  - `D:\snowmeet\SnowmeetApi\Controllers\User\MiniAppUserController.cs`（复用解密代码段）
- 小程序（ai）：
  - `D:\snowmeet\snowmeet_wechat_mini\pages\admin\reception\recept_new.js`（持久化 contact_*）
  - `D:\snowmeet\snowmeet_wechat_mini\pages\order\payment_entry.{js,wxml,wxss}`
  - `D:\snowmeet\snowmeet_wechat_mini\components\pay-identity-confirm\`（新建）
  - `D:\snowmeet\snowmeet_wechat_mini\components\order-summary-card\index.wxml`
  - `D:\snowmeet\snowmeet_wechat_mini\utils\data.js`
  - `D:\snowmeet\snowmeet_wechat_mini\components\auth\auth.js`（参考样板）

## 风险点
- **本地 SnowmeetApi 必须先 `git checkout ai`**，否则改的是 OrderOnline 而非 Order
- 微信开发者工具 `getPhoneNumber` 不返真实号 → 必须真机调试；建议加 `?mockCell=` 开发后门
- CheckPayerIdentity 必须只读幂等；ConfirmPayIdentity 进入先判 order.pay_member_id != null 直接返既有结果
- 切换支付方式（微信→支付宝二维码）需重做 CheckPayerIdentity；本期支付宝固定返 error
- 老订单 member_id NULL 兼容 — 走 direct_to_scanner 路径，对老报表无影响
- `phone_required` → 「该手机号已绑其他微信」按 PRD 1.4.1 拒绝，提示用户换号或换微信
- `recept_new` 前置改动**必须先做**，否则所有新订单 contact_num 都为空，全走 direct_to_scanner

## 验证清单
1. 后端单测 5 种 status：构造各种 (order.member_id, scanner_openid 关联状态, scanner.cell 状态) 组合
2. 前端真机扫码每种 status UI 路径走通到付款
3. 端到端：recept_entry 录手机号 → settle 生二维码 → 顾客真机扫 → 走完支付。覆盖：
   - 顾客本身就是开单手机号会员（direct）
   - 顾客是其他会员，开单匹配了别人（choose_identity → 归我）
   - 顾客是其他会员，开单匹配了别人（choose_identity → 代付）
   - 开单未填手机号（direct_to_scanner）
   - 顾客是新人（phone_required → 一键授权 → 转 direct_to_scanner / 等）
4. 回归：现有订单（contact_num 空）支付仍走 direct_to_scanner，不破坏老流程

## 待你晚些时候确认的开放问题
- 「归我」是否要把订单原归属人（开单匹配的会员）的关系完全清掉？还是保留某种"原顾客"字段以便统计？
- 代付订单退款是否走支付方账户？（PRD 1.4.5 写「代付订单退款原路退回至支付账户」— 需要前后端一起处理）
- 「phone_required」失败提示应该挂在 payment_entry 还是引导到统一的会员中心页（pay_hub 之类）？
- 后端 `Rent/SaveRentRecept` 是否已经把前端 payload 中的 `contact_num/contact_name/contact_gender` 写入 order 表？需要看 ai 分支 RentController 实现
