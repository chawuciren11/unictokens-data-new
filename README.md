# UniCTokens Generation Data

用于 UniCTokens / Sync-R1 个性化生成实验的结构化训练与测试标注，包含 **20 个概念、40 个 JSON 文件**。

## 文件与下载

- `concept/train/<concept>/info.json`：20 个训练侧知识与提示文件。
- `concept/test/<concept>/t2i_conditions.json`：20 个测试侧知识与提示文件。
- [下载全部 40 个文件的 ZIP](./unictokens-generation-data-40-files.zip)：保留上述目录结构，ZIP 内仅包含这 40 个 JSON。
- [SHA-256 清单](./manifest.json)：记录每个 JSON 及 ZIP 的校验值。

本仓库是生成任务标注子集。图片、模型权重、Show-o 分阶段问答文件、Rec/VQA/QA 数据和原始评测结果不包含在此包中；下文记录一版历史结果及其完整评分口径。

## 结构化题目数量

| 类别 | 每个概念训练 | 每个概念测试 | 全部训练 | 全部测试 |
|---|---:|---:|---:|---:|
| single | 12 | 6 | 240 | 120 |
| noisy | 8 | 4 | 160 | 80 |
| compositional | 6 | 3 | 120 | 60 |
| missing | 4 | 2 | 80 | 40 |
| 合计 | 30 | 15 | 600 | 300 |

以上统计针对四个结构化列表；测试 JSON 同时保留原有的其他生成提示字段。

## 四能力 Metric：从 meta 评测到最终得分

本节对应下表这一版历史结果，尤其以 SFT-15 的 `20.78 / 13.33 / 18.83` 和 SFT-20 的 `26.78 / 12.22 / 16.67` 标识版本。2026-10-10 使用服务器上的逐样本数据复算，表中 **9 行 × 4 列共 36 个数值均一致（保留两位小数）**。这一定义适用于该表，不把不同评测批次或指标统称为“最新版”。

| Method | Single ↑ | Compositional ↑ | Noisy ↑ | Missing ↑ |
|---|---:|---:|---:|---:|
| SFT-15 | 20.78 | 13.33 | 18.83 | 0.00 |
| SFT-20 | 26.78 | 12.22 | 16.67 | 0.00 |
| Cycle 40 | 42.56 | 38.22 | 30.67 | 100.00 |
| Cycle 50 | 42.89 | 30.67 | 40.67 | 100.00 |
| Cycle 60 | 38.44 | 44.44 | 31.83 | 100.00 |
| Cycle 70 | 45.78 | 34.89 | 40.17 | 100.00 |
| Cycle 80 | 39.11 | 23.33 | 37.67 | 100.00 |
| Cycle 90 | 40.89 | 44.89 | 37.17 | 100.00 |
| Cycle 100 | 43.22 | 34.44 | 34.17 | 100.00 |

该表仅覆盖 `adrien_brody`、`bo`、`dunpai` 三个概念。每个 setting 有 Single 18 题、Compositional 9 题、Noisy 12 题、Missing 6 题，共 45 题，其中 39 题需要图像评审。它不是本仓库全部 20 个概念的评测结果；Missing 结果对应 2026-10-10 六条 Missing 提示修订前的题目，尚未重测本仓库修订后的 Missing 提示。

### 1. 输入、生成与评审对象

从 `concept/test/<concept>/t2i_conditions.json` 读取概念类别、`extra_info`、`noisy_info` 和各题标注。主要字段的用途如下：

| 字段 | 用途 |
|---|---|
| `t2i_prompt` | 给被测模型的原始请求 |
| `t2i_exact_prompt` | 人工标注的完整目标描述，作为图片对齐评审的标准 |
| `grounding_info` | 完成本题必须体现的知识条目 |
| `extra_info` | 概念知识条目；其中未被本题选为 grounding 的条目不自动成为生成要求 |
| `noisy_info` | 干扰知识条目，用于检测噪声是否被转移进生成图 |
| `qwen_raw_output` | 生成阶段保存的 Qwen 原始输出，用于审计及 Missing 判定 |
| `generation_skipped` | 生成阶段保存的是否跳过出图标志 |

