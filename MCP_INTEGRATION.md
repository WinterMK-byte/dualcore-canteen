# MCP 接入说明

## 1. MCP Server

| 项 | 值 |
|---|---|
| 服务 | 麦当劳中国 MCP Server |
| 地址 | `https://mcp.mcd.cn` |
| 开放平台 | <https://open.mcd.cn/mcp> |
| 传输 | Streamable HTTP |
| 认证 | `Authorization: Bearer <MCP_TOKEN>` |
| 协议版本 | `2025-03-26` |
| 限流 / 错误码 | 600 次/分钟；`401` Token 无效、`429` 触发限流 |

客户端：`tools/mcd_mcp.py`，纯 Python 标准库实现，包含 `initialize` → `notifications/initialized` → `tools/list` → `tools/call` 四环节，同时兼容 `application/json` 与 `text/event-stream` 两种响应格式。

## 2. 实际使用的 Tool

| Tool | 用途 | 在游戏里的作用 | 必须 |
|---|---|---|---|
| `list-nutrition-foods` | 取餐品的能量、蛋白质、脂肪、碳水、钠 | **直接构成全部游戏数值**：ATK/DEF/HP/DEBUFF 都由它算出来 | 是 |
| `query-my-coupons` | 取用户手上的券 | 补给台的「折扣」展示 | 否 |
| `available-coupons` | 取可用券 | 上者为空时的备选源 | 否 |

## 3. 调用流程

```
sync_nutrition.py
   ├─ initialize ─────────────────────────▶ serverInfo + Mcp-Session-Id
   ├─ tools/call list-nutrition-foods ────▶ 餐品营养原始数据
   │     └─ 落盘 data/raw_nutrition.json
   │     └─ normalize_item() 多别名容错解析
   ├─ tools/call query-my-coupons ────────▶ 券列表（失败则跳过）
   └─ 写入 dualcore_canteen.html 的 <script id="nutrition-data">

打开 HTML
   └─ 解析内嵌 JSON → DISHES[]
        └─ 每项 : { kcal, pro, fat, carb, na, price }
             ├─ 补给台显示「热量 / 蛋 / 脂 / 碳 / 钠」
             ├─ 关卡判定：热量区间、蛋白质下限、脂肪/钠上限、预算
             └─ derive() 换算：
                   ATK  = pro × 2   （钠 > 2000 时 × 0.8）
                   DEF  = fat × 1.5
                   HP   = carb × 2
                   DEBUFF = na > 2000 ? 口渴 : 无
```

### 为什么营养数据要「内嵌」而不是每次请求

游戏必须能在断网、无 Token、`file://` 双击三种情况下都能玩。营养数据是**关卡设计的一部分**（三关的数值区间是按真实餐品的营养分布调的），把它做成运行时请求会让「没网就玩不了」，直接摧毁可体验性。所以采用「脚本刷新 + 内嵌快照」，一次同步，长期可玩。

## 4. 字段映射与容错

| 内部字段 | 探测的字段别名 |
|---|---|
| 名称 | `name` / `productName` / `mealName` / `title` / `cnName` / `餐品名称` / `名称` |
| 热量 | `energy` / `energyKcal` / `kcal` / `calorie` / `calories` / `热量` / `能量` |
| 蛋白质 | `protein` / `proteinG` / `蛋白质` |
| 脂肪 | `fat` / `fatG` / `脂肪` |
| 碳水 | `carbohydrate` / `carb` / `carbG` / `碳水` / `碳水化合物` |
| 钠 | `sodium` / `sodiumMg` / `钠` |
| 价格 | `price` / `priceCny` / `salePrice` / `价格` |

数值字段若是 `"550 kcal"` 这类带单位字符串，用正则取数字。响应若是被 markdown 围栏包裹的 JSON 串，先剥围栏再解析。任何一步失败都退回离线快照，并在界面**显式标注**数据来源，不用演示值冒充实时值。

## 5. 业务价值

| 价值 | 说明 |
|---|---|
| **把营养数据从「查询结果」变成「决策变量」** | 用户不是看一眼热量就关掉，而是**必须理解营养结构才能过关**——这是最有效的营养教育方式 |
| **让学习和吃饭形成闭环** | 答题换补给、配餐换属性，两者互相驱动，用户停留时间远超单一工具 |
| **零门槛可体验** | 双击即玩，不需要 Token、不需要联网、不需要安装。评委与路人 5 秒内就能理解这是什么 |
| **数据真实** | 关卡数值按真实餐品分布设计，配餐结论对现实生活有参考价值 |

## 6. 自行验证

```bash
python tools/mcd_mcp.py check            # 测连通性
python tools/sync_nutrition.py --dry     # 看会拉到什么
python tools/sync_nutrition.py           # 真拉并写入
python tools/sync_nutrition.py --offline # 恢复离线快照
```

## 7. 合规

- Token 只从环境变量 `MCD_MCP_TOKEN` 读取，不硬编码、不进版本库；`mcp-config.example.json` 全部使用占位符。
- 营养换算是游戏化设定，**不是营养学模型**，不构成任何健康建议。
- 餐品、价格与供应状态以麦当劳官方渠道实时结果为准。
- 本项目为参赛作品，非麦当劳官方产品。
