# pkmn/engine 代码库高层解读（中文）

> 这份文档面向"没接触过游戏引擎设计"的读者，从 high-level 角度讲解 `pkmn/engine` 这个仓库的整体设计、抽象、实体，并讨论如果想为类似的 PKM 对战游戏（例如《洛克王国》）开发一个对应的引擎，应该怎么做。
>
> 配套的可视化页面：[`ENGINE_ANALYSIS_zh.html`](./ENGINE_ANALYSIS_zh.html)

---

## 1. 这是个什么项目？

`pkmn/engine` 是一个**宝可梦对战模拟器**的底层引擎，目标只有一个：**把"两个训练家选指令 → 战斗状态更新 → 直到分出胜负"这件事做得极致快、极致紧凑，并且与官方原版游戏（GB 卡带）以及 Pokémon Showdown 模拟器在结果上**逐帧一致**。

它**不是一个完整的游戏**。它不渲染画面、不联机、不管玩家账号、不做队伍合法性校验，甚至**不打印一个字符串**——所有日志都是二进制协议。它只是一个被嵌入到其它系统里使用的"战斗内核"（library / dll / wasm 模块）。

| 特征 | 说明 |
| --- | --- |
| **核心实现** | Zig 语言（`src/lib`），编译成 `libpkmn.so` / `.dll` / WebAssembly |
| **驱动代码** | TypeScript（`src/pkg`），包装 native 库并发布为 `@pkmn/engine` npm 包 |
| **C 接口** | `src/include/pkmn.h`，供其它语言（C++、Python、Rust…）绑定 |
| **设计原则** | **No compromises on performance**：一切让位于性能 |
| **比较基准** | 比官方简化版 Pokémon Showdown 模拟器快 **1000 倍以上** |
| **当前状态** | Stage 1：实现 Gen 1（红蓝）与 Gen 2（金银），后续会做 Gen 3/4 |

---

## 2. 项目目录速览

```
pkmengine/
├── build.zig / build.zig.zon     ← Zig 构建系统
├── package.json / tsconfig.json  ← TS 构建系统
├── Makefile                      ← 顶层调度（调 build.zig 和 npm）
├── docs/                         ← DESIGN.md / PROTOCOL.md / TESTING.md 等设计文档
├── examples/                     ← C / JS / Zig 三套使用示例
└── src/
    ├── include/pkmn.h            ← 对外的 C 头文件（FFI 边界）
    ├── lib/                      ← ★ 引擎核心（Zig）
    │   ├── pkmn.zig              ← 库总入口
    │   ├── c.zig / node.zig / wasm.zig  ← 三种外部绑定
    │   ├── common/               ← 跨世代共享：data、protocol、rng、options...
    │   ├── gen1/                 ← 第一世代引擎（独立实现）
    │   │   ├── data.zig          ← 数据结构（Battle / Side / Pokemon / ActivePokemon…）
    │   │   ├── data/             ← 静态数据表（moves、species、types）
    │   │   ├── mechanics.zig     ← ★ 战斗主流程（3000+ 行的"大脑"）
    │   │   ├── chance.zig        ← RNG 事件追踪
    │   │   ├── calc.zig          ← 伤害计算覆写
    │   │   └── helpers.zig       ← 构造便捷器（测试/工具用）
    │   └── gen2/                 ← 第二世代引擎（独立实现，带 Item）
    ├── pkg/                      ← TS 驱动代码（@pkmn/engine）
    ├── data/                     ← JSON dump，供其它语言绑定参考
    ├── bin/                      ← install-pkmn-engine、pkmn-debug 等 CLI
    └── tools/                    ← 代码生成器（生成 moves.zig、species.zig）
```

**关键设计观察**：

1. **每一代都独立实现**（gen1、gen2 完全分开），不像 Pokémon Showdown 用一份"现代代码 + mods"覆盖所有世代。代价是有点代码重复，好处是每一代的代码都极简、极快、易推理。
2. **静态数据是代码生成的**（`tools/generate` 生成 `data/moves.zig`、`data/species.zig`），不是从 JSON 运行时加载——查表就是直接数组下标。
3. **没有字符串、没有动态分配、没有指针**——所有"标识符"都是小的 `enum`，所有引用都是数组下标（"handle"）。

