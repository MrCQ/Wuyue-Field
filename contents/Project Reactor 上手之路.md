# Project Reactor 上手之路

响应式编程来源于数据流和变化的传播，意味着由底层的执行模型负责通过数据流来自动传播变化

## Reactive Stream 实现项目对比

### RxJava

RxJava 库是 JVM 上反应式编程的先驱，也是反应式流规范的基础。RxJava 2 在 RxJava 的基础上做了很多的更新。
 
不过 RxJava 库也有其不足的地方。RxJava 产生于反应式流规范之前，虽然可以和反应式流的接口进行转换，但是由于底层实现的原因，使用起来并不是很直观。

RxJava 2 在设计和实现时考虑到了与规范的整合，不过为了保持与 RxJava 的兼容性，很多地方在使用时也并不直观。

### Project Reactor

Reactor 则是完全基于反应式流规范设计和实现的库，没有 RxJava 那样的历史包袱，在使用上更加的直观易懂。Reactor 也是 Spring 5 中反应式编程的基础。学习和掌握 Reactor 可以更好地理解 Spring 5 中的相关概念

## Reactive Stream 与 Java8 Stream 相似性

Reactive Stream 的很多实现相对于Java8 中的Stream来说，方法层面的相似性还是很大的，但是内在却完全不一样，Java8 中的Stream实际上还是通过迭代器模式对序列中的数据进行遍历操作，
而这种遍历模式是由调用者控制，本质上属于pull模式，调用者每次通过next()方式获取下一个值

而Reactive Stream采用的是push模式，也即是我们常见的发布者-订阅者模式，当发布者有新的数据时，数据会通过流传达到对应的订阅者，同时也不仅限于此，单纯使用push模式的话，一旦订阅者
处理信息的能力低于信息的产生速率，会导致订阅者本身的崩溃，Reactive Stream中采用**负压**的方式，即提供一种从订阅者到发布者的反馈渠道，订阅者通过request()方法申明其一次能够处理
的消息数量，生产者生产相对应数量的消息，直到下一次的request()发生，实际就变成了推拉结合的模式。

## Project Reactor 上手

Reactor 3 利用了java 8中的CompletableFuture、Stream、Duration，在此基础上发展出了二个更通用的模型：Flux 及 Mono

maven 依赖：

```
<dependency>
	<groupId>io.projectreactor</groupId>
	<artifactId>reactor-core</artifactId>
</dependency>
```

### Flux 与 Mono

Flux 表示的是包含 0 到 N 个元素的异步序列。

其中包含三种不同类型的消息通知：

* 正常的包含元素的消息，对应 onNext()
* 序列结束的消息，对应 onComplete()
* 序列出错的消息，对应 onError()

Mono 表示的是包含 0 或者 1 个元素的异步序列。该序列中同样可以包含与 Flux 相同的三种类型的消息通知。

并且，Flux 和 Mono 之间可以进行转换。

Flux 和 Mono 的核心在于发射数据，作为数据来源

###  常用方法

#### Flux 创建

* `just()`：可以指定序列中包含的全部元素。创建出来的 Flux 序列在发布这些元素之后会自动结束。
* `fromArray()，fromIterable() 和 fromStream()`：可以从一个数组、Iterable 对象或 Stream 对象中创建 Flux 对象。
* `empty()`：创建一个不包含任何元素，只发布结束消息的序列。
* `error(Throwable error)`：创建一个只包含错误消息的序列。
* `never()`：创建一个不包含任何消息通知的序列。
* `range(int start, int count)`：创建包含从 start 起始的 count 个数量的 Integer 对象的序列。
* `interval(Duration period) 和 interval(Duration delay, Duration period)`：创建一个包含了从 0 开始递增的 Long 对象的序列。其中包含的元素按照指定的间隔来发布。除了间隔时间之外，还可以指定起始元素发布之前的延迟时间。
* `intervalMillis(long period) 和 intervalMillis(long delay, long period)`：与 interval() 方法的作用相同，只不过该方法通过毫秒数来指定时间间隔和延迟时间。

Example: 

```java
Flux.just("Hello", "World").subscribe(System.out::println);
Flux.fromArray(new Integer[] {1, 2, 3}).subscribe(System.out::println);
Flux.empty().subscribe(System.out::println);
Flux.range(1, 10).subscribe(System.out::println);
Flux.interval(Duration.of(10, ChronoUnit.SECONDS)).subscribe(System.out::println);
Flux.intervalMillis(1000).subscribe(System.out::println);
```

上面的这些静态方法适合于简单的序列生成，当序列的生成需要复杂的逻辑时，则应该使用 `generate()` 或 `create()` 方法

