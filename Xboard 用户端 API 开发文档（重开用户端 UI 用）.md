# Xboard 用户端 API 开发文档

基于 `cedar2025/Xboard` 仓库 master 分支的路由、控制器、Request、Resource 源码整理（`app/Http/Routes/V1/*`、`app/Http/Controllers/V1/*`）。用于自制主题 UI，覆盖登录注册、购买、订阅、工单、邀请等用户端全部能力。

## 0. 先看这几点（对你给的清单的勘误与补充）

1. **`GET /user/knowledge/getCategory` 是坏接口**：路由注册了，但 `KnowledgeController` 里没有 `getCategory` 方法，调用会 500。分类请从 `knowledge/fetch` 的返回里取，返回结果已按 `category` 分组。
2. **清单漏了这些用户端接口**（源码里有）：`POST /user/getQuickLoginUrl`、`GET /user/getActiveSession`、`POST /user/removeActiveSession`、`POST /user/ticket/withdraw`、`POST /user/gift-card/check`、`POST /user/gift-card/redeem`、`GET /user/gift-card/history`、`GET /user/gift-card/detail`、`GET /user/gift-card/types`、`GET /user/telegram/getBotInfo`、`POST /user/comm/getStripePublicKey`。
3. **UI 不需要调用的**：`guest/payment/notify/*`（支付网关回调）、`guest/telegram/webhook`、`client/subscribe`、`client/app/getConfig`、`client/app/getVersion`（给代理客户端用）。订阅链接直接用 `user/getSubscribe` 返回的 `subscribe_url`。
4. **响应格式不统一**，有 5 个接口不走标准包装，见第 1.4 节，封装请求层时要单独处理。
5. 本文中标注「未核实」的是我没在这次读取里看到源码、按惯例推断的内容，上线前请自己对一下。

## 1. 通用约定

### 1.1 基础信息

| 项 | 值 |
| --- | --- |
| Base URL | `{站点域名}/api/v1` |
| CORS | `api/*` 允许任意来源、任意头（`config/cors.php`），主题与后端分域也能调 |
| 强制 JSON | 中间件 `ForceJson` 会把 `Accept` 设为 `application/json` |
| 语言 | 请求头 `Content-Language: zh-CN` 等，后端据此切换错误提示语言 |
| 请求体 | 表单或 JSON 均可（Laravel `$request->input`） |

### 1.2 认证

登录/注册成功返回：

```json
{
  "status": "success",
  "message": "success",
  "data": {
    "token": "订阅 token（用于拼订阅链接，不是登录凭证）",
    "auth_data": "Bearer 1xxxxxxxx",
    "is_admin": false
  },
  "error": null
}
```

- 之后所有 `/user/*` 请求带请求头 `Authorization: <auth_data 原样>`（值已含 ` Bearer  ` 前缀）。
- token 是 Sanctum 令牌，有效期 1 年。
- 未登录或过期：`/user/*` 中间件抛 **HTTP 403**，message 为「未登录或登陆已过期」。前端统一在 403 时清 token 并跳登录页。
- `changePassword` 成功后会删除该用户**其他所有**登录会话（保留当前这个）。

### 1.3 标准响应

成功：

```json
{ "status": "success", "message": "success", "data": <任意>, "error": null }
```

失败：

```json
{ "status": "fail", "message": "错误提示", "data": null, "error": null }
```

- 失败时 HTTP 状态码取业务码前 3 位（例如业务码 `400201`「邮箱已存在」→ HTTP 400）。常见：400 业务错误、403 未登录、422 参数/校验、429 注册频繁、500 服务端错误。
- Laravel 表单校验失败（`$request->validate`）返回 422，格式是 Laravel 原生的 `{ "message": "...", "errors": { "字段": ["..."] } }`，和上面的 fail 不同，请求层要兼容两种。
- 生产环境未捕获异常只返回 `{ "message": "..." }`。

### 1.4 不走标准包装的接口

| 接口 | 实际返回 |
| --- | --- |
| `GET /user/notice/fetch` | `{ "data": [...], "total": 数字 }`（无 status） |
| `GET /user/server/fetch` | `{ "data": [...] }`，带 `ETag` 头；请求带 `If-None-Match` 且未变化时返回 **304 空体** |
| `GET /user/invite/details` | `{ "data": [...], "total": 数字 }` |
| `GET /user/gift-card/history` | `{ "data": [...], "pagination": {current_page,last_page,per_page,total} }` |
| `POST /user/order/checkout` | `{ "type": 数字, "data": ... }`（无 status） |

