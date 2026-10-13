---
name: mathverse
display-name: MathVerse mini 评测复现
description: 复现 MathVerse mini（788 题多模态数学，含 6 个问题版本）评测分数所需的完整条件：VLMEvalKit 数据 URL+MD5、query 已内嵌选项的 prompt、两阶段 GPT 判分（抽取 + 一致性判定，均 max_tokens=128）、官方锚点分与踩坑。当用户要跑/复现 MathVerse、给模型测 MathVerse、或问"MathVerse 分数和官方对不上"时使用。
---

# MathVerse mini 评测复现

788 道带图数学题，按 `problem_version`（Text Dominant / Text Lite / Vision Intensive / Vision Only / …）分 6 个子集出分。

**没有官方 Qwen 专用脚本** → 口径 = VLMEvalKit `class MathVerse`。

## 0. 锚点分（Qwen3-VL 官方 model card）

| 变体 | MathVerse_mini |
|---|---|
| Qwen3-VL-4B-Thinking | **75.2** |
| Qwen3-VL-4B-Instruct | 46.8 |
| Qwen3-VL-8B-Thinking | 77.7 |

⚠️ 注意 Instruct(46.8) 和 Thinking(75.2) 差 **28 个点**——这个 bench 对"有没有推理"极度敏感，跑错变体标签会得出完全不同的结论。

## 1. 数据

- URL：`https://opencompass.openxlab.space/utils/benchmarks/MathVerse/MathVerse_MINIV.tsv`（**注意路径是 `utils/benchmarks/`，且文件名是 `MathVerse_MINIV.tsv`**，不是 `VLMEval/`）
- MD5：`5017caca32b7fa110c350a1bea861b65`
- 关键列：`index, question, image, answer, problem_version, metadata, query_cot`

## 2. Prompt（`image_vqa.py:767-786`）

```
[image(s)]
<line['question'] 原样>
```
- **无 system prompt**；图片在前。
- ⚠️ **选择题的选项已经写在 `question` 字段里了**，**不要再自己渲染选项**（渲染了会重复，和官方的不是同一套 prompt）。
- ⚠️ 只有 `*_cot` 变体才读 `query_cot` 列；`MathVerse_MINI` 读 `question`，用错列直接错。

## 3. 采样参数

| | Instruct | Thinking |
|---|---|---|
| max_new_tokens | 32768 | 40960 |
| temperature | 0.7 | 1.0 |
| top_p / top_k | 0.8 / 20 | 0.95 / 20 |
| presence_penalty | 1.5 | 0.0 |

## 4. 判分（**两阶段 GPT 判分，必须联网 judge**）

1. **预筛**：`post_check_score` = 预测与答案 strip 后严格相等 → 直接算对（短答案走这条）。
2. **抽取阶段**：judge 从模型输出里抽出答案（`build_mathverse_gpt4_extract_prompt`），`max_tokens=128`。
3. **判定阶段**：judge 比较"抽出的答案 vs 标准答案"是否一致，**输出必须是 `'0'` 或 `'1'`**，重试温度 `i*0.5`；`max_tokens=128`。
   - judge prompt 关键句（**含官方拼写错误 `Judement`，逐字保留**）：*"...If they are consistent, Judement is 1; if they are different, Judement is 0."*
4. 抽取器可能返回字面量 `null` —— 要当作抽取失败处理。
5. 汇总：按 `problem_version` 分组的准确率（`metadata` 列是 JSON，官方用 `.replace("'", '"')` 再解析）。

## 5. 汇报口径

`Overall` + **按 `problem_version` 分列**（这个 bench 必须分版本看）+ judge 失败率。只报 Overall 会掩盖版本间差异。

## 6. 坑

1. **自己渲染了选项** → 和官方 prompt 不同（选项已在 question 里）。
2. 用了 `query_cot` 列（只有 cot 变体该用）。
3. **判定阶段不是字符串相等**，是 LLM 判 0/1；用规则判分会系统性偏低。
4. judge 的 `max_tokens=128` 太小 → 长判定被截断。
5. 忘记分 `problem_version`。
6. 抽取返回 `null` 未处理。

## 7. checklist

- [ ] 数据 MD5 校验（注意 URL 是 `utils/benchmarks/...`）
- [ ] 抽查 prompt：选项没有重复渲染、图片在前
- [ ] 两阶段 judge 都接上，且判定输出限 `0/1`
- [ ] 按 `problem_version` 分列出分
- [ ] 与 75.2（4B-Thinking）对表并报差值
