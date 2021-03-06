### 1 概述

dapper最初是作为一个监控工具，逐渐发展为一个监控平台。

#### 目的
收集复杂的分布式系统的行为信息，呈现给开发者。dapper设计的初衷并不是取代所有的监控工具，trace数据往往侧重**性能**方面的调查。

#### 需求
* 低开销(low overhead)：trace系统对在线服务的影响足够小，表现在CPU、内存、IO等资源上
* 应用透明(application-level transparency)：应用程序的开发人员不需要知道trace系统的存在
* 无处不在的部署(ubiquitous deployment)：无所不在的部署，7x24小时监控
* 快速分析：trace数据写入数据库后一分钟内就能统计出来，提供足够快的信息反馈，就可以对生产环境下的异常状况做出快速反应

#### 文献调研
Pinpoint[9]、Magpie[3]和X-Trace[12]和Dapper最为相近，Dapper在许多高阶的设计思想上吸取了Pinpoint和Magpie的研究成果。但是dapper在低开销、应用透明这些方面做出了新的贡献。

### 2 Dapper的分布式追踪
#### 2.1 跟踪树和span
![](dapper-notes/1.png)
![](dapper-notes/3.png)
![](dapper-notes/2.png)

* span：从调用一个RPC到服务器返回结果的时间段是一个span，一次请求对应一个span。
* annotation：一个span中RPC客户端和服务器的交互的关键事件对应的时间节点，例如：客户端发送请求、服务端接收请求等。除此之外，应用程序还可以把自己的业务数据标记到span上，例如：上图中的foo。
* 树形结构：根据RPC调用的层次结构，span之间相应的构成了树状的结构。一棵span树就是一次trace，树中所有的span有相同的trace id。例如：前端A请求了中间层B和C，于是B和C的span就是A的span的子节点。同理，后端D和E的span是中间层C的子节点。

#### 2.2 植入点
dapper零侵入应用服务是通过改造基础组件实现的：

* 线程：如果一个线程位于一个被trace的路径中，dapper会把trace context。保存在threadlocal中，trace context是一个包含span id和trace id的小体积、易拷贝的容器。
* 回调：对于异步回调的计算过程，当回调函数被执行时，dapper会将trace context关联到执行回调函数的线程中，从而实现对异步调用路径的追踪。
* RPC：几乎所有的Google的进程间通信是建立在一个用C++和Java开发的RPC框架上，因此对RPC框架是dapper的重要的植入点。

#### 2.3 标记(annotation)
span上的annotation有两个来源：

* 对基础库组件的植入点产生的annotation，这个是不需要应用服务做任何操作的标记
* 应用服务通过dapper-API实现的带时间戳的自定义annotation，用于记录应用服务的行为，实际上就是日志。dapper支持普通的文本和k-v映射两种形式的annotation。

为了防止应用服务滥用annotation，dapper对于span上annotation的数量可以配置上限。

#### 2.4 采样率
如果这个工具价值未被证实但又对在线服务的性能有影响的话，运维同学是不愿意部署它的，并且某些类型的Web服务对植入带来的性能损耗确实非常敏感。

#### 2.5 trace数据的收集

dapper对trace数据的收集分为三个阶段：

1. 植入点和应用服务产生的span数据写入本地日志
2. dapper守护进程把本地日志中的数据从生产环境的主机中读取出来并发往dapper收集器
3. dapper收集器把trace数据写入bigtable

上述三个阶段（从植入的应用进程产生span数据到它被写入bigtable）总延时的50分位点小于15秒，98分为点是双峰的：75%小于2分钟，剩下25%可能高达数小时。

trace和span在bigtable中组织：

1. 一次trace是bigtable中的一行
2. trace中每个span是这一行的列，bigtable支持稀疏表格结构，使得trace可以有任意数量的span

![](dapper-notes/4.png)

>The Dapper system as described performs trace logging and collection out-of-band with the request tree itself.

