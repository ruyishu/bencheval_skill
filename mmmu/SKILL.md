---
name: mmmu
display-name: MMMU val 评测复现
description: 复现 MMMU val（900 题学科选择题）评测分数所需的完整条件：Qwen 官方专用脚本与 VLMEvalKit 两条口径、数据 URL+MD5、图片按 <image N> 交错插入的 prompt、开放题被预处理成二选一的特殊判分、Instruct/Thinking 两套参数、官方锚点分与踩坑。当用户要跑/复现 MMMU、给模型测 MMMU、或问"MMMU 分数和官方对不上"时使用。
---

# MMMU val 评测复现

900 道大学学科多模态题（选择题为主，**含图像作答/开放题**）。单指标 `acc`。

**M-MMU 有 Qwen 官方专用脚本**（和 MathVision 同一套结构），比第三方框架权威：
`https://github.com/QwenLM/Qwen3-VL/tree/main/evaluation/mmmu`

## 0. 锚点分（Qwen3-VL 官方 model card）

| 变体 | MMMU_val |
|---|---|
| Qwen3-VL-4B-Thinking | **70.8** |
| Qwen3-VL-4B-Instruct | 67.4 |
| Qwen3-VL-8B-Thinking | 74.1 |

## 1. 数据

- `https://opencompass.openxlab.space/utils/VLMEval/MMMU_DEV_VAL.tsv`
- MD5：`585e8ad75e73f75dcad265dfd0417d64`
- 列：`index, question, image, A, B, C, D, answer, split, ...`；`answer` 是字母，开放题的 `A` 列为 NaN。

## 2. Prompt（官方脚本 `run_mmmu.py`）

- **图片分辨率**：`MIN_PIXELS = 1280*28*28`、`MAX_PIXELS = 5120*28*28`（**和 MathVision 不同**，MathVision 是 768）。
- 基础部分同 VLMEvalKit 的 MCQ 模板（`Question: ...\nOptions:\nA. xxx\nB. xxx\n...\nPlease select the correct answer from the options above. \n`）。
- **关键差异**：`split_MMMU()` 会把图片**按 `<image N>` 占位符交错插进文本**（1-based）。题干里没有 `<image ` 时保持"图片全在前"。多图题不按这个插，图文顺序就变了。

## 3. 采样参数

| | Instruct | Thinking |
|---|---|---|
| max_new_tokens | 32768 | 40960 |
| temperature | 0.7 | **1.0** |
| top_p / top_k | 0.8 / 20 | 0.95 / 20 |
| presence_penalty | 1.5 | 0.0 |

vLLM：`max_model_len=128000`、`limit_mm_per_prompt={"image":10}`、`gpu_mem_util=0.9`。
Thinking 输出按 `</think>` 切尾段。

## 4. 判分（两阶段，官方 `eval_utils.py`）

1. **规则 prefetch**：`can_infer_option` / `can_infer_text` 抽选项字母；命中即采用。
2. **judge 兜底**：默认 `gpt-3.5-turbo-0125`（官方 `eval_instruct.sh`），也可 `gpt-4o-2024-05-13`；DashScope 或 OpenAI 兼容端点，重试温度 `i*0.5`。
3. **MMMU 专属预处理 `MMMU_preproc`**（`multiple_choice.py:62`）：**开放题（`A` 列为 NaN）被就地改写成二选一** —— `A = answer`、`B = 'Other Answers'`；**ground truth 不是 A–Z 字母的一律映射成 `'A'`**（等于白送对）。不做这个预处理，开放题会全判错。
4. `Overall = mean(hit)`，官方输出 `overall_accuracy` + `accuracy_by_split`。

## 5. 汇报口径

`Overall acc` + 按 `split`（子学科）分列 + 规则命中率 / judge 失败率。官方脚本还会输出逐题 CSV（`extracted_answer / extraction_method / hit`）。

## 6. 坑

1. **开放题没做 MMMU_preproc** → 开放题全错，总分被压低好几个点。
2. **`<image N>` 没交错插入** → 多图题图文错位。
3. 图片分辨率用了默认/别人的值（MMMU 要 1280，MathVision 是 768，别互相抄）。
4. 答案列里的非字母答案被映射成 `'A'`（这是官方行为，别"修"）。
5. 纯规则判分偏低，必须配 judge。
6. Thinking 没切 `</think>`。

## 7. checklist

- [ ] 数据 MD5 校验
- [ ] 抽查 prompt：选项格式、`<image N>` 交错、图片分辨率 1280
- [ ] 开放题走了 MMMU_preproc（抽查一条 A 列为 NaN 的题）
- [ ] 采样参数按变体取
- [ ] 与 70.8（4B-Thinking）对表并报差值
