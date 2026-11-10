# CMP 系统主题字体修复：逐文件代码变更说明

> - 分支：`fix-font-switch`
> - 基线：`90f0034e8ce34d43ab2ed3f248bb05983e95a7c2`
> - 合并后提交：`f9230f2f1595ccbd7943eefe790c27fd8947010e`
> - 原始提交数：33
> - 文件数：31
> - 最终差异：`+1322 / -37`

## 0. 变更文件索引

| 文件 | 作用 |
| --- | --- |
| `compose/export/export/src/ohosMain/kotlin/androidx/compose/export/ui/arkui/ArkUIViewController.kt` | 保留 Fusion 外部依赖的 C ABI 导出引用 |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/cpp/CMakeLists.txt` | HAR native 构建、共享库、rawfile、Kotlin runtime 和 skikobridge 链接 |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/cpp/compose/SystemThemeFontPathResolver.cpp` | native 主题字体路径解析 |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/cpp/compose/SystemThemeFontPathResolver.h` | native resolver 接口声明 |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/cpp/napi_init.cpp` | 导出 `getActiveThemeFontPath` NAPI |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/cpp/types/libcompose_arkui_utils/Index.d.ts` | NAPI 类型声明 |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/ets/compose/ComposeComponentHost.ets` | 明确 controller 类型，兼容 ArkTS 严格检查 |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/ets/compose/ComposeMessengerServices.ets` | 字体异步解析后直接通知 controller |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/ets/compose/density/DensityManager.ets` | 读取 fontId、主动解析路径并转发变化通知 |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/ets/compose/font/OhosSystemThemeFontResolver.ets` | 当前主题字体文件解析与异步缓存 |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/ets/compose/font/SystemThemeFontPathSelection.ets` | 字体路径选择和规范化纯函数 |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/ets/compose/locale/LocaleProvider.ets` | 向 Kotlin 暴露单路径和多路径读取函数 |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/ets/compose/window/WindowRotationGeometry.ets` | 使用旧 SDK 可编译的旋转 API 调用形式 |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/test/List.test.ets` | 注册字体路径选择测试 |
| `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/test/SystemThemeFontPathSelection.test.ets` | 字体路径选择分支测试 |
| `compose/ui/ui-text/src/commonMain/kotlin/androidx/compose/ui/text/font/SystemThemeFontPolicy.kt` | 定义哪些 FontFamily 跟随系统主题 |
| `compose/ui/ui-text/src/ohosArm64Main/kotlin/androidx/compose/ui/text/platform/NativeFont.ohos.kt` | 注入系统主题字体路径 provider |
| `compose/ui/ui-text/src/ohosArm64Main/kotlin/androidx/compose/ui/text/platform/OhosTextPlatformProvider.kt` | 扩展平台文本 provider 接口 |
| `compose/ui/ui-text/src/ohosTest/kotlin/androidx/compose/ui/text/platform/ParagraphDensityRetentionTest.kt` | 验证布局缓存可被清理 |
| `compose/ui/ui-text/src/skikoMain/kotlin/androidx/compose/ui/text/platform/ParagraphBuilder.skiko.kt` | 将主题路径纳入段落样式和缓存键 |
| `compose/ui/ui-text/src/skikoMain/kotlin/androidx/compose/ui/text/platform/PlatformFont.skiko.kt` | 加载主题字体文件、注册 Typeface 和 fallback alias |
| `compose/ui/ui-text/src/skikoTest/kotlin/androidx/compose/ui/text/font/SystemThemeFontPolicyTest.kt` | FontFamily 策略测试 |
| `compose/ui/ui-text/src/skikoTest/kotlin/androidx/compose/ui/text/platform/PlatformFontAliasTest.kt` | SystemFont alias 行为测试 |
| `compose/ui/ui-text/src/skikoTest/kotlin/androidx/compose/ui/text/platform/SkTextStyleThemeFontCacheKeyTest.kt` | 主题字体缓存键测试 |
| `compose/ui/ui/src/ohosArm64Main/kotlin/androidx/compose/ui/arkui/ArkUIViewController.kt` | 暴露 fontId 并注册主题路径 provider |
| `compose/ui/ui/src/ohosArm64Main/kotlin/androidx/compose/ui/arkui/density/DensityManager.kt` | Kotlin 侧 fontId 缓存和 Messenger 查询 |
| `compose/ui/ui/src/ohosArm64Main/kotlin/androidx/compose/ui/text/platform/OhosNapiTextPlatformProvider.kt` | 主线程字体路径快照与跨线程读取 |
| `compose/ui/ui/src/ohosArm64Main/kotlin/androidx/compose/ui/window/ComposeArkUIViewContainer.kt` | 字体身份变化时清理缓存、重建 resolver、刷新页面 |
| `compose/ui/ui/src/ohosArm64Main/kotlin/androidx/compose/ui/window/SystemFontIdentityTracker.kt` | 单容器字体身份和 generation 跟踪 |
| `compose/ui/ui/src/ohosTest/kotlin/androidx/compose/ui/arkui/density/DensityManagerFontIdTest.kt` | fontId 缓存与刷新测试 |
| `compose/ui/ui/src/ohosTest/kotlin/androidx/compose/ui/window/SystemFontIdentityTrackerTest.kt` | 字体身份变化测试 |

## 1. 改动总览

本次修改解决的是“Compose 不跟随 HarmonyOS 系统主题字体”问题，同时包含该功能源码调试所必需的 ArkUI HAR 构建、native 链接和 API 兼容修复。

整体链路分为五层：

```text
OHOS Configuration.fontId
  -> ArkTS DensityManager / SystemThemeFontResolver
  -> NAPI Provider
  -> Compose Text / ParagraphBuilder / FontCache
  -> Skia Typeface 或 Fusion OHOS FontCollection
