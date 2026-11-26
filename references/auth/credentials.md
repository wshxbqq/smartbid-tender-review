# 身份与恢复

使用技能入口文件 `SKILL.md` 中的 `metadata.version` 对照 `GET /skill/meta` 返回的 `minimumClientVersion`，版本过低时先更新技能包。确认 `enabled=true`、`policy.mode="fixed"` 及条款版本。

## 默认按单付款

访客使用 REST 接口。用户的上传指令已确定本单范围，或歧义已经澄清后，以 `mode:"guest"` 创建会话。保存服务器返回的 `sessionId`、`projectId` 和 `accessKey`，连同当前 `apiBaseUrl`、日志游标及后续报价、运行和付款幂等键私密保存到 `~/.smartbid/sessions/<UUID>.json`。

后续本会话请求用 `Authorization: Bearer <accessKey>`。UUID仅用于展示和客服定位，不能代替访问恢复信息。

创建会话的响应正文和响应头分别含恢复令牌和访客 Cookie。让发起 HTTPS 请求的程序在同一次调用中将它们保存到私密文件。工具的标准输出、标准错误、调试日志以及可展开的工具参数都属于对话记录。创建成功时只提取 `sessionId`、`projectId`、`mode` 和供用户查看的 `progressUrl`、`progressUrlExpiresAt`，失败时只输出 HTTP 状态和公开错误码。后续请求在程序内读取恢复令牌或账户 API Key，组装认证头。

创建新会话前，读取私密的 `~/.smartbid/visitors.json`，按apiBaseUrl查找已有访客Cookie并在同源创建请求中携带。

服务器通过 `Set-Cookie` 自动分配访客身份。按服务地址私密保存并在同源请求中继续携带此 Cookie，在会话文件保留恢复副本。同一访客同时最多保留一个未付任务。新订单由服务端确认到账后自动开始评审，同时允许开下一单。Cookie 由宿主按标准规则管理。

## 可选官网API Key

仅在用户已有 API Key，或明确想使用官网积分时填写。所有 API Key 属于官网账户的同一类密钥，可分别撤销；同一账户的多个 Key 共用积分余额与账务状态。

通过宿主安全输入或私密配置读取，保存到 `~/.smartbid/credentials.json`：

```json
{"apiBaseUrl":"https://biao.vvcut.cn/api/v1","apiKey":"<用户的API Key>"}
```

以 `mode:"account"`创建会话，所有请求使用此API Key。兼容旧本地配置的 `apiToken`字段值，无需重新生成密钥。401提示修正有效API Key，积分不足引导官网充值，不静默换成现金付款。

## 私密文件

Unix上目录0700、凭据和会话文件0600；其他系统使用仅当前用户可读的权限。只发送到凭据绑定的服务地址。切换测试地址时不能携带原地址的API Key或Cookie。不得回显完整认证值、Cookie或邮件下载凭证。
