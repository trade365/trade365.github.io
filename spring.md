

@[toc]
# Spring

## 常用注解

1. Profile：被Profile注释的Component只有当注释的值(value)与spring.profiles.active的值相同时才会生效。

2. ActiveProfiles：注释在Spring Boot单测类上，如：@ActiveProfiles("test")。

3. RunWith(SpringRunner.class)+SpringBootTest：注释在Spring Boot单测类上，等同于启动了Spring Boot应用。

4. Autowired，Component和Qualifier(表明哪个bean是需要注入的)：

Autowired只能按照type方式注入，按照如下方式注入接口的实现类。

```java
Component("implA")
class ImplA implements InterfaceX {
}

Component("implB")
class ImplB implements InterfaceX {
}

@Autowired
@Qualifier("implB")
InterfaceX interfaceX;
```

5. Before，After，Test：Test是注释在测试方法上，每个测试方法执行之前执行Before方法，执行之后执行After方法。

6. Configuration和Bean：

Configuration注释在类上，表示这个类的方法(用Bean注释)返回的对象可以作为Spring的bean。使用方法如下：
```java
public class MyBean {
    public void destroy() {
        System.out.println("my bean destroy");
    }

    public void init() {
        System.out.println("my bean init");
    }

    public MyBean() {
        System.out.println("my bean construct");
    }
}

@Configuration
class AppConfig {

    // initMethod表示init方法将在MyBean构造函数之后执行
    @Bean(initMethod = "init")
    public MyBean myBean() {
        return new MyBean();
    }
}
```

Component和Configuration的区别是：Configuration对Bean方法拦截，使其返回的都是同一个对象，举例如下：

```java
@Data
public class MyBean2 {
    private MyBean myBean;
}

@Component
public class AppConfig {

    @Bean(initMethod = "init")
    MyBean myBean() {
        return new MyBean();
    }

    @Bean
    MyBean2 myBean2() {
        MyBean2 myBean2 = new MyBean2();
        myBean2.setMyBean(myBean());
        return myBean2;
    }
}
```

用Component会构造两个MyBean对象，用Configuration会构造一个MyBean对象。

7. ComponentScan

和Configuration注解一起使用，替代spring xml配置中的component-scan标签，指定spring扫描component路径。

8. Resource注解，和Autowired注解的区别：https://www.cnblogs.com/think-in-java/p/5474740.html

## AOP

### JDK动态代理

根据原始对象生成一个invocationHandler，对原始对象的方法进行加工，再根据这个invocationHandler生成一个proxy对象，调用proxy对象的方法就会调用invocationHandler的invoke方法，代码如下：

```java
interface Person {
    void walk();
}

class Student implements Person {
    @Override
    public void walk() {
        System.out.println("student walk");
    }
}

class PersonInvocationHandler implements InvocationHandler {
    private Person person;
    public PersonInvocationHandler(Person person) {
        this.person = person;
    }

    @Override
    public Object invoke(Object o, Method method, Object[] objects) throws Throwable {
        System.out.println("begin");
        method.invoke(person);
        System.out.println("end");
        return null;
    }
}

// 调用
Person person = new Student();
PersonInvocationHandler personInvocationHandler = new PersonInvocationHandler(person);
Person personProxy = (Person) Proxy.newProxyInstance(Person.class.getClassLoader(),
                person.getClass().getInterfaces(), personInvocationHandler);
personProxy.walk();

/* 
输出：
begin
student walk
end
*/
```

### CGLIB动态代理

实现MethodInterceptor接口，对被代理方法进行加工。用被代理类和实现MethodInterceptor接口的类对象构造Enhancer对象，用该对象创建被代理对象。GCLIB原理是生成被代理类的子类，来增强被代理类。举例如下：

```java
public class PersonMethodInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("before walk");
        return methodProxy.invokeSuper(o, objects);
    }
}

// 调用
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(Student.class);
enhancer.setCallback(new PersonMethodInterceptor());
Student student = (Student) enhancer.create();
student.walk();

/*
输出：
before walk
student walk
*/
```

### Spring AOP

注释在Spring Component类上，表明这个类的方法可以用来增强指定方法，即AOP。

```java
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;

// AOP类
@Component
@Aspect
public class LogAdvice {

    @Before("within(com.xxx...*) && @annotation(log)")
    public void beforeLog(JoinPoint joinPoint, Log log) {
        System.out.println("method name=" + joinPoint.getSignature().getName());
        System.out.println("args=" + joinPoint.getArgs()[0]);
        System.out.println("before log, annotation info=" + log.detail());
    }

    @AfterReturning("within(com.xxx...*) && @annotation(log)")
    public void afterLog(JoinPoint joinPoint, Log log) {
        System.out.println("after log, annotation info=" + log.detail());
    }

    @AfterThrowing(value = "within(com.xxx...*) && @annotation(log)", throwing = "ex")
    public void afterLogThrowing(JoinPoint joinPoint, Log log, Exception ex) {
        System.out.println("after log throwing, annotation info=" + log.detail());
    }

    @Around(value = "within(com.xxx..*) && @annotation(log)")
    public void aroundLog(ProceedingJoinPoint proceedingJoinPoint, Log log) throws Throwable {
        System.out.println("around log , annotation info=" + log.detail());
        System.out.println("proceedingJoinPoint method name=" + proceedingJoinPoint.getSignature().getName());
        System.out.println("proceedingJoinPoint args=" + proceedingJoinPoint.getArgs()[0]);
        proceedingJoinPoint.proceed();
    }
}

// 增强被Log注释的方法
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Log {
    String detail();
}

@Component
public class LogAnnotationTest {

    @Log(detail = "hello world")
    public void testMethod(String msg) {
        System.out.println("test method, msg=" + msg);
    }
}

// 调用
logAnnotationTest.testMethod("this is msg");

/*
输出：
around log , annotation info=hello world
proceedingJoinPoint method name=testMethod
proceedingJoinPoint args=this is msg
method name=testMethod
args=this is msg
before log, annotation info=hello world
test method, msg=this is msg
after log, annotation info=hello world
*/
```

