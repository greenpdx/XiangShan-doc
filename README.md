# XiangShan-doc
Documentation for XiangShan

This is the official documentation repository for XiangShan.

* Xiangshan microarchitecture documents have been hosted on the ReadTheDocs platform. Welcome to https://xiangshan-doc.readthedocs.io

* The slides directory contains our technical reports, including the report of the 2021 RISC-V China Summit, etc.

## Contact us

Mailing list: xiangshan-all AT ict DOT ac DOT cn

## Follow us

- **WeChat public account: Xiangshan Open Source Processor**

![wechat](articles/wechat.png)

- **Zhihu: [Xiangshan Open Source Processor](https://www.zhihu.com/people/openxiangshan)**

- **Weibo: [Xiangshan Open Source Processor](https://weibo.com/u/7706264932)**

## Translations needed

We need to translate most of the documentations to English and other languages. Contributions welcomed!

## Interns

Welcome to join the Open Source Talent Internship Program (valid for a long time). For details, please click this [link](articles/开源英人才面试生计划.pdf).

## What is Xiangshan?

In 2019, with the support of the Chinese Academy of Sciences, the Institute of Computing Technology of the Chinese Academy of Sciences (http://www.ict.ac.cn/) initiated the "Xiangshan" high-performance open source RISC-V processor project, and developed the world's highest performance open source high-performance RISC-V processor core "Xiangshan". It has received more than 2,500 stars on GitHub, the world's largest open source project hosting platform, and has formed more than 277 branches (Fork), becoming one of the most watched open source hardware projects in the world. It has received active support from domestic and foreign companies. 16 companies jointly initiated an open source chip innovation consortium to further jointly develop around "Xiangshan", form demonstration applications, and accelerate the construction of the RISC-V ecosystem.

Our goal is to become an open source platform for architecture innovation facing the world, serving the architecture research needs of industry, academia, and individual enthusiasts. In addition, we hope to explore the agile development process of high-performance processors during the development of Xiangshan, establish a set of high-performance processor design, implementation, and verification processes based on open source tools, and greatly improve the efficiency of processor development and lower the threshold for processor development.

**Xiangshan will maintain a micro-structure iteration cycle and tape-out cycle of about half a year, and continue to innovate micro-structures and practice agile development methods. ** The official development of Xiangshan processors began in June 2020. 1e3fad1 is the hash value of our [first commit](https://github.com/OpenXiangShan/XiangShan/commit/1e3fad102a1e42f73b646332d264923bfbe9c77e). The commits before this in the code repository belong to [the NutShell processor produced by the first phase of the 2019 One Life One Chip plan](https://github.com/OSCPU/NutShell). In July 2021, the first version of the Xiangshan processor (codenamed Yanqi Lake) has been put into production, reaching a frequency of 1.3GHz at the 28nm process node. At present (March 2022), the RTL code of the second version of the Xiangshan processor architecture (codenamed Nanhu) has been frozen, and the back-end design verification process is underway, with the goal of reaching a frequency of 2GHz at the 14nm process node. We hope to gradually improve the PPA level of the Xiangshan processor through continuous optimization and tape-out verification, making Xiangshan an open source industrial-grade processor and an open source platform for architecture innovation facing the world.

**The Xiangshan processor will always adhere to the open source strategy and firmly open source all our design, verification, and basic tool codes. ** We are very grateful for the community's contribution to Xiangshan. In terms of hardware design, some module designs of the Xiangshan processor were inspired by open source processors, public papers, etc., and referenced existing open source rocket-chip, berkeley-hardfloat, SiFive block-inclusivecache and other codes. We have modified and improved the functions of the existing bus tools, floating-point units, and system caches of the Chisel open source community, and optimized performance indicators such as frequency and throughput. At the same time, we welcome the community to develop based on Xiangshan or use the code of the Xiangshan project. Among many open source protocols, we chose [Mulan Permissive License Version 2](http://license.coscl.org.cn/MulanPSL2/), hoping that (1) the openness of the Xiangshan processor will always be maintained. The Mulan Permissive License is not contagious and users can use it with confidence; (2) the Mulan Permissive License is based in China and faces the world. The Chinese and English versions of the Mulan Permissive License have equal legal effect.

**The Xiangshan processor actively embraces the open source community and welcomes contributions from the community. ** We have seen that some open source RISC-V processor projects rarely accept external code submissions. We understand that there are multiple conceptual and technical reasons behind this phenomenon, such as concerns about development plan conflicts, the need to take into account multiple requirements in processor design, and the difficulty in evaluating the quality of external contributions. For the Xiangshan project, in terms of concepts, we welcome external contributions, such as submitting issues, submitting feature requirements, submitting code, etc. We will seriously consider and evaluate every opinion and suggestion. For example, Chisel is still a new thing for the industry. If you are more familiar with Verilog/SystemVerilog but want to submit code contributions to Xiangshan, we welcome you to submit a Pull Request to [this repository](https://github.com/OpenXiangShan/XS-Verilog-Library). On the technical level, we hope to explore a set of processes and tools to evaluate the quality of code changes, and the basic process will determine whether to accept a code submission. For example, we hope to open a faster and more accurate performance sampling framework in the near future to evaluate the performance benefits brought by an architectural change. When the code submission meets the conditions of not destroying the existing modularity and scalability, having a certain performance benefit, and having a good code style, we will accept this code modification. If we use one sentence to summarize our developer strategy, we welcome any discussion, problem submission, code modification, etc. that is beneficial to Xiangshan.

**In addition to micro-architecture exploration, the Xiangshan project also hopes to explore and establish an agile development process for high-performance processors. ** The goal of the Xiangshan processor is to become an open source platform for architecture innovation facing the world. The establishment of basic capabilities, facilities, and processes is the key to the long-term high-quality development of the Xiangshan processor. We will maintain long-term and stable investment and continue to work hard to establish the infrastructure and basic processes for agile development of processors. In the early days of the Xiangshan project, we followed the development and verification framework of the Guoke processor. During the advancement of the Xiangshan project, we made a lot of improvements to it, adding functions including simulation checkpoints, compressed file loading, and multi-core verification support. At present, the verification environment of the Xiangshan processor has been greatly improved compared to the Guoke processor, and the rich basic tools support the agile verification process of Xiangshan, which is of a complex measurement level. In addition, the UCB and Chipyard frameworks are our role models, and we have referred to or used many open source projects initiated by them. We hope that with the advancement and deepening of the Xiangshan project, we can promote the continuous progress of the open source community, and work with the open source community to promote the establishment of agile development processes and infrastructure for high-performance processors.

The official development of the Xiangshan processor started in June 2020. After less than 2 years of hard work, we have achieved the tape-out of the Xiangshan Yanqi Lake version and are about to complete the tape-out preparations for the Nanhu version. From the perspective of the participants in the Xiangshan project, in the past 2 years or so, more than 20 teachers and students from universities and research institutions such as the Institute of Computing Technology of the Chinese Academy of Sciences, Pengcheng Laboratory, Shenzhen University, Huazhong University of Science and Technology, and Peking University have participated in the front-end process of the Yanqi Lake version of the Xiangshan processor. At the beginning of the project, most teachers and students in the Xiangshan team did not have rich experience in high-performance processor design and implementation, but after a year of training in the Xiangshan project, everyone has established a preliminary understanding of high-performance processors. In the Xiangshan processor, key codes including the core front-end, back-end, memory access pipeline, L1 Cache, L2 Cache consistency support, etc. are all implemented from scratch by the development team, and all our modification history of the code is visible in the Git commit record. The physical design process of the Xiangshan processor is mainly completed by our back-end engineering team in Pengcheng Laboratory.

**We clearly realize that Xiangshan processors are still far from the mainstream level of the industry. For example, we are not doing well enough in the selection of solutions for many technical points. The goal of Xiangshan processors is to become an open source platform for architecture innovation facing the world. We welcome suggestions and opinions from industry predecessors, high-performance processor enthusiasts, open source communities, etc. As long as they are beneficial to the Xiangshan processor project, we will accept and improve them. At the same time, we also welcome and encourage more people to join the development of Xiangshan processors to promote the continuous innovation of the Xiangshan project. **

## Recommended reports about Xiangshan

Xiangshan always adheres to the principle of seeking truth from facts, always maintains the original intention of establishing an open source platform and exploring agile development, and does not want any content that may cause misunderstanding in publicity.

Since Xiangshan was officially released at the RISC-V China Summit held in June 2021, we have seen a lot of reports on Xiangshan processors on social media, self-media, and news media, some of which are "self-play" by the author, and there is a possibility of misleading readers. For this purpose, we have specially created a [rumor-busting section](https://github.com/OpenXiangShan/XiangShan-doc/tree/main/clarifications) to clarify some widely circulated misunderstandings.

We recommend the following reports about Xiangshan processors, which come from our official release and popular science channels:

- [Introduction to Xiangshan processors](https://www.zhihu.com/question/466393646/answer/1955410750) written by Bao Yungang

- [Article](https://mp.weixin.qq.com/s/MAkxKZ1eS4UwBkvgD91Xng) from the official WeChat account of the China Open Instruction Ecosystem (RISC-V) Alliance

- [Source code](https://github.com/OpenXiangShan) of Xiangshan processors

One of the core concepts of Xiangshan is open source and openness. Regarding this topic, we recommend reading [On the Open Source Spirit by Academician Sun Ninghui](https://mp.weixin.qq.com/s/1Irs9a0EKoB7P-J_4ju66A).
