# MVP 开发文档（任务拆分 + 技术栈 + 数据库 + API + 节奏）

> 本文是从 PRD（26-33 号）到工程落地的桥梁。研发团队按本文档启动开发。
>
> 适用读者：CTO / 架构师 / 工程经理 / 全员研发 / DevOps / QA。
>
> **本文档不写代码**，但提供：技术栈选型、模块划分、数据库 ER 概要、API 列表、任务拆分到周/人、技术里程碑、上线检查项。

---

## 一、技术栈总选型

### 1.1 选型原则
1. **稳定优先**：MVP 阶段不追新技术
2. **生态完整**：选社区活跃的，避免冷门坑
3. **团队熟悉**：尽量用团队会的，缩短上手时间
4. **云原生**：所有服务容器化，方便扩缩容
5. **可观测**：监控、日志、链路追踪从 Day 1

### 1.2 主要技术栈

| 层 | 选型 | 备注 |
|---|---|---|
| **前端** | | |
| Web 端 | React + TypeScript + Ant Design | 标准互联网技术栈 |
| 移动端 H5 | React + Vite + Vant | 轻量 |
| 微信小程序 | uni-app（一码三端：小程序+H5+APP）| 减少多端开发量 |
| 业务员 / 司机 APP | uni-app + 原生扩展（扫码/摄像头/GPS）| 同上 |
| **后端** | | |
| 主语言 | Java（Spring Boot 3.x）/ Go | 用团队最熟悉的 |
| API 网关 | Apache APISIX 或 Spring Cloud Gateway | 限流+鉴权+路由 |
| 服务注册发现 | Nacos / Consul | 配置中心一体化 |
| 微服务通信 | RESTful + gRPC（内部高频） | |
| **数据存储** | | |
| 关系数据库 | PostgreSQL 16 | OLTP 主库 |
| OLAP | ClickHouse 23.x | 分析查询 |
| 时序 | TimescaleDB（PG 扩展）| 行情/监控 |
| 缓存 | Redis 7 | 会话/热点 |
| 全文搜索 | Elasticsearch 8 | 社区/产品 |
| 对象存储 | S3 兼容（云厂商 OSS）| 图片/视频/PDF |
| 消息队列 | Kafka 3.x | 事件总线 |
| 数据湖 | Iceberg + S3 | ODS/DWD |
| **算法/AI** | | |
| 主语言 | Python 3.11 + FastAPI | 数据/AI 服务 |
| 大模型 | 通义千问 / 文心一言 / Claude（API 调用） | 多模型组合 |
| ML 框架 | PyTorch / scikit-learn | 评分模型 |
| 流计算 | Flink 1.18 | 实时指标 |
| 批量计算 | Spark / Spark SQL | 离线 |
| **基础设施** | | |
| 容器 | Docker + Kubernetes | |
| CI/CD | GitLab CI / GitHub Actions | |
| 监控 | Prometheus + Grafana | |
| 日志 | ELK / Loki | |
| 链路追踪 | OpenTelemetry + Jaeger | |
| 告警 | PagerDuty / 自研 | |
| **第三方** | | |
| 短信 | 阿里云 / 腾讯云 | |
| 微信 | 公众号 / 视频号 / 小程序 / 企微 SDK | |
| 电子签 | e签宝 / 法大大 | |
| 支付 | 持牌支付通道（M7+）| |
| 行情 | 上海钢联 / Mysteel / Wind API | |
| 工商 | 企查查 / 天眼查 API | |

---

## 二、微服务划分（与 26 号文档对齐）

### 2.1 服务列表

| # | 服务名 | 职责 | 主要依赖 |
|---|---|---|---|
| 1 | gateway-service | API 网关 | 全部 |
| 2 | user-service | 用户/组织/权限/登录 | PG / Redis |
| 3 | master-data-service | 钢材/客户/仓库/钢厂主数据 | PG |
| 4 | event-bus-service | 事件投递订阅 | Kafka |
| 5 | billing-service | 套餐/计费/发票 | PG |
| 6 | growth-service | 埋点/看板/AB | ClickHouse / Kafka |
| 7 | price-service | P1 钢价行情 + AI 解读 | PG / Redis / Kafka |
| 8 | statement-service | P2 电子对账单 | PG / Kafka |
| 9 | delivery-service | P3 扫码出库 | PG / Kafka / OSS |
| 10 | quote-service | P4 AI 报价 | PG / Kafka |
| 11 | finance-service | P5 资金大盘 + 电子合同 | PG / Kafka |
| 12 | community-service | P6 钢贸圈社区 | PG / ES / Kafka |
| 13 | data-platform-service | 数据中台（多个子模块） | Kafka / Iceberg / CK |
| 14 | ai-service | AI 解读 / AI 报价 / AI 风控 | Python / 大模型 API |
| 15 | notification-service | 推送/短信/邮件/企微 | RabbitMQ |
| 16 | file-service | 文件上传/下载/CDN | OSS |

