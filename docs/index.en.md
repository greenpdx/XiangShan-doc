---
hide:
  - navigation
---

# Welcome to Xiangshan official documents!

![香山 Logo](figs/LOGO.png)

## Recent Updates

**Upcoming: We will continue to host hands-on tutorials for the Xiangshan project during the HPCA'25 conference. Look forward to seeing you at HPCA'25 in Las Vegas, USA. Learn more on [the HPCA'25 Tutorial Page](tutorials/hpca25.md).**

We will continue to host hands-on tutorials for the Xiangshan project during the ASPLOS'25 conference. Look forward to seeing you at ASPLOS'25 in Rotterdam, The Netherlands. Learn more on [the ASPLOS'25 Tutorial Page](tutorials/asplos25.md).

We have hosted tutorials at MICRO'24 in Austin, USA. Learn more on [the MICRO'24 Tutorial Page](tutorials/micro24.md).

The Xiangshan community held a hands-on tutorial for the Xiangshan project during the 4th XiangShan community have held a co-event at RISC-V China Summit 2024 in Hangzhou, China, during which we also hosted XiangShan tutorials. Learn more on [the RVSC'24 Tutorial Page](tutorials/rvsc24.md).

We have hosted tutorials at ASPLOS'24 in San Diego, USA. Learn more on [the ASPLOS'24 Tutorial Page](tutorials/asplos24.md).

We have hosted tutorials at HPCA'24 in Edinburgh, Scotland. Learn more on [the HPCA'24 Tutorial Page](tutorials/hpca24.md).

We have hosted tutorials at MICRO'23 with latest update on the third generation (Kunminghu) architecture. Learn more on [the MICRO'23 Tutorial Page](tutorials/micro23.md).

We have hosted tutorials at ASPLOS'23. Learn more on [the ASPLOS'23 Tutorial Page](tutorials/asplos23.md).

**For publications by the XiangShan team, check out [the Publications Page](tutorials/publications.md).**

## Guide Introduction

This is the official documentation repository for XiangShan. It includes an overview of the XiangShan project, code repository, integration guide, getting started tutorial, processor core introduction, development toolchain introduction, and other related content. Please use the left-side menu to navigate to the corresponding sections. If you are accessing the website on a mobile device, due to the limitations of the documentation framework, use the back button in the upper left corner to return to the parent menu and switch pages.

## What is Xiangshan? About XiangShan

