

#AOP原理分析
AOP术语
1) Aspect(切面):通常是一个类，里面可以定义切入点和通知
2) JointPoint(连接点):程序执行过程中明确的点，一般是方法的调用
3) Advice(通知):AOP在特定的切入点上执行的增强处理，有before,after,afterReturning,afterThrowing,around
4) Pointcut(切入点):就是带有通知的连接点，在程序中主要体现为书写切入点表达式
5) AOP代理：AOP框架创建的对象，代理就是目标对象的加强。Spring中的AOP代理可以使JDK动态代理，也可以是CGLIB代理，前者基于接口，后者基于子类

## 入口
aop　测试程序如下：
```java
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("tinyioc.xml");
HelloWorldService helloWorldService = (HelloWorldService) applicationContext.getBean("helloWorldService");
helloWorldService.helloWorld();
```

tinyioc.xml源文件如下，　helloWorldService 包含一个属性 outputService, 
outputService 引用的是一个 bean 对象．
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>
    <bean id="outputService" class="us.codecraft.tinyioc.OutputServiceImpl">
    </bean>

    <bean id="helloWorldService" class="us.codecraft.tinyioc.HelloWorldServiceImpl">
        <property name="text" value="Hello World!"></property>
        <property name="outputService" ref="outputService"></property>
    </bean>

    <bean id="autoProxyCreator" class="us.codecraft.tinyioc.aop.AspectJAwareAdvisorAutoProxyCreator"></bean>

    <!-- 通知 -->
    <bean id="timeInterceptor" class="us.codecraft.tinyioc.aop.TimerInterceptor"></bean>

    <!-- 切面 -->
    <bean id="aspectjAspect" class="us.codecraft.tinyioc.aop.AspectJExpressionPointcutAdvisor">
        <property name="advice" ref="timeInterceptor"></property>
        <property name="expression" value="execution(* us.codecraft.tinyioc.*.*(..))"></property>
    </bean>
</beans>
```

##ClassPathXmlApplicationContext构造函数
参考[spring ioc](./analyze_ioc.md)的分析

AspectJAwareAdvisorAutoProxyCreator实现了BeanFactoryAware，　BeanPostProcessor接口．
在ClassPathXmlApplicationContext#registerBeanPostProcessors 方法调用过程中，　autoProxyCreator会被实例化．
其它bean会在ClassPathXmlApplicationContext#onRefresh方法调用中实例化．　


timeInterceptor和aspectjAspect实例化时，autoProxyCreator#postProcessAfterInitialization会被回调，但函数中不做任何处理．
但是，对于outputService和helloWorldService的实例化，　autoProxyCreator#postProcessAfterInitialization会做切面处理．
```java
public class AspectJAwareAdvisorAutoProxyCreator implements BeanPostProcessor, BeanFactoryAware {

	private AbstractBeanFactory beanFactory;

	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws Exception {
		return bean;
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws Exception {
		if (bean instanceof AspectJExpressionPointcutAdvisor) {
			return bean;
		}
		if (bean instanceof MethodInterceptor) {
			return bean;
		}
		
		//获取所有切面
		List<AspectJExpressionPointcutAdvisor> advisors = beanFactory
				.getBeansForType(AspectJExpressionPointcutAdvisor.class);
		for (AspectJExpressionPointcutAdvisor advisor : advisors) {
			if (advisor.getPointcut().getClassFilter().matches(bean.getClass())) {//类可以匹配
                ProxyFactory advisedSupport = new ProxyFactory();
				advisedSupport.setMethodInterceptor((MethodInterceptor) advisor.getAdvice());
				advisedSupport.setMethodMatcher(advisor.getPointcut().getMethodMatcher());

				TargetSource targetSource = new TargetSource(bean, bean.getClass(), bean.getClass().getInterfaces());
				advisedSupport.setTargetSource(targetSource);

				return advisedSupport.getProxy();
			}
		}
		return bean;
	}

	@Override
	public void setBeanFactory(BeanFactory beanFactory) throws Exception {
		this.beanFactory = (AbstractBeanFactory) beanFactory;
	}
}
``` 

ProxyFactory继承了AdvisedSupport类, 这里的getProxy方法默认使用cglib代理方式。
```java
public class ProxyFactory extends AdvisedSupport implements AopProxy {
	@Override
	public Object getProxy() {
		return createAopProxy().getProxy();
	}

	protected final AopProxy createAopProxy() {
		return new Cglib2AopProxy(this);
	}
}
```

AdvisedSupport是一个dto，主要用于存储代理相关元数据．
```java
public class AdvisedSupport {
	private TargetSource targetSource;
    private MethodInterceptor methodInterceptor;
    private MethodMatcher methodMatcher;
}
```

```java
public interface AopProxy {
    Object getProxy();
}

public abstract class AbstractAopProxy implements AopProxy {
    protected AdvisedSupport advised;

    public AbstractAopProxy(AdvisedSupport advised) {
        this.advised = advised;
    }
}
```

AbstractAopProxy包含两个实现类Cglib2AopProxy和JdkDynamicAopProxy。

```java
public class Cglib2AopProxy extends AbstractAopProxy {
	public Cglib2AopProxy(AdvisedSupport advised) {
		super(advised);
	}

	@Override
	public Object getProxy() {
		Enhancer enhancer = new Enhancer();
		enhancer.setSuperclass(advised.getTargetSource().getTargetClass());
		enhancer.setInterfaces(advised.getTargetSource().getInterfaces());
		enhancer.setCallback(new DynamicAdvisedInterceptor(advised));
		Object enhanced = enhancer.create();
		return enhanced;
	}
}

public class JdkDynamicAopProxy extends AbstractAopProxy implements InvocationHandler {
    public JdkDynamicAopProxy(AdvisedSupport advised) {
        super(advised);
    }

	@Override
	public Object getProxy() {
		return Proxy.newProxyInstance(getClass().getClassLoader(), advised.getTargetSource().getInterfaces(), this);
	}
}
```


JDK动态代理和CGLIB字节码生成的区别？
 * JDK动态代理只能对实现了接口的类生成代理，而不能针对类
 * CGLIB是针对类实现代理，主要是对指定的类生成一个子类，覆盖其中的方法,因为是继承，所以该类或方法最好不要声明成final 
 * JDK在创建代理对象时的性能要高于CGLib代理，而生成代理对象的运行性能却比CGLib的低。
 
Spring AOP默认是使用JDK动态代理，如果代理的类没有接口则会使用CGLib代理。

在《精通Spring4.x 企业应用开发实战》给出了建议：
>如果是单例的我们最好使用CGLib代理，如果是多例的我们最好使用JDK代理


