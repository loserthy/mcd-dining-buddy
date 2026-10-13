# 麦当劳 MCP 接入说明（MCP_INTEGRATION）

> 本文说明「麦麦吃饭小搭子」**实际使用**的麦当劳 MCP Server、Tool、调用流程与业务价值。
> 所有工具名、参数与字段均于 2026-10-10 对照 `https://mcp.mcd.cn` 的真实 `tools/list` 校准。

---

## 一、使用的 MCP Server

| 项 | 值 |
|---|---|
| Server 名称 | `mcd-mcp` |
| 传输方式 | `streamablehttp` |
| 端点 | `https://mcp.mcd.cn` |
| 鉴权 | HTTP Header `Authorization: Bearer <MCP_TOKEN>` |
| Token 获取 | 麦当劳 MCP 控制台申请（见 [mcd-mcp-server](https://github.com/M-China/mcd-mcp-server)） |
| 接入宿主 | 腾讯 WorkBuddy（连接器 → 自定义连接器 → 配置 MCP） |

脱敏配置见仓库根目录 [`mcp-config.example.json`](./mcp-config.example.json)：

```json
{
  "mcpServers": {
    "mcd-mcp": {
      "type": "streamablehttp",
      "url": "https://mcp.mcd.cn",
      "headers": { "Authorization": "Bearer ${MCD_MCP_TOKEN}" }
    }
  }
}
```

> 仓库内不含任何真实 Token，仅使用占位符。

---

## 二、实际调用的 MCP Tools

本 Skill 只使用其中 **12 个**工具（Server 共暴露 30+ 个，其余如积分商城 `mall-*`、
派对 `party-*`、抽奖 `lottery` 等本场景用不到，不纳入调用范围）。

### 2.1 核心五件套（四个场景都用）

| Tool | 是否依赖门店 | 在本项目中的用途 |
|---|---|---|
| `query-meals` | ⚠️ 必传 `storeCode` | 拉取该门店菜单（餐品名 + 价格 + 分类 + 随单购） |
| `list-nutrition-foods` | 否（无入参） | 拉全量营养数据（热量 / 蛋白质 / 脂肪 / 碳水 / 钠 / 钙） |
| `available-coupons` | 否 | 「麦麦省」当前可领的券 |
| `query-my-coupons` | 否 | 用户卡包中已有的券资产 |
| `calculate-price` | ⚠️ 必传 `storeCode` | **唯一合法价格来源**，逐方案算价、终算、特调后重算 |

### 2.2 门店与流程类

| Tool | 用途 |
|---|---|
| `query-nearby-stores` | 到店自取 / 得来速找门店（`beType`=1/5，`searchType`=1 收藏店 / 2 按位置） |
| `delivery-query-addresses` | 外送：查用户配送地址列表 |
| `delivery-query-stores` | 外送：按 `addressId` 查可配送门店，拿 `storeCode` + `beCode` |
| `query-store-coupons` | 校验「这家店 + 这个订单类型」真正可用的券 |
| `query-meal-detail` | 餐品详情与特调结构（去酱 / 加料等 `modification`） |
| `auto-bind-coupons` | 一键领取麦麦省全部可领券 |
| `now-time-info` | 服务器当前时间，用于判断早/午/晚与工作日/周末的问候语 |
| `create-order` | 下单（仅在用户明确确认后调用） |
| `query-order` / `cancel-order` | 订单状态回执 / 撤单 |

---

## 三、调用流程

### 3.1 全局编排

```
第 0 步（所有点餐场景通用）：定用餐方式 + 门店
   到店自取/得来速：query-nearby-stores(beType=1|5, searchType=1|2) → storeCode
   麦乐送：delivery-query-addresses → delivery-query-stores(addressId, beType=2)
          → storeCode + beCode

场景一 · 控卡轻食
   list-nutrition-foods ∥ query-meals(storeCode)
   → 低热量高蛋白筛选与组合 → calculate-price（逐方案）
   → 热量换算"几碗饭"后出表格

场景二 · 月底省钱
   available-coupons ∥ query-my-coupons        （与菜单查询无依赖，并行）
   → query-meals(storeCode)
   → query-store-coupons(storeCode, orderType) 验证该店可用券
   → 在「用户选的搭配骨架 + 预算上限」内求解
   → calculate-price（带券参数，逐方案）

场景三 · 犒劳奢侈
   query-meals(storeCode) → 筛高评价/高复购（安格斯厚牛堡、分享盒等）
   → 组合 → calculate-price（逐方案）

场景四 · 自由点单
   query-meals(storeCode) → 按主食/小食/饮料三张小表平铺
   → 用户每增删改一次，calculate-price 重算一次

下单链路（高风险，需明确确认）
   calculate-price（终算）→ 复述清单请求确认 → create-order → query-order 回执
```

### 3.2 关键参数与单位（实践校准）

| 场景 | `beType` | `orderType` | 备注 |
|---|---|---|---|
| 到店自取 | 1 | 1 | 不传 `beCode`；下单必须传 `takeWayCode`（取自 `calculate-price` 返回的 `data.takeWayList[].code`） |
| 得来速 | 5 | 1 | 必传 `beCode` |
| 麦乐送 | 2 | 2 | 必传 `beCode` + `addressId`，可传 `remark`（≤50 字） |
| 企业团餐 | 6 | 2 | 需先 `query-meal-assistance` 拿 `gmServiceCode` |

**三个已踩过的坑（写进技能文档，避免复现）：**

1. **价格单位是「分」**——`calculate-price` / `create-order` 返回的金额必须 ÷100 才是元，
   否则 ¥28 会写成 ¥2800。
2. **不存在"不传门店直接查全菜单"**——`query-meals` 与 `calculate-price` 的
   `storeCode` + `orderType` + `beType` 三者必填，所以必须先选店。
3. **券是挂在单个商品上**的（`couponCode` + `couponId` 写在 `items` 里，不是整单一张），
   且"可领的券"≠"能用的券"——后者要靠 `query-store-coupons` 按门店验证。

另有单位换算坑：营养数据的能量若为千焦，需 `÷4.184` 换成大卡。

---

## 四、业务价值

### 4.1 对用户：把"决策 + 算账 + 省钱"三件事压缩成两轮对话

| 价值点 | 具体体现 |
|---|---|
| **降低决策成本** | 全流程"回一个字母"；用餐方式与门店一条消息问完；菜单拆三张小表 |
| **降低理解成本** | 热量换算成"几碗饭"，把专业营养数据翻译成生活语言 |
| **真实省钱** | 券按门店验证后再组合，方案给出多档价位而非贴着预算上限 |
| **价格可信** | 所有金额来自 `calculate-price`，无一处手算，杜绝"AI 编价格" |
| **决策辅助** | 每个方案附一句人话点评（蛋白质多少、适不适合独享） |

### 4.2 对麦当劳 MCP 能力的使用深度

本项目不是"调一次菜单 API 就完事"，而是把 MCP 的组合能力用满：

- **多源交叉**：菜单（`query-meals`）× 营养（`list-nutrition-foods`）× 券（三层券工具）× 价格（`calculate-price`）
- **门店维度贯穿**：所有查询与算价都绑定真实门店，价格是可下单的真实价，不是展示价
- **下单闭环**：从选店 → 选品 → 券 → 终算 → `create-order` → `query-order` 回执，端到端跑通
- **交易安全约束**：不可逆动作前置确认，"随便"不算确认，失败不自动重试

### 4.3 可复用的方法论

Skill 中沉淀的通用做法，可迁移到其他 MCP 场景：

1. **先校准再开发**：先拉真实 `tools/list` 核对工具名与入参，避免凭文档假设写流程
2. **单位与命名规范化**：分/元、kJ/kcal 这类单位陷阱统一在契约层解决，不让它泄漏到展示层
3. **交互语言统一**："需要用户回话就给选项"——降低操作成本的最有效手段
4. **降级路径明确**：工具不可用时如实说明，绝不编造数据

---

## 五、合规说明

- 本仓库及 Skill 内**不含真实 Token / 密钥 / 账号凭证**
- 调用动作均为用户主动发起；下单类不可逆操作前置明确确认
- 餐品信息、价格及供应状态以麦当劳官方渠道实时结果为准，项目输出仅供参考，
  不构成医疗、营养或其他专业建议
- 使用麦当劳 MCP 时遵守其服务规则：https://cdn.mcd.cn/cms/pages/MCPServerRules.html