In 2019, with the support of the Chinese Academy of Sciences, the "XiangShan" high-performance open source RISC-V processor project was initiated by the [Institute of Computing Technology of the Chinese Academy of Sciences](http://www.ict.ac.cn/), and the world's highest-performance open source high-performance RISC-V processor core "XiangShan" was developed. It has received more than 4,100 stars on GitHub, the world's largest open source project hosting platform, and has formed more than 550 branches (Fork), becoming one of the most watched open source hardware projects in the world, and has received active support from domestic and foreign companies —— 16 The companies jointly launched the open source chip innovation consortium [Beijing Institute of Open Source Chip (BOSC)] (https://www.bosc.ac.cn/), further jointly developed around "Xiangshan", formed demonstration applications, and accelerated the construction of the RISC-V ecosystem.

Our goal is to become a **world-oriented open source platform for architecture innovation**, serving the architecture research needs of industry, academia, and individual enthusiasts. In addition, we hope to **explore the agile development process of high-performance processors** during the development of Xiangshan, establish a set of high-performance processor design, implementation, and verification processes based on open source tools, and greatly improve the efficiency of processor development and lower the threshold for processor development.

**Xiangshan will maintain a micro-architecture iteration cycle and tape-out cycle of about half a year, and continue to innovate micro-architecture and practice agile development methods. ** The official development of Xiangshan processors began in June 2020. 1e3fad1 is the hash value of our [first commit](https://github.com/OpenXiangShan/XiangShan/commit/1e3fad102a1e42f73b646332d264923bfbe9c77e). All previous commits in the code repository belong to [the NutShell processor produced by the first phase of the 2019 One Life One Chip plan](https://github.com/OSCPU/NutShell). In the Xiangshan processor, key codes including the CPU pipeline front-end, back-end, memory access pipeline, L1 Cache, L2/L3 Cache, etc. are independently implemented by the Xiangshan team. All our code modification history is visible in the Git commit record. The physical design process of the Xiangshan processor is mainly completed by our back-end and SoC engineering team in Pengcheng Laboratory. We hope to gradually improve the PPA level of the Xiangshan processor through continuous optimization and tape-out verification, making Xiangshan an open source industrial-grade processor and an open source platform for architecture innovation facing the world.

**The first version of Xiangshan processor (Yanqi Lake architecture)** is aimed at single-core scenarios and supports the RV64GC instruction set. It was put into production in July 2021 and reached a frequency of 1.3GHz at the 28nm process node. In January 2022, the Yanqi Lake chip was returned and successfully powered on. It can run complex operating systems such as Linux/Debian correctly. The initial SPEC CPU2006 performance measured in the DDR4-1600 environment exceeded 7 points @1GHz. The design code is open source [here](https://github.com/OpenXiangShan/XiangShan/releases/tag/v1.0).

**Xiangshan processor version 2 (Nanhu architecture)** supports the RV64GCBK instruction set. The V1 version is for dual-core scenarios. The RTL code freeze was completed in February 2023, the GDSII freeze was completed in June 2023, and it was put into production in November 2023. The frequency reached 2GHz at the 14nm process node. The design code of the Nanhu architecture is open source at [here](https://github.com/OpenXiangShan/XiangShan/releases/tag/v2.0). The Nanhu V2 version includes improved designs such as MBIST. The RTL code freeze was completed in February 2023, and it was put into production in April 2023. It was returned in October 2023 and successfully lit up and started Linux. Further functional and performance testing of the board is currently underway. The design code is open source at [here](https://github.com/OpenXiangShan/XiangShan/releases/tag/v2.1). The Nanhu V3 version will include more micro-structure and PPA improvements, and the project is currently in progress.

**Xiangshan processor version 3 (Kunming Lake architecture)** is a work in progress and we very much welcome contributions from the community.

** Xiangshan processors will always adhere to open source, and we firmly open source all our design, verification, and basic tool codes. ** We are very grateful for the community's contribution to Xiangshan. In terms of hardware design, some module designs of Xiangshan processors were inspired by open source processors and public papers, and we have referred to existing open source rocket-chip, berkeley-hardfloat, SiFive block-inclusivecache and other codes. We have modified and improved the functions of the existing bus tools, floating point units, system caches, etc. of the Chisel open source community, and optimized performance indicators such as frequency and throughput. At the same time, we very much welcome the community to develop based on Xiangshan or use the code of the Xiangshan project. Among many open source protocols, we chose [Mulan Permissive License Version 2](http://license.coscl.org.cn/MulanPSL2), hoping to (1) always maintain the openness of Xiangshan processors. The Mulan Permissive License is not contagious and users can use it with confidence; (2) based in China and facing the world, the Mulan Permissive License is expressed in both Chinese and English, and the Chinese and English versions have equal legal effect.

**XiangShan actively embraces the open source community and welcomes contributions from the community.** We have seen that some open source RISC-V processor projects rarely receive external code submissions. We understand that there are multiple conceptual and technical reasons behind this phenomenon. For the XiangShan project, we welcome external contributions, such as submitting questions, submitting feature requirements, submitting code, etc. We will seriously consider and evaluate every opinion and suggestion. For example, Chisel is still new to the industry, if you are more familiar with Verilog/SystemVerilog but want to submit code contributions to XiangShan, we welcome you to [this repo](https://github.com/OpenXiangShan/XS-Verilog-Library) submit a Pull Request. At the technical level, we hope to explore a set of processes and tools for evaluating the quality of code changes, and the basic process determines whether to accept a code submission. For example, we hope that in the near future, we will open a faster and more accurate performance sampling framework to evaluate the performance benefits brought about by an architecture change. We will accept this code modification when it has certain performance benefits, good code style and other conditions. If we sum up our developer strategy in one sentence, we welcome any discussion, issue submission, code modification, etc. that are beneficial to XiangShan.

**In addition to microarchitecture exploration, the XiangShan project also hopes to explore and establish an agile development process for high-performance processors.** The goal of XiangShan Processor is to become an open source platform for architecture innovation facing the world. The establishment of basic capabilities, facilities, and processes is the key to the long-term high-quality development of XiangShan Processor. We will maintain long-term and stable investment and continue to strive to build processors The infrastructure and basic process of agile development. In the early stage of the XiangShan project, we followed the development and verification framework of the NutShell processor. During the advancement of the XiangShan project, we have made a lot of improvements to it, adding features including simulation checkpoints, compressed file loading, multi-core verification support, and more. At present, the verification environment of XiangShan processor has been greatly improved compared with that of NutShell processor, and rich basic tools support XiangShan's agile verification process of this complex scale. In addition, the UCB and Chipyard frameworks are role models for us to learn from, and we refer to or use many open source projects initiated by them. We hope that with the advancement and deepening of the XiangShan project, we can promote the continuous progress of the open source community, and together with the open source community, promote the establishment of an agile development process and infrastructure for high-performance processors.

**We clearly realize that there is still a big gap between XiangShan processors and the mainstream level of the industry. For example, we are not good enough in the selection of solutions for many technical points. The goal of XiangShan Processor is to become an open source platform for architecture innovation facing the world. We welcome suggestions and opinions from industry seniors, high-performance processor enthusiasts, and open source communities. As long as it is beneficial to the XiangShan Processor Project, we will Accept and improve. At the same time, we also welcome and encourage more people to join in the development of XiangShan processors to promote the continuous innovation of the XiangShan project. **


## Recommended reports about Xiangshan

Xiangshan always adheres to the principle of seeking truth from facts, always maintains the original intention of building an open source platform and exploring agile development, and does not want any content that may cause misunderstanding in publicity.

Since Xiangshan was officially released at the RISC-V China Summit held in June 2021, we have seen a lot of reports on Xiangshan processors on social media, self-media, and news media, some of which are "self-play" by the author and may mislead readers. For this reason, we have specially created a [Rumor Refuting Zone](https://github.com/OpenXiangShan/XiangShan-doc/tree/main/clarifications) to clarify some widely circulated misunderstandings.

We recommend the following reports about Xiangshan processors, which come from our official release and popular science channels:

- Bao Yungang's report at the 2023 RISC-V European Summit

- Bao Yungang's personal introduction to Xiangshan processors

- The official WeChat account of the China Open Instruction Ecosystem (RISC-V) Alliance

- The source code of Xiangshan processors

One of Xiangshan's core concepts is open source and openness. For this topic, we recommend reading [Academician Sun Ninghui's "On the Spirit of Open Source"](https://mp.weixin.qq.com/s/1Irs9a0EKoB7P-J_4ju66A).
