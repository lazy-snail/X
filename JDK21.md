---
title: JDK21初体验
categories: Java
tags: [jdk]
---
# 占有率
先看一下当前各版本占有率情况([from jrebel sample](https://www.jrebel.com/success/resources/2024-Java-developer-productivity-report)，2024年)：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/daf49687-876b-4fa4-82be-9178ea26befb.png" width="50%" />
</p>

对比地，2023年：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/8cd802ad-f39e-403e-81fc-178a20e37be1.png" width="50%" />
</p>
可见高版本的占有率正在快速升高。
可是，正如IPv6的处境类似，在探讨IPv6的普及为什么变成了现在这样一个长久拖沓的事情时，一个普遍的观点是：缺乏核心场景，即目前并没有一个爆发性的应用场景是IPv6应付得了但IPv4无法应对的——那何必花这冤枉钱和精力呢？

# 新特性
那我们来看看JDK roadmap至今的变化：
|版本|发布时间|核心new feature|
|  ----  | ----  | ----  |
| JDK8 | 2014.3 | λ 、 Stream |
| JDK11 | 2018.9 | 模块化，var关键字，增强的集合等类库 |
| JDK17 | 2021.9 | ZGC转正、switch模式匹配 |
| JDK21 | 2023.9 | 虚拟线程、分代ZGC、[GraalVM版本的JDK21](https://www.oracle.com/java/graalvm/what-is-graalvm) |

注: 以switch模式匹配为例，正式引入是在jdk14，这里只讨论LTS版本的变化，所以其可用的第一个LST版本是JDK17，具体一些新特性的引入可以去Oracle官网等处查看，比如https://www.javatpoint.com/java-8-vs-java-11

JDK11除了模块化似乎没有其他特别值得关注的变化，这个LTS在服务端场景显然没有足够的吸引力，但从低于JDK9的版本升级的话，最好选择这个LTS，因为后面的JDK17改动就比较大了。
JDK17呢？撇开几个漂亮的语法糖，这个LTS拥有转正的ZGC(但没有分代的ZGC)。个人认为ZGC带来的改变是肉眼可见的，但生产环境还是再观望观望吧。
JDK21是23年9月发布的，此时距离JDK8的发布差不多已经10年的时间，但前者受到的关注显然没有那么多，这与它所拥有的new feature相比似乎显得有些落寞。个人认为，其引入的上述三个核心新特性中的任意一个，都足够有升级的说服力。
Pandora、HSF等核心组件也已经于23年甚至更早的时间适配了JDK21、Spring Boot3.x，但升级过程还是会遇到一些障碍，个人踩过的坑见: [运营工作台应用治理](https://ata.atatech.org/articles/11000267599)

以下简单记录三个主要的feature内容。

# 虚拟线程
官方demo这里就不贴了，直接看实际场景的的一个case，批量读取IC的对比：
虚拟线程: 
```java
List<ItemInfoDTO> list = new ArrayList<>();
// 每个id提交一个虚拟线程，并执行
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) { 
    itemIds.forEach(i -> executor.submit(() -> {
        list.addAll(doQuery(i));
    }));
}
```

传统线程池: 
```java
// 维护一个线程池，传统操作
public final static ExecutorService pool = new ThreadPoolExecutor(20,
        20, 1000L,
        TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(5000),
        new ThreadPoolExecutor.AbortPolicy());

private List<ItemInfoDTO> batchQuery(Collection<Long> itemIds) {  
    // MultiThreadExecutor是一个自定义的多线程执行器，分批处理
    MultiThreadExecutor<Long, ItemInfoDTO, Object> multiThreadExecutor = MultiThreadExecutor.<Long, ItemInfoDTO, Object>builder()
            .partition(20)
            .divisionModels(itemIds)
            .build();
    return multiThreadExecutor.parallelRun(pool,
            new MultiThreadExecutor.MultiThreadExecutable<>() {
                @Override
                public List<ItemInfoDTO> callable(List<Long> divisionModelPartitions) {
                    return doQuery(divisionModelPartitions);
                }
            });
}
```

## 差异
首先是代码组织形式的变化，虚拟线程不需要“小心谨慎”地设定几个核心参数并维护一个池化的线程池，需要的时候直接新建一个虚拟线程扔给执行器即可，即便8C16G的生产环境机器，也可以轻松撑起数万个虚拟线程。甚至在多数场景连虚拟线程的数量都不需要考虑——当然极端场景是否会有激凸流量导致虚拟线程数异常大还是要考虑的。
性能和资源占用这里没有做深入对比，性能方面仅从接口层面对比了同时期一段时间内的调用，RT变化不大。以下是几个相关的虚拟线程实践或性能对比分析：
[Benchmark JDBC connectors and Java 21 virtual threads](https://mariadb.com/resources/blog/benchmark-jdbc-connectors-and-java-21-virtual-threads/)

[Virtual Threads Performance in Spring Boot](https://blog.fastthread.io/virtual-threads-performance-in-spring-boot/)

[Optimizing Java Performance With Virtual Threads, Reactive Programming, and MongoDB](https://www.mongodb.com/developer/languages/java/virtual-threads-reactive-programming/)

[Performance of Java virtual threads compared to native threads](https://www.reddit.com/r/java/comments/199wsil/performance_of_java_virtual_threads_compared_to/)

# 分代ZGC
首先ZGC并不是全方位更优秀的选项，它只是未来的发展方向，着眼当下，还未完全成熟，对比如下：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/b29a4236-1e4f-4500-a258-ce65633b653c.png" width="60%" />
</p>

## 生产环境表现
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/8ab7a941-cc93-4872-bc3f-746862e992d7.png" width="60%" />
</p>

<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/703ccd97-8bff-4f88-9ce0-8528c53ddf7a.png" width="60%" />
</p>
似乎已经建成了它的亚毫秒级停顿时间模型，但这个生产环境的应用承载的业务很轻，谨慎乐观看待。

## Benchmark
SPEC-JBB给出的benchmark
### Throughput
可见 JDK 17 以来的增益不大。但，JDK 8 与 G1 和 Parallel 的最新 JDK 之间存在显著差异。从性能角度来看，至少应该放弃 JDK 8。
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/aef5eebc-522e-4678-92f5-85abc7991817.png" width="60%" />
</p>

### Latency
G1 在 JDK 8 和 JDK 17 之间变化巨大，JDK 21 在此基础上有稍好的表现。这里要注意，JDK 8 和 JDK 17 之间经历了 7 年多的迭代优化，而 JDK 17 到 21 只有两年。时间跨度短，加之 GC 现在已经运行良好，因此很难在像这样的大型基准测试中获得巨大的进步。
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/df571122-8aac-474e-9ac5-9df27aee48df.png" width="60%" />
</p>

在分代 ZGC 中，可以看到与传统模式之间的显著变化。注意，这种增益大部分来自吞吐量得分的提高。两种 ZGC 模式的暂停长度差别不大，都远低于 1 毫秒。不过，从最坏情况的延迟来看，分代 ZGC 略优于传统模式。
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/44facf66-03ed-45f4-b42f-c17830bf9463.png" width="60%" />
</p>

### Footprint
在固定负载下运行基准测试时的峰值内存开销。从这个角度来看，Parallel 非常稳定。对于 G1，过去十年已经优化了许多效率低下的问题，由此节省的成本非常可观，在这个基准测试中，G1 是内存效率最高的收集器。
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/c92fef87-8b27-4471-a5e1-f02cbba395d7.png" width="60%" />
</p>

# GraalVM for JDK21
个人更倾向于把目前尚未成熟的GraalVM看作是占地盘用的，用以缓解JAVA之与GO/RUST/JS等语言的竞争中某些场景下处于劣势的焦虑——显然对云原生支持不足的焦虑: GO具有后发的先天适配优势，JVM在这里甚至是个负担——容器化部署的场景，我们需要的是一款具有优秀GC能力的本地语言，不需要再加一层VM，容器本身就可以充当/已经是VM层。GraalVM试图用JIT+AOT追平GO的优势，当然这或许会是个漫长的过程——甚至是个无法完成的目标，就像JetBrains试图采用Fleet以及一个长得非常像VS Code的全新IDEA UI来攫取云端编辑器/IDE份额的努力一样，即便最终很可能获得跟Atom编辑器一样的结果，但还是要争取，没有人会原地踏步，GO2.0草案同样有许多振奋人心的东西，所有这些对开发者来说应该算是幸事。

## what is...
从
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/13e86fa0-160f-4b42-beff-b85b56cd4054.png" width="60%" />
</p>

到：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/08def59d-8c0e-49a6-a2f1-ac3df22c3111.png" width="60%" />
</p>
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/9e729f67-0180-4ac9-a904-d0bdb15febea.png" width="60%" />
</p>

## openJDK->GraalVM
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/f9386714-bd25-4054-b5b2-ac640cc72f62.png" width="60%" />
</p>

Graal编译器和C2编译器的性能比较：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/c43d525e-9208-43e3-9c57-b486cafbedde.png" width="60%" />
</p>

Microprofile框架性能比较：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/d2c951d9-76bc-4b54-9c10-4ad8f0ff82cc.png" width="60%" />
</p>

Microprofile Framework内存使用情况比较：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/ad04a18d-bb48-42bf-9293-c89d07b1a58e.png" width="60%" />
</p>

## 跑个demo
生产/预发环境暂时无法部署，在本机尝试。配置好一个demo项目，基于GraalVM for JDK 21，进行本地构建，看上去会经历好几个阶段，且很慢：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/fd8c1850-0916-491f-b7fe-b33cd97ef7a9.png" width="60%" />
</p>

最后会给出一些构建信息和建议：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/560563a1-3c5e-4485-949e-01c623efc418.png" width="60%" />
</p>

二者生产的可运行文件大小差异：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/79944e05-2e84-46e7-8c8a-9316e50e053c.png" width="60%" />
</p>

运行的启动速度和资源占用差异还是很明显的。
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/628e8bd1-b722-4a11-ab98-0ae1b1f255ae.png" width="60%" />
</p>
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/a9dae10e-e285-4fbc-b19b-f2853e732987.png" width="60%" />
</p>

本地可运行文件：
不得不说，这个本地化编译部署，启动速度和内存占用就很离谱了。
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/3f735af1-26b0-4708-bb48-a5dc26d75680.png" width="60%" />
</p>
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/89b412ed-fc45-490b-bf24-891e6977e6e9.png" width="60%" />
</p>

# 是否升级
目前已经于去年陆续将4个应用升级至JDK11，1个应用的生产环境升级至JDK21，1个应用的预发环境升级至JDK21，目前运行及日常监控没有发现特殊故障或异常。
个人看法是，ZGC带来的GC优化、虚拟内存带来的资源降低，以及，JDK8以来积累的各种语法优化、语法糖等等更好的编程体验，都可以成为升级的足够理由。而GraalVM的优势同样非常诱人，期待在云原生场景下的表现，可惜短时间内生产环境是享受不到了，或许要取决于弹内容器部署什么时候支持。

## 选择哪个版本
在去年的这个时间，在与Pandora和SRE的同学沟通中，还是建议JDK8->JDK11这样的稳妥路线。今年依然如此，稳妥起见，最好先升级到JDK11，观察一段时间。如果期望进一步升级，那么下一个LTS，无论是JDK17还是21，跨度都很大，可以直接到JDK21。
当然，我尝试了一个应用大跨度从JDK8->JDK21，可能因为应用没有非常复杂的依赖(已经前置地进行过一次治理)，升级后并没有观察到异常。

## 避坑
除了上面另一个链接里遇到的Pandora等版本问题，一个JDK自身的bug—三元运算符与拆装箱问题也需要关注下：
[条件表达式遇上拆装箱和版本升级](./ce.md)

ref

https://javabetter.cn/jvm/gc-collector.html#zgc

https://wiki.openjdk.org/display/zgc/Main

https://tech.meituan.com/2020/08/06/new-zgc-practice-in-meituan.html

https://kstefanj.github.io/2023/12/13/jdk-21-the-gcs-keep-getting-better.html

https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html

https://www.reddit.com/r/java/comments/199wsil/performance_of_java_virtual_threads_compared_to/

https://www.spec.org/jbb2015/