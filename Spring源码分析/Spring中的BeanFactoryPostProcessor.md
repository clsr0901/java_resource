### BeanFactoryPostProcessor的定义

```java
@FunctionalInterface
public interface BeanFactoryPostProcessor {
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}

```

* 从定义中可以看出BeanFactoryPostProcessor是一个函数式接口
* BeanFactoryPostProcessor只有一个方法，这个方法返回了一个BeanFactory对象，这个对象可以理解为Spring中的容器。
* postProcessBeanFactory是在注册完BeanDefinition后回调，这个回调中可以修改BeanDefinition的属性从而达到修改实例化的Bean

#### BeanFactoryPostProcessor的拓展类---BeanDefinitionRegistryPostProcessor

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}
```

* BeanDefinitionRegistryPostProcessor继承了BeanFactoryPostProcessor，表示BeanDefinitionRegistryPostProcessor也是一个BeanFactoryPostProcessor
* BeanDefinitionRegistryPostProcessor是一个接口，新增了一个方法
* BeanDefinitionRegistryPostProcessor有一个重要的实现类ConfigurationClassPostProcessor

#### ConfigurationClassPostProcessor 定义

```java
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
		PriorityOrdered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware
```

* 主要关心实现BeanDefinitionRegistryPostProcessor接口的方法postProcessBeanDefinitionRegistry

##### postProcessBeanDefinitionRegistry 方法

```java
//这个是一个空壳方法
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    int registryId = System.identityHashCode(registry);
    if (this.registriesPostProcessed.contains(registryId)) {
        throw new IllegalStateException(
            "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
    }
    if (this.factoriesPostProcessed.contains(registryId)) {
        throw new IllegalStateException(
            "postProcessBeanFactory already called on this post-processor against " + registry);
    }
    this.registriesPostProcessed.add(registryId);
	//核心在这，跟进去
    processConfigBeanDefinitions(registry);
}