### 2.2 服务边界原则
- 每个服务有自己的数据库 schema，跨服务通过 API/事件通信
- 数据中台不允许业务服务直接查询，必须通过内部 API
- 共享数据通过事件总线广播

---

## 三、数据库 ER 概要

### 3.1 核心实体（节选关键，详细 schema 见各 PRD）

```
租户与用户
  Tenant
  OrgNode
  User
  Role
  Permission
  ExternalUser

主数据
  MaterialMaster
  CustomerMaster
  WarehouseMaster
  MillMaster
  ContractTemplateMaster

P1 行情
  PriceQuote
  DailyReport
  UserSubscription

P2 对账单
  StatementBatch
  Statement
  StatementItem
  StatementDispute
  StatementShareLog

P3 出库
  DeliveryOrder
  DeliveryItem
  ScaleReading
  QualityCertificate
  DriverApp
  ClientReceipt
  DeliveryAnomaly
  InventoryDiff

P4 报价
  QuoteRequest
  QuoteSuggestion
  Quote
  QuoteApproval
  ClientProfile
  SalesBehaviorScore

P5 资金/合同
  CashDashboard
  Contract
  Note
  Receivable
  Alert

P6 社区
  Channel
  Post
  Comment
  InquiryPost
  UserCertification
  OldDebtorBoard

数据中台
  ods_*  (原始)
  dwd_*  (明细)
  dws_*  (汇总)
  ads_*  (应用)

计费
  SubscriptionPlan
  Subscription
  UsageMetric
  Invoice / Payment
```

### 3.2 关键约束

| 约束 | 说明 |
|---|---|
| **多租户** | 所有业务表必须有 `tenant_id` + 索引 |
| **乐观锁** | 关键表加 `version` 字段 |
| **软删除** | 用 `deleted_at` 字段，不物理删除 |
| **审计字段** | `created_at / updated_at / created_by / updated_by` |
| **租户隔离** | 数据库行级安全（RLS）+ ORM 中间件双重保障 |

### 3.3 数据库分库分表策略

| 表 | 量级估计 (M12) | 策略 |
|---|---|---|
| 用户/客户主数据 | 10 万 | 单库 |
| 报价 | 100 万 | 单库 |
| 出库 | 50 万 | 单库 |
| 对账单 | 50 万 | 单库 |
| 事件 | 千万级 | Kafka + 数据湖 |
| 日志 | 亿级 | ELK + 数据湖 |

> MVP 阶段不分库，M12 根据实际量级再决定。

---

## 四、API 设计规范

### 4.1 通用规范

```
Base URL: https://api.gangliantong.com/v1

鉴权: JWT Bearer Token
租户: X-Tenant-Id Header
请求 ID: X-Request-Id Header (用于追踪)
版本: URL 路径包含版本 (/v1/, /v2/)

响应格式:
{
  "code": 0,                    // 0 = 成功，非 0 = 错误码
  "message": "ok",
  "data": { ... },
  "request_id": "...",
  "ts": 1234567890
}

错误码:
  0       成功
  10001   未登录
  10002   权限不足
  10003   参数错误
  10004   资源不存在
  10005   重复请求
  20001   业务错误（具体见各服务）
  50001   服务器内部错误
```

### 4.2 关键 API 清单（节选）

#### user-service

```
POST   /v1/auth/login                登录
POST   /v1/auth/wechat-login          微信登录
POST   /v1/auth/logout                登出
GET    /v1/users/me                   当前用户
PATCH  /v1/users/me                   修改资料
GET    /v1/tenants/me                 当前企业
GET    /v1/orgs/tree                  组织架构树
GET    /v1/users                      用户列表
POST   /v1/users                      创建用户
PATCH  /v1/users/{id}                 修改用户
GET    /v1/roles                      角色列表
POST   /v1/roles/{id}/users           给用户授角色
```

#### master-data-service

