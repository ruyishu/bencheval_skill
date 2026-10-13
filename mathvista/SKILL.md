---
name: mathvista
display-name: MathVista 评测复现
description: 复现 Qwen 官方 MathVista 多模态数学推理分数所需的完整评测条件（两套官方口径 VLMEvalKit / lmms-eval 的数据源与 MD5、prompt 与 hint 模板、采样参数冲突、判分两阶段与 judge 差异、官方锚点分与踩坑）。当用户要跑/复现 MathVista、给模型测 MathVista、或问"MathVista 分数和官方对不上"时使用。
---

# MathVista 评测复现

MathVista：1000 题（testmini）视觉数学推理，含选择题与自由作答，按 `task` 与 `skills` 两个维度出分。

**与 MathVision 的关键差别**：Qwen 仓库里**没有 MathVista 的专用脚本**（只有 MathVision/MMMU/RealWorldQA/VideoMME/ODinW-13）。官方 MathVista 分数来自**第三方框架**，因此本 skill 覆盖**两套官方口径**，必须先选定一套再跑。

| 口径 | 框架 | 证据强度 |
|---|---|---|
| **A. lmms-eval** | `mathvista_testmini_cot` | 该 task 带 tag **`public_eval_qwen3_5_family`** → 很可能是 Qwen 家族公开评测实际在用的 |
| B. VLMEvalKit | `MathVista_MINI` + judge `gpt-4o-mini` | Qwen README 自述评测框架包含 VLMEvalKit |

**默认动作**：先用 A 跑，再用 B 交叉验证；两者差值本身就是重要信息（说明分数对框架敏感）。

细节与逐字模板见 `references/official_config.md`。

## 0. 锚点分（官方 model card，API 口径）

| 变体 | MathVista(mini) |
|---|---|
| Qwen3-VL-4B-Thinking | **79.5** |
| Qwen3-VL-4B-Instruct | 73.7 |
| Qwen3-VL-8B-Thinking | 81.4 |
| Qwen3-VL-8B-Instruct | 77.2 |
| Qwen3-VL-2B-Thinking | 73.6 |
| Qwen3-VL-2B-Instruct | 61.3 |

## 1. 数据

| 口径 | 数据 | 校验 |
|---|---|---|
| A. lmms-eval | HF `AI4Math/MathVista`，`test_split: testmini` | dataset 仓库 revision |
| B. VLMEvalKit | `https://opencompass.openxlab.space/utils/VLMEval/MathVista_MINI.tsv` | MD5 `f199b98e178e5a2a20e7048f5dcb0464` |

VLMEvalKit 那份 TSV 的列（实测首行）：
`question | image | choices | answer | question_type | answer_type | index | category | context | skills | source | task | answer_option`

**重要**：`question` 字段**已经烧好官方 query**（含 `Hint: ...` 行），实测首行是
`"Hint: Please answer the question requiring a floating-point number with one decimal place and provide the final value, e.g., 1.2, 1.3, 1.4, at the end.\nQuestion: ..."`。
所以 VLMEvalKit 路径**直接用 `question`**；lmms-eval 路径用 `create_one_query()` 现场拼同等内容。

## 2. Prompt 构造

两套都是**单条 user 消息、无 system prompt、图片在前文本在后**：

```
messages = [{"role": "user", "content": [
    {"type": "image", "image": <path>},
    {"type": "text",  "text": <query>}
]}]
```

`<query>` 的三种来源：
- A（lmms-eval）：`create_one_query(problem, shot_type="step-by-step", shot_num=0, use_caption=False, use_ocr=False)`
- B（VLMEvalKit）：TSV 的 `question` 字段原样
- C（自己拼）：把官方 hint 逐字放最前，然后 `Question: ...`，选择题再补 `Choices:\n(A) ...\n(B) ...`

官方 hint 逐字（按 `question_type`/`answer_type` 选，**措辞会影响分数**）：

```
选择题 :
Hint: Please answer the question and provide the correct option letter, e.g., A, B, C, D, at the end.
整数   :
Hint: Please answer the question requiring an integer answer and provide the final value, e.g., 1, 2, 3, at the end.
浮点(1位):
Hint: Please answer the question requiring a floating-point number with one decimal place and provide the final value, e.g., 1.2, 1.3, 1.4, at the end.
浮点(2位):
Hint: Please answer the question requiring a floating-point number with two decimal places and provide the final value, e.g., 1.23, 1.34, 1.45, at the end.
列表   :
Hint: Please answer the question requiring a Python list as an answer and provide the final list, e.g., [1, 2, 3], [1.2, 1.3, 1.4], at the end.
```
选择题还要把选项渲染成 `Choices:\n(A) xxx\n(B) yyy`（`shot_type="solution"` 时 hint 用"provide the correct option letter"那版）。

## 3. 采样参数（有冲突，必须记录用了哪套）

