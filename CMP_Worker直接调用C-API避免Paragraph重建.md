# C API 能否让 Worker 避免 Paragraph 重建

> - 分析日期：2026-11-16
> - 分析对象：CMP OHOS SkiaRender
> - 参考提交：`cd98e8a3c532fb783e8b6ca796e0c2e73ad547a4`
> - 红色节点：方案的阻塞点

## 0. 明确回答

想在 Worker 中不经过 NAPI，直接调用一个公开 C API，取得“当前 ArkUI 已选主题字体的文件路径”：

> 基于 DevEco Studio 6.0.0 当前 SDK 头文件和 CMP 源码，没有找到这样的公开 C API。

已核对的相关能力只有：

| 已有 API | 实际结果 | 是否满足需求 |
| --- | --- | --- |
| `napi_create_threadsafe_function()` / `napi_call_threadsafe_function()` | 把请求异步投递到 NAPI owner thread | 否，仍然是 NAPI 路径 |
| `OH_Drawing_GetSystemFontFullNamesByType()` | 枚举可用字体 fullName | 否，不能说明当前主题选中了哪项 |
| `OH_Drawing_GetFontDescriptorByFullName()` | 从指定 fullName 取得 `descriptor.path` | 部分满足；前提是已经知道正确 fullName |
| `OH_Drawing_FontSetThemeFontFollowed()` | 让 `OH_Drawing_Font` 内部跟随主题字体 | 否，不返回字体路径 |

SDK 中当前主题选择对应的 `fontId` 位于 ArkTS 的 `Configuration.fontId`；Native 头文件中没有对应的直接 getter，也没有 `fontId → 当前主题字体路径` 的公开 C API。

## 1. 结论

Worker 通过 C API 获取字体，不能保证避免 Paragraph 重建，主要有两个阻塞点：

1. Drawing 字体对象和 Skia Paragraph 使用的字体对象不是同一套对象。
2. ArkTS 字体路径解析是异步的，Worker 调用 C API 时解析不一定已经完成。

需要特别澄清：

- `OH_Drawing_GetFontDescriptorByFullName()` 返回的 `descriptor.path` 可以使用。
- 当前项目也可以取得字体 fullName，ArkTS 已经在使用 `getSystemFontFullNamesByType()`。
- 真正的问题是：`fontId` 不能直接唯一映射到当前主题字体的 fullName；枚举得到的字体列表也不等于“当前用户选择的字体”。

当前 ArkTS Resolver 已经同时使用：

```text
text.getSystemFontFullNamesByType(GENERIC / STYLISH)
  → 取得 fullName
text.getFontDescriptorByFullName(fullName, fontType)
  → 取得 descriptor.path
```

所以不是“项目拿不到 fullName”，也不是“descriptor.path 不能用”。

## 2. 具体可用的 C API

```mermaid
flowchart LR
    subgraph API1["跨线程 NAPI：把请求交回主线程"]
        direction TB
        N1["napi_create_threadsafe_function()<br/>API 10"] --> N2["napi_call_threadsafe_function()<br/>API 10"]
        N2 --> N3["NAPI owner thread 执行回调"]
        N3 --> N4["调用 ArkTS<br/>__font_getSystemThemeFontPaths()"]
        N4 --> N5["读取当时已写入的 cachedPaths"]
    end

    subgraph API2["Drawing 字体枚举：查询可用字体"]
        direction TB
        D1["OH_Drawing_GetSystemFontFullNamesByType()<br/>API 14"] --> D2["GENERIC / STYLISH / INSTALLED / ALL"]
        D2 --> D2A["OH_Drawing_GetSystemFontFullNameByIndex()<br/>API 14"]
        D2A --> D3["OH_Drawing_GetFontDescriptorByFullName()<br/>API 14"]
        D3 --> D4["得到 OH_Drawing_FontDescriptor<br/>其中包含 descriptor.path"]
        D4 --> D5["descriptor.path 可以作为字体文件路径使用"]
        D5 --> D6{"能否由当前 fontId<br/>唯一确定 fullName？"}:::problem
    end

    subgraph API3["Drawing 主题字体：让 Drawing 字体跟随主题"]
        direction TB
        T1["OH_Drawing_FontSetThemeFontFollowed(font, true)<br/>API 15"] --> T2["仅作用于 OH_Drawing_Font<br/>Typeface 未设置时使用主题字体"]
    end

    D6 -->|"否"| X["无法保证选中的是<br/>ArkUI 当前主题字体"]:::problem
    N5 --> X
    T2 --> Y["Drawing 字体对象<br/>不能交给 Skia Paragraph"]:::problem
    X --> NAPI["自行枚举字体路径<br/>不能替代当前主题选择"]:::problem

    style API1 fill:#fafafa,stroke:#78909c,stroke-width:1px,stroke-dasharray:6 4
    style API2 fill:#fafafa,stroke:#78909c,stroke-width:1px,stroke-dasharray:6 4
    style API3 fill:#fafafa,stroke:#78909c,stroke-width:1px,stroke-dasharray:6 4
    classDef problem fill:#ffe5e5,stroke:#c62828,stroke-width:3px,color:#8b0000;
```

