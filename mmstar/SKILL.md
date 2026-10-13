---
name: mmstar
display-name: MMStar 评测复现
description: 复现 MMStar（1500 题多模态选择题）评测分数所需的完整条件：VLMEvalKit 官方数据 URL+MD5、prompt 逐字模板（Question/Options/Please select 三行式）、纯规则判分 + gpt-4o-mini 兜底抽取、官方锚点分与踩坑。当用户要跑/复现 MMStar、给模型测 MMStar、或问"MMStar 分数和官方对不上"时使用。
---

# MMStar 评测复现

1500 道多模态选择题（感知 + 推理，无泄漏设计），单指标 `acc`。

**口径**：VLMEvalKit `ImageMCQDataset` 通用路径（**MMStar 没有独立类**）。无官方 Qwen 专用脚本。

## 0. 锚点分（Qwen3-VL 官方 model card）

| 变体 | MMStar |
|---|---|
| Qwen3-VL-4B-Thinking | **73.2** |
| Qwen3-VL-4B-Instruct | 69.8 |
| Qwen3-VL-8B-Thinking | 75.3 |

## 1. 数据

- URL：`https://opencompass.openxlab.space/utils/VLMEval/MMStar.tsv`
- MD5：`e1ecd2140806c1b1bbf54b43372efb9e`
- 列：`index, question, image, A, B, C, D, answer, category, l2_category, ...`
- ⚠️ **选项在独立的 `A/B/C/D` 列里**，不在 `question` 字段里（与 MathVision/MathVista 相反）——所以要自己渲染选项。

## 2. Prompt（逐字，来自 `image_mcq.py:211-246`）

```
[image]
Question: <line['question'] 原样>
Options:
A. <line['A']>
B. <line['B']>
C. <line['C']>
D. <line['D']>
Please select the correct answer from the options above. 
```
- **无 system prompt**；图片在前，文本一条。
- 选项格式是 **`A. 文本`（点号）**，不是 `(A) 文本`。
- 若 TSV 有非空 `hint` 列，则在最前面加 `Hint: {hint}\n`（MMStar 一般没有）。
- 注意最后一行末尾**有一个空格**再换行。

## 3. 采样参数

| | Instruct | Thinking |
|---|---|---|
| max_new_tokens | 32768 | 40960 |
| temperature | 0.7 | 1.0 |
| top_p / top_k | 0.8 / 20 | 0.95 / 20 |
| presence_penalty | 1.5 | 0.0 |

（Qwen 全局推荐；bench 无专用覆盖。Thinking 输出按 `</think>` 切尾段再判分。）

## 4. 判分

1. **规则抽取** `can_infer(pred, choices)`：先 `can_infer_option`（把 `.()[],:;!*#{}` 换空格后切词找唯一选项字母；另有一条正则 `(?i)(?:correct\s+)?answer\s+is\s+\**([ABCD])\**`），否则 `can_infer_text`（选项文本子串唯一命中）。
2. **兜底**：规则抽不出来时调 **`gpt-4o-mini`** 让 judge 给单个字母（A–D 或 Z）；**judge 重试全失败时会随机选一个**（会引入噪声分）。
3. 命中：`letter == line['answer']`；`Overall = mean(hit)`。

## 5. 汇报口径

给 `Overall acc`（+ 可给 `category` / `l2_category` 分组）。要额外报**规则命中率**——否则看不出 judge 兜底是否大量介入。

## 6. 坑

1. **选项没渲染**：直接用 `question` 就错了（MMStar 的选项在 A–D 列）。
2. 选项格式写成 `(A)` 而不是 `A.`。
3. **纯规则判分会偏低**（自由作答抽不出字母），必须配 judge 兜底。
4. judge 失败随机给字母 → 少量样本被随机化，注意 judge 失败率。
5. Thinking 输出没切 `</think>`。

## 7. checklist

- [ ] 数据 MD5 校验
- [ ] 抽查 prompt：4 个选项都在、格式 `A. `、图片在前、无 system prompt
- [ ] 采样参数按变体取
- [ ] 判分两阶段齐全，报规则命中率 + judge 失败率
- [ ] 与 73.2（4B-Thinking）对表并报差值
