# 评审报告：CPF-CMP-Harness 专属工作流

> 依据：`ai-coding-best-practices`（五模型 + 五条硬性规则 + 四条落地原则）
> 对象：`/Users/jinshuo/Desktop/Workspace/CPF-CMP-Harness`
> 日期：2026-10-29

---

## 0. 一句话结论

**这是一个工程质量很高、知识治理很清醒的「实现侧」框架，但对齐侧是空的。**

它把「装什么知识、跑什么门禁、留什么证据」做到了工业级；却跳过了本方法论认定偏差主因的那一步——**在动手之前把需求缺口填掉**。结果是：门禁越严密，「所有检查都通过、但需求仍然是错的」这类偏差越难被发现（模型 1 阶段 3）。

| 维度 | 判定 |
|---|---|
| 需求对齐侧（HITL） | ❌ 缺失（CPF_CMP 路径）；部分（Harness 维护路径） |
| AFK 任务设计 | ⚠️ 一半合格（退出条件优秀，验收者缺位） |
| 实现侧（竖切 / Seam / TDD） | ❌ 缺失 |
| 上下文预算与知识沉淀 | ✅ 优秀，但存在死配置与多份真相 |
| 腐败风险控制 | ⚠️ 存在 3 个假设固化点 |

---

## 1. 评审范围与方法

### 1.1 使用的判据

| 模型 / 规则 | 用于判断 |
|---|---|
| 模型 1 软件腐败 8 阶段 | 哪些设计会把「未验证假设」合法化 |
| 模型 2 四类需求缺口 | 对齐侧是否真的在填补缺口 |
| 模型 3 AFK/HITL + 六要素 | 每个 task 的模式标注与通过条件 |
| 模型 4 对齐厚 / 实现薄 | 结构配比是否合理 |
| 模型 5 上下文边界 | 装载策略与知识单一真相源 |
| 硬性规则 1–6 | 逐条对照 |

### 1.2 审查对象

`AGENTS.md`、`README.md`、`route.yaml`（13 个 task_types / 45 个 aliases）、`docs/ai-agent-loading-guide.md`、`profiles/cpf-cmp.yaml`、`skills/manifest.yaml`、`skills/project/*`（8 个）、`skills/harness/cpf-cmp-harness-maintain`、`skills/external/*`（12 个）、`recipes/*`（14 个）、`contexts/*`、`standards/*`（15 个）、`verification/gates/*`（11 个）、`experience/`（active 8 / candidate 14）、`scheme/designs`（35）+ `scheme/verify`（25）、`templates/*`、`tools/weave/src`。

### 1.3 采集到的量化证据

| 指标 | 值 | 采集方式 |
|---|---|---|
| task_types / aliases | 13 / 45 | `python3 -c ... route.yaml` |
| 规范性约束词 | `必须` ×428、`不得` ×159、`禁止` ×153（AGENTS + 装载指南 + skills + recipes） | `grep -ro ... \| wc -l` |
| 核心文档体量 | `AGENTS.md` 63 行、装载指南 366 行、`route.yaml` 641 行 | `wc -l` |
| `stagesOk` 契约分布 | 8 处文档 + 4 处代码 | `grep -rl stagesOk` |
| `contextBudget.maxRefs` | `route.yaml:4` 声明，`tools/weave/src` 无任何读取 | `grep -rn maxRefs tools/weave/src` |
| weave 测试 | 26 个 `.mjs` 测试文件 | `ls tools/weave/tests` |

---

## 2. 逐项评审

### 2.1 架构骨架（模型 4）— ✅ 结构判断正确

`AGENTS.md` 开篇就把任务分成三条互斥路径，并明确**禁止过度装载**：

- 路 A：改 Harness 仓 → 只走维护 skill
- 路 B：改 CPF_CMP 源码 → 才装载 `docs/ai-agent-loading-guide.md`
- 路 C：解释/问答 → 直接做，**不要问「要不要装载」**

这一条比多数项目做得好。它实际是一种**注意力预算**机制：把「读什么」从「用户要求」变成「意图判定」，避免了「一上来通读 366 行装载指南」的典型浪费。这条路走对了。

`route.yaml` 只返回 refs 不展开正文（`policy: refs_only`），也符合模型 5 的核心结论：不要把所有资料塞进窗口，按需拉取。