原文没有讲带外收集具体是什么，下面是原文解释这样做的两点原因：
1. 带内收集模型(scheme)，即trace数据通过RPC响应的头部返回，这样会影响应用服务的网络动态状态。Google的许多大型系统中，一次trace有上千个span很常见，然而即使是靠近如此大规模的分布式trace根节点的RPC响应，它的大小也很小，通常不超过10kB。在这种情况下，带内收集会导致RPC中应用数据很少，但是用于分析的trace数据很多。
2. 带内收集模型假定所有的RPC是嵌套的，我们发现在一些中间件系统中，在所有的后端返回结果之前，中间件服务器就向调用它的客户端返回结果了，带内收集模型无法处理这类非嵌套的分布式执行过程。

#### 2.6 安全和隐私方面的考虑
尽管dapper如果能够trace到RPC的有效负载（RPC调用时的参数），可以让分析工具能够从负载数据中发现一些能够解释异常的原因，但是RPC的有效负载有时会包含一些用户的敏感信息，即使对于debug的工程师也不能看到。

基于安全性和隐私的考虑，目前dapper只存储RPC的方法名但不存储有效负载。作为一个替代的折衷方案，应用层的annotation可以用来关联一些应用层数据，用于支持后续span的分析。

dapper还提供了一些它的设计者都没有料到的安全上的便利（设计者都没有料到，那是怎么实现的？），例如：

1. 通过对公开的安全协议参数的trace，dapper可以完成相应级别的验证和加密，从而用于监控应用服务是否满足安全协议。
2. dapper还可以提供信息以确保基于某些策略的系统隔离按预期执行，比如带有敏感信息的应用不能和未验证的系统组件进行交互。

上述这些测量提供了比代码审计更强大的保障。

### 3 Dapper部署状况
Dapper作为我们生产环境下的追踪系统已经超过两年。本节中介绍的重点是Dapper如何满足无处不在的部署和应用级的透明。

#### 3.1 Dapper运行库
对基础RPC、线程库、控制流库（异步回调）等组件植入，实现span的创建、采样、写日志到本地磁盘。对植入代码的要求：

* 轻量级：核心的植入代码量1000行C++、800行Java
* 稳定健壮：Dapper运行库植入到大量的应用中，使得维护和修bug都很困难

#### 3.2 生产环境覆盖
dapper的侵入可以从两个维度评估：

1. 连接了dapper测量运行库的生产环境进程，它们产生了一部分trace数据
2. 运行了dapper收集守护程序的生产环境机器。
 
dapper守护进程是google机器使用的系统镜像的一部分，因此它存在在每台google的服务器上。因为没有产生trace数据的进程对dapper是不可见的，因此获得准确的可以支持dapper的进程数量是比较困难的，但是考虑到大规模的dapper植入运行库，因此我们可以估计几乎所有的google生产进程都支持trace。

dapper不能正确的跟踪控制流的case也是存在的，它们都是源于使用了非标准的控制流原语或是dapper错误的把路径关联到不相关的事件上。dapper提供了一个简单的库来帮助开发者手动控制跟踪传播作为一种变通方法。目前有40个C++应用程序和33个Java应用程序需要一些手动控制的追踪传播，不过这只是上千个的跟踪中的一小部分。也有非常小的一部分程序使用的非组件性质的通信库（比如原生的TCP Socket或SOAP RPC），因此不能直接支持Dapper的跟踪。

### 4 处理追踪开销
追踪系统的开销有两部分组成：

1. 生成并收集trace数据会降低被追踪系统的处理能力
2. 存储和分析trace数据需要额外的资源

尽管trace框架带来的价值相对性能上的降低是值得的，但是我们相信如果基本损耗能达到可以忽略的程度，那么对跟踪系统最初的推广会有极大的帮助。

本节主要介绍以下三个方面，除此之外还介绍了dapper的自适应采样帮助我们在低开销和trace典型数据之间取得平衡：

1. dapper组件操作的消耗
2. trace数据收集的消耗
3. dapper对生产环境负载的影响

#### 4.1 trace数据生成的开销
收集和分析可以在紧急状态下关闭，但是trace数据的生成是动态链接在应用服务中的，因此它是dapper开销中起决定性作用的一部分。

trace数据的生成中，占主要作用的是span和annotation的创建和销毁，以及把它们写入到本地磁盘。以下是在2.2GHz的x86服务器上测量的数据：