---

## 3. 引擎中的核心抽象与实体

下面这张表是理解整个引擎最重要的一张表：

### 3.1 数据实体（Entities，"名词"）

| 实体 | 文件 | 大小 | 含义 |
| --- | --- | --- | --- |
| **`Battle`** | `gen1/data.zig` | 384 字节 | 一场战斗的**全部**状态。包含两个 `Side` + 回合数 + RNG 状态 + 上一次伤害值 |
| **`Side`** | `gen1/data.zig` | 184 字节 | 一方训练家的状态：6 只队伍宝可梦 + 出场宝可梦 + 顺序 + 上一招 |
| **`Pokemon`** | `gen1/data.zig` | 24 字节 | **未出场**宝可梦的存储状态：原始能力值、4 个招式槽、HP、状态、种族、类型、等级 |
| **`ActivePokemon`** | `gen1/data.zig` | 32 字节 | **出场中**宝可梦的运行时状态：修改后能力值、当前种族（变身用）、当前类型、能力等级、**Volatiles**、招式槽 |
| **`MoveSlot`** | `gen1/data.zig` | 2 字节 | `{ 招式 ID, 剩余 PP }`，每只宝可梦 4 个 |
| **`Volatiles`** | `gen1/data.zig` | 一组 bit 位 | 临时状态：混乱、束缚、替身、屯能、变身、麻痹/睡眠剩余回合、Toxic 计数等 |
| **`Status`** | `gen1/data.zig` | 1 字节 enum | 主要状态：SLP / PSN / BRN / FRZ / PAR |
| **`Choice`** | `common/data.zig` | 1 字节 | 玩家的一个**指令**：`{type, data}`，type ∈ {Pass, Move, Switch} |
| **`Result`** | `common/data.zig` | 1 字节 | `update` 返回值：胜/负/平/继续，并告诉两方"下一步可以选什么类型的指令" |
| **`ID`** | `common/data.zig` | 1 字节 | 战场上某只宝可梦的引用 = `{player, slot}` 压成 8 位 |

**关键**：`Pokemon` 和 `ActivePokemon` **是两个不同的结构**。这是性能驱动的："存档状态"和"战斗中状态"分开存，使得最热的字段（active 的能力值、boost、volatiles）紧凑地放在一起，最大化 CPU cache 命中。

### 3.2 静态数据（Static Data，"图鉴"）

这些不是某次战斗的状态，而是"这个世代的世界规则"：

| 静态数据 | 文件 | 形态 |
| --- | --- | --- |
| **`Move` 枚举 + `Move.Data`** | `gen1/data/moves.zig` | 165 个招式枚举 + 长度 165 的 `Data` 数组，每条 4 字节（`{effect, bp, accuracy, type, target}`） |
| **`Species` + `Species.Stats`** | `gen1/data/species.zig` | 151 只宝可梦的基础种族值、类型 |
| **`Type` + 相性表** | `gen1/data/types.zig` | 15 种属性、属性相克表 |
| **`Item`**（Gen 2 起） | `gen2/data/items.zig` | 持有道具枚举 + 元数据 |

通过 `Move.get(id)` 这种 O(1) 数组下标访问，不需要任何 hash map。

### 3.3 行为抽象（Behaviors，"动词"）

引擎对**行为**做了几个关键抽象，这些都集中在 `gen1/mechanics.zig`：

| 抽象 | 函数 | 作用 |
| --- | --- | --- |
| **`update(c1, c2)`** | `mechanics.update` | **整个引擎唯一的对外动作**：吃两边的指令，前进一个回合 |
| **`choices(player, result)`** | `mechanics.choices` | 给定当前 `Result`，列出该玩家此刻能选的所有合法指令 |
| **`selectMove`** | 内部 | 处理"必须连续动（屯能、束缚）"等强制选指令 |
| **`turnOrder`** | 内部 | 比较速度、考虑优先度，决定本回合谁先动 |
| **`doTurn` → `executeMove`** | 内部 | 一方完成"出招"的完整流水线 |
| **`beforeMove`** | 内部 | 麻痹/睡眠/冰冻/混乱/退缩等**出招前**的判定 |
| **`calcDamage` / `adjustDamage` / `randomizeDamage`** | 内部 | 三段式伤害计算：原始公式 → STAB/相性 → 随机数 |
| **`checkHit` / `moveHit`** | 内部 | 命中判定 |
| **`applyDamage`** | 内部 | 真正扣 HP，处理替身吸收等 |
| **`Effects.*`** | 内部 | **招式的"副作用"实现**（详见 §4） |
| **`handleResidual`** | 内部 | 回合末：中毒、烧伤、Leech Seed 等"残留伤害" |
| **`endTurn`** | 内部 | 收尾：检查胜负 → 写日志 → 返回 `Result` |