**但**：这条链完整地止步于「装载资料」，没有进入「确认需求」。

---

### 2.2 需求对齐侧（模型 2 + 硬性规则 1/3）— ❌ CPF_CMP 路径为空

#### 做得对的两小块

1. **`cpf-cmp-harness-maintain` 的 P1 真硬停**：
   > 「用户回答前禁止：修改仓库、启动 Task 子 agent、全库复用审计、写方案/落盘、进入 P2+」
   并要求「至少 3 个实质问题；成功标准、作用范围、兼容性三类至少各 1 问」。
   这是全仓唯一真正落实「不要接受『我懂了』」的地方，值得保留。

2. **`templates/requirements-intake.template.md`** 有「§5 待用户确认项」，`author_core_spec` route 要求 intake 确认后才写正式规格。

#### 缺口一：源码修改路径完全没有对齐环节（P1，阻塞）

`recipes/edit-source-code.md` 的「硬序」是：

```
1. route resolve → 2. 识别目标子工程 → 3. 域分派 → 4. 实施源码修改
5. lint → 6. 本地 commit → 7. 询问 build-run
```

**第 3 步到第 4 步之间，没有任何「确认要做什么」的动作。** 用户说「给 demo 加按钮」，agent 拿到的全部信息就是这句话本身，然后直接改码。

对照硬性规则 3：

> **需求未对齐时不要开始实现。** 假设一旦写进代码并被测试合法化，就变成虚假的确定性。

这里更糟：假设写进代码后，还会被**自动本地 commit**（硬序 6）固化，然后被 build gate 的 `stagesOk == true` 合法化。

#### 缺口二：P1 的提问不是 BFS，且不落盘（P1）

维护 skill P1 是「一轮提问 → 用户回答 → 输出冻结说明」。问题是：

- 提问清单是固定的三类（成功标准 / 范围 / 兼容性），**不是按设计决策树逐层展开**。分支问题（「如果 A 则 B 还是 C」）在同一轮里问不深。
- **冻结说明不落盘**。`scheme/designs/` 是实施方案，不是对齐记录；删除模式的计划甚至明确禁止落盘。这意味着：**上下文一压缩，对齐状态就丢了**，后续 agent 只能靠冻结说明的对话残留继续。

对照 `align` / `grilling` 的核心机制：每轮落盘需求树，压缩代价才低。这条机制在这里缺席。

#### 缺口三：没有 `CONTEXT.md`，也没有 ADR（P2）

仓库有三类知识载体，但都不承担「统一语言 + 决策理由」的职责：

| 载体 | 实际职责 | 是否可替代 CONTEXT.md / ADR |
|---|---|---|
| `contexts/*.md` | 项目事实背景（repo map、工具链、渲染架构） | 否，是「事实」不是「语言」 |
| `scheme/designs/*.md` | 单个功能的实施方案 | 否，是方案不是被推翻的决策记录 |
| `experience/active/*.md` | 误判防护清单 | 否，粒度是「经验」 |
| `templates/requirements-doc.template.md` §3 | 术语表，但只存在于单篇规格内 | 否，不跨文档共享 |

后果：`strokeWidth`、`safe-area`、`split-mode`、`rendererBackend` 这类术语的准确含义，靠 agent 每次重新从 `contexts/` 里推断。这正是「未知的已知」没有被沉淀的表现。

---

### 2.3 AFK 设计（模型 3）— ⚠️ 退出条件优秀，验收者缺位

#### 做得好的：这是全仓最强的一块

`recipes/long-loop-task.md` + `standards/loop-engineering-policy.md` 几乎完整覆盖了 AFK 六要素：

| AFK 要素 | 对应实现 |
|---|---|
| 可度量目标 | 阶段表：每阶段有「退出条件」 |
| 通过条件 | `verification/gates/*.yaml` 的 `passCriteria` |
| 执行步骤 | 阶段表 5 阶段 + 重试预算（同命令最多重试 1 次、同验证最多修 3 轮） |
| **退出条件** | 阻塞七分类 + 「同一阻塞连续复现才标阻塞」 |
| 交付产物 | 检查点格式 + 交接 7 段式 |

`standards/validation-policy.md` 里那句话是全仓质量最高的表述：

