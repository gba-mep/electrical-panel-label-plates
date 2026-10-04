---
name: electrical-panel-label-plates
description: 机电工程「电箱标签牌」全自动制作技能。当用户要从 CAD 单线图（SLD / DWG / DXF）读出配电箱回路表并生成黑底白字 Word 标签牌（贴在每个断路器 MCB/RCD 下方的用途+回路编号条），或要由 Word 标签牌转成带尺寸标注的 CAD 向量加工图（DXF / SVG）给广告商/加工场 1:1 落料时使用。触发词：电箱标签、配电箱标签牌、回路标签、面板标签、MCB 标签、RCD 标签、单线图回路表、电气标签牌、标签牌 CAD 加工图、label plates、panel schedule to docx / to dxf。覆盖两种 SLD 网格格式（A 纵向量表 / B 横向网格），含 RCD 极数人工确认门槛（17:25 斜线数原则）、fixed-layout 栏宽锁定、verify gate。只出 Word 不出 PNG；可加 CAD 向量加工图。
---

# 电箱标签牌制作技能（electrical-panel-label-plates）

## 0. 这个技能做什么

机电工程中「电箱标签牌」= 贴在配电箱（电箱）内每个断路器下方的小标识条，黑底白字，写「用途 + 回路编号」。广告商据此印制后贴到现场每个 MCB/RCD 上。

本技能把流程标准化、脚本化：

```
CAD 单线图 (DWG) → DXF → 解析回路表 (JSON) → 人工确认 RCD 极数 → 生成 Word 标签牌 (docx)
                                                                      ↓（可选 --cad）
                                                           CAD 向量加工图 (DXF / SVG，带尺寸标注)
```

**本技能脚本路径（脚本全在这里，本技能只引用、不复制代码）：**
`<SKILL_DIR>/`
- 脚本：`scripts/` 子目录（30 个，见 §7 索引）
- 文档：规格文档 00~08 + README
- 回路表 JSON：示例项目电箱（多个版本）

**新任务直接照 §6 工作流做，不要重新发明。**

---

## 1. 触发条件

- 用户给出 DWG / DXF 单线图，要生成「电箱标签牌 / 配电箱标签 / 回路标签 / MCB 标识条」。
- 用户已有回路表 JSON，要重新生成或调整 Word 标签牌。
- 用户要由 Word 标签牌转 CAD 加工图（带尺寸标注的 DXF / SVG，给广告商/加工场落料）。
- 用户问「这个 RCD 是几 P」「栏宽对不对」「标签牌格式」「标签牌 CAD 图」等。

**不适用**：若用户只要「电箱大样图 / arrangement drawing」→ 改用 `cad-sld-to-arrangement` 技能。若用户要「材料报批」→ 改用 `material-approval-pipeline`。

---

## 2. 两条用户铁律（最高优先级，违反即返工）

1. **只出 Word（.docx），不出 PNG 图。** 除非用户明确要求图片（如给广告商的规格标注图特例）。
2. **用户手改过的 docx 只准 python-docx 读取理解，禁止覆写。** 重新生成时输出必须去 `<TEMP_DIR>/` 或加 `_v2`/`_修正版` 后缀，绝不直接覆盖用户已确认的文件。

---

## 3. ⚠️ 铁律：RCD 极数 = 单线图符号「斜线数」（17:25 原则，通用）

> 这是本技能最容易犯、也最致命的错误。

**RCD（漏电断路器）是几 P（极），一律以单线图 RCD 符号里的「斜线数」判定，绝对不可以按 rating（额定电流/漏电电流）硬推算或固定。**

- 例如 `25A / 300mA` 可以是 **2P** 也可以是 **4P** —— 逐图看符号斜线数决定，不可当成固定值。
- `30mA` 多数 2P、`300mA` 多数 4P 只是**粗略 heuristic，仅供快速估计，最终以符号斜线数 + 用户确认为准**。
- **馈线 MCCB（P1 / P2 / 备用）是例外**：不用符号斜线规则，改用 DXF 三相位置法判定（三个等距 32A 文字在三相位置 = 3P）。

