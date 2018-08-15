---
layout: post
category: java
comments: true
title: 使用Mockito修改Bean的依赖
tags: Mockito Spring JUnit
---

* content
{:toc}

## 概述
在使用单元测试时经常会遇到某些dependency依赖了外部资源，或者想主动绕过真正的方法执行mock返回结果而快速得到单元测试最终的期望结果，可能有以下两种场景，
对于TestCase A，设单元测试的方法是Service A的execute1方法和execute2方法，在执行execute1和execute2方法时都会调用ServiceB的不同方法，即ServiceA依赖了ServiceB；一个场景是完全对ServiceB进行Mock，如单元测试ServiceA#execute1方法时都通过Mock返回结果；一个场景是部分ServiceB的方法执行真实的业务逻辑（如查询数据库），一部分方法执行Mock返回结果，或Spy，如如单元测试ServiceA#execute2方法时，只mock ServiceB#b2结果，真正执行ServiceB#b1方法。

## 对TestCase的Service的依赖Bean的完全Mock
当对ServiceA的方法执行单元测试时，如ServiceA -> ServiceB，此时对ServiceB进行Mock，然后将其设置到ServiceA的属性中；后续ServiceA调用ServiceB的方法都降得到Mock后的结果；而对于ServiceB对象的本来的依赖本案暂且将其忽略，后续改进；


