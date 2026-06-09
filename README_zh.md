# 我正在推进的探索

我正在构建一系列工具，旨在将人工撰写的书籍书稿转化为可供查阅的知识图谱。

只需单一的源文件。人类负责书写与阅读，而机器则能对其中的每一个标题、每一次交叉引用以及每一种关系进行查询——并且支持完全追溯到源文件的具体行与列。

---

## “页面”模型的困境

现今大多数的文档工具都将知识视为独立页面的集合。每个页面拥有专属的 URL、侧边栏条目以及自洽的作用域。这种设计背后潜藏着一个假设：知识是由离散且自给自足的单元构成的。

然而，倘若知识真的是彼此独立的，为何每一个文档网站都会充斥着错综复杂的交叉链接？为什么“状态（State）”页面会链接到“组件（Components）”，继而指向“钩子（Hooks）”，最后又关联到“生命周期（Lifecycle）”？这些概念彼此依存。它们共同编织成了一张图谱（Graph），而非一片零星的孤岛。

将它们割裂成一个个独立的页面并不能消除这些依赖关系，仅仅是将它们隐藏了起来。内在结构依然存在——只是被强行塞进了读者的脑海中。在那里，它是不可见的、难以维护的，更是无法查询的。

数千年来，书籍早就解决并沉淀了这个问题。一本书有部、章、节。它包含目录、索引、词汇表和参考文献。它有着一套精心构思的阅读顺序，让概念如地基般层层叠压、稳固构建。它通过交叉引用，将各种思想明确地串联起来。这些绝非过时的出版传统。它们是趋同演化出的结构，完美契合了人类处理复杂信息的真实方式：线性推进、层级分明，并伴随着稳定的术语与清晰的依赖关系。

真正的问题在于：传统书籍无法被机器查询。其结构仅仅寄居于作者的排版选择与读者的主观理解之中。没有任何程序能仅凭读取一份 PDF，就回答出“哪些章节引用了身份验证模块”或者“哪些规则制约了这一设计决策”。

试想一下，如果一份书稿能兼具两者呢？在无需产生任何重复信息的情况下，同一份源文本既能供人类流畅阅读，又能供机器精准查询。

## 运作机制

