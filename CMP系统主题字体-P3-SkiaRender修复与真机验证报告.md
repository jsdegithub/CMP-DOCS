# CMP 系统主题字体 P3：SkiaRender 修复与真机验证报告

| 项目 | 结论 |
| --- | --- |
| 验证日期 | 2026-10-30 |
| 问题域 | OHOS 系统主题字体、SkiaRender、Compose 字体解析、异步字体源、缓存失效 |
| 总体结论 | **修复完成**：Fusion Renderer 与 SkiaRender 的默认字体均跟随系统主题字体，支持同一进程内热切换，显式 `SystemFont` 保持固定字体语义 |
| 验证设备 | `68Q0223B17000839`，HarmonyOS 6.1.0，API 24 |
| 应用 | `com.tencent.compose` / `EntryAbility` |
| Skia 运行后端 | `getRenderBackendId id=0 fnMode=1` |
| 最终设备字体 | “鸿蒙黑体” |
| 最终设备状态 | 已恢复默认字体，最新 Skia HAP 已运行 |
| Core 仓库 | `compose-multiplatform-core`，分支 `develop-1.9.2-OH` |
| Sample 仓库 | `kmp-cmp-test-demo`，HEAD `1ba7d6635` |

## 1. 最终结论

这个 bug **包含 SkiaRender**。原任务描述中的“Compose 通过 Skia 自绘，Skia 使用自己注册的默认字体”正是 SkiaRender 不跟随系统主题字体的直接表现，因此只修 Fusion Renderer 不能算完成。

P3 已经在真实 SkiaRender 路径完成以下闭环：

1. ArkTS 从 OHOS 字体接口解析当前主题字体文件。
2. `fontId` 和字体文件路径变化会通过宿主回调通知 Kotlin。
3. Skia 使用 `FontMgr.default.makeFromFile(path)` 加载主题字体，并注册独立别名。
4. `FontFamily.Default` / `FontFamily.SansSerif` 使用该 typeface。
5. 字体路径进入 `SkTextStyle` 缓存键，避免异步字体路径到达后继续复用首帧旧样式。
6. `fontId` 变化时清理字体与图层缓存、重建 Resolver，并触发重新布局和绘制。
7. 默认字体态使用 `HarmonyOS Sans SC`，自定义风格字体态使用 OHOS `STYLISH` 字体描述。

| 验收项 | 结果 | 证据 |
| --- | --- | --- |
| SkiaRender 冷启动跟随自定义主题字体 | 通过 | `backendId=0`，`ShuS-SC.ttf`，`resolved=true`，截图变为衬线字形 |
| SkiaRender 默认字体冷启动 | 通过 | `fontId=22200001`，`HarmonyOS_Sans_SC.ttf`，`resolved=true` |
| 默认字体切换到自定义字体 | 通过 | 同一 PID `64018`，`h0011`，generation 2，路径 `ShuS-SC.ttf` |
| 恢复鸿蒙黑体 | 通过 | 字体样式页显示“鸿蒙黑体”勾选，应用恢复无衬线字形 |
| 显式 `SystemFont` 固定语义 | 通过 | P2 已通过 Fusion 验证；P3 未改变该策略 |
| 复杂文本度量矩阵 | 未覆盖 | 见第 7 节残余风险 |

## 2. 根因

### 2.1 Skia 字体集合的固定默认 family

Skiko `FontCollection` 默认 family 为 `HarmonyOS Sans`。仅将 `FontFamily.Default` 的 family 置空，只能让 Skia 回退到它自己的默认集合，不能保证拿到 OHOS 当前主题字体。

同时，OHOS `OH_Drawing_TypefaceCreateDefault()` 返回的是系统默认 typeface，不保证是当前主题字体的 typeface。因此不能把“系统默认字体”直接等同于“当前主题字体”。

### 2.2 系统字体路径异步到达

ArkTS 获取主题字体描述是异步接口。首次排版可能早于路径返回；如果只通知页面 invalidate，而没有让文本样式缓存失效，后续重新排版仍可能命中首次生成的 `SkTextStyle`。

修复前，`ComputedStyle` 的缓存键没有主题字体路径。异步路径变化后，即使 Resolver 重建，相同 family/weight/style 仍可能命中旧缓存。

### 2.3 默认字体与自定义字体需要不同来源

实测发现：

