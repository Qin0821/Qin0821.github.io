### 1. 值获取来源不一样
just 值从外部获取的，而 fromCallable 值来自于内部生成，为了更清楚的了解，我们来看一下下面的代码：
```java
println("From Just")
val justSingle = Single.just(Random.nextInt())
justSingle.subscribe{ it -> println(it) }
justSingle.subscribe{ it -> println(it) }

println("\nFrom Callable")
val callableSingle = Single.fromCallable { Random.nextInt() }
callableSingle.subscribe{ it -> println(it) }
callableSingle.subscribe{ it -> println(it) }
```
复制代码对于 Just 和 fromCallable 分别调用 2 次 subscribe 执行结果如下所示：
```java
From Just
801570614
801570614

From Callable
1251601849
2033593269
```
复制代码你会发现对于 just 无论 subscribe 多少次，生成的随机值都保持不变，因为该值是从 Observable 外部生成的，而 Observable 只是将其存储供以后使用。
但是对于 fromCallable 它是从 Observable 内部生成的，每次 subscribe 都会都会生成一个新的随机数。
### 2. 立即执行和延迟执行

just 在调用 subscribe 方法之前值已经生成了，属于立即执行。
而 fromCallable 是调用 subscribe 方法之后执行的，属于延迟执行。

为了更清楚的了解，我们来看一下下面的代码：
```java
fun main() {
    println("From Just")
    val justSingle = Single.just(getRandomMessage())
    println("start subscribing")
    justSingle.subscribe{ it -> println(it) }
   
    println("\nFrom Callable")
    val callableSingle = Single.fromCallable { getRandomMessage() }
    println("start subscribing")
    callableSingle.subscribe{ it -> println(it) }
}

fun getRandomMessage(): Int {
    println("-Generating-")
    return Random.nextInt()
}
```
复制代码结果如下所示：
```java
From Just
-Generating-
start subscribing
1778320787

From Callable
start subscribing
-Generating-
1729786515
```
复制代码对于 just 在调用 subscribe 之前打印了 -Generating-，而 fromCallable 是在调用 subscribe 之后才打印 -Generating-。