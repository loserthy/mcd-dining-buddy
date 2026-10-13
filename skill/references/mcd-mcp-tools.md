# 麦当劳 MCP 工具契约与调用规范

> 本文件是「麦麦吃饭小搭子」与外部工具之间的合同。动手调用前先读这一节，
> 不要在没读的情况下凭记忆猜参数或字段。
> 以下契约于 2026-10-10 对照 `https://mcp.mcd.cn` 真实 `tools/list` 校准过。

## 一、前置检查：工具到底连上没有

实际工具名可能带命名空间前缀（如 `mcp__mcd-mcp__query-meals`），
用工具搜索按关键字 `meals` / `coupon` / `nutrition` / `calculate` 找。

**核心五件套（四个场景都要用）：**

| 逻辑名 | 门店依赖 | 用途 |
|---|---|---|
| `query-meals` | ⚠️ 需 storeCode | 查菜单（餐品列表） |
| `list-nutrition-foods` | 无需 | 全量营养数据（热量/蛋白质/脂肪/碳水/钠），无入参 |
| `available-coupons` | 无需 | 「麦麦省」当前可领的券列表 |
| `query-my-coupons` | 无需 | 用户卡包已有券（page/pageSize 可选，最大 5 页×200 条） |
| `calculate-price` | ⚠️ 需 storeCode | 算价（唯一合法价格来源） |

**门店与流程类（按需用）：**

| 逻辑名 | 用途 |
|---|---|
| `query-nearby-stores` | 到店找门店（必填 `beType`=1 到店自取 或 5 得来速、`searchType`=1 收藏店 / 2 按位置） |
| `delivery-query-addresses` | 外送：查用户配送地址列表 |
| `delivery-query-stores` | 外送：按地址 ID 查可配送门店（必填 beType=2 麦乐送 / 6 团餐），拿 storeCode + beCode |
| `query-store-coupons` | 查「指定门店+订单类型」下可用的券（需 storeCode） |
| `query-meal-detail` | 餐品详情 |
| `create-order` | 下单（高风险，见第五节） |
| `query-order` / `order-list` / `cancel-order` | 订单查询 / 历史 / 取消 |
| `now-time-info` | 服务器当前时间（可替代 shell `date` 做时段问候判定） |
| `query-my-account` | 积分余额 |

其余（积分商城 mall-*、派对 party-*、抽奖 lottery、问卷券 survey 等）本技能默认不用，
用户主动问起再查 schema。

**找不到工具时，不要假装调用，也不要凭记忆给数字。** 走第四节的降级流程。

## 二、⭐ 先选店，再点餐（最容易踩的坑）

`query-meals` 和 `calculate-price` 的必填参数是 **`storeCode` + `orderType` + `beType`**
三者缺一不可——不存在"不传门店直接拿全菜单"的用法。

所以每个点餐场景的**第 0 步**永远是先定用餐方式、拿到门店：

| 用餐方式 | beType | orderType | 拿 storeCode 的方式 | beCode |
|---|---|---|---|---|
| 到店自取 | 1 | 1 | `query-nearby-stores(beType=1, searchType=…)` | 不传 |
| 得来速 | 5 | 1 | `query-nearby-stores(beType=5, …)` | 必传 |
| 麦乐送 | 2 | 2 | `delivery-query-addresses` → `delivery-query-stores` | 必传 |
| 企业团餐 | 6 | 2 | 同外送 | 必传（另需 gmServiceCode） |

**交互上怎么做：** 用户一进入点餐流程，先问一句
"今天是到店吃还是外送呀？到店的话告诉我你在哪个城市 / 想去哪家店 🏪"。
用户给了城市或关键字就调 `query-nearby-stores`，把候选门店（店名 + 距离/地址）
列给用户选；选定后把 storeCode 记住，后续查菜单、算价、下单全部复用它。

**beCode 从哪来：** `query-nearby-stores` / `delivery-query-stores` 的返回结果里找。
到店自取（beType=1）不传 beCode，其余场景必传。

**门店检索的两个坑：**

- `searchType=1` 搜的是用户的**收藏门店**。如果用户没收藏过，会返回空 / 报错，
  这时改用 `searchType=2` 并按位置搜索——**此时 `city` 和 `keyword` 都必填**，
  所以要问用户"你在哪个城市、大概哪个位置（或想吃哪家店）"。
- 外送是两步：先 `delivery-query-addresses` 拿 `addressId`（用户可能存了多个地址，
  让他选一个；没有地址就用 `delivery-create-address` 建），
  再 `delivery-query-stores(addressId, beType)` 拿可配送门店。

## 三、各工具调用规范

### 1. `query-meals` — 菜单

