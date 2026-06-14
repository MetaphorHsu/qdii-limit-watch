# QDII 场外基金限额监控与定投偏差短信预警工具 PRD

## 0. 产品定位

### 0.1 一句话定位

面向场外 QDII 定投用户的**限额变化短信预警工具**。

用户手动添加关注基金、定投周期和单次定投计划金额。系统后台监控公开限额变化。当限额变化影响用户配置的定投计划时，通过短信提醒用户去支付宝等代销平台核对并自行调整定投计划。

### 0.2 产品本质

本产品不是基金推荐工具。
不是交易工具。
不是支付宝外挂。
不是收益预测工具。
不是自动定投修改工具。

本产品只做：

```txt
观察清单
+ 手机号验证
+ 公开限额监控
+ 定投执行偏差判断
+ 短信提醒
+ 核对反馈
```

### 0.3 最小闭环

```txt
用户验证手机号
→ 添加观察基金
→ 系统监控限额变化
→ 只在影响计划时发短信
→ 用户点击短信链接
→ 去支付宝核对
→ 回来反馈
```

### 0.4 核心边界

系统只提醒：

```txt
公开限额变化可能影响你手动配置的定投观察计划。
```

系统不判断：

```txt
该不该买
该买多少
是否加仓
是否卖出
收益是否更高
未来走势如何
```

---

# 1. MVP 功能范围

## 1.1 P0 必须完成

```txt
P0 用户输入手机号
P0 用户获取并验证短信验证码
P0 验证成功后创建观察清单
P0 用户添加 n 个 QDII 基金
P0 用户为每只基金设置定投周期和单次计划金额
P0 系统维护每只基金的当前公开限额 current_limits
P0 系统记录限额变化事件 limit_change_events
P0 系统根据 planned_amount 与 new_limit 判断是否影响计划
P0 只在影响计划时发送短信
P0 短信带提醒详情链接
P0 用户点击短信链接进入提醒详情页
P0 用户去支付宝核对
P0 用户回来反馈：一致 / 不一致 / 找不到入口
P0 用户可关闭单只基金提醒
P0 用户可退订全部短信提醒
P0 系统防重复发送
P0 数据可信状态：confirmed / probable / ambiguous / conflict / stale
```

## 1.2 P0 禁止做

```txt
不接支付宝登录
不读取支付宝交易流水
不读取真实持仓
不自动修改定投
不自动交易
不做基金推荐
不预测收益
不做重型 App
不做复杂账户体系
不做 OCR
不做完整公告爬虫
不做微信通知
不做 Web Push
不做邮件提醒
不做 B 端 API
```

## 1.3 MVP 技术选型

```txt
前端：Next.js App Router
语言：TypeScript
样式：Tailwind CSS
数据库：Supabase PostgreSQL
后台任务：Vercel Cron / Supabase Edge Function
短信：短信服务商抽象层，供应商可后选
身份：手机号 + 短信验证码 OTP + manage_token
状态校验：Zod + TypeScript union + PostgreSQL enum
部署：Vercel + Supabase
```

### 1.4 关键工程原则

```txt
短信发送必须走后端。
短信服务商 SDK 不允许散落在业务代码中。
是否发短信必须由 alert policy 决定。
React 组件不允许直接判断是否发短信。
金额计算不允许写在组件中。
状态机不允许散落在页面中。
```

---

# 2. 核心用户流程

## 2.1 首次创建观察清单

```txt
进入首页
→ 输入手机号
→ 获取短信验证码
→ 输入验证码
→ 验证成功
→ 创建观察清单
→ 生成 manage_token
→ 进入观察清单页
```

### 页面文案

```txt
创建 QDII 限购观察清单
输入手机号，限额变化影响你的配置计划时，我们用短信提醒你去支付宝核对。
```

底部说明：

```txt
系统不会读取你的支付宝账号、真实持仓或交易流水，只记录你手动配置的观察对象。
```

---

## 2.2 添加观察基金

```txt
点击添加基金
→ 输入基金代码 / 模糊搜索基金名称
→ 选择定投周期
→ 输入单次定投计划金额
→ 开启限额变化短信提醒
→ 系统读取当前公开限额
→ 展示当前是否可能受影响
→ 加入观察清单
```

### 字段

```txt
fund_code
fund_name
frequency
planned_amount
notify_enabled
```

### 关键定义

