# Snowmeet AI — 项目上下文

## 项目概览
滑雪场管理系统，包含两个子项目：
- `snowmeet_wechat_mini/` — 微信小程序客户端（原生小程序 + JS）
- `SnowmeetApi/` — 后端 API 服务（ASP.NET Core 9.0 + C# + SQL Server）

---

## 技术栈

**客户端 (snowmeet_wechat_mini)**
- 原生微信小程序（JS，无 TypeScript）
- UI 库：Vant WeApp、FirstUI、WeUI
- 工具库：linq.js
- 开发工具：微信开发者工具（WeChat DevTools）
- AppID：`wxd1310896f2aa68bb`

**服务端 (SnowmeetApi)**
- ASP.NET Core 9.0 / C#
- ORM：Entity Framework Core 9.0.6（SQL Server）
- 第三方：微信支付 TenpayV3、支付宝 SDK、腾讯云 OCR、NPOI（Excel）、QRCoder、ImageSharp
- API 文档：Swagger UI（`/swagger`）

---

## 启动命令

**客户端：** 用微信开发者工具打开 `snowmeet_wechat_mini/` 目录

**服务端：**
```bash
cd SnowmeetApi
dotnet run
# Swagger: https://localhost:5000/swagger
```

---

## 项目结构要点

**客户端核心路径：**
- `app.js` — 全局入口，globalData 管理
- `pages/` — 110 个页面（ski_pass、rent、tickets、order、admin、claude 等）
- `components/` — 24 个组件族
- `utils/util.js` — 公共工具函数

**服务端核心路径：**
- `Controllers/` — 39 个 Controller（Order、Rent、SkiPass、Member 等）
- `Models/` — 206+ 数据模型
- `Data/ApplicationDBContext.cs` — EF DbContext
- `Util.cs` — 全局工具方法
- `wwwroot/` — 静态管理后台页面

**API 路由规则：**
- 新接口：`/api/[controller]/[action]`
- 旧接口：`/core/[controller]/[action]`

---

## 代码约定
- 服务端直接在 Controller 中写业务逻辑（无 Repository 层）
- EF 查询使用 `.AsNoTracking()` + `.AsSplitQuery()`
- 全部异步（`async Task` / `await`）
- 客户端使用 `getApp().globalData` 管理全局状态
- 支付相关：微信支付 + 支付宝双通道

---

## 当前迭代：租赁现场开单流程重构

**目标：** 将旧版 `pages/admin/recept/` 重构为新版 `pages/admin/reception/`，采用 Alpine Operational Minimalist 设计规范。

**页面模版：** `pages/template/stitch/_1` ～ `_5`（设计稿原型，不直接使用）

**实现页面：** `pages/admin/reception/`

### 五步开单流程

| 步骤 | 功能 | 模版 | 实现文件 | 状态 |
|------|------|------|----------|------|
| 第一步 | 录入订单标识（姓名/手机号，非必填） | `_1/` | `recept_entry` | ✅ 完成 |
| 第二步 | 租赁开单 — 购物车/添加入口 | `_2/` | `recept_new` + `rent_recept_form` | ✅ 完成 |
| 第三步 | 选择套餐（分类筛选 + 多选 + 数量步进） | `_3/` | `recept_package` | ✅ 完成 |
| 第四步 | 已选装备 — 套餐/单品详情录入 + 租赁形式 | `_4/` | 内嵌于 `rent_recept_form`（卡片展开） | 🚧 进行中 |
| 第五步 | 支付结算 — 生成二维码 + 顾客扫码 + 会员匹配 | `_5/` | `pages/payment/settle/` + `components/{order-summary-card,order-payment}` | 🚧 进行中（mvp 完成） |

### 旧版参考（`pages/admin/recept/`）

| 旧文件 | 对应新文件（`pages/admin/reception/`） | 说明 |
|--------|----------------------------------------|------|
| `recept_entry` | `recept_entry` ✅ | 订单标识录入 |
| `recept_new` | `recept_new` 🚧 | 业务开单共享页 |
| `recept_auth_list` | — ⏳ | 身份验证列表 |
| `recept_member_info` | — ⏳ | 会员信息页 |
| `recept_list` | — ⏳ | 接待列表 |
| `rent_recepting_list` | — ⏳ | 租赁中列表 |

### 设计规范
- 主题：Alpine Operational Minimalist（`pages/template/stitch/alpine_operational_minimalist/DESIGN.md`）
- 主色：`#006495`（天蓝）/ 背景：`#f8f9ff`
- 圆角：8px / 间距基准：8px
- 字体：Lexend

### 显示规则

- **套餐内装备卡片（rentItem）折叠态标题**：必须显示该槽位所属品类的名称；如允许多品类（`canChooseCategory`），把所有可选品类名用 `/` 拼接（如 `双板/单板`）。**禁止**回落为 `待录入`。
- 持久化：品类名拼接结果写入 `class_name`（后端 `RentItem` 持久化字段），扛得住 `Rent/SaveRentRecept` 往返；`categoryName` / `chooseCategories` 是前端临时字段，后端不回传。
- **录入状态 chip（卡片右上角）**：基于 `evalEntry(item)` 派生 `_entered + _statusLabel`。
  - `noNeed=true` → chip 不显示
  - 完整 → 浅绿底 + 深绿字 + `已录入`（`chip-success`，`#dcfce7` / `#15803d`）
  - 缺项 → 浅红底 + 深红字 + 缺项文案（`chip-pending`，`#fee2e2` / `#b91c1c`），多项缺失只显示第一项
  - 文案：`编码未填` / `名称未填` / `模式未选`
- **「无编码」/「不需要」联动 disabled**（独立 boolean，无互斥）：
  - 名称 input disabled = `noNeed || !noCode`
  - 编码 input + 扫码按钮 disabled = `noNeed || noCode`
  - 备注 + 租赁模式按钮 disabled = `noNeed`
  - `noNeed=true` 时整张卡片底色变灰（`item-card--disabled`）；「不需要」按钮选中态用红色（`code-flag-btn--warn`）
  - 切换「无编码」/「不需要」时清空被禁用一侧的 `code`/`name`（`memo` 保留）
- **套餐内装备模式不一致**：套餐模式按钮组右侧显示橙色 ⚠ icon（`warning-o` `#d97706`），点击 toast「套餐内装备模式不一致」。`_modeMixed` 由 `_refreshRentals` 派生（非 noNeed items 的 `pick_type` 去重 size > 1）。
- **Rental（套餐）级录入完整性 chip**：基于 `evalRental(rental)` 派生 `_rentalEntered + _rentalStatusLabel`。
  - 优先级：`模式未选` → `起租时间未填` → `N 件未录入`（noNeed 不计） → `已录入`
  - **折叠态不显示** chip（避免抢标题空间），不完整时套餐名变红 `var(--error)` 起警示作用
  - **展开态显示** chip：完整 `chip-success`「已录入」/ 缺项 `chip-pending`「N 件未录入」等
  - 实施：`_updateRentalChip(ridx)` 在 6 个 mutator 末尾就地更新（与 rentItem chip 同步刷新模式一致），并触发 `_refreshSummary()` 让结算按钮 disable 状态即时反映
- **Rental 折叠/展开标题区**：两套结构 `wx:if="{{!_expanded}}"` / `wx:else class="pkg-row--expanded"`
  - 折叠态：单行（套餐名 + 套装/单品 chip + 押金/租金 + 箭头）
  - 展开态：第一行套餐名独占（`pkg-title-row`，超过 `RENTAL_TITLE_THRESHOLD = 18` 视觉宽度时跑马灯，与 `item-title-marquee` 同款 11s 周期）；第二行 chips + 押金/租金 + 箭头
- **结算按钮 disable**：`summary.canCheckout = displayRentals.length > 0 && every(r => r._rentalEntered)`，任一 rental 不完整即灰掉。`summary.count` 显示总件数（蓝色圆角徽章）。
- **起租日期**：使用 `van-calendar`（不用原生 `<picker mode="date">`）。组件根尾部单实例 modal；点日期文字 → 开 modal；「今」/「明」单字 pill 直接落点不开 modal。`_dateIsToday` / `_dateIsTomorrow` 派生 → 对应 pill 蓝底白字高亮（`date-quick-btn--active`）。`formatDate(d)` 用本地时区 `YYYY-MM-DD`，避开 `toISOString` 的 UTC 偏差。
- **起租日期/时间持久化**：唯一真理之源是 `start_date`（snake_case，ISO datetime `YYYY-MM-DDTHH:mm:00`），与后端 `Rental.start_date` (DateTime?) 字段对齐。前端 camelCase 字段 `startDate` / `startTime` **后端模型上不存在**，会被 `Rent/SaveRentRecept` 反序列化时 `System.Text.Json` 静默丢弃 → round-trip 后变 null。
  - 读：`splitISODateTime(r.start_date)` 切日期/时间；camelCase 字段做 fallback 兼容老数据
  - 写：`combineDateTime(date, time)` 合并；`_setPkgDate` / `onPkgTimeChange` / `onPkgModeTap` 都改写 `start_date` 单一字段
  - 通用警示：**前端发后端的字段必须用 snake_case** 与模型对齐；写新字段前先核对后端模型，不要假设 camelCase 通用
- **租赁模式联动起租日期/时间**：选模式时同步覆盖（每次切换都覆盖，即使用户已手改）
  - `立即租赁` / `先租后取` → 今天 + 当前时分（`HH:mm`）
  - `延时租赁` → 明天 + `00:00`
  - 实施：`dateTimeForMode(mode)` helper；`onPkgModeTap` 一次性 setData 写入 `start_date` + `_startDate` + `_startTime` + `_dateIsToday` + `_dateIsTomorrow`
  - 创建 rental 时（`recept_package.js onConfirm`）：万龙系 `pick_type=立即租赁` + startTime=当前时分；非万龙系 `pick_type=null` + startTime 仍设当前时分（不依赖 pick_type）
- **`atOnce` 字段必须为 boolean**：后端 `RentItem.atOnce` / `Rental.atOnce` 都是 `public bool`（非可空）。前端**不能发送 `null`**，否则 `Rent/SaveRentRecept` 反序列化失败 → `One or more validation errors occurred` 400。统一写 boolean 表达式，例：`atOnce: defaultPickType === '立即租赁'`（万龙→true / 非万龙→false）或 `atOnce: mode !== 'delay'`。
- **rentItem 装备编码录入**：使用搜索 modal（`components/reception/search_product_fuzzy/`）
  - 触发：点装备编码区域（`<view bindtap="onItemCodeTap">`，input 不再可手动键入，仅显示已录入 code 或 placeholder）。`noNeed` / `noCode` 时短路返回不开 modal
  - API：`Rent/GetRentProductFuzzy?key=xxx&categoryId=xxx`（包装在 `data.searchBarCodeFuzzyPromise(key, categoryId)`）；按 barcode/name 模糊匹配，`categoryId` 限定品类树（含子品类），不传则全库搜
  - categoryId 传值：`item.category_id || (item.category && item.category.id)`；多品类槽位（`canChooseCategory: true`）`category_id` 默认 `chooseCategories[0].id`，所以默认搜第一品类
  - 回填字段映射：`product.barcode → code` / `product.id → rent_product_id` / `product.category_id → category_id` / `product.category.name → class_name` / `product.name → name`；清 `memo`，刷新 `_entered` + `_statusLabel` + `_updateRentalChip` + `_emitSync`
  - 重复编码校验：购物车内除自己外不允许相同 `code`（`noNeed` / `noCode` 不参与），违反 toast「编码已被占用」拦截不写入
  - 扫码（`onItemScan`）仍然可用，独立于搜索 modal
- **主项 rentItem 必须选分类**：`evalEntry` 把 `!is_associate && !category_id` 视为最高优先级缺项，chip 显示 `分类未选`（先于 `名称未填` / `编码未填`）。`_refreshRentals` 派生：`needsCategory = !is_associate && !category_id && !noNeed` 时，标题派生为 `待选分类`，且 `expandedItem[ikey] === undefined` 默认 `true`（首次添加自动展开让用户立刻看到分类入口）。
- **附件项录入校验改为标准**：原 `evalEntry` 对 `is_associate=true` 的豁免分支已删除。附件项（如双板带的雪杖）现在与主项一套校验：`noCode=true` 默认 → 必须录名称；缺则 chip 显示 `名称未填`，rental 级派生 `N 件未录入`，结算按钮 disable。后端 `BuildAssociates` 默认 `noCode=true, atOnce=true, is_associate=true`，前端创建附件项时与之对齐。
- **「无码物品」入口流程**：点底部「无码物品」→ `recept_new._addBlankRental` 创建一个 `category_id=null` 的 rental + 一个主项 rentItem（`is_associate=false, noCode=true, category_id=null, name=null, code=null`）→ 卡片默认展开 → 用户点卡片中「分类」行打开 `van-tree-select` modal → 选定后 `_applyCategoryChange` 拉 `getRentCategoryPromise(catId)` + `getRentPriceListPromise` → 更新主项字段 + 删旧附件 + 按 `associateCategories` 重建附件 + 同步 `rental.category_id/name/guaranty/priceList` + `util.createRentalDetail` 重算 `pricePresets` → emit `syncRent`（needUpdate=true）父页保存。**反复切换主项分类**：每次切换都重建附件，从有附属分类切到无附属分类时附件项自动消失。
- **分类 modal 设计**：`van-popup position=bottom round` + `van-tree-select` + 取消/确认按钮。分类树懒加载：`_ensureCategoryTreeLoaded` 拉顶级（`getTopCategoriesPromise`），`_loadCategorySub(idx)` 按需拉子分类（`getSubCategoriesPromise`）。`_categoryChildMap`（按 sub id → 完整分类对象）只缓存在 component data，不持久化，重新进页面会重拉。
- **押金/租金编辑改为 modal 二次确认**：rental 详情卡里的押金、租金/日 不再是 input + blur，改为 `<view bindtap>` → `wx.showModal({editable:true})` 输入 → 第二个 `wx.showModal` 二次确认 → 调用 `_applyPkgDeposit / _applyPkgRate` 写入。**关键坑**：服务端 `Rent/SaveRentRecept` 往返**不保留** `realGuaranty`，`_refreshRentals` 用 `realGuaranty ?? guaranty` 取值，所以押金应用时必须同时更新 `guaranty=v` + `guaranty_discount=0`，否则 sync 回来后 UI 被刷回旧值。租金存在 `pricePresets[0].price` 里服务端原样返回，无此问题。
- **押金一律显示净额**：购物车栏、详情卡 row meta、kv-cell 三处押金统一显示 `realGuaranty − guaranty_discount`（2 位四舍五入，避开 `300 - 299.95 = 0.04999...`）。`_refreshRentals` 派生 `_depositLabel = netDeposit`、`_depositInput = String(netDeposit)`；`_refreshSummary` 求和后 `deposit` / `reduce` 各再 round 一次避免累计误差。减免量单独标「已减免 -¥xxx」（不是「减免」，强调"已生效"）。`_applyPkgDeposit` 是 modal 编辑入口，把用户输入直接作为新的目录押金 + 清零 `guaranty_discount`；外部减免（会员/券）需走各自路径写 `guaranty_discount`。
- **新页面不再引入 fui-* 组件**：项目计划逐步弃用 FirstUI（`fui-row` / `fui-col` / `fui-section` / `fui-button` 等）。新建或重做页面优先用纯 `view` + 自定义 wxss class（卡片 + flex 行 + 竖条标题模式，参考 `pages/order/payment_entry`）。vant-weapp（`van-button` / `van-popup` / `van-tree-select` / `van-calendar` 等）项目仍在用，可继续引入；旧页面里的 fui 不强制立刻拆除，但维护时遇到就尽量替换为纯 CSS 等效形态。
- **顾客扫码支付落地页（`pages/order/payment_entry`）布局规范**：
  - 整页背景 `#F8F8F8`，4 段卡片结构：A 订单信息 / B 业务明细（按 `order.type` 区分） / C 金额 / D 支付按钮。卡片白底 + `12rpx` 圆角 + `24rpx` 内边距，无阴影。
  - 分组标题：左侧 `6rpx` 蓝色（`#2EA6D0`）竖条 + `30rpx` 半粗体（`.section-title::before` 伪元素）。
  - 强调色：主色 `#2EA6D0`（按钮、竖条）/ 警示红 `#E64340`（需要支付金额、支付成功提示）。
  - B 段业务明细仅对识别的 `order.type` 渲染；未识别 type（餐饮 / 零售 / 押金等本期未做）走最小版（A + C + D 三段不报错）。
  - 租赁类型下：Rental 主行 = 商品名 + `N 件▾` + 押金/日租金一行（`.fee-row` + `.fee-group`，押金/日租金各 `300rpx` 列宽，按 5 位数字 `¥99999.00` 预算），点击 toggle 展开/收起。明细只列 **编码 / 名称 / 品类** 三字段，**不显示**取/还时间和状态。
  - 折叠用手写 `wx:if` + `bindtap` 切换 `rental.expanded`，不引入 `van-collapse` 等组件以保持轻量。

