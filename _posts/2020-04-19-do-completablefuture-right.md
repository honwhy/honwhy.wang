---
layout: post
category: java
comments: true
title: 正确地使用CompletableFuture
tags: future 
---
* content
{:toc}

## 概述
多线程编程并不是一项容易的事情，但是多线程有利于提高效率，减少耗时，在一个方法内使用多线程有助于提高这个方法的qps如此。在Java中编写多线程有很多代码可以参考的，朴素一点的做法是使用Future、FutureTask，而本文关注的是另外一种做法CompletableFuture，相比Future使用起来更加方便，但是不小心的话也会出毛病，本文会对此给出解决方法和优化思路，勉之共同进步。

## CompletableFuture的重要api方法
使用CompletableFuture会涉及到supplyAsync、allOf、get、join等方法，一个例子如下
```java
@Test
public void test() throws InterruptedException {
    ThreadPoolExecutor executor = ThreadPoolExecutorBuilder.build(2, 4, 100);
    List<CompletableFuture<?>> fs = new ArrayList<>(size);
    List<CompletableFuture<?>> others = new ArrayList<>();
    for (int i = 0; i < size; i++) {
        MyRandomSleepTask task = new MyRandomSleepTask(ai.incrementAndGet());
        CompletableFuture<?> cf = CompletableFuture.supplyAsync(() -> {
            task.run();
            return null;
        }, executor);
        if (i % 2 == 0) {
            System.out.println("pick task-" + (i + 1));
            fs.add(cf);
        } else {
            others.add(cf);
        }
    }

    CompletableFuture<?>[] fa = new CompletableFuture[0];
    try {
        CompletableFuture.allOf(fs.toArray(fa)).get(2500, TimeUnit.MILLISECONDS);
    } catch (InterruptedException | ExecutionException | TimeoutException e) {
        e.printStackTrace();
    }
    CompletableFuture.allOf(others.toArray(fa)).join();
}
```
使用supplyAsync发起异步执行任务，使用allOf聚合所有的Future，再选择get或者join等待所有任务执行完成，get方法还可以使用超时参数。在实际项目中应用超时等待的需求还是大量存在的，一方面可以控制接口的响应时间，一方面防止线程池的工作线程繁忙被长耗时的任务拖垮，用快速失败的方式换接口的稳定保障。

## 使用get超时等待可能会出现问题
由于项目实践中发现了很多get方法的Exception，即等待超时的TimeoutException，这不得不引起我们的注意，关于线程池的参数如何正确配置，关于get方法超时后会做什么；1）根据请求量，简单地数学计算，增加了核心线程数，调整了线程池队列的长度，我们的目的是当队列满的时候可以使用更多线程工作；2）关于get的怀疑，发生高并发时一次请求的Runnable应该是分散插入到线程池队列里的（假设已经出现了排队的请求），那么get先于Runnable超时结束了，它关联的Runnable是什么状态呢，会不会继续占据线程池队列呢

调参并不容易的，经过调整有一定的改善，但是并发请求量一旦提高了之后，导致了几乎所有的请求都出现了get超时，因此，大胆猜测get超时后，它关联的Runnable还是保留在队列里面，而这些Runnable的执行都可以看作是无意义的耗时，这些耗时通通会影响到其他请求的get等待。我们并不确定请求的先来后到和它们关联的Runnable的执行顺序的关系，不管是谁的Runnable出现了长耗时，惩罚的是所有请求。