### 3.4 跨切面抽象（Cross-cutting）

| 抽象 | 在哪 | 作用 |
| --- | --- | --- |
| **`PRNG`** | `common/rng.zig` + `gen1/data.zig` | 抽象 RNG 接口。Gen 1 默认用卡带的 RNG；如果开 `-Dshowdown` 则用 Showdown 的 RNG；测试时可换成 **`FixedRNG`** 强制指定每次掷骰结果 |
| **`Log` / Protocol** | `common/protocol.zig` | **可选**的二进制日志输出（开启 `-Dlog` 才编译进来）。结构化二进制，外部可解码成 Showdown 的文本协议 |
| **`Chance.Actions`** | `gen1/chance.zig` | 记录这次 `update` 里发生了哪些"RNG 事件"（命中了吗？爆击了吗？睡了几回合？）。开启 `-Dchance` 后可用，是 AI/MCTS 搜索的基础 |
| **`Calc`** | `gen1/calc.zig` | 允许"覆写"伤害结果用于伤害计算器场景 |
| **`Options`** | `common/options.zig` | 把 `log` / `chance` / `calc` 三个可选模块打包在一起，没启用时是 0 字节的 null 对象 |
| **`battle.options(...)`** | `common/battle.zig` | null object pattern：未启用的模块用 `NULL` 占位，编译器把对应代码完全消除 |

---

## 4. 招式 / 特性 / 道具的"效果"是怎么实现的？

这是你最关心的问题。引擎采用了一种**"枚举 + dispatch"**的模式，而不是 OOP 的"每个招式一个 class"或者 Showdown 那样"每个招式一个 JS 回调对象"。

### 4.1 招式的数据结构

每个招式是一行**4 字节的静态数据**：

```zig
// gen1/data/moves.zig
pub const Data = packed struct(u32) {
    effect: Effect,     // 1 字节：副作用类型枚举
    bp: u8,             // 威力
    accuracy: u8,       // 命中
    type: Type,         // 属性
    target: Target,     // 目标
};

// 示例：火焰喷射
.{ .effect = .BurnChance1, .bp = 95, .type = .Fire,
   .accuracy = percent(100), .target = .Other },
```

招式**没有自己的代码**。它只有一个 `effect` 字段，指向一个"效果种类"枚举：

```zig
pub const Effect = enum(u8) {
    None,
    Confusion, Conversion, FocusEnergy, Haze, Heal, LeechSeed,
    LightScreen, Mimic, Mist, Paralyze, Poison, Reflect, Splash,
    Substitute, SwitchAndTeleport, Transform,
    AccuracyDown1, AttackDown1, ...    // onEnd 类
    DrainHP, DreamEater, Explode, JumpKick, PayDay, Rage, Recoil,
    Binding, Charge, SpecialDamage, SuperFang, Swift, Thrashing,
    BurnChance1, BurnChance2, FlinchChance1, ParalyzeChance1, ...
    Disable, HighCritical, HyperBeam, Metronome, MirrorMove, OHKO,
};
```

**很多招式共用同一个 effect**——比如 "火焰喷射 / 火花 / 喷射火焰" 都是 `.BurnChance1`，区别只在威力/命中。这极大压缩了代码量。

### 4.2 effect 是怎么被触发的？

`Effect` 枚举值被**有意排序**：前 16 个是 "onBegin" 类（在伤害判定前生效，例如灼烧、变身），17–31 是 "onEnd" 类（命中后生效，例如能力升降），32–44 是"特殊伤害招式"，依此类推。这样能用一个**整数比较**就分类：

