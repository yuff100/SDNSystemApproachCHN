### 第7章 叶棘网络(Leaf-Spine Fabric)

本章介绍了由一组控制程序实现的叶脊交换结构。我们使用运行在ONOS上的SD-Fabric作为示例实现，在之前章节中我们已经介绍过了SD-Fabric的很多方面，所以在进入细节之前先总结一下这些内容。

- SD-Fabric支持叶脊拓扑，这种拓扑通常用于连接数据中心中的多个服务器机架(参见图10)，但也支持多站点部署(参见图17)。SD-Fabric基于裸金属交换机，并配备前几章介绍的软件来构建网络，可以混合使用固定功能流水线和可编程流水线，但在一般在生产中使用前者。
- SD-Fabric支持大部分L2/L3特性，所有这些特性都作为SDN控制程序重新实现(除了用于中继DHCP请求的DHCP服务器和用于与外部对等实体交换BGP路由的Quagga BGP服务器)。SD-Fabric实现每个服务器机架内部的L2连接，以及机架之间的L3连接。
- SD-Fabric支持接入/边缘网络技术，如PON(见图13)和RAN(见图17)，包括支持(a)路由IP流量到/从连接到这些接入网络的设备，以及(b)将接入网功能卸载到交换机上。

本章不会对所有这些特性进行全面介绍，重点关注的是数据中心架构用例，这足以说明使用SDN原则构建生产级网络的方法。关于SD-Fabric设计决策的更多信息，请访问SD-Fabric网站。

