# REST 接口：固定报价、收款与报告

默认基础地址 `https://biao.vvcut.cn/api/v1`。使用宿主已有网络和文件能力；不要求安装特定客户端。JSON请求用 `Content-Type: application/json`，金额为整数分，原始材料和PDF用字节流传输。

## 创建与身份

`GET /skill/meta`取得开放状态、fixed费率、`termsVersion`及上传限制。

```http
POST /api/v1/review-sessions
Content-Type: application/json

{"mode":"guest","source":"skill","title":"本次招投标评审","consent":{"accepted":true,"version":"<meta.termsVersion>"},"files":[{"name":"招标.pdf","role":"tender","size":12345,"sha256":"<真实SHA-256>"},{"name":"投标.docx","role":"bid","size":67890,"sha256":"<真实SHA-256>"}]}
```

响应：`{"sessionId":"<UUID>","projectId":"<UUID>","mode":"guest","accessKey":"<自动私密保存>"}`，以及访客身份的Set-Cookie。立即保存响应和Cookie。后续请求带 `Authorization: Bearer <accessKey>`，按同源Cookie规则携带访客Cookie。

有 curl 和 Python 的宿主可按下面的例子处理首次创建。准备本次站点的私密会话目录 `session_dir`（0700）、同源 Cookie 文件路径 `cookie_jar`、已依用户指令确定范围的请求文件 `manifest_file` 与接口基础地址 `api_base`。原始响应直接落盘，再从私密文件提取可展示的字段。用该私密回执及 Cookie 完成上文的会话恢复记录。

```sh
set -eu
umask 077
receipt=$(mktemp "$session_dir/create.XXXXXXXX")
curl --silent --show-error --fail --cookie "$cookie_jar" --cookie-jar "$cookie_jar" \
  --header 'Content-Type: application/json' --data-binary "@$manifest_file" \
  --output "$receipt" "$api_base/review-sessions"
python3 - "$receipt" <<'PY'
import json
import sys

with open(sys.argv[1], encoding="utf-8") as source:
    session = json.load(source)
print(json.dumps({key: session[key] for key in ("sessionId", "projectId", "mode", "progressUrl", "progressUrlExpiresAt") if key in session}))
PY
```

有官网API Key时改为 `mode:"account"`，所有请求使用 `Authorization: Bearer <API Key>`，不另发用户API Key。SkillPay入口只将source标记为skillpay，继续共用这些接口。

## 上传与预检

```http
POST /api/v1/review-sessions/<UUID>/uploads/intents
Authorization: Bearer <本次认证值>
Content-Type: application/json

{"originalName":"招标.pdf","role":"tender","size":12345,"sha256":"<真实SHA-256>"}
```

响应返回 `uploadPath`、`contentType`及意图ID。向该同源路径PUT原始字节流，Content-Length为真实大小；不使用multipart、JSON或对话文本代替文件。

- `GET /review-sessions/{id}/uploads`：查询恢复上传，items含fileId、状态、摘要和大小。
- `GET /review-sessions/{id}/preflight`：返回ready、failed、files、页数及规范化文本计量，未ready时继续免费评估。
- `PATCH /review-sessions/{id}/files/{fileId}`：按用户确认调整角色，正文 `{"role":"tender"}`或`{"role":"bid"}`；正式启动后材料冻结。

## 免费报价

`POST /review-sessions/{id}/quote`，正文 `{}`：

```json
{"quoteId":"<UUID>","sessionId":"<UUID>","inputDigest":"<SHA-256>","fixedAmountCents":4090,"feeBreakdown":{"documentPages":409,"images":0,"textBytes":0,"textBlocks":0,"documentCents":4090,"imageCents":0,"textCents":0,"amountCents":4090},"inputFiles":[{"id":"<文件UUID>","name":"招标.pdf","role":"tender","kind":"pdf"}],"expiresAt":"<报价有效期>","paymentTiming":"before_review"}
```

## 新访客单付款

用确认过材料范围的有效报价申请一笔付款单。先生成并保存幂等键，以便响应丢失时查询原单。

```http
POST /api/v1/review-sessions/<UUID>/payments
Authorization: Bearer <本次访客恢复凭据>
Idempotency-Key: <安全生成并保存的16至80位值>
Content-Type: application/json

{"quoteId":"<报价UUID>","confirmedInputDigest":"<该报价的inputDigest>"}
```

服务端返回 `paymentId`、`amountCents`、`expiresAt`、`browserUrl` 和 `qrPath`。先在工具内取回并验证二维码；通过用户可见的同一条消息首次展示报价、本地二维码及付款链接。之前的对话或进度只写材料与处理状态，不转贴接口返回的金额或付款提示。预计耗时从正式评审计算。此时 `runId` 为空，状态为等待付款。到账后服务端自动处理，Agent 查询原会话，无需调用 `/start`。确认放弃时请求取消，状态不确定时恢复原单核实。