`planned_amount` 指用户配置的**单次定投计划金额**，不是系统读取的真实扣款金额。

例如：

```txt
每周周五定投 100 元
planned_amount = 100
frequency = weekly_friday
```

系统只知道用户配置的观察计划，不知道真实定投是否存在。

---

## 2.3 日常监控

```txt
后台轮询或人工更新 current_limits
→ 发现限额有效变化
→ 生成 limit_change_events
→ 扫描关注该基金的 watchlist_items
→ 对每个用户配置执行 evaluateAlert()
→ 命中短信触发规则
→ 生成 alert_events
→ 生成 notification_logs
→ 去重
→ 发送短信
```

---

## 2.4 用户收到短信后

```txt
用户收到短信
→ 点击短信链接
→ 进入提醒详情页
→ 查看 old_limit → new_limit
→ 查看对自己 planned_amount 的影响
→ 点击“去支付宝核对”
→ 用户自行打开支付宝核对 / 修改
→ 回到提醒详情页
→ 反馈：一致 / 不一致 / 找不到入口
```

---

# 3. 模块一：观察清单与手机号验证

## 3.1 身份模型

MVP 不做账号密码。

采用：

```txt
phone + OTP + manage_token
```

解释：

```txt
phone：接收短信提醒
OTP：一次性验证码，用于证明手机号归属
manage_token：管理观察清单的长 token
```

### 禁止

```txt
不允许未验证手机号创建观察清单
不允许未验证手机号接收提醒
不允许前端直接调用短信发送接口
不允许把完整手机号暴露给前端页面
```

---

## 3.2 手机号验证流程

```txt
用户输入手机号
→ POST /api/sms/send-code
→ 系统校验频率限制
→ 生成 6 位验证码
→ 存储 code_hash
→ 发送短信验证码
→ 用户输入验证码
→ POST /api/sms/verify-code
→ 校验成功
→ 创建或读取 watchlist
→ 返回 manage_token
```

### 验证码规则

```txt
验证码有效期：5 分钟
同一手机号 60 秒内只能发送 1 次
同一手机号 1 小时最多发送 5 次
同一 IP 1 小时最多请求 20 次
验证码最多错误 5 次
验证码不明文入库，只保存 code_hash
验证成功后 consumed_at 写入时间
```

---

## 3.3 观察清单规则

一个手机号对应一个观察清单。

```txt
phone_hash unique
watchlist_id unique
manage_token_hash unique
```

前端 URL：

```txt
/watchlist?token={manage_token}
```

页面只展示脱敏手机号：

```txt
188****4310
```

---

## 3.4 观察基金卡片

每张卡片展示：

```txt
基金名称
基金代码
定投周期
单次计划金额
当前公开限额
可信状态
影响判断
最近提醒时间
操作按钮
```

按钮：

```txt
去支付宝核对
修改计划
关闭提醒
删除
```

状态：

```txt
可能缩水
限额恢复
暂未影响
需核对
数据冲突
数据过期
```

---

# 4. 模块二：限额状态机与报警逻辑

## 4.1 核心变量

```txt
Planned_Amount        用户设置的单次定投计划金额
Old_Limit             变化前公开限额
New_Limit             变化后公开限额
Old_Limit_Status      变化前限额状态
New_Limit_Status      变化后限额状态
Old_Confidence_State  变化前可信状态
New_Confidence_State  变化后可信状态
```

## 4.2 限额状态 limit_status

```txt
normal       未发现明确限购
limited      存在明确限额
suspended    暂停申购 / 限额为 0
resumed      恢复申购 / 恢复大额申购
unknown      无法判断
```

## 4.3 可信状态 confidence_state

```txt
confirmed    公告明确，业务范围明确，人工校验通过
probable     公开资料较清楚，但渠道或业务范围未完全确认
ambiguous    语义不清，无法确认是否影响定投
conflict     公告、第三方、用户反馈冲突
stale        数据过期或抓取失败多次
```

## 4.4 金额型计算准入

只有以下状态允许输出具体金额：

```txt
confirmed
probable
```

以下状态禁止输出具体少投金额：

```txt
ambiguous
conflict
stale
```

---

# 5. 短信触发规则

## 5.1 核心原则

系统不是限额变化广播器。
系统只在“限额变化影响用户配置计划”时打扰用户。

```txt
限额变化 ≠ 发短信
限额变化影响 planned_amount = 才考虑发短信
```

