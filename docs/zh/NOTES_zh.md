# 开发笔记（中文）

> 原文：[`docs/NOTES.md`](../NOTES.md)
>
> 这份文档是给**引擎核心维护者**看的"操作手册"：新增一个世代要走完哪 23 步、升级 `@pkmn/sim` 依赖怎么处理、以及调试常见错误的具体技巧。中文版在原文基础上补充了"每一步为什么这样做、做错了会怎样"的解释。
>
> 如果你只是想用 `@pkmn/engine`，可以跳过本文。

---

## 新增一个世代（Adding a new Generation）

> 实现一个新的世代是个**几千行 Zig 代码、上百个测试用例**的工程。下面 23 步是社区在做 Gen 1 / Gen 2 时总结出来的"经过实战验证的顺序"，强烈建议按顺序来——其中很多步骤的输出是后一步的输入。

### 1. **[研究](./RESEARCH_zh.md)** 数据结构和代码流程

从对应世代的 pret 反编译开始读，重点是：

- 战斗主循环（`core.asm` 或 `battle_main.c`）
- 招式效果的实现方式（handler 表 / 字节码 / 脚本）
- 临时状态（volatile status）的位定义
- 该世代的 RNG（包括 Pokémon Showdown 是否搞错了它）

**不要跳过这一步**。后面所有的"为什么写成这样"都需要能溯源到反编译代码。

### 2. 加一个 `data.zig` 文件，定义**基础数据类型**

包括 `Battle`、`Side`、`Pokemon`、`ActivePokemon`、`Stats`、`Boosts`、`Volatiles` 等。

- **此时先不优化布局**，能编译通过就行；
- 精确的字段顺序、位打包等会在第 11 步集中调优。

### 3. **[生成](../../src/tools/generate.ts)数据**文件

```sh
npm run generate -- data <GEN>
```

- 重新排列 enum 顺序以提高性能（让常用值靠前，配合 jump table）
- 必要时更新 [`Lookup`](../../src/pkg/data.ts)

### 4. **[生成](../../src/tools/generate.ts)测试**文件

```sh
npm run generate -- tests <GEN>
```

- 然后按"和前一代相同顺序"重新整理（这样跨代对比 diff 才有意义）
- 把已知的 Pokémon Showdown bug 和卡带 glitch 加进去作为单独的测试用例

### 5. **复制共享代码/文件**

- 拷一份 `README.md` 模板用于新世代
- 把 `mechanics.zig` 里的 imports 和公共函数骨架（`update`, `choices` 等）抄过来
- 把 `test.zig` 里的 `Test` 基础设施和"rolls 写法"抄过来
- 拷一份 `helpers.zig`

### 6. 针对 Pokémon Showdown 行为实现**单元 [tests](../../src/test/showdown/)**

> 这是顺序非常关键的一步。**先把 Showdown 的行为锁住**（用 fixed RNG 的测试一一对照），再去实现 Zig 端的 mechanics。

- 实现过程中如果发现 Showdown 的 bug，把它记到该世代文档的 Bugs 章节
- Showdown 的测试只测 Showdown 本身，不测引擎代码

### 7. 基于卡带研究在 `mechanics.zig` 中实现**机制**

- 协议如有变化（新的消息类型、新的 reason 等），同步更新：
  - [`protocol.zig`](../../src/lib/common/protocol.zig)
  - [`PROTOCOL.md`](../PROTOCOL.md) / [中文版](./PROTOCOL_zh.md)
  - 驱动端 [`protocol.ts`](../../src/pkg/protocol.ts)
  - 协议相关的测试
- 重新 [生成](../../src/tools/dump.zig) [`protocol.json`](../../src/data/protocol.json)

### 8. 调整 **mechanics 以兼容 Pokémon Showdown**

- 跟踪 RNG 差异，更新世代文档（所有 RNG 集中在 `Rolls` 子结构里）
- 把不可实现的副作用加入 blocklist（一般在测试里通过跳过来体现）
- 文档里要逐一列出所有已知 bug

### 9. **在两种模式下都跑过单元测试**

```sh
zig build test                 # 卡带模式
zig build test -Dshowdown=true # Showdown 兼容模式
```

### 10. 实现 **`MAX_LOGS` 单元测试**