**强制流程保障（human-in-the-loop）**：解析器输出 `pole_status = "NEEDS_HUMAN_REVIEW"`，编排器在生成前会拦截，要求先跑 `--review-poles` 人工确认。详见 §5。

---

## 4. 输出规格（标签牌）

| 项目 | 值 |
|---|---|
| 底色 / 字色 | 黑底 `000000` / 白字（自动）/ 白边框 0.5pt（sz=4）|
| 字型 | SimHei（黑体）|
| 行高 | 上行用途 1758 twips（3.1cm）/ 下行编号 510 twips（0.9cm），exact |
| 栏宽（必须 `tblLayout=fixed`）| MCB 1P = 1.8cm；RCD = 1.8×P（2P=3.6cm、4P=7.2cm）；馈线 3P = 5.4cm |
| 纸张 | 回路数 > 22 用 A3 横向（42×29.7cm），否则 A4 横向；边距 1.27cm |
| 结构 | 每个 RCD 配一组（含其下 6~12 个 MCB 回路栏），备（RE）全显示；无 RCD 的尾组单独成表 |

**栏宽锁死靠 `tblLayout=fixed`**：否则 Word 会把每张表自动拉到填满页宽，让 3P/1P/4P 的宽度差在不同表之间看不出差别。生成器已清除 `table.autofit=False` 自动注入的重复 `tblLayout`，并对每格清旧 `tcW` 再加。

---

## 5. 两阶段工作流（编排器入口）

**唯一入口**：`make_panel_labels.py`（在 KB `scripts/` 下）。它自动：解析输入 → 判断 SLD 格式 → （可选）印极数确认表 → 生成 docx → 强制跑 verify gate → （可选）生成 CAD 向量加工图。

### 阶段一：解析 + 人工确认极数（不直接生成）
```bash
# 从 DWG 解析并只印 RCD 极数待确认表（17:25 原则），不生成
PYTHON_EXE="<cad venv python>" python make_panel_labels.py \
    --input 单线图.dwg --out 标签.docx --review-poles

# 用户按单线图符号斜线数核对后，确认写回 JSON（pole_status=CONFIRMED）
PYTHON_EXE="<cad venv python>" python make_panel_labels.py \
    --input 单线图.dwg --out 标签.docx --review-poles --review-confirmed
```
`--review-poles` 会列出每个面板的 RCD 及其「建议极数」，并提醒「请按图符号斜线数确认」。加 `--review-confirmed` 才把 `CONFIRMED` 写回 JSON（格式 B 同时按 mA 写 `rcd_poles`）。

### 阶段二：生成（极数须已确认）
```bash
# JSON 已确认 → 直接生成 Word
PYTHON_EXE="<cad venv python>" python make_panel_labels.py \
    --input 回路表.json --out 标签.docx

# 自动判格式 + A3 纸
PYTHON_EXE="<cad venv python>" python make_panel_labels.py \
    --input 单线图.dwg --out 标签.docx --paper A3

# 生成 docx 同时产 CAD 加工图（DXF + SVG，带尺寸标注）
PYTHON_EXE="<cad venv python>" python make_panel_labels.py \
    --input 回路表.json --out 标签.docx --cad --cad-svg
```
**门禁**：若 JSON 仍有面板 `pole_status == "NEEDS_HUMAN_REVIEW"` 且未加 `--force`，编排器直接 `sys.exit` 拒绝生成（强制走人工确认）。生成后**强制**跑 `verify_label_widths.py --gate`，栏宽不在 `{1.8, 3.6, 5.4, 7.2}cm` 即报错退出。

> `--force` 跳过极数门禁（不推荐，会绕过 17:25 保护）。`--skip-verify` 仅 debug 用。
> `--cad` 在 docx 生成后调用 `gen_label_cad.py`；`--cad-out` 指定输出路径（`.dxf`/`.svg`），默认 = docx 同名 `.dxf`；`--cad-svg` 同时多写一份 `.svg`。

---