---

## 5.2 可短信提醒 alert_type

P0 只允许 4 类短信：

```txt
defensive_shortfall   防守型缩水
suspended             暂停 / 限额为 0
plan_recovery         恢复到覆盖用户计划
reopened              从 0 恢复可申购
```

P0 默认不发短信：

```txt
partial_recovery
shortfall_worsened 的低优先级重复变化
ambiguous
conflict
stale
limit_change_not_relevant_to_user_plan
```

---

## 5.3 报警触发规则表

| 编号  | 类型     | 条件                                                                 | 是否短信 | 原因                     |
| --- | ------ | ------------------------------------------------------------------ | ---: | ---------------------- |
| T1  | 防守型缩水  | `Old_Limit >= Planned_Amount` 且 `New_Limit < Planned_Amount`       |    是 | 原本可能完整执行，现在可能缩水        |
| T2  | 暂停申购   | `New_Limit == 0` 或 `New_Limit_Status == suspended`                 |    是 | 定投可能完全无法执行             |
| T3  | 恢复计划   | `Old_Limit < Planned_Amount` 且 `New_Limit >= Planned_Amount`       |    是 | 用户可能需要检查是否恢复原计划        |
| T4  | 开门迎客   | `Old_Limit == 0` 且 `New_Limit > 0`                                 |    是 | 从不可申购恢复为可申购            |
| T5  | 部分恢复   | `New_Limit > Old_Limit` 且 `New_Limit < Planned_Amount`             |  默认否 | 改善但仍无法覆盖计划，短信价值弱       |
| T6  | 缩水加剧   | `New_Limit < Old_Limit` 且 `New_Limit < Planned_Amount`             |  条件发 | 仅当距上次提醒超过 7 天或缩水比例显著扩大 |
| T7  | 大额无关变化 | `Old_Limit = 50000` 且 `New_Limit = 10000` 且 `Planned_Amount = 100` |    否 | 不影响用户计划                |
| T8  | 数据不确定  | `confidence_state = ambiguous`                                     |    否 | 避免制造误报                 |
| T9  | 来源冲突   | `confidence_state = conflict`                                      |    否 | 禁止金额提醒                 |
| T10 | 数据过期   | `confidence_state = stale`                                         |    否 | 没资格打扰用户                |
| T11 | 重复提醒   | `dedupe_key` 已存在                                                   |    否 | 防重复骚扰                  |

---

## 5.4 影响等级

```txt
critical:
  New_Limit == 0
  或 New_Limit <= Planned_Amount * 0.2

high:
  New_Limit < Planned_Amount
  且 New_Limit > Planned_Amount * 0.2

medium:
  New_Limit < Planned_Amount
  但该用户之前已处于受限状态

low:
  New_Limit >= Planned_Amount
  或变化不影响用户计划
```

---

## 5.5 少投金额计算

```txt
Estimated_Executable = min(Planned_Amount, New_Limit)
Estimated_Shortfall = Planned_Amount - Estimated_Executable
Execution_Ratio = Estimated_Executable / Planned_Amount
```

限制：

```txt
仅 confirmed / probable 可计算
probable 必须标注“公开限额估算”
ambiguous / conflict / stale 不输出金额
```

---

# 6. 防重复发送与防打扰规则

## 6.1 去重规则

每个提醒事件生成 dedupe_key：

```txt
watchlist_item_id + limit_change_event_id + alert_type
```

如果 `notification_logs.dedupe_key` 已存在：

```txt
不再发送短信
```

---

## 6.2 频率限制

```txt
同一 watchlist_item 同一 limit_change_event 只发 1 次
同一手机号每天最多 3 条短信
同一基金同一手机号 24 小时内最多 1 条短信
同一手机号 7 天内 ambiguous 不发短信
用户关闭 notify_enabled 后不发
用户退订后不发
confidence_state = conflict 不发
confidence_state = stale 不发
```

---

## 6.3 退订机制

用户必须可以退订。

退订入口：

```txt
短信链接中的退订入口
提醒详情页底部退订按钮
观察清单页关闭全部短信提醒
```

退订后：

```txt
watchlists.sms_unsubscribed_at 写入时间
watchlist_items.notify_enabled 全部置 false
后续不再发送短信
```

退订页文案：

