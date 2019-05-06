# RxJava



## 概述

### 什么是ReactiveX

微软给的定义是，Rx是一个函数库，让开发者可以利用可观察序列和LINQ风格查询操作符来编写异步和基于事件的程序。
使用Rx，开发者可以用Observables表示异步数据流，用LINQ操作符查询异步数据流，用Schedulers参数化异步数据流的并发处理。
Rx可以这样定义：Rx = Observables + LINQ + Schedulers。

ReactiveX.io给的定义是，Rx是一个使用可观察数据流进行异步编程的编程接口，ReactiveX结合了观察者模式、迭代器模式和函数式编程的精华。

### Observable

在ReactiveX中，一个观察者(Observer)订阅一个可观察对象(Observable)。观察者对可观察对象发射的数据或数据序列作出响应。
这种模式可以极大地简化并发操作，因为它创建了一个处于待命状态的观察者哨兵，在未来某个时刻响应可观察对象的通知，不需要阻塞等待可观察对象发射数据。

![rx][images/rx.png]

在很多软件编程任务中，或多或少你都会期望你写的代码能按照编写的顺序，一次一个的顺序执行和完成。
但是在ReactiveX中，很多指令可能是并行执行的，之后他们的执行结果才会被观察者捕获，顺序是不确定的。
为达到这个目的，你定义一种获取和变换数据的机制，而不是调用一个方法。在这种机制下，存在一个可观察对象(Observable)，
观察者(Observer)订阅(Subscribe)它，当数据就绪时，之前定义的机制就会分发数据给一直处于等待状态的观察者哨兵。

这种方法的优点是，如果你有大量的任务要处理，它们互相之间没有依赖关系。
你可以同时开始执行它们，不用等待一个完成再开始下一个（用这种方式，你的整个任务队列能耗费的最长时间，不会超过任务里最耗时的那个）。


## 用操作符组合Observable

对于ReactiveX来说，Observable和Observer仅仅是个开始，它们本身不过是标准观察者模式的一些轻量级扩展，目的是为了更好的处理事件序列。

ReactiveX真正强大的地方在于它的操作符，操作符让你可以变换、组合、操纵和处理Observable发射的数据。

Rx的操作符让你可以用声明式的风格组合异步操作序列，它拥有回调的所有效率优势，同时又避免了典型的异步系统中嵌套回调的缺点。

* 创建操作: Create, Defer, Empty/Never/Throw, From, Interval, Just, Range, Repeat, Start, Timer
* 变换操作: Buffer, FlatMap, GroupBy, Map, Scan和Window
* 过滤操作: Debounce, Distinct, ElementAt, Filter, First, IgnoreElements, Last, Sample, Skip, SkipLast, Take, TakeLast
* 组合操作: And/Then/When, CombineLatest, Join, Merge, StartWith, Switch, Zip
* 错误处理: Catch和Retry
* 辅助操作: Delay, Do, Materialize/Dematerialize, ObserveOn, Serialize, Subscribe, SubscribeOn, TimeInterval, Timeout, Timestamp, Using
* 条件和布尔操作: All, Amb, Contains, DefaultIfEmpty, SequenceEqual, SkipUntil, SkipWhile, TakeUntil, TakeWhile
* 算术和集合操作: Average, Concat, Count, Max, Min, Reduce, Sum
* 转换操作: To
* 连接操作: Connect, Publish, RefCount, Replay
* 反压操作: 用于增加特殊的流程控制策略的操作符









## 调度器Scheduler

如果你想给Observable操作符链添加多线程功能，你可以指定操作符（或者特定的Observable）在特定的调度器(Scheduler)上执行。

某些ReactiveX的Observable操作符有一些变体，它们可以接受一个Scheduler参数。这个参数指定操作符将它们的部分或全部任务放在一个特定的调度器上执行。

使用ObserveOn和SubscribeOn操作符，你可以让Observable在一个特定的调度器上执行。
ObserveOn指示一个Observable在一个特定的调度器上调用观察者的onNext, onError和onCompleted方法。
SubscribeOn更进一步，它指示Observable将全部的处理过程（包括发射数据和通知）放在特定的调度器上执行。

### 调度器的种类

下表展示了RxJava中可用的调度器种类：


|调度器类型|效果|
|:-------|:--|
|Schedulers.computation( )|用于计算任务，如事件循环或和回调处理，不要用于IO操作(IO操作请使用Schedulers.io())；默认线程数等于处理器的数量|
|Schedulers.from(executor)|使用指定的Executor作为调度器|
|Schedulers.immediate( )|在当前线程立即开始执行任务|
|Schedulers.io( )|用于IO密集型任务，如异步阻塞IO操作，这个调度器的线程池会根据需要增长；对于普通的计算任务，请使用Schedulers.computation()；Schedulers.io( )默认是一个CachedThreadScheduler，很像一个有线程缓存的新线程调度器|
|Schedulers.newThread( )|为每个任务创建一个新线程|
|Schedulers.trampoline( )|当其它排队的任务完成后，在当前线程排队开始执行|