# 数据库 Schema 完整 SQL + API OpenAPI 规范

> 本文是 26/27/34 号文档的工程级深化。提供完整可执行的 PostgreSQL DDL（不是真正的 SQL 编译执行，是规范文档）+ 关键 API 的 OpenAPI 3.0 风格规范。
>
> 适用读者：架构师 / 后端工程师 / DBA / 接口测试。
>
> **本文档不是真正的代码**，是工程师按本文档实现代码时的规范。

---

## 一、数据库设计原则

### 1.1 通用约定

| 约定 | 规则 |
|---|---|
| 命名 | 表/字段全部 snake_case 小写 |
| 主键 | 默认 BIGSERIAL（自增）+ 业务主键唯一索引 |
| 外键 | 不在 DB 层做外键约束（用应用层），避免锁问题 |
| 多租户 | 所有业务表必须有 `tenant_id` + 索引 |
| 时间 | TIMESTAMPTZ（带时区）|
| 钱 | NUMERIC(18,2) 不用 FLOAT |
| 软删除 | `deleted_at` TIMESTAMPTZ |
| 审计 | `created_at / updated_at / created_by / updated_by` |
| 乐观锁 | `version` BIGINT |
| 索引 | 高频查询字段 + 外键字段 |
| JSON | 配置/扩展用 JSONB |
| 分区 | 大表（事件/日志）按时间分区 |

### 1.2 核心 schema 列表

```
schema: gangliantong (主)
  - 用户/组织/权限相关表 (user_*)
  - 主数据 (md_*)
  - 业务表（每个 SaaS 独立前缀）
    - p1_* 行情
    - p2_* 对账单
    - p3_* 出库
    - p4_* 报价
    - p5_* 资金/合同
    - p6_* 社区
  - 事件 / 通知 / 计费 / 增长

schema: data_platform (数据中台)
  - ods_* / dwd_* / dws_* / ads_*
```

---

## 二、统一底座表 DDL

### 2.1 用户/组织/权限

```sql
-- 租户（一家钢贸企业）
CREATE TABLE tenants (
  id              BIGSERIAL PRIMARY KEY,
  tenant_code     VARCHAR(32) UNIQUE NOT NULL,         -- 平台分配
  name            VARCHAR(128) NOT NULL,
  usc_code        VARCHAR(32) UNIQUE,                  -- 统一社会信用代码
  size_tier       VARCHAR(16),                         -- small/medium/large/group
  region          VARCHAR(64),
  legal_person    VARCHAR(64),
  status          VARCHAR(16) DEFAULT 'active',        -- active/suspended/closed
  subscription_plan_id BIGINT,
  consent_data_contribute BOOLEAN DEFAULT FALSE,       -- 数据贡献授权
  consent_signed_at TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW(),
  deleted_at      TIMESTAMPTZ
);
CREATE INDEX idx_tenants_status ON tenants(status);
CREATE INDEX idx_tenants_usc ON tenants(usc_code);

-- 组织节点（多公司/分支机构，树形）
CREATE TABLE org_nodes (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  parent_id       BIGINT,
  name            VARCHAR(128) NOT NULL,
  type            VARCHAR(32),                         -- hq/branch/dept/warehouse
  level           INT,
  path            VARCHAR(256),                        -- 物化路径，如 /1/3/12
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  deleted_at      TIMESTAMPTZ
);
CREATE INDEX idx_org_tenant ON org_nodes(tenant_id);
CREATE INDEX idx_org_parent ON org_nodes(parent_id);

-- 用户
CREATE TABLE users (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  org_node_id     BIGINT,
  name            VARCHAR(64) NOT NULL,
  mobile          VARCHAR(16) UNIQUE NOT NULL,
  email           VARCHAR(128),
  wechat_unionid  VARCHAR(64),
  wechat_openid_mp VARCHAR(64),                        -- 小程序
  wechat_openid_oa VARCHAR(64),                        -- 公众号
  wechat_openid_video VARCHAR(64),                     -- 视频号
  password_hash   VARCHAR(256),
  status          VARCHAR(16) DEFAULT 'active',
  last_login_at   TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW(),
  deleted_at      TIMESTAMPTZ,
  version         BIGINT DEFAULT 0
);
CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_mobile ON users(mobile);
CREATE INDEX idx_users_unionid ON users(wechat_unionid);

-- 角色
CREATE TABLE roles (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT,                              -- NULL 表示全局角色
  code            VARCHAR(32) NOT NULL,                -- owner/sales/finance/warehouse/...
  name            VARCHAR(64) NOT NULL,
  description     TEXT,
  is_system       BOOLEAN DEFAULT FALSE,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 用户-角色关联
CREATE TABLE user_roles (
  user_id         BIGINT NOT NULL,
  role_id         BIGINT NOT NULL,
  granted_at      TIMESTAMPTZ DEFAULT NOW(),
  granted_by      BIGINT,
  PRIMARY KEY (user_id, role_id)
);

-- 权限点
CREATE TABLE permissions (
  id              BIGSERIAL PRIMARY KEY,
  code            VARCHAR(64) UNIQUE NOT NULL,         -- contract:read, quote:approve...
  name            VARCHAR(128),
  module          VARCHAR(32),
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 角色-权限关联
CREATE TABLE role_permissions (
  role_id         BIGINT NOT NULL,
  permission_id   BIGINT NOT NULL,
  PRIMARY KEY (role_id, permission_id)
);

-- 外部用户（散客买家、司机、第三方）
CREATE TABLE external_users (
  id              BIGSERIAL PRIMARY KEY,
  type            VARCHAR(16) NOT NULL,                -- buyer/driver/3rd_party
  mobile          VARCHAR(16) NOT NULL,
  name            VARCHAR(64),
  wechat_openid   VARCHAR(64),
  verified_at     TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE (type, mobile)
);
```

