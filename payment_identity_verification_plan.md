# 支付前身份验证 — A 后端全套（A1+A2+A3+A4）

> 完整方案来源：`D:\snowmeet\snowmeet_ai_doc\payment_identity_verification_plan.md`（已审批）
> 本 plan **覆盖**原方案的两处设计：
> 1. 代付标志从 Order 挪到 OrderPayment
> 2. 支付宝分支在本切片一并实现（原方案推迟）
> 3. 新增「微信未验证」订单级标志（用户最新流程要求）

## Context

按 PRD V0.13 「1.4.5 现场开单流程 — 支付环节账户匹配逻辑」+「1.4.1 注册场景二 — 支付环节自动注册」+ 用户最新完整流程图（含支付宝 + 「微信未验证」标记），需要在顾客扫码支付落地页（`pages/order/payment_entry`）做账户匹配 + 代付识别 + 支付宝订单未微信验证标记。当前流程**完全无验证**，顾客扫码直接 `Order/WechatPayByOrderPayment` 调起付款。

今日工作单元只做后端：让 `PaymentIdentityController` 在 swagger 里能跑通 5 种 status 分支 × 2 种 payerType（wechat / alipay），后端验证完后再做小程序侧改造。

**字段总览（最终设计）：**
| 字段 | 表 | 状态 | 用途 |
|---|---|---|---|
| `member_id` | Order | 已存在 | 订单归属（PRD「订单归于 xx 名下」对应此字段）|
| `wechat_unverified` | Order | **新加 bool default false** | 订单是否经过微信账号验证。alipay 支付一律置 true |
| `member_id` | OrderPayment | 已存在 | 付款方 member |
| `is_proxy_pay` | OrderPayment | **新加 bool default false** | 该笔付款是否代付（订单归属 ≠ 付款方时为 true）|

**已经做完的（不在本切片）：**
- ✅ B5（前置）`recept_new.js` `saveRentReceptOrder` 已发 `contact_name/contact_gender/contact_num`（[`recept_new.js:403-405`](D:\snowmeet\snowmeet_wechat_mini\pages\admin\reception\recept_new.js)）
- ✅ 后端 EF 自动持久化（`Models.Order` 已有 contact_*；[`RentController.cs:4135`](D:\snowmeet\SnowmeetApi\Controllers\RentController.cs) `SaveRentRecept` 写库）

## 实施步骤（按依赖顺序）

### Step 0：切到 ai 分支（前置必做）
```
cd D:\snowmeet\SnowmeetApi
git checkout ai
```
工作树已 clean（核查过），安全切换。**切完后所有改动落 ai 分支。**

### Step 1 — A1：Order.cs + OrderPayment.cs 加字段

**`D:\snowmeet\SnowmeetApi\Models\Order\Order.cs`**：加
```csharp
public bool wechat_unverified { get; set; } = false;
```

**`D:\snowmeet\SnowmeetApi\Models\Order\OrderPayment.cs`**：加
```csharp
public bool is_proxy_pay { get; set; } = false;
```

### Step 2 — A2：DB schema（用户执行）
```sql
ALTER TABLE [order] ADD wechat_unverified BIT NOT NULL DEFAULT 0;
ALTER TABLE [order_payment] ADD is_proxy_pay BIT NOT NULL DEFAULT 0;
```
旧数据默认值 0，向后兼容。

### Step 3 — A3：MemberSocialAccount.cs 加 type 常量
**文件：** `D:\snowmeet\SnowmeetApi\Models\Member\MemberSocialAccount.cs`（ai 分支路径已核查）

在类内加：
```csharp
// 第三方 ID 类型常量（与 type 字段对照）
public const string TYPE_WECHAT_MINI_OPENID = "wechat_mini_openid";
public const string TYPE_WECHAT_UNIONID = "wechat_unionid";
public const string TYPE_CELL = "cell";
public const string TYPE_ALIPAY_PAYERID = "alipay_payerid";
```

> 现有代码中的字符串字面量本期不重构，新代码用常量。

