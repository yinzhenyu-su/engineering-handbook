# 组件规范 — `content/dsa/widgets/`

一个组件一个文件。每个文件都是单个组件运行时规范的**权威来源**：type、level、modes、caps、commands、每个 panel 的标准输入、keyframes 轨迹，以及静态降级图片的路径。

章节 Markdown 的 frontmatter 只按 id 引用组件（例如 `widgets:
[w-09-sliding-window-expansion]`）。其余内容都放在本目录里。

## 为什么采用这种布局

三个理由。

**第一，章节 frontmatter 保持人能读的体量。** 作者只改约 14 行 frontmatter，而不是约 200 行。组件规范太长，没法内联进章节；又太细，没法在两三个共用同一个 panel 的章节里各存一份。

**第二，共享组件只有一个事实来源。** 像 `w-31-dp-table-fill` 这样的 panel 选择器组件会挂到四个章节上；如果规范放在章节 frontmatter 里，四个章节就得一直保持同步。

**第三，构建期就能校验 keyframes 轨迹，不必等到章节上线之后。** CI 的 `make check:widget-spec` 会解析本目录下的每个文件，断言每个 panel 至少有 ≥ 4 个 keyframes，并拒绝构建那些引用了组件 id、本目录里却没有对应文件的章节。

## 文件命名

`<widget-id>.yml` —— 与下面几处使用同一个 id：

- `_widget-registry.yml`（精简索引；“哪些章节用了这个组件”）
- 章节 frontmatter 里的 `widgets: [...]` 列表
- `widget-specs.md`（设计说明；本目录给出的是运行时规范 —— `widget-specs.md` 给出的是设计理由）

## 结构定义

```yaml
id: <widget-id>                             # 必须与文件名主干一致
title: "<Human-readable widget title>"
type: <single-panel | dual-panel-shared | three-way-shared | four-way-shared
       | editorial-single-panel>
level: <0 | 1 | 2>                          # Bostock —— 见 widget-architecture.md §6
modes: [preset, randomize, edit]            # 子集；preset 为必填

# 输入硬边界。表单遇到超过上限的输入会按 §6.3 的提示信息拒绝。
caps:
  array_length: { min: 4, max: 32 }
  values: { min: -99, max: 99 }
  # 各组件自由定义；每个组件的 caps 表见 widget-specs.md。

# AlgoStep 默认词汇表之外的组件专属命令
# （compare / swap / setVar 等，见 widget-architecture.md §2.4）。
commands:
  - name: <commandName>
    payload: { <field>: <type> }
    description: "<what the renderer does>"

# 构建期生成的静态降级图（供 prefers-reduced-motion 与无 JS 场景使用）。
static_fallback:
  generator: jsdom                          # 默认值；个别组件用手写 SVG
  output: site/static/widgets/<widget-id>-fallback.svg
  keyframes_used: [0, n_steps/4, n_steps/2, n_steps-1]   # 抓取哪几步

# 每个 panel 一个块。单 panel 组件只列 `default`。
panels:
  <panel-key>:
    chapter: <chapter-id>                   # 例如 "3.2"
    label: "<selector label shown in panel switcher>"
    canonical_input: |
      <YAML literal block — the exact preset input for this panel>
    presets:
      - id: <preset-id>
        label: "<dropdown label>"
        input: <input value or block>
    n_steps: <integer>                      # 步骤缓冲区的长度
    keyframes:                              # 重头戏 —— 完整步骤轨迹
      - step: 0
        kind: init
        state: { <free-form per widget> }
        narration: "<aria-live string for this step>"
      - step: 1
        kind: <commandName>                 # 上面 commands 里的某一条
        payload: { ... }
        state: { ... }
        narration: "..."
      # ... 渲染器消费的每一步 ...

selector_default: <panel-key>               # 仅多 panel：先渲染哪个 panel

notes: |                                    # 可选，自由格式
  <design notes, citations, links to widget-specs.md §N>
```

## 各部分放在哪里

| 关注点 | 文件 |
|---|---|
| 哪些章节用了某个组件 | `_widget-registry.yml`（精简索引） |
| 组件的设计理由（为什么是 Level 2、为什么只支持 preset 等） | `widget-specs.md` |
| 组件的运行时规范（panels、presets、keyframes、narration） | **本目录** |
| 渲染器 JS 代码 | `site/static/js/widgets/<widget-id>/` |
| 静态降级 SVG | `site/static/widgets/<widget-id>-fallback.svg`（构建时生成） |
| 挂载该组件的章节 | 章节 frontmatter 里的 `widgets: [<widget-id>]` |

## 按需创建

92 个组件里，大部分在本目录还没有文件。它们等到挂载自己的章节开始编写时才补上。`@dsa-write` agent 的约定是：

1. 读 `widget-specs.md §N`，拿到该组件的设计说明。
2. 读该章节的研究文档，拿到实例推演所用的精确 keyframes 轨迹。
3. 按上面的完整规范写出 `content/dsa/widgets/<widget-id>.yml`。
4. 更新 `_widget-registry.yml`，让其条目指向新文件。
5. 章节 frontmatter 永远只列 id。

如果组件是共享的（panel 选择器类型），由*第一个*挂载它的章节建文件，写入第一个 panel 的规范；后续章节通过 PR 追加 panel。

## Phase 1 范例

Phase 1（Part 3 垂直切片）已经搭好了四个文件：

- `w-08-two-pointers-3sum.yml` —— 单 panel 章节组件
- `w-09-sliding-window-expansion.yml` —— 类型为 dual-panel-shared（双 panel 共享）
- `w-10-prefix-sum-cumulative.yml` —— 单 panel 章节组件
- `w-35-prefix-sum-hash-combo.yml` —— 单 panel 章节组件（复用 #10 的数组行）

另外还有一个题解范例：

- `e-LC001-two-sum.yml` —— editorial-single-panel（把 LC 1 的最优解做成动画）

这五个文件就是参考形态。之后每个组件的 yml 都照着其中之一来写。
