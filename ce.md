---
title: JDK-bug 三元运算符遇上拆装箱和JDK版本升级
categories: Java
tags: [jdk]
---

条件表达式/三元运算符(Conditional Operator)想必大家常用，拆/装箱(Unboxing & Autoboxing)问题也熟，它俩碰面砸出来的大坑可能也有人掉进去过(like 本人)，糟糕的是，它和JDK版本有关系(JLS修改带来的兼容问题)，但还有更糟糕的......
血泪史就不提了，总结总结爬坑经验吧。

# 实例场景
```java {.line-numbers}
public class Test {
    public static void main(String[] args) {
        Boolean flag = true;
        Boolean a = null;
        Boolean v = flag ? a : false;
        System.out.println("result is: " + b);
    }
}
```
这段代码的输出结果想必没啥疑惑，很多博文包括我们的Java开发规范也用它做示例，结果如下 ```NPE```：
```java
Exception in thread "main" java.lang.NullPointerException
	at a.Test.main(Test.java:7)
```

继续，再来个例子：
```java {.line-numbers}
import java.util.*;

public class Test {
    public static void main(String[] args) {
        Map<String, Boolean> map = new HashMap<>();
        Boolean b = map != null ? map.get("nil") : false;
        System.out.println("result is: " + b);
    }
}
```
这段代码的输出结果是啥呢？别着急回答，先思考一下，**最好确认下自己的jdk版本**。
到这里依然没有很糟糕，无非是——
jdk7版本下：
```java
Exception in thread "main" java.lang.NullPointerException
	at Test.main(Test.java:7)
```

jdk8版本下：
```java
result is: null
```
为啥两个JDK版本表现不一样呢？不理解的可以顺着往下看，理解的可以移步下一节。

# JLS关于条件表达式的变化
是的，JLS中关于条件表达式的规范，在JDK7到JDK8的升级中改动了，而且改动比较大。

## JDK7下的规范
我们先看下JLS-JDK7中的内容(C15.25/P521)：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/3aebffc6-1417-4933-b7e9-92ceb058ab76.jpg" width="60%" />
</p>

太长不看版就是《阿里巴巴Java开发手册-嵩山版》的总结发言(P21):
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/49c6e00d-b319-4de8-8cd5-1b956d5741e2.jpg" width="60%" />
</p>

本质上包含两部分内容：
 * 拆/装箱
 * 类型对齐(提升)

套用到上面的例子也能顺利理解：因为第三个操作数是基础数据类型 ```boolean```，所以会导致先拆箱，此时第二个操作数为```null```，所以```NPE```

## JDK8下的规范
再来看看JLS-JDK8的内容(C15.25/P579)：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/2eb22528-42c5-4096-a668-81bd21b2ea49.jpg" width="60%" />
</p>

首先是把条件表达式归为3类讨论：
* 布尔型条件表达式: boolean conditional expressions
* 数值型条件表达式: numeric conditional expressions
* 引用型条件表达式: reference conditional expressions.

这三类的划分依据和各自的规范我就不继续贴原文了，很长，大家感兴趣可以去官网查阅，章节(C15.25)和页码(P579)我都标注了。简单总结原因就是：
```java
Boolean b = map != null ? map.get("nil") : false;
```
属于引用型条件表达式，紧接着规范描述了(C15.25.3/P587)：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/4b66678e-4cbb-419b-8e42-b93401e03f2c.jpg" width="60%" />
</p>

意即，这条引用型表达式同时满足“出现在赋值上下文或调用上下文中”，那么就符合“复合引用表达式(poly reference)”的表述，那么“复合引用表达式的类型与目标类型相同”，即实际上的效果是：
```java
Boolean b = map.get("nil");
```
即:
```java
Boolean b = null;
```

字节码层面看一下(JDK8)：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/b90f55e5-2f80-49c9-989a-9f91fc3c7242.png" width="60%" />
</p>