- 自定义字体时，`fontId=h0011`，`SystemFontType.STYLISH` 返回当前自定义字体 `ShuS-SC.ttf`。
- 恢复默认字体后，`fontId=22200001`，`STYLISH` 仍可能返回上一次风格字体路径，不能用于判断当前默认字体。
- 默认态应从 `SystemFontType.GENERIC` 中选择 `HarmonyOS Sans SC`，失败时再尝试 `HarmonyOS Sans`。

## 3. 修复方案

### 3.1 ArkTS 主题字体解析

`OhosSystemThemeFontResolver.ets` 按 `fontId` 区分：

```text
fontId == 22200001
  -> GENERIC
  -> 优先 HarmonyOS Sans SC
  -> 回退 HarmonyOS Sans

fontId != 22200001
  -> STYLISH
  -> 取当前风格字体描述
  -> 无风格字体时回退 GENERIC
```

解析结果通过 NAPI provider 暴露为绝对文件路径。

### 3.2 Skia typeface 加载

`PlatformFont.skiko.kt` 对默认/无衬线 family 使用主题字体路径：

```text
systemThemeFontPathProvider()
  -> FontMgr.default.makeFromFile(path)
  -> registerTypeface(typeface, CMP_OHOS_SYSTEM_THEME_FONT:<path-key>)
  -> FontLoadResult(alias, typeface)
```

显式 `SystemFont(...)` 不进入该分支，继续使用公开 family，保持固定字体选择。

### 3.3 文本样式缓存失效

`ComputedStyle` 增加 `systemThemeFontPath`，使以下两个状态拥有不同缓存键：

```text
路径未解析：systemThemeFontPath = null
路径已解析：systemThemeFontPath = "/system/fonts/..."
```

这样即使首帧已生成样式，异步路径返回后下一次排版也会重新走 Resolver，而不是复用旧 `SkTextStyle`。

### 3.4 同进程刷新链路

路径解析完成后，`ComposeMessengerServices` 直接调用控制器 `onConfigurationUpdate()`，并继续触发既有的 density/backend refresh 回调。

随后：

```text
fontId/path identity changed
  -> clear global typeface cache
  -> clear font load cache
  -> clear layer snapshot cache
  -> rebuild FontFamily.Resolver
  -> requestSyncRefresh
```

## 4. 真机验证

### 4.1 Skia 冷启动：自定义主题字体

证据：

- [首页截图](CMP系统主题字体-P3证据/p3-maven-local-cold.jpeg)
- [字体日志](CMP系统主题字体-P3证据/p3-maven-local-cold-font.log)

关键日志：

```text
getRenderBackendId id=0 fnMode=1
fontId=h0011 default=false stylish=1 candidates=1
descriptor fullName=ShuS-SC family=ShuS-SC path=/system/fonts/ShuS-SC.ttf
CMP_FONT, Skia paragraph style family=FontFamily.Default themePath=/system/fonts/ShuS-SC.ttf resolved=true
```

截图中英文均已从 HarmonyOS Sans 风格变为书宋/衬线风格。

### 4.2 默认字体到自定义字体热切换

设备从“鸿蒙黑体”切到“汉仪粗仿宋”后返回应用，前后 PID 均为 `64018`。

证据：

- [热切换截图](CMP系统主题字体-P3证据/p3-style-hot-final.jpeg)
- [热切换日志](CMP系统主题字体-P3证据/p3-style-hot-final-font.log)

关键日志：

```text
pageShow themeIdentity previous=22200001 current=h0011 generation=1 action=clearFontCaches
fontId=h0011 default=false stylish=1 candidates=1
resolved fontId=h0011 path=/system/fonts/ShuS-SC.ttf
themeIdentity previous=h0011 current=h0011|/system/fonts/ShuS-SC.ttf generation=2
fontId changed -> generation=2 -> resolver rebuilt -> relayout requested
Skia paragraph style family=FontFamily.Default themePath=/system/fonts/ShuS-SC.ttf resolved=true
```

这证明字体更新不是依赖进程重启。

### 4.3 恢复鸿蒙黑体

证据：

- [默认字体渲染截图](CMP系统主题字体-P3证据/p3-default-font-cold.jpeg)
- [默认字体日志](CMP系统主题字体-P3证据/p3-default-font-cold-font.log)
- [系统字体恢复截图](CMP系统主题字体-P3证据/p3-restore-final-settings.jpeg)

关键日志：

```text
fontId=22200001 default=true stylish=0 candidates=2
descriptor fullName=HarmonyOS Sans SC family=HarmonyOS Sans SC path=/system/fonts/HarmonyOS_Sans_SC.ttf
resolved fontId=22200001 path=/system/fonts/HarmonyOS_Sans_SC.ttf
Skia paragraph style family=FontFamily.Default themePath=/system/fonts/HarmonyOS_Sans_SC.ttf resolved=true
```

