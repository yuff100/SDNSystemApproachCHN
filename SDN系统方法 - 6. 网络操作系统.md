### 第6章 网络操作系统(Network OS)

现在，我们准备好从只有单一本地状态的独立交换机，转向由网络操作系统维护的全局、全网视图。思考NOS的最佳方式是认识到它和其他可水平伸缩的云应用程序一样，是由一组松散耦合的子系统(通常与微服务体系架构相关联)组成，包括一个可伸缩的高可用键/值存储。

本章以ONOS作为参考实现，介绍NOS的总体架构。其主要核心抽象是基于ONOS实现广泛的控制程序，并管理同样广泛的网络设备。本章还将讨论可伸缩性和高可用性这两个至关重要的问题。

#### 6.1 架构

我们以ONOS为模型，Network OS的架构如图30所示，由三个主要部分组成:

1. 北向接口(NBI)集合，应用程序通过这些接口保持网络状态(例如拓扑图穿透，拦截网络数据包)，也用来控制网络数据平面(例如，通过第三章介绍的FlowObjective API对流对象编程)。
2. 分布式核心，负责管理网络状态，并通知应用程序该状态的相关更改。核心内部是一个可伸缩的键/值存储，称为Atomix。
3. 由一组插件组成的南向接口(SBI)，包括共享协议库和特定于设备的驱动程序。