```

关键设计：

1. 只有 `FontFamily.Default` 和 `FontFamily.SansSerif` 跟随主题字体。
2. 显式 `Serif`、`Monospace`、`Cursive`、`SystemFont` 保持原有语义。
3. Skia 读取主题字体文件并调用 `makeFromFile()`。
4. Fusion 继续使用 OHOS 全局 `FontCollection`。
5. 主题字体路径由主线程缓存，后台排版线程只读快照。
6. 字体身份变化后清理文本布局缓存、Typeface 缓存和字体加载缓存。

## 2. ArkUI 导出、Native ABI 与 HAR 构建

### 2.1 `compose/export/export/src/ohosMain/kotlin/androidx/compose/export/ui/arkui/ArkUIViewController.kt`

**改动**

- 增加 `@Volatile private var keepFusionRendererRootNodeExport = false`。
- 增加 `keepFusionRendererRootNodeExportReachable()`。
- `initFusionRendererNode()` 改为先调用上述函数，再委托给
  `initFusionRendererNodePreComposed()`。

**为什么这样改**

样例工程链接 `libkn.so` 时使用了：

```text
--undefined=androidx_compose_ui_arkui_ArkUIViewController_setFusionRendererRootNode
```

如果 Kotlin/Native 最终二进制中没有任何代码引用该 C ABI 导出，链接器可能将其裁掉，导致样例工程链接失败。

新增函数保留了对：

```text
_ArkUIViewController_setFusionRendererRootNode
```

的引用。布尔值默认为 `false`，所以正常运行路径不会额外执行 Fusion 设置，仅用于稳定导出符号。

**影响**

- 默认不改变 Fusion 运行行为。
- 避免静态分析和链接优化移除外部工程依赖的符号。

### 2.2 `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/cpp/CMakeLists.txt`

该文件包含五组修改。

#### 2.2.1 `GLOB_RECURSE` 增加 `CONFIGURE_DEPENDS`

```cmake
CONFIGURE_DEPENDS
```

**原因**

新增 `compose/SystemThemeFontPathResolver.cpp` 后，原 CMake/Ninja 构建图不会自动感知新源文件，容易出现“源码已存在但未参与编译”或仍使用旧 `build.ninja` 的问题。

**作用**

让 CMake 在目录内容变化时重新执行配置。

#### 2.2.2 增加 `rawfile.z`

```cmake
find_library(OHOS_RAWFILE_LIB rawfile.z)
```

并加入 `target_link_libraries`。

**原因**

将 ArkUI native 模块从静态库改为共享库后，原先可由最终消费方吸收的未解析符号必须由共享库自身解析。`rawfile` 相关符号需要显式链接。

#### 2.2.3 `compose_arkui_utils` 从 `STATIC` 改为 `SHARED`

```cmake
add_library(compose_arkui_utils SHARED ...)
```

**原因**

ArkTS 通过 `.so` 导入 `libcompose_arkui_utils.so`。静态库不能作为 ArkTS 的直接运行时动态模块，也无法提供预期的动态导出接口。

#### 2.2.4 定位并链接样例 `libkn.so`

```cmake
COMPOSE_KN_LIBRARY
target_link_options(compose_arkui_utils PRIVATE "${COMPOSE_KN_LIBRARY}")
```

**原因**

ArkUI native wrapper 引用了 Kotlin/Native 导出的 C 符号。生成共享库时必须让链接器找到这些符号，否则会出现 undefined symbol。

**为什么不用普通 `target_link_libraries`**

这里主要目的是满足共享库链接期的符号解析，不希望通过普通依赖关系再次把 `libkn.so` 打包进 HAR。`target_link_options` 以 linker 输入方式处理，避免重复分发 Kotlin runtime。

搜索顺序：

1. 优先使用 `SAMPLE_PROJECT_PATH`；
2. 从当前源码目录向上寻找 workspace 中的 `kmp-cmp-test-demo`；
3. 找到后才加入链接。

这样既支持 Harness 编译环境，也支持 DevEco 普通目录布局。

#### 2.2.5 `compose_arkui_utils` 私有链接 `skikobridge`

```cmake
target_link_libraries(compose_arkui_utils PRIVATE skikobridge)
```

**原因**

`compose_arkui_utils` 的桥接代码需要 Skia bridge 提供符号。使用 `PRIVATE` 表示这是内部实现依赖，不把 `skikobridge` 作为 API 暴露给下游。

### 2.3 `SystemThemeFontPathResolver.h`

新增 native 接口：

```cpp
std::string FindActiveThemeFontPath();
```

**原因**

向 NAPI 桥接层和外部 native 消费方提供类型明确的主题字体路径查询接口，避免在 `napi_init.cpp` 内重复实现文件查找逻辑。

### 2.4 `SystemThemeFontPathResolver.cpp`

#### 固定主题根目录

```cpp
THEME_ROOT_A = "/data/themes/a/app"
THEME_ROOT_B = "/data/themes/b/app"
```

对应 ArkUI 下载主题的两套实际挂载位置。

#### `IsReadableRegularFile()`

同时检查：

- 是普通文件；
- 没有文件系统错误；
- 当前进程有读权限。

避免 manifest 指向不存在、目录或无权限路径。

#### `HasThemeFontPayload()`

同时要求：

```text
root/flag
root/fonts/
```

存在。`flag` 用于判断主题槽位已启用，`fonts/` 用于防止读取空目录或残留目录。

#### `ReadFile()`

二进制方式读取 `manifest.json`，避免编码换行转换影响 JSON 内容。

#### `ReadJsonStringValue()`

实现一个最小 JSON 字符串读取器，用于读取：

```json
{
  "ttfFileSrc": "..."
}
```

包含对以下转义的支持：

- 引号、反斜杠、斜线；
- `\b`、`\f`、`\n`、`\r`、`\t`；
- `\uXXXX` 到 UTF-8 的转换。

**原因**

native 侧只需要一个字段，没有必要引入完整 JSON 依赖，同时需要正确处理 Windows 风格路径和 Unicode。

#### `IsFontFile()` / `FindFirstFontFile()`

支持：

```text
.ttf
.otf
.ttc
```

manifest 不可读时，对目录内字体文件排序，选择第一个可读文件。

**原因**

主题包格式可能没有有效 manifest；此时仍应尽量找到实际字体，而不是直接回退系统字体。

#### `ResolveFromThemeRoot()`

1. 检查主题槽位；
2. 读取 manifest 的 `ttfFileSrc`；
3. 只取 basename，并拼回受控的 `fonts` 目录；
4. 文件不可读时扫描目录。

**只取 basename 的原因**

防止 manifest 中的相对路径或外部绝对路径突破主题 `fonts` 目录。

#### `FindActiveThemeFontPath()`

按 A、B 顺序查找，A 有效时优先 A，否则 B。

### 2.5 `napi_init.cpp`

新增：

```cpp
GetActiveThemeFontPath()
```

并将 `getActiveThemeFontPath` 注册到 NAPI exports。

**原因**

保留 native 层主题字体解析能力，使 ArkTS 或其他 native consumer 可以通过统一桥接查询路径。当前最终 ArkTS 主链路主要使用 `OhosSystemThemeFontResolver.ets` 直接访问文件系统，这个 native 接口作为兼容入口保留。

### 2.6 `libcompose_arkui_utils/Index.d.ts`

新增：

```ts
export declare function getActiveThemeFontPath(): string;
```

**原因**

ArkTS 调用原生 `.so` 时必须存在对应类型声明，否则构建和 IDE 检查会报“导出不存在”。

## 3. ArkTS 字体读取、缓存与通知

### 3.1 `ComposeComponentHost.ets`

修改：

```ts
const controller: InternalArkUIViewController | null = hostRef.controller;
```

**原因**

显式收窄控制器类型，避免源码本地依赖下 ArkTS 严格检查把 `controller` 推断成不兼容联合类型。无运行时逻辑变化。

### 3.2 `ComposeMessengerServices.ets`

**改动**

- 保存 `controller` 引用；
- `DensityManager` 回调增加 `source`；
- 字体路径异步解析完成时直接调用：

```ts
this.controller?.onConfigurationUpdate();
```

- `onDestroy()` 清空 controller。

**原因**

字体路径解析可能晚于首次 configuration update。仅依赖 Messenger 异步往返时，可能错过一次重排窗口。

直接调用 controller 可保证：

1. Kotlin 清理字体缓存；
2. 重建 resolver；
3. 请求同步刷新。

销毁时置空避免宿主生命周期结束后仍引用 controller。

### 3.3 `DensityManager.ets`

#### 增加 `fontId` 配置字段

```ts
fontId?: string;
```

用于读取 `context.config.fontId`。

#### 注册 `compose.ui:Density.getFontId`

Kotlin `DensityManager.getFontId()` 通过 Messenger 调用该 handler。

#### 启动时主动请求主题字体

```ts
requestSystemThemeFontPath(initialFontId)
```

**原因**

冷启动不一定产生新的 configuration 事件。必须主动解析一次，否则第一帧只能使用默认字体。

#### 字体路径监听

```ts
subscribeSystemThemeFontPath(...)
```

解析完成后调用：

```ts
notifyDensityChanged('systemThemeFontPath')
```

`source` 参数让上层区分普通 density 变化与字体身份变化。

#### 销毁时取消订阅

避免 Activity/Ability 销毁后全局 listener 仍持有旧 manager。

#### `getFontId()`

读取 `context.config.fontId`，同时触发路径解析，返回：

```text
fontId|path1|path2
```

组合身份。

**为什么身份包含路径**

同一个 `fontId` 下可能发生异步路径更新或主题文件替换；只比较 `fontId` 会漏掉文件变化。

#### 空 `fontId`

保留 `''` 查询能力。部分系统阶段 `fontId` 尚未可用，但 `/data/themes` 中已经存在有效主题字体，完全拒绝空 ID 会导致首次启动错过真实主题。

### 3.4 `OhosSystemThemeFontResolver.ets`

这是本次功能的核心 ArkTS 文件。

#### `appendFontPath()`

- 将反斜杠统一为 `/`；
- 只取 basename；
- 拼接固定 `fonts` 目录；
- 去重。

原因与 C++ resolver 相同：路径受控且避免 manifest 注入目录穿越。

#### `resolveThemeFontPathsFromRoot()`

读取顺序：

1. `ttfFileSrc`；
2. `ttfFileSrcExt`；
3. manifest 失败或无结果时扫描 `.ttf/.otf/.ttc`。

支持主字体 + 字形回退字体，例如 Latin 字体加 CJK 字体。

#### `resolveActiveThemeFontPaths()`

依次检查 A/B 两个主题槽位，返回第一组有效字体。

#### `resolveSystemThemeFontPath()`

核心选择逻辑：

```text
默认 fontId
  -> GENERIC 描述

