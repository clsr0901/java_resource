### 1. ClassPathXmlApplicationContext 构造方法

```java
//ClassPathXmlApplicationContext.java 82行
//configLocation 资源路径
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
    	//转调下面的构造方法
		this(new String[] {configLocation}, true, null);
	}

//ClassPathXmlApplicationContext.java 133行
//configLocations 传入资源路径的数组，表示可以传入多个资源路径
//refresh 是否自动刷新context，加载所有bean定义并创建所有单例，也可自己后面手动刷新，这里默认是自动刷新。
//parent context的父对象
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {
		//调用父类的构造函数
    	// 在 AbstractApplicationContext 类的216行 
    	//初始化了 this.resourcePatternResolver = getResourcePatternResolver();
    	//把传入的parent设置成父context
		super(parent);
    	//设置应用程序context的配置文件路径,如果不配置，使用默认值。现在我也不知道默认值是什么。下面详细介绍
    	//是父类AbstractRefreshableConfigApplicationContext的方法
		setConfigLocations(configLocations);
    	//refresh 这个方法是最重要的方法，整个ioc的功能都在refresh中
    	//可以看出这个refresh也有init的意思
		if (refresh) {
			refresh();
		}
	}
```

### 2. setConfigLocations 方法

```java
//AbstractRefreshableConfigApplicationContext.java 75行
//可以传入多个配置文件路径
//起始就是为configLocations赋值。 private String[] configLocations;
public void setConfigLocations(String... locations) {
		if (locations != null) {
			Assert.noNullElements(locations, "Config locations must not be null");
			this.configLocations = new String[locations.length];
			for (int i = 0; i < locations.length; i++) {
				this.configLocations[i] = resolvePath(locations[i]).trim();
			}
		}
		else {
			this.configLocations = null;
		}
	}
```

### 3. refresh 方法（很重要）

```java
//AbstractApplicationContext.java 509行
//可以看出这是父类的一个重写方法
@Override
	public void refresh() throws BeansException, IllegalStateException {
        //加锁，防止在刷新的过程中，其它线程又来了
         /** Synchronization monitor for the "refresh" and "destroy" */
		//private final Object startupShutdownMonitor = new Object();
		synchronized (this.startupShutdownMonitor) {
            //下面的几个方法接下来会一个一个来介绍
			//刷新context之前的准备方法，做一些标记和检验.
			prepareRefresh();

			// 让子类去刷新内部bean工厂
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			//准备bean工厂，设置类加载器等属性
			prepareBeanFactory(beanFactory);

			try {
				// 【这里需要知道 BeanFactoryPostProcessor 这个知识点，Bean 如果实现了此接口，
                 // 那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。】

                 // 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化
                 // 具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事
				postProcessBeanFactory(beanFactory);

				// 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 回调方法
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```

#### 1. prepareRefresh 方法

```java
//AbstractApplicationContext.java 577行
protected void prepareRefresh() {
    	//记录开始时间
		this.startupDate = System.currentTimeMillis();
    	//把当前上下文的关闭标志设置为false，这个closed是一个AtomicBoolean对象
		this.closed.set(false);
    	//把当前上下文的活动标志设置为true，这个active是一个AtomicBoolean对象
		this.active.set(true);
		//打印日志信息
		if (logger.isInfoEnabled()) {
			logger.info("Refreshing " + this);
		}

		// Initialize any placeholder property sources in the context environment
		initPropertySources();

		// 校验 xml 配置文件
		getEnvironment().validateRequiredProperties();

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
	}
```

这个方法主要是设置一些标记和检查配置，比较简单，接着往下看

#### 2. obtainFreshBeanFactory 方法

```java
//AbstractApplicationContext.java 613行
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    	//主要是这个方法，下面的getBeanFactory就是获取这个方法生成的beanFactroy
    	//这个需要子类来实现，当前类中是一个抽象方法
    	//继续跟进去看看
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```

##### refreshBeanFactory 方法