| 可用 API | 能做什么 | 不能解决什么 |
| --- | --- | --- |
| `napi_create_threadsafe_function()` / `napi_call_threadsafe_function()` | Worker 可把查询投递到 NAPI owner thread | 不会同步等待 ArkTS Promise，也不会直接读取 `napi_value` |
| `OH_Drawing_GetSystemFontFullNamesByType()` / `OH_Drawing_GetSystemFontFullNameByIndex()` | 可以获得可用字体的 fullName | 字体列表不等于当前主题选择 |
| `OH_Drawing_GetFontDescriptorByFullName()` | 可以从正确 fullName 取得 `descriptor.path`，该 path 可以使用 | 输入必须是 fullName，不能直接传当前 `fontId` |
| `OH_Drawing_FontSetThemeFontFollowed()` | 让 `OH_Drawing_Font` 在未设置 Typeface 时跟随主题字体 | 只作用于 Drawing 字体，不能更新 Skia Paragraph |

这三个问题需要分开：

```text
系统有哪些字体？
  → getSystemFontFullNamesByType()

指定某个 fullName，对应哪个文件 path？
  → getFontDescriptorByFullName(fullName, type)

当前 ArkUI 主题究竟选中了哪一个 fullName？
  → 字体枚举和 descriptor 查询都不能回答
```

如果已经知道“当前主题字体”的 fullName，那么 `OH_Drawing_GetFontDescriptorByFullName()` 可以取得它的 path。问题在于 fullName 列表没有当前选中标记，当前 `fontId` 也不能直接转换成 fullName。

当前能确定的信息可以分成三层：

| 信息 | 当前能否直接取得 | 来源 |
| --- | --- | --- |
| 当前主题选择的 `fontId` | 能 | ArkTS `Configuration.fontId` |
| 当前主题字体文件 path | 能 | 当前激活主题目录、`flag`、`manifest.json` |
| 系统可用字体的 fullName 列表 | 能 | `getSystemFontFullNamesByType()` 或 Drawing 枚举 API |
| “当前主题字体”对应的唯一 fullName | 没有直接映射 API | 需要用主题 path 反查，或直接使用已经取得的 path |

因此并不是“只能知道 fontId”。当前项目已经能取得当前主题文件 path；只是 system 没有提供一个这样的 API：

```text
Configuration.fontId
  → 当前主题字体 fullName
```

一旦通过激活主题目录和 manifest 取得了 path，Skia 就可以直接加载该 path，不再需要先转换出 fullName。

## 3. Drawing 对象体系与 Skia 对象体系

### 3.1 两套对象怎样对应