下载字体 fontId
  -> 优先实际主题文件
  -> 文件不可用时再尝试 STYLISH/GENERIC
```

**为什么不再把 STYLISH 作为首选**

`STYLISH` 只说明系统里存在风格字体，不说明用户当前选择的是哪一个。它可能返回汉仪仿宋，而不是“几分之几手写体”。

#### `pendingFontId` / `resolvedFontId`

异步 descriptor 查询可能乱序。完成时必须确认：

```ts
pendingFontId === fontId
```

否则旧查询结果可能覆盖新字体。

#### `hasRequestedFontId`

允许首次使用空字符串 ID 发起查询，同时仍然避免相同空 ID 重复请求。

#### listener

仅在 `cachedPaths` 非空时通知，确保 Kotlin 不会把一次无效解析误判为主题已切换。

### 3.5 `SystemThemeFontPathSelection.ets`

将选择逻辑抽成纯函数，便于单元测试。

规则：

```text
非默认 fontId + 有效 active paths
  -> active paths

其他情况
  -> descriptor fallback paths
```

同时负责：

- 去除 `file://`；
- trim；
- 去空；
- 去重。

**为什么抽成纯函数**

文件系统 API 和 `@kit.ArkGraphics2D` 在普通单元测试中难以构造，把决策逻辑分离出来后可以稳定覆盖边界条件。

