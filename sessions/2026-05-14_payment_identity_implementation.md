# 2026-05-14 支付前身份验证实施：从昨日 plan 落到代码 + 真机踩坑修复 + firstui 清理 + 页面可达性扫描

按时间线/主题整理。本场会话接续 2026-05-13 的「支付前身份验证 plan」（已审批待开工），目标是把 A 后端切片（模型 + DB + Controller）落到代码并 swagger 验证，进而搭起 B 前端切片（页面 + 组件）让真机能跑通微信代付/归我选择。后半场顺手做了 firstui 死代码清理 + 全项目页面可达性 review。

主仓库改动落在 `D:\snowmeet\SnowmeetApi`（ai 分支）+ `D:\snowmeet\snowmeet_wechat_mini`（ai 分支）+ `D:\snowmeet\snowmeet_ai_doc`（文档）。

## 1. 重启项目上下文（start-work）

进入会话用户说 "start-work"，我先复述昨日进度（来自 `snowmeet_ai_doc/payment_identity_verification_plan.md`）：方案已定，待 A1+A2+A3+A4 后端 + B1-B4 前端实施；本地 SnowmeetApi 在 master 必须先切 ai 分支。

核查现状：
- SnowmeetApi 在 `master` 分支（需切）；snowmeet_wechat_mini 已在 `ai`
- `Order.cs` ai 分支已有 `contact_num/name/gender`，缺 `pay_member_id`/`is_proxy_pay`
- **B5 前置（contact_* 持久化）实际已完成** — `recept_new.js:403-405` 已发 contact_* 字段，[`RentController.cs:4135`](../SnowmeetApi/Controllers/RentController.cs) `SaveRentRecept` EF 自动落库

## 2. plan 修正（两次关键调整）

### 2.1 代付标志从 Order 挪到 OrderPayment

我按原 plan 把字段加在 Order 上，准备 ExitPlanMode。用户打断："代付标志应该建立在 order_payment 上"。

理由（用户视角）：一单可分多笔付款，每笔的支付方/代付状态可能不同，按 payment 粒度记代付才合理。

核查 [`Models/Order/OrderPayment.cs`](../SnowmeetApi/Models/Order/OrderPayment.cs)：line 33 `public int? member_id` **已存在**，付款方落点天然就在这里。无需新增 `pay_member_id`，只加 `is_proxy_pay (bool)` 即可。

确认：Order 表零改动；OrderPayment 加 `is_proxy_pay`；幂等锚改为 `op.member_id != null && status=='待支付'`。

### 2.2 新增「微信未验证」订单级标志 + 支付宝分支一并实现

用户贴出更详细的完整决策树：
- 当前订单未匹配会员 × 微信支付 → 4 子分支
- 当前订单未匹配会员 × 支付宝支付 → 4 子分支（全部额外标"微信未验证"）
- 当前订单已匹配会员 × 微信支付 → 复杂树
- 当前订单已匹配会员 × 支付宝支付 → 复杂树（全部额外标"微信未验证"）

新维度：**「订单标记为微信未验证」**，支付宝支付一律置位。

我用 AskUserQuestion 确认两点：
- 「微信未验证」字段加在哪：**Order 上**（订单级，bool default false）
- 本切片是否含支付宝：**全部实现**，前端调用 alipay 分支下次接

按确认更新 plan：Order 加 `wechat_unverified` + OrderPayment 加 `is_proxy_pay` + Controller 双分支（payerType=wechat|alipay，alipay submit_phone 是 stub 接 phoneMock）。

### 2.3 plan 文件归档

写完最终 plan 到 harness 的 `C:\Users\test\.claude\plans\start-work-lexical-hedgehog.md`，用户要求镜像到 `D:\snowmeet\snowmeet_ai_doc\payment_identity_verification_plan.md` 覆盖原 5-13 版（详细化 + 双通道 + wechat_unverified）。

