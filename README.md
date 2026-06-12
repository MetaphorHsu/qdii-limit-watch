# QDII 限购变化观察器｜项目代码结构与架构约束

## 0. 项目性质

本项目不是普通 H5 页面。

本项目本质是：

> QDII 限购当前状态表 + 定投偏差估算器 + 支付宝核对任务单 + 用户反馈校验闭环。

第一版目标不是做完整数据平台，而是在 48 小时内验证：

1. 用户是否愿意输入基金代码和定投金额；
2. 用户是否会被“可能少投金额”刺激；
3. 用户是否愿意跳转支付宝核对；
4. 用户是否愿意回来点“一致 / 不一致 / 找不到入口”。

## 1. 技术选型

```txt
前端框架：Next.js App Router
语言：TypeScript
样式：Tailwind CSS
组件：shadcn/ui 可选，不强依赖
数据库：Supabase PostgreSQL
鉴权：MVP 不做用户账户
部署：Vercel + Supabase
表单校验：Zod
状态管理：不用 Zustand / Redux，MVP 不需要
邮件订阅：MVP 0.2 再做
爬虫 / OCR / API 数据源：MVP 不做
```

## 2. 架构总原则

### 2.1 状态机单点定义

所有可信状态只能在一个文件中定义：

```txt
src/lib/domain/confidence.ts
```

禁止在 React 组件里散落判断：

```tsx
if (confidence_state === 'confirmed') {}
if (confidence_state === 'probable') {}
```

所有页面只能通过 `ConfidencePolicy` 读取策略。

### 2.2 金额计算单点定义

所有金额计算只能在：

```txt
src/lib/domain/calculate.ts
```

禁止在页面组件、API route、数据库查询函数里重复写：

```ts
Math.min(plannedAmount, limitAmount)
```

### 2.3 当前限额表优先

MVP 不从历史事件表推导当前限额。

第一版只使用：

```txt
current_limits
```

原因：48 小时版本里，不要让 AI 写“取最新 effective_date 的事件作为当前限额”这种危险逻辑。

### 2.4 UI 不拥有业务规则

React 组件只负责展示。

组件不允许直接判断：

1. 哪些状态可计算；
2. 哪些状态可提醒；
3. 哪些状态展示金额；
4. 金额怎么算；
5. 偏差等级怎么算。

组件只能消费后端或 domain 层返回的 `ResultViewModel`。

---

# 3. 项目目录结构

