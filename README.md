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

## Missing 协议

当提示所要求的某项个性化事实无法从提供的 EXTRA INFO 中唯一确定时，模型应输出严格的 `None`，而不是猜测。即使一部分信息已知，仍缺少另一项必要信息时，也属于 Missing。

标注形式为：

```json
{
  "t2i_prompt": "<bo> is standing beside their usual grooming brush.",
  "t2i_exact_prompt": null,
  "grounding_info": []
}
```

## 2026-10-10 修订

替换了 butin、bo、mam、maeve_dog、skulls_mug、mydieu 各自第一条 Missing 测试提示，涉及的未知事实分别为旅行提笼、梳毛刷、猫抓柱、冬靴、旋转展台和猫隧道。

这六项均保持 `t2i_exact_prompt: null` 和 `grounding_info: []`。每个概念的题目数量保持不变。规范化文本检查确认，四类结构化题在同概念训练/测试之间的完全重复由 6 条降至 0 条；这不等同于证明不存在语义相似样本。

历史 Missing 分数属于修订前题目，本仓库未附带修订后的评测成绩。重测时应使用新的输出目录或刷新对应缓存。

## 校验

40 个 JSON 已通过解析、分组数量和 Missing 标注检查；六个修订文件的 SHA-256 与服务器部署版本一致。

ZIP SHA-256：

```text
c06cc7caee722378467a53f7ce12fd9cf3ee5915ce7f63a05f22cb888c6a7b5d
```
