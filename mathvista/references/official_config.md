# MathVista 官方配置 · 逐字细节与信源

本文件是 `mathvista/SKILL.md` 的深水区。

## 1. 信源清单（2026-10-13 实读）

| 内容 | 来源 |
|---|---|
| Qwen 全局采样参数（Instruct/Thinking） | `https://github.com/QwenLM/Qwen3-VL` README |
| Qwen 官方评测框架自述 | 同上（"Evaluation frameworks: VLMEvalKit, lmms-eval"，并注明"少数 bench 的 prompt 有改动"） |
| lmms-eval task 定义 | `https://github.com/EvolvingLMMs-Lab/lmms-eval/tree/main/lmms_eval/tasks/mathvista` |
| 其中 cot 档 | `mathvista_testmini_cot.yaml`（`shot_type: step-by-step`、`shot: 0`、`max_new_tokens: 16384`） |
| 其中 format 档 | `mathvista_testmini_format.yaml`（`shot_type: format-prompt`） |
| Prompt 构造 | `mathvista_evals.py::MathVistaEvaluator.create_one_query` |
| 抽取与判分 | `mathvista_evals.py::extract_answer / normalize_extracted_answer` |
| VLMEvalKit 数据集类 | `vlmeval/dataset/image_vqa.py::MathVista`（`DEFAULT_JUDGE_MODEL='gpt-4o-mini'`，`MathVista_MINI` MD5 `f199b98e178e5a2a20e7048f5dcb0464`） |
| VLMEvalKit 判分 | `vlmeval/dataset/utils/mathvista.py`（`post_check / MathVista_auxeval / MathVista_acc`） |
| 数据（VLMEvalKit） | `https://opencompass.openxlab.space/utils/VLMEval/MathVista_MINI.tsv` |
| 分数锚点 | Qwen3-VL model card 图 `qwen3vl_4b_8b_vl_{thinking,instruct}.jpg`、`qwen3vl_2b_32b_vl_*.jpg` |

**未决问题（重要）**：Qwen 没有 MathVista 专用脚本，**无法从公开材料 100% 判定 79.5 出自哪套框架**。
判断依据是 lmms-eval 的 task 带 tag `public_eval_qwen3_5_family`（"Qwen3.5 家族公开评测"），故优先怀疑 lmms-eval；
同时 Qwen README 又自述用了 VLMEvalKit。**结论：两套都要能跑，并报差值。**

## 2. lmms-eval 侧逐字细节

`mathvista_testmini_cot.yaml`（节选）：

```yaml
dataset_path: AI4Math/MathVista
task: "mathvista_testmini_cot"
test_split: testmini
tag:
  - visual_reasoning_collection
  - public_eval_qwen3_5_family
output_type: generate_until
doc_to_target: "answer"
generation_kwargs:
  max_new_tokens: 16384
metric_list:
  - metric: llm_as_judge_eval
lmms_eval_specific_kwargs:
  default:
    shot_type: "step-by-step"   # 可选 reason-first / solution / step-by-step
    shot: 0
    use_caption: False
    use_ocr: False
```

`mathvista_testmini.yaml` 的 metadata：

```yaml
metadata:
  gpt_eval_model_name: "gpt-3.5-turbo"
  quick_extract: false
```

`create_one_query(problem, shot_type, examples, shot_num=0, use_caption=False, use_ocr=False)` 行为：
- `shot_num == 0` → demo 段为空；**只用 [2] Test query 段**（即 hint + question + choices）。
- hint 选择逻辑（逐字）：
  - `multi_choice` → `Hint: Please answer the question and provide the correct option letter, e.g., A, B, C, D, at the end.`
  - `integer` → `Hint: Please answer the question requiring an integer answer and provide the final value, e.g., 1, 2, 3, at the end.`
  - `float` + `precision == 1` → `... requiring a floating-point number with one decimal place ... e.g., 1.2, 1.3, 1.4, ...`
  - `float` + `precision == 2` → `... with two decimal places ... e.g., 1.23, 1.34, 1.45, ...`
  - `list` → `... requiring a Python list as an answer and provide the final list, e.g., [1, 2, 3], [1.2, 1.3, 1.4], ...`
- 非 0-shot 时的 demo 形式：`Question: <q>` + `Choices:\n(A) ...` + 可选 `Image description:` / `Image detected text:`，再按 `shot_type` 追加 `Solution: ...` 或直接追加 solution。

`extract_answer(response, problem, quick_extract=False)` 顺序：
1. `question_type == "multi_choice"` 且 `response in choices` → 直接返回
2. `answer_type == "integer"` → `int(response)`
3. `answer_type == "float"` → `str(float(response))`
4. `quick_extract`（默认 False）：正则 `The answer is "(.*)"\.`
5. 兜底：LLM judge，`self.create_test_prompt(DEMO_PROMPT, query, response)`，**temperature=0, max_tokens=256, n=1**

`normalize_extracted_answer(extraction, choices, question_type, answer_type, precision)`：
- 选择题：strip → 正则 `\(([a-zA-Z])\)` 取首个字母并大写 → 若是合法选项字母则**映射回选项文本**，否则 `get_most_similar` 选最相似项
- 其他类型：按 `answer_type`/`precision` 归一

## 3. VLMEvalKit 侧逐字细节

数据集类（`ImageBaseDataset` 的默认 `build_prompt`，MathVista 未覆写）：