这里做了一个实验，如上文的*test*方法，打印日志如下，
```log
pick task-1
pick task-3
pick task-5
pick task-6
pick task-10
pick task-14
[pool-1-thread-1]MyRandomSleepTask-1 will Sleep(211ms)...
[pool-1-thread-2]MyRandomSleepTask-2 will Sleep(530ms)...
[pool-1-thread-1]MyRandomSleepTask-3 will Sleep(1622ms)...
[pool-1-thread-2]MyRandomSleepTask-4 will Sleep(174ms)...
[pool-1-thread-2]MyRandomSleepTask-5 will Sleep(274ms)...
[pool-1-thread-2]MyRandomSleepTask-6 will Sleep(1669ms)...
[pool-1-thread-1]MyRandomSleepTask-7 will Sleep(1581ms)...
java.util.concurrent.TimeoutException
	at java.util.concurrent.CompletableFuture.timedGet(CompletableFuture.java:1784)
	at java.util.concurrent.CompletableFuture.get(CompletableFuture.java:1928)
	at com.github.honwhy.CompletableFutureTest.test(CompletableFutureTest.java:59)
	...
[pool-1-thread-2]MyRandomSleepTask-8 will Sleep(1248ms)...
[pool-1-thread-1]MyRandomSleepTask-9 will Sleep(1185ms)...
[pool-1-thread-2]MyRandomSleepTask-10 will Sleep(1870ms)...
[pool-1-thread-1]MyRandomSleepTask-11 will Sleep(1990ms)...
[pool-1-thread-2]MyRandomSleepTask-12 will Sleep(588ms)...
[pool-1-thread-2]MyRandomSleepTask-13 will Sleep(87ms)...
[pool-1-thread-2]MyRandomSleepTask-14 will Sleep(868ms)...
[pool-1-thread-1]MyRandomSleepTask-15 will Sleep(321ms)...
```
get方法需要等待task-1,3,5,6,10,14完成，但是执行到task-6时就因为超时结束了，但是从日志中还可以看到task-10,14都执行了，这里验证了我们的猜测。

## 解决多余的Runnable被执行的问题
在网上也找到了关于这个get超时后没有清理Runnable的问题，
>http://arganzheng.life/writing-asynchronous-code-with-completablefuture.html   
>默认情况下，allOf 会等待所有的任务都完成，即使其中有一个失败了，也不会影响其他任务继续执行。
但是大部分情况下，一个任务的失败，往往意味着整个任务的失败，继续执行完剩余的任务意义并不大。
在 谷歌的 Guava 的 allAsList 如果其中某个任务失败整个任务就会取消执行:

引入Guava后解决方案并没有解决这个问题（未继续验证），最后采用的方法也很简单，
```java
try {
    CompletableFuture.allOf(fs.toArray(fa)).get(2500, TimeUnit.MILLISECONDS);
} catch (InterruptedException | ExecutionException | TimeoutException e) {
    e.printStackTrace();
    if (e instanceof TimeoutException) {
        fs.forEach(f -> f.cancel(true));
    }
}
```
出现异常时，在catch中取消这些future。打印日志，
```log
pick task-1
pick task-3
pick task-5
pick task-6
pick task-10
pick task-14
[pool-1-thread-1]MyRandomSleepTask-1 will Sleep(1034ms)...
[pool-1-thread-2]MyRandomSleepTask-2 will Sleep(1409ms)...
[pool-1-thread-1]MyRandomSleepTask-3 will Sleep(954ms)...
[pool-1-thread-2]MyRandomSleepTask-4 will Sleep(42ms)...
[pool-1-thread-2]MyRandomSleepTask-5 will Sleep(174ms)...
[pool-1-thread-2]MyRandomSleepTask-6 will Sleep(359ms)...
[pool-1-thread-2]MyRandomSleepTask-7 will Sleep(851ms)...
[pool-1-thread-1]MyRandomSleepTask-8 will Sleep(973ms)...
java.util.concurrent.TimeoutException
	at java.util.concurrent.CompletableFuture.timedGet(CompletableFuture.java:1784)
	at java.util.concurrent.CompletableFuture.get(CompletableFuture.java:1928)
	at com.github.honwhy.CompletableFutureTest.test4(CompletableFutureTest.java:157)
    ...
[pool-1-thread-2]MyRandomSleepTask-9 will Sleep(836ms)...
[pool-1-thread-1]MyRandomSleepTask-11 will Sleep(181ms)...
[pool-1-thread-1]MyRandomSleepTask-12 will Sleep(1892ms)...
[pool-1-thread-2]MyRandomSleepTask-13 will Sleep(1753ms)...
[pool-1-thread-1]MyRandomSleepTask-15 will Sleep(1204ms)...
```
可以看到task-6之后get超时了，后续的task-10,14都没有执行了。

## 使用缓存加速响应
使用缓存当然是一个提高响应速度的一个好办法了，是不是在这个场景中也适用呢。实际上也是适用的，假设在业务的高峰期，用户的请求参数几乎没有改变的，不同用户的请求也可能存在参数相同的情况，偶尔的方法体内出现了get超时，用户获取了空的响应，再不断地重试，针对这些增加缓存肯定是有助于提高响应效率的。本文设计的解决方法是使用Guava Cache，它可以控制Cache的大小，命中的cache还继续延长存活时间，还可以使用弱引用对gc友好等，属于堆内缓存，相比Redis等外部缓存或许Guava Cache更适合。

