# CMP 字体切换修复方案疑问分析

> - 分析日期：2026-11-16
> - 分析对象：`compose-multiplatform-core`
> - 参考提交：`cd98e8a3c532fb783e8b6ca796e0c2e73ad547a4`
> - 结论性质：基于当前源码、DevEco Studio 6.0.0 SDK 头文件和 Android/iOS 源码的静态分析
> - 本文只做分析，不修改代码

## 1. 结论摘要

### 问题一：Worker 是否有 C API

有，但不等价于“Worker 可以同步读取 ArkTS 对象”。

1. HarmonyOS NAPI 提供 `napi_create_threadsafe_function()` 和 `napi_call_threadsafe_function()`。Worker 可以把请求投递到 NAPI owner thread 执行。
2. 当前 CMP 已经引入这些 API，`skikobridge/ComposeDispatcher.cpp` 也实际使用 ThreadSafeFunction 完成主线程 JS 调度。
3. ThreadSafeFunction 的正常模式是异步回调，不是 Worker 直接读取 `napi_value`。虽然有 blocking 模式，但在“主线程等 Worker、Worker 等主线程”的结构下存在死锁风险。
4. HarmonyOS Native Drawing 还提供字体描述符、FontCollection 和 `OH_Drawing_FontSetThemeFontFollowed()` 等 C API。它们可被 Native 线程调用，但当前没有证据表明可以直接把“ArkUI 当前主题选择”转换为 Skiko/Skia 所需的 `SkTypeface`，也没有直接提供 `fontId -> 当前主题文件路径`。

因此：

> Worker 确实有跨线程 C API，但当前“主线程生成不可变路径快照，Worker 只读快照”的方案仍是低风险、低延迟且不会阻塞主线程的方案。

### 问题二：是否可以避免 `commonMain` 改动

可以。

`cd98e8a3` 中涉及 `ui-text/commonMain` 的新增只有：

```text
compose/ui/ui-text/src/commonMain/kotlin/androidx/compose/ui/text/font/SystemThemeFontPolicy.kt
```

它只提供 `Default/SansSerif` 是否跟随主题的纯策略判断。该函数只被 Skiko 源集和 Skiko 测试使用，因此不需要放在 `commonMain`。

实际与第 19 步缓存键相关的代码在：

```text
compose/ui/ui-text/src/skikoMain/.../ParagraphBuilder.skiko.kt
compose/ui/ui-text/src/skikoMain/.../PlatformFont.skiko.kt
```

而不是本次提交新增的 `commonMain` 文件。

推荐方案：

1. 将 `SystemThemeFontPolicy.kt` 移到 `ui-text/skikoMain`。
2. 保留 OHOS 运行时判断，不影响 iOS、desktop 等其它 Skiko 平台。
3. 继续在 Skiko 层把主题路径加入 `ComputedStyle` 和 Typeface alias。
4. Paragraph 缓存继续通过重建 `FontFamilyResolver` 失效。

这样可以去掉本次 `commonMain` 新增，但不能完全避免修改 `skikoMain`。原因是当前的 `ParagraphBuilder`、`FontCache`、TextStyle cache 本身就在 Skiko 共享源集中；只要修复 SkiaRender，就必须经过这些实现。

## 2. 问题一：Worker 和 NAPI C API

### 2.1 NAPI ThreadSafeFunction 确实存在

DevEco Studio 6.0.0 SDK 头文件：

```text
/Applications/DevEco-Studio-6.0.0.app/Contents/sdk/default/
openharmony/native/sysroot/usr/include/napi/native_api.h
```

声明包括：

```c
napi_status napi_create_threadsafe_function(...);
napi_status napi_call_threadsafe_function(
    napi_threadsafe_function func,
    void* data,
    napi_threadsafe_function_call_mode is_blocking
);
```

SDK 还提供：

```c
napi_status napi_call_threadsafe_function_with_priority(...);
```

调用模式：

