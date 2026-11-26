# 智审标 · 招投标标书审查 Skill

让 Agent 通过[智审标](https://biao.vvcut.cn/)托管的服务完成招投标标书审查：上传招标文件与投标文件，服务端完成格式转换、文字识别（OCR）、多路交叉核验与报告生成，返回**带原件页码定位、可追溯**的审查报告。

> 本仓库只包含 Agent Skill 的 Markdown 指令文件，不包含服务端代码、模型、密钥或用户数据。

---

## 关于智审标

智审标是一款面向投标团队的 AI 标书审查服务，用来在投标文件提交前，把招标要求与投标响应逐项对照，找出容易漏看、难以逐页核对的风险点。

- 官网：<https://biao.vvcut.cn/>
- 示例报告：<https://biao.vvcut.cn/sample-report/>
- 定价说明：<https://biao.vvcut.cn/pricing/>
- 数据安全：<https://biao.vvcut.cn/data-security/>
- 技能安装页：<https://biao.vvcut.cn/skill.md>

服务端负责格式转换、OCR、模型审查、原件核验和报告生成；Agent 只负责确认材料范围、传输原件、展示价格、等待进度和交付结果。即使宿主装有 Word、PDF 或 OCR 工具，也不需要在本机转换整份文件或另写审查结论。

## 能做什么

- **五类核验**：资格条件、符合性、前后逻辑、金额计算、商务条款。
- **汇总复核**：多路并行初审后再统一复核，避免各支路结论互相矛盾。
- **可追溯报告**：每个问题标注严重程度与类别，附原件页码和引文定位，点开即可核对原页。
- **交付方式**：网页查看、证据页核对、PDF 下载；已付报告在服务端保留 30 天。
- **材料格式**：PDF、Word、PPT、Excel、图片、压缩包等常见投标材料。

## 安装

方式一：下载技能包并解压到 Agent 的技能目录。

```sh
mkdir -p ~/.agents/skills
curl -L -o /tmp/smartbid-tender-review.zip https://biao.vvcut.cn/downloads/smartbid-tender-review.zip
unzip -o /tmp/smartbid-tender-review.zip -d ~/.agents/skills
```

方式二：按 <https://biao.vvcut.cn/skill.md> 的说明安装。

安装后让 Agent 读取 `~/.agents/skills/smartbid-tender-review/SKILL.md` 即可开始使用。

## 使用流程

1. 在 Agent 里说明材料所在目录，确认哪份是招标文件、哪份是投标文件。
2. Agent 上传原件并免费预检，按实际页数给出固定总价。
3. 访客扫码付款，或账户确认积分后，服务端自动开始评审。
4. 在进度页跟进预检、OCR、初审、复核和报告生成；报告可交付后查看或下载。

## 计价

按量计价，开审前确认固定总价：

| 项目 | 价格 |
| --- | --- |
| 文档 | 每页 0.10 元 |
| 独立图片 | 每张 0.10 元 |
| 规范化文本 | 每 10KB 0.10 元，不足一档按一档 |

- 模型重试、输出长度和实际成本不改变本批价格。
- 报告验证前取消、系统失败或实际执行超时，实付评审费全额原路退回。
- 已付报告保留 30 天。

## 隐私与边界

- 材料只用于本次审查；恢复凭据、访客 Cookie 和付款信息私密保存，不进入对话记录。
- 进度链接为只读链接，默认 24 小时有效；OAuth 回调等一次性凭据不进入统计或对话。
- 报告只陈述核查结论与存疑，不承诺中标，也不把系统故障写成业务风险。
- 最终投标文件仍建议由专业人员复核后提交。

## 目录结构

```
SKILL.md                     技能入口与行为说明
VERSION                      当前版本
references/
  api/callback.md            任务回调
  api/rest.md                REST 接口契约
  auth/credentials.md        身份与恢复
  delivery/reports.md        报告读取与下载
  errors/recovery.md         错误与恢复
  payments/local-qr.md       本地付款附件
  payments/settlement.md     报价、付款与结算
  progress/watch.md          进度跟进与恢复
  uploads/preflight.md       材料范围与免费预检
```

## 版本

当前版本见 [`VERSION`](./VERSION)，与 <https://biao.vvcut.cn/skill.md> 发布的技能包保持一致。

## 参与贡献

欢迎提交 Issue 反馈使用问题、文档歧义或改进建议。涉及服务端行为的问题请附会话 UUID 和公开错误码，不要粘贴 API Key、Cookie、任务恢复信息或付款凭据。

## License

[MIT](./LICENSE)

---

## English

**SmartBid Tender Review Skill** — an Agent Skill that connects an Agent to the hosted [SmartBid](https://biao.vvcut.cn/) service for Chinese tender/bid document review. The Agent uploads the tender and bid files; the server performs OCR, cross-checking and report generation, returning a traceable report with original page references. This repository contains only the Markdown skill instructions, no server code or secrets. Licensed under MIT.

## 特别感谢

感谢智审标（SmartBid）。智审标提供招投标文件审查，支持核对资格条件、条款响应与报价，审查报告附原件页码供复核。

[智审标官网](https://biao.vvcut.cn/) · [智审标开源 Skill](https://github.com/wshxbqq/smartbid-tender-review)