```txt
你已关闭 QDII 限额变化短信提醒。
系统不会再向该手机号发送观察基金提醒。
你仍可通过管理链接查看或删除观察清单。
```

---

# 7. 数据轮询与降级策略

## 7.1 数据生产策略

MVP 不追求全自动公告解析。

采用：

```txt
人工维护 current_limits
+ 半自动抓取公告
+ 人工确认可信状态
+ 后台比对变化
```

原因：

```txt
基金公告语义脏
渠道范围不清
定投是否覆盖经常不明
全自动解析容易误报
短信误报成本高
```

---

## 7.2 数据流

```txt
raw_sources
→ parsed_candidates
→ current_limits
→ limit_change_events
→ alert_events
→ notification_logs
→ feedback_logs
```

---

## 7.3 current_limits

`current_limits` 是当前公开限额的唯一来源。

禁止：

```txt
从 limit_change_events 反推当前限额
从最近一条公告直接推当前限额
在前端计算当前限额
```

---

## 7.4 limit_change_events

`limit_change_events` 只记录变化历史。

当以下字段变化时，生成事件：

```txt
limit_amount
limit_status
confidence_state
effective_date
source_url
```

但并非所有变化都会触发短信。

---

## 7.5 降级策略

### ambiguous

```txt
不输出少投金额
不发短信
页面提示：公开信息存在疑似变化，但无法确认是否影响定投
进入人工复核队列
```

### conflict

```txt
不输出少投金额
不发短信
展示冲突来源
进入人工复核队列
```

### stale

```txt
不输出少投金额
不发短信
保留上次记录
页面标注：数据可能过期
```

---

# 8. 短信文案

## 8.1 文案原则

禁止出现：

```txt
建议买入
建议加仓
建议卖出
收益机会
抄底
推荐
配置建议
```

允许出现：

```txt
公开限额变化
你配置的观察计划
可能影响定投执行
请去支付宝核对
是否修改由你自行决定
```

---

## 8.2 防守型缩水短信

```txt
【QDII限额提醒】你观察的{基金简称}公开限额由{old_limit}元降至{new_limit}元，你配置的定投为{planned_amount}元，后续可能每次少投约{shortfall_amount}元。请去支付宝核对：{link}
```

---

## 8.3 暂停申购短信

```txt
【QDII限额提醒】你观察的{基金简称}公开信息显示可能暂停申购或限额为0，你配置的定投计划可能无法执行。请去支付宝核对：{link}
```

---

## 8.4 限额恢复短信

```txt
【QDII限额提醒】你观察的{基金简称}公开限额由{old_limit}元恢复至{new_limit}元。如你此前降低过定投金额，请核对是否需要调回原计划：{link}
```

---

## 8.5 从 0 恢复短信

```txt
【QDII限额提醒】你观察的{基金简称}公开限额已由0元恢复至{new_limit}元。请进入支付宝核对定投设置是否仍符合你的计划：{link}
```

---

# 9. 提醒详情页 Action Loop

## 9.1 页面目标

短信只能提供入口。
完整解释必须放在提醒详情页。

提醒详情页必须回答：

```txt
发生了什么
对我的计划有什么影响
我现在该去哪里核对
我核对后怎么反馈
```

---

## 9.2 页面结构

```txt
顶部：限额变化提醒
卡片 1：基金信息
卡片 2：old_limit → new_limit
卡片 3：对观察计划的影响
卡片 4：支付宝核对清单
按钮：去支付宝核对
按钮：复制核对清单
反馈区：一致 / 不一致 / 找不到入口
底部：免责声明 + 关闭提醒 / 退订入口
```

---

## 9.3 支付宝核对清单

```txt
1. 当前定投计划金额
2. 最近一次实际扣款金额
3. 基金确认金额
4. 是否存在退款或部分失败
5. 是否需要手动调整定投金额
```

---

## 9.4 反馈选项

默认只展示三个按钮：

```txt
一致
不一致
找不到入口
```

点击“不一致”后可选填写：

```txt
支付宝实际显示限额
支付宝实际扣款金额
备注
```

禁止强制长表单。

---

# 10. 核心数据结构 DDL

## 10.1 Enum

```sql
create type confidence_state as enum (
  'confirmed',
  'probable',
  'ambiguous',
  'conflict',
  'stale'
);

create type limit_status as enum (
  'normal',
  'limited',
  'suspended',
  'resumed',
  'unknown'
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

create type feedback_type as enum (
  'matched',
  'mismatched',
  'entry_not_found'
);

create type notification_status as enum (
  'pending',
  'sent',
  'failed',
  'skipped',
  'deduped'
);
```

