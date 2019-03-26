---
layout: post
category: java
comments: true
title: Transactional是怎么工作的
tags: aop transaction
---
* content
{:toc}

## 概述
@Transactional是Spring事务管理使用到的一个注解，在一个方法中加上了这个注解，那么这个方法就将是有事务的，方法内的操作要么一起提交、要么一起回滚；关于它背后的工作原理涉及到不少知识点，本文重点是梳理整个过程，更具体的事务配置属性，Spring事务与第三方ORM框架的集成等不是本文关心的重点；

## 使用@Transactional
在application-context.xml配置文件中开启对@Transactional注解的支持，配置Scan路径等
```xml
<tx:annotation-driven />
```
在Service方法中加上注解
```java
@Service
public class UserServiceImpl implements UserService {
    @Transactional
    public int register(User user) {
        addUser(user);
        auditLog(user);
    }
}
```
以上就是让@Transactional正常工作的简单的配置及使用方式了；
## 原理探析
@Transactional要做的事情大致是这样子的，
```java
try { 
    tx.begin(); 
    businessLogic();
    tx.commit(); 
} catch(Exception ex) { 
    tx.rollback(); 
    throw ex; 
}
```
这也是事务管理器TransactionManager的常规操作，只是在这里用一个注解“完成”了手工编写事务管理代码的方式；所有其他需要开启事务的Service方法都只是配置这么一个注解就可以省去编写代码的方式，这就是切面编程的思想，也是Spring AOP结合注解方式完成的；针对上面的xml配置及代码，进一步分析Spring最终是如何实现这个切面的；

### BeanDefinition and BeanPostProcessor

首先是对tx命名空间的支持，`TxNamespaceHandler`中注册了`AnnotationDrivenBeanDefinitionParser`解析器，提供对annotation-driven属性解析的支持；
在这里构造三个重要的BeanDefinition以及一个BeanPostProcessor，分别是，
BeanDefinition，
* TransactionAttributeSource // 用于获取@Transactional注解的属性
* TransactionInterceptor // 方法拦截器 依赖TransactionManager Bean；依赖transactionAttributeSource

* TransactionAttributeSourceAdvisor // 依赖transactionAttributeSource； 依赖TransactionInterceptor；
TransactionAttributeSourceAdvisor是PointcutAdvisor，是Spring AOP的内容，包括pointCut和advice
BeanPostProcessor，
* InfrastructureAdvisorAutoProxyCreator // 用于创建Proxy

### postProcessAfterInitialization

当然一个Bean依赖UserService bean时，首先要初始化UserServiceImpl，应用后置处理器，当应用InfrastructureAdvisorAutoProxyCreator后置处理器时，发现UserServiceImpl使用了@Transaction注解，所以它应该被代理，即这里开始创建proxy，将proxy实例返回给依赖UserService的Bean；分析创建proxy的过程，
```java
// Create proxy if we have advice.
Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
if (specificInterceptors != DO_NOT_PROXY) {
    this.advisedBeans.put(cacheKey, Boolean.TRUE);
    Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
    this.proxyTypes.put(cacheKey, proxy.getClass());
    return proxy;
}
```
在创建proxy前获得TransactionInterceptor方法拦截器，然后交给createProxy方法来完成；在createProxy中的过程就是Spring AOP的过程，方法拦截器组织成Advisor，根据是否代理接口创建InvocationHandler，这里的InvocationHandler是JdkDynamicAopProxy，最终由它来创建proxy实例，
```java
@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
    if (logger.isTraceEnabled()) {
        logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
    }
    Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
    findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
    return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```
### invoke on proxy instance
当调用Service方法时，实际调用的是代理类的方法，这里调用的由JdkDynamicAopProxy作为InvocationHandler创建的proxy实例，最终会调用到JdkDynamicAopProxy的invoke方法上（注：JDK动态代理），而invoke方法，将会使用到前面的TransactionInterceptor，
```java
// Get the interception chain for this method.
List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
// We need to create a method invocation...
invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
// Proceed to the joinpoint through the interceptor chain.
retVal = invocation.proceed();
```
ReflectiveMethodInvocation最终会调用TransactionInterceptor的invoke方法

### TransactionInterceptor
TransactionInterceptor的invoke就可以发现了事务管理的痕迹了，
它调用父类的invokeWithinTransaction,做这样的操作，
```java
if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
    // Standard transaction demarcation with getTransaction and commit/rollback calls.
    TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
    Object retVal = null;
    try {
        // This is an around advice: Invoke the next interceptor in the chain.
        // This will normally result in a target object being invoked.
        retVal = invocation.proceedWithInvocation();
    }
    catch (Throwable ex) {
        // target invocation exception
        completeTransactionAfterThrowing(txInfo, ex);
        throw ex;
    }
    finally {
        cleanupTransactionInfo(txInfo);
    }
    commitTransactionAfterReturning(txInfo);
    return retVal;
}
```
tm即TransactionManager，在BeanDefinition注册阶段就被TransactionInterceptor依赖的；分析到这里并未见到
```
tx.begin()
tx.commit()
tx.rollback()
```
等，其实这些和事务相关的方法，全部都由TransactionManager来完成的，跟踪上面的
`createTransactionIfNecessary`方法，会发现在TransactionManager的getTransaction方法会执行doBegin操作，doBegin方法将从TransactionManager的dataSource中创建数据库链接Connection，最后将dataSource和Connection关联起来，绑定到当前线程，即存在ThreadLocal中，
```java
TransactionSynchronizationManager.bindResource(this.obtainDataSource(), txObject.getConnectionHolder());
```
`commitTransactionAfterReturning`就相当于`tx.commit`，`completeTransactionAfterThrowing`就相当于`tx.rollback`；
### TransactionInterceptor vs TransactionManager vs JdbcTemplate vs TransactionSynchronizationManager
到这里已经梳理清楚了，Spring使用BeanPostProcessor对使用了@Transactional注解的类进行代理，由BeanPostProcessor创建的InvocationHandler使用了TransactionInterceptor方法拦截器，在这个方法拦截器实现事务管理的操作，实现事务管理的操作是交给TransactionInterceptor的TransactionManager来完成的，TransactionManager使用dataSource来创建数据库链接；

然而，仍然存在一个问题，TransactionInterceptor创建的Connection，怎么给到`businessLogic`，给到`UserServiceImpl#register`使用的呢；这里我们以`JdbcTemplate`为例进行分析，
`JdbcTemplate`的操作中需要获取数据库链接Connection，它是这样实现的，
```java
Connection con = DataSourceUtils.getConnection(this.obtainDataSource());
```
跟踪DataSourceUtils代码，最终会发现这样的代码
```java
ConnectionHolder conHolder = (ConnectionHolder)TransactionSynchronizationManager.getResource(dataSource);
```
这样就前文的绑定对应起来了，绑定到ThreadLocal，从ThreadLocal获取都是和TransactionSynchronizationManager相关的；
其实，mybatis使用Spring事务管理，它的获取链接的方式也是用了DataSourceUtils；
## Links
- [How Does Spring @Transactional Really Work?](https://dzone.com/articles/how-does-spring-transactional)