### 3.6 `LocaleProvider.ets`

新增两个 Kotlin 可调用函数：

```ts
__font_getSystemThemeFontPath()
__font_getSystemThemeFontPaths()
```

**原因**

Kotlin 不能直接持有 ArkTS 对象。通过已有 Locale/String Provider 桥接对象暴露字体路径，避免为字体单独建立一条生命周期不同的全局桥。

单路径和多路径同时保留：

- 单路径适合只需要主字体的调用方；
- 多路径保留 `ttfFileSrcExt` 回退顺序。

### 3.7 `WindowRotationGeometry.ets`

新增本地结构接口：

```ts
interface RotationAwareWindow extends window.Window
```

并用本地数值常量：

```text
DISPLAY_ORIENTATION_INFO_TYPE = 0
WINDOW_ORIENTATION_INFO_TYPE = 1
```

替代新版 SDK 才有的 `window.RotationInfoType` 枚举。

**原因**

HAR 仍要兼容 `compatibleSdkVersion 17/12` 等旧 target。直接引用新枚举会让 HAR 编译失败，但底层运行时方法实际存在，因此通过结构类型调用并传稳定数值。

## 4. ArkTS 测试

### 4.1 `List.test.ets`

注册新的字体选择测试套件。

**原因**

Hypium 测试不会自动发现任意文件，必须加入总测试列表。

