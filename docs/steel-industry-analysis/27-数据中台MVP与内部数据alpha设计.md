# 数据中台 MVP 与内部数据 alpha 设计

> 本文是 16 / 19 / 26 号文档的实施细化，专门解决"数据中台 MVP 上线"和"内部数据 alpha"两个被前面文档反复强调但未具体落地的命题。
>
> 适用读者：CDO / 数据架构师 / 数据工程师 / 数据产品 PM。
>
> **MVP 范围**：M1-M3 完成数据中台基础底座 + 内部 alpha API；**M4-M6 完成 6 类核心数据采集**；**对外的数据指数产品（CSP/CSC 等）放到 M7+**，不在 MVP 范围内。

---

## 一、数据中台 MVP 的"3 个不"原则

| 不做 | 做 |
|---|---|
| ❌ 不做对外数据产品（C 端/金融机构售卖） | ✅ 内部 alpha：仅供 6 大 SaaS 使用 |
| ❌ 不做完整数据治理（5 大主题域） | ✅ 围绕 6 大 SaaS 必须的数据域优先 |
| ❌ 不做实时大数据集群（Hadoop全套） | ✅ 轻量数据栈（Kafka + Iceberg + ClickHouse） |

> **MVP 阶段的数据中台 = 内部数据基础设施**，不是数据产品公司。

---

## 二、数据中台总体架构

```
┌────────────────────────────────────────────────────────────────┐
│  数据接入层（Ingest Layer）                                    │
│  ─────────────────────────────────────────────────────────── │
│  ① 事件流接入（Kafka） ← 来自所有 SaaS 业务事件                │
│  ② 历史数据接入（CDC） ← 来自老 ERP 数据库                     │
│  ③ 第三方数据接入       ← 钢联 / Mysteel / 企查查              │
│  ④ 用户主动上传         ← Excel/PDF（OCR 处理）                │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│  数据治理层（Governance Layer）                                │
│  ─────────────────────────────────────────────────────────── │
│  ① 实体合并（按 usc_code 统一）                                │
│  ② 数据脱敏（K-匿名 + 敏感字段加密）                            │
│  ③ 数据质量（完整性/准确性/时效性校验）                         │
│  ④ 主数据对齐（钢材 sku/客户/仓库/钢厂）                        │
│  ⑤ 用户授权检查（数据贡献授权状态）                             │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│  数据存储层（Storage Layer）                                   │
│  ─────────────────────────────────────────────────────────── │
│  ① 数据湖（对象存储 + Iceberg） — 原始 + ODS + DWD             │
│  ② OLAP（ClickHouse） — DWS + ADS（应用层指标）                │
│  ③ 缓存（Redis） — 高频查询                                    │
│  ④ 时序库（TimescaleDB） — 价格序列                            │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│  数据计算层（Compute Layer）                                   │
│  ─────────────────────────────────────────────────────────── │
│  ① 实时流计算（Flink） — 行情指数 / 库存浮盈 / 风险预警         │
│  ② 批量计算（Spark / Spark SQL）— 客户画像 / 信用评分           │
│  ③ 算法服务（Python） — AI 报价模型 / AI 解读 / 评分模型        │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│  数据服务层（Serving Layer）                                   │
│  ─────────────────────────────────────────────────────────── │
│  ① 内部 alpha API（喂给 6 大 SaaS）                            │
│  ② 内部分析 BI（运营/老板看板）                                 │
│  ③ 数据导出（脱敏后 Excel/CSV）                                │
│  ④ M7+ 对外 API（数据指数产品，预留）                           │
└────────────────────────────────────────────────────────────────┘
```

---

## 三、6 大数据域（围绕 6 大 SaaS）

### 3.1 数据域 A：行情数据域（Market）

#### 数据来源
- 第三方：钢联 / Mysteel / Wind（外采 API）
- 自有：平台扫码出库的真实成交（核心独家）
- 期货：上期所行情（参考）

#### 关键表
```
ods_market_quote_external      原始外部行情（按品种/区域/日）
ods_real_transaction_internal  原始平台真实成交
dwd_price_unified              统一价格事实表
dws_price_index_daily          每日价格指数
dws_price_index_realtime       实时价格指数（5min 更新）
ads_csp_v0                     CSP V0 版指数（内部 alpha）
```

