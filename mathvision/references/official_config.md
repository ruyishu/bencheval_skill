# MathVision 官方配置 · 逐字细节与信源

本文件是 `SKILL.md` 的深水区：逐字模板、判分代码语义、官方文件清单与信源 URL。
所有内容来自 **Qwen 官方仓库** 与 **VLMEvalKit 源码**（2026-10-13 实读）。

## 1. 信源清单

| 内容 | 来源 |
|---|---|
| 官方评测脚本 | `https://github.com/QwenLM/Qwen3-VL/tree/main/evaluation/MathVision` |
| 推理/判分主逻辑 | `evaluation/MathVision/run_mathv.py` |
| 抽取与判分 | `evaluation/MathVision/eval_utils.py` |
| 数据加载 | `evaluation/MathVision/dataset_utils.py` |
| 数据（官方 TSV） | `https://opencompass.openxlab.space/utils/VLMEval/MathVision{,_MINI}.tsv` |
| 分数锚点 | Qwen3-VL model card 图 `qwen3vl_4b_8b_vl_{thinking,instruct}.jpg`、`qwen3vl_2b_32b_vl_*.jpg` |

官方 `infer_think.sh` 原文（参数即此处）：

```bash
python run_mathv.py infer \
    --model-path /path/to/Qwen3-VL-Thinking \
    --dataset MathVision --output-file results/mathvision_predictions_thinking.jsonl \
    --max-new-tokens 40960 --temperature 1.0 --top-p 0.95 --top-k 20 \
    --repetition-penalty 1.0 --presence-penalty 0.0
```

官方 `eval_think.sh` 原文：

```bash
python run_mathv.py eval \
    --input-file results/mathvision_predictions_thinking.jsonl \
    --dataset MathVision \
    --eval-model gpt-4o-2024-05-13 --api-type dash --nproc 16
```

`--api-type dash` 读 `CHATGPT_DASHSCOPE_API_KEY` + `DASHSCOPE_API_BASE`（DashScope 兼容模式）；
`--api-type mit` 读 `MIT_SPIDER_TOKEN` + `MIT_SPIDER_URL`。

## 2. 数据字段与坑（实测）

`MathVision_MINI.tsv` 表头：
```
index | question | choices | image | answer | level | category
```
首行 `question` 实际值（注意 hint 被烧进字段）：
```
Hint: Please answer the question and provide the final answer at the end.
Question: How many different digits can you find in this picture?
```
→ 因此**不要再自己拼 hint**，直接用 `question` 即可；但**必须用这份 TSV**（原始 HF 集没烧）。

`image` 为 base64；`dump_image()` 会落到 `$LMUData/images/MathVision/<index>.jpg`（`$LMUData` 由 `--data-dir` 设置）。
选项题型的选项在被烧进 `question` 的同时，`choices` 列仍单独存在（判分阶段要用，格式是 `"['A','B']"` 这类字符串，需 `eval()` 取列表）。

## 3. 判分逐字细节

### 3.1 规则 prefetch（`post_check(line, prefetch=True)`）

```python
if len(eval(line['choices'])) > 0:            # 有选项
    choices = list_to_dict(eval(line['choices']))   # ['x','y'] -> {'A':'x','B':'y'}
    res = can_infer(response, choices)              # option 优先，其次文本匹配
else:                                          # 无选项
    res = str(response); ans = str(ans)
```

`can_infer_option(answer, choices)` 语义：
1. 命中拒答话术 `["Sorry, I can't help with images of people yet.", "I can't process this file.", "I'm sorry, but without the image provided", 'Cannot determine the answer']` → 返回 `'Z'`
2. 把 `'.()[],:;!*#{}'` 全替换成空格 → 切词
3. 统计命中选项字母个数：`==1` 时取该字母，但若 `'A' in splits and len(splits) > 3` 则判失败（防"That is A ..."这类误命中）
4. `count==0` 且恰好命中 `{'Z',''}` 之一 → `'Z'`；其余返回 `False`

`can_infer_text(answer, choices)`：全部小写后，选项**文本**作为子串恰好命中一个 → 返回其字母，否则 False。
`can_infer = can_infer_option or can_infer_text`。

### 3.2 GPT-4o 抽取

prompt 组装（`build_mathv_gpt4_prompt`）：

