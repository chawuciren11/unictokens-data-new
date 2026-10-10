# UniCTokens Generation Data

用于 UniCTokens / Sync-R1 个性化生成实验的结构化训练与测试标注，包含 **20 个概念、40 个 JSON 文件**。

## 文件与下载

- `concept/train/<concept>/info.json`：20 个训练侧知识与提示文件。
- `concept/test/<concept>/t2i_conditions.json`：20 个测试侧知识与提示文件。
- [下载全部 40 个文件的 ZIP](./unictokens-generation-data-40-files.zip)：保留上述目录结构，ZIP 内仅包含这 40 个 JSON。
- [SHA-256 清单](./manifest.json)：记录每个 JSON 及 ZIP 的校验值。

本仓库是生成任务标注子集。图片、模型权重、Show-o 分阶段问答文件、Rec/VQA/QA 数据和评测结果不包含在此包中。

## 结构化题目数量

| 类别 | 每个概念训练 | 每个概念测试 | 全部训练 | 全部测试 |
|---|---:|---:|---:|---:|
| single | 12 | 6 | 240 | 120 |
| noisy | 8 | 4 | 160 | 80 |
| compositional | 6 | 3 | 120 | 60 |
| missing | 4 | 2 | 80 | 40 |
| 合计 | 30 | 15 | 600 | 300 |

以上统计针对四个结构化列表；测试 JSON 同时保留原有的其他生成提示字段。
