# 引擎设计（中文）

> 原文：[`docs/DESIGN.md`](../DESIGN.md)
>
> 这份文档讲的是 `pkmn/engine` 的**整体设计哲学**——为什么要这么写。中文版在原文基础上**大量补充了"为什么这个决定影响性能 / 怎么落到代码里 / 例子是什么"**，便于第一次读源码时建立全局直觉。
>
> 各世代具体的实现细节请看：
>
> - [Generation I](../../src/lib/gen1/README.md)（第一代：红蓝/黄）
> - [Generation II](../../src/lib/gen2/README.md)（第二代：金银/水晶）

---

## 核心原则：**No compromises on performance**（不为任何东西妥协性能）

`pkmn/engine` 把性能放在**绝对优先**的位置。当性能和"易用性 / 简洁性 / 便利性"冲突时，**永远是性能赢**。这条原则导出了下面这些具体的设计选择。

### 1. 引擎的**作用域比同类项目小得多**

| 项目 | 包含什么 |
| --- | --- |
| **原版卡带** | 整个 RPG：地图、剧情、菜单、对话…… |
| **Pokémon Showdown** | 完整模拟器 + 聊天室服务端 + 房间管理 + 队伍校验 + 格式定义…… |
| **`pkmn/engine`** | **只做** Pokémon Showdown 中 [`Battle`](https://github.com/smogon/pokemon-showdown/blob/master/sim/battle.ts) 类**子集**的事 |

具体来说，引擎**有意不做**的事情包括：

- **不提供 `BattleStream` 等价物**——Showdown 的 `BattleStream` 是异步、基于文本的，两者都会引入额外延迟。pkmn 是**同步、二进制**的。
- **不做队伍/格式/自定义规则校验**——这些应该由更高层（驱动代码、服务端）处理。
- **不做输入（choice）校验**——引擎假设外部传进来的 `Choice` 一定合法。要么驱动代码先校验，要么由"只会生成合法 choice 的代码"驱动它。

> 💡 **例子**：如果你写一个 MCTS（蒙特卡洛树搜索）的 AI 来跑大量随机对局，那你压根不需要"输入校验"——AI 永远只从 `choices()` 给出的合法选项里挑。引擎为这种"自己生成自己消费"的场景做了**所有妥协**。

### 2. [**数据导向设计**](https://github.com/dbartolini/data-oriented-design)（Data-Oriented Design, DOD）

为了**最小化缓存缺失、提升数据局部性**：

- 结构体字段**经过精心打包**，参考 [Structure Packing](http://www.catb.org/esr/structure-packing/)。位域、`packed struct`、按访问频率排序……能压就压。
- **抛弃指针**，改用 ['handles'](https://floooh.github.io/2018/06/17/handles-vs-pointers.html)（句柄）——本质上是**直接索引到数组的小整数**。

> 💡 **handle vs pointer 例子**：
>
> 想表示"当前出场的宝可梦"。
>
> - **pointer 写法**：`active: *Pokemon`，访问要解一次指针；如果整个 `Battle` 想被序列化，指针不能直接 memcpy。
> - **handle 写法**：`active: u4`（0~5 的队伍槽位号），加上 `team: [6]Pokemon`。访问是 `team[active]`，是一次内存读，编译器还能把数组放进 cache 里一起拉过来。整个 `Battle` 直接 `memcpy` 就能 clone / 序列化。
>
> 后者是 pkmn 的选择。

### 3. **每个世代独立实现**

代码或数据**只在零开销的情况下共享**。一代宝可梦的实现**不应该背负其它代的复杂度**。

- 最坏情况会有一些代码重复，但**任何单一世代都易于推理和优化**。
- 二进制体积仍然小，因为**数据被极度精简**，只留必要的字段。

> 💡 **对比 Pokémon Showdown**：PS 的核心代码**永远反映最新世代**，老世代通过"mods"打补丁实现。结果是即使你在跑 Gen 1，老世代代码也要**为现代世代的所有特性买单**——比如查找 ability 表（Gen 3 才有 ability），比如绕过气候/场地等机制。在 pkmn 里，`gen1/data.zig` 里根本**没有 `ability` 这个字段存在**。

### 4. **序列化 = 直接当字节数组用**

引擎的状态序列化**就是把结构体当字节数组**。对于日志[协议](./PROTOCOL_zh.md)，则是用"最快的方式"写字节。

带来的副作用：

- **协议和 API 因系统而异**：所有整数用**本机字节序**（native-endianness），因为这在任何一台具体机器上都是最快的读写方式。
- 跨机器传递日志时需要**驱动代码做字节序适配**（这是个明确的取舍）。

### 5. **不用字符串**

> "字符串是上层的事，引擎只处理小而高效的原始类型。"

所有标识符都用小 `enum` 表示，可以**直接索引到数据数组**，不需要哈希、不需要间接寻址。

> 💡 **例子**：Pokémon Showdown 把"皮卡丘"表示为字符串 `"pikachu"`，每次查物种数据要：`Dex.species.get("pikachu")` → 在 map 里查 hash → 拿到对象指针 → 读字段。在 pkmn 里，皮卡丘是 `Species.Pikachu`（一个 `u8` enum 值），查数据就是 `SPECIES[@intFromEnum(Species.Pikachu)]`——一条数组索引指令。
>
> PS 的 [`toID`](https://github.com/smogon/pokemon-showdown/blob/master/sim/dex.ts) 函数是其**最热的函数**之一，原因正是字符串到 ID 的反复转换。pkmn 绕过了这整类成本。

### 6. **永不动态分配内存**

引擎只实现已经存在的、跑在受限硬件上的、对战系统，所以可以要求用户**预分配固定大小缓冲区**。

- 没有 GC 暂停
- 没有 malloc/free 抖动
- 整个 `Battle` 在栈上或单一 buffer 中存活

> 💡 **这一条对 AI 场景特别关键**。MCTS 需要快速 clone 战斗状态——pkmn 的 `Battle` 是定长 POD，`clone = memcpy`，能在纳秒级完成；而 Showdown 中 clone 一个 `Battle` 需要重建大量对象、复制 map、重连引用，慢得多。

### 7. 数据结构设计成"**查找通常不需要**"

- 大多数情况用**范围检查**代替查找：状态码用连续整数表示，`if (status >= POISON and status <= TOXIC)` 一行搞定。
- 必须查找时用**高效线性扫描**（小数组下比 hash 快）。
- 极端情况下使用[**完美哈希**](https://en.wikipedia.org/wiki/Perfect_hash_function)，编译期把"id → 数据"的映射变成无碰撞、无探测的直接索引。

---

## "No compromises"最大的代价：**编译选项**

由于上述各种妥协都偏向"卡带行为"，要做"PS 兼容"或"输出日志"就必须**通过编译选项打开**——而不是在运行时分支判断。

引擎默认实现**卡带原版**行为。但在线竞技玩家社区约定俗成了一些为提升竞技性的修改，这些由 Pokémon Showdown 实现。打开 **`-Dshowdown`** 后，引擎切换到：

- **匹配 Showdown 的 RNG 语义**而不是卡带的。
  > Showdown 没有为每个世代实现正确的 PRNG——它**只实现了 Gen V & VI 的 PRNG，并把它套用到所有世代**，而且调用次数、参数、顺序都和卡带不一样。引擎在 `-Dshowdown` 下精确复现这套"错法"。
- **实现 Showdown 自身的 bug**。
- **实现 Showdown "Standard" 规则集的修改**（Endless Battle Clause、Sleep/Freeze/Desync/Switch Priority Clause Mod 等）。

集成测试会验证：`-Dshowdown` 下的引擎输出与（被打补丁的）Showdown 完全一致——见 [`TESTING_zh.md`](./TESTING_zh.md#patches-补丁)。

---

默认情况下，引擎**对战斗状态不产生任何输出**，只通过 `Result` 类型通报"是否结束/谁赢了/下一步谁选什么"。

但是卡带和 Showdown 在游玩时都会输出消息（"皮卡丘使用了十万伏特！" 之类）。某些用例需要这些信息（比如重放、调试、给人类显示），某些用例不需要（比如 MCTS 随机模拟）。

所以输出也是 **`-Dlog`** 编译选项。`-Dlog` 模式下产生的是**精简的二进制[协议](./PROTOCOL_zh.md)**，但信息**足以重建**与卡带 / Showdown 等价的人类可读日志。

> 💡 **完整编译选项组合**（建议放进 memory）：
>
> | 场景 | 推荐 flag |
> | --- | --- |
> | 跑大量随机模拟（AI / MCTS） | （什么都不开） |
> | 与 PS 比较 / 重放 | `-Dshowdown=true -Dlog=true` |
> | 调试 / 抓概率 | `-Dshowdown=true -Dlog=true -Dchance=true -Dcalc=true` |
> | 集成测试 | `-Dshowdown=true -Dlog=true` |

---

## 项目结构

- [`Makefile`](../../Makefile)：**顶层调度**，把 `build.zig`（Zig）和 `package.json`（JS）的任务串起来
  - [`build.zig`](../../build.zig)：所有 Zig 代码的构建
  - [`package.json`](../../package.json)：所有 JS 代码的构建
- [`examples`](../../examples)：跨所有支持平台的使用示例（C / JS / Zig 三套）
- [`src/lib`](../../src/lib)：**`libpkmn` 引擎核心**（Zig）
  - [`pkmn.zig`](../../src/lib/pkmn.zig)：Zig 库入口
  - [`c.zig`](../../src/lib/c.zig)/[`node.zig`](../../src/lib/node.zig)/[`wasm.zig`](../../src/lib/wasm.zig)：**对外暴露 libpkmn 给非 Zig 使用者**的代码（C ABI、Node.js native addon、WebAssembly）
  - [`common`](../../src/lib/common)：跨世代共享的代码（数据结构、RNG、协议逻辑）
  - `gen*/`：每代各自的实现
- [`src/pkg`](../../src/pkg)：**`@pkmn/engine` JS 包的驱动代码**
- [`src/test`](../../src/test)：高层测试代码（集成、基准、fuzz）。**单元测试不在这里**——它们以 inline test 形式与 `lib/` 和 `pkg/` 的代码放在一起。
- [`src/tools`](../../src/tools)：辅助脚本和工具

> 📐 **目录与设计原则的对应关系**：
>
> | 目录 | 哪条设计原则在主导 |
> | --- | --- |
> | `src/lib/gen1`、`src/lib/gen2` | "每代独立实现" |
> | `src/lib/common/protocol.zig` | "序列化 = 字节数组" + native-endianness |
> | `src/lib/common/rng.zig` | "no compromises on performance" + `-Dshowdown` 切换 |
> | `src/pkg/data` | "no strings in engine"——字符串处理被推到这里 |
> | `src/lib/c.zig` | 不动态分配内存的 C ABI |

---

## 附录：与前辈引擎的对比

> 这一节解释 `pkmn/engine` 为什么比"原版卡带"和"Pokémon Showdown"**更简单也更快**。读这一节最好结合 [`docs/analysis/ENGINE_ANALYSIS_zh.md`](../analysis/ENGINE_ANALYSIS_zh.md) 一起看。

### Pokémon Red & Blue（原版卡带）

原版游戏的战斗引擎是**在受限的硬件、紧迫的时间压力下，作为一个完整 RPG 的一部分**写出来的：

- [**GB Z80**](https://rgbds.gbdev.io/docs/v0.5.1/gbz80.7) 硬件**不能高效执行乘除指令**（现代 SIMD 更别提了）。
- 卡带战斗引擎里包含很多在"竞技对战"中**用不上的功能**：
  - ["老人捕捉教程"](https://bulbapedia.bulbagarden.net/wiki/Old_man_(Kanto))
  - [Safari Zone](https://bulbapedia.bulbagarden.net/wiki/Kanto_Safari_Zone)
  - 未识别的鬼（紫苑塔剧情）
  - 战斗内使用道具
  - "switch" vs. "set" 模式
  - 徽章加成 / 不听话
  - 捕捉宝可梦
  - 逃跑
  - 经验值
- 游戏数据是**有机生长的**，结果是**按"先来后到"而不是"按访问频率"摆放**，缓存非常不友好。

把这些精简掉、再用现代指令集重写，**复杂度和性能都能改善**。

### Pokémon Showdown!

> ⚠️ 注意：下面这一节是**对 Showdown 架构的技术批评**——这不是贬低，而是说明 PS 关注的是另一组约束（可扩展性、易开发），其架构与"极致性能"在根本上是冲突的。

PS 是宝可梦对战引擎的"**净室实现**"（clean room implementation），**专注于扩展性和易开发性**。这让新手能轻松创建自定义格式（实际上几小时内就能添加一整代新宝可梦）。但代价是若干**与峰值性能根本相悖**的设计取舍：

#### 单代码库支持所有世代

PS 的核心代码**反映当前世代**，老世代通过对数据文件和 handler 打补丁（"mods"）实现。

- ✅ 修改最新世代很方便。
- ❌ 核心流程因此**带着所有世代的分支与 hook**——简单的老世代代码**也要为现代世代的复杂度付费**。
- ❌ 老世代的某些行为可能**继承自更新的世代**，**与机制的实际历史演变相反**，使得"某段行为到底从哪里来"难以定位。

#### 通用事件系统

PS 围绕一个**自定义的通用事件系统**（带冒泡和优先级）展开。

- ✅ 非常强大。
- ❌ **事件分发开销大**，是引擎里最慢的部分。虽然这个瓶颈[早就被识别](https://pkmn.cc/optimize)并改进过，**但"搜索 handler"这种模式本身就慢**。
- 💡 一个更好的模型是**预注册 handler 而不是搜索**——目前事件循环要遍历所有可能的来源，即使通常只有 0 或 1 个 handler 真正需要运行。

#### 基础类型 `ID`

PS 最基础的类型是 **`ID`**——一个去掉特殊字符的小写字符串。

- ✅ 开发者可以瞄一眼就知道指的是谁。
- ❌ **依赖编译器做字符串 interning**，而且比整数占更多内存（JS 数字技术上都是 8 字节，但 V8 之类的运行时通常对 32 位整数做 ['Smi' 优化](https://github.com/v8/v8/blob/a9e3d9c7/include/v8.h#L253)）。
- ❌ PS 频繁调用 `toID` 把字符串转为 `ID`，到了 `toID` 是 PS **最热函数**的程度。理论上 PS 可以用 TS 类型检查保证"仅在输入处调用 `toID`"，但即便那样也无法完全消除成本。

#### 全功能的数据层

PS 的数据层设计为**支持远超对战所需的各种用途**。

- ✅ 同一套数据可以服务于很多额外工具。
- ❌ 更通用的 API 带来**膨胀**，拖累性能。
- ❌ 类似地，许多核心类**为了便利而设计**而不是为了性能：例如不区分 `ActivePokemon` 和队伍里的 `Pokemon`，导致**冗余数据填满 cache line**。

#### 不关心 [monomorphism（单态化）](https://mrale.ph/blog/2015/01/11/whats-up-with-monomorphism.html)

PS 经常**以低效方式初始化关键数据对象**（基础类型常用的 **`Object.assign(this, data)`** 模式）。

> 💡 V8 之类的现代 JS 引擎给"字段总是按相同顺序初始化"的对象做了大量优化（"hidden class"/"shape"）。`Object.assign` 模式让顺序变得不可预测，从而失去这些优化。在字段只有几个时容易保证一致，但 PS 的核心对象有 50–100 个字段，几乎做不到。

#### Map 查 key

PS 的核心 API 大多是**按 `ID` 在 map 中查找**——本质上比直接数组索引慢。虽然两者都是 $\Theta(1)$，但 hash + 指针追逐导致 cache miss 和性能损失。

#### 总是产生文本日志

PS 在**所有情况下都产生文本协议日志**。对调试无价，但：

- ❌ 文本日志**产生和解析都昂贵**。
- ❌ 在很多用例（比如随机模拟）下**整个工作量都是被丢弃的**。

#### 用 JavaScript/TypeScript 写

- ❌ 在 JS 里**精确布局数据结构、使用最小字段**非常**不符合人体工学**。数字最少 4–8 字节，除非全部操作 `ArrayBuffer`（这在 JS 里非常难写）。
- ❌ 大量依赖**动态内存分配**——比"复用栈上对象"天然慢。
- ❌ **第三方开发者要用 PS 引擎**，要么自己也写 JS，要么嵌入 JS runtime（付出边界开销），要么走标准输入输出流（**syscall 开销**）。

#### 总结

PS 的设计选择带来了一个**灵活、易扩展的引擎**，但其架构**与达成峰值性能在根本上是冲突的**。`pkmn/engine` 选择了相反的取舍。

---

## 设计原则的实操检查清单

当你在 review 一个 `pkmn/engine` 的 PR 时，可以用下面这张表自检：

- [ ] 改动**没有为非性能特性牺牲性能**了吗？
- [ ] 有没有引入**字符串**作为内部标识？
- [ ] 有没有引入**动态分配**？
- [ ] 数据结构改动后，[`layout.json`](../../src/data/layout.json) 同步更新了吗？
- [ ] 字段顺序的改变，[做过 cache line 影响评估](http://www.catb.org/esr/structure-packing/) 了吗？
- [ ] 引入新的运行时分支，是不是应该改成**编译期 flag**（`-D...`）？
- [ ] `-Dshowdown` 与默认模式都测过了吗？
- [ ] 二进制协议如有变化，[`PROTOCOL_zh.md`](./PROTOCOL_zh.md) 也更新了吗？
