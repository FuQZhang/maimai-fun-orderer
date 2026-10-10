# 统一下单流程 · mcd-mcp 工具调用指南

所有人格皮肤选品完成后，都走这套流程：**确认门店 → 领券 → 算价 → 二次确认 → 下单**。

## 0. 关键参数总表

**beType（业务类型）与 orderType（订单类型）配对规则：**

| 场景 | beType | orderType | 是否传 beCode | 取号/地址 |
|------|--------|-----------|---------------|-----------|
| 到店自取 | 1 | 1 | ❌ 不传 | 需 `takeWayCode` |
| 得来速 DT | 5 | 1 | ✅ 必传（query-nearby-stores 获取） | 需 `takeWayCode` |
| 麦乐送 | 2 | 2 | ✅ 必传（delivery-query-stores 获取） | 需 `addressId` |
| 企业团餐 | 6 | 2 | ✅ 必传（delivery-query-stores 获取） | 需 `addressId` + `gmServiceCode` |

> ⚠️ 到店自取(beType=1) 传 beCode 会报错；`storeCode` 必须为纯数字。
> ⚠️ 价格单位一律为"**分**"，对外展示需 `/100` 转"元"。
> ⚠️ 预约场景才传 `reservationDate`，格式 `yyyy-MM-dd HH:mm`；即时单不传。

## 1. 确认门店

**问清场景**：到店自取 / 得来速 / 麦乐送 / 团餐。

**到店自取 / 得来速：**
先调 `now-time-info` 拿当前时间备预约用，然后：
- `query-nearby-stores(beType=1|5, searchType=1)` —— 收藏门店；无收藏则用 `searchType=2` 并传 `city` + `keyword`。
- 返回后向用户展示门店名与 `storeCode`；得来速额外展示 `beCode`。
- 让用户选定 → 记录 `storeCode`（得来速再记 `beCode`）。

**麦乐送 / 团餐：**
- `delivery-query-addresses()` —— 拿用户地址列表，选定 `addressId`。
- `delivery-query-stores(beType=2|6, addressId)` —— 拿该地址可配送门店的 `storeCode` + `beCode`。
- 团餐还需问预算 / 人数，并可 `query-promotions` 看满减满折。

## 2. 领券

- `available-coupons()` —— 查当前**可领取**的券。
- `auto-bind-coupons()` —— 一键领取麦麦省全部可领券。
- `query-store-coupons(storeCode, orderType, beType[, beCode])` —— 查**当前门店+订单类型可用**的券（下单前挑券用这个）。
- `query-my-coupons(page, pageSize)` —— 查用户卡包里的券（仅展示，不校验门店可用性）。

选券要点：优先匹配用户已点餐品的券；把券的 `couponId`/`couponCode` 带到 `calculate-price` 里试算，对比优惠力度后再定。

## 3. 查餐品 / 拿 productCode

- `query-meals(storeCode, orderType, beType[, beCode])` —— 拿真实餐品与 `productCode`（选品必须以此为准，别编造商品）。
- `query-meal-detail(storeCode, orderType, beType, code[, beCode])` —— 看单品组成、默认搭配、是否可特调（`supportModify`）。
  - 特调规则：选中的特调项 `key=selectedKey`；未选中但 `unselectedKey` 非空的也必须传入 `key=unselectedKey`。

## 4. 算价

`calculate-price(storeCode, orderType, beType, items[], [beCode], [needTableware], [reservationDate], [withOrder])`

- `items[]`：`{ productCode, quantity, [couponId], [couponCode], [modification], [roundList] }`
- 返回：各商品价、优惠、总价（**分**），以及 `takeWayList[]`（到店场景的取餐方式列表）。
- 到店/得来速下单要用的 `takeWayCode` 就来自 `data.takeWayList[].code`。

## 5. 二次确认（强制）

把以下信息**完整复述**给用户，并等明确同意：
- 门店名 + 取餐/配送方式
- 每一项餐品与数量
- 用了哪张券 / 优惠多少
- **最终应付价（元，= 总价分 / 100）**

用户说"确认/下单/可以"才继续。

## 6. 下单

`create-order(storeCode, orderType, beType, items[], [takeWayCode], [addressId], [beCode], [remark], [needTableware], [reservationDate], [gmServiceCode])`

- orderType=1（到店/得来速）：必传 `takeWayCode`（来自算价的 takeWayList）。
- orderType=2（外送/团餐）：必传 `addressId`；外送可引导填 `remark`（≤50 字，如"无接触配送"）。
- 下单成功后引导用户：扫码或打开麦当劳 APP 完成支付。

## 7. 支付后 / 查单

- 用户回复"支付完成/支付失败"时，用 `query-order(orderId)` 查最新状态并反馈。
- 其他可查：`query-my-account()`（积分）、`query-my-coupons()`（卡包券）。

## 常见坑

- beType=1 误传 beCode → 报错。
- 忘了把价格分转元 → 展示错误。
- 到店下单漏 `takeWayCode` / 外送下单漏 `addressId` → 报错。
- 用 `query-my-coupons` 的券直接下单但门店不适用 → 应改用 `query-store-coupons` 校验。
- 未做二次确认就 create-order → 违反铁律。