```zig
pub fn onBegin(effect: Effect) bool {
    return @intFromEnum(effect) > 0 and @intFromEnum(effect) <= 16;
}
```

在 `mechanics.zig` 的 `doMove` 流程里，到了相应阶段就会调用一个大 switch 分发：

```zig
fn moveEffect(battle, player, move, options) !void {
    return switch (move.effect) {
        .BurnChance1, .BurnChance2 => Effects.burnChance(battle, player, move, options),
        .ConfusionChance            => Effects.confusion(battle, player, move, options),
        .FlinchChance1, .FlinchChance2 => Effects.flinchChance(battle, player, move, options),
        .FreezeChance               => Effects.freezeChance(battle, player, move, options),
        .HyperBeam                  => Effects.hyperBeam(battle, player, options),
        .ParalyzeChance1, .ParalyzeChance2 => Effects.paralyzeChance(...),
        .PoisonChance1, .PoisonChance2 => Effects.poison(...),
        .AttackDownChance, .DefenseDownChance, .SpecialDownChance, .SpeedDownChance =>
            Effects.unboost(battle, player, move, options),
        else => {},
    };
}
```

`Effects` 是一个**全是函数的命名空间**（`pub const Effects = struct { fn bide(...){} fn burnChance(...){} ... }`）。每种效果一个函数，直接操纵 `battle/side/foe/active.volatiles` 等结构体字段。

### 4.3 道具（Gen 2 引入）

Gen 2 的 `Pokemon` 多了 `item: Item = .None` 字段。`Item` 也是 8-bit enum。道具的处理在 `gen2/mechanics.zig` 里的不同时机以同样的 switch dispatch 方式生效（例如：能力提升道具、Quick Claw 优先度、Leftovers 回血、Berry 解状态、King's Rock 退缩）。

### 4.4 特性（Abilities）

**Gen 1/2 没有特性概念，所以引擎暂时也没有这个抽象**。但根据这套设计，特性以后实现就会是：
- `Pokemon` 多一个 `ability: Ability = .None` 字段（1 字节 enum）；
- 在出场、出招前、被打中、回合末等钩子点上加 switch dispatch；
- 跟 Showdown 那种"事件订阅 + 优先度冒泡"系统**形成鲜明对比**——pkmn/engine 故意拒绝那种模式，因为它需要在每次事件都遍历所有可能的事件源去找处理器（这是 Showdown 性能瓶颈，见 `docs/DESIGN.md`）。

### 4.5 对比 Pokémon Showdown 的"通用事件系统"

| 方面 | Pokémon Showdown | pkmn/engine |
| --- | --- | --- |
| 每个招式 | 一个 JS 对象，带 `onTryHit`, `onModifyMove`, `secondary` 等钩子回调 | 一行 4 字节静态数据 + 一个 `Effect` enum |
| 派发方式 | 事件冒泡：遍历招式/精灵/场地/道具/特性查找处理器 | 在固定流水线节点上做一次 `switch` |
| 扩展性 | 极强，几小时就能加个新世代 | 极弱，每个世代都要手写——但因此每代都跑得飞快 |
| 性能特征 | 慢（动态分配 + map 查找 + 字符串）| 极快（数组下标 + 位字段 + 无分配）|

---

## 5. 整个引擎是怎么"跑"起来的？

无论用 C、TypeScript 还是 Zig 写驱动，调用模式**永远是同一个循环**：

```text
1. 构造 Battle 结构（填两队的种族、招式、HP、PRNG seed）
2. c1 = Pass, c2 = Pass           （第 0 回合特殊：表示"派出第一只"）
3. loop:
       result = battle.update(c1, c2, options)
       if result.type != None: break         ← 胜负已定
       legal_c1 = battle.choices(P1, result.p1)   ← 引擎告诉你能选什么
       legal_c2 = battle.choices(P2, result.p2)
       c1 = 由 AI/玩家/随机 从 legal_c1 里挑一个
       c2 = 同上
4. 根据 result.type 判定 Win/Lose/Tie
   （若开了 -Dlog，还要把 options.log 缓冲区里的二进制日志解码出来）
```