```java
//AbstractRefreshableApplicationContext.java 120行
//这个是刷新beanFactory的实际方法
@Override
	protected final void refreshBeanFactory() throws BeansException {
        //判断当前是否已经有beanFactory，如果有销毁所有的bean并关闭beanFactory
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
            //为当前context创建一个内部的beanFactory
            //就是使用父context对象创建了一个DefaultListableBeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
            //指定一个id进行序列化，应该使用不到这个功能，序列化BeanFactory？
			beanFactory.setSerializationId(getId());
            //这个很重要，定制beanFactory
            //设置beanFactory是否允许被覆盖、是否允许循环引用
			customizeBeanFactory(beanFactory);
            //加载bean到beanFactory中，这个也很重要
			loadBeanDefinitions(beanFactory);
            //把创建的beanFactory赋值给当前context
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

##### createBeanFactory 方法

```java
//AbstractRefreshableApplicationContext.java 199行
//为当前context创建一个内部的beanFactory
protected DefaultListableBeanFactory createBeanFactory() {
    	//使用父Context//初始化DefaultListableBeanFactory类
    	//这里为什么会用到DefaultListableBeanFactory？因为它实现了BeanFactory下一层所有的接口
    	//DefaultListableBeanFactory是最全的BeanFactory
    	//getInternalParentBeanFactory就是获取父类的context。
    	//这里啰嗦一句，ApplicationContext也是BeanFactory的子类。
    	//BeanFactory是最顶层的接口
		return new DefaultListableBeanFactory(getInternalParentBeanFactory());
	}
```

##### customizeBeanFactory 方法

```java
//AbstractRefreshableApplicationContext.java 217行
//定制beanFactory
//设置beanFactory是否允许被覆盖、是否允许循环引用
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
    	//设置beanFactory是否允许被覆盖
		if (this.allowBeanDefinitionOverriding != null) {
			beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
    	//设置beanFactory是否允许循环引用
		if (this.allowCircularReferences != null) {
			beanFactory.setAllowCircularReferences(this.allowCircularReferences);
		}
	}
```

* allowBeanDefinitionOverriding默认为null，表示 配置文件中定义 `bean` 时使用了相同的`id`或 `name `，如果在同一文件中的话会抛出异常，如果不是在同一文件中会覆盖
* allowCircularReferences默认为null，表示允许循环依赖。但是如果在构造方法中A依赖B，B依赖A还是不行

##### loadBeanDefinitions 方法

```java
//AbstractXmlApplicationContext.java 80行
//加载bean到beanFactory中
@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// 使用给定的BeanFactory创建一个新的XmlBeanDefinitionReader
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		//使用context配置beanDefinitionReader
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		//初始化beanDefinitionReader，默认是空的，让子类覆盖
        //在这没看到子类的实现，应该不重要
		initBeanDefinitionReader(beanDefinitionReader);
        //加载BeanDefinition，这个方法是核心，BeanDefinition会专门介绍
		loadBeanDefinitions(beanDefinitionReader);
	}
```

##### loadBeanDefinitions 方法

```java
//AbstractXmlApplicationContext.java 120行
//使用给定的XmlBeanDefinitionReader加载BeanDefinition
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    	//下面的两个分支最后都会进入到同一个地方（XmlBeanDefinitionReader.java 303行）
    	//往下看
    
    	//getConfigResources 直接获取配置的资源
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
    	//getConfigLocations 获取最开始配置的配置文件路径
    	//路径最后也会转化为Resource进入上面一样的方法
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}
```

##### loadBeanDefinitions 方法

```java
//XmlBeanDefinitionReader.java 303行
@Override
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(new EncodedResource(resource));
	}
```

##### loadBeanDefinitions 方法

```java
//XmlBeanDefinitionReader.java 314行
//从指定的xml文件中加载BeanDefinition
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource.getResource());
		}
		//resourcesCurrentlyBeingLoaded 是一个ThreadLocal对象，使用它来存放配置文件资源
    	//从 resourcesCurrentlyBeingLoaded 中获取配置资源文件
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<EncodedResource>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
                //重点看这个方法
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
```

##### doLoadBeanDefinitions 方法

```java
//XmlBeanDefinitionReader.java 388行
//真正的从xml文件中加载BeanDefinition的方法
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
            //将xml转换为Document对象
			Document doc = doLoadDocument(inputSource, resource);
            //接着往下看
			return registerBeanDefinitions(doc, resource);
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}
```

##### registerBeanDefinitions 方法

```java
//XmlBeanDefinitionReader.java 505行
//从指定的Document对象中注册BeanDefinition
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    	//创建一个解析对象
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    	//获取已经注册BeanDefinition的数量
		int countBefore = getRegistry().getBeanDefinitionCount();
    	//这个是核心，跟进去
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    	//返回当前注册的数量
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