### 4.2 `SystemThemeFontPathSelection.test.ets`

覆盖：

1. 下载字体优先于 `STYLISH` descriptor；
2. 主字体和扩展字体顺序不变；
3. `fontId` 暂为空时仍保留有效 active path；
4. 默认字体 ID 不误用 stale 主题路径；
5. active path 无效时回退 descriptor。

这些用例直接对应真机问题中的错误分支。

## 5. Compose 文本策略与 Skia

### 5.1 `SystemThemeFontPolicy.kt`

新增唯一策略函数：

```kotlin
FontFamily.shouldUseSystemThemeFont(isOhos)
```

只有以下两种返回 `true`：

```text
DefaultFontFamily
GenericFontFamily(name == SansSerif)
```

**原因**

Material3 默认正文通常走 `SansSerif`，需要跟随主题。显式 `Serif`、`Monospace`、`Cursive` 和自定义字体不能被主题覆盖。

### 5.2 `NativeFont.ohos.kt`

新增：

```kotlin
setOhosSystemThemeFontPathProvider(...)
```

**原因**

`ui-text` 不能直接依赖 `ui` 模块中的 NAPI provider。通过函数注入 provider，让 `ui` 模块在初始化时把路径读取能力送入 `ui-text`。

### 5.3 `OhosTextPlatformProvider.kt`

