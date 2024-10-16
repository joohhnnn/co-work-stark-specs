# co-work-stark-specs

先中文版，后续统一通过AI翻译成英文版

临时仓库 积累一定量的内容后进行迁移，并公开继续构建

任务先都写在已经创建的文件里，后续进行补充。

从sequencer入手（到底有几个版本？） 粗略浏览： Blockifier 和 Starknet_in_Rust

和几个官方的人建立了联系，后续进行几次会议确定流程

资源： https://book.starknet.io/ch03-02-sequencers.html https://www.spaceshard.io/blog/blog-sequencers-nodes-starknet

https://github.com/starkware-libs/stone-prover（初步认定官方版，来自discord titan 不可靠） https://github.com/starkware-libs/sequencer（初步认定官方版，来自discord titan 极其不可靠，甚至没有readme） https://github.com/keep-starknet-strange/madara（官方推的sequncer 不确定） https://github.com/lambdaclass/starknet_stack/tree/main/sequencer（官方推的sequncer 不确定）

目前研究感觉内容有限复杂，经过讨论，我们可能会舍弃部分内容，比如具体的ZK验证（难度太大），Cairo语言（太偏应用层）等内容，然后内容横跨太多仓库了，比如sharp都没有开源。有点难度，后续再思索一下怎么把这个东西穿起来吧。