```txt
qdii-limit-watch/
├── app/
│   ├── layout.tsx                         # 全局布局
│   ├── page.tsx                           # 首页：基金代码 + 定投金额输入
│   ├── result/
│   │   └── [checkId]/
│   │       └── page.tsx                   # 结果页：展示估算结果 + 核对任务单 + 反馈入口
│   ├── fund/
│   │   └── [fundCode]/
│   │       └── page.tsx                   # 基金详情页：当前限额、可信状态、来源
│   ├── admin/
│   │   └── page.tsx                       # 可选：人工维护 current_limits，MVP 可暂不做
│   └── api/
│       ├── check/
│       │   └── route.ts                   # 创建体检记录，返回 checkId
│       ├── feedback/
│       │   └── route.ts                   # 提交用户反馈
│       └── funds/
│           └── route.ts                   # 搜索基金 / 获取热门基金列表
│
├── src/
│   ├── components/
│   │   ├── home/
│   │   │   ├── FundSearchBox.tsx          # 基金代码搜索框
│   │   │   ├── AmountInput.tsx            # 定投金额输入
│   │   │   └── HotFundList.tsx            # 热门限购基金
│   │   │
│   │   ├── result/
│   │   │   ├── ResultSummaryCard.tsx      # 结果摘要卡片
│   │   │   ├── ShortfallCard.tsx          # 可能少投金额展示
│   │   │   ├── ConfidenceBadge.tsx        # 可信状态标签
│   │   │   ├── VerifyTaskCard.tsx         # 支付宝核对任务单
│   │   │   ├── FeedbackSheet.tsx          # 一致 / 不一致 / 找不到入口
│   │   │   └── DataDegradedCard.tsx       # ambiguous / conflict / stale 降级展示
│   │   │
│   │   ├── fund/
│   │   │   ├── CurrentLimitCard.tsx       # 当前限额卡片
│   │   │   └── SourceInfoCard.tsx         # 来源信息卡片
│   │   │
│   │   └── common/
│   │       ├── Disclaimer.tsx             # 全站免责声明
│   │       ├── PageShell.tsx              # 页面外壳
│   │       └── EmptyState.tsx             # 空状态
│   │
│   ├── lib/
│   │   ├── domain/
│   │   │   ├── confidence.ts              # 可信状态枚举与策略表
│   │   │   ├── calculate.ts               # 定投偏差计算
│   │   │   ├── deviation.ts               # 偏差等级判断
│   │   │   ├── result-state.ts            # 页面结果状态组装
│   │   │   └── copywriting.ts             # 不同状态下的文案模板
│   │   │
│   │   ├── db/
│   │   │   ├── supabase-server.ts         # 服务端 Supabase client
│   │   │   ├── supabase-browser.ts        # 浏览器 Supabase client，尽量少用
│   │   │   ├── fund.queries.ts            # funds 查询
│   │   │   ├── current-limit.queries.ts   # current_limits 查询
│   │   │   ├── check.queries.ts           # check_sessions 写入/读取
│   │   │   └── feedback.queries.ts        # feedback_logs 写入
│   │   │
│   │   ├── schemas/
│   │   │   ├── check.schema.ts            # 创建体检记录的 Zod schema
│   │   │   ├── feedback.schema.ts         # 用户反馈 Zod schema
│   │   │   └── fund.schema.ts             # 基金搜索 Zod schema
│   │   │
│   │   ├── tracking/
│   │   │   └── local-check-session.ts     # localStorage 记录 checkId 与回流状态
│   │   │
│   │   └── utils/
│   │       ├── money.ts                   # 金额格式化
│   │       ├── date.ts                    # 日期格式化
│   │       └── url.ts                     # URL 参数处理
│   │
│   ├── types/
│   │   ├── database.ts                    # Supabase 自动生成或手写 DB 类型
│   │   ├── domain.ts                      # 业务类型
│   │   └── view-model.ts                  # 前端展示模型类型
│   │
│   └── data/
│       └── seed-funds.ts                  # MVP 种子基金数据，可选
│
├── supabase/
│   ├── migrations/
│   │   └── 001_init.sql                   # 初始化数据库表
│   └── seed.sql                           # 8 只种子基金 + 当前限额
│
├── docs/
│   ├── 项目代码结构.md                    # 当前文件
│   ├── 状态机规则.md                      # confirmed/probable/... 的唯一解释
│   ├── 数据库设计.md                      # 表结构与字段解释
│   ├── Cursor开发规则.md                  # 给 AI 的代码生成边界
│   └── MVP开发顺序.md                     # 48 小时开发路线
│
├── .env.local
├── package.json
├── tailwind.config.ts
├── next.config.ts
└── README.md
```

---

# 4. 数据库架构

## 4.1 第一版只建 4 张核心表

MVP 0.1 不做完整历史事件流。

```txt
funds
current_limits
check_sessions
feedback_logs
```

## 4.2 枚举类型

```sql
create type confidence_state as enum (
  'confirmed',
  'probable',
  'ambiguous',
  'conflict',
  'stale'
);

create type feedback_type as enum (
  'matched',
  'mismatched',
  'entry_not_found'
);

create type business_type as enum (
  'subscription',
  'recurring_investment',
  'conversion_in',
  'all',
  'unknown'
);

create type limit_type as enum (
  'single_order',
  'daily_total',
  'single_account',
  'single_channel',
  'unknown'
);

create type channel_scope as enum (
  'all_channels',
  'agency_channels',
  'alipay_unconfirmed',
  'unknown'
);
```

