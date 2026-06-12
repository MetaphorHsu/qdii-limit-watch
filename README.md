# QDII 场外基金限额监控与定投偏差预警工具 PRD

## 0. 产品定位

### 0.1 一句话定位

面向场外 QDII 定投用户的**限额变化监控与定投偏差预警工具**。

用户手动添加关注基金和每日定投计划金额。系统静默监控公开限额变化。当限额变化可能影响用户定投执行时，主动提醒用户去支付宝等代销平台核对并自行调整计划。

### 0.2 产品本质

不是基金推荐工具。
不是交易工具。
不是支付宝外挂。
不是收益预测器。

本产品只做四件事：

```txt
用户观察清单
+ 公开限额监控
+ 定投金额偏差判断
+ 跨平台核对提醒
```

### 0.3 核心判断

用户真正要的不是“知道基金限购了”。
用户要的是：

```txt
我设置的每日定投金额，是否还会按原计划执行？
如果限额变了，我是否需要去支付宝手动调整？
```

---

# 1. MVP 功能范围

## 1.1 MVP 必须做

```txt
P0 用户添加 n 个 QDII 基金
P0 用户为每只基金设置每日定投计划金额
P0 系统维护每只基金的当前公开限额
P0 系统记录限额变化事件
P0 系统根据 planned_amount 与 new_limit 触发提醒
P0 手机后台提醒用户去支付宝核对 / 修改定投计划
P0 用户点击提醒进入核对页，反馈一致 / 不一致 / 找不到入口
P0 数据可信状态：confirmed / probable / ambiguous / conflict / stale
```

## 1.2 MVP 坚决不做

```txt
不做支付宝登录
不读取支付宝交易记录
不读取用户真实持仓
不做自动交易
不做自动修改定投
不做基金推荐
不预测收益率
不做重型 App
不做复杂账户体系
不做站内消息中心
不做 OCR
不做完整 B 端数据 API
```

## 1.3 MVP 技术选型

```txt
前端：Next.js App Router
数据库：Supabase PostgreSQL
部署：Vercel
后台轮询：Vercel Cron / Supabase Edge Function
邮件：Resend
身份：手机号 + manage_token
登录：不做密码，不做复杂 Auth
状态校验：Zod + TypeScript union + DB enum
```

---

# 2. 核心用户流程

## 2.1 首次使用

```txt
进入首页
→ 创建观察清单
→ 输入基金代码
→ 输入每日定投计划金额
→ 系统读取当前公开限额
→ 显示是否可能影响定投
→ 加入观察清单
```

## 2.2 日常监控

```txt
后台轮询公开限额数据
→ 发现 current_limit 发生变化
→ 生成 limit_change_event
→ 匹配所有关注该基金的 watchlist_items
→ 执行 alert logic
→ 只对真正影响用户计划的事件发邮件
```

## 2.3 收到提醒后

```txt
用户收到邮件
→ 查看本次变化：old_limit → new_limit
→ 看到对自己 planned_amount 的影响
→ 点击“去支付宝核对”
→ 打开支付宝自行核对 / 修改
→ 回到工具反馈：一致 / 不一致 / 找不到入口
```

---

# 3. 模块一：监控配置流 User Config

## 3.1 设计原则

用户配置成本必须低到 30 秒内完成。

用户只需要提供：

```txt
手机号
基金代码
每日定投计划金额
提醒开关
```

不要求：

```txt
真实姓名
手机号
支付宝账号
持仓截图
交易流水
银行卡
风险测评
```

## 3.2 身份模型

MVP 不做账户密码。

采用：

```txt
email + manage_token
```

系统生成一个不可猜测的管理链接：

```txt
/watchlist/{manage_token}
```

用户通过这个链接管理观察清单。

## 3.3 添加基金流程

### 输入项

```txt
fund_code          基金代码
planned_amount     每日计划定投金额
frequency          默认 daily，MVP 可先固定 daily
notify_enabled     默认 true
```

### 添加后系统动作

```txt
1. 查询 funds 是否存在
2. 查询 current_limits 是否存在
3. 如果存在当前限额，立即计算影响
4. 写入 watchlist_items
5. 保存 last_seen_limit_amount
6. 保存 last_seen_confidence_state
```

### 最小前端交互