## 通用结算页设计约定

- **结算页是通用页面，非租赁专用**：路径 `/pages/payment/settle/index?orderId=...`（在 subpackage `pages/payment` 下，app.json 写 `"settle/index"`）。任何业务下单完成后用同一行 `wx.navigateTo` 跳入即可。两个核心组件都只吃 `orderId` prop：
  - `components/order-summary-card/` — 可折叠订单卡，调 `getOrderByStaffPromise` 拉单，展示 rentals.name；缺失时用 `getPackagePromise` / `getRentCategoryPromise` 补全
  - `components/order-payment/` — 微信/支付宝/其他三选一。微信走 `Order/GetWepayPayment/{id}` + `MediaHelper/GetQRCode` + WebSocket 监听 `paymentpaid`；**支付宝当前 mock 成微信二维码**（标了 `// TODO: 切换到支付宝小程序后替换`）；其他方式弹红色「确认收款」按钮 → `wx.showModal` 二次确认 → `Order/EffectUnpaidOrder?payMethod=...&payLater=false`。支付完成统一 `triggerEvent('paid', {orderId, payMethod, order})`，父页面后续处理待定
- **页面 UI 约束**：用 `@import "/pages/template/stitch/tokens.wxss"`；**不要再画自定义 topbar**（小程序默认导航栏已有，画两个会重复）；`util.showAmount` 返回值已带 `¥` 前缀，拼接时勿再加；底部需挂 `<reception-tabbar active="open"/>` 否则 tab 栏消失
- **订单号显示**：订单卡片副标题用 `#{{order.code || order.id}}`。`order.code` 由服务端 `OrderController.GenerateOrderCode` 生成：`{shopCode}_{bizCode}_{yyMMdd}_{序号5位}`（如 `WL_ZL_260511_00001`，租赁 bizCode=ZL，序号 = 同前缀订单数+1），仅在 `valid=1` 时生成（`UpdateOrder` 自动触发或 `PlaceRentOrder` 显式调用）。未 placed 订单回退到内部 id 兼容历史数据
- **结算闭环约定**：业务页面的 `onCheckout` 必须串成 `await saveRentReceptOrder() → Order/PlaceRentOrder/{id} → setData({ order: rentOrder }) → wx.navigateTo settle`。先 await 落盘是为了规避用户改完字段立即点结算时、syncRent 触发的保存还在飞行的竞态。`saveRentReceptOrder` 返回 Promise（成功 resolve(submitted)、失败 reject）；fire-and-forget 调用点（`onSyncRent` / `_appendRentals`）必须补 `Promise.resolve(this.saveRentReceptOrder()).catch(() => {})` 吞 rejection

---

## 当前状态（截至 2026-05-17）

**已可走通**：录入订单 → 选店 → 进入租赁开单 → 添加套餐（按品类筛选 + 万龙系店铺默认「立即租赁」+ 雪服/护具等非编码品类默认勾选「无编码」+ 创建时 startTime 默认当前时分）→ 购物车展示（rental 折叠态紧凑单行；展开态两层标题 + 跑马灯；rental 级 + rentItem 级双层完整性 chip；不完整时套餐名变红）→ 卡片展开编辑详情（套餐备注 + 起租日期 van-calendar 弹窗 + 今/明高亮快捷按钮 + 起租时间 picker；选租赁模式自动联动起租日期/时间：立即/先租后取=今天+当前时分、延时=明天+00:00；无编码/不需要 disabled 联动 + 不需要时整卡灰显）→ 装备编码录入（点编码区开搜索 modal，按品类模糊搜索租赁物，单选确认后回填 code/name/category_id/rent_product_id/class_name + 重复编码校验；扫码仍然可用）→ 押金/租金点击 tap 弹 `wx.showModal` 二次确认编辑（押金净额显示 = `realGuaranty − guaranty_discount`，下方购物车栏「押金 ¥净额 已减免 -¥xxx」）→ 套餐选模式时未自选 item 跟随 + 内部模式不一致显示 ⚠ → 左划删除 → 底部 4 个快捷入口横向紧凑按钮 + 单行结算条（件数徽章 + 押金 + 已减免 + 租金 + 去结算按钮，全部 rental 完整才允许点击）→ 点「去结算」先 await `saveRentReceptOrder` 落盘最新编辑、再调 `Order/PlaceRentOrder/{id}` 让服务端 `GenerateOrderCode` 生成 `WL_ZL_yyMMdd_xxxxx` 正式订单号 + `valid=1` + 写 Guaranty，返回的 order 回填 `this.data.order` → 跳 `/pages/payment/settle/index?orderId=...` → 结算页订单卡显示 `order.code || order.id` + 三选一支付方式（微信扫码 / 支付宝 mock / 其他确认收款）→ **顾客扫支付二维码进入 `pages/order/payment_entry`：轻量化纯 CSS 卡片版（订单信息 / 租赁内容折叠 / 金额 / 微信支付按钮），租赁明细只列 编码/名称/品类，押金 + 日租金同行各 300rpx 列宽** → 小程序客户端所有 `wx.request` 的 `POST` 请求在全局请求层统一对 payload 内 URL 编码中文执行 `urldecode`（含嵌套对象/数组）。每次结构变更/字段失焦自动 `Rent/SaveRentRecept` 同步后端，起租日期/时间通过 `start_date` (ISO datetime) 真持久化。→ **顾客扫码 payment_entry 落地后增加支付前身份验证**：onShow 调 `PaymentIdentity/CheckPayerIdentity` 拉 5 状态 → 未绑手机号弹一键授权 / 订单已匹配别人弹「正常支付（订单转归我）」「替人代付（订单仍归原会员）」二选一 modal / 订单未匹配会员则确认「订单将归我」→ `ConfirmPayIdentity` 立即落库 `Order.member_id` / `OrderPayment.member_id` / `is_proxy_pay` / `wechat_unverified`（支付宝支付一律置 `wechat_unverified=true`）→ status 转 `direct` 后才显示原微信支付按钮。**支付宝手机号解密目前是 stub**（待支付宝小程序对接）。

**关键文件**
- 页面：`pages/admin/reception/recept_entry`、`recept_new`、`recept_package`、`pages/order/payment_entry`（顾客扫码支付落地页）
- 组件：`components/reception/rent_recept_form`（购物车 + 详情卡片 + 日历 modal + 编码搜索 modal）、`components/reception/search_product_fuzzy`（编码搜索弹窗，可复用）、`components/order-summary-card` + `components/order-payment`（结算页订单卡 + 二维码组件）
- 数据接口（已对接）：`Order/GetShops`、`Rent/GetRentPackageList`、`Rent/GetRentPackage/{id}`、`Rent/GetRentPriceList`、`Rent/SaveRentRecept`、`Order/GetShopByName`、`Rent/GetRentProductFuzzy`、`Rent/GetTopRentCategories`、`Rent/GetSubRentCategories/{id}`、`Rent/GetRentCategory/{id}`、`Order/GetOrderFromPaymentByCustomer/{paymentId}`、`Order/WechatPayByOrderPayment/{paymentId}`、`PaymentIdentity/CheckPayerIdentity`、`PaymentIdentity/ConfirmPayIdentity`
- 支付身份验证后端：`Controllers/Order/PaymentIdentityController.cs`（5 状态决策树 + submit_phone / choose / confirm_direct 三 action），模型 `Models/Order/Order.cs` (+`wechat_unverified`) / `Models/Order/OrderPayment.cs` (+`is_proxy_pay`) / `Models/Member/MemberSocialAccount.cs` (+`TYPE_WECHAT_MINI_OPENID` 等 4 个 type 常量)
- 支付身份验证小程序：`components/pay-identity-confirm/`（4 文件，渲染 phone_required/direct_to_scanner/choose_identity/error 四态卡片）、`utils/data.js` 新增 `checkPayerIdentityPromise` + `confirmPayIdentityPromise`、`pages/order/payment_entry.{js,wxml,json}` 接入 identity 状态机

**下一步要做的**
- ✅ 第五步：支付结算页 mvp 完成（settle/index + order-summary-card + order-payment，微信支付走通、支付宝 mock、其他方式确认收款）
- ✅ 顾客扫码支付落地页（`pages/order/payment_entry`）轻量化重做 + 租赁订单友好展示
- payment_entry 其它订单类型友好展示（餐饮 / 零售 / 押金等当前走最小版，留待后续按业务需要扩展）
- 第五步剩余：支付宝小程序对接（替换当前 mock）、支付完成后父页面 `onPaid` 处理（跳转 `rent_details` 或工作台）
- 第二步剩余：扫描条码（`Rent/QueryByBarcode`）入口（目前仅 toast 占位）
- 第二步：去结算按钮入口（已在 `onCheckout` 接通 `Order/PlaceRentOrder` + navigateTo settle）
- 养护 / 零售 业务的接待表单组件（目前仅租赁完成）
- 旧版页面迁移：`recept_auth_list`、`recept_member_info`、`recept_list`、`rent_recepting_list`
- ✅ 支付前身份验证 A+B 切片完成：后端模型 / DB / `PaymentIdentityController` + 小程序 `pay-identity-confirm` 组件 + payment_entry 接入；swagger 烟测只读路径通过。**待真机端到端测试**：用其它账号扫顾客二维码触发 `choose_identity` 选「正常支付/替人代付」走完支付闭环 + DB 校验 `Order.member_id` / `OrderPayment.is_proxy_pay` / `wechat_unverified` 是否按预期写入
- 支付宝真实手机号解密（接 `alipay.system.oauth.token` + `alipay.user.info.share`），当前是 stub（传 `phoneMock` 字段走通）
- 决策时机改为"支付完成后"语义：把 `ConfirmPayIdentity` 的 Order.member_id 写入挪到 `OrderPaymentController` 的 wechat/alipay notify 回调里，本期是"立即生效"简化版
- 未使用 fui-* 组件清理（本次删了 6 个：`fui-badge / fui-tabs / fui-toast / fui-top-popup / fui-utils / fui-wing-blank`，剩 17 个继续逐步弃用）
- 页面可达性 review：`snowmeet_ai_doc/unreachable_pages.md` 列出 75 个从 index/mine BFS 不可达的页面（含 62 个完全孤立），需人工逐项区分 QR 扫码入口 vs 死代码后清理

