# 贡献指南（中文）

> 原文：[`docs/CONTRIBUTING.md`](../CONTRIBUTING.md)
>
> 这份文档介绍了如何参与 `pkmn/engine` 的开发：风格规范、Issue 与 PR 模板、以及"最有效的贡献方式"。中文版在原文基础上补充了一些上下文，帮助新贡献者快速融入项目。

## 阅读前提

在参与开发之前，请先把这些读一遍：

1. 仓库根目录的 [`README.md`](../../README.md) —— 了解项目是什么、当前状态如何（目前仍为 WIP，主分支可能有大量未完成的破坏性变更）。
2. [`docs/`](../) 目录下的全部设计文档（中文版可见 [`docs/zh/`](./)）：
   - [`DESIGN_zh.md`](./DESIGN_zh.md)：引擎的整体设计哲学（"No compromises on performance"）。
   - [`PROTOCOL_zh.md`](./PROTOCOL_zh.md)：二进制日志协议参考。
   - [`TESTING_zh.md`](./TESTING_zh.md)：测试体系如何运作。
   - [`NOTES_zh.md`](./NOTES_zh.md)：开发笔记。
   - [`RESEARCH_zh.md`](./RESEARCH_zh.md)：研究资料索引。
3. [`pkmn.cc/@pkmn`](https://pkmn.cc/@pkmn/) —— `@pkmn` 项目家族的整体介绍与约定。

## 代码风格

`pkmn/engine` 同时包含两种语言的代码，二者各自遵循不同的风格约定：

| 部分 | 语言 | 风格指南 |
| --- | --- | --- |
| 引擎核心（`src/lib/`） | Zig | 大致遵循 [TigerBeetle 的"老虎风格"](https://github.com/tigerbeetledb/tigerbeetle/blob/main/docs/TIGER_STYLE.md) |
| 驱动代码（`src/pkg/`） | TypeScript | 遵循 [`@pkmn` 项目通用风格](https://pkmn.cc/@pkmn/#style) |

### TigerBeetle 风格的核心思想（简译）

虽然完整规则请以原文为准，但其中几条对本仓库特别重要的"硬要求"值得在这里提一提：

- **函数尽量短**（建议 70 行以内），让审阅者一屏能看完整个控制流。
- **断言（assert）比注释更可信**：把不变式写成 `assert(...)`，编译期能催发的就用 `comptime`。
- **不动态分配内存**：所有缓冲区在初始化阶段就分配好，运行时只复用。
- **避免随机的"helper 方法"**：每一个新增的函数都必须有明确的、不重复的职责。
- **关注 cache line**：结构体的字段顺序、大小、对齐都要刻意设计（这是引擎极致性能的来源之一）。

如果你提交的 PR 引入了违反这些点的代码，多半会在 review 中被指出。

## 提 Issue 与 PR

**请使用已经存在的 Issue / PR 模板**（仓库中 `.github/ISSUE_TEMPLATE/` 与 `.github/PULL_REQUEST_TEMPLATE/` 有定义），并尽量把每一栏都填好。模板存在的目的是让维护者更快理解你的意图，少填了字段会让 review 周期被拉长。

### PR 是否一定要带测试？

- **绝大多数情况是的。** 仓库的工作流是"先有失败的测试 / 复现，再有修复"。没有测试的 PR 多半合并不进去。
- **例外：可以先开一个"求反馈/求帮助测试"的 PR**，在描述里说明清楚。维护者会和你一起想测试怎么写。

### 一个典型的小修复 PR 流程

下面给一个虚构但贴近实际操作的例子，帮你建立直觉：

> 你发现 Gen 1 在某个边界情况下与卡带行为不一致——比如某个招式在 PP 为 0 时的回退逻辑有问题。

1. **复现问题**：到 `src/lib/gen1/mechanics.zig` 找到对应分支，在 `src/lib/gen1/test.zig` 里加一个最小化的单元测试，让它**故意失败**。
2. **本地跑测试** 确认能复现：
   ```sh
   zig build test -Dshowdown=false
   ```
3. **改 mechanics**：在 `mechanics.zig` 里修正行为。
4. **同时在 Showdown 兼容模式下验证**：
   ```sh
   zig build test -Dshowdown=true
   ```
   如果两边的预期 roll 不一样，需要同时调整测试中的 `cartridge` 和 `showdown` 两组 rolls。
5. **集成测试**（可选但推荐）：
   ```sh
   npm run test:integration
   ```
6. **提 PR**，描述里贴上：
   - 复现链接（论坛帖、视频、glitchcity wiki 等）
   - 改动前后对应的行为对比
   - 引用了哪份反编译代码（pret/pokered 的具体文件与行号）

## 最高 ROI 的贡献方式

> **直接给引擎本身打补丁不是回报最大的方式。**

`pkmn/engine` 几乎所有的"正确性事实"都来源于两类上游项目：

1. **[`smogon/pokemon-showdown`](https://github.com/smogon/pokemon-showdown)**：目前的事实标准对战模拟器。引擎在 `-Dshowdown` 模式下要与它的"被打补丁后版本"逐 bit 对齐（细节见 [`TESTING_zh.md`](./TESTING_zh.md) 中"Patches"一节）。
   - **去 Showdown 仓库修 bug 或简化代码**，对引擎收益更大：每减少一个非确定性、就少一处需要在 pkmn 中绕过的"reproduce-a-bug"逻辑。
2. **[`pret`](https://github.com/pret) 反编译项目**（[`pokered`](https://github.com/pret/pokered)、[`pokecrystal`](https://github.com/pret/pokecrystal) 等）：这是卡带行为的"原始真相"来源。
   - **去 pret 仓库改进注释、补充常量名、整理可读性**，整个宝可梦开发社区都会受益，引擎在做"为什么这里要这样写"的考古时也会少猜很多。

换句话说：

> **想要 `pkmn/engine` 更好，最有效的不是给它写代码，而是让它依赖的上游更可靠。**

## 行为准则

`pkmn/engine` 隶属于 `@pkmn` 家族，遵守通用的 [Contributor Covenant](https://www.contributor-covenant.org/) 行为准则。简单概括：

- 友善、有建设性、对事不对人。
- 技术争论用证据和测试用例说话，不用立场。
- 维护者保留对不文明行为采取必要措施的权利。