> **skip ≠ pass。** 当关键运行时 gate 被跳过时，状态不能写成无条件完成。

以及 `docs/implementation/harness-benefit-validation.md` 主动划出解释边界：

> 不能说「Harness 已量化证明 agent 效率提升」。

这种「拒绝虚假确定性」的自觉，正是模型 1 阶段 3 的解药。

#### 问题一：13 个 task 没有一个是「标注过模式的」（P1）

模型 3 的硬性规则 2：

> **开工前标注工作模式。** AFK 或 HITL。说不清客观验收条件的任务，不配 AFK。

`route.yaml` 的 13 个 task 里，只有 `user_script_build_run` / `ohos_compose_test` / `ohos_performance_test` 具备**客观**通过条件（退出码、`stagesOk`、用例级结果）。但它们在 schema 上与 `review_core_change`、`author_user_manual`、`submit_gitcode_pr` **长得完全一样**——都是 refs 数组，没有任何字段说明「这个任务谁验收、怎么验收」。

#### 问题二：主观任务用了 AFK 自评门禁（P1）

`verification/gates/review.gate.yaml`：

```yaml
passCriteria:
  blockingFindings: 0
```

「blockingFindings 为 0」由谁判定？由跑 review 的那个 agent 自己。这是教科书级的：

> **客观上达到通过条件 ≠ Agent 认为达到通过条件。**

同理 `experience-promotion.gate.yaml` 的 `reviewerApproved: true`——`reviewer` 字段是一个字符串，agent 完全可以自己填。

正确做法不是加强 schema，而是**降级为 HITL 验收门禁**：主流程 AFK，报告产出后由人判定 `disposition`。

#### 问题三：`review.gate.yaml` 与 `route.yaml` 直接冲突（P1，阻塞）

三处对同一件事的表述互相矛盾：

| 位置 | 说法 |
|---|---|
| `route.yaml` → `edit_source_code.verification` | 「Review is **not required** for this task (manual `review_*` only)」 |
| `verification/gates/review.gate.yaml` → `skipPolicy` | 「Review **may not be skipped** for product source changes」 |
| `standards/review-policy.md` | 「评审硬门禁」（面向 PR/MR 合入） |

从设计意图推测，三者边界是「本地 commit 不评审 / PR 合入必须评审」。但这个边界**没有任何一处写清楚**，agent 读到哪份就信哪份。

在模型 1 的框架里，这不是文案问题：它意味着**唯一的「需求做对了吗」关卡，在两个入口上被静默移除**。

---

### 2.4 实现侧（模型 4 下半 + 原则 3/4）— ❌ 三件套全部缺席

| 要素 | 状态 | 证据 |
|---|---|---|
| Tracer Bullet（竖切） | ❌ 无 | 全仓 skills/recipes 无「竖切 / 最薄路径 / 端到端」概念 |
| Seam | ❌ 无 | 无「先找可替换连接点」的步骤 |
| TDD 判定 | ❌ 无 | `edit-source-code.md` 只有 lint/commit/build 三段收尾 |
| 测试能否校验需求 | ⚠️ 只有评审侧有 | `cpf-review` D 轴质量很高，但只在评审时用 |

`recipes/edit-source-code.md` 的 126 行里，「验证」的全部内容是：lint（`ktlint` / `detekt` / `codelinter` / `clang-tidy`）+ 构建 `stagesOk`。

**这两者都只能证明「代码能编译、风格合规」，不能证明「需求做对了」。** 对 CMP 这种跨仓、native + 双渲染后端 + 平台边界模糊的项目，横切式改动的返工成本极高——恰恰是最需要竖切和 Seam 的场景。

#### 值得表扬的一点

`skills/project/cpf-review/SKILL.md` 的 **D 轴（测试完整性）** 写得比大多数团队的评审规范都好：

> 「检查断言能够在旧实现或错误实现下失败」
> 「不允许用固定用例数量证明『已经全量执行』」

这几乎逐字对应模型 1 阶段 3 和原则 4「测试的难点是能否准确校验需求」。

**结论：能力已经有了，只是放错了阶段。** 它在「事后评审」，而应该在「事前实现」。

---

### 2.5 上下文预算与知识治理（模型 5）— ✅ 优，但有 3 处漏气

#### 优秀