**已知遗留**
- **macOS 上 pyodbc + msodbcsql18**：unixODBC 默认查 `/etc/odbcinst.ini` 但 brew 装的 msodbcsql18 注册在 `/opt/homebrew/etc/odbcinst.ini`。所有 pyodbc 脚本启动前要 `export ODBCSYSINI=/opt/homebrew/etc`（写到 shell rc 或脚本 wrapper 都行）。已在 `snowmeet_ai_doc/skills/export_rent_order/SKILL.md` 文档化
- **本机(Intel Mac) ODBC 配置异于上条**：上条 `/opt/homebrew/etc` + Driver 18 是给 Apple Silicon 同步机的；Intel Mac（brew 在 `/usr/local`）需 `export ODBCSYSINI=/usr/local/Cellar/unixodbc/2.3.4/etc` + 用 `--conn` 覆盖成 `DRIVER={ODBC Driver 13 for SQL Server}`（脚本 DEFAULT_CONN 写死 Driver 18，本机只装了 13）
- **数据库里 rental_detail.charge_type 只有'租金'、'超时费'、'赔偿金'三种值**：用户口语的'损坏赔偿'实际是'赔偿金'。新查询写 `IN ('赔偿金','损坏赔偿')` 兼容
- **discount 归属计算用"detail 级 + 非 detail rental 级"严格归一**：详见 `snowmeet_ai_doc/skills/export_rent_order/SKILL.md` 减免金额定义。直接 `order_id` 匹配会让多 rental 订单的全单 discount 在每条 rental 上重复计入
- **租赁数据导出脚本现状**：`snowmeet_ai_doc/export_wanlong_rent_orders.py` 是旧的万龙单店脚本（保留作历史）；`snowmeet_ai_doc/skills/export_rent_order/export_rent_orders.py` 是通用版本（任何店铺）。维护时改通用版，旧脚本不再演进
- `needIntercom`（雪板类租赁默认加对讲机）相关逻辑已注释，未来需要时可恢复
- `recept_new.onMemberDetail` 仍跳转旧版 `pages/admin/recept/recept_member_info`，待新版会员详情页完成后切换
- `_modeFromPkg` 是组件内部临时字段（`_` 前缀，由 `stripUI` 过滤），不持久化；页面重载后所有 item 视为"已自选模式"，不会再被套餐传导覆盖（保守、符合预期）
- 跑马灯阈值常量：rentItem `TITLE_MARQUEE_THRESHOLD = 11`、rental `RENTAL_TITLE_THRESHOLD = 18`（按视觉宽度估算，汉字 1.0 / 半角 0.5），标题踩边时可能误判滚动/不滚动，调阈值即可
- rental 备注字段名 `memo`（与 rentItem 一致），后端 `Rental` 模型如未支持会被 `Rent/SaveRentRecept` 静默丢弃；如发现 reload 后丢失，需要核对后端字段名
- van-calendar 范围 `min-date = 今天`、`max-date = 一年后`（不允许选过去日期，如需后台补单可放宽）
- 装备编码 input 改成 view + bindtap 后**用户无法手动键入编码**，只能通过搜索 modal 或扫码录入；与旧版语义一致，但若有客户特殊编码不在数据库里则无法处理（极端场景）
- 多品类槽位（`canChooseCategory: true`）搜索 modal 限定 `chooseCategories[0]`（第一品类）；如需搜其他品类需手动改 `item.category_id` 或后续做品类切换 UI
- 全局中文 `urldecode` 目前仅拦截 `wx.request` 的 `POST` 且仅处理 `data`；`GET` query 参数和非 `wx.request` 通道（如 `wx.uploadFile`）不在本次覆盖范围
- 分类树 `categoryItems / _categoryChildMap` 不持久化，重新进入 `recept_new` 时第一次点开分类 modal 会重新拉取顶级 + 子分类（懒加载）。如频繁打开影响体验，可改成 page 级缓存或 globalData
- 主项分类切换会触发 `Rent/SaveRentRecept`（通过 `triggerEvent('syncRent', { needUpdate: true })`），保存返回的 rental 经 properties observer 回流刷新。如果后端返回的 priceList 不含我们刚拉的内容会被覆盖（目前未发现问题）
- 结算页支付宝当前为微信二维码 mock，扫码会按微信支付完成（已标 TODO，等支付宝小程序方案落地）
- 结算页 `onPaid` 仅 `console.log`，未做跳转/刷新；父页面后续处理待定
- 支付组件 WebSocket 仅在选中微信/支付宝并生成二维码后开启；切换支付方式时关闭旧 socket 再开新的，若用户在 prepay 调用中途切换会有短暂残留请求（无功能影响）
- `pages/order/payment_entry` 目前仅对 `order.type=='租赁'` 做友好明细展示（编码/名称/品类 + 押金/日租金）；餐饮/零售/押金等其它类型走"最小版"（订单信息 + 金额 + 按钮），后续按业务需要扩展
- **`Member.wechatMiniOpenId` 是后端计算属性**（getter 遍历 `memberSocialAccounts` 找 type=`wechat_mini_openid`），需要序列化时 MSA 集合被一并带回。顾客扫码 payment_entry 这种深链场景下 `app.globalData.member` 可能不齐全，导致前端取该字段为空。新接口（如 `PaymentIdentity/CheckPayerIdentity`）若需要扫码方 openid 都得做 sessionKey → `mini_session.member_id` 反查兜底
- **`PaymentIdentityController` 用"立即生效"语义**：用户在 payment_entry 选完归我/代付后即写 `Order.member_id`，不等支付实际完成。如业务实测发现"用户中途放弃"导致归属错乱，需挪到 wepay/alipay notify 回调；本期为简化采用"用户确认即落地"
- **支付宝 submit_phone stub**：`PaymentIdentityController._submitPhone` 当 `payerType=alipay` 时若传 `phoneMock` 字段直接用，否则返 `alipay_phone_pending`。真支付宝解密待支付宝小程序对接（`alipay.system.oauth.token` + `alipay.user.info.share`）
- **`components/firstui/` 17 个组件仍在用**：含 `fui-config`（喂 `wx.$fui` 给 fui-button/icon/section/list-cell/white-space）+ `fui-css`（`app.wxss` 全局 `@import`）+ 其它 15 个有 wxml 引用。本次删的 6 个 (`fui-badge / fui-tabs / fui-toast / fui-top-popup / fui-utils / fui-wing-blank`) 是 0 引用残留
- **页面可达性报告**：`snowmeet_ai_doc/unreachable_pages.md` — 117 个 page 中 62 个全项目零引用，但部分是 QR 扫码外部入口（如 `pages/order/payment_entry` 是顾客扫码落地页，必须留），删之前要逐项区分
- payment_entry 折叠交互手写 `wx:if`，未引入 `van-collapse` 等组件以保持轻量；一个 Rental 内 rentItem 数量上限按 ~10 件设计
- payment_entry 押金/日租金列宽固定 `300rpx`（5 位数字预算 `¥99999.00`），超出会被挤压；如业务出现万元以上押金需要回来调
- payment_entry `pay()` 内成功回调里第二次拉单时把 `payment.id` 当成 paymentId 传，但拉回来的对象是新的 order（含 nonce 等微信字段已是 undefined），这一段是历史代码，本轮 UI 改造未触碰，留待后续清理
- **数据库 schema 新旧并存**：旧 schema `order_online` / `rent_list` / `rent_list_detail` 在 2025-10-15 后已无数据；新 schema 在用 `[Table("order")]` (Order.cs) / `rental` / `rental_detail` / `rent_item`。所有新查询和报表都走新 schema。本地 SnowmeetApi 当前 master 没有 `Order.cs`，开发需先 `git checkout ai`
- **生产数据库**：`100.28.143.19:1433` SQL Server 2022 CU21，库 `snowmeet_new`；连接字符串保存在仓库外 `config.sqlServer` 文件（gitignore），不在 appsettings.json
- **退款判定标准**：`payment_refund.state=1 OR refund_id 非空非空字符串`，与 `Models/Rent/RentOrder.cs:519` 旧逻辑一致；仅 `state=1` 会漏掉绝大多数已发起但未回调的退款（万龙时段实测漏 538 万）
- **wepay_key 关联**：`order_payment.mch_id` 实际存的是 `wepay_key.id`（如 5/10/12），真实微信商户号在 `wepay_key.mch_id`（如 1604236346 万龙租赁主力账户）。统计需 JOIN
- **rental_detail.charge_type 三种值**：`租金` / `超时费` / `赔偿金`（中文，注意"赔偿金"非"赔偿"）。按 rental 分组求和
- **未结算订单虚账**：`rental.settled=0` 的 rental 会持续按天累积 `rental_detail` 应收记录（如雪季初一直没关单的，累积到 189 天 ¥9 万）。做收入报表必须过滤已结算/已关闭，否则虚增
- **`api/Rent/GetConfirmedRentOrder` (RentController.cs:5544) 的"确认订单"5 条规则**：paidAmount > 0 AND closed=1 AND close_date != null AND !hide AND 不含非微信非支付宝支付（现金/储值/转账等会被排除）；做对账报表时这是参考过滤口径
- **`punch_card` / `punch_card_used` 表存在但 EF 未接**：DB 有 `punch_card`(36 行, 字段 id/biz_type/card_name/member_id/mi7_code/total/punches) + `punch_card_used`(**0 行**, 字段 id/card_id/order_id/biz_type/biz_id/payment_id/punch_count/valid)。`SnowmeetApi/Models/` 下**无** `PunchCard` / `PunchCardUsed` 模型（grep 0 命中）。当前业务核销「次卡支付」仍走 `order_online.pay_memo='次卡支付'`（6 单）/ `[order].pay_option='次卡支付'` 字符串标记的老路径，新结构化的 punch_card_used 明细表尚无写入代码
- **同步以 skill 步骤为准，别依赖本机 hook**：start-work 已把 `git -C snowmeet_ai_doc pull --ff-only` 内置为 `SKILL.md` 第 1 步（入库、跨机生效）。`.claude/settings.local.json` 的 PreToolUse(pull) / Stop(push) hook 是 gitignored / 机器本地；本会话实测 PreToolUse **未触发**（疑非标准 `if` 键），仅作冗余。Stop hook 已收紧为仅 `git add -- sessions CLAUDE.md`（不再 `git add .` 吞 WIP），非归档改动不会被自动 push，留待手动

---

## 开发日志

### 2026-05-01
- ✅ 第一步 `recept_entry`（录入订单标识信息）
  - 支持国际手机号校验（E.164 + 本地格式）
  - 依赖 `components/shop_selector`（内部调用 `Order/GetShops`）
- 🚧 开始第二步 `recept_new`（业务开单共享页）
  - 顶层页持有客户信息 + 订单数据；通过 `rent-recept-form` 子组件事件回传

### 2026-05-02
- ✅ 第二步"添加套餐"功能
  - 新增 `pages/admin/reception/recept_package` 页（4 文件）
  - 接入真实套餐：`Rent/GetRentPackageList?shop=` → 列表，`Rent/GetRentPackage/{id}` → 完整套餐，`Rent/GetRentPriceList` → 价格
  - 支持多套餐 × 多份步进选择，确认后通过 eventChannel 回传给 `recept_new`
  - `recept_new.onAddAction(package)` 跳转逻辑 + `_appendRentals` 追加并保存
- ✅ 套餐选择页按 `package_type` 分类筛选
  - 服务端：`RentPackage` 模型加 `package_type` 字段映射（无 `[NotMapped]`，EF 自动映射数据库列）
  - 前端：分类 tabs — 全部 / 双板 / 单板 / 雪服 / 护具 / 其他（package_type 为 null）
- ✅ 购物车支持左划删除（`van-swipe-cell` + 二次确认弹窗）
  - `recept_new.saveRentReceptOrder`：已存在订单（id > 0）时，删除最后一项也会同步空购物车到后端
- ✅ 暂停 `needIntercom` 相关逻辑
  - 服务端：`Order.cs` 字段定义注释、`RentController.cs` 中 `AddInterCom` 调用注释
  - 客户端：旧版 `rent_recept.js` 的 `del()`、旧版 `recept_new.js` 的 `rentDataUpdated` 中相关分支注释
- ✅ 租赁卡片折叠/展开（参考模版 `_4`）
  - 收起：单行（套餐名 + 押金 + 租金/日 + 展开箭头）
  - 展开：租赁形式（立即/先租后取/延时）+ 押金/租金输入 + 起租日期/时间 picker + 内层装备清单
  - 内层装备项也可折叠，展开后录入：名称、编码（含扫码）、无编码 / 不需要、备注、租赁模式
  - 状态保持：`expandedPkg/expandedItem` map（按 timeStamp/id 稳定 key），跨后端保存往返保留
  - 字段映射：`pick_type` / `realGuaranty` / `pricePresets[0].price` / `startDate` / `rentItems[].name|code|memo|noCode|noNeed|pick_type`
  - **注意**：组件内部不能用 `data.rentals`（与 `properties.rentals` 同名导致死循环），改用 `displayRentals` 作为渲染数据；`stripUI()` 在 `triggerEvent('syncRent')` 时去掉 `_xxx` 临时字段

### 2026-05-03（上午）
- ✅ 套餐内装备卡片折叠态显示品类名（修复"待录入"占据主标题位）
  - `recept_package.js`：加入购物车时把该槽位所有可选品类名用 `/` 拼接，写入 `class_name`（持久化字段）+ `categoryName`（前端临时字段）
  - `rent_recept_form.js`：标题派生改为 `it.class_name || it.categoryName || (it.category && it.category.name) || it.name || '待录入'`
  - 关键：`class_name` 是 `RentItem` 模型的持久化字段，能扛 `Rent/SaveRentRecept` 往返；`categoryName`/`chooseCategories` 是前端临时字段，后端不回传
  - 显示规则已写入"### 显示规则"小节，禁止再回落"待录入"做主标题
- 📌 教训：微信开发者工具的 JS bundle 缓存会拦截组件改动，仅刷新页面无效。改完 `components/reception/*` 后用户反馈"没生效"时，先 `Tools → Cache → Clear all data` + `Clear file cache` + 工具栏"编译"，再判断是否真的有 bug

### 2026-05-03（下午） — 装备卡片交互完善

主要文件：`components/reception/rent_recept_form/{js,wxml,wxss}` + `pages/admin/reception/recept_package.js`

- ✅ **「无编码」/「不需要」按钮解耦**：原 `_codeFlag` 互斥单选 → 两个独立 boolean (`noCode` / `noNeed`)
  - 「无编码」选中 → 启用名称、禁用编码 + 扫码；切换时清空被禁用一侧
  - 「不需要」选中 → 整张卡片所有输入禁用 + 卡片底色变灰 (`item-card--disabled`) + chip 隐藏；按钮选中态用红色 (`code-flag-btn--warn` / `var(--error)`)
  - `onItemScan` / `onItemModeTap` 顶部加 `if (item.noNeed || ...) return` guard
- ✅ **rentItem 默认 `noCode` 由品类大类推断**
  - `recept_package.js`：常量 `CODE_REQUIRED_PREFIXES = ['01','02','03','04']`（双板 / 单板 / 双板鞋 / 单板鞋）
  - 槽位的所有可选品类的 `cat.code.substr(0, 2)` 都不在白名单 → 默认 `noCode = true`（雪服 / 护具 / 雪杖 / 头盔等）
  - 仅在「添加套餐」新建时生效，后端回传值不被覆盖
- ✅ **录入完整性 chip 重写**
  - 新加 `evalEntry(item)` helper：`noNeed` → 跳过；否则按 `noCode` 校验 `code` 或 `name`，且 `pick_type` 必选
  - 文案：`编码未填` / `名称未填` / `模式未选`；多项缺失只显示第一项；完整 → `已录入`
  - 配色：`chip-success` `#dcfce7` / `#15803d`；`chip-pending` `#fee2e2` / `#b91c1c`
  - 所有 mutator (`onItemFieldBlur` / `onItemCodeFlag` / `onItemModeTap` / `onPkgModeTap` / `onItemScan`) 同步更新 `_entered + _statusLabel`
- ✅ **卡片副标题「名称：xxx」移除**（用户反馈不需要）；卡片左侧袋子图标 `goods-collect-o` 移除
- ✅ **标题跑马灯**：超出容器时 3s 静止 → 8s 滚动 → 循环
  - `visualLen()` 估算字符宽度（汉字 1.0 / 半角 0.5），阈值 `TITLE_MARQUEE_THRESHOLD = 11`
  - CSS keyframes：`0%, 27.27% { translateX(0) }; 100% { translateX(-100%) }`，11s 周期
- ✅ **万龙系店铺默认「立即租赁」**
  - `recept_package.js` 的 `onConfirm()`：`shop` 以 "万龙" 开头 → rental + 每个 rentItem 都默认 `pick_type = '立即租赁'` + `atOnce = true`
- ✅ **套餐租赁模式联动 + 不一致提示**
  - 套餐选模式时，未自选的 item 跟随；已手选的不动（用临时字段 `_modeFromPkg` 标记）
  - `onItemModeTap` 标记 `_modeFromPkg = false`；`onPkgModeTap` 跟随条件 `!it.pick_type || it._modeFromPkg`
  - `_refreshRentals` 派生 `_modeMixed`（非 noNeed items 的 `pick_type` 去重 size > 1）
  - mixed 时套餐模式按钮组右侧显示橙色 ⚠ icon（`warning-o` `#d97706`），点击 toast「套餐内装备模式不一致」

### 2026-05-04（上午） — Rental 级完整性 + 底部交互压缩 + 日历选择

主要文件：`components/reception/rent_recept_form/{js,wxml,wxss,json}`。本次改动通过 plan 文件审批后实施（`/Users/cangjie/.claude/plans/playful-coalescing-quill.md`）。

- ✅ **Rental（套餐）级录入完整性 chip**（复用 rentItem 视觉语言）
  - 新加 `evalRental(rental)` helper，按优先级返回第一个缺项：`模式未选` → `起租时间未填` → `N 件未录入`（noNeed 不计） → `已录入`
  - `_refreshRentals` 派生 `_rentalEntered + _rentalStatusLabel`
  - 新加 `_updateRentalChip(ridx)` 方法在 6 个 mutator 末尾就地更新（`onPkgModeTap` / `onPkgDateTap...` / `onItemFieldBlur` / `onItemCodeFlag` / `onItemModeTap` / `onItemScan`），与 rentItem chip 同步刷新模式一致；末尾再调 `_refreshSummary()` 让结算 disable 状态实时反映