* 根span创建和销毁：204纳秒
* 其他span：176纳秒，根span需要分配一个全局唯一的trace id
* 添加annotation到未被采样的span：9纳秒，dapper运行时库中进行一次查找
* 添加annotation到采样的span：40纳秒
* 写日志到磁盘：写操作和trace是异步的并且多个写操作可以合并成一次，这样大大降低了开销，因为写磁盘是昂贵的操作。如果每个请求都被trace，对于吞吐量较高的服务，写日志还是会产生可以察觉的性能影响，4.3节以网页搜索为例给出了量化数据。

#### 4.2 trace数据收集开销

* CPU：读取本地磁盘的trace数据同样会干扰前台的服务进程工作负载，下表是最坏情况下，dapper收集日志的守护进程在高于实际情况的负载基准下进行测试时的CPU使用率，单核不超过0.3%。
* 网络：dapper是一个轻量级的网络资源消费者，每个span在bigtable中平均大小是426字节，在google的生产环境中，dapper收集trace数据所占用的网络流量不足0.01%。

![](dapper-notes/5.png)

#### 4.3 生产环境下对负载的影响

下表是在google的网页搜索集群上控制采样率测量出的dapper对平均时延和吞吐量的影响结果，结论如下：尽管吞吐量随采样率的影响很小，但是为了避免明显的延时降级，需要调整合适的采样率。保持较低采样率：

1. 允许应用进程有较大的宽裕度使用annotation-API而不用担心性能损失
2. 允许磁盘上的trace数据能在GC前存活较长的时间，从而为收集器带来一些灵活性

![](dapper-notes/6.png)

#### 4.4 自适应采样

采样率和dapper的开销成正比。

统一采样率：第一个dapper的生产版本使用了统一的采样率1/1024。

* 优点：简单有效。因为线上服务吞吐量高，大感兴趣的事件可以复现并且能够被捕捉。
* 缺点：对于低流量的服务会错失感兴趣的事件，但是提高采样率会造成开销的增加。

自适应采样率：

* 思想：dapper试图避免人工干预采样率，设定参数化的期望采样速率。并且会记录在trace数据中用于后续的分析。
* 低流量服务自动增加采样速率，高流量服务相应降低采样速率。

问题：原文中使用了sampling probability和sampling rate描述采样速率，这两个描述是否等同尚不清楚。

#### 4.5 处理激进采样
背景：dapper的新用户会怀疑低至0.01%的采样概率是否会影响到后面trace数据的分析。

回答：对于高吞吐量的服务，激进采样不会妨碍最重要的分析，因为一个值得注意的操作出现了一次，那么它还会出现很多次。对于低流量的服务（QPS低于100），可以有足够的处理能力来承担trace每个请求。这也是为什么我们实现了自适应采样。

#### 4.6 收集过程中的附加采样
背景：dapper需要控制最终写入到bigtable中的数据大小，因此我们实现了第二轮采样。

需求：目前我们的生产环境集群每天产生1TB的trace数据，dapper的用户需要保存最近两周的trace数据。trace数据密度的增加必须要和dapper数据库的机器以及磁盘开销权衡。并且对高比例的请求进行采样会导致dapper的收集器逼近bigtable吞吐量的上限。

实现：把trace id哈希到[0,1]区间内，哈希后的值记为z。收集器从配置文件中读取sampling coefficient，如果z比采样常数小，则保留这个span并写入bigtable，否则丢弃这个span。

原理：同一个trace的所有span具有相同的trace id，所以对于同一个trace的所有span，要么全丢掉要么全被采集。

优点：方便了dapper采样率的管理。收集器部署在全局唯一的机器上，因此只要调整这一个sampling coefficient就可以控制最终写入bigtable的数据量。所有链接了dapper运行库的在线服务不需要调整采样率，当然因为这样的在线服务太多了，也不好逐一调整。

### 5 dapper通用工具
几年前，当dapper还只是个原型的时候，它只能在dapper开发者耐心的支持下使用。从那时起，我们逐渐迭代的建立了收集组件，编程接口，和基于Web的交互式用户界面，帮助dapper的用户独立解决自己的问题。本节总结哪些的方法有用，哪些用处不大，我们还提供关于这些通用的分析工具的基本的使用信息。

