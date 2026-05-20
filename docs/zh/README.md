# pkmn/engine 中文文档

> 本目录是 `docs/` 下英文设计文档的中文翻译版本。在忠于原文的基础上，**适当扩写了技术背景、关键概念解释、以及一些便于理解的例子**，方便中文读者快速上手。
>
> 如果你想看原文，请前往 [`docs/`](../) 上层目录。如果你想从"项目整体长什么样"开始读，可以先看一遍 [`docs/analysis/ENGINE_ANALYSIS_zh.md`](../analysis/ENGINE_ANALYSIS_zh.md)。

## 推荐阅读顺序

不同身份的读者建议按下面的顺序阅读：

| 你是谁 | 推荐顺序 |
| --- | --- |
| **想了解项目是干嘛的** | [`ENGINE_ANALYSIS_zh.md`](../analysis/ENGINE_ANALYSIS_zh.md) → [`DESIGN_zh.md`](./DESIGN_zh.md) |
| **想用 `@pkmn/engine` 写驱动 / AI** | [`DESIGN_zh.md`](./DESIGN_zh.md) → [`PROTOCOL_zh.md`](./PROTOCOL_zh.md) |
| **想给引擎贡献代码** | [`CONTRIBUTING_zh.md`](./CONTRIBUTING_zh.md) → [`DESIGN_zh.md`](./DESIGN_zh.md) → [`TESTING_zh.md`](./TESTING_zh.md) → [`NOTES_zh.md`](./NOTES_zh.md) |
| **想新增世代支持** | 先把上一行读完，再读 [`NOTES_zh.md`](./NOTES_zh.md) 与 [`RESEARCH_zh.md`](./RESEARCH_zh.md) |
| **想性能/对比分析** | [`TESTING_zh.md`](./TESTING_zh.md) 的 "Benchmark / 基准测试" 一节 |

## 文档列表

| 中文版 | 原文 | 内容概要 |
| --- | --- | --- |
| [`CONTRIBUTING_zh.md`](./CONTRIBUTING_zh.md) | [`CONTRIBUTING.md`](../CONTRIBUTING.md) | 如何参与贡献：风格规范、PR 流程、上游项目说明 |
| [`DESIGN_zh.md`](./DESIGN_zh.md) | [`DESIGN.md`](../DESIGN.md) | 引擎核心设计思想：性能优先、数据导向、`-Dshowdown` / `-Dlog`、项目结构、与 Pokémon Showdown 的对比 |
| [`NOTES_zh.md`](./NOTES_zh.md) | [`NOTES.md`](../NOTES.md) | 开发笔记：新增世代的 23 步实践流程、依赖升级流程、调试技巧 |
| [`PROTOCOL_zh.md`](./PROTOCOL_zh.md) | [`PROTOCOL.md`](../PROTOCOL.md) | 引擎二进制日志协议完整参考：Choice / Result / Message / PokemonIdent / `MAX_LOGS` 上界推导 |
| [`RESEARCH_zh.md`](./RESEARCH_zh.md) | [`RESEARCH.md`](../RESEARCH.md) | 各世代游戏反编译 / RNG / 漏洞研究 / 其它引擎实现的资料索引 |
| [`TESTING_zh.md`](./TESTING_zh.md) | [`TESTING.md`](../TESTING.md) | 测试体系：单元测试、集成测试、基准测试、Fuzz 测试、回归测试 |

## 名词对照表

为了行文流畅，本中文版在多数情况下保留英文术语，但下面这些概念的常见中文译法做了统一：

| 英文 | 中文（本文档使用） | 备注 |
| --- | --- | --- |
| Pokémon battle engine | 宝可梦对战引擎 | |
| cartridge | 卡带 / 卡带原版 | 指原版 GB 游戏 ROM |
| Pokémon Showdown | Pokémon Showdown | 不译；偶尔写作"PS" |
| frame-accurate | 逐帧一致 | |
| bug-for-bug compatible | 行为完全等价（包括 bug） | |
| RNG / PRNG | 随机数生成器 / 伪随机数生成器 | |
| roll | 掷骰 / 掷骰结果 | RNG 产出的一次值 |
| handle | 句柄 | 直接索引到数组的小整数，对标"指针" |
| volatile status | 临时状态 | 如混乱、束缚等只在当前对战中存在的状态 |
| move | 招式 | |
| species | 种族 | 例如皮卡丘、喷火龙这种类别 |
| stat | 能力值 | HP/攻击/防御/速度等 |
| boost / unboost | 提升 / 下降 | 战斗内能力等级变化 |
| critical hit | 击中要害 / 暴击 | |
| OHKO | 一击必杀（One-Hit KO） | |
| speed tie | 速度并列 / 速度相同 | 双方速度一致时需掷骰决定先后 |
| protocol | 协议 | 指引擎日志的二进制协议 |
| driver code | 驱动代码 | 包装引擎、提供更高层 API 的代码 |
| fuzz / fuzzing | 模糊测试 | |

---

> 翻译以"准确传达技术意图"为最高优先级；如果发现翻译错误或表述不清楚的地方，欢迎按 [`CONTRIBUTING_zh.md`](./CONTRIBUTING_zh.md) 中的流程提 Issue / PR。