- ✅ **Rental 折叠/展开标题区两层布局**
  - 折叠态：单行（套餐名 + 套装/单品 chip + 押金/租金 + 箭头）。完整性 chip **不显示**，避免抢标题空间；不完整时套餐名 `var(--error)` 红色起警示作用
  - 展开态：第一行套餐名独占 `pkg-title-row`（`_displayMarquee = visualLen > 18` 时跑马灯，与 `item-title-marquee` 同款 keyframes）；第二行 chips + 押金/租金 + 箭头
  - 共用 wxml 结构：`<view wx:if="{{!_expanded}}" class="pkg-row">` / `<view wx:else class="pkg-row pkg-row--expanded">`
- ✅ **不完整时套餐名变红**（折叠/展开两态都适用）
  - `pkg-row-name--pending` / `pkg-title--pending` 都用 `var(--error)`
  - 折叠态：完整 → 纯净标题；不完整 → 红字（替代 chip 起警示）
- ✅ **4 个快捷入口按钮压扁**（添加套餐 / 扫描条码 / 搜索单品 / 无码物品）
  - icon 22px → 16px；`flex-direction: column` → `row` 横向布局；font 10 → 12；padding 8 → 6
  - 高度 ~55px → ~32px
- ✅ **底部结算条压扁 + 件数显示 + canCheckout 控制 disable**
  - 单行布局：`[共 N 件]` 蓝色徽章 + 押金 + 减免 + 租金 + 去结算按钮
  - 押金字号 26 → 16；按钮高度 48 → 36；padding 12 → 8；总高 ~96px → ~52px
  - 加 `summary.count` + `summary.canCheckout = count > 0 && every(r => r._rentalEntered)`
  - `_refreshSummary` 计算；`_updateRentalChip` 末尾触发它
- ✅ **rental 备注字段**（起租日期/时间下方）
  - 全宽 `kv-cell` + `kv-input`，字段名 `memo`（与 rentItem 一致）
  - `onPkgMemoBlur` 失焦持久化；不参与完整性判定
- ✅ **起租日期改用 van-calendar + 今/明 单字快捷按钮**
  - `rent_recept_form.json` 注册 `van-calendar`；组件根尾部加单实例 modal
  - 新加 `formatDate(d)` helper（本地时区 `YYYY-MM-DD`，避开 `toISOString` 的 UTC 偏差）
  - State：`calendarShow` / `calendarRidx` / `calendarDefault` / `calendarMin`(今天) / `calendarMax`(一年后)
  - 新加 `_setPkgDate(ridx, date)` 公共写入路径；移除旧 `onPkgDateChange`（已被取代，保留 `onPkgTimeChange` 走原生 picker）
  - 新加 4 个 handler：`onPkgDateTap`（点日期文字开 modal）/ `onPkgDateQuick`（今/明直接落点不开 modal）/ `onCalendarClose` / `onCalendarConfirm`
  - 「今」/「明」pill 固定 22×18px 方框，`white-space: nowrap` 防日期文字折行，cell 高度由原折行的两行变回单行
- ✅ **今/明 pill 高亮当前日期**
  - `_refreshRentals` 派生 `_dateIsToday` / `_dateIsTomorrow`；`_setPkgDate` 同步更新
  - `.date-quick-btn--active` 蓝底白字（`var(--primary)` / `var(--on-primary)`）；覆盖 `:active` 防按下退色

**plan 文件**：`/Users/cangjie/.claude/plans/playful-coalescing-quill.md`（仅 rental chip 走过 plan 流程，后续几项是用户即时反馈直接修改，未单独立 plan）。

### 2026-05-04（下午） — 起租日期/时间持久化修复 + 编码搜索 modal

主要文件：`components/reception/rent_recept_form/{js,wxml,wxss,json}` + `pages/admin/reception/recept_package.js` + 新建 `components/reception/search_product_fuzzy/{js,wxml,wxss,json}`

#### 一、租赁模式联动起租日期/时间（plan 流程）

- ✅ **选模式自动设日期+时间**（每次切换都覆盖，即使用户已手改）
  - `立即租赁` / `先租后取` → 今天 + 当前时分（`HH:mm`）
  - `延时租赁` → 明天 + `00:00`
  - 新加 `formatTime(d)` + `dateTimeForMode(mode)` helper
  - `onPkgModeTap` 在 setData 里一次性写入所有日期/时间字段（避免多次 setData + 多次 emit）
- ✅ **创建 rental 时初始也按规则设**
  - 万龙系：`pick_type=立即租赁` + startTime=当前时分（与切模式行为对齐）
  - 非万龙系：`pick_type=null` + startTime=当前时分（仅日期/时间初始化，模式仍待选）
  - `recept_package.js` `onConfirm` 加 `startDateTime` 局部变量

**plan 文件**：`/Users/cangjie/.claude/plans/eager-nibbling-volcano.md`

#### 二、修起租日期/时间 round-trip 后丢失的 bug（关键根因）

- 📌 **根因**：后端 `Rental` 模型只有 snake_case 的 `start_date` (DateTime?)，**没有** `startDate` / `startTime` / `start_time`。前端写的 camelCase `startDate`、`startTime` 经 `Rent/SaveRentRecept` 反序列化时 `System.Text.Json` 静默丢弃，回来全是 null，UI 显示「请选择」+「09:00」回退
- ✅ **修复策略**：把日期+时间合并写入 `start_date` (ISO datetime `YYYY-MM-DDTHH:mm:00`) 做唯一真理之源
  - 新加 helper `combineDateTime(date, time)` / `splitISODateTime(sd)`
  - `_refreshRentals` 派生 `_startDate` / `_startTime` 改从 `r.start_date` 切；camelCase 字段做 fallback 兼容老数据
  - `_setPkgDate` 写 `start_date`（合并新日期 + 旧时间，保留时间不丢失）
  - `onPkgTimeChange` 写 `start_date`（合并旧日期 + 新时间）+ 触发 `_updateRentalChip`（之前漏了）
  - `onPkgModeTap` 把 date+time 合并写入 `start_date`，不再写 camelCase
  - `recept_package.js` 创建 rental 时用 `start_date: startDateTime`（ISO datetime）替代 `startDate`+`startTime`

#### 三、修非万龙系加套餐 400 报错

- 📌 **根因**：后端 `RentItem.atOnce` 是 `public bool atOnce = false`（**非可空**）。前端 `recept_package.js` 写的 `atOnce: defaultPickType === '立即租赁' ? true : null`，非万龙系发送 `null`，反序列化为非可空 bool 失败 → `One or more validation errors occurred` 400
- ✅ 改成 `atOnce: defaultPickType === '立即租赁'`（boolean 表达式，万龙→`true` / 非万龙→`false`），与后端默认值对齐
- 注：rent_recept_form.js 的 `onPkgModeTap` / `onItemModeTap` 中用的 `mode !== 'delay'` 本来就是 boolean，不受影响

#### 四、rentItem 装备编码搜索 modal（参考旧版流程）

- ✅ **新建** `components/reception/search_product_fuzzy/`（4 文件）
  - 底部弹起 modal（`van-popup` `position="bottom"` `round`）
  - 结构：标题 + 关闭 X + 当前品类标签 + 输入框 + 查询按钮 + 滚动结果列表（单选）+ 取消/确认
  - Properties：`show` / `categoryId` / `categoryName`；Events：`select`(detail.product) / `close`
  - `observers.show` → `true` 时自动重置 keyword/products/loading 内部状态
  - 复用现有 `data.searchBarCodeFuzzyPromise(key, categoryId)` → `Rent/GetRentProductFuzzy`
- ✅ **`rent_recept_form` 集成**
  - `rent_recept_form.json` 注册 `search-product-fuzzy`
  - 装备编码 input 改成 `<view bindtap="onItemCodeTap">`（同旧版语义；input 不再可手动键入，仅显示已录入 code 或 placeholder「点此搜索或扫码录入」）
  - 组件根尾部加单实例 modal（与 van-calendar 同位置）
  - 加 state：`searchShow` / `searchRidx` / `searchIidx` / `searchCategoryId` / `searchCategoryName`
  - 加 3 个 handler：`onItemCodeTap` / `onProductConfirm`(回填) / `onSearchClose`
  - 加 wxss `.form-input--tap` / `.form-input--disabled` / `.form-input-text`
- ✅ **回填字段映射**（与后端 `RentItem` 模型对齐）
  - `product.barcode` → `item.code`
  - `product.id` → `item.rent_product_id`
  - `product.category_id` → `item.category_id`
  - `product.category.name` → `item.class_name`（持久化字段，rentItem 折叠态标题用）
  - `product.name` → `item.name`
  - 清 `memo`，刷新 `_entered` + `_statusLabel` + `_updateRentalChip` + `_emitSync`
- ✅ **重复编码校验**：购物车内除自己外不允许相同 code（noNeed/noCode 不参与），违反 toast「编码已被占用」拦截
- ✅ **多品类槽位 categoryId**：传 `item.category_id || (item.category && item.category.id)`；多品类槽位（`canChooseCategory: true`）`category_id` 默认为 `chooseCategories[0].id`，所以默认限定第一品类内搜；`null` → 全库搜
- 📌 **wxml 编译坑**：`wx:else` 不能与 `wx:for` 在同一节点（`<block wx:else wx:for>` 报 `wx:if not found`）。修复：拆成 `<block wx:else>` 外层 + 内层 `<view wx:for>`

**plan 文件**：`/Users/cangjie/.claude/plans/eager-nibbling-volcano.md`（仅模式联动日期时间走过 plan，后续几项是用户即时反馈直接修改）。

### 2026-05-05（下午） — 小程序 POST 中文参数全局解码

主要文件：`snowmeet_wechat_mini/app.js`

- ✅ **全局封装 `wx.request` 的 POST 数据预处理**
  - 在 `onLaunch` 初始化阶段注入一次性 patch（`patchWxRequestPostDataDecoder`），避免多次覆盖原生请求函数
  - 仅在 `method === 'POST'` 且存在 `data` 时生效，降低对既有 GET 链路影响
- ✅ **递归处理 payload，支持对象/数组深层字段**
  - 新增 `decodeChineseInPostData(data)`，对数组与对象做深拷贝式递归遍历
  - 字符串字段进入 `tryDecodeChinese(str)`：仅当包含 `%` 且 `decodeURIComponent` 后出现中文字符时才替换
- ✅ **容错与回退策略**
  - 非法编码字符串 decode 失败时保留原值，不阻断请求
  - 增加 `wx.__snowmeetPostDataDecodedPatched` 防重入标记，确保 patch 幂等

### 2026-05-10 — 附件项录入校验修复 + 「无码物品」入口落地

主要文件：`components/reception/rent_recept_form/{js,wxml,wxss,json}` + `pages/admin/reception/recept_new.js`

#### 一、附件项录入校验修复（plan 流程）

- 📌 **根因**：`evalEntry` 对 `is_associate=true` 做了特殊豁免（只校验 `pick_type`，跳过 `code/name`），导致搜索单品自动带出的附件项（如双板带雪杖）默认显示 `已录入`，即使名称/编码都为空 → 用户被误导直接结算 → 漏录入数据被提交后端
- ✅ **修复**：删除 `is_associate` 豁免分支（`rent_recept_form.js:37-54`），附件项走与主项一致的 noCode/name/code/pick_type 校验。附件项默认 `noCode=true` → 必须录名称才算完整
- 下游自动生效：`_refreshRentals` / 6 个 mutator / `evalRental` / `_refreshSummary` 都已对接 `evalEntry`，无需另改

**plan 文件**：`/Users/cangjie/.claude/plans/stockli-stockli-noble-moon.md`

#### 二、「无码物品」入口完整实现

- ✅ **页面入口** (`recept_new.js`)：`onAddAction` 处理 `action='noCode'` 分支，新增 `_addBlankRental()` 方法 — 构造 `category_id=null` 的 rental + 主项 rentItem（`is_associate=false, noCode=true, name=null, code=null, pick_type=defaultPickType`）→ `_appendRentals` 追加并保存
- ✅ **`evalEntry` 增加分类必填**：`!is_associate && !category_id` → 最高优先级返回 `分类未选`，先于 `名称未填` / `编码未填` / `模式未选`
- ✅ **`_refreshRentals` 派生**：`needsCategory = !is_associate && !category_id && !noNeed` → 标题改为 `待选分类`、`expandedItem[ikey]` 首次默认 `true`（无码物品创建后立刻展开）
- ✅ **rentItem 卡片增加「分类」form-group**：展开态首行，仅 `!rit.is_associate` 显示；可点击区，显示 `class_name` 或 placeholder「点此选择分类」
- ✅ **分类选择 modal**：`van-popup position=bottom round + van-tree-select` 单实例（组件根尾部，与 van-calendar / search-product-fuzzy 同位置）；分类树懒加载（顶级 + 子分类按需拉）
  - `rent_recept_form.json` 注册 `van-popup` / `van-tree-select`
  - State：`categoryShow / categoryRidx / categoryIidx / categoryItems / categoryActiveId / categoryMainActiveIndex / categoryRaw / _categoryChildMap`
  - Handlers：`onItemCategoryTap` / `_ensureCategoryTreeLoaded` / `_loadCategorySub` / `onCategoryNav` / `onCategoryItemTap` / `onCategoryClose` / `onCategoryConfirm` / `_applyCategoryChange`
- ✅ **核心联动 `_applyCategoryChange(ridx, iidx, newCat)`**：
  1. 并行拉 `getRentCategoryPromise(newCat.id)`（含 `associateCategories`）+ `getShopByNamePromise(shop)`
  2. 拉 `getRentPriceListPromise(shopId, '分类', newCat.id, '门市')`
  3. 主项 rentItem 字段更新（`category_id / category / class_name / categoryName / chooseCategories / canChooseCategory`）；**保留 `noCode/noNeed`**，不改用户已有的「无码」状态
  4. 删除原所有 `is_associate=true` 附件项 + 按新分类的 `associateCategories` 重建（字段对齐 `BuildAssociates` 默认值）
  5. 同步 rental 字段：`category_id / category / name / guaranty / realGuaranty / guaranty_discount / priceList`
  6. `util.createRentalDetail` 重算 `pricePresets`（`getDailyRate` 取 `pricePresets[0].price`）
  7. emit `syncRent` (needUpdate=true) → 父页 `saveRentReceptOrder`
- ✅ **反复切换主项分类**：每次切都触发完整重建。从有附属分类切到无附属分类 → 附件项自动消失；反之自动带出
- 📌 **后端兼容**：`Rental.category_id` / `RentItem.category_id` 都是 `int?`（可空），允许保存 `category_id=null` 的无码物品 rental 到后端
- 📌 **缓存提示**：改完 `components/reception/*` 后微信开发者工具需 `Tools → Cache → Clear all data + Clear file cache + 编译`，否则可能看到旧行为

**plan 文件**：`/Users/cangjie/.claude/plans/stockli-stockli-noble-moon.md`（仅第一项走过 plan，「无码物品」基于用户多轮 feedback 直接实施）

### 2026-05-11 — 通用结算页 + 押金/租金 modal 编辑

