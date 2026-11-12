# MiniMax Code 签到接入设计

用户已于 2026-11-12 批准实现。

## 意图
让现有单文件 Cloudflare Worker 在原有三个平台之外，自动签到 MiniMax Code 国内版。部署仅更新 worker.js，复用现有绑定和 Cron。

## 方案与取舍
- 采用：新增独立 MiniMax IIFE 模块，使用 Workers fetch/Web Crypto 和加密 KV；控制台延续原样式。
- 不采用：直接复制 Node 插件（依赖文件系统与常驻计时器，不适合 Worker）。
- 不采用：额外服务、数据库或 Durable Object（增加部署配置，不符合本次最小更新目标）。

## 接口与数据
- /minimax/api/login/start POST → nonce、verificationUrl、userCode、expiresAt、interval（毫秒）。
- /minimax/api/login/poll?nonce=... GET → status=pending/authorized/expired/failed，pending 附 interval/message。
- /minimax/api/list GET → accounts，每项仅返回 id、label、lastRun、lastState、lastMsg、lastCredit、totalCredit、signedToday、ranToday、lastOkDate。
- /minimax/api/run POST（force=1 强制查询），/rename POST，/remove POST。
- 所有管理接口要求 ADMIN_TOKEN；令牌和临时授权数据使用 MASTER_KEY 加密保存，KV 前缀 minimax:。
- OAuth 表单、PKCE、请求签名与严格回执校验依据参考项目固定版本 eaa387563abe9f195abea2604c0b0779df08dec3/src/minimax.js。
- 自动续期在临近过期时发生；轮换后的令牌先落盘再请求身份。模块内对同账号运行和同会话轮询合并，KV 不承诺跨节点强一致互斥。

## 展示
新增 MiniMax 标签、账号表、官方授权入口与授权码/状态；按钮禁用、过期提示、slow_down 按服务端间隔处理；退出停止轮询，不外泄令牌。

## 汇总与验证
接入全部签到、原有外部 cron、Cloudflare scheduled、通知汇总。scheduled 复用 runAll 使通知一致。模拟 HTTP/KV 验证成功、已签、无可领、错误回执、刷新持久化、授权过期和慢轮询。真实账户及 Cloudflare 出口尚未验证。

## 非目标
不支持国际版，不读取本机 CLI 凭据，不自动处理验证码，不更改现有三平台协议，不新增必填绑定，不部署或提交。
