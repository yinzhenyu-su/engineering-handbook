# 为 Engineering Handbook 做贡献

谢谢你有兴趣参与。这个项目能存在，是因为像你这样的人愿意公开地写、审、翻译和改进它。

这份指南讲三件事：你能贡献**什么**、**怎么**提交改动，以及每份贡献都要达到的**质量标准**。

## 目录

- [三本书共用一套流程](#三本书共用一套流程)
- [参与方式](#参与方式)
- [动手之前](#动手之前)
- [贡献流程](#贡献流程)
- [质量标准](#质量标准)
- [图表](#图表)
- [引用与来源](#引用与来源)
- [评审流程](#评审流程)
- [认可与署名](#认可与署名)
- [问题与联系](#问题与联系)

## 三本书共用一套流程

本仓库托管三门并列的开源课程：HLD Handbook 在 [`content/hld/`](content/hld/)，DSA Handbook 在 [`content/dsa/`](content/dsa/)，AI Engineering Handbook 在 [`content/ai-eng/`](content/ai-eng/)。三门课共用这套贡献流程、同一个 CC BY-SA 4.0 许可、同一套 CI 质量标准，但**章节骨架各不相同**，写的时候要对着你改的那本书来。

- **HLD 章节**按下文[质量标准](#质量标准)里的章节模板来写：引言 → 第一性原理 → 图 → 完整案例 → 权衡 → 生产环境坑点 → 参考文献。文风与结构见 [STYLE_GUIDE.md](STYLE_GUIDE.md)。
- **DSA 章节**以练为主：一张紧凑的速查表、对数据结构或模式的深入讲解、代表性 LeetCode 题的提示卡，以及 `<details>` 折叠的题解和常见错误块。代码样例作为同级文件放在每章目录下（`sol.py`、`sol.java`、`sol.cpp`、`sol.go`）；Python 题解内联在章节里，其余语言给链接。标准形态见 [`content/dsa/part-1-linear-data-structures/00-arrays.md`](content/dsa/part-1-linear-data-structures/00-arrays.md)，字段规则见 [STYLE_GUIDE.md § DSA 专有差异](STYLE_GUIDE.md#dsa-specific-deviations)。

- **AI Engineering 章节**没有固定骨架：每篇都从一个真实事故或反例切入，讲清机制，最后落到读者能直接用上的东西。插图是各 part `images/` 目录下的 PNG，带 alt 文本和图注；代码用 Python；易过期的信息（价格、模型名、基准分数）都要标“截至”日期并附脚注来源。见 [`content/ai-eng/README.md`](content/ai-eng/README.md) 和 [STYLE_GUIDE.md § AI Engineering 专有差异](STYLE_GUIDE.md#ai-engineering-specific-deviations)。

除非特别说明，本指南其余部分对三本书都适用。

## 参与方式

按你手头有多少时间来选贡献级别。

| 时间 | 你能做什么 | 需要先开 issue？ |
|------|-----------------|-------------------|
| 5 分钟 | 修错别字、失效链接或格式问题 | 不需要 |
| 30 分钟 | 补一个真实案例、更新过时的数字、把一段话说清楚 | 不需要 |
| 1 小时 | 给现有章节画一张 Mermaid 图 | 可选 |
| 2 到 4 小时 | 校对现有章节的准确性（留评审意见） | 需要 |
| 4 到 8 小时 | 按提纲写一整章 | **需要，必须** |
| 持续参与 | 把一章或一节翻译成其他语言 | 需要 |

> [!TIP]
> 如果你亲手做过我们某个案例里的系统（聊天系统、支付系统、信息流等），哪怕没时间写，也请帮忙评审那篇案例。一个有生产经验的人花 30 分钟评审，胜过一个人闷头推演一个月。

## 动手之前

### 至少读一章现有章节

完整读一章，感受一下预期的深度、结构和语气。推荐的几篇：

- [0.0 系统设计中的网络基础](content/hld/part-0-prerequisites/00-networking-fundamentals.md)
- [1.4 量级估算](content/hld/part-1-core-fundamentals/04-back-of-envelope-estimation.md)
- [8.0 设计一个 URL 短链接服务（TinyURL / bit.ly）](content/hld/part-8-case-studies/00-url-shortener.md)

### 读一遍风格指南

[STYLE_GUIDE.md](STYLE_GUIDE.md) 写明了文风、结构和排版规则，每个 PR 都会对照它检查。

### 查看未关闭的 issue

动手做较大的贡献之前，先搜一遍 [issue 列表](https://github.com/handbook-academy/engineering-handbook/issues) 和 [PR 列表](https://github.com/handbook-academy/engineering-handbook/pulls)，避免重复劳动。

## 贡献流程

### 小修小补（错别字、链接、小改动）

1. Fork 仓库。
2. 建分支：`fix/typo-in-cap-chapter` 之类。
3. 改。
4. （可选）本地跑 `npm install && npm run check:all`，提前发现问题。
5. 提 PR。在描述里关联相关 issue。
6. 等评审。维护者会合并，或者提改动要求。

### 新写一章（基于提纲）

1. **先开 issue**，标题写 `Writing: [Chapter Name]`。这样能占住这一章，避免两个人撞车。写清：
   - 你打算覆盖的范围（要讲哪些主题）
   - 预计字数
   - 自己定的截止时间

2. 等维护者确认。

3. Fork 并建分支：`chapter/part-2-caching` 之类。

4. **用 writing-guides 里的模板。** Part 8 的案例必须按 [writing-guides/case-study-template.md](writing-guides/case-study-template.md) 写，权衡页必须按 [writing-guides/trade-off-template.md](writing-guides/trade-off-template.md) 写。

5. 按[章节模板](#章节模板)写这一章。

6. （可选）本地跑检查，推送前先拿到反馈：

   ```bash
   npm install                # 只需装一次
   npm run lint               # markdownlint 检查
   npm run check:frontmatter  # YAML 结构校验
   npm run check:citations    # 脚注完整性
   npm run check:mermaid      # 图表语法
   npm run check:typos        # 拼写检查
   npm run check:all          # 全部检查
   npm run stats              # 查看内容统计
   ```

   这些检查不跑也行——CI 会在你的 PR 上自动全跑一遍，并告诉你哪里要改。

7. 提 PR。描述里链上跟踪 issue，并概述你的关键设计选择（比如“我把 cache-aside 当作默认做法，把 write-through 作为备选来讨论，因为……”）。

8. 做好被认真评审的准备。章节会从技术准确性、表达清晰度、与全书一致性三个方面被审。

### 翻译

1. 开一个 issue，标题写 `Translation: [Language] [Chapter Name]`。
2. 翻译目录结构如果还不存在，由我们来加。
3. 流程和新写一章一样。
4. 译文合并进 `content/i18n/[lang]/` 目录。

## 质量标准

无论贡献者经验深浅，每份贡献都按同一套标准衡量。

### 硬性规则（没有商量余地）

| 规则 | 原因 |
|------|-----|
| **章节正文里不许用 emoji 当结构图标。** frontmatter 标签和提示框图标可以用。 | emoji 在各平台渲染不一致；正文里请用规范的 SVG 或纯文本。 |
| **只写真实 URL。** 绝不编造链接。拿不准就写“作者, 年份, 标题”而不带 URL。 | 读者信任我们的引用，编造的 URL 会毁掉这份信任。 |
| **原创写作。** 不要从 ByteByteGo、DesignGurus、Educative 或任何付费课程里粘贴内容。可以讨论同样的概念，但要从头自己写。 | 法律（DMCA 风险）与伦理。 |
| **技术准确。** 每个具体数字（QPS、延迟、吞吐量）都必须可核实。给出出处。 | 我们不发布传闻。 |
| **frontmatter 完整。** `title`、`description`、`part`、`module`、`difficulty`、`prerequisites`、`date_created`、`date_updated`。 | CI 依赖这些字段。 |

### 章节模板

每章新内容都必须按这个顺序包含以下小节：

1. **Frontmatter**（顶部的 YAML）
2. **标题**（`# H1`，与 frontmatter 的 title 一致）
3. **TL;DR**（引用块，一到两句话）
4. **学习目标**（清单，3 到 6 条）
5. **直觉理解**（类比或具体场景）
6. **原理**（核心内容，用 `###` 分小节）
7. **真实案例**（引一家具体公司和他们的工程博客）
8. **权衡**（至少 3 行的对比表）
9. **常见陷阱**（配 `[!WARNING]` 提示框）
10. **练习**（带 `<details>` 提示和 `<details>` 参考答案）
11. **要点回顾**（3 到 5 条）
12. **延伸阅读**（真实 URL）
13. **记忆卡片**（2 到 3 组问答，放在引用块里）

示例可以看 `content/hld/part-0-prerequisites/` 里任意一篇完整章节。Part 8 的案例和权衡页请用 [writing-guides/](writing-guides/) 里的专用模板。

### 图表

我们只用 **Mermaid** 和 **Excalidraw**。不允许 ASCII 框图（用 `+`、`-`、`|` 拼的那种）：它和同页的 Mermaid 图容易走形，换比例字体就散架，而 Mermaid 能表达得更好。

- **Mermaid** 是默认选择，用于时序图、流程图、状态机、ER 图和架构总览。在 GitHub 和网站上零插件直接渲染。
- **Excalidraw** 用于 Mermaid 表达不了的图：多地域部署拓扑、威胁模型、自由白板标注。

每篇教学章节至少要有**三张 Mermaid 图**（案例至少四张），除非内容本身无法图示。每张图正下方配一句斜体图注，点明这张图要说明什么。完整规则见 `STYLE_GUIDE.md`。

### 引用与来源

| 来源类型 | 怎么引 |
|-------------|-------------|
| 工程博客 | `[文章标题](URL)，公司名，年份` |
| 学术论文 | `作者 et al., "标题," 会议/期刊 年份. [DOI 或 arxiv URL]` |
| 图书 | `作者, _书名_, 出版社 年份, 章节/页码` |
| RFC 或规范 | `[RFC XXXX: 标题](https://datatracker.ietf.org/doc/html/rfcXXXX)` |
| 官方文档 | `[主题](官方文档 URL)` |

优先用一手来源（工程博客、论文、RFC），而不是二手转述（比如总结工程博客的 Medium 文章）。

## 评审流程

维护者批量审 PR，不是实时响应。错别字和链接修复审得最快，整章新增通常要过不止一轮。

### 评审看什么

1. **技术准确性。** 论断站得住吗？数字能核实吗？
2. **清晰度。** 一个从没接触过这个主题的读者能看懂吗？
3. **一致性。** 和全书其余部分的语气一致吗？
4. **完整性。** 该有的小节都在吗？
5. **合规性。** 没有付费内容的拷贝、frontmatter 合法、图能正常渲染。

### 反馈文化

我们的评审意见要做到**具体、善意、可执行**。

- 具体：“第 42 行的 QPS 估算差了 10 倍。DAU 是 100M，100M / 86,400 秒 ≈ 1,200 QPS 平均，不是 12,000。”
- 善意：不评判人，只针对论断。
- 可执行：永远给出一个具体改法。

收到评审意见后，把改动推到同一个分支，不要另开 PR。

## 认可与署名

所有贡献者都列在 [CONTRIBUTORS.md](CONTRIBUTORS.md) 里，按首次贡献时间排序。

章节作者署名在自己写的那章开头，评审者署名在该章的致谢小节。

我们明确**不**用积分或排名制度。任何规模的贡献都同等重要。

## 问题与联系

- **写作上的技术问题：** 开一个带 `question` 标签的 issue，或到 [GitHub Discussions](https://github.com/handbook-academy/engineering-handbook/discussions) 里问。
- **内容有错（错别字、事实错误、失效链接）：** 开一个带 `content: correction` 标签的 issue。
- **hld.handbook.academy 站点的 bug：** 在这里用[站点 bug 模板](.github/ISSUE_TEMPLATE)开 issue，附截图和 URL，或发邮件到 [hello@handbook.academy](mailto:hello@handbook.academy)。
- **私下反映（行为准则、法律、安全）：** 发邮件到 [hello@handbook.academy](mailto:hello@handbook.academy)，或在 GitHub 上开[私密安全公告](https://github.com/handbook-academy/engineering-handbook/security/advisories/new)，也可以通过 [@invincible04](https://github.com/invincible04) 的 GitHub 主页联系维护者。

提交贡献即表示你同意自己的贡献按项目其余部分相同的条款授权（CC BY-SA 4.0）。

欢迎上船。