```

##### processConfigBeanDefinitions 方法

```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    //定义一个list存放app 提供的bd（项目当中提供了@Compent）
    List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
    //获取容器中注册的所有bd名字
    //7个,系统六个加上一个自己的配置类一共七个
    String[] candidateNames = registry.getBeanDefinitionNames();

    /**
		 * Full的配置类会被代理
		 * Lite的配置类不会被代理
		 */
    for (String beanName : candidateNames) {
        BeanDefinition beanDef = registry.getBeanDefinition(beanName);
        if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
            ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
            //如果BeanDefinition中的configurationClass属性为full或者lite,则意味着已经处理过了,直接跳过
            //这里需要结合下面的代码才能理解
            if (logger.isDebugEnabled()) {
                logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
            }
        }
        //判断是否是Configuration类，如果加了Configuration下面的这几个注解就不再判断了
        // 还有  add(Component.class.getName());
        //		candidateIndicators.add(ComponentScan.class.getName());
        //		candidateIndicators.add(Import.class.getName());
        //		candidateIndicators.add(ImportResource.class.getName());
        //beanDef == appconfig
        else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
            //BeanDefinitionHolder 也可以看成一个数据结构
            configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
        }
    }

    // Return immediately if no @Configuration classes were found
    if (configCandidates.isEmpty()) {
        return;
    }

    // 排序，根据order,不重要
    // Sort by previously determined @Order value, if applicable
    configCandidates.sort((bd1, bd2) -> {
        int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
        int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
        return Integer.compare(i1, i2);
    });

    // Detect any custom bean name generation strategy supplied through the enclosing application context
    SingletonBeanRegistry sbr = null;
    //如果BeanDefinitionRegistry是SingletonBeanRegistry子类的话,
    // 由于我们当前传入的是DefaultListableBeanFactory,是SingletonBeanRegistry 的子类
    // 因此会将registry强转为SingletonBeanRegistry
    if (registry instanceof SingletonBeanRegistry) {
        sbr = (SingletonBeanRegistry) registry;
        if (!this.localBeanNameGeneratorSet) {//是否有自定义的
            BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
            //SingletonBeanRegistry中有id为 org.springframework.context.annotation.internalConfigurationBeanNameGenerator
            //如果有则利用他的，否则则是spring默认的
            if (generator != null) {
                this.componentScanBeanNameGenerator = generator;
                this.importBeanNameGenerator = generator;
            }
        }
    }

    if (this.environment == null) {
        this.environment = new StandardEnvironment();
    }

    // Parse each @Configuration class
    //实例化ConfigurationClassParser 为了解析各个配置类
    ConfigurationClassParser parser = new ConfigurationClassParser(
        this.metadataReaderFactory, this.problemReporter, this.environment,
        this.resourceLoader, this.componentScanBeanNameGenerator, registry);

    //实例化2个set,candidates用于将之前加入的configCandidates进行去重
    //因为可能有多个配置类重复了
    //alreadyParsed用于判断是否处理过
    Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
    Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
    do {
        //candidates中存放的是配置类，这里解析配置类
        //这里很重要
        parser.parse(candidates);
        parser.validate();
        //map.keyset
        Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
        configClasses.removeAll(alreadyParsed);

        // Read the model and create bean definitions based on its content
        if (this.reader == null) {
            this.reader = new ConfigurationClassBeanDefinitionReader(
                registry, this.sourceExtractor, this.resourceLoader, this.environment,
                this.importBeanNameGenerator, parser.getImportRegistry());
        }

        /**
			 * 这里值得注意的是扫描出来的bean当中可能包含了特殊类
			 * 比如ImportBeanDefinitionRegistrar那么也在这个方法里面处理
			 * 但是并不是包含在configClasses当中
			 * configClasses当中主要包含的是importSelector
			 * 因为ImportBeanDefinitionRegistrar在扫描出来的时候已经被添加到一个list当中去了
			 * 上面解析出来的@Bean和@Import（三种）都会放入configClasses中
			 */
        //bd 到 map 除却普通
        //跟进去
        this.reader.loadBeanDefinitions(configClasses);
        alreadyParsed.addAll(configClasses);

        candidates.clear();
        //由于我们这里进行了扫描，把扫描出来的BeanDefinition注册给了factory
        if (registry.getBeanDefinitionCount() > candidateNames.length) {
            String[] newCandidateNames = registry.getBeanDefinitionNames();
            Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
            Set<String> alreadyParsedClasses = new HashSet<>();
            for (ConfigurationClass configurationClass : alreadyParsed) {
                alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
            }
            for (String candidateName : newCandidateNames) {
                if (!oldCandidateNames.contains(candidateName)) {
                    BeanDefinition bd = registry.getBeanDefinition(candidateName);
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                        !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                        candidates.add(new BeanDefinitionHolder(bd, candidateName));
                    }
                }
            }
            candidateNames = newCandidateNames;
        }
    }
    while (!candidates.isEmpty());

    // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
    if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
        sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
    }

    if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
        // Clear cache in externally provided MetadataReaderFactory; this is a no-op
        // for a shared cache since it'll be cleared by the ApplicationContext.
        ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
    }
}