## 4.3 funds：基金基础表

```sql
create table funds (
  id uuid primary key default gen_random_uuid(),
  fund_code text unique not null,
  fund_name text not null,
  fund_company text,
  fund_type text default 'QDII',
  is_qdii boolean default true,
  target_index text,
  theme text,
  share_class text,
  enabled boolean default true,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
```

## 4.4 current_limits：当前限额表

```sql
create table current_limits (
  id uuid primary key default gen_random_uuid(),
  fund_id uuid references funds(id) on delete cascade,
  fund_code text not null,
  limit_amount numeric(12, 2),
  old_limit_amount numeric(12, 2),
  business_type business_type default 'unknown',
  limit_type limit_type default 'unknown',
  currency text default 'CNY',
  channel_scope channel_scope default 'unknown',
  confidence_state confidence_state not null,
  effective_date date,
  source_url text,
  raw_excerpt text,
  manual_verified_at timestamptz,
  updated_at timestamptz default now()
);
```

## 4.5 check_sessions：体检记录表

```sql
create table check_sessions (
  id uuid primary key default gen_random_uuid(),
  fund_id uuid references funds(id),
  current_limit_id uuid references current_limits(id),

  fund_code text not null,
  planned_amount numeric(12, 2) not null,
  frequency text default 'unknown',

  current_limit_amount numeric(12, 2),
  estimated_executable_amount numeric(12, 2),
  estimated_shortfall_amount numeric(12, 2),
  estimated_execution_ratio numeric(8, 4),

  confidence_state confidence_state not null,
  result_mode text not null,
  calculation_version text not null default 'v0.1',

  source_channel text default 'h5',
  created_at timestamptz default now()
);
```

## 4.6 feedback_logs：用户反馈表

```sql
create table feedback_logs (
  id uuid primary key default gen_random_uuid(),
  check_session_id uuid references check_sessions(id) on delete set null,
  fund_id uuid references funds(id),
  current_limit_id uuid references current_limits(id),

  fund_code text not null,
  planned_amount numeric(12, 2),
  feedback_type feedback_type not null,
  actual_payment_amount numeric(12, 2),
  comment text,
  source_page text default 'result',

  created_at timestamptz default now()
);
```

---

# 5. 状态机规则

## 5.1 可信状态定义

```ts
export const ConfidenceStates = [
  'confirmed',
  'probable',
  'ambiguous',
  'conflict',
  'stale',
] as const;

export type ConfidenceState = typeof ConfidenceStates[number];
```

## 5.2 状态策略表

```ts
export const ConfidencePolicy: Record<ConfidenceState, {
  calculable: boolean;
  showAmount: boolean;
  allowReminder: boolean;
  resultMode: 'exact_estimate' | 'weak_estimate' | 'verify_only' | 'conflict' | 'expired';
  primaryCTA: 'verify' | 'show_sources' | 'refresh_or_verify';
}> = {
  confirmed: {
    calculable: true,
    showAmount: true,
    allowReminder: true,
    resultMode: 'exact_estimate',
    primaryCTA: 'verify',
  },
  probable: {
    calculable: true,
    showAmount: true,
    allowReminder: true,
    resultMode: 'weak_estimate',
    primaryCTA: 'verify',
  },
  ambiguous: {
    calculable: false,
    showAmount: false,
    allowReminder: false,
    resultMode: 'verify_only',
    primaryCTA: 'verify',
  },
  conflict: {
    calculable: false,
    showAmount: false,
    allowReminder: false,
    resultMode: 'conflict',
    primaryCTA: 'show_sources',
  },
  stale: {
    calculable: false,
    showAmount: false,
    allowReminder: false,
    resultMode: 'expired',
    primaryCTA: 'refresh_or_verify',
  },
};
```

## 5.3 扩展页面状态

数据库可信状态只描述“已有数据的可信度”。

页面还需要两个状态：

