---
name: chartqa
display-name: ChartQA 评测复现
description: 复现 ChartQA test（2500 题图表问答）分数所需的完整条件：VLMEvalKit 数据 URL+MD5、带"单个词或短语"指令的 prompt、5% 相对容差的 relaxed accuracy 判分（含 % 与 $ 的处理细节）、锚点分说明与踩坑。当用户要跑/复现 ChartQA、给模型测 ChartQA、或问"ChartQA 分数和官方对不上"时使用。
---

# ChartQA 评测复现

2500 道图表问答（约一半是"数值"题、一半是人类编写的推理题）。单指标：**relaxed accuracy**。

**没有官方 Qwen 专用脚本**；且 **Qwen3-VL 官方成绩表里没有 ChartQA**（官方模型卡未列）→ **本 bench 没有官方锚点可对**，只能自洽比较或跟 VLMEvalKit 榜单比。

## 0. 锚点分

| 来源 | ChartQA |
|---|---|
| Qwen3-VL 官方 model card | **未列（无官方值）** |
| 参考（第三方/榜单，仅供量级参考） | 强模型约 80-90 |

→ 结论：**这个 bench 只能看"我们自己前后是否自洽"，不能声称复现官方**。

## 1. 数据

- URL：`https://opencompass.openxlab.space/utils/VLMEval/ChartQA_TEST.tsv`
- MD5：`c902e0aa9be5582a7aad6dcf52734b42`
- 关键列：`index, question, image, answer, type`
- `answer` 可能是列表（一题多标准答案）。

## 2. Prompt（`ImageVQADataset.build_prompt`）

```
[image]
<line['question'] 原样>
Answer the question using a single word or phrase.
```
- **无 system prompt**；图片在前。
- 那行指令**必须逐字加上**（官方在 VQA 基类里统一追加）。

## 3. 采样参数

| | Instruct | Thinking |
|---|---|---|
| max_new_tokens | 32768 | 40960 |
| temperature | 0.7 | 1.0 |
| top_p / top_k | 0.8 / 20 | 0.95 / 20 |
| presence_penalty | 1.5 | 0.0 |

## 4. 判分（relaxed accuracy，纯规则，无 judge）

对每个标准答案：
1. **数值路径**：`_to_float(text)` —— 若以 `%` 结尾则去掉 `%` 再 `/100`；能转成 float 才走这条。
2. 双方都能转数值 → `abs(pred - gt) / abs(gt) <= 0.05`（**5% 相对容差**）。
3. 否则 → `pred.lower() == gt.lower()`（**只 strip，不做标点/冠词归一化**）。
4. 该行分数 = 所有标准答案里的 `max`（任一答对即可）。
5. `Overall = mean(行分数) * 100`。

⚠️ **`$` 不做特殊处理**（只有 `%` 被特判）。自己实现时"顺手把 `$` 也剥掉"会导致和官方不一致。

## 5. 汇报口径

`Overall` + 按 `type`（numeric / human）分列。**必须标注"本 bench 无官方锚点"**。

## 6. 坑

1. prompt 少了 `Answer the question using a single word or phrase.` 那一行。
2. 判分用严格相等（不做 5% 容差）→ 明显偏低。
3. 剥了 `$` / 做了 VQA 式标点归一化 → 与官方不同。
4. `%` 与小数（`70%` vs `0.7`）没做双向归一化。
5. 多答案题没取 `max`。
6. 没区分 numeric/human 两类，看不出差在哪。

## 7. checklist

- [ ] 数据 MD5 校验
- [ ] 抽查 prompt：末尾有那行指令、图片在前、无 system prompt
- [ ] 判分是 5% 相对容差 + lower 相等兜底，`$` 未剥离
- [ ] 分 numeric/human 报数
- [ ] 结果标注"无官方锚点，仅内部自洽可比"