#### 内部 alpha API
- `GET /internal/market/realtime?sku=xxx&region=xxx`
  - 返回：实时价格 + 7/30 日均价 + 趋势
- `GET /internal/market/forecast?sku=xxx`（M3 后）
  - 返回：AI 价格预测

#### 服务对象
- P1 钢价行情 SaaS（页面展示）
- P4 AI 报价（提供基准价）
- P5 资金大盘（计算库存浮盈）

---

### 3.2 数据域 B：交易数据域（Transaction）

#### 数据来源
- P3 扫码出库事件（核心）
- P4 AI 报价事件
- P2 对账单事件
- 老 ERP 历史数据迁移

#### 关键表
```
ods_quote_event                报价事件
ods_contract_event             合同事件
ods_delivery_event             出库事件（含 bundle_code）
ods_payment_event              收款/付款事件
ods_note_event                 票据事件

dwd_transaction_unified        统一交易事实表
dws_transaction_daily          日聚合
dws_customer_dealing_summary   客户成交汇总
```

#### 内部 alpha API
- `GET /internal/transaction/customer-summary?customer_id=xxx`
  - 返回：客户最近 N 天成交记录、平均价、平均账期
- `POST /internal/transaction/anomaly-check`
  - 返回：是否异常交易（用于 P4 业务员风控）

#### 服务对象
- P4 AI 报价（客户画像）
- P5 资金大盘（应收账款）
- 内部风控

---

### 3.3 数据域 C：库存数据域（Inventory）

#### 数据来源
- P3 扫码出库（出/入/移库事件）
- 月度盘点
- ERP 历史

#### 关键表
```
ods_inventory_event            库存事件流
dwd_inventory_snapshot         库存快照（日级）
dws_inventory_turnover         周转指标
dws_inventory_floating_pl      库存浮盈
ads_csi_v0                     CSI V0 版库存指数（M3 内部 alpha）
```

#### 内部 alpha API
- `GET /internal/inventory/snapshot?tenant_id=xxx&warehouse_id=xxx`
- `GET /internal/inventory/turnover?sku=xxx&region=xxx`
- `GET /internal/inventory/floating-pl?tenant_id=xxx`（实时计算）

#### 服务对象
- P5 资金大盘
- P3 扫码出库（库存差异溯源）
- 老板驾驶舱

---

### 3.4 数据域 D：客户与履约数据域（Customer & Performance）

#### 数据来源
- 客户主数据
- 全平台对该客户的交易/对账/付款记录
- 客诉/异议事件
- 第三方（企查查/失信名单）

#### 关键表
```
ods_customer_master            客户主数据
ods_customer_complaint         客诉事件
ods_court_record               法院公开数据
ods_business_registry          工商数据

dwd_customer_360               客户 360 视图（脱敏后）
dws_customer_payment_score     付款评分
dws_customer_risk_score        风险评分
ads_csc_v0                     CSC V0 版信用指数（M3 内部 alpha）
```

#### 内部 alpha API
- `GET /internal/customer/profile?customer_id=xxx`
  - 返回：360 画像 + 评分 + 风险等级
- `GET /internal/customer/payment-score?customer_id=xxx`

#### 服务对象
- P4 AI 报价（建议价时考虑客户画像）
- P5 资金大盘（应收风险预警）
- 内部风控

#### 关键约束
- **数据脱敏**：单家企业数据非授权方不可识别
- **客户主权**：当前仅为内部使用，不对外
- **撤回机制**：贡献者撤回授权，对应数据 30 天内停用

---

### 3.5 数据域 E：财务与资金数据域（Finance）

#### 数据来源
- P5 资金大盘事件
- P2 对账单事件
- P3 出库（应收）
- 银行接入（M6+）

#### 关键表
```
ods_payment_event              收付款
ods_note_event                 票据
ods_receivable_event           应收账款变动

dwd_capital_unified            资金事实表
dws_aging_analysis             账龄分析
dws_cash_flow_daily            现金流
dws_real_net_asset             真实净资产
ads_cfp_v0                     CFP V0 版资金压力（M3 内部 alpha，行业聚合）
```