* `generate()`

该方法通过同步和逐一的方式来产生 Flux 序列，通过调用所提供的 SynchronousSink 对象的 next()，complete() 和 error(Throwable) 方法来完成的，注意next()只能被调用一次
如果没有显式调用complete()，则会一致不断生产发送到订阅者

Example: 

```java
Flux.generate(sink -> {
    sink.next("Hello");
    sink.complete();
}).subscribe(System.out::println);
 
 
final Random random = new Random();
Flux.generate(ArrayList::new, (list, sink) -> {
    int value = random.nextInt(100);
    list.add(value);
    sink.next(value);
    if (list.size() == 10) {
        sink.complete();
    }
    return list;
}).subscribe(System.out::println);
```

* `create()`

create() 方法与 generate() 方法的不同之处在于所使用的是 FluxSink 对象，其支持同步和异步的消息产生，并且可以在一次调用中产生多个元素，next()不受限于1次调用

```java
Flux.create(sink -> {
    for (int i = 0; i < 10; i++) {
        sink.next(i);
    }
    sink.complete();
}).subscribe(System.out::println);
```

#### Mono 创建

Mono 也包含同 Flux 的创建方法，如just()，empty()，error() 和 never() ，同时也存在其特有的方法：

* `fromCallable()、fromCompletionStage()、fromFuture()、fromRunnable() 和 fromSupplier()`：分别从 Callable、CompletionStage、CompletableFuture、Runnable 和 Supplier 中创建 Mono。
* `delay(Duration duration) 和 delayMillis(long duration)`：创建一个 Mono 序列，在指定的延迟时间之后，产生数字 0 作为唯一值。
* `ignoreElements(Publisher<T> source)`：创建一个 Mono 序列，忽略作为源的 Publisher 中的所有元素，只产生结束消息。
* `justOrEmpty(Optional<? extends T> data) 和 justOrEmpty(T data)`：从一个 Optional 对象或可能为 null 的对象中创建 Mono。只有 Optional 对象中包含值或对象不为 null 时，Mono 序列才产生对应的元素。

还可以通过 create() 方法来使用 MonoSink 来创建 Mono

Example:

```java
Mono.fromSupplier(() -> "Hello").subscribe(System.out::println);
Mono.justOrEmpty(Optional.of("Hello")).subscribe(System.out::println);
Mono.create(sink -> sink.success("Hello")).subscribe(System.out::println);
```

### 操作符

#### `buffer() / bufferTimeout()`

这两个操作符的作用是把当前流中的元素收集到集合中，并把集合对象作为流中的新元素, 在进行收集时可以指定不同的条件：**所包含的元素的最大数量或收集的时间间隔**

Example:

```java
//输出的是 5 个包含 20 个元素的数组
Flux.range(1, 100).buffer(20).subscribe(System.out::println);
//输出的是 2 个包含了 10 个元素的数组
Flux.intervalMillis(100).bufferMillis(1001).take(2).toStream().forEach(System.out::println);
//输出的是 5 个包含 2 个元素的数组
Flux.range(1, 10).bufferUntil(i -> i % 2 == 0).subscribe(System.out::println);
//输出的是 5 个包含 1 个元素的数组，数组里面包含的只有偶数
Flux.range(1, 10).bufferWhile(i -> i % 2 == 0).subscribe(System.out::println);
```

#### `filter()`

数据过滤，简单直观

Example:

```java
Flux.range(1, 10).filter(i -> i % 2 == 0).subscribe(System.out::println);
```

#### `zipWith()`

zipWith 操作符把当前流中的元素与另外一个流中的元素按照一对一的方式进行合并，且在合并时可以不做任何处理，由此得到的是一个元素类型为 Tuple2 的流。

Example:

```java
Flux.just("a", "b")
        .zipWith(Flux.just("c", "d"))
        .subscribe(System.out::println);
Flux.just("a", "b")
        .zipWith(Flux.just("c", "d"), (s1, s2) -> String.format("%s-%s", s1, s2))
        .subscribe(System.out::println);
```

#### `take()`

take 系列操作符用来从当前流中提取元素。提取的方式包括：

* `take(long n)，take(Duration timespan) 和 takeMillis(long timespan)`：按照指定的数量或时间间隔来提取。
* `takeLast(long n)`：提取流中的最后 N 个元素。
* `takeUntil(Predicate<? super T> predicate)`：提取元素直到 Predicate 返回 true。
* `takeWhile(Predicate<? super T> continuePredicate)`： 当 Predicate 返回 true 时才进行提取。
* `takeUntilOther(Publisher<?> other)`：提取元素直到另外一个流开始产生元素。

