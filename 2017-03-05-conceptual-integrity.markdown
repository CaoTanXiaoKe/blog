---
layout:     post
title:      "「译文」概念完整性"
subtitle:   "概念完整性是指一个系统的在概念上能平稳的黏合在一块。"
date:       2017-03-05
author:     "ChenWenKe"
header-img: "img/post-bg-conceptual-integrity.jpg"
tags:
    - 译文
    - 设计
    - 读书
    - 编程思想
---

>Conceptual integrity means that a system's central concepts work together as a smooth,
cohesive whole. The components match and work well together; the architecture achieves
an effective balance between flexibility, maintainability, efficiency, and responsiveness.
The architecture of a software system refers to the way in which the system is structured
to provide the desired features and capabilities. An effective architecture is what gives a
system conceptual integrity.

概念完整性是指一个系统的在概念上能平稳的黏合在一块。 各个组件能很好的契合；构建了一个灵活性，可维护性，高效率，和快响应相互之间高效平衡的架构。一个软件系统的架构是指供后续的细节和潜在功能填充的结构。 一个良好的体系架构一定是概念完整性的。 

> How is conceptual integrity achieved? In designing a complex machine like an
automobile, hundreds of engineers are involved over a period of about three years.
Hundreds of specialized parts are developed by specialized engineering groups, and
thousands upon thousands of detailed decisions and tradeoffs are made. The key to
achieving conceptual product integrity in an automobile is the effectiveness of the
communication mechanisms developed among these groups as all of these decisions are
made. [5]

如何实现概念完整性呢？ 在一个复杂的机器设计中，如汽车， 需要上千的工程师一起工作大约 3 年，需要由许多特定工程部门开发的成千的特定部件，需要做成千上万的微决策和权衡。 在设计汽车的过程中，要实现概念完整性的关键在于 各个部门以及各个工程师之间在所有的决策上的良好交流。 

> An automobile's architecture is not something that is decided at the beginning of a
development effort. True, a car has an engine, a body, a drive train, and so forth. But
layout and styling engineers have very different ideas about how a car should look. And
manufacturing engineers have quite a different view how new parts should fit together
than do the engineers who design the parts. In a very real sense, the architecture of the
automobile emerges as these groups work together. If they work together effectively, the
product will have conceptual integrity.

一款汽车的构架不是一件在开发开始前就完成的一件事。 的确，一辆车有引擎，车身，传动轴以及其他很多组件。 而且布局和外观设计师之间在车的外观上有很多不同的观点，制造工程师和组件工程师之间在各个组件之间如何契合上有很多不同的观点。只有他们之间能实现微妙的平衡，这款车的设计才能实现概念完整性。  

> There are two key practices used by automotive companies to achieve conceptual
integrity. First, the use of existing parts immediately removes many degrees of freedom
and thus reduces the complexity and need for communication. When a new car has novel
body styling and a new engine, it helps to use a proven suspension system.
The second practice automotive companies use to achieve conceptual integrity is to use
integrated problem solving to assure excellent technical information flow. As we noted
earlier, Clark and Fujimoto's research showed that conceptual integrity is a reflection of
the integrity of the upstream and downstream technical information flow in the product
development process. [6] Product development is a system of interconnected problem-
solving cycles, and frequent problem-solving cycles that effectively span upstream and
downstream engineers are common practice in automotive companies with high product
integrity. [7]

汽车公司在实现汽车设计的概念完整性方面有两条实践经验。 

1. 及时利用已设计好的组件，从而减少大量的设计自由度，从而减少设计的复杂性，进而减少了工程师之间的交流工作量。 例如，当一款新车已经确定了车身样式和新引擎后，就更容易确定设计怎样的悬挂系统。 
2. 为了确保关键信息的流通，他们采用**“问题集中解决”**的方式。正如 Clark 和 Fujimoto 的研究说显示的那样：概念完整性是上层技术信息和下层技术信息在产品开发工程中相互交融的表现。 产品开发在高度集成的产品上（如汽车），是在上层工程师和下层工程师经过不断的有效交流，不断的解决互联性的问题的一个系统。 

<br/>

---

> 原文摘自：Addison Wesley《Lean Software Development - An Agil Tookit》Tool 18: Conceptual Integrity
- [5] Clark and Fujimoto, Product Development Performance, 30–31.
- [6] Ibid., 30.
- [7] Ibid., 206.

<br/>
