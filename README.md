# Xboard-themes
Xboard简约用户端界面主题achen，后台上传即可。
![图片1](https://cfbed.1314883.xyz/file/1792538175230_be64327d73696969e4ba470c12678b9e.webp)
![图片2](https://cfbed.1314883.xyz/file/1792538297490_f8510b720c6c3cedc586bcc512815d44.webp)
![图片3](https://cfbed.1314883.xyz/file/1792538304876_ca6ec1dabee9a3881f55aa6c4e0420a9.webp)

### 1.4.0

发布日期：2026-10-22

#### 支付方式图标对接后端插件

- 支付页支付方式列表优先显示后端 `GET /user/order/getPaymentMethod` 返回的 `icon`（支付插件自定义图标）。
- 无 `icon` 或图片加载失败时，回退为默认蓝色渐变首字母图标（`onerror` 兜底，不会留白块）。
- 购物中心订单详情新增「支付方式」行，同样使用插件图标 + 默认兜底。

#### 公告背景支持后端自定义图片

- 公告列表卡片、公告详情、首次进入的公告弹窗，均支持后端 `notice.img_url` 作为背景 / 头图。
- 无 `img_url` 时保持默认样式（图标 + 发丝线卡片）；图片加载失败自动回退默认。

#### 订阅流量进度条

- 进度条上方显示已用占比百分比（保留一位小数）。
- 按用量占比显示四档颜色：绿（<50%）、黄（50–75%）、红（75–90%）、暗红（≥90%）。
- 新增脉冲呼吸动画与流光扫过效果，占比越高脉冲越快；深色模式同步适配文字配色。

#### 一键导入弹窗

- 「一键导入」按钮改为打开弹窗：标题「一键导入」，提供桌面端 / 移动端切换。
- 桌面端：Clash、V2rayN；移动端：Shadowrocket、Hiddity，点击即通过各自 URL Scheme 导入订阅。
- 弹窗底部两个按钮：「复制订阅」（复制订阅链接）、「使用教程」（跳转帮助中心）。
- 客户端兼容性提示更新为：Shadowsocks、VMess、VLESS、Trojan、Hysteria、TUIC、SOCKS、HTTP、Naive、Mieru、AnyTLS。

#### 站点图标

- `dashboard.blade.php` 输出 `favicon` 与 `apple-touch-icon`：后台配置了 Logo 时用 Logo，未配置时使用内置默认图标（品牌蓝圆角 A）。

---
