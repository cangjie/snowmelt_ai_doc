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

## 当前状态（截至 2026-05-12 下午）

**已可走通**：录入订单 → 选店 → 进入租赁开单 → 添加套餐（按品类筛选 + 万龙系店铺默认「立即租赁」+ 雪服/护具等非编码品类默认勾选「无编码」+ 创建时 startTime 默认当前时分）→ 购物车展示（rental 折叠态紧凑单行；展开态两层标题 + 跑马灯；rental 级 + rentItem 级双层完整性 chip；不完整时套餐名变红）→ 卡片展开编辑详情（套餐备注 + 起租日期 van-calendar 弹窗 + 今/明高亮快捷按钮 + 起租时间 picker；选租赁模式自动联动起租日期/时间：立即/先租后取=今天+当前时分、延时=明天+00:00；无编码/不需要 disabled 联动 + 不需要时整卡灰显）→ 装备编码录入（点编码区开搜索 modal，按品类模糊搜索租赁物，单选确认后回填 code/name/category_id/rent_product_id/class_name + 重复编码校验；扫码仍然可用）→ 押金/租金点击 tap 弹 `wx.showModal` 二次确认编辑（押金净额显示 = `realGuaranty − guaranty_discount`，下方购物车栏「押金 ¥净额 已减免 -¥xxx」）→ 套餐选模式时未自选 item 跟随 + 内部模式不一致显示 ⚠ → 左划删除 → 底部 4 个快捷入口横向紧凑按钮 + 单行结算条（件数徽章 + 押金 + 已减免 + 租金 + 去结算按钮，全部 rental 完整才允许点击）→ 点「去结算」先 await `saveRentReceptOrder` 落盘最新编辑、再调 `Order/PlaceRentOrder/{id}` 让服务端 `GenerateOrderCode` 生成 `WL_ZL_yyMMdd_xxxxx` 正式订单号 + `valid=1` + 写 Guaranty，返回的 order 回填 `this.data.order` → 跳 `/pages/payment/settle/index?orderId=...` → 结算页订单卡显示 `order.code || order.id` + 三选一支付方式（微信扫码 / 支付宝 mock / 其他确认收款）→ **顾客扫支付二维码进入 `pages/order/payment_entry`：轻量化纯 CSS 卡片版（订单信息 / 租赁内容折叠 / 金额 / 微信支付按钮），租赁明细只列 编码/名称/品类，押金 + 日租金同行各 300rpx 列宽** → 小程序客户端所有 `wx.request` 的 `POST` 请求在全局请求层统一对 payload 内 URL 编码中文执行 `urldecode`（含嵌套对象/数组）。每次结构变更/字段失焦自动 `Rent/SaveRentRecept` 同步后端，起租日期/时间通过 `start_date` (ISO datetime) 真持久化。

**关键文件**
- 页面：`pages/admin/reception/recept_entry`、`recept_new`、`recept_package`、`pages/order/payment_entry`（顾客扫码支付落地页）
- 组件：`components/reception/rent_recept_form`（购物车 + 详情卡片 + 日历 modal + 编码搜索 modal）、`components/reception/search_product_fuzzy`（编码搜索弹窗，可复用）、`components/order-summary-card` + `components/order-payment`（结算页订单卡 + 二维码组件）
- 数据接口（已对接）：`Order/GetShops`、`Rent/GetRentPackageList`、`Rent/GetRentPackage/{id}`、`Rent/GetRentPriceList`、`Rent/SaveRentRecept`、`Order/GetShopByName`、`Rent/GetRentProductFuzzy`、`Rent/GetTopRentCategories`、`Rent/GetSubRentCategories/{id}`、`Rent/GetRentCategory/{id}`、`Order/GetOrderFromPaymentByCustomer/{paymentId}`、`Order/WechatPayByOrderPayment/{paymentId}`

**下一步要做的**
- ✅ 第五步：支付结算页 mvp 完成（settle/index + order-summary-card + order-payment，微信支付走通、支付宝 mock、其他方式确认收款）
- ✅ 顾客扫码支付落地页（`pages/order/payment_entry`）轻量化重做 + 租赁订单友好展示
- payment_entry 其它订单类型友好展示（餐饮 / 零售 / 押金等当前走最小版，留待后续按业务需要扩展）
- 第五步剩余：支付宝小程序对接（替换当前 mock）、支付完成后父页面 `onPaid` 处理（跳转 `rent_details` 或工作台）
- 第二步剩余：扫描条码（`Rent/QueryByBarcode`）入口（目前仅 toast 占位）
- 第二步：去结算按钮入口（已在 `onCheckout` 接通 `Order/PlaceRentOrder` + navigateTo settle）
- 养护 / 零售 业务的接待表单组件（目前仅租赁完成）
- 旧版页面迁移：`recept_auth_list`、`recept_member_info`、`recept_list`、`rent_recepting_list`
- 支付前身份验证（PRD 1.4.5 + 1.4.1）— plan 已审批待开工，见 `snowmeet_ai_doc/payment_identity_verification_plan.md`：后端加 `Order.pay_member_id` + `Order.is_proxy_pay` + 新建 `PaymentIdentityController`；小程序 `payment_entry` 改造 + 新建 `pay-identity-confirm` 组件

**已知遗留**
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
