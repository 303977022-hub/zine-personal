# ZINE STUDIO V6.1｜第一阶段审查报告与技术方案

日期：2026-11-25。范围：只审查和设计，未开发 V6.1 功能。

## 1. V6 实际基线

唯一代码基线为 `outputs/zine-studio-v6.html`，单文件 HTML/CSS/JavaScript，SHA-256：`94B57300EC7A0F18B59EF1E063D90AA1F90B4F109360B683E1BB7AAB0B136E93`。已复制相同内容到 `work/zine-studio-v6.1-dev.html` 作为独立开发起点；V6 原件未改动。

| 现有层 | 源码位置 | 实际结构与行为 |
| --- | --- | --- |
| 页面和素材 | `PG`、`ASSETS`、`mkPg`、`registerAsset` | `PG` 是页面数组；页面 `id` 等于数组下标，含 `blocks`、`overflowBlocks`、`paths`、背景、文字色、布局标识。素材存 Data URL，图片块用 `assetId` 和 `src` 引用。 |
| 自动成册 | `autoCompose` | 按素材导入顺序：空封面首图，其余每页两图；不读取主题、章节、文字或图片需求。跳过已上页素材，尽量保留已有内容。 |
| 版式及编辑 | `TEMPLATES`、`applyTemplate`、`rCanvas`、`rCtrl`、`transformBlock` | 7 种模板；按块类型匹配位置，多余块进入“待放置内容”。画布支持文字、引用、图片、蒙版、手绘及图片块移动缩放。 |
| 保存恢复 | `serializeProject`、`validateProject`、`restoreProject` | JSON `format: zine-studio-v6, version: 1`，存素材、页面、选中页和出血线状态；无策划信息。载入前校验。 |
| PDF | `exportPDF`、打印样式 | 渲染页面并调用浏览器打印；用户在打印窗口保存为 PDF。 |

设计约束：现有 `id` 会在增删页时重排，不能用作策划页的稳定标识；当前页面与文案块没有关联字段；`autoCompose` 直接改 `PG`，无法承担“用户确认大纲后才排版”的新流程；V6 JSON 不能直接存策划层。代码没有 AI API 配置或后端，不应在前端放密钥。

## 2. 建议架构

推荐 **在 V6 编辑器旁新增独立策划层，并用显式映射器生成画布**。继续沿用单文件起点，但把策划逻辑写成可独立测试的纯函数模块；不要在第一轮重写 `rCanvas`、图片交互和打印逻辑。

另两种方案：

- 直接给 `PG` 加策划字段：改动少，但大纲调序、文案更新容易与手动布局互相覆盖，不推荐。
- 全面迁移到新框架和服务端：模块边界清晰，但会扩大范围并增加 V6 编辑器回归风险，本轮不采用。

### 三层数据

```text
ProjectBrief
  id, rawIntent, decisions: {
    topic, audience, purpose, scope, pageCount,
    textStyle, materialStatus, narrativeMode, visualMood
  }
  每项 decision: {value, source: user|suggested|unknown, confirmed, evidence?}
  interview: {questionsAsked[], answers[]}, status: draft|confirmed

EditorialPlan
  id, briefId, status: draft|confirmed, revision
  title: {value, source, confirmed}
  chapters[]: {id, title, order, pageIds[]}
  pages[]: {
    id(稳定 ID), chapterId, order, role,
    title, body, imageNeeds: [{count, orientation, subjectHint, source}],
    layoutHint, unresolved[], confirmed
  }

LayoutDocument
  沿用 V6 的 PG/ASSETS/blocks/overflowBlocks/paths
  页面增加 planPageId（可为空）
  由策划生成的文字块增加 planField（title|body）与来源标记
  独立记录手动调整状态，不修改策划原文
```

`source` 区分用户提供、系统建议和未知；`confirmed` 单独表示用户是否接受该值。模拟模式也使用 `suggested`，不能标成 AI 已验证事实。个人经历只从用户原话或明确回答提取；缺失经历以待填写提示呈现。建议页数与章节分配可由系统提出，但在用户确认前不传入排版引擎。

### 可替换策划接口

`extractBrief(intent) → ProjectBrief`；`nextQuestion(brief, history) → question | null`；`proposeChapters(brief) → chapters`；`expandPages(brief, chapters) → EditorialPlan`。先用清楚标注的规则模拟实现；将来真实 AI 适配器必须返回同一数据结构并经过校验。访谈最多 3 问，只询问会改变主题、范围或叙事结构的未知项；已知页数、用途、文字量不得再问。用户可跳过并接受可修改建议。

