---
name: hallusionbench
display-name: HallusionBench 评测复现
description: 复现 HallusionBench（幻觉评测）分数所需的完整条件：VLMEvalKit 数据 URL+MD5、极简 prompt（无 Yes/No 指令）、Yes/No 抽取 + gpt-4o-mini 兜底、aAcc/fAcc/qAcc 三指标取平均的官方口径、官方锚点分与踩坑。当用户要跑/复现 HallusionBench、给模型测幻觉、或问"HallusionBench 分数和官方对不上"时使用。
---

# HallusionBench 评测复现

图片幻觉（视觉依赖 vs 语言先验）评测，约 1129 题，Yes/No + 少数开放题。

**这个 bench 的分数不是普通准确率**：官方报的是 **`Avg = (aAcc + fAcc + qAcc) / 3`**，其中后两项是"图组/题组全对才算对"的严格指标。

## 0. 锚点分（Qwen3-VL 官方 model card）

| 变体 | HallusionBench |
|---|---|
| Qwen3-VL-4B-Thinking | **64.1** |
| Qwen3-VL-4B-Instruct | 57.6 |
| Qwen3-VL-8B-Thinking | 65.4 |

## 1. 数据

- URL：`https://opencompass.openxlab.space/utils/VLMEval/HallusionBench.tsv`
- MD5：`0c23ac0dc9ef46832d7a24504f2a0c7c`
- 关键列：`index, question, image, answer, category, l2-category`
- ⚠️ `answer` 列已经是 **`'Yes'` / `'No'`** 字符串（不需要自己做 Y/N→Yes/No 映射）。`index` 是**下划线分隔**的（`..._<set_id>_<figure_id>_<question_id>`），后三项被用于分组。

## 2. Prompt（`ImageBaseDataset.build_prompt`，HallusionBench 未覆写）

```
[image]
<line['question'] 原样>
```
- **无 system prompt**。
- ⚠️ **HallusionBench 不追加 `Please answer yes or no.`**（那句只对 AMBER 数据集加）。自己加了 → 和官方不同套。

## 3. 采样参数

| | Instruct | Thinking |
|---|---|---|
| max_new_tokens | 32768 | 40960 |
| temperature | 0.7 | 1.0 |
| top_p / top_k | 0.8 / 20 | 0.95 / 20 |
| presence_penalty | 1.5 | 0.0 |

## 4. 判分

**第一步 · 抽取 `YOrN_Extraction`**（纯规则，`utils/yorn.py:255`）：
- 预测 → 小写 + 去标点 → 按词判断：出现 `yes` 且没有 `no` → `'Yes'`；反之 `'No'`；否则 `'Unknown'`。

**第二步 · 兜底 judge（仅对 `'Unknown'` 的行）**：
- judge 默认 **`gpt-4o-mini`**，prompt 要求输出 `Yes` / `No` / `Unknown` 三选一，重试 5 次、温度 `0.5*i`，仍失败就给 `'Unknown'`。

**第三步 · 计分（三指标）**：
```
score = (answer == extracted)                     # 严格相等（'Yes'/'No'）
aAcc = mean(score) * 100                          # 每题平均
fAcc = 按 f"{l2-category}_{set_id}_{figure_id}" 分组，组内 np.all → 组准确率 * 100
qAcc = 按 f"..._{question_id}"                   分组，组内 np.all → 组准确率 * 100
Avg  = (aAcc + fAcc + qAcc) / 3                   # ← 官方/榜单报的就是这个
```
- `set_id/figure_id/question_id` 由 `index.split('_')[3/4/5]` **按位置解析**——index 格式变了会静默错分组。

## 5. 汇报口径

**必须同时给 `aAcc / fAcc / qAcc / Avg`**，并注明 `Avg` 才是可比官方 64.1 的数。只报普通准确率（≈aAcc）会显著高于官方值。

## 6. 坑

1. **只报 aAcc** 当总分 → 虚高，和 64.1 对不上（差 5-10 个点很常见）。
2. 自己加了 Yes/No 指令 → prompt 不同套。
3. `index` 分组解析错 → fAcc/qAcc 全错但 aAcc 正常，很难察觉。
4. 抽取规则写成"找最后一个 yes/no" → 和官方"词表成员判断"不同。
5. judge 失败率高的行被当答错 → 压低 aAcc。
6. `Y`/`N` 手写映射（官方这版**没有**映射，answer 已是 Yes/No）。

## 7. checklist

- [ ] 数据 MD5 校验
- [ ] 抽查 prompt：题干原样、**没有** Yes/No 指令
- [ ] 抽取规则按"词表成员"实现
- [ ] aAcc/fAcc/qAcc 三个都算，并核对 `index` 分组的三个位置
- [ ] 与 64.1（4B-Thinking）对表并报差值（报 Avg）