```ts
export type ResultState =
  | ConfidenceState
  | 'untracked'
  | 'input_error';
```

规则：

```ts
if (!fund) return 'input_error';
if (!currentLimit) return 'untracked';
return currentLimit.confidence_state;
```

---

# 6. 金额计算规则

## 6.1 计算函数

```ts
export function calculateShortfall(params: {
  plannedAmount: number;
  limitAmount: number | null;
  confidenceState: ConfidenceState;
}) {
  const policy = ConfidencePolicy[params.confidenceState];

  if (!policy.calculable || params.limitAmount === null) {
    return {
      estimatedExecutableAmount: null,
      estimatedShortfallAmount: null,
      estimatedExecutionRatio: null,
      calculable: false,
    };
  }

  const estimatedExecutableAmount = Math.min(
    params.plannedAmount,
    params.limitAmount
  );

  const estimatedShortfallAmount =
    params.plannedAmount - estimatedExecutableAmount;

  const estimatedExecutionRatio =
    params.plannedAmount === 0
      ? null
      : estimatedExecutableAmount / params.plannedAmount;

  return {
    estimatedExecutableAmount,
    estimatedShortfallAmount,
    estimatedExecutionRatio,
    calculable: true,
  };
}
```

## 6.2 MVP 0.1 只输出单次估算

不要做复杂 7 天 / 30 天日历推算。

第一版文案只允许写：

```txt
按当前公开限额估算，你的单次定投可能少投约 X 元。
如果该限额持续，后续每次定投都可能出现类似偏差。
```

禁止写：

```txt
未来 30 天一定少投 X 元。
```

---

# 7. 页面路由设计

## 7.1 首页 `/`

功能：

1. 展示一句核心判断；
2. 输入基金代码；
3. 输入定投金额；
4. 展示热门限购基金；
5. 点击“检查我的定投”。

提交后调用：

```txt
POST /api/check
```

返回：

```json
{
  "checkId": "uuid"
}
```

然后跳转：

```txt
/result/{checkId}
```

## 7.2 结果页 `/result/[checkId]`

展示：

1. 基金名称；
2. 用户输入金额；
3. 当前公开限额；
4. 可信状态；
5. 可计算时展示可能少投金额；
6. 不可计算时展示降级原因；
7. 支付宝核对任务单；
8. 反馈入口。

## 7.3 基金详情页 `/fund/[fundCode]`

展示：

1. 当前限额；
2. 生效日期；
3. 可信状态；
4. 来源链接；
5. 原文摘录；
6. 最近反馈概况，MVP 可不做。

---

# 8. 支付宝核对任务单链路

## 8.1 点击核对前

用户点击“去支付宝核对”时：

1. localStorage 写入当前 `checkId`；
2. 记录 `verify_started_at`；
3. 复制任务单；
4. 可尝试打开支付宝搜索页，打不开也不影响。

```ts
localStorage.setItem('active_check_id', checkId);
localStorage.setItem('verify_started_at', Date.now().toString());
```

## 8.2 用户切回 H5

监听页面可见性：

```ts
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'visible') {
    const checkId = localStorage.getItem('active_check_id');
    if (checkId) {
      openFeedbackSheet(checkId);
    }
  }
});
```

## 8.3 反馈按钮

只保留三按钮：

```txt
一致
不一致
找不到入口
```

可选填写：

```txt
支付宝实际确认金额
```

---

# 9. API 设计

## 9.1 创建体检记录

```txt
POST /api/check
```

入参：

```json
{
  "fundCode": "160213",
  "plannedAmount": 100,
  "frequency": "daily"
}
```

服务端动作：

1. Zod 校验；
2. 查询 `funds`；
3. 查询 `current_limits`；
4. 根据 `ConfidencePolicy` 判断是否可计算；
5. 调用 `calculateShortfall`；
6. 写入 `check_sessions`；
7. 返回 `checkId`。

## 9.2 提交反馈

```txt
POST /api/feedback
```

入参：