另外 `GET /passport/auth/token2Login?verify=` 成功时返回 `{ "data": {...} }`，失败返回 `{ "message": "..." }` + HTTP 400。

### 1.5 单位与枚举

- **金额**：整数，单位**分**（`balance`、`commission_balance`、订单 `total_amount` 等）。展示时 `/100`。
- **套餐价格**：`PlanResource` 已把价格乘 100 输出，也是分，展示 `/100`。不可购买的周期为 `null`。
- **流量**：字节。`u` 上传、`d` 下载、`transfer_enable` 总额度。已用 = `u + d`。
- **时间**：Unix 时间戳（秒）。`expired_at` 为 `null` 表示长期有效。
- **周期（period）** 前端统一使用旧版键名：`month_price`、`quarter_price`、`half_year_price`、`year_price`、`two_year_price`、`three_year_price`、`onetime_price`、`reset_price`（流量重置包）。
- **订单状态** `status`：0 待支付、1 开通中、2 已取消、3 已完成、4 已折抵。
- **订单类型** `type`：1 新购、2 续费、3 升级、4 流量重置包。
- **工单状态** `status`：0 进行中、1 已关闭；`reply_status`：0 等待客服回复、1 已回复；`level`：0 低、1 中、2 高。
- **流量重置方式** `reset_traffic_method`：null 跟随系统、0 每月 1 号、1 按月、2 不重置、3 每年 1 月 1 日、4 按年。

## 2. 主题如何挂载（做主题必看）

后端路由 `GET /` 渲染当前主题的 `dashboard.blade.php`，把站点配置注入为全局变量，你的前端从这里读取运行时配置。

**主题目录结构**（最小集）：

```
MyTheme/
├── config.json            # 必须，含 name、version、configs（后台可配项）
├── dashboard.blade.php    # 必须，入口模板
└── assets/                # 你的构建产物（js/css/图片）
```

- 打包成 zip 上传（后台主题管理）。`config.json` 里 `name` 不能与系统主题 `Xboard`、`v2board` 同名；更新同名主题要求 `version` 更高。
- 上传后主题放在 `storage/theme/{name}`，首次访问时复制到 `public/theme/{name}`，资源路径为 `/theme/{name}/assets/...`。
- `config.json` 的 `configs` 数组定义后台可配置项（字段：`label`、`placeholder`、`field_name`、`field_type`、`select_options`、`default_value`），官方 Xboard 主题有 `theme_color`（select）、`background_url`（input）、`custom_html`（textarea）作为参考。

**`dashboard.blade.php` 注入的变量**（官方主题写法，你照搬即可）：

```html
<script>
  window.routerBase = "/";            // API 基础地址（请求 routerBase + 'api/v1/...'）
  window.settings = {
    title: '{{$title}}',               // 站点名
    description: '{{$description}}',
    assets_path: '/theme/{{$theme}}/assets',
    theme: { color: '{{ $theme_config["theme_color"] ?? "default" }}' },
    version: '{{$version}}',
    background_url: '{{$theme_config["background_url"]}}',
    logo: '{{$logo}}',
    i18n: ['zh-CN','en-US','ja-JP','vi-VN','ko-KR','zh-TW','fa-IR']
  }
</script>
<div id="app"></div>
{!! $theme_config['custom_html'] !!}
```

- Blade 可用变量：`$title`、`$theme`、`$version`、`$description`、`$logo`、`$theme_config`（你在 `config.json` 里定义的配置项取值）。
- 官方主题使用 hash 路由（`/#/login?verify=...&redirect=dashboard`），`token2Login` 的跳转也按这个格式拼，所以**你的路由最好也用 hash 模式**，否则邮件链接登录和快捷登录要自己改。
- 本地开发时参考官方的 `env.example.js`：`window.routerBase = 'http://127.0.0.1:8000/'`，配合 CORS 可前后端分离调试。

## 3. Passport（无需登录）

### 3.1 获取站点配置（先调这个）

`GET /guest/comm/config`

返回字段（决定登录/注册页要渲染什么）：