Example:

```java
Flux.range(1, 1000).take(10).subscribe(System.out::println);
Flux.range(1, 1000).takeLast(10).subscribe(System.out::println);
Flux.range(1, 1000).takeWhile(i -> i < 10).subscribe(System.out::println);
Flux.range(1, 1000).takeUntil(i -> i == 10).subscribe(System.out::println);
```

#### `reduce() / reduceWith()`

reduce 和 reduceWith 操作符对流中包含的所有元素进行累积操作，得到一个包含计算结果的 Mono 序列，而累积操作是通过一个 BiFunction 来表示的。

Example:

```java
Flux.range(1, 100).reduce((x, y) -> x + y).subscribe(System.out::println);
Flux.range(1, 100).reduceWith(() -> 100, (x, y) -> x + y).subscribe(System.out::println);
```

#### `merge() / mergeSequential()`

merge 和 mergeSequential 操作符用来把多个流合并成一个 Flux 序列，不同之处在于 merge 按照所有流中元素的实际产生顺序来合并，而 mergeSequential 则按照所有流被订阅的顺序，以流为单位进行合并

Example:

```java
Flux.merge(Flux.intervalMillis(0, 100).take(5), Flux.intervalMillis(50, 100).take(5))
        .toStream()
        .forEach(System.out::println);
Flux.mergeSequential(Flux.intervalMillis(0, 100).take(5), Flux.intervalMillis(50, 100).take(5))
        .toStream()
        .forEach(System.out::println);
```

#### `flatMap() / flatMapSequential()`

flatMap 和 flatMapSequential 操作符把流中的每个元素转换成一个流，再把所有流中的元素进行合并，

flatMapSequential 和 flatMap 之间的区别与 mergeSequential 和 merge 之间的区别是一样的

Example:

```java
Flux.just(5, 10)
        .flatMap(x -> Flux.intervalMillis(x * 10, 100).take(x))
        .toStream()
        .forEach(System.out::println);
```

flatMap 与 merge 的区别在于，传入merge的是流数组，是已经存在的流，而flatmap传入的是生成流的函数，再将生成的流合并

#### `concatMap()`

concatMap 操作符的作用也是把流中的每个元素转换成一个流，再把所有流进行合并

与 flatMap 不同的是，concatMap 会根据原始流中的元素顺序依次把转换之后的流进行合并；

与 flatMapSequential 不同的是，concatMap 对转换之后的流的订阅是动态进行的，而 flatMapSequential 在合并之前就已经订阅了所有的流

Example:

```java
Flux.just(5, 10)
        .concatMap(x -> Flux.intervalMillis(x * 10, 100).take(x))
        .toStream()
        .forEach(System.out::println);
```

#### `combineLatest()`

combineLatest 操作符把所有流中的最新产生的元素合并成一个新的元素，作为返回结果流中的元素

Example:

```java
Flux.combineLatest(
        Arrays::toString,
        Flux.intervalMillis(100).take(5),
        Flux.intervalMillis(50, 100).take(5)
).toStream().forEach(System.out::println);
```

主： 会指定合并的方式，通过lambda表达式实现

### 消息处理

通过 subscribe 方法来添加相应的订阅逻辑。在调用 subscribe 方法时可以指定需要处理的消息类型

可以选择只处理正常消息，也可以处理异常或完成信息

* 通过subscribe处理正常以及异常的消息

```java
Flux.just(1, 2)
        .concatWith(Mono.error(new IllegalStateException()))
        .subscribe(System.out::println, System.err::println);
```

* 设置异常返回默认值

```java
Flux.just(1, 2)
        .concatWith(Mono.error(new IllegalStateException()))
        .onErrorReturn(0)
        .subscribe(System.out::println);
```

* 出现错误时使用另外的流

```java
Flux.just(1, 2)
        .concatWith(Mono.error(new IllegalStateException()))
        .switchOnError(Mono.just(0))
        .subscribe(System.out::println);
```

* 出现错误时根据异常类型来选择流

```java
Flux.just(1, 2)
        .concatWith(Mono.error(new IllegalArgumentException()))
        .onErrorResumeWith(e -> {
            if (e instanceof IllegalStateException) {
                return Mono.just(0);
            } else if (e instanceof IllegalArgumentException) {
                return Mono.just(-1);
            }
            return Mono.empty();
        })
        .subscribe(System.out::println);
```

* 使用 retry 操作符进行重试

```java
Flux.just(1, 2)
        .concatWith(Mono.error(new IllegalStateException()))
        .retry(1)
        .subscribe(System.out::println);
```

