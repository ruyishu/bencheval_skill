# bencheval_skill

给 agent 用的 **benchmark 评测条件**技能库：每个 bench 一个 skill，把「复现官方分数」所需的全部条件写死——
**数据源与校验值、prompt 结构、采样参数、答案抽取与判分、官方锚点分、已知踩坑**。

## 为什么存在

同一个模型、同一个 benchmark，换一套 prompt / 抽取器 / temperature / top_p / top_k，分数就能差好几个点，
于是「对不上官方」变成了最大的复现噪声。这个仓库把这些条件变成**可被 agent 自动匹配、可被 diff 的配置**。

设计约定：

1. **官方优先**：条件必须来自官方（官方仓库脚本 / 官方自述框架 / 官方数据文件），并在 `references/` 里给出信源 URL。
2. **不确定就标出来**：官方没说的（例如报的是哪个 split、用了哪套框架）一律写进「已知不确定项」，不许当事实传播。
3. **带锚点**：每个 skill 必须有该 bench 的官方分数表，跑完要对表并报差值，而不是只说"跑完了"。
4. **报口径**：结果必须能回答"用哪套数据、哪套 prompt、哪个 judge、什么参数"，否则数字不可比。

## 结构

```
<bench>/
├── SKILL.md                      # 主文件：一句话口径、锚点、数据、prompt、参数、判分、坑、checklist
└── references/
    └── official_config.md        # 深水区：逐字模板、判分代码语义、信源清单、不确定项
```

`SKILL.md` 的 frontmatter 用 `name` + `description`（agent 靠 description 匹配触发场景），
可加 `display-name` 给中文展示名。

现有：

| skill | bench | 官方口径来源 |
|---|---|---|
| [`mathvision`](mathvision/SKILL.md) | MathVision（304 / ~3000） | Qwen 官方仓库 `evaluation/MathVision`（vLLM + GPT-4o judge） |
| [`mathvista`](mathvista/SKILL.md) | MathVista testmini（1000） | 无官方脚本；lmms-eval `mathvista_testmini_cot` / VLMEvalKit `MathVista_MINI` 两套 |
| [`mmmu`](mmmu/SKILL.md) | MMMU val（900） | Qwen 官方仓库 `evaluation/mmmu` + VLMEvalKit `MMMU_DEV_VAL` |
| [`mmstar`](mmstar/SKILL.md) | MMStar（1500） | VLMEvalKit `ImageMCQDataset` 通用路径 |
| [`mathverse`](mathverse/SKILL.md) | MathVerse mini（788） | VLMEvalKit `MathVerse`（两阶段 GPT 判分） |
| [`ocrbench`](ocrbench/SKILL.md) | OCRBench（1000） | VLMEvalKit `OCRBench`（纯规则，无 judge） |
| [`chartqa`](chartqa/SKILL.md) | ChartQA test（2500） | VLMEvalKit `ImageVQADataset`（relaxed accuracy 5%） |
| [`docvqa`](docvqa/SKILL.md) | DocVQA（VAL/TEST） | VLMEvalKit `ImageVQADataset`（ANLS 0.5） |
| [`hallusionbench`](hallusionbench/SKILL.md) | HallusionBench（~1129） | VLMEvalKit `ImageYORNDataset`（aAcc/fAcc/qAcc 平均） |

> 每个 skill 的判分细节（prompt 逐字模板、抽取规则、judge prompt、坑）**内联在 `SKILL.md`** 里；
> `mathvision` / `mathvista` 另有 `references/official_config.md` 放更长的逐字代码与信源清单。
> 其余 bench 的逐字细节足够短，暂不单开 references。

## 跨 bench 的共性结论（新写 skill 时先套这些）

1. **prompt 结构**：官方口径基本都是**单条 user 消息、图片在前、问题文本在后、无 system prompt**。
2. **判分是两阶段**：先规则 prefetch（命中即采用），未命中再用 LLM judge 抽取；**只报规则分不能与官方比**。
3. **必须用官方托管的数据文件**：官方 TSV 往往已经把 hint / 选项**烧进**字段里，
   改用 HF 原始集会静默丢信息（最典型：MathVision 选项丢失 → 选择题全错）。
4. **Instruct 与 Thinking 是两套参数**，且 **bench 专用脚本可能覆盖全局推荐值**（如 MathVision Thinking 用 T=1.0）。
5. **judge 差异是分数量级差异**：judge 模型、版本、`max_tokens`、重试温度都要记录，且**逐字保留官方 prompt（包括笔误）**。

## 新增一个 bench

1. 找官方口径：先翻 `QwenLM/Qwen3-VL/evaluation/`，没有再看官方自述的第三方框架源码。
2. 落 `SKILL.md`：锚点表 → 数据（含 MD5/URL）→ prompt → 采样参数 → 判分 → 汇报口径 → 坑 → checklist。
3. 落 `references/official_config.md`：信源 URL、逐字模板/ICE、判分代码语义、**已知不确定项 + 我们既有实现的差异**。
4. 抽查 3 条 prompt 是否真的带上 hint 与选项；跑通后与锚点对表，把差值写进结论。