```

##### checkConfigurationClassCandidate 方法

```java
//如果传入的BeanDefinition有Configuration注解，把BeanDefinition的CONFIGURATION_CLASS_ATTRIBUTE属性设置为full，这个属性在实例化的时候会生成代理对象
//如果传入的BeanDefinition有Component、ComponentScan、Import、ImportResource注解，把BeanDefinition的CONFIGURATION_CLASS_ATTRIBUTE属性设置为lite
public static boolean checkConfigurationClassCandidate(BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {
		String className = beanDef.getBeanClassName();
		if (className == null || beanDef.getFactoryMethodName() != null) {
			return false;
		}

		AnnotationMetadata metadata;
		// 配置类默认为AnnotatedGenericBeanDefinition对象
		if (beanDef instanceof AnnotatedBeanDefinition &&
				className.equals(((AnnotatedBeanDefinition) beanDef).getMetadata().getClassName())) {
			// Can reuse the pre-parsed metadata from the given BeanDefinition...
			//如果BeanDefinition 是 AnnotatedBeanDefinition的实例,并且className 和 BeanDefinition中 的元数据 的类名相同
			// 则直接从BeanDefinition 获得Metadata
			metadata = ((AnnotatedBeanDefinition) beanDef).getMetadata();
		}
		else if (beanDef instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) beanDef).hasBeanClass()) {
			// Check already loaded Class if present...
			// since we possibly can't even load the class file for this Class.
			//如果BeanDefinition 是 AbstractBeanDefinition的实例,并且beanDef 有 beanClass 属性存在
			//则实例化StandardAnnotationMetadata
			Class<?> beanClass = ((AbstractBeanDefinition) beanDef).getBeanClass();
			metadata = new StandardAnnotationMetadata(beanClass, true);
		}
		else {
			try {
				MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(className);
				metadata = metadataReader.getAnnotationMetadata();
			}
			catch (IOException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Could not find class file for introspecting configuration annotations: " + className, ex);
				}
				return false;
			}
		}
		//判断当前这个bd中存在的类是不是加了@Configruation注解
		//如果存在则spring认为他是一个全注解的类
		if (isFullConfigurationCandidate(metadata)) {
			//如果存在 Configuration 注解,则为BeanDefinition 设置configurationClass属性为full
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
		}
		//判断是否加了以下注解，摘录isLiteConfigurationCandidate的源码
		//     candidateIndicators.add(Component.class.getName());
		//		candidateIndicators.add(ComponentScan.class.getName());
		//		candidateIndicators.add(Import.class.getName());
		//		candidateIndicators.add(ImportResource.class.getName());
		//如果不存在Configuration注解，spring则认为是一个部分注解类
		else if (isLiteConfigurationCandidate(metadata)) {
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
		}
		else {
			return false;
		}

		// It's a full or lite configuration candidate... Let's determine the order value, if any.
		Integer order = getOrder(metadata);
		if (order != null) {
			beanDef.setAttribute(ORDER_ATTRIBUTE, order);
		}

		return true;
	}
```

##### parse 方法

```java
//解析配置类，主要是把Configuration注解类里面导入的类解析为对应的BeanDefinition并注册到容器中去
//注意这个configCandidates里面只有自己的配置类
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    this.deferredImportSelectors = new LinkedList<>();
    //根据BeanDefinition 的类型 做不同的处理,一般都会调用ConfigurationClassParser#parse 进行解析
    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        try {
            if (bd instanceof AnnotatedBeanDefinition) {
                //解析注解对象，并且把解析出来的bd放到map，但是这里的bd指的是普通的
                //何谓不普通的呢？比如@Bean 和各种beanFactoryPostProcessor得到的bean不在这里put
                //但是是这里解析，只是不put而已
                parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
            }
            else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
            }
            else {
                parse(bd.getBeanClassName(), holder.getBeanName());
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
        }
    }

    //处理延迟加载的importSelect？跳过
    processDeferredImportSelectors();
}