Spring AOP几种无法执行切面的情况：

1.调用自身的其他增强方法

```java
@Component
public class TestServiceImpl implements TestService {

    @Override
    public void callA() {
        System.out.println("call a");
        callB(); // callB无法执行切面，因为这个对象已经变成代理对象了，执行切面需要另外一个代理。
    }

    @Override
    public void callB() {
        System.out.println("call b");
    }

    public void callC() {
        callB();// callB无法执行切面
    }
}

ConfigurableApplicationContext ctx = SpringApplication.run(MySpringBootApplication.class, args);
TestServiceImpl testService = ctx.getBean(TestServiceImpl.class);
testService.callA();
testService.callC();
```

根据代理对象获取被代理对象：

```java
if (AopUtils.isCglibProxy(proxyClass)) {
    Field h = currClass.getDeclaredField("CGLIB$CALLBACK_0");
    h.setAccessible(true);
    Object dynamicAdvisedInterceptor = h.get(proxyClass);
    Field advised = dynamicAdvisedInterceptor.getClass().getDeclaredField("advised");
    advised.setAccessible(true);
    Object target = ((AdvisedSupport) advised.get(dynamicAdvisedInterceptor)).getTargetSource().getTarget();
    currClass = target.getClass();
} else if (AopUtils.isJdkDynamicProxy(proxyClass)) {
    Field h = currClass.getSuperclass().getDeclaredField("h");
    h.setAccessible(true);
    AopProxy aopProxy = (AopProxy) h.get(proxyClass);
    Field advised = aopProxy.getClass().getDeclaredField("advised");
    advised.setAccessible(true);
    Object target = ((AdvisedSupport) advised.get(aopProxy)).getTargetSource().getTarget();
    currClass = target.getClass();
}
```

Spring主要通过AopProxy接口来实现Aop，主要包括JdkDynamicAopProxy和CglibAopProxy

getProxy方法返回代理对象。

* JdkDynamicAopProxy：JDK动态代理，通过JDK的InvocationHandler实现了AOP
* CglibAopProxy：通过CGLIB动态代理实现AOP

## SpringBoot

SpringBoot的启动代码：

```java
@SpringBootApplication
public class TestApp {
  public static void main(String[] args) {  
    SpringApplication.run(TestApp.class, args);  
  }
}
```

SpringBootApplication是EnableAutoConfiguration，SpringBootConfiguration，和ComponentScan的复合注解。

run方法会执行SpringApplication的run方法（非静态）：

```java

// Allows post-processing of the bean factory in context subclasses.
postProcessBeanFactory(beanFactory);

// 根据ComponentScan或ImportResource注解找到basePackage，
// classloader.getReources()找到目录，扫描目录的bean
invokeBeanFactoryPostProcessors(beanFactory);

// 找到BeanPostProcessors，比如自定义注解处理器等
registerBeanPostProcessors(beanFactory);

// Initialize message source for this context.
initMessageSource();

// Initialize event multicaster for this context.
initApplicationEventMulticaster();

// 启动Tomcat
onRefresh();

// 检测定义的listeners
registerListeners();

// 初始化bean->注入bean->bean后处理->init方法
// 代理何时被初始化？？
finishBeanFactoryInitialization(beanFactory);

// 发布事件
finishRefresh();

// 调用runner
afterRefresh(context, applicationArguments);
```

**BeanPostProcessor**

BeanPostProcessor可以在Bean执行init方法或者destroy方法之前执行一些自定义的动作，比如给Bean的某些属性依据自定义的注解注入对象。


# 为什么要写spring.factories
https://blog.csdn.net/SkyeBeFreeman/article/details/96291283
ComponentScan只能扫自己的包的bean
需要EnableAutoConfiguration注解，以及引入包添加spring.factories文件指明需要注入的Bean。


# BeanFactory和FactoryBean什么区别
https://cloud.tencent.com/developer/article/1457284
FactoryBean的应用：
ProxyFactoryBean：https://blog.csdn.net/c5113620/article/details/83578114。
修饰了被代理对象
getBean("xxxService")返回的其实是ProxyFactoryBean的getObject方法。返回的是代理对象。

# Mybatis问题