#### 内部 alpha API
- `GET /internal/finance/dashboard?tenant_id=xxx`
- `GET /internal/finance/aging-analysis?tenant_id=xxx`
- `GET /internal/finance/forecast?tenant_id=xxx&days=30`

---

### 3.6 数据域 F：钢厂与代理数据域（Mill & Agency）

#### 数据来源
- 钢厂主数据
- 代理任务数据
- 平台真实提货量

#### 关键表
```
ods_mill_master
ods_agency_agreement
ods_mill_pickup_event

dws_mill_pickup_progress       钢厂提货进度
ads_cmp_v0                     CMP V0 版钢厂提货指数（M4 alpha）
```

#### 内部 alpha API
- `GET /internal/mill/pickup-progress?tenant_id=xxx&mill=xxx&period=xxx`
- `GET /internal/mill/agency-task-status?tenant_id=xxx`

---

## 四、4 个 V0 版"内部指数"（M3-M6 上线）

> 这是数据中台 MVP 阶段的"alpha 版数据指数"，**仅在内部使用**（喂给 SaaS / 老板看板 / 内部风控），不对外销售。M7+ 演进为对外 V1 版。

| 指数 | 全称 | M3-M6 状态 | M7+ 演进 |
|---|---|---|---|
| **CSP-V0** | 钢贸真实成交价指数 | 内部 alpha（喂给 P1 / P4） | V1 对外销售 |
| **CSC-V0** | 钢贸商信用指数 | 内部 alpha（喂给 P4 / P5 / 风控） | V1 对外+B2B 服务 |
| **CSI-V0** | 钢材库存周转指数 | 内部 alpha（喂给 P5 老板看板） | V1 对外 |
| **CFP-V0** | 钢贸资金压力指数 | 内部 alpha（脱敏行业聚合） | V1 对外（仅政府）|

### 4.1 CSP-V0 设计（最简版）

```
计算口径：
  对每个 (sku_id, region, date)：
    real_avg_price = SUM(出库金额) / SUM(出库吨位)
    transaction_volume = SUM(出库吨位)
    confidence = MIN(1.0, transaction_count / 10)

发布频率：
  每日 06:00 计算前一日数据
  实时（5 min）滑窗（M5 后）

存储：
  ads_csp_v0 表
  按 (sku_id, region, date) 分区
```

### 4.2 CSC-V0 设计（最简版）

```
评分维度（仅 5 个，避免复杂）：
  1. 付款及时率（占 40%）= 按时还款笔数 / 总笔数
  2. 平均逾期天数（占 25%）
  3. 合作密度（占 15%）= 在多少同行处有交易
  4. 司法风险（占 10%）= 是否被执行/失信
  5. 经营稳定性（占 10%）= 注册年限 + 注销/吊销状态

输出：
  - 总分 0-100
  - 风险等级 A/B/C/D/E
  - 5 个分项明细（用于解释）

约束：
  - 每个评分必须可解释
  - 申诉通道（M6+ 上线）
```

### 4.3 CSI-V0 设计（最简版）

```
计算口径：
  对每个 (sku_id, region, date)：
    total_inventory = SUM(各企业库存)
    daily_consumption = SUM(出库) / 7（7 日均出库）
    days_of_inventory = total_inventory / daily_consumption

发布频率：
  每日 18:00
```

### 4.4 CFP-V0 设计（最简版）

```
计算口径（行业聚合）：
  对每个 (region, date)：
    avg_receivable_days = AVG(企业应收账款 / 日均销售)
    avg_overdue_rate = AVG(逾期金额 / 总应收)
    avg_note_ratio = AVG(承兑票占应收比)

仅对内部老板看板可见（数据极敏感，外部见 M18+）
```

---

## 五、数据脱敏 SOP（关键合规）

### 5.1 字段分级

| 级别 | 字段示例 | 处理方式 |
|---|---|---|
| L1 极敏 | 企业名 / usc_code / 法人 / 联系方式 | 哈希 + 不可逆 / 仅业主自见 |
| L2 敏 | 具体合同金额 / 客户/供应商名 / 仓库地址 | 模糊化 / 区间映射 / 加噪 |
| L3 中敏 | 品种 / 规格 / 区域 / 时间 | 保留（聚合统计） |
| L4 公开 | 价格指数（聚合）/ 行业平均 | 完全公开 |

