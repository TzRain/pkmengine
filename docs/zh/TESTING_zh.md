# 测试体系（中文）

> 原文：[`docs/TESTING.md`](../TESTING.md)
>
> 这份文档讲清楚了 `pkmn/engine` 的**四层测试体系**：单元测试（unit）、集成测试（integration）、基准测试（benchmark）、模糊测试（fuzz）。中文版在原文基础上**补充了"每种测试针对哪类 bug、为什么必须存在、怎么本地跑"**，便于贡献者把握测试该写到哪一层。

测试代码主要在两处：

- 各模块内联的 unit 测试（与被测代码同文件）
- 顶层 [`src/test`](../../src/test) 目录下的 harness（integration / benchmark / fuzz）

---

## Unit / 单元测试

由于多种构建选项（如 `-Dshowdown` 或 `-Dlog`）和宝可梦本身的**随机性**，pkmn 的单元测试比一般项目多一点样板。所以引擎提供了若干 helper 来消除大部分模板：

### `Test`：单元测试的主助手

一个测试通常写成：

```zig
var t = Test(rolls).init(p1, p2);
defer t.deinit();   // 释放资源，必须紧跟 init 之后 defer

// 在 t.expected 上追加期望的 update / 日志消息

try t.verify();     // 比较 t.actual 与 t.expected
```

- `t.expected` 用来声明"我期望这次 update 产生哪些日志消息、哪些状态变更"。
- `t.actual` 是引擎实际跑出来的。
- 结尾 `t.verify()` 一次性核对。
- 启用 `-Dchance` 时还可以 `expectProbability(...)` 检查特定分支概率。
- 同时启用 `-Dcalc` 时，每一次 update 都会**用原始的 chance actions 作为覆写在原始状态上重新跑一遍**，确保所有 RNG 都被正确"账户化"。

### `Battle.fixed`：固定 RNG 战斗

底层上，`Test` 用的就是 `Battle.fixed` 来创建带 **`FixedRNG`** 的战斗。`FixedRNG` 会**返回预定义好的一串结果（"rolls"）**——这给了你对"事件是否发生"的完全控制。

但有个麻烦：**`-Dshowdown` 模式下需要的 rolls 数量和顺序与卡带模式不同**（因为 PS 的 RNG 调用次数和顺序与卡带不同）。因此 fixed-rolls 测试必须**同时声明两套 rolls**。

最后**很重要**：测试结束时要校验**所有提供的 rolls 都用上了**——

```zig
try expect(battle.rng.exhausted());
```

`Test.verify()` 会自动做这个检查。**意外没用上的 roll 往往意味着 bug**——比如某个本该触发的事件被跳过了。

---

## Patches / 补丁

> 这是整个测试体系最微妙的部分。建议读完再读 [`DESIGN_zh.md`](./DESIGN_zh.md) 里"`-Dshowdown`"那一节。

引擎想在 `-Dshowdown` 模式下匹配 PS 的行为，**但完全复刻 PS 的行为意味着也要复刻它有问题的架构**——特别是其事件 / handler / action 系统**会人为引入大量 speed tie**（速度并列），从而**额外推进 RNG**。

把这种"为了 bug-for-bug 兼容而搬一整套 byzantine 逻辑"的事情**算作引擎的范畴是不合算的**：引擎匹配 PS 只是为了**实用目的**——集成测试方便、给基于 PS 的 AI 训练提供更"准确"的回合数据。

**解决方案**：让引擎匹配一个 **被打补丁的** PS 版本。这些补丁**做最小化的改动**，让 PS 的行为更接近卡带"host ordering"（先 P1 后 P2）语义，**消除不必要的非确定性**：

- `Battle#eachEvent` 和 `Battle#fieldEvent` 在 Gen 1 & 2 中**不再 `speedSort`**——事件按添加顺序执行，等价于"P1 事件先于 P2 事件无论速度"，复现卡带的默认 host ordering。
- `BattleQueue#insertChoice` 也被 patch 成 Gen 1 & 2 中遵守 host ordering。
- 在各种 handler 上**加优先级**来打破 speed tie，让事件**要么没有多余 roll、要么按卡带顺序确定地结算**。