# JDK8后为啥又"回退"了
到这里好似已经解决问题了，很多博文也说到JDK8及后续版本表现一致，但是——
出于好奇和验证心理，随便换一个大于JDK8的版本，比如用JDK11，再来一下：
```java
Exception in thread "main" java.lang.NullPointerException
	at Test.main(Test.java:7)
```

```NPE```又回来了，且报错内容和JDK7一致。第一时间我猜测是不是JDK10引入的```var```有关，由于JLS实在太长找起来很麻烦，直接下载个JDK9先验证一下，结果失望，同样的报错，继续验证JDK17、JDK21，报错内容变成了：
```java
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "java.lang.Boolean.booleanValue()" because the return value of "java.util.Map.get(Object)" is null
	at Test.main(Test.java:7)
```

看上去和JDK11没有本质区别，无非是拆箱步骤内的错误日志调整(猜测，暂未求证)，不影响探索方向: 肯定是JDK8之后对引用型表达式规范做了调整，或者这个场景不再是引用型表达式(二选一，盲猜是前者)。但是咕噜噜的肚子迫使我抬头一看表，好家伙连验证带记录仨小时过去了，赶紧去干饭先。
## JDK8后的规范
回来继续顺着这个思路，直接打开JDK1.8和JDK21的描述：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/ffcb5bf1-2532-4334-966f-ad5c232e577e.png" width="70%" />
</p>

好家伙一眼就定位到了，多了一行！不难读懂，这行就是原因了！
为了确认，再次从JDK9-JDK17的文档里查看，跟JDK21一样。

同样地，字节码层面看一下(JDK21)：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/2d8de918-728a-46c5-a49f-d13b1d437e7a.png" width="60%" />
</p>

对比地，JDK7: 
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/ff7c14ef-1522-4574-8b9a-d3a37b637923.png" width="60%" />
</p>

**这里引申出一个思考问题：如果使用JDk21编译时，指定Language Level为8，并且使用JDK8环境运行，会发生什么？**