`Battle` 整个就是一段 384 字节的连续内存。要存档/读档？直接 `memcpy`。要并行跑 100 万局蒙特卡洛搜索？直接复制 384 字节，互不干扰。这就是"data-oriented design"的威力。

---

## 6. 三个层级的协作（实战示例）

下面用三个具体场景，看看不同抽象层级如何协作。

### 示例 A：用"火焰喷射"打中对面飞行系并附带烧伤

| 层级 | 谁负责 | 做了什么 |
| --- | --- | --- |
| **指令层** | 驱动代码 / 玩家 | 提交 `Choice{ .type = .Move, .data = 1 }`（招式槽 1） |
| **流程层** | `mechanics.update` | 决定先后手 → 进入 `doTurn` → `executeMove` |
| **出招前** | `beforeMove` | 检查麻痹/冰冻/混乱/退缩 → 都没有 → ok |
| **静态数据** | `Move.get(.Flamethrower)` | 取出 `{effect=BurnChance1, bp=95, type=Fire, accuracy=100%}`，4 字节查表 |
| **命中判定** | `moveHit` → `Rolls` (`chance.zig` + `rng.zig`) | 用 PRNG 投一次命中骰 |
| **伤害公式** | `calcDamage` | 卡带原版公式：攻/防、等级、STAB、暴击 |
| **属性相克** | `adjustDamage` + `types.zig` 相克表 | 火 vs 飞行 = 1×（普通效果）|
| **随机扰动** | `randomizeDamage` | 217–255/255 的伤害随机数 |
| **扣血** | `applyDamage` | 写入 `foe.active.stored().hp`，写入 `battle.last_damage` |
| **副作用 dispatch** | `moveEffect` switch | `.BurnChance1 → Effects.burnChance(...)` |
| **副作用执行** | `Effects.burnChance` | 检查替身、状态、属性免疫；用 PRNG 投一次副作用骰；通过 → 设置 `foe.stored().status = Status.init(.BRN)`，攻击半减 |
| **日志层** | `options.log.status(...)` | 若启用 `-Dlog`，写入二进制 protocol；否则编译期消除 |
| **回合末** | `handleResidual` | 烧伤会在下回合末扣 1/16 HP |

### 示例 B：玩家选择"换宝可梦"

| 层级 | 谁负责 | 做了什么 |
| --- | --- | --- |
| **可选指令查询** | `battle.choices(P1, .Switch, out)` | 遍历 `side.pokemon`，把所有 HP > 0 且不是自己的位置作为 `Choice{.Switch, slot}` 写入缓冲区 |
| **驱动选择** | 用户/AI | 从合法列表里挑一项 |
| **流程层** | `selectMove` → `executeMove` | 看到 `.Switch` 类型直接进入 `switchIn` 分支 |
| **状态清空** | `switchIn` → `clearVolatiles` | 把所有 `volatiles` 位字段清 0、boost 清 0 |
| **激活实体** | `switchIn` | 把目标 `Pokemon` 的字段拷贝到 `ActivePokemon`，调整 `side.order` |
| **状态修正** | `statusModify` | 若中毒/烧伤会按规则调整能力 |
| **日志** | `options.log.switch_(...)` | 输出 `|switch|` 协议消息（若启用） |

### 示例 C：触发"屯能(Bide)"——一个多回合持续效果

| 层级 | 谁负责 | 做了什么 |
| --- | --- | --- |
| **第 N 回合首次使用 Bide** | `Effects.bide` | 设置 `active.volatiles.Bide = true`、`volatiles.state = 0`（累计伤害）、`volatiles.attacks = 2 或 3`（持续回合，PRNG 决定） |
| **`Chance` 模块** | `options.chance.observe(.attacking, ...)` | 记录"我们这次抽到了 2 还是 3"——AI 搜索可以重放 |
| **后续回合 `selectMove`** | 检测到 `volatiles.Bide == true` | **不让玩家选指令**，强制重复 Bide |
| **被打中时** | `applyDamage` | 累加进 `volatiles.state` |
| **`volatiles.attacks` 计数 = 0** | `doMove` 的 Bide 释放分支 | 把累计伤害 ×2 砸向对面，清空 `Bide` 状态 |