##### registerBeanDefinitions 方法

```java
//DefaultBeanDefinitionDocumentReader.java 90行
@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		logger.debug("Loading bean definitions");
		Element root = doc.getDocumentElement();
        //从 xml 根节点开始解析
		doRegisterBeanDefinitions(root);
	}
```

##### doRegisterBeanDefinitions 方法

```java
//DefaultBeanDefinitionDocumentReader.java 116行
//在给定的根节点{@code <beans />}元素中注册每个bean定义。
protected void doRegisterBeanDefinitions(Element root) {
    	//所有<beans>标签都会在这个方法里进行递归
    	//所以这个root并不一定是根节点，只能算父节点，可能存在beans标签嵌套。每次递归的根节点？
    	//BeanDefinitionParserDelegate 是一个委托类，它负责解析bean定义，这个类很重要
    	//第一次进来这个delegate对象为null
		BeanDefinitionParserDelegate parent = this.delegate;
    	//创建解析bean定义的委托对象
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
            // 这块说的是根节点 <beans ... profile="dev" /> 中的 profile 是否是当前环境需要的，
      		// 如果当前环境配置的 profile 不包含此 profile，那就直接 return 了，不对此 <beans /> 解析
            //生产环境不会解析测试环境需要的bean
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}
		//解析前处理，钩子函数，这里没有实现
		preProcessXml(root);
    	//这个是解析的核心，跟进去
		parseBeanDefinitions(root, this.delegate);
    	//解析后处理，钩子函数，这里没有实现
		postProcessXml(root);

		this.delegate = parent;
	}

```

##### parseBeanDefinitions 方法

```java
//DefaultBeanDefinitionDocumentReader.java 161行
//解析根级别的标签，如：import", "alias", "bean".
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    	//default namespace 涉及到的就四个标签 <import />、<alias />、<bean /> 和 <beans />，
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
                        	//解析default namespace 的元素
						parseDefaultElement(ele, delegate);
					}
					else {
                        	//解析自定义或者其他namespace的元素
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
            //解析自定义或者其他namespace的标签
			delegate.parseCustomElement(root);
		}
	}
```

##### parseDefaultElement 方法

```java
//DefaultBeanDefinitionDocumentReader.java 182行
//解析default namespace 的元素
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
            //解析<import/>标签
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
            //解析<alias/>标签
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
            //解析<bean/>标签，跟进去看一下。其它的自己研究
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
            //解析<beans/>标签,进入上面的方法进行递归
			doRegisterBeanDefinitions(ele);
		}
	}
```

#####  <span id="processBeanDefinition">processBeanDefinition 方法 </span>

```java
//DefaultBeanDefinitionDocumentReader.java 298行
//解析<bean/>标签
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    	//把<bean/>标签解析成一个BeanDefinition对象，跟进去
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
            // 如果有自定义属性的话，进行相应的解析，跳过
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// 注册beanDefinition
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			//注册完成后，发送事件
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

##### parseBeanDefinitionElement 方法

```java
//BeanDefinitionParserDelegate.java 428行
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
    	//转调下面的方法
		return parseBeanDefinitionElement(ele, null);
	}