增加：

```kotlin
fun getSystemThemeFontPath(): String
```

返回多行路径：

```text
主字体
回退字体1
回退字体2
```

**为什么使用换行字符串**

保持 provider 接口简单，同时能传递多文件回退列表；`ui-text` 侧统一通过 `decodeThemeFontPaths()` 解码。

### 5.4 `ParagraphBuilder.skiko.kt`

#### `ComputedStyle` 内部可见并加入主题路径

```kotlin
internal data class ComputedStyle(
    var systemThemeFontPaths: List<String> = emptyList(),
```

**原因**

`ComputedStyle` 是 `SkTextStyle` 缓存键。主题路径必须加入 data class 相等性，否则异步路径到达后仍会命中旧缓存。

#### 读取主题路径

```kotlin
systemThemeFontPathProvider?.invoke().decodeThemeFontPaths()
```

缓存键因此包含：

```text
路径顺序 + 全部回退文件
```

#### 选择是否调用 resolver

```kotlin
val useResolvedThemeTypeface =
    useSystemThemeFont &&
    !isFusionRenderBackendActive &&
    systemThemeFontPaths.isNotEmpty()
```

- Fusion：不解析具体文件，让 OHOS 全局 FontCollection 使用主题字体；
- Skia 且路径有效：由 `FontCache` 加载主题文件；
- 无路径：保留默认 family，让底层走平台默认逻辑。

#### 无 resolved result 时的字重

```kotlin
val resolvedWeight = resolved?.targetWeight ?: defaultWeight
```

**原因**

Fusion/无路径分支不再强制调用 `fontFamilyResolver.resolve()`，必须从原始字重重新构造 `SkFontStyle`，不能解引用空的 `resolved`。

#### 仅在有 resolver 结果时合成字体

避免对平台尚未解析的 family 提前执行 fake-bold / synthetic style。

### 5.5 `PlatformFont.skiko.kt`

#### 显式 OHOS `SystemFont` 使用公开 family 名

```kotlin
resolvedFontAlias(...)
```

OHOS 上 `SystemFont` 改用 `font.identity`，例如：

```text
HarmonyOS Sans
```

而不是内部 cache key。

**原因**

内部 cache key 会把显式 SystemFont 绑定到旧主题环境。公开 family 名允许 OHOS 全局字体集合在当前主题下正确解析该物理家族。

#### 主题文件按顺序加载

```kotlin
themeFontPaths.forEachIndexed { ... FontMgr.default.makeFromFile(path) ... }
```

第一个成功加载的 typeface 作为 primary，其余作为 aliases/fallback。

**作用**

- 支持 `ttfFileSrcExt`；
- 保留 manifest 顺序；
- 单文件失败不会阻断后续可用字体。

#### 主题字体缓存别名

```text
CMP_OHOS_SYSTEM_THEME_FONT:<pathHash>
```

缓存键包含全部字体路径、weight 和 slant。

**原因**

