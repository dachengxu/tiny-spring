

#IOC原理分析

## 入口
ioc　测试程序如下：
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
        <property name="helloWorldService" ref="helloWorldService"></property>
    </bean>

    <bean id="helloWorldService" class="us.codecraft.tinyioc.HelloWorldServiceImpl">
        <property name="text" value="Hello World!"></property>
        <property name="outputService" ref="outputService"></property>
    </bean>

    <bean id="beanInitializeLogger" class="us.codecraft.tinyioc.BeanInitializeLogger">
    </bean>
</beans>
```

## ClassPathXmlApplicationContext构造函数
在　ClassPathXmlApplicationContext　构造函数中保存　configLocation, 和 beanFactory,
之后调用 refresh　方法．
```java
public class ClassPathXmlApplicationContext extends AbstractApplicationContext {
	private String configLocation;

	public ClassPathXmlApplicationContext(String configLocation) throws Exception {
		this(configLocation, new AutowireCapableBeanFactory());
	}

	public ClassPathXmlApplicationContext(String configLocation, AbstractBeanFactory beanFactory) throws Exception {
		super(beanFactory);//调用父类构造函数
		this.configLocation = configLocation;
		refresh();
	}
}
```

##refresh功能　
refresh 方法实现三个功能:
1) 调用　loadBeanDefinitions　方法, 加载 beanDefinitions, 
2) 调用　registerBeanPostProcessors方法, 注册　BeanPostProcessor，
3) 调用 onRefresh　方法．实例化单件．
```java
public abstract class AbstractApplicationContext implements ApplicationContext {
	protected AbstractBeanFactory beanFactory;

	public AbstractApplicationContext(AbstractBeanFactory beanFactory) {
		this.beanFactory = beanFactory;//这里的beanFactory是AutowireCapableBeanFactory
	}

	public void refresh() throws Exception {
		loadBeanDefinitions(beanFactory);
		registerBeanPostProcessors(beanFactory);
		onRefresh();
	}
	
	protected abstract void loadBeanDefinitions(AbstractBeanFactory beanFactory) throws Exception;
}
```

###loadBeanDefinitions方法
loadBeanDefinitions方法在　ClassPathXmlApplicationContext　实现类中实现，
1) 新建XmlBeanDefinitionReader实例，解析　tinyioc.xml　配置文件，存放到Map<String,BeanDefinition> registry．
2) 把registry中的BeanDefinition　注册到 beanFactory 中．
```java
public class ClassPathXmlApplicationContext extends AbstractApplicationContext {
	@Override
	protected void loadBeanDefinitions(AbstractBeanFactory beanFactory) throws Exception {
		XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(new ResourceLoader());
		xmlBeanDefinitionReader.loadBeanDefinitions(configLocation);
		for (Map.Entry<String, BeanDefinition> beanDefinitionEntry : xmlBeanDefinitionReader.getRegistry().entrySet()) {
			beanFactory.registerBeanDefinition(beanDefinitionEntry.getKey(), beanDefinitionEntry.getValue());
		}
	}
}
```

BeanDefinition 类定义如下
```java
public class BeanDefinition {
	private Object bean;
	private Class beanClass;
	private String beanClassName;
	private PropertyValues propertyValues = new PropertyValues();
}

public class PropertyValues {
	private final List<PropertyValue> propertyValueList = new ArrayList<PropertyValue>();
}

public class PropertyValue {
    private final String name;
    private final Object value;
}

public class BeanReference {
    private String name;
    private Object bean;
}
```

XmlBeanDefinitionReader#loadBeanDefinitions 主要功能是解析　xml配置文件，　
需要注意的是当property标签中包含ref属性的时候，PropertyValue中的value属性存放的是一个BeanReference实例．
```java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {
private void processProperty(Element ele, BeanDefinition beanDefinition) {
		NodeList propertyNode = ele.getElementsByTagName("property");
		for (int i = 0; i < propertyNode.getLength(); i++) {
			Node node = propertyNode.item(i);
			if (node instanceof Element) {
				Element propertyEle = (Element) node;
				String name = propertyEle.getAttribute("name");
				String value = propertyEle.getAttribute("value");
				if (value != null && value.length() > 0) {
					beanDefinition.getPropertyValues().addPropertyValue(new PropertyValue(name, value));
				} else {
					String ref = propertyEle.getAttribute("ref");
					if (ref == null || ref.length() == 0) {
						throw new IllegalArgumentException("Configuration problem: <property> element for property '"
								+ name + "' must specify a ref or value");
					}
					BeanReference beanReference = new BeanReference(ref);　//引用实例
					beanDefinition.getPropertyValues().addPropertyValue(new PropertyValue(name, beanReference));
				}
			}
		}
	}
}
```

### registerBeanPostProcessors方法
通过　getBeansForType　查找所有BeanPostProcessor实现类，加入到beanFactory中．
```java
public abstract class AbstractApplicationContext implements ApplicationContext {
	protected void registerBeanPostProcessors(AbstractBeanFactory beanFactory) throws Exception {
		List beanPostProcessors = beanFactory.getBeansForType(BeanPostProcessor.class);
		for (Object beanPostProcessor : beanPostProcessors) {
			beanFactory.addBeanPostProcessor((BeanPostProcessor) beanPostProcessor);
		}
	}
}
```

getBeansForType遍历beanDefinitionNames，　调用getBean获取对应实例返回．
后面会分析getBean方法．
```java
public abstract class AbstractBeanFactory implements BeanFactory {