思路是在TestCase中依赖ServiceA的同时标示Mock ServiceB，待TestCase依赖注入完成后，新建ServiceB的Mock对象替换ServiceA中的ServiceB依赖；
```java
@TestExecutionListeners({MockitoDependencyInjectionTestExecutionListener.class})
public class AServiceMockTest extends BaseTest {
    @Mock
    private BService bservice;
    @Autowired
    private AService aservice;
    @Before
    public void setup(){
        doReturn("mock").when(bservice).b1();
    }
    @Test
    public void test() {
        a.execute1();
    }
}
@Service
public class AServiceImpl implements AService {
    @Autowired
    private BService bservice;
     
    @Override
    public String execute1() {
        return bservice.b1(); //will return mock after Mock
    }
}
```
当a.execute()执行时将调用aservice的属性bservice的b1方法，返回结果就是在setup方法中指定的结果；
## 监听TestCase的Service的依赖Bean
当对ServiceA进行单元测试时，依赖了ServiceB，需要获取ServiceB的b1方法的真正执行结果，Mock b2方法的结果，此时可以采用Spy方式；由于ServiceA依赖了ServiceB，而这个属性可能是个AopProxy对象，并不能直接使用Mockito.mock(bservice)或者Mockito.spy(bservice)，所以这里@Spy注解指定的是实现类，通过MockitoDependencyInjectionTestExecutionListener处理后，获得一个Spy对象，同时这个Spy对象设置到bservice（AopProxy对象）中去；
```java
@TestExecutionListeners({MockitoDependencyInjectionTestExecutionListener.class})
public class AServiceMockTest extends BaseTest {
    @Spy
    private BServiceImpl bserviceImpl;
    @Autowired
    private AService aservice;
    @Before
    public void setup(){
        doReturn(true).when(bserviceImpl).b2(any(String.class));
    }
    @Test
    public void test() {
        a.execute();
    }
}
@Service
public class AServiceImpl implements AService {
    @Autowired
    private BService bservice;
     
    @Override
    public boolean execute2() {
        String str = bservice.b1();
        return bservice.b2(str);
    }
}
```
## MockitoDependencyInjectionTestExecutionListener的实现
```java
public class MockitoDependencyInjectionTestExecutionListener extends DependencyInjectionTestExecutionListener {

    private Set<Field> injectFields = new HashSet<>();
    private Map<String,Object> mockObjectMap = new HashMap<>();
    @Override
    protected void injectDependencies(TestContext testContext) throws Exception {
        super.injectDependencies(testContext);
        init(testContext);
    }

    /**
     * when A dependences on B
     * mock B or Spy on targetObject of bean get from Spring IoC Container whose type is B.class or beanName is BImpl
     * @param testContext
     */
    private void init(TestContext testContext) throws Exception {

        AutowireCapableBeanFactory factory =testContext.getApplicationContext().getAutowireCapableBeanFactory();
        Object bean = testContext.getTestInstance();
        Field[] fields = bean.getClass().getDeclaredFields();

        for (Field field : fields) {
            Annotation[] annotations = field.getAnnotations();
            for (Annotation annotation : annotations) {
                if(annotation instanceof Mock){
                    Class<?> clazz = field.getType();
                    Object object = Mockito.mock(clazz);
                    field.setAccessible(true);
                    field.set(bean, object);
                    mockObjectMap.put(field.getName(), object);
                } else if(annotation instanceof Spy) {
                    Object fb = factory.getBean(field.getName()); //may be a proxy that can not be spy because $Proxy is final
                    Object targetSource = AopTargetUtils.getTarget(fb);
                    Object spyObject = Mockito.spy(targetSource);
                    if (!fb.equals(targetSource)) { //proxy
                        if (AopUtils.isJdkDynamicProxy(fb)) {
                            setJdkDynamicProxyTargetObject(fb, spyObject);
                        } else { //cglib
                            setCglibProxyTargetObject(fb, spyObject);
                        }
                    } else {
                        mockObjectMap.put(field.getName(), spyObject);
                    }
                    field.setAccessible(true);
                    field.set(bean, spyObject);
                }else if (annotation instanceof Autowired){
                    injectFields.add(field);
                }
            }
        }
        for(Field field: injectFields) {
            field.setAccessible(true);
            Object fo = field.get(bean);
            if (AopUtils.isAopProxy(fo)) {
                Class targetClass = AopUtils.getTargetClass(fo);
                if(targetClass ==null)
                    return;
                Object targetSource = AopTargetUtils.getTarget(fo);
                Field[] targetFields =targetClass.getDeclaredFields();
                for(Field targetField : targetFields){
                    targetField.setAccessible(true);
                    if(mockObjectMap.get(targetField.getName()) ==null){
                        continue;
                    }
                    ReflectionTestUtils.setField(targetSource,targetField.getName(), mockObjectMap.get(targetField.getName()));
                }

            } else {
                Object realObject = factory.getBean(field.getType());
                if(null != realObject) {
                    Field[] targetFields = realObject.getClass().getDeclaredFields();
                    for(Field targetField : targetFields){
                        targetField.setAccessible(true);
                        if(mockObjectMap.get(targetField.getName()) ==null){
                            continue;
                        }
                        ReflectionTestUtils.setField(fo,targetField.getName(), mockObjectMap.get(targetField.getName()));
                    }
                }
            }
        }
    }

    private void setCglibProxyTargetObject(Object proxy, Object spyObject) throws NoSuchFieldException, IllegalAccessException {
        Field h = proxy.getClass().getDeclaredField("CGLIB$CALLBACK_0");
        h.setAccessible(true);
        Object dynamicAdvisedInterceptor = h.get(proxy);
        Field advised = dynamicAdvisedInterceptor.getClass().getDeclaredField("advised");
        advised.setAccessible(true);
        ((AdvisedSupport) advised.get(dynamicAdvisedInterceptor)).setTarget(spyObject);

    }

    private void setJdkDynamicProxyTargetObject(Object proxy, Object spyObject) throws NoSuchFieldException, IllegalAccessException {
        Field h = proxy.getClass().getSuperclass().getDeclaredField("h");
        h.setAccessible(true);
        AopProxy aopProxy = (AopProxy) h.get(proxy);
        Field advised = aopProxy.getClass().getDeclaredField("advised");
        advised.setAccessible(true);
        ((AdvisedSupport) advised.get(aopProxy)).setTarget(spyObject);
    }
}
```
代码的思路是，首先由`BaseTest`处理好TestCase的依赖注入问题，即示例中@Autowired注解的属性，然后分别针对@Mock、@Spy和@Autowired进行处理，
* @Mock的处理

TestCase中加上@Mock的属性可以是接口也可以是具体实现类，获得属性的类型Class，执行Mock；
* @Spy的处理

TestCase中加上@Spy的属性只能是具体实现类，这里通过属性的名字首先先从容器中获取，返回的Spring Bean有可能是一个AopProxy对象，而我们Spy的目标是AopProxy对象的目标对象，使用Mockito.spy目前对象然后替换；如果不是AopProxy对象，
执行Spy后后面做法与Mock相同；

* @Autowired的处理

Mock和Spy之后可能要将结果设置到@Autowired的属性的内部属性中去，同样需要区分@Autowired属性是否是AopProxy对象，将fake后的对象按照属性名字设置到AopProxy目标对象的属性中（有点绕）；

## 附录
### maven依赖
JUnit、Mockito
### AopTargetUtils
AopTargetUtils工具类参考[在spring中获取代理对象代理的目标对象工具类](http://jinnianshilongnian.iteye.com/blog/1613222)