```c
napi_tsfn_nonblocking
napi_tsfn_blocking
```

### 2.2 CMP 已经在使用 ThreadSafeFunction

Kotlin 侧：

```text
compose/ui/ui/src/ohosArm64Main/kotlin/androidx/compose/ui/napi/JsEnv.kt
```

已经封装：

```kotlin
createThreadsafeFunction(...)
callThreadsafeFunction(...)
```

当前 CMP 的调用使用非阻塞模式：

```kotlin
napi_call_threadsafe_function(
    function,
    data,
    napi_tsfn_nonblocking,
)
```

Native 桥接侧：

```text
compose/ui/ui-arkui/src/ohosArm64Main/cpp/.../skikobridge/.../ComposeDispatcher.cpp
```

也通过 `napi_create_threadsafe_function()` 把工作调度到主 JS 线程。

因此“Worker 完全没有任何 C API”这个说法不正确。

### 2.3 为什么 ThreadSafeFunction 不能直接替代当前快照

ThreadSafeFunction 的真实语义是：

```text
Worker
  -> 把请求放入 NAPI 队列
  -> NAPI owner thread 执行 JS/ArkTS 回调
  -> 结果通过回调或另一份线程安全状态返回 Worker
```

它不是：

```text
Worker
  -> 直接读取 ArkTS 的 cachedPaths 内存
```

如果 Worker 必须同步获取路径，需要额外实现：

```text
请求队列
结果队列
条件变量/信号量
阻塞等待
生命周期取消
错误传播
```

使用 blocking 模式仍有死锁风险：

```text
主线程等待 Worker 完成 Paragraph
Worker 阻塞等待主线程执行 NAPI 回调
```

因此 ThreadSafeFunction 可以作为通用异步桥，但不适合嵌入当前同步的 `getSystemThemeFontPath()`。

### 2.4 Native Drawing 字体 C API

DevEco 6.0.0 当前 SDK 包含：

```text
native_drawing/drawing_font_collection.h
native_drawing/drawing_font_mgr.h
native_drawing/drawing_text_font_descriptor.h
native_drawing/drawing_font.h
```

可用能力包括：

```c
OH_Drawing_CreateFontCollection()
OH_Drawing_CreateSharedFontCollection()
OH_Drawing_GetFontCollectionGlobalInstance()
OH_Drawing_GetSystemFontFullNamesByType(...)
OH_Drawing_GetFontDescriptorByFullName(...)
OH_Drawing_MatchFontDescriptors(...)
```

还存在一个与主题字体直接相关的 API：

```c
OH_Drawing_FontSetThemeFontFollowed(
    OH_Drawing_Font* font,
    bool followed
);
```

SDK 注释说明：

> 当 typeface 未设置时，`followed=true` 会使用主题字体。

该 API 自 API 15 提供。

### 2.5 为什么这些 C API 不是当前问题的直接替代

#### 限制一：字体对象体系不同

当前 SkiaRender 使用的是：

```text
Skiko
  -> org.jetbrains.skia.FontMgr
  -> SkTypeface
  -> SkParagraph
```

Native Drawing 提供的是：

```text
OH_Drawing_FontMgr
  -> OH_Drawing_Typeface
  -> OH_Drawing_Font
  -> OH_Drawing_Typography
```

当前源码和 SDK 中未找到可靠、公开的桥接方式，可以把：

```text
OH_Drawing_Typeface
  -> SkTypeface
```

直接交给 Skiko/SkParagraph。

#### 限制二：描述符枚举仍不等于当前主题文件

C API 可以列出：

```text
GENERIC
STYLISH
INSTALLED
ALL
```

但“系统里有哪些字体”仍然不等于“当前 ArkUI 选择了哪个主题文件”。

当前仍需要：

```text
fontId
  + 激活主题槽位 flag
  + manifest
  + 实际文件路径
```

才能得到确定的主字体和 fallback 文件。

#### 限制三：API 版本兼容

