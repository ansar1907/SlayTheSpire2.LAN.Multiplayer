# STS2 Mod 版本升级避坑指南

针对 STS2 Early Access 阶段频繁发版的特点，总结以下通用原则。

## 原则一：始终针对目标版本的 `sts2.dll` 编译

### 问题
模组引用的 `sts2.dll` 版本与实际游戏版本不匹配，导致 API 缺失、签名变化或行为异常。

### 做法
- 在 `.csproj` 中使用本地游戏目录的 `sts2.dll` 作为直接引用，不要 NuGet 包或固定路径
- 每个版本升级时，将 `Sts2DataDir` 指向对应当前游戏版本的 `data_sts2_<platform>_<arch>` 目录
- 用条件 PropertyGroup 区分平台，避免跨平台时引用错文件
- 让 `Sts2Path` 可通过环境变量或命令行参数覆盖，便于 CI 或他人使用

### 应避免
- 硬编码不可移植的绝对路径
- 引用与游戏版本无关的独立副本 `sts2.dll`

---

## 原则二：接口新增成员是最高频断裂点

### 表现
编译错误 `CS0535` — 类型未实现接口的全部成员。

### 常见接口
- `INetMessage` / `IPacketSerializable` — 网络序列化
- `INetGameService` — 网络游戏服务
- `IMessageHandler` 系列

### 做法
- 升级后优先 grep 项目中实现的所有 `MegaCrit.Sts2.*` 接口
- 对照新 `sts2.dll` 中接口定义的属性与方法，补齐所有缺失成员
- 新成员通常有合理的默认值（如 `ShouldBuffer => false`）

---

## 原则三：Harmony Patch 目标方法可能被移除或重命名

### 表现
编译错误 `CS0117` — 类型中找不到指定名称的成员。

### 做法
- 列出项目中所有 `[HarmonyPatch]` 和 `AccessTools.Method` 的目标
- 用反编译工具检查新 `sts2.dll` 中这些方法是否仍存在
- 如果方法已被移除，寻找语义等价的替代方法（通常有相似的命名模式）

### 更安全的补丁写法
- 用 `[HarmonyPatch]` + `TargetMethods()` 配合 `AccessTools.Method` 做动态查找，找不到时跳过而非崩溃

---

## 原则四：`void` → `Task` 的异步化重构会静默产生 Bug

### 表现
编译警告 `CS4014` — 异步调用未被等待。

### 原因
游戏代码在 Early Access 中持续进行异步化重构，方法签名从 `void` 变为 `async Task`。

### 做法
- `dotnet build` 的 warning 和 error 同等重要，`CS4014` 是危险信号
- 升级后搜索项目中所有调用 `RunManager.*`、`SaveManager.*`、`GameManager.*` 等核心类的代码段
- 遗漏 `await` 会导致异步操作未完成就执行后续逻辑，引发难以排查的竞态条件

---

## 原则五：Godot 相关类型命名空间可能漂移

### 表现
编译时类型找不到，或运行时绑定失败。

### 风险类型
- `NButton`、`NPauseMenuButton`、`NClickableControl` 等 UI 组件
- `NMainMenu`、`NPauseMenu`、`NRun` 等场景节点
- 信号名称（`SignalName.Released` 等）

### 做法
- 使用反编译工具列出所有引用类型的完整命名空间
- 特别注意 `MegaCrit.Sts2.Core.Nodes` 下子命名空间的变化
- 尽量通过父类或接口引用，而非具体类型

---

## 原则六：检查 `export_presets.cfg` 与 `--export-pack` 参数一致

### 问题
Godot 的 .pck 导出会尝试再次编译 .NET 项目，如果与已有构建冲突会产生非致命警告，但仍然会生成可用的 .pck。

### 做法
- 确保 `export_presets.cfg` 中的预设名称（如 `"Windows Desktop"`）与 `.csproj` 的 `GodotPublish` target 中 `--export-pack` 参数完全一致
- Godot 版本必须与游戏使用的引擎版本一致（查看 `release_info.json` 或游戏日志）
- 路径中避免空格，或正确加引号

---

## 原则七：渐进式验证清单

每次版本升级按顺序执行：

| 步骤 | 检查项 | 验证方式 |
|------|--------|----------|
| 1 | 游戏版本 | 检查 `release_info.json` 中的 `version` 字段 |
| 2 | 模块引用 | `.csproj` 的 `Sts2DataDir` 存在并指向正确目录 |
| 3 | 接口完整性 | `dotnet build` 无 CS0535 |
| 4 | Harmony 目标 | 所有 Patch 的方法在新 `sts2.dll` 中存在 |
| 5 | 异步签名 | 无 CS4014 警告 |
| 6 | Godot 导出 | `.pck` 正常生成 |
| 7 | 文件部署 | `mods/<ModId>/` 下包含 `.dll` + `.pck` + `.json` |

---

## 原则八：安全地检查 `sts2.dll`

在无法直接反编译或加载的 Windows 环境中：

- 创建临时控制台项目，直接引用目标 `sts2.dll`（以及 `GodotSharp.dll`、`0Harmony.dll`），用反射列出类型和方法签名
- 不要用 `Add-Type` 或 `Assembly.LoadFrom` 直接加载 `sts2.dll` — 它依赖 Godot 原生库，会抛出 `FileLoadException` 或段错误
- 保持 Godot 引擎版本与游戏使用的版本一致（MegaDot v4.5.1.m.8 等）
