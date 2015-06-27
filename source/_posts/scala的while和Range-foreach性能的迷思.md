title: scala的while和Range.foreach性能的迷思
date: 2015-06-26 17:24:06
tags:
- scala
categories:
- 技术
---
本文讨论while和Range.foreach性能的比较，想快速知道结论的可以跳到最后的图表，关心过程的就慢慢看我写来。

最近同事在工作上要处理一个大数据集，比如一亿条，用scala写的实现性能很差，后来改用java写，性能提高百倍以上。 但最初的代码是用`(1 to 100000000).foreach`来做的，我们都知道用这个跟java的while比不是很公平，应该也用scala的while。 经过测试scala的while和java的while的性能是接近的。
<!-- more -->

我写了简单的程序对比scala while和Range.foreach的性能
```scala
import java.util.Date

object PerfOfWhile extends App {
  def whileLoop(size: Int): Int = {
    var c = 0
    while(c < size){
      c += 1
    }
    c
  }

  def foreachLoop(size: Int): Int = {
    var c = 0
    (1 to size).foreach(_ => c += 1)
    c
  }

  val size = 100000000
  var start = new Date().getTime
  println(s"foreach loop rs: ${foreachLoop(size)}")
  var end = new Date().getTime
  println(s"time used: ${end - start}")

  start = new Date().getTime
  println(s"while loop rs: ${whileLoop(size)}")
  end = new Date().getTime
  println(s"time used: ${end - start}")
}
```
编译运行后结果发现循环次数越大，两者的差距越大，在1千万次循环时：性能相差30倍
```
foreach loop rs: 10000000
time used: 30
while loop rs: 10000000
time used: 1
```
而在1亿条的时候相差超过100倍
```
foreach loop rs: 100000000
time used: 220
while loop rs: 100000000
time used: 2
```
我没有用nanoTime，所以可能不太精确，但却是差距就是这么大，那么问题来了，这么大性能差距的原因是什么呢？
 熟悉scala的同学可能认为(1 to 100000000)是创建了1亿个元素的Range对象，然后遍历，但其实不是，Range里只是记录了start=1,step=1,end=1亿而已。 Range的foreach实现如下：
```scala
@inline final override def foreach[@specialized(Unit) U](f: Int => U) {
    validateMaxLength()
    val isCommonCase = (start != Int.MinValue || end != Int.MinValue)
    var i = start
    var count = 0
    val terminal = terminalElement
    val step = this.step
    while(
      if(isCommonCase) { i != terminal }
      else             { count < numRangeElements }
    ) {
      f(i)
      count += 1
      i += step
    }
  }
```
所以从代码上看，while前的语句只占用常量时间，对最终结果没有多大影响，while里面也只是不复杂的语句。
我开始怀疑是不是f(i)造成了频繁的压栈出栈，但在把whileLoop里的c += 1也改成一个函数后，测试结果没有明显改变。
这个问题彻底激起了我的好奇心，所以我祭出[scalaMeter](http://scalameter.github.io/)来做microbenchmark，这个工具会做JVM预热一遍得出更准确的结果，还能给出漂亮的报告。
我这个测试先测0到1千万的循环，之后测0到2千万，直到最多0到1亿。
scalaMeter里的代码`val sizes: Gen[Int] = Gen.range("size")(10000000, 100000000, 10000000)`  意思就是从1千万到1亿，步长1千万。 测试结果如下：
![Image Title](http://7u2h31.com1.z0.glb.clouddn.com/while_loop_foreach_loop.jpg)
从报告上看，foreach的性能（橙色线）是线性的，消耗时间随着size增长而增长。而while则是一条接近于0的横线。
单独看while就更有趣了

![Image Title](http://7u2h31.com1.z0.glb.clouddn.com/only_while_loop.jpg)
竟然是size越大耗时越少了。 看来scalaMeter做的预热还不够，在运行时完成第一次1千万循环后，运行2千万次循环式速度大幅提高，但之后的耗时竟然就不随循环次数增长了。 这样就导致循环次数越多，while和foreach的性能差距就越大。
为什么while的耗时不随循环次数增长的可靠的原因我还没有找到，字节码里看不出什么，应该是运行时优化了。看上去貌似算0到2千万的时候重用了0到1千万的结果，只有这样才能解释了，有大拿知道的请不吝赐教。而由于未知的原因，foreach里的while里的代码无法很好的优化。具体原因和对foreach的优化点还有待进一步研究。

补充，多谢advancedxy和hongjiang：
advanced:
>  对于 foreach 而言, 匿名函数是用匿名 class 来实现的. 然后匿名函数调用的时候也是调用匿名 class 的方法来实现. 用 jvisualvm 可以观察到跑 foreachLoop 产生了一堆的 java.lang.Integer 对象. 对于匿名函数 class 的具体实现不清楚, 但只能说肯定是调用匿名方法的时候产生的. 于是, 这样的临时中间对象产生 size 个, 速度肯定不会快的. 另外, 如果 jvm 的 heap size 默认只有256M, 对于 1e9 这样的 size, gc 肯定会在运算中 kick in. 这又是对性能的影响. 于是, 博主这边的性能差距只有100倍来看, 肯定是机器的性能很不错, 且设置的 heap size 较大.

大魔头：
> 确实产生了一堆java.lang.Integer,怀疑是基础数据类型autoboxing的缘故，匿名函数对象本身只产生一次，在循环里一直在重用这个函数对象，应该不是影响性能的主要因素。

hongjiang:
> `while ( i<num) x=x+const` 这种会被jvm优化，所以不管while循环多少次这里耗时都是一个常量。
要是把循环里的写法稍微改一下，比如改为：
`if ( c % 19 == 0) c+=3 else c+=1`
这样比较时，两者就是一个数量级了，测试差距大概只在一倍左右。

我修改后做了测试，果然如此：

![Image Title](http://7u2h31.com1.z0.glb.clouddn.com/while_changed.png)

多谢两位！