`OH_Drawing_FontSetThemeFontFollowed()` 从 API 15 开始提供。如果最低兼容版本低于 15，还需要：

```text
运行时 API 版本判断
旧版本回退
新 API 不存在的动态查找或条件编译
```

不能把它作为所有设备上的统一实现。

### 2.6 可行但不同风险的备选方案

| 方案 | 可行性 | 风险 | 结论 |
| --- | --- | --- | --- |
| Worker 直接调用普通 NAPI | 不可行 | 跨线程使用 `napi_env/napi_value` 不安全 | 放弃 |
| ThreadSafeFunction + 同步等待 | 技术上可做 | 队列、阻塞、死锁、生命周期复杂 | 不推荐 |
| ThreadSafeFunction + 异步回调 | 可行 | Paragraph 无法同步等待，需要异步重排 | 可作为长期通用桥，但不简化当前问题 |
| 主线程路径快照 + Worker 只读 | 已验证 | 需要处理启动空快照与异步刷新 | 当前推荐 |
| OH_Drawing 字体 C API | 对 OH_Drawing/Fusion 有潜力 | 无法直接成为 SkTypeface；API 15 兼容性 | 需单独 POC，不能直接替换 |

## 3. 问题二：第 19 步与 `commonMain`

### 3.1 先澄清提交中实际改了什么

`cd98e8a3` 在 `ui-text/commonMain` 新增的文件是：

```text
SystemThemeFontPolicy.kt
```

内容只包含：

```kotlin
internal fun FontFamily.shouldUseSystemThemeFont(
    isOhos: Boolean,
): Boolean {
    if (!isOhos) return false
    return when (this) {
        is DefaultFontFamily -> true
        is GenericFontFamily ->
            name == FontFamily.SansSerif.name
        else -> false
    }
}
```

它回答：

```text
哪些 Compose family 允许跟随 OHOS 主题字体？
```

它不在缓存键中保存文件路径。

### 3.2 第 19 步缓存键实际从哪里来

#### SkTextStyle 缓存键

```text
ParagraphBuilder.skiko.kt
```

修改：

```kotlin
internal data class ComputedStyle(
    var systemThemeFontPaths: List<String> = emptyList(),
    // 其它样式字段
)
```

这是 `skikoMain`，不是 `commonMain`。

#### Typeface 缓存 alias

```text
PlatformFont.skiko.kt
```

修改：

```kotlin
val cacheAliases = themeFontPaths.map {
    "CMP_OHOS_SYSTEM_THEME_FONT:${it.hashCode()}"
}
```

这也是 `skikoMain`，不是 `commonMain`。

#### ParagraphIntrinsics 缓存

```text
SkiaParagraphIntrinsics.skiko.kt
```

原本 key 已包含：

```kotlin
fontFamilyResolver
```

本次没有把它改到 `commonMain`。

#### Foundation MultiParagraph 缓存

```text
foundation/src/commonMain/.../MultiParagraphLayoutCache.kt
```

key 原本已经包含：

```kotlin
fontFamilyResolver
```

本次没有修改这个文件。

#### TextMeasurer 布局缓存

```text
ui-text/src/commonMain/.../TextMeasurer.kt
```

`CacheTextLayoutInput` 原本已经用：

```kotlin
fontFamilyResolver !== other.fontFamilyResolver
```

区分布局缓存。

本次也没有修改它。

### 3.3 因此 `commonMain` 改动不是第 19 步必需的

当前真正的缓存有效化机制是：

```text
字体身份变化
  -> 清理显式文本/Typeface 缓存
  -> 重建 FontFamilyResolver
  -> 以 resolver 为键的缓存自然不命中
```

同时：

```text
SkTextStyle 缓存
  -> ComputedStyle 加入主题路径

Typeface 缓存
  -> path hash 进入 alias
```

这两项都在 `skikoMain`。

`SystemThemeFontPolicy` 只是被 Skiko 代码调用的纯策略函数，可以迁移。

