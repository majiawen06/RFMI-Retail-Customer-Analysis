# 基于 RFM-I 改进模型的线上零售用户分层与转化卡点分析

针对传统 RFM 只看成交结果的缺陷，新增用户意向维度，区分真流失与假性流失、定位转化卡点，对比两套分层结果，输出差异化运营策略

## 一、项目概述
### 1. 研究背景
电商平台用户体量持续增长，运营预算有限，需要精准识别不同价值的用户群体，实现资源倾斜、提升复购与转化。传统用户分层大多依靠 RFM 模型，仅依托订单消费结果进行划分，缺少对用户浏览、加购、停留等前置行为的捕捉，容易造成对用户真实意向误判，进而导致运营策略错位、营销资源浪费。本项目依托仿真电商多表行为数据集，结合用户消费数据与线上互动行为开展用户价值分析。

### 2. 传统 RFM 模型存在的缺陷
传统 RFM 依靠最近消费时间 R、消费频次 F、消费金额 M 划分用户，存在两处明显短板：
① 只看成交结果，无法识别**高意向观望用户**——这类用户频繁浏览、加购，但暂时没有下单，会被模型归类为低价值流失用户，错失转化机会；
② 忽略用户消费能力的边界差异，同等消费金额下，用户购买力天花板不一样，统一的分层标准会给出不符合业务实际的运营指引，难以支撑差异化预算分配。

### 3. 研究目的
针对传统 RFM 的局限，构建改进的 RFM‑I 用户分层模型，新增意向维度 I，量化用户购买意向、转化摩擦程度与活跃状态；区分假性流失用户与真正流失用户，定位用户转化卡点；对比传统 RFM 与 RFM‑I 的分层结果，输出分人群的运营策略，帮助平台把有限营销预算投向转化潜力更高的用户。

### 4. 技术栈
数据清洗、多表关联、指标计算与用户分层：MySQL
指标复核、交叉校验、异常样本检查：Excel
指标可视化、交互式分析看板搭建：PowerBI


## 二、数据说明

### 1. 数据集来源
数据集来自 Kaggle：**E-Commerce Behavior: 5 Relational Tables**，为仿真线上零售业务数据，由 5 张可关联的 CSV 表构成，覆盖用户、商品、会话行为、交易、评价五个业务域。数据时间跨度为 2023-01-01 至 2024-12-30（约两年）。

数据规模：用户 10,000 条、商品 1,000 条、会话行为 80,000 条、交易明细 120,000 条、商品评价 25,000 条，合计约 21.5 万条记录。

### 2. 五张表结构与关联关系

| 表名 | 行数 | 主键 | 核心字段 |
| --- | --- | --- | --- |
| customers（用户表） | 10,000 | customer_id | signup_date 注册日期、age 年龄（18-75，均值 35.3）、gender 性别、country 国家（10 国）、segment 平台既有分层标签、is_churned 是否流失、lifetime_value 生命周期价值、email_opt_in 邮件订阅、has_app 是否装 App |
| sessions（会话行为表） | 80,000 | session_id | customer_id、session_date、device 设备（mobile/desktop/tablet）、channel 渠道（organic/paid_search/social/direct/email/referral）、duration_seconds 停留时长、pages_viewed 浏览页数、converted 是否转化、bounced 是否跳出、cart_additions 加购次数 |
| transactions（交易明细表） | 120,000 | transaction_id | customer_id、product_id、transaction_date、quantity 数量、unit_price 成交单价、total_amount 成交金额、discount_applied 折扣率、status 状态（completed/refunded/cancelled/pending）、payment_method 支付方式、shipping_cost 运费 |
| products（商品表） | 1,000 | product_id | product_name、category 品类（15 类）、brand 品牌、price 标价、avg_rating 平均评分、num_ratings 评价数、stock_quantity 库存、discount_pct 折扣、is_featured 是否主推、weight_kg 重量 |
| reviews（评价表） | 25,000 | review_id | customer_id、product_id、review_date、rating 评分（1-5）、review_text 评价文本、helpful_votes 有用数、verified_purchase 是否真实购买（79.9%） |

关联关系：
- `customers` 是全局唯一用户主键，一对多关联 `sessions`、`transactions`、`reviews`
- `products` 一对多关联 `transactions`、`reviews`
- `transactions` 与 `reviews` 通过 `(customer_id, product_id)` 形成「购买 → 评价」行为链路
- 结构说明：`transactions` 是**商品级明细**（一行一个商品），并非「订单头 + 订单明细」两层结构，因此用户级消费指标需按 `customer_id` 聚合，而不是按订单号聚合