### 2.2 主数据

```sql
-- 钢材主数据
CREATE TABLE md_materials (
  id              BIGSERIAL PRIMARY KEY,
  sku_code        VARCHAR(64) UNIQUE NOT NULL,         -- 品种+钢厂+牌号+规格
  category        VARCHAR(32) NOT NULL,                -- 板/管/型/线/特钢
  sub_category    VARCHAR(64) NOT NULL,                -- 螺纹钢/线材/...
  mill_code       VARCHAR(32),                         -- 钢厂代码
  grade           VARCHAR(32),                         -- HRB400E/Q235B/SS400
  spec            VARCHAR(64),                         -- Φ16/12mm/...
  theoretical_weight_per_m NUMERIC(10,4),              -- 理论重量
  default_unit    VARCHAR(8),                          -- ton/piece/bundle
  aliases         JSONB,                               -- 同义词
  status          VARCHAR(16) DEFAULT 'active',
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_materials_category ON md_materials(category, sub_category);
CREATE INDEX idx_materials_mill_grade ON md_materials(mill_code, grade);

-- 客户主数据
CREATE TABLE md_customers (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  usc_code        VARCHAR(32),                         -- 统一信用代码
  name            VARCHAR(128) NOT NULL,
  aliases         JSONB,
  parent_id       BIGINT,                              -- 关联企业
  type            VARCHAR(32),                         -- 工地/加工厂/小钢厂/批发/零售
  region          VARCHAR(64),
  contact_name    VARCHAR(64),
  contact_mobile  VARCHAR(16),
  main_skus       JSONB,                               -- 主营品类
  credit_limit    NUMERIC(18,2),
  payment_score   INT,                                 -- 0-100
  risk_level      VARCHAR(8),                          -- A/B/C/D/E
  sales_owner_id  BIGINT,                              -- 业务员归属
  first_dealing_at TIMESTAMPTZ,                        -- 首次成交（用于客户主权）
  main_supplier_tenant_id BIGINT,                      -- 主权方
  main_supplier_until TIMESTAMPTZ,                     -- 主权到期
  client_choice_unbind_at TIMESTAMPTZ,                 -- 客户主动换卖家时间
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW(),
  deleted_at      TIMESTAMPTZ,
  version         BIGINT DEFAULT 0
);
CREATE INDEX idx_customers_tenant ON md_customers(tenant_id);
CREATE INDEX idx_customers_usc ON md_customers(usc_code);
CREATE INDEX idx_customers_main_supplier ON md_customers(main_supplier_tenant_id, main_supplier_until);

-- 仓库主数据
CREATE TABLE md_warehouses (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  name            VARCHAR(128) NOT NULL,
  type            VARCHAR(16),                         -- self/social/forward/mill
  address         VARCHAR(256),
  gps_lat         NUMERIC(10,7),
  gps_lng         NUMERIC(10,7),
  capacity        NUMERIC(18,2),                       -- 吨
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  deleted_at      TIMESTAMPTZ
);

-- 仓位/垛位（树形）
CREATE TABLE md_warehouse_locations (
  id              BIGSERIAL PRIMARY KEY,
  warehouse_id    BIGINT NOT NULL,
  parent_id       BIGINT,
  code            VARCHAR(32) NOT NULL,                -- 如 A-03
  name            VARCHAR(64),
  level           INT,                                 -- 1=区 2=垛位
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE (warehouse_id, code)
);

-- 钢厂主数据
CREATE TABLE md_mills (
  id              BIGSERIAL PRIMARY KEY,
  code            VARCHAR(32) UNIQUE NOT NULL,
  name            VARCHAR(128) NOT NULL,
  usc_code        VARCHAR(32),
  region          VARCHAR(64),
  specialty_skus  JSONB,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 钢厂代理协议
CREATE TABLE md_agency_agreements (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  mill_id         BIGINT NOT NULL,
  year            INT NOT NULL,
  monthly_min_volume NUMERIC(18,2),                    -- 月度最低提货
  rebate_per_ton  NUMERIC(10,2),                       -- 返点
  penalty_per_ton NUMERIC(10,2),                       -- 罚款
  status          VARCHAR(16) DEFAULT 'active',
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 资源可见度（4 级）
CREATE TABLE md_resource_visibility (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  sku_id          BIGINT NOT NULL,
  level           VARCHAR(8) NOT NULL,                 -- L0/L1/L2/L3
  target_circle_id BIGINT,                             -- L1 用
  upgraded_at     TIMESTAMPTZ,
  upgraded_by     BIGINT,
  UNIQUE (tenant_id, sku_id)
);
```

### 2.3 事件总线

