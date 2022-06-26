### SDN的未来

SDN还处于早期阶段，云托管的控制平面已经部署在生产网络中，但在接入网应用SDN以及在数据平面应用可编程流水线还只是刚刚开始。企业在不同程度上采用了网络虚拟化和SD-WAN，但传统网络仍远多于SDN网络。

随着技术的成熟和API的稳定，我们期望看到越来越多的人采用前面讨论的用例，但对SDN最终扮演的角色影响最大的可能是还没有出现的新用例。事实上，支持传统网络中不可能实现的功能的能力是SDN承诺的重要组成部分。 

本章着眼于一个很有前途的机会: 围绕正确性的验证。众所周知，网络很难对失败、攻击和配置错误进行健壮性和安全性的验证。虽然多年来网络验证一直是令人感兴趣的领域，但由于网络缺乏明确的抽象概念，限制了可以取得的进展。大多数网络仍然使用封闭/专有的软件和复杂/固定功能的硬件，其设计来源不明，其正确性很难被证明。众所周知，确定网络如何运行的分布式算法是很难推理的，BGP就是一个典型，其故障模式已经让研究人员和实践者忙碌了几十年。 

5G网络及相关应用的出现只会加剧这种情况。5G网络不仅将连接智能手机和人，还将连接从门铃、电灯、冰箱、自动驾驶汽车和无人机等一切事物。如果我们不能正确配置和保护这些网络，网络灾难的风险将比迄今为止所经历的任何事情都要严重得多。

可靠和安全的互联网的关键能力是可验证性，从而确保网络中的每个包遵循运营商指定的路径，并且只在运营商希望的设备中匹配一组转发规则，不多不少。

经验表明，在整个系统以组合(即解耦)的方式构建的环境中，可以最好的实现验证。因为这样能够对小块组件进行推理，从而使验证变得容易处理，将组件组合到复合系统中所需的推理也可以带来洞见。以解耦为基础，可验证性来自于(a)在网络级别而不是设备级别上陈述意图的能力，以及(b)在细粒度和实时观察行为的能力。 这正是SDN带来的价值，这让人们乐观的认为，可验证的闭环控制现在已经触手可及。