```
GET    /v1/materials                  钢材主数据查询
POST   /v1/materials                  新建
GET    /v1/customers                  客户列表
POST   /v1/customers                  创建客户
PATCH  /v1/customers/{id}             修改客户
GET    /v1/customers/{id}/sovereignty 客户主权状态
POST   /v1/customers/{id}/transfer    主权转移（特殊）
GET    /v1/warehouses                 仓库列表
GET    /v1/mills                      钢厂列表
```

#### price-service (P1)

```
GET    /v1/price/quote/realtime       实时行情
GET    /v1/price/quote/daily          日线
GET    /v1/price/index/csp            CSP 指数
GET    /v1/price/reports/daily        每日早报
POST   /v1/price/subscriptions        订阅设置
```

#### statement-service (P2)

```
POST   /v1/statements/batches         创建对账批次
GET    /v1/statements/{id}            对账单详情
POST   /v1/statements/{id}/send       发送
POST   /v1/statements/share/{token}/confirm   客户确认（H5）
POST   /v1/statements/share/{token}/dispute   客户异议（H5）
GET    /v1/statements/disputes        异议列表
PATCH  /v1/statements/disputes/{id}   处理异议
```

#### delivery-service (P3)

```
POST   /v1/delivery/orders            创建出库单
POST   /v1/delivery/orders/{id}/recommend  智能推荐捆号
POST   /v1/delivery/orders/{id}/scan-item  扫码捆号
POST   /v1/delivery/orders/{id}/weighing   过磅记录
POST   /v1/delivery/orders/{id}/complete   完成出库
GET    /v1/delivery/orders/{id}/cert       质保书 PDF
POST   /v1/delivery/share/{token}/sign     客户 H5 签收
GET    /v1/delivery/inventory-diff/trace   差异溯源
```

#### quote-service (P4)

```
POST   /v1/quote/requests             发起报价请求
GET    /v1/quote/requests/{id}/suggestion  AI 4 数字 + 建议
POST   /v1/quote/quotes               生成报价单
POST   /v1/quote/quotes/{id}/send     发给客户
POST   /v1/quote/share/{token}/accept   客户 H5 确认
POST   /v1/quote/share/{token}/reject   客户 H5 拒绝
POST   /v1/quote/quotes/{id}/approve  老板审批
POST   /v1/quote/quotes/{id}/reject   老板驳回
GET    /v1/quote/sales-behavior       业务员行为分析
```

#### finance-service (P5)

```
GET    /v1/finance/dashboard          资金大盘
GET    /v1/finance/aging              账龄分析
GET    /v1/finance/notes              票据池
POST   /v1/finance/notes/{id}/discount  贴现（M7+）
GET    /v1/finance/forecast           资金预测
GET    /v1/finance/alerts             告警

POST   /v1/finance/contracts          创建合同
GET    /v1/finance/contracts/{id}     合同详情
POST   /v1/finance/contracts/{id}/audit  AI 风险审查
POST   /v1/finance/contracts/share/{token}/sign  双方签字
```

#### community-service (P6)

```
GET    /v1/community/channels         频道列表
GET    /v1/community/posts            帖子流
POST   /v1/community/posts            发帖
GET    /v1/community/posts/{id}       帖子详情
POST   /v1/community/posts/{id}/like  点赞
POST   /v1/community/posts/{id}/comment  评论
GET    /v1/community/ai/daily         AI 今日要闻
POST   /v1/community/messages         私聊
GET    /v1/community/users/{id}/credit-preview  征信预览
GET    /v1/community/old-debtor-board  老赖榜
```

#### data-platform-service (内部 API)

```
GET    /internal/data/v1/market/quote/realtime
GET    /internal/data/v1/market/index/csp
POST   /internal/data/v1/market/ai/explain
GET    /internal/data/v1/transaction/customer/summary
POST   /internal/data/v1/transaction/anomaly/check
GET    /internal/data/v1/transaction/sales/behavior
GET    /internal/data/v1/customer/profile/360
GET    /internal/data/v1/customer/score/payment
GET    /internal/data/v1/customer/score/risk
GET    /internal/data/v1/inventory/snapshot
GET    /internal/data/v1/inventory/floating-pl
GET    /internal/data/v1/finance/dashboard
GET    /internal/data/v1/finance/forecast
GET    /internal/data/v1/mill/pickup-progress
POST   /internal/data/v1/risk/alerts/list
```

---

## 五、12 周开发任务拆分

