# RxJava


## 目录

* [概述](#概述)
  * [什么是ReactiveX](#什么是ReactiveX)
  * [Observable](#Observable)
* [用操作符组合Observable](#用操作符组合Observable)
* [调度器Scheduler](#调度器Scheduler)
  * [调度器的种类](#调度器的种类)
  * [默认调度器](#默认调度器)
  * [使用调度器](#使用调度器)


## 概述

### 什么是ReactiveX

微软给的定义是，Rx是一个函数库，让开发者可以利用可观察序列和LINQ风格查询操作符来编写异步和基于事件的程序。
使用Rx，开发者可以用Observables表示异步数据流，用LINQ操作符查询异步数据流，用Schedulers参数化异步数据流的并发处理。
Rx可以这样定义：Rx = Observables + LINQ + Schedulers。

ReactiveX.io给的定义是，Rx是一个使用可观察数据流进行异步编程的编程接口，ReactiveX结合了观察者模式、迭代器模式和函数式编程的精华。

### Observable

在ReactiveX中，一个观察者(Observer)订阅一个可观察对象(Observable)。观察者对可观察对象发射的数据或数据序列作出响应。
这种模式可以极大地简化并发操作，因为它创建了一个处于待命状态的观察者哨兵，在未来某个时刻响应可观察对象的通知，不需要阻塞等待可观察对象发射数据。

![rx](images/rx.png)

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

* 创建操作: 用于创建Observable的操作符。
  * Create：通过调用观察者的方法从头创建一个Observable
  * Defer：在观察者订阅之前不创建这个Observable，为每一个观察者创建一个新的Observable
  * Empty/Never/Throw：创建行为受限的特殊Observable
    * Empty:创建一个不发射任何数据但是正常终止的Observable
    * Never:创建一个不发射数据也不终止的Observable
    * Throw:创建一个不发射数据以一个错误终止的Observable
  * From：将其它的对象或数据结构转换为Observable
  * Interval：创建一个定时发射整数序列的Observable
  * Just：将对象或者对象集合转换为一个会发射这些对象的Observable
  * Range：创建发射指定范围的整数序列的Observable
  * Repeat：创建重复发射特定的数据或数据序列的Observable
  * Start：创建发射一个函数的返回值的Observable
  * Timer：创建在一个指定的延迟之后发射单个数据的Observable
* 变换操作: 这些操作符可用于对Observable发射的数据进行变换，详细解释可以看每个操作符的文档。
  * Buffer：缓存，可以简单的理解为缓存，它定期从Observable收集数据到一个集合，然后把这些数据集合打包发射，而不是一次发射一个
  * FlatMap：扁平映射，将Observable发射的数据变换为Observables集合，然后将这些Observable发射的数据平坦化的放进一个单独的Observable，可以认为是一个将嵌套的数据结构展开的过程。
  * GroupBy：分组，将原来的Observable分拆为Observable集合，将原始Observable发射的数据按Key分组，每一个Observable发射一组不同的数据
  * Map：映射，通过对序列的每一项都应用一个函数变换Observable发射的数据，实质是对序列中的每一项执行一个函数，函数的参数就是这个数据项
  * Scan：扫描，对Observable发射的每一项数据应用一个函数，然后按顺序依次发射这些值
  * Window：窗口，定期将来自Observable的数据分拆成一些Observable窗口，然后发射这些窗口，而不是每次发射一项。类似于Buffer，但Buffer发射的是数据，Window发射的是Observable，每一个Observable发射原始Observable的数据的一个子集
* 过滤操作: 这些操作符用于从Observable发射的数据中进行选择。
  * Debounce：只有在空闲了一段时间后才发射数据，通俗的说，就是如果一段时间没有操作，就执行一次操作
  * Distinct：去重，过滤掉重复数据项
  * ElementAt：取值，取特定位置的数据项
  * Filter：过滤，过滤掉没有通过谓词测试的数据项，只发射通过测试的
  * First：首项，只发射满足条件的第一条数据
  * IgnoreElements：忽略所有的数据，只保留终止通知(onError或onCompleted)
  * Last：末项，只发射最后一条数据
  * Sample：取样，定期发射最新的数据，等于是数据抽样，有的实现里叫ThrottleFirst
  * Skip：跳过前面的若干项数据
  * SkipLast：跳过后面的若干项数据
  * Take：只保留前面的若干项数据
  * TakeLast：只保留后面的若干项数据
* 组合操作: 组合操作符用于将多个Observable组合成一个单一的Observable。
  * And/Then/When：通过模式(And条件)和计划(Then次序)组合两个或多个Observable发射的数据集
  * CombineLatest：当两个Observables中的任何一个发射了一个数据时，通过一个指定的函数组合每个Observable发射的最新数据（一共两个数据），然后发射这个函数的结果
  * Join：无论何时，如果一个Observable发射了一个数据项，只要在另一个Observable发射的数据项定义的时间窗口内，就将两个Observable发射的数据合并发射
  * Merge：将两个Observable发射的数据组合并成一个
  * StartWith：在发射原来的Observable的数据序列之前，先发射一个指定的数据序列或数据项
  * Switch：将一个发射Observable序列的Observable转换为这样一个Observable：它逐个发射那些Observable最近发射的数据
  * Zip：打包，使用一个指定的函数将多个Observable发射的数据组合在一起，然后将这个函数的结果作为单项数据发射
* 错误处理: 这些操作符用于从错误通知中恢复。
  * Catch：捕获，继续序列操作，将错误替换为正常的数据，从onError通知中恢复
  * Retry：重试，如果Observable发射了一个错误通知，重新订阅它，期待它正常终止
* 辅助操作: 一组用于处理Observable的操作符。
  * Delay：延迟一段时间发射结果数据
  * Do：注册一个动作占用一些Observable的生命周期事件，相当于Mock某个操作
  * Materialize/Dematerialize：将发射的数据和通知都当做数据发射，或者反过来
  * ObserveOn：指定观察者观察Observable的调度程序（工作线程）
  * Serialize：强制Observable按次序发射数据并且功能是有效的
  * Subscribe：收到Observable发射的数据和通知后执行的操作
  * SubscribeOn：指定Observable应该在哪个调度程序上执行
  * TimeInterval：将一个Observable转换为发射两个数据之间所耗费时间的Observable
  * Timeout：添加超时机制，如果过了指定的一段时间没有发射数据，就发射一个错误通知
  * Timestamp：给Observable发射的每个数据项添加一个时间戳
  * Using：创建一个只在Observable的生命周期内存在的一次性资源
* 条件和布尔操作: 这些操作符可用于单个或多个数据项，也可用于Observable。
  * All：判断Observable发射的所有的数据项是否都满足某个条件
  * Amb：给定多个Observable，只让第一个发射数据的Observable发射全部数据
  * Contains：判断Observable是否会发射一个指定的数据项
  * DefaultIfEmpty：发射来自原始Observable的数据，如果原始Observable没有发射数据，就发射一个默认数据
  * SequenceEqual：判断两个Observable是否按相同的数据序列
  * SkipUntil：丢弃原始Observable发射的数据，直到第二个Observable发射了一个数据，然后发射原始Observable的剩余数据
  * SkipWhile：丢弃原始Observable发射的数据，直到一个特定的条件为假，然后发射原始Observable剩余的数据
  * TakeUntil：发射来自原始Observable的数据，直到第二个Observable发射了一个数据或一个通知
  * TakeWhile：发射原始Observable的数据，直到一个特定的条件为真，然后跳过剩余的数据
* 算术和集合操作: 这些操作符可用于整个数据序列。
  * Average：计算Observable发射的数据序列的平均值，然后发射这个结果
  * Concat：不交错的连接多个Observable的数据
  * Count：计算Observable发射的数据个数，然后发射这个结果
  * Max：计算并发射数据序列的最大值
  * Min：计算并发射数据序列的最小值
  * Reduce：按顺序对数据序列的每一个应用某个函数，然后返回这个值
  * Sum：计算并发射数据序列的和
* 转换操作
  * To：将Observable转换为其它的对象或数据结构
  * Blocking：阻塞Observable的操作符
* 连接操作: 一些有精确可控的订阅行为的特殊Observable。
  * Connect：指示一个可连接的Observable开始发射数据给订阅者
  * Publish：将一个普通的Observable转换为可连接的
  * RefCount：使一个可连接的Observable表现得像一个普通的Observable
  * Replay：确保所有的观察者收到同样的数据序列，即使他们在Observable开始发射数据之后才订阅
* 反压操作: 用于增加特殊的流程控制策略的操作符。


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

### 默认调度器

在RxJava中，某些Observable操作符的变体允许你设置用于操作执行的调度器，其它的则不在任何特定的调度器上执行，或者在一个指定的默认调度器上执行。下面的表格个列出了一些操作符的默认调度器：

|操作符|调度器|
|:----|:----|
|buffer(timespan)|computation|
|buffer(timespan, count)|computation|
|buffer(timespan, timeshift)|computation|
|debounce(timeout, unit)|computation|
|delay(delay, unit)|computation|
|delaySubscription(delay, unit)|computation|
|interval|computation|
|repeat|trampoline|
|replay(time, unit)|computation|
|replay(buffersize, time, unit)|computation|
|replay(selector, time, unit)|computation|
|replay(selector, buffersize, time, unit)|computation|
|retry|trampoline|
|sample(period, unit)|computation|
|skip(time, unit)|computation|
|skipLast(time, unit)|computation|
|take(time, unit)|computation|
|takeLast(time, unit)|computation|
|takeLast(count, time, unit)|computation|
|takeLastBuffer(time, unit)|computation|
|takeLastBuffer(count, time, unit)|computation|
|throttleFirst|computation|
|throttleLast|computation|
|throttleWithTimeout|computation|
|timeInterval|immediate|
|timeout(timeoutSelector)|immediate|
|timeout(firstTimeoutSelector, timeoutSelector)|immediate|
|timeout(timeoutSelector, other)|immediate|
|timeout(timeout, timeUnit)|computation|
|timeout(firstTimeoutSelector, timeoutSelector, other)|immediate|
|timeout(timeout, timeUnit, other)|computation|
|timer|computation|
|timestamp|immediate|
|window(timespan)|computation|
|window(timespan, count)|computation|
|window(timespan, timeshift)|computation|

### 使用调度器

除了将这些调度器传递给RxJava的Observable操作符，你也可以用它们调度你自己的任务。下面的示例展示了Scheduler.Worker的用法：

```
worker = Schedulers.newThread().createWorker();
worker.schedule(new Action0() {

    @Override
    public void call() {
        yourWork();
    }
});
// some time later...
worker.unsubscribe();
```