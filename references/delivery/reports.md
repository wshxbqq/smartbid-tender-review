# 付费后的报告与下载

先确认服务器返回 `billingStatus="settled"`、`deliveryAvailable=true`。已付而仍在评审时继续跟进；未结清时按原账单付款；402 不要求重新评审。

## 读取报告

公开报告使用 `format="review-report"`、`schemaVersion=1`，包含 review、scope、summary 和确认问题 issues。保留服务器 P 编号、严重度、分类、标题、分析、招标来源、投标来源及可选建议，不自行重编号或补造结论。

先说明 summary.conclusion 和问题数量，再按需展开问题与来源。按服务端 summary.byCategory 展示分类统计：通常先列资格性、符合性和逻辑性问题，再列本单确有的补充大类，沿用报告给出的类名和数量。零问题使用“本次提交材料中未发现可确认的问题”，保留真实范围限制。风险参考分不是通过率或整体合格结论。

REST 使用 `/report` 读取完整报告，`/report/summary` 读取概览，`/issues?page=1&pageSize=20` 分页读取问题；筛选支持 q、severity、category、fileId，每页 10/20/50 条。MCP 的 `get_review_report(task_id,page,page_size,category,q)` 返回当前页问题和 next_page，保持筛选条件继续读取，直到 next_page 为空。

网页、JSON 与 PDF 使用相同编号、统计、正文和来源。来源按文件显示真实页码或工作表范围；不把模型、供应商、内部运行日志或技术待办写入交付。

## 下载与邮件

`GET /review-sessions/{id}/report/pdf` 下载真实 PDF 字节。先确定保存位置，不覆盖已有用户文件；检查 HTTP 成功、文件非空、PDF 头与 EOF。有服务端摘要时核对下载完整性，最后给出可点击的实际文件路径。

用户填写邮箱后，后台自动发送报告；小于等于 8 MB 的 PDF 直接附送，较大文件使用限定报告的下载链接。邮件失败不影响报告下载，可通过原任务更新邮箱。

已付报告从可交付起保留 30 天，旧规则已完成的待付报告仍自验证完成起保留 24 小时。本机下载副本由用户保管，邮件链接有效期不超过材料剩余保留期。