## 6. 标准工作流（新任务照做）

```
1. 收到 DWG 单线图
   ↓
2. DWG→DXF（accoreconsole，须用 ASCII 临时路径，中文路径会静默失败；stdout 是 UTF-16LE 勿在 Python subprocess 读）
   ↓
3. explore_dxf.py 探索结构 → 判断 SLD 格式
   - 有 "N CIRCUITO" → 格式 B（横向网格）→ parse_el01.py
   - 有「编号/用途/功率(kVA)」中文行 → 格式 A（纵向量表）→ parse_sld_panels.py
   ↓
4. 解析 → 回路表 JSON（含 RCD 分组 + pole_status=NEEDS_HUMAN_REVIEW）
   ⚠️ 分组铁律见 §8；极数见 §3（17:25）
   ↓
5. 交叉核对：找同目录「落实版 PDF」，fitz 提取文字核对（用 --pdf 可自动比）
   ↓
6. 阶段一：--review-poles 印极数表 → 用户按斜线数确认 → --review-confirmed 写回
   ↓
7. 阶段二：--input 回路表.json 生成 Word 标签牌（只出 Word！）
   ↓
7.5 （如需 CAD 加工图）编排器加 --cad [--cad-svg]，或单独跑 gen_label_cad.py --docx 标签.docx --out 标签.dxf [--svg]
   ↓
8. 交付：docx（必出）+ dxf/svg（加工用，按需）放用户任务资料夹；同时把回路表 JSON + 脚本落盘到本技能目录
```

---

## 7. 脚本索引（KB `scripts/` 下，编排器已串好路径）

| 脚本 | 用途 |
|---|---|
| `make_panel_labels.py` | **编排入口**（解析→格式判断→极数确认→生成→verify gate→可选 CAD）|
| `parse_sld_panels.py` | 格式 A（纵向量表）解析器 → JSON（`--dxf`/`--out`）|
| `parse_el01.py` | 格式 B（横向网格）解析器 → JSON（`--dxf`/`--out`）；v4 含 `group_by_rccb_span()` 几何跨度量度分组 |
| `gen_panel_labels_v2.py` | 格式 A 生成器（`--json`/`--out`/`--paper`）|
| `gen_label_docx_el01.py` | 格式 B 生成器（`--json`/`--out`/`--paper`）|
| `verify_label_widths.py` | 栏宽校验 gate（`--docx`/`--gate`，退出码 1=不合规）|
| `gen_label_cad.py` | **Word → CAD 向量加工图**（DXF/SVG 带尺寸标注；读 docx 栏宽/行高 1:1 还原）|
| `verify_label_cad.py` | ⭐ **CAD 加工图验证器**：查每格 ≤2 行 + DXF/SVG 全部文字入框（正常行距 + 行距 2x 最坏情况）；`--glob` 避中文路径坑 |
| `regen_label_cad.py` | ✅ **重生成驱动**：glob 解析 docx 路径，一句重生成 DXF+SVG（RCD 栏**自动合并**，不用 flag；`--no-svg`）|
| `render_check_label_cad.py` | ✅ **无头自渲染自检**：ezdxf→PNG 自渲染 + 几何自检（入框/居中/重叠/相撞/RCD 行数），不用等用户截图；`--glob`/`--no-render`/`--png` |
| `dx2dwg_scr.py` | ✅ **DXF→DWG 交接地**：由 DXF 生成 AutoCAD Script (.scr)，可 `--console` 自动找 accoreconsole 无头转档出 .dwg |
| `tests/test_label_cad.py` | ✅ **pytest regression gate**：自制 fixture docx 生成 DXF+SVG 后跑全部几何自检 + 验证 RCD auto-merge 真·合并；`python -m pytest tests/ -q` |
| `gen_big_labels_word.py` | 电箱名牌大标签 Word（80×50mm，48pt 两行，用户确认版）|
| `explore_dxf.py` | DXF 结构探索（诊断）|
| `explore_feeder_p1p2.py` / `render_feeder.py` | 馈线 MCCB 三相位置法极数判定 |
| `analyze_rccb_span*.py` / `explore_rccb_geo.py` | RCD 几何跨度量度调试 |
| 其余 | 早期探索/测量脚本，新任务一般不需直接调用 |

