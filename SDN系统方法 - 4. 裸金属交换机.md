### 第4章 裸金属交换机

本章将介绍作为SDN网络底层硬件基础的裸金属交换机，我们的目标不是介绍详细的硬件原理，而是提供足够的设计概念，从而能够欣赏在其上运行的软件栈。请注意，这是个仍在发展中的技术栈，不同厂商采用了不同的实现方法。本章讨论了作为交换机数据平面编程语言的P4以及作为第一代解决方案的OpenFlow。我们将从更一般的、可编程的P4开始，按照逆时间顺序介绍这两种方法。

#### 4.1 交换层概要(Switch-Level Schematic)

首先，我们把裸金属交换机作为整体来考虑，可以类比为一台由一组商业化的、现成的组件组装而成的PC。事实上，利用这些组件构建交换机的完整架构规范可以在*开放计算项目(OCP, Open Compute Project)* 上在线获得。这是类似于开源软件的硬件项目，类似于家用PC，任何人都有可能依此构建高性能交换机。但就像PC生态系统包括戴尔和惠普这样的商品服务器供应商一样，也可以从裸金属交换机供应商(如EdgeCore、Delta等)那里购买预构建(OCP兼容)交换机。

图18给出了裸金属交换机的高层示意图。*网络处理单元(NPU, Network Processing Unit)* 是一种商业硅交换芯片，被优化来解析包头并做出转发决策。NPU能够以每秒太比特(Tbps)的速度处理和转发数据包，其速度足以与32x100-Gbps端口或图中所示的48x40-Gbps端口保持一致。在撰写本文时，这些芯片的最先进技术是在400 Gbps端口实现的25.6 Tbps。