- **参数**：`storeCode`、`orderType`（1 到店 / 2 外送）、`beType`，预约场景另传
  `reservationDate`（格式 `yyyy-MM-dd HH:mm`）。
- **拿到后**：按分类归拢，餐品名与价格原样保留，不要改写名称、不要四舍五入价格。
- 细节拿不准时用 `query-meal-detail` 补。

### 2. `list-nutrition-foods` — 营养数据

- **无入参**，直接调，返回常见餐品的能量、蛋白质、脂肪、碳水、钠、钙。
- **坑**：能量单位若是千焦（kJ），必须换算：`1 kcal ≈ 4.184 kJ`。
  展示前统一成"大卡"，别把千焦当大卡写出去，会虚高四倍多。
- **生活参照换算（展示时必做）**：**1 碗米饭（约 200g 熟重）≈ 230 kcal**，
  全技能统一这个基准，不许每次乱换。表格热量列写成 `522 kcal ≈ 2碗饭`
  （碗数 = 热量 ÷ 230，四舍五入到 0.5）。
  没有参照的热量数字，对不健身的人就是一串乱码。
  运动参照（可选，放点评里，别每条都带）：慢跑约 10 kcal/分钟。
- **用途**：场景一的筛选与表格热量列。

### 3. 券：三个工具，三层含义

| 工具 | 含义 | 需要门店？ |
|---|---|---|
| `available-coupons` | 麦麦省可领的券（领了才能用） | 否 |
| `query-my-coupons` | 卡包里已有的券资产 | 否 |
| `query-store-coupons` | 这家店这个订单类型下**真正能用**的券 | 是 |

场景二的正确顺序：`available-coupons` + `query-my-coupons` 并行拉全量，
组合初定后用 `query-store-coupons(storeCode, orderType, beCode?)` 验证该店能不能用；
**能用的券才算进方案**，只是"可以领"的券要在卡片里注明"需先去领"。

用户答应去领券时，调 `auto-bind-coupons` 一键领取麦麦省全部可领券（无需传券 ID），
领完再重新跑一次 `query-my-coupons` / `query-store-coupons` 刷新可用范围。

`query-meals` 返回里可能带**随单购**商品（`withOrder`），价格更划算，
可以在场景二里顺口提一句"这个加购只要 X 元，要不要带上"。

### 4. `calculate-price` — 价格（红线工具）

- **必填**：`storeCode`、`orderType`、`beType`，外送/得来速/团餐另加 `beCode`。
- **⚠️ 单位陷阱（最容易翻车）**：返回的价格字段单位是 **「分」**，
  必须 ÷100 才是「元」。写卡片时统一转好，别把 `2800` 当成 2800 元写出去。
- **⚠️ 到店必取 takeWayCode**：`orderType=1` 时返回 `data.takeWayList[]`，
  里面的 `code` 是取餐方式（堂食/外带等），下单时必须原样回传，
  且可以顺便让用户选一下（"堂食还是带走呀？"）。
- **items 结构**（每个商品一项）：
  - 商品与数量字段以工具 schema 为准（productId/code + 数量）；
  - `couponCode` + `couponId`：**券是挂在单个商品上**的，不是整单一张；
  - `modification`：商品特制选项（去酱、加料等，影响价格）。
- **调用时机**：生成方案卡片前（逐方案算）；用户替换餐品后重算；
  自由点单每次增删改后重算；下单前终算一次，以这一次为准。
- **禁止**：自己加总、自己减券、自己算折扣。哪怕组合再简单也不行。
- **禁果（错误示例）**：
  - ❌ 巨无霸 25 + 薯条 14 = 39 元
  - ✅ 调 `calculate-price`，用返回值 39.0 元

### 5. `query-meal-detail` — 特调（「去酱」怎么实现）

场景一 "去酱 / 换饮料 / 加料" 这类需求**不是改商品名**，而是走特调参数：

1. 先调 `query-meal-detail(storeCode, orderType, beType, code)` 拿 `modification` 结构。
2. 展示规则：`supportModify=true` 的商品 / choice 名字后面标【可特调】；
   **不要主动展开特调选项列表**，用户问了再展示。
3. 回传规则（很容易错）：选中项用 `selectedKey` 作 key；
   未选中但该项有 `unselectedKey` 的，**也必须一起传**，key 用 `unselectedKey`。
   即：含 `unselectedKey` 的特调组要整组传全。
4. 改完特调一定要重新 `calculate-price`，特调会改价。

### 6. 团餐场景的特殊规则（beType=6）

- `calculate-price` 之前必须先调 `query-meal-assistance` 拿 `gmServiceCode`，
  再传给 `calculate-price` / `create-order`。