**执行环境**：须用装有 `ezdxf` + `python-docx` + `PyMuPDF` 的 Python。本机用：
`<CAD_PYTHON>`
编排器默认取 `PYTHON_EXE` 环境变量，否则用运行本文件的解释器。

---

## 8. ⚠️ 分组铁律（格式 B 最重要，取代固定 6 个规则）

每个 RCD 的回路数 = **RCCB 在图中横跨的竖线范围内上方连接的回路 MCB 数**（一般 6~12、大电箱 6~9，**无固定数，不准硬编 6**）。

- **Primary = 几何跨度量度**（`group_by_rccb_span()`）：① 收集 RCCB 行竖线；② 按 x 间距分组（gap > max(1.5, 栏距×1.5)）；③ 每组竖线数 = 回路数；④ 竖线总数 = 回路数才算对齐成功，否则 fallback。
- **Fallback = RCCB 文字锚点对齐**（实测 x 对齐群组内第 4 个回路栏，Δx≈-0.07 → `circuits[anchor_idx-3 .. anchor_idx+2]`）。
- **不要用「每个回路配右边最近 RCCB」**（会令第一组只得 4~5 个）。
- **单一主 RCD 例外**：电箱只有 1 个 RCCB → 覆盖全箱所有回路。备（RE）都要入组显示。
- **连续无 RCD 组合并为单一尾组。**
- 面板间文字污染坑：RCCB 文字(mA)在不同面板有相近 y，须用各面板精确 `PANEL_Y` 分离。

---

## 9. 踩过的坑（绝对不要再踩）

### Word/排版类
1. **twips 换算**：1cm = 567 twips。80mm = 8cm = 4536（不是 `int(80*567)`=45360，那是 80cm！）。50mm = 2835。
2. **OOXML 唯一元素要先清再加**：`w:tcW`、`w:tblGrid` 等 schema 规定只一个，不清旧就 add 会出现两个值，Word 用错那个。
3. **run rPr 用高层 API**：`run.bold` / `run.font.size` 会自动创建 rPr；别用 `OxmlElement('w:rPr')` 从零搭（schema 顺序问题会令 run 完全空白）。
4. **`cell.text=''` 怪行为**：会保留第一个空 paragraph，后续 `add_run` 可能失效。安全做法：彻底删所有 paragraphs 再 rebuild。
5. **浮动表格 + tblLayout=fixed + tblGrid 明确 gridCol widths** 才锁得住栏宽（48pt 中文字才不爆框）。

### 流程/规矩类
6. 只出 Word 不出 PNG（§2）。
7. 用户手改过的 docx 禁止覆写（§2）。
8. 电箱名牌大标签格式（用户确认）：80×50mm、两行都 48pt 粗体、名称统一「配电箱」（不带位置后缀）、总开关 64pt 粗体单行、无规格说明栏。

### SLD 解析类
9. SLD 两种排版格式（§6 步骤 3 判断）：A 纵向量表 / B 横向网格。
10. 格式 A：DWG 文字用 `align_point`（群码 11/21/31）非 insert，差 4 twips 会抢配对 → greedy unique 配对（text 不重用 + 有 kVA 跳过「备用」）。
11. 格式 B：`x_nc` 必须用「N CIRCUITO」文字本身的 x（别用 y 带 min x——会拿到左边「A-1-01」等文字）；RCCB 标记要独立取（别用每个回路最近邻，会碎成 20+ 组）。
12. 分组铁律见 §8。**RCD 极数见 §3（17:25 斜线数原则），绝不可按 rating 硬编。**