![图30. ONOS三层架构，托管一组控制应用。](https://files.mdnice.com/user/28815/370f9368-8173-4619-98a7-5ffd070539fd.png)

如图30所示，这是高度模块化的设计，对于特定部署可以配置包含所需的子模块。我们将在最后一节讨论模块化的确切形式(例如Karaf, Kubernetes)，同时会讨论可伸缩性问题。在此之前，重点是ONOS功能的组织形式。

在深入了解每一层细节之前，关于图30还有三件事需要注意。首先是NBI的广度。如果将ONOS视为操作系统，这就说得通了，所有对底层硬件的访问，无论是通过控制程序还是人工运维，都以ONOS作为中介。这意味着所有北向API组合在一起必须足以完成网络的配置、运维和控制。例如，北向接口包括gNMI和gNOI，分别用于配置和运维。同时也意味着NBI包括一个拓扑API，控制程序通过它来了解底层网络状态的变化(例如，端口是否可工作)，以及用于控制底层交换机的FlowObjective API。

说个题外话，虽然我们通常把运行在Network OS上的应用程序视为网络控制平面的实现，但实际上有各种各样的应用程序运行在ONOS上，通过其GUI，可以实现从网络状态监控到运维人员用来发布指令的传统CLI的一切。

基于ONOS的应用程序包含零接触管理平面，帮助网络配置新硬件，确保安装正确的软件、证书、配置参数和流水线定义。如图31所示，其引申结论是ONOS没有固定的NBI，在ONOS上可能有多层应用程序和服务，每一层都在其下方的应用程序和服务之上提供更多价值。无论是声明在ONOS中提供零接触配置还是基于ONOS之上提供零接触配置都是武断的，这引申出了ONOS与传统操作系统不同的一个重要方面: 没有系统调用等效接口来区分内核特权域和用户域之间的边界。换句话说，ONOS目前运行在单个信任域中。 

![图31. Zero-Touch Provisioning(ZTP)应用程序示例，为被安装的交换机提供"角色规范"，ONOS为交换机提供相应的配置。](https://files.mdnice.com/user/28815/fa1e04d2-8ab2-48d9-9ea4-6d5fd06f59a2.png)

关于图30要注意的第二点是，ONOS将控制程序希望施加在网络上的行为的抽象规范映射到需要与网络中每个交换机通信的具体指令上。应用程序可以从多种方式中选择影响网络运行的方式。一些应用程序使用高级*意图(Intents)*，是网络范围内的、与拓扑无关的编程结构。其他需要更细粒度控制的使用Flow Objectives，这是一种以设备为中心的编程构造。Flow Objectives很像流规则，只不过它们是独立于流水线的。应用程序使用它们来控制固定功能流水线和可编程流水线。如图32中突出显示的那样，ONOS被明确设计用来解决在面对各种各样的转发流水线时完成工作所面临的复杂性。 

![图32. ONOS管理着网络范围内行为的抽象规范映射到每个设备的指令集合。](https://files.mdnice.com/user/28815/95617886-91ce-4c42-a8e8-7ffaab67ba2c.png)

关于图30要注意的第三点是，信息通过ONOS同时"向下"和"向上"流动。我们很容易关注到应用程序通过ONOS NBI来控制网络，但南向插件也会将底层网络的信息传递给ONOS核心，包括拦截数据包、发现设备及其端口、报告链路质量等。ONOS核心和网络设备之间的交互是由一组适配器(例如，OpenFlow, P4Runtime)来处理的，这些适配器隐藏了与设备通信的细节，从而将ONOS核心和运行其上的应用程序与多样化的网络设备解耦，从而ONOS可被用于控制私有交换机、裸金属交换机、光设备和蜂窝基站。

#### 6.2 分布式核心(Distributed Core)

ONOS核心由许多子系统组成，每个子系统负责网络状态的特定方面(例如拓扑结构、主机跟踪、包拦截、流编程)。每个子系统维护自己的*服务抽象(service abstraction)*，其实现负责在整个集群中传播状态。

许多ONOS服务是通过分布式表(map)构建的，而分布式表又使用分布式键/值存储来实现。对于看过现代云服务设计的人来说，应该对这类存储很熟悉，它可以跨一组分布式服务器进行扩展，并实现一种共识算法来在发生故障时实现容错。ONOS中使用的具体算法是Raft，在Diego Ongaro和John Ousterhout的一篇论文中有很好的描述，下面提供的网站还有一个很好用的可视化工具。

>延伸阅读:\
>D. Ongaro and J. Ousterhout. [The Raft Consensus Algorithm](https://raft.github.io).

ONOS使用Atomix作为其存储，Atomix在核心Raft算法的基础上提供了一组丰富的编程原语，ONOS使用这些原语来管理分布式状态，并为控制程序提供提供了访问状态的简单接口。

##### 6.2.1 Atomix原语

前面将Atomix介绍为键/值存储，事实确实如此，但也可以将Atomix描述为构建分布式系统的通用工具，它是一个基于Java的系统，支持:

- 分布式数据结构，包括map、set、tree和counter。
- 分布式通信，包括直接消息传递和发布/订阅。
- 分布式协调，包括锁、leader选举和屏障(barrier)。
- 管理组成员。

例如，Atomix包含```AtomicMap```和```DistributedMap```原语，两者都用额外方法扩展了Java的```Map```实现。对于```AtomicMap```来说，该原语使用乐观锁执行原子更新，从而保证所有操作都是原子的(map中的每个值都有一个单调递增的版本号)。与此相反，```DistributedMap```原语支持最终一致性，而不是强一致性。这两个原语都支持对对应map的基于事件的变更通知，客户端可以通过在map上注册事件监听器来监听插入/更新/删除事件。

我们将在下一小节中看到，map是ONOS使用的主要原语。我们通过查看Atomix在ONOS中扮演的另一个角色来结束本节: 协调所有ONOS实例<sup>[1]</sup>，这种协调有两个方面。

>[1] 对于本文的讨论，我们假设ONOS被打包为一个整体，然后跨多个虚拟化实例进行扩展。另一种将ONOS的功能划分为可独立扩展的微服务的方法将在第6.5节中讨论。

首先，作为可水平伸缩的服务，在任何给定时间运行的ONOS实例数量取决于工作负载以及在出现故障时保证可用性所需的副本级别。Atomix*组成员(group membership)* 原语用于确定可用的实例集，从而可以检测已经启动的新实例和失败的现有实例。(请注意，ONOS实例集与Atomix实例集是不同的，两者都能独立扩展。本节和下一段将重点介绍ONOS实例。)

其次，每个实例的主要工作是监视和控制网络中物理交换机的一个子集。ONOS采用的方法是为每个交换机选择一个主实例，只有主实例向给定交换机发出(写入)控制指令。所有实例都能够监视(读取)交换机状态。然后，这些实例使用Atomix *leader-election*原语确定每个交换机的主实例。如果主实例失效，同样的原语被用来为交换机选择一个新的主实例，当新的交换机上线时，也可以遵循相同的流程。

##### 6.2.2 服务

ONOS通过定义一组构建在Atomix上的核心表(map)而构建，这些表又打包为一组服务，用于控制应用程序(和其他服务)。表和服务是观察同一事物的两种方式: 一种是键/值对的集合，另一种是应用程序和其他服务与这些键/值对交互的接口。图33描述了相应的层次关系，其中中间的三个组件(Topology、Link和Device)是ONOS服务的示例。

![图33. ONOS基于Atomix中对应的表(Map)提供了一组服务，如Topology、Device和Link服务。](https://files.mdnice.com/user/28815/98a02761-74c1-4767-8d80-e680be9d54e4.png)

注意，图33中的Topology Service没有关联的map，而是间接访问由Link和Device Services定义的map。Topology Service将产生的网络拓扑图缓存到内存中，为应用提供了一种低延迟、只读的方式来访问网络状态。Topology Service还计算图的生成树，以确保所有应用程序看到相同的广播树。

总的来说，ONOS定义了一个互联的服务图，图33只显示了一个子图。图34扩展了该视图，展示了ONOS核心的一些其他方面，不过这次将Atomix map简化显示为一些(但不是所有)服务的属性。

![图34. 在构建Path Service时涉及到的服务依赖图(有些有自己的键/值映射)。](https://files.mdnice.com/user/28815/23ec4879-0cd8-49cf-a21d-0d86336cfa94.png)

关于这个依赖关系图，有几点需要注意。首先，应用程序可以通过查询Path Service来了解主机对之间的端到端路径，它既依赖于Topology Service(跟踪网络图)，也依赖于Host Service(跟踪连接到网络的主机)。注意，箭头方向指明依赖关系，但是正如我们在图34中所示，信息是双向流动的。

其次，Host Service有一个北向接口和一个南向接口。Path Service通过北向接口读取与主机相关的信息，而主机位置提供程序(Host Location Provider)使用南向接口写入与主机相关信息。Host Service本身只不过是Atomix Map的一个包装器，存储关于主机的信息。我们将在6.4节中返回到*Provider* 抽象，但简单来说，它们是与底层网络设备交互的模块。

第三，Host Location Provider负责窥探网络流量，例如拦截ARP、NDP和DHCP数据包以了解连接到网络的主机，然后将其提供给Host Service。反过来，Host Location Provider依赖Packet Service来帮助拦截这些数据包。Packet Service为其他ONOS服务定义了一种设备无关的方式，用于指示底层交换机捕获选定的数据包并将其转发到控制平面。ONOS服务也可以通过Packet Service向数据平面注入报文。

最后，虽然图34中描述的服务图旨在发现网络拓扑，但在许多场景中，拓扑是固定并且已知的，当控制平面为特定拓扑量身定制时，这种情况经常发生，就像本书中讨论的叶脊拓扑一样。对于这种情况，拓扑服务从依赖关系图<sup>[2]</sup>中位于其上方的控制应用程序(或高级服务)接受配置指令。ONOS包括了这样一个配置服务，称为*Network Config*，如图35所示。Network Config反过来接受来自人工运维人员或自动协调器(例如图31中的示例ZTP控制程序)的配置指令。

>[2] Topology Service仍然需要从底层网络收集真实信息，以验证其是否与上面传递来的配置指令相匹配，并在有差异时通知Network Config Service。

![图35. Network Config Service支持配置程序和人工运维。](https://files.mdnice.com/user/28815/6a5568ee-5a1c-4e8f-b157-c65623470c73.png)

我们通过刚才的示例(图33、34和35)说明了如何通过组件构建ONOS的基本原理。为了完整起见，下面总结了最常用的ONOS服务:

- **主机(Host):** 记录连接到网络的终端系统(裸机或虚拟机)，由一个或多个主机发现应用程序提供数据，通常通过拦截ARP、NDP或DHCP报文实现。
- **设备(Device):** 记录包括端口在内的基础设施设备相关信息(交换机、ROADM等)。由一个或多个设备发现应用程序提供数据。
- **链路(Link):** 记录一对基础设备/端口之间的链路属性。由一个或多个链接发现应用程序提供数据(例如，通过发送和拦截LLDP数据包)。
- **拓扑(Topology):** 通过图抽象将网络表示为一个整体。它构建在设备和链路服务之上，并提供由基础设施设备为顶点以及基础设施链接为边组成的一致的图。当接收到有关设备和链路的事件时，网络拓扑以最终一致性的方式收敛图抽象。
- **Mastership:** 通过分布式共识算法(使用Atomix leader-election原语)选举ONOS集群中的主节点实例。在ONOS实例失效的情况下(例如，服务器断电)，确保为所有没有master的设备尽快选举新的master。
- **集群(Cluster):** 管理ONOS集群配置，提供Atomix集群节点以及所有对等ONOS节点的信息。Atomix节点形成了实际的集群，这是共识的基础，而ONOS节点实际上只是用于将控制逻辑和I/O扩展到网络设备的客户机。ONOS使用Atomix membership原语设置条目。
- **网络配置(Network Config):** 定义网络元信息，如设备及其端口、主机、链接等。提供关于网络的外部信息，以及告诉ONOS核心和应用程序应该如何管理网络。由调度器、ZTP控制程序或运维人员手动设置。
- **组件配置(Component Config):** 管理ONOS核心和应用中各种软件组件的配置参数。这些参数(例如，如何处理外部流规则、地址或DHCP服务器、轮询频率，等等)允许定制软件的行为，由运营商根据部署需要设置。
- **报文(Packet):**　允许核心业务和应用程序拦截报文，并将报文发送回网络。这是大多数主机和链路发现方法(如ARP、DHCP、LLDP)的基础。

因为这些服务提供了关于网络设备及其拓扑结构的信息，因此几乎每个应用程序都需要使用这些服务。然而，还有更多服务，包括那些允许应用程序使用不同构造和不同抽象级别对网络行为进行编程的服务。我们将在下一节更深入讨论，但现在，我们先列举其中的一部分:

- **路由(Route):** 定义到下一跳映射的前缀。可通过控制程序设置，也可由运维人员手动配置。
- **多播(Mcast):** 定义组IP、源和汇聚位置。由控制程序设置或由运维人员手动配置。
- **组(Group):** 聚合设备中的端口或动作。流条目可以指向一个已定义的组，从而允许复杂的转发方式，例如组内端口之间的负载均衡、组内端口之间的故障转移或组内指定的所有端口的多播。组还可以用于聚合不同流的通用操作，因此在某些场景中，只需要修改所引用的流条目，而不必修改所有条目。
- **测量(Meter):** 表示对设备处理的特定网络流量强制执行服务质量的速率限制。
- **流规则(Flow Rule):** 提供以设备为中心的match/action对，用于对设备的数据平面转发行为进行编程。要求流规则条目按照设备的流水线结构和功能进行组合。
- **Flow Objective:** 提供一个以设备为中心的抽象，以流水线无关的方式对设备的转发行为进行编程。它依赖于Pipeliner子系统(请参阅下一节)来实现表无关的flow objective和表特定的流(或组)规则(flow rule)的映射。
- **意图(Intent):** 提供一种拓扑无关的方式建立跨网络的流。高级规范(既意图)指出端到端路径的各种提示和约束，包括流量类型以及源和目标主机，或请求连接的入口和出口端口。该服务通过适当的路径提供这种连通性，然后持续监测网络，在面对变化的网络条件时，随着时间的推移改变路径，从而持续满足意图定义的目标。

上述每个服务都含有自己的分布式存储和通知功能，各个应用可以自由使用自己的服务扩展这一集合，并用自己的分布式存储支持自己的实现。这就是ONOS为应用程序提供直接访问Atomix原语(如```AtomicMaps```和```DistributedMaps```)的原因。当我们在下一章仔细研究SD-Fabric时，将看到这种扩展的例子。

#### 6.3 北向接口

ONOS北向接口由多个部分组成。首先，在ONOS特定配置中包含的每个服务都有相应的API。例如，图30所示的"Topology"接口正是图33所示的Topology Service所提供的API。其次，由于ONOS允许应用程序定义和使用自己的Atomix表，因此可以将Atomix编程接口视为ONOS NBI的一部分。第三，ONOS NBI包括gNMI和gNOI，这些是独立于ONOS的标准化接口，但作为ONOS NBI的一部分。注意，gNMI和gNOI后面的实现也是封装在Atomix映射上的ONOS服务。最后，也是最有趣的一点是，ONOS提供了一组用于控制底层交换机的接口。图30描述了其中两个，Flow Rule和Flow Objective。前一种借鉴自OpenFlow，跟特定流水线有关。第二个是流水线中立的，也是本节其余部分的重点。 

有三种类型的流对象: *过滤(Filtering)*、*转发(Forwarding)* 和*下一步(Next)*。Filtering对象根据流量*选择器(Selector)* 确定是否允许流量进入流水线。Forwarding对象通常通过匹配数据包中的选择字段与转发表决定哪些流量被允许流出流水线。Next对象指出应该对流量做出什么样的*处理(Treatment)*，例如如何重写报头。如果你觉得这听起来像是一个抽象的三阶段流水线:

<center><b>Filtering → Forwarding → Next</b></center>

你就会理解流对象背后的理念。例如，Filter对象(阶段)可以指定匹配特定MAC地址、VLAN标签和IP地址的数据包被允许进入管道；相应的Forwarding对象(阶段)在路由表中查找IP地址；最后Next对象(阶段)根据需要重写报头，并将数据包转发给输出端口。当然，所有这三个阶段都不知道底层交换机到底使用了哪些表组合来实现相应的match/action对序列。 

挑战在于将这些流水线无关的对象映射到相应的流水线依赖规则上。在ONOS中，此映射由Flow Objective Service管理，如图36所示。为了简单起见，该示例主要关注Filtering对象指定的选择器(match)，关键是表示希望选择的特定输入端口、MAC地址、VLAN标记和IP地址组合，而不考虑实现该组合的流水线表的确切序列。

![图36. Flow Objective Service管理流水线无关的对象到流水线特定规则的映射。](https://files.mdnice.com/user/28815/8e4503a0-bbbc-46da-aadd-825492272ddc.png)

在内部，Flow Objective Service被组织为一组特定设备处理程序的集合，每个处理程序都使用ONOS设备驱动机制实现。将流对象指令映射到具体流规则操作实现的设备驱动行为被称为*Pipeliner*。图36显示了两个示例交换机流水线的Pipeliners。

Pipeliner能够将流对象映射到流规则(在固定功能流水线的情况下)和P4编程流水线上。图36中给出的示例显示了前一种情况，包括到OpenFlow 1.3的映射。在后一种情况下，Pipeliner利用了*Pipeconf* 结构，该结构维护以下元素之间的关联:

1. 每个目标交换机的流水线模型。
2. 需要在特定交换机上部署流指令的驱动程序。
3. 用于将流对象映射到特定目标的转换器。

如第5.2节所述，Pipeconf基于P4编译器输出的```.p4info```文件中提取的信息来维护这些绑定。

如今，(1)中所说的"模型"是ONOS定义的，意味着对于开发人员来说，端到端的工作流包括在编程数据平面时使用P4架构模型(例如```v1model.p4```)，以及在使用流对象编程控制平面时使用ONOS模型。最终，P4很可能将会统一这些流水线模型的不同层。

站在编程角度，流对象是用相关的构造函数例程打包的数据结构。控制程序构建一个对象列表，并传递给ONOS执行。下面的代码示例显示了构建流对象以指定通过网络的端到端流，将其应用到底层设备的过程在其他地方完成，没有包括在示例中。

```cpp
public void createFlow(
      TrafficSelector originalSelector,
      TrafficTreatment originalTreatment,
      ConnectPoint ingress, ConnectPoint egress,
      int priority, boolean applyTreatment,
      List<Objective> objectives,
      List<DeviceId> devices) {
   TrafficSelector selector = DefaultTrafficSelector.builder(originalSelector)
      .matchInPort(ingress.port())
      .build();

   // Optionally apply the specified treatment
   TrafficTreatment.Builder treatmentBuilder;
   if (applyTreatment) {
      treatmentBuilder = DefaultTrafficTreatment.builder(originalTreatment);
   } else {
      treatmentBuilder =
      DefaultTrafficTreatment.builder();
   }

   objectives.add(DefaultNextObjective.builder()
      .withId(flowObjectiveService.allocateNextId())
      .addTreatment(treatmentBuilder.setOutput(
	  egress.port()).build())
      .withType(NextObjective.Type.SIMPLE)
      .fromApp(appId)
      .makePermanent()
      .add());
   devices.add(ingress.deviceId());

   objectives.add(DefaultForwardingObjective.builder()
      .withSelector(selector)
      .nextStep(nextObjective.id())
      .withPriority(priority)
      .fromApp(appId)
      .makePermanent()
      .withFlag(ForwardingObjective.Flag.SPECIFIC)
      .add());
   devices.add(ingress.deviceId());
}
```

上面的例子创建了一个Next对象和一个Forwarding对象，Next对象对流应用了一个Treatment，改Treatment会设置输出端口，但也可以选择将```originalTreatment```作为```createFlow```的输入参数。

#### 6.4 南向接口

ONOS灵活性的一个关键点是能够适应不同的控制协议。虽然控制交互和相关抽象的本质是受到OpenFlow协议的启发，但ONOS的设计是为了确保核心(以及在核心之上编写的应用程序)与控制协议的细节解耦。

本节将详细介绍ONOS如何适应多种协议和异构网络设备，基本方法是基于插件架构，有两种类型的插件: *协议提供程序(Protocol Providers)* 和*设备驱动程序(Device Drivers)*，下面章节将依次对展开描述。

##### 6.4.1 Provider插件

ONOS定义了南向接口(SBI)插件框架，其中每个插件定义了南向(面向网络)API，每个插件也被称为*协议提供者(Protocol Provider)*，充当SBI和底层网络之间的代理，在底层网络中，没有限制每个插件可以使用什么控制协议与网络通信。Provider将自己注册到SBI插件框架中，并开始充当ONOS应用程序和核心服务(上层)以及网络环境(下层)之间传递信息和控制指令的通道，如图37所示。

![图37. 通过Provider插件扩展ONOS南向接口(SBI)。](https://files.mdnice.com/user/28815/c473780a-09a3-43aa-ab0a-073b7d8c9ae5.png)

图37包含了两种常见的Provider插件。第一种类型是特定于协议的，典型例子是OpenFlow和gNMI。这些Provider中的每一个都将API与实现相应协议的代码有效捆绑在一起。第二种类型(图中显示的是*DeviceProvider*、*HostProvider*和*LinkProvider*)使用一些其他ONOS服务与环境间接交互。我们在6.2.2节看到了一个例子，Host Location Provider(一个ONOS服务)位于*HostProvider*(一个SBI插件)之后，后者定义了主机发现API，而前者定义了一种发现主机的特定方法(例如，使用包服务拦截ARP、NDP和DHCP报文)。同样，LLDP Link Provider Service(对应于*LinkProvider* SBI插件)通过Packet Service拦截LLDP和BDDP报文，用于推测基础设施设备之间的链路。

##### 6.4.2 设备驱动(Device Drivers)

除了将核心与协议细节隔离开来之外，SBI框架还支持设备驱动插件作为一种机制，将代码(包括Provider)与特定设备解耦。设备驱动是一组模块的集合，每个模块都实现控制或配置功能的一部分。与Protocol Provider一样，对设备驱动如何实现这些功能没有任何限制。设备驱动程序也被部署为ONOS应用程序，从而允许被动态安装和卸载，允许运维人员动态引入新的设备类型和型号。 

#### 6.5 可扩展性能(Scalable Performance)

ONOS是一个逻辑上集中的SDN控制器，必须确保能够及时响应可伸缩规模的控制事件，还必须在失效时保持可用。本节描述ONOS如何扩展以满足这些性能和可用性需求。我们从一些规模和性能数据出发，从而介绍集中式网络控制最先进的进展(在撰写本文时):

- **规模(Scale):** ONOS最多支持50个网络设备、5000网络端口、5万用户、1百万线路、5百万流量规则/组/测量器。
- **性能(Performance):** ONOS支持每天高达10k的配置操作、500k流操作/秒(持续)、1k拓扑事件/秒(峰值)、50ms内检测端口/切换事件、5ms内检测端口/交换机关闭事件、流操作探测在3ms内完成、切换事件(RAN)在6毫秒内完成。

生产部署至少运行三个ONOS实例，但这更多是为了可用性而不是性能。每个实例都运行在一个32核/128GB内存的服务器上，并以Docker容器部署在Kubernetes上。每个实例捆绑了一个相同的(但可配置的)核心服务、控制应用程序和Protocol Provider的集合，ONOS使用Karaf作为其内部模块化框架。该组件包还包括Atomix，ONOS还支持其他键值存储的选择，可以独立于ONOS的其他部分扩展。

![图38. 多个ONOS实例通过Atomix共享网络状态，提供可扩展性能和高可用性。](https://files.mdnice.com/user/28815/29f78a33-5e90-4024-82ec-ab074a23c26e.png)

图38展示了ONOS跨多个实例的可伸缩性，其中实例集通过Atomix Maps共享网络状态。该图还显示，每个实例负责底层硬件交换机的一个子集。如果给定实例失效，其余实例使用Atomix leader-election原语选择一个新实例来取代它，从而确保高可用性。

ONOS的重构也在进行中，以更紧密遵循微服务架构。新版本称为μONOS，利用了ONOS现有的模块化，但独立封装和扩展了不同的子系统。虽然原则上，本章介绍的每个核心服务都可以打包为独立的微服务，但这样做太细粒度了，且不切实际。相反，µONOS采用以下方法。首先，它将Atomix封装在自己的微服务中。其次，它将每个控制应用程序和南向适配器作为单独的微服务运行。第三，它将核心划分为四个不同的微服务: (1)提供网络图(Network Graph)API的*Topology Management*微服务; (2)提供P4Runtime API的*Control Management*微服务; (3)提供gNMI API的*Configuration Management*微服务; (4)提供gNOI API的*Operations Management*微服务。
