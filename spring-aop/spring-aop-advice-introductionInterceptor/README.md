## IntroductionInterceptor

- [IntroductionInterceptor](#IntroductionInterceptor)
    - [一、基本信息](#一基本信息)
    - [二、基本描述](#二基本描述)
    - [三、主要功能](#三主要功能)
    - [四、接口源码](#四接口源码)
    - [五、主要实现](#五主要实现)
    - [六、最佳实践](#六最佳实践)
    - [七、源码分析](#七源码分析)
    - [八、常见问题](#八常见问题)

### 一、基本信息

✒️ **作者** - Lex 📝 **博客** - [掘金](https://juejin.cn/user/4251135018533068/posts) 📚 **源码地址** - [github](https://github.com/xuchengsheng/spring-reading)

### 二、基本描述

`IntroductionInterceptor` 接口是 Spring AOP 中的一个关键接口，用于实现引介（Introduction）功能，允许向目标对象动态地添加新的方法或属性，而无需修改目标类的代码，从而实现横切关注点的功能，如日志记录、事务管理等。

### 三、主要功能

1. **引介新的接口或类**
   + 通过实现 `introduce()` 方法，在目标对象的方法调用之前，向目标对象引介新的接口或类，从而使目标对象具有额外的功能或属性。

### 四、接口源码

`IntroductionInterceptor` 接口，它是 AOP 联盟 `MethodInterceptor` 的子接口，允许拦截器实现额外的接口，并通过使用该拦截器的代理对象来使用这些接口。`IntroductionInterceptor` 接口体现了 AOP 中的引介（Introduction）概念，通过引介，可以将额外的功能添加到目标对象中，类似于混合（mixins）的概念，使得可以构建复合对象，实现类似于 Java 中多继承的目标。

```java
/**
 * AOP联盟 MethodInterceptor 的子接口，允许拦截器实现额外的接口，并通过使用该拦截器的代理对象来使用这些接口。这是一个基本的AOP概念，称为<b>引介</b>。
 *
 * <p>引介通常是<b>混合</b>，允许构建能够实现多继承目标的复合对象。
 *
 * @author Rod Johnson
 * @see DynamicIntroductionAdvice
 */
public interface IntroductionInterceptor extends MethodInterceptor, DynamicIntroductionAdvice {

}
```

### 五、主要实现

1. **DelegatingIntroductionInterceptor**

   + `DelegatingIntroductionInterceptor` 是 Spring AOP 提供的通用引介拦截器，允许我们定义自定义的引介逻辑，并在需要时将其应用于目标对象。通过配置，可以动态地向目标对象添加新的方法或属性，而不必修改目标类的代码，提高了代码的可维护性和灵活性。

2. **DelegatePerTargetObjectIntroductionInterceptor**

   + `DelegatePerTargetObjectIntroductionInterceptor` 是 `DelegatingIntroductionInterceptor` 的子类，为每个目标对象创建一个独立的引介代理对象。这意味着每个目标对象都可以拥有自己独立的引介逻辑，而不会受到其他目标对象的影响。这种灵活性特别适用于需要为不同的目标对象动态添加不同功能或属性的场景，提供了更高级的定制能力。

### 六、最佳实践

使用 Spring AOP 中的引介功能。它创建了一个代理工厂，并通过设置强制使用 CGLIB 代理来创建代理对象。然后，它添加了一个通知器，将自定义的引介通知（`MyMonitoringIntroductionAdvice`）应用于目标对象（`MyService` 类），使得目标对象实现了 `MyMonitoringCapable` 接口。最后，它调用了代理对象的方法，并在必要时启用了监控功能，展示了如何在运行时动态地向目标对象引入新的功能。

```java
public class IntroductionInterceptorDemo {

    public static void main(String[] args) {
        // 创建代理工厂&创建目标对象
        ProxyFactory proxyFactory = new ProxyFactory(new MyService());
        // 强制私用CGLIB
        proxyFactory.setProxyTargetClass(true);
        // 创建通知
        proxyFactory.addAdvisor(new DefaultIntroductionAdvisor(new MyMonitoringIntroductionAdvice(), MyMonitoringCapable.class));
        // 创建代理对象
        MyService proxy = (MyService) proxyFactory.getProxy();
        // 调用代理对象的方法
        proxy.doSomething();
        // 开始监控
        ((MyMonitoringCapable) proxy).toggleMonitoring();
        // 再次调用代理对象的方法
        proxy.doSomething();
    }
}
```

`MyMonitoringIntroductionAdvice` 类是一个实现了 `DelegatingIntroductionInterceptor` 接口和 `MyMonitoringCapable` 接口的自定义引介通知类。它具有一个 `active` 属性来表示监控是否处于激活状态，并提供了一个方法 `toggleMonitoring()` 来启用监控功能。在被监控的方法被调用时，如果监控处于激活状态，该类会输出相应的日志信息，包括方法执行时间等。通过继承 `doProceed()` 方法，它能够在方法执行前后添加自定义逻辑，实现了监控功能的动态引入。

```java
public class MyMonitoringIntroductionAdvice extends DelegatingIntroductionInterceptor implements MyMonitoringCapable {

    private boolean active = false;

    public void setActive(boolean active) {
        this.active = active;
    }

    @Override
    public void toggleMonitoring() {
        setActive(true);
    }

    // 当被监控的方法被调用时，如果监控处于激活状态，则输出日志
    @Override
    protected Object doProceed(MethodInvocation mi) throws Throwable {
        if (this.active) {
            System.out.println("开启监控...");
            long startTime = System.currentTimeMillis();
            Object result = super.doProceed(mi);
            long endTime = System.currentTimeMillis();
            System.out.println(mi.getClass().getName() + "." + mi.getMethod().getName() + " 耗费时间：" + (endTime - startTime) + " 毫秒");
            System.out.println("结束监控...");
            return result;
        }
        return super.doProceed(mi);
    }
}
```

`MyMonitoringCapable` 接口定义了一个 `toggleMonitoring()` 方法，用于启用或禁用监控功能。

```java
public interface MyMonitoringCapable {
    void toggleMonitoring();
}
```

`MyService` 类是一个简单的服务类，其中包含了一个名为 `doSomething()` 的方法。在上下文中，`MyService` 类被用作目标对象，即需要被拦截和增强的对象。

```java
public class MyService {

    public String doSomething() {
        System.out.println("Doing something...");
        return "hello world";
    }
}
```

### 七、源码分析

暂无

### 八、常见问题

1. **引介的作用和优势是什么？**

   - 这个问题探讨了引介在 AOP 中的作用和优势，以及它与其他 AOP 概念（如通知、切点）的区别。

2. **如何实现引介？**

   - 这个问题涉及到如何使用 `IntroductionInterceptor` 接口以及其子类来实现引介功能，以及在 Spring AOP 中如何配置和应用引介。

3. **引介和通知之间的区别是什么？**

   - 这个问题探讨了引介和其他类型的通知（如前置通知、后置通知）之间的区别，以及它们在 AOP 中的不同用途。

4. **引介在哪些场景下会被使用？**

   - 这个问题讨论了引介在实际开发中的应用场景，以及它如何帮助解决特定的横切关注点。

5. **引介是否有性能影响？**

   - 这个问题关注引介在运行时对性能的影响，以及如何优化引介以减少潜在的性能开销。

6. **如何处理引介与目标类之间的依赖关系？**

   - 这个问题涉及引介与目标类之间的依赖关系管理，以及如何确保引介逻辑能够正确地与目标类交互。

7. **引介在 Spring 中的实现原理是什么？**

   - 这个问题探讨了 Spring AOP 是如何利用代理机制实现引介功能的，以及它与其他 AOP 框架的实现方式的区别。