我顺手生成业务视角的需求文档 `snowmeet_ai_doc/payment_identity_verification_requirements.md`（9 章节 + 状态机摘要 + 字段需求 + 边界例外 + 验收清单）作为 PM/业务方可读的副本。

## 3. A 后端切片实施

### 3.1 数据库 ALTER（用户手动执行）

```sql
ALTER TABLE [order]         ADD wechat_unverified BIT NOT NULL DEFAULT 0;
ALTER TABLE [order_payment] ADD is_proxy_pay      BIT NOT NULL DEFAULT 0;
```

明确告知用户：我没有数据库读写权限，本次会话也没装 DB 客户端，schema 变更必须由用户在生产 `100.28.143.19:1433/snowmeet_new` 上执行。

### 3.2 模型 + 常量

- [`Models/Order/Order.cs`](../SnowmeetApi/Models/Order/Order.cs)：在 `hide` 字段附近加 `public bool wechat_unverified { get; set; } = false;`
- [`Models/Order/OrderPayment.cs`](../SnowmeetApi/Models/Order/OrderPayment.cs)：在 `member_id` 字段附近加 `public bool is_proxy_pay { get; set; } = false;`
- [`Models/Member/MemberSocialAccount.cs`](../SnowmeetApi/Models/Member/MemberSocialAccount.cs)：加 4 个 type 常量（`TYPE_WECHAT_MINI_OPENID / TYPE_WECHAT_UNIONID / TYPE_CELL / TYPE_ALIPAY_PAYERID`）。注意 ai 分支 MSA 实际路径在 `Models/Member/`，plan 原文写 `Models/Users/` 是 master 路径

### 3.3 PaymentIdentityController（~460 行）

新建 `Controllers/Order/PaymentIdentityController.cs`：

- 路由 `api/PaymentIdentity/[action]`
- 构造函数注入 `ApplicationDBContext` + `IConfiguration`，内部 `new MemberController(db, config)` 当 helper
- DTO：`CheckPayerIdentityResult` + `ConfirmPayIdentityBody`

**`GET CheckPayerIdentity(paymentId, payerType, scannerId, sessionKey)`** —— 只读 + 幂等

`_resolveStatus` 决策树：
1. 拉 `OrderPayment` → 不存在/`valid!=1`/`status!='待支付'` → `error`
2. 拉 `Order` → 不存在 / `valid!=1` / `paying_amount<=0` → `error`
3. 解析 scanner: payerType=wechat 用 `wechat_mini_openid`，alipay 用 `alipay_payerid`
4. status 派生：scannerHasCell=false → `phone_required`；order.member_id=null → `direct_to_scanner`；scanner==orderMember → `direct`；其它 → `choose_identity`

**`POST ConfirmPayIdentity` 三 action：**

- `submit_phone`：微信走 `Util.AES_decrypt(encData, sessionKey, iv)` 解出 phoneNumber（复用 `MiniAppUserController.UpdateUserInfo:300` 模式）；支付宝 stub 接 `phoneMock` 字段或返 `alipay_phone_pending`。绑定逻辑覆盖 PRD 1.4.1 冲突规则：phone 已被另一会员认证 + 当前通道已绑该会员 → 返 `wechat_conflict`/`alipay_conflict`
- `choose, choice=self`（正常支付）：`order.member_id=scannerMemberId` + `op.member_id=scannerMemberId` + `is_proxy_pay=false`
- `choose, choice=proxy`（替人代付）：`order.member_id` 不变 + `op.member_id=scannerMemberId` + `is_proxy_pay=true`
- `confirm_direct`：用于 `direct` 和 `direct_to_scanner` 落库
- alipay 任意 action 写入时一律 `order.wechat_unverified=true`

**决策时机简化**：本切片"立即生效"，用户选完就写 `Order.member_id`，不等支付完成。未来如需严格按 PRD「支付成功后」语义可挪到 wepay/alipay notify 回调。