---

## 10.2 funds

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

---

## 10.3 watchlists

```sql
create table watchlists (
  id uuid primary key default gen_random_uuid(),

  phone_hash text unique not null,
  phone_e164 text not null,
  phone_masked text not null,

  phone_verified_at timestamptz,
  manage_token_hash text unique not null,

  sms_opt_in_at timestamptz,
  sms_unsubscribed_at timestamptz,

  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
```

说明：

```txt
phone_e164 仅服务端使用
phone_hash 用于唯一性判断
phone_masked 用于前端展示
manage_token_hash 不存明文 token
```

---

## 10.4 sms_verification_codes

```sql
create table sms_verification_codes (
  id uuid primary key default gen_random_uuid(),

  phone_hash text not null,
  phone_e164 text not null,
  code_hash text not null,

  purpose text not null default 'create_watchlist',

  expires_at timestamptz not null,
  consumed_at timestamptz,

  failed_attempts integer default 0,
  request_ip text,

  created_at timestamptz default now()
);
```

---

## 10.5 watchlist_items

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

---

## 10.6 current_limits

```sql
create table current_limits (
  id uuid primary key default gen_random_uuid(),

  fund_id uuid references funds(id) on delete cascade,
  fund_code text unique not null,

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
  updated_at timestamptz default now()
);
```

---

## 10.7 limit_change_events

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

---

## 10.8 alert_events

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

  action_token_hash text unique not null,
  dedupe_key text unique not null,

  created_at timestamptz default now()
);
```

---

## 10.9 notification_logs

```sql
create table notification_logs (
  id uuid primary key default gen_random_uuid(),

  alert_event_id uuid references alert_events(id) on delete cascade,
  watchlist_id uuid references watchlists(id) on delete cascade,
  watchlist_item_id uuid references watchlist_items(id) on delete cascade,

  phone_hash text not null,
  phone_masked text not null,

  channel text default 'sms',
  sms_provider text,
  sms_template_id text,
  sms_request_id text,

  status notification_status default 'pending',
  sent_at timestamptz,
  error_message text,

  dedupe_key text unique not null,

  created_at timestamptz default now()
);
```

---

## 10.10 feedback_logs

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

  source_page text default 'alert_detail_page',

  created_at timestamptz default now()
);
```

---

# 11. 核心函数

## 11.1 evaluateAlert()

唯一报警判断入口。

```ts
type EvaluateAlertInput = {
  plannedAmount: number;
  oldLimit: number | null;
  newLimit: number | null;
  oldStatus: LimitStatus;
  newStatus: LimitStatus;
  confidenceState: ConfidenceState;
};

type EvaluateAlertOutput = {
  shouldCreateAlert: boolean;
  shouldSendSms: boolean;
  alertType?: AlertType;
  severity?: AlertSeverity;
  calculable: boolean;
  reason?: string;
};
```

规则：

```txt
React 组件不得直接判断是否提醒
API route 不得绕过 evaluateAlert()
后台任务必须统一调用 evaluateAlert()
```

---

## 11.2 calculateShortfall()

```ts
function calculateShortfall(plannedAmount, newLimit, confidenceState) {
  if confidenceState not in confirmed/probable:
    return null;

  estimatedExecutable = min(plannedAmount, newLimit);
  estimatedShortfall = plannedAmount - estimatedExecutable;
  executionRatio = estimatedExecutable / plannedAmount;

  return result;
}
```

---

## 11.3 shouldSendSms()

```ts
function shouldSendSms(alert, watchlist, watchlistItem, notificationHistory) {
  if watchlist.sms_unsubscribed_at exists: return false;
  if watchlistItem.notify_enabled === false: return false;
  if alert.confidence_state in ambiguous/conflict/stale: return false;
  if alert.alert_type not in allowedSmsAlertTypes: return false;
  if dedupe_key exists: return false;
  if phone daily count >= 3: return false;

  return true;
}
```

---

# 12. API 设计

## 12.1 POST /api/sms/send-code

用途：发送验证码。

入参：

```json
{
  "phone": "18810244310"
}
```

服务端动作：

```txt
校验手机号格式
检查发送频率
生成验证码
写入 sms_verification_codes
调用短信 provider
返回成功
```