### 5.2 K-匿名性（M3 上线，对内）

```
对所有 ads_*_v0 数据：
  任意 K 条记录组合后, 不能唯一识别某家企业
  默认 K = 10
  对极敏感数据集 K = 50

实现方式：
  - 区域聚合（不到具体仓库）
  - 时间窗口（不到具体时刻）
  - 数值加噪（拉普拉斯噪声）
```

### 5.3 用户授权状态实时检查

```
每次数据中台计算时, 必须检查：
  for each tenant in 数据贡献者:
    if tenant.consent_status != 'active':
      该 tenant 数据从本次计算中排除

授权状态变更（撤回）:
  - 实时更新 consent_status 表
  - 30 天内停用相关数据（不允许写入新指数）
  - 历史已聚合数据保留（已脱敏）
```

---

## 六、数据中台 MVP 上线节奏（12 周）

### W1-W2：数据基础设施搭建

| 任务 | 责任人 | 交付 |
|---|---|---|
| 数据湖搭建（对象存储 + Iceberg） | 数据架构师 | 数据湖可写 |
| Kafka 集群部署 + 主题规划 | 数据工程师 | 事件可投递 |
| ClickHouse / Redis / TimescaleDB 部署 | 数据工程师 | 查询可用 |
| 数据中台基础服务骨架 | 数据工程师 | 服务可启动 |

### W3-W4：数据接入

| 任务 | 责任人 | 交付 |
|---|---|---|
| Kafka 事件接入器 | 数据工程师 | SaaS 事件落湖 |
| ERP 历史数据迁移 CDC | 数据工程师 | 历史数据入湖 |
| 第三方钢联/Mysteel 接入 | 数据工程师 + 商务 | 行情数据每日入库 |
| 数据脱敏中间件 v0 | 数据工程师 + 法务 | 脱敏策略可配置 |
| 主数据对齐（钢材 / 客户 / 仓库 / 钢厂）| 数据 PM | 主数据完整 |

### W5-W6：行情域 + CSP-V0

| 任务 | 责任人 | 交付 |
|---|---|---|
| 行情数据 ETL 流水线 | 数据工程师 | dwd/dws 表完成 |
| CSP-V0 算法实现 | 算法 + PM | CSP-V0 表完成 |
| 内部 alpha API（行情）| 数据工程师 | 给 P1 / P4 调用 |
| 行情看板（内部） | BI 工程师 | 老板内部可见 |

### W7-W8：交易域 + 客户域

| 任务 | 责任人 | 交付 |
|---|---|---|
| 交易事件 ETL | 数据工程师 | dwd/dws 完成 |
| 客户 360 画像 | 数据工程师 | dwd_customer_360 完成 |
| CSC-V0 评分算法 | 算法工程师 | 评分可计算 |
| 内部 alpha API（客户/交易）| 数据工程师 | 给 P4 / P5 调用 |

### W9-W10：库存域 + 财务域

| 任务 | 责任人 | 交付 |
|---|---|---|
| 库存事件 ETL | 数据工程师 | 实时库存快照 |
| 库存浮盈实时计算（Flink）| 数据工程师 | 实时浮盈 API |
| CSI-V0 算法 | 算法工程师 | CSI 表完成 |
| 财务域 ETL | 数据工程师 | 应收账龄等指标 |
| 内部 alpha API（库存/财务）| 数据工程师 | 给 P5 调用 |

### W11-W12：钢厂域 + 全链路联通

| 任务 | 责任人 | 交付 |
|---|---|---|
| 钢厂提货 ETL | 数据工程师 | dws_mill_pickup_progress |
| CMP-V0 算法 | 算法工程师 | CMP 表完成 |
| 全链路压测 | 数据工程师 + QA | 性能达标 |
| 数据合规 review | 法务 + 数据合规 | 合规通过 |
| 数据中台监控告警 | 数据工程师 | 告警上线 |

---

## 七、内部数据 alpha API 详细规范

> 所有 SaaS 通过这套 API 访问数据中台，**不允许直接查数据中台的存储**。

