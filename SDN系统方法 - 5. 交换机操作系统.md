### 第5章 交换机操作系统(Switch OS)

本章将介绍运行在裸金属交换机上的操作系统，可以把它想象成类似于服务器操作系统: 有一个基于Linux的操作系统运行在通用处理器上，加上一个"包转发加速器"，其作用有点类似于GPU。

*ONL(Open Network Linux)* 是交换机操作系统的基金会，是开放计算项目(Open Compute Project)的一个开源项目。ONL从Linux Debian发行版开始，支持了包括如图18所示的*Small Form-factor Pluggable (SFP)* 接口模块等在内的交换机独有硬件扩展。

本章不讨论这些底层设备驱动细节，而是关注Switch OS导出到控制平面的*北向接口(NBI)*，该控制平面运行在交换机上(作为运行在Switch OS上用户空间程序)或在交换机外(作为ONOS这样的SDN控制器)。正如第三章所介绍的，我们将使用Stratum作为在ONL之上实现NBI软件层的具体示例。Stratum有时被称为*Thin Switch OS*，其中的关键词是"Thin"，因为它本质上是实现了一个API shim，其有趣之处在于支持的一组API，本章大部分内容都集中在这些API上。

#### 5.1 Thin Switch OS

本节将介绍裸金属交换机上Switch OS实现SDN北向接口的组件。详细信息来自于Stratum，这是ONF的一个开源项目，最初由谷歌贡献了产品级代码。图27给出了Stratum概述，再次强调，其公开接口(P4Runtime, gNMI和gNOI)是本章的重要内容。在本节中，我们将展示这些实现细节，作为开发人员实现基于SDN解决方案的端到端工作流的基础。