前五项来自标注；后两项来自生成结果清单。Qwen 根据请求和提供的知识输出用于出图的提示，图像生成模型产生候选图；Missing 应输出 `None` 并停止出图。

对于 Single、Compositional、Noisy，每题通过一次多模态 judge 调用，同时评估三个原始维度。该批评测配置的 judge 名称为 `gpt-5.6-terra`。输入依次包含：

1. 目标概念的参考图：优先使用 `concept/test/<concept>/0.png`，没有时选择该目录按文件名排序后的首张支持格式图片。
2. 当前题的生成图：本批取 `image_000.png`。
3. 概念 token、概念类别、全部信息条目，以及标注的 `t2i_exact_prompt`。

全部信息条目按 `extra_info + noisy_info` 的顺序拼接，分配 `I01`、`I02` 等 ID。评审标准使用标注的 exact prompt，不把模型自己改写的提示当作正确答案。评审只依据生成图中可见内容；文字中提到某个属性，不代表图里已经体现。

### 2. Meta 评测：judge 输出什么

Judge 输出的是以下三个原始维度及可见证据，最终四能力分数由后处理代码计算。

**主体相似度 `concept_similarity.score`（记为 S）**：比较生成主体与参考主体的稳定身份或固有外观。

| 分数 | 判断标准 |
|---:|---|
| 0.0 | 主体缺失、完全不同，或无法辨认 |
| 0.2 | 只有大类相同，几乎没有个体特征相似性 |
| 0.4 | 有部分特征相似，但更可能是不同个体或实例 |
| 0.6 | 较相似，可能是同一主体，但仍有明显差异 |
| 0.8 | 高度相似，很可能是同一主体，只有轻微差异 |
| 1.0 | 明确是同一主体，身份及固有外观高度一致 |

人物看稳定面部身份等特征；动物看个体脸部、毛色分布等特征；物品看形状、结构、材质和独特组件。背景、姿势、光照或临时服饰不同，本身不应降低该分数。

**逐条信息体现 `information_reflection[].score`（记为 rᵢ）**：对全部信息条目分别判断，不预设所有条目都应该出现在图中。

| 分数 | 判断标准 |
|---:|---|
| 0.0 | 未体现、对应内容缺失，或与信息矛盾 |
| 0.5 | 有部分正向视觉证据，但关键属性、对象、动作或关系不完整或不确定 |
| 1.0 | 信息描述的可见内容完整、清晰且正确体现 |

完全遮挡、裁切、太小或模糊而无法确认的内容记 0；只有实际存在部分正向证据才记 0.5。属性必须落在正确对象上，空间关系必须正确，不能仅凭对象同时出现就判满分。对于“最喜欢”“通常”等抽象关系，评估其可见表现，例如“最喜欢的帽子是灰色”可由主体戴着灰帽子体现，不要求图片证明“最喜欢”。

若噪声条目描述另一个概念，检查该条目的属性、动作或场景是否被错误应用到当前目标主体。例如，其他人的蓝色地毯出现在目标人物的卧室场景中，也算噪声转移，不要求另一人物同时出现。

**完整提示对齐 `exact_prompt_alignment.score`（记为 A）**：整体判断生成图与 `t2i_exact_prompt` 的可见内容是否一致。

| 分数 | 判断标准 |
|---:|---|
| 0.0 | 与要求无关，或主要场景缺失 |
| 0.2 | 只有微弱或泛化的一致性，大部分关键内容缺失或错误 |
| 0.4 | 体现部分主要内容，但多项重要要求缺失或错误 |
| 0.6 | 主要场景及相当一部分要求正确，仍有至少一项重要要求缺失、不清晰或错误 |
| 0.8 | 几乎所有重要内容正确，仅有轻微遗漏或不确定性 |
| 1.0 | 所有重要可见要求完整正确，无明显矛盾 |

该维度覆盖动作、物品、属性、数量、场景、空间关系及 grounded 细节。主体身份相似度单独由 S 评价，不仅因身份不够像而重复扣 A；但主体的大类仍须正确。除非 exact prompt 明确要求，不因画面漂亮、写实或构图好而加分。