### 7.1 通用规范

```
Base URL: https://internal-api.glt.local/data/v1

鉴权：
  - SaaS 间使用 mTLS 双向证书 + Service Token
  - 每个调用必须带 X-Service-Name + X-Tenant-Id

限流：
  - 默认 100 QPS / SaaS
  - 重要 API 单独配额

返回格式：
  {
    "code": 0,
    "data": {...},
    "request_id": "...",
    "trace": "...",
    "ts": 1234567890
  }
```

### 7.2 关键 API 清单（MVP 内部使用）

#### 行情域

```
GET /market/quote/realtime?sku=xxx&region=xxx
  返回：实时价格 + 5min 序列
  调用方：P1 / P4

GET /market/quote/daily?sku=xxx&region=xxx&start=xxx&end=xxx
  返回：历史日线
  调用方：P1

GET /market/index/csp?level=master&date=xxx
  返回：CSP 指数（master / 子指数）
  调用方：P1 / P4 / 老板看板

POST /market/ai/explain
  body: { sku, region, date }
  返回：AI 解读文本（结构化）
  调用方：P1
```

#### 交易域

```
GET /transaction/customer/summary?customer_id=xxx&period=30d
  返回：成交笔数 / 平均价 / 账期等
  调用方：P4

POST /transaction/anomaly/check
  body: { quote, customer, sales }
  返回：是否异常 + 原因
  调用方：P4

GET /transaction/sales/behavior?sales_id=xxx
  返回：业务员行为画像（毛利偏离 / 客户集中度 / 异常）
  调用方：风控 / 老板看板
```

#### 客户域

```
GET /customer/profile/360?customer_id=xxx
  返回：客户 360 画像（已脱敏）
  调用方：P4 / P5

GET /customer/score/payment?customer_id=xxx
  返回：付款评分
  调用方：P4 / P5 / 风控

GET /customer/score/risk?customer_id=xxx
  返回：风险评分（A-E）+ 5 项明细
  调用方：P4 / P5 / 老板看板
```

#### 库存域

```
GET /inventory/snapshot?tenant_id=xxx&warehouse_id=xxx
  返回：当前库存（按 sku 维度）
  调用方：P3 / P5

GET /inventory/floating-pl?tenant_id=xxx
  返回：实时浮盈
  调用方：P5

GET /inventory/turnover?sku=xxx&region=xxx
  返回：周转天数
  调用方：P5 / 老板看板

POST /inventory/anomaly/trace
  body: { tenant_id, warehouse_id, sku, period }
  返回：库存差异溯源（关联出入库流水）
  调用方：P3
```

#### 财务域

```
GET /finance/dashboard?tenant_id=xxx
  返回：资金大盘聚合数据
  调用方：P5

GET /finance/aging?tenant_id=xxx
  返回：应收账龄分析
  调用方：P5

GET /finance/forecast?tenant_id=xxx&days=30
  返回：未来 30/60/90 天现金流预测
  调用方：P5
```

#### 钢厂域

```
GET /mill/pickup-progress?tenant_id=xxx&mill=xxx&period=xxx
  返回：本月提货达成率 + 缺量预警
  调用方：P4 / P5
```

#### 风险域（M3+）

```
POST /risk/alerts/list?tenant_id=xxx
  返回：所有未处理风险告警
  调用方：P5 老板驾驶舱

POST /risk/alerts/dismiss
  body: { alert_id, reason }
  调用方：P5
```

---

## 八、数据质量与监控

### 8.1 数据质量规则（每日检查）

| 维度 | 规则 |
|---|---|
| 完整性 | 所有 ods 表必须有数据；缺失率 < 5% 告警 |
| 准确性 | 关键字段非空率 ≥ 99%；数值异常（如负价、负重）触发告警 |
| 时效性 | 实时数据延迟 ≤ 5 min；批量数据延迟 ≤ 1 day |
| 一致性 | 跨表关联校验（出库总和 = 客户总和）|
| 脱敏 | K-匿名性自动校验 |

### 8.2 监控告警

