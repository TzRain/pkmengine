# 二进制协议（中文）

> 原文：[`docs/PROTOCOL.md`](../PROTOCOL.md)
>
> 这份文档是 pkmn/engine **二进制日志协议**的**完整参考**。中文版在原文基础上**补充了**：
>
> - 协议的"为什么"与设计取舍
> - 解析协议的**伪代码示例**
> - `PokemonIdent` 位打包的逐位拆解
> - `MAX_LOGS` 上界推导的背景解释
>
> 字节布局表格（每条消息的"Byte/Bit"图）保持英文/原值不变——它们是 binary 协议的事实定义，不应翻译。

---

## 总览

在最高层，pkmn 引擎根据双方的 [choices（选择）](#choice--选择) **更新**战斗状态，返回一个 [result（结果）](#result--结果)，告诉调用者**战斗是否已结束**以及**双方下一步各自的选项类型**。

如果 result 不是终态，可以把它传给世代特定的 `choices()` 方法，得到**所有合法行动**[^1]——当然你也可以**直接 inspect 战斗内存**来判断哪些招式可选。各世代的数据结构不同，[设计](./DESIGN_zh.md)上是有意如此；具体每代字段如何布局请见对应世代的 README。

如果想要更多关于战斗过程的信息，在编译时加 **`-Dlog`**。这让引擎把**本文档定义的二进制 wire 协议**写到一个 `Log`。

通常 `Log` 应该是一个由静态分配定长 buffer 支撑的 [`FixedBufferStream`](https://ziglang.org/documentation/master/std/#root;io.FixedBufferStream) [`Writer`](https://ziglang.org/documentation/master/std/#root;io.Writer)——**单次 `update` 写入的最大字节数是有界的**（[见下文 `Size`](#size--日志大小)）。每次 `update` 后 `reset` 该 buffer 即可。当然任意 `Writer` 都可以用，比如直接写到 stdout（**注意 Zig 的 stdout writer 默认不带 buffer**，请用 [`BufferedWriter`](https://zig.news/kristoff/how-to-add-buffering-to-a-writer-reader-in-zig-7jd) 包一层以获得合理性能）。

引擎的 wire 协议本质上是 [Pokémon Showdown 模拟器协议](https://github.com/smogon/pokemon-showdown/blob/master/sim/SIM-PROTOCOL.md) 的**精简二进制翻译**，去掉了若干冗余消息（如 `|upkeep|` 或 `|`），并对若干消息做了微调（没有 `|split|`——pkmn 永远只产生"全知流（omniscient）"，其它视角由驱动代码计算）。

> 像引擎其它部分一样，协议使用**本机字节序**（native-endianness）。每个世代的协议略有差异（如 Gen I/II 之后 `Move` 不再只占 1 字节），差异在对应章节会显式标出。

[^1]: `choices()` 方法在某些情况下会**泄露信息**——比如某只宝可梦已经被对手 Imprison 锁住某招式、或者已经被困住不能换人——这些"用户尝试后才知道的不合法 choice"对**机器随机对局**是问题不大的，但**对人类对战的 UI** 而言就需要你**自己实现一个不泄露信息的 `choices`**。

### Debugging / 调试

[`pkmn-debug`](../../README.md#pkmn-debug) 工具可解码二进制战斗 + 日志数据并渲染 HTML，对应 [人类可读调试 UI](https://pkmn.cc/debug.html)。工具要求由 `-Dlog` 启用的二进制提供以下结构：

#### Header / 头

调试日志必须以 header 开头，包含**是否启用 Showdown 兼容模式**、[世代号](https://bulbapedia.bulbagarden.net/wiki/Generation)、以及**初始战斗状态**：

| Start | End | Description |
| ----- | --- | ----------- |
| 0     | 1   | 是否启用 Pokémon Showdown 兼容模式（1 字节） |
| 1     | 2   | 宝可梦世代号（1 字节） |
| 2     | 4   | 2 字节有符号 native-endian 整数 $N$，每个 log buffer 的固定大小；**负数表示变长** |
| 4     | 8   | 4 字节有符号 native-endian 整数 $X$，额外数据 buffer 的固定大小；**负数表示变长** |
| 8     | B+8 | $B$ 字节——初始战斗状态的序列化（按对应世代 layout） |

#### Frame / 帧

Header 之后可以有任意多"frame"，最后一个可能只写了一半：

| Start   | End     | Description |
| ------- | ------- | ----------- |
| 0       | N       | $N$ 字节的协议日志，**以 `0x00` 或 EOF 终止**（若 $N < 0$） |
| N       | N+X     | $X$ 字节的可选额外数据，或变长以 `0x00` / EOF 终止（若 $X < 0$） |
| N+X+1   | N+X+B+1 | $B$ 字节——更新后的战斗状态（按对应世代 layout） |
| N+X+B+2 | N+X+B+3 | 本次更新的 [result](#result--结果) |
| N+X+B+3 | N+X+B+4 | P1 的下一个 [choice](#choice--选择) |
| N+X+B+4 | N+X+B+5 | P2 的下一个 [choice](#choice--选择) |

> ⚠️ **约定**：debug log **从首次 update 之后开始**（即双方 `|switch|` 出第一只宝可梦之后）——**初始的"无宝可梦在场"状态和首个"双方都 pass"的 choice/result 不会被记录**[^2]。

[^2]: 在带 [Team Preview](https://bulbapedia.bulbagarden.net/wiki/Appendix:Metagame_terminology#Team_Preview) 的世代中，这个约定可能会变（可能会用一个 `0x00` dummy 字节作为 pre-battle 日志），不过目前 pkmn 还没实现那些世代，所以这类讨论是推测性的。

---

## Choice / 选择

`choices()` 返回的合法选项分三类：`pass`、`move`、`switch`。和 [Pokémon Showdown 自己的 SIM-PROTOCOL](https://github.com/smogon/pokemon-showdown/blob/master/sim/SIM-PROTOCOL.md#possible-choices) 中的同名 choice 命令对应。

| Raw    | Type     | Data? |
| ------ | -------- | ----- |
| `0x00` | `pass`   | 无 |
| `0x01` | `move`   | 0–4   |
| `0x02` | `switch` | 2–6   |

- `pass`：**仅在"只有对方需要做决定"时出现**。例如自己在场的宝可梦倒下、或对手用了 Baton Pass 之后，自己要等对方先选要换上来谁。
- `switch`：数据值是一个**基于 1 的队伍槽位号**（必须 > 1，因为不能"切换上当前在场的宝可梦"）。
- `move`：数据值是一个**基于 1 的招式槽位号**。**某些情况下卡带不让你选招式**（如 Gen 1 的 Wrap、Bide 期间），此时数据值应为 `0`[^3]。

判断"当前哪些 choice 合法"很微妙，应**让引擎来算**——不在 `choices()` 返回数组里的选项**就是不合法的，强行传入可能让战斗状态损坏甚至崩溃**。

[^3]: 在 **Showdown 兼容模式**下，因为 PS 的选择行为不同（其实是"错"），`move` 的数据值必须在 1–4 范围内，不能传 0。

> 💡 **驱动端解析 Choice 的示意**：
>
> ```ts
> // 假设从 native 端读到一个 u8
> function parseChoice(raw: number): Choice {
>   const type = raw & 0x03;        // 低 2 位
>   const data = (raw >> 2) & 0x3F; // 高 6 位
>   switch (type) {
>     case 0: return { type: 'pass' };
>     case 1: return { type: 'move',   slot: data };  // 1..4 或 0
>     case 2: return { type: 'switch', slot: data };  // 2..6
>     default: throw new Error('invalid choice');
>   }
> }
> ```
> 具体编码细节请以各世代 `c.zig` / `node.zig` 的实现为准。

---

## Result / 结果

每次战斗 update 都返回一个**结果对象**，由三部分构成——result type、P1 的 choice type、P2 的 choice type。**任何非 `None` 的 result 都意味着战斗已结束**，再调用 update 会崩溃。

| Raw    | Description      |
| ------ | ---------------- |
| `0x00` | None（战斗继续） |
| `0x01` | P1 Wins |
| `0x02` | P2 Wins |
| `0x03` | P1 & P2 Tie |
| `0x04` | Error |

- **`Error`** 只在**desync / glitch**导致的崩溃路径上返回。由于 PS 改了自己的引擎代码避免这些情况，所以 **`-Dshowdown` 模式下永远不会返回 `Error`**。
- 但是 **`libpkmn` 的 C API 还会在另一种情况下返回 `Error`**：`-Dlog` 开启时 protocol 日志 buffer 不够大（写到一半就溢出）。**与是否 `-Dshowdown` 无关**。

结果里的 choice type 与上一节同义，但更接近 PS 中所谓的 **`requestType`**——告诉对应一方下一步该提供什么类型的 [choice request](https://github.com/smogon/pokemon-showdown/blob/master/sim/SIM-PROTOCOL.md#choice-requests)。在 pkmn 中你应该把这个 choice type 传回 `choices()` 来获得本方可选 choice 列表。

---

## Overview / 日志总览

`-Dlog` 开启后，[messages（消息）](#messages--消息) 被写到 `Log`：

- 每条消息的**第一个字节**是一个整数，表示该消息的 `ArgType`。
- 之后是 0 或更多字节的 payload。

游戏对象（招式 / 种族 / 特性 / 道具 / 属性等）**用其内部标识符**写——通常与公开编号一致，但若不一致，[`ids.json`](../../src/pkg/data/ids.json) 可用于解码。一份机器可读的 [`protocol.json`](../../src/data/protocol.json) 也可用于查 `ArgType` 与各种 "[reason](#reason--原因)" enum 值的人类可读名字。

**消息为变长，但每种消息的长度由 ArgType 决定**——解析器读到下一个 ArgType 头的位置即为本消息结束。**当下一个本应是 ArgType header 的位置读到 `0x00` 时，整段日志解析终止**。

> ⚠️ **注意：`0x00` 也可能作为消息 payload 内的合法字节出现**——只有当 `0x00` 出现在**消息 header 位置**时才表示结束。即使前一个 ArgType（如 `|win|` 或 `|turn|`）已经足够指明结束，引擎仍然会写一个 `0x00` 终止字节。

不像 Showdown 的协议，pkmn **不产生 `|request|` 消息**——驱动应该 inspect `Battle` 内存 + `Result` 自行判断"哪些 choice 可选"，并据此（考虑哪一方的私密信息）展示给用户。

### Reason / 原因

许多协议消息有一个 "**reason**" 字段，**提供进一步信息 / 上下文**，并可能**指示 payload 后面还有附加字节**。

- 仅在某些 reason 取值下才存在的字节，用一个**尾随 `?`** 标注（如 `[from]?`）。
- 每条带 reason 的消息会在其文档中列出所有可能取值、以及哪些取值会带额外字节。

reason 字段的存在是为了能**编码 PS 中存储在 "keyword args"（kwArgs）里的信息**——如 `[from]` 或 `[of]`。

### `PokemonIdent`

许多消息把 source / target / actor **编码为一个 `PokemonIdent`**（宝可梦 ID）。引用 PS 的文档：

> 宝可梦 ID 的格式是 `POSITION: NAME`：
>
> - `POSITION`：宝可梦的位置——由 `PLAYER`（见 `|player|`）加位置字母（单打里是 `a`）组成。
> - `NAME`：昵称（无昵称时为种族名）。
>
> 例：`p1a: Sparky` 可能是个昵称为 Sparky 的喷火龙。`p1: Dragonite` 可能是个未出场的快龙，被 Heal Bell 净化中。
>
> 大多数命令里光看位置就够了——只有 `|switch|`、`|replace|`、`|drag|`、`|detailschange|` 会换走那个位置上的宝可梦，并且这几条都会附带 `DETAILS` 让你更新视图。

pkmn 引擎略有不同——**昵称不在引擎里**。我们仍然编码 player + position letter + 宝可梦身份，但用**单个位打包字节**：

```
Bit:     7  6  5  4  3   2  1  0
        ┌──┬──┬──┬──┬──┬───────────┐
        │ always 000 │P │S │   slot │
        └──┴──┴──┴──┴──┴───────────┘
                      │  │       │
                      │  │       └── 队伍 1..6 槽位（开场时的原始位置）
                      │  └────────── 玩家：0 = P1，1 = P2
                      └───────────── 位置：0 = a，1 = b（双打时才用）
```

具体来说：

- **最高 3 位永远是 `0`**。
- **第 4 高位（bit 4）**：位置——`0` 为 `a`，`1` 为 `b`（仅双打有意义）。
- **第 5 高位（bit 3）**：玩家——`0` 为 P1，`1` 为 P2。
- **最低 3 位（bit 0–2）**：**该宝可梦在队伍中开场时的原始槽位号**（1–6 inclusive）。

> ⚠️ 注意是**"开场原始槽位"**，**不是**"当前在第几个位置"。这点关键：交换队伍顺序、Baton Pass 等都不会改变 ident。**ident 在战斗全程是稳定的，可作为该宝可梦的"身份证"使用**。

要从 pkmn 的 `PokemonIdent` 转换为 PS 风格的字符串（如 `p1a: Sparky`），驱动只需要维护两张映射表：

- **原始槽位 → 该位置宝可梦的昵称**
- **原始槽位 → 该宝可梦当前是否在场（如果是，在 `a` 还是 `b` 位）**

```ts
function toShowdownIdent(byte: number, p1Names: string[], p2Names: string[]): string {
  const slot     = byte & 0b00000111;          // 1..6
  const playerB  = (byte & 0b00001000) !== 0;  // 0=P1, 1=P2
  const posIsB   = (byte & 0b00010000) !== 0;  // 0=a,  1=b
  const player   = playerB ? 'p2' : 'p1';
  const position = posIsB ? 'b' : 'a';
  const name     = (playerB ? p2Names : p1Names)[slot - 1];
  return `${player}${position}: ${name}`;
}
```

### `LastStill` / `LastMiss`

不幸的是 PS 的协议**无法逐消息独立翻译**——`|move|` 消息可能被一个 `ArgType` 为 `LastStill`（`0x01`）或 `LastMiss`（`0x02`）的"后处理消息"修饰。

在 PS 模拟器里，`attrLastMove` 方法**在一批消息一起写之前**就修改了 `|move|`。但 **pkmn 是流式写出**、**不批处理**，所以解析者要自己做这件事：

- **遇到 `LastStill`（`0x01`）**：如果同一 buffer 中此前有一条 `|move|`，**给它追加一个 `[still]` keyword arg**。
- **遇到 `LastMiss`（`0x02`）**：给最近的 `|move|` **追加 `[miss]`**。

> 💡 **解析器伪代码**：
>
> ```ts
> function parseLog(buf: Uint8Array): Message[] {
>   const msgs: Message[] = [];
>   let lastMove: MoveMessage | null = null;
>   let i = 0;
>   while (i < buf.length && buf[i] !== 0x00) {
>     const arg = buf[i];
>     if (arg === 0x01) {            // LastStill
>       if (lastMove) lastMove.kwArgs.still = true;
>       i += 1;
>     } else if (arg === 0x02) {      // LastMiss
>       if (lastMove) lastMove.kwArgs.miss = true;
>       i += 1;
>     } else if (arg === 0x03) {      // |move|
>       const m = parseMove(buf, i);
>       msgs.push(m);
>       lastMove = m;
>       i += m.byteLength;
>     } else {
>       const m = parse(arg, buf, i);
>       msgs.push(m);
>       i += m.byteLength;
>     }
>   }
>   return msgs;
> }
> ```

---

## Messages / 消息

下面是每种消息的格式。**字节布局图、Reason 表格保持英文/原值**——它们是 binary 协议的事实定义；中文版只翻译描述性文字。

### `|move|` (`0x03`)

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0| 0x03          | Source        | Move          | Target        |
      +---------------+---------------+---------------+---------------+
     4| Reason        | [from]?       |
      +---------------+---------------+

`Source` 是用某 `Move` 攻击 `Target` 的宝可梦的 [`PokemonIdent`](#pokemonident)，原因为 `Reason`。若 `Reason` 为 `0x02`，则随后的一个字节指明 `Move` 是 `[from]` 哪个 Move 触发的。本消息可能被同一 buffer 中后续的 `LastStill` / `LastMiss` 修饰。

<details><summary>Reason</summary>

| Raw    | Description | `[from]`? |
| ------ | ----------- | --------- |
| `0x00` | None        | No        |
| `0x01` | `\|[from]`  | Yes       |

</details>

### `|switch|` (`0x04`)

#### Gen I

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0| 0x04          | Ident         | Species       | Level         |
      +---------------+---------------+---------------+---------------+
     4| Current HP                    | Max HP                        |
      +---------------+---------------+---------------+---------------+
     8| Status        |
      +---------------+

由 [`Ident`](#pokemonident) 标识的宝可梦换上场了，它是 `Level` 级的 `Species`，当前 HP / 最大 HP / 状态由后续字段给出。

#### Gen II

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0| 0x04          | Ident         | Species       | Gender        |
      +---------------+---------------+---------------+---------------+
     4| Level         | Current HP                    | Max HP ->     |
      +---------------+---------------+---------------+---------------+
     8| <- Max HP     | Status        | Reason        |
      +---------------+---------------+---------------+

Gen II 比 Gen I 多了 `Gender`（性别，因为 Gen II 才引入性别）与 `Reason`（Baton Pass 触发的换场要特殊标记）。

<details><summary>Reason</summary>

| Raw    | Description           |
| ------ | --------------------- |
| `0x00` | None                  |
| `0x01` | `\|[from] Baton Pass` |

</details>

### `|cant|` (`0x05`)

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0| 0x05          | Ident         | Reason        | Move?         |
      +---------------+---------------+---------------+---------------+

由 [`Ident`](#pokemonident) 标识的宝可梦因为 `Reason` 没能执行行动。若 `Reason` 为 `0x05`（Disable），则后续字节指明被禁用的 `Move`。

<details><summary>Reason</summary>

| Raw    | Description        | Move? |
| ------ | ------------------ | ----- |
| `0x00` | `slp`（睡眠）       | No |
| `0x01` | `frz`（冰冻）       | No |
| `0x02` | `par`（麻痹）       | No |
| `0x03` | `partiallytrapped`（束缚中） | No |
| `0x04` | `flinch`（畏缩）    | No |
| `0x05` | `Disable`          | **Yes** |
| `0x06` | `recharge`（充能回合） | No |
| `0x07` | `nopp`（PP 耗尽）   | No |

</details>

### `|faint|` (`0x06`)

    Byte/     0       |       1       |
       /              |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+
     0| 0x06          | Ident         |
      +---------------+---------------+

[`Ident`](#pokemonident) 标识的宝可梦倒下了。

### `|turn|` (`0x07`)

    Byte/     0       |       1       |       2       |
       /              |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+
     0| 0x07          | Turn                          |
      +---------------+---------------+---------------+

当前进入第 `Turn` 回合（2 字节 native-endian 整数）。

### `|win|` (`0x08`)

    Byte/     0       |       1       |
       /              |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+
     0| 0x06          | Player        |
      +---------------+---------------+

`Player` 一方赢得了战斗。（注：上面的字节图中第 0 字节标着 `0x06` 是原文笔误，实际 `|win|` 的 ArgType 是 `0x08`。）

### `|tie|` (`0x09`)

    Byte/     0       |
       /              |
      |0 1 2 3 4 5 6 7|
      +---------------+
     0| 0x09          |
      +---------------+

战斗以平局结束。

### `|-damage|` (`0x0A`)

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0| 0x0A          | Ident         | Current HP                    |
      +---------------+---------------+---------------+---------------+
     4| Max HP                        | Status        | Reason        |
      +---------------+---------------+---------------+---------------+
     8| [of]?         |
      +---------------+

[`Ident`](#pokemonident) 标识的宝可梦受到了伤害，HP 现在为 `Current HP / Max HP`，状态为 `Status`。**若 `Reason` 为 `0x05`，后续字节为伤害源（"`[of]`"）的 [`PokemonIdent`](#pokemonident)**。

<details><summary>Reason</summary>

| Raw    | Description     | `[of]`? |
| ------ | --------------- | ------- |
| `0x00` | None            | No |
| `0x01` | `psn`（中毒掉血） | No |
| `0x02` | `brn`（烫伤掉血） | No |
| `0x03` | `confusion`（混乱自残） | No |
| `0x04` | `Leech Seed`    | No |
| `0x05` | `Recoil\|[of]`（反伤） | **Yes** |
| `0x06` | `[from] Spikes` | No |

</details>

### `|-heal|` (`0x0B`)

字节布局与 `|-damage|` 相同。**若 `Reason` 为 `0x02`，后续字节为"`[of]`"——即 drain 招式吸取自哪只宝可梦**。

<details><summary>Reason</summary>

| Raw    | Description            | `[of]`? |
| ------ | ---------------------- | ------- |
| `0x00` | None                   | No |
| `0x01` | `\|[silent]`           | No |
| `0x02` | `\|[from] drain\|[of]` | **Yes** |
| `0x03` | `\|[from] Leftovers`   | No |

</details>

### `|-status|` (`0x0C`)

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0| 0x0C          | Ident         | Status        | Reason        |
      +---------------+---------------+---------------+---------------+
     4| [from]?       |
      +---------------+

[`Ident`](#pokemonident) 的宝可梦中了 `Status`。若 `Reason` 为 `0x02`，后续字节指明 `Status` 来自哪个 `Move`。

<details><summary>Reason</summary>

| Raw    | Description  | `[from]`? |
| ------ | ------------ | --------- |
| `0x00` | None         | No |
| `0x01` | `\|[silent]` | No |
| `0x02` | `\|[from]`   | **Yes** |

</details>

### `|-curestatus|` (`0x0D`)

[`Ident`](#pokemonident) 的宝可梦从 `Status` 中恢复。

<details><summary>Reason</summary>

| Raw    | Description  |
| ------ | ------------ |
| `0x00` | `\|[msg]`    |
| `0x01` | `\|[silent]` |

</details>

### `|-boost|` / `|-unboost|` (`0x0E`)

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0| 0x0E          | Ident         | Reason        | Num           |
      +---------------+---------------+---------------+---------------+

[`Ident`](#pokemonident) 的宝可梦在 `Reason` 指明的能力上**变化了 `Num` − 6 个等级**——也就是说 `Num=6` 表示无变化，`Num=7` 表示 +1，`Num=5` 表示 −1，以此类推。**这种"加偏移 6"的编码可以让 `Num` 始终是无符号的**。

<details><summary>Reason</summary>

| Raw    | Description        |
| ------ | ------------------ |
| `0x00` | `atk\|[from] Rage` |
| `0x01` | `atk`              |
| `0x02` | `def`              |
| `0x03` | `spe`              |
| `0x04` | `spa`              |
| `0x05` | `spd`              |
| `0x06` | `accuracy`         |
| `0x07` | `evasion`          |

</details>

### `|-clearallboost|` (`0x0F`)

清除场上所有宝可梦的能力变化（双方 + 双打的两侧）。

### `|-fail|` (`0x10`)

[`Ident`](#pokemonident) 的宝可梦使用的某动作因自身机制原因失败。

<details><summary>Reason</summary>

| Raw    | Description                |
| ------ | -------------------------- |
| `0x00` | None                       |
| `0x01` | `slp` |
| `0x02` | `psn` |
| `0x03` | `brn` |
| `0x04` | `frz` |
| `0x05` | `par` |
| `0x06` | `to`（对目标 fail） |
| `0x07` | `move: Substitute` |
| `0x08` | `move: Substitute\|[weak]`（HP 不够分身） |

</details>

### `|-miss|` (`0x11`)

[`Ident`](#pokemonident) 的宝可梦的招式未命中。

### `|-hitcount|` (`0x12`)

[`Ident`](#pokemonident) 的宝可梦被多段招式击中 `Num` 次。

### `|-prepare|` (`0x13`)

[`Ident`](#pokemonident) 的宝可梦正在为 `Move` **蓄力**（如 Solar Beam、Sky Attack 的蓄力回合）。

### `|-mustrecharge|` (`0x14`)

[`Ident`](#pokemonident) 的宝可梦本回合必须**充能（休息）**（如使用了 Hyper Beam 后的下一回合）。

### `|-activate|` (`0x15`)

`Reason` 指明的某杂项效果在 [`Ident`](#pokemonident) 上**触发**。

<details><summary>Reason</summary>

| Raw    | Description            |
| ------ | ---------------------- |
| `0x00` | `Bide`                 |
| `0x01` | `confusion`            |
| `0x02` | `move: Haze`           |
| `0x03` | `move: Mist`\*         |
| `0x04` | `move: Struggle`       |
| `0x05` | `Substitute\|[damage]` |
| `0x06` | `\|\|move: Splash`     |

\* *PS 会把 Mist 在协议层"升级"为 `|-block|` 消息。*

</details>

### `|-fieldactivate|` (`0x16`)

场地状态触发。

### `|-start|` (`0x17`)

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0| 0x17          | Ident         | Reason        | Move/Types?   |
      +---------------+---------------+---------------+---------------+
     4| [of]?         |
      +---------------+

[`Ident`](#pokemonident) 的宝可梦获得了一个由 `Reason` 描述的**临时状态（volatile status）**。

- 若 `Reason` 为 `0x09`（Conversion 改属性），后续字节指明 `Types`（变成的属性）与 [`PokemonIdent`](#pokemonident) `[of]`（属性参考对象）。
- 若 `Reason` 为 `0x0A`（Disable）或 `0x0B`（Mimic），后续字节为对应的 `Move`。

<details><summary>Reason</summary>

| Raw    | Description                                      | Move/Types? | `[of]`? |
| ------ | ------------------------------------------------ | ----------- | ------- |
| `0x00` | `Bide`\* | No | No |
| `0x01` | `confusion` | No | No |
| `0x02` | `confusion\|[silent]` | No | No |
| `0x03` | `move: Focus Energy` | No | No |
| `0x04` | `move: Leech Seed` | No | No |
| `0x05` | `Light Screen` | No | No |
| `0x06` | `Mist` | No | No |
| `0x07` | `Reflect` | No | No |
| `0x08` | `Substitute` | No | No |
| `0x09` | `typechange\|...\|[from] move: Conversion\|[of]` | **Yes** | **Yes** |
| `0x0A` | `Disable\|` | **Yes** | No |
| `0x0B` | `Mimic\|`   | **Yes** | No |

\* *`0x00` 在 Gen II 中对应 `move: Bide`。*

</details>

### `|-end|` (`0x18`)

[`Ident`](#pokemonident) 的宝可梦身上由 `Reason` 引起的临时状态**结束**。

> FIXME（原文标记）：leechseed 的 `[from]` / `[of]` 待补完。

<details><summary>Reason</summary>

| Raw    | Description |
| ------ | ----------- |
| `0x00` | `Disable`\* |
| `0x01` | `confusion` |
| `0x02` | `Bide`\* |
| `0x03` | `Substitute` |
| `0x04` | `Disable\|[silent]` |
| `0x05` | `confusion\|[silent]` |
| `0x06` | `mist\|[silent]` |
| `0x07` | `focusenergy\|[silent]` |
| `0x08` | `leechseed\|[silent]` |
| `0x09` | `Toxic counter\|[silent]` |
| `0x0A` | `lightscreen\|[silent]` |
| `0x0B` | `reflect\|[silent]` |
| `0x0C` | `move: Bide\|[silent]` |

\* *`0x00` 在 Gen II 中对应 `move: Disable`；`0x02` 对应 `move: Bide`。*

</details>

### `|-ohko|` (`0x19`)

一击必杀招式成功使用。

### `|-crit|` (`0x1A`)

对 [`Ident`](#pokemonident) 的宝可梦造成了**击中要害**（暴击）。

### `|-supereffective|` (`0x1B`)

对 [`Ident`](#pokemonident) 的宝可梦造成了**效果绝佳**。

### `|-resisted|` (`0x1C`)

对 [`Ident`](#pokemonident) 的宝可梦造成了**效果不太好**。

### `|-immune|` (`0x1D`)

[`Ident`](#pokemonident) 的宝可梦**免疫**了招式。

<details><summary>Reason</summary>

| Raw    | Description |
| ------ | ----------- |
| `0x00` | None        |
| `0x01` | `\|[ohko]`  |

</details>

### `|-transform|` (`0x1E`)

`Source` ([`PokemonIdent`](#pokemonident)) **变身**为 `Target` ([`PokemonIdent`](#pokemonident))。

### `|drag|` (`0x1F`)

字节布局类似 Gen II 的 `|switch|`（无 `Reason`）：

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0| 0x1F          | Ident         | Species       | Gender        |
      +---------------+---------------+---------------+---------------+
     4| Level         | Current HP                    | Max HP ->     |
      +---------------+---------------+---------------+---------------+
     8| <- Max HP     | Status        |
      +---------------+---------------+

宝可梦被**强制拽上场**（如 Whirlwind / Roar 触发）。

### `|-item|` (`0x20`)

    Byte/     0       |       1       |       2       |       3       |
       /              |               |               |               |
      |0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|0 1 2 3 4 5 6 7|
      +---------------+---------------+---------------+---------------+
     0| 0x20          | Target        | Item          | Source        |
      +---------------+---------------+---------------+---------------+

`Target` 的持有道具变成 / 被揭示为 `Item`，**由 `Source` 通过 Thief 等触发**。

### `|-enditem|` (`0x21`)

[`Ident`](#pokemonident) 持有的 `Item` **因 `Reason` 丢失或消耗**。

<details><summary>Reason</summary>

| Raw    | Description |
| ------ | ----------- |
| `0x00` | None        |
| `0x01` | `\|[eat]`   |

</details>

### `|-cureteam|` (`0x22`)

[`Ident`](#pokemonident) 用 Heal Bell 让全队恢复状态。

### `|-sethp|` (`0x23`)

[`Ident`](#pokemonident) 的 HP 被直接**设置**为 `Current HP / Max HP`，状态为 `Status`。

<details><summary>Reason</summary>

| Raw    | Description                         |
| ------ | ----------------------------------- |
| `0x00` | `[from] move: Pain Split`           |
| `0x01` | `[from] move: Pain Split\|[silent]` |

</details>

### `|-setboost|` (`0x24`)

[`Ident`](#pokemonident) 的攻击能力被**直接设置**（而非增加）为 `Num` − 6，由 Belly Drum 触发。

### `|-copyboost|` (`0x25`)

`Source` 复制 `Target` 的能力变化。

### `|-sidestart|` (`0x26`)

`Player` 一侧**开启**了 `Reason` 对应的场地状态。

<details><summary>Reason</summary>

| Raw    | Description          |
| ------ | -------------------- |
| `0x00` | `Safeguard`          |
| `0x01` | `move: Light Screen` |
| `0x02` | `Reflect`            |
| `0x03` | `Spikes`             |

</details>

### `|-sideend|` (`0x27`)

`Player` 一侧的某场地状态**结束**。`Reason` 与 `|-sidestart|` 相同；若 `Reason` 为 `0x03`（Spikes 被 Rapid Spin 移除），后续字节为 [`PokemonIdent`](#pokemonident) `[of]`（移除者）。

### `|-singlemove|` (`0x28`)

[`Ident`](#pokemonident) 使用 `Move` 引发了**只持续本招式期间**的临时效果。

### `|-singleturn|` (`0x29`)

[`Ident`](#pokemonident) 使用 `Move` 引发了**只持续本回合**的临时效果。

### `|-weather|` (`0x2A`)

天气进入 / 持续。`Reason` 为 `0x01` 表示这是 upkeep。

<details><summary>Weather</summary>

| Raw    | Description |
| ------ | ----------- |
| `0x00` | None        |
| `0x01` | Rain（下雨） |
| `0x02` | Sun（日照） |
| `0x03` | Sandstorm（沙暴） |

</details>

<details><summary>Reason</summary>

| Raw    | Description  |
| ------ | ------------ |
| `0x00` | None         |
| `0x01` | `\|[upkeep]` |

</details>

---

## Size / 日志大小

如前所述，任何 `Writer` 都可以作为 `Log` 的后端——你可以用一个 [`ArrayList.Writer`](https://ziglang.org/documentation/master/std/#root;ArrayList) 来支持任意大小的日志。但**性能上**我们更希望预分配定长 buffer。**推荐大小**是 `pkmn.LOGS_SIZE`，它至少能容纳 `pkmn.MAX_LOGS` 字节（这些常量取所有世代中的最大值；每个世代也有自己的并行常量，如 `pkmn.gen1.LOGS_SIZE`）。

**确定"单次 update 的最大日志字节数"**是非平凡的，并且会根据约束变化：

- **Standard**：最严的约束——PS "Standard" 竞技规则（Species Clause、Cleric Clause、移动池合法性 + 禁招等）。
- **Cartridge**：卡带合法性——仍要求合法获得 + 可用于 link battle，但不强制 PS 的 clauses / mods。
- ["**Hackmons**"](https://www.smogon.com/articles/pure-hackmons-introduction)：比卡带更宽——可以 hack 任何招式 / 道具 / 特性 / 属性 / 数值组合。
- **Fuzzing**：测试用——Hackmons 之上还允许**战斗中任意操纵状态**（设置不可能的 volatile 组合等）。

此外，RNG 解释还可以分**lax**（任何"理论上可能的事件序列"都算）和 **strict**（只考虑卡带或 PS 实际 RNG 能产出的）。

**总体上**，常量定义为**在 cartridge + lax RNG 假设下足够**（某些情况下需要 strict 解释来收紧上界）。实践中常量通常按 hackmons 给——因为 fuzz 用 hackmons。具体推导见每代下文。**多数推导都借助 [Z3 定理证明器](https://github.com/Z3Prover/z3)。**

> *以下 `MAX_LOGS` 场景的推导得益于 [**@gigalh128**](https://github.com/gigalh128) 的帮助。*

### Generation I

Gen I 的 `MAX_LOGS` 常量为 **180**。要达到这个数需要构造一个非常具体的场景：

> **两只都烫伤的化石翼龙（Aerodactyl），都学了 Leech Seed、Confuse Ray、Metronome，其中一只比另一只慢。**
>
> - **回合 1–2**：两只各自用 Leech Seed 然后 Confuse Ray。
> - **目标回合**：快的那只用 Metronome 触发**击中要害的 Fury Swipes**，恰好打满 5 段；慢的那只用 Metronome 触发 Mirror Move，**再触发**击中要害的 Fury Swipes 5 段。
>
> 初始 seed 为 $\{180, 137, 181, 165, 16, 97, 148, 20, 25\}$。

<details><summary>详细推导</summary>

要让 Gen I 的单回合日志最大化，要注意若干关键观察：

- `|move|`、`|switch|`、`|-damage|`、`|-heal|` **占字节最多**。
- `|switch|` 反而比 `|move|` 字节少——它**不会触发其它消息**，而 `|move|` 能触发一堆（`|-damage|`、`|-heal|` 等）。
- 一个 `|move|` 若**同时是击中要害且效果绝佳/不好**，会多出 `|-crit|` + `|-supereffective|` / `|-resisted|`。
- `|move|` 之前可能触发 confusion，之后毒/烫伤的残伤、Leech Seed 都会触发。
- **双方都不能在结尾倒下**。乍看 `|faint|` + `|win|` 是好事（2×`|faint|` + 1×`|win|` = 6 字节，单个 `|faint|` + `|turn|` 只有 5 字节），但**一旦有一方倒下，就丢失了 Leech Seed 的"伤害 + 治疗"环节**——那 16 字节大得多。
- **Substitute 触发实际上减小日志**——`|-activate|` 替代了更大的 `|-damage|`；并且 Gen I 中分身破坏会**取消招式剩余效果**。
- 多段（MultiHit）招式是**最优的**——它能产生 5 条 `|-damage|` + 1 条 `|-hitcount|`。

**最关键的观察是：Metronome 和 Mirror Move 可以相互递归无限放大日志大小。**

Metronome 和 Mirror Move 各自有"防止自我递归"的检查，但**互相之间没有检查**。所以一只宝可梦可以：

```
Metronome → 选中 Mirror Move → 复制对手的 Metronome → 选中 Mirror Move → ...
```

但有个限制：Mirror Move 用的是 `last_used_move` 字段，**Metronome 调用新招式会覆盖此字段**——所以 Mirror Move 想复制对手的 Metronome，**对手必须被"卡在某个蓄力招式里"**（这样他这回合不能挥手覆盖 last_used_move）。这又意味着**另一方不能也搞同样的递归**。

理论上每层 Metronome → Mirror Move 的概率是 $({1\over 163})^N$（Metronome 在 163 招中挑），听起来 vanishingly small。但 Red 和 Showdown 用的是**伪随机**——**实际上你可以通过精心选择初始 seed 让连续 N 层都成立**。

在 PS 中，每次 RNG 调用之间有若干额外帧推进（命中判定、re-target 等），实际上设不出来。但在 **Pokémon Red** 中，Metronome 和 Mirror Move 之间**只有一次无意义的击中要害掷骰**（值被丢弃）——seed 可以被精心安排出 9 个有用值。

定义 $X$、$X' = 5X + 1 \bmod 256$、$X'' = 5X' + 1 \bmod 256$。要让 Metronome 选中 Mirror Move，需要 $X'' = 119$。所以 $X' = 126$、$X = 25$。

**从 seed $\{25, 126, 56, 25, 126, 56, 25, 126, 56\}$ 开始可以达到 9 层递归。** 这是一个**理论上界**（实际上你还要"先安排好前两回合的 Leech Seed + Confuse Ray + 状态"+"另一边的设置"，留给你的初始 seed 自由度并没那么多）。而且只有**一方**能享受这种递归。

于是上界场景分两种：

#### 场景 1：一方深度递归 + 另一方蓄力锁定（188 字节）

| 项 | 计算 |
| --- | --- |
| `|-activate|` confusion × 2 | 2×3 = 6 |
| `|move|` Metronome → Mirror Move P1 递归 | 1×5 + 9×6 = 59 |
| `|move|` multi-hit | 6 |
| `|move|` P2 turn 2 蓄力 | 6 |
| `|-crit|` × 2 | 2×2 = 4 |
| `|-supereffective|` 或 `|-resisted|` × 2 | 2×2 = 4 |
| `|-damage|` multi-hit × 5 | 5×8 = 40 |
| `|-hitcount|` | 3 |
| `|-damage|` 蓄力 | 8 |
| `|-damage|` poison/burn × 2 | 2×8 = 16 |
| `|-damage|` Leech Seed × 2 | 2×8 = 16 |
| `|-heal|` Leech Seed × 2 | 2×8 = 16 |
| `|turn|` | 3 |
| `0x00`（终止字节） | 1 |
| **合计** | **188** |

#### 场景 2：双方都一次 Metronome → Mirror Move → multi-hit（186 字节）

| 项 | 计算 |
| --- | --- |
| `|-activate|` confusion × 2 | 2×3 = 6 |
| `|move|` × (Metronome → Mirror Move → multi-hit) | 2×5 + 4×6 = 34 |
| `|-crit|` × 2 | 2×2 = 4 |
| `|-supereffective|`/`|-resisted|` × 2 | 2×2 = 4 |
| `|-damage|` multi-hit × 10 | 10×8 = 80 |
| `|-hitcount|` × 2 | 2×3 = 6 |
| `|-damage|` poison/burn × 2 | 16 |
| `|-damage|` Leech Seed × 2 | 16 |
| `|-heal|` Leech Seed × 2 | 16 |
| `|turn|` | 3 |
| `0x00` | 1 |
| **合计** | **186** |

#### Z3 验证

可以用 Z3 测试这些场景。Z3 显示：

- **场景 1 不可达**——即使从初始 seed 出发能做到 9 层递归，但**前两回合的 setup 把太多 roll 用光**了，无法在 turn 3 满足"先做完所有 Metronome → Mirror Move 的特定值再加上 multi-hit 的所有相关 roll"。
- **场景 2 也不可达**——P1 要触发 Mirror Move 需要**额外一回合 setup**，吃掉的 roll 也太多。

但是**只要让 P1 不做 Mirror Move、直接 Metronome → multi-hit**，就只损失 6 字节——**最终上界是 180 字节**。

下面是用于验证的 Python + Z3 脚本（与原文相同，保留 ASCII / 原变量名以便复制运行）：

```py
#!/usr/bin/env python
from z3 import *

for name, move, hit in [
   ('Spike Cannon', 131, 255),
   ('Double Slap', 3, 215),
   ('Comet Punch', 4, 215),
   ('Fury Attack', 31, 215),
   ('Pin Missile', 42, 215),
   ('Barrage', 140, 215),
   ('Fury Swipes', 154, 203),
]:
  N = 5
  for d1 in range(N):
    for d2 in range(N):
      for m1 in range(N):
        for m2 in range(N):
          total = 9 + 15 + d1 + d2 + m1 + m2
          state = [BitVec('state%s' % (i + 1), 8) for i in range(total)]

          s = Solver()

          for i in range(total - 9):
            s.add(state[i + 9] == state[i] * 5 + 1)

          # NOTE: first 9 states must all be < 253
          s.assert_and_track(ULE(state[0] * 5 + 1, 228), 'Turn 1: P1 Leech Seed hit')
          s.assert_and_track(ULE(state[1] * 5 + 1, 228), 'Turn 1: P2 Leech Seed hit')

          s.assert_and_track(ULT(state[2] * 5 + 1, 253), 'Turn 2: P1 Confuse Ray hit')
          s.assert_and_track(And(ULT(state[3] * 5 + 1, 253), UGE(((state[3] * 5 + 1) & 3) + 2, 3)),
                             'Turn 2: P2 confusion duration (any)')
          s.assert_and_track(ULT(state[4] * 5 + 1, 128), 'Turn 2: P2 avoid confusion self-hit')
          s.assert_and_track(ULT(state[5] * 5 + 1, 253), 'Turn 2: P2 Confuse Ray hit')
          s.assert_and_track(ULT(state[6] * 5 + 1, 253), 'Turn 2: P1 confusion duration (any')

          s.assert_and_track(ULT(state[7] * 5 + 1, 128), 'Turn 3: P1 avoid confusion self-hit')
          s.assert_and_track(ULT(state[8] * 5 + 1, 253), 'Turn 3: P1 Metronome crit (any)')

          i = 9
          for m in range(m1):
            s.assert_and_track(UGE(state[i] * 5 + 1, 163), f'Turn 3: P1 Metronome no-op {m}')
            i += 1
          s.assert_and_track(state[i] * 5 + 1 == move, f'Turn 3: P1 Metronome proc {name}')
          i += 1
          s.assert_and_track(ULT(RotateLeft(state[i] * 5 + 1, 3), 65), f'Turn 3: P1 {name} crits')
          i += 1
          for d in range(d1):
            s.assert_and_track(ULT(RotateRight(state[i] * 5 + 1, 1), 217),
                               f'Turn 3: P1 {name} damage roll no-op {d}')
            i += 1
          s.assert_and_track(UGE(RotateRight(state[i] * 5 + 1, 1), 217),
                             f'Turn 3: P1 {name} damage roll')
          i += 1
          s.assert_and_track(ULE(state[i] * 5 + 1, hit), f'Turn 3: P1 {name} hit')
          i += 1
          s.assert_and_track(UGE((state[i] * 5 + 1) & 3, 2), f'Turn 3: P1 {name} first hitcount')
          i += 1
          s.assert_and_track(((state[i] * 5 + 1) & 3) + 2 == 5, f'Turn 3: P1 {name} max hitcount')
          i += 1
          s.assert_and_track(ULT(state[i] * 5 + 1, 128), 'Turn 3: P2 avoid confusion self-hit')
          i += 1
          s.assert_and_track(ULT(state[i] * 5 + 1, 255), 'Turn 3: P2 Metronome crit (any)')
          i += 1
          for m in range(m2):
            s.assert_and_track(UGE(state[i] * 5 + 1, 163), f'Turn 3: P2 Metronome no-op {m}')
            i += 1
          s.assert_and_track(state[i] * 5 + 1 == 119, 'Turn 3: P2 Metronome proc MirrorMove')
          i += 1
          s.assert_and_track(ULT(state[i] * 5 + 1, 255), 'Turn 3: P2 MirrorMove crit (any)')
          i += 1
          s.assert_and_track(ULT(RotateLeft(state[i]  * 5 + 1, 3), 65), f'Turn 3: P2 {name} crits')
          i += 1
          for d in range(d2):
            s.assert_and_track(ULT(RotateRight(state[i] * 5 + 1, 1), 217),
                               f'Turn 3: P2 {name} damage roll no-op {d}')
            i += 1
          s.assert_and_track(UGE(RotateRight(state[i] * 5 + 1, 1), 217),
                             f'Turn 3: P2 {name} damage roll')
          i += 1
          s.assert_and_track(ULE(state[i] * 5 + 1, hit), f'Turn 3: P2 {name} hit')
          i += 1
          s.assert_and_track(UGE((state[i] * 5 + 1) & 3, 2), f'Turn 3: P2 {name} first hitcount')
          i += 1
          s.assert_and_track(((state[i] * 5 + 1) & 3) + 2 == 5, f'Turn 3: P2 {name} max hitcount')

          print(name, end = ': ')
          if (s.check() != unsat):
            m = s.model()

            for i in range(8):
              print(m[state[i]], end = ', ')
            print(m[state[8]])
            exit(0)
          else:
              print(s.unsat_core())

exit(1)
```

</details>

### Generation II

TODO（推导仍在进行中——Gen II 引入了道具与天气，状态空间显著增大）。

---

## 附录：完整解析流程（伪代码）

> 这是一个**端到端**的"如何从 `-Dlog` 输出读出可视化日志"的示意，便于你写自己的驱动 / 调试器。

```ts
async function processBattle() {
  // 1. 启动引擎、读 header
  const header = await readHeader();
  const isShowdown = header[0] === 1;
  const gen        = header[1];
  const N          = readI16LE(header, 2);  // log buffer 大小（<0 表示变长）
  const X          = readI32LE(header, 4);  // 额外数据 buffer 大小（<0 表示变长）
  const initial    = readState(header, 8, gen);  // 初始 Battle 状态

  // 2. 主循环：处理每一帧
  while (true) {
    const frame = await readFrame(N, X);

    // 2a. 解析日志消息（参考前文 LastStill/LastMiss 章节）
    const messages = parseLog(frame.log);

    // 2b. 读取额外数据（chance/calc 等）
    const extra    = parseExtra(frame.extra);

    // 2c. 读取战斗状态（按对应世代的 layout）
    const state    = readState(frame.state, 0, gen);

    // 2d. 读取本次更新的结果
    const result   = frame.result;  // 1 字节
    const resType  = result & 0x07;
    const p1Choice = (result >> 3) & 0x03;
    const p2Choice = (result >> 5) & 0x03;

    // 2e. 终态？
    if (resType !== 0) {
      console.log('battle ended:', resType);
      break;
    }

    // 2f. 拿到下回合双方的 next choice 字节
    const p1NextChoice = parseChoice(frame.p1Choice);
    const p2NextChoice = parseChoice(frame.p2Choice);

    // ... 渲染日志 / 推进 UI ...
  }
}
```

> 实际生产代码请参考 [`src/pkg/protocol.ts`](../../src/pkg/protocol.ts) 与 [`src/tools/debug.ts`](../../src/tools/debug.ts)。

---

## 协议设计与字节数总结

| 协议特性 | pkmn 选择 | PS 选择 |
| --- | --- | --- |
| 编码 | 二进制（fixed-length-per-message） | 文本（行 + 管道分隔） |
| 字节序 | native-endian | UTF-8 文本 |
| 字段 | 位打包到极致 | JSON-like kwArgs |
| `|split|` 双流 | 无（只产生 omniscient） | 有 |
| `|request|` 消息 | 无 | 有 |
| `LastStill` / `LastMiss` 批处理 | **解析方负责** | **写入方负责** |
| 终止 | `0x00` 字节 + 长度有界 | 流式 + EOF |
| 协议跨代差异 | 显式（按 ArgType 各代独立定义） | 按"mod"分支 |

> 协议设计的根本思想与 [`DESIGN_zh.md`](./DESIGN_zh.md) 列出的引擎设计原则**完全一致**：把字符串处理、UI 适配、双流分发等**昂贵或不通用**的工作都推到驱动代码层，让引擎本身只做"写最少的字节"。
