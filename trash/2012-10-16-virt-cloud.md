---
title: 虚拟化、云计算、开放源代码及其他
layout: post
category: coding
tags: virtualization cloud compute vmware xen
---

特意转载一篇文章，并作标注，只因这篇文章契合我的想法——我早想做一个云计算领域概述，但因为了解有限而不得实行。  
以下是正文。  

**[虚拟化、云计算、开放源代码及其他](http://www.qyjohn.net/?p=1552) ——by qyjohn**  

借国庆长假的机会写了这篇长文，全面地整理了个人从虚拟化到云计算各个层面的看法。主要的内容涉及虚拟化、虚拟化管理、数据中心虚拟化、云计算、公有云与私有云、以及开放源代码。本文的全部内容均属于作者的个人观点，而不代表任何公司的观点。欢迎讨论。  

**A、虚拟化**

![](http://www.qyjohn.net/wp-content/uploads/2012/10/Server-virtualization.jpg.gif)

虚拟化是指在同一台物理机器上模拟多台虚拟机的能力。每台虚拟机在逻辑上拥有独立的处理器、内存、硬盘和网络接口。使用虚拟化技术能够提高硬件资源的利用率，使得多个应用能够运行在同一台物理机上各自拥有彼此隔离的运行环境。

虚拟化的也有不同的层次，例如硬件层面的虚拟化和软件层面的虚拟化。**硬件虚拟化**指的是通过模拟硬件的方式获得一个类似于真实计算机的环境，可以运行一个完整的操作系统。在硬件虚拟化这个层面，又有Full Virtualization（**全虚拟化**，几乎是完整地模拟一套真实的硬件设备。大部分操作系统无须进行任何修改即可直接运行在全虚拟化环境中。）、Partial Virtualization（**部分虚拟化**，仅仅提供了对关键性计算组件或者指令集的模拟。操作系统可能需要做某些修改才能够运行在部分虚拟化环境中。）和Paravirtualization（**半虚拟化**，不对硬件设备进行模拟，虚拟机拥有独立的运行环境，通过虚拟机管理程序共享底层的硬件资源。大部分操作系统需要进行修改才能够运行在半虚拟化环境中。）等不同的实现方式。**软件层面的虚拟化**，往往是指在同一个操作系统实例的基础上提供多个隔离的虚拟运行环境，也常常被称为**容器技术**。

在硬件虚拟化的层面，现代的虚拟化技术通常是全虚拟化和半虚拟化的混合体。常见的虚拟化技术例如VMWare、Xen和KVM都同时提供了对全虚拟化和半虚拟化的支持。以硬件虚拟化的方式所提供的虚拟机，通常都在运行一个完整的操作系统，在同一台宿主机上存在大量相同或者相似的进程和内存页，从而导致明显的性能损耗。目前，通过KSM等技术可以识别与合并含有相同内容的内存页，但是还没有对大量相同或者相似的进程进行优化处理的有效手段。因此，硬件虚拟化也往往被称为**重量级虚拟化**，在同一宿主机上能够同时运行的虚拟机数量是相当有限的。在软件虚拟化的层面，同一宿主机上的所有虚拟机共享同一个操作系统实例，不存在由于运行多个操作系统实例所造成的性能损耗。因此，软件虚拟化也往往被称为**轻量级虚拟化**，在同一宿主机上能够同时运行的虚拟运行环境数量是比较宽松的。以Solaris操作系统上的Container为例，一个Solaris操作系统的实例理论上可以支持多达8000个Container（实际能够运行的Container数量取决于系统资源和负载）。与此类似，Linux操作系统上的LXC可以轻松地在同一宿主机上同时支持数量可观的虚拟运行环境。

在虚拟化这个领域，国内的公司对硬件虚拟化的兴趣较大，在研发和生产环境中也大都采用硬件虚拟化技术。淘宝是国内较早地研究并应用软件虚拟化技术的，他们在淘宝主站的实践经验表明使用**cgroup**替代Xen能够提升资源利用率。至于在一个实际的应用场景中到底应该选择硬件虚拟化还是软件虚拟化，则应该**重点考虑**最终用户是否需要对操作系统的完全控制权（例如升级内核版本）。如果最终用户仅仅需要对运行环境的控制权（例如PaaS层面的各种App Engine服务），软件虚拟化可能性价比更高。对于为同一应用提供横向扩展能力的应用场景，软件虚拟化也是比较好的选择。

对于需要深入了解虚拟化技术的技术人员来说，VMWare发表的白皮书《[Understanding Full Virtualization, Paravirtualization, and Hardware Assist](http://www.vmware.com/files/pdf/VMware_paravirtualization.pdf)》是一份很好的参考资料。

通常来讲，能够直接使用虚拟化技术的用户数量是比较少的。以Linux操作系统为例，能够进行虚拟机生命周期管理的用户，一般就是具有访问libvirt权限的用户。在一个公司或者其他实体中，这些用户通常是系统管理员。

**B、虚拟化管理**

![](http://www.qyjohn.net/wp-content/uploads/2012/10/virt-manager-screenshot.png)

早期的虚拟化技术，解决的是在同一台物理机上提供多个相互独立的运行环境的问题。当需要管理的物理机数量较小时，系统管理员可以手动登录到不同的物理机上进行虚拟机生命周期管理（资源配置、启动、关闭等等）。当需要管理的物理机数量较大时，就需要写一些脚本／程序来提高虚拟机生命周期管理的自动化程度。以管理和调度大量物理／虚拟计算资源为目的软件，称为虚拟化管理工具。虚拟化管理工具使得系统管理员可以从同一个位置执行如下任务：（1）对不同物理机上的虚拟机进行生命周期管理；（2）对所有的物理机和虚拟机进行查询甚至监控；（3）建立虚拟机命名与虚拟机实例直接的映射关系，使得虚拟机的识别和管理更加容易。Linux操作系统上的VirtManager是一个简单的虚拟化管理工具。在VMWare产品家族中，VMWare vSphere是一个功能强大的虚拟化管理工具。

虚拟化管理工具是虚拟化技术的自然延伸。简单的虚拟化管理工具，解决的是由于物理机数量增多所导致的工作内容繁杂问题。在这个层面，虚拟化管理通常和集群的概念同时出现。一个虚拟化管理工具，往往需要获得各台物理机上的虚拟机生命周期管理权限（例如具有访问libvirt权限的用户名和密码）。在同一个集群当中，为了方便起见，可能需要设定一个在整个集群层面通用的管理用户。可以认为，虚拟化管理**为系统管理员提供了便利**，但是并没有将虚拟机生命周期管理的权限下放给其他用户。

**C、数据中心虚拟化**

![](http://www.qyjohn.net/wp-content/uploads/2012/10/virtualization-diag.gif)

在数据中心的层面，系统管理员需要面对大量不同类型的硬件和应用。与小型的集群相比较，数据中心的系统复杂度大大提高了。这时简单的虚拟化管理工具已经无法满足系统管理员的要求，因此在虚拟化管理工具的基础上又发展出各种数据中心虚拟化管理系统。在硬件层面，数据中心虚拟化管理系统通过划分资源池（一个资源池通常是一个集群）的方式对硬件资源进行重新组织，并以虚拟基础构架（Virtual Infrastructure）的方式将计算资源暴露给用户。在软件层面，数据中心虚拟化管理系统引入系统管理员和普通用户两种不同的角色，甚至是基于应用场景的需要设定颗粒度更细的基于角色的权限控制（Role Based Access Control，RBAC）。系统管理员对整个数据中心的物理机和虚拟机拥有管理权限，但是一般不对正常的虚拟机进行干涉。普通用户只能在自己具有权限的资源池内进行虚拟机生命周期管理操作，不具有控制物理机的权限。在极端的情况下，普通用户只能够看到分配给自己的资源池，而不了解组成该资源池物理机细节。

在数据中心虚拟化之前，创建虚拟机的动作是需要系统管理员来完成的。在数据中心虚拟化管理系统中，通过基于角色的权限控制，虚拟机生命周期管理的权限被下放给所谓的“普通用户”，在一定程度上可以减轻系统管理员的负担。但是，出于系统安全的考虑，并不是公司里所有的员工都能够拥有这样的“普通用户”账号。一般来说，这种“普通账号”只能够分配给某个团队的负责人。可以认为，一直到数据中心虚拟化这个层面，虚拟机的生命周期还是集中式管理的。

**数据中心虚拟化管理系统是虚拟化管理工具的进一步延伸**，它所解决的是由于硬件和应用规模上升所带来的系统复杂度问题。具体的物理设备被抽象成资源池之后，公司高管只需要了解各个资源池的规模、负载和健康状况，最终用户只需要了解分配给自己的资源池的规模、负载和健康状况。只有系统管理员还需要对每一台物理设备的配置、负载和故障了如指掌，但是资源池的概念也从逻辑上对所有的物理设备进行了重新整理和分类，使得系统管理员的工作变得更加容易了。

现代的数据中心虚拟化管理系统，往往提供了大量有助于运维自动化的功能。这些功能包括 （1）基于模板快速部署一系列相同或者是相似的运行环境；（2）监控、报表、预警、会计功能；和（3）高可用性、动态负载均衡、备份与恢复等等。一些相对开放的数据中心虚拟化管理系统，甚至以开放API的方式使得系统管理员能够根据自身的应用场景和流程开发额外的扩展功能。

在VMWare产品家族中，VMWare vCenter是一个数据中心虚拟化管理软件。*(这个说法是存疑的，在我看来VMWare vSphere才是一套数据中心虚拟化管理软件，能够同一管理存储设备，形成资源池，向上提供数据中心等概念)。*其他值得推荐的数据中心虚拟化管理软件包括Convirt、XenServer、Oracle VM、OpenQRM等等。

**D、云计算**

![](http://www.qyjohn.net/wp-content/uploads/2012/10/7384.Figure5.png-550x413.png)

云计算是对数据中心虚拟化的**进一步封装**。在云计算管理软件中，同样需要有云管理员和普通用户两种（甚至更多）不同的角色以及不同的权限。管理员对整个数据中心的物理机和虚拟机拥有管理权限，但是一般不对正常的虚拟机进行干涉。普通用户可以通过浏览器自助地进行虚拟机生命周期管理 ，也可以编写程序通过Web Service自动地进行虚拟机生命周期管理。

在云计算这个层面，<u>虚拟机生命周期管理的权限被彻底下放真正的普通用户，但是也将资源池和物理机等等概念从普通用户的视野中屏蔽了</u>。普通用户可以获得计算资源，但是无需对其背后的物理资源有任何了解。从表面看，云计算似乎就是以与Amazon EC2/S3相兼容的模式提供计算资源。在实质上，云计算是**计算资源管理的模式**发生了改变，最终用户不再需要系统管理员的帮助即可自助地获得获得和管理计算资源。

对于云管理员来说，将虚拟机生命周期管理权限下放到最终用户并没有降低其工作压力。相反，他有了更加令人头疼的事情需要去处理。在传统的IT架构中，往往 是一个应用配备一套计算资源，应用之间存在物理隔离，问题诊断也相对容易。升级到云计算模式之后，多个应用可能共享同一套计算资源，应用之间存在资源竞 争，问题诊断就相对困难。因此，云管理员往往希望选用的云计算管理软件能够有相对全面的数据中心虚拟化管理功能。对于云管理员来说，至关重要的功能包括 （1）监控、报表、预警、会计功能；（2）高可用性、动态负载均衡、备份与恢复等等；和（3）动态迁移，可以用于局部负载调整以及故障诊断。

显而易见，从虚拟化到云计算，<u>对物理资源的封装程度不断提高，虚拟机生命周期的管理权限逐步下放</u>。

在VMWare产品家族中，VMWare vCloud是一个云计算管理软件。其他值得推荐的云计算管理软件包括OpenStack、OpenNebula、Eucalyptus和CloudStack。虽然OpenStack、OpenNebula、Eucalyptus和CloudStack都是云计算管理软件，但是其功能有较大的差别，这些差异源于不同 的软件具有不同的设计理念。OpenNebula和CloudStack最初的设计目标是数据中心虚拟化管理软件，因此具有比较全面的数据中心虚拟化管理 功能。云计算的概念兴起之后，OpenNebula增加了OCCI和EC2接口，CloudStack则提供了称为CloudBridge的额外组件 （CloudStack从 4.0版本开始缺省地包含了CloudBridge组件），从而实现了与Amazon EC2的兼容。Eucalyptus和OpenStack则是以Amazon EC2为原型自上而下地设计成云计算管理软件的，从一开始就考虑与Amazon EC2的兼容性（OpenStack还增加了自己的扩展），但是在数据中心虚拟化管理方面的功能尚有所欠缺。在这两者当中，Eucalyptus项目由于起步较早，在数据中心虚拟化管理方面的功能明显强于OpenStack项目。

**E、私有云与公有云**

![](http://www.qyjohn.net/wp-content/uploads/2012/10/simpsons-cloud.png)

如D所述的云计算，仅仅是一种狭义上的云计算，或者是与Amazon EC2相类似的云计算。 广义上的云计算，可以泛指是指各种通过网络访问物理／虚拟计算机并利用其计算资源的实践，包括如D所述的云计算和如C所述的数据中心虚拟化。这两者的共同点在于云计算服务提供商以虚拟机的方式向用户提供计算资源，用户无须了解虚拟机背后实际的物理资源状况。如果某个云平台仅对某个集团内部提供服务，那么这个云平台也可以被称为“私有云”；如果某个云平台对公众提供服务，那么这个云平台也可以被称为“公有云”。一般来说，私有云服务于集团内部的不同部门（或者应用），强调虚拟资源调度的**灵活性**（例如最终用户能够指定虚拟机的处理器、内存和硬盘配置）；公有云服务于公众，强调虚拟资源的**标准性**（例如公有云服务提供商仅提供有限的几个虚拟机产品型号，每个虚拟机产品型号的处理器、内存和硬盘配置是固定的，最终用户只能够选择与自身需求最为接近的虚拟机产品型号）。

对于公有云服务提供商来说，其业务模式与Amazon EC2相类似。因此，公有云服务提供商通常应该选择如D所述的云计算管理软件。对于私有云服务提供商来说，则应该根据集团内部计算资源的管理模式来决定选用的软件。如果对计算资源进行集中式管理，仅仅将虚拟机生命周期管理的权限下放到部门经理或者是团队负责人这个级别，那么就应该选择如C所述的数据中心虚拟化管理系统。如果要将虚拟机生命周期管理的权限下放到真正需要计算资源的最终用户，则应该选择如D所述的云计算管理软件。

传统上，人们认为私有云是建立在企业内部数据中心和自有硬件的基础上的。但是硬件厂商加入云计算服务提供商的行列之后，私有云与公有云之间的界限变得越来越模糊。Rackspace推出的私有云服务，客户可以选择使用自有的数据中心和硬件，也可以选择租用Rackspace的数据中心和硬件。Oracle最近更进一步提出了“由Oracle拥有并管理”（ Owned by Oracle, Managed by Oracle）的私有云服务。在这种新的业务模式下，客户所独享的私有云是仅仅是云服务提供商的公有云当中与其他客户相对隔离的一个资源池（you got private cloud in my public cloud）。而对于云服务提供商来说，用于提供公有云服务的基础构架可能仅仅是其自有基础构架（私有云）中的一个资源池，甚至是硬件厂商自有基础构架（私有云）中的一个资源池（you got public cloud in my private cloud）。

对于客户来说，使用基于云服务提供商的数据中心和硬件的私有云服务在财务上是合理的。这样做意味着自建数据中心和采购硬件设备的固定资产投入（CapEX）变成了分期付款的运营费用（OPEX），宝贵的现金则可以作为用于拓展业务的周转资金。即使长期下来拥有此类私有云的总体费用比自建数据中心和采购硬件设备要高，但是利用多出来的现金进行业务拓展所带来的回报可能会超过两个方案之间的费用差额。在极端的情况下，即使企业最终没有获得成功，也无需心疼新近购置的一大堆硬件设备。除非是房地产市场在短时间内有较大的起色，一家濒临倒闭的公司通常是不会为没有自建一个数据中心而感到后悔的。（需要指出的是，对于一家能够长时间运作的公司来说，通过房地产来盈利是完全有可能的。在Sun 公司被Oracle公司收购之前，就曾经通过变卖祖业的方式使得财报扭亏为盈。）

那么，硬件厂商在这场游戏里面扮演的是什么角色呢？当用户的固定资产投入（CapEX）变成了分期付款的运营费用（OPEX）时，硬件厂商难道不是需要更长的时间才能够收回货款吗？

1865年，英国经济学家威廉杰文斯（Willian Jevons，1835-1882）写了一本名为《煤矿问题》（The Coal Question）的书。杰文斯描述了一个似乎自相矛盾的现象：蒸汽机效率方面的进步提高了煤的能源转换率，能源转换率的提高导致了能源价格降低，能源价格的降低又进一步导致了煤消费量的增加。这种现象称为**杰文斯悖论**，其核心思想是资源利用率的提高导致价格降低，最终会增加资源的使用量。在过去150年当中，杰文斯悖论在主要的工业原料、交通、能源、食品工业等多个领域都得到了实证。

公共云计算服务的核心价值，是将服务器、存储、网络等等硬件设备从自行采购的固定资产变成了按量计费的公共资源。虚拟化技术提高了计算资源的利用率，导致了计算资源价格的降低，最终会增加计算资源的使用量。明白了这个逻辑，就能够明白为什么HP会果断加入OpenStack的阵营并在OpenStack尚未成熟的情况下率先推出基于基于OpenStack的公有云服务。固然，做云计算不一定能够拯救HP于摇摇欲坠之中，但是如果不做云计算，HP恐怕就时日不多了。同样，明白了这个逻辑，就能够明白为什么Oracle会从对云计算嗤之以鼻摇身一变称为云计算的实践者。收购了Sun公司之后，Oracle一夜之间变成了世界领先的硬件提供商。当时云计算的概念刚刚兴起，Oracle不以为然的态度说明它尚未充分适应自身地位的变化。如今云计算已经从概念炒作进入实战演习阶段，作为主要硬件厂商之一的Oracle如果不打算从云计算中分一杯羹的话，那就是真正的反射弧过长了。

根据杰文斯悖论，对于用户来说，价格降低是用量增加的前提。那么，应该如何给云计算资源定价呢？

目前，大部分公有云服务提供商的虚拟机产品都是按照配置定价的。以Amazon EC2为例，其中型（Medium）虚拟机（3.75 GB内存，2 ECU计算单元，410 GB存储，0.16美元每小时）的配置是小型（Small）虚拟机（1.7 GB内存，1 ECU计算单元，160 GB存储，0.08美元每小时）的两倍，其价格也是小型虚拟机的两倍。新近推出的HP Cloud Services，以及国内的盛大云和阿里云，基本上都照搬Amazon EC2的定价方法。问题在于，虚拟机的配置提高之后，虚拟机的性能并没有得到同比提高。一系列针对Amazon EC2、HP Cloud Services、盛大云和阿里云的性能测试结果表明，对于多种类型的应用来说，随着虚拟机配置的提高，其性价比实际上是不断降低的。这样的定价策略，显然不能达到鼓励用户使用更多计算资源的目的。

按照虚拟机的性能来定价可能是一个更加合适的做法。举个例子说，某个牌子的肥皂有大小两种包装，小包装有一块肥皂而大包装有两块肥皂。用户愿意花双倍的钱购买大包装，往往是因为大包装能够洗两倍的衣服而不是因为它看起来更大。同理，来自同一公有云服务提供商的不同虚拟机产品，应该尽可能使其性价比维持在同一水平线上。问题在于，不同类型的应用对处理器、内存和存储等计算资源的需求存在较大差异，其“性能–配置”变化曲线也各有不同。因此，在公有云服务领域需要一个对虚拟机性能进行综合评估的框架，通过该框架获得的评估结果可以表示一台虚拟机的综合处理能力，而不仅仅是处理器、内存和存储当中的任何一项。基于这样一个测试框架，不仅可以对同一公有云服务提供商的产品进行比较，还可以对不同公有云服务提供商的产品进行比较。

**F、开放源代码**

![](http://www.qyjohn.net/wp-content/uploads/2012/10/open_source_cloud.jpg)

近些年来，我们在信息技术领域观察到**一个规律**。当一个闭源的解决方案在市场上取得成功时，很快就会出现一个甚至是多个提供类似功能（或者服务）的开源或者闭源的追随者。（首先出现开源软件，然后出现与之竞争的闭源软件的案例比较少见。）在操作系统领域，Linux逐渐达到甚至是超越了Unix的技术水平，进而取代Unix的市场地位。在虚拟化领域，Xen和KVM紧紧跟随VMWare的技术发展并有所突破，逐步蚕食VMware的市场份额。在云计算领域，Enomaly率先推出了以Amazon EC2为蓝本的闭源解决方案，紧跟着又出现了以Eucalyptus和OpenStack为代表的开源解决方案。与此同时，传统意义上的闭源厂商对开源项目和社区的**态度也在发生转变**。例如，多年来对开源项目持敌视态度的微软于今年四月组建了一家名为“微软开放技术”（Microsoft Open Technologies）的子公司，其目标是推进微软向开放领域的投资，包括互操作性、开放标准和开源软件。

我们今天所处的商业环境，与上个世纪80年代自由软件运动（Free Software Movement）刚刚兴起的时候已经有了较大的不同。自1998年NetScape第一次提出开放源代码（Open Source）这个术语起，开放源代码就已经成为一种新的软件研发、推广与销售模式，而不再是与商业软件相对立的替代品了。与传统的闭源软件商业模式相对比，基于开放源代码的商业模式具有如下特点：

（1）在项目萌芽阶段，通过开源软件或者自由软件等关键词吸引潜在客户以及合作伙伴。对于潜在客户来说，选择开源软件能够免费或者是低价获得闭源软件的（部分）功能。对于合作伙伴来说，其兴趣点可能在于销售基于开源软件的增强版本（例如企业版），提供基于开源软件的解决方案，或者是该开源软件的成功可能对其自身的产品的销售有促进作用。

（2）在项目成长阶段，主要的研发人员来自发起项目的企业以及该项目的企业合作伙伴。虽然也有一些单纯出于兴趣而向开源项目贡献代码的个人开发者，但是其数量相对较少。我们在开源软件的宣传资料当中经常会见到类似于“由某某社区开发”的描述。最近10年来，各种“社区”中的主要研发力量始终来自数量极为有限的企业合作伙伴。但是有些开源项目在宣传中通常会有意无意地淡化企业合作伙伴的重要性，甚至是误导受众以为社区的主要成分是个人开发者。

（3）在项目收割阶段，项目发起者以及主要合作伙伴可以通过销售增强版本或者是提供解决方案获取财务回报。虽然其他厂商也可以提供类似的产品或者服务，但是开源项目的主要参与者往往在市场上拥有更大的话语权和权威性。关于开源项目的盈利问题，Marten Mickos（Eucalyptus的CEO）在担任MySQL公司CEO期间曾指出：“**如果要在开源软件上取得成功**，那么你需要服务于：（A）愿意花费时间来省钱的人；和（B）愿意花钱来节约时间的人。”如果说一个公司在开源方面取得了成功，那么它从开源软件的销售和服务方面获得的回报至少应该大于在研发和推广方面的投入。显而易见，某些用户之所以能够免费使用开源软件，一方面固然是因为他们的参与降低了开源软件在研发和推广方面的投入，另一方面则是因为付费用户为开源软件付出了更多的钱。

那么，为什么基于开源软件的解决方案通常要比其闭源的竞争对手更便宜呢？通常来说，闭源软件作为一个领域的开创者，在市场研究、产品设计、研发测试、推广销售等等环节都面临很大的挑战。开源软件作为闭源软件的追随者，在市场研究方面有闭源软件作为成功案例，在产品设计方面有闭源软件作为参考模板，在推广销售方面也得益于闭源软件的市场拓展。在研发方面，开源软件出现的时间要稍晚于闭源软件，在这个时间段里发生的技术进步会明显降低开源软件进入相关领域的门槛。除此之外，开源软件可能在某些特性方面超越闭源软件，但在总体水平上其功能的完备性、易用性、稳定性、可靠性会稍逊于闭源软件。因此，基于开源软件的解决方案通常会采取“以闭源软件30%的价格提供闭源软件80%的功能”这样的营销思路。除此之外，基于开源软件的解决方案的可定制性对于某些客户来说也有特别的吸引力。

在中国的商业环境中，IT公司（或者说互联网公司）通常是愿意花费时间来省钱的，而非IT公司（或者说传统行业）通常是愿意花钱来节约时间的。需要指出的是，中国的非IT公司往往不在乎软件是否开源，但是**非常注重开源软件的可定制性**。

开放源代码作为一种新的商业模式，并不比传统的闭源模式具有更高的道德水准。同理，在道德层面上对不同的开放源代码实践进行评判也是不合适的。在OpenStack项目的萌芽阶段，Rackspace公司的宣传文案声称OpenStack是“世界上唯一真正开放源代码的IaaS系统”。CloudStack、Eucalyptus和OpenNebula等具有类似功能的开源项目由于保留了部分闭源的企业版（2012年4 月以前，CloudStack项目和Eucalyptus均同时发布完全开源的社区版和部分闭源的企业版。2012年4 月之后，Eucalyptus项目宣布全面开源，CloudStack项目被Citrix收购并捐赠给Apache基金会后也全面开源。）、或者是仅向付费客户提供的自动化安装包（OpenNebula Pro是一个包含了增强功能的自动化安装包，但是其全部组件都是开放源代码的。）而被Rackspace归类为“不是真正的开放源代码项目”。类似的宣传持续了接近两年时间，直到Rackspace公司推出了基于OpenStack项目的Rackspace Private Cloud软件 — 一个性质上与OpenNebula Pro类似的自动化包。OpenNebula Pro是一个仅向付费用户提供的软件包，但是任何用户都可以免费地下载与使用Rackspace Private Cloud软件。问题在于，当用户所管理的节点数量超过20台服务器时，就需要向Rackspace公司寻求帮助（购买必要的技术支持）。这里我们暂且不讨论将节点数量限制为20台服务器这部分代码是否开源的问题。开源项目的发起者和主要贡献者在其重新打包的发行版中添加了限制该软件应用范围的功能，从道德层面来看很难解释，但是在商业层面来看就很正常。在过去两年中，OpenStack项目在研发、推广、社区等领域所采取的种种措施，都堪称是基于开放源代码的商业模式的经典案例。

前面我们提到，在同一领域往往存在多个相互竞争的开源项目。以广义上的云计算为例，除了我们熟悉的CloudStack、Eucalyptus、OpenNebula、OpenStack之外，还有Convirt、XenServer、Oracle VM、OpenQRM等等诸多选择。针对一个特定的应用场景，如何在众多的开源方案中进行选型呢？根据我个人的经验，可以将整个**方案选型过程**分为需求分析、技术分析、商务分析三个阶段。

（1）在需求分析阶段，针对特定的应用场景深入挖掘该项目采用云计算技术的真正目的。在中国，很多项目决策者对云计算的认识往往停留在“提高资源利用率、降低运维成本、提供更多便利”的阶段，并没有意识到这个列表已经是大部分开源软件均可提供的基本功能。除此之外，很多项目决策者缺省地将VMWare vCenter提供的全部功能作为对开源软件的要求，而没有考虑特定项目是否需要这些功能。因此，非常有必要针对特定的应用场景进行调研，明确将其按照数据中心虚拟化和狭义上的云计算归类，并进一步挖掘项目在功能上的具体要求。在很多情况下，数据中心虚拟化和狭义上的云计算均能够满足客户的总体需求，那么销售的任务就是将客户的具体需求往有利于自身的方向上引导。这个技巧，我们称之为客户期望值管理（Expectation Management）。通过需求分析，明确特定应用场景的分类，可以过滤掉一部分选项。

（2）在技术分析阶段，首先比较各个开源软件的参考架构，重点考虑在特定应用场景下按照参考构架进行实施所面临的困难。其次在功能的层面对各个开源软件进行对比，并将必须具备的功能（Must Have）和能够加分的功能（Good to Have）区别对待。除此之外，还可以对安装配置的难易程度、具体功能的易用性、参考文档的完备性、二次开发的可能性等等进行评估。通过技术分析，可以给各个开源软件打分排名，在此基础上可以淘汰掉得分最低的选项。

（3）在商务分析阶段，必须明确决策者是否愿意为开源的解决方案付费。如果决策者不愿意为付费，那么该项目就属于“愿意花费时间来省钱”的场景，反之则属于“愿意花钱来节约时间”的场景。对于愿意花费时间来省钱的应用场景，主要依赖于开源社区获得技术支持，可以将开源项目的社区活跃度作为重要的参考数据。对于愿意花钱来节省时间的应用场景，主要依赖于服务提供商获得技术支持，应该重点考察服务提供商在业界的影响力以及在本地的服务能力，开源项目的社区活跃度则显得无关紧要了。

在中国（狭义上）的云计算市场， 对于愿意付费的客户来说，CloudStack和Eucalyptus是值得优先考虑的选项。这两个项目的启动时间比较早，具有更好的稳定性和可靠性，在业界有较大的影响力，并且在国内有团队可以提供支持和服务。与此同时，国内一些创业团队开始提供基于OpenStack的解决方案，但是在短时间内很难积累必要的实战经验，而**具备丰富经验的新浪SAE团队**尚未开拓对外提供技术支持的业务。国内虽然也有一些单位在使用OpenNebula，但是在近期内很难形成对第三方提供技术服务的能力。对于愿意花时间的客户来说，CloudStack和OpenStack的优势较为明显，因为两者的社区活跃度相对较高。在这两者当中，CloudStack的功能更加丰富，也有更多的企业级客户以及成功案例，可能是短期内的更佳选择。从长远来看，基于OpenStack的解决方案会越来越流行，但是其他解决方案在技术和市场上也都在不断取得进步，因此在未来三年内很难形成一统天下的局面。单纯从商业上考虑，CloudStack和Eucalyptus获得成功的几率可能会更大一些。

**G、其他**

有些朋友希望我补充一些云计算在中国的现状。坦率地说，目前我尚不掌握充足的数据，在这里暂不展开论述。刘黎明（新浪微博@刘黎明3000）最近发布了一篇题为《点评阿里云盛大云代表的云计算IaaS产业》的文章，值得参考。

关于不同开源项目的社区活跃度比较，可以参考我最近的一篇博客文章《CY12-Q3 OpenStack, OpenNebula，Eucalyptus，CloudStack社区活跃度比较》。另外，我在《HP Cloud Services性能测试》一文中，也初步提出了一个对公有云进行性能评测的方法。

本文中的所有插图，全部来自Google搜索。除此之外，部分概念性内容参考了维基百科的相关条目进行了改写。