| 作用 | HarmonyOS Drawing C API | CMP SkiaRender |
| --- | --- | --- |
| 字体管理器 | `OH_Drawing_FontMgr` | `org.jetbrains.skia.FontMgr` |
| 字体集合 | `OH_Drawing_FontCollection` | `org.jetbrains.skia.paragraph.FontCollection` |
| 字形资源 | `OH_Drawing_Typeface` | `org.jetbrains.skia.Typeface`（SkTypeface） |
| 字体实例 | `OH_Drawing_Font` | `org.jetbrains.skia.Font`（SkFont） |
| 文本样式 | `OH_Drawing_TextStyle` | `org.jetbrains.skia.paragraph.TextStyle` |
| 文本构建器 | `OH_Drawing_TypographyCreate` | `org.jetbrains.skia.paragraph.ParagraphBuilder` |
| 最终排版结果 | `OH_Drawing_Typography` | `org.jetbrains.skia.paragraph.Paragraph`（SkParagraph） |

两边概念相似，但公开类型、ABI 和生命周期不同。即使 Drawing 内部实现也使用 Skia，`OH_Drawing_*` 对外仍是不透明对象，不能把它的指针当成 `SkTypeface*` 或 `SkParagraph*` 使用。

Drawing 对象由 `OH_Drawing_*Create()` 创建，并通过对应的 `OH_Drawing_*Destroy()` 释放；Skiko 的 `Typeface`、`Font`、`Paragraph` 则是 Kotlin 侧对 Skia native 对象的封装，由 Skiko 的引用计数和 finalizer 管理。这两套对象不能共享同一个可公开交换的指针所有权。

### 3.2 两套文本对象体系

```mermaid
flowchart TB
    subgraph DRAW["HarmonyOS Drawing 文本体系"]
        direction LR
        DC["OH_Drawing_FontCollection"] --> DTC["OH_Drawing_TypographyCreate"]
        DTS["OH_Drawing_TextStyle"] -.-> DTC
        DF["OH_Drawing_Font"] -.-> DTS
        DTF["OH_Drawing_Typeface"] --> DF
        DTC --> DT["OH_Drawing_Typography<br/>最终排版结果"]
    end

    subgraph SKIA["CMP SkiaRender：Skiko / Skia 文本体系"]
        direction LR
        SC["org.jetbrains.skia.paragraph.FontCollection"] --> SB["SkParagraphBuilder"]
        SS["SkTextStyle"] -.-> SB
        SF["SkFont"] -.-> SS
        ST["SkTypeface"] --> SF
        SB --> SP["SkParagraph<br/>最终排版结果"]
    end

    DTF -.->|"没有公开 ABI 桥接"| BRIDGE["OH_Drawing_Typeface<br/>不能当 SkTypeface 使用"]:::problem
    ST -.-> BRIDGE

    PATH["另一条路线：字体文件 path 字符串"] --> LOAD["FontMgr.makeFromFile(path)"]
    LOAD --> ST
    LOAD -.-> PATHNOTE["path 是数据，可以交给 Skia 加载；前提是 path 对应正确字体"]

    style DRAW fill:#fafafa,stroke:#78909c,stroke-width:1px,stroke-dasharray:6 4
    style SKIA fill:#fafafa,stroke:#78909c,stroke-width:1px,stroke-dasharray:6 4
    classDef problem fill:#ffe5e5,stroke:#c62828,stroke-width:3px,color:#8b0000;
    classDef note fill:#f5f5f5,stroke:#9e9e9e,stroke-width:1px,stroke-dasharray:4 3,color:#424242;
    class PATHNOTE note;
```

这两条路线的区别是：

```text
传递对象：OH_Drawing_Typeface → SkTypeface
结果：没有公开可靠转换，不能直接使用

传递数据：descriptor.path → FontMgr.makeFromFile(path)
结果：可以使用，但必须先保证 path 是当前主题对应的正确文件
```

### 3.3 为什么 Drawing 主题字体不能直接进入 Skia Paragraph