## 4. 不涉及 `commonMain` 的可行方案

### 4.1 推荐方案 A：将策略移到 `skikoMain`

改动形态：

```text
删除：
ui-text/src/commonMain/.../SystemThemeFontPolicy.kt

新增：
ui-text/src/skikoMain/.../SystemThemeFontPolicy.kt
```

或直接放入现有：

```text
ui-text/src/skikoMain/.../PlatformFont.skiko.kt
```

并保持：

```kotlin
currentPlatform() == Platform.OHOS
```

运行时条件。

优点：

- 删除 `commonMain` 新增。
- 行为不变。
- 仍可被 `ParagraphBuilder.skiko.kt`、`PlatformFont.skiko.kt` 和 Skiko 测试访问。
- iOS/desktop 不进入 OHOS 分支。

缺点：

- 仍需修改 `skikoMain`，因为 SkiaRender 的实现本身就在该源集。

这是最小、最稳妥的方案。

### 4.2 方案 B：从 OHOS 注入 family 判断 provider

思路：

```text
ui-text/skikoMain 保存一个主题 family 判断 provider
OHOS ui 初始化时注入 provider
Skiko 调用 provider 判断 Default/SansSerif
```

这样可以把 OHOS 策略细节从未共享文件移到 OHOS 初始化代码。

但仍需在 `skikoMain` 增加 provider 调用点和存储变量，因此只能消除 `commonMain`，不能消除 `skikoMain`。

### 4.3 方案 C：不改缓存键，只在字体变化时清空 SkTextStyle 缓存

思路：

```text
保留原 ComputedStyle key
  -> 字体身份变化
  -> 清 SkTextStyle cache
  -> 清 Typeface cache
  -> 清 Paragraph cache
  -> 重建 resolver
```

优点：

```text
不需要 systemThemeFontPaths 进入 ComputedStyle
```

风险：

1. `SkTextStyle` 缓存是私有 `WeakKeysCache`，当前没有安全清理入口。
2. 清缓存后，仍在运行的 Worker 可能把旧字体样式重新写回缓存。
3. 只依赖清缓存，无法自然区分两个并发字体环境。
4. 以后新增类似 Typeface 缓存时，容易漏清理。

结论：

技术可行，但健壮性低于“路径参与 key + 清缓存 + 新 resolver”组合，不建议替代当前方案。

### 4.4 方案 D：为 OHOS 单独实现 ParagraphBuilder/FontCache

理论上可以复制 Skiko 实现到 OHOS 源集，但这会：

```text
复制大量 ParagraphBuilder/字体缓存代码
失去上游 Skiko 修复合并能力
增加双实现漂移风险
```

不建议。

## 5. Android 如何处理文本布局缓存键

### 5.1 Android 使用公共 resolver 身份

Android `AndroidComposeView` 持有：

```kotlin
override var fontFamilyResolver: FontFamily.Resolver by
    mutableStateOf(
        createFontFamilyResolver(context),
        referentialEqualityPolicy()
    )
```

公共 `TextLayoutCache` 的 key 包含：

```kotlin
result = 31 * result + fontFamilyResolver.hashCode()
```

比较时：

```kotlin
if (fontFamilyResolver !== other.textLayoutInput.fontFamilyResolver) {
    return false
}
```

因此 Android 也不依赖把字体文件路径直接加入通用布局缓存键。

### 5.2 Android 的 resolver 重建条件

当前源码中明确重建 resolver 的条件是：

```kotlin
if (
    newConfig.fontWeightAdjustmentCompat !=
    currentFontWeightAdjustment
) {
    currentFontWeightAdjustment =
        newConfig.fontWeightAdjustmentCompat
    fontFamilyResolver =
        createFontFamilyResolver(context)
}
```

即 Android 主要处理 `fontWeightAdjustment`。

Android 的 default family 使用：

```text
Roboto / Noto Sans
```

当前 fork 没有 OHOS 这种：