结果文件的相关结构如下，仅为字段示例，实际须包含所有信息 ID：

```json
{
  "status": "complete",
  "concept": "bo",
  "prompt_group": "single",
  "grounding_info": ["对应的必要信息文本"],
  "information_items": [
    {"id": "I01", "text": "对应的必要信息文本"}
  ],
  "judge": {
    "parsed": {
      "concept": "<bo>",
      "concept_similarity": {"score": 0.8, "evidence": "主体相似度证据"},
      "information_reflection": [
        {"id": "I01", "score": 0.5, "evidence": "部分体现的可见证据"}
      ],
      "exact_prompt_alignment": {"score": 0.8, "evidence": "整体对齐证据"}
    }
  }
}
```

解析后必须校验：概念 token 正确、分数属于允许的离散集合、每个输入信息 ID 恰好出现一次且顺序一致。缺 ID、重复 ID、额外 ID 或非法分数属于评审错误，不能直接当成有效 meta 数据。

### 3. 从 meta 提取计算字段

先通过信息 ID 将 `information_reflection` 的分数与 `information_items` 的文本对应起来，再用标注文本选取必要信息和噪声信息。

设 R 为该题 `grounding_info` 对应的信息集合，N 为该概念全部 `noisy_info` 对应的信息集合：

```text
S = judge.parsed.concept_similarity.score
A = judge.parsed.exact_prompt_alignment.score
G_min = min(rᵢ for i in R)          # target_grounding_min
N_max = max(rⱼ for j in N)          # noisy_info_max
```

可回答题的 R 必须非空且每项能映射到 meta 分数。`compare_rl_to_sft.py` 在 N 为空时将 N_max 设为 0。其他原始字段也可能被保存，例如 `target_grounding_mean`、`non_target_extra_mean`、`noisy_info_mean`，但它们不进入本表的四能力公式。

### 4. 单题得分

| 类别 | 单题得分（0–100） |
|---|---|
| Single | `100 * S * A * G_min` |
| Compositional | `100 * S * A * G_min` |
| Noisy | `100 * S * A * G_min * (1 - N_max)` |

Single 与 Compositional 的公式相同；Compositional 需要组合多条知识，因此任何一条必要信息得到 0，G_min 就为 0，整题归零。必要信息分为 1 和 0.5 时，G_min 为 0.5，不是均值 0.75。

仅 Noisy 类额外乘噪声因子：N_max 为 0 时不扣分，0.5 时减半，1 时整题归零。这里取最严重的一条噪声，不取噪声平均分。

例如 S=0.8、A=0.8、G_min=0.5：Single / Compositional 得 `100*0.8*0.8*0.5=32` 分；如果是 Noisy 且 N_max=0.5，则得 16 分。

### 5. Missing 的判定

Missing 不做上述图片 meta 评审，也不对空的 grounding_info 计算最小值。四能力脚本 `rl_missing_score` 使用：

```python
success = (
    str(prompt.get("qwen_raw_output", "")).strip() == "None"
    and prompt.get("generation_skipped") is True
)
sample_score = 100.0 if success else 0.0
```

允许首尾空白，但 `None.`、小写 `none`、带引号的 `"None"` 或附加解释不通过；仅跳过出图而未严格输出 `None` 也不通过。

版本差异须保留：`compare_rl_to_sft.py` 的 `missing_exact_none_rate` 只检查严格输出 `None`，不检查跳过标志；该字段与四能力脚本的完整判定不能一般性混用。在本表这批数据中，RL 的 6 道 Missing 全部严格输出 None 并跳过出图；SFT 的严格 None 为 0/6，因此两种口径对应的表中数值一致。**本表 SFT 的 0 是实际拒绝失败的结果，不能解释成其他评测配置中的排除占位值。**

### 6. 从单题到表格

对类别 k，先计算每个概念 c 的该类题目平均分，再对概念等权平均：

```text
concept_score(c, k) = sum(sample_score(c, k, i)) / n(c, k)
table_score(k) = sum(concept_score(c, k)) / concept_count
```