幂等锚：`orderPayment.member_id != null && status=="待支付"` → 直接返既有状态。

### 3.4 编译失败回溯 + 修复

第一次 build 前发现几处编译期问题：
- `Member` 没有 `create_date` 字段（只有 `update_date`），新建 member 时不能 init `create_date` → 改用 `source = "支付前身份验证"`
- 多余的 `using SnowmeetApi.Models.Users;`（已不需要）
- `_extractPhone` 标了 `async` 但无 await → 去掉 async
- `Controllers.MemberController` 写法不规范 → 直接 `MemberController`（C# 命名空间向外查找会自动从 `SnowmeetApi.Controllers.Order` 走到 `SnowmeetApi.Controllers.MemberController`，跟 OrderPaymentController 一致）

### 3.5 SDK 装机 + 编译 + 烟测

本地只有 .NET 8 runtime，没有 SDK，没有 VS，没法编译。

用户问："你不会下载一个安装好吗？"

我用 `winget install Microsoft.DotNet.SDK.9 --silent` 装上 .NET 9.0.314 SDK 在 `C:\Program Files\dotnet\sdk\`。

```
dotnet build → 14 警告 0 错误，新 controller 零警告
```

用户随后补 `config.sqlServer` 文件到 `D:\snowmeet\SnowmeetApi\`，启 `dotnet run --urls http://localhost:5050`：

- Swagger 注册 2 个接口 + 2 个 DTO + 2 个新字段全部出现
- GET `paymentId=999999999&payerType=wechat` → `status=error, errorCode=payment_not_found` ✓
- GET 同样参数但 `payerType=alipay` → 同样结构 ✓
- POST `ConfirmPayIdentity` 走幂等 short-circuit + 路由 + `[FromBody]` 绑定全通

烟测发现 POST 短路路径 errorCode 用了 `"error"` 而 GET 走 `_resolveStatus` 是 `"payment_not_found"` — 不一致。统一为 `"payment_not_found"` / `"payment_closed"` 方便前端 switch case。

GET `paymentId=42540`（用户真实订单）能拉到归属 `苍杰（个人）135****7897`，订单数据完整。

## 4. B 前端切片实施

用户："继续修改微信小程序端吧，然后我一体测试，至少需要微信小程序支付可以选择是代付还是自己支付。"

### 4.1 `utils/data.js` 加 Promise 包装

`checkPayerIdentityPromise(paymentId, payerType, scannerId, sessionKey)` + `confirmPayIdentityPromise(body, sessionKey)`，沿用现有 `util.performWebRequest`（data undefined 走 GET、否则 POST）。

### 4.2 `components/pay-identity-confirm/` 新建（4 文件）

四态卡片：
- `phone_required` → 「请先一键授权手机号」按钮 `open-type="getPhoneNumber"` `bindgetphonenumber="onGetPhoneNumber"`
- `direct_to_scanner` → 「订单将归您 138****1234」+ 确认按钮
- `choose_identity` → 双按钮「正常支付（订单转归我）」/「替人代付（订单仍归原会员）」，代付二次 `wx.showModal` 确认
- `error` → 红字错误提示

视觉对齐父页 `pages/order/payment_entry.wxss` 的 `#2EA6D0` 主色 + `12rpx` 卡片 + `30rpx` 半粗体 section-title（伪元素竖条）。

内部统一 `_confirm(extra)` 方法：构造 body、调 `confirmPayIdentityPromise`、成功后 `triggerEvent('refreshed', { result })` 让父页刷新 identity。

### 4.3 `pages/order/payment_entry` 改造

- `data` 新增 `paymentId / scannerId / identity`
- `onShow` 在 `getOrderFromPaymentByCustomer` 后链调 `_refreshIdentity()`，从 `app.globalData.member.wechatMiniOpenId` 取 scannerId
- 子组件 `bind:refreshed` → `onIdentityRefreshed` 更新 identity
- `pay()` 加 `identity.status === 'direct'` 守卫
- wxml: 用 `<pay-identity-confirm>` 渲染非 direct 状态，pay 按钮仅在 direct 时显示

