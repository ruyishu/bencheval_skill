---
name: mathvision
display-name: MathVision 评测复现
description: 复现 Qwen 官方 MathVision 多模态数学推理分数所需的完整评测条件（官方数据源与 MD5、prompt 结构、Instruct/Thinking 两套采样参数、规则+GPT-4o 两阶段判分、官方锚点分与常见踩坑）。当用户要跑/复现 MathVision、给某个模型测 MathVision、或问"为什么我的 MathVision 分数和官方对不上"时使用。
---

# MathVision 评测复现

MathVision：304（testmini）/ ~3000（full）道**带图**的数学题，多为选择题 + 自由作答混合，按学科 `category` 分类计分。

**官方口径一句话**：vLLM 推理 + **单条 user 消息（图片在前、question 在后，无 system prompt）** + **两阶段判分**（规则 prefetch 命中即采用，否则 GPT-4o 抽取）+ 与锚点对表。

配置的权威来源是 **Qwen 官方仓库里的专用脚本**（不是第三方框架）：
`https://github.com/QwenLM/Qwen3-VL/tree/main/evaluation/MathVision`（含 `infer_instruct.sh` / `infer_think.sh` / `eval_*.sh` / `run_mathv.py` / `eval_utils.py`）。
细节与逐字模板见 `references/official_config.md`。

## 0. 先确定对标哪个数字

Qwen3-VL 官方 MathVision（官方 model card，API 口径）：

| 变体 | MathVision |
|---|---|
| Qwen3-VL-4B-Thinking | **60.0** |
| Qwen3-VL-4B-Instruct | 51.6 |
| Qwen3-VL-8B-Thinking | 62.7 |
| Qwen3-VL-8B-Instruct | 53.9 |
| Qwen3-VL-2B-Thinking | 45.9 |
| Qwen3-VL-2B-Instruct | 31.6 |

⚠️ **官方未说明报的是哪个 split**。社区/榜单常报的「MathVision」= `MathVision_MINI`（304 题，testmini）；官方脚本默认跑 full（~3000 题）。**跑之前先跟用户确认 split**，并把用的 split 写进结果里。

## 1. 数据：只能用 VLMEvalKit 托管的那份 TSV

| split | URL | 题量 | MD5 |
|---|---|---|---|
| MathVision_MINI | `https://opencompass.openxlab.space/utils/VLMEval/MathVision_MINI.tsv` | 304 | `060fe4fa5d868987ce179307bd5f8a33` |
| MathVision | `https://opencompass.openxlab.space/utils/VLMEval/MathVision.tsv` | ~3000 | `93f6de14f7916e598aa1b7165589831e` |

**不要用 HF 上的 `MathLLMs/MathVision` 原始集**。根因：官方 TSV 已经把 **hint 和选项烧进 `question` 字段**（实测首行就是
`Hint: Please answer the question and provide the final answer at the end.\nQuestion: How many different digits can you find in this picture?`）。
原始集的选项在独立的 `choices` 字段里、question 里没有 —— 只拼 question 会**丢掉选项**，选择题系统性全错（这是最容易把分数打崩的一处）。

列名：`index, question, choices, image, answer, level, category`；`image` 是 base64，需落盘成图片再喂给模型。多图题 `image`/`image_path` 是列表。

## 2. Prompt 构造（逐字照做）

单条 user 消息，**无 system prompt**：

```
messages = [{"role": "user", "content": [
    {"type": "image", "image": <path>, "min_pixels": 768*28*28, "max_pixels": 5120*28*28},
    {"type": "text",  "text": line["question"]}          # 已含 hint + 选项
]}]
text = processor.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)
```

- 图片**在前**、问题文本在最后；多图则多张图排在前。
- 分辨率：`min_pixels = 768*28*28`（≈0.6M）、`max_pixels = 5120*28*28`（≈4M）——**比 Qwen2.5-VL 的常用默认值大**，调小会掉分。
- 可选 `--use-cot`：在文本末尾追加 `" Let's think step by step."`（官方默认**不开**）。

## 3. 采样参数（Instruct / Thinking 不同！）

| 参数 | Instruct | Thinking |
|---|---|---|
| max_new_tokens | 32768 | **40960** |
| temperature | 0.7 | **1.0** |
| top_p | 0.8 | 0.95 |
| top_k | 20 | 20 |
| repetition_penalty | 1.0 | 1.0 |
| presence_penalty | 1.5 | 0.0 |

vLLM：`max_model_len=128000`、`gpu_memory_utilization=0.9`、`limit_mm_per_prompt={"image": 10}`、`seed=42`、`trust_remote_code=True`、`VLLM_WORKER_MULTIPROC_METHOD=spawn`。