| 字段 | 说明 |
| --- | --- |
| `tos_url` | 服务条款链接 |
| `is_email_verify` | 1 = 注册需邮箱验证码 |
| `is_invite_force` | 1 = 必须填邀请码 |
| `email_whitelist_suffix` | 开启白名单时为后缀数组，否则 `0` |
| `is_captcha` | 1 = 开启人机验证 |
| `captcha_type` | `recaptcha`、`recaptcha-v3`、`turnstile` |
| `recaptcha_site_key`、`recaptcha_v3_site_key`、`turnstile_site_key` | 前端渲染验证码组件用 |
| `recaptcha_v3_score_threshold` | v3 分数阈值 |
| `app_description`、`app_url`、`logo` | 站点信息 |
| `is_recaptcha` | 旧字段，兼容用 |

### 3.2 人机验证参数

开启 `is_captcha` 后，`register` 与 `sendEmailVerify` 需要额外带参数（按 `captcha_type`）：

| captcha\_type | 请求参数 |
| --- | --- |
| `turnstile` | `turnstile_token` |
| `recaptcha-v3` | `recaptcha_v3_token` |
| `recaptcha` | `recaptcha_data` |

### 3.3 注册

`POST /passport/auth/register`

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `email` | 是 | 严格邮箱格式 |
| `password` | 是 | 至少 8 位 |
| `invite_code` | 视配置 | `is_invite_force=1` 时必填 |
| `email_code` | 视配置 | `is_email_verify=1` 时必填，6 位数字 |
| 验证码参数 | 视配置 | 见 3.2 |

返回：`{ token, auth_data, is_admin }`（注册即登录）。

可能的失败：注册已关闭、邮箱后缀不在白名单、Gmail 别名（含 `.` 或 `+`）被禁、邀请码无效、验证码错误、邮箱已存在（业务码 400201）、同 IP 注册过于频繁（429）。

### 3.4 登录

`POST /passport/auth/login`，参数：`email`、`password`（≥8 位）。返回同上。

### 3.5 发送邮箱验证码

`POST /passport/comm/sendEmailVerify`，参数：`email`（开启验证码时带 3.2 的参数）。

- 验证码 6 位，有效期 5 分钟；同一邮箱 **60 秒**内只能发一次，前端按钮做 60 秒倒计时。
- 开启邮箱白名单时，未注册邮箱的后缀必须在白名单内。
- 用于注册和忘记密码两处。

### 3.6 忘记密码

`POST /passport/auth/forget`，参数：`email`、`email_code`（6 位数字）、`password`（≥8 位）。成功返回 `data: true`。

### 3.7 邮件链接登录

`POST /passport/auth/loginWithMailLink`，参数：`email`（必填）、`redirect`（可选，登录后跳转路径）。后端发邮件，邮件里的链接指向 `GET /passport/auth/token2Login?token=xxx&redirect=xxx`，该接口会 302 跳到 `{app_url}/#/login?verify=xxx&redirect=xxx`。

前端在登录页检测到 URL 里有 `verify` 参数时，调用：

`GET /passport/auth/token2Login?verify=<verify>`，成功返回 `{ "data": { token, auth_data, is_admin } }`，失败 HTTP 400 + `{ "message": "..." }`。

### 3.8 快捷登录 URL

`POST /passport/auth/getQuickLoginUrl`，参数：`auth_data`（或请求头 `Authorization`）、`redirect`（可选）。返回一个可免密登录的 URL（用于从别的入口跳转到网页版）。无有效登录态返回 401。

登录后也可以调 `POST /user/getQuickLoginUrl`（同样参数 `redirect`），效果一致。

### 3.9 邀请码访问统计

`POST /passport/comm/pv`，参数：`invite_code`。用户通过邀请链接打开注册页时调用一次，给该邀请码 PV +1，始终返回 `true`。

## 4. 用户信息与账户（需登录）

### 4.1 检查登录状态

`GET /user/checkLogin` → `{ is_login: true, is_admin?: true }`

### 4.2 用户信息

`GET /user/info`

```json
{
  "email": "",
  "transfer_enable": 0,
  "last_login_at": 0,
  "created_at": 0,
  "banned": 0,
  "remind_expire": 0,
  "remind_traffic": 0,
  "expired_at": null,
  "balance": 0,
  "commission_balance": 0,
  "plan_id": null,
  "discount": null,
  "commission_rate": null,
  "telegram_id": null,
  "uuid": "",
  "avatar_url": "https://cdn.v2ex.com/gravatar/<md5(email)>?s=64&d=identicon"
}
```