禁止：

```txt
不返回验证码
不允许前端指定短信模板
不允许绕过限流
```

---

## 12.2 POST /api/sms/verify-code

用途：验证手机号并创建观察清单。

入参：

```json
{
  "phone": "18810244310",
  "code": "123456"
}
```

服务端动作：

```txt
校验 code_hash
检查 expires_at
检查 consumed_at
验证成功后写 consumed_at
创建或读取 watchlist
生成 manage_token
写入 sms_opt_in_at
返回 manage_token
```

---

## 12.3 GET /api/watchlist

用途：读取观察清单。

入参：

```txt
token
```

返回：

```txt
watchlist
watchlist_items
current_limits
alert summary
```

---

## 12.4 POST /api/watchlist/items

用途：添加观察基金。

入参：

```json
{
  "token": "manage_token",
  "fundCode": "019172",
  "plannedAmount": 100,
  "frequency": "weekly_friday",
  "notifyEnabled": true
}
```

服务端动作：

```txt
校验 token
查询 fund
查询 current_limit
写入 watchlist_items
保存 last_seen_limit_amount
返回当前影响判断
```

---

## 12.5 PATCH /api/watchlist/items/:id

用途：修改观察计划。

允许改：

```txt
planned_amount
frequency
notify_enabled
```

---

## 12.6 DELETE /api/watchlist/items/:id

用途：删除观察基金。

---

## 12.7 POST /api/admin/current-limit/update

用途：后台更新当前限额。

服务端动作：

```txt
读取旧 current_limit
写入新 current_limit
生成 limit_change_event
触发 alert evaluation job
```

仅管理员可用。

---

## 12.8 POST /api/jobs/evaluate-alerts

用途：后台任务扫描变化并生成提醒。

服务端动作：

```txt
读取未处理 limit_change_events
查询对应 watchlist_items
调用 evaluateAlert()
生成 alert_events
调用 shouldSendSms()
写入 notification_logs
调用短信 provider
```

禁止前端调用。

---

## 12.9 POST /api/feedback

用途：用户提交反馈。

入参：

```json
{
  "alertToken": "action_token",
  "feedbackType": "mismatched",
  "actualLimitAmount": 50,
  "actualPaymentAmount": 10,
  "comment": "支付宝显示限额不同"
}
```

---

## 12.10 POST /api/watchlist/unsubscribe

用途：退订短信提醒。

入参：

```json
{
  "token": "manage_token"
}
```

服务端动作：

```txt
写入 sms_unsubscribed_at
关闭所有 watchlist_items.notify_enabled
返回退订成功
```

---

# 13. 页面结构

## 13.1 创建观察清单页

目标：验证手机号。

展示：

```txt
手机号输入框
获取验证码
验证码输入框
创建观察清单
隐私与提醒说明
```

---

## 13.2 观察清单页

目标：管理观察基金。

展示：

```txt
脱敏手机号
正在监控基金数量
添加基金按钮
观察基金卡片列表
退订入口
```

卡片状态：

```txt
需核对
限额恢复
暂未影响
数据不确定
数据冲突
数据过期
```

---

## 13.3 添加观察基金页

目标：30 秒内完成配置。

字段：

```txt
基金代码 / 名称
定投周期
单次定投金额
开启短信提醒
```

加入前必须展示：

```txt
当前公开限额
当前可信状态
当前是否可能影响计划
```

---

## 13.4 限额变化提醒详情页

目标：把短信点击转成支付宝核对。

展示：

```txt
限额变化：old_limit → new_limit
用户观察计划：frequency + planned_amount
预计可执行金额
预计少投金额
可信状态
支付宝核对清单
去支付宝核对按钮
复制核对清单按钮
反馈按钮
退订入口
免责声明
```

---

## 13.5 反馈弹窗

默认：

```txt
一致
不一致
找不到入口
```

不一致展开：

```txt
支付宝实际显示限额，可选
支付宝实际扣款金额，可选
备注，可选
```

---

# 14. Vibe Coding 开发顺序

## Step 1：项目骨架

```txt
Next.js
Tailwind
Supabase client
环境变量
基础布局
```

## Step 2：数据库 migration

```txt
enum
funds
watchlists
sms_verification_codes
watchlist_items
current_limits
limit_change_events
alert_events
notification_logs
feedback_logs
```