这个例子非常清楚地展示了 **`Volatiles` 位字段 + `Effects` 函数 + 主流程检查**这三者如何形成一个"小型状态机"。

---

## 7. 关键文件 / 第一次读源码的建议路径

如果你想真的开始看代码，建议这个顺序：

1. **`docs/DESIGN.md`** — 设计理念
2. **`src/lib/gen1/README.md`** — Gen 1 的数据布局表
3. **`src/lib/gen1/data.zig`** — `Battle/Side/Pokemon/ActivePokemon/Volatiles` 结构体
4. **`examples/zig/example.zig`** — 看一个完整 100 行 demo 怎么用
5. **`src/lib/gen1/mechanics.zig`**：
   - `update` (line 53) — 入口
   - `executeMove` (line 383) — 出招主流水线
   - `doMove` (line 844) — 伤害与效果的核心
   - `moveEffect` (line 1803) + `Effects` 命名空间 — 所有招式效果实现
6. **`src/lib/gen1/data/moves.zig`** — 看 4 字节静态数据如何描述全部 165 个招式

---

## 8. 如何为"洛克王国"这类游戏写一个类似引擎？

如果你想给一款国产 PKM 风格对战游戏（比如《洛克王国》、《赛尔号》）做一个 pkmn/engine 级别的高性能对战内核，建议按以下顺序：

### 8.1 第 0 步：明确你的"游戏战斗规则"边界

写一份**规则文档**，详细列出（以洛克王国为例）：

- **战斗格式**：1v1？2v2？多人混战？
- **指令系统**：每回合每个玩家选什么（攻击招式、技能、换精灵、使用道具）？
- **属性 / 相克表**：火、水、草、电、土、机械、神秘…
- **能力值系统**：攻、防、特攻、特防、速度、HP？等级公式？性格？训练值？
- **招式属性**：物理 / 法术 / 变化？威力？命中？PP？目标？特效？优先度？
- **状态系统**：中毒、烧伤、麻痹、冰冻、睡眠、混乱、束缚…
- **特性 / 个性 / 套装** 系统是否存在？
- **道具 / 法宝 / 魂玉** 系统？
- **场地效果 / 天气 / 五行 / 缘分**？
- **RNG 哪里发生**（命中、暴击、副作用概率、伤害浮动）？

**先把规则枚举清楚，再设计数据结构**。这是 pkmn/engine 能写得这么小这么快的根本原因——它实现的是一个**完全冻结的、不再变化的、有限的规则集合**。

### 8.2 第 1 步：定义"最小完备"的状态结构

参照 `gen1/data.zig`，定义你的：

- `Battle`：整场战斗
- `Side`：一方
- `Pet`（精灵）：未出场存储态
- `ActivePet`：出场战斗态（boost / volatiles / 修改后能力值分离出来）
- `SkillSlot`：技能槽 `{id, pp}`
- `Volatiles`：用 **bitfield**（Zig 的 `packed struct`、C 的位域、Rust 的 `bitflags`、TS 的 number 位运算）表达"是否中毒 / 是否屯能 / 替身 HP / 混乱剩余回合数"等
- `Choice`：玩家可下的指令
- `Result`：update 返回值

**遵守的设计原则**（直接从 pkmn/engine 借鉴）：
1. 一场战斗的全部状态打包在一个**定长**结构里（可以 memcpy / 序列化）
2. **不用指针**——用整数下标当 "handle"
3. **不用字符串**——所有 ID 是 enum
4. **冷数据 / 热数据分离**——经常访问的字段放一起
5. **不动态分配**——所有缓冲区大小常量化（`CHOICES_SIZE`, `LOGS_SIZE`）

### 8.3 第 2 步：把"图鉴"做成代码生成的静态数据

- 用 Python / Node 写一个 `tools/generate` 脚本，从 JSON / Excel / 服务器导出读取**精灵图鉴 / 技能表 / 道具表**
- 生成成目标语言的源代码（Zig 的 `[_]Data{}` 数组、Rust 的 `static`、C 的 `const` 数组）
- **关键**：精灵和技能的 ID 就是数组下标，O(1) 查表，没有 hash map