主要文件：
- 新建 `pages/payment/settle/{js,wxml,wxss,json}`
- 新建 `components/order-summary-card/{js,wxml,wxss,json}`
- 新建 `components/order-payment/{js,wxml,wxss,json}`
- 改 `pages/admin/reception/recept_new.js`（onCheckout 接通 PlaceRentOrder + navigateTo）
- 改 `app.json`（payment subpackage 注册 settle/index）
- 改 `components/reception/rent_recept_form/{js,wxml,wxss}`（押金/租金 modal）

#### 一、通用结算页（settle，非租赁专用）
- 用户最初提议名 `rent_settle`，确认后改为 `settle`（养护/零售共用）
- 旧版 `components/payment/payment.*` 保留不动，新组件全部走 orderId-only 接口
- 微信支付：`Order/GetWepayPayment/{id}` → `MediaHelper/GetQRCode` → WebSocket 监听 `paymentpaid`
- 支付宝 mock：复用微信 prepay 接口，标 TODO，等支付宝小程序方案
- 其他方式：红色按钮 → `wx.showModal` 二次确认 → `Order/EffectUnpaidOrder?payMethod=...&payLater=false`
- 📌 一次性踩坑：app.json 把页面注册到主 pages 但 `pages/payment` 已是 subpackage root → 编译报 "Should not exist in subPackages"，改注册到 subpackage 内 `"settle/index"`
- UI 调整：删自定义 topbar 避免与默认导航栏重叠；`util.showAmount` 已带 ¥ 不要再拼；底部挂 `reception-tabbar`；main 加 safe-area 底部 padding

#### 二、reception/recept_new onCheckout 接通
- 原本只是 `wx.showToast('去结算（下一步迭代）')`
- 改为：`Order/PlaceRentOrder/{id}` 把订单转 valid=1 → `wx.navigateTo({url: '/pages/payment/settle/index?orderId=...'})`
- 失败时统一 toast「下单失败」

#### 三、押金/租金 modal 二次确认
- 原 input + blur 改为 view + bindtap，wxml 用 `<text class="kv-input--display">`
- 流程：tap → `wx.showModal({editable:true, content: 当前值})` → 输入 → 第二个 modal 确认金额 → `_applyPkgDeposit` / `_applyPkgRate`
- 📌 押金 round-trip 坑：服务端不保留 `realGuaranty`，`_refreshRentals` 用 `realGuaranty ?? guaranty` 取值。`_applyPkgDeposit` 必须同时设 `guaranty=v` + `guaranty_discount=0`，否则 sync 回来 UI 被刷回旧值。租金存在 `pricePresets[0].price`，服务端原样返回，无此问题
- 加 `.kv-cell--tap:active` 按压反馈样式

### 2026-05-11（晚上） — 押金净额显示 + 订单号回填 + 结算闭环

主要文件：
- 改 `components/reception/rent_recept_form/{js,wxml}`
- 改 `components/order-summary-card/index.wxml`
- 改 `pages/admin/reception/recept_new.js`

#### 一、押金显示改为净额
- ✅ **`_refreshRentals` 派生 `netDeposit`**：`realGuaranty − guaranty_discount`，`Math.round(x * 100) / 100` 规避 `300 − 299.95 = 0.04999...` 浮点；`_depositLabel` / `_depositInput` 都改用 netDeposit。`realGuaranty <= 0` 时取 0（新建无目录的 rental）
- ✅ **`_refreshSummary` 求和后再 round**：`deposit` / `reduce` 各 `Math.round(* 100) / 100`，避免多 rental 累加放大浮点误差
- ✅ **购物车栏文案「减免」→「已减免」**（`rent_recept_form.wxml`）告诉用户减免已生效，不需要再操作
- ✅ **合并冲突清理**：白天 commit f06a21b 的 modal-tap 写法与本地基于旧 input blur 假设的改动冲突。保留 modal-tap（`onPkgDepositTap`/`_applyPkgDeposit`），丢弃 blur 分支（wxml 已不是 input，blur 分支代码跑不到）

#### 二、订单号显示正式编号
- ✅ **`order-summary-card/index.wxml`**：`#{{order.id}}` → `#{{order.code || order.id}}`，下单后展示 `WL_ZL_260511_00001` 服务端生成码；未 placed 回退到内部 id 兼容历史数据
- 📌 **服务端码规则**（`SnowmeetApi/Controllers/OrderController.cs:389 GenerateOrderCode`）：`{shopCode}_{bizCode}_{yyMMdd}_{序号5位}`，租赁 `bizCode=ZL`，序号按同前缀订单数+1。仅在 `UpdateOrder` 看到 `code==null && valid==1`、或 `PlaceRentOrder` 显式调用时生成

#### 三、结算闭环
- ✅ **`saveRentReceptOrder` 改返 Promise**：成功 `resolve(submitted)`，失败 `reject(err)`；fire-and-forget 调用点（`onSyncRent` / `_appendRentals`）补 `Promise.resolve(this.saveRentReceptOrder()).catch(() => {})` 吞掉 rejection，避免 unhandled rejection 警告
- ✅ **`onCheckout` 串成完整链**：
  1. `await` `saveRentReceptOrder`（确保最新编辑落盘，规避用户改完押金立即点结算、syncRent 触发的保存还在飞行的竞态）
  2. 调 `Order/PlaceRentOrder/{order.id}` → 服务端 `GenerateOrderCode` + `valid=1` + 写 Guaranty + 算 `paying_amount`
  3. `setData({ order: rentOrder })` 回填本地，含新生成的 `code`
  4. `wx.navigateTo` 跳 `/pages/payment/settle/index?orderId=...`
- ✅ **统一 loading + catch 兜底**，失败 toast「下单失败」

### 2026-05-12 — payment_entry 顾客扫码支付页轻量化重做

主要文件：`pages/order/payment_entry/{js,wxml,wxss}`

入口：顾客扫店员侧的支付二维码（由 `components/order-payment` 或 `components/payment/payment.js` 生成，URL 形如 `https://mini.snowmeet.top/mapp/order/payment_entry?paymentId={id}`）落地到本页。原页面只有 5 行裸 `view` + `van-button`，视觉粗糙、缺业务明细。

- ✅ **舍弃 fui-* 改纯 CSS 卡片布局**
  - 整页背景 `#F8F8F8`；信息分 4 段卡片（订单信息 / 租赁内容 / 金额 / 支付按钮），白底 + `12rpx` 圆角 + `24rpx` 内边距，无阴影
  - 分组标题：左侧 `6rpx` 蓝色竖条 + `30rpx` 半粗体（替代 `fui-section`，靠 `::before` 伪元素实现）
  - 行 (`.row`)：flex space-between，标签 `#666` 左 / 值 `#333` 右；金额行类 `.value--amount`、需支付红色高亮 `.value--pay`（`#E64340 + 32rpx + 600`）
  - 主色 `#2EA6D0`（按钮、竖条）/ 警示红 `#E64340`（需要支付金额、支付成功提示）
- ✅ **租赁明细折叠交互**（手写 wx:if，未引入 `van-collapse`）
  - Rental 主行 `bindtap="toggleRental"` 切换；右上角 `▾` icon，展开时 rotate 180°（`.rental-head--open`）
  - `payment_entry.js` 新增 `toggleRental(e)`：`setData({['order.rentals[' + idx + '].expanded']: !this.data.order.rentals[idx].expanded})`
  - 默认折叠（`rental.expanded = false`）；展开后浅灰底 `#FAFAFA` 圆角块内列各 rentItem
- ✅ **租赁卡内容**（按用户多轮反馈最终形态）
  - Rental 主行：`displayName` + `N 件▾` + 押金/日租金一行（`.fee-row` + `.fee-group`，各占 `300rpx` 按 5 位数字预算 `¥99999.00` 对齐）
  - rentItem 明细只列：**编码**（`item.code`）/ **名称** / **品类**（`category.name || class_name || '-'`）。**舍弃**取/还时间和状态字段（用户明确不要）
- ✅ **`renderData(order)` 扩展**
  - 新增 `order.total_amountStr = util.showAmount(order.total_amount)`
  - `order.type == '租赁'` 时遍历 `order.rentals` 派生：`displayName`（`rental.name || rentItems[0].name || '租赁'`）、`guarantyStr`、`totalRentalAmountStr`、`expanded=false`；每个 rentItem 派生 `categoryName`
- ✅ **不动的部分**
  - `onLoad` / `onShow` / `pay()` 全部保留；入参解析（`options.paymentId` 或 `options.q` 二维码 scene）保持原样
  - 后端 API 未动（复用 `Order/GetOrderFromPaymentByCustomer/{paymentId}` 拉单 + `Order/WechatPayByOrderPayment/{paymentId}` 调起支付）
  - `van-button` 沿用（项目仍保留 vant-weapp，仅 fui-* 是计划弃用对象）
- ✅ **非租赁类型最小版**：`B 段租赁内容` wx:if `order.type=='租赁'` 跳过；餐饮/零售/押金等仅渲染 订单信息 + 金额 + 按钮三段，留待后续扩展
- 📌 **`pay()` 内的旧 bug 顺手未改**：`pay()` 第二次拉单时把 `payment.id` 当成 paymentId 传，但拉回来的字段是 `nonce/prepay_id/sign/timestamp`，第二次读这些就是 undefined。本次不在范围内，保留原状

**plan 文件**：`/Users/cangjie/.claude/plans/pages-order-payment-entry-valiant-sky.md`

### 2026-05-13 — 万龙租赁数据导出 + CSV 对账 + 身份验证 plan