- 在 [`PROTOCOL_zh.md`](./PROTOCOL_zh.md#size--日志大小) 中写明推导过程
- 用 Z3 形式化验证：单回合最坏情况下日志真的不会超过这个上界

### 11. **优化数据结构布局**

- 重排字段、用位打包压缩、把热点字段放进 cache line 的前半
- 重新 [生成](../../src/tools/dump.zig) [`layout.json`](../../src/data/layout.json) 与 [`data.json`](../../src/data/data.json)
- **关键不变式：所有测试在优化前后必须输出一致**

### 12. 实现**驱动端的序列化/反序列化**

JS 驱动需要知道字段在 buffer 里的偏移。基于 `layout.json` 生成 `Battle` 类的 getter/setter，并写测试覆盖往返（Zig → bytes → JS → bytes → Zig）。

### 13. **暴露 API**

把新世代加进所有"绑定层"：

- [`pkmn.zig`](../../src/lib/pkmn.zig)
- C 绑定：[`c.zig`](../../src/lib/c.zig)
- Node 绑定：[`node.zig`](../../src/lib/node.zig)
- WASM 绑定：[`wasm.zig`](../../src/lib/wasm.zig)
- C 头文件 [`pkmn.h`](../../src/include/pkmn.h)
- TS 入口 [`index.ts`](../../src/pkg/index.ts)

### 14. 写 **`helpers.zig`** 和 **`choices`** 方法

- `choices` 用于生成"当前状态下所有合法选择"。这是 AI / 集成测试的基础设施。
- 测试中需要匹配的 `Choices` 代码在 [showdown](../../src/test/showdown/index.ts) 里。

### 15. 确保 **[fuzz 测试](../../src/test/benchmark.zig)** 通过

- 更新 [`fuzz.ts`](../../src/test/fuzz.ts) 和 [`debug.ts`](../../src/tools/debug.ts)
- fuzz 会跑大量随机种子，能暴露很多 RNG 顺序 / 边界条件的问题

### 16. 确保 **[集成测试](../../src/test/integration.ts)** 通过

```sh
npm run test:integration
```

### 17. 加 **`chance.zig`** 和 **`calc.zig`** 文件，定义数据类型

- `chance` 用于追踪本回合每个 RNG 事件的概率（让用户能 `expectProbability`）。
- `calc` 用于"伤害计算覆写"，让 AI 在做 MCTS 等场景时能跳过具体伤害掷骰，直接用上下界。

### 18. **在 mechanics 中插入 `Chance` 与 `Calc` 调用**

每个 RNG 调用点都要：

- 通知 `Chance`："我刚刚做了一个 X 类型的掷骰，结果是 r，概率分支是 …"
- 给 `Calc` 一次"是否要覆写本次结果"的机会

### 19. 更新单元测试用 **`expectProbability`**

- 确保 chance/calc 的覆写能 roundtrip（覆写后回放 → 结果完全一致）

### 20. 实现 **`transitions` 函数**

- `transitions(battle, p1, p2)` 列出某个状态在给定选择下能转移到的**所有概率分支**
- 给新世代添加 `Rolls` helper
- 把 `transitions` 调用塞进 fuzz 测试
- 测出 `MAX_FRONTIER_SIZE`（"前沿队列"的最坏尺寸），加进 API 常量

### 21. **JS 驱动 `calc` 与 `chance` 的支持**

- 更新 [`layout.json`](../../src/data/layout.json) 把额外的 offset 也加上

### 22. **[基准测试](../../src/test/benchmark.zig)** 新世代

按 [`TESTING_zh.md`](./TESTING_zh.md#benchmark--基准测试) 的环境设置运行 benchmark，把数字记到该世代的 README。

### 23. **整理该世代文档**

包括：

- 一份"该世代的 bug 与 glitch 列表"
- 一份"该世代与卡带 / Showdown 的差异"
- 一份"该世代 RNG 的精确语义"

---

## 升级 `@pkmn/sim` 依赖

> Pokémon Showdown 的代码在不断演进，pkmn/engine 会通过 `@pkmn/sim` 镜像它。每次升级都可能改变 Showdown 的行为，进而影响引擎在 `-Dshowdown` 模式下的测试预期。

1. **更新版本号**：改 [`package.json`](../../package.json) 里 `@pkmn/sim` 的 pin，跑 `npm install`。
2. **运行集成测试**，根据新行为更新 [`src/test/showdown`](../../src/test/showdown) 下的测试 rolls 和断言：
   ```sh
   npm run test:integration
   ```
3. **更新 Zig 端的 mechanics 测试**，使其与集成测试保持一致。
4. **更新引擎实现**，让 mechanics 测试通过。
5. **更新文档**：哪些 bug 被修了？哪些行为变了？这些都要在世代文档里反映。
6. 如果某些副作用在新版 Showdown 中已经可以正确处理，**从 blocklist / helper 中移除**对它们的跳过逻辑。

---

## 调试测试

### 回归测试（Regression Tests）

回归测试用 fixture 形式保存：一个失败种子 + 一段 dump。当你定位某个失败时，希望能**重新生成 HTML 调试 UI** 来逐帧查看。

打开 [`integration.ts`](../../src/test/integration.ts) 的 `play` 函数末尾的 `catch` 块，**注释掉 `if (!replay)` 包裹**——这样即使在回放模式下也会写 `logs/pkmn.html` 和 `logs/showdown.html`：

```diff
-if (!replay) {
 const num = toBigInt(seed);
 const stack = err.stack.replace(ANSI, '');
 errors?.seeds.push(num);
 errors?.stacks.push(stack);
 try {
   console.error('');
   dump(
     gen,
     stack,
     num,
     rawInputLog,
     frames,
     partial,
   );
 } catch (e) {
   console.error(e);
 }
-}
```

跑完后用浏览器打开 `logs/pkmn.html`，即可可视化逐帧对比 pkmn 引擎与 Showdown 的状态。

### 具体错误类型

#### "Unexpected shuffle"（非预期的乱序）

这个错误一般是 **Showdown 在 `runEvent` 中触发了不该触发的 speed sort / speed tie**，造成 RNG 帧提前。要先弄清楚是哪个事件名导致的：

在 [`showdown.ts`](../../src/test/showdown.ts) 的 `patch.battle` 里加日志：

```diff
 battle: (battle: Battle, prng = false, debug = false) => {
+   const run = battle.runEvent.bind(battle);
+   battle.runEvent = (...args) => {
+     console.debug(args[0]);
+     return run(...args);
+   };
   battle.trunc = battle.dex.trunc.bind(battle.dex);
```

跑一次失败的 seed，控制台就会打印所有 `runEvent` 调用的事件名。当你定位到导致 speed tie 的那个事件后，**在 `patch.generation` 里给对应的 handler 显式分配优先级**，强制顺序确定下来。

#### "Mismatched seeds"（种子不一致）

意思是 Zig 引擎和 Showdown 在某个点对 RNG 状态的推进不一致。最直接的办法是**在两边都打印 RNG 状态**，看分歧在哪一帧出现。

**在 Zig 端**，修改 [`rng.zig`](../../src/lib/common/rng.zig) 的 `Gen56` RNG `advance`：

```patch
 pub fn advance(self: *Gen56) void {
+    DEBUG(self.seed);
     self.seed = 0x5D588B656C078965 *% self.seed +% 0x0000000000269EC3;
 }
```

> ⚠️ **小心字节序**：Zig 这里用的是 little-endian，而 Pokémon Showdown 把同一个 64 位整数按 big-endian 显示，**所以打印出来的数字字面看着不一样，但其实是同一个值**。比对前要么把两边都转成同一种字节序、要么手动 reverse。

**如果你不需要 seed 值、只需要知道在哪一处 advance**，可以在 mechanics 里给每个 `Rolls` 函数打 `@src()`：

```diff
 pub const Rolls = struct {
     fn speedTie(battle: anytype, options: anytype) !bool {
+      DEBUG(@src());
```

这样会打印调用的 Zig 源文件和行号，配合 Showdown 端的事件日志能很快锁定不一致的位置。

---

## 速查表

| 我想做的事 | 命令 |
| --- | --- |
| 跑 Zig 单元测试（卡带模式） | `zig build test` |
| 跑 Zig 单元测试（Showdown 兼容模式） | `zig build test -Dshowdown=true` |
| 跑集成测试 | `npm run test:integration` |
| 跑 fuzz | `npm run --silent fuzz -- <pkmn\|showdown> <GEN> <DURATION> <SEED?>` |
| 跑 benchmark | `npm run benchmark -- --battles=10000` |
| 生成数据文件 | `npm run generate -- data <GEN>` |
| 生成测试桩 | `npm run generate -- tests <GEN>` |
| 重新 dump layout/data | 见 [`src/tools/dump.zig`](../../src/tools/dump.zig) |

更多测试细节见 [`TESTING_zh.md`](./TESTING_zh.md)。