```java
private static final Cache<String, String> cache = CacheBuilder.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(3, TimeUnit.SECONDS)
        .expireAfterAccess(3, TimeUnit.SECONDS)
        .weakKeys()
        .weakValues()
        .build();
```
模拟了http请求的缓存保存和读取，
```java
public class HttpRequestTask implements Callable<String> {
    @Override
    public String call() {
        try {
            if (Thread.currentThread().isInterrupted()) {
                System.out.printf("[" + Thread.currentThread().getName() + "]" + this.getClass().getSimpleName() + "-" + this.id + " is interrupted");
                return null;
            }
            long left = score % 600;
            try {
                System.out.println("[" + Thread.currentThread().getName() + "]" + this.getClass().getSimpleName() + "-" + this.id + " will Sleep(" + left + "ms)...");
                TimeUnit.MILLISECONDS.sleep(left);
            } catch (InterruptedException e) {
                //ignore
                //Thread.currentThread().interrupt();
            }
            String result = cache.getIfPresent(query);
            if (result != null) {
                System.err.println("get result from cache: key=" + query);
                return result;
            }
            String requestUrl = "https://cn.bing.com/search?q=" + query;
            System.out.println("[" + Thread.currentThread().getName() + "]" + this.getClass().getSimpleName() + "-" + this.id + " request (" + requestUrl + ")...");
            result = doRequest(requestUrl);
            if (result != null) {
                cache.put(query, result);
            }
            return result;
        } catch (Exception e) {
        }
        return null;
    }
    protected String doRequest(String requestUrl) throws Exception {
        return MyHttpClient.get(requestUrl);
    }    
}
```
测试方法，和上文的test区别在于`MyRandomSleepTask`换成了`HttpRequestTask`
```java
for (int i = 0; i < size; i++) {
    HttpRequestTask task = new HttpRequestTask(ai.incrementAndGet(), randomChar());
    CompletableFuture<?> cf = CompletableFuture.supplyAsync(() -> {
        return task.call();
    }, executor);
    if (i % 2 == 0) {
        System.out.println("pick task-" + (i + 1));
        fs.add(cf);
    } else {
        others.add(cf);
    }
}
```

## 继续异步
如果方法体有需要多线程http请求外部接口的话，是否可以使用nio的方式，将执行线程等待http响应时将其挂起让出cpu，或者去执行其他任务，设想还算美好。
使用异步的http客户端只需要引入`AsyncHttpClient`，
```java
public class AsyncHttpRequestTask extends HttpRequestTask {

    private static AsyncHttpClient asyncHttpClient = asyncHttpClient(Dsl.config().setConnectTimeout(2000).setReadTimeout(2000));

    public AsyncHttpRequestTask(int id, String query) {
        super(id, query);
    }

    @Override
    protected String doRequest(String requestUrl) throws Exception {
        Future<Response> whenResponse = asyncHttpClient.prepareGet("http://www.example.com/").execute();
        Response response = whenResponse.get();
        if (response != null) {
            return response.getResponseBody(StandardCharsets.UTF_8);
        }
        return null;
    }

    public static void closeClient() {
        try {
            asyncHttpClient.close();
        } catch (IOException e) {
            //
        }
    }
}
```
对测试方法做调整，
```java
for (int i = 0; i < size; i++) {
    AsyncHttpRequestTask task = new AsyncHttpRequestTask(ai.incrementAndGet(), randomChar());
    CompletableFuture<?> cf = CompletableFuture.supplyAsync(() -> {
        return task.call();
    }, executor);
    if (i % 2 == 0) {
        System.out.println("pick task-" + (i + 1));
        fs.add(cf);
    } else {
        others.add(cf);
    }
}
```
实际的测试效果并不理想，`AsyncHttpRequestTask`执行结果不如`HttpRequestTask`。

## 总结
至此，本文关于CompletableFuture的正确使用及优化方案都所有提及，正确的使用方案一定是符合业务需求的，优化的目的和手段也是和业务需求一致的，在追求多线程编程正确及效率上还有很多的期待。   
参考代码，[fgc](https://github.com/honwhy/fgc)