//这个是空壳方法，接着跟进去
protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
    processConfigurationClass(new ConfigurationClass(metadata, beanName));
}
```

##### processConfigurationClass 方法

```java
//处理配置类
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    //判断是否应该跳过解析
    if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
        return;
    }

    // 处理Imported 的情况
    //就是当前这个注解类有没有被别的类import
    ConfigurationClass existingClass = this.configurationClasses.get(configClass);
    if (existingClass != null) {
        if (configClass.isImported()) {
            if (existingClass.isImported()) {
                existingClass.mergeImportedBy(configClass);
            }
            // Otherwise ignore new imported config class; existing non-imported class overrides it.
            return;
        }
        else {
            // Explicit bean definition found, probably replacing an imports.
            // Let's remove the old one and go with the new one.
            this.configurationClasses.remove(configClass);
            this.knownSuperclasses.values().removeIf(configClass::equals);
        }
    }

    // Recursively process the configuration class and its superclass hierarchy.
    SourceClass sourceClass = asSourceClass(configClass);
    do {
        //这个里面会注册BeanDefinition，重点看这步
        sourceClass = doProcessConfigurationClass(configClass, sourceClass);
    }
    while (sourceClass != null);
    //一个map，用来存放扫描出来的bean（注意这里的bean不是对象，仅仅bean的信息，因为还没到实例化这一步）
    this.configurationClasses.put(configClass, configClass);
}
```

##### doProcessConfigurationClass 方法

```java
//真正解析配置类的方法
//注意这个ConfigurationClass对象是从ConfigurationClassPostProcessor中传过来的
//@Bean、@Import（三种）解析出来的类都放入ConfigurationClass对象中
//ConfigurationClassPostProcessor会调用this.reader.loadBeanDefinitions(configClasses)方法来统一解析
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
    throws IOException {

    // Recursively process any member (nested) classes first
    //处理内部类，跳过
    processMemberClasses(configClass, sourceClass);

    // Process any @PropertySource annotations
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
        sourceClass.getMetadata(), PropertySources.class,
        org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        }
        else {
            logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                        "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }

    // Process any @ComponentScan annotations
    //处理所有@ComponentScan注解
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
        sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    if (!componentScans.isEmpty() &&
        !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            //扫描普通类=componentScan=com.test
            //这里扫描出来所有@@Component
            //并且把扫描的出来的普通bean放到map当中
            //在这注册普通的BeanDefinition,跟进去
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            //检查扫描出来的类当中是否还有configuration
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                //检查  todo
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    //如果扫描@ComponentScan里面还有配置类，进行递归，回到上面的parse方法
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    /**
		 * 上面的代码就是扫描普通类----@Component
		 * 并且放到了map当中
		 */
    // Process any @Import annotations
    //处理@Import  imports 3种情况
    //ImportSelector
    //普通类
    //ImportBeanDefinitionRegistrar
    //这里和内部地柜调用时候的情况不同
    /**
		 * 这里处理的import是需要判断我们的类当中时候有@Import注解
		 * 如果有这把@Import当中的值拿出来，是一个类
		 * 比如@Import(xxxxx.class)，那么这里便把xxxxx传进去进行解析
		 * 在解析的过程中如果发觉是一个importSelector那么就回调selector的方法
		 * 返回一个字符串（类名），通过这个字符串得到一个类
		 * 继而在递归调用本方法来处理这个类
		 *
		 * 判断一组类是不是imports（3种import）
		 *
		 *
		 */
    //跟进去
    processImports(configClass, sourceClass, getImports(sourceClass), true);

    // Process any @ImportResource annotations
    AnnotationAttributes importResource =
        AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
    if (importResource != null) {
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    // Process individual @Bean methods
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    // Process default methods on interfaces

    processInterfaces(configClass, sourceClass);

    // Process superclass, if any
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (superclass != null && !superclass.startsWith("java") &&
            !this.knownSuperclasses.containsKey(superclass)) {
            this.knownSuperclasses.put(superclass, configClass);
            // Superclass found, return its annotation metadata and recurse
            return sourceClass.getSuperClass();
        }
    }

    // No superclass -> processing is complete
    return null;
}
```

##### parse 方法

```java
public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
    //这里的scanner是重新创建的，不是初始化AnnotationConfigApplicationContext时创建的对象
    ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
                                                                                componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);

    //BeanNameGenerator
    Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
    boolean useInheritedGenerator = (BeanNameGenerator.class == generatorClass);
    scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator :
                                 BeanUtils.instantiateClass(generatorClass));

    //web应用的，跳过
    ScopedProxyMode scopedProxyMode = componentScan.getEnum("scopedProxy");
    if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
        scanner.setScopedProxyMode(scopedProxyMode);
    }
    else {
        Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
        scanner.setScopeMetadataResolver(BeanUtils.instantiateClass(resolverClass));
    }

    scanner.setResourcePattern(componentScan.getString("resourcePattern"));

    //遍历当中的过滤
    for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
        for (TypeFilter typeFilter : typeFiltersFor(filter)) {
            scanner.addIncludeFilter(typeFilter);
        }
    }
    for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
        for (TypeFilter typeFilter : typeFiltersFor(filter)) {
            scanner.addExcludeFilter(typeFilter);
        }
    }

    //默认false
    boolean lazyInit = componentScan.getBoolean("lazyInit");
    if (lazyInit) {
        scanner.getBeanDefinitionDefaults().setLazyInit(true);
    }

    //获取@ComponentScan注解的值
    Set<String> basePackages = new LinkedHashSet<>();
    String[] basePackagesArray = componentScan.getStringArray("basePackages");
    for (String pkg : basePackagesArray) {
        String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
                                                               ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
        Collections.addAll(basePackages, tokenized);
    }
    for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
        basePackages.add(ClassUtils.getPackageName(clazz));
    }
    if (basePackages.isEmpty()) {
        basePackages.add(ClassUtils.getPackageName(declaringClass));
    }

    scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
        @Override
        protected boolean matchClassName(String className) {
            return declaringClass.equals(className);
        }
    });
	//这个是核心，扫描路径并把符合条件的类转换为BeanDefinition对象
    return scanner.doScan(StringUtils.toStringArray(basePackages));
}
```

##### doScan 方法

```java
//扫描路径
//把符合条件的类转换为对应的BeanDefinition对象
//为BeanDefinition对象赋值
//把BeanDefinition对象注册到容器中
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
        //扫描basePackage路径下的java文件
        //符合条件的并把它转成BeanDefinition类型，跟进去
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);

        for (BeanDefinition candidate : candidates) {
            //解析scope属性
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
            if (candidate instanceof AbstractBeanDefinition) {
                //如果这个类是AbstractBeanDefinition的子类
                //则为他设置默认值，比如lazy，init destory
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
            }
            if (candidate instanceof AnnotatedBeanDefinition) {
                //检查并且处理常用的注解
                //这里的处理主要是指把常用注解的值设置到AnnotatedBeanDefinition当中
                //当前前提是这个类必须是AnnotatedBeanDefinition类型的，说白了就是加了注解的类
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
            }
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                definitionHolder =
                    AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
                //加入到map当中，这里是注册功能把BeanDefinition注册到容器中
                registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }
    return beanDefinitions;
}
```

##### findCandidateComponents 方法

```java
//扫描包，空壳方法
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
        return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
    }
    else {
        //看这个
        return scanCandidateComponents(basePackage);
    }
}
```

##### scanCandidateComponents 方法

```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
    Set<BeanDefinition> candidates = new LinkedHashSet<>();
    try {
        //这一步会把packageSearchPath的值修改为 classpath*:com/ktcatv/**/*.class
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
            resolveBasePackage(basePackage) + '/' + this.resourcePattern;
        //asm 读取class文件生成Resource对象，跳过吧，内容太多了不是重点
        Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
        boolean traceEnabled = logger.isTraceEnabled();
        boolean debugEnabled = logger.isDebugEnabled();
        for (Resource resource : resources) {
            if (traceEnabled) {
                logger.trace("Scanning " + resource);
            }
            if (resource.isReadable()) {
                try {
                    //这里可以把Resource对象转换为ScannedGenericBeanDefinition对象，只记住这点就行了
                    MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                    if (isCandidateComponent(metadataReader)) {
                        ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                        sbd.setResource(resource);
                        sbd.setSource(resource);
                        if (isCandidateComponent(sbd)) {
                            if (debugEnabled) {
                                logger.debug("Identified candidate component class: " + resource);
                            }
                            candidates.add(sbd);
                        }
                        else {
                            if (debugEnabled) {
                                logger.debug("Ignored because not a concrete top-level class: " + resource);
                            }
                        }
                    }
                    else {
                        if (traceEnabled) {
                            logger.trace("Ignored because not matching any filter: " + resource);
                        }
                    }
                }
                catch (Throwable ex) {
                    throw new BeanDefinitionStoreException(
                        "Failed to read candidate component class: " + resource, ex);
                }
            }
            else {
                if (traceEnabled) {
                    logger.trace("Ignored because not readable: " + resource);
                }
            }
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
    }
    //返回解析后的BeanDefinition集合
    return candidates;
}
```

* 回到doScan方法接着看

##### registerBeanDefinition 方法

```java
//注册BeanDefinition，空壳方法
protected void registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry) {
    //跟进去
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, registry);
}