⚠️ **Thinking 的 temperature 是 1.0**：bench 专用脚本与**权重自带的 `generation_config.json`**（`temperature:1.0 / top_p:0.95 / top_k:20 / repetition_penalty:1.0`）**一致**；Qwen 主 README 全局推荐的那套（Thinking T=0.6）与之**冲突**。以 bench 脚本 + 权重 `generation_config` 为准。
（Instruct 侧三方一致：`generation_config` = T0.7 / top_p0.8 / top_k20，与脚本、README 都对得上；脚本额外加 presence_penalty 1.5。）

**Thinking 模型输出要按 `</think>` 切**：`prediction = response.split("</think>")[-1].strip()`，判分只用切分后的尾段（带思考的全文另存为 `gen_raw`）。实测 4B-Thinking 权重 `chat_template.json` 用的就是**字面 `<think>` / `</think>`**（不是全角特殊 token），与官方脚本一致；换模型后建议复核一遍，切错 token 会静默把思考段一起送去判分。

## 4. 判分：两阶段（规则 prefetch → GPT-4o 抽取）

**第一阶段 · 规则 prefetch**（命中即采用，不再调 API）：
- 有选项（`choices` 非空）→ `can_infer_option`：把 `.()[],:;!*#{}` 换成空格后切词，若**恰好命中一个**选项字母则该字母（有 `'A' in splits and len(splits) > 3` 的反例保护）；否则 `can_infer_text`：把选项**文本**小写后做子串匹配，恰好命中一个则取该字母。命中拒答话术 → `'Z'`。
- 无选项 → 直接拿原响应当答案。

**第二阶段 · GPT-4o 抽取**（prefetch 未命中时）：
- judge = `gpt-4o-2024-05-13`（DashScope 或 OpenAI 兼容端点），`max_tokens=4096`，逐条串行/多进程（官方 `--nproc 16`）
- prompt = 固定 task_description + **5 个 in-context example** + `line['question']` + `Model response: <prediction>` + `Extracted answer: `（ICE 逐字文本见 references，**照抄**）
- 重试 5 次，第 i 次 `temperature = i * 0.5`（0 / 0.5 / 1.0 / 1.5 / 2.0）
- judge 返回的**原文**直接作为 `res`（不做后处理）

**最终比对** `is_equal(res, ans)`：小写 strip 相等 → `eval()` 后数值差 < 1e-6 → `latex2sympy` 符号相等。有选项时先把 `res` 过一遍 `can_infer` 取字母再比。

## 5. 汇报口径（不报这些等于没法判断可信度）

至少给：`Overall acc`、**per-category acc**、`prefetch_rate`、以及 rule / gpt-4o 各自的抽取成功率。
官方脚本还会输出 `Subject | tot | prefetch | hit | prefetch_rate | acc` 的 CSV。

**只有规则判分、没有 judge 兜底时，必须显式说明**——否则分数不可与官方比（规则判分对自由作答的数学题漏检很多）。

## 6. 坑（按踩坑概率排序）

1. **选项丢了**：用了 HF 原始集而不是 VLMEvalKit TSV → 选择题全错。先查 prompt 里有没有选项文本。
2. **temperature 用错**：Thinking 用 0.6 而非 1.0（或反之），分数会有几个点漂移。
3. **输出没切 `</think>`**：Thinking 的思考段进了判分文本，抽取器容易抓错。
4. **判分只有规则**：prefetch 对自由作答命中率有限，会系统性偏低。
5. **图像分辨率用默认值**：丢失细节题（图表/几何）掉分。
6. **`max_model_len` 与显存**（已实测）：L20 46G + 4B-Thinking 跑 `max_model_len=128000` 时 KV cache 只有 193,792 tokens → **最大并发 1.51x**，聚合输出 ~40 tok/s、每题 15–20 s。官方 README 也提示显存不够就降；降到 49152（≈图片 prompt 上限 + 40960 输出）并发可到 ~4x。这是**服务参数、不改评测语义**，前提是没有序列被截断——降了必须检查 `finish_reason=length` 占比。
7. **API judge 的失败静默**：`extract_flag=False` 的样本若当作「答错」，会把分数压低；要单独统计。

## 7. 复现 checklist

- [ ] 明确 split（MINI 304 / full ~3000）并写进结果
- [ ] 数据 MD5 校验通过
- [ ] 抽查 3 条 prompt：含 hint、含选项（选择题）、图片在前
- [ ] 采样参数按变体取（Instruct/Thinking 两套），并用权重自带 `generation_config.json` 复核
- [ ] Thinking 输出按 `</think>` 切分（换模型后复核该 token）
- [ ] 判分两阶段都启用，且单独统计 prefetch / judge 成功率
- [ ] 结果与 0. 节锚点对表，**差值写进结论**（差 ≥3pp 时先怀疑上面 6 条，而不是直接下"复现成功/失败"）