```txt
添加基金
→ 输入基金代码或模糊搜索基金名称
→ 自动带出基金名称
→ 输入每日定投金额
→ 展示当前公开限额
→ 展示当前是否受影响
→ 确认加入观察清单
```

## 3.4 观察清单展示

每只基金卡片展示：

```txt
基金名称
基金代码
用户计划：每日 X 元
当前公开限额：Y 元
可信状态
影响判断：可能缩水 / 已恢复 / 暂未影响 / 需人工核对
操作：重新检测 / 去支付宝核对 / 删除
```

## 3.5 关键约束

页面文案只能叫：

```txt
观察清单
定投观察
关注基金
```

禁止叫：

```txt
我的持仓
我的资产
我的真实定投
```

原因：系统不知道用户真实持仓，只知道用户手动配置的观察对象。

---

# 4. 模块二：核心状态机与报警阈值 Alert Logic

## 4.1 核心变量

```txt
Planned_Amount       用户设置的每日计划定投金额
Old_Limit            变化前公开限额
New_Limit            变化后公开限额
Old_State            变化前可信状态
New_State            变化后可信状态
Limit_Status         normal / limited / suspended / resumed / unknown
Confidence_State     confirmed / probable / ambiguous / conflict / stale
```

## 4.2 限额状态定义

```txt
normal       未发现明确限购
limited      存在明确限额
suspended    暂停申购 / 暂停大额申购且限额可视为 0
resumed      恢复申购 / 恢复大额申购
unknown      无法判断
```

## 4.3 可报警状态

只有以下状态允许进入金额型报警：

```txt
confirmed
probable
```

以下状态默认不做金额型报警：

```txt
ambiguous
conflict
stale
```

## 4.4 报警触发规则表

| 规则编号 | 类型       | 条件                                                                 | 系统动作            | 用户价值           |
| ---- | -------- | ------------------------------------------------------------------ | --------------- | -------------- |
| T1   | 防守型缩水提醒  | `New_Limit < Planned_Amount` 且 `Old_Limit >= Planned_Amount`       | 发送强提醒           | 用户原计划可能被动缩水    |
| T2   | 缩水加剧提醒   | `New_Limit < Old_Limit` 且 `New_Limit < Planned_Amount`             | 发送提醒            | 用户少投金额扩大       |
| T3   | 开门迎客提醒   | `Old_Limit == 0` 且 `New_Limit > 0`                                 | 发送提醒            | 基金从暂停 / 0 限额恢复 |
| T4   | 恢复计划提醒   | `Old_Limit < Planned_Amount` 且 `New_Limit >= Planned_Amount`       | 发送强提醒           | 用户可检查是否恢复原定投金额 |
| T5   | 部分恢复提醒   | `New_Limit > Old_Limit` 且 `New_Limit < Planned_Amount`             | 发送弱提醒           | 限额改善但仍低于计划     |
| T6   | 暂停申购提醒   | `New_Limit == 0` 或 `Limit_Status == suspended`                     | 发送强提醒           | 用户定投可能完全无法执行   |
| T7   | 静默屏蔽     | `New_Limit >= Planned_Amount` 且不是从受限状态恢复                           | 不提醒，只记录         | 变化不影响用户        |
| T8   | 大额无关变化屏蔽 | `Old_Limit = 50000` 且 `New_Limit = 10000` 且 `Planned_Amount = 100` | 不提醒，只记录         | 防止噪音           |
| T9   | 状态不可信    | `New_State in ambiguous/conflict/stale`                            | 不发金额提醒，可发低频核对提醒 | 避免误报           |
| T10  | 重复变化     | `dedupe_key` 已存在                                                   | 不提醒             | 防止重复骚扰         |

## 4.5 影响等级

```txt
critical:
  New_Limit == 0
  或 New_Limit <= Planned_Amount * 0.2

high:
  New_Limit < Planned_Amount
  且 New_Limit > Planned_Amount * 0.2

medium:
  New_Limit < Planned_Amount
  但 Old_Limit 也已低于 Planned_Amount，仅属于缩水加剧

low:
  New_Limit >= Planned_Amount
  或限额变化不影响用户计划
```

## 4.6 少投金额计算

```txt
Estimated_Executable = min(Planned_Amount, New_Limit)
Estimated_Shortfall = Planned_Amount - Estimated_Executable
Execution_Ratio = Estimated_Executable / Planned_Amount
```