### 调度器

调度器（Scheduler）可以指定以上各类操作执行的方式和所在的线程

* 当前线程，通过 Schedulers.immediate() 方法来创建。
* 单一的可复用的线程，通过 Schedulers.single() 方法来创建。
* 使用弹性的线程池，通过 Schedulers.elastic() 方法来创建。线程池中的线程是可以复用的。当所需要时，新的线程会被创建。如果一个线程闲置太长时间，则会被销毁。该调度器适用于 I/O 操作相关的流的处理。
* 使用对并行操作优化的线程池，通过 Schedulers.parallel() 方法来创建。其中的线程数量取决于 CPU 的核的数量。该调度器适用于计算密集型的流的处理。
* 使用支持任务调度的调度器，通过 Schedulers.timer() 方法来创建。
* 从已有的 ExecutorService 对象中创建调度器，通过 Schedulers.fromExecutorService() 方法来创建。

通过 publishOn() 和 subscribeOn() 方法可以切换执行操作的调度器。其中 publishOn() 方法切换的是操作符的执行方式，而 subscribeOn() 方法切换的是产生流中元素时的执行方式

Example:

```java
Flux.create(sink -> {
    sink.next(Thread.currentThread().getName());
    sink.complete();
})
	.publishOn(Schedulers.single())
	.map(x -> String.format("[%s] %s", Thread.currentThread().getName(), x))
	.publishOn(Schedulers.elastic())
	.map(x -> String.format("[%s] %s", Thread.currentThread().getName(), x))
	.subscribeOn(Schedulers.parallel())
	.toStream()
	.forEach(System.out::println);
```

### 测试

|测试用方法|含义|
|---|---|
|expectNext|判断下一个值|
|assertNext|使用Lambda表达式和断言来判断值|
|expectNextCount|流中数的个数|
|expectSubscription|开始订阅，配合withVirtualTime使用|
|expectNoEvent|期望指定时间内流中无数据产生|
|thenAwait|等待指定时间，这段时间订阅正常发生(和expectNoEvent的区别)|
|expectComplete|期望从流中获取完成信号|
|varifyComplete|从流中获取完成信号，并进行测试|
|thenRequet|用于背压测试，向流中请求处理的元素个数。使用thenRequest后必须要消费(expectNext或其他方式)，才能继续请求|
|thenCancel|用于背压测试，退出、不再消费流中的元素|

* 使用 StepVerifier 验证流中的元素

expectNext() 方法用来声明测试时所期待的流中的下一个元素的值，而 verifyComplete() 方法则验证流是否正常结束，类似的方法还有 verifyError() 来验证流由于错误而终止

```java
StepVerifier.create(Flux.just("a", "b"))
        .expectNext("a")
        .expectNext("b")
        .verifyComplete();
```

* 操作测试时间

通过 StepVerifier.withVirtualTime() 方法可以创建出使用虚拟时钟的 StepVerifier。通过 thenAwait(Duration) 方法可以让虚拟时钟前进

```java
StepVerifier.withVirtualTime(() -> Flux.interval(Duration.ofHours(4), Duration.ofDays(1)).take(2))
        .expectSubscription()
        .expectNoEvent(Duration.ofHours(4))
        .expectNext(0L)
        .thenAwait(Duration.ofDays(1))
        .expectNext(1L)
        .verifyComplete();
```

解释：需要验证的流中包含两个产生间隔为一天的元素，并且第一个元素的产生延迟是 4 个小时。在通过 StepVerifier.withVirtualTime() 方法包装流之后，expectNoEvent() 方法用来验证在 4 个小时之内没有任何消息产生，然后验证第一个元素 0 产生；接着 thenAwait() 方法来让虚拟时钟前进一天，然后验证第二个元素 1 产生；最后验证流正常结束。

* 使用 TestPublisher 创建测试所用的流

TestPublisher 的作用在于可以控制流中元素的产生

```java
final TestPublisher<String> testPublisher = TestPublisher.create();
testPublisher.next("a");
testPublisher.next("b");
testPublisher.complete();
 
StepVerifier.create(testPublisher)
        .expectNext("a")
        .expectNext("b")
        .expectComplete();
```

## 总结

参考：

* [使用 Reactor 进行反应式编程](https://www.ibm.com/developerworks/cn/java/j-cn-with-reactor-response-encode/index.html)
* [Reactor 3 学习笔记](https://www.cnblogs.com/yjmyzz/p/reactor-tutorial-1.html)
* [什么是响应式编程——响应式 Spring 的道法术器 ](http://blog.51cto.com/liukang/2090170)