### CAD 转档类
13. DWG→DXF：accoreconsole `_DXFOUT` 到 **ASCII 临时路径**（中文路径会静默失败）；stdout 是 UTF-16LE，别在 Python subprocess 读。
14. ezdxf 只读 DXF 不读 DWG（先转 DXF）；COM 并发先 `taskkill /F /IM acad.exe` + sleep。
15. **DXF 尺寸标注**：用 `msp.add_linear_dim(...).render()`；绘制单位设 `INSUNITS=5`（cm）令标注量度值直接显示 cm。dimstyle 设 `dimtxt`/`dimgap`/`dimasz`/`dimlunit=2`/`dimdec=2`。
16. **⚠️ DXF 文字出框（最高优先）**：**绝对不可以用「整栏一个 MTEXT + 空白行推落 + 摆板垂直中心 + MIDDLE_CENTER」**。这招 AutoCAD 实际渲染时 `line_spacing_factor` / `MIDDLE_CENTER` 解读差异会令成块字高出框外。**正确做法 = 逐 row 一个 MTEXT、垂直置中在该 row band 内**。每 row ≤2 行 → 字块 ≤0.96cm ≪ row_h，绝对入框。SVG 天生逐 row 定位无事。验证用 `verify_label_cad.py`（座标解析，不靠目测）。
17. **⚠️ 路径字符坑**：用户资料夹路径如果有特殊字/罕用字，**不要在 bash 手打写死路径**（shell Unicode 编码会静默失败，cp/test 都无反应）；要用 `glob.glob()` 在 Python 内部解析（见 `regen_label_cad.py` / `verify_label_cad.py` 的 `--glob`）。
18. **⚠️ verify 入框检查会「假 pass」**：整张图框（最外围 sheet border）都是 color=1 LWPOLYLINE、面积最大，会包住所有文字。`_plates_from_dxf` 必须剔走面积最大那块（= sheet 框），净低先是各电箱板；否则 in-bounds 检查永远真。**自己写 DXF 板侦测时切记排除 sheet 框**。
19. **⚠️ DXF 文字样式 font 只接受档名**：`doc.styles.add("SIM", font=...)` 的 `font` 属性只收字型**档名**（如 `simhei.ttf`），**不可以给完整 Windows 路径**。给完整路径 → AutoCAD 找不到字型 → 自动替换另一只字型 → 字形 metrics 改变 → 文字可能出框/错位（修正：`font_name = os.path.basename(font)`）。精确字形量度如需完整路径，另存 `FONT_TTF_PATH[0]` 供 verify 用。
20. **⚠️ 字型大小有意妥协（1:1 几何 vs Word 实际 pt）**：CAD 输出 `TEXT_H=0.30cm` 是**固定常量**，不再由 Word `run.font.size`（pt→cm）推算。原因：Word pt 字高转 cm 后偏细、实体标签难读；加工场要「每格清清楚楚」、统一字高比跟 Word 原 pt 重要。**几何（栏宽/行高/对齐/位置）100% 跟 Word，唯独字高统一 0.30cm**。若日后要用 Word 原 pt，改令 `TEXT_H` 由 `font_pt` 推算并重跑 verify。
21. **⚠️ RCD 合并 flag 改名**：旧 `--merge-rcd`（开启合并）已改成**默认自动合并**；新 flag 是 `--no-merge-rcd`（关闭）。手动落指令时**不要再传 `--merge-rcd`**（会变 unrecognized argument 报错）。`regen_label_cad.py` 已不用传任何 RCD flag。

---

## 10. 已知卡点（环境限制，照现状处理）

- **DWG→PDF 自动转档**：accoreconsole PLOT（prompt 乱码 + 纸张名格式敏感）、AutoCAD COM（gen_py cache bug）、PowerShell COM（被安全策略挡）三种方法都有环境阻碍。现状方案：优先检查资料夹有无「落实版 PDF」，有就直接用 fitz 提取文字核对；无则告知用户需人手转。
- **卡点不阻断标签牌制作**：标签牌只需回路用途 + 编号（已能从 DWG 解析），PDF 核对只是附加保险。

---

## 11. CAD 向量加工图输出（Word → DXF / SVG 带尺寸标注）【新增能力】