```
Please read the following example.
Then extract the answer from the model response and type it at the end of the prompt.

<ICE_1>

<ICE_2>
...

<ICE_5>

<question>
Model response: <prediction>
Extracted answer: 
```

5 个 ICE 逐字（`get_gpt4_ICE()`）：

```
Hint: Please answer the question and provide the final answer at the end.\n
Question: Which number is missing?\n
Model response: The number missing in the sequence is 14.\n
Extracted answer: 14
```
```
Hint: Please answer the question and provide the final answer at the end.\n
Question: What is the fraction of females facing the camera?\n
Model response: The fraction of females facing the camera is 0.6,
which means that six out of ten females in the group are facing the camera.\n
Extracted answer: 0.6
```
```
Hint: Please answer the question and provide the final answer at the end.\n
Question: How much money does Luca need to buy a sour apple candy and a butter-scotch candy? (Unit: $)\n
Model response: Luca needs $1.45 to buy a sour apple candy and a butterscotch candy.\n
Extracted answer: 1.45
```
```
Hint: Please answer the question and provide the final answer at the end.\n
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

调用参数：`max_tokens=4096`，第 i 次重试 `temperature=i*0.5`，失败标记 `'Failed to obtain answer via API.'`。
返回值：`dict(log=..., res=<judge 原文>, extract_model='rule'|'gpt-4o-...'|..., extract_flag=bool)`。

### 3.3 最终比对 `is_equal(asw, gt)`

```
lower+strip 相等
→ eval(asw) / eval(gt) 数值差 < 1e-6
→ latex2sympy 后数值或符号差 < 1e-6（需 pip install latex2sympy2，缺了就跳过这一档）
```

### 3.4 汇总 `MATH_V_acc`

按 `Subject`（`category`）+ `Overall` 输出：`tot / prefetch / hit / prefetch_rate(%) / acc(%)`，
并附 `rule_success/rule_all`、`gpt-4o-..._success/_all` 两列抽取成功率。

## 4. 采样参数的三方交叉验证（2026-10-13 实测权重）

| 来源 | Instruct | Thinking |
|---|---|---|
| 官方 bench 脚本（`infer_*.sh`） | T0.7 / top_p0.8 / top_k20 / rep1.0 / **presence1.5** / out32768 | T**1.0** / top_p0.95 / top_k20 / rep1.0 / presence0.0 / out40960 |
| 权重 `generation_config.json` | T0.7 / top_p0.8 / top_k20 / rep1.0 | T**1.0** / top_p0.95 / top_k20 / rep1.0 |
| Qwen 主 README「全局推荐」 | T0.7 / top_p0.8 / top_k20 / presence1.5 | T**0.6** / top_p0.95 / presence0.0 |

结论：
- **Instruct 三方一致**（presence_penalty 只出现在脚本/README，不在 generation_config）。
- **Thinking 的 temperature 有冲突**：脚本与权重 `generation_config` 都是 **1.0**，主 README 写 0.6。以脚本 + generation_config 为准（2:1，且两者是"实际执行"方）。
- `generation_config` 里 `do_sample: true`、`eos_token_id: [151645, 151643]`。

Thinking 输出切分（官方脚本逐字）：
```python
response_final = str(response).split("</think>")[-1].strip()   # gen  = 尾段，用于判分
# 全文保留在 gen_raw
```
实测 4B-Thinking 的 `chat_template.json` 中出现的就是字面 `<think>` / `</think>`（无全角特殊 token 变体）。

## 5. 已知不确定项（别当成事实传播）

- **官方报的 split 未标明**：脚本默认 full（~3000），榜单常在 MINI（304）上报。数字不可跨 split 比。
- **官方 model card 注**：默认评测走 **API** 调用，且视频类用 256k 上下文；我们本地权重 + 自建判分**只能逼近**。
- 官方 README 未给出 judge 的 `system prompt` / 重试策略以外的细节；上面是从源码读出的**实际行为**。

## 6. 与我们既有实现的差异（迁移提醒）

`/zk/bin/mmopsd_official_eval.py` 的 mathvision 分支历史上用的是 **HF `MathLLMs/MathVision` 原始集 + 自拼 prompt**，
因此出现过「选项没进 prompt → 选择题≈0 分」的结论。**迁移到本 skill 时，第一件事是换成 VLMEvalKit TSV 并复核选项是否在 prompt 里。**