//这个才是真实的注册BeanDefinition对象的方法
public static void registerBeanDefinition(
    BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
    throws BeanDefinitionStoreException {

    // Register bean definition under primary name.
    String beanName = definitionHolder.getBeanName();
    //这里注册BeanDefinition，这个跟进去
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // 注册别名，如果有的话
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            registry.registerAlias(beanName, alias);
        }
    }
}

//DefaultListabaleBeanFactory中的方法
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
    throws BeanDefinitionStoreException {

    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");

    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            ((AbstractBeanDefinition) beanDefinition).validate();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                                                   "Validation of bean definition failed", ex);
        }
    }

    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    if (existingDefinition != null) {
        if (!isAllowBeanDefinitionOverriding()) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                                                   "Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
                                                   "': There is already [" + existingDefinition + "] bound.");
        }
        else if (existingDefinition.getRole() < beanDefinition.getRole()) {
            // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
            if (logger.isWarnEnabled()) {
                logger.warn("Overriding user-defined bean definition for bean '" + beanName +
                            "' with a framework-generated bean definition: replacing [" +
                            existingDefinition + "] with [" + beanDefinition + "]");
            }
        }
        else if (!beanDefinition.equals(existingDefinition)) {
            if (logger.isInfoEnabled()) {
                logger.info("Overriding bean definition for bean '" + beanName +
                            "' with a different definition: replacing [" + existingDefinition +
                            "] with [" + beanDefinition + "]");
            }
        }
        else {
            if (logger.isDebugEnabled()) {
                logger.debug("Overriding bean definition for bean '" + beanName +
                             "' with an equivalent definition: replacing [" + existingDefinition +
                             "] with [" + beanDefinition + "]");
            }
        }
        this.beanDefinitionMap.put(beanName, beanDefinition);
    }
    else {
        if (hasBeanCreationStarted()) {
            //这里是处理已经被注册的逻辑，应该是被其他的BeanFactoryPostProcessor修改后进入这里
            // Cannot modify startup-time collection elements anymore (for stable iteration)
            synchronized (this.beanDefinitionMap) {
                this.beanDefinitionMap.put(beanName, beanDefinition);
                List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                if (this.manualSingletonNames.contains(beanName)) {
                    Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
                    updatedSingletons.remove(beanName);
                    this.manualSingletonNames = updatedSingletons;
                }
            }
        }
        else {
            // 这里是正常的逻辑，注册BeanDefinition就是放进beanDefinitionMap对象中
            this.beanDefinitionMap.put(beanName, beanDefinition);
            this.beanDefinitionNames.add(beanName);
            this.manualSingletonNames.remove(beanName);
        }
        this.frozenBeanDefinitionNames = null;
    }

    if (existingDefinition != null || containsSingleton(beanName)) {
        resetBeanDefinition(beanName);
    }
}
```

* 回到 doProcessConfigurationClass 方法继续看

##### processImports 方法

```java
//处理@Import注解，这个后面会重新开一篇来讲解
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
                            Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

    if (importCandidates.isEmpty()) {
        return;
    }

    if (checkForCircularImports && isChainedImportOnStack(configClass)) {
        this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
    }
    else {
        this.importStack.push(configClass);
        try {
            for (SourceClass candidate : importCandidates) {
                //处理ImportSelector类
                if (candidate.isAssignable(ImportSelector.class)) {
                    // Candidate class is an ImportSelector -> delegate to it to determine imports
                    Class<?> candidateClass = candidate.loadClass();
                    //反射实现一个对象
                    ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                    ParserStrategyUtils.invokeAwareMethods(
                        selector, this.environment, this.resourceLoader, this.registry);
                    if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
                        this.deferredImportSelectors.add(
                            new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
                    }
                    else {
                        //回调
                        String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                        Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                        //递归，这里第二次调用processImports
                        //如果是一个普通类，会进else
                        processImports(configClass, currentSourceClass, importSourceClasses, false);
                    }
                }
                //处理ImportBeanDefinitionRegistrar类
                else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                    // Candidate class is an ImportBeanDefinitionRegistrar ->
                    // delegate to it to register additional bean definitions
                    Class<?> candidateClass = candidate.loadClass();
                    ImportBeanDefinitionRegistrar registrar =
                        BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                    ParserStrategyUtils.invokeAwareMethods(
                        registrar, this.environment, this.resourceLoader, this.registry);
                    //添加到一个list当中和importselector不同
                    configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                }
                else {
                    // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                    // process it as an @Configuration class
                    // 否则，加入到importStack后调用processConfigurationClass 进行处理
                    //processConfigurationClass里面主要就是把类放到configurationClasses
                    //configurationClasses是一个集合，会在后面拿出来解析成bd继而注册
                    //可以看到普通类在扫描出来的时候就被注册了
                    //如果是importSelector，会先放到configurationClasses后面进行出来注册
                    this.importStack.registerImport(
                        currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                    processConfigurationClass(candidate.asConfigClass(configClass));
                }
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                "Failed to process imports candidates for configuration class [" +
                configClass.getMetadata().getClassName() + "]", ex);
        }
        finally {
            this.importStack.pop();
        }
    }
}
```

* 回到 processConfigBeanDefinitions 方法继续

##### loadBeanDefinitions 方法

```java
//空壳方法
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
		TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
		for (ConfigurationClass configClass : configurationModel) {
            //跟进去
			loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
		}
	}
