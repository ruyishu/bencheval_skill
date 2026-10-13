---
name: ocrbench
display-name: OCRBench 评测复现
description: 复现 OCRBench（1000 题 OCR 能力评测）分数所需的完整条件：VLMEvalKit 数据 URL+MD5、极简 prompt（图片+题干原样，无指令）、10 个类别按子串包含判分、五组加总与归一化口径、官方锚点分与踩坑。当用户要跑/复现 OCRBench、给模型测 OCRBench、或问"OCRBench 分数和官方对不上"时使用。
---

# OCRBench 评测复现

1000 道 OCR 题，分 10 个类别（文字识别 6 类 + 场景 VQA + 文档 VQA + 关键信息抽取 + 手写公式），最终报 **五组加总的分**。

**没有官方 Qwen 专用脚本** → 口径 = VLMEvalKit `class OCRBench`。**全程不用 judge**（纯规则）。

## 0. 锚点分（Qwen3-VL 官方 model card）

| 变体 | OCRBench |
|---|---|
| Qwen3-VL-4B-Thinking | **808.0** |
| Qwen3-VL-4B-Instruct | 881 |
| Qwen3-VL-8B-Thinking | 819.0 |

⚠️ 注意：**Instruct 比 Thinking 高**（881 vs 808）——这个 bench 上"多想"反而掉分（思考段干扰抽取）。别以为搞反了。

## 1. 数据

- URL：`https://opencompass.openxlab.space/utils/VLMEval/OCRBench.tsv`
- MD5：`e953d98a987cc6e26ef717b61260b778`
- 关键列：`index, question, image, answer, category`
- ⚠️ `answer` 列是**字符串化的 Python 列表**（`"['abc']"`），要 `eval()` 成列表逐个比。

## 2. Prompt（`ImageBaseDataset.build_prompt`，OCRBench 未覆写）

```
[image]
<line['question'] 原样>
```
- **无 system prompt、无任何附加指令**（不追加"Answer in a word"之类）。加任何东西都和官方不是同一套。

## 3. 采样参数

| | Instruct | Thinking |
|---|---|---|
| max_new_tokens | 32768 | 40960 |
| temperature | 0.7 | 1.0 |
| top_p / top_k | 0.8 / 20 | 0.95 / 20 |
| presence_penalty | 1.5 | 0.0 |

## 4. 判分（纯规则子串包含）

对每个候选答案 j（`answer` 是列表）：
- **手写公式类**（`Handwritten Mathematical Expression Recognition`）：答案与预测都先去掉空白/换行，再 `answer in predict`。
- **其它类**：都 `.lower().strip()`，再 `answer in predict`（**子串包含**，不是相等）。
- 任一候选命中即该题得 1 分。

**汇总（官方口径）**：
```
Text Recognition = 6 个识别类之和
Final Score = Text Recognition + Scene Text-centric VQA + Doc-oriented VQA + Key Information Extraction + Handwritten Math
Final Score Norm = Final Score / 10
```
→ 报的 808.0 就是 **Final Score**（原始加总），不是 percentage。

## 5. 汇报口径

**必须报 10 个类别的分项**，不能只报总分——不同能力（手写/表格/公式）差异很大。另报每题命中数。

## 6. 坑

1. **原始 OCRBench 的"答案不超过 20 字符"规则在 VLMEvalKit 里没有实现** —— 如果你自己实现时加了长度上限，分数会**偏低**（啰嗦但含正确答案的输出会被判错）。
2. 用**相等**而不是**子串包含**判分 → 系统性偏低。
3. 手写公式类没去空白 → 偏低。
4. 类别名写错（10 个字符串必须完全一致），会被静默丢弃、不参与加总。
5. 给 prompt 加了额外指令（如"只回答单词"）→ 与官方不同套。
6. 把 `answer` 当字符串比而不是 `eval()` 成列表。

## 7. checklist

- [ ] 数据 MD5 校验
- [ ] 抽查 prompt：题干原样、无附加指令、无 system prompt
- [ ] 判分是"子串包含 + lower/strip"（未加 20 字符上限）
- [ ] 10 个类别全部命中且分项报出
- [ ] 与 808.0（4B-Thinking）/ 881（4B-Instruct）对表并报差值
