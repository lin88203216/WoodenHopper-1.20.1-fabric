# 1.20.1 Fabric 移植说明

上游：[cech12/WoodenHopper](https://github.com/cech12/WoodenHopper) 的 `1.20.4` 分支（`1.20.4-2.2.2.0`）。
本目录是**只保留 Fabric** 的 1.20.1 回移版本。

## 改了什么

| 文件 | 改动 |
|---|---|
| `settings.gradle` | 删除 `forge` / `neoforge` 两个 `include`，以及两个加载器的 plugin 仓库；只留 Fabric + Sponge |
| `build.gradle` | 删除 NeoGradle 必需的 `org.jetbrains.gradle.plugin.idea-ext` 插件；`expandProps` 里删掉 `forge_*` / `neoforge_*` / `minecraft_version_range`；`filesMatching` 去掉 `META-INF/mods.toml` |
| `gradle.properties` | `minecraft_version=1.20.1`、`fabric_version=0.92.12+1.20.1`、`cloth_config_version=11.1.136`、`mod_menu_version=7.2.2`；删掉 forge/neoforge 段 |
| `fabric/src/main/resources/fabric.mod.json` | `"minecraft": ">=${minecraft_version} <1.20.2"`（原来只有 `>=`，会把 1.20.2+ 也放进来，而 1.20.2 起 `FriendlyByteBuf` 改名、`renderBackground` 变 4 参，二进制不兼容） |
| `common/.../client/WoodenHopperScreen.java` | `renderBackground(guiGraphics, mouseX, mouseY, partialTicks)` → `renderBackground(guiGraphics)` |
| `README.md` / `CHANGELOG.md` | 标注 Fabric-only 与 1.20.1 版本线 |
| 删除 | `forge/`、`neoforge/`、`.github/`（CI 引用了多加载器发布流程） |

**唯一的源码改动是 `renderBackground`。** 其余 Java 代码一行没动。

## 为什么只有这一处

用 1.20.1 官方 Mojang 客户端映射（`client.txt`，104049 行）逐项核对过：

- 把工程里全部 55 个 `import net.minecraft.*` 拿去 1.20.1 映射里查，**全部存在**，没有一个类缺失。
- 把源码里所有 `.method(` 调用名与 1.20.1 映射比对，再与 1.20.4 映射求差集（即"1.20.2+ 才有的方法名"），命中的只有 `getBlockPos`——这是正则漏匹配造成的假阳性（1.20.1 映射里它是 `160:160:net.minecraft.core.BlockPos getBlockPos()`，确实存在）。
- `Screen.renderBackground` 在 1.20.1 只有 `renderBackground(GuiGraphics)`（`Screen` 的 331:336 行）；4 参重载是 1.20.2 才加的，1.20.4 映射里才有。作者自己的 1.20.1 分支写的就是 `this.renderBackground(guiGraphics)`。

之前担心的两个点，实测**都不是问题**：

- `ContainerEntity`：`net.minecraft.world.entity.vehicle.ContainerEntity` 在 1.20.1 就有（还有 `ContainerEntity$1`），yarn 里叫 `VehicleInventory`。所以 `entity instanceof ContainerEntity` 与 `InventoryStorage.of((ContainerEntity) entity, side)` 原样编译。
- `ItemStack.isSameItem(ItemStack, ItemStack)`：1.20.1 映射 `463:463` 行就有，`isSameItemSameTags` 在 `467:473`。不用改。

Fabric API 侧同样核对过（1.20.1 分支源码）：

- `ExtendedScreenHandlerFactory.writeScreenOpeningData(ServerPlayerEntity, PacketByteBuf)` —— yarn 的 `PacketByteBuf` 就是 Mojang 的 `FriendlyByteBuf`，现有重写签名成立。
- `ExtendedScreenHandlerType(ExtendedFactory<T>)` 存在（`create` 方法标了 `@Deprecated`，warning 而已）。
- `ItemStorage.SIDED`、`StorageUtil.simulateExtract`、`Storage.nonEmptyViews()`、`Transaction.openOuter()/commit()/abort()`、`ItemVariant.of(ItemStack)`、`InventoryStorage.of(Container, Direction)` 全部在 1.20.1 的 `fabric-transfer-api-v1` 里。
- `ItemStorage.SIDED` 在 1.20.1 自带 vanilla 兜底（`InventoryProvider` 与实现 `Inventory` 的方块实体都会被包成 `InventoryStorage`），所以从普通箱子抽取这条主路径不会因为版本下降而失效。

数据包/资源包零迁移：`pack.mcmeta` 的 `pack_format 15` 正是 1.20.1，`recipes/`、`loot_tables/`、`tags/blocks/`、`advancements/` 路径和函数名在 1.20.1 一致，`c:wooden_chests` 标签在 1.20.1 的 Fabric convention 里同名。

## 构建与验证

```bash
cd WoodenHopper-1.20.1-fabric
./gradlew :fabric:build          # 产物 fabric/build/libs/woodenhopper-fabric-1.20.1-2.2.2.0.jar
./gradlew :fabric:runClient      # 起客户端，检查配置界面
./gradlew :fabric:runServer      # 起服务端，确认无 NoClassDefFoundError
```

进游戏逐项验证：

1. 合成：`W W / W C W / W `（C = 任意 `c:wooden_chests`，即箱子/陷阱箱）。
2. 放置后 16 tick 才动一次（比原版漏斗慢一倍）。
3. 上方放箱子 → 能抽取；上方丢物品 → 能吸取；下方放箱子 → 能推出。
4. 两个木漏斗串联 → 下游空容器时 cooldown 会被设成 `16 - k`，不会卡住。
5. Cloth Config 三个开关（cooldown / 从世界吸 / 从容器抽）改动后立即生效；ModMenu 里能打开配置页。
6. 破坏方块 → 内容物掉落，自定义名保留。

## 依赖下限的实跑修正

最初照抄上游的 `fabric_loader_version=0.15.7`，冒烟测试直接失败两次，逐层暴露：

1. `fabric-api 0.92.12+1.20.1` 里的 `fabric-tag-api-v1 1.3.0` 要求 loader **≥0.15.11** → 提到 0.15.11。
2. 再跑，`fabric-api 0.92.12+1.20.1` 与 `fabric-object-builder-api-v1 11.1.5` 要求 loader **≥0.16.10** → 最终定为 **0.16.10**。

所以 `fabric.mod.json` 现在声明 `fabricloader: ">=0.16.10"`。这个下限不是猜的，是 loader 在启动时用 `HARD_DEP` 报出来的。

## 服务端冒烟测试：被 Android 环境挡住，非模组问题

`:fabric:runServer` 在加载阶段成功列出 `Loading 49 mods: ... - woodenhopper 1.20.1-2.2.2.0`，随后崩溃在：

```
java.lang.NoClassDefFoundError: Could not initialize class com.sun.jna.Native
  at oshi.software.os.linux.LinuxOperatingSystem.<clinit>
  at net.minecraft.SystemReport.putHardware(SystemReport.java:75)
  at net.minecraft.CrashReport.preload(CrashReport.java:274)
  at net.minecraft.server.Main.main(Main.java:102)
```

这是 MC 启动时预加载崩溃报告、经 oshi/JNA 读取硬件信息，在 Android 上 JNA 原生库载不进来导致的，与模组代码无关（崩溃点在模组初始化之前，日志里模组本身无任何报错）。结论：**服务端验证在这台设备上做不了，运行时验证只能走 FCL 客户端**。

## 本机构建结果（实跑，非静态推断）

在 Android/Termux（aarch64，JDK 17.0.21，Gradle 8.7，Loom 1.6.12）上实际跑通：

```
> Task :common:compileJava      OK   ← renderBackground 改动编译通过
> Task :fabric:compileJava      OK   ← ContainerEntity / isSameItem / Transfer API 全部通过
> Task :fabric:jar              OK
BUILD SUCCESSFUL in 13m 7s      (15 actionable tasks)
产物：fabric/build/libs/woodenhopper-fabric-1.20.1-2.2.2.0.jar  (51 KB, 19 个 class)
```

产物里的 `fabric.mod.json` 已展开：`version=1.20.1-2.2.2.0`、`minecraft=">=1.20.1 <1.20.2"`、`fabric-api=">=0.92.12+1.20.1"`、`cloth-config=">=11.1.136"`。class 文件里是 intermediary 名（`class_XXXX`），这是 Fabric 生产 jar 的正常形态。

本机构建需要的三个环境补丁（**仅供复现，不属于工程内容**）：

1. `GRADLE_USER_HOME` 显式指向可写目录（JDK 把 `user.home` 写死成不存在的 `com.termux` 前缀）。
2. `-Djava.io.tmpdir=$PREFIX/tmp`（同理，默认 tmpdir 指向不存在的路径）。
3. JDK 的 `lib/security/cacerts` 是断链，需自建信任库并用 `-Djavax.net.ssl.trustStore=...` 指过去。

## 已知限制

- 只做了 `build`，没有跑 `runClient`（无显示）与 `runServer` 冒烟测试。
- 保留 `common` + `fabric` 的 Multiloader 骨架（ServiceLoader 三件套），没有拍平成单模块——这样以后跟上游 1.20.4 分支 diff 时改动面最小。
- Loom 仍是 `[1.6.6,1.7)`、Gradle wrapper 8.7、Java 17；这套组合对 1.20.1 可用，但如果你更想要 1.20.1 时代的稳定组合，可以把 Loom 降到 `[1.5,1.6)`。