最终设备保持在“鸿蒙黑体”，并在应用首页确认已恢复无衬线字形。

## 5. 最终产物

| 项目 | 路径 / 值 |
| --- | --- |
| 最终 HAP | `/Users/jinshuo/Desktop/Workspace/CPF_CMP/kmp-cmp-test-demo/harmonyApp/hap-archives/20261030040923_CMP系统主题字体P3-Skia默认字体恢复修复_release_E.hap` |
| HAP SHA-256 | `3d61b4e16a10a7e88794f93d693cfba18cf3cffac1f2ec6c4e667c9ac9d6c85f` |
| 包内 `libkn.so` SHA-256 | `8c4bf1a98662be45451cf1134df4b764ff21a88a56e6358885dfd48fb8d4dcc6` |
| `compose.har` SHA-256 | `0f3c830a3a948e03fe7f6a21856d94724c0ed4f9da933df0998dfadb6ce1c72d` |
| HAP 大小 | `105,371,989` bytes |
| 归档清单 | `/Users/jinshuo/Desktop/Workspace/CPF_CMP/kmp-cmp-test-demo/harmonyApp/hap-archives/20261030040923_CMP系统主题字体P3-Skia默认字体恢复修复_release_E.json` |
| 构建选项 | `E`，release，SkiaRender |
| 构建结果 | `stages_ok=true failed_stage=none exit=0` |

## 6. 源码与提交

Core 仓库本地提交：

```text
0c5dfd15a25 fix(ohos): refresh cached text styles when theme font resolves
20c62160a02 fix(ohos): trigger Skia font refresh through host
9cdee445fca fix(ohos): resolve default theme font from generic family
4e2b324b1dc fix(ohos): prefer stylish theme font descriptors
f92a20778bd fix(ohos): follow theme font on skia
0f4b30fbf00 fix(ohos): resolve skia theme font from default font file
9b49654d0b5 fix(ohos): resolve active theme font descriptor
eb089bb1c9a fix(ohos): keep theme font resolver ArkTS strict
4b509826550 fix(ohos): request theme font path on startup
cef517de6d5 fix(ohos): preserve explicit system font family
```

Skiko 原型提交：

```text
fd58f6c0 fix(ohos): let skia font collection use theme default
```

该 Skiko 原型不是最终生效方案，原因是当前 Skiko worktree 缺少 `skia-pack`，对应 OHOS Skia 下载返回 404，无法在本轮闭环重编发布。最终方案不依赖该未构建提交。

本次所有 CMP 提交均只在本地，未 push；未提交签名配置；未写 Git hooks。

## 7. 验证限制与残余风险

1. `DEFAULT_FONT_ID = 22200001` 是当前设备/系统版本的出厂字体 ID。其他产品或系统版本若使用不同默认 ID，需要再抽象默认态识别。
2. 默认字体 family 优先匹配简体中文 `HarmonyOS Sans SC`，再回退通用 `HarmonyOS Sans`；未覆盖所有语言区域下的主题字体映射。
3. 已覆盖 Default/SansSerif、默认/自定义字体冷启动和同进程双向切换；尚未形成空行、trim、baselineShift、TextField 光标/选区、maxLines/省略、多窗口及高频切换的完整度量矩阵。
4. Kotlin/Native 测试源码中新增了 `SkTextStyleThemeFontCacheKeyTest`，但当前 OHOS test runner 因仓库既有并发测试源码错误无法完成全量 test task；生产编译、发布、HAP 构建和真机验证均通过。
5. Lint 的 `ktlint`/`detekt` 在目标 Kotlin 文件中仍报告既有格式和复杂度基线问题；`codelinter` 的既有 PATH 包装指向不存在的 DevEco 5.0.4。最终 ArkTS 代码由 Hvigor 编译通过。
6. Sample 仓库存在构建产物和 HAR 替换产生的未提交文件，这些不是本次业务源码修复的一部分。

## 8. 与前序报告的关系

P2 报告中“SkiaRender 未支持”描述的是 P2 当时验证到的边界，不是最终产品范围。P3 已完成 Skia 实现和真机闭环，该旧限制已被本报告取代。

P0/P1/P2 的 Fusion 结论继续有效；本报告补齐 SkiaRender，因此原始“CMP 不跟随系统字体”问题在本轮验收范围内已关闭。