![图27. 运行在开放网络Linux上的Thin Switch OS Stratum的高层示意图。](https://files.mdnice.com/user/28815/250d2d61-6040-408a-b541-efc3b4ffee40.png)

Stratum导出三个主要的北向接口:(1)使用P4Runtime控制交换机转发行为;(2)使用gNMI对交换机进行配置;(3)使用gNOI访问交换机上的其他运维数据。所有这三个接口都是gRPC服务(未在图中显示)，这意味着有一组相应的*Protocol Buffers(protobufs)*，指定API方法以及每个方法支持的参数。关于gRPC和protobufs的教程超出了本书范围，可以在网上阅读关于这两者的简要介绍。

>延伸阅读:\
>[gRPC](https://book.systemsapproach.org/e2e/rpc.html#grpc). Computer Networks: A Systems Approach, 2020.\
>[Protocol Buffers](https://book.systemsapproach.org/data/presentation.html#protobufs). Computer Networks: A Systems Approach, 2020.

重要的是，通过使用protobufs和gRPC, Stratum不需要再为格式、可靠性、后向兼容性和安全问题操心了，在过去很多协议(包括OpenFlow)在这些事情上花费了大量时间。此外，P4编译器将protobufs作为其自动生成代码的目标，也就是说，P4工具链可以输出protobufs，这些protobufs指定了P4Runtime接口类型和参数。这些API以及对应的客户端和服务器端stub大部分都是自动生成的。第5.2节介绍了创建该运行时契约的工具链细节。

在Stratum底层利用了两个组件的能力。一是板载交换芯片SDK，由交换机供应商提供，如果是Broadcom，大致对应于第4.5节中描述的OF-DPA层，Barefoot也为Tofino芯片提供了类似的SDK。可以将这些SDK视为类似于传统操作系统中的设备驱动程序，用于间接读写相应芯片上的内存位置。第二个是*ONL平台(ONLP, ONL Platform)*，用于导出图27所示的平台API，提供对硬件计数器、监视器、状态变量等的访问。

通过简单的例子有助于说明固定功能流水线和可编程流水线之间的根本区别，Broadcom SDK定义了```bcm_l3_route_create```方法更新L3转发表，而Barefoot提供的流水线无关的对应方法是```bf_table_write```。

在Stratum内部，图27所示其他组件主要是为了使Stratum与供应商解耦。在使用Tofino这样的可编程芯片的情况下，Stratum在很大程度上是透传的: 来自上层的P4Runtime调用直接传递给Barefoot SDK。在使用Tomahawk这样的固定功能芯片的情况下，Stratum需要维护运行时状态，以便将P4Runtime调用转换为对应的Broadcom SDK。粗略的说，这意味着将```switch.p4```(4.5.1节)中的P4Runtime调用映射为Broadcom SDK调用。例如，```switch.p4```(4.5.1节)中更新表项的P4Runtime调用将被映射到Broadcom SDK调用来更新ASIC表中的条目。

#### 5.2 P4Runtime

可以将图27中所示的P4Runtime接口视为用于控制交换机的服务器端RPC stub，还有一个相应的客户端stub，包含在SDN控制器中。两者共同实现了控制器和交换机之间的*P4Runtime契约(P4Runtime Contract)*。生成该契约的工具链如图28所示，在前面的图中，我们将原始P4转发程序表示为一个抽象图，而不是实际的P4源代码。

![图28. P4工具链实现了ASIC无关的以及自动生成的P4Runtime契约(实现为Protocol Buffer规范)。](https://files.mdnice.com/user/28815/12c588b1-57f6-4c10-9a52-b2538073e87e.png)

从图28中得到的一个关键结论是，P4编译器生成加载到每个交换芯片的二进制文件和(通过Switch OS)用于控制交换芯片的*运行时接口*<sup>[1]</sup>。编译器借助特定供应商后端来完成这一任务，图28显示了两个可能的示例。请注意，这些特定供应商的后端必须通过特定体系架构模型(由```arch.p4```定义)编写。换句话说，是P4语言、专用ASIC后端和架构模型的组合，该架构模型定义了将功能注入到数据平面的编程环境。 

>[1] 当我们说将二进制代码加载到交换芯片时，采用的是通用处理器的成熟术语。实际过程是依赖于不同的ASIC，可能包括通过SDK初始化各种片上数据表等操作。

这一端到端故事的最后一部分是运行时契约与加载到数据平面的原始程序之间的连接。以第4.4节中介绍的简单转发程序为例，我们看到```forward.p4```定义了一个查找表，在这里重申一下:

```c
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
```

相应的，编译器输出的```forward.p4info```指定了P4Runtime契约。如下示例所示，其包含足够信息通知控制器和交换机如何格式化以及解释插入、读取、修改和删除表项所需的一组gRPC方法。例如，```table```定义了标识要匹配的字段(```hdr.ipv4.dstAddr```)和匹配类型(```LPM```)，以及三个可能的```actions```。

```c
actions {
    preamble {
        id: 16800567
        name: "NoAction"
        alias: "NoAction"
    }
}
actions {
    preamble {
        id: 16805608
        name: "MyIngress.drop"
        alias: "drop"
    }
}
actions {
    preamble {
        id: 16799317
        name: "MyIngress.ipv4_forward"
        alias: "ipv4_forward"
    }
    params {
        id: 1
        name: "dstAddr"
        bitwidth: 48
    }
    params {
        id: 2
        name: "port"
        bitwidth: 9
    }
}
tables {
    preamble {
        id: 33574068
        name: "MyIngress.ipv4_lpm"
        alias: "ipv4_lpm"
    }
    match_fields {
        id: 1
        name: "hdr.ipv4.dstAddr"
        bitwidth: 32
        match_type: LPM
    }
    action_refs {
        id: 16799317
    }
    action_refs {
        id: 16805608
    }
    action_refs {
        id: 16800567
    }
    size: 1024
}
```

gRPC工具链从那里开始接管，工具链必须知道哪些P4语言元素是可控制的，可以由```p4runtime.proto```"公开"。此类信息包含在```forward.p4info```，其精确指定了一组可控元素及其在P4源程序<sup>[2]</sup>中定义的属性。```table```元素是一个明显的例子，但还有其他元素，包括```counters```和```meters```，分别用于向控制器报告状态信息以及允许控制器指定QoS速率。但是，在我们的示例程序中没有包含这两个参数。 

>[2] 原则上，P4Info文件不是严格必需的，因为控制器和交换机可以使用P4源程序获得处理P4Runtime方法所需的所有信息。但是，P4Info通过从P4程序中提取相关信息，并以更结构化的protobuf定义的格式提供这些信息，使这一点变得容易得多，使用protobuf库可以直接解析这些信息。

最后，控制器实际上向该表写入了一个条目。通常控制器会运行在ONOS之上，并间接的与交换机交互。我们介绍一个更简单的例子，一个Python程序实现了控制器的功能，并直接将一个条目写入表中(通过P4Runtime库的帮助)。

```python
import p4runtime_lib.helper
	...
table_entry = p4info_helper.buildTableEntry(
    table_name="MyIngress.ipv4_lpm",
    match_fields={
        "hdr.ipv4.dstAddr": (dst_ip_addr, 32)
    },
    action_name="MyIngress.ipv4_forward",
    action_params={
        "dstAddr": next_hop_mac_addr,
        "port": outport,
    })
ingress_sw.WriteTableEntry(table_entry)
```

#### 5.3 gNMI和gNOI

配置和操作网络设备的核心挑战是定义运维人员可以在设备上进行```GET```和```SET```操作的变量集，另外还有一个要求，这个变量集字典应该在不同设备之间是统一的(即供应商中立)。互联网已经经过了长达数十年的实践来定义这样一个字典，从而产生了与SNMP结合使用的*管理信息库(MIB, Management Information Base)*。但是MIB更专注于读取设备状态变量，而不是写入设备配置变量，后者在历史上一直使用设备的*命令行接口(CLI, Command Line Interface)* 来完成。SDN演进的一个结果是推动行业向支持可编程配置API的方向发展，这意味着要重新审视网络设备的信息模型。 

从SNMP和MIB出现到现在，主要技术进步是实用建模语言的可用性，在这方面，YANG是过去几年出现的主流选择。YANG是*Yet Another Next Generation*的缩写，之所以选择这个名字，是为了取笑重新再定义一种数据格式的必要性。YANG可以被视为XSD的受限版本，XSD是一种定义XML模式的语言。YANG定义了数据结构，但与XSD不同，不是特定于xml的。相反，YANG可以与不同的网络消息格式结合使用，既包括XML，也可以与protobufs和JSON结合使用。如果不熟悉这些首字母缩写，或者对不同标记语言模式之间的区别很模糊，可以在网上查看一些简单的介绍。

>延伸阅读:\
>[Markup Languages (XML)](https://book.systemsapproach.org/data/presentation.html#markup-languages-xml). Computer Networks: A Systems Approach, 2020.

从这方面来说，重要的是以可编程的形式定义具有可读可写语义的变量数据模型，而不仅仅只是提供标准文档中的文本。此外，虽然所有硬件供应商确实都在推广产品的独特功能，但并不是每个供应商都定义了独特的模型。这是因为，购买网络硬件的网络运营商有强烈动机推动类似设备的模型走向融合，而供应商也有同样强烈的动机坚持自己的模型。YANG使创建、使用和修改模型的过程可编程，因此可以适应这个迭代过程。

这就是名为*OpenConfig*的全行业标准化工作发挥作用的地方。OpenConfig网络运营商组成的组织，试图推动行业使用以YANG为建模语言的通用配置模型。OpenConfig官方对于访问设备配置和状态变量协议是中立的，但gNMI(gRPC网络管理接口)是其正在积极推进的一种方法。从它名字就可以猜到，gNMI使用gRPC(运行在HTTP/2之上)，这意味着gNMI也采用protobufs作为实际通信数据的方式，gNMI旨在作为网络设备的标准管理接口。 

为了完整起见，请注意NETCONF是另一个后SNMP协议，可用于将配置信息传递给网络设备。OpenConfig与NETCONF也有合作，但我们目前的评估是，gNMI作为未来的管理协议在行业中占有重要地位。基于这个原因，在我们完整的SDN软件栈中会着重强调gNMI。

>**云计算最佳实践**
>
>我们对OpenConfig和NETCONF的比较基于SDN的基本原则，即将云计算最佳实践引入网络。这涉及到一些大的想法，比如将网络控制平面实现为可伸缩的云服务，但也包括一些更小的好处，比如使用像gRPC和protobufs这样的现代消息框架。
>
>这种特殊情况下的优势很明显:(1)使用HTTP/2和基于protobuf编码来改进和优化传输，而不是使用SSH加上手工编码;(2)二进制数据编码，而不是基于文本的编码;(3)基于增量的数据交换，而不是基于快照;(4)对服务器推送和客户端流的原生支持。

OpenConfig定义了对象类型的层次结构。例如，网络接口的YANG模型如下所示: 

```yang
Module: openconfig-interfaces
	+--rw interfaces	 							
		+--rw interface*   [name]
			+--rw name
			+--rw config
		 	 |   ...	
			+--ro state
			 |    ...	 							
			+--rw hold-time	
			 |    ...	 
			+--rw subinterfaces 							
		    	 |    ...
```

这是一个基础模型，基于此可以增强添加其他功能，例如，为以太网接口建模:

```yang
Module: openconfig-if-ethernet
	augment /ocif:interfaces/ocif:interface:
		+--rw ethernet
		+--rw config
		 |	+--rw mac-address?
		 |	+--rw auto-negotiate?
		 |	+--rw duplex-mode?
		 |	+--rw port-speed?
		 |	+--rw enable-flow-control? 
		+--ro state
			+--ro mac-address?
			+--ro auto-negotiate?
			+--ro duplex-mode?
			+--ro port-speed?
			+--ro enable-flow-control?
			+--ro hw-mac-address?
			+--ro counters
			       ...
```

可以定义其他类似扩展，以支持链路聚合、IP地址分配、VLAN标签等。

OpenConfig层次结构中的每个模型都定义为配置状态和运维状态的组合，配置状态可以被客户端读取和写入(在示例中表示为```rw```)，操作状态可以报告设备状态(在示例中表示为```ro```，表示它在客户端是只读的)。声明性配置状态和运行时反馈状态之间的区别是任何网络设备接口的基本内容，OpenConfig明确专注于将后者通用化，并包含运维人员需要跟踪的网络遥测数据。 

拥有一组有意义的模型是必要的，但是完整的配置系统也包括其他元素。在我们的案例中，关于Stratum和OpenConfig模型之间的关系有三个要点。

首先，Stratum依赖于YANG工具链。图29显示了将一组基于yang的OpenConfig模型转换为gNMI使用的客户端和服务器端gRPC stub所涉及的步骤。图中所示的gNMI服务器与图27所示的gNMI接口门户相同。工具链支持多种目标编程语言(Stratum正好使用C++)，其中gRPC客户端和服务器端不需要用同一种语言编写。

![图29. 用于为gNMI生成基于gRPC运行时的YANG工具链。](https://files.mdnice.com/user/28815/d46b921c-56c5-4dde-b8ec-b9f2eda1d9f2.png)

请记住，YANG既不和gRPC绑定，也不和gNMI绑定。工具链能够从完全相同的OpenConfig模型开始，为网络设备(例如，分别使用NETCONF或RESTCONF)读取或写入的数据产生XML或JSON表示。但在我们的环境中，目标格式是protobufs，这是Stratum用来支持在gRPC上运行gNMI的格式。

第二点是gNMI定义了一组特定的gRPC方法来操作这些模型。该集合在protobuf规范中被集体定义为一个Service:

```protobuf
Service gNMI {
    rpc Capabilities(CapabilityRequest)
        returns (CapabilityResponse);
    rpc Get(GetRequest) returns (GetResponse);
    rpc Set(SetRequest) returns (SetResponse);
    rpc Subscribe(stream SubscribeRequest)
        returns (stream SubscribeResponse);
}
```

```Capabilities```方法用于检索设备支持的模型定义集。```Get```和```Set```方法用于读写某些模型中定义的相应变量。```Subscribe```方法用于设置来自设备的遥测更新数据流。相应的参数和返回值(例如，```GetRequest```, ```GetResponse```)定义为protobuf ```Message```，包括YANG模型内的各种字段，通过在数据模型树中给出其完整路径名来指定给定字段。

第三点是，Stratum不一定关心OpenConfig模型的全部范围。这是因为作为一个支持集中式控制器的Switch OS，Stratum关心配置数据平面的各个方面，但通常不参与配置像BGP这样的控制平面协议。在基于SDN的解决方案中，这样的控制平面协议不再在交换机上实现(尽管仍然在网络操作系统的范围内，但实现了对应的集中式协议)。具体来说，Stratum跟踪以下OpenConfig模型: 接口、VLAN、QoS和LACP(链路聚合)，以及一组系统和平台变量(交换机的风扇转速是每个人最喜欢的例子)。

我们通过将注意力简单转向gNOI来结束本节，但并没有太多要说的，因为gNOI的底层机制与gNMI完全相同，而且在更大范围内，交换机配置接口和运维接口之间几乎没有区别。一般来说，持久化状态由gNMI处理(并定义了相应的YANG模型)，而清除或设置临时状态则由gNOI处理。另外，像重启和ping这样的非幂等操作也属于gNOI的范围。在任何情况下，两者都足够紧密对齐，可以统称为gNXI。 

作为一个说明gNOI用途的例子，下面是```System```服务的protobuf规范:

```protobuf
service System {
    rpc Ping(PingRequest)
	returns (stream PingResponse) {}
    rpc Traceroute(TracerouteRequest)
        returns (stream TracerouteResponse) {}
    rpc Time(TimeRequest)
	returns (TimeResponse) {}
    rpc SetPackage(stream SetPackageRequest)
        returns (SetPackageResponse) {}
    rpc Reboot(RebootRequest)
	returns (RebootResponse) {}
    // ...
}
```

例如，下面的protobuf消息定义了```RebootRequest```参数:

```protobuf
message RebootRequest {
     // COLD, POWERDOWN, HALT, WARM, NSF, ... 
    RebootMethod method = 1;
    // Delay in nanoseconds before issuing reboot.
    uint64 delay = 2; 
    // Informational reason for the reboot.
    string message = 3;
    // Optional sub-components to reboot.
    repeated types.Path subcomponents = 4;
    // Force reboot if sanity checks fail.
    bool force = 5; 
}
```

再次提醒，如果不熟悉protobufs，可以在网上找到简短概述。

>延伸阅读:\
>[Protocol Buffers](https://book.systemsapproach.org/data/presentation.html#protobufs). Computer Networks: A Systems Approach, 2020.

#### 5.4 SONiC

就像SAI是行业范围内的交换机抽象(见4.5节)一样，SONiC是一个与厂商无关的交换机操作系统，在行业中有很强的发展势头。它最初是由微软开源，并继续作为Azure Cloud的Switch OS。SONiC利用SAI作为供应商中立SDK，并包括一个交换机定制Linux发行版，也就是说，Stratum和SONiC试图填补同样的需求。今天，他们各自的方法在很大程度上是互补的，这两个开源社区正在努力实现"两全其美"的解决方案，这种努力的结果就是*PINS(P4 Integrated Network Stack)*。

>延伸阅读:\
>[PINS: P4 Integrated Network Stack](https://opennetworking.org/pins).

SONiC和Stratum都支持配置接口，因此将它们统一起来的主要问题是协调各自的数据模型和工具链。其中主要区别在于Stratum支持可编程转发流水线(包括P4和P4Runtime)，而SAI则采用最小公共性的转发方式。这两个开源项目的开发人员正在共同努力，以制定一个路线图，使感兴趣的网络能够以一种增量的、低风险的方式利用可编程流水线。

这项工作的目标是: (1)使远程SDN控制器/应用程序使用P4Runtime和gNMI与SAI交互，(2)使用P4扩展SAI，以提升数据平面特性引入速度。这两个目标都依赖SAI行为模型的新表示以及基于P4的流水线(所谓的```sai.p4```程序，如4.6节的图26所示)。如果说有什么事情可以加速这一协调工作，那么应该是采用可编程流水线(以及相应的工具链)可以促进这一工作。 