- `experience/` 四态（raw / candidate / active / retired）+ **晋级门禁**（`source_tasks` + `verification_evidence` + `reviewer` + `review_after`）+ doctor 强校验「route 不得引用 raw/candidate」。这是本方法论「把一次性注意力投入转化为长期资产」的**最佳实现**，比原框架只有 `CONTEXT.md` + ADR 更可运营。
- `alignment` 前置的「禁止过度装载」清单，避免了「对齐前先花 5000 token 读环境」。
- `scheme/designs` + `scheme/verify` 成对落盘 + 双 agent 隔离审视（P3a / P3b 禁止 resume 冒充），对应 `review` skill 的「双轴必须独立 subagent」要求，落地得比原框架更硬。

#### 漏气一：`contextBudget` 是死配置（P2）

```yaml
# route.yaml:3-7
"contextBudget": { "maxRefs": 16, "maxExpandedTokens": 0, "policy": "refs_only" }
```

`grep -rn maxRefs tools/weave/src` → 无命中。`route.ts` 只把它透传到输出（`route.ts:281/309/356/511`），唯一断言来自测试文件 `route-resolve.test.mjs:56` 检查 `policy === "refs_only"`。

即：**`maxRefs: 16` 从未被强制执行**。当前 `edit_source_code` 的 refs 总量（4 contexts + 6 standards + 1 recipe + 1 skill + 3 gates + 3 experience ≈ 18）已经超了。声明而不执行，比不声明更危险——它制造「预算已被控」的错觉。

#### 漏气二：单一契约分散在多处（P2）

`build` gate 的 11 个必填字段，正文出现在：`route.yaml`、`recipes/user-script-build-run.md`、`skills/project/cpf-cmp-build-run/SKILL.md`、`standards/validation-policy.md`、`verification/gates/build.gate.yaml`、`verification/schemas/build.schema.yaml`、3 份 `scheme/`、`docs/observability/langfuse-setup.md`。

这正是本仓自己 `AGENTS.md` 里写的反面：

> 规则只在上面的文件里定义一次……重复会导致多份真相漂移。

#### 漏气三：规范密度过高（P2）

`必须` ×428 / `不得` ×159 / `禁止` ×153 = **740 处约束性表述**，而 `AGENTS.md` 只有 63 行（靠表格压缩）、装载指南 366 行。

对齐侧「厚」是对的（模型 4），但**厚度应该体现在「把决策问清楚」，而不是「把禁止事项列全」**。当前的结构是：用约束清单替代对齐过程。约束清单的边际收益递减，而它的成本是 agent 的注意力——恰恰是本方法论认为最稀缺的资源。

---

### 2.6 腐败风险点（模型 1）— 3 个假设固化入口

| # | 位置 | 固化机制 | 后果 |
|---|---|---|---|
| 1 | `recipes/edit-source-code.md` 硬序 6 | 每次源码改动**自动本地 commit**，不询问 | 未对齐假设进入可追溯历史；后续 agent 把 commit 当既成事实（阶段 4「其他模块依赖」） |
| 2 | `route.yaml` → `edit_source_code` | review 显式「not required」 | 唯一的「需求对不对」关卡被移除，只剩「能不能过 lint/build」 |
| 3 | `verification/gates/build.gate.yaml` | `stagesOk: true && exitCode: 0` 即 pass | 绿色 = 完成的心智，是最容易被误读成需求已对齐的信号（阶段 3） |

三个入口叠起来形成的闭环是：

```
用户一句话 → 直接改码 → lint 过 → 自动 commit → build 过 → 报告「完成」
```

**这条链上没有任何一步在问「用户要的是这个吗」。**

---

## 3. 结论

### 3.1 阻塞项（必须修）

| # | 级别 | 位置 | 问题 | 建议 |
|---|---|---|---|---|
| B1 | P1 | `recipes/edit-source-code.md` 硬序 3→4 | 源码修改路径无任何需求对齐门禁 | 见 4.1 |
| B2 | P1 | `review.gate.yaml` / `route.yaml` / `review-policy.md` | review 是否必做三处冲突 | 明确拆成「本地改动不评审 / PR 合入必评审」，三处同步措辞 |
| B3 | P1 | 全仓 | 13 个 task 无 `mode`（AFK/HITL）标注；主观任务用 agent 自评门禁 | 见 4.2 |
| B4 | P1 | 实现侧 | 无竖切 / Seam / TDD 判定 | 见 4.4 |
| B5 | P2 | `route.yaml:4` | `maxRefs: 16` 声明但未实现 | 在 `route.ts` 强制截断+报错，或删除该字段 |