![图18. 裸金属交换机高层概要。](https://files.mdnice.com/user/28815/5882753a-99a3-4352-8cb1-d0558ddaf5fb.png)

请注意，我们对术语NPU的使用可能会被认为有点不标准。过去，NPU指的是更狭义的网络处理芯片，用于实现智能防火墙或深度包检测。它们不像我们在本章讨论的NPU那样通用，性能也不高。然而，长期趋势是NPU的性能越来越接近固定功能ASIC，同时提供更高的灵活性。目前的商用硅交换芯片似乎有可能淘汰早期的专用网络处理器。这里使用的NPU命名规范与构建可编程特定处理器的行业趋势一致，包括用于图形处理的GPU(图形处理单元, Graphic Processing Unit)和用于AI的TPU(张量处理单元, Tensor Processing Unit)。

图18显示的NPU由基于SRAM的内存(在处理数据包时进行缓冲)和基于ASIC的转发流水线组合实现，这些流水线通过一系列(Match, Action)对实现，我们将在下一节中更详细介绍转发流水线。交换机还包括一个通用处理器，通常是一个x86芯片，用以控制NPU。如果控制面在交换机上运行，那么CPU将运行BGP或OSPF，但就我们的目的而言，控制面是在交换机外以Switch OS的方式运行的，因此需要导出控制数据平面的API。该控制处理器与NPU通信，并通过标准PCIe总线连接到外部管理网络。

图18还显示了使这一切变得实用的其他商品组件。无论是40G以太网、10G PON还是SONET或者光学设备，都可以购买可插拔收发模块，用于处理所有媒介接入细节。这些收发器都符合SFP+等标准化要求，也可以反过来通过标准化总线连接到其他组件(如SFI)。最关键的是，网络行业正在进入计算机行业在过去20年里所享受的同样的商品化世界。

最后，虽然图18中没有显示，但每个交换机都包括BIOS，很像微处理器中的对应物，用来提供和引导裸金属交换机的固件。在OCP的指导下，出现了一种被称为*开放网络安装环境(ONIE, Open Network Install Environment)* 的标准BIOS，因此我们在本章的其余部分假定基于ONIE讨论。

#### 4.2 转发流水线(Forwarding Pipeline)

高速交换机使用多级流水线处理数据包。使用多级流水线而不是单级处理器的原因在于，单个包的转发可能涉及到查看多个报头字段，每个阶段都可以通过编程来处理不同的领域组合。多阶段流水线给每个包增加了一点端到端延迟(以纳秒为单位)，但意味着可以同时处理多个包。例如，在阶段1对包B进行初始查找时，阶段2可以对包A进行二次查找，以此类推。从而意味着流水线作为一个整体能够匹配所有输入端口的总带宽。重复上一节中的数字，目前最先进的是25.6 Tbps。

一个特定NPU如何实现流水线的主要区别在于，每个阶段功能是固定的(即每个阶段都知道如何处理某些固定协议的报头)还是可编程的(即每个阶段都是通过动态编程知道要处理哪些报头字段)。在接下来的讨论中，我们从更一般的情况(可编程流水线)开始，并在最后回到固定功能的对应流水线。

在体系架构层面，可编程流水线通常被称为*协议独立交换架构(PISA, Protocol Independent Switching Architecture)*。图19给出了PISA概述，包括三个主要组成部分。第一个是*解析器(Parser)*，通过编程定义哪些报头字段(以及在包中的位置)将被后面的阶段识别和匹配。第二个是*匹配操作单元(Match-Action Unit)*，每个单元都被编程来匹配(并可能对其进行操作)一个或多个已标识的报头字段。第三个是*编码器(Deparser)*，将包元数据重新序列化到包中，然后在输出链路上传输。deparser根据之前处理阶段缓存在内存中的所有报头字段重新构建在链路上传输的每个包。

图中没有显示关于穿越流水线的数据包的元数据集合，包括每个包的状态(如输入端口和到达时间戳)，以及跨连续包计算的流级别状态(如交换机计数器和队列深度)。这种元数据(对应于ASIC的寄存器)，可用于各个阶段的读写，也可以被Match-Action Unit使用(例如匹配输入端口)。

![图19所示. PISA多级流水线概要。](https://files.mdnice.com/user/28815/840a9259-9111-4a6d-adae-5234f0699f6e.png)

图19中的各个Match-Action Unit值得我们仔细观察。图中显示的内存通常是基于SRAM和TCAM组合构建，实现了一个存储正在处理的数据包中匹配的位模式的表。在这一特定组合中，TCAM比SRAM更昂贵、更耗电，但能够支持通配符匹配。具体来说，TCAM中的"CAM"代表"内容可寻址内存(Content Addressable Memory,)"，意味着希望在表中查找的键可被有效用作实现该表的内存地址。"T"代表"三元(Ternary)"，这是一种技术方法，表示想要查找的键可以包含通配符(例如，键10\*1可以同时匹配1001和1011)。从软件角度来看，主要结论是通配符匹配比精确匹配更昂贵，应该尽可能避免。

然后，图中显示的ALU与相应的模式配对实现操作。可能的操作包括修改特定报头字段(例如，减少TTL)，增加或弹出标签(例如，VLAN, MPLS)，增加或清除交换机内部的各种计数器(例如，处理的数据包个数)，以及设置用户/内部元数据(例如，路由表中使用的VRF ID)。

直接编写parser、match-action unit和deparser会很乏味，类似于编写微处理器汇编代码，所以我们改用像P4这样的高级语言来表示所需行为，并依赖编译器生成等价的低级程序。我们将在后面章节讨论P4的细节，现在我们用图20所示的更抽象的方式来代替所需的转发流水线。(为了与其他示例保持一致，我们将此程序称为```forward.p4```。)该示例程序首先匹配L2报头字段，然后匹配IPv4或IPv6报头字段，最后对数据包应用一些ACL规则，然后允许它们通过(例如，将后者视为防火墙过滤规则)。这是第1.2.3节中图7中所示OpenFlow流水线的一个示例。

除了将流水线的高级表示转换为底层PISA之外，P4编译器还负责分配可用的PISA资源。在这种情况下，有4个槽位(或行)可用作Match-Action Unit，如图19所示。在可用的Match-Action Unit中分配槽位(slot)类似于传统编程语言在通用微处理器上分配寄存器。在我们的例子中，假设IPv4的Match-Action规则比IPv6或ACL规则多得多，编译器可以相应的在可用的Match-Action Unit中分配条目。

![图20. 将期望的转发行为(由图示P4程序指定)映射到PISA。](https://files.mdnice.com/user/28815/0e6cae11-398b-4815-9b8e-3d0ccf37d053.png)

#### 4.3 流水线抽象(Abstracting the Pipeline)

另一个难题是考虑不同交换芯片实现的不同物理流水线。要做到这一点，需要一个足够通用，可以公平表示可用硬件的抽象(规范)流水线，以及抽象流水线如何映射到物理流水线的定义。有了这样一个流水线的逻辑模型，就能够支持流水线无关的控制器，如图21所示。

理想情况下，只有一个逻辑流水线，P4编译器负责将该逻辑流水线映射到各种对应的物理流水线。不幸的是，市场还没有统一的单一逻辑流水线，但我们暂时把这种复杂性放在一边。另一方面，目前需要考虑的目标ASIC大约有10个。市场上有数十家交换机供应商，但在实践中，我们主要需要考虑那些为高端市场服务的供应商。

![图21. 通过定义逻辑流水线作为支持流水线无关控制平面的通用方法。](https://files.mdnice.com/user/28815/eb805f2f-d649-4a9e-91c0-e7248e78f15e.png)

如何指定逻辑流水线？同样是通过P4程序实现，结果如图22所示。注意，我们正在重新审视图15中引入的两个P4程序。第一个程序(```forward.p4```)定义了我们希望从可用的交换芯片获得的功能。这个程序是由想要建立数据平面行为的开发人员编写的。第二个程序(```arch.p4```)本质上是一个头文件，表示P4程序和P4编译器之间的契约。具体来说，```arch.p4```定义了可用的P4可编程块、每个阶段的接口以及每个阶段的功能。谁负责编写这样的架构程序？P4联盟是这种定义的一个来源，但不同的交换机供应商已经创建了自己的架构规范，从而密切描述其交换芯片的能力。因为有单一公共架构，可以在不同供应商的不同ASIC上执行相同的P4程序，并且该架构能够最好很好的表示任何给定ASIC的能力差异。

图22所示例子被称为*可移植交换机架构(PSA, Portable Switch Architecture)*，旨在为P4开发人员提供实现转发程序(如```forward.p4```)的抽象的目标机，类似于Java虚拟机。其目标与Java相同，支持一次编写、随处运行的编程范式。(注意，图22包括了通用```arch.p4```作为体系架构模型规范，但实际上体系架构模型是特定于PSA的，类似于```psa.p4```。)

![图22. P4架构被称为PSA(Portable Switch Architecture)。包括通用```arch.p4```作为架构模型规范，但是对于PSA来说，这将被```psa.p4```所取代。](https://files.mdnice.com/user/28815/01b01bbb-7eb3-4818-9320-344d2846ff8e.png)

与图19和图20中使用的更简单的PISA模型相比，我们看到了两个主要差异。首先，流水线包含一个新的固定功能阶段: *流量管理器(Traffic Manager)* 。这个阶段负责对数据包进行排队、复制和调度，可以用定义良好的方式配置(例如，设置队列大小和调度策略等参数)，但不能用通用方式重新编程(例如，定义一个新的调度算法)。其次，流水线被分为两部分: *入口处理(ingress processing)*(流量管理器左边)和*出口处理(egress processing)*(流量管理器右边)。

到底```arch.p4```定义了什么？基本上是三件事:

1. 如图22所示，根据输入和输出信号(想想“函数参数和返回类型”)定义了块间接口签名。P4开发人员为每个P4可编程块提供对应的实现，处理输入信号(如输入端口收到数据包)，并将输出信号写到下一个受影响的可编程块(例如数据包指示的输出队列/端口)。
2. 被声明为*扩展(externs)* 的类型，可被视为由目标硬件开放的附加固定功能的服务，可以被P4开发人员调用。这类外部服务的例子包括checksum以及哈希计算单元、数据包或字节计数器、加密/解密数据包负载，等等。P4体系架构中不会指定这些外部服务的实现，但会指定其接口。
3. 扩展核心P4语言类型，包括可选匹配类型(如4.4.3节中描述的```range```和```lpm```)。

P4编译器(像所有编译器一样)有一个与硬件无关的前端，为被编译程序生成*抽象语法树(AST, Abstract Syntax Tree)*，以及与硬件相关的后端，从而输出特定ASIC的可执行文件。```arch.p4```只是类型和接口定义的集合。

##### 4.3.1 V1Model

图22中所示的PSA仍在开发中，其代表了一个理想的架构，位于P4开发人员和底层硬件之间，但是开发人员目前正在编写的架构模型稍微简单一些。该模型称为V1Model，如图23.1所示<sup>[1]</sup>。它不包括流量管理器之后的重新解析步骤，相反，它隐式连接了从入口到出口处理的所有元数据。此外，V1Model包括一个checksum验证/更新块，而PSA将checksum视为一个扩展，并支持在入口/出口处理流程的任何点上进行增量计算。

>[1] V1Model最初是作为P4早期版本(即P4_14 "V1Model")的参考体系架构引入，随后被用于简化P4程序从P4_14到P4_16的移植。

本书其余部分将使用这个更简单的模型。说句题外话，V1Model被广泛使用的最重要因素是交换机供应商没有提供从PSA映射到各自ASIC的编译器后端。因此在此之前，PSA还是主要停留在纸面上。

![图23. V1Model在实践中用于抽象出不同物理转发流水线的细节，开发人员为这个抽象架构模型编写P4。](https://files.mdnice.com/user/28815/6e1b78e4-9be4-4037-b99e-8a9c67170b3d.png)

当我们说P4开发人员"编写这个模型"时，其实际含义比你想的更多。在实践中，每个P4程序都从下面的模板开始，它实际为图23所示的每个可编程元素的抽象描述提供了一个代码块。

```c
#include <core.p4>
#include <v1model.p4>

/* Headers */
struct metadata { ... }
struct headers {
	ethernet_t	ethernet;
	ipv4_t		ipv4;
}

/* Parser */
parser MyParser(
	packet_in packet,
	out headers hdr,
	inout metadata meta,
	inout standard_metadata_t smeta) {
    ...
}

/* Checksum Verification */
control MyVerifyChecksum(
	in headers, hdr,
	inout metadata meta) {
    ...
}

/* Ingress Processing */
control MyIngress(
	inout headers hdr,
	inout metadata meta,
	inout standard_metadata_t smeta) {
    ...
}

/* Egress Processing */
control MyEgress(
	inout headers hdr,
	inout metadata meta,
	inout standard_metadata_t smeta) {
    ...
}

/* Checksum Update */
control MyComputeChecksum(
	inout headers, hdr,
	inout metadata meta) {
    ...
}

/* Deparser */
parser MyDeparser(
	inout headers hdr,
	inout metadata meta) {
    ...
}

/* Switch */
V1Switch(
    MyParser(),
    MyVerifyChecksum(),
    MyIngress(),
    MyEgress(),
    MyComputeChecksum(),
    MyDeparser()
) main;
```

也就是说，在包含两个定义文件(```core.p4```, ```v1model.p4```)以及定义流水线将要处理的包头之后，开发人员需要编写p4代码用于解析、验证checksum、处理输入等等。最后一个块(```V1Switch```)是"main"函数，将所有部件整合到一起形成完整的交换机流水线。至于模板中每个"…"对应的细节，我们将在后面章节中详细介绍。现在，最重要的是看到```forward.p4```是一个高度程式化的程序，其结构来自```v1model.p4```中定义的抽象模型。

##### 4.3.2 TNA

正如刚才提到的，V1Model是许多可能的流水线体系架构之一。PSA是另一种，但不同的交换机供应商也提供了自己的体系架构定义。这样做有不同的动机，一是随着时间的推移，供应商不断发布新的芯片，他们有自己版本的多ASIC问题。另一个原因是，它使供应商能够暴露其ASIC的独特功能，而不受标准化过程的限制。*Tofino Native Architecture(TNA)* 就是一个例子，它是Barefoot为其可编程交换芯片家族定义的架构模型。

我们不打算在这里详细介绍TNA(详细信息可以在GitHub上查看)，但可以作为第二个实际案例帮助说明这一领域的自由度。实际上，P4语言定义了一个编写程序的通用框架(我们将在下一节中看到其语法)，但只有在定义了P4体系架构(通常我们称之为```arch.p4```)时(具体例子是```v1model.p4```, ```psa.p4```和```tna.p4```)，开发人员才能够实际编写和编译转发程序。

>延伸阅读:\
>[Open Tofino](https://github.com/barefootnetworks/Open-Tofino "Open Tofino"). 2021.

与渴望在不同交换芯片之间抽象出共性的```v1model.p4```和```psa.p4```相反，```tna.p4```忠实的定义了给定芯片的底层功能。通常，这样的功能就是使像Tofino这样的芯片区别于竞争对手的地方。在为新的P4程序选择架构模型时，一定要问这样的问题: 我要编程的交换机支持哪些可用架构？我的程序是否需要访问特定芯片的功能(例如，用来加密/解密数据包有效负载的P4扩展)，或者能否只依赖通用/非区分特性(例如，简单的匹配操作表或计数数据包的P4扩展)?

至于转发程序(我们通常称之为```forward.p4```)，一个有趣的例子是一个忠实实现传统L2/L3交换机所有特性的程序，我们称之为```switch.p4```<sup>[2]</sup>。奇怪的是，这只是让我们重现构建了一个本可以从几十个供应商处购买的传统交换机，但是有两个显著差异: (1)我们可以使用SDN控制器通过P4Runtime控制交换机，(2)当我们发现需要某个新功能时可以很容易的修改程序。

>[2] Barefoot为他们的芯片组编写了这样一个程序，使用```tna.p4```作为架构模型，但不开源。有一个称为```fabric.p4```的大致相同的开源变体，使用```v1model.p4```，支持大多数L2/L3特性，这些特性是为第7章中介绍的SD-Fabric用例定制的。

总而言之，总体目标是使控制程序的开发不需要考虑设备转发流水线的具体细节。引入P4体系架构模型有助于实现这一目标，使相同的转发流水线(P4程序)可以在支持相应体系架构模型的多个目标(交换芯片)之间移植。然而，这并不能完全解决问题，因为业界仍然可以自由定义多个转发流水线。但从目前情况来看，拥有一个或多个可编程交换机为控制程序和转发流水线的可编程性打开了大门，从而向开发人员展示最终的可编程性，即包括数据面转发芯片在内都是可编程的。因此如果有一个创新的新功能想要注入到网络中，那么需要同时编写该功能的控制平面和数据平面两部分，并通过工具链加载到SDN软件栈中！与几年前相比，这是一个重大进步。在几年前，也许可以修改路由协议(因为是在软件中)，但没有机会改变转发流水线，因为都是在固定功能的硬件中。

>**这种复杂性值得吗?**
>
>*此时，你可能想知道引入的所有复杂性是否值得，我们甚至还没有接触控制平面！到目前为止，所讨论的都是有或没有SDN的复杂问题。这是因为我们工作在软硬件的边界，硬件被设计成以每秒太比特的速度转发数据包。这种复杂性通常隐藏在专有设备中。SDN所做的就是向市场施压，为其他人的创新打开空间。*
>
>*在任何人能够创新之前，第一步是复制我们以前运行的东西，只是现在基于开放接口和可编程硬件。尽管本章使用了```forward.p4```作为假设的新的数据平面功能，实际上通过```switch.p4```这样的程序(以及下一章描述的Switch OS)与传统网络设备建立对应关系。一旦有了这些，就可以准备好做一些新事情了。但是做什么呢？*
>
>*我们的目标不是明确回答这个问题。第二章中介绍的VNF卸载和INT示例可以作为一个开始，软件定义的5G网络(第9章)和闭环验证(第10章)是潜在的杀手级应用。但历史告诉我们，杀手级应用是不可能被准确预测的。另一方面，历史上也有很多关于开放封闭、功能固定的系统如何产生新功能的例子。*

#### 4.4 P4程序

最后我们简要介绍一下P4语言，以下不是P4的全面参考手册，我们的目标是让人们了解P4程序的概要，从而串联起之前引入的所有的点。我们通过示例(即通过遍历实现基本IP转发的P4程序)来实现这一点，这个例子摘自P4教程，可以在网上访问该教程并自己尝试。

>延伸阅读:\
>[P4 Tutorials](https://github.com/p4lang/tutorials "P4 Tutorials"). P4 Consortium, May 2019.

为了帮助理解，可以将P4看作类似于C语言。P4和C共享类似的语法，这很好理解，因为都是为低级系统设计的程序语言。然而，与C不同的是，P4不支持循环、指针或动态内存分配。如果你记得我们的目的是定义在单个流水线阶段发生的事情时，不支持循环就很好理解了。实际上，P4"展开"了可能的循环，在一个控制块序列(即阶段)中实现每个迭代。在下面的示例程序中，可以想象将每个代码块插入到上一节所示的模板中。

##### 4.4.1 头声明与元数据(Header Declarations and Metadata)

首先是协议头声明，对于我们的简单示例，包括以太网头和IP头，这也是定义希望与正在处理的数据包关联的任何特定于程序的元数据的地方。示例中该结构为空，但是```v1model.p4```为整个体系架构定义了标准的元数据结构。尽管下面的代码块中没有显示，但这个标准元数据结构包括诸如```ingress_port```(数据包到达的端口)、```egress_port```(选择发送数据包的端口)和```drop```(置位表示数据包将被丢弃)等字段。这些字段可以由组成程序其余部分的功能块读取或写入<sup>[3]</sup>。

>[3] V1Model的奇特之处在于，元数据结构中有两个出口端口字段。其中一个(```egress_port```)是只读的，仅在出口处理阶段有效。第二个(```egress_spec```)是从入口处理阶段写入的字段，用于选择输出端口。 PSA和其他架构通过为入口和出口流水线定义不同的元数据来解决这个问题。

```c
/***** P4_16 *****/
#include <core.p4>
#include <v1model.p4>

const bit<16> TYPE_IPV4 = 0x800;

/****************************************************
************* H E A D E R S  ************************
****************************************************/

typedef bit<9>  egressSpec_t;
typedef bit<48> macAddr_t;
typedef bit<32> ip4Addr_t;

header ethernet_t {
    macAddr_t dstAddr;
    macAddr_t srcAddr;
    bit<16>   etherType;
}

header ipv4_t {
    bit<4>    version;
    bit<4>    ihl;
    bit<8>    diffserv;
    bit<16>   totalLen;
    bit<16>   identification;
    bit<3>    flags;
    bit<13>   fragOffset;
    bit<8>    ttl;
    bit<8>    protocol;
    bit<16>   hdrChecksum;
    ip4Addr_t srcAddr;
    ip4Addr_t dstAddr;
}

struct metadata {
   /* empty */
}

struct headers {
    ethernet_t   ethernet;
    ipv4_t       ipv4;
}
```

##### 4.4.2 解析器(Parser)

下一个块实现了解析器。解析器的底层编程模型是状态转换图，包括内置的```start```、```accept```和```reject```状态。开发人员可以添加其他状态(在我们的示例中是```parse_ethernet```和```parse_ipv4```)，以及状态转换逻辑。例如，下面的解析器总是从```start```状态转换到```parse_ethernet```状态，如果在以太网报头的```etherType```字段中找到```TYPE_IPV4```(请参阅前面代码块中的常量定义)，那么接下来就转换到```parse_ipv4```状态。作为遍历每个状态的副作用，将从数据包中提取相应的报头。这些内存结构中的值随后可用于其他例程，如下所示。

```c
/****************************************************
************* P A R S E R  **************************
****************************************************/

parser MyParser(
	packet_in packet,
	out headers hdr,
	inout metadata meta,
	inout standard_metadata_t standard_metadata) {

    state start {
        transition parse_ethernet;
    }

    state parse_ethernet {
        packet.extract(hdr.ethernet);
        transition select(hdr.ethernet.etherType) {
            TYPE_IPV4: parse_ipv4;
            default: accept;
        }
    }

    state parse_ipv4 {
        packet.extract(hdr.ipv4);
        transition accept;
    }
}
```

与本节所有代码块一样，解析器的函数签名是由体系架构模型定义的，在本例中为```v1model.p4```。我们没有对具体参数做进一步介绍，只是笼统指出P4是与架构无关的，所编写的程序很大程度上依赖于所包含的架构模型。

##### 4.4.3 入口处理(Ingress Processing)

入口处理分为两部分。首先是验证checksum，在我们的例子中只是应用默认值<sup>[4]</sup>。该示例引入的有趣的新特性是```control```结构，它实际上是P4版本的过程调用。虽然开发人员也可以根据模块化的设计来定义"子例程"，但从整体上看，控制块与逻辑流水线模型定义的阶段一一匹配。

>[4] 这是V1Model特有的，PSA在入口或出口阶段没有明确的checksum验证或计算。

```c
/****************************************************
***  C H E C K S U M    V E R I F I C A T I O N   ***
****************************************************/

control MyVerifyChecksum(
	inout headers hdr,
	inout metadata meta) {   
    apply {  }
}
```

现在进入转发算法的核心，其在Match-Action流水线的入口段实现。我们发现定义了两个```actions```: ```drop()```和```ipv4_forward()```。第二个很有趣，它以```dstAddr```和出口端口作为参数，将端口分配给标准元数据结构中的相应字段，在数据包的以太网报头中设置```srcAddr/dstAddr```字段，并减小IP报头的ttl字段。执行完该动作后，与报文相关的报文头和元数据中包含了足够的信息，可以正确做出转发决策。

但是这个决策是如何做出的呢？这就是```table```结构的目的。```table```定义包含一个要查找的```key```、一组可能的```actions```(```ipv4_forward```、```drop```、```NoAction```)、表的大小(```1024```个条目)以及当没有匹配时所采取的默认操作(```drop```)。关键规范包括要查找的报头字段(IPv4报头的```dstAddr```字段)和期望的匹配类型(```lpm```表示最长前缀匹配)。其他可能的匹配类型包括```exact```(精确匹配)和```ternary```(三元匹配)，后者有效应用掩码来选择在匹配中考虑的键中的哪些位域。```lpm```、```exact```和```ternary```是核心P4语言类型的一部分，其定义可以在```core.p4```中找到。P4体系架构可以扩展额外的匹配类型，例如PSA还定义了```range```(范围)和```selector```(选择器)匹配。

入口处理的最后一步是"应用"刚刚定义的表(只有当解析器或之前的流水线阶段将IP头标记为有效时才会这样做)。

```c
/****************************************************
******  I N G R E S S   P R O C E S S I N G   *******
****************************************************/

control MyIngress(
	inout headers hdr,
	inout metadata meta,
	inout standard_metadata_t standard_metadata) {			

    action drop() {
        mark_to_drop(standard_metadata);
    }
    
    action ipv4_forward(macAddr_t dstAddr,
			egressSpec_t port) {
	standard_metadata.egress_spec = port;
        hdr.ethernet.srcAddr = hdr.ethernet.dstAddr;
        hdr.ethernet.dstAddr = dstAddr;
        hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
    }
    
    table ipv4_lpm {
        key = {
            hdr.ipv4.dstAddr: lpm;
        }
        actions = {
            ipv4_forward;
            drop;
            NoAction;
        }
        size = 1024;
       default_action = drop();
    }
   
    apply {
        if (hdr.ipv4.isValid()) {
            ipv4_lpm.apply();
        }
   }
}
```

##### 4.4.4 出口处理(Egress Processing)

在我们的简单示例中出口处理没有任何操作，但通常这是基于出口端口执行操作的机会，而这一信息在入口处理期间可能不知道(例如，可能依赖于流量管理器)。例如，可以通过在入口处理中设置相应的内部元数据将一个数据包复制到多个出口端口以进行多播，这种元数据的含义由体系架构定义。出口处理将看到与流量管理器生成的相同包的副本数量相同的副本。再举个例子，如果期望交换机的一个端口发送带VLAN标签的报文，则必须用VLAN id扩展报文头。处理这种情况的一种简单方法是创建一个与元数据的```egress_port```匹配的表。其他例子包括对组播/广播数据包进行入口端口修剪，并为传递到控制平面的拦截数据包添加特殊的“CPU头”。

```c
/****************************************************
*******  E G R E S S   P R O C E S S I N G   ********
****************************************************/

control MyEgress(
	inout headers hdr,
	inout metadata meta,
	inout standard_metadata_t standard_metadata) {
   
    apply {  }
}

/****************************************************
***   C H E C K S U M    C O M P U T A T I O N   ****
****************************************************/

control MyComputeChecksum(
	inout headers  hdr,
	inout metadata meta) {
   
     apply {
	update_checksum(
	    hdr.ipv4.isValid(),
              { hdr.ipv4.version,
	        hdr.ipv4.ihl,
                hdr.ipv4.diffserv,
                hdr.ipv4.totalLen,
                hdr.ipv4.identification,
                hdr.ipv4.flags,
                hdr.ipv4.fragOffset,
                hdr.ipv4.ttl,
                hdr.ipv4.protocol,
                hdr.ipv4.srcAddr,
                hdr.ipv4.dstAddr },
            hdr.ipv4.hdrChecksum,
            HashAlgorithm.csum16);
    }
}
```

##### 4.4.5 编码器(Deparser)

Deparser的处理通常比较直接。我们在包处理过程中可能更改了各种报头字段，现在有机会将更新了报头字段的报文发送出去。如果在流水线的某个阶段更改了报头，则需要记住在发包的时候带上对应的改动。只有那些被标记为有效的报头才会被重新序列化到包中。不需要对包的其余部分(即有效负载)做任何处理，因为默认情况下，在我们停止解析的数据之后的所有字节都将被包含在输出消息中。数据包怎样被发送的细节由架构指定。例如，TNA支持根据deparser的特殊元数据值的设置来截断有效负载。

```c
/****************************************************
*************  D E P A R S E R  *********************
****************************************************/

control MyDeparser(
	packet_out packet,
	in headers hdr) {
		
    apply {
        packet.emit(hdr.ethernet);
        packet.emit(hdr.ipv4);
    }
}
```

##### 4.4.6 交换机定义(Switch Definition)

最后，P4程序必须从整体上定义交换机的行为，如下示例由V1Switch包给出。这个包中的元素集是由```v1model.p4```，由对上面定义的所有其他例程的引用组成。

```c
/****************************************************
*************  S W I T C H  *************************
****************************************************/

V1Switch(
    MyParser(),
    MyVerifyChecksum(),
    MyIngress(),
    MyEgress(),
    MyComputeChecksum(),
    MyDeparser()
) main;
```

请记住，这是一个最小示例，但确实有助于说明P4程序的基本思想。本例中隐藏了控制平面用来向路由表注入数据的接口，```table ipv4_lpm```定义了这个表，但我们没有填充相应的值。在第5章讨论P4Runtime时，我们将解决控制平面将值填入表中的问题。

#### 4.5 固定功能流水线(Fixed-Function Pipelines)

我们现在回到固定功能转发流水线，目标是将它们置于更大的生态系统中。记住，固定功能交换芯片仍然主导着市场<sup>[5]</sup>，我们并不是要低估它们的价值，它们无疑将继续发挥作用，但它们确实少了一个自由度，即重新编程数据平面的能力，这有助于突出本章中介绍的所有组件之间的关系。

>[5] 固定功能流水线和可编程流水线之间的区别并不像本文所讨论的那样明确，因为固定功能流水线也支持配置。但参数化交换芯片和为交换芯片编程在本质上是不同的，只有后者能够引入新的功能。

##### 4.5.1 OF-DPA

我们从一个具体例子开始: Broadcom为其交换芯片提供的硬件抽象层--*OpenFlow数据平面抽象(OF-DPA, OpenFlow—Data Plane Abstraction)* 。OF-DPA定义了可用于将流规则配置到底层Broadcom ASIC中的API。从技术上讲，OpenFlow代理位于OF-DPA之上(实现了OpenFlow协议)，而Broadcom SDK位于OF-DPA之下(实现了基于底层芯片细节的专有接口)，但OF-DPA层提供了Tomahawk ASIC固定转发流水线的抽象表示。图24显示了最终的软件栈，其中OF-Agent和OF-DPA是开源的(OF-Agent对应一个名为Indigo的软件模块，最初由Big Switch编写)，而Broadcom SDK是专有的。图25描述了OF-DPA流水线的样子。

![图24. Tomahawk固定功能转发流水线软件栈。](https://files.mdnice.com/user/28815/429a74e8-ca79-4d60-91c8-dec370dfbf01.png)

![图25. 由OF-DPA定义的固定功能逻辑流水线。](https://files.mdnice.com/user/28815/30abddaf-ab79-42e2-9011-a5768b0c767d.png)

我们不会深入探究图25的细节，但是读者可以识别出几个知名协议。就我们的目的而言，有意义的是了解OF-DPA与可编程流水线的对应关系。在可编程场景下，直到我们实现一个类似```switch.p4```这样的程序，才能得到大致相当于OF-DPA的东西。也就是说，```v1model.p4```定义了可用的阶段(控制块)，但直到我们添加了```switch.p4```，才能将这些阶段的功能运行起来。

考虑到这种关系，我们可能希望在单个网络中综合使用可编程交换机和固定功能交换机，并运行公共SDN软件栈。这可以通过将两种类型的芯片隐藏在```v1model.p4```(或类似的)架构模型后面来实现，并让P4编译器输出各自SDK能够理解的后端代码。显然，这不适用于想要做一些Tomahawk芯片不支持的事情的P4程序，但适用于标准L2/L3交换行为。

##### 4.5.2 SAI

正如我们所看到的供应商定义和社区定义的体系架构模型(分别是TNA和V1Model)一样，也存在供应商定义和社区定义的固定功能逻辑流水线。前者是OF-DPA，而*交换机抽象接口(SAI, Switch Abstraction Interface)* 是后者的一个例子。因为SAI必须跨一系列交换机(以及转发流水线)工作，所以必须专注于所有供应商都认同的功能子集，也就是最小公约数上。

SAI包括一个配置接口和一个控制接口，其中后者与本节最为相关，因为它抽象了转发流水线。另一方面，研究另一个转发流水线并没有什么价值，所以我们建议感兴趣的读者参考SAI规范以获得更多细节。

>延伸阅读:\
>[SAI Pipeline Behavioral Model](https://github.com/opencomputeproject/SAI/blob/master/doc/behavioral%20model/pipeline_v6.pdf). Open Compute Project.

我们将在下一章讨论配置API。

#### 4.6 比较

关于逻辑流水线及其与P4程序的关系的讨论是微妙的，值得重述一遍。一方面，对物理流水线进行抽象表示有明显的价值，如图21中引入的一般概念。当以这种方式使用时，逻辑流水线就是引入硬件抽象层这种经过验证的想法的一个例子。在我们的案例中，有助于控制平面的可移植性。OF-DPA是Broadcom固定功能交换芯片硬件抽象层的特定示例。

另一方面，P4提供了一个编程模型，通过```v1model.p4```和```tna.p4```等架构向P4通用语言结构(例如，```control```、```table```、```parser```)添加细节。实际上，这些架构模型是通用转发设备基于语言的抽象，可以通过添加特定P4程序(如```switch.p4```)将其完全解析为逻辑流水线。P4体系架构模型没有定义匹配操作表的流水线，但定义了可以被P4开发人员用来定义流水线(无论是逻辑的还是物理的)的构建块(包括签名)。因此，从某种意义上说，P4架构相当于传统交换机SDK，如图26中五个并排的示例所示。

![图26. 5个示例流水线/SDK/ASIC堆栈。最左边的两个加上第四个，至今仍然存在；中间是假设的栈；最右边是一个未完成的栈。](https://files.mdnice.com/user/28815/2a11fd89-2e0d-4e5f-b7e1-912350156d68.png)

图26中的每个示例由三层组成: 交换芯片专用ASIC、用于编程ASIC的厂商特定SDK，以及转发流水线定义。通过提供编程接口，中间层的SDK有效抽象了底层硬件。它们要么是传统的(例如，在第二个和第四个例子中显示的Broadcom SDK)，要么是，正如刚才所指出的，逻辑上对应于P4架构模型与专用于ASIC的P4编译器。这五个示例的最顶层定义了一个逻辑流水线，随后可以通过OpenFlow或P4Runtime之类的接口来控制(没有显示)。这五个示例的不同在于流水线是由P4程序定义还是通过其他方法(例如，OF-DPA规范)定义。

请注意，只有那些在栈顶使用P4定义的逻辑流水线的配置(即第一、第三、第五个示例)可以使用P4Runtime进行控制。这是出于实用的原因，P4Runtime接口是使用下一章介绍的工具基于P4程序自动生成的。

最左边的两个例子现在已经存在了，分别代表了可编程和固定功能ASIC的规范层。中间的例子纯粹是假设，但说明了即使对于固定功能流水线，也可以定义基于P4的堆栈(并暗示使用P4Runtime进行控制)。第四个例子同样存在，即Broadcom ASIC如何遵从SAI定义的逻辑流水线。最后，最右边的例子将会在未来出现，SAI将会被扩展以支持P4的可编程性，并在多个ASIC上运行。