### Step 4 — A4：新建 PaymentIdentityController.cs
**文件：** `D:\snowmeet\SnowmeetApi\Controllers\Order\PaymentIdentityController.cs`（新建 `Controllers\Order\` 子目录，与 `OrderController.cs` 等分组）

路由：`api/PaymentIdentity/[action]`，以 **paymentId** 为锚定。

#### 决策树（5 状态 × 2 payerType）

5 状态：`phone_required` / `direct` / `direct_to_scanner` / `choose_identity` / `error`

判定流程（与 payerType 无关，仅决定查 MSA 的 type 字段）：

```
1. 查 OrderPayment(paymentId)
   - 不存在 / valid != 1 / status != "待支付"          → error
2. 查 Order(orderPayment.order_id)
   - valid != 1 或 paying_amount <= 0                  → error
3. 解析 scanner member:
   - wechat: 按 (openid, "wechat_mini_openid") 查 MSA → scannerMemberId
   - alipay: 按 (payerid, "alipay_payerid") 查 MSA   → scannerMemberId
4. 查 scanner 是否绑 cell (type="cell" MSA): scannerHasCell
5. 状态判定:
   - scannerHasCell == false                          → phone_required
   - order.member_id == null                          → direct_to_scanner
   - scannerMemberId == order.member_id               → direct
   - 否则                                              → choose_identity