```text
可下载主题字体
fontId
active theme manifests
多个主题 fallback 文件路径
```

对应机制，因此 Android 代码不需要 OHOS 风格的路径身份。

### 5.3 Android ParagraphIntrinsics

`AndroidParagraphIntrinsics` 内部通过：

```kotlin
fontFamilyResolver.resolve(...)
```

得到 Typeface，并维护异步 Typeface dirty tracker：

```kotlin
resolvedTypefaces
hasStaleResolvedFonts
```

它不是用 OHOS 的自定义 `SkiaParagraphIntrinsicsCache` 来区分主题文件路径。

## 6. iOS 如何处理文本布局缓存键

### 6.1 iOS 使用 Skiko 共享 resolver

iOS Compose UI 的根节点创建共享：

```kotlin
override val fontFamilyResolver =
    createFontFamilyResolver()
```

公共 `TextLayoutCache` 同样通过 resolver 身份区分布局：

```kotlin
fontFamilyResolver !== other.fontFamilyResolver
```

### 6.2 iOS 没有 OHOS 主题文件路径切换链路

iOS Skiko default family 使用：

```text
.AppleSystemUIFont
Helvetica Neue
Helvetica
```

当前代码中没有发现类似 OHOS 的：

```text
主题下载目录
fontId
active flag
manifest
CJK fallback 路径列表
```

因此 iOS 不需要为每个 `ComputedStyle` 注入页面主题文件路径。

### 6.3 自定义 SkiaParagraphIntrinsics 缓存只在 HARMONY 生效

`SkiaParagraphIntrinsicsCache` 的使用条件是：

```kotlin
if (CurrentPlatform == PlatformType.HARMONY) {
    // key 包含 fontFamilyResolver
} else {
    // 直接创建，不走该 OHOS 专用缓存
}
```

因此 iOS/desktop 不依赖这套 OHOS 自定义 intrinsics cache。

## 7. 对照结论

| 平台 | 通用布局缓存如何失效 | 是否有主题字体路径 | 当前修复是否适用 |
| --- | --- | --- | --- |
| Android | resolver 实例变化；fontWeightAdjustment 变化时重建 resolver；异步字体有 dirty tracker | 代码中未发现 OHOS 式主题文件路径机制 | 不需要路径型 key |
| iOS | resolver 实例变化；公共 TextMeasurer cache 比较 resolver | 未发现 OHOS 式 fontId/manifest/主题目录 | 不需要路径型 key |
| OHOS SkiaRender | resolver 重建 + 清缓存 + SkTextStyle/Typeface 路径 key | 有 `fontId + 主题目录 + manifest + fallback` | 需要路径型和身份型失效 |

## 8. 最终建议

### 对问题一

1. 不把“Worker 无法调用 NAPI”理解为“没有任何 C API”。
2. 保留当前主线程路径快照方案作为默认实现。
3. 如果后续需要更通用的后台访问，可评估 ThreadSafeFunction 的异步协议，但不要引入同步阻塞等待。
4. 对 `OH_Drawing_FontSetThemeFontFollowed()` 可以单独做 POC，但这属于 Native Drawing/可能适用于 Fusion 的方向，不是 Skia/Skiko 的直接替代。

### 对问题二

推荐执行方案：

```text
1. 将 SystemThemeFontPolicy.kt
   从 ui-text/commonMain
   移到 ui-text/skikoMain

2. 保留 OHOS 运行时条件

3. 保留：
   - ComputedStyle.systemThemeFontPaths
   - Typeface path alias
   - resolver 重建
   - 字体相关缓存清理
```

这样处理后：

```text
commonMain 新增改动：0
skikoMain 改动：保留
ohosArm64Main 改动：保留
iOS/desktop 行为：保持原逻辑
OHOS SkiaRender：保持当前修复能力
```

如果要求“连 skikoMain 也不能改”，则只能采用 OHOS 专用 hook/清缓存方案，但会牺牲缓存并发正确性和上游维护性，不建议作为首选。