### 3. 分析所需字段说明
- **R 最近消费**：`transactions.transaction_date`，仅取 `status = 'completed'`
- **F 消费频次**：`completed` 状态下按 `customer_id` 去重计数交易笔数
- **M 消费金额**：`SUM(transactions.total_amount)`，仅取 `completed`；不使用 `unit_price × quantity` 重算
- **I 意向分**：`sessions.pages_viewed`、`duration_seconds`、`cart_additions`，聚焦未转化会话的浏览深度
- **I 转化摩擦系数**：`sessions.cart_additions` 与 `converted` 的差额关系，衡量「加购但未成交」的流失程度
- **I 活跃连接度**：`sessions` 会话频次 + `reviews` 评价条数，衡量用户与平台的互动黏性
- **购买力修正**：参考 `customers.segment`（平台既有分层）与 `lifetime_value`，仅用于结果校验，**不参与建模**，避免标签泄漏
- **辅助分析维度**：`products.category`（品类）、`sessions.device` / `sessions.channel`（渠道与设备）

### 4. 数据质量初检结论
- **主键与关联**：5 张表主键均无重复记录，外键完整率 100%，无孤儿数据
- **缺失值**：各表均无空值字段
- **折扣字段与金额不一致**：34,857 条带折扣交易中，98.5% 的 `total_amount` 仍等于 `quantity × unit_price`，未扣减折扣 → 统一以 `total_amount` 为成交金额口径
- **成交单价偏移**：29% 的交易 `unit_price` 低于商品标价，且无高于标价的记录 → 说明 `unit_price` 已含促销价，M 指标以成交价而非标价计算
- **订单状态分布**：completed 57.25%、refunded 14.28%、cancelled 14.24%、pending 14.22% → 仅 completed 计入 RFM，退款与取消不计入消费贡献
- **会话逻辑矛盾**：648 条记录 `bounced = 1` 同时 `converted = 1`；62 条会话时长 > 3600 秒，19 条浏览页数 > 100 → 标记为异常会话并剔除
- **无效用户**：1 个用户仅有注册信息，无任何会话与交易记录

### 5. 数据集自带标签的使用边界（重要说明）
`customers` 表自带两个已完成的标签字段：`lifetime_value`（用户生命周期总价值）与 `is_churned`（平台标记用户是否已流失）。这两个字段**不参与 RFM-I 模型的任何指标计算与分层打分**，仅作为事后校验的参照标准：

- **`lifetime_value` — 校验计算逻辑是否正确**
本项目所有用户的 M（消费金额）均由 `transactions` 交易明细表自行聚合计算得出。计算完成后，将结果与该字段做比对，用于反向核验 SQL 聚合逻辑与过滤条件（如仅取 `completed` 状态）是否准确，确认无遗漏、无重复计入。

- **`is_churned` — 校验改进模型的业务合理性**
该字段是平台依据真实业务判定的流失标签。本项目在 RFM-I 分层完成后，将其作为外部验证口径，检验模型识别出的「真流失」与「假性流失」人群是否与平台真实标签的分布一致，以此佐证改进模型对高意向潜客、假性流失用户的区分能力具备业务可信度。

使用原因：`lifetime_value` 本质是数据集预先聚合完成的消费结果，若直接作为 M 指标输入模型，属于数据泄露，会使模型失去独立计算价值；`is_churned` 属于结果标签，若作为建模输入会直接导致分层结论自我循环。因此两者严格限定在「**只校验、不建模**」的范围内。

## 三、数据预处理（MySQL能力展示）
1. MySQL多表关联查询（用户表+订单表+行为会话表联表）
2. 数据清洗：去重、空值处理、异常订单/异常价格过滤
3. 指标计算（MySQL实现）
   - R / F / M 基础指标
   - I维度：意向分、转化摩擦系数、活跃连接度、购买力潜力
4. NTILE窗口函数对各指标分箱打分
5. CASE WHEN 条件判断，输出RFM-I用户分层标签

## 四、数据校验与二次处理（Excel能力展示）
1. 将MySQL计算后的明细结果导出至Excel
2. 数据透视表核验指标分布、分层占比合理性
3. 异常样本复核、分层结果交叉校验
4. 整理、输出干净的最终分析数据集，供给PowerBI

## 五、模型构建与分析核心（⭐项目最大亮点，面试重点讲这块）
1. RFM-I改进模型构建逻辑
2. I维度4个新增指标业务解释
3. 传统RFM VS RFM-I分层结果对比分析
4. 高意向高摩擦潜客人群特征分析
5. 假性流失用户特征分析
6. 不同购买力层级用户行为差异分析

## 六、可视化看板搭建（PowerBI能力展示）
1. 星型模型搭建、维度表/事实表建立关系
2. DAX度量值编写（RFM、RFM-I核心指标）
3. 交互式看板三大模块
   - 整体业务数据大盘
   - RFM & RFM-I分层对比看板
   - 用户摩擦卡点分析看板
4. 切片器、筛选、钻取交互功能应用

## 七、业务结论与运营策略建议
（对应你笔记：有限预算怎么分配、高意向潜客、消费到天花板用户、假性流失用户的差异化策略）

## 八、项目总结与个人收获

## 📁 仓库文件目录说明

```
`RFMI-Retail-Customer-Analysis`
`├── README.md # 项目介绍（当前文件）
`├── sql/ # MySQL 脚本：清洗、指标计算、分层代码
`├── excel/ # Excel 校验文件、透视表核对表
`├── dashboard/ # PowerBI 看板截图（pbix 源文件可选存放）
`└── report/ # 完整详细分析报告
```