- 另有 `query-promotions` 查该店团餐促销（满减/满折），别漏了。
- 团餐预算搭配的官方建议（只用来**校验搭配是否合理**，不是要达成的目标）：
  20 元以下 → 小食；20–30 元 → 汉堡 + 小食或饮料；30–40 元 → 汉堡 + 薯条/小食 + 饮料；
  40–50 元 → 汉堡 + 薯条 + 小食 + 饮料；50 元以上 → 丰富组合，小食尽量不重复。
  ⚠️ **不要把这张表当成"预算内必须配满"的靶子**——它是官方按价位给的结构参考，
  不是"一定要花到某一档"的要求。本技能一律以用户选的骨架为准
  （见 `SKILL.md` 场景二 2.1、2.2），预算只当上限。

## 四、调用顺序（推荐编排）

```
第 0 步（所有点餐场景）：问用餐方式 → 到店 query-nearby-stores / 外送
                          delivery-query-addresses + delivery-query-stores → 拿 storeCode

场景一：list-nutrition-foods → query-meals(门店) → 组合 → calculate-price（逐方案）
场景二：available-coupons ∥ query-my-coupons → query-meals(门店)
        → query-store-coupons 验券 → 组合 → calculate-price（带券，逐方案）
场景三：query-meals(门店) → 筛高评价/高复购 → 组合 → calculate-price（逐方案）
场景四：query-meals(门店) → 分类平铺 → 每次增删改后 calculate-price
下单前：calculate-price（终算）→ 用户确认 → create-order → query-order 回执
```

- 场景二的券查询与菜单查询无依赖，可并行调用，省时间。
- 多个方案的 `calculate-price` 尽量合并或连续发出，不要一个方案等一轮。

## 五、下单章节（高风险，逐条遵守）

只有在用户**明确确认**后才执行（"确认""就这个了""下单吧"这类明确表态算，
"随便""都行""你看着办"不算）。

1. **先终算**：调 `calculate-price` 拿最终金额。
2. **再复述**：用卡片贴出清单 + 份数 + 券 + 应付金额 + 用餐方式（到店/外送）
   + 门店名，请求确认。
   —— 如果这是确认前的那一次，就不要调下单工具。
3. **后下单**：用户确认后调 `create-order`，必填同样是
   `storeCode`、`orderType`、`beType`，另外：
   - 到店 / 得来速（`orderType=1`）→ **必须传 `takeWayCode`**，
     值取自上一步 `calculate-price` 返回的 `data.takeWayList[].code`；
   - 外送 / 团餐（`orderType=2`）→ 传 `addressId`（+ 团餐 `gmServiceCode`），
     可引导用户填 `remark` 备注（如"无接触配送"，最多 50 字）；
   - items 结构与 `calculate-price` 完全一致（券挂在商品上）。
   下单前必须核对：餐品、数量、门店、取餐方式、金额。
4. **引导支付**：下单成功后，告诉用户
   “扫码支付链接或打开麦当劳 APP 完成支付，付完回我一句『支付完成』就行～”，
   用户回复后再调 `query-order` 查最新状态。
5. **回执**：拿到订单状态后用回执卡片呈现订单号 / 取餐码 / 金额，附庆祝式治愈短句。
6. **改主意 / 失败**：用户想撤单用 `cancel-order`（仅非商城订单）；
   下单失败如实转达原因，不要重试超过一次，不要替用户改单。

> ⚠️ 下单是不可逆的外部动作。宁多问一句，也不要下错一单。

## 六、降级流程（工具不可用时）

工具没连上或调用失败时，**如实告知**，然后给出可选项，绝不编数据：

1. **报错类失败**（超时、参数错、权限错）：把失败原因用一句人话说清楚，
   例如"优惠券那边好像卡住了，我先不带券给你配，成吗？"然后按能用的工具继续。
2. **401 / 鉴权失效**：提示用户"麦当劳的钥匙好像过期了"，去 MCP 服务管理面板
   重新连接认证；不要用旧缓存数据硬撑。
3. **完全不支持**（找不到任何麦当劳工具）：
   - 直说："我这边的麦当劳菜单还没接上，价格我不敢乱报 🙈"
   - 可以做的：只聊口味偏好、给搭配思路（明确标注「仅供参考，不是官方菜单价」）、
     把组合清单写成待用户自行核对的清单。
   - **不可以做的**：给出具体金额、假装查到了券、声称能下单。
4. 降级时也要保留手账风排版与治愈句，不要让语气突然变冷漠。

## 七、字段缺失与容错

- 某个方案拿不到营养数据 → 该方案卡片省掉热量行，不要写"约 0 大卡"。
- 拿不到券 → 场景二改为"无券最优组合"，并在文案里说明。
- 价格字段是区间（如"¥19.0–¥21.0"）→ 原样展示，别自己取中位数。
- 门店列表为空（城市没匹配到）→ 请用户换个说法或给定位，别硬选一家。