>延伸阅读:\
>[SD-Fabric](https://opennetworking.org/sd-fabric). Open Networking Foundation, 2021.

#### 7.1 特性集

SDN提供了定制网络的机会，但出于实用原因，采用SDN的第一个要求是重新实现现有功能，并以复制(或改进)传统解决方案的弹性和可伸缩性的方式来实现。SD-Fabric满足了这一要求，我们在这里总结一下。

首先，关于L2连接，SD-Fabric支持VLAN，包括基于VLAN id转发流量的原生支持，以及基于外部/内部VLAN id对的Q-in-Q支持。对Q-in-Q的支持与访问网络特别相关，其中使用双标记来隔离属于不同服务类别的流量。此外，SD-Fabric支持跨L3网络的L2隧道(包括单标签和双标签)。

其次，关于L3连接，SD-Fabric支持单播和组播地址的IPv4和IPv6路由。对于后者，SD-Fabric实现了集中式组播树构造(与运行像PIM这样的协议相反)，包含了对希望加入/离开组播组的终端主机的IGMP支持。SD-Fabric还支持ARP(用于IPv4地址转换)和NDP(用于IPv6邻居发现)，同时支持DHCPv4和DHCPv6。

然后，SD-Fabric在面对链路或交换机故障时提供高可用性。它通过组合众所周知的技术来实现这一点: dual-homing、链路绑定和ECMP链路组。 如图39所示，SD-Fabric集群中的每台服务器都连接到一对ToR(或叶子)交换机上，运行在每个计算服务器上的操作系统实现active-active链路绑定。然后，每个叶交换机由一对连接到两个或更多脊交换机的链接连接起来，ECMP组定义了将每个或每组叶交换机连接到给定脊交换机的链接对或者链接组。集群作为一个整体有多个到外部路由的连接，如图中的叶交换机3和4所示。图39中没有显示的事实是，SD-Fabric运行在ONOS之上，ONOS本身是为了可用性而构建的。在如下所示的配置中，ONOS(以及SD-Fabric控制应用程序)在3到5个服务器上都有副本。

![图39. 通过dual-homing、链路绑定和ECMP组的组合实现高可用性。](https://files.mdnice.com/user/28815/ff98ae3e-74a3-4561-88a8-b56232971eb6.png)

链路聚合和ECMP的用处很明显，可以增强包转发机制，在一组(如一对)链路(出口端口)之间实现负载均衡，而不只有一个"最佳"出口链路(出口端口)。这既提高了带宽，又在发生单个链路故障时可以自动恢复。另一种情况是交换机转发流水线支持端口组，因此一旦建立了等价链接，就可以一直为数据平面服务。

明确的说，ECMP是SD-Fabric在网络中的所有交换机上统一应用的一种转发策略。SD-Fabric控制程序感知拓扑，并相应的将端口组推送到每个网络交换机上，然后交换机将这些端口组应用到其转发流水线中，流水线通过端口组转发数据包，不需要控制平面的额外参与。

最后，在可扩展性方面，SD-Fabric拥有支持多达120k路由和250k流表的能力。该配置包括两个脊交换机和八个叶交换机，后者意味着支持多达四个机架的服务器。与可用性一样，SD-Fabric的可扩展性直接得益于ONOS的扩展能力。

#### 7.2 段路由(Segment Routing)

上一节重点介绍了SD-Fabric的功能，本节主要讨论如何才能做到这一点。SD-Fabric的核心策略基于*段路由(Segment Routing, SR)*。"segment routing"一词来源于这样一种想法: 任意一对主机之间的端到端路径可以由一组段序列构建，即使用标签交换沿着一段端到端路径遍历一组段序列。Segment routing是一种通用源路由方法，可以通过多种方式实现。对于SD-Fabric，segment routing利用了*MPLS(Multi-Protocol Label Switching)* 转发平面，细节可以参考网络信息。 

>延伸阅读:\
>[Multi-Protocol Label Switching](https://book.systemsapproach.org/scaling/mpls.html). Computer Networks: A Systems Approach, 2020.

当应用于叶棘网络时，总是涉及两个部分: 从叶到棘和从棘到叶。SD-Fabric为交换机编写程序，以匹配有标签或无标签数据包，并根据需要插入或弹出MPLS标签。图40说明了SR如何在SD-Fabric中工作，通过简单的配置在一对主机(10.0.1.1和10.0.2.1)之间转发流量。本例中，与Leaf 1相连的服务器在10.0.1/24子网中，与Leaf 2相连的服务器在10.0.2/24子网中，每个交换机都分配了一个MPLS id: 101、103、102和104。

![图40. 用于在一对主机之间转发流量的Segment Routing示例。](https://files.mdnice.com/user/28815/2175e4e9-8d2b-42da-93ba-983927556075.png)

当主机1发送目的地址为10.0.2.1的数据包时，默认情况下被转发到服务器的ToR/叶交换机。Leaf 1匹配目的IP地址，了解到这个包需要穿过网络并在Leaf 2出现以到达子网10.0.2/24，因此将MPLS标签102压到包上。因为有了ECMP, Leaf 1可以将产生的数据包转发到任意一个脊交换机，此时交换机匹配MPLS标签102，弹出标签头，并将其转发到Leaf 2。最后，Leaf 2匹配目的IP地址并将数据包转发给主机2。

从该示例中可以了解到，SR是高度格式化的。对于给定的叶脊交换机组合，SD-Fabric首先分配所有标识符，每个机架配置为共享一个IP前缀并在同一个VLAN上。SD-Fabric预计算可能的路径，并在底层交换机中配置相应的match/action规则。ECMP负责处理多路径负载均衡的复杂性，它同样不感知任何端到端路径。从实现角度来看，实现SR的SD-Fabric控制程序将这些match/action规则传给ONOS, ONOS反过来将它们配置在底层交换机上。SD-Fabric还维护自己的Atomix map，以管理连接叶脊交换机的ECMP组。

#### 7.3 路由和多播(Routes and Multicast)

除了在叶交换机之间建立数据路径的Segment Routing之外，SD-Fabric还利用了第6章介绍的Route和Mcast服务，它们决定每个IP前缀都有哪些叶脊交换机为其服务，以及在哪里能找到连接到每个多播组的所有主机。

SD-Fabric不通过类似OSPF或者PIM这样的分布式协议来学习路由以及构建组播树。相反，它根据全局信息计算出正确答案，然后将映射推送到Route和Mcast服务。因为SD-Fabric强加了简化约束，即每个机架恰好对应一个IP子网，因此这么做很简单。

为了更具体讨论问题，考虑到在第6章中描述的所有ONOS服务都可以通过RESTful API调用，或者通过封装了REST的```GET```、```POST```和```DELETE```调用的CLI调用。下面通过CLI来举例说明(因为更容易理解)，可以通过查询Route Service了解现有路由，如下所示:

```shell
onos> routes

B: Best route, R: Resolved route

Table: ipv4
B R  Network            Next Hop        Source (Node)
     0.0.0.0/0          172.16.0.1      FPM (127.0.0.1)
> *  1.1.0.0/18         10.0.1.20       STATIC
> *  10.0.99.0/24       10.0.1.1        FPM (127.0.0.1)
  *  10.0.99.0/24       10.0.6.1        FPM (127.0.0.1)
   Total: 2

Table: ipv6
B R  Network                   Next Hop                  Source (Node)
> *  2000::7700/120            fe80::288:ff:fe00:1       FPM (127.0.0.1)
> *  2000::8800/120            fe80::288:ff:fe00:2       FPM (127.0.0.1)
> *  2000::9900/120            fe80::288:ff:fe00:1       FPM (127.0.0.1)
  *  2000::9900/120            fe80::288:ff:fe00:2       FPM (127.0.0.1)
   Total: 3
```

可以给Route Service添加一个类似的静态路由:

```shell
onos> route-add <prefix> <nexthop>
onos> route-add 1.1.0.0/18 10.0.1.20 
onos> route-add 2020::101/120 2000::1
```

需要注意一点，路由有两种可能的源，一种是```STATIC```，意味着SD-Fabric在插入路由时完全知道它分配给集群中每个机架的前缀。(运维人员也可以使用CLI添加```STATIC```路由，但这只是一个例外，而不是规则。)

第二种源是```FPM```。FPM(转发平面管理器, Forwarding Plane Manager)是另一个ONOS服务，属于SD-Fabric服务套件之一。它负责从外部源学习路由，输入给被配置为BGP邻区的本地Quagga进程。每当FPM学习到外部路由，就会向Route Service添加相应的基于前缀的下一跳映射，表明通过连接网络和上游的叶交换机(例如图39中的交换机3和4)，目的地前缀是可达的。

多播的情况与此类似，使用同样的ONOS CLI可以创建新的多播路由并添加聚合器。例如:

```shell
onos> mcast-host-join -sAddr *
    -gAddr 224.0.0.1
    -srcs 00:AA:00:00:00:01/None
    -srcs 00:AA:00:00:00:05/None
    -sinks 00:AA:00:00:00:03/None
    -sinks 00:CC:00:00:00:01/None
```

该示例指定*ASM (Any-Source Multicast)* (```sAddr *```)、组播组地址(```gAddr```)、组源地址(```srcs```)和组聚合地址(```sinks```)。聚合地址可以通过如下指令移除:

```shell
onos> mcast-sink-delete -sAddr *
    -gAddr 224.0.0.1
    -h  00:AA:00:00:00:03/None
```

这里同样没有运行PIM，但是SD-Fabric为网络运维人员提供了编程接口，通过一系列这样的调用来定义多播树。例如，当SD-Fabric作为向订户发送IPTV的接入网络的一部分运行时，一种选择是让运营商机顶盒发出类似上面所示的调用(当然，使用RESTful API而不是CLI)。另一种选择是让机顶盒发送IGMP消息，SD-Fabric通过Packet Service(类似于拦截ARP和DHCP报文的Host Service)拦截IGMP消息。因此，当你下一次使用电视遥控器转换频道时，可能就在本书介绍的SDN软件栈上触发了过程调用! 

#### 7.4 定制转发(Customized Forwarding)

SD-Fabric是SDN的典型用例，它是一组运行在网络操作系统(Network OS)上控制程序，这些控制程序又运行在一组可编程交换机上，这些交换机以叶脊拓扑的架构连接，每个交换机运行本地交换机操作系统。通过这种方式，SD-Fabric位于自下而上的SDN软件栈的顶点。

但如果我们从一开始就知道支持SD-Fabric特性集的叶脊网络正是我们想要的，那么可能会回到较低的层级，并以此为目的进行调整。随着时间的推移，这就是SD-Fabric所发生的情况，产生了一个名为```fabric.p4```的P4程序实现了定制转发平面。我们通过对```fabric.p4```的概要总结来结束本章，重点介绍其设计如何与软件栈的其他部分相结合。

在此之前，必须认识到，一开始就清楚知道想从网络中获得什么是非常高的标准。网络的发展基于使用和运维的经验，没人从一开始就知道如何编写```fabric.p4```，但在整个软件栈的其他层实现了一系列迭代之后(包括引入Tofino作为可编程转发流水线)，```fabric.p4```出现了。*关键点是，可编程平台使我们能够不断、快速的实现网络的演进*。

换句话说，我们在第4章中提到的```forard.p4```是"按照我们想要的方式定制转发平面"的典型例子，但该章剩余部分都在讨论无需重新实现特定网络的功能使```forward.p4```成为可能的所有机制。简言之，```fabric.p4```是```forward.p4```的一个具体例子，不过我们现在只介绍了它与控制平面的关系。

关于```fabric.p4```有三件事情值得注意。首先，虽然它基于Broadcom OF-DPA流水线，但是松耦合的。这很有意义，因为SD-Fabric最初是在一组基于Tomahawk的交换机上实现的。```fabric.p4```流水线比OF-DPA更简单，消除了SD-Fabric不需要的表，从而使得```fabric.p4```更容易控制。

其次，```fabric.p4```被设计来模拟ONOS的FlowObjective API，从而简化了将FlowObjective映射到P4Runtime的过程。图41显示了```fabric.p4```的入口流水线，虽然没有显示出口流水线，但在一般情况下，只是对头字段的简单重写。

![图41. 由```fabric.p4```支持的逻辑流水线，用于对FlowObjective API的Filtering, Forwarding和Next阶段的并行处理。](https://files.mdnice.com/user/28815/3b38e3f7-343c-4433-99a9-a196a5d2ac75.png)

最后，```fabric.p4```被设计为可配置的，从而可以有选择的包含额外的功能。在编写基于ASIC的转发流水线进行优化的代码时，这并不容易，而且在实践中会大量使用预处理条件(例如，```#ifdefs```)。下面显示的代码片段是```fabric.p4```的入口函数的主要控制块。第9章将在较高层次上更深入讨论下面这些可选扩展:

- **UPF(用户平面功能, User Plane Function):** 增强IP功能，支持4G/5G移动网络。
- **BNG(宽带网络网关, Broadband Network Gateway):** 增强IP功能，支持光纤到家。
- **INT(带内网络遥测, Inband Network Telemetry):** 增加度量收集和遥测输出指令。

```c
apply {
#ifdef UPF
    upf_normalizer.apply(hdr.gtpu.isValid(), hdr.gtpu_ipv4,
	hdr.gtpu_udp, hdr.ipv4, hdr.udp, hdr.inner_ipv4,
	hdr.inner_udp);
#endif // UPF

    // Filtering Objective
    pkt_io_ingress.apply(hdr, fabric_metadata, standard_metadata);
    filtering.apply(hdr, fabric_metadata, standard_metadata);
#ifdef UPF
    upf_ingress.apply(hdr.gtpu_ipv4, hdr.gtpu_udp, hdr.gtpu,
	hdr.ipv4, hdr.udp, fabric_metadata, standard_metadata);
#endif // UPF

    // Forwarding Objective
    if (fabric_metadata.skip_forwarding == _FALSE) {
        forwarding.apply(hdr, fabric_metadata, standard_metadata);
    }
    acl.apply(hdr, fabric_metadata, standard_metadata);

    // Next Objective
    if (fabric_metadata.skip_next == _FALSE) {
        next.apply(hdr, fabric_metadata, standard_metadata);
#if defined INT
        process_set_source_sink.apply(hdr, fabric_metadata,
	    standard_metadata);
#endif // INT
    }	
#ifdef BNG
    bng_ingress.apply(hdr, fabric_metadata, standard_metadata);
#endif // BNG
}
```

例如，```upf.p4```(未显示)实现了UPF扩展的转发平面，支持包括3GPP蜂窝网络标准要求的GTP隧道封装/解封装，将SD-Fabric网络连接到无线接入网络基站。同样，```bng.p4```(未显示)实现PPPoE终端，一些无源光网络部署通过它将SD-Fabric网络连接到家庭路由器。最后，这段没什么实际意义的代码片段介绍了```fabric.p4```核心功能的基本结构。首先应用*filtering对象*(```filtering.apply```)，然后应用*forwarding对象*(```forwarding.apply```和```acl.apply```)，最后应用*next对象*(```next.apply```)。 

除了选择包含哪些扩展之外，预处理器还定义了几个常量，包括每个逻辑表的大小。显然，这种实现是构建可配置转发流水线的底层方法。设计更高级的语言结构，包括在运行时向流水线动态添加功能的能力，还是一个未完成的研究课题。

>VNF卸载
>
>*UPF和BNG扩展是被称为VNF卸载的优化技术的例子。VNF是虚拟网络功能(Virtual Network Function)的缩写，指的是在虚拟机中作为软件运行的功能。卸载是指重新实现该功能，使其运行在交换机转发流水线中，而不是在通用服务器上。通过这种方式，数据包可以从源直接转发到目的地，而不必被服务器处理，因此通常会提供更好的性能。*
>
>*调用UPF和BNG等功能作为卸载"优化"可以说是选择性记忆的一个例子。准确的说，我们把IP转发能力放到了交换机上，而不是通过运行在通用处理器上的软件处理。简单来说，UPF和BNG是专门的IP路由器，分别提供蜂窝和有线接入网络的特定功能。从宏观上讲，网络是由转发功能的一组组合构建而成的，我们现在有了更多选择，可以选择实现每个转发功能的最合适的硬件芯片。*