```
关键监控指标：
  - 事件队列堆积量
  - ETL 任务成功率
  - API 可用性（99.9%）
  - API P95 延迟（< 200ms）
  - 数据质量得分

告警渠道：
  - 企业微信群（实时告警）
  - 短信（重大告警）
  - 邮件（每日汇总）

告警分级：
  P0: 数据中台不可用 → 立即响应
  P1: 数据延迟 > 30 min / API 不可用 → 30 min 内
  P2: 数据质量异常 → 24h 内
  P3: 一般 → 工作日
```

---

## 九、数据安全

### 9.1 安全架构

```
访问控制：
  - SaaS 之间 mTLS
  - 服务级账户 + 最小权限

数据加密：
  - 传输 TLS 1.3
  - 存储 AES-256（业务库 + 数据湖）

审计日志：
  - 所有 API 调用记录
  - 所有跨域查询记录
  - 90 天内可追溯

关键操作：
  - 数据导出必须审批
  - 大批量查询限流
  - 异常访问 IP 自动封禁
```

### 9.2 合规保障

- 等保 2 级（W12 完成测评启动）
- 数据服务备案（W11 启动）
- 用户授权状态实时同步
- 撤回机制 30 天内执行

---

## 十、数据团队配置（M1-M3）

| 角色 | 人数 | 职责 |
|---|---|---|
| CDO | 1 | 战略 + 团队 |
| 数据架构师 | 1 | 整体架构设计 |
| 数据工程师 | 2 | ETL + 服务 |
| 算法工程师 | 1 | 指数算法 + AI 模型 |
| 数据 PM | 1 | 主数据 + 数据产品 |
| 数据合规专员 | 1 | 合规 + 法务对接 |

**总人头：7 人**（M1-M3 启动配置，详见 22 号文档）

---

## 十一、技术选型（MVP 阶段轻量化）

| 层 | 选型 | 理由 |
|---|---|---|
| 消息队列 | Kafka | 行业标配，生态完整 |
| 数据湖 | 对象存储 + Iceberg | 表格式 + ACID |
| 流计算 | Flink | 低延迟 + Exactly-Once |
| 批量计算 | Spark / Spark SQL | 离线处理 |
| OLAP | ClickHouse | 列式 + 查询快 |
| 缓存 | Redis | 热点查询 |
| 时序 | TimescaleDB | 价格序列 |
| ETL 编排 | Apache Airflow | 任务调度 |
| 数据治理 | DataHub（开源）| 元数据管理 |
| 服务化 | Python（FastAPI）/ Go | 数据服务 |

---

## 十二、风险与应对

| 风险 | 应对 |
|---|---|
| 数据接入延期 | M2 末必须有事件流入；用 mock 数据先撑住 SaaS 联调 |
| 主数据对齐难 | M2 启动主数据 review，建立同义词 + 模糊匹配 |
| 第三方 API 不稳定 | 多源备份 + 降级策略 |
| 计算性能瓶颈 | MVP 阶段单实例 ClickHouse 够用，M6+ 集群化 |
| 数据合规事故 | 法务前置审核 + 等保认证 |
| 数据团队招聘慢 | M2 末关键岗位必须到位 |

---

## 十三、配套阅读

- 14 号：最新战略基线
- 16 号：钢贸真实交易数据指数策划
- 19 号：数据资产货币化
- 20 号：数据资产估值财务模型
- 21 号：数据合规手册
- 26 号：MVP 总体架构
- 28-33 号：6 大 SaaS（数据消费方）
- 34 号：开发文档（含数据中台技术细节）

---

## 十四、最终结论

> **数据中台 MVP = 6 大 SaaS 的"能源核电站"**：
>
> - W4 末完成基础设施 + 第一个数据域（行情）
> - W8 末覆盖 4 个核心数据域 + 4 个 V0 版内部指数
> - W12 末全链路打通 + 内部 alpha API 完整可用
>
> **关键判断**：
> 1. 数据中台**与 SaaS 同期开发**，不能等
> 2. **MVP 阶段不对外卖数据**（避开合规和估值陷阱）
> 3. **alpha 阶段先内部跑通**，M7+ 才对外
> 4. 数据团队**M2 末关键岗位必须到位**（CDO + 架构师 + 算法）
>
> **这是公司未来估值最大的杠杆，必须从 Day 1 就重视。**