### 4.3 订阅信息

`GET /user/getSubscribe`

返回：`plan_id`、`token`、`expired_at`、`u`、`d`、`transfer_enable`、`email`、`uuid`、`device_limit`、`speed_limit`、`next_reset_at`，以及：

- `plan`：当前套餐完整对象（有套餐时），套餐被删则返回失败「订阅计划不存在」；
- `subscribe_url`：订阅链接（站点可配置多个订阅域名，后端随机选一个），直接展示/复制/生成二维码；
- `reset_day`：距离下次流量重置的天数。

仪表盘的流量进度条、到期时间、订阅链接、一键导入都用这个接口。

### 4.4 统计角标

`GET /user/getStat` → **数组** `[待支付订单数, 进行中工单数, 已邀请用户数]`。

### 4.5 更新提醒设置

`POST /user/update`，参数：`remind_expire`（0/1）、`remind_traffic`（0/1），返回 `true`。目前只允许改这两项。

### 4.6 修改密码

`POST /user/changePassword`，参数：`old_password`、`new_password`（≥8 位）。

### 4.7 重置订阅安全信息

`GET /user/resetSecurity`：重新生成 `uuid` 和订阅 `token`，**旧订阅链接立即失效**，返回新的订阅链接字符串。UI 需要二次确认弹窗。

### 4.8 佣金转余额

`POST /user/transfer`，参数：`transfer_amount`（整数，单位分，≥1，不能超过 `commission_balance`）。成功后佣金减少、余额增加。

### 4.9 登录会话管理

- `GET /user/getActiveSession`：返回当前用户全部 Sanctum token 记录数组；
- `POST /user/removeActiveSession`，参数：`session_id`：踢掉某个会话。

（返回记录的字段是 Sanctum `personal_access_tokens` 表字段，`id`、`name`、`last_used_at`、`created_at` 等，未逐项核实。）

### 4.10 站点配置（登录后）

`GET /user/comm/config`：

| 字段 | 说明 |
| --- | --- |
| `is_telegram` | 是否启用 Telegram 机器人 |
| `telegram_discuss_link` | 群组链接 |
| `stripe_pk` | Stripe 公钥 |
| `withdraw_methods` | 可用提现方式数组 |
| `withdraw_close` | 1 = 关闭提现 |
| `currency`、`currency_symbol` | 货币代码与符号 |
| `commission_distribution_enable` / `_l1` / `_l2` / `_l3` | 三级分销开关与比例 |

`POST /user/comm/getStripePublicKey`，参数：`id`（支付方式 id，且必须是 `StripeCredit` 类型），返回该支付方式的 Stripe 公钥。

### 4.11 Telegram 绑定信息

`GET /user/telegram/getBotInfo` → `{ username }`，用于拼 `https://t.me/<username>`。绑定指令是在 Telegram 里发给机器人 `/bind <订阅链接>`，机器人逻辑不在 Web API 里（未在本次读取里核实指令格式）。

## 5. 套餐、订单与支付

### 5.1 套餐列表

- 未登录（落地页展示）：`GET /guest/plan/fetch`
- 登录后：`GET /user/plan/fetch`，可带 `id` 查单个套餐（该用户不可买则返回失败）。

套餐对象字段：

```json
{
  "id": 1, "group_id": 1, "name": "", "tags": [], "content": "描述（已替换占位符）",
  "month_price": 1000, "quarter_price": null, "half_year_price": null, "year_price": null,
  "two_year_price": null, "three_year_price": null, "onetime_price": null, "reset_price": null,
  "capacity_limit": null,
  "transfer_enable": 100, "speed_limit": null, "device_limit": null,
  "show": true, "sell": true, "renew": true,
  "reset_traffic_method": null, "sort": 1, "created_at": 0, "updated_at": 0
}
```

- `content` 里的 `{{transfer}}`、`{{speed}}`、`{{devices}}`、`{{reset_method}}` 已由后端替换，直接渲染（可能含 HTML/Markdown，渲染前注意做 XSS 过滤）。
- `capacity_limit`：`null` 不限；正整数为剩余名额；售罄时返回**字符串**（「Sold out」的本地化文本），注意类型是 `int | string | null`。
- `transfer_enable` 在套餐里的单位是 GB（用户表里是字节），展示时别混用。
- 价格为 `null` 的周期不要渲染选项。`sell=false` 的套餐不可下单。