仅在 `confirmed / probable` 状态下输出具体金额。

## 4.7 静默屏蔽原则

系统不是限额变化广播器。
系统是用户计划影响判断器。

所以：

```txt
限额变化 ≠ 一定提醒
限额变化影响用户计划 = 才提醒
```

示例：

```txt
用户每日计划 100 元
限额 50000 → 10000
不提醒

用户每日计划 100 元
限额 100 → 10
提醒

用户每日计划 100 元
限额 10 → 100
提醒

用户每日计划 100 元
限额 10 → 50
弱提醒

用户每日计划 100 元
限额 0 → 10
提醒，但标注仍低于计划
```

---

# 5. 模块三：数据轮询与降级策略 Data Engine

## 5.1 数据生产原则

MVP 不追求全自动。
先采用：

```txt
人工维护 current_limits
+ 半自动抓取公告
+ 人工确认可信状态
+ 后台定时比对变化
```

原因：基金公告脏、渠道表述混乱、业务范围不稳定。全自动解析会制造误报。

## 5.2 数据层级

```txt
raw_sources
→ parsed_candidates
→ current_limits
→ limit_change_events
→ alert_events
→ notification_logs
```

## 5.3 可信状态定义

| 状态        | 定义                   | 是否计算金额    | 是否触发提醒 |
| --------- | -------------------- | --------- | ------ |
| confirmed | 公告明确限额、业务范围明确、人工校验通过 | 是         | 是      |
| probable  | 第三方或公告语义较明确，但渠道未完全确认 | 是，但必须标注估算 | 是，弱提醒  |
| ambiguous | 语义不清，无法确认是否影响定投      | 否         | 默认不提醒  |
| conflict  | 公告、第三方、用户反馈冲突        | 否         | 不发金额提醒 |
| stale     | 数据过期或抓取失败多次          | 否         | 不提醒    |

## 5.4 ambiguous 降级策略

当系统抓到无法解析的公告：

```txt
不输出具体少投金额
不发强提醒
不生成“限额变化已确认”文案
只进入“需核对”状态
```

可选低频提醒：

```txt
你关注的【基金名称】出现疑似限购变化，但公开资料未明确是否影响支付宝定投。请进入支付宝核对定投计划和实际确认金额。
```

触发限制：

```txt
同一基金 7 天内最多 1 次 ambiguous 提醒
```

## 5.5 conflict 降级策略

当来源冲突：

```txt
不计算
不提醒金额
不写“限额已变为 X”
展示冲突来源
进入人工复核队列
```

用户侧文案：

```txt
你关注的【基金名称】限购信息出现来源冲突。系统暂不输出金额判断，避免误导。请以支付宝页面和基金公告为准。
```

## 5.6 stale 降级策略

当数据超过有效期：

```txt
保留上次记录
不触发提醒
页面标注：数据可能过期
后台进入重新抓取队列
```

有效期建议：

```txt
confirmed: 7 天
probable: 3 天
ambiguous: 1 天
conflict: 1 天
stale: 不可提醒
```

## 5.7 数据变化检测逻辑

后台每次更新 `current_limits` 时：

```txt
1. 读取旧 current_limit
2. 写入新 current_limit
3. 比较关键字段：
   - limit_amount
   - limit_status
   - confidence_state
   - effective_date
4. 若发生有效变化，写入 limit_change_events
5. 扫描 watchlist_items
6. 执行 alert logic
7. 写入 alert_events
8. 去重后写 notification_logs
9. 发送邮件
```

---

# 6. 模块四：闭环引导与防流失 The Action Loop

## 6.1 核心问题

提醒不是终点。
真正的闭环是：

```txt
收到提醒
→ 理解影响
→ 打开支付宝
→ 核对定投计划
→ 修改或不修改
→ 回到工具反馈
```

最大断点是跨 App。

所以提醒必须做到：

```txt
让用户立刻知道自己可能损失的是“计划执行权”，不是收益机会
```

## 6.2 文案设计原则

禁止写：

```txt
建议买入
建议加仓
建议卖出
收益机会
低位布局
```

必须写：

```txt
你的原定投计划可能没有按金额执行
请去支付宝核对定投金额、实际扣款和确认金额
是否修改由你自行决定
```