	private Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>();

	private final List<String> beanDefinitionNames = new ArrayList<String>();

	private List<BeanPostProcessor> beanPostProcessors = new ArrayList<BeanPostProcessor>();

	public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) throws Exception {
		this.beanPostProcessors.add(beanPostProcessor);
	}

	public List getBeansForType(Class type) throws Exception {
		List beans = new ArrayList<Object>();
		for (String beanDefinitionName : beanDefinitionNames) {
			if (type.isAssignableFrom(beanDefinitionMap.get(beanDefinitionName).getBeanClass())) {
				beans.add(getBean(beanDefinitionName));//调用getBean获取bean实例
			}
		}
		return beans;
	}
}
```

### onRefresh方法
```java
public abstract class AbstractApplicationContext implements ApplicationContext {
	protected AbstractBeanFactory beanFactory;

	public AbstractApplicationContext(AbstractBeanFactory beanFactory) {
		this.beanFactory = beanFactory;
	}

	public void refresh() throws Exception {
		loadBeanDefinitions(beanFactory);
		registerBeanPostProcessors(beanFactory);
		onRefresh();
	}

	protected void onRefresh() throws Exception{
        beanFactory.preInstantiateSingletons();
    }
 }
```

preInstantiateSingletons调用 getBean方法实例化．
```java
public abstract class AbstractBeanFactory implements BeanFactory {
	private final List<String> beanDefinitionNames = new ArrayList<String>();

	public void preInstantiateSingletons() throws Exception {
		for (Iterator it = this.beanDefinitionNames.iterator(); it.hasNext();) {
			String beanName = (String) it.next();
			getBean(beanName);
		}
	}
}
```

## ClassPathXmlApplicationContext#getBean方法
```java
public abstract class AbstractApplicationContext implements ApplicationContext {
	protected AbstractBeanFactory beanFactory;

	@Override
	public Object getBean(String name) throws Exception {
		return beanFactory.getBean(name);
	}
}
```

getBean方法，根据name查找，如果找到直接返回；否则，返回创建的实例．

###doCreateBean方法
doCreateBean方法，首先创建实例，之后给实例中的property赋值．
1) 如果bean实现了BeanFactoryAware接口，调用回调方法; 
2) 如果是属性是基本类型，通过反射给属性赋值；
3) 如果属性是BeanReference类型，需要首先实例化依赖的Bean，之后再通过反射给属性赋值；
```java
public abstract class AbstractBeanFactory implements BeanFactory {

	private Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>();

	private final List<String> beanDefinitionNames = new ArrayList<String>();

	private List<BeanPostProcessor> beanPostProcessors = new ArrayList<BeanPostProcessor>();

	@Override
	public Object getBean(String name) throws Exception {
		BeanDefinition beanDefinition = beanDefinitionMap.get(name);
		if (beanDefinition == null) {
			throw new IllegalArgumentException("No bean named " + name + " is defined");
		}
		Object bean = beanDefinition.getBean();
		if (bean == null) {
			bean = doCreateBean(beanDefinition);
            bean = initializeBean(bean, name);
            beanDefinition.setBean(bean);
		}
		return bean;
	}

	protected Object initializeBean(Object bean, String name) throws Exception {
		for (BeanPostProcessor beanPostProcessor : beanPostProcessors) {
			bean = beanPostProcessor.postProcessBeforeInitialization(bean, name);
		}

		// TODO:call initialize method
		for (BeanPostProcessor beanPostProcessor : beanPostProcessors) {
            bean = beanPostProcessor.postProcessAfterInitialization(bean, name);
		}
        return bean;
	}

	protected Object createBeanInstance(BeanDefinition beanDefinition) throws Exception {
		return beanDefinition.getBeanClass().newInstance();
	}

	protected Object doCreateBean(BeanDefinition beanDefinition) throws Exception {
		Object bean = createBeanInstance(beanDefinition);
		beanDefinition.setBean(bean);
		applyPropertyValues(bean, beanDefinition);
		return bean;
	}

	protected void applyPropertyValues(Object bean, BeanDefinition beanDefinition) throws Exception {

	}
}

public class AutowireCapableBeanFactory extends AbstractBeanFactory {

	protected void applyPropertyValues(Object bean, BeanDefinition mbd) throws Exception {
		if (bean instanceof BeanFactoryAware) {　//如果bean实现了BeanFactoryAware接口，调用接口
			((BeanFactoryAware) bean).setBeanFactory(this);
		}
		for (PropertyValue propertyValue : mbd.getPropertyValues().getPropertyValues()) {
			Object value = propertyValue.getValue();
			if (value instanceof BeanReference) {
				BeanReference beanReference = (BeanReference) value;
				value = getBean(beanReference.getName());
			}

			try {
				Method declaredMethod = bean.getClass().getDeclaredMethod(
						"set" + propertyValue.getName().substring(0, 1).toUpperCase()
								+ propertyValue.getName().substring(1), value.getClass());
				declaredMethod.setAccessible(true);

				declaredMethod.invoke(bean, value);
			} catch (NoSuchMethodException e) {
				Field declaredField = bean.getClass().getDeclaredField(propertyValue.getName());
				declaredField.setAccessible(true);
				declaredField.set(bean, value);
			}
		}
	}
}
```

### initializeBean方法
遍历beanPostProcessors队列，执行回调方法postProcessBeforeInitialization，　postProcessAfterInitialization．