### 5.2 校验优惠码

`POST /user/coupon/check`，参数：`code`（必填）、`plan_id`、`period`（旧版周期键）。

成功返回优惠券完整对象（含 `type`、`value`、`limit_plan_ids`、`limit_period` 等；其中 `limit_plan_ids` 为字符串数组、`limit_period` 为旧版周期键数组）。

失败消息包括：优惠码无效、已用完、未开始、已过期、不适用该套餐、不适用该周期、超过每人使用次数。

优惠券 `type`：沿用 v2board 约定，1 为固定金额（分）、2 为百分比，本次未在源码里核实，上线前请确认。

### 5.3 创建订单

`POST /user/order/save`

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `plan_id` | 是 | 套餐 id |
| `period` | 是 | 旧版周期键，取值见 1.5 |
| `coupon_code` | 否 | 优惠码 |

返回 `data: "<trade_no>"`（订单号字符串）。

限制：已有未支付/开通中的订单时拒绝（提示先取消）；还会做套餐可购买性校验（售罄、不允许续费、重置包需有有效订阅等），错误会以 fail 返回。

### 5.4 支付方式

`GET /user/order/getPaymentMethod` → 数组，每项 `{ id, name, payment, icon, handling_fee_fixed, handling_fee_percent }`。

手续费 = `订单金额 × handling_fee_percent / 100 + handling_fee_fixed`（四舍五入取整，单位分）。结算页要展示手续费和最终应付金额。

### 5.5 订单结算（发起支付）

`POST /user/order/checkout`

| 参数 | 说明 |
| --- | --- |
| `trade_no` | 订单号 |
| `method` | 支付方式 id（`getPaymentMethod` 返回的 `id`） |
| `token` | Stripe 等需要前端先生成支付 token 的方式使用 |

返回（**注意无 status 包装**）：`{ "type": N, "data": ... }`

| type | 含义 | data | 前端动作 |
| --- | --- | --- | --- |
| `-1` | 订单金额为 0（余额/优惠全额抵扣），后端已直接完成 | `true` | 直接提示成功并刷新用户信息 |
| `0` | 二维码支付 | 二维码内容字符串 | 渲染二维码，同时轮询订单状态 |
| `1` | 跳转支付 | 支付页 URL | `window.location` 或新窗口打开，回来后轮询 |
| 其他 | 由具体支付插件定义（如 Stripe） | 按插件而定 | 需按所用插件逐个适配 |

`type` 的 0/1 含义根据内置支付插件（AlipayF2f 返回 0；Btcpay、CoinPayments、Coinbase、Epay、Mgate 返回 1）推断，其他插件请自查其 `pay()` 返回值。

订单必须是待支付状态，否则返回「订单不存在或已支付」。

### 5.6 轮询订单状态

`GET /user/order/check?trade_no=xxx` → `data: <status 数字>`。

支付后每 2 到 3 秒轮询，`status` 变为 3（已完成）即成功，变为 2 即已取消。

### 5.7 订单列表 / 详情 / 取消

- `GET /user/order/fetch`，可选 `status`（0 到 3）筛选，按创建时间倒序，返回订单数组（含 `plan` 对象）。
- `GET /user/order/detail?trade_no=xxx`：订单详情，包含 `plan`、`payment`（`{id,name,payment,icon}`）、`try_out_plan_id`（试用套餐 id），有折抵时含 `surplus_orders`。
- `POST /user/order/cancel`，参数：`trade_no`；只有待支付订单可取消。

订单对象常用字段：`trade_no`、`plan_id`、`period`（旧版键）、`type`、`status`、`total_amount`、`handling_amount`、`balance_amount`、`discount_amount`、`surplus_amount`、`coupon_id`、`payment_id`、`paid_at`、`created_at`、`updated_at`、`plan`。

## 6. 节点、流量与公告

### 6.1 节点列表

`GET /user/server/fetch`

- 用户订阅无效（已封禁、无流量额度、已过期）时返回**空数组**。
- 每项：`id`、`type`（协议）、`version`、`name`、`rate`（倍率）、`tags`、`is_online`、`cache_key`、`last_check_at`。
- 支持 ETag：保存响应的 `ETag`，下次带 `If-None-Match`，未变化返回 304，可以省流量。