```json
{
  "checkId": "uuid",
  "feedbackType": "matched",
  "actualPaymentAmount": 10,
  "comment": ""
}
```

服务端动作：

1. Zod 校验；
2. 查询 `check_sessions`；
3. 写入 `feedback_logs`；
4. 清理前端 localStorage 由客户端完成。

---

# 10. Cursor 开发规则

## 10.1 禁止事项

Cursor 不允许：

1. 在组件中写金额计算；
2. 在组件中直接解释可信状态；
3. 新增状态枚举但不更新 `ConfidencePolicy`；
4. 从 `limit_events` 推导当前限额；
5. 默认把查不到数据当成 `stale`；
6. 在前端信任用户传来的 `current_limit_amount`；
7. 做登录系统；
8. 做邮箱订阅；
9. 做历史事件库；
10. 做 OCR；
11. 做基金推荐；
12. 做支付宝数据读取。

## 10.2 必须事项

Cursor 每次新增代码必须满足：

1. TypeScript 类型通过；
2. 所有用户输入经过 Zod；
3. 所有金额计算走 `calculateShortfall`；
4. 所有可信状态走 `ConfidencePolicy`；
5. 所有数据库写入在 API route 内完成；
6. 结果页必须支持 `confirmed / probable / ambiguous / conflict / stale / untracked / input_error`；
7. 反馈链路必须支持 localStorage 回流弹窗。

---

# 11. 48 小时开发顺序

## Day 1 上午：项目骨架

1. 初始化 Next.js；
2. 接 Tailwind；
3. 建 Supabase；
4. 写 migration；
5. seed 8 只基金；
6. 写 Supabase server client。

## Day 1 下午：核心 domain

1. `confidence.ts`；
2. `calculate.ts`；
3. `result-state.ts`；
4. Zod schemas；
5. `/api/check`。

## Day 1 晚上：首页 + 结果页

1. 首页输入；
2. 热门基金列表；
3. 创建 check session；
4. 跳转结果页；
5. 结果页读取 check session；
6. 展示金额估算或降级卡片。

## Day 2 上午：核对任务单 + 反馈

1. 任务单组件；
2. 复制任务单；
3. localStorage 写入 checkId；
4. visibilitychange 回流弹窗；
5. `/api/feedback`；
6. 三按钮反馈。

## Day 2 下午：兜底与上线

1. 空状态；
2. 错误状态；
3. 免责声明；
4. 移动端检查；
5. Vercel 部署；
6. Supabase RLS 简化配置；
7. 埋点或简单日志。

---

# 12. 架构验收清单

每次 Cursor 或程序员改代码，都问这 10 个问题：

1. 是否出现了第二套可信状态定义？
2. 是否有组件直接判断 `confirmed / probable`？
3. 是否有组件直接计算少投金额？
4. 是否有接口相信前端传来的限额金额？
5. 是否有 SQL 用 `order by effective_date desc limit 1` 直接取当前限额？
6. 是否保留了 `current_limit_id`，保证历史体检可追溯？
7. 是否支持 `ambiguous / conflict / stale` 的降级页？
8. 查不到基金时是否使用 `untracked`，而不是污染成 `stale`？
9. 点击支付宝核对后，是否写入 localStorage？
10. 用户切回页面时，是否自动弹出反馈卡？

只要有一个答案是否定的，就说明代码已经偏离架构。

---

# 13. MVP 暂不做清单

以下内容不允许进入 48 小时版本：

```txt
App
账户体系
我的定投列表
站内提醒中心
邮箱订阅
限额变化邮件
历史事件库
完整 limit_events
OCR
爬虫
支付宝登录
自动读取交易记录
自动修改定投
基金推荐
B 端 API
复杂后台管理
```

第一版只做一个锋利闭环：

```txt
输入基金代码和金额
→ 读取当前限额
→ 输出单次可能少投
→ 生成支付宝核对任务
→ 用户反馈一致 / 不一致 / 找不到入口
```