```

##### loadBeanDefinitionsForConfigurationClass 方法

```java
private void loadBeanDefinitionsForConfigurationClass(
			ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

		if (trackedConditionEvaluator.shouldSkip(configClass)) {
			String beanName = configClass.getBeanName();
			if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
				this.registry.removeBeanDefinition(beanName);
			}
			this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
			return;
		}

		//如果一个类是被import的，会被spring标记
		//在这里完成注册
		if (configClass.isImported()) {
			registerBeanDefinitionForImportedConfigurationClass(configClass);
		}
		//@Bean
		for (BeanMethod beanMethod : configClass.getBeanMethods()) {
			loadBeanDefinitionsForBeanMethod(beanMethod);
		}

		 //xml
		loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());

		//注册Registrar
		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
	}
```

* 到这就把配置文件解析完成了

####  ConfigurationClassPostProcessor 小结

* ConfigurationClassPostProcessor 是Spring内部最重要的一个BeanDefinitionRegistryPostProcesso的实现类，它的主要工作就是解析配置文件
* ConfigurationClassPostProcessor 工作流程
  * 1. 从容器中获取所有的BeanDefinition对象
    2. 筛选出需要解析的配置类的BeanDefinition对象
    3. 遍历筛选出来的BeanDefinition对象，进行解析
    4. 解析@ComponentScan获取路径
    5. 扫描路径下符合条件的类，把符合条件的类转换为对应的BeanDefinition，这里不会处理@Bean和@Import只会添加到ConfigurationClass对象中
    6. 注册转换的BeanDefinition对象
    7. 使用 ConfigurationClassBeanDefinitionReader 加载上面扫描出来的其他类（@Bean和@Import）转换为BeanDefinition并注册到容器中

### BeanFactoryPostProcessor 的自定义使用

#### Dao类

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

#### Service类

```java
public interface IService {
	void service();
}

//注意这里没有注解，不能被Spring扫描到
public class IndexService implements IService {
	@Override
	public void service() {
		System.out.println("service");
	}
}
```

#### Appconfig类

```java
@Configuration
@ComponentScan("com.ktcatv")
public class Appconfig {
}
```

#### BeanFactoryPostPorcessor类

```java
@Component
public class BeanFactoryPostPorcessorTest implements BeanFactoryPostProcessor {
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        //这里获取了IndexDao的BeanDefinition对象，修改了它的beanCalss属性
		BeanDefinition indexDao = beanFactory.getBeanDefinition("indexDao");
		indexDao.setBeanClassName(IndexService.class.getName());
	}
}
```

#### Test类

```java
public class Test {
	public static void main(String[] args) {
		AnnotationConfigApplicationContext configApplicationContext = new AnnotationConfigApplicationContext(Appconfig.class);
        //可以正常执行，说明上面的BeanFactoryPostPorcessorTest类成功修改了 IndexDao 类的实例化过程
		IService service = configApplicationContext.getBean(IService.class);
		service.service();
	}
}
```