### 8.4 第 3 步：实现主流水线

参照 `gen1/mechanics.zig` 的骨架：

```
update(c1, c2)
  ├─ selectMove(P1, c1)            ← 强制连续招处理
  ├─ selectMove(P2, c2)
  ├─ turnOrder(...)                 ← 速度+优先度
  └─ doTurn(first, second)
        ├─ executeMove(first)
        │     ├─ beforeMove           ← 麻痹/睡眠/混乱
        │     ├─ canMove              ← PP / Disable
        │     └─ doMove
        │           ├─ checkHit
        │           ├─ calcDamage
        │           ├─ adjustDamage   ← 相性 + STAB
        │           ├─ randomizeDamage
        │           ├─ applyDamage
        │           └─ moveEffect     ← ★ 副作用 dispatch
        │                 switch(effect) { ... Effects.xxx(...) ... }
        ├─ checkFaint
        ├─ executeMove(second)
        ├─ handleResidual              ← 中毒/烧伤/Leech 末伤
        └─ endTurn                     ← 写日志 + 返回 Result
```

### 8.5 第 4 步：实现"技能效果 / 道具效果 / 特性"

- 给每种"效果类别"定义一个 `Effect` enum 值（不是每个具体技能！多个相似技能共用同一个 effect）
- 在 `Effects` 命名空间里给每种 effect 写一个函数，直接操纵 `Battle` 状态
- 在主流水线的合适节点（onBegin / onHit / onEnd / onResidual）做 `switch (effect)` 分发

如果你的游戏有"特性 / 个性 / 缘分"等会**影响多种事件**的机制，有两种选择：
1. **像 pkmn/engine 一样硬编码**：在每个事件点的 switch 里加入"如果有 X 特性 → 改变行为"——快但代码丑
2. **像 Pokémon Showdown 一样建事件总线**：能力强、可扩展性高、但慢

选择取决于你的目标：服务器实时对战、AI 训练 → 选 1；玩家可自定义 mods → 选 2。

### 8.6 第 5 步：日志 / RNG / 测试

- **二进制 protocol**：定义一套字节码（`|switch|`、`|move|`、`|damage|`…）作为可选编译开关，关闭时编译器消除日志代码
- **RNG 抽象**：实现一个 `FixedRNG`，可强制指定每次抽签结果——这是单元测试和 AI 重放的基石
- **集成测试**：跑大量 fixture 战斗，对比"参考实现"（你的服务器现有代码）的逐回合输出
- **基准测试**：用 `zig build benchmark` 这样的工具持续监控性能回归

### 8.7 第 6 步：对外绑定

- **C ABI**：在 Zig/Rust/C++ 里导出 `pkmn_battle_update` 之类的 C 函数；不暴露内部结构，只暴露不透明 handle
- **Node / WASM**：用 N-API 或 WebAssembly 包一层，给前端用
- **驱动层**：用 TypeScript / Python 实现易用的高层 API（自动初始化、打印协议、AI 玩家接口）

---

## 9. TL;DR 要点回顾

1. **pkmn/engine ≠ 完整游戏**，它是一个为极致性能而生的"对战内核库"。
2. **每个世代独立硬编码**，不抽象成"通用引擎 + 数据"。
3. **核心实体只有 5 个**：`Battle / Side / Pokemon / ActivePokemon / MoveSlot`，加上若干位字段（`Volatiles`、`Status`）。
4. **招式没有自己的代码**，只有 4 字节数据 + 一个 `Effect` enum 标签，由 `mechanics.zig` 里的大 switch 分发到 `Effects.xxx` 函数。
5. **道具同理**（Gen 2 起），特性会以同样的模式加进去。
6. **整个 API 就一对函数**：`update(c1, c2) -> Result` 和 `choices(player, result) -> [Choice]`。
7. **二进制 / 无字符串 / 无分配 / 无指针**，是它比 Pokémon Showdown 快 1000 倍的根本原因。
8. **想做洛克王国版本**：先冻结规则 → 设计紧凑结构体 → 代码生成图鉴 → 实现固定流水线 → 用 enum + switch 实现效果 → 加 RNG 抽象与日志协议 → C ABI 暴露给上层。
