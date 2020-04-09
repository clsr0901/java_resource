### FactoryBean 定义

```java
public interface FactoryBean<T> {
    //返回产生的对象
	@Nullable
	T getObject() throws Exception;
	//返回产生的对象类型
	@Nullable
	Class<?> getObjectType();
	//是否为单例对象 true 单例 false 原型
	default boolean isSingleton() {
		return true;
	}
}
```

* FactoryBean是一个泛型类的接口
* FactoryBean产生的对象默认是单例的

### FactoryBean 的使用

#### Service 类

```java
public interface IService {
	void service();
}

public class IndexService implements IService {
	@Override
	public void service() {
		System.out.println("service");
	}
}
```

#### FactoryBeanTest 类

```java
@Component
public class FactoryBeanTest implements FactoryBean{
	@Override
	public Object getObject() throws Exception {
		return new IndexService();
	}

	@Override
	public Class<?> getObjectType() {
		return IndexService.class;
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
        //注意如果直接使用名字不带 & 返回的是getObject创建的对象
        //如果带了 & 返回的是FactoryBean本身
        //所以FactoryBean会创建两个bean
		IService service = (IService) configApplicationContext.getBean("factoryBeanTest");
		service.service();
		Object bean = configApplicationContext.getBean("&factoryBeanTest");
		System.out.println(bean);

	}
}
```