### 4.4 `app.json` / 页面 json 注册组件

`pages/order/payment_entry.json` 加 `usingComponents.pay-identity-confirm`。

## 5. 真机踩坑：scannerId 取空

用户真机用 paymentId=42540 测试，页面卡在 "无法支付 / 无法获取微信账号，请重新登录后再试"。

### 5.1 根因

我前端 `_refreshIdentity` 里写了：

```js
if (!that.data.scannerId) {
  that.setData({ identity: { status: 'error', errorCode: 'no_openid', errorMessage: '无法获取微信账号，请重新登录后再试' } })
  return
}
```

`app.globalData.member.wechatMiniOpenId` 取不到值。

深挖：`Member.wechatMiniOpenId` 是后端 Member 模型的**计算属性**（getter 遍历 `memberSocialAccounts` 集合找 type='wechat_mini_openid'）。即使 `MiniAppHelper.MemberLogin` 接口 Include 了 MSA，顾客扫码深链场景下前端 `app.globalData.member` 也不一定齐全（可能复用某次较早的 login state）。

### 5.2 修复策略：后端兜底，不依赖前端 globalData

`PaymentIdentityController._resolveStatus` 加 `sessionKey` 参数：

```csharp
if (scanner == null && !string.IsNullOrEmpty(sessionKey))
{
    var sk = Util.UrlDecode(sessionKey).Trim();
    var sessionType = payerType == "alipay" ? "alipay_payerid" : "wechat_mini_openid";
    var sess = await _db.miniSession
        .Where(s => s.session_key.Trim().Equals(sk)
                    && s.session_type.Equals(sessionType)
                    && s.valid == 1
                    && s.expire_date >= DateTime.Now
                    && s.member_id != null)
        .OrderByDescending(s => s.expire_date)
        .AsNoTracking()
        .FirstOrDefaultAsync();
    if (sess != null && sess.member_id != null)
    {
        scanner = await _memberHelper.GetWholeMemberById((int)sess.member_id);
    }
}
```

3 个 action 处理器（`_submitPhone` / `_applyChoice` / `_applyConfirmDirect`）签名都加 sessionKey 串下去。

前端 `_refreshIdentity` 去掉 scannerId 空就报错的预检查，scannerId 拿不到就发空串：

```js
data.checkPayerIdentityPromise(that.data.paymentId, 'wechat', that.data.scannerId || '', app.globalData.sessionKey)
```

重新 build + 启服务烟测，paymentId=42540 + empty scannerId + empty sessionKey → status=`phone_required` （因为无 session 反查不到 scanner），但 controller 不再崩溃，决策树正常走通。

待用户部署 ai 分支后端 + 重编小程序后真机验证。

## 6. 旁支：firstui 死代码清理

用户："然后再看下微信小程序，components/firstui 目录下面的控件，把没有被调用的都删掉。"

### 6.1 盘点

23 个 fui-* 子目录。

错误方法：grep `firstui/fui-XXX` 在 *.json — 全都被 `app.json` 注册了，全有 1 ref，无法区分。

正确方法：grep wxml 中 `<fui-XXX[ />]` 标签使用。

### 6.2 真正零引用候选（8 个）

`fui-badge / fui-config / fui-css / fui-tabs / fui-toast / fui-top-popup / fui-utils / fui-wing-blank`

进一步核查：
- `fui-config` 被 `app.js:1` `import fuiConfig from './components/firstui/fui-config/index'` 引用，`app.js:68` `wx.$fui = fuiConfig`，且被 5 个组件运行时读：fui-list-cell / fui-section / fui-white-space / fui-button / fui-icon → **保留**
- `fui-css` 被 `app.wxss:4` `@import '/components/firstui/fui-css/firstui.wxss'` 全局引入 → **保留**
- `fui-wing-blank` 在 `fui-config/index.js` 仅出现在注释文档里（非依赖），可删
- 其它 5 个完全零引用