## Step 3：Domain 层

```txt
confidence.ts
calculateShortfall.ts
evaluateAlert.ts
shouldSendSms.ts
buildDedupeKey.ts
maskPhone.ts
hashToken.ts
```

## Step 4：短信抽象层

```txt
src/lib/sms/provider.ts
src/lib/sms/templates.ts
src/lib/sms/sendSms.ts
```

注意：

```txt
业务代码只调用 sendSms()
不直接调用具体服务商 SDK
```

## Step 5：手机号验证接口

```txt
POST /api/sms/send-code
POST /api/sms/verify-code
```

## Step 6：观察清单接口

```txt
GET /api/watchlist
POST /api/watchlist/items
PATCH /api/watchlist/items/:id
DELETE /api/watchlist/items/:id
POST /api/watchlist/unsubscribe
```

## Step 7：页面

```txt
创建观察清单页
观察清单页
添加基金页
修改计划弹窗
提醒详情页
反馈弹窗
退订页
```

## Step 8：后台任务

```txt
admin 更新 current_limits
生成 limit_change_events
evaluate alerts
send sms
notification_logs
```

## Step 9：闭环测试

必须跑通：

```txt
验证手机号
→ 添加基金
→ 模拟限额 100 → 10
→ 生成 defensive_shortfall
→ 发送短信
→ 点击短信链接
→ 查看提醒详情
→ 去支付宝核对
→ 反馈不一致
→ feedback_logs 入库
```

---

# 15. 验收标准

## 15.1 最小闭环验收

必须完整跑通：

```txt
用户验证手机号
→ 添加观察基金
→ 系统监控限额变化
→ 只在影响计划时发短信
→ 用户点击短信链接
→ 去支付宝核对
→ 回来反馈
```

如果任一环缺失，MVP 不成立。

---

## 15.2 报警验收用例

| 场景     | Planned |   Old |   New | 是否短信 |
| ------ | ------: | ----: | ----: | ---: |
| 缩水     |     100 |   100 |    10 |    是 |
| 大额无关   |     100 | 50000 | 10000 |    否 |
| 恢复覆盖计划 |     100 |    10 |   100 |    是 |
| 从 0 恢复 |     100 |     0 |    10 |    是 |
| 暂停     |     100 |   100 |     0 |    是 |
| 数据不清   |     100 |   100 |  null |    否 |
| 来源冲突   |     100 |   100 |    10 |    否 |
| 数据过期   |     100 |   100 |    10 |    否 |
| 重复事件   |     100 |   100 |    10 |    否 |

---

## 15.3 防重复验收

```txt
同一 alert_event 不能发两次
同一手机号一天不能超过 3 条
退订后不能发送
关闭单基金 notify_enabled 后不能发送
ambiguous / conflict / stale 不发送金额短信
```

---

## 15.4 文案验收

不得出现：

```txt
建议买入
建议加仓
建议卖出
收益机会
抄底
推荐
```

必须出现：

```txt
公开限额变化
你配置的观察计划
可能影响定投执行
请去支付宝核对
是否修改由你自行决定
不构成投资建议
```

---

# 16. 成功指标

## 16.1 用户行为指标

```txt
手机号验证完成率 ≥ 50%
验证后添加基金率 ≥ 60%
单用户平均添加基金数 ≥ 2
添加基金后开启短信提醒率 ≥ 80%
短信点击提醒详情率 ≥ 20%
点击后反馈率 ≥ 5%
退订率 ≤ 10%
```

## 16.2 系统指标

```txt
短信重复发送率 = 0
短信发送失败率 ≤ 5%
有效提醒命中率 ≥ 70%
误报反馈率 ≤ 10%
confirmed / probable 数据占比 ≥ 70%
conflict 平均处理时长 ≤ 24 小时
```

---

# 17. 最终边界

本产品最终只负责：

```txt
用户手动配置观察基金
系统监控公开限额
判断是否影响配置计划
通过短信提醒用户核对
收集用户反馈修正数据可信度
```

本产品永远不负责：

```txt
读取真实交易
判断真实持仓
修改定投计划
提供买卖建议
预测收益
推荐基金
```

最终闭环：

```txt
用户验证手机号
→ 添加观察基金
→ 系统监控限额变化
→ 只在影响计划时发短信
→ 用户点击短信链接
→ 去支付宝核对
→ 回来反馈
```