我使用 [AsciiDoc](https://docs.asciidoctor.org/asciidoc/latest/) 作为创作载体。AsciiDoc 是一种专为技术书籍设计的轻量级标记语言。它原生支持文档层级（部、章、节）、交叉引用、属性列表、索引项以及包含指令（include directives）——囊括了一本书所需的全部核心要素。

核心的洞察在于：AsciiDoc 的属性系统能够承载结构化的元数据，同时丝毫不破坏阅读体验。例如下面这样一个标题：

```adoc
[#delivery-policy.policy, status=active, owner=ops]
== Delivery Policy

This policy depends on xref:capacity-rule[Capacity Rule, rel=depends-on].

```

作为自然语言阅读时，它流畅如常。但同时，它也在严谨地声明：

* 一个稳定的锚点：`#delivery-policy`
* 一个语义标识：`.policy`（表明该标题是一项策略，而非概念或规则）
* 表层字段：`status=active`，`owner=ops`
* 一种具态关系（typed relation）：该标题**依赖于（depends on）**“容量规则（Capacity Rule）”标题。

**[asciidoc-abundant-tree](https://github.com/MichengLiang/asciidoc-abundant-tree)** 是这套系统的映射引擎（projection engine）。它负责解析 AsciiDoc 源文件（或是包含多源文件的完整书籍图谱），并生成符合 RDF 1.2 标准的知识图谱——格式为 Turtle 或 JSON-LD。每一个标题都会转化为一个具态节点。每一次交叉引用都会变成一条带有命名谓词的有向边。每一个属性都会成为一个可查阅的字段。而每一个节点，都完整保留了其原始文件的路径、行号、列跨度以及原始创作文本。

**[inkrail](https://github.com/MichengLiang/inkrail)** 则是查询工作台。它负责管理项目记忆，追踪映射的新鲜度，将 Turtle 数据加载到内嵌的 SPARQL 引擎（[Oxigraph](https://github.com/oxigraph/oxigraph)）中，并执行查询操作——既支持原生 SPARQL 语句，也提供内置快捷动作，如 `outline`（大纲）、`xrefs unbound`（断链引用）、`degree out`（出度）、`neighborhood`（邻接节点）以及 `audit duplicate-labels`（重复标签审计）。

完整的工作流如下：

1. 你遵循一小套简单的约定（稳定的 ID、角色设定、关系谓词），用 AsciiDoc 撰写一本书。
2. `asciidoc-abundant-tree` 将其映射为 RDF 图谱。
3. `inkrail` 让你能在图谱中进行查阅、审计和导航——随后将每一个结果精准追溯到源文件的具体行。

无需维护独立的数据库。无需为知识保留第二份副本。书稿**即是**数据库。

## 一个微型示例

假设你正在编写一本包含两章的技术书籍。一章定义了一项规则（rule），另一章描述了一项依赖于该规则的策略（policy）。

**源 AsciiDoc 文件：**

```adoc
[#capacity-rule.rule, status=active]
== Capacity Rule

Capacity Rule describes when delivery must degrade.

[#delivery-policy.policy, status=active, owner=ops]
== Delivery Policy

Delivery Policy depends on xref:capacity-rule[Capacity Rule, rel=depends-on].

```

**映射完成后，你可以进行如下查询：**

```bash
# 哪些标题被赋予了 "policy" 角色？
inkrail query --json 'PREFIX aat: <https://micheng.dev/ns/asciidoc-abundant-tree#>
SELECT ?headline ?path ?line WHERE {
  ?h aat:role "policy" ; aat:headline ?headline ;
     aat:relativePath ?path ; aat:headingLine ?line .
}'

# "Delivery Policy" 依赖于什么？
inkrail neighborhood out --label delivery-policy --depth 1

# 是否存在失效的交叉引用？
inkrail xrefs unbound

# 哪些标题没有被任何其他内容引用？
inkrail degree in --json

```

每一条查询结果都会附带源文件的路径与行号。你可以阅读查询输出，直接打开对应文件，一字不差地看到作者最初写下的内容。

## 这将开启的潜能

这绝不仅是一种理论上的可能。这些工具已经切实存在，发布在了 npm 上，并已投入使用。当你的书稿化作可查询的图谱时，以下愿景便触手可及：

**溯源能力 (Source traceability)。** 在一个改编项目中——例如将一部小说改编为动画剧本——每一个场景都可以声明 `rel=adapted-from`（改编自），指向原作的一段内容；每一个角色都可以声明 `rel=derived-from`（衍生自），指向确立其人设的文本；每一个视觉表达的选择都能声明 `rel=externalizes`（外化），精准指向它所替代的那段内心独白。只需一条 SPARQL 查询，就能回答：“哪些场景缺乏源头追溯？”这不再是轻飘飘的文体修改建议，而是一场确凿可证的严格审计。

**一致性校验 (Consistency checking)。** 当全书始终贯彻 `.rule` 角色代表“此标题为一条规则”，且 `rel=constrains` 关系代表“此规则制约该设计”时，系统便能自动校验：“是否存在没有任何制约对象的规则？是否存在不受任何规则制约的设计？”在非结构化文档中，此类更新异常与参照完整性缺失往往会暗中滋生，直至有人依据过时的定义做出了错误的决策才会暴露；而现在，一切无可遁形。

**可导航的结构 (Navigable structure)。** 执行 `inkrail neighborhood out --label authentication --depth 2`，即可向你展示与“身份验证（authentication）”标题相距两跳范围内的所有节点信息——它依赖什么、谁依赖它、什么制约了它、它又引用了什么。这是为你呈现的一张局部子图，既为你提供了“恰到好处的上下文”，又免于让你淹没在茫茫的整篇文档中。

**领域可移植性 (Domain-portable)。** 底层机制——稳定 ID、角色、关系谓词、表层字段——是与具体领域无关的。但其词汇表（vocabulary）却可以针对特定领域量身定制。一本航空安全手册可以使用诸如 `.regulation`（规章）、`.procedure`（程序）、`.hazard`（危险源）等角色，以及 `rel=mitigates`（缓解）、`rel=supersedes`（取代）等关系。一个医学知识库则可以使用 `.diagnosis`（诊断）、`.symptom`（症状）、`.treatment`（治疗），配合 `rel=indicates`（适应症）、`rel=contraindicates`（禁忌症）。工具链无需“懂”你的专业领域，它只需你的词汇表在书稿范围内保持稳定。

## 为什么这对语言模型至关重要

语言模型——包括协助我编写代码与润色文章的模型——本质上都是自回归系统（autoregressive systems）。它们生成的每一个 Token 都深刻受制于之前出现的所有内容。它们并没有“退后一步，进行反思”的机制。它们的输出，极其深刻且不可避免地，被赋予它们的上下文结构所塑造。

抛给语言模型一个 README 模板，它产出的只会是零星的片段。给它一个 Wiki 页面模板，它会生成一份默认读者毫无背景知识的封闭式总结。但倘若你提供一份包含了部、章、稳定术语、交叉引用以及关系谓词的书稿结构，它就能生成出结构严密、互相关联且可追溯的内容——因为**媒介本身赋予了这种引导性**。

结构化的写作约定不仅仅是一份风格指南。它更是一套高阶的提示词（Prompt）。它在告诉模型：这些是你可以分配给标题的角色；这些是你可以声明的内在关系；而这，是你的输出在宏大结构中所处的位置。模型不需要你来教 AsciiDoc 语法——它早就烂熟于心。它需要获取的，仅仅是你的专属词汇表。

这意味着你可以将语言模型视为一位**结构级联合创作者 (structural co-author)**：它在书籍的骨架内起草内容，遵循已声明的角色与关系，且它所提出的每一个论断都能无缝追溯到特定的源标题或明确的改编抉择上。你审查的不再只是散文的辞藻，更是骨架的结构。你查询的不仅是文字，更是逻辑的图谱。

## 工具生态

| 项目 | 核心职能 |
| --- | --- |
| [asciidoc-abundant-tree](https://github.com/MichengLiang/asciidoc-abundant-tree) | 将 AsciiDoc 解析为保留来源感知的文档树。并映射至 RDF 1.2 标准的 Turtle 与 JSON-LD 格式。 |
| [inkrail](https://github.com/MichengLiang/inkrail) | 为 AsciiDoc 书籍工作区提供项目记忆、映射新鲜度管理及 SPARQL 查询工作流。 |
| [asciidoc-multi-book-workspace](https://www.google.com/search?q=PLACEHOLDER_LINK) | 为结构化 AsciiDoc 创作提供的一套模板、规范与示例文献。 |

该 RDF 映射全面采用 W3C 标准词汇——涵盖 RDF 1.2、用于溯源的 PROV，以及一个精简的项目自有命名空间（`aat:`），后者专门用于处理标题结构、交叉引用证据与表层字段。生成的图谱完全兼容任何 SPARQL 引擎、RDF 验证器，以及任何支持读取 Turtle 或 JSON-LD 的外部工具。

## 延伸阅读

* [结构化写作规范示例书](https://www.google.com/search?q=PLACEHOLDER_LINK) — 一本具有自证性质的书籍，它通过践行规范来传授规范本身。
* [环境、媒介与行为：跨学科综述](https://www.google.com/search?q=PLACEHOLDER_LINK) — 一份关于媒介格式如何塑造认知与行为的文献调查，涵盖了 Innis、McLuhan、Postman、Gibson、Barker 以及 Bourdieu 的理论思想。
* [RDF 1.2 标题映射规范](https://michengliang.github.io/asciidoc-abundant-tree/books/06-rdf12-line-projection/book.html) — 详细规定源文档如何映射为图谱资源的正式规范。

## 驱动这一切的初衷

高德纳（Donald Knuth）撰写《计算机程序设计艺术》（*The Art of Computer Programming*）已有六十余载。其中每一个算法皆附有证明，每一处引用皆精确到页，每一个术语在各卷之间始终如一。他为此创造了 TeX 与 WEB 系统以维系这种极致的一致性——并非出于制造工具的爱好，而是因为这项宏伟的工作本身渴求着它们。

在 Git 诞生之前，林纳斯·托瓦兹（Linus Torvalds）纯粹依靠电子邮件来管理 Linux 内核。成千上万的补丁，皆在大脑中评估，凭直觉与判断力合并。后来他花了短短两周时间写出了 Git——它不是作为一款商业产品出现的，而是他脑海中早已成熟的版本控制模型的一次具象“外化”。

这两位巨匠所共通的，绝非通俗意义上的天才禀赋。那是一种对信息组织极其严苛的自律——索引、交叉引用、稳定术语、可溯源依赖、版本感知——他们在几十年的岁月里反复践行，其展现出的水平看似超乎常人，但本质上是**结构化**的力量。

我的愿景，便是通过工具将这种强大的结构能力普惠于众。并非妄想去复制他们的神迹——那无人能及——而是为广大的普通作者、研究员、工程师和团队搭建底层支撑体系，让结构化的知识能够从“书写”这一自然行为中自发涌现。你写下一个标题，赋予它一个 ID 与角色，声明它的关系——无需任何额外的建模步骤，无需配置数据库，也不必拥有知识工程学位，你就这样自然而然地获得了一座可查询、可审计、可追溯的知识图谱。

自泥板时代起，书籍便已成为人类梳理复杂知识的首要媒介。那些古老的约定之所以历久弥新，正是因为它们极其有效。真正前所未有的是，如今我们已经能够将这些古老的法则，直接映射为机器可读的知识图谱——并且，语言模型终于可以深度参与到这场漫长的创作纪元之中。

这，便是我正在全心构建的未来。