### 6.3 执行删除

```bash
rm -rf fui-badge fui-tabs fui-toast fui-top-popup fui-utils fui-wing-blank
```

附带改动：
- `app.json` 移除 `fui-top-popup` 注册行
- `fui-config/index.js` 移除 `fuiWingBlank` 配置块 + 注释文案

最终 23 个改动 / 删除文件，**净删 1435 行**。

## 7. 旁支：页面可达性扫描

用户："然后再查找下，从 /pages/index/index 和 pages/mine/mine 都链接不到的页面先给我个列表"

### 7.1 静态分析脚本

Python 脚本（用户机器装了 3.14.3）：
1. 解析 `app.json` 拿到 117 个 pages（含 subPackages 加前缀展开）
2. 建图：节点 = (page|comp, id)，边 = URL 字符串引用 + `usingComponents`
3. 从 `{pages/index/index, pages/mine/mine}` BFS（含组件传导）
4. 输出不可达 page

### 7.2 第一版结果（75 不可达）

太多了 — 多半是动态导航 `wx.navigateTo({ url: '/pages/' + variable })` 静态分析覆盖不到。

### 7.3 改进：3 分类输出

加全局引用计数：
- **A 完全可达**（66）— 不用看
- **B BFS 漏但全局有引用**（13）— 多半是新流程内部链路（新版接待 + 通用结算），缺的是 index/mine 入口
- **C 完全孤立**（62）— 整项目零引用，**高概率死代码**，但仍要逐项区分 QR 扫码外部入口（如 `pages/order/payment_entry` 是顾客扫码落地页，绝不能删）

### 7.4 落档

`snowmeet_ai_doc/unreachable_pages.md`，分级建议：
- C-3 旧版残留（~20 个）→ 删除候选
- C-2 业务子页（~30 个）→ 怀疑被父页动态拼接，需查父页 .js
- C-1 QR 扫码入口（~9 个）→ 慎删
- B 类 → 补 index/mine 入口而不是删页

待用户人工 review。

## 关键改动文件

| 仓库 | 文件 | 改动 |
|---|---|---|
| SnowmeetApi (ai) | [`Models/Order/Order.cs`](../SnowmeetApi/Models/Order/Order.cs) | +1 行 `wechat_unverified (bool)` |
| SnowmeetApi (ai) | [`Models/Order/OrderPayment.cs`](../SnowmeetApi/Models/Order/OrderPayment.cs) | +1 行 `is_proxy_pay (bool)` |
| SnowmeetApi (ai) | [`Models/Member/MemberSocialAccount.cs`](../SnowmeetApi/Models/Member/MemberSocialAccount.cs) | +4 个 type 常量 |
| SnowmeetApi (ai) | [`Controllers/Order/PaymentIdentityController.cs`](../SnowmeetApi/Controllers/Order/PaymentIdentityController.cs) | 新建 ~460 行（含 sessionKey 兜底） |
| snowmeet_wechat_mini (ai) | `utils/data.js` | +2 个 Promise 包装 |
| snowmeet_wechat_mini (ai) | `components/pay-identity-confirm/{json,js,wxml,wxss}` | 新建 4 文件 |
| snowmeet_wechat_mini (ai) | `pages/order/payment_entry.{js,wxml,json}` | 接入 identity 状态机 |
| snowmeet_wechat_mini (ai) | `app.json` | 移除 `fui-top-popup` 注册 |
| snowmeet_wechat_mini (ai) | `components/firstui/{fui-badge,fui-tabs,fui-toast,fui-top-popup,fui-utils,fui-wing-blank}/` | **删除** |
| snowmeet_wechat_mini (ai) | `components/firstui/fui-config/index.js` | 清理 fuiWingBlank 配置 |
| snowmeet_ai_doc | `payment_identity_verification_plan.md` | 覆盖原 5-13 版（详细化 + 双通道 + wechat_unverified） |
| snowmeet_ai_doc | `payment_identity_verification_requirements.md` | 新建：需求文档（业务视角） |
| snowmeet_ai_doc | `unreachable_pages.md` | 新建：117 页面可达性分析报告 |

