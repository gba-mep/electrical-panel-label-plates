# electrical-panel-label-plates · 电箱标签牌自动化

---
<div align="center">

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](https://python.org)
[![Part of the MEP Automation Toolkit](https://img.shields.io/badge/Toolkit-MEP%20automation-1565C0?logo=github&logoColor=white)](https://github.com/gba-mep)

**Full automation pipeline for electrical panel label plates**

CAD SLD → parsed circuit table → Word label plates → CAD vector fabrication drawing

[快速开始](#工作流) · [文件结构](#文件结构) · [技术栈](#技术栈)

</div>

![电箱标签牌全自动管线](assets/pipeline-overview.jpg)

---

> 从 CAD 单线图（DWG/DXF）→ 解析回路表（JSON）→ Word 标签牌（docx）→ CAD 向量加工图（DXF/SVG，1:1 尺寸注解）。全自动管线。

## 解决什么问题

电气工程中「电箱标签牌」= 贴在配电箱内每个断路器下方的细标识条，黑底白字，写「用途 + 回路编号」。广告商据此印制后贴到现场每个 MCB/RCD 上。

人手做的痛点：
- 几十个回路逐个打字，容易出错
- RCD 极数判断错（最常见、最致命）
- 栏宽不一致，印出来歪歪斜斜
- 要转 CAD 加工图给广告商，重新画一次

**electrical-panel-label-plates** 将整个流程标准化、脚本化。

## 核心特性

### ⚡ 两种 SLD 格式支援
- **Format A** — 纵向量表格式
- **Format B** — 横向网格格式

### 🎯 17:25 斜线数原则
RCD 极数 = 单线图符号的**斜线数目**，不是看额定值。
这个是最容易犯、亦最致命的错误。

### 👤 Human-in-the-Loop 确认关卡
RCD 极数自动判断后，必须人工确认先至生成。

### 📐 固定布局栏宽锁定
- `tblLayout=fixed` 锁死栏宽
- 黑底 / 白字 / 黑体字体规格
- A3 / A4 自动纸张尺寸选择

### 🛠️ CAD 向量输出
- DXF + SVG 双格式
- 1:1 尺寸注解
- 逐行 MTEXT 定位（避免 AutoCAD 渲染溢出）
- RCD 栏自动合并侦测

### 📏 双行字幕系统
- `balance_lines` + `cell_lines`
- 2 行 cap 机制，确保不会超出标签高度

### ✅ 完整验证套件
- `verify_label_widths.py` — 宽度验证
- `verify_label_cad.py` — CAD 输出验证
- `render_check_label_cad.py` — 无头自渲染检查
- DXF → DWG 转换（AutoCAD script）
- Pytest 回归测试闸

### 🐛 21 个 documented pitfalls
Word 排版 / SLD 解析 / CAD 输出三大类坑位全记录。

## 工作流

```
CAD 单线图 (DWG)
    ↓
DXF 转换
    ↓
解析回路表 (JSON)
    ↓
RCD 极数自动判断（17:25 原则）
    ↓
👤 人工确认关卡
    ↓
生成 Word 标签牌 (docx)
    ↓ （可选 --cad）
CAD 向量加工图 (DXF / SVG，带尺寸注解)
```

## 适用场景

| 场景 | 例子 |
|:---|:---|
| 从单线图生成标签牌 | 新装修/改装工程电箱标识 |
| 由 Word 转 CAD 加工图 | 给广告商/加工场 1:1 落料 |
| 批量调整标签内容 | 更改回路名称、编号 |
| 现有标签牌样式核对 | 验证栏宽、字体、格式 |

**不适用**：电箱大样图/arrangement drawing；材料报批。

## 文件结构

```
electrical-panel-label-plates/
├── README.md       # 本文件
└── DOCUMENTATION.md        # 完整技能文档（规格 + 工作流 + 坑位大全）
```

## 技术栈

- **Python** + **ezdxf** — DXF 解析与 CAD 输出
- **python-docx** — Word 标签牌生成
- **PyMuPDF** — PDF 渲染与验证

## 两条用户铁律

1. **只出 Word（.docx），不出 PNG 图。** 除非用户明确要求图片（如给广告商的规格标注图特例）。
2. **用户手改过的 docx 只准读取理解，禁止覆写。** 重新生成时输出必须去 temp 或加 `_v2` 后缀。

详细用法请参阅 [DOCUMENTATION.md](DOCUMENTATION.md)。

---
## License

MIT License — feel free to use, modify, and share.

---

## Related repositories

Part of the **[MEP & construction document automation toolkit](https://github.com/gba-mep)** — open-source tools built from real jobsite workflows.

- **Handbook** — [ai-agent-manual](https://github.com/gba-mep/ai-agent-manual) (8-level AI cultivation for engineers)
- **Document generation** — [material-approval-pipeline](https://github.com/gba-mep/material-approval-pipeline) · [material-submittal-generator](https://github.com/gba-mep/material-submittal-generator) · [excel-template-filler](https://github.com/gba-mep/excel-template-filler) · [python-docx-photo-grid](https://github.com/gba-mep/python-docx-photo-grid) · [daily-construction-log](https://github.com/gba-mep/daily-construction-log) · [officecli-workflow](https://github.com/gba-mep/officecli-workflow)
- **Engineering calculation** — [lighting-lux-calculator](https://github.com/gba-mep/lighting-lux-calculator) · [ups-discharge-time-calculator](https://github.com/gba-mep/ups-discharge-time-calculator) · [gantt-chart-pro](https://github.com/gba-mep/gantt-chart-pro) · [electrical-test-report-generator](https://github.com/gba-mep/electrical-test-report-generator)
- **Data & OCR** — [ocr-skill](https://github.com/gba-mep/ocr-skill) · [VBA-Macro-Reader-v2.0.0](https://github.com/gba-mep/VBA-Macro-Reader-v2.0.0)
- **Compliance & AI ops** — [confined-space-planner](https://github.com/gba-mep/confined-space-planner) · [路由规则](https://github.com/gba-mep/路由规则) · [consulting-services](https://github.com/gba-mep/consulting-services)