```python
question = line['question']           # 已含 hint + 选项
msgs = [dict(type='image', value=p) for p in tgt_path] + [dict(type='text', value=question)]
```

judge 构建：`build_judge(max_tokens=128, **judge_kwargs)`，`DEFAULT_JUDGE_MODEL = 'gpt-4o-mini'`，
`nproc` 默认 4，需要 `model.working()`（缺 API key 会直接断言失败）。

`post_check(line, prefetch=False)`：
```python
ans = line['answer']
response = line['prediction'] if prefetch else line['res']
if line['question_type'] == 'multi_choice':
    ans = line['answer_option']
    choices = list_to_dict(eval(line['choices']))
    res = can_infer(response, choices)
    if prefetch: return res
else:
    int / float / str 按 answer_type 转换
return (res == ans) if not prefetch else ...
```

`MathVista_auxeval(model, line)`：
```python
if post_check(line, prefetch=True):        # 规则命中
    return dict(log='Prefetch succeed', res=res)
for i in range(5):                          # judge 兜底
    res = model.generate(prompt, temperature=i * 0.5)
    if FAIL_MSG not in res: return dict(log='Succeed', res=res)
```

judge prompt（`build_mathvista_gpt4_prompt`）——**注意官方笔误**：

```
Please read the following example.
Then extract the answer from the model response and type it at the end of the prompt.

<ICE_1> ... <ICE_5>

<question>
Model respone: <prediction>Extracted answer:
```
（`Model respone` 少一个 s 且**有**尾空格；`Extracted answer:` **无**尾空格；prediction 与 "Extracted answer:" 之间**没有换行**）

5 个 ICE（`get_gpt4_ICE`，逐字，与 MathVision 版**不同**）：

```
Hint: Please answer the question requiring an integer answer and provide the final value,
e.g., 1, 2, 3, at the end.\n
Question: Which number is missing?\n
Model response: The number missing in the sequence is 14.\n
Extracted answer: 14
```
```
Hint: Please answer the question requiring a floating-point number with one decimal place and provide the final value,
e.g., 1.2, 1.3, 1.4, at the end.\n
Question: What is the fraction of females facing the camera?\n
Model response: The fraction of females facing the camera is 0.6,
which means that six out of ten females in the group are facing the camera.\n
Extracted answer: 0.6
```
```
Hint: Please answer the question requiring a floating-point number with two decimal places and provide the final value,
e.g., 1.23, 1.34, 1.45, at the end.\n
Question: How much money does Luca need to buy a sour apple candy and a butter-scotch candy? (Unit: $)\n
Model response: Luca needs $1.45 to buy a sour apple candy and a butterscotch candy.\n
Extracted answer: 1.45
```
```
Hint: Please answer the question requiring a Python list as an answer and provide the final list,
e.g., [1, 2, 3], [1.2, 1.3, 1.4], at the end.\n
Question: Between which two years does the line graph saw its maximum peak?\n
Model response: The line graph saw its maximum peak between 2007 and 2008.\n
Extracted answer: [2007, 2008]
```
```
Hint: Please answer the question and provide the correct option letter, e.g., A, B, C, D, at the end.\n
Question: What fraction of the shape is blue?\n
Choices: (A) 3/11 (B) 8/11 (C) 6/11 (D) 3/5\n
Model response: The correct answer is (B) 8/11.\n
Extracted answer: B
```

`MathVista_acc` 输出维度：`Task&Skill`（`task` 与 `skills` 各占若干行）+ `Overall`，列为 `tot/prefetch/hit/prefetch_rate/acc`。

## 4. 数据字段对照

VLMEvalKit `MathVista_MINI.tsv`：
`question, image, choices, answer, question_type, answer_type, index, category, context, skills, source, task, answer_option`

HF `AI4Math/MathVista`（lmms-eval 用）：含 `pid, question, image, decoded_image, choices, unit, precision, answer, question_type, answer_type, skills, source, query, caption, ocr, ...`
—— lmms-eval 的 `mathvista_doc_to_text` 正是读这些字段（`unit/caption/ocr/precision`）来拼 query。

## 5. 已知不确定项（别当成事实传播）

- **79.5 出自哪套框架未证实**（推论见 §1）。
- **Thinking 的 temperature 有两套**：Qwen 主 README 写 0.6，但**权重 `generation_config.json`（实测 4B-Thinking：T1.0 / top_p0.95 / top_k20 / rep1.0）与官方 bench 脚本都是 1.0**。
  Instruct 侧三方一致（T0.7 / top_p0.8 / top_k20）。
- lmms-eval task 只钉了 `max_new_tokens: 16384`，**与 README 的 32768/40960 不同**；两者如何组合官方未说明。
- model card 注明：默认评测走 **API 调用**（非本地权重）。
- VLMEvalKit 的 `MathVista.verifier` 模式（截断/重复判无效）会让分数更低；官方未说明是否启用。

## 6. 与我们既有实现的差异（迁移提醒）

- 我们既有脚本用的是官方 `query` 字段（自带 hint+选项）；VLMEvalKit 用 `question`（实测同样已烧入 hint）。
  两者**内容等价但不保证逐字一致**，切换时要 diff 一批样本。
- 我们历史结论里的 MathVista 数字来自「官方 query + 自建判分」，与 VLMEEvalKit/lmms-eval 的 judge 口径不同，
  **不能直接与 79.5 对比**；要对标就得按 §2/§3 复刻对应框架的判分。