> ⚠️ 这些补丁**不修复**真正的 PS 实现 bug（这超出范围），**也不修复**所有 RNG 帧因 speed tie 多推进的问题（比如带 `beforeTurnCallback` 的招式仍可能触发 speed tie roll）。
>
> 这些补丁严格意义上**还会让 PS 变得更快**（少做了 sorting 和 RNG advances），所以也对得起被引擎对比的角色——这是 ["steelmanning"](https://en.wikipedia.org/wiki/Straw_man#Steelmanning)（"用最强版本来比较"）。

### `showdown/` 测试

为了**验证 PS 自身的行为**，引擎的许多单元测试在 [`showdown/`](../../src/test/showdown/) 目录下有**镜像**。

⚠️ **这些是测**被补丁后的 PS**，不是测 pkmn 引擎**。引擎自己的代码并没有在这些测试里被测试。

为什么不能复用 PS 自己的单元测试？

- PS 自己的测试**大多覆盖最新世代**
- **不用固定 RNG**
- **不验证日志**（这两点对匹配 PS 的 RNG 与输出至关重要）

### 编写新单元测试的指南

下面 13 条来自原文（中文版整理为更便于扫读的列表）：

1. **先用工具生成桩**：
   ```sh
   npm run generate -- tests <GEN>
   ```
   生成的桩需要不少手工调整，但能省去打字时间。
2. **保持顺序**：测试用例的顺序尽量与前代相同，新效果应与同类效果分组。**通用战斗流程测试**也要保留。
3. **覆盖效果的全部行为**：Bulbapedia 与 Smogon 上描述的所有行为都要测（但**不要假定这些来源百分百正确**）。**所有已报告的 PS bug 和卡带 glitch 也要有测试**。
4. **能抄就抄**：从前代测试用例尽量复制——这样跨代 diff 才能反映"行为变化"而不是"风格变化"。如果行为大改，再考虑全新写一个。
5. **覆盖面广**：理想情况下每个种族 / 道具 / 招式 / 特性都至少出现一次。
   - 选**"自然"配招**：相近 tier 的种族、招式池里实际有的招式
   - ["招牌招式"](https://bulbapedia.bulbagarden.net/wiki/Signature_move) 与[招牌特性](https://bulbapedia.bulbagarden.net/wiki/Signature_Ability)放在正确的进化线上
6. **复现 bug 时贴近原情景**：如果在测某个 PS bug 或某个 glitch 视频，**尽量复刻原始复现设置**。
7. **用"默认"数值**：Gen 1 & 2 满 Stat Experience、Gen 3+ 没有 EV、等级 100。
8. **倾向于"长测试 + 完整覆盖"**：和大多数测试最佳实践相反——pkmn 偏好用**一个长测试用例覆盖一个效果的大部分行为**，因为**减少 setup 成本**比"测试隔离"更重要。
9. **复杂效果只测核心**：像 Substitute、Baton Pass 这种与众多其它效果交互的效果，**在它自己的测试里只测最基础的行为**，与其它效果的交互则放进**那些效果各自的测试**里。
10. **避免多余 rolls 和日志**：用 Splash 这种"什么都不做"的招式作为占位；除非专门测 speed tie 否则**避免双方同速**。
11. **避免在测试中间人为改战斗状态**：要测异常状态，就**通过测试本身让宝可梦中状态**（PS 的 cleric clause 不允许带状态进入战斗）；HP/PP 应在测试开始**之前**就调好。
12. **少用子测试**：只有当子测试的 setup 与主测试**显著不同**时才用。
13. **单 vs 双打**：如果某效果在单打和双打差异巨大，**两边都测**；否则在整体测试集中混搭单打/双打即可。

---

## Integration / 集成测试

[集成测试](../../src/test/integration.test.ts)的目的是确保 **在 `-Dshowdown` 模式下编译的 pkmn 引擎** 与 **被补丁的 PS** 输出**可比较**。

机制：

- 对每个支持的世代，**同时跑** PS 和 pkmn 引擎；
- 跑的是 [`ExhaustiveRunner`](https://github.com/smogon/pokemon-showdown/blob/master/sim/tools/exhaustive-runner.ts)，它会**穷举尽可能多的效果**；
- 收集双方结果对比。

⚠️ 但 PS 永远会产生文本协议流，而 **pkmn 必须特别用 `-Dlog` 编译才会有可观测的输出**。

### 协议**不**会逐字节相等

pkmn 的[二进制协议](./PROTOCOL_zh.md)**预期不会与 PS 的文本协议完全等价**，原因如下：

- pkmn **没有 "format" 或 "custom rules" 概念**（[PS formats.ts](https://github.com/smogon/pokemon-showdown/blob/master/config/formats.ts)、[Custom Rules](https://github.com/smogon/pokemon-showdown/blob/master/config/CUSTOM-RULES.md)）
- PS 的关键字参数（kwArgs）**顺序未严格定义**
- PS 的若干协议消息是**冗余或实现细节相关**
- pkmn **永远只产生一个流**（PS 称之为 "omniscient stream"），**始终包含精确 HP**；按 side 过滤的其它流应由驱动代码计算
- [虽然 PS 自称如此](https://pokemonshowdown.com/pages/rng)，但它**没有为每个世代实现正确的 PRNG**——见前文。

集成测试包含把 PS 的输出"按摩"成可与 pkmn 输出比较的形式的逻辑。**当二者意见不一时，以卡带反编译为准**。但仍可能两者**独立都错了**——这种情况虽然小概率但理论上存在[^1]。

集成测试失败时**通常会催生一个新的单元测试**；同时失败的日志会被存为 [fixture](../../src/test/regression/fixtures)，进入[回归测试](../../src/test/regression/)防止再次回退。

集成测试还支持**独立模式 + 时长参数**，可用于 [fuzz](#fuzz--模糊测试)：

```sh
npm run integration -- --duration=15m
```

[^1]: 一个延伸目标是**直接用实际卡带的代码跑集成测试**。[已有示例](https://github.com/jsettlem/elo_world_pokemon_red/blob/master/battle_x_as_y.py)在模拟器里跑脚本对战，但要让"link battling + 检测 desync"都对得上仍然是非常 nontrivial 的工程。

### Unimplementable / 不可实现的子集

PS 的某些 bug **过于诡异**，即使打了上面的补丁也**无法在 pkmn 中等价复现**。引擎尽力复现 PS 中最被误解、最坏的机制，但反过来——**从一个对齐卡带的架构出发去复现 PS 的 bug**——也很难。

具体处理：

- **基准测试**：可以选择**根本不生成包含问题种族 / 道具 / 特性 / 招式的队伍**。
- **集成测试**：还是要加一些复杂度尽量多覆盖一些；队伍在开战前先校验（防止把无法在一起使用的招式组进同一队），运行中如果 PS 进入了不期望的状态就**中止本场**并切到下一场。

---

## Benchmark / 基准测试

> 单独运行一个 [`hyperfine`](https://github.com/sharkdp/hyperfine) 并**不够**——V8 有 runtime 开销和 warmup 周期需要排除掉（`hyperfine --warmup` 是为了 disk 缓存预热的，不是 JIT 预热）。

所以 pkmn 有[自己的 benchmark 工具](../../src/test/benchmark.ts)。它测的是 **N 场随机生成的对战跑到结束的总耗时**，**排除掉**：

- 初始化 setup 时间
- JS 的 warmup 时间

这个场景**近似于 [MCTS](https://en.wikipedia.org/wiki/Monte_Carlo_tree_search) 用例**——每个回合都要跑大量 playout 到终局以决定最优行动。

### 不测什么

- **不测 PS 的 `BattleStream`** 抽象：虽然不难用，但 PS 内部对 Promise 处理不好，**很容易遇到 race 把基准搞 desync**。
- **不测 `pokemon-showdown` 二进制**：本质上 `BattleStream` 等价但**没有 I/O 开销**，而二进制版无法 inspect `Battle` 来避免做出 unavailable choice。

### 三种配置

1. **`DirectBattle`**：基于 PS 的 `Battle` 类**剥掉所有不必要的功能**：
   1. 任何往 battle log 写消息的方法 → 立即丢弃
   2. `sendUpdates` → 空实现
   3. `makeRequest` → 不为每方序列化 request

   并且**同步使用**（不走异步的 `BattleStream`），快约 10%，且无需处理 race。
   这是 PS **能跑多快的接近上界**（还能再优化但收益递减）。
   这个配置最接近"pkmn 引擎**不开 `-Dlog`**"的对照。
   `DirectBattle` 还应用上面提到的 **[补丁](#patches--补丁)** 消除非必要工作。

2. **`@pkmn/engine`**：通过 `@pkmn/engine` 这个 JS 驱动包跑 pkmn 引擎——**经过 FFI 一层**。

3. **`libpkmn`**：**直接用 `libpkmn` 库跑**，完全不经过 JS。基准 runner 调 [`benchmark.zig`](../../src/test/benchmark.zig) 直跑并报告结果。

两个 pkmn 配置都**只开 `-Dshowdown`**，**其它 flag 都关**（特别是 `-Dlog`）。两个 PS 配置都先跑一段 warmup 以保证测出的是稳态时长。

### Checksum / 校验

要让三种配置确实测的"同一件事"，必须保证：

- **完全相同的对战被生成**
- **完全相同的选择被做出**
- **完全相同的结果**

所以所有基准都用**相同种子初始化的相同 PRNG**，"生成对战 / 选择招式"的逻辑**在 Zig 和 TS 两端复制**。除了总耗时，工具还跟踪**所有对战的总回合数**与**最终 RNG seed**作为 checksum——验证三种配置实际是一致的。

PS 还有一些特殊要求：

- 必须**先序列化队伍**再传给 `Battle` 构造器（PS 会 mutate 队伍）
- 必须给两边玩家**各自一个独立 PRNG**，且与 `Battle` 自己的 PRNG 也分开（PS 内部有大量 race 与[不愉快](https://github.com/smogon/pokemon-showdown/issues/8546)）

### 关于"对战长度"的提醒

一场战斗的耗时**强依赖于队伍**。pkmn 的 benchmark 用[Challenge Cup](https://bulbapedia.bulbagarden.net/wiki/Challenge_Cup) 语义生成队伍——里面**包含大量次优招式**（比如同时有 Thunder Shock 和 Thunderbolt 而不是只有后者），因此**预期比 [Random Battle](https://github.com/pkmn/randbats) 或手搓队伍慢**。

> 经验上 benchmark 用的随机队伍**比实战典型队伍慢 2–3 倍**。

[^2]: 让"能 inspect `Battle`"和"不能 inspect `Battle`"的配置保持同步的方法是：**每次都保存上一次 RNG 调用的原始结果**，下次遇到 "Unavailable choice" 时**复用同一个 `r` 重选**（不重新调用 RNG，只是把 `r % N` 换成 `r % M`）。

### Results / 结果

下表来自 [pkmn/engine@9ce6e379](https://github.com/pkmn/engine/commit/9ce6e379)，跑在 GCP `n2d-standard-48`（192 GB 内存、AMD EPYC 7B12 CPU、64-bit x86 Linux），经过下文 tuning 后用 `npm run benchmark -- --battles=10000`：

| 世代 | `libpkmn` | `@pkmn/engine` | `DirectBattle` |
| --- | --- | --- | --- |
| **RBY** | 195 ms | 737 ms（3.78×） | 618 s（**3167×**） |

> 不同机器下相对差距会有所不同，但**量级稳定**。

<details><summary>CPU 详情</summary>
（与原文相同，略，参见 <a href="../TESTING.md">原文</a>）
</details>

<details><summary>环境准备</summary>

在 GCP 上拉一台 [spot](https://cloud.google.com/compute/docs/instances/spot) 机器，跑完自动销毁：

```sh
gcloud beta compute instances create pkmn-engine-benchmark \
	--zone=us-central1-a \
	--machine-type=n2d-standard-48 \
	--image-project=ubuntu-os-cloud \
	--image-family=ubuntu-minimal-2204-lts \
	--max-run-duration=30m \
	--provisioning-model=SPOT \
	--instance-termination-action=DELETE

sleep 15
gcloud compute ssh pkmn-engine-benchmark
```

机器上：

```sh
sudo apt update
sudo apt --assume-yes install git cpuset
git clone --depth 1 https://github.com/pkmn/engine.git
cd engine
curl -fsSL https://raw.githubusercontent.com/tj/n/master/bin/n | sudo bash -s lts
npm install
export PATH="$(pwd)/build/bin/zig:$PATH"
```

最后以 root 跑：

```sh
sudo --preserve-env=PATH env ./benchmark.sh
```

</details>

<details><summary>Tuning / 调优脚本</summary>

```sh
#!/bin/bash

function cleanup() {
    # 恢复 hyperthreading
    for cpu in {1..47}; do
      echo 1 > /sys/devices/system/cpu/cpu$cpu/online
    done
    # 解除 CPU shield
    cset shield --reset >/dev/null 2>&1
}
trap cleanup EXIT

# 基于 thread_siblings 关掉 hyperthreading
for cpu in {24..47}; do
  echo 0 > /sys/devices/system/cpu/cpu$cpu/online
done

# 在 GCP 上无法关掉 CPU boosting 或切到 performance governor，跳过

# 给 CPU 1-9 建一个 shield 并把所有线程（包括内核线程）赶出去
cset shield -c 1-9 -k on >/dev/null 2>&1

# Drop 文件系统缓存
echo 3 > /proc/sys/vm/drop_caches
sync

# 在 shield 内以最高优先级跑 benchmark
cset shield --exec -- nice -n -19 node build/test/benchmark
```

</details>

### Regression / 回归基准

除了对比 PS，benchmark 工具还有**专门用于检测引擎自身性能回归**的模式。`--iterations` 标志会让它**用同一个 seed 跑多轮**并输出 TSV：

```sh
npm --silent run benchmark -- --iterations=50 > logs/before.tsv
# <做改动>
npm --silent run benchmark -- --iterations=50 logs/before.tsv
```

第二次调用传入第一次的 TSV，工具会算"前后差异"。可选 `--summary` 输出文本或 JSON 概览——为了**降低噪声**，会**剔除离群值后取均值**。

---

## Fuzz / 模糊测试

[集成测试](#integration--集成测试) 和独立的 [fuzz](../../src/test/fuzz.zig) 都用于 [fuzz](https://en.wikipedia.org/wiki/Fuzzing)。

- 有一个 [GitHub workflow](../../.github/workflows/fuzz.yml) **定时**从随机种子跑这些测试不同时长，挖掘潜在 bug。
- Fuzz 与 benchmark 的区别：fuzz **按时间长度跑**而不是按"跑几场"，并且**开启那些在 `-Dshowdown` 兼容模式下通常被跳过**的 [不可实现](#unimplementable--不可实现的子集) 效果。
- 当加 `-Dlog` 时，**崩溃时会 dump 额外二进制数据**，便于用 [`fuzz.ts`](../../src/test/fuzz.ts) 和[调试 UI](https://pkmn.cc/debug.html)（由 [`debug.ts`](../../src/tools/debug.ts) 渲染）联合调试。
- 同时开启 `-Dchance` 与 `-Dcalc` 时，fuzz **会同时验证 `transitions` 函数能正确枚举所有合法状态转移**而不崩。

### 本地跑 fuzz

```sh
npm run --silent fuzz -- <pkmn|showdown> <GEN> <DURATION> <SEED?>
```

例：

```sh
# Gen 1 在 pkmn 引擎模式下跑 5 分钟，随机种子
npm run --silent fuzz -- pkmn 1 5m

# Gen 2 在 showdown 模式下跑 1 小时，指定种子（便于复现）
npm run --silent fuzz -- showdown 2 1h 0x12345678
```

---

## 速查：本地跑测试的命令

| 任务 | 命令 |
| --- | --- |
| Zig 单元测试（卡带模式） | `zig build test` |
| Zig 单元测试（Showdown 模式） | `zig build test -Dshowdown=true` |
| 启用日志的 Zig 测试 | `zig build test -Dshowdown=true -Dlog=true` |
| 启用 chance + calc | `zig build test -Dshowdown=true -Dchance=true -Dcalc=true` |
| JS 端测试 | `npm test` |
| 集成测试 | `npm run test:integration` |
| 集成测试（连续 15 分钟） | `npm run integration -- --duration=15m` |
| Benchmark | `npm run benchmark -- --battles=10000` |
| Benchmark（回归模式） | `npm run benchmark -- --iterations=50` |
| Fuzz | `npm run --silent fuzz -- <pkmn\|showdown> <GEN> <DURATION>` |

---

## 该把测试写到哪一层？决策表

| 我在测的东西 | 推荐层级 | 文件 |
| --- | --- | --- |
| 单个招式 / 效果的具体行为 | **Unit** | `src/lib/genX/test.zig`（pkmn 端） + `src/test/showdown/genX.test.ts`（PS 端镜像） |
| 跨多招式 / 整局对战的行为 | **Unit**（一个长测试用例） | 同上 |
| 引擎 vs PS 的"是否一致" | **Integration** | `src/test/integration.test.ts` |
| 性能是否退化 | **Benchmark（回归模式）** | `npm run benchmark -- --iterations=50` |
| 想挖未知 bug | **Fuzz** | `src/test/fuzz.zig` |
| 已经发现的失败 seed 不再回退 | **Regression fixture** | `src/test/regression/fixtures` |
