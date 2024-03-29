+++
title = "Snowflake 的增量处理方案及其引发的思考"
date = "2023-12-11T00:26:19+08:00"
draft = false
+++
## 引言
一直以来，增量处理算法都是流处理的核心与灵魂所在。通过使用增量处理算法，可以在数据流到达时进行实时处理，而无需对整个数据集进行重新计算，从而搭建出低延迟、低成本的数据处理工作流。
[What’s the Difference? Incremental Processing with Change Queries in Snowflake](https://dl.acm.org/doi/10.1145/3589776)
这篇论文阐述了 Snowflake 增量数据处理方法的设计决策与实现细节，并对 Snowflake 提供的不同类型的增量处理方法在实际应用场景中的使用情况进行了分析。

这是一篇比较偏应用场景的论文，所以我这种没有研究背景的人读起来也不会吃力。文中没有讨论什么关于增量处理的高深技巧，更多的是从应用场景、产品设计角度出发，讨论 Snowflake 增量处理方法的具体技术实现与决策细节。论文里所实现的增量处理能力也并不新鲜，在现在的开源社区里其实都可以找到对应的项目实现。但是值得关注的是 Snowflake 其增量处理方案的产品能力，通过极致的易用性让用户可以无负担的享用增量处理带来的便利。

## 问题与挑战
增量处理是流处理的基础，即在可分割的时间片上执行计算的能力。从增量处理的角度出发，我们可以将数据集分为 Table 和 Stream 两种形式，Table 表示数据集在每个时间点的完整状态，而 Stream 捕捉数据集在某一个时间段内的变化（Change log）。Table 和 Stream 是在以不同方式表示着随着时间而改变的数据集。

为了保持 Table 中数据的正确性、实效性，我们通常需要基于一些新的数据对已有的数据表进行变更，即实现 Stream 到 Table 的转换。现代 SQL 为各种更新逻辑都提供了完善的语法支持，INSERT，UPDATE，DELETE 还有万能的 MERGE 都可以将变更数据注入到表中，实现 Stream 到 Table 的转换。

但是现代 SQL 却一直缺乏从 Table 中提取出变更数据的能力（Table 到 Stream 的转换），这给以增量计算为基础的流式任务造成了掣肘。用户为了在 SQL 中实现增量处理逻辑，只能选择额外维护一副 Change log 数据，或者自己计算出变更的数据，但这通常需要付出较高的计算代价，且实现难度也不低。

## Snowflake 的解决方案：CHANGES AND STREAMS

面对当前 SQL 对 Table 变更查询支持的不足，Snowflake 的解决方案是引入 CHANGES 查询以及 STREAM 对象。

CHANGES 查询就是查询 Table 在某一段时间内的数据变更：新插入了哪些记录，删除了哪些记录，更新了哪些记录，在一些系统里这也被称为 CDC（Change Data Capture）查询。其实现在几个比较火的开源湖仓项目也都支持 Table 上的 CDC 查询，但 Snowflake 对 CHANGES 查询的支持更加完善，它还实现了在 View 上的 CHANGES 查询，这是目前的几个开源湖仓项目都还无法做到的。

相较于 CHANGES 查询，我更感兴趣的是 Snowflake 的 STREAM 对象。Snowflake 将 STREAM 对象定义为一个 schema-level 的 catalog 对象，可以在 Table 或 View 上创建，STREAM 的状态，被称为其 *frontier*，表示在该 *frontier* 之前对其源表的所有 CHANGES 已被消费。可以说说 STREAM 的状态就是一个指针，用来记录在 Table / View 上的 CHANGES 查询的消费进度。「指针的移动」与「数据的消费」是原子性的，所以消费者可以以事务的方式来消费 STREAM。

值得注意的是 STREAM 对象是完全托管的，其存在于 Snowflake 的云服务中，与具体的数据任务解耦。托管式的 STREAM 对象来来的另一个好处是，平台可以直接通过追踪 STREAM 的状态来感知消费者的消费进度。Snowflake 甚至会通过感知 STREAM 的消费进度来**自动**延迟 STREAM 的 source table 的数据过期时间，避免因为消费不及时导致的数据过期问题。

回想一下我们常见的基于开源项目搭建的平台的增量数据处理方式，对数据源的消费状态要么存储在作业的状态里，要么需要只能自己实现一套存储更新方案，将其存储到外部存储。这两种处理方式，前者状态与作业绑定，使用不便，而后者很难实现「状态更新」和「消费操作」的原子性更新。

如果说 CHANGES 查询提供了构建增量计算的基础能力，那么 STREAM 对象则是**让数据流处理变得更加丝滑**。它以一种既灵活又可靠的方式来传递数据流，让数据的消费状态与数据处理任务解耦，使得 CHANGES 查询更加可靠和易用。CHANGES 查询和 STREAM 对象一起组成了 Snowflake 的增量处理解决方案。

## 从 Snowflake 的增量数据处理方案说开来...

作为一个非数据企业的大数据基础平台工程师，我偶尔会关注这个行业的领先企业的产品、技术，并且思考差距在哪里，思考这个行业的发展方向是什么。除了长久以来大家都反复提及的「云」和「弹性」两个关键词外，我感觉到大数据行业还有一个非常重要的发展趋势是**数据的处理方式越来越向「声明式」靠拢**。

在这方面最明显的转变就是 SQL 在数据处理中变得越来越重要。不过虽然 SQL 足够声明式，但它毕竟也只是一个数据处理的工具，而真实的数据处理场景往往复杂而多变，为了进一步降低数据处理中开发和维护的复杂性，数据厂商们都在发力让数据处理流程中的各个环节都变得更加的「声明式」。在这方面本文提到的 Snowflake 的 STREAM 对象就是一个非常棒的例子：试想一下，假如只有 CHANGES 查询的话，数据工程师除了编写增量处理的工作外，还需要编写额外的代码处理消费进度延迟上游数据可能过期的情况，需要处理增量消费进度更新的事务性。虽然数据工程师真正关心的只有新增的数据，但为了保证数据处理的正常进行，数据工程师将不得不分出精力做一些数据处理之外的工作，这就非常的不「声明式」了。

而 STREAM 对象就可以让数据工程师只需关心变更的数据，而不必耗费精力去关注上游数据是否过期，关注消费状态的更新。可以说基于 SQL 的 CHANGES 查询和托管式的 STREAM 对象一起组成了「声明式」的增量处理解决方案。

再仔细思考下「云」、「弹性」和「声明式」几个发展方向，可以发现数据处理之所以朝着这几个方向演进的背后有着相同的底层逻辑，那就是「成本」。引用我们团队 leader 在一次技术探讨会上的一句话：“人类历史是一场性价比之史/成本之史，先进的生产力，只有门槛/成本足够低，才能普及，才能规模化，才有足够的价值呈现”，个人非常认同这个观点，不过我可能会换一个角度来讲述这句话，我认为追求性价比/成本其实只是一个结果，而这个结果背后的原因是人类永远在追求利润的最大化。所以我会说“人类发展历史是一场追求利润最大化的过程”。

从追去利润的角度出发，我会把创新技术的落地过程分为两个阶段：「圈地期」和「经营期」（胡乱起的名字），当一项新技术刚出现时，往往只有少数的头部企业能够掌握它，这个阶段企业通常更倾向于先将技术应用上抢占市场，所以我把这个阶段起名为「圈地期」，这个阶段的成本是不那么敏感的，因为创新技术带来的收益太高了；但是当这项技术成熟后，大部分的企业都可以轻易的使用上这门技术了，当技术领先的优势不再，控制成本就变得愈发重要了，所以我把这个阶段起名为「经营期」。

大数据这项技术的发展目前就在逐渐进入一个成熟阶段，所以其应用逐渐进入了「经营期」，使用者更加注重使用的性价比。「云」和「弹性」是在追求基础设施的性价比，而「声明式的数据处理方式」则是在追求使用成本的性价比（人力资源）。借鉴下后端开发现状，「声明式」的数据处理方式一定会变得越来越重要，大家都变成 SQL boy。至于再往后怎么发展，现在生成式 AI 这么强，也许这个行业从业者会逐渐成为 Chat GPT prompt boy，也许不会再有 boy ...