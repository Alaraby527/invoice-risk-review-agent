# 企业报销票据重复与异常风险审核 Agent

一个可评测、可追溯、有人机兜底的 AI PM 作品集原型：先可靠取得票据关键字段，再用确定性规则检查批次内重复、历史重复和近似异常。

## 工作流

```text
票据图片 → 文件哈希查重 → 二维码结构化提取
                         ↘ OCR / 视觉模型降级
        → 字段校验 → 历史台账规则 → 自动通过 / 人工复核 / 拒绝
```

## 已验证能力

| 模块 | 评测结果 | 结论 |
| --- | ---: | --- |
| 本地二维码解析 | 5/5 三字段全对 | 作为票号、日期、金额的首选提取路径 |
| Tesseract 本地 OCR | 票号 4/5；日期 2/5；金额 5/5 | 只能作为降级路径 |
| 豆包客户端视觉识别 | 票号 2/5；日期 5/5；金额 5/5 | 人工触发基线，不是 API 集成 |
| 确定性风险规则 | 36/36 | 合成数据规则回归，不是线上拦截率 |

5 张测试图片中有 1 组完全重复图片，共 4 张不同票据。二维码审计还发现并修正了原人工黄金集中的两条漏位票号；修正前标签已在私有目录备份。

## 风险规则

- R10：同一批次文件哈希相同；
- R11：同一批次发票号码相同；
- R20：与历史台账的发票号码相同；
- R30：销售方、金额相同且日期相差不超过 3 天，但号码不同，仅标记为疑似；
- R00/R01：日期、金额或发票号码异常，进入人工复核。

## 本地运行

```powershell
python -m pip install -r requirements.txt
python src/audit.py --self-test
python src/audit.py --incoming data/incoming_invoices.json --ledger data/historical_ledger.json --output audit_results.json
python src/qr_baseline.py --images "<本地票据图片目录>" --gold "<私有黄金集.json>" --private-output "<私有结果.json>" --public-output "eval/qr_baseline_results.json"
```

## 评测证据

- `eval/field_extraction_comparison.md`：三条字段提取路线对比；
- `eval/gold_label_audit.json`：不含真实票号的标签修正记录；
- `eval/qr_baseline_results.json`：二维码公开评测；
- `eval/results_v1.json`：36 条合成规则回归。

## 数据与边界

原始票据、二维码原文、真实字段和逐条客户端结果只保存在本地私有目录。公开数据是脱敏或合成数据。

二维码解析成功只证明字段被读取，不证明发票真实有效。项目尚未接入税务平台验真、红冲或作废状态查询，也没有生产级权限、审计日志和人工复核界面；因此真实风险决策自动放行率仍设为 0%。