```sql
-- 事件主题（业务表，仅做注册和监控用，事件实际在 Kafka）
CREATE TABLE event_topics (
  id              BIGSERIAL PRIMARY KEY,
  topic           VARCHAR(64) UNIQUE NOT NULL,
  description     TEXT,
  partition_count INT DEFAULT 3,
  retention_days  INT DEFAULT 30,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 事件订阅
CREATE TABLE event_subscriptions (
  id              BIGSERIAL PRIMARY KEY,
  topic           VARCHAR(64) NOT NULL,
  subscriber_service VARCHAR(64) NOT NULL,
  consumer_group  VARCHAR(64) NOT NULL,
  filter_expr     TEXT,
  status          VARCHAR(16) DEFAULT 'active',
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 事件死信表（无法处理的事件归档）
CREATE TABLE event_dead_letters (
  id              BIGSERIAL PRIMARY KEY,
  topic           VARCHAR(64) NOT NULL,
  partition       INT,
  offset_value    BIGINT,
  payload         JSONB,
  error_message   TEXT,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 2.4 计费

```sql
-- 套餐
CREATE TABLE subscription_plans (
  id              BIGSERIAL PRIMARY KEY,
  code            VARCHAR(32) UNIQUE NOT NULL,         -- free/business/enterprise/group
  name            VARCHAR(64) NOT NULL,
  annual_price    NUMERIC(18,2),
  features        JSONB,                               -- 启用的功能列表
  quotas          JSONB,                               -- 限量列表
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 订阅
CREATE TABLE subscriptions (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  plan_id         BIGINT NOT NULL,
  start_at        DATE NOT NULL,
  end_at          DATE NOT NULL,
  status          VARCHAR(16),                         -- active/expired/grace
  auto_renew      BOOLEAN DEFAULT TRUE,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 用量
CREATE TABLE usage_metrics (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  metric_name     VARCHAR(64) NOT NULL,                -- statement_count/quote_count/...
  period          VARCHAR(8) NOT NULL,                 -- 2026-04
  quota           BIGINT,
  used            BIGINT DEFAULT 0,
  overage         BIGINT DEFAULT 0,
  UNIQUE (tenant_id, metric_name, period)
);

-- 发票
CREATE TABLE invoices (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  amount          NUMERIC(18,2),
  period          VARCHAR(16),
  status          VARCHAR(16),                         -- pending/paid/cancelled
  paid_at         TIMESTAMPTZ,
  payment_method  VARCHAR(32),
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 2.5 增长分析（埋点表）

```sql
-- 埋点事件（OLAP，建议进 ClickHouse）
CREATE TABLE growth_events (
  id              BIGSERIAL PRIMARY KEY,
  user_id         BIGINT,
  tenant_id       BIGINT,
  event_name      VARCHAR(64),
  event_props     JSONB,
  device_type     VARCHAR(16),
  ip              INET,
  ua              TEXT,
  occurred_at     TIMESTAMPTZ DEFAULT NOW()
)
PARTITION BY RANGE (occurred_at);

-- A/B 实验
CREATE TABLE growth_experiments (
  id              BIGSERIAL PRIMARY KEY,
  name            VARCHAR(64),
  variants        JSONB,                               -- ["A", "B"]
  status          VARCHAR(16),
  start_at        TIMESTAMPTZ,
  end_at          TIMESTAMPTZ
);
```

---

## 三、6 大 SaaS 关键表 DDL

### 3.1 P1 行情

```sql
-- 行情报价（高频写入，建议时序表）
CREATE TABLE p1_price_quotes (
  id              BIGSERIAL,
  sku_id          BIGINT NOT NULL,
  region          VARCHAR(32),
  source          VARCHAR(16),                         -- ganglian/mysteel/internal
  quote_price     NUMERIC(10,2),
  quote_volume    NUMERIC(18,2),
  quote_type      VARCHAR(16),                         -- spot/futures/mill
  quoted_at       TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (id, quoted_at)
)
PARTITION BY RANGE (quoted_at);

-- 每日早报
CREATE TABLE p1_daily_reports (
  id              BIGSERIAL PRIMARY KEY,
  report_date     DATE NOT NULL,
  type            VARCHAR(16),                         -- morning/evening
  title           VARCHAR(256),
  summary         TEXT,
  ai_analysis     JSONB,
  key_prices      JSONB,
  factors         JSONB,
  proprietary_data JSONB,
  editor_id       BIGINT,
  status          VARCHAR(16) DEFAULT 'draft',         -- draft/reviewed/published
  published_at    TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- 用户订阅设置
CREATE TABLE p1_user_subscriptions (
  user_id         BIGINT PRIMARY KEY,
  sku_filters     JSONB,
  region_filters  JSONB,
  threshold_change INT,                                -- 涨跌阈值
  push_channels   JSONB,                               -- ["wechat", "sms", "appex"]
  push_times      JSONB                                -- ["07:00", "17:00"]
);
```

### 3.2 P2 对账单

```sql
CREATE TABLE p2_statement_batches (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  period_start    DATE,
  period_end      DATE,
  status          VARCHAR(16),                         -- draft/sending/sent/closed
  creator_id      BIGINT,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE p2_statements (
  id              BIGSERIAL PRIMARY KEY,
  batch_id        BIGINT,
  tenant_id       BIGINT NOT NULL,
  customer_id     BIGINT NOT NULL,
  opening_balance NUMERIC(18,2),
  closing_balance NUMERIC(18,2),
  total_outbound  NUMERIC(18,2),
  total_invoiced  NUMERIC(18,2),
  total_received  NUMERIC(18,2),
  total_notes     NUMERIC(18,2),
  share_token     VARCHAR(64) UNIQUE,
  status          VARCHAR(16),                         -- draft/sent/viewed/confirmed/disputed
  sent_at         TIMESTAMPTZ,
  viewed_at       TIMESTAMPTZ,
  confirmed_at    TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_statements_tenant_status ON p2_statements(tenant_id, status);
CREATE INDEX idx_statements_share_token ON p2_statements(share_token);

CREATE TABLE p2_statement_items (
  id              BIGSERIAL PRIMARY KEY,
  statement_id    BIGINT NOT NULL,
  biz_type        VARCHAR(16),                         -- outbound/invoice/payment/note/refund
  biz_doc_no      VARCHAR(64),
  biz_date        DATE,
  sku_id          BIGINT,
  quantity        NUMERIC(18,4),
  unit            VARCHAR(8),
  amount          NUMERIC(18,2),
  remark          TEXT
);

CREATE TABLE p2_statement_disputes (
  id              BIGSERIAL PRIMARY KEY,
  statement_id    BIGINT NOT NULL,
  item_ids        JSONB,
  dispute_type    VARCHAR(32),
  reason          TEXT,
  evidence_files  JSONB,
  status          VARCHAR(16),                         -- open/in_progress/resolved
  assigned_to     BIGINT,                              -- 业务员
  replied_by      BIGINT,
  reply           TEXT,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  resolved_at     TIMESTAMPTZ
);

-- 病毒漏斗（核心增长数据）
CREATE TABLE p2_statement_share_logs (
  id              BIGSERIAL PRIMARY KEY,
  statement_id    BIGINT NOT NULL,
  share_channel   VARCHAR(16),                         -- wechat/sms/email
  viewer_openid   VARCHAR(64),
  viewer_ip       INET,
  viewer_device   VARCHAR(64),
  viewed_at       TIMESTAMPTZ DEFAULT NOW(),
  converted_to_register BOOLEAN DEFAULT FALSE,
  converted_user_id BIGINT
);
```

### 3.3 P3 出库

```sql
CREATE TABLE p3_delivery_orders (
  id              BIGSERIAL PRIMARY KEY,
  order_no        VARCHAR(32) UNIQUE NOT NULL,
  tenant_id       BIGINT NOT NULL,
  customer_id     BIGINT NOT NULL,
  contract_id     BIGINT,
  warehouse_id    BIGINT NOT NULL,
  expected_weight NUMERIC(18,2),
  actual_weight   NUMERIC(18,2),
  status          VARCHAR(16),                         -- pending/scanning/weighing/completed/disputed
  created_by      BIGINT,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  completed_at    TIMESTAMPTZ
);
CREATE INDEX idx_delivery_tenant_status ON p3_delivery_orders(tenant_id, status);

CREATE TABLE p3_delivery_items (
  id              BIGSERIAL PRIMARY KEY,
  order_id        BIGINT NOT NULL,
  bundle_code     VARCHAR(32) NOT NULL,
  sku_id          BIGINT,
  weight          NUMERIC(18,4),
  heat_no         VARCHAR(32),
  location_id     BIGINT,
  scanned_at      TIMESTAMPTZ,
  scanned_by      BIGINT,
  is_recommended  BOOLEAN,                             -- 是否在推荐列表
  substitution_reason TEXT                             -- 替代原因
);

CREATE TABLE p3_scale_readings (
  id              BIGSERIAL PRIMARY KEY,
  order_id        BIGINT NOT NULL,
  vehicle_no      VARCHAR(16),
  gross_weight    NUMERIC(18,2),
  tare_weight     NUMERIC(18,2),
  net_weight      NUMERIC(18,2),
  theoretical_weight NUMERIC(18,2),
  deviation_pct   NUMERIC(8,4),
  photos          JSONB,                               -- 4 张照片 URL
  scale_id        VARCHAR(32),
  measured_by     BIGINT,
  measured_at     TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE p3_quality_certificates (
  id              BIGSERIAL PRIMARY KEY,
  order_id        BIGINT NOT NULL,
  pdf_url         TEXT,
  bundle_codes    JSONB,
  heat_nos        JSONB,
  chemical_data   JSONB,
  mechanical_data JSONB,
  generated_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE p3_driver_app (
  id              BIGSERIAL PRIMARY KEY,
  order_id        BIGINT NOT NULL,
  driver_id       BIGINT NOT NULL,                     -- external_users.id
  vehicle_no      VARCHAR(16),
  load_at         TIMESTAMPTZ,
  load_photos     JSONB,
  signature       TEXT,
  delivered_at    TIMESTAMPTZ,
  delivery_photos JSONB
);

CREATE TABLE p3_gps_tracking (
  id              BIGSERIAL,
  order_id        BIGINT,
  driver_id       BIGINT,
  lat             NUMERIC(10,7),
  lng             NUMERIC(10,7),
  speed           NUMERIC(8,2),
  recorded_at     TIMESTAMPTZ,
  PRIMARY KEY (id, recorded_at)
)
PARTITION BY RANGE (recorded_at);

CREATE TABLE p3_client_receipts (
  id              BIGSERIAL PRIMARY KEY,
  order_id        BIGINT UNIQUE NOT NULL,
  signed_at       TIMESTAMPTZ,
  signature       TEXT,
  signer_name     VARCHAR(64),
  dispute_type    VARCHAR(32),
  dispute_evidence JSONB
);

CREATE TABLE p3_delivery_anomalies (
  id              BIGSERIAL PRIMARY KEY,
  order_id        BIGINT NOT NULL,
  type            VARCHAR(32),                         -- wrong_sku/wrong_qty/scale_deviation/...
  severity        VARCHAR(8),                          -- low/medium/high
  detected_at     TIMESTAMPTZ DEFAULT NOW(),
  resolved_at     TIMESTAMPTZ,
  resolved_by     BIGINT
);

CREATE TABLE p3_inventory_diff (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  warehouse_id    BIGINT,
  sku_id          BIGINT,
  period          VARCHAR(8),
  book_qty        NUMERIC(18,2),
  actual_qty      NUMERIC(18,2),
  diff            NUMERIC(18,2),
  trace_data      JSONB,
  reviewed_at     TIMESTAMPTZ,
  reviewed_by     BIGINT
);
```

### 3.4 P4 报价

```sql
CREATE TABLE p4_quote_requests (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  sales_id        BIGINT NOT NULL,
  customer_id     BIGINT NOT NULL,
  sku_id          BIGINT NOT NULL,
  quantity        NUMERIC(18,2),
  input_method    VARCHAR(16),                         -- voice/text/template
  raw_input       TEXT,
  requested_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE p4_quote_suggestions (
  id              BIGSERIAL PRIMARY KEY,
  request_id      BIGINT NOT NULL,
  weighted_cost   NUMERIC(10,2),
  recent_low_price NUMERIC(10,2),
  client_avg_price NUMERIC(10,2),
  market_price    NUMERIC(10,2),
  suggested_min   NUMERIC(10,2),
  suggested_max   NUMERIC(10,2),
  suggested_recommended NUMERIC(10,2),
  generated_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE p4_quotes (
  id              BIGSERIAL PRIMARY KEY,
  request_id      BIGINT,
  quote_no        VARCHAR(32) UNIQUE,
  actual_price    NUMERIC(10,2),
  total_amount    NUMERIC(18,2),
  lock_until      TIMESTAMPTZ,
  share_token     VARCHAR(64) UNIQUE,
  status          VARCHAR(16),                         -- pending_approval/sent/accepted/rejected/expired
  approval_required BOOLEAN,
  approval_status VARCHAR(16),
  approver_id     BIGINT,
  sent_to_client_at TIMESTAMPTZ,
  accepted_at     TIMESTAMPTZ,
  rejected_at     TIMESTAMPTZ,
  rejection_reason VARCHAR(64),
  ai_suggested    BOOLEAN
);

CREATE TABLE p4_quote_approvals (
  id              BIGSERIAL PRIMARY KEY,
  quote_id        BIGINT,
  requested_at    TIMESTAMPTZ,
  approver_id     BIGINT,
  decision        VARCHAR(16),                         -- approve/reject/modify
  decision_at     TIMESTAMPTZ,
  new_price       NUMERIC(10,2),
  reason          TEXT
);

CREATE TABLE p4_sales_behavior (
  id              BIGSERIAL PRIMARY KEY,
  sales_id        BIGINT NOT NULL,
  period          VARCHAR(8),                          -- 2026-04
  margin_avg      NUMERIC(10,2),
  margin_team_avg NUMERIC(10,2),
  client_concentration NUMERIC(8,4),
  anomaly_count   INT,
  risk_score      INT,                                 -- 0-100
  computed_at     TIMESTAMPTZ DEFAULT NOW()
);
```

### 3.5 P5 资金/合同

```sql
CREATE TABLE p5_cash_dashboard (
  tenant_id       BIGINT NOT NULL,
  snapshot_date   DATE NOT NULL,
  bank_total      NUMERIC(18,2),
  bank_breakdown  JSONB,
  receivables_aging JSONB,
  payables        JSONB,
  notes_pool      NUMERIC(18,2),
  notes_aging     JSONB,
  inventory_floating_pl NUMERIC(18,2),
  real_net_asset  NUMERIC(18,2),
  daily_change    NUMERIC(18,2),
  PRIMARY KEY (tenant_id, snapshot_date)
);

CREATE TABLE p5_contracts (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  contract_no     VARCHAR(32) UNIQUE,
  type            VARCHAR(32),                         -- agency/spot/exchange/processing/group_buy
  party_a_id      BIGINT,
  party_b_id      BIGINT,
  total_amount    NUMERIC(18,2),
  template_id     BIGINT,
  content         TEXT,                                -- 合同正文
  ai_audit_result JSONB,
  signed_at       TIMESTAMPTZ,
  sign_method     VARCHAR(32),                         -- esign/fadada
  status          VARCHAR(16),
  related_documents JSONB,                             -- 关联出库单/磅单等
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE p5_notes (
  id              BIGSERIAL PRIMARY KEY,
  owner_tenant_id BIGINT NOT NULL,
  note_no         VARCHAR(32),
  face_amount     NUMERIC(18,2),
  due_date        DATE,
  issuer_bank     VARCHAR(64),
  credit_grade    VARCHAR(16),                         -- aa/a/bbb/...
  status          VARCHAR(16),                         -- held/discounted/endorsed/expired
  source_payment_id BIGINT,
  discount_at     TIMESTAMPTZ,
  discount_rate   NUMERIC(8,4),
  endorsed_to     VARCHAR(128),
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE p5_receivables (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  customer_id     BIGINT NOT NULL,
  source_doc_id   BIGINT,                              -- 关联出库/合同
  amount          NUMERIC(18,2),
  due_date        DATE,
  status          VARCHAR(16),                         -- pending/received/overdue/written_off
  aging_bucket    VARCHAR(16),                         -- 0-30/30-60/60-90/90+
  overdue_days    INT,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_receivables_tenant_aging ON p5_receivables(tenant_id, aging_bucket);

CREATE TABLE p5_alerts (
  id              BIGSERIAL PRIMARY KEY,
  tenant_id       BIGINT NOT NULL,
  type            VARCHAR(32),                         -- aging/payable_due/note_discount_opp/...
  severity        VARCHAR(8),
  target_object_type VARCHAR(32),
  target_object_id BIGINT,
  message         TEXT,
  action_options  JSONB,
  triggered_at    TIMESTAMPTZ DEFAULT NOW(),
  dismissed_at    TIMESTAMPTZ,
  dismissed_by    BIGINT
);
```

### 3.6 P6 社区

```sql
CREATE TABLE p6_channels (
  id              BIGSERIAL PRIMARY KEY,
  code            VARCHAR(32) UNIQUE,
  name            VARCHAR(64),
  type            VARCHAR(16),                         -- material/region/topic/special
  description     TEXT,
  member_count    INT DEFAULT 0,
  is_certified_only BOOLEAN DEFAULT FALSE,
  is_paid         BOOLEAN DEFAULT FALSE,
  status          VARCHAR(16) DEFAULT 'active'
);

CREATE TABLE p6_posts (
  id              BIGSERIAL PRIMARY KEY,
  channel_id      BIGINT,
  author_user_id  BIGINT NOT NULL,
  tenant_id       BIGINT,
  title           VARCHAR(256),
  content         TEXT,
  images          JSONB,
  videos          JSONB,
  visibility      VARCHAR(16),                         -- public/certified/circle
  status          VARCHAR(16),                         -- active/审核中/被屏蔽
  like_count      INT DEFAULT 0,
  comment_count   INT DEFAULT 0,
  view_count      INT DEFAULT 0,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_posts_channel ON p6_posts(channel_id, created_at DESC);

CREATE TABLE p6_comments (
  id              BIGSERIAL PRIMARY KEY,
  post_id         BIGINT NOT NULL,
  user_id         BIGINT NOT NULL,
  parent_comment_id BIGINT,
  content         TEXT,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE p6_user_certifications (
  user_id         BIGINT PRIMARY KEY,
  cert_type       VARCHAR(32),
  tier            VARCHAR(32),                         -- normal/老板/总代/KOL
  tags            JSONB,
  certified_at    TIMESTAMPTZ
);

CREATE TABLE p6_old_debtor_board (
  id              BIGSERIAL PRIMARY KEY,
  period          VARCHAR(8),                          -- 2026-04
  debtor_company_name VARCHAR(128),
  debtor_usc_code VARCHAR(32),
  total_debt      NUMERIC(18,2),
  affected_traders_count INT,
  avg_overdue_days INT,
  level           VARCHAR(16),                         -- l1/l2/l3
  evidence        JSONB,
  status          VARCHAR(16),                         -- published/appealing/removed
  reviewer_id     BIGINT,
  published_at    TIMESTAMPTZ
);
```

---

## 四、数据中台表 DDL（节选关键）

```sql
-- 原始事件流（实际进 Kafka，落湖到 Iceberg，这里仅是元数据）

-- 行情聚合（DWS 层，进 ClickHouse）
CREATE TABLE data_platform.dws_price_index_daily (
  index_date      DATE,
  level           VARCHAR(16),                         -- master/sub
  sub_type        VARCHAR(64),                         -- screw_steel/...
  region          VARCHAR(32),
  index_value     NUMERIC(10,4),
  change_value    NUMERIC(10,4),
  change_pct      NUMERIC(10,4),
  components      JSONB
);

-- CSC-V0 信用评分
CREATE TABLE data_platform.ads_csc_v0 (
  customer_usc_code VARCHAR(32) PRIMARY KEY,
  computed_at     TIMESTAMPTZ,
  payment_score   INT,
  default_score   INT,
  cooperation_score INT,
  legal_score     INT,
  stability_score INT,
  total_score     INT,
  risk_level      VARCHAR(8),
  explanation     JSONB                                -- 评分明细
);

-- 客户付款行为
CREATE TABLE data_platform.dws_customer_payment (
  customer_usc_code VARCHAR(32),
  period          VARCHAR(8),
  total_invoiced  NUMERIC(18,2),
  total_received  NUMERIC(18,2),
  on_time_count   INT,
  overdue_count   INT,
  avg_overdue_days NUMERIC(10,2),
  payment_score   INT,
  PRIMARY KEY (customer_usc_code, period)
);
```

---

## 五、API OpenAPI 规范（节选关键 API）

> 完整 API 列表见 34 号文档。本文给出关键 API 的 OpenAPI 3.0 风格规范。

### 5.1 通用约定

```yaml
openapi: 3.0.3
info:
  title: 钢链通 API
  version: v1
  description: 6 大 SaaS + 数据中台

servers:
  - url: https://api.gangliantong.com/v1

security:
  - BearerAuth: []

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  parameters:
    TenantId:
      name: X-Tenant-Id
      in: header
      required: true
      schema:
        type: string
    RequestId:
      name: X-Request-Id
      in: header
      required: false
      schema:
        type: string

  schemas:
    BaseResponse:
      type: object
      required: [code, message]
      properties:
        code:
          type: integer
          description: 0=成功，非0=错误码
        message:
          type: string
        data:
          type: object
        request_id:
          type: string
        ts:
          type: integer

    ErrorResponse:
      allOf:
        - $ref: '#/components/schemas/BaseResponse'
```

### 5.2 用户登录 API

```yaml
paths:
  /auth/login:
    post:
      summary: 手机号+验证码登录
      tags: [Auth]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [mobile, code]
              properties:
                mobile:
                  type: string
                  pattern: '^1[3-9]\d{9}$'
                code:
                  type: string
                  pattern: '^\d{6}$'
                tenant_code:
                  type: string
                  description: 多租户切换时使用
      responses:
        '200':
          description: 登录成功
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/BaseResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          token:
                            type: string
                          user_id:
                            type: integer
                          tenants:
                            type: array
                            items:
                              type: object

  /auth/wechat-login:
    post:
      summary: 微信登录（小程序）
      tags: [Auth]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [code]
              properties:
                code:
                  type: string
                  description: wx.login() 拿到的 code
      responses:
        '200':
          description: 登录成功（同上）
```

### 5.3 钢价行情 API

```yaml
  /price/quote/realtime:
    get:
      summary: 实时行情
      tags: [Price]
      parameters:
        - name: sku_id
          in: query
          required: true
          schema:
            type: integer
        - name: region
          in: query
          schema:
            type: string
      responses:
        '200':
          description: 实时价格 + 5min 序列
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/BaseResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          current_price:
                            type: number
                          change:
                            type: number
                          change_pct:
                            type: number
                          quoted_at:
                            type: string
                            format: date-time
                          series_5min:
                            type: array
                            items:
                              type: object
                              properties:
                                ts:
                                  type: string
                                price:
                                  type: number

  /price/index/csp:
    get:
      summary: CSP 综合指数
      tags: [Price]
      parameters:
        - name: level
          in: query
          schema:
            type: string
            enum: [master, sub]
        - name: sub_type
          in: query
          schema:
            type: string
        - name: region
          in: query
          schema:
            type: string
        - name: date
          in: query
          schema:
            type: string
            format: date
      responses:
        '200':
          description: CSP 指数
```

### 5.4 电子对账单 API

```yaml
  /statements/batches:
    post:
      summary: 创建对账批次
      tags: [Statement]
      parameters:
        - $ref: '#/components/parameters/TenantId'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [period_start, period_end, customer_filter]
              properties:
                period_start:
                  type: string
                  format: date
                period_end:
                  type: string
                  format: date
                customer_filter:
                  type: object
                  description: 客户范围过滤
                  properties:
                    type:
                      type: string
                      enum: [all, group, ids]
                    group:
                      type: string
                    ids:
                      type: array
                      items:
                        type: integer
      responses:
        '200':
          description: 创建成功
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/BaseResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          batch_id:
                            type: integer
                          statement_count:
                            type: integer

  /statements/{id}/send:
    post:
      summary: 发送对账单
      tags: [Statement]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [channels]
              properties:
                channels:
                  type: array
                  items:
                    type: string
                    enum: [wechat, sms, email]
      responses:
        '200':
          description: 发送成功

  /statements/share/{token}/confirm:
    post:
      summary: 客户在 H5 确认（无需登录）
      tags: [Statement Public]
      security: []
      parameters:
        - name: token
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [signature, openid]
              properties:
                signature:
                  type: string
                openid:
                  type: string
      responses:
        '200':
          description: 确认成功
```

### 5.5 扫码出库 API

```yaml
  /delivery/orders:
    post:
      summary: 创建出库单
      tags: [Delivery]
      parameters:
        - $ref: '#/components/parameters/TenantId'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [customer_id, sku_list, expected_weight, warehouse_id]
              properties:
                customer_id:
                  type: integer
                sku_list:
                  type: array
                  items:
                    type: object
                    required: [sku_id, quantity, unit]
                    properties:
                      sku_id:
                        type: integer
                      quantity:
                        type: number
                      unit:
                        type: string
                expected_weight:
                  type: number
                warehouse_id:
                  type: integer
                client_preferences:
                  type: object
                  properties:
                    mill_codes:
                      type: array
                      items:
                        type: string
                    heat_no_pattern:
                      type: string
      responses:
        '200':
          description: 创建成功

  /delivery/orders/{id}/recommend:
    post:
      summary: 智能推荐捆号
      tags: [Delivery]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: 推荐结果
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/BaseResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          recommendations:
                            type: array
                            items:
                              type: object
                              properties:
                                bundle_code:
                                  type: string
                                weight:
                                  type: number
                                heat_no:
                                  type: string
                                location_code:
                                  type: string
                                score:
                                  type: number
                                reason:
                                  type: string

  /delivery/orders/{id}/scan-item:
    post:
      summary: 扫码捆号
      tags: [Delivery]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [bundle_code]
              properties:
                bundle_code:
                  type: string
                substitution_reason:
                  type: string
                  description: 如果扫的不是推荐的，必填
      responses:
        '200':
          description: 扫码记录
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/BaseResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          is_recommended:
                            type: boolean
                          progress:
                            type: object
                            properties:
                              scanned_count:
                                type: integer
                              total_count:
                                type: integer
```

### 5.6 AI 报价 API

```yaml
  /quote/requests:
    post:
      summary: 发起报价请求
      tags: [Quote]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [customer_id, sku_id, quantity]
              properties:
                customer_id:
                  type: integer
                sku_id:
                  type: integer
                quantity:
                  type: number
                input_method:
                  type: string
                  enum: [voice, text, template]
                raw_input:
                  type: string
      responses:
        '200':
          description: 创建成功
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/BaseResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          request_id:
                            type: integer
                          suggestion:
                            type: object
                            properties:
                              weighted_cost:
                                type: number
                              recent_low_price:
                                type: number
                              client_avg_price:
                                type: number
                              market_price:
                                type: number
                              suggested_min:
                                type: number
                              suggested_max:
                                type: number
                              suggested_recommended:
                                type: number
                              client_payment_score:
                                type: integer

  /quote/quotes:
    post:
      summary: 生成报价单
      tags: [Quote]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [request_id, actual_price, lock_minutes]
              properties:
                request_id:
                  type: integer
                actual_price:
                  type: number
                lock_minutes:
                  type: integer
                  enum: [30, 60, 1440]
                explanation:
                  type: string
                  description: 异常报价时业务员解释
      responses:
        '200':
          description: 报价单创建成功
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/BaseResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          quote_id:
                            type: integer
                          quote_no:
                            type: string
                          share_token:
                            type: string
                          approval_required:
                            type: boolean
                          h5_url:
                            type: string
```

### 5.7 资金大盘 API

```yaml
  /finance/dashboard:
    get:
      summary: 资金大盘
      tags: [Finance]
      parameters:
        - $ref: '#/components/parameters/TenantId'
        - name: snapshot_date
          in: query
          schema:
            type: string
            format: date
      responses:
        '200':
          description: 资金大盘数据
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/BaseResponse'
                  - type: object
                    properties:
                      data:
                        type: object
                        properties:
                          real_net_asset:
                            type: number
                          daily_change:
                            type: number
                          monthly_real_profit:
                            type: number
                          bank:
                            type: object
                          receivables:
                            type: object
                          payables:
                            type: object
                          notes_pool:
                            type: object
                          inventory_floating_pl:
                            type: number
                          alerts:
                            type: array
                            items:
                              type: object
```

### 5.8 内部数据中台 API（仅 SaaS 间使用）

```yaml
  /internal/data/v1/customer/score/risk:
    get:
      summary: 客户风险评分
      tags: [Internal Data]
      parameters:
        - name: X-Service-Name
          in: header
          required: true
          schema:
            type: string
        - name: customer_usc_code
          in: query
          required: true
          schema:
            type: string
      responses:
        '200':
          description: 风险评分
          content:
            application/json:
              schema:
                type: object
                properties:
                  total_score:
                    type: integer
                  risk_level:
                    type: string
                    enum: [A, B, C, D, E]
                  details:
                    type: object
                    properties:
                      payment_score:
                        type: integer
                      default_score:
                        type: integer
                      cooperation_score:
                        type: integer
                      legal_score:
                        type: integer
                      stability_score:
                        type: integer
                  explanation:
                    type: string
                  computed_at:
                    type: string
                    format: date-time
```

---

## 六、错误码表

```
0       成功

1xxxx   认证授权
10001   未登录
10002   token 过期
10003   权限不足
10004   租户不匹配

2xxxx   参数与资源
20001   参数错误
20002   资源不存在
20003   资源已删除
20004   状态不允许此操作

3xxxx   业务错误（按服务分段）
30xxx   user-service
31xxx   master-data-service
32xxx   price-service (P1)
33xxx   statement-service (P2)
34xxx   delivery-service (P3)
35xxx   quote-service (P4)
36xxx   finance-service (P5)
37xxx   community-service (P6)
38xxx   data-platform-service

4xxxx   外部依赖
40001   钢联 API 不可用
40002   微信 API 不可用
40003   支付通道不可用
40004   电子签章不可用

5xxxx   服务器错误
50001   内部错误
50002   数据库错误
50003   消息队列错误
```

---

## 七、关键实现注意事项

### 7.1 多租户隔离

```
所有业务表查询必须带 tenant_id:

GOOD:
  SELECT * FROM p2_statements 
  WHERE tenant_id = ? AND status = 'sent'

BAD:
  SELECT * FROM p2_statements WHERE status = 'sent'

实现方式:
  1. ORM 中间件自动注入 tenant_id（如 MyBatis Interceptor）
  2. PostgreSQL Row Level Security (RLS) 双重保障
  3. 单元测试覆盖跨租户访问场景
```

### 7.2 客户主权检查

```
在以下场景强制检查:
  - 创建报价 (P4)
  - 平台主动推荐资源 (M7+ 资源圈)
  - 平台撮合询价 (M7+)

伪代码:
  function check_sovereignty(target_customer, current_tenant):
    customer = get_customer(target_customer)
    if customer.main_supplier_until > now():
      if customer.main_supplier_tenant_id != current_tenant:
        # 客户主权期内, 当前租户不是主权方
        if customer.client_choice_unbind_at:
          # 客户主动解绑过, 允许
          return ALLOW
        return DENY  # 屏蔽
    return ALLOW  # 主权期外或主权方自身
```

### 7.3 数据脱敏

```
对内部数据 alpha API 输出做脱敏:

涉及单家企业明细的字段:
  - 企业名 → 哈希
  - 金额 → 区间映射 (如 ¥56.8万 → "10-100 万")
  - 地址 → 区域级 (如 上海闵行 → 华东)

K-匿名校验:
  - 任何 query 结果如果只有 1-9 条记录 → 拒绝返回
  - 必须 ≥ K=10 条聚合后才返回
```

### 7.4 幂等性

```
关键 POST 接口必须幂等:
  - 创建合同
  - 创建出库单
  - 触发支付
  - 提交订单

实现:
  - 客户端必须传 X-Idempotency-Key Header
  - 服务端 Redis 存 key 24h, 重复请求直接返回上次结果
```

### 7.5 限流

```
API 网关层限流:
  - 默认 100 QPS / 用户
  - 关键 API（行情/AI 报价）单独配额
  - 公开 H5 API（对账单/合同）按 token 限流

防爬虫:
  - 异常 IP 自动封禁
  - User-Agent 黑名单
  - 行为分析（访问频率、模式）
```

---

## 八、配套阅读

- 26 号：MVP 总体架构
- 27 号：数据中台 MVP
- 28-33 号：6 大 SaaS PRD
- 34 号：MVP 开发文档
- 36 号：关键算法详细设计
- 37 号：UI 视觉设计规范

---

## 九、最终结论

> **本文档是研发工程师按 26-34 号文档实施时的"工程级规范"。**
>
> **使用方式**：
> - 后端工程师按 DDL 建库
> - 接口工程师按 OpenAPI 实现 API
> - QA 按 API 写自动化测试
> - DBA 按规范维护数据库
>
> **下一步**：
> - 36 号文档给出关键算法的详细设计
> - 37 号文档给出 UI 视觉规范