```

#### Endpoint 1: `GET CheckPayerIdentity`
参数：`paymentId, payerType={wechat|alipay}, scannerId, sessionKey`
- 微信：`scannerId` = openid
- 支付宝：`scannerId` = payerid

**只读 + 幂等**。返回 DTO：
```csharp
public class CheckPayerIdentityResult {
    public string status { get; set; }         // 5 种 + error
    public int paymentId { get; set; }
    public int orderId { get; set; }
    public string payerType { get; set; }      // wechat | alipay
    public int? orderMemberId { get; set; }
    public string orderMemberMaskedCell { get; set; }    // "138****1234"
    public string orderMemberName { get; set; }
    public int? scannerMemberId { get; set; }
    public bool scannerHasCell { get; set; }
    public string scannerMaskedCell { get; set; }
    public string errorMessage { get; set; }
}
```

#### Endpoint 2: `POST ConfirmPayIdentity`
请求 body：
```csharp
public class ConfirmPayIdentityBody {
    public int paymentId { get; set; }
    public string payerType { get; set; }      // wechat | alipay
    public string scannerId { get; set; }      // openid (wechat) | payerid (alipay)
    public string action { get; set; }         // submit_phone | choose | confirm_direct
    public string choice { get; set; }         // self | proxy（仅 action=choose）
    public string encData { get; set; }        // 仅 wechat + submit_phone
    public string iv { get; set; }             // 仅 wechat + submit_phone
    public string phoneMock { get; set; }      // alipay + submit_phone 的 stub 入参（本切片用，前端先不传）
}
```

**幂等锚**：进入时若 `orderPayment.member_id != null && status == "待支付"` → 直接调 CheckPayerIdentity 返既有 status，不重复写。

逻辑（按 action 分支）：

- **`action=submit_phone`** — 拿到 scanner 手机号后绑会员
  - `payerType=wechat`: 调 `Util.AES_decrypt(encData, sessionKey, iv)` 解出 phoneNumber（复用 `MiniAppUserController.UpdateUserInfo` 内部段）
  - `payerType=alipay`: **本切片 stub**：若 body 含 `phoneMock` 字段直接用；否则返 `{ code:1, errorCode:"alipay_phone_pending", message:"支付宝手机号解密待支付宝小程序对接" }`。已留 TODO 注释，未来 PR 接 alipay `getAuthCode` → 服务端 `alipay.system.oauth.token` + `alipay.user.info.share` API
  - 拿到 phone 后：
    - scanner 无 member → `MemberController.GetMemberByCell(phone)` 找已有；找不到 → 创建新 member + 加 cell + 加 scannerId（按 payerType 选 MSA type 常量）两条 MSA
    - scanner 有 member 但无 cell：
      - phone 未被任何会员认证 → 加 cell MSA + 更新 `member.cell_number`
      - phone 已被另一会员认证（PRD 1.4.1 规则）：
        - wechat 当前 + 该会员未绑 wechat → 把 openid/unionid 链到该会员 → scannerMember = 该会员
        - wechat 当前 + 该会员已绑 wechat → 返 `{ code:1, errorCode:"wechat_conflict" }`
        - alipay 当前 + 该会员未绑 alipay → 把 payerid 链到该会员 → scannerMember = 该会员
        - alipay 当前 + 该会员已绑 alipay → 返 `{ code:1, errorCode:"alipay_conflict" }`
  - 完成后再调 CheckPayerIdentity 返新 status（通常进入 `direct` / `direct_to_scanner` / `choose_identity` 分支）

- **`action=choose, choice=self`（"正常支付"）**：
  - `order.member_id = scannerMemberId`（订单切换归属到扫码方）
  - `orderPayment.member_id = scannerMemberId`
  - `orderPayment.is_proxy_pay = false`
  - **若 `payerType=alipay` → `order.wechat_unverified = true`**
  - 落库 → CheckPayerIdentity 返 `direct`

- **`action=choose, choice=proxy`（"替人代付"）**：
  - `order.member_id` 不变（订单仍归原会员）
  - `orderPayment.member_id = scannerMemberId`
  - `orderPayment.is_proxy_pay = true`
  - **若 `payerType=alipay` → `order.wechat_unverified = true`**
  - 落库 → CheckPayerIdentity 返 `direct`

- **`action=confirm_direct`**（用于 `direct` 和 `direct_to_scanner` 状态的最终落库）：
  - 来自 `direct`：`orderPayment.member_id = scannerMemberId`; `is_proxy_pay = false`
  - 来自 `direct_to_scanner`：`order.member_id = scannerMemberId`（首次锚定订单归属）+ OrderPayment 同上
  - **若 `payerType=alipay` → `order.wechat_unverified = true`**
  - 落库 → CheckPayerIdentity 返 `direct`

返回结构：复用 `ApiResult<CheckPayerIdentityResult>` 包裹（与 ai 分支既有风格对齐）。`code=1` 时附 `errorCode` + `message`（如 `wechat_conflict` / `alipay_conflict` / `alipay_phone_pending`）。

## 决策时机：立即生效 vs 支付完成才落地

本切片采用 **立即生效**：用户在 payment_entry 上点确认（归我/代付/确认归属）后即刻写 Order.member_id 和 OrderPayment.member_id/is_proxy_pay/wechat_unverified。

**理由：**
- 实现简单，端到端可在 swagger 单独验证
- 用户已选「正常支付」即表示意图归属切换，未付完成不会造成业务损失（员工事后看「该单归 xx 但未付」可正常追踪）
- 代付场景下 `order.member_id` 本来就不动，无回滚顾虑

**未来若需要严格按 PRD「支付成功后」语义**：把这部分写入逻辑挪到 `WechatPaymentNotify` / `AlipayPaymentNotify` 回调里（[`OrderPaymentController.cs` 现有 callback 入口](D:\snowmeet\SnowmeetApi\Controllers\Order\OrderPaymentController.cs)），ConfirmPayIdentity 只锚定 OrderPayment 的 pending 状态。属下个切片范围。

## 关键文件清单

| 文件 | 操作 |
|---|---|
| `D:\snowmeet\SnowmeetApi\Models\Order\Order.cs` | Edit — 加 `wechat_unverified (bool)` |
| `D:\snowmeet\SnowmeetApi\Models\Order\OrderPayment.cs` | Edit — 加 `is_proxy_pay (bool)` |
| `D:\snowmeet\SnowmeetApi\Models\Member\MemberSocialAccount.cs` | Edit — 加 4 个 type 常量 |
| `D:\snowmeet\SnowmeetApi\Controllers\Order\PaymentIdentityController.cs` | Write — 新建 |
| 复用：`Controllers\User\MemberController.cs` `GetMember` / `GetMemberByCell` / `UpdateDetailInfo` | 仅读 |
| 复用：`Controllers\User\MiniAppUserController.cs` `UpdateUserInfo` 的 AES_decrypt 段 | 抽公共方法或 inline 调用 |
| 复用：`Util.cs` `GetStaffBySessionKey` / `AES_decrypt` | 仅读 |

## 风险点

- **本地 SnowmeetApi 当前在 master**：必须先 `git checkout ai`，否则改的是 OrderOnline 而非 Order/OrderPayment
- **DB schema 改动用户手动执行**：脚本不动生产 DB
- **支付宝 submit_phone 是 stub**：本切片后端不能真的解密支付宝授权码；前端 alipay 分支调用 `submit_phone` 时若不传 `phoneMock` 会返 `alipay_phone_pending`。下次 PR 接真实支付宝 API
- **微信开发者工具 `getPhoneNumber` 不返真实号**：本切片不涉及前端，下次做 B 切片建议加 `?mockCell=` 后门
- **AES_decrypt 复用**：现成在 `MiniAppUserController.UpdateUserInfo`，建议本期直接调用其内部段（或抽 `Util.DecryptWechatPhone(encData, sessionKey, iv) → string` 公共方法）
- **MSA 路径**：plan 写 `Models/Users/...`，实际 ai 分支在 `Models/Member/MemberSocialAccount.cs`（已核查纠正）
- **`OrderPayment.member_id` 既有语义**：可能已被 `Order/WechatPayByOrderPayment` 等老路径填写，幂等条件用 `member_id != null && status==待支付` 初版判定；遇到冲突再细化
- **决策时机简化**：本切片"立即生效"，不严格按 PRD「支付成功后」。未来如需严格语义，挪到支付回调（下个切片）

## 验证清单

1. **编译通过**：`cd D:\snowmeet\SnowmeetApi && dotnet build`
2. **Swagger 手测 5×2 = 10 个分支**：本地 `dotnet run` → `http://localhost:5050/swagger`：
   - 微信 × {error, direct_to_scanner, direct, choose_identity, phone_required}
   - 支付宝 × {error, direct_to_scanner, direct, choose_identity, phone_required}