```mermaid
flowchart TD
    subgraph OBJ1["阶段一：Drawing C API 实际做了什么"]
        direction LR
        A0["背景：Worker 要创建的是<br/>Skia Paragraph"] --> A1["调用 OH_Drawing_FontCreate()<br/>创建一个 Drawing 字体对象"]
        A1 --> A2["调用 FontSetThemeFontFollowed(font, true)<br/>告诉 Drawing：没有显式 Typeface 时使用主题字体"]
        A2 --> A3["输出：一个已开启主题跟随的<br/>OH_Drawing_Font"]
        A3 --> A4["它可以供<br/>OH_Drawing Typography / 绘制使用"]
        A3 -.-> A5["它不返回<br/>当前主题字体文件路径"]:::problem
        A3 -.-> A6["它不能直接成为<br/>Skia Paragraph 使用的 SkTypeface"]:::problem
    end

    subgraph OBJ2["阶段二：交给 SkiaRender 时发生什么"]
        direction LR
        B1["Skia Paragraph 创建链路"] --> B2["需要 Skia 自己的<br/>SkTypeface / SkFont / SkParagraph"]
        B3{"Drawing 字体对象与 Skia 字体对象<br/>是否同一套对象体系？"}:::problem
        B3 -->|"否"| B4["没有可靠公开桥接<br/>不能直接替换"]:::problem
        B3 -->|"即使能够转换"| B5["已有 Paragraph 仍需替换字体环境"]
    end

    subgraph OBJ3["阶段三：为什么最终仍然需要重建"]
        direction LR
        C1["已有 SkParagraph 不能原地改字体"]:::problem
        C1 --> C2["废弃旧 Paragraph"]
        C2 --> C3["重新创建 SkParagraph"]
        C3 --> C4["重新测量、断行和 draw"]
    end

    A3 --> B3
    A5 --> B3
    A6 --> B3
    B2 --> C1
    B4 --> C1
    B5 --> C1
    C1 -.-> NOTE1["说明：Paragraph 已包含字形、字宽、断行和文本尺寸，不能只替换一个字体路径"]

    style OBJ1 fill:#fafafa,stroke:#78909c,stroke-width:1px,stroke-dasharray:6 4
    style OBJ2 fill:#fafafa,stroke:#78909c,stroke-width:1px,stroke-dasharray:6 4
    style OBJ3 fill:#fafafa,stroke:#78909c,stroke-width:1px,stroke-dasharray:6 4
    classDef problem fill:#ffe5e5,stroke:#c62828,stroke-width:3px,color:#8b0000;
    classDef note fill:#f5f5f5,stroke:#9e9e9e,stroke-width:1px,stroke-dasharray:4 3,color:#424242;
    class NOTE1 note;
```

即使 Worker 拿到了 Drawing 主题字体，它也不是当前 Skiko Paragraph 直接使用的 `SkTypeface`。已有 Paragraph 不能原地换字体，仍然要创建新的 SkParagraph。

阶段一不是在“解析主题字体路径”。它只是创建一个 Drawing 字体对象，再设置一个“跟随主题”的开关。这个 API 不返回路径，输出的 `OH_Drawing_Font` 也只归 Drawing 文本系统使用；Skia Paragraph 无法直接拿它替换已有的 `SkTypeface`。

另一条“fullName → descriptor.path”路线确实能取得可用的 path，但前提是已经知道正确的 fullName。字体枚举 API 不能把当前 `fontId` 直接映射成 ArkUI 当前选择的主题字体。

## 4. 阻塞点二：C API 查询时 ArkTS 不一定解析完成