- **目的**：广告商/加工场要 1:1 向量档落料。由 Word 标签牌 docx 读每张 table 的**栏宽（tcW）** + **行高（trHeight）**（单位 cm，python-docx 直接返回准确 cm），1:1 还原几何，再加 dimension 标注（尺寸标柱）。
- **脚本**：`gen_label_cad.py`
  - 必填：`--docx 标签.docx`
  - 输出：`--out 路径.dxf`（默认）或 `--out 路径.svg`；`--svg` 在 DXF 之外再多写一份 sibling `.svg`
  - 可选：`--sheet-w 42`（图框宽 cm，自动取 max(此值, 最宽板+边距)）、`--no-percol`（跳过每栏宽标注，只留总宽/总高）、`--title "..."`、**RCD 宽列自动合并**（默认侦测含「漏电/RCD/RCCB」最左栏即合并为一格高 cell；`--no-merge-rcd` 可关掉）
- **输出格式怎么选（重要）**：
  - **DXF（推荐）**：属于用户所列 `.DWG / .dxf` 家族，AutoCAD / CorelDRAW / Illustrator **均可直接「汇入」**，1:1 保留尺寸与标柱。
  - **SVG**：Illustrator / CorelDRAW 汇入最干净（黑底板 + 白字 + 红色标柱线），适合直接进排版。
  - **.ai / .cdr 原生格式**：属专有封闭格式，脚本**无法直接产生**；但 DXF 与 SVG 都可被 Adobe Illustrator 与 CorelDRAW 直接开启/汇入并保留 1:1 尺寸。若硬要 `.ai`/`.cdr`，在该软件「档案→汇入」DXF/SVG 后「另存」即可。
- **尺寸标柱内容（每张板）**：
  - 总宽：板下水平标注（cm）
  - 每栏宽：板下第二行，对齐各栏（cm）—— 即「规格尺寸标柱」核心
  - 总高：板右垂直标注（cm）
  - 每格保留用途/编号文字（黑底白字）
- **版面**：所有板排进一张图框（A3 宽 42cm 起，自动换行），含图框线 + 标题 + 图例。
- **已验证 1:1 准确**：DXF 量度值 = Word 栏宽之和（例：表1 = 7.2 四P + 6×1.8 一P = 18.0cm），回读 `DIMENSION.get_measurement()` 实测 18.009 / 7.204 / 1.801 cm，与 Word 栏宽一致。

### 11.1 文字必须与 Word 文档「一模一样」（关键铁律）

用户明确要求：DXF/SVG 的**文字排列、位置、大小**要和来源 Word 文档**完全相同**，不能自行估算。

- **字型大小（用户最终指定）**：字高统一 0.30cm（`TEXT_H=0.30`），不再用 Word 实际 pt。所有标签文字（用途/编号/RCD）一律 0.30cm；图面标题另设 0.6cm。
- **对齐（用户最终指定）**：多行文字**水平居中 + 垂直置中**（horizontal CENTER、vertical CENTER——整组多行在格内居中、行与行等距上下对齐）。DXF `MIDDLE_CENTER`、SVG `text-anchor="middle"`，x 取 `xs[c]+col_w[c]/2`（格水平中心）。**行距**：行与行间隙 `LINE_GAP=0.18cm`（用户指定 0.15–0.22 宽松，不再用倍率），`line_h = TEXT_H + LINE_GAP = 0.48cm`。RCD 合并高格同样**水平居中 + 垂直置中**。
- **文字框结构（用户第 5 次确认「不要一行一字一个框」+ 第 7 次确认「字必须在框内」）**：
  - **SVG**：每栏 ONE `<text>`（内含多个 `<tspan>` 行），每行列 band 中心 y、`text-anchor="middle"`。`cell_lines` 封顶 2 行。
  - **DXF（关键修复）**：**不再用「整栏一个 MTEXT + 空白行推落 + 摆板垂直中心」**——那招在 AutoCAD 实际渲染时，`line_spacing_factor` / `MIDDLE_CENTER` 解读差异会令成块字高出框外。改为**逐 row 一个 MTEXT、垂直置中在该 row band 内**（同 SVG 逻辑一致）：`cx=xs[c]+col_w[c]/2`、`cy=y_mid_r=(top_r+bot_r)/2`、`attachment_point=5`。每 row 最多 2 行 → 字块高 ≤2×line_h≈0.96cm ≪ row_h(≥0.9cm)，**绝对出不到框**。RCD 合并格仍 ONE MTEXT 摆板中心（跨整块板高、2 行、板高够高无溢出）。DXF 内文前置 `\pqc;`（段落水平居中）+ `line_spacing_style=2`、`line_spacing_factor=line_h/TEXT_H=1.6`。已验证：335 白字行，正常行距 0 出界，行距 2x 最坏情况 0 出界；每格 ≤2 行 0 超标。