| 来源 | max_new_tokens | temperature | top_p | top_k | presence |
|---|---|---|---|---|---|
| 权重自带 `generation_config.json`（实测 4B） | 模型默认 | Thinking **1.0** / Instruct 0.7 | 0.95 / 0.8 | 20 / 20 | — |
| 官方 bench 脚本（MathVision 那份，可作旁证） | Thinking 40960 / Instruct 32768 | Thinking **1.0** / Instruct 0.7 | 0.95 / 0.8 | 20 | 0.0 / 1.5 |
| lmms-eval yaml（`mathvista_testmini_cot`） | **16384** | — | — | — | — |
| Qwen 主 README「全局推荐」 | 32768 / 40960 | Thinking **0.6** / Instruct 0.7 | 0.95 / 0.8 | 20 | 0.0 / 1.5 |

⚠️ **Thinking 的 temperature 有两套说法**：主 README 写 0.6，而**权重 `generation_config.json` 与官方 bench 脚本都是 1.0**。实际跑 bench 建议用 **1.0**（并记录），用 0.6 时要预期几个点的漂移。

⚠️ lmms-eval 的 task 只钉了 `max_new_tokens: 16384`，采样细节由模型 wrapper 决定。**对齐 Qwen 官方数字 → 用权重/脚本那套；对齐 lmms-eval 榜单 → 用其框架默认**。两套跑出来的分数不可互相比较。

## 4. 判分

**两阶段**（A/B 同源，但细节不同）：

| 项 | A. lmms-eval | B. VLMEvalKit |
|---|---|---|
| 规则阶段 | 响应文本∈choices 直接返回；integer→`int()`；float→`float()` | 选择题→`answer_option` + `can_infer`；integer→`int()`；float→`float()`；否则字符串相等 |
| 快速抽取 | `quick_extract`（默认 **false**）：正则 `The answer is "(.*)"\.` | 无 |
| judge 兜底 | `create_test_prompt(DEMO_PROMPT, query, response)`，**temperature=0, max_tokens=256** | 5 个 ICE，重试 `temperature=i*0.5`（0/0.5/…/2.0），**max_tokens=128** |
| judge 默认模型 | `gpt-3.5-turbo`（task metadata） | **`gpt-4o-mini`**（`DEFAULT_JUDGE_MODEL`） |
| 答案归一化 | `normalize_extracted_answer`：`(A)`→A→映射回选项文本；否则取最相似选项 | `post_check` 直接比对 |
| 额外模式 | — | `use_verifier`：截断/重复判无效（更严） |

**VLMEvalKit 的 judge prompt 里有官方笔误，要逐字保留**：
`'Model respone: ' + prediction`（少一个 s，末尾带空格）、`'Extracted answer:'`（**无尾空格**）。
MathVision 那份是拼写正确 + 带尾空格的版本 —— 两个 bench 的 prompt 不一样，别互相抄。

## 5. 汇报口径

给：`Overall acc`、按 **`task`** 分列、按 **`skills`** 分列、`prefetch_rate`、judge 抽取失败率。
VLMEvalKit 输出 `Task&Skill | tot | prefetch | hit | prefetch_rate | acc`。

**只有规则判分没有 judge 时，必须显式标注**：MathVista 自由作答占比高（几何/代数计算），规则命中率低，纯规则分不可与官方比。

## 6. 坑

1. **judge 模型不同**：VLMEvalKit 默认 `gpt-4o-mini`、lmms-eval 默认 `gpt-3.5-turbo` —— 同一次推理换 judge 能差好几个点。必须记录 judge 名称与版本。
2. **judge max_tokens 差 2 倍**（128 vs 256）：长答案被截断会判错。
3. **hint 措辞/精度**：float 的"一位/两位小数"提示与数据里的 `precision` 绑定，写错会导致数值格式不匹配。
4. **`Model respone` 笔误**：改了就跟官方不是同一套 prompt。
5. **两套框架的分数不可互换比较**：先选一套，全程用同一套。
6. **`skills` 字段是字符串化的列表**（`"['...', '...']"`），统计前要 `eval()`。
7. **图像分辨率**：不像 MathVision 有官方钉死的 min/max_pixels，沿用模型默认即可，但要**在结果里写清**。

## 7. 复现 checklist

- [ ] 选定口径（A lmms-eval / B VLMEvalKit）并全程一致
- [ ] 数据 split = testmini（1000 题）且 MD5/revision 校验
- [ ] 抽查 3 条 prompt：含 `Hint:` 行、选择题含 `Choices:`、图片在前
- [ ] 采样参数与所选口径匹配并记录
- [ ] 判分两阶段齐全，judge 名称/版本/温度/max_tokens 记录在案
- [ ] 与 0. 节锚点对表，差值写进结论；差 ≥3pp 先排查第 6 节