本表三个概念的同类题数一致，所以也等于该类别所有题直接平均；如果题数不一致，两种平均方式就不等价。先计算逐题乘积，再做平均，不能用 S、A、G_min、N_max 的总体均值代入单题公式。中间值保留精度，最后展示时保留两位小数。本表没有定义将四列再平均的 Overall。

历史 CSV 汇总入口 `image_category_scores` 只读取 `status == "complete"` 的行；本表各类别的完整行数为 18/9/12，复现时应同时核对题数。RL 评审器对没有候选图的可回答题可写出全零 meta，状态为 `complete_deterministic_zero`，但该 CSV 汇总入口不会自动包含这种状态。新评测如出现这种情况，应明确将其以零分纳入应测分母并记录口径变化，不能静默丢弃后仍声称完整评测。Judge 调用失败或结果非法也须作为评测未完成处理，不能当作正常有效结果。

### 7. 本表的数据来源与脚本对应关系

以下是实验项目中的相对路径，用于定位原始结果；这些脚本、图片和逐题 judge 文件未随本标注仓库上传。

| 内容 | 项目及路径 |
|---|---|
| 图像 meta 评审入口 | `Sync-R1/evaluation/scripts/evaluate_qwen_images_grounded_vlm.py` |
| RL 评审入口 | `UniCTokens/joint_qwen_showo_grpo/evaluate_rl_grounded.py` |
| Judge 提示词、分档及输出校验 | `Sync-R1/evaluation/scorers/grounded_vlm_judge_template.py` |
| Meta 转逐样本字段、SFT/RL 匹配 | `UniCTokens/joint_qwen_showo_grpo/compare_rl_to_sft.py` |
| 四能力公式及概念平均 | `Sync-R1/evaluation/scripts/compute_four_capability_scores.py` |
| RL 各行输入 | `Sync-R1/joint_grpo_evaluation_v4_cycle{40,50,60,70,80,90,100}/comparison/matched_sample_rows.csv` 中 `method=joint_grpo` |
| 本表 SFT 两行输入 | `Sync-R1/joint_grpo_evaluation_v4_cycle100/comparison/matched_sample_rows.csv` 中 `method=sft_epoch15` / `sft_epoch20` |
| 原始 meta 文件布局 | `<generation_root>/judgments/gpt-5.6-terra/<concept>/epoch_<n>/<prompt_id>.json` |
| Missing 清单及统计 | `<generation_root>/enhanced_prompts/<concept>.json`、各 comparison 目录的 `rl_vs_sft.json` |

SFT/RL 对比脚本默认使用 `Sync-R1/qwen3_enhanced_checkpoint_generation` 作为 SFT 评测根目录。本表按上述 matched CSV 的 SFT 行重算。不要直接拿 `qwen3_enhanced_checkpoint_generation_fair_v2/four_capability_summary.json` 的 SFT 行替换：其中 SFT-15 为 `7.56 / 8.89 / 0.67`，SFT-20 为 `5.67 / 11.11 / 0.33`，属于不同的 SFT 评测数据来源。

### 8. 与其他 Metric 版本的区别

- **IRA**：后来单独确定的信息选择指标，单题公式为 `50 * (target_grounding_mean + 1 - non_target_extra_mean)`。Single、Compositional、Noisy 都用同一式，Missing 不参与；Overall IRA 对 39 个可回答样本平均。它不生成本表数值，且依据最终图片 meta，仅是信息选择的间接评价。
- **Strict Score**：`100 * S * A * G_min * (1 - N_max)`，对所有可回答类别都乘噪声因子；本表只对 Noisy 类乘该因子。它与四能力表不是同一个汇总指标。
- **旧的几何平均辅助函数**：`grounded_vlm_judge_template.py` 中另有 `compute_sample_scores`，其 `final_score` 为 `100 * (S * A * G_min * (1 - N_max)) ** (1/3)`。本表通过四能力脚本读取原始 meta 后计算，**没有使用此开立方公式**，不能仅看同文件中的 `final_score` 就判断本表口径。