### 3.2 建议项（可讨论）

| # | 位置 | 建议 |
|---|---|---|
| S1 | 仓库根 | 新增 `CONTEXT.md`（统一语言）与 `docs/adr/`（决策理由），并在 `AGENTS.md` 写清与 `contexts/`、`scheme/` 的分工 |
| S2 | `cpf-cmp-harness-maintain` P1 | 冻结说明落盘为 `specs/<n>.tree.md` 或 `scheme/align/<slug>.md`，让对齐状态可跨压缩恢复 |
| S3 | `build.gate.yaml` 等 | 把 11 个字段的权威定义收敛到 `verification/schemas/build.schema.yaml`，其余位置改为引用 |
| S4 | 全仓 | 740 处约束表述做一次「合并同类项」，目标砍掉 30% |
| S5 | `edit-source-code.md` | 自动 commit 改为「先报告 diff 摘要 → 得到确认再 commit」 |

### 3.3 通过项（做对了，不要改）

- **门闸三路径互斥设计**：把「读什么」变成意图判定，符合注意力预算原则。
- **`long-loop-task` 的阶段表 + 检查点 + 重试预算**：AFK 退出条件的工程化范本。
- **`standards/validation-policy.md` 的「skip ≠ pass」与「历史经验不能当 gate 证据」**：直接对冲虚假确定性。
- **`experience/` 四态 + 晋级门禁 + doctor 强校验**：把「沉淀」从口号变成可审计机制。
- **`scheme/designs` + `scheme/verify` 成对 + P3 双 agent 隔离**：比原框架的双轴审查更硬。
- **`harness-benefit-validation.md` 主动声明「proxy 覆盖 ≠ 效率提升」**：罕见的自我设限。
- **`cpf-review` 的 D 轴**：测试完整性判据质量高于行业均值。

---

## 4. 最小改动方案（按投入产出排序）

### 4.1 在 edit 路径插入「对齐三问」（0.5 天，收益最大）

在 `recipes/edit-source-code.md` 硬序「3. 域分派」与「4. 实施源码修改」之间插入一个 **HITL 轻门禁**，不新建复杂流程：

```md
### 3.5 对齐确认（HITL，必做）

在改码前用一句话复述并请用户确认：
1. 目标状态：改完后**用户看到什么**（不是改哪个文件）
2. 非目标：这次**不改**什么
3. 验收方式：用户怎么判断做对了

若三条中有任一条无法从用户原话直接得出 → 提问，不要猜。
升级条件（满足任一则改走 /align 级 BFS）：
- 改动跨越 ≥2 个 git 子仓
- 触及 public API / Gradle DSL / native ABI
- 触及渲染路径或并发模型
```

**为什么这条最划算**：它对现有 13 个 task、45 个 alias、所有 gate 零改动，只是在「改码」前加了一个 3 行确认。

### 4.2 给 task_types 加两个字段（1 天）

```yaml
edit_source_code:
  mode: hitl            # afk | hitl | afk-with-hitl-gate
  acceptance: objective # objective | subjective
```

- `objective` → 门禁保持机检
- `subjective` → `verification` 字段必须写明「由人验收」，且 gate 的 `passCriteria` 不允许出现 agent 自评字段
- `route resolve` 输出透传；`weave doctor` 校验：`acceptance: subjective` 的 task 若引用了 `blockingFindings: 0` 类自评 gate → **fail**

### 4.3 收敛 review 边界（0.5 天）

```
本地改动（未推送）      → 不评审，但报告必须列出 diff 摘要
本地 commit             → 不评审
PR / MR 合入            → 必评审（review-policy.md 硬门禁）
```
三处（route.yaml / review.gate.yaml / review-policy.md）用**同一句话**表述。

### 4.4 把 D 轴前移到实现侧（2 天）

在 `edit-source-code.md` 的「4. 实施源码修改」内部，加一节，形式是其已有 D 轴的简化版：