3. **ConfirmPayIdentity 写入 DB 校验**：
   - 归我（wechat）：`order.member_id = scannerMemberId`、`order.wechat_unverified = 0`、`order_payment.member_id = scannerMemberId`、`is_proxy_pay = 0`
   - 归我（alipay）：上面 + `order.wechat_unverified = 1`
   - 代付（wechat）：`order.member_id` 不变、`order.wechat_unverified = 0`、`order_payment.is_proxy_pay = 1`
   - 代付（alipay）：上面 + `order.wechat_unverified = 1`
   - direct_to_scanner 确认（wechat）：`order.member_id = scannerMemberId`、`order.wechat_unverified = 0`
   - direct_to_scanner 确认（alipay）：上面 + `order.wechat_unverified = 1`
4. **submit_phone 校验**：
   - wechat encData 解密成功 → 绑 cell + 进入下一 status
   - wechat phone 冲突 → `errorCode=wechat_conflict`
   - alipay 传 phoneMock → 走同样逻辑（仅 MSA type 用 alipay_payerid）
   - alipay 不传 phoneMock → `errorCode=alipay_phone_pending`
5. **幂等校验**：对已 `orderPayment.member_id != null && status==待支付` 的 payment 再调 ConfirmPayIdentity，应返既有 status 不重写
6. **回归**：跑一个老 payment（`is_proxy_pay=0` / `wechat_unverified=0`），现有 `Order/WechatPayByOrderPayment` 不受影响

## 不在本切片范围（下次工作单元）

- B1：`payment_entry.{js,wxml,wxss}` 改造
- B2：`components/pay-identity-confirm/` 新建
- B3：`utils/data.js` 加 `checkPayerIdentity` / `confirmPayIdentity` Promise 包装
- B4：`order-summary-card/index.wxml` 显示 contact_name / contact_num
- 支付宝真实手机号解密（接 alipay.system.oauth.token + alipay.user.info.share）
- 决策时机改为「支付完成后」语义（挪到 OrderPayment notify 回调）