## 账户积分启动

API Key 用户确认材料范围和固定报价后，按以下请求启动：

```http
POST /api/v1/review-sessions/<UUID>/start
Authorization: Bearer <本次认证值>
Content-Type: application/json

{"quoteId":"<UUID>","acceptedQuote":true,"confirmedInputDigest":"<报价的inputDigest>"}
```

响应含 runId、sessionId 和 queued 状态。重复请求原报价返回同一运行。API Key 任务在启动时冻结固定金额，报告验证后结算。402 时显示实际缺口及充值链接，由用户决定何时打开浏览器。

## 可选邮箱与进度

用户愿意接收报告时保存邮箱：

```http
PATCH /api/v1/review-sessions/<UUID>/contact
Authorization: Bearer <本次认证值>
Content-Type: application/json

{"email":"<用户明确提供的邮箱>"}
```

跳过时email为null。邮箱按本批保存，不要求绑定账户，也不阻塞分析。

`GET /status?afterSequence=0`返回analysisStatus、billingStatus、deliveryAvailable、billing、reportEmail、emailDelivery及增量事件；`GET /events?afterSequence=<游标>`为短时SSE。支持标准CRLF/LF、多data行和心跳。事件显示后保存游标，hasMoreEvents为true时继续分页，不跳过尚未读取的事件。

```json
{"sessionId":"<UUID>","runId":null,"paymentTiming":"before_review","billingStatus":"awaiting_payment","deliveryAvailable":false,"billing":{"totalCents":4090,"paidCents":0}}
```

- `GET <qrPath>` 返回同单的 `paymentMarkdown`、`qrImageUrl`、`qrPngBase64`、`terminalQr` 和 `browserUrl`。按[本机付款附件](../payments/local-qr.md)先把 PNG 写入用户侧私密临时目录，并用原字符生成离线紧凑预览。Markdown 模板中的相对附件引用必须落实到实际本地文件；`qrImageUrl` 仅用于工具下载。展示本地图片、文本码及付款链接后进入原单等待，按回调或 `/status` 的 `nextPollAfterMs` 续查；付款前通常约 5 秒。
- `GET /review-sessions/{id}/payments/{paymentId}`查询原单，或GET status恢复付款状态。
- account方式自动尝试扣清积分，不调用临时现金payments；不足时引导官网充值。
- 已有运行编号的历史访客任务按原付款时机继续。在原账单已到付款阶段时，以原 `billId` 申请付款，并复用已有订单。尚未启动的旧报价按新流程重新报价出码。

付款响应丢失或状态未知时保留原幂等键查询，不重复创建；只有原单明确closed才能换新幂等键。恢复不是重新运行模型。

## 交付与取消

只有billingStatus为settled且deliveryAvailable为true时，才能GET `/report`或`/report/pdf`，以及读取原文证据。报告未就绪时按状态继续等待，历史报告已完成但待付款时返回402。所有权不符先返回401/404，UUID不能代替认证。

交付接口均位于 `/review-sessions/{id}`：

- `GET /report`：完整已验证报告，`format="review-report"`、`schemaVersion=1`，包含 review、scope、summary、issues。
- `GET /report/summary`：概览，不携带问题大列表。
- `GET /issues?page=1&pageSize=20`：问题分页，支持 q、severity、category、fileId，每页 10/20/50 条。
- `GET /report/pdf`：下载真实 PDF。

问题包含 id、固定 P 编号 code、severity、category、title、analysis、tenderRefs、bidRefs 及可选 recommendation。来源为文件页码或工作表范围。筛选与分页不改变编号。

官网 API Key 可使用 `/analyses/{projectId}/report`、`/report/summary` 和 `/issues`。MCP 的 `get_review_report(task_id,page,page_size,category,q)` 返回当前页问题和 next_page；按同一筛选继续读取，直到 next_page 为空。

服务器负责付费后的可选邮件发送。宿主负责询问桌面、下载文件夹或其他目录，再下载真实PDF并校验。`POST /cancel`可取消尚未完成的分析，本次不收费。

完整流程见[付款](../payments/settlement.md)、[进度](../progress/watch.md)和[交付](../delivery/reports.md)。

所有新审查统一采用高质量精审，包含原始证据核验和独立复核，无需选择效率或深度档位。旧客户端传来的reviewPreference不改变新任务标准；已开始任务恢复时仍沿用其冻结记录。网页、API与PDF使用同一报告版本及材料依据。
可选使用[签名回调](callback.md)接收阶段和资金状态，五分钟内没有新业务记录时主动核对状态。简报用 view=brief，轮询间隔约 60 秒。