### 5.1 团队结构（启动）

| 团队 | 人数 | 职责 |
|---|---|---|
| 架构组 | 1 + 1 | 架构师 + DevOps，统一底座 |
| 数据中台组 | 5 | CDO + 架构师 + 2 工程师 + 1 算法 |
| P1 行情组 | 2 | 1 后端 + 1 前端 |
| P2 对账组 | 2 | 1 后端 + 1 前端 |
| P3 出库组 | 3 | 2 后端 + 1 前端（含磅秤集成） |
| P4 报价组 | 2 | 1 后端 + 1 前端 |
| P5 资金/合同组 | 2 | 1 后端 + 1 前端 |
| P6 社区组 | 2 | 1 后端 + 1 前端 |
| AI 组 | 1 | AI 解读/报价/风控算法 |
| 测试 | 2 | 自动化 + 手工 |
| **合计** | **23** | |

### 5.2 12 周里程碑

#### W1: 项目初始化

| 任务 | 责任组 | 交付 |
|---|---|---|
| 代码仓库 / CI/CD 搭建 | 架构 | GitLab + 流水线 |
| 云平台账号 / VPC | DevOps | K8s 集群 |
| 监控/日志/链路 | DevOps | Prometheus / Grafana / ELK |
| 数据库部署 | DevOps | PG / Redis / Kafka / CK / ES |
| 项目骨架 | 架构 | 各服务 hello world |

#### W2: 统一底座 v1

| 任务 | 责任组 | 交付 |
|---|---|---|
| user-service 用户/组织/权限 | 架构 | 注册/登录/JWT/RBAC |
| master-data-service 主数据 | 架构 | 4 类主数据 CRUD |
| event-bus-service 事件骨架 | 架构 | Kafka 投递订阅 |
| billing-service 套餐管理 | 架构 | 4 个套餐配置 |
| 微信小程序登录 | 架构 + 前端 | unionid 联合 |
| 数据中台基础设施 | 数据 | 数据湖 + Kafka 接入 |

#### W3: 行情数据接入 + P1 启动

| 任务 | 责任组 | 交付 |
|---|---|---|
| 钢联 / Mysteel API 接入 | 数据 | 行情每日入库 |
| 价格指数算法 v0 | 数据/算法 | CSP-V0 表 |
| price-service 骨架 | P1 | 行情查询 API |
| AI 解读 prompt | AI | 早报模板 |
| 公众号 / 视频号矩阵申请 | 运营 | 账号矩阵 |

#### W4: P1 上线

| 任务 | 责任组 | 交付 |
|---|---|---|
| price-service 全功能 | P1 | 实时/日线/CSP/订阅 |
| 移动端首页 + 早报详情 | P1 前端 | H5 + 小程序 |
| 公众号自动推送 | 运营 + P1 | 早报推送 |
| 内部数据 alpha API v0（行情） | 数据 | API 上线 |
| **P1 钢价行情 v1.0 上线** | 全员 | 灰度 50 客户 |

#### W5: P2 对账单开发

| 任务 | 责任组 | 交付 |
|---|---|---|
| statement-service 后端 | P2 | 生成/发送/确认 |
| H5 对账单页面 | P2 前端 | 移动端优先 |
| 病毒位 4 个 + AB 实验 | P2 + 运营 | 漏斗看板 |
| 多渠道发送 | P2 | 微信/短信/邮件 |

#### W6: P2 上线 + 病毒拉新

| 任务 | 责任组 | 交付 |
|---|---|---|
| 异议派单 | P2 | 自动派给业务员 |
| 全量推送 2000 家客户 | 运营 + 客户成功 | 激活 ≥ 60% |
| **P2 电子对账单 v1.0 上线** | 全员 | 月底对账场景 |
| 增长漏斗看板 | 数据 + 运营 | 实时漏斗 |

#### W7: P3 扫码出库开发

| 任务 | 责任组 | 交付 |
|---|---|---|
| delivery-service 后端 | P3 | 出库主链路 |
| 磅秤直连（先支持耀华） | P3 | 边缘网关 |
| 智能推荐捆号算法 | P3 + 算法 | 推荐评分 |
| 仓管 PAD 端 | P3 前端 | 扫码主流程 |
| 司机 APP MVP | P3 前端 | 接单+签收 |
| 客户收货 H5 | P3 前端 | 签收+异议 |
| 质保书 PDF 生成 | P3 + 模板 | 自动 PDF |

