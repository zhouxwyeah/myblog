---
# 常用定义
title: "Reactive Steam笔记"           # 标题
date: 2019-04-05T10:01:23+08:00    # 创建时间
lastmod: 2019-04-05T10:01:23+08:00 # 最后修改时间
draft: false                       # 是否是草稿？
tags: ["reactive", "webflux"]  # 标签
categories: ["tech"]              # 分类
author: "zhouxwyeah"                  # 作者
description: "spring in action 5th reactive read note"
# 用户自定义
# 你可以选择 关闭(false) 或者打开(true) 以下选项
comment: true   # 关闭评论
toc: true       # 关闭文章目录
# 你同样可以自定义文章的版权规则
reward: false	 # 关闭打赏
mathjax: true    # 打开 mathjax
---

本文是对于reactor的学习笔记.

#### RxJava

最早了解反应式编程，更多的是用在客户端或者WEB，对应事件和数据流的响应，听起来很自然，也合情合理。后面netflix全家桶的流行，发现Rxjava的存在，也逐步了解到[ReactiveX](http://reactivex.io/intro.html)范式，提到的Reactivex一个基于观察者模式的扩展，对于被观察者提供的异步、事件驱动的编程库

> ReactiveX is a library for composing asynchronous and event-based programs by using observable sequences.
>
> It extends [the observer pattern](http://en.wikipedia.org/wiki/Observer_pattern) to support sequences of data and/or events and adds operators that allow you to compose sequences together declaratively while abstracting away concerns about things like low-level threading, synchronization, thread-safety, concurrent data structures, and non-blocking I/O.

其中提到了对于Observable和Iterable最大的差别，是数据的获取方式，reactive 是push,能做到背压控制，而传统的迭代，是pull，就是尽力获取数据。

具体的Rxjava的操作API，和java8的steam很像，如果熟悉java steam的API，对于Rxjava也不会陌生，具体可以查看[官方文档](<http://reactivex.io/documentation/observable.html>)

反应式编程是想解决传统命令式编程下，操作同步阻塞下导致的高负荷、高延迟的问题，相对于其他领域解决阻塞的问题，其实做法都差不多，无非是非阻塞、eventloop,消息驱动的方式，而在传统的命令式的编程其实也提供了相应的特性去解决，比如java的Future,CompleteableFuture，但是这种方式，背后都是一个类似ExecuteService的线程池，线程同样会自己被阻塞住，而且线程管理和并发编程也会大大提升的编程的复杂性。

反应式编程就是一种解决方式，它提供了基于流，组织pipline处理的方式，提供了很自然的描述方式来表达行为，而不用关注背后的线程、调度的复杂度(理想情况)(类似go routine)。可查看[反应式宣言](https://www.reactivemanifesto.org/zh-CN)

#### Reactive Steam

[Reactive Steam](<http://www.reactive-streams.org/>)正是由netflix、Light-bend、Pivoal提出的一种反应式编程规范，目标是为异步非阻塞数据流处理，提供一种通用的处理方式。而在JDK 9中[java.util.concurrent.Flow](<https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.html>)已经是该规范的一种实现，也就是在JDK9中，我们就能使用反应式编程了。

规范本身的概念很简洁，其中就4个接口，从接口名称我们可以看出，他们和观察者模式由一定的渊源：

Publisher<T> 发布者，数据的生产方，支持订阅消费者

```java
public interface Publisher<T> {
  void subscribe(Subscriber<? super T> subscriber);
  }
```

Subscriber<T>     数据消费者，可以处理生产者发布过来的数据，绑定处理行为

```java
public interface Subscriber<T> {
  void onSubscribe(Subscription sub);
  void onNext(T item);
  void onError(Throwable ex);
  void onComplete();
  }
```

Subscription  订阅事件，如果消费者订阅数据，会收到一个subscription事件

```java
public interface Subscription {
  void request(long n);
  void cancel();
  } 
```

Processor<T,R>  生成消费合体

```java
public interface Processor<T, R>
                 extends Subscriber<T>, Publisher<R> {}
```

#### Reactor

[Reactor](<https://projectreactor.io/>)是Pivotal的一种reactive steam的一种实现,整体API和 Rxjava2十分接近

Reactor中，提供两种反应式对象

- Mono<T>

  单个异步数据，类似Rxjava中的Single<T>,或者是java8中 CompleteableFuture<T>

- Flux<T>

  异步数据流，类似Rxjava中的Observable<T>,或者是java8中 Steam<T>

和Rxjava和Java8 steam的操作类似，基于数据流的行为定义可分为四类。 创建、组合、转换、逻辑运算操作四类，下面我们分别对应这四类操作逐个说明：

##### 创建

- just() 使用just()方法创建Mono或者Flux对象

```java
@Test
public void firstMono() {
   Mono.just("Huajuan").map(s -> s.toUpperCase()).map(s -> "hello " + s + " !").subscribe(System.out::println);
}

@Test
public void createAFlux_just() {
   Flux<String> flux = Flux.just("Apple", "Pie", "Orange");
   StepVerifier.create(flux).expectNext("Apple").expectNext("Pie").expectNext("Orange");
}
```

我们看到，可以使用.subscribe()方法增加订阅者。同时我们看到Reactor提供了StepVerifier来帮助你完成单元测试，如果没有验证器，我们只能通过debug来查看行为了，这点比较贴心。

- Flux还可以从数组中创建
- 从Iterable中创建
- 从Steam中创建

```java
@Test
public void createFrom_Containers() {
   // by array  iterable    steam
   String[] fruits = new String[] {"Apple", "Pie", "Orange"};

   Flux<String> flux = Flux.fromArray(fruits);

   StepVerifier.create(flux).expectNext("Apple").expectNext("Pie").expectNext("Orange").verifyComplete();

   List<String> fruitList = Arrays.asList("Apple", "Pie", "Orange");

   Flux<String> listFlux = Flux.fromIterable(fruitList);

   StepVerifier.create(listFlux).expectNext("Apple").expectNext("Pie").expectNext("Orange").verifyComplete();

   Stream<String> steam = Stream.of("Apple", "Pie", "Orange");

   Flux<String> steamFlux = Flux.fromStream(steam);

   StepVerifier.create(steamFlux).expectNext("Apple").expectNext("Pie").expectNext("Orange").verifyComplete();

}
```

- 支持生成Flux数据，可以指定区间，以及周期

```java
@Test
public void createFlux_Range() {
   Flux<Integer> integerFlux = Flux.range(1, 5);
   StepVerifier.create(integerFlux).expectNext(1).expectNext(2).expectNext(3).expectNext(4).expectNext(5)
         .verifyComplete();
}

@Test
public void createFlux_Interval() {
   // take limit
   Flux<Long> integerFlux = Flux.interval(Duration.ofSeconds(1)).take(5);
   StepVerifier.create(integerFlux).expectNext(0L).expectNext(1L).expectNext(2L).expectNext(3L).expectNext(4L)
         .verifyComplete();
}
```

##### 组合  

当你操作多个Mono或者Flux的时候，你经常会有组合的需求

- merge 将两个反应式数据流合为一个，没有其他操作

  ![image-20190406003203019](../images/image-20190406003203019.png)

  ```java
  @Test
  public void mergeFlux() {
     Flux<String> fruitFlux = Flux.just("Apple", "Pie", "Orange").delayElements(Duration.ofMillis(500));
     Flux<String> sportFlux = Flux.just("Football", "BasketBall", "Swimming")
           .delaySubscription(Duration.ofMillis(250)).delayElements(Duration.ofMillis(500));
     //
     Flux<String> mergeFlux = fruitFlux.mergeWith(sportFlux);
     StepVerifier.create(mergeFlux).expectNext("Apple").expectNext("Football").expectNext("Pie")
           .expectNext("BasketBall").expectNext("Orange")
           .expectNext("Swimming").verifyComplete();
  }
  ```

- zip 将反应式数据流合并，每个流的数据各取一个，合并为一个对象

![image-20190406003354629](../images/image-20190406003354629.png)

```java
@Test
public void mergeZip() {
   Flux<String> fruitFlux = Flux.just("Apple", "Pie", "Orange");
   Flux<String> sportFlux = Flux.just("Football", "BasketBall", "Swimming");
   //
   Flux<Tuple2<String, String>> zipFlux = Flux.zip(fruitFlux, sportFlux);
   StepVerifier.create(zipFlux).expectNextMatches(p -> p.getT1().equals("Apple") && p.getT2().equals("Football"))
         .expectNextMatches(p -> p.getT1().equals("Pie") && p.getT2().equals("BasketBall"))
         .expectNextMatches(p -> p.getT1().equals("Orange") && p.getT2().equals("Swimming")).verifyComplete();
}
```

- zipWith 指定函数zip 每个流的数据各取一个，通过函数处理返回为一个对象

![image-20190406003546835](../images/image-20190406003546835.png)

```java
@Test
public void mergeZipWithFunction() {
   Flux<String> fruitFlux = Flux.just("Apple", "Pie", "Orange");
   Flux<String> sportFlux = Flux.just("Football", "BasketBall", "Swimming");
   //
   Flux<String> zipFlux = Flux.zip(fruitFlux, sportFlux, (c, p) -> c + " like " + p);
   StepVerifier.create(zipFlux)
         .expectNext("Apple like Football")
         .expectNext("Pie like BasketBall")
         .expectNext("Orange like Swimming").verifyComplete();
}
```

- first 取最快发射数据的那个Flux

![image-20190406003832104](../images/image-20190406003832104.png)

```java
@Test
public void firstFlux() {
   Flux<String> slowFlux = Flux.just("tortoise", "snail", "sloth")
         .delaySubscription(Duration.ofMillis(100));
   Flux<String> fastFlux = Flux.just("hare", "cheetah", "squirrel");
   Flux<String> firstFlux = Flux.first(slowFlux, fastFlux);
   StepVerifier.create(firstFlux)
         .expectNext("hare")
         .expectNext("cheetah")
         .expectNext("squirrel")
         .verifyComplete();
}
```

##### 转换和过滤

和steam类似，有时候我们需要对于数据流进行筛选和转换(一般叫map)，也有Steam类似的操作

 - skip 跳过，可以指定时间

   ![image-20190406004634082](../images/image-20190406004634082.png)

```java
@Test
public void skip() {
   Flux<String> sportFlux = Flux.just("Football", "BasketBall", "Swimming", "Running").skip(3);
   StepVerifier.create(sportFlux)
         .expectNext("Running").verifyComplete();
}
```

 - take   skip的反操作，获取几个

![image-20190406004840043](../images/image-20190406004840043.png)

```java
@Test
public void take() {
   Flux<String> sportFlux = Flux.just("Football", "BasketBall", "Swimming", "Running").take(1);
   StepVerifier.create(sportFlux)
         .expectNext("Football").verifyComplete();
}
```

 - filter 指定过滤器predicate

![image-20190406004937787](../images/image-20190406004937787.png)

```java
@Test
public void filter() {
   Flux<String> sportFlux = Flux.just("Football", "BasketBall", "Swimming", "Running")
         .filter(c -> c.contains("ball"));
   StepVerifier.create(sportFlux)
         .expectNext("Football").verifyComplete();
}
```

 - distinct  去重

![image-20190406005054261](../images/image-20190406005054261.png)

```java
@Test
public void distinct() {
   Flux<String> sportFlux = Flux.just("Football", "BasketBall", "Football", "Swimming", "Running").distinct();
   StepVerifier.create(sportFlux)
         .expectNext("Football").expectNext("BasketBall").expectNext("Swimming").expectNext("Running")
         .verifyComplete();
}
```

 - map 对每个元素进行映射,map这个动作是同步的，当生产者生成的时候，同步去做的

![image-20190406005136560](../images/image-20190406005136560.png)

```java
@Test
public void map() {
   // sync
   Flux<String> sportFlux = Flux.just("Football", "BasketBall");

   Flux<String> sportFluxMap = sportFlux.map(n -> {
      return "hello" + n;
   });
   StepVerifier.create(sportFluxMap).expectNext("helloFootball").expectNext("helloBasketBall").verifyComplete();
}
```

 - flatmap 对每个元素进行映射,flatmap这个动作是异步的，所以不能用onnext去校验。

![image-20190406005822692](../images/image-20190406005822692.png)

​	

```java
@Test
public void flatMap() {
   // async parallel by call fixed pool size  cpu cores
   Flux<Player> sportFlux = Flux.just("Football CR", "BasketBall JD", "Swimming FP", "Running BT")
         .flatMap(n -> Mono.just(n)
               .map(p -> {
                  String[] str = p.split(" ");
                  return new Player(str[0], str[1]);
               }).subscribeOn(Schedulers.parallel()));

   List<Player> players = Arrays
         .asList(new Player("Football", "CR"), new Player("BasketBall", "JD"), new Player("Swimming", "FP"), new Player("Running", "BT"));

   StepVerifier.create(sportFlux)
         .expectNextMatches(p -> players.contains(p))
         .expectNextMatches(p -> players.contains(p))
         .expectNextMatches(p -> players.contains(p))
         .expectNextMatches(p -> players.contains(p))
         .verifyComplete();
}
```

异步由scheduler调度，有以下几种并发模式：

​	![image-20190406005929351](../images/image-20190406005929351.png)



##### 逻辑运算

- all

判断所有元素是否满足条件

![image-20190406010548224](../images/image-20190406010548224.png)

```java
@Test
public void all() {
   Flux<String> flux = Flux.just("ab", "ac", "ad", "ak");
   Mono<Boolean> hasAMono = flux.all(p -> p.contains("a"));
   StepVerifier.create(hasAMono).expectNext(true).verifyComplete();

   Mono<Boolean> hasKMono = flux.all(p -> p.contains("k"));
   StepVerifier.create(hasKMono).expectNext(false).verifyComplete();

}
```

- any

判断是否有元素符合条件

![image-20190406010632550](../images/image-20190406010632550.png)

```java
@Test
public void any() {
   Flux<String> flux = Flux.just("ab", "ac", "ad", "ak");
   Mono<Boolean> hasKany = flux.any(p -> p.contains("k"));
   StepVerifier.create(hasKany).expectNext(true).verifyComplete();
}
```

 

Reactor的操作很多，也没法一一列举，建议多参考[官方文档](<https://projectreactor.io/docs/core/release/reference/>)

参考资料

- Spring in action 5th
- [Get Reactive With Project Reactor and Spring 5](https://speakerdeck.com/olehdokuka/get-reactive-with-project-reactor-and-spring-5?slide=70)