## 6.3 防守型缩水提醒模板

标题：

```txt
【需核对】你关注的 QDII 限额已下调，可能影响每日定投
```

正文：

```txt
你观察的【{fund_name}】公开限额已由 {old_limit} 元调整为 {new_limit} 元。

你设置的观察计划是：每日 {planned_amount} 元。

按当前公开限额估算，后续每次定投可能只有 {executable_amount} 元进入申购，约 {shortfall_amount} 元可能无法按原计划执行。

请进入支付宝核对：
1. 当前定投计划金额
2. 最近一次实际扣款金额
3. 基金确认金额
4. 是否存在退款或部分失败
5. 是否需要手动调整定投金额

本提醒仅用于定投执行检查，不构成投资建议。
```

按钮：

```txt
去支付宝核对
我已核对，反馈结果
```

## 6.4 进攻型恢复提醒模板

标题：

```txt
【限额恢复】你关注的 QDII 限额已上调，请检查定投计划
```

正文：

```txt
你观察的【{fund_name}】公开限额已由 {old_limit} 元上调至 {new_limit} 元。

你设置的观察计划是：每日 {planned_amount} 元。

如果你此前曾因限额下调手动降低定投金额，现在可能需要进入支付宝核对当前定投设置，确认是否仍符合你的资金计划。

本提醒不构成买入建议，只提醒你检查定投执行设置。
```

按钮：

```txt
去支付宝核对定投设置
```

## 6.5 暂停申购提醒模板

标题：

```txt
【高优先级】你关注的 QDII 可能暂停申购
```

正文：

```txt
你观察的【{fund_name}】公开信息显示当前可能暂停申购或限额为 0。

你设置的观察计划是：每日 {planned_amount} 元。

这意味着后续定投可能无法按原计划进入申购。请进入支付宝核对实际扣款、确认金额和定投状态。
```

## 6.6 不确定性提醒模板

标题：

```txt
【需人工核对】你关注的 QDII 限购信息不明确
```

正文：

```txt
你观察的【{fund_name}】出现疑似限购变化，但公开资料未明确是否影响支付宝定投。

系统暂不输出少投金额，避免误报。

请进入支付宝核对实际页面，并反馈是否一致。
```

## 6.7 回流机制

每封邮件都必须带：

```txt
check_url = /check/{alert_event_id}?token={manage_token}
```

用户点击后进入核对页。

核对页只问三个按钮：

```txt
一致
不一致
找不到入口
```

如果不一致，允许补充：

```txt
支付宝实际显示限额
支付宝实际扣款金额
备注
```

不要问长表单。
长表单会杀死反馈率。

---

# 7. 核心实体数据结构 DDL 友好

## 7.1 Enum

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

create type alert_type as enum (
  'defensive_shortfall',
  'shortfall_worsened',
  'reopened',
  'plan_recovery',
  'partial_recovery',
  'suspended',
  'uncertain_check_required'
);

create type alert_severity as enum (
  'critical',
  'high',
  'medium',
  'low'
);