#### W8: P3 试点上线

| 任务 | 责任组 | 交付 |
|---|---|---|
| 5 家试点仓库部署 | 客户成功 + P3 | 现场陪伴 |
| 库存差异溯源 | P3 + 数据 | 一键溯源 |
| **P3 扫码出库 v1.0 试点** | 全员 | 5 家客户 |
| 内部数据 alpha v1（出库数据） | 数据 | API 升级 |

#### W9: P4 AI 报价开发

| 任务 | 责任组 | 交付 |
|---|---|---|
| quote-service 后端 | P4 | 报价主链路 |
| AI 4 数字算法 | AI + 数据 | 实时调用 |
| 报价 H5 + 锁价倒计时 | P4 前端 | 客户端 |
| 老板审批流 | P4 + 通知 | 推送+审批 |
| 业务员风控引擎 | P4 + 数据 | 评分模型 |
| 客户主权机制实现 | 主数据 | 6 个月主权 |

#### W10: P4 上线

| 任务 | 责任组 | 交付 |
|---|---|---|
| 业务员 APP 一键报价 | P4 前端 | 语音 + 文字 |
| 客户画像 | P4 + 数据 | 360 视图 |
| 30 家业务员密集客户内测 | 客户成功 | 反馈收集 |
| **P4 AI 报价 v1.0 上线** | 全员 | 业务员激活 |

#### W11: P5 + P6 开发

| 任务 | 责任组 | 交付 |
|---|---|---|
| finance-service 资金大盘 | P5 | 4 大块 + 告警 |
| 电子合同 + AI 风险审查 | P5 + AI | 模板库 + 签章 |
| community-service 后端 | P6 | 频道+发帖+评论 |
| 钢贸圈社区前端 | P6 前端 | 主页+频道 |
| AI 今日要闻 | P6 + AI | 自动总结 |

#### W12: 全量联调 + 复盘 + 融资

| 任务 | 责任组 | 交付 |
|---|---|---|
| 6 大 SaaS 互联互通 | 全员 | 飞轮成型 |
| 数据中台全链路 | 数据 | 6 大数据域 |
| 性能压测 | QA + DevOps | P95 < 500ms |
| 安全渗透测试 | 安全 | 主要漏洞 0 |
| 法务合规专项启动 | 法务 + CDO | 等保 2 级启动 |
| MVP 复盘报告 | CEO + 全员 | 90 天报告 |
| Pre-A 融资材料 | CFO + CEO | BP + 财务模型 |

---

## 六、技术约束与最佳实践

### 6.1 必须遵守

| 约束 | 说明 |
|---|---|
| 所有 API 必须 JWT 鉴权 | 不允许任何无认证接口 |
| 多租户严格隔离 | ORM 中间件 + RLS 双重保障 |
| 关键操作审计日志 | 用户行为 / 数据访问 / 合规审计 |
| 数据脱敏 | 涉及敏感字段必须脱敏存储 / 显示 |
| 幂等性 | 关键写操作（创建合同/出库/支付）必须幂等 |
| 重试 | 第三方调用必须有重试 + 超时 |
| 限流 | API 网关层限流 |
| 监控告警 | 关键指标必须有告警 |
| 单元测试 | 核心业务覆盖率 ≥ 70% |
| 代码 review | 所有 PR 必须 1 人以上 review |

### 6.2 性能目标

| 指标 | 目标 |
|---|---|
| API P95 延迟 | < 500 ms |
| 关键 API（首页、报价）P95 | < 200 ms |
| H5 首屏 | < 1.5 s |
| 数据中台实时查询 | < 1 s |
| 行情推送延迟 | < 5 min |
| 出库扫码响应 | < 100 ms |

### 6.3 安全要求

- HTTPS 全站
- TLS 1.3
- 敏感字段加密存储（AES-256）
- 密码 bcrypt
- API 限流（防爬）
- WAF 防护
- 等保 2 级（M6 末完成测评）

---

## 七、与外部系统集成

### 7.1 必须对接（M1-M3）

| 外部 | 用途 | 状态 |
|---|---|---|
| 微信公众号 | 推送 + 引流 | M1 申请 |
| 微信视频号 | 内容矩阵 | M1 申请 |
| 微信小程序 | 主入口 | M1 申请 |
| 微信支付 | 套餐订阅 | M2 申请 |
| 上海钢联 / Mysteel | 行情数据 | M1 商务谈判 |
| 短信 / 邮件 | 通知 | M1 |
| 阿里云 OSS / 腾讯云 COS | 文件存储 | M1 |

