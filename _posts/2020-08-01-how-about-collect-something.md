---
layout: post
category: java
comments: true
title: 关于聚合和多线程的处理套路
tags: collect
---
* content
{:toc}

## 概述
无差别地请求多个外部接口并聚合所有请求结果，应该有属于它自己的套路，应该将所有多线程的操作屏蔽之，我们只关心参数和结果。因此，应该抛弃Callable/FutureTask/Future等这些手工模式，这些代码应该交给框架来实现。

## 手工模式
何为手工模式，我们以Callable为例设计请求外部的接口，可能像下面这样子，参数是NumberParam，两个外部接口分别是IntToStringCallable和DoubleToStringCallable，
```java
    class IntToStringCallable implements Callable<String> {
        private final NumberParam param;

        IntToStringCallable(NumberParam numberParam) {
            this.param = numberParam;
        }
        @Override
        public String call() {
            return Integer.toHexString(param.getAge());
        }
    }

    class DoubleToStringCallable implements Callable<String> {

        private final NumberParam param;

        DoubleToStringCallable(NumberParam numberParam) {
            this.param = numberParam;
        }
        @Override
        public String call() {
            return Double.toHexString(param.getMoney());
        }
    }
```
如果采用FutureTask的方式多线程执行这两个接口，可能是这样子的，
```java
        FutureTask<String> r1 = new FutureTask<>(new IntToStringCallable(numberParam));
        new Thread(r1).start();
        FutureTask<String> r2 = new FutureTask<>(new DoubleToStringCallable(numberParam));
        new Thread(r2).start();
        try {
            List<String> ret = new ArrayList<>();
            ret.add(r1.get());
            ret.add(r2.get());
            log.info("ret=" + ret);
        } catch (Exception ignore) {

        }
```
需要首先构造FutureTask，然后使用Thread比较原始的api去执行，当然还可以再简化一下，比如使用Future方式，
```java
        ExecutorService threadPool = Executors.newFixedThreadPool(2);
        Future<String> r1 = threadPool.submit(new IntToStringCallable(numberParam));
        Future<String> r2 = threadPool.submit(new DoubleToStringCallable(numberParam));
        try {
            List<String> ret = new ArrayList<>();
            ret.add(r1.get());
            ret.add(r2.get());
            log.info("ret=" + ret);
        } catch (Exception ignore) {

        }
```
我相信这是一种普遍常见的做法了。这里没有必要继续评论这些做法的问题了。

## Java 8之后
Java 8之后有了更加方便的异步编程方式了，不用再辛苦地去写Callable的，一句话就可以表达Callable+FutureTask/...，
```java
CompletableFuture<String> pf = CompletableFuture.supplyAsync(() -> new IntToStringCallable(numberParam).call());
```
改造之前的做法结果可能就是这个样子了，
```java
        CompletableFuture<String> r1 = CompletableFuture.supplyAsync(() -> new IntToStringCallable(numberParam).call());
        CompletableFuture<String> r2 = CompletableFuture.supplyAsync(() -> new DoubleToStringCallable(numberParam).call());
        try {
            List<String> ret = new ArrayList<>();
            ret.add(r1.get());
            ret.add(r2.get());
            log.info("ret=" + ret);
        } catch (Exception ignore) {

        }
```
其实可以看出来，这个时候我们不一定需要一个Callable了，提供异步的能力是supplyAsync来完成的，我们只需要正常的入参出参的普通方法就可以了。

## Java 8之后再之后
Java 8之后的异步编程方式确实简单了很多，但是在我们的业务代码中还是出现了和异步编程相关的无关业务逻辑的事情，可否继续简化呢。本案的设计灵感来自同样Java 8的优秀设计——ParallelStream，举个简单的例子，

```java
Arrays.asList("a", "b", "c").parallelStream().map(String::toUpperCase).collect(Collectors.toList());
```
异步及多线程是ParallelStream来完成的，用户只需要完成String::toUpperCase部分。

本案的设计主要有三个interface来实现，分别是，
```java
public interface MyProvider<T,V> {
    T provide(V v);
}
public interface MyCollector<T> {

    void collectList(T t);

    List<T> retList();
}
public interface MyStream<T,V> {

    List<T> toList(List<MyProvider<T,V>> providers, V v);
}
```
其实MyProvider表达是请求外部接口，MyStream表示一种类似ParallelStream的思想，一种内化异步多线程的操作模式，MyCollector属于内部设计api可以不暴露给用户；
一个改写上面的例子的例子，

```java
    @Test
    public void testStream() {
        MyProvider<String,NumberParam> p1 = new IntToStringProvider();
        MyProvider<String,NumberParam> p2 = new DoubleToStringProvider();
        List<MyProvider<String, NumberParam>> providers = Arrays.asList(p1, p2);
        MyStream<String, NumberParam> myStream = new CollectStringStream();

        List<String> strings = myStream.toList(providers, numberParam);
        log.info("ret=" + strings);
    }

```
在这个方法内一点异步编程的内容都没有的，用户只需要编程自己关心的逻辑即可，当然是要按照Provider的思路去写，这或许有一点心智负担。
这个CollectStringStream帮我们完成来一些脏活累活，
```java
    public List<String> toList(List<MyProvider<String, NumberParam>> myProviders, NumberParam param) {
        MyCollector<String> myCollector = new NoMeaningCollector();
        List<CompletableFuture<Void>> pfs = new ArrayList<>(myProviders.size());
        for (MyProvider<String, NumberParam> provider : myProviders) {
            CompletableFuture<Void> pf = CompletableFuture.runAsync(() -> myCollector.collectList(provider.provide(param)), executor);
            pfs.add(pf);
        }
        try {
            CompletableFuture.allOf(pfs.toArray(new CompletableFuture[0])).get(3, TimeUnit.SECONDS);
        } catch (Exception e) {
            if (e instanceof TimeoutException) {
                pfs.forEach(p -> {
                    if (!p.isDone()){
                        p.cancel(true);
                    }});
            }
        }
        return myCollector.retList();
    }
```
这样看起这个设计又不美了，但是如果有更多的外部接口需要调用，CollectStringStream就显得很有价值了，新加入再多的请求外部接口要改动的代码很少很少，所以这种思想我觉得是值得推广的。

## 总结
照例附上参考代码，不过值得思考的是我们如何像优秀的代码学习并运用到自己的项目中。  
参考代码，[java-toy](https://github.com/honwhy/java-toy)