```mermaid
flowchart TD
    subgraph ASYNC1["阶段一：ArkTS 异步解析字体路径"]
        direction LR
        A1["requestSystemThemeFontPath(fontId)"] --> A2["异步读取主题槽位<br/>manifest / descriptor"]
        A2 --> A3["写入 cachedPaths"]
        A3 --> A4["通知 Compose"]
    end

    subgraph ASYNC2["阶段二：Worker 创建 Paragraph"]
        direction LR
        B1["Worker 准备创建 Paragraph"] --> B2["napi_call_threadsafe_function()<br/>请求主线程查询字体"]
        B2 --> B2A["主线程执行<br/>__font_getSystemThemeFontPaths()"]
        B2A --> B3{"查询时 ArkTS<br/>解析是否完成？"}
        B3 -->|"是"| B4["取得最新路径<br/>创建正确 Paragraph"]
        B3 -->|"否"| B5["返回空路径或旧路径"]:::problem
        B5 --> B6["先创建默认字体或旧字体 Paragraph"]:::problem
    end

    subgraph ASYNC3["阶段三：异步结果晚到"]
        direction LR
        C1["cachedPaths 稍后完成"] --> C2["字体身份变化"]
        C2 --> C3["旧 Paragraph 失效"]:::problem
        C3 --> C4["重新创建 Paragraph"]
        C4 --> C5["下一次 draw 显示新字体"]
    end

    A2 -.->|"解析仍在进行"| B2A
    A4 --> C1
    B6 --> C3
    B2A -.-> NOTE1["__font_getSystemThemeFontPaths() 只返回当前 cachedPaths<br/>不会等待 ArkTS Promise"]

    style ASYNC1 fill:#fafafa,stroke:#78909c,stroke-width:1px,stroke-dasharray:6 4
    style ASYNC2 fill:#fafafa,stroke:#78909c,stroke-width:1px,stroke-dasharray:6 4
    style ASYNC3 fill:#fafafa,stroke:#78909c,stroke-width:1px,stroke-dasharray:6 4
    classDef problem fill:#ffe5e5,stroke:#c62828,stroke-width:3px,color:#8b0000;
    classDef note fill:#f5f5f5,stroke:#9e9e9e,stroke-width:1px,stroke-dasharray:4 3,color:#424242;
    class NOTE1 note;
```

Worker 调用 C API 只能看到“调用这一刻已写入的字体结果”。如果 ArkTS 还在读取主题目录、manifest 或 descriptor，Worker 仍会先创建默认字体或旧字体 Paragraph；等路径完成后，必须重建。

## 5. 两条阻塞点的共同结论

```mermaid
flowchart LR
    P1["Drawing 字体对象<br/>不能直接用于 SkParagraph"]:::problem --> R["C API 不能保证<br/>避免 Paragraph 重建"]:::result
    P2["ArkTS 字体解析异步<br/>调用时结果可能未完成"]:::problem --> R

    classDef problem fill:#ffe5e5,stroke:#c62828,stroke-width:3px,color:#8b0000;
    classDef result fill:#fff3e0,stroke:#ef6c00,stroke-width:3px,color:#8a4300;
```

一句话总结：

> C API 可以改变字体信息的获取方式，但不能改变 Skia Paragraph 的对象模型，也不能保证调用时 ArkTS 已经解析完成。

## 6. 关键源码

| 作用 | 源码 |
| --- | --- |
| Worker 创建 Skia Paragraph | `compose/foundation/foundation/src/commonMain/kotlin/androidx/compose/foundation/text/modifiers/TextPrefetchSupport.kt` |
| Skiko Paragraph 创建 | `compose/ui/ui-text/src/skikoMain/kotlin/androidx/compose/ui/text/platform/ParagraphBuilder.skiko.kt` |
| Skia 字体集合注入 ParagraphBuilder | `compose/ui/ui-text/src/skikoMain/kotlin/androidx/compose/ui/text/font/SkiaFontLoader.skiko.kt` |
| ArkTS 异步字体解析和 `cachedPaths` | `compose/ui/ui-arkui/src/ohosArm64Main/cpp/compose/src/main/ets/compose/font/OhosSystemThemeFontResolver.ets` |
| Drawing 字体对象 | SDK `native_drawing/drawing_font.h`、`drawing_font_mgr.h` |
| Drawing 字体枚举和 descriptor | SDK `native_drawing/drawing_text_font_descriptor.h` |
| 跨线程 NAPI | SDK `napi/native_api.h` |
| Skia 对象定义 | `third_party/skiko/skiko/skiko/src/commonMain/kotlin/org/jetbrains/skia/Typeface.kt`、`FontMgr.kt`、`paragraph/ParagraphBuilder.kt`、`paragraph/Paragraph.kt` |

