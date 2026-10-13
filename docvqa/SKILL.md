---
name: docvqa
display-name: DocVQA 评测复现
description: 复现 DocVQA 分数所需的完整条件：VLMEvalKit 数据 URL+MD5、带"单个词或短语"指令的 prompt、ANLS（编辑距离阈值 0.5）判分细节、VAL 与 TEST 两个 split 的口径差异、官方锚点分与踩坑。当用户要跑/复现 DocVQA、给模型测 DocVQA、或问"DocVQA 分数和官方对不上"时使用。
---

# DocVQA 评测复现

文档图像问答（5000 题量级）。单指标：**ANLS**（Average Normalized Levenshtein Similarity，阈值 0.5）。

**没有官方 Qwen 专用脚本** → 口径 = VLMEvalKit `ImageVQADataset` 的 `anls` 分支。

## 0. 锚点分（Qwen3-VL 官方 model card）

| 变体 | DocVQA |
|---|---|
| Qwen3-VL-4B-Thinking | **94.2** |
| Qwen3-VL-4B-Instruct | 95.3 |
| Qwen3-VL-8B-Thinking | 95.3 |

⚠️ **split 陷阱**：官方模型卡写的是 **DocVQA_TEST**，而 VLMEvalKit 默认跑 **DocVQA_VAL**。两者题集不同、分数不可直接比。要对标 94.2 就要用 TEST（但 TEST 的公开标注情况要自己确认）。

## 1. 数据

| split | URL | MD5 |
|---|---|---|
| DocVQA_VAL | `https://opencompass.openxlab.space/utils/VLMEval/DocVQA_VAL.tsv` | `d5ee77e1926ff10690d469c56b73eabf` |
| DocVQA_TEST | `.../VLMEval/DocVQA_TEST.tsv` | 按需自行校验 |

关键列：`index, question, image, answer`（`answer` 常为多答案列表）。

## 2. Prompt（`ImageVQADataset.build_prompt`）

```
[image]
<line['question'] 原样>
Answer the question using a single word or phrase.
```
- **无 system prompt**；图片在前；那行指令逐字加上。

## 3. 采样参数

| | Instruct | Thinking |
|---|---|---|
| max_new_tokens | 32768 | 40960 |
| temperature | 0.7 | 1.0 |
| top_p / top_k | 0.8 / 20 | 0.95 / 20 |
| presence_penalty | 1.5 | 0.0 |

## 4. 判分（ANLS，纯规则，无 judge）

对每个标准答案：
```
gt  = ' '.join(groundtruth.strip().lower().split())   # 折叠空白 + 小写
det = ' '.join(prediction.strip().lower().split())
dist = levenshtein(gt, det)
length = max(len(groundtruth.upper()), len(prediction.upper()))   # 注意用的是原始串长度
sim = 0.0 if length == 0 else 1 - dist/length
行分数 = 0.0 if sim < 0.5 else sim        # 只在最小距离（最相似）的那个 GT 上算
Overall = mean(行分数) * 100
```
⚠️ 三个容易写错的点：
1. **`dist` 用折叠空白+小写的串，`length` 却用原始大写串的长度**（不对称，照抄）。
2. 取 **min 距离**（最相似的那个 GT），不是 max。
3. 低于 0.5 直接归零；**0.5 以上保留连续值**（所以 Overall 是连续 ANLS 的均值，不是"对了多少题"）。

## 5. 汇报口径

`Overall ANLS` + 用的是哪个 split + 每题 ANLS 的分布（0.5 阈值附近堆积会影响分数）。

## 6. 坑

1. **VAL / TEST 混用** → 和官方 94.2 不可比。
2. ANLS 用对称归一化（`max(len(gt), len(pred))` 对两侧都折叠）→ 与官方有偏差。
3. 用 max 而不是 min 取多答案。
4. 把 ANLS 当二值（过 0.5 记 1）→ 分数虚高。
5. prompt 少了那行指令。

## 7. checklist

- [ ] 明确 split 并写进结果（对标官方要用 TEST）
- [ ] 数据 MD5 校验
- [ ] 抽查 prompt：末尾有指令、图片在前
- [ ] ANLS 实现与上面 4 条逐条对齐
- [ ] 与 94.2（4B-Thinking, TEST）对表并报差值 + 注明 split