### 策划到画布的映射

1. 用户确认 `EditorialPlan` 后，逐个稳定 `planPageId` 建立/关联 V6 页面；初始模板由 `role + imageNeeds + layoutHint` 决定，先复用现有模板，必要时调整有限规则，不以大量新模板代替策划。
2. `title`、`body` 生成带 `planField` 的文字块；图片需求生成空图片槽或从素材池手动匹配。本阶段不做 AI 选图。
3. 大纲改文案只更新对应文字块的 `content`，保留页面其他块及所有几何坐标、蒙版和手绘。直接在画布改文字时记录“画布文字改动”，避免下一次策划同步静默覆盖；冲突由用户选择保留哪一版。
4. 大纲改章节顺序时按稳定 `planPageId` 移动页面对象，不重建块。删除已生成且含编辑内容的策划页，应先显示受影响内容并保留可恢复副本；不能直接丢弃。
5. “重新排版”是独立按钮：预览受影响页面与手动调整，用户明确执行后才改变布局；普通大纲编辑绝不调用 `applyTemplate` 或 `autoCompose`。

## 3. 保存、恢复与兼容

新格式建议 `format: zine-studio-v6.1, version: 2`，含 `brief`、`editorialPlan`、`layoutDocument`（完整 V6 素材与页面数据）、`bindings`。保存前校验三层引用，载入时先解析、迁移和校验，成功后一次性替换内存状态。

读取 V6 版本 1 时，完整复制 `assets`、`pages`、`selectedPage`、`showGuides`；`brief` 与 `editorialPlan` 设为空，页面新增稳定内部 ID，但原有页面和块数据原样保留。界面明确显示“旧项目已导入，尚无策划大纲”，并允许用户日后建立大纲。不要臆造主题或把旧文字当作用户确认的策划。V6.1 保存为版本 2，不回写原版本 1 文件；保留原文件供回退。

## 4. 交互流程与实施阶段

| 阶段 | 工作内容 | 验收标准 |
| --- | --- | --- |
| A（本次） | 审查 V6、确定数据层和接口 | 本报告确认；V6 基线未改。 |
| B | 规则模拟策划 POC：输入、已知信息提取、最多 3 问、章节后逐页生成 | 已知信息不重复问；跳过可用建议；输出有来源和待确认标记；无真实 AI 伪装。 |
| C | 独立可视化大纲：章节和页面卡片，增删移动、文案与图片需求编辑 | 总页数实时正确；章节页数一致；未确认不能误触排版。 |
| D | 大纲确认、图片手动匹配、映射至 V6 页面；增量文案同步、显式重排 | 修改文案不改图片位置；章节重排不丢块；手动编辑冲突可见；V6 图片编辑不回归。 |
| E | JSON v2、V6 v1 迁移与完整回归 | 旧项目导入内容一致；往返恢复三层；损坏文件无部分覆盖；保存/PDF/图片操作可用。 |
| F | 十年友情 24 页案例验证与修正 | 按给定 3/3/5/3/8/2 页章节分配；近期素材需求较多；不编造经历；确认后生成可编辑页面。 |

每阶段提交目标、文件和模块改动、实际测试结果、失败或未验证项、V6 回归情况、风险与下一步，并等待验收再推进。

## 5. 测试计划及本次结果

- 纯函数测试：提取已知项、问题优先级与最多 3 问、来源标记、章节页数总和、页面稳定 ID、跳过行为、个人经历不编造。
- 状态同步测试：只改标题不改图片几何；调章节顺序后块 ID 不变；删除含内容页有恢复路径；显式重排前后差异可见。
- 兼容测试：V6 v1 → V6.1 v2 数据一致；v2 往返；无效引用、损坏文件和未知版本拒绝且原状态不变。
- 手动浏览器验收：策划对话、页面卡片编辑、图片拖动缩放、保存恢复、打印 PDF；浏览器结果逐项记录。

本次只运行现有基线检查：图片移动缩放及 V6 阶段 1–5 共 6 个 Node 测试脚本通过；提取的 V6 JavaScript `node --check` 通过。V6.1 功能测试 **未运行**，因为尚未开发。浏览器实测 **未验证**；现有自动化策略禁止代理访问本地 `file://` 页面。

## 6. 待确认的产品决策

最重要的一项：当用户在画布里手动改过某个标题，随后又在大纲中修改同一标题，系统应提示选择“采用大纲文案”或“保留画布文案”。这是推荐规则；若你希望大纲始终覆盖画布文字，需要在阶段 D 前明确。其他实现细节可按上面的保守规则推进。
