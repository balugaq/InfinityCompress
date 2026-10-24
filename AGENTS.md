# AGENTS.md

> 本文件面向在本仓库干活的 AI / 新 contributor。改代码前先读完「核心约定」和「注意事项」。

## 1. 项目是什么

InfinityCompress 是一个 **Slimefun4 附加插件**（Minecraft 服务端，Spigot/Paper，api-version 1.16）。
为 [InfinityExpansion](https://github.com/Mooy1/InfinityExpansion) 提供一大批**压缩版机器**，让无尽锭产线用更少的机器实体、更高单机效率来跑，缓解服务器卡顿。顺带：

- 为 [LiteXpansion](https://github.com/Slimefun-Addon-Community/LiteXpansion) 加自动 UU 机、自动金属锻造厂/磨坊/冶炼厂
- 为 [FN-FAL's Amplifications](https://github.com/FN-FAL113/FN-FAL-s-Amplifications) 加一批自动祭坛/组装站/回收站
- 为 [GlobalWarming](https://github.com/poma123/GlobalWarming) 加压缩空气压缩机

`groupId: io.github.acdeasdff`，`artifactId: InfinityCompress`，version 固定 `1`。

## 2. 构建

```bash
mvn clean package        # defaultGoal 即 clean package，直接 mvn 也行
```

产物：`target/InfinityCompress v1.jar`（注意 `finalName` 带空格和 `v1`）。

- 编译目标 **Java 8**（source/target 1.8），别顺手用 9+ 语法。
- 依赖四个**本地 jar**（`lib/` 下，`system` scope）：`InfinityExpansion.jar`、`GlobalWarming.jar`、`LiteXpansion.jar`、`FNAmplifications.jar`。克隆仓库后必须确认它们在位，否则编译直接失败。
- Slimefun4 依赖国人魔改版（jitpack `SlimefunGuguProject`，commit `b39097e015`），官方版大概率跑不通。
- shade 插件会把 `io.github.mooy1.infinitylib` 重定位到 `io.github.acdeasdff.infinityCompress.infinitylib` 再打进最终 jar，并 `minimizeJar`。

测试：JUnit 5 + MockBukkit-v1.19 已在 test scope 里，`mvn test` 可跑（目前基本没测试，加了是加分项）。

## 3. 代码结构

```
src/main/java/io/github/acdeasdff/infinityCompress/
├── InfinityCompress.java        # 主类，extends AbstractAddon(InfinityLib)
├── categories/
│   ├── Groups.java              # 所有 Slimefun ItemGroup（分类页）
│   └── InfinityGroup.java       # 自定义分类页（从 InfinityExpansion 拷的）
├── items/
│   ├── blocks/
│   │   ├── Blocks.java          # ⚠️ 2500+ 行：物品定义 + setup() 注册入口（ InfinityExpansion 侧）
│   │   ├── Blocks2.java         # 第二批物品定义 + setup()
│   │   ├── AutoInfinityWorkbench.java   # 自动无尽合成台（extends AutoCrafter）
│   │   ├── TweakedGenerator.java        # 压缩发电机
│   │   ├── TweakedMaterialGenerator.java# 压缩材料生成机
│   │   ├── TweakedGearTransformer.java  # 装备材质转换机
│   │   ├── TweakedGEOQuarry(_Filter).java# 压缩地理矿机 / 带过滤版
│   │   └── ...
│   ├── Multiblock_Autocrafter.java     # 多方块自动合成机基类（抄自 FluffyMachines）
│   ├── LiteXpansion/  AutoMetalForge / AutoMill / AutoSmeltry + BlocksLiteXpansion
│   ├── FNFALsAmplifications/  AutoAssemblyStation / Auto*Altar / AutoScrapRecycler + BlocksFN
│   └── GlobalWarming/  TweakedAirCompressor + BlocksGlobalWarming
├── AutoCrafter.java             # 自动合成机基类（抄自 FluffyMachines，36 格输入）
├── CraftingBlockRecipe.java
├── fixbug/                      # newInfinityWorkbench / newCraftingBlock(_Recipe)：上游类的补丁版拷贝
└── commands/  GiveRecipe / SetData / PrintItem   # SubCommand，挂在 /infinityCompress 下
```

注册流程：主类 `enable()` 依次调 `Groups.setup()` → `Blocks.setup()` → `Blocks2.setup()`，然后**软依赖探测**——只有 `Bukkit.getPluginManager().isPluginEnabled(...)` 为真才注册 GlobalWarming / LiteXpansion / FNAmplifications 的物品。新增软依赖物品务必沿用这个模式。

## 4. 核心约定

- **物品 ID 前缀**：所有 SlimefunItemStack id 以 `INFCP_` 开头（如 `INFCP_GEO_QUARRY_TIER1`）。新物品必须遵守，`config.yml` 的 `pollution.absorption.machines` 就是按这个 id 配的。
- **显示名/本地化**：没有 `language.properties`，中文显示名**硬编码**在 `SlimefunItemStack` 构造里（`&f`/`&7` 颜色码 + `MachineLore.energy(x)` / `energyPerSecond(x)` / `energyBuffer(x)`  lore 助手）。要支持其他语言只能逐个物品改。
- **分级机器**：Tier1–Tier7 逐级 5 倍能耗（矿机 3600 → 11,520,000 J/t），加新 Tier 时延续这个节奏。
- **Lombok**：`lombok.config` 开启 fluent accessor（无 `get` 前缀）、chain setter、jsr305 空注解、stopBubbling。读代码时注意 `charge()` 就是 `getCharge()`。
- **GUI/存档**：GUI 用 `BlockMenuPreset` + `DirtyChestMenu`；机器开关等状态存 `BlockStorage.addBlockInfo(block, key, value)`，**键名小写驼峰**（`enabled`、`recipeLocked`）。
- **电力**：机器实现 `EnergyNetComponent`，`getEnergyComponentType()` 返回 `CONSUMER`；tick 逻辑挂在 `BlockTicker`（`isSynchronized()` 视情况），耗电常量 `ENERGY_CONSUMPTION`、容量 `CAPACITY = ENERGY_CONSUMPTION * 3`。
- **物品运输**：`getSlotsAccessedByItemTransport` 区分 WITHDRAW（输出槽）/ INSERT（输入槽），AutoCrafter 里还有按存量排序分配的 `compareSlots` 逻辑，改动前先读懂。
- **命令权限**：节点形如 `infinitycompress.setdata` / `.printitem` / `.giverecipe`，默认 op。
- **上游依赖改动的处理方式**：需要改上游类行为时，拷一份到 `fixbug/` 包改名 `newXxx` 再改（已有 `newInfinityWorkbench` 等先例），不要直接反射 hack。

## 5. 注意事项 / 坑

- `plugin.yml` 里 `website: 127.0.0.1` 是占位符，`version: ${project.version}` 靠 maven filtering 注入，改版本号改 pom 而不是这里。
- `AutoCrafter.INFWB_TYPE` 是**实例字段**且每台机器各自持一份配方表，注册机器多时注意内存；`recipes` 字段目前声明了但主流程未用（历史遗留，清理前先搜引用）。
- 大量注释掉的死代码（`//        Storage.setup(this);`、`// MobData.setup(this);` 等）是作者刻意留的开关，**别当垃圾删**，拿不准先问。
- 本 README 用国人魔改 Slimefun 编译，换环境复现 bug 时先确认对方 Slimefun 版本。
- 提交权归主人：AI 只把工作区留到可通过验收的状态，不自行 commit / push。

## 6. 验收口径

改完至少做到：

1. `mvn clean package` 全绿，产物在 `target/InfinityCompress v1.jar`；
2. 新增/改动的物品 ID 带 `INFCP_` 前缀，显示名中文，lore 用 `MachineLore` 助手；
3. 软依赖物品仍走 `isPluginEnabled` 探测注册；
4. 涉及 GUI 槽位改动时，同步检查 `getSlotsAccessedByItemTransport` 和破坏时的 `dropItems` 槽位数组。
