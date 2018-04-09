---
layout: post
category: java
comments: true
---

你可能不需要引入第三方依赖，或者自己重新造轮子就能在JDK中找到需要的API，当然并不能否认`commons-lang`和`guava`等开源项目的贡献，在平时的工作项目中几乎都会引入这些*语言扩展包*，直接使用他们也使得编程风格统一，而且还能够对低版本的JDK提供支持。
你可能已经使用到了`native`但是自己并不清楚；
你以为使用了一个优雅的方法，但其实JDK是在卖萌。
以下收集的代码片段可能会逐渐增加，也可能不会。

## java.util.Objects
`java.util.Objects`工具类，我觉得好用的几个方法
```
    public static boolean equals(Object var0, Object var1) {
        return var0 == var1 || var0 != null && var0.equals(var1);
    }
    public static int hashCode(Object var0) {
        return var0 != null ? var0.hashCode() : 0;
    }
    public static <T> T requireNonNull(T var0) {
        if (var0 == null) {
            throw new NullPointerException();
        } else {
            return var0;
        }
    }

    public static <T> T requireNonNull(T var0, String var1) {
        if (var0 == null) {
            throw new NullPointerException(var1);
        } else {
            return var0;
        }
    }        
```
除此之外还应该从`Objects`学习到编写工具类的正确的规范，
* 定义为final class
* 只定义一个无参的构造函数且抛出断言错误
* 工具方法都是静态方法
* 静态方法中只抛出unchecked异常

## java.lang.System
这个最早应该是在Hello World程序中见到的，推荐它的一个方法
```
    /**
     * Returns the same hash code for the given object as
     * would be returned by the default method hashCode(),
     * whether or not the given object's class overrides
     * hashCode().
     * The hash code for the null reference is zero.
     *
     * @param x object for which the hashCode is to be calculated
     * @return  the hashCode
     * @since   JDK1.1
     */
    public static native int identityHashCode(Object x);
```
注释写得很明白了，不管一个对象实例的class有没有覆盖Object的hashCode方法，都能使用这个方法返回hash值。
## sun.reflect.Reflection
这个工具类是和反射相关的，让大家知道有这么一个方法
```
    @CallerSensitive
    public static native Class<?> getCallerClass();
```
我们用一段代码尝试调用这个方法
```
public class CalleeApp {

    public void call() {
        Class<?> clazz = Reflection.getCallerClass();
        System.out.println("Hello " + clazz);
    }
}
public class CallerApp {

    public static void main(String[] args) {
        CalleeApp app = new CalleeApp();
        Caller1 c1 = new Caller1();
        c1.run(app);
    }

    static class Caller1 {
        void run(CalleeApp calleeApp) {
            if (calleeApp == null) {
                throw new IllegalArgumentException("callee can not be null");
            }
            calleeApp.call();
        }
    }

}
```
执行main方法会抛出异常
```
Exception in thread "main" java.lang.InternalError: CallerSensitive annotation expected at frame 1
```
关于这个疑惑可以参考SegmentFault上的一个问题，[Reflection.getCallerClass()使用的问题](https://segmentfault.com/q/1010000014119325)
使用带参数的同名方法，传入指定的参数才能达到`getCallerClass`直觉上的含义的结果
```
    @Deprecated
    public static native Class<?> getCallerClass(int var0);
```
但是这个方法已经被标注为废弃了，因此，准确地讲**不推荐使用**这两个方法，或许这里需要第三方支持，但是`native`方法都无法完善支持的方法，第三方能做到吗？
## Object.wait(long timeout, int nanos)
## 附
* [you-dont-need-serial](https://wdd.js.org/you-dont-need-serial.html)