同类字体在主题文件变化后必须重新注册 Typeface，不能继续命中旧 alias。

#### `decodeThemeFontPaths()`

统一完成：

- 多行拆分；
- trim；
- 移除 `file://`；
- 去空。

避免 ParagraphBuilder 和 FontCache 各自实现不同解析规则。

## 6. `ui-text` 测试

### 6.1 `SystemThemeFontPolicyTest.kt`

验证：

- OHOS `Default` / `SansSerif` 跟随主题；
- 其他 generic family 不跟随；
- 非 OHOS 平台不改变原行为。

### 6.2 `PlatformFontAliasTest.kt`

验证显式 OHOS `SystemFont` 使用公开 family 名，而其他情况继续使用 cache key。

### 6.3 `SkTextStyleThemeFontCacheKeyTest.kt`

验证：

- 路径从空变为有效时必须产生不同 `ComputedStyle`；
- 只改变回退字体列表时也必须使缓存失效。

### 6.4 `ParagraphDensityRetentionTest.kt`

新增：

```kotlin
clearingTextPlatformLayoutCacheDropsCachedIntrinsics()
```

验证主题切换所调用的 `TextPlatformLayoutCache.clear()` 确实会移除段落 intrinsics，防止旧字体布局被复用。

## 7. Compose `ui` 模块桥接与刷新

### 7.1 `ui/.../ArkUIViewController.kt`

#### 暴露 `systemFontId`

Kotlin controller 从 `DensityManager` 获取：

```text
fontId|path1|path2
```

**原因**

Compose 容器需要比较字体身份，而不是只看 density。

#### 注册 theme font provider

```kotlin
setOhosSystemThemeFontPathProvider {
    ...getSystemThemeFontPath()
}
```

在 `_ArkUIViewController_setEnv()` 中完成，确保 NAPI 环境和文本 provider 已初始化。

### 7.2 `ui/.../density/DensityManager.kt`

新增：

- `cachedFontId`；
- `getFontId()`；
- `SEND_GET_FONT_ID`；
- `resetCache()` 清空 fontId。

**原因**

字体 ID 也会变化，必须和 density、fontScale、fontWeightScale 一样有缓存及配置更新失效。

### 7.3 `OhosNapiTextPlatformProvider.kt`

#### `cachedSystemThemeFontPath`

所有线程共享线程安全缓存。

#### `invalidatePlatformCaches()` 主动预热

```kotlin
if (JsEnv.isOnNapiOwnerThread()) {
    getSystemThemeFontPath()
}
```

**原因**

NAPI 只能在 owner thread 读取。配置更新发生在主线程时提前取快照，后台排版线程随后可直接读取，不会得到空路径。

#### `getSystemThemeFontPath()`

执行顺序：

1. 先读缓存；
2. 非 NAPI owner thread 且无缓存时返回空；
3. 主线程调用 ArkTS provider；
4. 用换行连接多路径；
5. 写入缓存。

**解决的真实问题**

修复前同一进程日志中：

```text
主线程 58342: resolved=true
后台线程 58486: resolved=false
```

后台线程直接访问 NAPI 得到空值，最终同一页面出现两种字体。

#### `callGlobalArrayFunction()`

新增数组读取能力：

- 检查 owner thread；
- 获取 provider 函数；
- 遍历 JS 数组；
- 过滤空值；
- 异常时记录日志并返回空列表。

**原因**

主题可能包含主字体和多个回退字体，不能只桥接第一个路径。

### 7.4 `ComposeArkUIViewContainer.kt`

#### `SystemFontIdentityTracker`

每个 Compose 容器单独维护字体身份和 generation。

#### 身份变化时清理缓存

```kotlin
TextPlatformLayoutCache.clear()
TextPlatformFontCache.clearGlobalTypefaceCache()
TextPlatformFontCache.clearFontLoadCaches()
```

分别清理：

- Paragraph/Intrinsics 文本布局缓存；
- 已注册 Typeface；
- 每个 `FontCache` 的文件加载缓存。