>延伸阅读:\
>N. Foster, et. al. [Using Deep Programmability to Put Network Owners in Control](https://ccronline.sigcomm.org/2020/ccr-october-2020/using-deep-programmability-to-put-network-owners-in-control "Using Deep Programmability to Put Network Owners in Control"). ACM SIGCOMM Computer Communication Review, October 2020.

图60说明了其基本思想。本书介绍的软件栈增强了可验证闭环控制所需的测量、代码生成和组件验证能力。基于INT(Inband Network Telemetry)实现的细粒度测量，允许转发组件对每个包进行标记，以表明经过的路径、经历的排队延迟以及匹配的规则。这些度量可以被分析，并反馈到代码生成和正式的验证工具中。该闭环补充了解耦的内在价值，使得根据构造进行正确推理成为可能。 

![图60. INT生成精细的测量值，这些测量值反过来提供验证网络行为的闭环控制回路。](https://files.mdnice.com/user/28815/b50caf67-e73f-4b30-b3f7-847ff4248c5a.png)

其目标是使网络运营商能够自顶向下指定网络行为，在每个接口上验证正确性。在最低层，P4程序指定如何处理数据包，这些程序编译后在转发平面组件内执行。这种方法代表了传统设计中不可能实现的新的基本功能，提供了如下两个关键见解。

首先，虽然网络控制平面本质上很复杂，但通过P4数据平面捕捉网络的真实情况，因此是部署验证技术的极有吸引力的平台。通过观察并验证数据平面级别的行为，可以减少可信计算的基础: 交换机操作系统、驱动程序和其他低级组件不需要被信任。此外，虽然控制平面倾向于用通用语言编写，并且比较复杂，但数据平面必然相对简单，并且最终被编译成具有简单数据类型和有限状态的高效前馈流水线体系结构。在一般情况下，对通用软件进行验证是不可能的，但数据平面验证功能强大且实用。

这种实用性的主张基于当前的先进技术，一旦知道了转发行为，就可以通过转发表状态来定义。例如，如果已知所有路由器都是IPv4转发，那么所有路由器的转发表状态就足以定义网络行为。这种想法已经被Veriflow和Header Space Analysis(HSA)等技术简化为实践，现在已经商业化。知道这种状态足以验证具有固定转发行为的网络意味着我们"仅仅"增加了一个新的自由度: 允许网络运营商使用P4对转发行为进行编程(并随着时间的推移进行进化)。使用P4对数据平面进行编程是关键: 该语言小心排除了循环和基于指针的数据结构等特性，这些特性通常会使分析不切实际。要了解更多关于这个机会的信息，推荐Jed Liu及其同事的一篇论文。 

>延伸阅读:\
>J. Liu, et. al. [p4v: Practical Verification for Programmable Data Planes](http://yuba.stanford.edu/~nickm/papers/p4v.pdf "p4v: Practical Verification for Programmable Data Planes"). ACM SIGCOMM 2018.

>**自上而下的验证(Top-Down Verification)**
>
>*本章描述的验证网络的方法类似于芯片设计中使用的方法。最上面是行为模型，然后在寄存器转移级别是Verilog或VHDL模型，最后在下面的是晶体管、多边形和金属。通过工具在每个边界和抽象级别上正式验证正确性。*
>
>*这就是我们在这里讨论的一个模型: 在自顶向下的设计方法中跨边界验证。这是由软件栈定义的新的SDN接口和抽象实现的，一直扩展到交换芯片提供的可编程转发流水线。*
>
>*硬件验证的经验表明，这种方法在组合系统中最有效，每个最小组件都可以单独验证或可靠的测试。然后，当组件在层边界上组合时，就应用正式的工具。*

第二个观点是，除了构建分析网络程序的工具之外，开发将网络运营商的高级意图映射到实现该意图的代码的技术也很重要。目前的网络验证方法面临的挑战之一是，它们以现有网络设备及其复杂的分布式控制平面为出发点，建立这些控制平面行为的数学模型。如果现实与模型不精确匹配，那么验证就不能确保网络按要求运行。但是在SDN的集中控制模型中，控制平面被设计成将一个集中指定的请求映射成一组可以在数据平面中实现的控制指令。我们开始看到SDN控制平面本身是由其所需属性的高级规范编译而成的系统。因此，可以希望看到的控制平面是正确构建的，而不用试图建立模型，以准确捕捉历史上难以分析的系统行为，比如BGP<sup>[1]</sup>。 

>[1] 在域间路由领域，很难想象BGP会完全消失，但至少对于大量域内用例来说，为可验证性设计的机会似乎是可能的。

![图61. 展望未来，SDN的第三阶段专注于可验证的、自上而下的网络行为控制。](https://files.mdnice.com/user/28815/52dcc4ec-5227-4f01-b8c0-97c7c8435e7e.png)

为了将这一切放在历史背景中，图61说明了SDN的三个阶段的视图。公平的说，我们处于第二阶段的早期，最先进的运营商已经能够通过分散的控制平面控制他们的软件，通过P4可编程数据平面控制数据包处理。我们看到了第三阶段的兴起，在此期间，可验证的闭环控制将使网络运营商获得定义其网络的软件的完全所有权。他们不仅能够通过软件来确定网络的行为，而且还能够提供网络实现他们的意图。就像硬件行业在芯片投入生产之前就已经建立了高度的信心，相信芯片会按照预期工作一样，网络运营商也会相信他们的网络是可靠的、安全的，并且满足他们指定的目标。

### 动手编程

本书提供了一组编程练习，用来实践书中描述的软件，包括:

- 使用Stratum的P4Runtime、gNMI、OpenConfig和gNOI接口
- 使用ONOS控制P4可编程交换机
- 编写ONOS应用程序实现控制平面逻辑
- 在Mininet中使用bmv2测试软件栈
- 使用PTF测试基于P4的转发平面

这些练习假设你熟悉Java和Python，每个练习都附带初学者代码，因此不需要很高的熟练程度。练习也使用*Mininet*网络模拟器，基于P4的*bmv2*交换机模拟器，*PTF*包测试框架，以及*Wireshark*协议分析器。关于这些软件工具的附加信息在单独的练习中提供。

这些练习起源于ONF制作的*下一代SDN教程*，附带一系列在线教程幻灯片，介绍了练习中涉及的主题:

- [Advanced Next-Gen SDN Tutorial Overviewhttp](https://docs.google.com/presentation/d/1_Iw5Bbm_cxIi-0eIWKpOpDuHuRQrQwT_4rEYdWCaWQ8/edit#slide=id.p "Advanced Next-Gen SDN Tutorial Overviewhttp")

这些幻灯片与这本书涉及的材料有很大重叠，所以不必从幻灯片开始学习，但是很好的补充资源。

#### 环境

可以在笔记本电脑上运行的Linux虚拟环境中进行练习。本节将介绍安装环境的准备工作。

##### 系统需求

虚拟机当前配置为4GB内存，4核CPU，这是完成练习的推荐的最低系统需求，虚拟机还占用大约8GB硬盘空间。为了获得平稳的体验，建议在资源至少翻倍的系统上运行虚拟机。

##### 下载虚拟机

单击以下链接下载虚拟机(4GB):

- http://bit.ly/ngsdn-tutorial-ova

虚拟机格式为```.ova```，基于VirtualBox v5.2.32创建。可以使用任何现代虚拟化系统来运行虚拟机，但建议使用VirtualBox。以下链接提供了如何获取VirtualBox并导入虚拟机的说明:

- https://www.virtualbox.org/wiki/Downloads
- https://docs.oracle.com/cd/E26217_01/E26796/html/qs-import-vm.html

或者，可以使用[脚本](https://github.com/opennetworkinglab/ngsdn-tutorial/tree/advanced/util/vm "Scripts to build the tutorial VM")在机器上通过Vagrant构建虚拟机。

>**Windows用户**\
>所有脚本都在MacOS和Ubuntu上测试过了，虽然应该可以在Windows上运行，但还没有经过测试，建议Windows用户下载所提供的虚拟机。

然后可以启动虚拟机(Ubuntu系统)，并使用```sdn```/```rocks```登录。本节剩余部分给出的说明(以及练习本身)将在运行的VM中执行。

##### 克隆仓库

为了进行练习，需要克隆以下仓库:

```shell
$ cd ~
$ git clone -b advanced https://github.com/opennetworkinglab/ngsdn-tutorial --depth 1
```

如果```ngsdn-tutorial```目录已经存在于虚拟机中，请确保更新其内容:

```shell
$ cd ~/ngsdn-tutorial 
$ git pull origin advanced 
```

请注意，仓库有多个分支，每个分支有不同的练习配置，一定要确保是在```advanced```分支上。

##### 升级依赖

虚拟机可能附带了比练习所需的更旧的依赖版本，可以使用以下命令升级到最新版本:

```shell
$ cd ~/ngsdn-tutorial 
$ make deps 
```

该命令将下载所有必需的Docker镜像(约1.5 GB)，从而允许我们离线完成练习。

##### 使用IDE

练习中将需要用多种语言(如P4, Java, Python)编写代码，虽然不需要使用任何特定IDE或代码编辑器，但推荐使用[Java IDE IntelliJ IDEA Community Edition](https://www.jetbrains.com/idea "Java IDE IntelliJ IDEA Community Edition")，它预装了用于P4语法高亮显示和Python开发的插件，特别是在ONOS应用程序开发中，可以为所有ONOS API提供代码补全。

##### 代码库结构

代码库结构如下:

- ```p4src\``` → 数据平面实现(P4)
- ```yang\``` → 配置模型(YANG)
- ```app\``` → 自定义ONOS应用程序(Java)
- ```mininet\``` → 2x2叶棘网络(Mininet)
- ```util\``` → 工具脚本(Bash)
- ```ptf\``` → 数据平面单元测试(PTF)

注意，练习需要用到GitHub上各种文件的链接，不要忘记在笔记本电脑上克隆相应文件。

##### 命令

为了便于练习，代码库提供了一组```make```目标，以控制流程的不同方面。具体命令在练习中介绍，以下是一个快速参考:

- ```make deps``` → 拉取并构建所有必需的依赖项
- ```make p4-build``` → 构建P4程序
- ```make p4-test``` → 执行PTF测试
- ```make start``` → 启动Mininet和ONOS容器
- ```make stop``` → 停止所有容器
- ```make restart``` → 重新启动容器并清除任何先前状态
- ```make onos-cli``` → 进入ONOS CLI(密码: ```rocks```, Ctrl-D退出)
- ```make onos-log``` → 显示ONOS日志
- ```make mn-cli``` → 进入Mininet命令行(Ctrl-D退出)
- ```make mn-log``` → 显示Mininet日志(例如CLI的输出信息)
- ```make app-build``` → 构建自定义ONOS应用程序
- ```make app-reload``` → 安装并激活ONOS应用程序
- ```make netcfg``` → 将网络配置文件```netcfg.json```推送到ONOS

>**执行命令**\
>提醒一下，这些命令需要在刚刚创建的虚拟机中打开的终端窗口中执行，同时确保位于克隆的代码库的根目录中(主```Makefile```所在的位置)。

#### 练习

下面列出(并链接)每个练习。练习1和2的重点是Stratum，最好在阅读完第5章后尝试。练习3至6关注ONOS，最好在阅读了第6章后尝试。练习7和8主要关注SD-Fabric，最好在阅读第7章之后进行<sup>[1]</sup>。注意，这些练习是相互建立的，所以最好按顺序完成它们。

>[1] SD-Fabric以前被称为Trellis，现在仍然可以在代码中看到。UPF以前被称为SPGW，现在同样可以在代码中看到。

1. [P4Runtime Basics](https://github.com/opennetworkinglab/ngsdn-tutorial/blob/advanced/EXERCISE-1.md "P4Runtime Basics")
2. [YANG, OpenConfig, gNMI Basics](https://github.com/opennetworkinglab/ngsdn-tutorial/blob/advanced/EXERCISE-2.md "YANG, OpenConfig, gNMI Basics")
3. [Using ONOS as the Control Plane](https://github.com/opennetworkinglab/ngsdn-tutorial/blob/advanced/EXERCISE-3.md "Using ONOS as the Control Plane")
4. [Enabling ONOS Built-in Services](https://github.com/opennetworkinglab/ngsdn-tutorial/blob/advanced/EXERCISE-4.md "Enabling ONOS Built-in Services")
5. [Implementing IPv6 Routing with ECMP](https://github.com/opennetworkinglab/ngsdn-tutorial/blob/advanced/EXERCISE-5.md "Implementing IPv6 Routing with ECMP")
6. [Implementing SRv6](https://github.com/opennetworkinglab/ngsdn-tutorial/blob/advanced/EXERCISE-6.md "Implementing SRv6")
7. [SD-Fabric (Trellis) Basics](https://github.com/opennetworkinglab/ngsdn-tutorial/blob/advanced/EXERCISE-7.md "SD-Fabric (Trellis) Basics")
8. [GTP Termination with fabric.p4](https://github.com/opennetworkinglab/ngsdn-tutorial/blob/advanced/EXERCISE-8.md "GTP Termination with fabric.p4")

可以在代码库的```solution```子目录找到每个练习的解决方案，如果遇到问题，可以将自己的解决方案与参考解决方案进行比较。

>**图形界面**\
>当练习需要查看图形输出时，将会看到对*ONF Cloud Tutorial Portal*的引用，这是ONF运行教程期间使用的云托管虚拟机，因此并不适用。在同样的地方还可以看到如何访问本地运行的GUI的介绍。

### 关于本书

[本仓库](https://github.com/SystemsApproach/SDN "Software-Defined Networks: A Systems Approach")包含*Software-Defined Networks: A Systems Approach*的所有源文件，可根据[创作共用协议(CC BY-NC-ND 4.0)许可条款](https://creativecommons.org/licenses/by-nc-nd/4.0 "创作共用协议(CC BY-NC-ND 4.0)许可条款")获得。我们邀请社区在相同的条款下提供更正、改进、更新和新材料。虽然本授权并不自动授予制作衍生作品的权利，但非常希望与感兴趣的各方讨论衍生作品(如翻译)。请联系discuss@systemsapproach.org。

如果使用了该作品，其版权归属应包括以下信息:

*Title: Software-Defined Networks: A Systems Approach*\
*Authors: Larry Peterson, Carmelo Cascone, Brian O’Connor, Thomas Vachuska, and Bruce Davie*\
*Source: https://github.com/SystemsApproach/SDN*\
*License: CC BY-NC-ND 4.0*

#### 阅读本书

本书是[系统方法系列](https://www.systemsapproach.org "系统方法系列")的一部分，其在线版本发布在https://sdn.systemsapproach.org。

要跟踪进度并接收关于新版本的通知，可以关注[Facebook](https://www.facebook.com/Computer-Networks-A-Systems-Approach-110933578952503/ "Computer Networks A Systems Approach Facebook")和[Twitter](https://twitter.com/SystemsAppr "Systems Approach Twitter")。要阅读关于互联网是如何发展的连续评论，请关注[Substack](https://systemsapproach.substack.com/ "Systems Approach Substack")。

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

如果你使用这些材料，希望也愿意回馈。如果你是开放源码的新手，可以查看[How to Contribute to Open Source](https://opensource.guide/how-to-contribute "How to Contribute to Open Source")。你可以了解如何发布Issue，如何发出Pull Request合并你的改进，以及其他需要的东西。

如果想投稿，并且正在寻找一些需要关注的内容，请查看[wiki](https://github.com/SystemsApproach/SDN/wiki "Systems Approach wiki")上的当前待办事项列表。

### 关于作者

**Larry Peterson**，普林斯顿大学计算机科学系Robert E. Kahn名誉教授，从2003年到2009年担任主席。研究主要集中在互联网大规模分布式系统的设计、实现和操作，包括广泛使用的PlanetLab和MeasurementLab平台。他目前正在为开放网络基金会(ONF)的Aether边缘云项目做出贡献，并作为首席科学家提供建议。Peterson是美国国家工程院院士，ACM和IEEE院士，2010年IEEE Kobayashi计算机与通信奖得主，2013年ACM SIGCOMM奖得主。1985年获普渡大学博士学位。 

**Carmelo Cascone**，开放网络基金会(ONF)技术人员，目前在ONF项目(如ONOS、CORD和Aether)中领导可编程交换机、P4和P4Runtime的技术活动。2017年，Cascone在与École Polytechnique de Montréal的联合项目中获得了米兰理工大学博士学位。他对计算机网络和系统有着广泛的兴趣，专注于数据平面可编程性和软件定义网络(SDN)。

**Brian O’connor**，开放网络基金会(ONF)技术人员，目前领导着交换机操作系统方面的技术活动。O’connor于2012年和2013年分别获得斯坦福大学计算机科学学士和硕士学位。

**Thomas Vachuska**，开放网络基金会(ONF)首席架构师，领导ONOS项目。在加入ONF之前，Vachuska是惠普的软件架构师。Vachuska于1994年获得加州州立大学萨克拉门托分校数学学士学位。

**Bruce Davie**，计算机科学家，因其在网络领域的贡献而闻名。Davie是VMware亚太区前副总裁兼首席技术官，在收购软件定义网络(SDN)初创公司Nicira期间加入VMware。在此之前，他是Cisco Systems的研究员，领导一个架构师团队，负责多协议标签交换(MPLS)。Davie拥有超过30年的网络行业经验，并合著了17个RFC。他于2009年成为ACM研究员，并于2009年至2013年担任ACM SIGCOMM主席。他还在麻省理工学院做了五年的访问讲师。Davie是多本书的作者，拥有40多项美国专利。
