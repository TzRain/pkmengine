# 研究资料（中文）

> 原文：[`docs/RESEARCH.md`](../RESEARCH.md)
>
> 这份文档汇总了实现 `pkmn/engine` 时所依赖的"事实之源"——卡带反编译、随机数研究、漏洞研究，以及其它已有的对战引擎实现。它本质上是一份**外部资源索引**，所以中文版主要补充每一类资源"读它能解决什么问题、应该怎么用"的说明，链接则尽量保留原样。

## 为什么需要这些研究资料？

`pkmn/engine` 是 "**bug-for-bug compatible**"（行为完全等价，包括 bug）的实现。这意味着任何一个看似奇怪的行为——比如某个状态在某个时点不会失效、某个 PP 边界条件下会出现的小故障——都必须能在**原始卡带的反编译代码**里找到出处。否则就只能两眼一抹黑地猜。

因此，本文档下面列的所有链接，都不是"参考资料"那种意义上的可选项，而是**实现时必须查阅的一手材料**。每一行写到 `mechanics.zig` 里的逻辑，理想情况下都应该有一条注释指向 pret 反编译里的具体行号。

---

## Games（游戏卡带反编译）

### [Gen 1（红蓝）](https://github.com/pret/pokered/) — pokered

第一代游戏（Pokémon Red/Blue/Yellow）的 [pret 反编译项目](https://github.com/pret/pokered)。下面分目录列出关键文件，**按"我要找什么"的角度组织**。

#### `constants/`（常量定义）

- [`battle_constants.asm`](https://pkmn.cc/pokered/constants/battle_constants.asm)
  - [战斗状态（volatiles）](https://pkmn.cc/pokered/constants/battle_constants.asm#L73-L100) —— 临时状态位的定义（混乱、束缚、Bide 等），决定了 pkmn 里 `ActivePokemon.volatiles` 字段的布局。
- [`move_constants.asm`](https://pkmn.cc/pokered/constants/move_constants.asm) / [`move_effect_constants.asm`](https://pkmn.cc/pokered/constants/move_effect_constants.asm) —— 招式 ID、招式效果 ID。后者是"招式效果 → handler 指针"的间接索引，对应 pkmn 中 `Effects` 大 switch。
- [`pokedex_constants.asm`](https://pkmn.cc/pokered/constants/pokedex_constants.asm) / [`pokemon_data_constants.asm`](https://pkmn.cc/pokered/constants/pokemon_data_constants.asm)
- [`type_constants.asm`](https://pkmn.cc/pokered/constants/type_constants.asm)

#### `data/`（数据表）

- [`battle/`](https://github.com/pret/pokered/tree/master/data/battle)
  - [`stat_modifiers.asm`](https://pkmn.cc/pokered/data/battle/stat_modifiers.asm) —— 能力等级 → 倍率（卡带里写死的查表）。
  - "Special moves"：[`residual_effects_1.asm`](https://pkmn.cc/pokered/data/battle/residual_effects_1.asm)、[`residual_effects_2.asm`](https://pkmn.cc/pokered/data/battle/residual_effects_2.asm) —— 列出哪些效果在结算后还要触发"残留"逻辑。
- [`moves/`](https://github.com/pret/pokered/tree/master/data/moves)
  - [招式数据](https://pkmn.cc/pokered/data/moves/moves.asm)：威力、命中、PP、效果 ID。
- [`pokemon/`](https://github.com/pret/pokered/tree/master/data/pokemon)
  - [种族数据](https://github.com/pret/pokered/tree/master/data/pokemon/base_stats) —— 每个种族一个 `.asm` 文件。
  - [学习招式表](https://pkmn.cc/pokered/data/pokemon/evos_moves.asm)。
- [`types/`](https://github.com/pret/pokered/tree/master/data/types)
  - [`type_matchups.asm`](https://pkmn.cc/pokered/data/types/type_matchups.asm) —— **属性相克表**。注意卡带是**线性扫描查找**而非二维数组：[扫描代码](https://pkmn.cc/pokered/engine/battle/core.asm#L5230-L5289)。pkmn 选择直接预生成一张 17×17 的查找表（详见 [`DESIGN_zh.md`](./DESIGN_zh.md) 中"perfect hashing"一节）。

#### [`engine/battle/`](https://pkmn.cc/pokered/engine/battle)

- [`move_effects/`](https://github.com/pret/pokered/tree/master/engine/battle/move_effects) —— 每个特殊招式的 handler，类似 Pokémon Showdown 里的 `statuses` / `scripts` 目录。
- [**`core.asm`**](https://pkmn.cc/pokered/engine/battle/core.asm) —— 战斗主循环。**这是整个 Gen 1 战斗逻辑的中枢**，pkmn 的 `gen1/mechanics.zig` 基本就是这个文件的"现代复刻"。
- [`decrement_pp.asm`](https://pkmn.cc/pokered/engine/battle/decrement_pp.asm)
- [`effects.asm`](https://pkmn.cc/pokered/engine/battle/effects.asm)

#### `wram.asm`（工作 RAM 布局）

- [宏定义](https://pkmn.cc/pokered/macros/ram.asm)
- [未修改的种族值与修正](https://pkmn.cc/pokered/ram/wram.asm#L525)
- [当前招式](https://pkmn.cc/pokered/ram/wram.asm#L1148)
- [战斗数据](https://pkmn.cc/pokered/ram/wram.asm#L1206)
  - [战斗状态（volatiles）](https://pkmn.cc/pokered/ram/wram.asm#L1253-L1276)

> 💡 **阅读建议**：先把 `core.asm` 里的"turn loop"读两遍，再回过头看 `data/` 里的查表，最后才看具体 move handler。这样你对"哪些字段在 turn 内会变、哪些是常量"会有非常清晰的感觉。

### [Gen 2（金银水晶）](https://github.com/pret/pokecrystal/) — pokecrystal

- `constants/`
  - [`battle_constants.asm`](https://pkmn.cc/pokecrystal/constants/battle_constants.asm)
  - [`item_constants.asm`](https://pkmn.cc/pokecrystal/constants/item_constants.asm) / [`item_data_constants.asm`](https://pkmn.cc/pokecrystal/constants/item_data_constants.asm#L61-L135) —— Gen 2 引入了持有道具（item），这两个文件定义了道具 ID 和属性。
  - [`move_constants.asm`](https://pkmn.cc/pokecrystal/constants/move_constants.asm) / [`move_effect_constants.asm`](https://pkmn.cc/pokecrystal/constants/move_effect_constants.asm)
  - [`pokemon_constants.asm`](https://pkmn.cc/pokecrystal/constants/pokemon_constants.asm)
  - [`type_constants.asm`](https://pkmn.cc/pokecrystal/constants/type_constants.asm)
- `data/`
  - [`battle/`](https://github.com/pret/pokecrystal/tree/master/data/battle)
    - [命中率倍率](https://pkmn.cc/pokecrystal/data/battle/accuracy_multipliers.asm) / [击中要害几率](https://pkmn.cc/pokecrystal/data/battle/critical_hit_chances.asm) / [能力倍率](https://pkmn.cc/pokecrystal/data/battle/stat_multipliers.asm) / [天气修正](https://pkmn.cc/pokecrystal/data/battle/weather_modifiers.asm)
  - [`items/`](https://github.com/pret/pokecrystal/tree/master/data/items)
    - [道具属性表](https://pkmn.cc/pokecrystal/data/items/attributes.asm)
  - [`moves/`](https://github.com/pret/pokecrystal/tree/master/data/moves)
    - [招式数据](https://pkmn.cc/pokecrystal/data/moves/moves.asm)
    - [**effects**](https://pkmn.cc/pokecrystal/data/moves/effects.asm)（[指针表](https://pkmn.cc/pokecrystal/data/moves/effects_pointers.asm)、[优先级](https://pkmn.cc/pokecrystal/data/moves/effects_priorities.asm)） —— Gen 2 把招式效果编成了一种"小型字节码"，每个效果是一串"指令序列"。pkmn 实现里没有保留这种字节码结构，而是直接以 Zig 函数实现，但行为要等价。
    - 特殊威力计算：[Flail](https://pkmn.cc/pokecrystal/data/moves/flail_reversal_power.asm) / [Magnitude](https://pkmn.cc/pokecrystal/data/moves/magnitude_power.asm) / [Present](https://pkmn.cc/pokecrystal/data/moves/present_power.asm) / [Metronome 例外](https://pkmn.cc/pokecrystal/data/moves/metronome_exception_moves.asm) / [Hidden Power](https://pkmn.cc/pokecrystal/engine/battle/hidden_power.asm)
  - [`pokemon/`](https://github.com/pret/pokecrystal/tree/master/data/pokemon)
    - [种族数据](https://github.com/pret/pokecrystal/tree/master/data/pokemon/base_stats)
    - 学习招式：[升级习得](https://pkmn.cc/pokecrystal/data/pokemon/evos_attacks.asm) / [蛋招式](https://pkmn.cc/pokecrystal/data/pokemon/egg_moves.asm)
  - [`types/`](https://github.com/pret/pokecrystal/tree/master/data/types)
    - [`type_matchups.asm`](https://pkmn.cc/pokecrystal/data/types/type_matchups.asm)
    - [`type_boost_items.asm`](https://pkmn.cc/pokecrystal/data/types/type_boost_items.asm) —— 加成属性威力的道具（如咒符等）。
- `engine/battle/`
  - [`move_effects/`](https://github.com/pret/pokecrystal/tree/master/engine/battle/move_effects)
  - [**`core.asm`**](https://github.com/pret/pokecrystal/tree/master/engine/battle/core.asm)
  - [`effect_commands.asm`](https://pkmn.cc/pokecrystal/engine/battle/effect_commands.asm) —— "字节码指令集"的实现。
  - [`/home/battle_vars.asm`](https://pkmn.cc/pokecrystal/home/battle_vars.asm) —— 战斗变量的统一访问接口。
- `wram.asm`
  - [战斗数据](https://pkmn.cc/pokecrystal/ram/wram.asm#L352-L621)、[宏](https://pkmn.cc/pokecrystal/macros/ram.asm)
- 随机数：[`Random`](https://pkmn.cc/pokecrystal/home/random.asm)（全局 RNG）、[`BattleRandom`](https://pkmn.cc/pokecrystal/engine/battle/core.asm#L6881-L6947)（战斗专用 RNG）

> ⚠️ Gen 2 的两个 RNG 是**分开的**（一个全局、一个战斗内），这点在写 `chance.zig` 时需要特别注意。

### [Gen 3（绿宝石）](https://github.com/pret/pokeemerald/) — pokeemerald

Gen 3 切到了 GBA，反编译以 C 语言重写。结构变化较大：

- `include/`
  - [`battle.h`](https://pkmn.cc/pokeemerald/include/battle.h)（[`battle_util.h`](https://pkmn.cc/pokeemerald/include/battle_util.h) / [`battle_scripts.h`](https://pkmn.cc/pokeemerald/include/battle_scripts.h) / [`battle_main.h`](https://pkmn.cc/pokeemerald/include/battle_main.h)）
  - [`random.h`](https://pkmn.cc/pokeemerald/include/random.h)
  - [`pokemon.h`](https://pkmn.cc/pokeemerald/include/pokemon.h#L160-L241)
  - [`item.h`](https://pkmn.cc/pokeemerald/include/item.h)
- `include/constants/`
  - [`abilities`](https://pkmn.cc/pokeemerald/include/constants/abilities.h) —— Gen 3 引入"特性 ability"。
  - [`species`](https://pkmn.cc/pokeemerald/include/constants/species.h)
  - [属性 / 性格](https://pkmn.cc/pokeemerald/include/constants/pokemon.h)
  - [招式](https://pkmn.cc/pokeemerald/include/constants/moves.h) / [招式效果](https://pkmn.cc/pokeemerald/include/constants/battle_move_effects.h)
  - [道具持有效果](https://pkmn.cc/pokeemerald/include/constants/hold_effects.h)
  - [战斗常量](https://pkmn.cc/pokeemerald/include/constants/battle.h) / [战斗脚本指令](https://pkmn.cc/pokeemerald/include/constants/battle_script_commands.h)
- `src/data/`
  - [种族信息](https://pkmn.cc/pokeemerald/src/data/pokemon/species_info.h)
    - [进化链](https://pkmn.cc/pokeemerald/src/data/pokemon/evolution.h) / [蛋招式](https://pkmn.cc/pokeemerald/src/data/pokemon/egg_moves.h) / [升级习得](https://pkmn.cc/pokeemerald/src/data/pokemon/level_up_learnsets.h)
  - [招式数据](https://pkmn.cc/pokeemerald/src/data/battle_moves.h)
  - [道具数据](https://pkmn.cc/pokeemerald/src/data/items.h)
- `src/`
  - 战斗：[`battle_main.c`](https://pkmn.cc/pokeemerald/src/battle_main.c) / [`battle_util.c`](https://pkmn.cc/pokeemerald/src/battle_util.c)
  - [`pokemon.c`](https://pkmn.cc/pokeemerald/src/pokemon.c)
  - [`random.c`](https://pkmn.cc/pokeemerald/src/random.c)
- [`data/battle_scripts_1.s`](https://pkmn.cc/pokeemerald/data/battle_scripts_1.s) —— Gen 3 把招式效果继续用"脚本字节码"实现（更复杂的版本）。

### [Gen 4（钻石/珍珠）](https://github.com/pret/pokediamond) — pokediamond

TODO（pret 项目自身仍在进行中；pkmn/engine 目前也尚未实现 Gen 3/4，因此暂未整理详细索引）。

---

## Appendix（附录）

### Engines（其它已有的对战引擎实现）

逛逛它们的源代码对设计自己的引擎很有启发：

- [**Pokémon Showdown!**](https://github.com/smogon/pokemon-showdown) —— 当今的事实标准。文档：
  - [新文档](https://gist.github.com/scheibo/c9ef943ef6e01e350940c8429c378e3b)
  - [当前文档](https://raw.githubusercontent.com/smogon/pokemon-showdown/master/simulator-doc.txt)
  - [旧文档](https://raw.githubusercontent.com/smogon/pokemon-showdown/master/old-simulator-doc.txt)
- [PokemonBattleEngine](https://github.com/Kermalis/PokemonBattleEngine) —— C# 实现。
- [pokebattle](https://github.com/sarenji/pokebattle-sim) —— CoffeeScript（早期社区引擎，已停更）。
- [Pokemon Online](https://github.com/po-devs/pokemon-online) —— C++ 实现，文档：[RBY 行为](https://raw.githubusercontent.com/po-devs/pokemon-online/master/bin/database/rby-stuff.txt)。
- [PokemonLab / Shoddy Battle](https://github.com/cathyjf/PokemonLab) —— Java，更早的尝试。
- [poke-engine](https://github.com/pmariglia/poke-engine)（也叫 [poke-engine](https://github.com/SirSkaro/poke-engine)） —— Rust，AI 友好。
- [Pokemon Battle Simulator](https://github.com/hiimvincent/poke-battle-sim) —— Python。
- [PokeSim](https://github.com/aed3/poke-sim) —— C++。
- [ninjax](https://github.com/ndarwin314/ninjax/tree/master)

### RNG（随机数生成）

随机数是宝可梦对战引擎里**最容易出错**的部分。每一代 RNG 算法都不同；Pokémon Showdown 把 Gen V/VI 的 RNG 错用到了所有世代上。强烈建议在开始写一个新世代之前，先把下面这些读完：

- [Pseudorandom number generation in Pokémon](https://bulbapedia.bulbagarden.net/wiki/Pseudorandom_number_generation_in_Pokémon) —— Bulbapedia 上的总览，每代的 RNG 一览无遗。
- [`Admiral-Fish/PokeFinder` RNG 实现](https://github.com/Admiral-Fish/PokeFinder/tree/master/Source/Core/RNG) —— C++ 实现，覆盖所有世代。
- [Pokémon Gen 1 RNG Mechanics](https://glitchcity.wiki/Luck_manipulation_(Generation_I)#Mechanics_of_the_RNG) —— Gen 1 的 RNG 细节，对实现 `MAX_LOGS` 的 Z3 模型也有用（见 [`PROTOCOL_zh.md`](./PROTOCOL_zh.md#size--日志大小)）。
- [Pokémon Yellow DSUM Manipulation](http://wiki.pokemonspeedruns.com/index.php/Pokémon_Red/Blue/Yellow_DSum_Manipulation) —— speedrun 社区的"DSUM"操作手册，对理解卡带 RNG 的状态推进很有帮助。

### Glitches（漏洞汇总）

实现"bug-for-bug compatible"必须知道这些漏洞列表：

- [Gen 1 漏洞列表](https://bulbapedia.bulbagarden.net/wiki/List_of_glitches_(Generation_I))
- [Pokémon Crystal - Bugs & Glitches](https://pkmn.cc/pokecrystal/docs/bugs_and_glitches.md) —— 这是 pret 项目里自己整理的"我们知道但故意不修"的漏洞清单。
- [Red/Blue UE 错误汇总](https://sites.google.com/site/crystalglitchystuff/research/compilation-of-red-blue-eu-errors) —— Crystal_ 整理。
- [Pokémon Showdown RBY Bugs](https://www.smogon.com/forums/posts/5933177/show) —— 在 Showdown 上观察到、但与卡带不一致的行为。

### Other（其它技术资料）

- [Technical Machine](https://github.com/davidstone/technical-machine) —— david stone 写的另一个 AI 友好引擎。
- [Gen I Main Battle Function](https://www.smogon.com/forums/posts/5878612/show) —— Crystal_ 对 Gen 1 主战斗函数的逐句解读。
- [the ultimate POKéMON CENTER](https://web.archive.org/web/20170622160244/http:/upcarchive.playker.info/0/upokecenter/content/pokemon-ruby-version-sapphire-version-and-emerald-version-timing-notes.html) —— Peter O 写的 Gen 3 时序笔记。
- [Gen II Move effect handling flow](https://gist.github.com/scheibo/700be8fbfe7349f564b35dba376f0ef3) —— Gen 2 招式效果字节码的流程图。
- [Pokémon Showdown 协议按世代](https://gist.github.com/scheibo/f36e17700a2a7bcca7b86a4350843159) —— PS 的文本协议在各代有哪些差异。

### Benchmarks（基准测试方法学）

如果你要给 `pkmn/engine` 跑性能数据、或对比它和其它引擎，下面这些是行业常用工具：

- **环境搭建：**
  - [Perf measurement environment on Linux](https://easyperf.net/blog/2019/08/02/Perf-measurement-environment-on-Linux) —— 关闭 turbo / 锁频 / 隔离 CPU 的实战指南。
  - [LLVM Benchmarking tips](https://llvm.org/docs/Benchmarking.html)
  - [Performance Tracking for Zig](https://github.com/ziglang/gotta-go-fast) —— Zig 官方的性能跟踪基础设施。
  - [`uarch_bench.sh`](https://github.com/travisdowns/uarch-bench/blob/master/uarch-bench.sh)
- **工具：**
  - [google/benchmark](https://github.com/google/benchmark) —— C++ micro benchmark 框架。
  - [Benchmark.js](https://benchmarkjs.com/) —— JS 端，注意 V8 warmup。
  - [`hyperfine`](https://github.com/sharkdp/hyperfine) —— 终端命令的端到端计时。
  - [`perf`](https://perf.wiki.kernel.org/index.php/Main_Page) —— Linux 上看 CPU 计数器、缓存命中率、分支预测等。

> 关于 `pkmn/engine` 自己的 benchmark 设计原则，详见 [`TESTING_zh.md`](./TESTING_zh.md) 的 "Benchmark / 基准测试" 一节。
