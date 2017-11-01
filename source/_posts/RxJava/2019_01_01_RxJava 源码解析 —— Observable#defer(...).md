title: RxJava 源码解析 —— Observable#defer(...)
date: 2019-01-01
tags:
categories: RxJava
permalink: RxJava/observable-defer

-------

摘要: 原创出处 http://www.iocoder.cn/RxJava/observable-defer/ 「芋道源码」欢迎转载，保留摘要，谢谢！

**本文主要基于 RxJava 1.2.X 版本**  

本系列写作目的，为了辅助 Hystrix 的理解，因此会较为零散与琐碎，望见谅见谅。

-------

在一些业务场景下，我们需要 Observable 是**动态**的，例如说，[《Hystrix 源码解析 —— 执行结果缓存》](http://www.iocoder.cn/Hystrix/command-execute-result-cache/?self) 分享的缓存 Observable ，无法在创建 Observable 阶段就知道是否有缓存，通过 `Observable#defer(...)` 方法，声明**动态**的 Observable 。示例代码如下：

```Java
public static void main(String[] args) {
   Observable.defer(new Func0<Observable<String>>() { // #defer(...)
       @Override
       public Observable<String> call() {
           String name = Math.random() > 0.5 ? "小明" : "小贾"; // 随机名字
           return Observable.just(name);
       }
   }).subscribe(new Action1<String>() { // #subscribe(...)
       @Override
       public void call(String s) {
           System.out.println(s);
       }
   });
}
```

-------

`Observable#defer(...)` 方法，代码如下：

```Java
// Observable.java
public static <T> Observable<T> defer(Func0<Observable<T>> observableFactory) {
   return create(new OnSubscribeDefer<T>(observableFactory));
}

public static <T> Observable<T> create(OnSubscribe<T> f) {
   return new Observable<T>(RxJavaHooks.onCreate(f));
}
```

* 使用传入 `observableFactory` 参数，生成**动态**的 Observable 。  

-------

OnSubscribeDefer 类，代码如下：

```Java
public final class OnSubscribeDefer<T> implements OnSubscribe<T> {
    final Func0<? extends Observable<? extends T>> observableFactory;

    public OnSubscribeDefer(Func0<? extends Observable<? extends T>> observableFactory) {
        this.observableFactory = observableFactory;
    }

    @Override
    public void call(final Subscriber<? super T> s) {
        Observable<? extends T> o;
        try {
            o = observableFactory.call();
        } catch (Throwable t) {
            Exceptions.throwOrReport(t, s);
            return;
        }
        o.unsafeSubscribe(Subscribers.wrap(s));
    }

}
```

* 在 `Observable#subscribe(...)` 方法调用时，调用 `OnSubscribeDefer#call(...)` 方法 ：
    * 调用 `Func0#call()` 方法，创建**动态**的 Observable 。
    * 调用 `Observable#unsafeSubscribe(...)` 方法，**继续订阅逻辑**。