//真正解析<bean/>的逻辑
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
    	//获取id属性
		String id = ele.getAttribute(ID_ATTRIBUTE);
    	//获取name属性，这个可以配置多个
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
		
		List<String> aliases = new ArrayList<String>();
    	
		if (StringUtils.hasLength(nameAttr)) {
            //以",; "来切割name属性
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            //把切割后的name属性加入到别名列表
			aliases.addAll(Arrays.asList(nameArr));
		}
		//bean名称默认为id
		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
            //如果id为空，把第一个name作为bean名称
			beanName = aliases.remove(0);
			if (logger.isDebugEnabled()) {
				logger.debug("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}

		if (containingBean == null) {
            //如果传入的beanDefinition为null，默认为null
            //检查bean名称和别名的唯一性
			checkNameUniqueness(beanName, aliases, ele);
		}
		// 根据 <bean ...>...</bean> 中的配置创建 BeanDefinition，然后把配置中的信息都设置到实例中
    	//这个很重要，跟进去
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
    	//到这里整个<bean/>标签解析完成
		if (beanDefinition != null) {
            //beanDefinition不为null的情况
			if (!StringUtils.hasText(beanName)) {
                //如果没设置id和name，beanName为空
				try {
                    //传入的containingBean默认是为null的
					if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
                        	//从解析的beanDefinition生成bean名称
                        	//debug的时候记得把id和name去掉
                        	//从示例中解析出来是 com.ktcatv.springmvc.service.impl.MessageServiceImpl#0
						beanName = this.readerContext.generateBeanName(beanDefinition);
						//从示例中解析出来是 com.ktcatv.springmvc.service.impl.MessageServiceImpl
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                             // 把 beanClassName 设置为 Bean 的别名
							aliases.add(beanClassName);
						}
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Neither XML 'id' nor 'name' specified - " +
								"using generated bean name [" + beanName + "]");
					}
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
             // 返回 BeanDefinitionHolder
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}
		//如果解析的beanDefinition为null，返回null
		return null;
	}
```

##### parseBeanDefinitionElement 方法

```java
//BeanDefinitionParserDelegate.java 522行
//根据配置创建 BeanDefinition 实例
//解析<bean/>本身，不考虑name和alias
public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
    	//解析class属性
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}

		try {
			String parent = null;
            //解析父属性
			if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
				parent = ele.getAttribute(PARENT_ATTRIBUTE);
			}
            //根据classname和parent创建BeanDefinition对象
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);

            //将<bean/>的属性设置给BeanDefinition对象
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
            //设置此BeanDefinition的可读描述。
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

            // 解析 <meta />
			parseMetaElements(ele, bd);
            // 解析 <lookup-method />
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
            // 解析 <replaced-method />
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
			// 解析 <constructor-arg />
			parseConstructorArgElements(ele, bd);
             // 解析 <property />
			parsePropertyElements(ele, bd);
            // 解析 <qualifier />
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch (ClassNotFoundException ex) {
			error("Bean class [" + className + "] not found", ele, ex);
		}
		catch (NoClassDefFoundError err) {
			error("Class that bean class [" + className + "] depends on not found", ele, err);
		}
		catch (Throwable ex) {
			error("Unexpected failure during bean definition parsing", ele, ex);
		}
		finally {
			this.parseState.pop();
		}

		return null;
	}