## why?
这节是个人的啰里啰嗦碎碎念，可以不看 (╯' - ')╯(┻━┻ 
为什么一个直观上应该很明确的三元运算符，在Java语言里都这么多弯弯绕绕呢？主要问题还是在于：
* ```null```究竟代表什么，它跟上下文和所处的位置有关——这个最多算是语言设计的取舍，就像```return/throw/catch/finally```的执行顺序也是语言相关的一样(这也是个很有意思的问题)。类似这种本就需要取舍的设计哲学在多个维度分析时，时而利大于弊，时而弊大于利，取决于上下文和立场以及和“他者”的对比，但本质上，“取舍”即利弊；
* 基础数据类型和封装类带来的自动拆装箱问题——个人认为这是个败笔了。设计初衷是好的，可惜复杂性是难以预判的，结果就是现在的局面，但有一点可以肯定的是，基础数据类型与Java的"everything in Java is an object"是冲突的。设计者们也知道，但在做取舍时显然下错棋子了——站在当下看，一开始倒不如坚持不要基础数据类型，性能差点没关系，相信随着技术发展进步会有解决方案，就像```synchronized```不就是这么过来的嘛——性能问题事小，语言/版本分裂和兼容性问题事大。

再回到版本迭代产生的结果差异，个人认为是JDK8这个版本做了错误的取舍——即便假设后续的版本表现等同于JDK8，我也依然这么认为：
* 它引入了更多的复杂性

是的，就这个理由就足够了。作为一个应用开发者，你(语言开发者)可以告诉我三元运算符在拆装箱场景有两条明确的规范来定义所有的行为，这没问题，我如果觉得不好记或者不好用，我放弃它选择别的实现逻辑就行。但你中途修改规范并且又加入新的规范就很糟糕了，不兼容是大忌，更抽象的是，它在下下个版本又改回去了(从结果上看)——这迷人的操作实在是哔哔哔~ (╯' - ')╯(┻━┻ 

# one more step
问题到这里虽然找到了答案，但一个疑问却萦绕心头：这在官方是怎么定性的？不可能平白无故多一条规范出来，直观上无非是两个可能性：新版JDK引入的new feature；或旧版JDK的bug fix。

## bug or feature?
顺着这个思路继续探索，不过JLS本身并不会解释某条规范的增删缘由，得从别处寻找答案。首先直观地，JDK7->JDK8还能理解，但JDK9及之后的改动却很难解释，高度怀疑是官方承认的bug并做了修复。
尝试往这个方向找一些线索或者记录，经历了一番搜索，果不其然，bugdatabase真的有这条记录: [Define method invocation compatibility without forcing lambdas to be checked](https://bugs.java.com/bugdatabase/view_bug?bug_id=8069544)
大意是说在含有复合方法调用(poly method invocation)的重载解析和表达式类型检查的场景，某些测试兼容性无法通过，这波及到复合引用表达式的类型解析兼容性。修复方案里，涉及复合引用表达式的部分就是新增了这条规范 (bug 是JDK8引入的，在JDK9解决的，所以这条规范最先出现在JDK9)：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/be5fb5fc-7787-4ab5-b2f6-80b04d28aaa9.png" width="60%" />
</p>

## done?
当找到上面那个官方bug记录时，总结心情就是意满离。至此应该是结束了，再叠个但是——
我们的Java技术手册，需要更新吗？是需要的：第一种场景的描述，在JDK8不成立。其他“类型对齐”场景，概括了三类不同的引用，严口径起见，至少应该标注一下JDK不同版本的不同表现，比如JDK8是个特例。结合我们厂内目前多数应用依然停留在JDK8的现状，以及目前正在推动升级JDK11/17/21，还是很有可能因此造成升级故障的。

# 一点总结
上面这个问题是有意义的，毕竟生产环境很容易遇到，细究其原因也很有必要，不然就会陷入case by case的经验性总结，甚至有些简中区博文都怀疑到CPU上去并且头头是道分析起来了。也有一些没啥意义的问题，比如：
* 网络中的所谓黏包问题，遥想当年校招第一次被问到这个问题时，和面对其他很多我不知道的问题一样，无知带给我的是对对方“知识渊博”的佩服，然后自己查阅资料时发现，这TM就是个伪问题啊；
* Spring三级缓存，加入阿里时的面试就有这个，好家伙我就直接“首先，循环依赖是个糟糕的设计，我觉得spring这个解法很不优雅甚至糟糕。但作为面试题我还是解释一下所谓的三级缓存......”。然后Spring Boot-2.6就默认关闭了这个选项，其实官方也给了不少探讨和最终这么做的解释：[Prohibit circular references by default](https://github.com/spring-projects/spring-boot/issues/27652)
* 以及，身边发生的，生产环境执行个sql："你这里换成count(1)，不要用count(*)"。立马引起了我的警觉，因为：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/70fd9437-0680-4f18-9697-ad3692d51c2e.png" width="60%" />
</p>

诸如此类。个人也从某一次在简中区低质量博文中踩过一次大坑之后，果断转向了遇事不决看官方文档。至今来看，所有坑的最终解释/解决，还从未超出官方文档的范畴，以Java+Spring项目为例，还没有 [JLS](https://docs.oracle.com/javase/specs/jls/se21/jls21.pdf)、[JVMS](https://docs.oracle.com/javase/specs/jvms/se21/html/index.html)、[spring-framework](https://docs.spring.io/spring-framework/reference) 解决不了的疑惑。多数时候直接一头扎进源码反而吃力不讨好，官方文档就够了——当然这是业务工程开发的视角，而非中间件、框架开发者的视角。
最后叠个甲，简中区也有很多质量非常高的博文，只是相对来说踩坑的几率大一些，尤其是CSDN，记得在大学它还没有那么糟糕，而且现在也朝着奇奇怪怪的方向狂奔不止，噫吁嚱——

# ref
使用的JDK版本：
<p align = "left">
<img src="https://oss-ata.alibaba.com/article/2024/07/ea5fcf05-a816-48a9-bd4c-423f9ae49027.png" width="60%" />
</p>