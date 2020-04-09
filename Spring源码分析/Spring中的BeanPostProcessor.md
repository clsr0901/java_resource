### BeanPostProcessor 定义

```java
public interface BeanPostProcessor {
	//这个方法会在bean实例化后，调用初始化方法之前执行
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
    //这个方法会在bean实例化后，调用初始化方法之后执行
    //这里的before和after是相对于初始化方法的调用
	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
```

* 从定义中可以看出BeanPostProcessor是一个接口
* BeanPostProcessor定义了两个方法，都是default修饰的。

### BeanPostProcessor 使用

#### Dao 类 

```java
public interface IDao {
	void query();
}

@Repository("indexDao")
public class IndexDao implements IDao {
	@Override
	public void query() {
		System.out.println("index dao");
	}
}
```

#### InvocationHandler 类

```java
public class InvocationHandlerTest implements InvocationHandler{

	private Object target;

	public InvocationHandlerTest(Object target) {
		this.target = target;
	}

	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		System.out.println("invoke proxy");
		return method.invoke(target, args);
	}
}
```

#### BeanPostProcessorTest 类

```java
@Component
public class BeanPostProcessorTest implements BeanPostProcessor {
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (beanName.equals("indexDao")) {
			System.out.println("before");
            //使用JDK动态代理，返回一个代理类
			return Proxy.newProxyInstance(bean.getClass().getClassLoader(), bean.getClass().getInterfaces(), new InvocationHandlerTest(bean));
		}
		return bean;
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (beanName.equals("indexDao")) {
			System.out.println("after");
		}
		return bean;
	}
}
```

#### Appconfig 类

```java
@Configuration
@ComponentScan("com.ktcatv")
public class Appconfig {
}
```

#### Test 类

```java
public class Test {

	public static void main(String[] args) {
		AnnotationConfigApplicationContext configApplicationContext = new AnnotationConfigApplicationContext(Appconfig.class);
        //这里打断点可以看出，这个IDao对象是一个代理类
        //说明原始对象在BeanPostProcessorTest中被替换了
		IDao dao = (IDao) configApplicationContext.getBean("indexDao");
		dao.query();

	}
}
```





