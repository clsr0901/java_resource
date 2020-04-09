### AOP的入口`@EnableAspectJAutoProxy`定义

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
	//是否使用CGLib代理，ture 使用CGLib代理 false 使用JDK动态代理
	//默认false
	boolean proxyTargetClass() default false;
	//是否暴露改代理对象，默认false不暴露
	boolean exposeProxy() default false;
}
```

* EnableAspectJAutoProxy是一个注解类，有两个属性

* EnableAspectJAutoProxy 的核心在`@Import`中的 `AspectJAutoProxyRegistrar` 类

* EnableAspectJAutoProxy 通过`@Import`导入` AspectJAutoProxyRegistrar `类、

  为什么能导入？因为 `AspectJAutoProxyRegistrar`是`ImportBeanDefinitionRegistrar`的子类，所以能被`@Import`导入，可以看下 `@Import`定义.`@Import` 能够导入`ImportSelector`和`ImportBeanDefinitionRegistrar`的实现类

### ---------------------------------------下面的是实例化对象时进入的方法 ---------------------------------------------

### AspectJAutoProxyRegistrar定义

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
	@Override
	public void registerBeanDefinitions(
			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		//这里是核心，会注册一个 AutoProxyCreator 类到容器中
        //所有的代理都会通过 AutoProxyCreator 来实现，跟进去
		AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
		//下面是获取EnableAspectJAutoProxy注解的属性
        //设置是否需要使用CGLib动态代理和是否需要暴露代理对象
		AnnotationAttributes enableAspectJAutoProxy =
				AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
		if (enableAspectJAutoProxy != null) {
			if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
			}
			if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
				AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
			}
		}
	}
}

//这是一个空壳方法
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
    //跟进去
    return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, null);
}
//还是一个空壳方法
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
    BeanDefinitionRegistry registry, @Nullable Object source) {
	//跟进去
    //注意这里传入了 AnnotationAwareAspectJAutoProxyCreator 这个类
    return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}
```

#### registerOrEscalateApcAsRequired 方法

```java
private static BeanDefinition registerOrEscalateApcAsRequired(
    Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");

    if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
        BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
        if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
            int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
            int requiredPriority = findPriorityForClass(cls);
            if (currentPriority < requiredPriority) {
                apcDefinition.setBeanClassName(cls.getName());
            }
        }
        return null;
    }
	//正常情况走这里
    //注意这里生成的是一个 RootBeanDefinition 类，Spring内部定义的都是RootBeanDefinition
    //代码很简单就是直接注册一个 RootBeanDefinition 类
    RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
    beanDefinition.setSource(source);
    //这里有一个order属性，设置的值为Integer.MIN_VALUE，越小越先执行
    //表示这个RootBeanDefinition要最早被实例化
    beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
    beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
    return beanDefinition;
}
```

### 被注入的`AnnotationAwareAspectJAutoProxyCreator`

* AnnotationAwareAspectJAutoProxyCreator 自己内部没有什么实现，重点看他的父类AbstractAdvisorAutoProxyCreator
* 需要记住的是 AnnotationAwareAspectJAutoProxyCreator  是一个 SmartInstantiationAwareBeanPostProcessor
  1. SmartInstantiationAwareBeanPostProcessor 是 InstantiationAwareBeanPostProcessor 的子类
  2. InstantiationAwareBeanPostProcessor  是 BeanPostProcessor 的子类

#### AbstractAutoProxyCreator 

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware{
    
    //后置处理会先进入这个方法
    public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
        if (bean != null) {
            //检查缓存
            Object cacheKey = getCacheKey(bean.getClass(), beanName);
            if (!this.earlyProxyReferences.contains(cacheKey)) {
                //重点在这里，跟进去
                return wrapIfNecessary(bean, beanName, cacheKey);
            }
        }
        return bean;
    }
    
    
    protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
        if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
            return bean;
        }
        if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
            return bean;
        }
        if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return bean;
        }

        // Create proxy if we have advice.
        //获取代理的信息
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
        if (specificInterceptors != DO_NOT_PROXY) {
            this.advisedBeans.put(cacheKey, Boolean.TRUE);
            //创建代理对象
            Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            this.proxyTypes.put(cacheKey, proxy.getClass());
            return proxy;
        }

        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
    
    
}
```

#### getAdvicesAndAdvisorsForBean 方法

```java
//获取类的代理方法
protected Object[] getAdvicesAndAdvisorsForBean(
    Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
    //重点是这个
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}
```

#### findEligibleAdvisors 方法

```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    //这里会把所有的
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}
```