### 7.2 第二批（M4-M6）

| 外部 | 用途 |
|---|---|
| e签宝 / 法大大 | 电子合同 |
| 企查查 / 天眼查 | 工商数据 |
| 磅秤厂商（耀华/上海大华）| 设备接入 |
| 银行 API（部分支持） | 卡余额聚合 |

### 7.3 第三批（M7+，金融业务）

| 外部 | 用途 |
|---|---|
| 持牌支付公司（连连/汇付） | 担保账户 |
| 持牌保理公司 | 保理放款 |
| 银行供应链金融部门 | 联合放款 |
| 持牌票据公司 | 票据贴现 |
| 货拉拉 / 满帮 / 自有车队 | 物流撮合 |
| 上海/深圳数据交易所 | 数据挂牌 |
| 保险公司（人保 / 平安）| 信用险 |

---

## 八、上线检查清单（W12 末）

### 8.1 技术
- [ ] 所有 16 个微服务部署完成
- [ ] 数据中台 6 大数据域 + 4 个 V0 指数
- [ ] CI/CD 自动化部署
- [ ] 监控告警 7×24
- [ ] 性能压测达标
- [ ] 安全测试无重大漏洞
- [ ] 数据库备份 + 灾备
- [ ] API 文档完整

### 8.2 产品
- [ ] P1-P6 全部 v1 上线
- [ ] 6 大 SaaS 互联互通
- [ ] 移动端 + Web 端 + H5 全覆盖
- [ ] 关键路径用户测试通过

### 8.3 数据
- [ ] 数据中台事件流接入
- [ ] 主数据完整性检查
- [ ] 内部 alpha API 可用
- [ ] 数据质量监控

### 8.4 合规
- [ ] 用户协议/隐私政策/数据贡献条款
- [ ] 等保 2 级测评启动
- [ ] 法务专项 review 通过

### 8.5 运营
- [ ] 公众号矩阵开通
- [ ] 增长漏斗看板可用
- [ ] 客户成功 SOP

### 8.6 业务
- [ ] 2000 家存量客户激活率 ≥ 60%
- [ ] DAU ≥ 5,000
- [ ] 月发对账单 ≥ 5w 张
- [ ] Pre-A 融资 LOI ≥ 1 份

---

## 九、风险与应对

| 风险 | 影响 | 应对 |
|---|---|---|
| 关键技术人员离职 | 延期 | 知识库 + 期权 + 双备份 |
| 数据中台延期 | 影响 SaaS | mock 数据先撑 + 数据组优先级 |
| 磅秤集成困难 | P3 试点延期 | 先支持 1 款（耀华），后续扩展 |
| 钢联/Mysteel 商务卡 | P1 影响 | 提前 2 个月谈，多源备份 |
| 数据合规风险 | 业务停滞 | 法务 W1 介入 |
| 客户激活率低 | DAU 不达预期 | 客户成功投入加倍 |

---

## 十、研发文化与节奏

### 10.1 例会
- **每日站会**（15 分钟）：各组同步进度
- **每周技术评审**（1 小时）：跨组同步 + 决策
- **每周复盘**（1 小时）：周五下午
- **每月技术回顾**（半天）

### 10.2 文档
- 所有技术决策必须有文档（ADR - Architecture Decision Record）
- 所有 API 必须有文档（OpenAPI / Swagger）
- 关键模块必须有设计文档

### 10.3 文化
- 不写过度抽象的代码（YAGNI）
- 优先做"够用"的方案，再迭代
- 任何写死的字符串必须有理由
- 任何 magic number 必须有注释

---

## 十一、配套阅读

- 26 号：MVP 总体架构
- 27 号：数据中台 MVP
- 28-33 号：6 大 SaaS PRD
- 24 号：3 个月 MVP 启动指南（项目管理视角）

---

## 十二、最终结论

> **MVP 不追求完美，追求"跑通飞轮 + 验证产品价值 + 完成 Pre-A 融资"。**
>
> **W12 末关键标准**：
> - 6 大 SaaS 上线
> - 数据中台 4 个 V0 指数
> - 5,000 DAU
> - 飞轮跑通
> - Pre-A LOI ≥ 1 份
>
> **这是从战略到执行的最后一公里。从 W1 开始，按本文档执行。**