#### 5.1 Depot API(DAPI)
功能：直接访问dapper数据库中的分布式trace数据，这些数据是存储在数据库中的原始格式。

访问trace数据的三种可行方式：

1. 根据trace id访问
2. 批量(bulk)访问：DAPI利用mapreduce并行访问上亿的trace数据
3. 索引访问：从特定主机或者服务名映射到(commonly-requested
trace features)到某个trace，这是最快的访问一个特定的服务或者是主机关联的trace的方法。

选择合适的自定义索引是设计DAPI最具有挑战性的部分，因为压缩存储在trace数据中的索引仅仅比trace数据本身小26%，所以索引的开销是很大的。

开始我们设计了两个索引：主机和服务名。然而我们发现主机索引没有什么卵用，当用户关注某台主机时，他们也会关注（这台机器上）一个特定的服务，所以我们最终把这两个索引合并为一个符合索引，这个索引支持高效率的查询，依次按照服务名、主机名以及时间戳。

DAPI在google中有三类应用：

1. 基于DAPI的持久性web应用
2. 维护良好的可以在控制台上调用的基于DAPI的工具
3. 可以被写入，运行、不过大部分已经被忘记了的一次性分析工具

我们知道的有3个持久性的基于DAPI的应用程序，8个额外的按需定制的基于DAPI分析工具，以及使用DAPI框架构建的约15~20一次性的分析工具。

#### 5.2 dapper的用户接口

下图展示了一个典型的用户使用基于web的dapper用户接口的过程，这也是绝大多数dapper用户使用dapper的方式。下面的序号和图片中的数字对应：

1. 用户指定他们感兴趣的服务和事件以及span名称等信息（这些信息用于区分出特定模式的trace）。除此之外，用户还可以指定一个开销的指标。
2. 表格展示了与给定的服务关联的分布式execution pattern。用户从中选出一个详细查看。
3. 图形化的展示了execution pattern，examination中的服务会被高亮。
4. 步骤1中选择了开销的指标以后，dapper的UI会呈现一个基于该指标的简单频率分布图。在这个例子中，对于选中的execution pattern显示出一个对数坐标的延迟正态分布图。用户还可以看到一个列表，列表显示了落在分布图中不同区间的trace。用户可以点击列表中的trace显示出trace检查视图。
5. dapper的大多数用户最终会检查某个trace的情况，希望能收集一些信息去了解系统行为的根源所在。我们没有足够的空间来做trace视图的审查，但UI提供了一个全局时间轴并能够展开和折叠的树形结构，这也很有特点。分布式跟踪树的相邻层用嵌套的不同颜色的矩形表示。每一个RPC的span被从时间上分解为一个服务器进程中的消耗（绿色部分）和在网络上的消耗（蓝色部分）。用户的annotation没有显示在这个截图中，但它们可以选择性的以span的形式包含在全局时间轴上。

![](dapper-notes/7.png)

对于需要实时数据的用户，dapper的UI能够直接和每个生产环境机器上的dapper守护程序通信。在这种模式下，就不能看到系统层面的框图，但是还是可以基于延迟或者网络特性选择一个特定的trace，实时数据再几秒内会看到。

根据我们的日志，每个工作日大约200个不同的google工程师使用dapper的UI，每周大约有750-1000个独立的用户。在内部的新特征通告上，这些数字是逐月连续的。通常用户会发送特定trace的连接，这将不可避免地在查询跟踪情况时中产生很多一次性的，持续时间较短的交互。

### 6 经验
dapper广泛应用在google中，直接的是通过web的用户界面，间接的是通过可编程的API（DAPI？）或者基于这些API开发的应用。这一节我们不会罗列出已知dapper每一种已知的使用方式，而是尝试给出dapper使用场景的“正交向量”，从而说明哪类应用中dapper最成功。

#### 6.1 开发中使用dapper
google的adword系统是围绕一个大型的关键词定位准则和相关文字广告的数据库搭建的。当新的关键词或者广告被添加或者修改时，它们必须被检查是否符合服务策略（例如：语言是否合适，这个过程使用自动检查系统是更有效的）。