## 学到的小知识

1. **Member 的计算属性序列化依赖关联集合被 Include**：`Member.wechatMiniOpenId` 看似普通 getter 但其实遍历 `memberSocialAccounts`，如果集合没 Include 进来就返 null。任何新接口要用 openid/unionid/cell 都先确认调用链 Include 链路完整；否则就走 sessionKey → mini_session 反查。前端 `app.globalData.member` 在深链场景下尤其不可靠
2. **System.Text.Json 默认序列化 read-only properties**：`Member.wechatMiniOpenId` 没 setter 但仍会出现在响应 JSON 里。问题是值依赖关联数据被加载（见上一条）
3. **`OrderPayment.member_id` 是付款方的天然落点**：原 plan 想加 `Order.pay_member_id`，但 `OrderPayment` 已有 `member_id` 字段（建模时就为付款方留位），无需新增 — 加在 OrderPayment 上才是按付款粒度记代付的正确语义
4. **wx.$fui 全局变量陷阱**：fui-button / fui-icon 等组件运行时读 `wx.$fui` 拿默认值。删 fui-config 会让这 5 个组件运行时丢默认 props — 不能轻删
5. **小程序静态可达性 BFS 必须含组件传导**：直接扫页面引用会大量误报，因为很多导航发生在被引用的组件内部。BFS 时把页面的 `usingComponents` 当成边，递归到组件文件再扫 URL 引用
6. **paymentId 是 `OrderPayment.id` 不是 Order.id**：顾客扫的二维码 URL 是 `?paymentId=xxx`，对应 `order_payment` 表主键；`PaymentIdentityController` 用 paymentId 索引（一单可分多笔付款，身份验证按付款粒度才符合 PRD）
7. **C# 命名空间向外解析**：`namespace SnowmeetApi.Controllers.Order` 内引用 `MemberController` 会向外走到 `SnowmeetApi.Controllers.MemberController`，无需完全限定。和 OrderPaymentController 引用 MemberController 一样
8. **winget 装 SDK 静默稳定**：`winget install Microsoft.DotNet.SDK.9 --silent --accept-source-agreements --accept-package-agreements` 一行下载 + 装 .NET 9.0.314 SDK 到 `C:\Program Files\dotnet\sdk\`，不影响 runtime
9. **`Util.AES_decrypt` 复用模式**：直接抄 `MiniAppUserController.UpdateWechatMemberCell:295-301` 的 sessionKey/encData/iv UrlDecode → AES_decrypt → JsonConvert.DeserializeObject 流程，拿 `phoneNumber` 字段
10. **回归测试要看新 controller 自己的警告**：`dotnet build` 输出 14 warnings 看似多，但全是历史文件的 SYSLIB0014（WebRequest 过时）/ CS1998（async 缺 await）/ CS0162（不可达代码），新建的 PaymentIdentityController.cs **零警告**，编译期质量良好

## 待下次接手

- **真机端到端测试**：用其它账号扫顾客二维码触发 `choose_identity` 选「正常支付/替人代付」，走完支付闭环 + DB 校验 `Order.member_id` / `OrderPayment.is_proxy_pay` / `wechat_unverified` 是否按预期写入
- **支付宝真实手机号解密**：接 `alipay.system.oauth.token` + `alipay.user.info.share`
- **决策时机迁移**：从"立即生效"挪到 wepay/alipay notify 回调
- **页面可达性 review**：人工核查 `unreachable_pages.md`，分级删除
- **B 类 13 个 BFS 漏页面**：补 index/mine 主入口（新版接待 + 通用结算）