### 6.2 流量记录

`GET /user/stat/getTrafficLog`：**本月**每日流量记录数组，每项 `{ d, u, record_at, server_rate, user_id? }`，按 `record_at` 倒序。用于流量统计图。

### 6.3 公告

`GET /user/notice/fetch?current=1`：分页（每页固定 5 条），返回 `{ data, total }`。公告对象字段按 Notice 表（`id`、`title`、`content`、`img_url`、`tags`、`created_at` 等，未逐项核实）。

### 6.4 知识库

`GET /user/knowledge/fetch`

| 参数 | 说明 |
| --- | --- |
| 无 `id` | 列表：返回 `{ "分类名": [ {id, category, title, body, updated_at}, ... ] }`，按分类分组 |
| `language` | 列表模式下用于过滤语言，如 `zh-CN`；**务必传**，否则过滤条件为空 |
| `keyword` | 搜索标题和正文 |
| `id` | 单篇文章详情，返回含 `body` |

正文里后端会替换占位符：`{{siteName}}`、`{{subscribeUrl}}`、`{{urlEncodeSubscribeUrl}}`、`{{safeBase64SubscribeUrl}}`。无有效订阅的用户，`<!--access start-->...<!--access end-->` 之间的内容会被替换成 `<div class="v2board-no-access">…</div>`，你的 CSS 要给这个类名样式。正文是 HTML/Markdown，渲染要做 XSS 过滤。

## 7. 工单

- `GET /user/ticket/fetch`：我的工单列表（倒序）。字段：`id`、`level`、`reply_status`、`status`、`subject`、`created_at`、`updated_at`、`user_id?`。
- `GET /user/ticket/fetch?id=xx`：工单详情，多出 `message` 数组，每条 `{ id, ticket_id, is_me, message, created_at, updated_at }`，`is_me=true` 为用户自己发的。
- `POST /user/ticket/save`：新建，参数 `subject`、`level`（0/1/2）、`message`，三项必填，成功返回 `true`。
- `POST /user/ticket/reply`：回复，参数 `id`、`message`。已关闭工单不能回复；若后台开启「必须等待客服回复」，用户连续回复会被拒绝。
- `POST /user/ticket/close`：关闭，参数 `id`。
- `POST /user/ticket/withdraw`：提现申请（本质是后台自动开一张高优先级工单），参数 `withdraw_method`（必须在 `withdraw_methods` 内）、`withdraw_account`。要求佣金余额 ≥ 后台设置的最低提现额；`withdraw_close=1` 时不可用。

注意：`ticket/fetch?id=` 在工单不存在或不属于该用户时，源码里先对 `null` 调了 `->load()`，会抛 500 而不是友好提示，前端要对 500 兜底。

## 8. 邀请与佣金

- `GET /user/invite/fetch`：
  - `codes`：我未使用的邀请码数组，每项 `{ user_id?, code, pv, status, created_at, updated_at }`；
  - `stat`：**数组** `[已邀请注册人数, 累计有效佣金, 确认中佣金, 佣金比例(%), 可用佣金]`，金额为分。
- `GET /user/invite/save`：生成新邀请码，达到上限（后台 `invite_gen_limit`，默认 5）会失败。注意是 **GET**。
- `GET /user/invite/details?current=1&page_size=10`：佣金明细，`page_size` 小于 10 时按 10 处理。返回 `{ data: [{id, order_amount, trade_no, get_amount, created_at}], total }`。

邀请链接格式由前端自己拼：`{站点}/#/register?code=<邀请码>`，注册页读到 `code` 参数时先调一次 `passport/comm/pv`。

## 9. 礼品卡

- `GET /user/gift-card/types`：礼品卡类型映射。
- `POST /user/gift-card/check`，参数 `code`：返回 `{ code_info, reward_preview, can_redeem, reason }`，用于兑换前预览与判断能否兑换。
- `POST /user/gift-card/redeem`，参数 `code`（8 到 32 位）：返回 `{ message, rewards, invite_rewards, template_name }`。
- `GET /user/gift-card/history?page=1&per_page=15`：兑换记录，`per_page` 最大 100；`code` 已脱敏，只显示前 8 位。
- `GET /user/gift-card/detail?id=xx`：单条兑换详情，含模板信息（`name`、`description`、`type`、`type_name`、`icon`、`theme_color`）。

