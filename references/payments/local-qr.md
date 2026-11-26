# 保存本地付款附件

展示任何付款图片之前，在用户实际接收附件的设备或宿主文件工作区处理本次同源取码回执。首选在工具进程内严格解码 `qrPngBase64` 并保存原始 PNG 字节。若旧服务只有图片下载地址，在工具内从同源下载，核对成功状态、订单及 PNG 格式后落盘。`qrImageUrl` 仅供取文件，不得写入给用户的图片 Markdown。Base64、恢复凭据和完整取码 JSON 不进入对话。

以下 Python 标准库示例读取私密回执文件。三个参数依次为取码回执路径、已保存的订单回执路径和本次站点的 API 基础地址。工具仅输出已写入的路径。其他宿主可用等价的本地文件和附件能力。

```python
import base64
import json
import os
import re
import struct
import sys
import tempfile
from itertools import groupby
from pathlib import Path
from urllib.parse import quote, urlsplit

qr_path, order_path, api_base = sys.argv[1:]
qr = json.loads(Path(qr_path).read_text(encoding="utf-8"))
order = json.loads(Path(order_path).read_text(encoding="utf-8"))
for key in ("paymentId", "sessionId", "amountCents", "browserUrl"):
    if qr[key] != order[key]:
        raise ValueError("二维码与本次订单不一致")
link, site = urlsplit(qr["browserUrl"]), urlsplit(api_base)
if (site.scheme not in ("https", "http") or not site.netloc or link.username or link.password
        or (link.scheme, link.netloc) != (site.scheme, site.netloc)):
    raise ValueError("付款入口所属站点不一致")

png = base64.b64decode(qr["qrPngBase64"], validate=True)
if (len(png) < 45 or not png.startswith(b"\x89PNG\r\n\x1a\n\x00\x00\x00\rIHDR")
        or not png.endswith(b"\x00\x00\x00\x00IEND\xaeB`\x82")):
    raise ValueError("付款图片不是完整 PNG")
width, height = struct.unpack(">II", png[16:24])
if width == 0 or width != height:
    raise ValueError("付款图片尺寸无效")

rows = qr["terminalQr"].split("\n")
side = len(rows[0])
if (not 1 <= side <= 185 or len(rows) != (side + 1) // 2
        or any(len(row) != side or not re.fullmatch("[ ▀▄█]+", row) for row in rows)):
    raise ValueError("付款文本码格式无效")

# 每个字符代表上下两个方格；等宽字符仍可选择复制，黑白格由背景绘制，不依赖字体行高。
unit = max(2, min(8, 304 // side))
colors = {" ": "white", "█": "black", "▀": "upper", "▄": "lower"}
rendered = []
for row in rows:
    spans = []
    for char, group in groupby(row):
        run = "".join(group)
        spans.append(f'<span class="{colors[char]}" style="width:{len(run)*unit}px">{run}</span>')
    rendered.append('<div class="row">' + "".join(spans) + '</div>')
preview = """<!doctype html><html lang="zh-CN"><meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta http-equiv="Content-Security-Policy" content="default-src 'none'; style-src 'unsafe-inline'">
<title>支付宝付款文本码 · 紧凑预览</title>
<style>
:root{color-scheme:only light;background:white}
body{margin:0;min-height:100vh;display:grid;place-items:center}
main{overflow:hidden;forced-color-adjust:none;user-select:all}
.row{display:flex;white-space:pre;height:var(--row);line-height:var(--row)}
.row span{flex:none;overflow:hidden;color:transparent;font-family:monospace}
.white{background:#fff}.black{background:#000}
.upper{background:linear-gradient(to bottom,#000 50%,#fff 50%)}
.lower{background:linear-gradient(to bottom,#fff 50%,#000 50%)}
</style>"""
preview += (f'<main aria-label="支付宝付款文本码" style="width:{side*unit}px;'
            f'height:{side*unit}px;--row:{2*unit}px">' + "".join(rendered) + "</main></html>")

os.umask(0o077)
directory = Path(tempfile.mkdtemp(prefix="smartbid-pay-")).resolve()
image = directory / "alipay-qr.png"
compact = directory / "alipay-qr-compact.html"
image.write_bytes(png)
compact.write_text(preview, encoding="utf-8")
if image.read_bytes() != png:
    raise OSError("付款图片写入不完整")
display = (f'![支付宝付款二维码](<{quote(image.as_posix(), safe="/:")}>)\n\n'
           f'[紧凑查看文本码](<{quote(compact.as_posix(), safe="/:")}>)\n\n'
           + "```text\n" + qr["terminalQr"] + "\n```\n\n"
           + f'[支付宝付款链接]({qr["browserUrl"]})')
print(json.dumps({"imagePath": str(image), "compactPath": str(compact), "displayMarkdown": display},
                 ensure_ascii=False))
```

将两个文件的实际路径与本次订单号保存在私密恢复记录里。对支持本机绝对路径的宿主，向用户原样展示 `displayMarkdown`；若使用附件 API，先登记文件并换成真实的用户侧附件标识。付款等待期间继续查询原单即可；重新展示时先确认订单仍有效、文件仍存在。临时目录被清理或服务返回新码时重新保存同单附件。

在同一条回复中依次展示以下内容：

1. 用 `imagePath` 实际发送本地 PNG 图片附件。宿主支持本地 Markdown 图片时，可使用 `![支付宝付款二维码](<实际本地路径>)`；只支持文件链接的终端提供可打开的本地 PNG 文件。远程运行的宿主应先把文件交给用户侧的附件系统，返回用户可访问的附件标识。
2. 在一个 `text` 代码块内保留原始 `terminalQr`。紧邻代码块给出「紧凑查看文本码」的本地 `compactPath` 文件链接。宿主的代码块行高由客户端控制；紧凑视图离线打开，按固定正方格绘制相同的 UTF-8 文本码，无需修改用户设置或自动打开浏览器。
3. 最后一行用本单真实的 `browserUrl` 给出支付宝付款链接。

`paymentMarkdown` 中的 `./alipay-qr.png` 和 `./alipay-qr-compact.html` 是本地相对文件名。仅当 Markdown 文件与附件保存在同一目录时直接使用；在对话里替换成宿主实际可访问的本地附件。旧版模板若包含网络图片引用，应按以上顺序用本地文件重新组合。宿主不支持打开 HTML 时仍展示已保存的 PNG、原始文本码和付款链接。若 PNG 写入失败，先换用可写临时目录重试；仍失败时如实说明，保留文本码、链接及原单等待。
