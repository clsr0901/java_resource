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
//refresh 是否自动刷新context，加载所有bean定义并创建实例，也可自己后面手动刷新，这里默认是自动刷新。
//parent context的父对象，这里默认是null
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {
		//调用父类的构造函数
    	// 在 AbstractApplicationContext 类的216行 
    	//初始化了 this.resourcePatternResolver = getResourcePatternResolver(); 获得一个支持ANT风格的路径解析器
    	//把传入的parent设置成父context，这里parent默认是null
		super(parent);
    	//设置配置文件路径
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
//可以传入多个配置文件路径
//本质是把解析的路径加入configLocations。 
//configLocations是一个String数组
public void setConfigLocations(String... locations) {
		if (locations != null) {
			Assert.noNullElements(locations, "Config locations must not be null");
			this.configLocations = new String[locations.length];
			for (int i = 0; i < locations.length; i++) {
                //resolvePath 把占位符解析为地址 例如：classpath需要被解析为真实路径
				this.configLocations[i] = resolvePath(locations[i]).trim();
			}
		}
		else {
			this.configLocations = null;
		}
	}
```

### 3. refresh 方法（很重要， 这才是重点） 

```java
//AbstractApplicationContext.java 509行
//可以看出这是父类的一个重写方法
@Override
	public void refresh() throws BeansException, IllegalStateException {
        //加锁，防止在刷新的过程中，其它线程又来操作，避免乱套
		synchronized (this.startupShutdownMonitor) {
			//刷新context之前的准备方法，做一些标记和检验.
            //自己看，很简单
			prepareRefresh();

			//obtainFreshBeanFactory() 主要做下面几件事：
            //1. 创建工厂
            //2. 解析配置文件生成BeanDefinition对象
            //3. 把beanName和对应的BeanDefinition存入容器中（还有别名）
            //注意，这一步并没有实例化对象（先不考虑AOP的内容，AOP有例外），这个方法很重要
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean
			prepareBeanFactory(beanFactory);

			try {
				// 【这里需要知道 BeanFactoryPostProcessor 这个知识点，Bean 如果实现了此接口，
                 // 那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。】

                 // 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化
                 // 具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事
				postProcessBeanFactory(beanFactory);

				// 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 回调方法
				invokeBeanFactoryPostProcessors(beanFactory);

				// 注册 BeanPostProcessor 的实现类，注意看和 BeanFactoryPostProcessor 的区别
         		// 此接口两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization
         		// 两个方法分别在 Bean 初始化之前和初始化之后得到执行。
				registerBeanPostProcessors(beanFactory);

				// 初始化当前 ApplicationContext 的 MessageSource，国际化
				initMessageSource();

				// 初始化当前 ApplicationContext 的事件广播器
				initApplicationEventMulticaster();

				// 子类实现
				onRefresh();

				// 注册监听器，监听器需要实现 ApplicationListener 接口
				registerListeners();

				// 初始化所有的 singleton beans （lazy-init 的除外）
                // 这个是我们的重点
				finishBeanFactoryInitialization(beanFactory);

				//广播事件，ApplicationContext 初始化完成
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
                // 销毁已经初始化的 singleton 的 Beans，以免有些 bean 会一直占用资源
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

#### obtainFreshBeanFactory 方法

```java
//AbstractApplicationContext.java 613行
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    	//主要是这个方法，下面的getBeanFactory就是获取这个方法生成的beanFactroy
    	//这个需要子类来实现，当前类AbstractApplicationContext中是一个抽象方法
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
        //如果已经存在，销毁之前的
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
            //创建一个DefaultListableBeanFactory对象
            //这个对象很重要，最厉害的BeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
            //指定一个id进行序列化，应该使用不到这个功能，序列化BeanFactory？
			beanFactory.setSerializationId(getId());
            //设置beanFactory是否允许被覆盖、是否允许循环引用
			customizeBeanFactory(beanFactory);
            //加载beanDefinition到beanFactory中
            //核心在这里
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

##### customizeBeanFactory 方法 

```java
//AbstractRefreshableApplicationContext.java 217行
//设置beanFactory是否允许被覆盖、是否允许循环引用
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
    	//设置beanFactory是否允许被覆盖
		if (this.allowBeanDefinitionOverriding != null) {
            //默认false，不允许被覆盖
			beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
    	//设置beanFactory是否允许循环引用
		if (this.allowCircularReferences != null) {
            //默认false，不允许被循环引用
			beanFactory.setAllowCircularReferences(this.allowCircularReferences);
		}
	}
```

##### loadBeanDefinitions 方法 

```java
//AbstractXmlApplicationContext.java 80行
//加载beanDefinition到beanFactory中
@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		//创建一个XmlBeanDefinitionReader对象
        //XmlBeanDefinitionReader对象持有BeanFactory的引用
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
        //设置环境等属性
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		//初始化beanDefinitionReader，默认是空的，让子类覆盖
        //在这没看到子类的实现，应该不重要
		initBeanDefinitionReader(beanDefinitionReader);
        //加载BeanDefinition，这个方法是核心
		loadBeanDefinitions(beanDefinitionReader);
	}
```

##### loadBeanDefinitions 方法 

```java
//AbstractXmlApplicationContext.java 120行
//使用给定的XmlBeanDefinitionReader加载BeanDefinition
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    	//下面的两个分支最后都会进入到同一个地方（XmlBeanDefinitionReader.java 303行）
    	//getConfigResources 直接获取配置的资源
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
            //这个方法跟进去
			reader.loadBeanDefinitions(configResources);
		}
    	//getConfigLocations 获取最开始配置的配置文件路径
    	//配置文件路径最后也会读取内容然后转化为Resource进入上面一样的方法
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
        //继续跟，这里把Resource对象转换为EncodedResource对象
		return loadBeanDefinitions(new EncodedResource(resource));
	}
```

##### loadBeanDefinitions 方法

```java
//XmlBeanDefinitionReader.java 314行
//从指定的xml文件中读取内容转换为BeanDefinition对象
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

##### oLoadBeanDefinitions 方法

```java
//XmlBeanDefinitionReader.java 388行
//真正的从指定的xml文件中读取内容转换为BeanDefinition对象
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
                        	//解析自定义或者其他namespace的元素，例如AOP或者Dubbo里面的标签
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
            //解析自定义或者其他namespace的标签，例如AOP或者Dubbo里面的标签
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

##### <span id="processBeanDefinition">processBeanDefinition 方法 </span>

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
//containingBean 默认为null
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
                        	//从示例中解析出来是 com.test.springmvc.service.impl.MessageServiceImpl#0
						beanName = this.readerContext.generateBeanName(beanDefinition);
						//从示例中解析出来是 com.test.springmvc.service.impl.MessageServiceImpl
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
             // 返回 BeanDefinitionHolder对象
            //BeanDefinitionHolder对象包含三个属性
            // private final BeanDefinition beanDefinition;
			// private final String beanName;
			// private final String[] aliases;
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
            //这个方法自己可以跟进去看看，很简单，从xml中获取属性值，设置到BeanDefinition对象对应的属性
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
		// oldBeanDefinition 对应得beanName可能已经注册过了
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
                //正常情况进入这里
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

到这就完成了beanDefinition的注册，这也只是把BeanDefinition保存到了注册中心，到这一步并没有初始化bean。小结：

* 创建BeanFactory
* 加载配置文件
* 解析配置文件
* 生成BeanDefiniton
* BeanDefinition注册到BeanFactory

#### finishBeanFactoryInitialization 方法

```java
//AbstractApplicationContext.java 834行
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// 初始化 conversion service
    	// conversion service 它用来将前端传过来的参数和后端的 controller 方法上的参数进行绑定的时候用
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
				@Override
				public String resolveStringValue(String strVal) {
					return getEnvironment().resolvePlaceholders(strVal);
				}
			});
		}

		// 初始化 LoadTimeWeaverAware类型的bean
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// 开始初始化
		beanFactory.preInstantiateSingletons();
	}