主要产出（`D:\snowmeet\snowmeet_ai_doc\`）：
- `export_wanlong_rent_orders.py` + `wanlong_rent_orders_2025-10-15_2026-04-15.xlsx`：万龙体验中心 2025-10-15~2026-04-15 租赁订单导出，3 个 sheet（订单汇总 2325 / 订单明细 2839 / 支付明细 2125），所有日期字段拆为「日期+时间」两列，支付明细按 wepay_key JOIN 出真实微信商户号
- `compare_detail_vs_csv.py` + `comparison_report.xlsx`：与外部下载的 3 个 `ZuLinDingDan_*.csv` 对账（5 sheet），CSV 仅取 `WT_` 开头
- `export_csv_excel_diff.py` + `split_excel_only_by_reason.py` + `csv_excel_diff.xlsx`：差异表 8 sheet。仅 Excel 有的 791 行明细按 `api/Rent/GetConfirmedRentOrder` 5 条规则拆为 6 类（paid为0 / closed为0_未关闭 / close_date为空 / hide为1_隐藏 / 含非微信非支付宝 / 应通过但CSV没有）
- `payment_identity_verification_plan.md`：支付前身份验证实施方案（按 PRD V0.13 流程图 image1.png + 用户 4 条修正版），决策树 4 状态 + 错误，**待开工**
- 旧版全店导出脚本/产出：`export_rent_orders.py` + `rent_orders_2025-10-15_2026-04-15.xlsx`（可保留对比，也可清理）

📌 关键发现 / 教训：
- 之前 CLAUDE.md 提到的 `Order/PlaceRentOrder` / `OrderController.GenerateOrderCode` 在 master 分支不存在，全部在 `origin/ai` 分支。涉及订单业务的后端开发前必须先 `git checkout ai`，否则改的是 `OrderOnline.cs` 而非新的 `Order.cs`
- `OrderOnline.payer` 字段几乎是死字段（仅 `Mi7OrderController.cs:125` 单点写入未读），新功能不可重用，需独立加 `pay_member_id`
- 万龙 2325 单实付 ¥7,204,721 / 退款 ¥6,604,799 / 结余 ¥599,922 — 押金大头基本都退回了，季度净留存 60 万
- 万龙微信支付分 3 商户：1604184933(万龙租赁，主力 67% / 1349 笔 ¥483 万) / 1636313350(旗舰租赁 / 316 笔 ¥83 万 — 历史遗留) / 1636404775(万龙零售 / 9 笔 ¥1.1 万)
- Excel 明细 ¥250 万 vs CSV ¥53 万 差 ¥197 万，主因是 `rental.settled=0` 的未归还订单（如 `WT_ZL_251030_00009` "试滑双板(有用勿删)" / "测试" 已付 ¥0.04 但 rental_detail 累积 189 天 ¥7-9 万虚账）
- 微信开发者工具 `getPhoneNumber` 不返真实号，身份验证测试必须真机；建议加 `?mockCell=` 开发后门

### 2026-05-13 晚 ~ 2026-05-14 — 接口排查 + xlsx 重构 + skill 落地

接续下午的万龙租赁导出工作。本次三条主线：诊断接口数据为何在前端报表里看不见、把导出脚本通用化成 skill、把今晚的对账逻辑（测试列+临时订单+异常标红）固化进 skill。

#### 一、`api/Rent/GetConfirmedRentOrder` 接口数据排查

主要文件：`SnowmeetApi/wwwroot/background/rent/rent_report_new.html` + 直查 DB

- 📌 用户报告 `WT_ZL_260314_00006` "查不出来"，DB 直查所有字段都满足接口 5 条规则；本地起 SnowmeetApi 用真实 sessionKey 调接口 — **数据确实返回**（rows=89, has_target=True）
- 📌 真正根因 1：`rent_report_new.html:87-91` 的 var 提升 bug — `var tData = []; render(); var totalAmount = 0` — `render()` 在 `totalAmount` 赋值前调用，264 行 `totalAmount.toFixed(2)` 因 undefined 抛错。修：把 `var totalAmount = 0` 移到 `render()` 之前一行
- 📌 真正根因 2：这条订单 `rental.entertain=true`，`rent_report_new.html:123` 的 `if (rental.entertain != 0) continue` 把它跳过（招待单不计入"租赁订单报表"，业务语义正确）
- 📌 类似根因覆盖更多订单：
  - `WT_ZL_260316_00004`（"5 条 5 标签都未命中"之一）：`totalRentalAmount=220` 被 220 的 rental 级减免（biz_type='租赁' AND biz_id=rental.id）抵消为 0，前端 `>= 1` 过滤掉
  - `WT_ZL_260103_00013`：rental_detail 中 `charge_type='租金'` 的明细 `valid=0` 失效，仅剩 `超时费 120` 有效。`totalRentalAmount`（按 valid=1 求和）= 0 被过滤；120 元收入实为超时费不是租金，数据质量问题

#### 二、csv_excel_diff.xlsx「应通过但CSV没有」sheet 加分类列

把 194 行可能的 CSV 漏单按规则归类（DB 实时查 rental/discount/rental_detail）。新增 6 列：

| 列 | 规则 | 命中数 |
|---|---|---|
| 招待 | `rental.entertain=1` | 18 |
| 体验 | `rental.experience=1` | 74 |
| 减免 | `discount.sub_biz_type='日租金' AND biz_id=rental.id` 总和 | 48 |
| 免除 | 该 rental 在 rental_detail 中无 `valid=1` 明细 | 48 |
| 测试 | `_订单已付金额 < 10` | 31 |
| 减免2 | `discount.biz_type='租赁' AND biz_id=rental.id` 总和（不限 sub_biz_type） | 49 |

剩 6 条 5 标签都不命中的核心样本中，已在第一节定位到 2 条根因（260316 / 260103）；其余 3 条（`WT_ZL_251205_00004` / `WT_ZL_251230_00009` / `WT_ZL_260212_00013` 等）的 `discount.order_id` 全 NULL，没 discount 记录，减免不是 CSV 缺失的根因，需另查

#### 三、wanlong_rent_orders xlsx 重构（订单明细 9→15 列 + 3 sheet 测试列 + 对账后处理）

主要文件：`snowmeet_ai_doc/export_wanlong_rent_orders.py`

- 走 plan mode 评审（plan 文件 `~/.claude/plans/wanlong-rent-orders-2025-10-15-2026-04-rustling-whistle.md`）
- ✅ 修 `OUT` 路径 Windows → macOS 绝对路径
- ✅ pyodbc ODBC 驱动注册：brew 装的 msodbcsql18 + unixodbc 配置在 `/opt/homebrew/etc/odbcinst.ini`，但 pyodbc 默认查 `/etc/odbcinst.ini` → 解决方案 `export ODBCSYSINI=/opt/homebrew/etc`（比改 `~/.odbcinst.ini` 更轻量）
- ✅ DETAIL_SQL 重构 14 列：新增 `是否招待 / 是否体验 / 应付租金 / 减免金额 / 损毁赔偿 / 实付金额`。损毁赔偿用 `charge_type IN ('赔偿金','损坏赔偿')` 兼容（DB 实际只有'赔偿金'，没有'损坏赔偿'）
- ✅ **减免金额最终口径**（用户拍板，每条 rental 严格归属自己的 discount）：
  - A：`discount.sub_biz_id` 指向该 rental 的某个 `rental_detail`（`valid=1`）
  - B：`discount.biz_type='租赁' AND discount.biz_id=rental.id`，且 `sub_biz_id` 不指向该 rental 的任何 detail
  - A ∪ B 取 distinct discount row 求和。**每条 discount 只归一条 rental**，多 rental 单子不重复算
- ✅ 实付金额 = 应付租金 − 减免金额 + 超时费 + 损毁赔偿
- ✅ 3 个 sheet 都加测试列：规则统一为 `订单的 paid_amount < 5` OR `店员姓名含 '苍'`
  - 订单汇总 333 行 / 订单明细 531 行 / 支付明细 95 行
- ✅ 对账后处理：「订单结余 != 订单明细该订单非测试 rental 实付合计」差额 ≥ 0.01 → 订单号标红
  - A 类（结余>0 但订单明细无非测试 rental 行）135 条 → 加「临时订单」列='是'，订单号不标红
  - B 类（rental 存在但金额对不上）23 条 → 订单号标红（B 类负差额大多是 `rental.settled=0` 虚账，正差额是付款进账但 rental_detail 没记够）

#### 四、固化为 skill：`snowmeet_ai_doc/skills/export_rent_order/`

通用化版本，未来导其他店铺/时间段直接复用。

- 新建 `snowmeet_ai_doc/skills/export_rent_order/SKILL.md`（8.5 KB，触发条件 + 环境要求 + 调用方式 + 列结构 + 排错全套文档）
- 新建 `snowmeet_ai_doc/skills/export_rent_order/export_rent_orders.py`（15 KB，argparse 参数化：`--shop --start --end --out --conn --no-postprocess`）
- 已知 6 个店铺预置英文 prefix 映射（`万龙体验中心→wanlong / 万龙服务中心→wanlong_service / 渔阳→yuyang / 南山→nanshan / 怀北→huaibei / 崇礼旗舰店→chongli`），默认输出文件名 `{prefix}_rent_orders_{start}_{end}.xlsx`
- 后处理 `post_process` 函数内化了"临时订单不会标红"的互斥规则（A 类 `continue` 掉，永远不进标红分支）
- 冷启动验证：换机后只需 `brew install msodbcsql18 unixodbc + pip install pyodbc openpyxl + export ODBCSYSINI=/opt/homebrew/etc`

#### 五、聊天记录归档

新建 `snowmeet_ai_doc/sessions/2026-05-13_rent_order_diff_and_skill.md`（9 KB），把今晚 7 个主题完整记录（接口排查 → 分类列 → xlsx 重构 → 测试列 → 标红 → 临时订单 → skill 落地）+ 关键改动文件清单 + 6 条小知识

📌 关键发现 / 教训：
- **macOS pyodbc 看不到驱动**：`export ODBCSYSINI=/opt/homebrew/etc` 一行解决，不要碰系统 odbcinst.ini
- **var 提升只前置声明不前置赋值**：`var x = 0` 在 `render()` 后面 → render 内拿到 `undefined.toFixed()`。所有顶层初始化必须放在第一次调用前
- **discount 表三字段在生产实际同时填**：万龙时段 274 条 discount 全部填了 `order_id + biz_type='租赁' biz_id + sub_biz_type='日租金' sub_biz_id`，所以三 bucket 完全重叠；但脚本逻辑要按字面分类做，应付未来字段稀疏
- **rental_detail.charge_type 只有'租金/超时费/赔偿金'三种值**：DB 不存在'损坏赔偿'，写 SQL 用 `IN ('赔偿金','损坏赔偿')` 兼容
- **rental_detail.valid=0 的失效租金明细会让 totalRentalAmount=0**：前端用 `>= 1` 过滤掉整行，是部分订单"CSV 没有"的根因（数据质量问题，非脚本 bug）
- **多 rental 订单 discount 归属必须严格按 detail/rental 层级匹配**：不能简单 `order_id OR biz_id OR sub_biz_id` 三 bucket OR，否则全单 discount 在每条 rental 上重复算（如 WT_ZL_251230_00011 ¥879.95 会变 ×6=¥5279.70）
- **rental.settled=0 的虚账**：未归还订单按天累积 `rental_detail.amount` 应收记录，做收入分析时要意识到「订单明细.租金总额」可能远超实际应收。报表只看 ≤ 实付金额、不参考租金总额做收入估算

### 2026-05-14（晚） — wanlong_rent_orders_api xlsx 补「订单结余」+ 清科学计数法

主要文件：新建 `snowmeet_ai_doc/add_balance_to_api_xlsx.py`，目标产物 `snowmeet_ai_doc/wanlong_rent_orders_api_2025-10-15_2026-04-15.xlsx`

- ✅ **补列脚本**（plan 流程，文件 `~/.claude/plans/snowmeet-ai-doc-wanlong-rent-orders-api-abstract-bonbon.md`）
  - 读源（数据库直查版）`wanlong_rent_orders_2025-10-15_2026-04-15.xlsx` 的 `订单汇总` sheet，按表头定位 `订单号 / 订单结余` 列号（不写死索引），构 dict
  - 写目标（API 版）`wanlong_rent_orders_api_2025-10-15_2026-04-15.xlsx` 的 `订单` sheet，末尾追加「订单结余」列，复用现有表头样式（粗体白字 + `1F4E78` 蓝底 + 居中，与 `export_wanlong_rent_orders_by_api.py:62-67` 一致）
  - 列宽按视觉宽度 + 上限 36 自适应（仿 `export_wanlong_rent_orders_by_api.py:71-82`）
  - 幂等：检测到已存在「订单结余」列时覆盖写入，不重复追列
- 📌 **源表 2325 行 dict 后变 2319**：数据库直查版「订单汇总」有 6 个订单号重复（dict 覆盖去重）；目标 2325 行未命中 = 0，全部命中
- ✅ **修科学计数法**：用户报告 Excel 打开新表有科学计数法显示
  - 根因：`订单结余` 列有 `-3.63806207381856e-14` 之类的浮点零误差极小值（DB 端计算累加产生），Excel General 格式下自动 `-3.64E-14`；`总计租金` 同时有 `42220.00999999999` 之类小数尾巴
  - 修：脚本写入「订单结余」前 `round(float(v), 2)`；同时对「总计租金」列（API 脚本生成时已有浮点尾巴）做 `round(2)` 清洗；两列都设 `number_format = '0.00'` 锁定显示格式
  - 注：根因在 `export_wanlong_rent_orders_by_api.py` 的 `compute_displayed_rental` 浮点累加，本次仅在 xlsx 层补丁；API 脚本下次重跑仍会带尾巴，需在那时再跑补列脚本兜底（脚本会顺手清掉）

📌 **关键发现 / 教训**：
- **Excel General 格式 + 浮点零误差 = 科学计数法**：DB 端浮点累加产生的 `±1e-14` 级别数值，Excel 默认显示为 `-3.64E-14`。导出脚本写金额到 xlsx 时强制 `round(2)` + `number_format = '0.00'` 一并兜住，比依赖 General 格式可靠
- **API 版与数据库直查版同区间订单数对齐**：两份各 2325 单（其中数据库直查版含 6 个重复订单号）。后续若要给 API 版加任何 DB 派生字段（订单结余 / 实付金额 / 招待标记 等），按订单号查表的模式可复用本脚本
### 2026-05-14 — 支付前身份验证实施 + firstui 清理 + 页面可达性分析

下午到晚上，从前一天的 plan 落到代码，端到端搭起 A 后端 + B 前端 mvp；并把 firstui 死代码清掉、做了全项目页面可达性 review。

#### 一、A 后端切片（origin/ai 分支）

- 新加字段：`Order.wechat_unverified (bool default false)` / `OrderPayment.is_proxy_pay (bool default false)`；DB 用户手工执行 `ALTER TABLE [order] ADD wechat_unverified BIT NOT NULL DEFAULT 0` + `ALTER TABLE [order_payment] ADD is_proxy_pay BIT NOT NULL DEFAULT 0`
- `MemberSocialAccount.cs` 加 4 个 type 常量：`TYPE_WECHAT_MINI_OPENID / TYPE_WECHAT_UNIONID / TYPE_CELL / TYPE_ALIPAY_PAYERID`
- 新建 [`Controllers/Order/PaymentIdentityController.cs`](../SnowmeetApi/Controllers/Order/PaymentIdentityController.cs)（~460 行）：
  - `GET CheckPayerIdentity(paymentId, payerType, scannerId, sessionKey)` 只读 + 幂等，5 状态决策树（error / phone_required / direct / direct_to_scanner / choose_identity）
  - `POST ConfirmPayIdentity` 3 action：`submit_phone`（微信 AES_decrypt encData / 支付宝 stub 接 phoneMock）/ `choose (self|proxy)` / `confirm_direct`
  - 幂等锚 `op.member_id != null && status=='待支付'` → 直接返既有
  - `payerType=alipay` 一律 `Order.wechat_unverified = true`
- 用 `winget install Microsoft.DotNet.SDK.9` 装 .NET 9 SDK，`dotnet build` 0 错误（14 警告全部源自历史文件，新 controller 0 警告）
- 本地 `dotnet run` swagger 烟测：GET 5 状态 × 2 payerType 路由 + POST `ConfirmPayIdentity` `[FromBody]` 绑定 + 幂等 short-circuit 全部正常；DB 连接通过 `config.sqlServer` 走生产读取，paymentId=42540 真实订单能拉到归属 `苍杰（个人）135****7897`

#### 二、B 前端切片（snowmeet_wechat_mini ai 分支）

- `utils/data.js` 加 `checkPayerIdentityPromise` + `confirmPayIdentityPromise`（沿用 `util.performWebRequest` 的 GET/POST 语义：data 为 undefined 走 GET，否则 POST）
- 新建 `components/pay-identity-confirm/`（4 文件）：phone_required / direct_to_scanner / choose_identity / error 四态卡片；choose_identity 用 2 个按钮「正常支付（订单转归我）」+「替人代付」（代付二次 `wx.showModal` 确认）；视觉对齐 `pages/order/payment_entry` 的 `#2EA6D0` 主色 + 12rpx 卡片
- `pages/order/payment_entry.{js,wxml,json}` 改造：
  - `data` 新增 `paymentId / scannerId / identity` 三个字段
  - `onShow` 在 `getOrderFromPaymentByCustomer` 后链调 `_refreshIdentity()`
  - 子组件 `bind:refreshed` → `onIdentityRefreshed` 更新 identity state
  - 支付按钮加 `identity.status === 'direct'` 守卫（wxml `wx:if` 直接隐藏，pay() 内再守一层防御性 toast）
  - 注册 `pay-identity-confirm` 到 page json `usingComponents`

#### 三、踩坑 + 修复（顾客真机 paymentId=42540）

- **现象**：扫码进 payment_entry 后页面报「无法支付 / 无法获取微信账号，请重新登录后再试」
- **根因**：我前端代码里 `app.globalData.member.wechatMiniOpenId` 取不到值的兜底分支被命中。深挖：`Member.wechatMiniOpenId` 是后端 Member 模型的**计算属性**（getter 遍历 `memberSocialAccounts` 集合），依赖序列化时 MSA 集合被一并带回。顾客扫码深链场景下 `app.globalData.member` 不一定齐全
- **修复策略**：让后端兜底。`_resolveStatus` 加 `sessionKey` 参数 → `scannerId` 为空时按 `mini_session.member_id` 反向定位扫码方会员；3 个 action 处理器（`_submitPhone` / `_applyChoice` / `_applyConfirmDirect`）都串上 sessionKey；前端去掉 scannerId 空就报错的预检查，scannerId 拿不到时发空串给后端

#### 四、清理 + 分析

- **firstui 死代码清理**：删除 6 个未使用组件（`fui-badge / fui-tabs / fui-toast / fui-top-popup / fui-utils / fui-wing-blank`），净删 1435 行；同步从 `app.json` 移除 `fui-top-popup` 注册 + 从 `fui-config/index.js` 移除 `fuiWingBlank` 配置块。保留 `fui-config`（喂 `wx.$fui`）+ `fui-css`（全局 @import）+ 其它 15 个有 wxml 引用的活组件
- **页面可达性分析**：写 Python 静态可达性脚本（[`unreachable_pages.md`](unreachable_pages.md)），从 `pages/index/index` + `pages/mine/mine` 出发递归 BFS（含组件 `usingComponents` 传导），117 页面归 3 类：
  - A 完全可达：66
  - B BFS 漏但全局有引用：13（多半新流程链路缺主入口）
  - C 完全孤立：62（其中部分是 QR 扫码外部入口，要逐项区分）

#### 五、关键产出

| 项 | 状态 |
|---|---|
| 后端 controller + 模型 + DB schema | ✅ 编译 + swagger 烟测过 |
| 前端组件 + payment_entry 改造 | ✅ 静态完整，运行时未真机验证 |
| sessionKey 兜底修复 | ✅ 后端 build + 本地烟测过 |
| 顾客扫码 → 选代付/归我 → 完成支付端到端 | ⏳ **待用户部署 ai 分支后端 + 重编小程序后真机测试** |
| 支付宝真实手机号解密 | ⏳ 下次切片 |
| 决策时机迁到 wepay/alipay notify 回调 | ⏳ 下次切片 |
| firstui 死代码清理 6 个 | ✅ |
| 页面可达性报告 | ✅ 已生成，待用户 review 决定删哪些 |

#### 关键改动文件