- **排列 / 行序**：row0 = 用途（上行 3.1cm）、row1 = 编号（下行 0.9cm）。**分界（水平分隔）线必须由顶向下累计**画在 `y = py+ph-Σrow_h[0..r-1]`，即 row0/row1 真正边界，**绝不可**把分界线画在 py+row_h[0]（那会停在顶部 0.9cm 处，与文字错位）。
- **多行字数分配（关键新规则，`balance_lines` + `cell_lines`，用户第 6 次确认）**：**每格（用途 row0 / 编号 row1 各自）最多 2 行**——用户明言「文字行同线路行的段落行数太多，1-2 行就可以」。用户手改 docx 的示例已示范为 1 行，脚本须同步统一。
  - `balance_lines(text)`：n≤2 → 1 行；n≥3 → **2 行**（前半/后半尽量平衡，首行最多多 1 字）：3→2+1、4→2+2、5→3+2、6→3+3、7→4+3、8→4+4、9→5+4。
  - `cell_lines(txt)`：先按显式 `\n` 分段。**若已有 ≥2 段（作者显式断行）→ 保留每段 1 行、不再拆、整格封顶 2 行（取前 2 段）**；若只有 1 段，过长先过 `balance_lines`（封顶 2 行），非 CJK 原样 1 行。
  - 关键修正：旧版 `balance_lines` 会把单 CJK 段拆成 3–4 行，令长名爆到 3 行；且会把显式断行再拆，变成 3 行。新版封顶 2 行后全部格 ≤2。
- **RCD 列自动合并（默认自动）**：docx 的 RCD 宽列（c0）在 row0 与 row1 若都写了「漏电断路掣\nR C D」（非合并单元格），**默认脚本自动侦测含「漏电/RCD/RCCB」最左栏并合并**为**一格跨整块板高**的高 cell，文字取 row0 一次、按 `\n` 分两段**水平居中 + 垂直置中**，**分隔线不穿过该列**（只从 RCD 右界画到板右）。不用再手打 flag；要关掉用 `--no-merge-rcd`。
  - **RCD 段换行规则（`rcd_lines` 专属，≠ `balance_lines`）**：CJK 段「漏电断路掣」若 `len×TEXT_H ≤ 列宽−2·LEFT_MARGIN`（**一行摆得落**）就**整段一行**；摆不落才拆成**三行**（按余数 1/2/3 分配字数）。

### 11.2 生成后必跑验证（锁死「字必须在框内」+ 每格 ≤2 行）

用 `verify_label_cad.py`（座标解析，不靠肉眼），不要只靠 preview：

