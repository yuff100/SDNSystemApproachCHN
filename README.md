[Github仓库](https://github.com/SystemsApproach/SDN)包含*Software-Defined Networks: A Systems Approach*的所有源文件，可根据[创作共用协议(CC BY-NC-ND 4.0)许可条款](https://creativecommons.org/licenses/by-nc-nd/4.0)获得。我们邀请社区在相同的条款下提供更正、改进、更新和新材料。虽然本授权并不自动授予制作衍生作品的权利，但非常希望与感兴趣的各方讨论衍生作品(如翻译)。请联系discuss@systemsapproach.org。

如果使用了该作品，其版权归属应包括以下信息:

*Title: Software-Defined Networks: A Systems Approach*\
*Authors: Larry Peterson, Carmelo Cascone, Brian O’Connor, Thomas Vachuska, and Bruce Davie*\
*Source: https://github.com/SystemsApproach/SDN*\
*License: CC BY-NC-ND 4.0*

#### 阅读本书

本书是[系统方法系列](https://www.systemsapproach.org)的一部分，其在线版本发布在https://sdn.systemsapproach.org。

要跟踪进度并接收关于新版本的通知，可以关注[Facebook](https://www.facebook.com/Computer-Networks-A-Systems-Approach-110933578952503)和[Twitter](https://twitter.com/SystemsAppr)。要阅读关于互联网是如何发展的连续评论，请关注[Substack](https://systemsapproach.substack.com)。

#### 发布和版本

我们不断在GitHub上发布开源内容，并不时发布印刷和电子书版本。最新的印刷和电子书(第二版)对应于```v2.0```标签。

一般来说，```master```包含连贯以及内部一致的版本的材料。(如果是代码，那么本书就会构建并运行。)小补丁会直接检入到```master```中，但是开发中的新内容会检入到分支中，直到可以被合并到```master```而不会破坏一致性。该书的网页版本会不断从```master```生成并可在https://sdn.systemsapproach.org上访问。

只要有足够的改进或新内容，就会标记小版本(例如```v1.1```)。这样做主要是为了创建快照，以便课程中的每个人都可以知道他们使用的是同一个版本。

#### 构建本书

要构建可供网页浏览的版本，首先需要下载源码:

```shell
$ mkdir ~/SDN
$ cd ~/SDN
$ git clone https://github.com/SystemsApproach/SDN.git
```

构建过程维护在Makefile中，并且需要安装Python。Makefile将创建一个虚拟环境(```doc_venv```)，用于安装文档生成工具集。可能还需要使用系统的包管理器安装```enchant```C库，以便拼写检查器正常工作。

执行```make HTML```，在```_build/html```中生成HTML。

执行```make lint```检查本书格式。

执行```make spelling```进行检查拼写。如果有拼写正确但字典中没有的单词、名称或首字母缩写词，请添加到```dict.txt```文件中。

运行```make```查看其他可用的输出格式。

#### 贡献

如果你使用这些材料，希望也愿意回馈。如果你是开放源码的新手，可以查看[How to Contribute to Open Source](https://opensource.guide/how-to-contribute)。你可以了解如何发布Issue，如何发出Pull Request合并你的改进，以及其他需要的东西。

如果想投稿，并且正在寻找一些需要关注的内容，请查看[wiki](https://github.com/SystemsApproach/SDN/wiki)上的当前待办事项列表。