| 仓库 | 文件 | 操作 |
|---|---|---|
| SnowmeetApi (ai) | `Models/Order/Order.cs` | +1 行 `wechat_unverified` |
| SnowmeetApi (ai) | `Models/Order/OrderPayment.cs` | +1 行 `is_proxy_pay` |
| SnowmeetApi (ai) | `Models/Member/MemberSocialAccount.cs` | +4 个 type 常量 |
| SnowmeetApi (ai) | `Controllers/Order/PaymentIdentityController.cs` | 新建 ~460 行（含 sessionKey 兜底） |
| snowmeet_wechat_mini (ai) | `utils/data.js` | +2 个 Promise 包装 |
| snowmeet_wechat_mini (ai) | `components/pay-identity-confirm/{json,js,wxml,wxss}` | 新建 4 文件 |
| snowmeet_wechat_mini (ai) | `pages/order/payment_entry.{js,wxml,json}` | 接入 identity 状态机 |
| snowmeet_wechat_mini (ai) | `app.json` | 移除 fui-top-popup 注册 |
| snowmeet_wechat_mini (ai) | `components/firstui/{fui-badge,fui-tabs,fui-toast,fui-top-popup,fui-utils,fui-wing-blank}/` | 删除 |
| snowmeet_wechat_mini (ai) | `components/firstui/fui-config/index.js` | 清理 fuiWingBlank 配置 |
| snowmeet_ai_doc | `payment_identity_verification_plan.md` | 覆盖原 5-13 旧版（详细化 + 双通道 + wechat_unverified） |
| snowmeet_ai_doc | `payment_identity_verification_requirements.md` | 新建：需求文档（业务视角）|
| snowmeet_ai_doc | `unreachable_pages.md` | 新建：可达性分析报告 |

#### 学到的小知识

1. **Member 的计算属性序列化依赖关联集合被 Include**：`Member.wechatMiniOpenId` 看似普通 getter 但其实遍历 `memberSocialAccounts`，如果集合没 Include 进来就返 null。任何新接口要用 openid/unionid/cell 都先确认调用链 Include 链路完整；否则就走 sessionKey → mini_session 反查
2. **System.Text.Json 默认序列化 read-only properties**：`Member.wechatMiniOpenId` 没 setter 但仍会出现在响应 JSON 里。问题是值依赖关联数据被加载（见上一条）
3. **`OrderPayment.member_id` 是付款方的天然落点**：原 plan 想加 `Order.pay_member_id`，但 `OrderPayment` 已有 `member_id` 字段（建模时就为付款方留位），无需新增 — 加在 OrderPayment 上才是按付款粒度记代付的正确语义
4. **wx.$fui 全局变量陷阱**：fui-button / fui-icon 等组件运行时读 `wx.$fui` 拿默认值。删 fui-config 会让这 5 个组件运行时拿不到默认 props，组件可能正常工作但视觉默认值丢失 — 不能轻删
5. **小程序静态可达性 BFS 必须含组件传导**：直接扫页面引用会大量误报，因为很多导航发生在被引用的组件内部。BFS 时把页面的 `usingComponents` 当成边，递归到组件文件再扫 URL 引用
6. **paymentId 是 OrderPayment.id 不是 Order.id**：顾客扫的二维码 URL 是 `?paymentId=xxx`，对应 `order_payment` 表主键；`PaymentIdentityController` 用 paymentId 索引（一单可分多笔付款，身份验证按付款粒度）

#### 文档落地

- 需求文档：[`payment_identity_verification_requirements.md`](payment_identity_verification_requirements.md)（业务视角，9 章节，PM 可读）
- 实施方案：[`payment_identity_verification_plan.md`](payment_identity_verification_plan.md)（开发视角，覆盖前一天的 plan）
- 可达性报告：[`unreachable_pages.md`](unreachable_pages.md)（待用户人工 review 决定删除范围）

### 2026-05-14（深夜） — Claude Code hook 配置：start-work 前自动 pull / end-work 后自动 push

主要文件：`.claude/settings.local.json`（仓库 `snowmeet_ai/` 下，本机 gitignore，不入库）

- ✅ **PreToolUse / Skill(start-work)** → `git -C snowmeet_ai_doc pull --ff-only`
  - `--ff-only`：仅 fast-forward，本地有未推送提交或分叉时拒绝合并、不自作主张产 merge commit
  - 失败（网络/冲突）不阻断 start-work，仅打印 warn
- ✅ **Stop hook** → 检测 `snowmeet_ai_doc/sessions/*.md` 最近 3 分钟有改动时自动 `git add . + commit + push`
  - 时机选择关键：`PostToolUse + Skill(end-work)` 在 skill 工具返回瞬间触发，**早于** end-work 实际写入 CLAUDE.md / sessions 文件，无东西可推；改用 Stop hook + 启发式（sessions/ 最近 mtime）匹配真实写入完成时机
  - 幂等：working tree 干净时 push 输出 `Everything up-to-date` no-op；非 end-work 场景（普通对话停下、/clear、/compact）由于 sessions/ 没新改动也不会误触发
  - 输出含改动列表（`?? = 新文件 / M = 修改 / D = 删除`），看得见 hook 实际做了什么

📌 **关键发现 / 教训**：
- **Skill 工具的 PostToolUse 时机不等于 skill 工作流完成**：Skill 工具返回 = skill 指令加载进 context，模型尚未执行其步骤。所以"在 skill X 之后做 Y"的 hook 不能简单 `PostToolUse + Skill(X)`，要么 Stop hook + 启发式，要么 Write/Edit 路径过滤
- **`git add -A` ≈ `git add .` (from repo root)**：两者在 `git -C $REPO` 上下文下功能一致；新文件（untracked）通过 `git status --porcelain` 的 `??` 状态码呈现，二者都会暂存
- **doc 仓库存在过未解决的 merge conflict**：本次 end-work 写文件时发现 CLAUDE.md 残留 `<<<<<<< HEAD` / `>>>>>>>` 标记被提交进了 d0d80e1 merge commit。修复方法：手动改掉再 commit。**未来 merge 后必须先 `git status` 查 unmerged，不能直接 commit**

### 2026-05-15 — 新增财年版租赁导出 skill（单 sheet 宽表，财务视角）

主要产出：`snowmeet_ai_doc/skills/export_rent_order_fiscal_year/{SKILL.md,export_rent_orders_fy.py}`，产物 `D:/snowmeet/wanlong_rent_orders_fy_2025-05-01_2026-04-30.xlsx`（98 列 × 2325 行）

- 走 plan mode（plan 文件 `~/.claude/plans/vast-launching-clover.md`）。用户按 3 张截图逐列口述定义表头，最终澄清表结构是**5 段动态拼接**而非固定 62 列：固定前缀(17) + 动态支付区(maxPay×5) + 动态退款区(maxRefund×4) + 固定中段(14) + 固定后缀(13)
- 复用对账版 `../export_rent_order/export_rent_orders.py` 的 `SHOP_PREFIX / REFUND_COND / DEFAULT_CONN / write_sheet`（sys.path import 单点真理，两 skill 必须 sibling）
- 实现：预查询 maxPay/maxRefund → 主查询订单级（聚合/标量子查询保粒度）→ 支付/退款明细各一条 → Python 端按 order_id 拼动态列 + 财年体系列，headers 与 row 同处生成防错位
- 端到端验证：2325 行段2/段3 逐笔金额加总 == 支付/退款合计（0 偏差，含 53 个多笔支付单）；测试 333 / 临时订单 135 与对账版同期记录吻合；快照优先 fallback member 验证通过

📌 关键发现 / 教训：
- **`order_payment` 支付成功时间列是 `paid_date`**（不是 create_date；待支付行 paid_date 为 null，create_date 有值）。对账版 PAYMENT_SQL 没取支付时间所以没踩到，本 skill 需要支付日期列时实探发现
- **`payment_refund` 表无退款方式列**：退款方式只能经 `payment_refund.payment_id → order_payment.pay_method` 取原支付通道
- **年度/财年报表必须按 `biz_date` 过滤，不是 `create_date`**：按 create_date 拉 2025-05~2026-04 会带出 biz_date 在 22-23/23-24/24-25 财年的老单尾巴（晚结算/退押金），财年列全乱。改 biz_date 过滤后默认区间订单财年恒 25-26。**代价**：与对账版（create_date 口径）不可 1:1 交叉对账，单列金额仍同源可按订单号比对
- **滑雪租赁 biz_date 天然落在雪季**：万龙 25-26 全 2325 单 biz_date 都在营业区间 2025-10-21~2026-04-09 内，「营/非」全 `营业` 属正常（淡季无租赁单），非 bug
- **`rentProperties.rentStatus` 纯 SQL 无法精确复现**：是 `Order.cs:1062` 依赖 realStartDate/totalSummary/guaranties.payStatus 计算属性的状态机；本 skill 用 SQL 化字段按 `Order.cs:1134-1172` 判定顺序做近似，SKILL.md 标注为「近似需验收」
- **减免合计订单级 vs rental 级口径不同**：本 skill 是订单级三类(order_id / biz_type=租赁 biz_id / sub_biz_type∈日租金,租赁项 sub_biz_id) discount.id 去重 SUM；对账版 sheet2 是 rental 级 A∪B 严格归属。不可复用对账版 SQL 片段
- **discount 表确有 `valid` 列**（int），三关联字段 order_id/biz_id/sub_biz_id 齐全

补充（同日）：用户要求重导一份「order 不论 valid 都导」的版本，结果放 `snowmeet_ai_doc/`。
- 加 `--include-invalid` 开关：用 `__VALID__` 运行期占位（ORDER_FILTER 里放 token，穿过 f-string，main 里 `.replace`）实现可逆放宽，**仅作用于 order 表**；rental/order_payment/discount/order_share/payment_share/member_social_account 的 valid 过滤全不动（用户只点名 order 表）
- 万龙 25-26 带开关实测 98 列 × 3094 行；DB 同条件 `COUNT(*)`=3094 全匹配，`valid=1` 子集=2325（与初版一致）→ 证明超集正确无重无漏
- 含作废单后：营/非 出现 218「非营业」（淡季废弃/测试单 biz_date 落在雪季外，反证营非逻辑正确）、测试 1102（大量未支付测试单 paid<5）、正/闭 关闭 849（作废单多未支付）
- 产物：`snowmeet_ai_doc/wanlong_rent_orders_fy_2025-05-01_2026-04-30.xlsx`（786.8 KB）

再补（同日）：用户要求「只万龙 + code 为空不导」。ORDER_FILTER **恒加 `o.code IS NOT NULL AND LTRIM(RTRIM(o.code))<>''`**（无 code = 未下单/废弃单，非真实业务记录，即便 --include-invalid 也排除，无需额外开关）。
- 万龙 25-26 + --include-invalid 最终：98 列 × **2434 行**（3094 → 2434，剔 660 空 code 行）
- 与 DB「万龙 biz_date区间 + type=租赁 + code非空」全集双向零差：DB 2434 行/2428 去重 ↔ xlsx 2434 行/2428 去重；差 6 = DB 内重复订单号（CLAUDE 早记录的已知现象，非空保留不剔）
- 覆盖核查教训：脚本 `--shop` 必填→导出天然单店；该区间 type=租赁 code非空全 DB 共 2965 单分 5 店（万龙2434/南山250/崇礼227/渔阳31/怀北23），单店产物只含本店，问「是否全包含」要先分清全表 vs 单店口径

再补（同日）：用户要求对重复订单号去重，规则「有成功支付记录 > valid=1 > id 最大」保留一条。
- **重复 code 根因 = `OrderController.GenerateOrderCode` 序号竞态**：序号取「同前缀订单数+1」，并发/快速重复下单算到同一序号 → 同 code（万龙 25-26 有 6 个：5 个 0 付款空单双插 + `WT_ZL_251129_00016` 一条空单 + 一条真单 ¥1000/¥880）。属 DB 数据质量，根治需后端发号加唯一约束/原子自增
- 实现：Python 端按 code 分组，`max(key=(有成功支付, valid==1, id))` 选留；`o.valid` 加进主查询做判据；maxPay/maxRefund 改为去重后保留集 Python 取 max（删原 PREQUERY_SQL 预查询，少一次往返且列数精确）
- 实测 2434 → 2428 行（去 6 重复），关键校验 `WT_ZL_251129_00016` 正确留带钱条而非空单孪生

再补（同日）：用户问「按天 code 尾号有无不连续」。分析去重后 2428 行：168 天每天都从 00001 起，仅 3 天有缺号共 6 个（251031 缺 7/11/14、251107 缺 11/13、251129 缺 15）。逐个查 DB 证实这 6 个尾号**从未生成**（非过滤/去重副作用）。
- **缺号与重复号是同一发号竞态的镜像**：`GenerateOrderCode` 两单同时读到订单数 N、都写 N+1（→ 1 个重复号），订单数已 +2 但只用掉 N+1，下一单读 N+2 写 N+3 → N+2 永久跳过（1 个缺号）。故每次碰撞 = 1 重复 + 1 缺号，6↔6 账完全对上（251031:3 碰撞 3 缺 / 251107:2 / 251129:1）
- 结论：导出完整无丢单，缺号是系统压根没发的序号；脚本无需改，根治在后端发号

### 2026-05-15（续晚） — 财年导出 xlsx 加「次卡」列 + 次卡表勘察

主要文件：新建 `snowmeet_ai_doc/add_cika_column_to_fy_xlsx.py`、改 `snowmeet_ai_doc/wanlong_rent_orders_fy_2025-05-01_2026-04-30.xlsx`

- ✅ **「次卡」列补列**（plan 流程，文件 `~/.claude/plans/start-work-ethereal-allen.md`）
  - 规则（与用户澄清）：`rental.valid=1 AND use_card=1` → "是" / 否则 `order_payment.status='支付成功'` 笔数 ≥ 1 → "否" / 否则 → "-"
  - 实施：仿 `add_balance_to_api_xlsx.py` 的补列模式；一次 SQL 拉两份 dict（命中 use_card 的 code set + 每订单支付成功笔数），按订单号查表填值；幂等（已存在「次卡」列则覆盖）
  - 结果：xlsx 第 100 列 2428 行 `是 19 / 否 2062 / - 347`，3 类样本 spot-check vs DB 全 PASS
  - 注：xlsx 已 6 条重复 code 去重，DB 端 use_card 命中 19 单全部留存（无重复 code 命中）
- 🔍 **次卡相关表盘点**（DB 直查）
  - 核心：`punch_card`(36 行) + `punch_card_used`(0 行)，字段如新增「已知遗留」所述
  - 周边卡券系列：`card`(16365) + `card_detail`(681) + `ticket`(12244) + `ticket_template`(18) + `product_ticket_template`(11)
  - 旧路径：`order_online.pay_memo='次卡支付'`(6 单) / `[order].pay_option='次卡支付'`(RentController.cs:1629)
  - 📌 **关键发现**：`Models/` 下无 `PunchCard` / `PunchCardUsed` C# 模型，表是裸建的 — 写入逻辑可能压根没接通（已记入已知遗留）
- 🔍 **WT_ZL_251222_00009 排查（未完成，被打断）**
  - DB 查实有：`id=64707, valid=1, closed=1, recepting=1, hide=False, pay_option='普通'`
  - **但 `order_payment` 表对 `order_id=64707` 0 行**
  - 推测命中 `api/Rent/GetConfirmedRentOrder` 的 `paidAmount > 0` 过滤被剔出 → 小程序查不到
  - 待续查：rental 数据 / 是否有退款 / `close_date` 是否为空（影响第 3 条规则）