```

##### preInstantiateSingletons 方法

```java
//DefaultListableBeanFactory.java 834行
@Override
	public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}

		// this.beanDefinitionNames保存了所有的beanName
		List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

		// 开始初始化bean
		for (String beanName : beanNames) {
            //获取beanDefinition对象，这里有个合并父bean配置的操作
            // bean继承是指<bean id="" class="" parent="" />的bean会继承parent中的属性配置
            //很简单，可以自己跟进去看看
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
            //这里会判断bean 不是抽象、是单例、不是懒加载 才初始化bean
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
                //初始化FactoryBean，我们用不到，这里不用看，看下面的else
				if (isFactoryBean(beanName)) {
					final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
							@Override
							public Boolean run() {
								return ((SmartFactoryBean<?>) factory).isEagerInit();
							}
						}, getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
				else {
                    //正常情况进入这里初始化bean，这个getBean方法很重要
					getBean(beanName);
				}
			}
		}
        
		//到这里所有的非懒加载的single bean都初始化了，上面是for循环初始化
        
		// Trigger post-initialization callback for all applicable beans...
        //触发所有实现了 SmartInitializingSingleton 接口bean的回调方法
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged(new PrivilegedAction<Object>() {
						@Override
						public Object run() {
							smartSingleton.afterSingletonsInstantiated();
							return null;
						}
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

##### getBean 方法

```java
//AbstractBeanFactory.java 196行
@Override
	public Object getBean(String name) throws BeansException {
        //跟进doGetBean方法，这个带do的应该是做事的方法
		return doGetBean(name, null, null, false);
	}
```

##### doGetBean 方法

```java
//AbstractBeanFactory.java 235行
protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {
		//获取真正的beanName，传入的有可能是别名或者FactoryBean（前面带‘&’）
		final String beanName = transformedBeanName(name);
		Object bean;

		// 检查单例缓存是否有手动注册的单例
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
            // 下面这个方法：如果是普通 Bean 的话，直接返回 sharedInstance，
      		// 如果是 FactoryBean 的话，返回它创建的那个实例对象
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			//这里如果已经存在当前beanName的prototype类型实例，表示在循环引用中A->B->C->A
			// 这种情况直接抛出异常
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// 检查 beanDefinition是否存在beanFactory中
            //如果当前beanFactory不存在，检查父beanFactory中是否存在
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				//不存在，检查父beanFactory
				String nameToLookup = originalBeanName(name);
				if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			if (!typeCheckOnly) {
                //如果typeCheckOnly为false，把beanName添加到alreadyCreated集合（Set）中
				markBeanAsCreated(beanName);
			}

			try {
                //获取beanDifinition对象，这里涉及到bean继承
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                //检查组合后的beanDefinition对象
				checkMergedBeanDefinition(mbd, beanName, args);

				// 获取依赖 depends-on。会先初始化依赖对象
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
                        	//注册依赖关系
                        	//在对应的依赖集合中添加beanName
						registerDependentBean(dep, beanName);
                        //初始化依赖对象
						getBean(dep);
					}
				}

				// 开始创建bean实例
				if (mbd.isSingleton()) {
                    //创建 single bean
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
                                	//创建bean的方法
								return createBean(beanName, mbd, args);
							}
							catch (BeansException ex) {
								// Explicitly remove instance from singleton cache: It might have been put there
								// eagerly by the creation process, to allow for circular reference resolution.
								// Also remove any beans that received a temporary reference to the bean.
								destroySingleton(beanName);
								throw ex;
							}
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
                    //创建prototype bean，每次都创建一个新的实例对象
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
                        //创建bean的方法
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
                    // 如果不是 singleton 和 prototype 的话，需要委托给相应的实现类来处理
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
							@Override
							public Object getObject() throws BeansException {
								beforePrototypeCreation(beanName);
								try {
                                    //创建bean的方法
									return createBean(beanName, mbd, args);
								}
								finally {
									afterPrototypeCreation(beanName);
								}
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// 最后检查类型，如果类型匹配直接返回bean，不匹配抛出异常
		if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
			try {
				return getTypeConverter().convertIfNecessary(bean, requiredType);
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

##### createBean 方法

```java
//AbstractAutowireCapableBeanFactory.java 447行
//创建一个bean实例，填充bean实例，应用后处理器等。
@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// 确保beanDefinition的class被加载
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// 准备方法覆写，这里又涉及到一个概念：MethodOverrides，它来自于 bean 定义中的 <lookup-method /> 和 <replaced-method />
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			//给 BeanPostProcessors 返回一个代理类，AOP中会详细介绍这个方法
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}
		//真正创建实例的方法，核心点
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		if (logger.isDebugEnabled()) {
			logger.debug("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
```

##### doCreateBean 方法

```java
//AbstractAutowireCapableBeanFactory.java 504行
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
            //为null表示不是FactoryBean
            //真正实例化bean在这
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
    	//这个bean就是我们的实例，createBeanInstance返回的是一个包装类
		final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    	// 获取类型
		Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
		mbd.resolvedTargetType = beanType;

		// Allow post-processors to modify the merged bean definition.
    	//允许后处理器修改合并的bean定义。涉及接口：MergedBeanDefinitionPostProcessor，跳过
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// 解决循环依赖的问题，以后再讲
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
			});
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
            //这里填充bean实例的属性，前面只是创建了实例，并没有给属性设置值
			populateBean(beanName, mbd, instanceWrapper);
			if (exposedObject != null) {
                // 还记得 init-method 吗？还有 InitializingBean 接口？还有 BeanPostProcessor 接口？
         		// 这里就是处理 bean 初始化完成后的各种回调
				exposedObject = initializeBean(beanName, exposedObject, mbd);
			}
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

##### createBeanInstance 方法

```java
//AbstractAutowireCapableBeanFactory.java 1057行
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
		// 确保这里beanDefinition的class被加载
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
            //如果不是 public 修饰的类，直接抛出异常
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		if (mbd.getFactoryMethodName() != null)  {
            //采用工厂方法实例化
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// 如果不是第一次创建，比如第二次创建 prototype bean。
        // 这种情况下，我们可以从第一次创建知道，采用无参构造函数，还是构造函数依赖注入 来完成实例化
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
         
		if (resolved) {
			if (autowireNecessary) {
                 // 构造函数依赖注入
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
                 // 无参构造函数
				return instantiateBean(beanName, mbd);
			}
		}

		// 判断是否采用有参构造函数
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
            // 构造函数依赖注入
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		 // 无参构造函数，这个跟进去
		return instantiateBean(beanName, mbd);
	}
```

##### instantiateBean 方法

```java
//AbstractAutowireCapableBeanFactory.java 1134行
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
		try {
			Object beanInstance;
			final BeanFactory parent = this;
			if (System.getSecurityManager() != null) {
				beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
					@Override
					public Object run() {
						return getInstantiationStrategy().instantiate(mbd, beanName, parent);
					}
				}, getAccessControlContext());
			}
			else {
                //实例化
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
			}
            //包装返回
			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
			initBeanWrapper(bw);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
		}
	}
```

##### instantiate 方法

```java
//SimpleInstantiationStrategy.java 59行
@Override
	public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
		// Don't override the class with CGLIB if no overrides.
         // 如果不存在方法覆写，那就使用 java 反射进行实例化，否则使用 CGLIB,
		if (bd.getMethodOverrides().isEmpty()) {
			Constructor<?> constructorToUse;
			synchronized (bd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse == null) {
					final Class<?> clazz = bd.getBeanClass();
					if (clazz.isInterface()) {
                        //如果是接口，直接抛出异常
						throw new BeanInstantiationException(clazz, "Specified class is an interface");
					}
					try {
						if (System.getSecurityManager() != null) {
							constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
								@Override
								public Constructor<?> run() throws Exception {
									return clazz.getDeclaredConstructor((Class[]) null);
								}
							});
						}
						else {
                            //获取空的构造方法
							constructorToUse =	clazz.getDeclaredConstructor((Class[]) null);
						}
						bd.resolvedConstructorOrFactoryMethod = constructorToUse;
					}
					catch (Throwable ex) {
						throw new BeanInstantiationException(clazz, "No default constructor found", ex);
					}
				}
			}
            // 利用构造方法进行实例化
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// Must generate CGLIB subclass.
            // 存在方法覆写，利用 CGLIB 来完成实例化，需要依赖于 CGLIB 生成子类，这里就不展开了。
      		// tips: 因为如果不使用 CGLIB 的话，存在 override 的情况 JDK 并没有提供相应的实例化支持
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
	}
```

##### instantiateClass 方法

```java
//BeanUtils.java 138行
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
		Assert.notNull(ctor, "Constructor must not be null");
		try {
			ReflectionUtils.makeAccessible(ctor);
            //使用反射创建实例
			return ctor.newInstance(args);
		}
		catch (InstantiationException ex) {
			throw new BeanInstantiationException(ctor, "Is it an abstract class?", ex);
		}
		catch (IllegalAccessException ex) {
			throw new BeanInstantiationException(ctor, "Is the constructor accessible?", ex);
		}
		catch (IllegalArgumentException ex) {
			throw new BeanInstantiationException(ctor, "Illegal arguments for constructor", ex);
		}
		catch (InvocationTargetException ex) {
			throw new BeanInstantiationException(ctor, "Constructor threw exception", ex.getTargetException());
		}
	}
```

回到doCreateBean方法继续往下看

##### populateBean 方法

```java
//AbstractAutowireCapableBeanFactory.java 1203行
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    	//bean的所有属性都在这里
    	//可以去看看PropertyValue怎么使用的
		PropertyValues pvs = mbd.getPropertyValues();

		if (bw == null) {
			if (!pvs.isEmpty()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				return;
			}
		}

		// 到这步的时候，bean 实例化完成（通过工厂方法或构造方法），但是还没开始属性设值，
   		// InstantiationAwareBeanPostProcessor 的实现类可以在这里对 bean 进行状态修改，跳过
		boolean continueWithPropertyPopulation = true;
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    // 如果返回 false，代表不需要进行后续的属性设值，也不需要再经过其他的 BeanPostProcessor 的处理
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}

		if (!continueWithPropertyPopulation) {
			return;
		}

		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// 根据名称添加基于自动装配的属性值
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}

			//根据类型添加基于自动装配的属性值
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
		}

		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

		if (hasInstAwareBpps || needsDepCheck) {
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
                        // 这里有个非常有用的 BeanPostProcessor 进到这里: AutowiredAnnotationBeanPostProcessor 对采用 @Autowired、@Value 注解的依赖进行设值
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
			if (needsDepCheck) {
				checkDependencies(beanName, mbd, filteredPds, pvs);
			}
		}
    	//上面都只是把属性名和属性值放到pvs中
		// 这里才是真正设置 bean 实例的属性值
		applyPropertyValues(beanName, mbd, bw, pvs);
	}
```

##### autowireByName 方法

```java
//AbstractAutowireCapableBeanFactory.java 1288行
protected void autowireByName(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

    	//获取属性名称的数组
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    	//遍历属性名
		for (String propertyName : propertyNames) {
            //判读beanFactory中是否包含指定name的bean
			if (containsBean(propertyName)) {
                //根据指定name获取bean
				Object bean = getBean(propertyName);
                //添加到pvs中，这里并没有给对象设置属性值
				pvs.add(propertyName, bean);
                //添加到依赖的集合中去
				registerDependentBean(propertyName, beanName);
				if (logger.isDebugEnabled()) {
					logger.debug("Added autowiring by name from bean name '" + beanName +
							"' via property '" + propertyName + "' to bean named '" + propertyName + "'");
				}
			}
			else {
				if (logger.isTraceEnabled()) {
					logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
							"' by name: no matching bean found");
				}
			}
		}
	}
```

##### autowireByType 方法

```JAVA
//AbstractAutowireCapableBeanFactory.java 1322行
protected void autowireByType(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}

		Set<String> autowiredBeanNames = new LinkedHashSet<String>(4);
    	//获取属性名称的数组
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    	//遍历属性名
		for (String propertyName : propertyNames) {
			try {
				PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
				// Don't try autowiring by type for type Object: never makes sense,
				// even if it technically is a unsatisfied, non-simple property.
				if (Object.class != pd.getPropertyType()) {
					MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
					// Do not allow eager init for type matching in case of a prioritized post-processor.
					boolean eager = !PriorityOrdered.class.isAssignableFrom(bw.getWrappedClass());
					DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
                    //解决针对此工厂中定义的Bean的指定依赖关系。
					Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
					if (autowiredArgument != null) {
                        //添加到pvs中，这里并没有给对象设置属性值
						pvs.add(propertyName, autowiredArgument);
					}
					for (String autowiredBeanName : autowiredBeanNames) {
                        //添加到依赖的集合中去
						registerDependentBean(autowiredBeanName, beanName);
						if (logger.isDebugEnabled()) {
							logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" +
									propertyName + "' to bean named '" + autowiredBeanName + "'");
						}
					}
                    //清除数据，help GC
					autowiredBeanNames.clear();
				}
			}
			catch (BeansException ex) {
				throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
			}
		}
	}
```

回到doCreateBean方法继续往下看

##### initializeBean 方法

```java
//AbstractAutowireCapableBeanFactory.java 1604行
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged(new PrivilegedAction<Object>() {
				@Override
				public Object run() {
					invokeAwareMethods(beanName, bean);
					return null;
				}
			}, getAccessControlContext());
		}
		else {
             // 如果 bean 实现了 BeanNameAware、BeanClassLoaderAware 或 BeanFactoryAware 接口，回调
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
            // BeanPostProcessor 的 postProcessBeforeInitialization 回调
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
             // 处理 bean 中定义的 init-method，
      	// 或者如果 bean 实现了 InitializingBean 接口，调用 afterPropertiesSet() 方法
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}

		if (mbd == null || !mbd.isSynthetic()) {
            // BeanPostProcessor 的 postProcessAfterInitialization 回调
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}
		return wrappedBean;
	}
```