create type limit_status as enum (
  'normal',
  'limited',
  'suspended',
  'resumed',
  'unknown'
);
```

## 7.2 funds

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

## 7.3 watchlists

```sql
create table watchlists (
  id uuid primary key default gen_random_uuid(),
  email text not null,
  manage_token text unique not null,
  email_verified boolean default false,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
```

## 7.4 watchlist_items

```sql
create table watchlist_items (
  id uuid primary key default gen_random_uuid(),
  watchlist_id uuid references watchlists(id) on delete cascade,
  fund_id uuid references funds(id) on delete cascade,

  fund_code text not null,
  planned_amount numeric(12, 2) not null,
  frequency text not null default 'daily',

  notify_enabled boolean default true,

  last_seen_limit_amount numeric(12, 2),
  last_seen_limit_status limit_status,
  last_seen_confidence_state confidence_state,

  created_at timestamptz default now(),
  updated_at timestamptz default now(),

  unique(watchlist_id, fund_code)
);
```

## 7.5 current_limits

```sql
create table current_limits (
  id uuid primary key default gen_random_uuid(),
  fund_id uuid references funds(id) on delete cascade,
  fund_code text not null,

  limit_amount numeric(12, 2),
  old_limit_amount numeric(12, 2),
  limit_status limit_status default 'unknown',

  business_type text default 'unknown',
  limit_type text default 'unknown',
  currency text default 'CNY',
  channel_scope text default 'unknown',

  confidence_state confidence_state not null,

  effective_date date,
  source_url text,
  raw_excerpt text,
  manual_verified_at timestamptz,

  created_at timestamptz default now(),
  updated_at timestamptz default now(),

  unique(fund_code)
);
```

## 7.6 limit_change_events

```sql
create table limit_change_events (
  id uuid primary key default gen_random_uuid(),

  fund_id uuid references funds(id) on delete cascade,
  fund_code text not null,

  old_limit_amount numeric(12, 2),
  new_limit_amount numeric(12, 2),

  old_limit_status limit_status,
  new_limit_status limit_status,

  old_confidence_state confidence_state,
  new_confidence_state confidence_state,

  effective_date date,
  source_url text,
  raw_excerpt text,

  change_hash text unique not null,

  created_at timestamptz default now()
);
```

## 7.7 alert_events

```sql
create table alert_events (
  id uuid primary key default gen_random_uuid(),

  watchlist_id uuid references watchlists(id) on delete cascade,
  watchlist_item_id uuid references watchlist_items(id) on delete cascade,
  limit_change_event_id uuid references limit_change_events(id) on delete cascade,

  fund_code text not null,

  alert_type alert_type not null,
  severity alert_severity not null,

  planned_amount numeric(12, 2) not null,
  old_limit_amount numeric(12, 2),
  new_limit_amount numeric(12, 2),

  estimated_executable_amount numeric(12, 2),
  estimated_shortfall_amount numeric(12, 2),
  estimated_execution_ratio numeric(8, 4),

  confidence_state confidence_state not null,

  dedupe_key text unique not null,
  created_at timestamptz default now()
);
```

## 7.8 notification_logs

```sql
create table notification_logs (
  id uuid primary key default gen_random_uuid(),

  alert_event_id uuid references alert_events(id) on delete cascade,
  watchlist_id uuid references watchlists(id) on delete cascade,
  watchlist_item_id uuid references watchlist_items(id) on delete cascade,

  email text not null,
  channel text default 'email',

  status text not null default 'pending',
  sent_at timestamptz,
  error_message text,

  dedupe_key text unique not null,
  created_at timestamptz default now()
);
```

## 7.9 feedback_logs

```sql
create table feedback_logs (
  id uuid primary key default gen_random_uuid(),

  alert_event_id uuid references alert_events(id) on delete set null,
  watchlist_id uuid references watchlists(id) on delete set null,
  watchlist_item_id uuid references watchlist_items(id) on delete set null,

  fund_code text not null,
  feedback_type feedback_type not null,

  actual_limit_amount numeric(12, 2),
  actual_payment_amount numeric(12, 2),
  comment text,

  source_page text default 'alert_check_page',
  created_at timestamptz default now()
);
```

---

# 8. 核心报警判断伪代码

```ts
function evaluateAlert(params: {
  plannedAmount: number;
  oldLimit: number | null;
  newLimit: number | null;
  oldStatus: LimitStatus;
  newStatus: LimitStatus;
  confidenceState: ConfidenceState;
}) {
  const {
    plannedAmount,
    oldLimit,
    newLimit,
    oldStatus,
    newStatus,
    confidenceState,
  } = params;

  if (confidenceState === 'conflict') {
    return {
      shouldAlert: false,
      reason: 'conflict_no_amount_alert',
    };
  }

  if (confidenceState === 'stale') {
    return {
      shouldAlert: false,
      reason: 'stale_no_alert',
    };
  }

  if (confidenceState === 'ambiguous') {
    return {
      shouldAlert: true,
      alertType: 'uncertain_check_required',
      severity: 'low',
      calculable: false,
    };
  }

  if (newStatus === 'suspended' || newLimit === 0) {
    return {
      shouldAlert: true,
      alertType: 'suspended',
      severity: 'critical',
      calculable: true,
    };
  }

  if (oldLimit === 0 && newLimit !== null && newLimit > 0) {
    return {
      shouldAlert: true,
      alertType: 'reopened',
      severity: 'high',
      calculable: true,
    };
  }

  if (
    oldLimit !== null &&
    newLimit !== null &&
    oldLimit < plannedAmount &&
    newLimit >= plannedAmount
  ) {
    return {
      shouldAlert: true,
      alertType: 'plan_recovery',
      severity: 'high',
      calculable: true,
    };
  }

  if (
    oldLimit !== null &&
    newLimit !== null &&
    oldLimit >= plannedAmount &&
    newLimit < plannedAmount
  ) {
    return {
      shouldAlert: true,
      alertType: 'defensive_shortfall',
      severity: newLimit <= plannedAmount * 0.2 ? 'critical' : 'high',
      calculable: true,
    };
  }

  if (
    oldLimit !== null &&
    newLimit !== null &&
    newLimit < oldLimit &&
    newLimit < plannedAmount
  ) {
    return {
      shouldAlert: true,
      alertType: 'shortfall_worsened',
      severity: 'medium',
      calculable: true,
    };
  }

  if (
    oldLimit !== null &&
    newLimit !== null &&
    newLimit > oldLimit &&
    newLimit < plannedAmount
  ) {
    return {
      shouldAlert: true,
      alertType: 'partial_recovery',
      severity: 'low',
      calculable: true,
    };
  }

  return {
    shouldAlert: false,
    reason: 'limit_change_not_relevant_to_user_plan',
  };
}
```

---

# 9. 极简交互流程

## 9.1 首页

```txt
标题：
QDII 限购观察清单

副标题：
添加你正在场外定投的 QDII 基金。公开限额变化时，提醒你去支付宝核对定投计划。

主按钮：
添加第一只基金
```

## 9.2 添加基金页

```txt
基金代码 / 名称
每日计划定投金额
开启限额变化提醒
邮箱
提交
```

提交后：

```txt
已加入观察清单
当前公开限额：X 元
你的计划：每日 Y 元
当前判断：暂未影响 / 可能受影响 / 数据需核对
```

## 9.3 观察清单页

```txt
基金 A
每日计划：100 元
当前限额：10 元
状态：可能缩水
按钮：去支付宝核对 / 修改观察金额 / 删除

基金 B
每日计划：100 元
当前限额：10000 元
状态：暂未影响
按钮：重新检测 / 删除
```

## 9.4 邮件提醒落地页

```txt
本次限额变化
基金名称
old_limit → new_limit
你的计划金额
预计可执行金额
预计少投金额
可信状态
去支付宝核对
反馈一致 / 不一致 / 找不到入口
```

---

# 10. Vibe Coding 开发顺序

## Day 1

```txt
1. 初始化 Next.js + Supabase
2. 建表：funds / watchlists / watchlist_items / current_limits
3. 做首页 + 添加基金流程
4. 做观察清单页
5. seed 8 只热门 QDII 基金
```

## Day 2

```txt
1. 建 limit_change_events / alert_events / notification_logs
2. 写 evaluateAlert()
3. 写模拟后台任务：手动触发限额变化
4. 写邮件模板
5. 接 Resend
6. 写提醒落地页 + 反馈按钮
```

## Day 3 可选

```txt
1. 增加 ambiguous / conflict / stale 降级页
2. 增加去重逻辑
3. 增加反馈日志
4. 增加管理链接 token
5. 增加简单后台 current_limits 编辑页
```

---

# 11. 成功指标

## 11.1 行为指标

```txt
访问后创建观察清单比例 ≥ 8%
创建观察清单后添加基金比例 ≥ 60%
单用户平均添加基金数 ≥ 2
添加基金后填写定投金额比例 ≥ 80%
收到提醒后点击落地页比例 ≥ 20%
点击后反馈比例 ≥ 5%
```

## 11.2 数据指标

```txt
热门基金 current_limits 覆盖率 ≥ 80%
confirmed / probable 数据占比 ≥ 70%
误报反馈率 ≤ 10%
重复提醒率 ≤ 1%
conflict 平均处理时长 ≤ 24 小时
```

---

# 12. 最终边界

系统只提醒：

```txt
公开限额变化可能影响你的定投计划执行
```

系统不告诉用户：

```txt
该不该买
买多少
什么时候买
未来收益如何
是否应该加仓
```

最终闭环：

```txt
用户添加 n 个基金
→ 系统监控公开限额
→ 限额变化影响用户计划
→ 系统提醒
→ 用户去支付宝核对 / 修改
→ 用户反馈
→ 数据可信度提升
```