📌 **关键发现 / 教训**：
- **`punch_card` 表结构齐全但无 C# 模型 + `punch_card_used` 0 行**：DB schema 与代码层不同步的典型案例。改/对账次卡相关功能前必须先翻 controller 看实际走哪条路径（pay_option 字符串 vs punch_card 表），不要假定有结构化表就一定接通了
- **xlsx 补列前先核对 sheet 名**：财年版 sheet 名是「年度租赁」而非「订单」（与对账版不同）。补列脚本第一次跑因死写 "订单" 报 KeyError，靠 `wb.sheetnames` 兜底打印才发现
- **`pyodbc.connect` 参数化执行 SQL 时用 `?` 占位符防注入**：本次脚本里店铺/日期/N'支付成功' 都走参数化，含中文常量也无编码问题

### 2026-05-16 — 财年导出 xlsx 增加「支付明细」+「支付流水」2 个 sheet

主要文件：新建 `snowmeet_ai_doc/add_payment_detail_sheet_to_fy_xlsx.py`（~290 行），改 `snowmeet_ai_doc/wanlong_rent_orders_fy_2025-05-01_2026-04-30.xlsx`（追加 2 sheet）。**plan 文件**：`/Users/cangjie/.claude/plans/start-work-graceful-pine.md`（多轮 plan 演进按用户口径迭代列定义）。

#### 一、「支付明细」sheet（22 列 × 2141 行，每笔成功支付一行）

固定 10 列：
- 订单号 / 支付订单号 (op.id) / 支付方式
- **支付账户**：微信支付=JOIN `wepay_key.mch_id` 取真实商户号（万龙 3 个：`1604236346` 主力 1349 笔 / `1636313350` 旗舰租赁 332 笔 / `1636404775` 万龙零售 9 笔）；支付宝=空；其他=空
- **顾客ID**：`COALESCE(NULLIF(op.open_id,''), op.ali_buyer_id)`（微信 openid / 支付宝 ali_buyer_id）
- 支付日期 / 支付时间 (来自 paid_date，NULL 时 create_date 兜底) / 支付金额 / 退款金额 / 支付结余 (= 支付金额 − 退款金额)

动态列：
- maxRefund × 4：退款k 日期/时间/金额/方式（=原支付通道，因 payment_refund 表无 pay_method 列）
- maxShare × 3：分账k 金额/成功/对象（`order_share_relation.name`），成功 4 态「是/否/作废/空」对应 `(success, valid)`：
  - 是：success=1（成功入账）
  - 否：success=0（接口驳回失败，全 12 笔都是支付宝）
  - 作废：valid=0（请求生成后立即软删，submit_time 多为 NULL，未真实发出）
  - 空：success=NULL valid=1（待回调）

订单号集合来自主 sheet「年度租赁」的「订单号」列（不重做 DB dedup），与年度租赁 1:1 可交叉对账。

#### 二、「支付流水」sheet（8 列 × 5783 行，3 类成功交易合并时间线）

列：订单号 / 支付方式 / 支付账户 / 商户订单号 / 类型 / 交易金额 / 日期 / 时间

3 类成功交易按日期+时间升序穿插：
- 支付 2141 笔（op.status=支付成功 AND op.valid=1，金额正）
- 退款 2088 笔（命中 REFUND_COND `state=1 OR refund_id<>''`，金额负）
- 分账 1554 笔（success=1 AND valid=1，金额负）

商户订单号字段：支付走 `op.out_trade_no`、退款走 `pr.out_refund_no`、分账走 `ps.out_trade_no`。out_trade_no 命名约定编码了交易类型：`{订单号}_ZF_NN`（支付）/`..._ZF_NN_TK_MM`（退款）/`..._ZF_NN_FZ_MM`（分账）。

支付方式/支付账户：退款/分账继承自所属 payment。交易金额合计 ¥376,027.23（含符号 SUM = 实际净流入）。

#### 三、对账校验全部通过

| 项 | 支付流水 | 年度租赁 | 结果 |
|---|---|---|---|
| 支付总额 | 7,209,321.57 | 【支付k】sum 7,209,321.57 | ✓ |
| 退款 abs | 6,604,799.33 | 【退款k】sum 6,604,799.33 | ✓ |
| 分账 abs | 228,495.01 | 实分账金额 sum 228,495.01 | ✓ |

#### 四、关键发现

- **`payment_refund` 表无 `valid` 列**：所有过滤只能走 REFUND_COND；盲加 `pr.valid=1` 会 SQL 报错（参考 export_rent_order skill 的 PAYMENT_SQL 也不写）
- **支付账户 ≠ 顾客 ID**：`open_id`/`ali_buyer_id` 是顾客侧 ID；"支付账户"语义应取 `JOIN wepay_key.mch_id` 真实商户号
- **年度租赁的「实分账金额」严格等于 `payment_share` 中 success=1 AND valid=1 的 SUM**（228,495.01 完全等值）；整表 2428 行 `应分 − 实分 = 待分` 行级零差异
- **9 单 ¥2,919.98 应分但 payment_share 不齐**：4 单完全无 ps 行 + 5 单 ps 行金额不齐；其中 `WT_ZL_251127_00009` 应分 ¥0.02 但生成 2 笔 ¥0.04 是**反向多生成**，用 abs(diff) > tol 才不漏
- **WT_ZL_260223_00007 是典型「应分但作废」**：order_share os_id=1519 amt=650 dealed=1，但 payment_share ps_id=1413 valid=False（submit_time=NULL，订单 closed=1+hide=True 后系统主动放弃这笔分账）
- **分账失败 12 笔全部支付宝**（微信 0 笔）：8 笔 ILLEGAL_SETTLE_STATE（退款 → 分账时序问题）/ 1 笔 BALANCE_NOT_ENOUGH / 1 笔 ALLOC_AMOUNT_VALIDATE_ERROR（分账 > 可分余额）/ 1 笔 DISCORDANT_REPEAT_REQUEST。真实业务损失 ~¥370（260325_00005 ¥260 + 251203_00003 ¥110）
- **SQL Server `IN` CTE 双使用时参数翻倍**：CTE 里两处 IN 同批次 → 占位符重复一遍，每批 ≤1000 才不超 2100 上限
- **`out_trade_no` 命名编码业务类型**：可凭字符串判断（`_ZF_` / `_TK_` / `_FZ_`）
- **`should - got > tol` 单向比较会漏反向差额**：用 abs(diff) > tol 才完整

#### 五、明日（2026-05-17）待验证

- Excel 打开 xlsx 肉眼检查 3 sheet 列结构 + 样本数据
- 9 单 ¥2,919.98 应分缺口订单是否需要人工补分账
- 分账失败 12 笔归因是否准确（按错误码归类后告知运营）
- 是否需要在「支付明细」加「应分账金额」列（合并 order_share + payment_share 维度）

### 2026-05-17 — 财年 xlsx 加「店员openid」+「union id」改「顾客openid」+ 支付对账验证

主要文件：改 `snowmeet_ai_doc/skills/export_rent_order_fiscal_year/export_rent_orders_fy.py` + 同目录 `SKILL.md`；新建只读 `snowmeet_ai_doc/verify_payment_reconcile.py`；重生成 `wanlong_rent_orders_fy_2025-05-01_2026-04-30.xlsx`。**plan**：`/Users/cangjie/.claude/plans/sheet-openid-adaptive-teacup.md`。

#### 一、支付对账验证（接 5-16 待办）
- `verify_payment_reconcile.py` 只读两 sheet 按订单号汇总，`最终金额 = 支付 − 退款 − 分账`
- 「仅成功分账」口径 2079 单逐单零差异 ✓；「全部分账」差 ¥14,045.38 = 64 单失败/作废分账（支付流水只收 success=1，设计预期非 bug）

#### 二、「年度租赁」新增「店员openid」列（紧邻「店员姓名」右）
- 路径 `order.staff_id → staff_social_account → social_account_for_job.wechat_mini_openid`
- 口径 3 轮收敛：①窗口+`ssa.valid=1`（13 空）→ ②**去 valid**（4 空，离职店员旧账号 valid=0 仍要还原历史归集）→ ③**两级偏好**（窗口覆盖 biz_date 优先，否则回退该店员 start_date DESC 最近曾用账号）→ **0 空 / 2319 行全覆盖**
- 实现：仿 `msa_cell`/`big_pay` 的 `OUTER APPLY ... staff_oid`，`ORDER BY CASE WHEN 窗口命中 THEN 0 ELSE 1 END, start_date DESC, id DESC`，TOP 1 防行数翻倍

#### 三、「union id」→「顾客openid」（列名 + 数据源都换）
- 原 `member_social_account[type=wechat_unionid]` → `type=wechat_mini_openid`（小程序 openid，与店员openid 同类型），alias `msa_uid → msa_oid`
- 非空 2274/2319（98.1%）；其余空 = 该会员无 wechat_mini_openid 记录（纯线下/未授权小程序顾客）

#### 四、关键发现
- **本机 Intel Mac ODBC**：CLAUDE.md「已知遗留」那条 `/opt/homebrew/etc`+Driver18 是 Apple Silicon 同步机；本机需 `ODBCSYSINI=/usr/local/Cellar/unixodbc/2.3.4/etc` + `--conn` 覆盖 Driver 13
- **财年脚本整本重建 xlsx**：每次重跑后必须紧接着重跑 `add_payment_detail_sheet_to_fy_xlsx.py`，否则「支付明细/支付流水」两 sheet 丢失（曾因 cd 到 skill 子目录导致第二脚本路径失败、漏掉两 sheet）
- **staff_social_account.valid 语义**：离职/换号后旧记录置 valid=0；历史报表归集**不能过滤 valid**，否则离职店员经手的历史订单 openid 丢失
- 行数 2319（非 5-16 记的 2428）：当前生产数据已变（脚本走自身规范过滤口径，正常）

#### 五、待验证/可选
- 那 ~45 个无 顾客openid 的订单是否需关注（多为未授权小程序顾客，预期空）
- 是否需把「店员openid」两级偏好回退口径同样应用到其它导出脚本

### 2026-05-17（续） — 崇礼/南山多店财年导出 + git push 工作流修正

主要文件：新建 `snowmeet_ai_doc/chongli_rent_orders_fy_2025-05-01_2026-04-30.xlsx` + `nanshan_rent_orders_fy_2025-05-01_2026-04-30.xlsx`（纯用现有脚本跑，无代码改动）。

#### 一、多店财年导出（同款规则，无分账店铺）
- 用 `export_rent_orders_fy.py --shop X` + `add_payment_detail_sheet_to_fy_xlsx.py --xlsx X` 两脚本跑崇礼/南山
- 崇礼旗舰店：年度租赁 63列×192行 / 支付明细 18列×184行 / 支付流水 8列×355行（支付184+退款171）
- 南山（`order.shop='南山'`）：年度租赁 54列×232行 / 支付明细 14列×231行 / 支付流水 8列×462行
- **无分账店铺自适应**：DB 核实两店 order_share=0 / payment_share=0；支付明细 maxShare=0→无分账列、支付流水无分账行（数据驱动自动省略）；但「年度租赁」的 3 个**固定**列 应/实/待分账金额仍在，值全 0（同款规则保留结构）
- 动态支付/退款区列数按各店实际最大笔数：万龙 maxPay/Ref=6、崇礼=2、南山=1 → 故列总数 99/63/54 不同，属正常

#### 二、git push 工作流根因 + 修正（用户两次强调）
- 用户问「为什么没执行 git push」。排查：自动 push 是 5-14 配的 **Stop hook**，写在 `.claude/settings.local.json`——该文件 **gitignored、机器本地、不跨机同步**；且 hook 命令硬编码路径是另一台机的 `/Users/cangjie/source/...`，本机是 `/Users/cangjie/Projects/...`。**本机 settings.local.json（5-10 版）根本无 hooks 段** → 这台机 end-work 不会自动 push
- 用户明确："每次 end-work 之后 snowmeet_ai_doc 整理出来的所有文件和上下文都要全部提交到 GitHub，下次未必用这台电脑"
- 修正：**git commit+push 改为 end-work 的固定收尾动作，由我主动做（不依赖机器本地 hook）**，已写进 auto-memory feedback。不能靠 hook（gitignored 不跨机），记忆跨会话/跨机持久才可靠

#### 三、待验证/可选
- 其余店铺（渔阳/怀北/万龙服务中心）按需同法导出
- 是否把无分账店铺「年度租赁」的 3 个空分账固定列也去掉（需脚本支持按店自适应；目前保留=同款规则）

### 2026-05-17（续2） — start-work 内置 git pull + Stop hook 收紧

主要文件：改 `snowmeet_ai_doc/.claude/skills/start-work/SKILL.md`（入库，已随 `e899295` 推送）+ 仓库根 `.claude/settings.local.json` 的 Stop hook（gitignored/机器本地，不入库）。plan：`/Users/cangjie/.claude/plans/start-work-synthetic-comet.md`。

#### 一、根因：start-work 加载到过期上下文
- 会话起始本地 HEAD=`ffbb27e`，缓存 origin/main=`ffbb27e`，`git ls-remote` 查真实远端=`dbaa546` → 连 fetch 都没发生，start-work 读了旧 CLAUDE.md
- 同步本应由 `.claude/settings.local.json` 的 `PreToolUse/Skill(start-work)` hook（`git pull --ff-only`）做；本会话该 hook **未执行**（最可疑：用了非标准 `"if"` 键 + `|| echo warn` 吞错不阻断）

#### 二、修正 1：git pull 写进 SKILL.md 第 1 步（用户指定）
- `## Process` 新增第 1 步 `git -C snowmeet_ai_doc pull --ff-only`，原 Read/Present/Format 顺延为 2/3/4；失败显式告警「⚠️ 同步失败」不静默
- 理由：skill 入库、跨机生效；不再依赖 gitignored/机器本地、且实测不可靠的 hook

#### 三、修正 2：Stop hook 收紧（用户要求）
- 旧 Stop hook：sessions/*.md 近 3 分钟有改动就 `git add .` 全量 commit+push → 把本会话一个有意的 SKILL.md 改动用 `auto: end-work session archive` 自动推到了共享远端（即 `e899295`）
- 收紧为：`git add -- sessions CLAUDE.md`（仅归档产物）；仅这两路径有改动才 commit；**仅 commit 成功后**才 push；其余改动 `git status --porcelain` 列出提示「留待手动处理」
- 隔离临时仓库实跑两场景通过：A 三类改动同改→只提交 sessions+CLAUDE.md、无关文件留工作区；B 仅无关文件改→无 commit、origin 未 push

#### 四、关键发现 / 教训
- `git status` 的「up to date with origin/main」比的是**本地缓存的 origin/main**，未 fetch 时谎报；真实远端用 `git ls-remote origin refs/heads/main`（只读）
- 本会话 SKILL.md 改动「看不到」是因 Stop hook 已 commit+push（HEAD=`e899295` 已含），非未改；提交链线性无分叉，`dbaa546` 那批未同步工作也已并入
- `.claude/settings.local.json` gitignored/机器本地/不跨机；start-work 的 pull、end-work 的 push 可靠性必须落在「入库 skill 步骤 + 跨会话记忆」，hook 仅本机冗余