## 10. 关键业务流程

### 10.1 启动流程

1. 读取 `window.settings` 和 `window.routerBase`；
2. 调 `GET /guest/comm/config` 缓存站点配置；
3. 本地有 `auth_data` 时调 `GET /user/checkLogin`：403 则清除并跳登录页；
4. 登录态下并行拉 `user/info`、`user/getSubscribe`、`user/comm/config`、`user/getStat`、`user/notice/fetch`。

### 10.2 购买流程

```
选套餐 → (可选) coupon/check → order/save 得 trade_no
  → getPaymentMethod → 用户选支付方式 → checkout
  → 按返回 type 处理 (-1 完成 / 0 二维码 / 1 跳转)
  → 轮询 order/check 直到 status=3
  → 刷新 user/info 与 getSubscribe
```

已有待支付订单时 `order/save` 会拒绝：提示用户去订单页继续支付或取消。

### 10.3 注册流程

按 `guest/comm/config` 动态渲染表单：邮箱白名单后缀下拉、邀请码输入（`is_invite_force` 决定是否必填）、邮箱验证码输入与发送按钮（60 秒倒计时）、验证码组件。

## 11. 页面与接口对照（建议的页面清单）

| 页面 | 接口 |
| --- | --- |
| 登录 / 注册 / 忘记密码 | `guest/comm/config`、`passport/auth/*`、`passport/comm/*` |
| 仪表盘 | `user/getSubscribe`、`user/info`、`user/getStat`、`user/notice/fetch`、`user/knowledge/fetch`（教程入口） |
| 套餐 / 购买 | `user/plan/fetch`、`user/coupon/check`、`user/order/save` |
| 订单列表 / 详情 / 支付 | `user/order/*`、`user/comm/config` |
| 节点列表 | `user/server/fetch` |
| 流量统计 | `user/stat/getTrafficLog` |
| 知识库 | `user/knowledge/fetch` |
| 工单 | `user/ticket/*` |
| 邀请 / 佣金 / 提现 | `user/invite/*`、`user/transfer`、`user/ticket/withdraw`、`user/comm/config` |
| 个人中心 | `user/changePassword`、`user/update`、`user/resetSecurity`、`user/getActiveSession`、`user/removeActiveSession`、`user/telegram/getBotInfo` |
| 礼品卡 | `user/gift-card/*` |

## 12. 请求层封装建议（TypeScript 骨架）

```ts
const BASE = (window as any).routerBase + 'api/v1'

async function request<T>(path: string, init: RequestInit = {}): Promise<T> {
  const auth = localStorage.getItem('auth_data')
  const res = await fetch(BASE + path, {
    ...init,
    headers: {
      'Content-Type': 'application/json',
      'Content-Language': localStorage.getItem('lang') || 'zh-CN',
      ...(auth ? { Authorization: auth } : {}),
      ...init.headers,
    },
  })
  if (res.status === 403) { localStorage.removeItem('auth_data'); location.hash = '#/login'; throw new Error('未登录') }
  if (res.status === 304) return undefined as T
  const json = await res.json()
  if (!res.ok) throw new Error(json.message ?? Object.values(json.errors ?? {})[0] as string)
  // 标准包装取 data；notice/server/invite details/checkout/gift-card history 等直接返回原对象
  return (json.status === 'success' ? json.data : json) as T
}
```

`GET` 请求参数放 query string；`POST` 用 JSON 或 `x-www-form-urlencoded` 都可以。

## 13. 注意事项汇总

- 金额统一按分处理，套餐里的 `transfer_enable` 单位是 GB，用户的 `transfer_enable`、`u`、`d` 是字节。
- `getStat`、`invite/fetch.stat` 返回的是**位置数组**，别当对象用，下标含义见上文。
- 渲染 `plan.content`、`knowledge.body`、`notice.content` 都要做 XSS 过滤。
- 邮件登录与快捷登录的跳转链接默认指向 hash 路由，请保持路由兼容。
- `invite/save` 是 GET 但有副作用，别被预取或缓存。
- 所有结论均来自读取 master 分支源码；后台设置项（`admin_setting`）会改变接口行为，联调时用你自己的站点验证一遍。