```

到这一个BeanDefinition就创建完成，现在 [回到 processBeanDefinition ](#processBeanDefinition) 方法，接着往下看注册beanDefinition

##### registerBeanDefinition 方法

```java
//BeanDefinitionReaderUtils.java 143行
//注册指定的beanDefinition到beanFactory
public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		//下面这两步可以看出，是吧beanName和beanDefinition以键值对的方法注册到beanFactory中
		String beanName = definitionHolder.getBeanName();
    	//注册方法，跟进去
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		//下面是注册别名，可以看出别名是跟beanName绑定的
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
```

##### registerBeanDefinition 方法

```java
//DefaultListableBeanFactory.java 793行
//注册beanDefinition到BeanFactory
//注意，这里并没有初始化bean
@Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {
		//参数检查
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
		// oldBeanDefinition allowBeanDefinitionOverriding这个配置是否允许覆盖
		BeanDefinition oldBeanDefinition;
		/** Map of bean definition objects, keyed by bean name */
	//private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(256);
        //所有的bean注册后都会放在这个beanDefinitionMap对象里面
		oldBeanDefinition = this.beanDefinitionMap.get(beanName);
		if (oldBeanDefinition != null) {
            //如果bean已经被注册，根据isAllowBeanDefinitionOverriding来判断是否允许被覆盖
			if (!isAllowBeanDefinitionOverriding()) {
                //不允许被覆盖直接抛出异常
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
						"': There is already [" + oldBeanDefinition + "] bound.");
			}
            //允许覆盖，打印覆盖条件的日志
			else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				if (this.logger.isWarnEnabled()) {
					this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							oldBeanDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(oldBeanDefinition)) {
				if (this.logger.isInfoEnabled()) {
					this.logger.info("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + oldBeanDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
				if (this.logger.isDebugEnabled()) {
					this.logger.debug("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + oldBeanDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
            //真正的覆盖操作
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
            //没有对应beanName的beanDefinition的情况
            
            //是否有bean已经在初始化了
			if (hasBeanCreationStarted()) {
                //已经有bean在初始化了
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {
                //正常情况进入这
				// 将 BeanDefinition 放到这个 map 中，这个 map 保存了所有的 BeanDefinition
				this.beanDefinitionMap.put(beanName, beanDefinition);
                // 这是个 ArrayList，所以会按照 bean 配置的顺序保存每一个注册的 Bean 的名字
				this.beanDefinitionNames.add(beanName);
                //这里是移除手动注册的singleton bean
                //到这一步应该是自动注册
                // 手动指的是通过调用以下方法注册的 bean ：
        		 // registerSingleton(String beanName, Object singletonObject)
				this.manualSingletonNames.remove(beanName);
			}
            //这个不重要
            /** Cached array of bean definition names in case of frozen configuration */
			this.frozenBeanDefinitionNames = null;
		}

		if (oldBeanDefinition != null || containsSingleton(beanName)) {
            //重置给定bean的所有beanDefinition缓存
			resetBeanDefinition(beanName);
		}
	}
```

到这就完成了beanDefinition的注册，这也只是把BeanDefinition保存到了注册中心，到这一步并没有初始化bean。这步的核心是

* 创建BeanFactory
* 加载配置文件
* 解析配置文件
* 生成BeanDefiniton
* BeanDefinition注册到BeanFactory

#### 3. prepareBeanFactory 方法

```java
//AbstractApplicationContext.java 627行
//准备bean工厂，设置类加载器等属性
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// 设置类加载器以用于加载bean类。
		beanFactory.setBeanClassLoader(getClassLoader());
    	//为beanDefinition中的表达式指定解析策略
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    	//添加一个PropertyEditorRegistrar，所有bean创建过程
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// 配置Bean工厂的上下文回调
    	//添加BeanPostProcessor,实现了Aware接口的 beans 在初始化的时候，这个 processor 负责回调
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    	//如果某个 bean 依赖于以下几个接口的实现类，在自动装配的时候忽略它们，
   		// Spring 会通过其他方式来处理这些依赖。
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		/**
        * 下面几行就是为特殊的几个 bean 赋值，如果有 bean 依赖了以下几个，会注入这边相应的值，
        * 之前我们说过，"当前 ApplicationContext 持有一个 BeanFactory"，这里解释了第一行。
        * ApplicationContext 还继承了 ResourceLoader、ApplicationEventPublisher、MessageSource
        * 所以对于这几个依赖，可以赋值为 this，注意 this 是一个 ApplicationContext
        * 那这里怎么没看到为 MessageSource 赋值呢？那是因为 MessageSource 被注册成为了一个普通的 bean
        */
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// 添加BeanPostProcessor,在bean实例化后，如果是 ApplicationListener 的子类，将其添加到 listener 列表中
    	//可以理解成：注册事件监听器
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// 如果发现LoadTimeWeaver，请准备织入
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		//如果没有定义 environment 这个bean，beanFactory会手动注册一个.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
    	//如果没有定义 systemProperties 这个bean，beanFactory会手动注册一个.
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
    	//如果没有定义 systemEnvironment 这个bean，beanFactory会手动注册一个.
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

#### 4. postProcessBeanFactory 方法

```java
//AbstractApplicationContext.java 678行	
/**
	 * Modify the application context's internal bean factory after its standard
	 * initialization. All bean definitions will have been loaded, but no beans
	 * will have been instantiated yet. This allows for registering special
	 * BeanPostProcessors etc in certain ApplicationContext implementations.
	 * @param beanFactory the bean factory used by the application context
	 */
	//子类实现
	protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	}
```

#### 5. invokeBeanFactoryPostProcessors 方法

```java
//AbstractApplicationContext.java 686行	
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```