#### resolver 重建条件加入 `systemFontIdentityChanged`

字体文件变化必须创建新的 `FontFamilyResolver`，否则 resolver 中仍可能持有旧的 family/typeface 状态。

#### 请求同步刷新

加入 `systemFontIdentityChanged` 后，确保字体变化立即提交一帧，不用等待下一次滚动或用户交互。

#### `refreshSystemFontIdentityAfterPageShow()`

页面从后台恢复时：

1. 调用 `super.onConfigurationUpdate()` 清 ArkTS 缓存；
2. 重新读取字体身份；
3. 有变化时清字体/布局缓存并重建 resolver；
4. 请求刷新。

**原因**

应用在后台冻结期间可能收不到 font configuration event；只在 `onConfigurationUpdate` 处理会漏掉“设置中换字体后再切回应用”的场景。

### 7.5 `SystemFontIdentityTracker.kt`

规则：

```text
第一次观察 -> 只记录，不触发 relayout
相同身份   -> 不增加 generation
身份改变   -> generation + 1
```

**为什么第一次不触发**

首个 resolver 已按当前系统配置创建，首次初始化再清缓存和重排是无效成本。

## 8. Compose `ui` 测试

### 8.1 `DensityManagerFontIdTest.kt`

验证：

- 第一次读取缓存 fontId；
- 系统 fontId 改变但未 reset 时仍返回旧值；
- `resetCache()` 后读取新 fontId；
- Messenger 实际被重新查询。

### 8.2 `SystemFontIdentityTrackerTest.kt`

验证：

- 首次只 seed；
- 相同身份不增加 generation；
- 每次真实变化都递增；
- 空身份后续变为可用身份时能触发更新。

## 9. 改动之间的关系

不能用单个文件解释这次修复，完整闭环如下：

```text
1. ArkTS 读取真实主题字体
   OhosSystemThemeFontResolver

2. ArkTS 通知 Kotlin 重新配置
   DensityManager -> ComposeMessengerServices

3. Kotlin 生成线程安全路径快照
   OhosNapiTextPlatformProvider

4. Compose 判断哪些 family 应跟随主题
   SystemThemeFontPolicy

5. Skia 加载实际字体文件
   ParagraphBuilder + PlatformFont

6. 字体变化后清理旧布局和旧 Typeface
   ComposeArkUIViewContainer + SystemFontIdentityTracker

7. HAR/native 打包保证桥接符号可链接
   CMake + C ABI export
```

## 10. 行为边界

### 会跟随主题字体

- `FontFamily.Default`
- `FontFamily.SansSerif`

### 不会跟随主题字体

- `FontFamily.Serif`
- `FontFamily.Monospace`
- `FontFamily.Cursive`
- 自定义 `LoadedFontFamily`
- 显式指定 family 的 `SystemFont`

### 正常字体回退

如果主题字体不包含中文 Glyph，中文继续显示系统 CJK 字体。这是字体文件覆盖范围导致的正常 fallback，不属于 Compose 未刷新。

### Fusion

Fusion 不直接把主题文件设置为 typeface，而是保留默认 family，让 OHOS 全局 `FontCollection` 解析当前主题。

### Skia

Skia 必须使用实际字体文件，因为 Skia 的默认 `FontCollection` 不会自动包含 OHOS 当前主题字体。

## 11. 验证结果

本次代码最终通过：

- Skia debug HAP 构建；
- HAR native 编译；
- HAP 安装；
- 应用启动；
- 冷启动主题字体解析；
- 主线程和后台排版线程均为 `resolved=true`；
- 同一进程切换“几分之几手写体”和“汉仪粗仿宋”；
- 首屏、混排区、后续滚动区同步切换。

最终结果证明：

1. 字体文件选择正确；
2. Skia 能加载指定文件；
3. 后台排版线程能读取主线程快照；
4. 旧文本布局缓存能被清除；
5. Fusion 的默认 family 语义没有被破坏。