```md
实施前先列一张表（不落盘，报告里给出）：

| # | 改动段落 | 位置 | 需要先写测试? | 理由 |
|---|---|---|---|---|
| 1 | <行为契约> | path:line | 是 | 错了不会报错（静默失败） |
| 2 | <装配/胶水> | path:line | 否 | 出错立刻编译失败 |

若超过一半段落都标记「是」→ 说明这次是横切改动，停下来重新切片。
```

**加上「最薄端到端路径」一句**：改完后先说清「用什么操作能看到结果」，再动手。

### 4.5 补 `CONTEXT.md` + `docs/adr/`（0.5 天，持续维护）

| 文件 | 写什么 | 不写什么 |
|---|---|---|
| `CONTEXT.md` | 术语表（`split-mode`、`safe-area`、`rendererBackend`、`strokeWidth`…）、代码组织约定、五仓职责 | 事实背景（留给 `contexts/`）、方案（留给 `scheme/`） |
| `docs/adr/NNNN-*.md` | 被推翻或确立的**决策 + 理由**，例如「为什么 review 在本地 commit 不强制、在 PR 强制」 | 实施步骤 |

ADR 的第一条建议就写 B2 的结论——它能防止 B2 复发。

### 4.6 死配置处理（0.2 天）

`route.ts` 在 `resolveRouteRefs` 返回前检查 `refs` 总长度 > `contextBudget.maxRefs` → 抛错并提示「请下沉 refs 到 skill 内部按需加载」。或直接删除 `contextBudget` 字段，避免假承诺。

---

## 5. 值得反向吸收到 `ai-coding-best-practices` 的 5 点

这个项目有三个地方比参考框架更成熟，建议回灌：

1. **经验四态 + 晋级门禁**（`raw → candidate → active → retired`，配 `source_tasks` / `reviewer` / `review_after`，且 doctor 强制 route 不得引用未晋级经验）。
   → 原框架的「沉淀」只给了 `CONTEXT.md` + ADR，缺「经验如何被验证后才可信」的机制。

2. **gate 的 `skipPolicy` 表述**：「skip ≠ pass」「缺工具不计入修复轮次」「不能用固定用例数量证明已全量执行」。
   → 可直接补进 `/afk` 的通过条件模板。

3. **「proxy 覆盖 ≠ 效率提升」的解释边界**：主动声明「本验证只能证明覆盖，不能证明收益」。
   → 这正是防模型 1 阶段 3 的具体写法，建议作为 `/review` 的报告条款。

4. **阶段表 + 检查点 + 重试预算**：把 AFK 的「退出条件」从一句话变成可执行的三件套（阶段退出条件 / 检查点 6 字段 / 重试预算上限）。
   → 建议升级 `/afk` 模板的「执行步骤」与「退出条件」两节。

5. **`refs_only` 路由**：把「装载什么」从 skill 内硬编码改为运行时按 task 解析的引用列表。
   → 对 `/align` 的「每轮落盘需求树」，可借鉴此机制做「需求树节点 → 按需展开」。

---

## 6. 证据清单

```bash
# task 数量
python3 -c "import json;d=json.load(open('route.yaml'));print(len(d['task_types']),len(d['aliases']))"
# → 13 45

# 规范密度
grep -ro "必须" AGENTS.md docs/ai-agent-loading-guide.md skills recipes | wc -l   # 428
grep -ro "不得" AGENTS.md docs/ai-agent-loading-guide.md skills recipes | wc -l   # 159
grep -ro "禁止" AGENTS.md docs/ai-agent-loading-guide.md skills recipes | wc -l   # 153

# 死配置
grep -rn "maxRefs" tools/weave/src route.yaml
# → 仅 route.yaml:4 声明，src 无读取

# 契约分散
grep -rl "stagesOk" . --exclude-dir=node_modules --exclude-dir=dist --exclude-dir=.git | wc -l
# → 16（8 文档 + 4 代码 + 4 examples/schema）
```

**未能自动验证、需人工确认的项**：

- 实际使用中 `edit_source_code` 是否真的在无对齐的情况下改码 → 需抽查 3 条会话记录
- `review.gate.yaml` 是否实际被任何 route 挂载 → 需跑 `weave route resolve --task core/review --json` 观察 `verification` 字段
- `blockingFindings: 0` 在实际评审中由谁判定 → 需抽查 2 份评审报告