```bash
PY="<CAD_PYTHON>"

# 重新生成（glob 解析路径，避开字符坑；RCD 栏自动合并，不用 flag）
$PY scripts/regen_label_cad.py --glob "<DESKTOP_DIR>/*电箱*/*项目*.docx"

# 验证：每格 ≤2 行 + DXF/SVG 全部文字入框（含行距 2x 最坏情况）
$PY scripts/verify_label_cad.py --glob "<DESKTOP_DIR>/*电箱*/*项目*.docx"
#   预期输出：✅ 全部格 ≤2 行 / ✅ 全部入框（含最坏情况）/ ✅ 全部通过
#
# 自动化更彻底的「无头自检」（ezdxf→PNG 自渲染 + 7 项几何自检，不用等用户截图）：
$PY scripts/render_check_label_cad.py --glob "<DESKTOP_DIR>/*电箱*/*项目*.dxf" --png <TEMP_DIR>/check.png
#   自检项目：[2]入框(含2x行距) / [4]水平居中 / [5]字同字重叠 / [6]板同板/标柱相撞 / [7]RCD行数
#   全部 ✅ → 「自检全部通过（不用等用户截图）」
#
# DXF → DWG 交接地（生成 .scr，可 --console 无头转档出 .dwg）
$PY scripts/dx2dwg_scr.py --glob "<DESKTOP_DIR>/*电箱*/*项目*.dxf"
#
# regression gate（pytest，自制 fixture 不靠用户档）
$PY -m pytest tests/ -q
```

若报出界 → 第一时间检查 DXF 文字定位是不是误用了「整栏一个 MTEXT + 空白行 + 板中心」旧招（见 §9 第 16 条）。若报「不居中 / 重叠 / 相撞」→ 查 `read_tables` 栏宽解析或 `layout` 排版 gap 参数（见 §9 第 18、19 条）。

---

## 12. 关键参数速查

- 标签牌：黑底 `000000` / 白字 / 白边框 0.5pt / 字型 SimHei
- 行高：上行用途 1758 twips（3.1cm）/ 下行编号 510 twips（0.9cm）
- 栏宽：MCB 1P=1021 twips（1.8cm）；RCD=1021×P（2P=3.6cm、4P=7.2cm）；馈线 3P=5.4cm
- 纸张：回路 >22 用 A3 横向；边距 1.27cm
- 大标签：80×50mm=4536×2835 twips；编号+「配电箱」48pt 粗体；总开关 64pt
- CAD 图框：默认宽 42cm（A3），高自动；标注单位 cm
- 执行环境：`<CAD_PYTHON>`（ezdxf + python-docx + PyMuPDF）

---

## 13. 成熟度守则（agent 行为）

### 13.1 由「病徵」重诊断，不要靠估（座标解析 > 肉眼/截图）
- 用户反馈「字出界」，**不要第一时间当是断词/排版问题**。先问清楚现象（「是字飞出框？还是位置错？」）。
- 系统不支持直接看图时，**靠 parse DXF/SVG 座标诊断**：逐一比对 MTEXT insert 座标 vs 板框 LWPOLYLINE、计算行中心有没有超出 `[y0,y1]`。本任务就是这么发现「整栏一个 MTEXT + 板中心」在 AutoCAD 实际渲染会顶出框。
- 一旦锁定根因，做**可量化**的修复（逐 row MTEXT + 垂直置中 row band），再用 `verify_label_cad.py` / `render_check_label_cad.py` 座标自检锁死。

### 13.2 确认即沉淀（不要等到最后才写）
- 用户确认「全完正确」当下，就应该顺手写进对应文档（README / SKILL），而不要等到会话末尾才一次补齐。确认 = 当下可沉淀。
- 每次改了脚本/参数/采坑，**同一回合内**就更新对应文档与记忆；断层会令下次又重蹈覆辙。

### 13.3 自动化优先（不用等用户出手）
- 凡是「要人手贴图确认」的步骤，问自己：可不可以变成脚本自检？`render_check_label_cad.py`（ezdxf→PNG 自渲染 + 几何自检）就是为了让 agent 不用等用户截图就知有没有出框/不居中/重叠/相撞。
- 生成器改动后，**先跑 regression gate**（`python -m pytest tests/ -q`）再交付；不要只是手动 preview。

### 13.4 路径/字型等「看不到的坑」要主动锁死
- 中文/罕字路径坑 → 一律 `--glob` 在 Python 内部解析。
- DXF style `font` 只收档名 → 用 `os.path.basename()`。
- verify 板侦测要剔走 sheet 外框 → 否则 in-bounds 假 pass。
- 这些都是「肉眼查不到、只在 AutoCAD 实际开档才爆」的坑，写入 §9 对应条目，下次直接避开。
