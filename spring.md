

## Spring源码解读

#### Spring IOC(XML)

##### setConfigLocations方法

```
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {

    //配置文件数组
    @Nullable
	private Resource[] configResources;
	
	//指定父容器
	public ClassPathXmlApplicationContext(ApplicationContext parent) {
		super(parent);
	}
	
	public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		//根据提供的路径，处理成配置文件数组
		setConfigLocations(configLocations);
		if (refresh) {
		    //重点，后面仔细分析
			refresh();
		}
	}
	
}

    public void setConfigLocations(@Nullable String... locations) {
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
	
    protected String resolvePath(String path) {
	//创建环境变量
		return getEnvironment().resolveRequiredPlaceholders(path);
	}
	
	@Override
	public ConfigurableEnvironment getEnvironment() {
		if (this.environment == null) {
			this.environment = createEnvironment();
		}
		return this.environment;
	}
	
	protected ConfigurableEnvironment createEnvironment() {
		return new StandardEnvironment();
	}
	
	createEnvironment()方法会返回一个StandardEnvironmentl类对象，这个类中的customizePropertySources方法会给资源列表中添加Java进程中的变量和系统的环境变量。
	
    protected void customizePropertySources(MutablePropertySources propertySources) {
		propertySources.addLast(
				new PropertiesPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME,              getSystemProperties()));  
		propertySources.addLast(
				new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
    }


   resolveRequiredPlaceholders方法最终调用的重点代码是parseStringValue。这个方法主要是处理使用${}方式的占位符。
   
   protected String parseStringValue(
			String value, PlaceholderResolver placeholderResolver, @Nullable Set<String> visitedPlaceholders) {

		int startIndex = value.indexOf(this.placeholderPrefix);
		if (startIndex == -1) {
			return value;
		}

		StringBuilder result = new StringBuilder(value);
		while (startIndex != -1) {
			int endIndex = findPlaceholderEndIndex(result, startIndex);
			if (endIndex != -1) {
				String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex);
				String originalPlaceholder = placeholder;
				if (visitedPlaceholders == null) {
					visitedPlaceholders = new HashSet<>(4);
				}
				if (!visitedPlaceholders.add(originalPlaceholder)) {
					throw new IllegalArgumentException(
							"Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
				}
				placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders);
				// Now obtain the value for the fully resolved key...
				String propVal = placeholderResolver.resolvePlaceholder(placeholder);
				if (propVal == null && this.valueSeparator != null) {
					int separatorIndex = placeholder.indexOf(this.valueSeparator);
					if (separatorIndex != -1) {
						String actualPlaceholder = placeholder.substring(0, separatorIndex);
						String defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
						propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
						if (propVal == null) {
							propVal = defaultValue;
						}
					}
				}
				if (propVal != null) {
					propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);
					result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
					if (logger.isTraceEnabled()) {
						logger.trace("Resolved placeholder '" + placeholder + "'");
					}
					startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
				}
				else if (this.ignoreUnresolvablePlaceholders) {
					// Proceed with unprocessed value.
					startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
				}
				else {
					throw new IllegalArgumentException("Could not resolve placeholder '" +
							placeholder + "'" + " in value \"" + value + "\"");
				}
				visitedPlaceholders.remove(originalPlaceholder);
			}
			else {
				startIndex = -1;
			}
		}
		return result.toString();
	}

```

![ConfigurableEnvironment](C:\Users\wuxiaowen\Desktop\ConfigurableEnvironment.png)

这个接口比较重要的就是两部分内容了，**一个是设置Spring的环境就是我们经常用的spring.profile配置。另外就是系统资源Property**。

ClassPathXmlApplicationContext整体看重要实现方法只有setConfigLocations和refresh两个。setConfigLocations方法的主要工作有两个：**创建环境对象ConfigurableEnvironment和处理ClassPathXmlApplicationContext传入的字符串中的占位符**。



##### refresh方法

```
@Override
	public void refresh() throws BeansException, IllegalStateException {
	    //同步机制,为了避免refresh()还没结束，再次发起启动或者销毁容器引起的冲突
		synchronized (this.startupShutdownMonitor) {
			//做一些准备工作，记录容器的启动时间、标记“已启动”状态、检查环境变量等,下文详解。
			prepareRefresh();

			// 负责BeanFactory的初始化、Bean的加载和注册等事件
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 设置BeanFactory的类加载器、添加几个 BeanPostProcessor、手动注册几个特殊的bean
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
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
###### prepareRefresh（）

```
prepareRefresh

protected void prepareRefresh() {
		// Switch to active.
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isDebugEnabled()) {
			if (logger.isTraceEnabled()) {
				logger.trace("Refreshing " + this);
			}
			else {
				logger.debug("Refreshing " + getDisplayName());
			}
		}

		// 初始化加载配置文件方法，并没有具体实现，一个留给用户的扩展点
		initPropertySources();

		// 检查环境变量
		getEnvironment().validateRequiredProperties();

		// Store pre-refresh ApplicationListeners...
		if (this.earlyApplicationListeners == null) {
			this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
		}
		else {
			// Reset local application listeners to pre-refresh state.
			this.applicationListeners.clear();
			this.applicationListeners.addAll(this.earlyApplicationListeners);
		}

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}
	
	// 检查环境变量的核心方法为，简单来说就是如果存在环境变量的value为空的时候就抛异常（MissingRequiredProperty是一个hashset常量），然后停止启动Spring
	public void validateRequiredProperties() {
		MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();
		for (String key : this.requiredProperties) {
			if (this.getProperty(key) == null) {
				ex.addMissingRequiredProperty(key);
			}
		}
		if (!ex.getMissingRequiredProperties().isEmpty()) {
			throw ex;
		}
	}
```
###### obtainFreshBeanFactory()

```
obtainFreshBeanFactory

protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();  //重点
		return getBeanFactory();
}

protected final void refreshBeanFactory() throws BeansException {
        // 判断当前ApplicationContext是否存在BeanFactory，如果存在的话就销毁所有 Bean，关闭 BeanFactory
        // 注意，一个应用可以存在多个BeanFactory，这里判断的是当前ApplicationContext是否存在BeanFactory
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
		    // 初始化DefaultListableBeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			
			// 设置 BeanFactory 的两个配置属性：是否允许 Bean 覆盖、是否允许循环引用
			customizeBeanFactory(beanFactory);
			
			// 加载 Bean 到 BeanFactory 中
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

最开始实例化的DefaultListableBeanFactory继承关系：

![image-20210819190229037](C:\Users\wuxiaowen\AppData\Roaming\Typora\typora-user-images\image-20210819190229037.png)

loadBeanDefinitions()：我们知道BeanFactory是一个Bean容器，而**BeanDefinition就是Bean的一种形式（它里面包含了Bean指向的类、是否单例、是否懒加载、Bean的依赖关系等相关的属性）**。BeanFactory中就是保存的BeanDefinition。

```
BeanDefinition接口

public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

    // Bean的生命周期，默认只提供sington和prototype两种，在WebApplicationContext中还会有request, session, globalSession, application, websocket 等
	String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
	String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;


	int ROLE_APPLICATION = 0;

	int ROLE_SUPPORT = 1;

	int ROLE_INFRASTRUCTURE = 2;

	// Modifiable attributes

    // 设置父Bean
	void setParentName(@Nullable String parentName);

    // 获取父Bean
	@Nullable
	String getParentName();

    // 设置Bean的类名称
	void setBeanClassName(@Nullable String beanClassName);

    // 获取Bean的类名称
	@Nullable
	String getBeanClassName();

    // 设置bean的scope
	void setScope(@Nullable String scope);

	@Nullable
	String getScope();

    // 设置是否懒加载
	void setLazyInit(boolean lazyInit);

	boolean isLazyInit();

    // 设置该Bean依赖的所有Bean
	void setDependsOn(@Nullable String... dependsOn);

	@Nullable
	String[] getDependsOn();

    // 设置该Bean是否可以注入到其他Bean中
	void setAutowireCandidate(boolean autowireCandidate);

    // 该Bean是否可以注入到其他Bean中
	boolean isAutowireCandidate();

    // 同一接口的多个实现，如果不指定名字的话，Spring会优先选择设置primary为true的bean
	void setPrimary(boolean primary);

    // 是否是primary的
	boolean isPrimary();

    // 指定工厂名称
	void setFactoryBeanName(@Nullable String factoryBeanName);

    // 获取工厂名称
	@Nullable
	String getFactoryBeanName();
    
    // 指定工厂类中的工厂方法名称
	void setFactoryMethodName(@Nullable String factoryMethodName);

	@Nullable
	String getFactoryMethodName();

    // 构造器参数
	ConstructorArgumentValues getConstructorArgumentValues();

	default boolean hasConstructorArgumentValues() {
		return !getConstructorArgumentValues().isEmpty();
	}

	MutablePropertyValues getPropertyValues();

	default boolean hasPropertyValues() {
		return !getPropertyValues().isEmpty();
	}

	void setInitMethodName(@Nullable String initMethodName);

	@Nullable
	String getInitMethodName();

	void setDestroyMethodName(@Nullable String destroyMethodName);

	@Nullable
	String getDestroyMethodName();

	void setRole(int role);

	int getRole();

	void setDescription(@Nullable String description);

	@Nullable
	String getDescription();


	// Read-only attributes

	ResolvableType getResolvableType();

    // 是否 singleton
	boolean isSingleton();

    // 是否 prototype
	boolean isPrototype();

    // 如果这个 Bean 是被设置为 abstract，那么不能实例化，常用于作为 父bean 用于继承
	boolean isAbstract();

	@Nullable
	String getResourceDescription();

	@Nullable
	BeanDefinition getOriginatingBeanDefinition();

}
```

```
loadBeanDefinitions()

    @Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// 实例化XmlBeanDefinitionReader
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// 初始化 BeanDefinitionReader
		initBeanDefinitionReader(beanDefinitionReader);
		// 加载BeanDefinitions
		loadBeanDefinitions(beanDefinitionReader);
	}
	
	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}
	
	第一个if是看有没有系统指定的配置文件，如果没有的话就走第二个if加载我们最开始传入的`classpath:application-ioc.xml`
	@Override
	public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
		Assert.notNull(resources, "Resource array must not be null");
		int count = 0;
		// 循环，处理所有配置文件，这里就传了一个
		for (Resource resource : resources) {
			count += loadBeanDefinitions(resource);
		}
		// 返回加载的所有BeanDefinition的数量
		return count;
	}
	
	@Override
	public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(location, null);
	}
	
	public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
			    // 将配置文件转换为Resource对象
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
				// 选择XmlBeanDefinitionReader实现类下的loadBeanDefinitions,下面说
				int count = loadBeanDefinitions(resources);
				if (actualResources != null) {
					Collections.addAll(actualResources, resources);
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
				}
				return count;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			Resource resource = resourceLoader.getResource(location);
			int count = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
			}
			return count;
		}
	}
```

loadBeanDefinitions最终会依赖XmlBeanDefinitionReader解析器(重点)，附上继承关系：

![image-20210819193953297](C:\Users\wuxiaowen\AppData\Roaming\Typora\typora-user-images\image-20210819193953297.png)

```
loadBeanDefinitions(EncodedResource encodedResource)

public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
		    // 获取文件流
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
				// 加载
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

doLoadBeanDefinitions(inputSource, encodedResource.getResource())
// doLoadBeanDefinitions分为两步
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {

		try {
		    // 将xml文件转换位Document对象
			Document doc = doLoadDocument(inputSource, resource);
			//根据Document对象注册Bean，下面说
			int count = registerBeanDefinitions(doc, resource);
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + count + " bean definitions from " + resource);
			}
			return count;
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
	
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        //构建读取Document的工具类
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		//获取已注册的bean数量
		int countBefore = getRegistry().getBeanDefinitionCount();
		// 注册Bean对象，下面说
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		//总注册的bean减去之前注册的bean就是本次注册的bean
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
	
@Override
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
	this.readerContext = readerContext;
	// 获取Document的根节点，并注册，下面说
	doRegisterBeanDefinitions(doc.getDocumentElement());
}

protected void doRegisterBeanDefinitions(Element root) {
        //先将当前的BeanDefinitionParserDelegate（当前根节点）保存到一个新的变量，parent代表子<beans>的父<beans>
		BeanDefinitionParserDelegate parent = this.delegate;
		//创建新的BeanDefinitionParserDelegate
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
		    // 获取 <beans ... profile="***" /> 中的 profile参数与当前环境是否匹配，如果不匹配则不再进行解析
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				// We cannot use Profiles.of(...) since profile expressions are not supported
				// in XML config. See SPR-12458 for details.
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

// 前置扩展点（这里用到了模板方法的设计模式，是spring框架设计出的一个扩展点，默认preProcessXml（）和postProcessXml（）只提供空方法具体实现又开发人员根据自己业务的需求自己在子类中重写此方法，一般这两个方法的作用是：当我们在xml自定义元素时（不是spring里面规定的元素，而是可能为了实现某种功能，我们自己编写的元素，spring框架本身不能解析识别），才需要重写这两个方法）
		preProcessXml(root);
		// 下面说
		parseBeanDefinitions(root, this.delegate);
		// 后置扩展点
		postProcessXml(root);

		this.delegate = parent;
	}
	
     核心解析方法parseBeanDefinitions：
     protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        // default namespace 涉及到的就四个标签 <import />、<alias />、<bean /> 和 <beans />
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
					    // 解析 default namespace 下面的几个元素，这个方法下面说。
						parseDefaultElement(ele, delegate);
					}
					else {
					    // 解析其他 namespace 的元素
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
		    // 解析其他 namespace 的元素
			delegate.parseCustomElement(root);
		}
	}
	
	default标签处理
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
		    // 处理 <import /> 标签
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
		    // 处理 <alias /> 标签
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
		    // 处理 <bean /> 标签定义
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
		    // 处理 <beans /> 标签
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
	
	Bean标签的处理方式：
	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	    //创建BeanDefinition，下面说
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
		    // 如果有自定义属性的话，进行相应的解析
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// 注册Bean，下面说
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder,         getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// 注册完成后，发送事件
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
	
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
		return parseBeanDefinitionElement(ele, null);
	}
	
	Bean标签解析关键：
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
		String id = ele.getAttribute(ID_ATTRIBUTE);
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

		List<String> aliases = new ArrayList<>();
		// 将 name 属性的定义按照 “逗号、分号、空格” 切分，形成一个 别名列表数组，
		if (StringUtils.hasLength(nameAttr)) {
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}

		String beanName = id;
		// 如果没有指定id, 那么用别名列表的第一个名字作为beanName
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			beanName = aliases.remove(0);
			if (logger.isTraceEnabled()) {
				logger.trace("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}

		if (containingBean == null) {
			checkNameUniqueness(beanName, aliases, ele);
		}

        // 根据 <bean ...>...</bean> 中的配置创建 BeanDefinition，然后把配置中的信息都设置到实例中,
        // 这行执行完毕，一个 BeanDefinition 实例就出来了。等下接着往下看
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		
		// <bean /> 标签完成
		if (beanDefinition != null) {
		    // 如果没有设置 id 和 name，那么此时的 beanName 就会为 null
			if (!StringUtils.hasText(beanName)) {
				try {
					if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
						
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() >  beanClassName.length() &&
							!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							// 把 beanClassName 设置为 Bean 的别名
							aliases.add(beanClassName);
						}
					}
					if (logger.isTraceEnabled()) {
						logger.trace("Neither XML 'id' nor 'name' specified - " +
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

		return null;
	}
	
	//创建BeanDefinition(重点)
	@Nullable
	public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, @Nullable BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}
		String parent = null;
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}

		try {
		    // 创建 BeanDefinition，然后设置类信息(Bean类对象和类名)
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
            
            // 设置 BeanDefinition 的一堆属性，这些属性定义在 AbstractBeanDefinition 中
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
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
	
	回到Bean标签的处理方式：protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate)，上面提到创建完BeanDefinition成功后，调用BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry())注册Bean,下面接着说：
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		String beanName = definitionHolder.getBeanName();
		//注册这个Bean,下面说
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// 如果配置有别名的话，也要根据别名全部注册一遍
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
	
	registerBeanDefinition:
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

        // 所有的 Bean 注册后都会被放入到这个beanDefinitionMap 中，查看是否已存在这个bean
		BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
		
		// 处理重复名称的 Bean 定义的情况
		if (existingDefinition != null) {
			if (!isAllowBeanDefinitionOverriding()) {
			    // 如果不允许覆盖的话，抛异常
				throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
			}
			else if (existingDefinition.getRole() < beanDefinition.getRole()) {
				// 用框架定义的 Bean 覆盖用户自定义的 Bean
				if (logger.isInfoEnabled()) {
					logger.info("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							existingDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(existingDefinition)) {
			    // 用新的 Bean 覆盖旧的 Bean
				if (logger.isDebugEnabled()) {
					logger.debug("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
			    // log...用同等的 Bean 覆盖旧的 Bean
				if (logger.isTraceEnabled()) {
					logger.trace("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			// 覆盖
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
		    // 判断是否已经有其他的 Bean 开始初始化了.注意，"注册Bean"这个动作结束，Bean依然还没有初始化,在 Spring 容器启动的最后会预初始化所有的 singleton beans
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					removeManualSingletonName(beanName);
				}
			}
			else {
				// 将 BeanDefinition 放到这个 map 中，这个 map 保存了所有的 BeanDefinition
				this.beanDefinitionMap.put(beanName, beanDefinition);
				// 这是个 ArrayList，所以会按照 bean 配置的顺序保存每一个注册的 Bean 的名字
				this.beanDefinitionNames.add(beanName);
				// 这是个 LinkedHashSet，代表的是手动注册的 singleton bean
				removeManualSingletonName(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (existingDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}
	
	到这里已经初始化了 Bean 容器，<bean/>的配置也相应的转换为了一个个BeanDefinition，然后注册了所有的BeanDefinition到beanDefinitionMap.
```

###### prepareBeanFactory()

```
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// 设置为加载当前ApplicationContext类的类加载器
		beanFactory.setBeanClassLoader(getClassLoader());
		// 设置 BeanExpressionResolver
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// 这里是Spring的又一个扩展点,在所有实现了Aware接口的bean在初始化的时候，这个 processor负责回调，
		// 这个我们很常用，如我们会为了获取 ApplicationContext 而 implement ApplicationContextAware	
		// 注意：它不仅仅回调 ApplicationContextAware，还会负责回调 EnvironmentAware、ResourceLoaderAware 等
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		
		// 下面几行的意思就是，如果某个 bean 依赖于以下几个接口的实现类，在自动装配的时候忽略它们，Spring 会通过其他方式来处理这些依赖。
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		//下面几行就是为特殊的几个 bean 赋值，如果有 bean 依赖了以下几个，会注入这边相应的值
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// 注册 事件监听器
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// 如果存在bean名称为loadTimeWeaver的bean则注册一个BeanPostProcessor
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// 如果没有定义 "environment" 这个 bean，那么 Spring 会 "手动" 注册一个
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		// 如果没有定义 "systemProperties" 这个 bean，那么 Spring 会 "手动" 注册一个
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		// 如果没有定义 "systemEnvironment" 这个 bean，那么 Spring 会 "手动" 注册一个
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

###### postProcessBeanFactory()

如果有Bean实现了BeanFactoryPostProcessor接口，那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事。

###### invokeBeanFactoryPostProcessor()

调用BeanFactoryPostProcessor各个实现类的postProcessBeanFactory(factory) 方法

###### registerBeanPostProcessors()

注册 BeanPostProcessor 的实现类，注意不是BeanFactoryPostProcessor

此接口有两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization分别会在Bean初始化之前和初始化之后得到执行

###### initMessageSource()

初始化当前 ApplicationContext 的 MessageSource.

###### initApplicationEventMulticaster

初始化当前ApplicationContext的事件广播器

```
protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		//如果用户配置了自定义事件广播器，就使用用户的
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		}
		else {
		    //使用默认的时间广播器
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
						"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
			}
		}
	}
```

###### onRefresh()

扩展点

###### registerListeners()

```
protected void registerListeners() {
		//先添加手动set的一些监听器
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		//取到监听器的名称，设置到广播器
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// 如果存在早期应用事件，发布
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (earlyEventsToProcess != null) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
```

###### finishBeanFactoryInitialization()

这个方法就是负责初始化所有的没有设置懒加载的singleton bean

```
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		//先初始化 LoadTimeWeaverAware 类型的 Bean
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		//停止使用用于类型匹配的临时类加载器
		beanFactory.setTempClassLoader(null);

		//冻结所有的bean定义，即已注册的bean定义将不会被修改或后处理
		beanFactory.freezeConfiguration();

		//初始化
		beanFactory.preInstantiateSingletons();
	}
	
```

conversionService：用于将前端传过来的参数和后端的controller方法上的参数格式转换时使用

EmbeddedValueResolver：利用EmbeddedValueResolver可以很方便的实现读取配置文件的属性。

```
@Component
public class PropertiesUtil implements EmbeddedValueResolverAware {

    private StringValueResolver resolver;

    @Override
    public void setEmbeddedValueResolver(StringValueResolver resolver) {
        this.resolver = resolver;
    }


    /**
     * 获取属性时直接传入属性名称即可
     */
    public String getPropertiesValue(String key) {
        StringBuilder name = new StringBuilder("${").append(key).append("}");
        return resolver.resolveStringValue(name.toString());
    }

}
```

preInstantiateSingletons()：

```
	@Override
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		// this.beanDefinitionNames 保存了所有的 beanNames
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		for (String beanName : beanNames) {
		    // 合并父 Bean 中的配置，主意<bean id="" class="" parent="" /> 中的 parent属性
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			// 不是抽象类、是单例的且不是懒加载的
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			    // 处理 FactoryBean
				if (isFactoryBean(beanName)) {
				    //在 beanName 前面加上“&” 符号
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						// 判断当前 FactoryBean 是否是 SmartFactoryBean 的实现	
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
				    // 不是FactoryBean的直接使用此方法进行初始化
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		// 如果bean实现了 SmartInitializingSingleton 接口的，那么在这里得到回调
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
	
	FactoryBean 是否是 SmartFactoryBean 的实现都会调用getBean
	@Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}
	
	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

        // 获取beanName，处理两种情况，一个是前面说的 FactoryBean(前面带 ‘&’)，再一个这个方法是可以根据别名来获取Bean的，         //所以在这里是要转换成最正统的BeanName,主要逻辑就是如果是FactoryBean就把&去掉如果是别名就把根据别名获取真实名称后面         //就不贴代码了
        final String beanName = transformedBeanName(name);
		final String beanName = transformedBeanName(name);
		
		//最后的返回值
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		// 检查是否已初始化
		Object sharedInstance = getSingleton(beanName);
		//如果已经初始化过了，且没有传args参数就代表是get，直接取出返回
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			// 这里如果是普通Bean的话，直接返回，如果是 FactoryBean 的话，返回它创建的那个实例对象
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// 如果存在prototype类型的这个bean
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// 如果当前BeanDefinition不存在这个bean且具有父BeanFactory
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				// 返回父容器的查询结果
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}

			if (!typeCheckOnly) {
			    // typeCheckOnly 为 false，将当前 beanName 放入一个 alreadyCreated 的 Set 集合中。
				markBeanAsCreated(beanName);
			}

            //开始创建bean
			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				// 先初始化依赖的所有 Bean， depends-on 中定义的依赖
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
					    // 检查是不是有循环依赖
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						// 注册一下依赖关系
						registerDependentBean(dep, beanName);
						try {
						    // 先初始化被依赖项
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// Create bean instance.
				// 如果是单例的
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
						    // 执行创建 Bean，下面说
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

                // 如果是prototype
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						// 执行创建 Bean
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}
                
                // 如果不是 singleton 和 prototype 那么就是自定义的scope、例如Web项目中的session等类型，这里就交给自定义scope的应用方去实现
				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
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
		
		// Check if required type matches the type of the actual bean instance.
		//检查bean的类型
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
}

解析doGetBean我们知道了原来Spring本身只定义了两种Scope，也知道了SpringMVC的几种Scope是如何实现的了。然后发现一开始会先判断bean存不存在，如果存在就直接返回了。如果不存在那就要接着往下看createBean方法了
        @Override
	    protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		// 确保 BeanDefinition 中的 Class 被加载
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		// 准备方法覆写，如果bean中定义了 <lookup-method /> 和 <replaced-method />
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			// 如果有代理的话直接返回
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
		    // 创建 bean,下面说
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
	
	doCreateBean:
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
		    //如果是.factoryBean则从缓存删除
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
		    // 实例化 Bean，这个方法里面才是终点，下面说
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		//bean实例
		final Object bean = instanceWrapper.getWrappedInstance();
		//bean类型
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
				// 循环调用实现了MergedBeanDefinitionPostProcessor接口的postProcessMergedBeanDefinition方法
				// Spring对这个接口有几个默认的实现，其中大家最熟悉的一个是操作@Autowired注解的
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		// 解决循环依赖问题
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			//当正在创建A时，A依赖B，此时通过（8将A作为ObjectFactory放入单例工厂中进行early expose，此处B需要引用A，但A正             //在创建，从单例工厂拿到ObjectFactory，从而允许循环依赖
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
		    // 负责属性装配,很重要，下面说
			populateBean(beanName, mbd, instanceWrapper);
			// 这里是处理bean初始化完成后的各种回调，例如init-method、InitializingBean 接口、BeanPostProcessor 接口
			exposedObject = initializeBean(beanName, exposedObject, mbd);
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
        
        //同样的，如果存在循环依赖
		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
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
		// 把bean注册到相应的Scope中
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
	
    创建Bean实例createBeanInstance()：
    protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.
		// 确保已经加载了此bean class
		Class<?> beanClass = resolveBeanClass(mbd, beanName);
        
        // 校验类的访问权限
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
		    //扩展的实例提供器，可以直接从里面获取实例
			return obtainFromSupplier(instanceSupplier, beanName);
		}

		if (mbd.getFactoryMethodName() != null) {
		    // 采用工厂方法实例化
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		//是否第一次
		boolean resolved = false;
		//是否采用构造函数注入
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
			    //有解析的构造器或者工厂方法
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		//有解析的构造器或者工厂方法
		if (resolved) {
			if (autowireNecessary) {
			    //构造器有参数的
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
			    // 无参构造函数
				return instantiateBean(beanName, mbd);
			}
		}

		// Candidate constructors for autowiring?
		// 判断是否采用有参构造函数(从bean后置处理器中为自动装配寻找构造方法)
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			// 构造函数依赖注入
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// Preferred constructors for default construction?
		// 找出最合适的默认构造方法
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// No special handling: simply use no-arg constructor.
		// 调用无参构造函数
		return instantiateBean(beanName, mbd);
	}
	
	选择一个最简单的默认构造方法看一下：
	protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
		try {
			Object beanInstance;
			final BeanFactory parent = this;
			if (System.getSecurityManager() != null) {
				beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
						getInstantiationStrategy().instantiate(mbd, beanName, parent),
						getAccessControlContext());
			}
			else {
			    // 具体实例化的实现，下面说
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
			}
			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
			initBeanWrapper(bw);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
		}
	}
	
	@Override
	public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
		// Don't override the class with CGLIB if no overrides.
		// 如果不存在方法覆写，那就使用 java 反射进行实例化，否则使用 CGLIB,
		if (!bd.hasMethodOverrides()) {
			Constructor<?> constructorToUse;
			synchronized (bd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse == null) {
					final Class<?> clazz = bd.getBeanClass();
					if (clazz.isInterface()) {
						throw new BeanInstantiationException(clazz, "Specified class is an interface");
					}
					try {
						if (System.getSecurityManager() != null) {
							constructorToUse = AccessController.doPrivileged(
									(PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
						}
						else {
							constructorToUse = clazz.getDeclaredConstructor();
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
			// 存在方法覆写，利用 CGLIB 来完成实例化，需要依赖于 CGLIB 生成子类，这里就不展开了
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
	}
	
	上面的doCreateBean方法除了createBeanInstance还有一个重要方法就是populateBean属性注入：
	protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		if (bw == null) {
			if (mbd.hasPropertyValues()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				return;
			}
		}

		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
		// state of the bean before properties are set. This can be used, for example,
		// to support styles of field injection.
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

        // bean的所有属性
		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

		int resolvedAutowireMode = mbd.getResolvedAutowireMode();
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
			// Add property values based on autowire by name if applicable.
			// 通过名字找到所有属性值，如果是 bean 依赖，先初始化依赖的 bean。记录依赖关系
			if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}
			// Add property values based on autowire by type if applicable.
			// 通过类型装配。复杂一些
			if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}
			pvs = newPvs;
		}

		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

		PropertyDescriptor[] filteredPds = null;
		if (hasInstAwareBpps) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						if (filteredPds == null) {
							filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
						}
						// 这里就是上方曾经提到过得对@Autowired处理的一个BeanPostProcessor了
               // 它会对所有标记@Autowired、@Value 注解的属性进行设值
						pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvsToUse == null) {
							return;
						}
					}
					pvs = pvsToUse;
				}
			}
		}
		if (needsDepCheck) {
			if (filteredPds == null) {
				filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			}
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}

		if (pvs != null) {
		    // 设置 bean 实例的属性值
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
	}
```

###### finishRefresh()

```
protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		// 清理刚才一系列操作使用到的资源缓存
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		// 初始化LifecycleProcessor
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		// 这个方法的内部实现是启动所有实现了Lifecycle接口的bean
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		//发布ContextRefreshedEvent事件
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		// 检查spring.liveBeansView.mbeanDomain是否存在，有就会创建一个MBeanServer
		LiveBeansView.registerApplicationContext(this);
	}
```

###### **resetCommonCaches()**

清除缓存



##### **总结：**

Spring IOC基于XML方式我们以ClassPathXmlApplicationContext启动类举例，首先构造方法中先执行的是setConfigLocations(提供的路径)，这个方法的作用是**根据提供的路径，处理成配置文件数组**。进入这个方法里之后进行resolvePath(locations[i])解析传入的路径，这个方法返回getEnvironment().resolveRequiredPlaceholders(path)。getEnvironment()创建了环境变量，进入getEnvironment()再进入重点createEnvironment()，createEnvironment()方法会返回一个StandardEnvironmentl类对象，**这个类中的customizePropertySources方法会给资源列表中添加Java进程中的变量和系统的环境变量**。**resolveRequiredPlaceholders方法最终调用的重点代码是parseStringValue。这个方法主要是处理使用${}方式的占位符。**所以setConfigLocations方法的主要工作有两个：**创建环境对象ConfigurableEnvironment和处理ClassPathXmlApplicationContext传入的字符串中的占位符**。

下来是ClassPathXmlApplicationContext实现的下一个重点方法refresh，首先refresh方法整体利用了**同步机制synchronized**,为了避免refresh()还没结束，再次发起启动或者销毁容器引起的冲突。这个过程比较复杂，**先是prepareRefresh()做一些准备工作，记录容器的启动时间、标记“已启动”状态、检查环境变量等**，在记录好容器的启动时间、标记好启动状态后，**该执行initPropertySources()，这个方法初始化加载配置文件方法，并没有具体实现，一个留给用户的扩展点**，之后执行getEnvironment().validateRequiredProperties()检查环境变量（其中第一个方法getEnvironment在setConfigLocations中的resolvePath解析路径中出现过，已经创建好了环境变量），**检查环境变量的核心逻辑简单来说就是如果存在环境变量的value为空的时候就抛异常（MissingRequiredProperty是一个hashset常量），然后停止启动Spring。**

做好了准备工作之后就要开始**负责BeanFactory的初始化、Bean的加载和注册等事件（obtainFreshBeanFactory()）**。进入该方法后有两个语句首先执行refreshBeanFactory()，进入该方法首先if判断hasBeanFactory()当前ApplicationContext是否存在BeanFactory，如果存在的话就销毁所有 Bean，关闭 BeanFactory（一个应用可以存在多个BeanFactory，这里判断的是当前ApplicationContext是否存在BeanFactory）。判断完了之后createBeanFactory()初始化DefaultListableBeanFactory，接着给bean工厂设置序列化id，**customizeBeanFactory(beanFactory)设置 BeanFactory 的两个配置属性：是否允许 Bean 覆盖、是否允许循环引用**。之后再进行loadBeanDefinitions(beanFactory)**加载bean到beanFactory**，进入到这个方法体内，方法的逻辑先实例化XmlBeanDefinitionReader，再配置此上下文的beanDefinitionReader，之后再初始化BeanDefinitionReader()（initBeanDefinitionReader(beanDefinitionReader)），最后加载BeanDefinitions（loadBeanDefinitions(beanDefinitionReader)），进入此方法，此方法有两个判断，第一个if是看有没有系统指定的配置文件，如果没有的话就走第二个if加载我们最开始传入的classpath:application-ioc.xml调用重载方法loadBeanDefinitions(configLocations)，这个方法里会遍历传入的所有配置文件返回加载所有BeanDefinitions的数量，然后处理调用loadBeanDefinitions(resource)，**处理的逻辑是先获取资源加载器resourceLoader，通过resourceLoader将传入的配置文件location转化为resource对象，之后选择XmlBeanDefinitionReader实现类下的loadBeanDefinitions（EncodedResource）**,而**loadBeanDefinitions最终依赖于XmlBeanDefinitionReader解析器**。loadBeanDefinitions（EncodedResource）会先获取传入resource对象的文件流，之后加载doLoadBeanDefinitions(inputSource, encodedResource.getResource())，doLoadBeanDefinitions过程又分为两步，先将Xml转为Document对象，再根据Document对象注册Bean(doLoadBeanDefinitions(inputSource, encodedResource.getResource())，**注册方法逻辑又分为四步，先构建读取Document工具类，获取已经注册的Bean数量，根据获取的工具类注册Bean对象RegisterBeanDefinitions(doc, createReaderContext(resource))，最后返回总注册的bean数量减去之前注册的bean数量也就是当前注册的bean数量。**进入注册流程，获取Document的根节点，并注册调用doRegisterBeanDefinitions(doc.getDocumentElement())，此方法先将BeanDefinitionParseDelegate（当前根节点）保存到一个新的变量<beans>的父节点<beans>，再创建新的BeanDefinitionParseDelegate。此方法的目的是**在给定的根<beans/>元素中注册每个bean定义。**获取 <beans ... profile="*" /> 中的 profile参数与当前环境是否匹配，如果不匹配则不再进行解析，之后再经过前置扩展点，核心解析方法parseBeanDefinitions，后置扩展点。核心解析方法解析两种标签，一种是default默认一种是其他标签，default namespace 涉及到的就四个标签 <import />、<alias />、<bean /> 和 <beans />。深入讨论bean标签的处理parseBeanDefinitions过程，首先解析返回BeanDefinition，然后检查是否有自定义属性进行解析，再将返回的BeanDefinition注册，注册完成后发送事件。而解析Bean逻辑又是这样的，先将name属性的定义按照“逗号、分号、空格”切分，形成一个别名列表数组。再检查是否有指定id，如果没有就用刚生成的别名列表的第一个作为beanName。再根据 <bean ...>...</bean> 中的配置创建 BeanDefinition，然后把配置中的信息都设置到实例中，这时执行完毕一个BeanDefinition实例生成（parseBeanDefinitionElement(ele, beanName, containingBean)）（bean标签完成）。之后看刚生成的BeanDefinition如果没有设置id和bean的话，那么此时的beanName为null，便将beanClassName设置为bean的别名，最后返回生成BeanDefinitionHolder。创建BeanDefinition的逻辑是这样的（parseBeanDefinitionElement(ele, beanName, containingBean)），**获取解析bean节点属性的classname和parent，创建beanDefinition，然后将获取的属性设置类信息（Bean的类对象和类名），再设置BeanDefinition的一堆属性，将属性定义在AbstractBeanDefinition（最后要返回的），最后是解析一堆标签比如<meta /> <constructor-arg /><property />等和设置resource。**parseBeanDefinitions第一步获取到beanDefinition后进行注册（如果配置有别名也要根据别名全部注册一遍），首先spring的bean注册后都会被放入到beanDefinitionMap中，通过beanDefinitionMap.get(beanName)判断这个bean是否已经被注册过了，如果存在这个bean判断这个bean能否被覆盖，不能覆盖抛出异常，可以覆盖根据不同覆盖方式打印日志并且覆盖，如果map里没有，判断是否有其他的Bean开始初始化（注册结束，Bean依然没有初始化，在Spring容器启动的最后会预初始化所有的singleton beans）。如果没有初始化，将BeanDefinition放到这个map中，这个map保存了所有的beanDefinition。然后将每一个注册的beanName放入到beanDefinitionNames（ArrayList），更新手动注册的singleton bean（removeMunualSingletonName方法内部存储结构是LinkedHashSet）。到这里已经初始化了 Bean 容器，<bean/>的配置也相应的转换为了一个个BeanDefinition，然后注册了所有的BeanDefinition到beanDefinitionMap.

bean加载 注册后prepareBeanFactory（）（设置BeanFactory的类加载器、添加几个 BeanPostProcessor、手动注册几个特殊的bean），首先设置为加载当前ApplicationContext类的类加载器，设置 BeanExpressionResolver。**beanExpressionResolve:为了能够让我们的beanFactory去解析bean表达式,模板默认以前缀“#{”开头，以后缀“}”结尾,可以修改默认额前缀后缀,通过beanFactory.getBeanExpressionResolver()获得BeanExpressionResolver然后resolver.setExpressionPrefix("%{");resolver.setExpressionSuffix("}");**之后添加PropertyEditor属性编辑器（addPropertyEditorRegistrar，可以将我们的property动态设置为bean里面对应的属性类型，可以自定义属性编辑器，通过实现PropertyEditorSupport接口，spring中自带的属性编辑器也是这么做的），再添加一个BPP（addBeanPostProcessor(new ApplicationContextAwareProcessor(this))，ApplicationContextAwareProcessor：能够在bean中获得各种*Aware），添加后跳过几个属性的自动注入，因为在ApplicationContextAwareProcessor中已经完成了手动注入，之后为特殊的几个 bean 赋值，如果有 bean 依赖了以下几个，会注入这边相应的值（自动装配）（BeanFactory，ResourceLoader，ApplicationEventPublisher，ApplicationContext），再添加一个BPP（addBeanPostProcessor(new ApplicationListenerDetector(this)) 处理事件监听器）接着如果存在bean名称为loadTimeWeaver的bean则注册一个BeanPostProcessor（再添加一个BPP，这个BPP用来处理LoadTimeWeaverAware接口的，LoadTimeWeaverAwareProcessor(beanFactory)），最后就是一些系统配置和系统环境信息，如果发现没有这些bean则spring自己注册（environment，systemProperties，systemEnvironment）。

如果有Bean实现了BeanFactoryPostProcessor接口（用户子类实现，空壳），那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事。

调用BeanFactoryPostProcessor各个实现类的postProcessBeanFactory(factory) 方法。

注册 BeanPostProcessor 的实现类，注意不是BeanFactoryPostProcessor，此接口有两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization分别会在Bean初始化之前和初始化之后得到执行。

再初始化当前 ApplicationContext 的 MessageSource。设置国际化资源相关的调用,将实现了MessageSource接口的bean存放在ApplicationContext的成员变量中,先看是否有此配置,如果有就实例化,否则就创建一个DelegatingMessageSource实例的bean。

初始化当前ApplicationContext的事件广播器。如果用户自定义了事件广播器就用用户的，否则用默认时间广播器。

添加手动set的一些监听器，取到监听器的名称，设置到广播器，如果存在早期应用事件，发布。

初始化所有的没有设置懒加载的singleton bean，先初始化 LoadTimeWeaverAware 类型的 Bean，停止使用用于类型匹配的临时类加载器，冻结所有的bean定义，即已注册的bean定义将不会被修改或后处理，最后初始化：从beanDefinitionNames获取所有的beanName,不是抽象类、是单例的且不是懒加载的,最终都会调用doGetBean方法（中间会判断如果bean实现了 SmartInitializingSingleton 接口的，是的话有个回调），解析doGetBean我们知道了原来Spring本身只定义了两种Scope，也知道了SpringMVC的几种Scope是如何实现的了。然后发现一开始会先判断bean存不存在，如果存在就直接返回了。如果不存在那就要接着往下看createBean方法了，createBean方法首先要确保beanDefinition中的Class被加载，然后如果bean中定义了lookup-method标签和replaced-method标签的话准备方法覆盖，接着如果有代理的话直接返回，最后创建bean(doCreateBean)，doCreateBean会先判断如果是.factoryBean则会从缓存删除，接着实例化Bean(createBeanInstance())，处理循环依赖问题，进行属性装配（populateBean()），处理各种回调方法，再次处理循环依赖问题，最后将bean注册到相应的scope中。**实例化Bean(createBeanInstance)先确保已经加载了此bean class，再校验类的访问权限，中间可以利用扩展的实例提供器直接从里面获取实例，采用工厂方法实例化，接着判断有解析的构造器或者工厂方法（是否第一次），有的话用有参构造，不然用无参构造，接着判断是否采用有参构造函数(从bean后置处理器中为自动装配寻找构造方法)，是用依赖注入，不是选择最适合的构造方法，前面都不匹配的话最后用无参构造**，再最简单的无参构造里，如果存在方法覆盖的话就用java反射实例化，否则用cglib。**属性装配的话先获得所有的属性，再通过之前定义过的BeanPostProcessor对注解@Autowired,@Value设值，最后设置 bean 实例的属性值**

清理刚才一系列操作使用到的资源缓存，初始化LifecycleProcessor，启动所有实现了Lifecycle接口的bean，发布ContextRefreshedEvent事件，检查spring.liveBeansView.mbeanDomain是否存在，有就会创建一个MBeanServer。

最后清除缓存。

#### Spring IOC(基于注解)

![image-20210820150658896](C:\Users\wuxiaowen\AppData\Roaming\Typora\typora-user-images\image-20210820150658896.png)

AnnotationConfigApplicationContext继承关系图

```
public AnnotationConfigApplicationContext(String... basePackages) {
		this();
		scan(basePackages);
		refresh();
	}
```

##### this无参构造

```
public AnnotationConfigApplicationContext() {
        //注解bean读取器
		this.reader = new AnnotatedBeanDefinitionReader(this);
        //注解bean扫描器
		this.scanner = new ClassPathBeanDefinitionScanner(this);
}

同时子类的构造方法执行之前肯定会先执行父类的构造方法，所以还有父类GenericApplicationContext的构造方法
public GenericApplicationContext() {
		this.beanFactory = new DefaultListableBeanFactory();
	}
```

##### scan

```
    @Override
	public void scan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		//这里调用的是bean扫描器ClassPathBeanDefinitionScanner的scan方法
		this.scanner.scan(basePackages);
	}
	
	public int scan(String... basePackages) {
	    //获取当前注册bean的数量
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

        //下面说
		doScan(basePackages);

		// Register annotation config processors, if necessary.
		if (this.includeAnnotationConfig) {
		    //注册配置处理器
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}
        //返回此次注册的数量
		return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
	}
	
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		//遍历需要扫描的包路径
		for (String basePackage : basePackages) {
		    //先跟进去看，下面的方法先忽略（获取所有符合条件的BeanDefinition）
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
			    //绑定BeanDefinition与Scope
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				//查看配置类是否指定bean的名称，如没指定则使用类名首字母小写
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				//检查bean是否存在
				if (checkCandidate(beanName, candidate)) {
				    //又包装了一层
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					//检查scope是否创建，如未创建则进行创建
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					//重点（注册节点，下面说）
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
	
	findCandidateComponents（扫描包）
	public Set<BeanDefinition> findCandidateComponents(String basePackage) {
	    //判断是否使用Filter指定忽略包不扫描
		if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
			return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
		}
		else {
		    //扫描包
			return scanCandidateComponents(basePackage);
		}
	}
	
	在看scanCandidateComponents之前先了解一下这个接口
	public interface MetadataReader {

	/**
	 * Return the resource reference for the class file.
	 */
	 //配置类的资源对象
	Resource getResource();

	/**
	 * Read basic class metadata for the underlying class.
	 */
	 //类的元数据
	ClassMetadata getClassMetadata();

	/**
	 * Read full annotation metadata for the underlying class,
	 * including metadata for annotated methods.
	 */
	 //注解的元数据
	AnnotationMetadata getAnnotationMetadata();

}

    scanCandidateComponents
    private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
		Set<BeanDefinition> candidates = new LinkedHashSet<>();
		try {
		    //组装扫描路径（组装完成后是这种格式：classpath*:cn/shiyujun/config/**/*.class）
			String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
					resolveBasePackage(basePackage) + '/' + this.resourcePattern;
			//根据路径获取资源对象
			Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
			boolean traceEnabled = logger.isTraceEnabled();
			boolean debugEnabled = logger.isDebugEnabled();
			for (Resource resource : resources) {
				if (traceEnabled) {
					logger.trace("Scanning " + resource);
				}
				if (resource.isReadable()) {
					try {
					    //根据资源对象通过反射获取资源对象的MetadataReader，具体就不展开说了
						MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
						//查看配置类是否有@Conditional一系列的注解，然后是否满足注册Bean的条件
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
		return candidates;
	}
	
	来源:doscan方法(注册BeanDefinition)
	protected void registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry) {
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, registry);
	}
	
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		// 注册bean，下面说
		String beanName = definitionHolder.getBeanName();
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
		//如果存在别名则循环注册别名，逻辑跟上方差不多，就不展开了
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
	
	这个注册bean的方法实现是来自于DefaultListableBeanFactory,这个方法在前面spring ioc基于XML中也出现过，流程如下
	@Override
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

        // 所有的 Bean 注册后都会被放入到这个beanDefinitionMap 中，查看是否已存在这个bean
		BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
		
		// 处理重复名称的 Bean 定义的情况
		if (existingDefinition != null) {
			if (!isAllowBeanDefinitionOverriding()) {
			    // 如果不允许覆盖的话，抛异常
				throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
			}
			else if (existingDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				// 用框架定义的 Bean 覆盖用户自定义的 Bean
				if (logger.isInfoEnabled()) {
					logger.info("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							existingDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(existingDefinition)) {
			    // 用新的 Bean 覆盖旧的 Bean
				if (logger.isDebugEnabled()) {
					logger.debug("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
			    // log...用同等的 Bean 覆盖旧的 Bean
				if (logger.isTraceEnabled()) {
					logger.trace("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			// 覆盖
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
		    // 判断是否已经有其他的 Bean 开始初始化了.注意，"注册Bean" 这个动作结束，Bean依然还没有初始化。                           //在 Spring 容器启动的最后，会预初始化所有的 singleton beans
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					removeManualSingletonName(beanName);
				}
			}
			else {
				// Still in startup registration phase
				// 将 BeanDefinition 放到这个 map 中，这个 map 保存了所有的 BeanDefinition
				this.beanDefinitionMap.put(beanName, beanDefinition);
				// 这是个 ArrayList，所以会按照 bean 配置的顺序保存每一个注册的 Bean 的名字
				this.beanDefinitionNames.add(beanName);
				// 这是个 LinkedHashSet，代表的是手动注册的 singleton bean，
				removeManualSingletonName(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (existingDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}
```

##### refresh

```
    //阅读完refresh方法，发现与使用xml方式的refresh基本相同，不同的是obtainFreshBeanFactory()实现上，
    //因为原先obtainFreshBeanFactory()的作用是创建bean容器，但是在注解这创建bean容器是在scan方法里创建好的，
    //所以在此处obtainFreshBeanFactory()的实现上较为简单，下面列出。
    //refresh方法里最后的重点是初始化，这个初始化逻辑在DefaultListableBeanFactory实现。
    @Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
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
	
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		return getBeanFactory();
	}
	
	@Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

##### 总结

基于注解的IOC过程启动类是AnnotationConfigApplicationContext()，有参构造开始时会调用无参构造器，调用无参构造之前肯定会先调用父类的构造方法（GenericApplicationContext的构造方法，该构造方法会new 一个DefaultListableBeanFactory），然后就是注册bean的读取器和扫描器。

之后是方法scan，调用的是bean扫描器ClassPathBeanDefinitionScanner的scan方法，方法内先获取当前注册bean的数量，再调用doScan(basePackages)，之后调用注册配置处理器，方法返回注册的数量。doScan会先遍历需要扫描的包路径，获取所有符合条件的BeanDefinition（findCandidateComponents(basePackage)），这个方法会先判断这个包是否被Filter指定，不指定则继续扫描（scanCandidateComponents(basePackage)），扫描的过程先组装扫描路径，根据路径获取资源对象，遍历资源对象，根据资源对象通过反射获取资源对象的MetadataReader。获取符合条件的BeanDefinition后，遍历绑定BeanDefinition与Scope，查看配置类是否指定bean名称，如没指定则使用类名首字母小写（beanName），检查bean是否存在checkCandidate(beanName, candidate)，存在则再包装beanDefinition，然后检查scope是否创建，如未创建则进行创建，创建完**注册节点**，进入注册节点方法，先注册bean，之后获取别名如果别名存在则给每个别名都注册。注册的方法在DefaultListableBeanFactory实现，XML方式也出现过，具体流程：**根据beanName在beanDefinitionMap中get检查是否存在，如果存在并且可以覆盖的话则覆盖不能覆盖抛出异常。没有存在先检查是否有bean初始化，没有的话把beanDefinition作为value，beanName作为key添加到map中（还要添加beanName到beanDefinitionNames（Arraylist），和LinkedHashSet）**

与使用xml方式的refresh基本相同，不同的是obtainFreshBeanFactory()实现，因为原先obtainFreshBeanFactory()的作用是创建bean容器，但是在注解这创建bean容器是在scan方法里创建好的，

#### SpringAOP

##### @EnableAspectJAutoProxy注解

```
//元注解不再粘贴
//import注解，可以引入一个类，将这个类注入到Spring IOC容器中被当前Spring管理
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
    //proxyTargetClass属性，默认false，尝试采用JDK动态代理织入增强(如果当前类没有实现接口则还是会使用CGLIB)；               //如果设为true，则强制采用CGLIB动态代理织入增强
	boolean proxyTargetClass() default false;
    //通过aop框架暴露该代理对象，aopContext能够访问。为了解决类内部方法之间调用时无法增强的问题
	boolean exposeProxy() default false;

}
```

------

> ```
> class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
> 
>    @Override
>    public void registerBeanDefinitions(
>          AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
> 
>       //注册一个AOP代理实现的Bean，往下看 
>       AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
> 
>       AnnotationAttributes enableAspectJAutoProxy =
>             AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
>       if (enableAspectJAutoProxy != null) {
>          if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
>             AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
>          }
>          if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
>             AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
>          }
>       }
>    }
> 
> }
> 
>     //registerAspectJAnnotationAutoProxyCreatorIfNecessary方法的主要功能是注册或者升级AnnotationAwareAspectJAutoProxyCreator类
>     @Nullable
> 	public static BeanDefinition       registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
> 		return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, null);
> 	}
> 	
> 	@Nullable
> 	public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
> 			BeanDefinitionRegistry registry, @Nullable Object source) {
> 
> 		return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
> 	}
> 	
> 	//根据@Point注解定义的切点来自动代理与表达式匹配的类。(重要)
> 	@Nullable
> 	private static BeanDefinition registerOrEscalateApcAsRequired(
> 			Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {
> 
> 		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
> 
> 		if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
> 			BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
> 			//判断优先级，如果优先级较高则替换原先的bean
> 			if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
> 				int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
> 				int requiredPriority = findPriorityForClass(cls);
> 				if (currentPriority < requiredPriority) {
> 					apcDefinition.setBeanClassName(cls.getName());
> 				}
> 			}
> 			return null;
> 		}
> 
>         //注册AnnotationAwareAspectJAutoProxyCreator到容器中，此类负责基于注解的AOP动态代理实现
> 		RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
> 		beanDefinition.setSource(source);
> 		beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
> 		beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
> 		registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
> 		return beanDefinition;
> 	}
> ```

上面提到了@EnableAspectJAutoProxy注解作用之一是注册AnnotationAwareAspectJAutoProxyCreator到容器中，AnnotationAwareAspectJAutoProxyCreator的继承关系图如下

![image-20210820204308108](C:\Users\wuxiaowen\AppData\Roaming\Typora\typora-user-images\image-20210820204308108.png)

观察类图后发现，AnnotationAwareAspectJAutoProxyCreator这个类间接实现了BeanPostProcessor接口。我们之前在对SpringIOC的源码进行解析时提到过，Spring在实例化Bean的前后会分别调用方法postProcessBeforeInstantiation和postProcessAfterInstantiation。AOP的整体逻辑就是通过这两个方法来实现的

##### postProcessBeforeInstantiation逻辑

> ```
> //这个方法是再bean实例化之前调用的，主要针对切面类。这个方法不在AnnotationAwareAspectJAutoProxyCreator这个类中，    //而是在其父类AbstractAutoProxyCreator中
> @Override
> public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
>    Object cacheKey = getCacheKey(beanClass, beanName);
> 
>    if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
>       if (this.advisedBeans.containsKey(cacheKey)) {
>          return null;
>       }
>       //加载所有增强,下面说shouldSkip
>       if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
>          this.advisedBeans.put(cacheKey, Boolean.FALSE);
>          return null;
>       }
>    }
> 
>    TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
>    if (targetSource != null) {
>       if (StringUtils.hasLength(beanName)) {
>          this.targetSourcedBeans.add(beanName);
>       }
>       Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
>       Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
>       this.proxyTypes.put(cacheKey, proxy.getClass());
>       return proxy;
>    }
> 
>    return null;
> }
> 
>     @Override
> 	protected boolean shouldSkip(Class<?> beanClass, String beanName) {
> 		// TODO: Consider optimization by caching the list of the aspect names
> 		//查找所有标识了@Aspect注解的类，这里是重点，接着往下看
> 		List<Advisor> candidateAdvisors = findCandidateAdvisors();
> 		for (Advisor advisor : candidateAdvisors) {
> 			if (advisor instanceof AspectJPointcutAdvisor &&
> 					((AspectJPointcutAdvisor) advisor).getAspectName().equals(beanName)) {
> 				return true;
> 			}
> 		}
> 		return super.shouldSkip(beanClass, beanName);
> 	}
> 	
> 	protected List<Advisor> findCandidateAdvisors() {
> 		Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
> 		return this.advisorRetrievalHelper.findAdvisorBeans();
> 	}
> 	
> 	@Override
> 	protected List<Advisor> findCandidateAdvisors() {
> 		// Add all the Spring advisors found according to superclass rules.
> 		List<Advisor> advisors = super.findCandidateAdvisors();
> 		// Build Advisors for all AspectJ aspects in the bean factory.
> 		if (this.aspectJAdvisorsBuilder != null) {
> 			advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
> 		}
> 		return advisors;
> 	}
> 	
> 	public List<Advisor> buildAspectJAdvisors() {
> 	    //所有Aspect类的名称集合
> 		List<String> aspectNames = this.aspectBeanNames;
> 
> 		if (aspectNames == null) {
> 			synchronized (this) {
> 				aspectNames = this.aspectBeanNames;
> 				//这个双重检查是不是在学习安全的单例模式的时候见过
> 				if (aspectNames == null) {
> 					List<Advisor> advisors = new ArrayList<>();
> 					aspectNames = new ArrayList<>();
> 					//获取所有Bean名称
> 					String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
> 							this.beanFactory, Object.class, true, false);
> 					for (String beanName : beanNames) {
> 					    //判断是否符合条件，比如说有时会排除一些类，不让这些类注入进Spring
> 						if (!isEligibleBean(beanName)) {
> 							continue;
> 						}
> 						// We must be careful not to instantiate beans eagerly as in this case they
> 						// would be cached by the Spring container but would not have been weaved.
> 						Class<?> beanType = this.beanFactory.getType(beanName);
> 						if (beanType == null) {
> 							continue;
> 						}
> 						//判断Bean的Class上是否标识@Aspect注解
> 						if (this.advisorFactory.isAspect(beanType)) {
> 							aspectNames.add(beanName);
> 							AspectMetadata amd = new AspectMetadata(beanType, beanName);
> 							if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
> 								MetadataAwareAspectInstanceFactory factory =
> 										new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
> 								//下一步说，重点的重点（生成增强）	
> 								List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
> 								if (this.beanFactory.isSingleton(beanName)) {
> 								    //将解析的Bean名称及类上的增强缓存起来,每个Bean只解析一次
> 									this.advisorsCache.put(beanName, classAdvisors);
> 								}
> 								else {
> 									this.aspectFactoryCache.put(beanName, factory);
> 								}
> 								advisors.addAll(classAdvisors);
> 							}
> 							else {
> 								// Per target or per this.
> 								if (this.beanFactory.isSingleton(beanName)) {
> 									throw new IllegalArgumentException("Bean with name '" + beanName +
> 											"' is a singleton, but aspect instantiation model is not singleton");
> 								}
> 								MetadataAwareAspectInstanceFactory factory =
> 										new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
> 								this.aspectFactoryCache.put(beanName, factory);
> 								advisors.addAll(this.advisorFactory.getAdvisors(factory));
> 							}
> 						}
> 					}
> 					this.aspectBeanNames = aspectNames;
> 					return advisors;
> 				}
> 			}
> 		}
> 
> 		if (aspectNames.isEmpty()) {
> 			return Collections.emptyList();
> 		}
> 		List<Advisor> advisors = new ArrayList<>();
> 		for (String aspectName : aspectNames) {
> 		    //从缓存中获取当前Bean的切面实例，如果不为空，则指明当前Bean的Class标识了@Aspect，且有切面方法
> 			List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
> 			if (cachedAdvisors != null) {
> 				advisors.addAll(cachedAdvisors);
> 			}
> 			else {
> 				MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
> 				advisors.addAll(this.advisorFactory.getAdvisors(factory));
> 			}
> 		}
> 		return advisors;
> 	}
> 	
> 	advisorFactory.getAdvisors方法会从@Aspect标识的类上获取@Before，@Pointcut等注解的信息及其标识的方法的信息，生成     增强
> 	@Override
> 	public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
> 		Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
> 		String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
> 		//校验类的合法性相关
> 		validate(aspectClass);
> 
> 		// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
> 		// so that it will only instantiate once.
> 		MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
> 				new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);
> 
> 		List<Advisor> advisors = new ArrayList<>();
> 		//获取这个类所有的增强方法
> 		for (Method method : getAdvisorMethods(aspectClass)) {
> 		    //生成增强实例
> 			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
> 			if (advisor != null) {
> 				advisors.add(advisor);
> 			}
> 		}
> 
> 		// If it's a per target aspect, emit the dummy instantiating aspect.
> 		if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
> 			Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
> 			advisors.add(0, instantiationAdvisor);
> 		}
> 
> 		// Find introduction fields.
> 		for (Field field : aspectClass.getDeclaredFields()) {
> 			Advisor advisor = getDeclareParentsAdvisor(field);
> 			if (advisor != null) {
> 				advisors.add(advisor);
> 			}
> 		}
> 
> 		return advisors;
> 	}
> 	
> 	//获取类的增强方法
> 	private List<Method> getAdvisorMethods(Class<?> aspectClass) {
> 		final List<Method> methods = new ArrayList<>();
> 		ReflectionUtils.doWithMethods(aspectClass, method -> {
> 			// Exclude pointcuts
> 			//在@Aspect标识的类内部排除@Pointcut标识之外的所有方法，得到的方法集合包括继承自父类的方法，                         //包括继承自Object的方法
> 			if (AnnotationUtils.getAnnotation(method, Pointcut.class) == null) {
> 				methods.add(method);
> 			}
> 		}, ReflectionUtils.USER_DECLARED_METHODS);
> 		//对得到的所有方法排序，
> 		//如果方法标识了切面注解，则按@Around, @Before, @After, @AfterReturning, @AfterThrowing的顺序排序
> 		//如果没有标识这些注解，则按方法名称的字符串排序,
> 		//有注解的方法排在无注解的方法之前
> 		//最后的排序应该是这样的Around.class, Before.class, After.class, AfterReturning.class,                       //AfterThrowing.class。。。
> 		methods.sort(METHOD_COMPARATOR);
> 		return methods;
> 	}
> 	
> 	调用生成增强实例的方法
> 	@Override
> 	@Nullable
> 	public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
> 			int declarationOrderInAspect, String aspectName) {
> 
>         //再次校验类的合法性
> 		validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
> 
>         //切点表达式的包装类里面包含了切点表达式，下面说getPointcut
> 		AspectJExpressionPointcut expressionPointcut = getPointcut(
> 				candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
> 		if (expressionPointcut == null) {
> 			return null;
> 		}
> 
>         //根据方法、切点、AOP实例工厂、类名、序号生成切面实例，详细代码往下看
> 		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
> 				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
> 	}
> 	
> 	@Nullable
> 	private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
> 	    //查询方法上的切面注解，根据注解生成相应类型的AspectJAnnotation,在调用AspectJAnnotation的构造函数的同时
> 	    //根据注解value或pointcut属性得到切点表达式，有argNames则设置参数名称
> 		AspectJAnnotation<?> aspectJAnnotation =
> 				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
> 		//过滤那些不含@Before, @Around, @After, @AfterReturning, @AfterThrowing注解的方法
> 		if (aspectJAnnotation == null) {
> 			return null;
> 		}
> 
>         //生成带表达式的切面切入点，设置其切入点表达式
> 		AspectJExpressionPointcut ajexp =
> 				new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
> 		ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
> 		if (this.beanFactory != null) {
> 			ajexp.setBeanFactory(this.beanFactory);
> 		}
> 		return ajexp;
> 	}
> 	
> 	//InstantiationModelAwarePointcutAdvisorImpl的构造方法
> 	public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
> 			Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
> 			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
> 
> 		this.declaredPointcut = declaredPointcut;
> 		this.declaringClass = aspectJAdviceMethod.getDeclaringClass();
> 		this.methodName = aspectJAdviceMethod.getName();
> 		this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
> 		this.aspectJAdviceMethod = aspectJAdviceMethod;
> 		this.aspectJAdvisorFactory = aspectJAdvisorFactory;
> 		this.aspectInstanceFactory = aspectInstanceFactory;
> 		this.declarationOrder = declarationOrder;
> 		this.aspectName = aspectName;
> 
> 		if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
> 			// Static part of the pointcut is a lazy type.
> 			Pointcut preInstantiationPointcut = Pointcuts.union(
> 					aspectInstanceFactory.getAspectMetadata().getPerClausePointcut(), this.declaredPointcut);
> 
> 			// Make it dynamic: must mutate from pre-instantiation to post-instantiation state.
> 			// If it's not a dynamic pointcut, it may be optimized out
> 			// by the Spring AOP infrastructure after the first evaluation.
> 			this.pointcut = new PerTargetInstantiationModelPointcut(
> 					this.declaredPointcut, preInstantiationPointcut, aspectInstanceFactory);
> 			this.lazy = true;
> 		}
> 		else {
> 			// A singleton aspect.
> 			this.pointcut = this.declaredPointcut;
> 			this.lazy = false;
> 			//重点
> 			this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
> 		}
> 	}
> 	
> 	private Advice instantiateAdvice(AspectJExpressionPointcut pointcut) {
> 	    //生成增强
> 		Advice advice = this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pointcut,
> 				this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
> 		return (advice != null ? advice : EMPTY_ADVICE);
> 	}
> 	
> 	@Override
> 	@Nullable
> 	public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
> 			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
> 
> 		Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
> 		//再次校验
> 		validate(candidateAspectClass);
> 
> 		AspectJAnnotation<?> aspectJAnnotation =
> 				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
> 		if (aspectJAnnotation == null) {
> 			return null;
> 		}
> 
> 		// If we get here, we know we have an AspectJ method.
> 		// Check that it's an AspectJ-annotated class
> 		if (!isAspect(candidateAspectClass)) {
> 			throw new AopConfigException("Advice must be declared inside an aspect type: " +
> 					"Offending method '" + candidateAdviceMethod + "' in class [" +
> 					candidateAspectClass.getName() + "]");
> 		}
> 
> 		if (logger.isDebugEnabled()) {
> 			logger.debug("Found AspectJ method: " + candidateAdviceMethod);
> 		}
> 
> 		AbstractAspectJAdvice springAdvice;
> 
>         //根据注解类型生成不同的通知实例
> 		switch (aspectJAnnotation.getAnnotationType()) {
> 			case AtPointcut:
> 				if (logger.isDebugEnabled()) {
> 					logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
> 				}
> 				return null;
> 			case AtAround:
> 				springAdvice = new AspectJAroundAdvice(
> 						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
> 				break;
> 			case AtBefore:
> 				springAdvice = new AspectJMethodBeforeAdvice(
> 						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
> 				break;
> 			case AtAfter:
> 				springAdvice = new AspectJAfterAdvice(
> 						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
> 				break;
> 			case AtAfterReturning:
> 				springAdvice = new AspectJAfterReturningAdvice(
> 						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
> 				AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
> 				if (StringUtils.hasText(afterReturningAnnotation.returning())) {
> 					springAdvice.setReturningName(afterReturningAnnotation.returning());
> 				}
> 				break;
> 			case AtAfterThrowing:
> 				springAdvice = new AspectJAfterThrowingAdvice(
> 						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
> 				AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
> 				if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
> 					springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
> 				}
> 				break;
> 			default:
> 				throw new UnsupportedOperationException(
> 						"Unsupported advice type on method: " + candidateAdviceMethod);
> 		}
> 
> 		// Now to configure the advice...
> 		//设置通知方法所属的类
> 		springAdvice.setAspectName(aspectName);
> 		//设置通知的序号,同一个类中有多个切面注解标识的方法时,按上方说的排序规则来排序，
> 		//其序号就是此方法在列表中的序号，第一个就是0
> 		springAdvice.setDeclarationOrder(declarationOrder);
> 		//获取通知方法的所有参数
> 		String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
> 		//将通知方法上的参数设置到通知中
> 		if (argNames != null) {
> 			springAdvice.setArgumentNamesFromStringArray(argNames);
> 		}
> 		//计算参数绑定工作，此方法详解请接着往下看
> 		springAdvice.calculateArgumentBindings();
> 
> 		return springAdvice;
> 	}
> 	
> 	//校验方法参数并绑定
> 	public final synchronized void calculateArgumentBindings() {
> 		// The simple case... nothing to bind.
> 		if (this.argumentsIntrospected || this.parameterTypes.length == 0) {
> 			return;
> 		}
> 
> 		int numUnboundArgs = this.parameterTypes.length;
> 		Class<?>[] parameterTypes = this.aspectJAdviceMethod.getParameterTypes();
> 		//切面注解标识的方法第一个参数要求是JoinPoint,或StaticPart，若是@Around注解则也可以是ProceedingJoinPoint
> 		if (maybeBindJoinPoint(parameterTypes[0]) || maybeBindProceedingJoinPoint(parameterTypes[0]) ||
> 				maybeBindJoinPointStaticPart(parameterTypes[0])) {
> 			numUnboundArgs--;
> 		}
> 
> 		if (numUnboundArgs > 0) {
> 			// need to bind arguments by name as returned from the pointcut match
> 			//绑定属性
> 			bindArgumentsByName(numUnboundArgs);
> 		}
> 
> 		this.argumentsIntrospected = true;
> 	}
> 	
> 	//绑定属性
> 	private void bindArgumentsByName(int numArgumentsExpectingToBind) {
> 	    //获取方法参数的名称
> 		if (this.argumentNames == null) {
> 			this.argumentNames = createParameterNameDiscoverer().getParameterNames(this.aspectJAdviceMethod);
> 		}
> 		if (this.argumentNames != null) {
> 			// We have been able to determine the arg names.
> 			// 下面说
> 			bindExplicitArguments(numArgumentsExpectingToBind);
> 		}
> 		else {
> 			throw new IllegalStateException("Advice method [" + this.aspectJAdviceMethod.getName() + "] " +
> 					"requires " + numArgumentsExpectingToBind + " arguments to be bound by name, but " +
> 					"the argument names were not specified and could not be discovered.");
> 		}
> 	}
> 	
> 	
> 	private void bindExplicitArguments(int numArgumentsLeftToBind) {
> 		Assert.state(this.argumentNames != null, "No argument names available");
> 		//此属性用来存储方法未绑定的参数名称，及参数的序号
> 		this.argumentBindings = new HashMap<>();
> 
> 		int numExpectedArgumentNames = this.aspectJAdviceMethod.getParameterCount();
> 		if (this.argumentNames.length != numExpectedArgumentNames) {
> 			throw new IllegalStateException("Expecting to find " + numExpectedArgumentNames +
> 					" arguments to bind by name in advice, but actually found " +
> 					this.argumentNames.length + " arguments.");
> 		}
> 
> 		// So we match in number...argumentIndexOffset代表第一个未绑定参数的顺序 
> 		int argumentIndexOffset = this.parameterTypes.length - numArgumentsLeftToBind;
> 		for (int i = argumentIndexOffset; i < this.argumentNames.length; i++) {
> 			this.argumentBindings.put(this.argumentNames[i], i);
> 		}
> 
> 		// Check that returning and throwing were in the argument names list if
> 		// specified, and find the discovered argument types.
> 		//如果是@AfterReturning注解的returningName 有值，验证，解析，同时得到定义返回值的类型
> 		if (this.returningName != null) {
> 			if (!this.argumentBindings.containsKey(this.returningName)) {
> 				throw new IllegalStateException("Returning argument name '" + this.returningName +
> 						"' was not bound in advice arguments");
> 			}
> 			else {
> 				Integer index = this.argumentBindings.get(this.returningName);
> 				this.discoveredReturningType = this.aspectJAdviceMethod.getParameterTypes()[index];
> 				this.discoveredReturningGenericType = this.aspectJAdviceMethod.getGenericParameterTypes()[index];
> 			}
> 		}
> 		if (this.throwingName != null) {
> 			if (!this.argumentBindings.containsKey(this.throwingName)) {
> 				throw new IllegalStateException("Throwing argument name '" + this.throwingName +
> 						"' was not bound in advice arguments");
> 			}
> 			else {
> 				Integer index = this.argumentBindings.get(this.throwingName);
> 				this.discoveredThrowingType = this.aspectJAdviceMethod.getParameterTypes()[index];
> 			}
> 		}
> 
> 		// configure the pointcut expression accordingly.
> 		configurePointcutParameters(this.argumentNames, argumentIndexOffset);
> 	}
> 	
> 	private void configurePointcutParameters(String[] argumentNames, int argumentIndexOffset) {
> 		int numParametersToRemove = argumentIndexOffset;
> 		if (this.returningName != null) {
> 			numParametersToRemove++;
> 		}
> 		if (this.throwingName != null) {
> 			numParametersToRemove++;
> 		}
> 		String[] pointcutParameterNames = new String[argumentNames.length - numParametersToRemove];
> 		Class<?>[] pointcutParameterTypes = new Class<?>[pointcutParameterNames.length];
> 		Class<?>[] methodParameterTypes = this.aspectJAdviceMethod.getParameterTypes();
> 
> 		int index = 0;
> 		for (int i = 0; i < argumentNames.length; i++) {
> 			if (i < argumentIndexOffset) {
> 				continue;
> 			}
> 			if (argumentNames[i].equals(this.returningName) ||
> 				argumentNames[i].equals(this.throwingName)) {
> 				continue;
> 			}
> 			pointcutParameterNames[index] = argumentNames[i];
> 			pointcutParameterTypes[index] = methodParameterTypes[i];
> 			index++;
> 		}
> 
>         //剩余的未绑定的参数会赋值给AspectJExpressionPointcut(表达式形式的切入点)的属性，以备后续使用
> 		this.pointcut.setParameterNames(pointcutParameterNames);
> 		this.pointcut.setParameterTypes(pointcutParameterTypes);
> 	}
> ```

##### postProcessAfterInstantiation逻辑

> 这个方法是在bean实例化之后调用的，它是适用于所有需要被代理的类的
>
> ```
> @Override
> public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
>    if (bean != null) {
>       Object cacheKey = getCacheKey(bean.getClass(), beanName);
>       if (this.earlyProxyReferences.remove(cacheKey) != bean) {
>          //下面说
>          return wrapIfNecessary(bean, beanName, cacheKey);
>       }
>    }
>    return bean;
> }
> 
>     protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
>         //如果已经处理过
> 		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
> 			return bean;
> 		}
> 		//如果当前类是增强类
> 		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
> 			return bean;
> 		}
> 		//查看类是否是基础设施类，或者是否被排除
> 		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
> 			this.advisedBeans.put(cacheKey, Boolean.FALSE);
> 			return bean;
> 		}
> 
> 		// Create proxy if we have advice.
> 		//校验此类是否应该被代理，获取这个类的增强(重点，下面说)
> 		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
> 		//如果获取到了增强则需要针对增强创建代理
> 		if (specificInterceptors != DO_NOT_PROXY) {
> 			this.advisedBeans.put(cacheKey, Boolean.TRUE);
> 			//创建代理（重点，下面说）
> 			Object proxy = createProxy(
> 					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
> 			this.proxyTypes.put(cacheKey, proxy.getClass());
> 			return proxy;
> 		}
> 
> 		this.advisedBeans.put(cacheKey, Boolean.FALSE);
> 		return bean;
> 	}
> 	
> 	@Override
> 	@Nullable
> 	protected Object[] getAdvicesAndAdvisorsForBean(
> 			Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
>         //重点
> 		List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
> 		if (advisors.isEmpty()) {
> 			return DO_NOT_PROXY;
> 		}
> 		return advisors.toArray();
> 	}
> 	
> 	protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
> 	    //获取容器中的所有增强
> 		List<Advisor> candidateAdvisors = findCandidateAdvisors();
> 		//验证beanClass是否该被代理，如果应该，则返回适用于这个bean的增强
> 		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
> 		extendAdvisors(eligibleAdvisors);
> 		if (!eligibleAdvisors.isEmpty()) {
> 			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
> 		}
> 		return eligibleAdvisors;
> 	}
> 	
> 	上述方法又分为两部分，获取全部和根据全部处理bean相关：
> 	获取全部增强：
> 	@Override
> 	protected List<Advisor> findCandidateAdvisors() {
> 		// Add all the Spring advisors found according to superclass rules.
> 		List<Advisor> advisors = super.findCandidateAdvisors();
> 		// Build Advisors for all AspectJ aspects in the bean factory.
> 		if (this.aspectJAdvisorsBuilder != null) {
> 		    //重点
> 			advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
> 		}
> 		return advisors;
> 	}
> 	
> 	//获取增强的代码实现
> 	//代码逻辑：1.获取所有beanName 2.找出所有标记AspectJ注解的类 3.对标记Aspect的类提取增强
> 	public List<Advisor> buildAspectJAdvisors() {
> 		List<String> aspectNames = this.aspectBeanNames;
> 
> 		if (aspectNames == null) {
> 			synchronized (this) {
> 				aspectNames = this.aspectBeanNames;
> 				if (aspectNames == null) {
> 					List<Advisor> advisors = new ArrayList<>();
> 					aspectNames = new ArrayList<>();
> 					//获取所有的bean
> 					String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
> 							this.beanFactory, Object.class, true, false);
> 					for (String beanName : beanNames) {
> 					    //校验不合法的类，Spring的一个扩展点，可以从子类中做排除切面的操作
> 						if (!isEligibleBean(beanName)) {
> 							continue;
> 						}
> 						// We must be careful not to instantiate beans eagerly as in this case they
> 						// would be cached by the Spring container but would not have been weaved.
> 						Class<?> beanType = this.beanFactory.getType(beanName);
> 						if (beanType == null) {
> 							continue;
> 						}
> 						//是否带有Aspect注解
> 						if (this.advisorFactory.isAspect(beanType)) {
> 							aspectNames.add(beanName);
> 							AspectMetadata amd = new AspectMetadata(beanType, beanName);
> 							if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
> 								MetadataAwareAspectInstanceFactory factory =
> 										new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
> 							    //解析所有的增强方法，下面说
> 								List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
> 								if (this.beanFactory.isSingleton(beanName)) {
> 									this.advisorsCache.put(beanName, classAdvisors);
> 								}
> 								else {
> 									this.aspectFactoryCache.put(beanName, factory);
> 								}
> 								advisors.addAll(classAdvisors);
> 							}
> 							else {
> 								// Per target or per this.
> 								if (this.beanFactory.isSingleton(beanName)) {
> 									throw new IllegalArgumentException("Bean with name '" + beanName +
> 											"' is a singleton, but aspect instantiation model is not singleton");
> 								}
> 								MetadataAwareAspectInstanceFactory factory =
> 										new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
> 								this.aspectFactoryCache.put(beanName, factory);
> 								advisors.addAll(this.advisorFactory.getAdvisors(factory));
> 							}
> 						}
> 					}
> 					this.aspectBeanNames = aspectNames;
> 					return advisors;
> 				}
> 			}
> 		}
> 
> 		if (aspectNames.isEmpty()) {
> 			return Collections.emptyList();
> 		}
> 		List<Advisor> advisors = new ArrayList<>();
> 		for (String aspectName : aspectNames) {
> 			List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
> 			if (cachedAdvisors != null) {
> 				advisors.addAll(cachedAdvisors);
> 			}
> 			else {
> 				MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
> 				advisors.addAll(this.advisorFactory.getAdvisors(factory));
> 			}
> 		}
> 		return advisors;
> 	}
> 	
> 	//各个增强器获取方法的实现
> 	@Override
> 	public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
> 	    //获取所有Aspect类、类名称、并校验
> 		Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
> 		String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
> 		validate(aspectClass);
> 
> 		// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
> 		// so that it will only instantiate once.
> 		MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
> 				new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);
> 
> 		List<Advisor> advisors = new ArrayList<>();
> 		//取出类的所有方法
> 		for (Method method : getAdvisorMethods(aspectClass)) {
> 		    //获取增强方法，往下看
> 			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
> 			if (advisor != null) {
> 				advisors.add(advisor);
> 			}
> 		}
> 
> 		// If it's a per target aspect, emit the dummy instantiating aspect.
> 		// 如果需要增强且配置了延迟增强则在第一个位置添加同步实例化增强方法
> 		if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
> 			Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
> 			advisors.add(0, instantiationAdvisor);
> 		}
> 
> 		// Find introduction fields.
> 		// 获取属性中配置DeclareParents注解的增强
> 		for (Field field : aspectClass.getDeclaredFields()) {
> 			Advisor advisor = getDeclareParentsAdvisor(field);
> 			if (advisor != null) {
> 				advisors.add(advisor);
> 			}
> 		}
> 
> 		return advisors;
> 	}
> 	
> 	//普通增强的获取
> 	@Override
> 	@Nullable
> 	public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
> 			int declarationOrderInAspect, String aspectName) {
> 
> 		validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
> 
>         //获取切点
> 		AspectJExpressionPointcut expressionPointcut = getPointcut(
> 				candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
> 		if (expressionPointcut == null) {
> 			return null;
> 		}
> 
>         //根据切点生成增强
> 		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
> 				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
> 	}
> 	
> 	//上面普通增强的获取又分为两步先说切点信息的获取
> 	public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
> 			Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
> 			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
> 
> 		this.declaredPointcut = declaredPointcut;
> 		this.declaringClass = aspectJAdviceMethod.getDeclaringClass();
> 		this.methodName = aspectJAdviceMethod.getName();
> 		this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
> 		this.aspectJAdviceMethod = aspectJAdviceMethod;
> 		this.aspectJAdvisorFactory = aspectJAdvisorFactory;
> 		this.aspectInstanceFactory = aspectInstanceFactory;
> 		this.declarationOrder = declarationOrder;
> 		this.aspectName = aspectName;
> 
> 		if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
> 			// Static part of the pointcut is a lazy type.
> 			Pointcut preInstantiationPointcut = Pointcuts.union(
> 					aspectInstanceFactory.getAspectMetadata().getPerClausePointcut(), this.declaredPointcut);
> 
> 			// Make it dynamic: must mutate from pre-instantiation to post-instantiation state.
> 			// If it's not a dynamic pointcut, it may be optimized out
> 			// by the Spring AOP infrastructure after the first evaluation.
> 			this.pointcut = new PerTargetInstantiationModelPointcut(
> 					this.declaredPointcut, preInstantiationPointcut, aspectInstanceFactory);
> 			this.lazy = true;
> 		}
> 		else {
> 			// A singleton aspect.
> 			this.pointcut = this.declaredPointcut;
> 			this.lazy = false;
> 			//初始化对应的增强器，重点
> 			this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
> 		}
> 	}
> 	
> 	private Advice instantiateAdvice(AspectJExpressionPointcut pointcut) {
> 	    //下面说
> 		Advice advice = this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pointcut,
> 				this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
> 		return (advice != null ? advice : EMPTY_ADVICE);
> 	}
> 	
> 	@Override
> 	@Nullable
> 	public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
> 			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
> 
> 		Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
> 		validate(candidateAspectClass);
> 
> 		AspectJAnnotation<?> aspectJAnnotation =
> 				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
> 		if (aspectJAnnotation == null) {
> 			return null;
> 		}
> 
> 		// If we get here, we know we have an AspectJ method.
> 		// Check that it's an AspectJ-annotated class
> 		if (!isAspect(candidateAspectClass)) {
> 			throw new AopConfigException("Advice must be declared inside an aspect type: " +
> 					"Offending method '" + candidateAdviceMethod + "' in class [" +
> 					candidateAspectClass.getName() + "]");
> 		}
> 
> 		if (logger.isDebugEnabled()) {
> 			logger.debug("Found AspectJ method: " + candidateAdviceMethod);
> 		}
> 
> 		AbstractAspectJAdvice springAdvice;
> 
>         //根据不同的注解类型封装不同的增强器
> 		switch (aspectJAnnotation.getAnnotationType()) {
> 			case AtPointcut:
> 				if (logger.isDebugEnabled()) {
> 					logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
> 				}
> 				return null;
> 			case AtAround:
> 				springAdvice = new AspectJAroundAdvice(
> 						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
> 				break;
> 			case AtBefore:
> 				springAdvice = new AspectJMethodBeforeAdvice(
> 						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
> 				break;
> 			case AtAfter:
> 				springAdvice = new AspectJAfterAdvice(
> 						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
> 				break;
> 			case AtAfterReturning:
> 				springAdvice = new AspectJAfterReturningAdvice(
> 						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
> 				AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
> 				if (StringUtils.hasText(afterReturningAnnotation.returning())) {
> 					springAdvice.setReturningName(afterReturningAnnotation.returning());
> 				}
> 				break;
> 			case AtAfterThrowing:
> 				springAdvice = new AspectJAfterThrowingAdvice(
> 						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
> 				AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
> 				if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
> 					springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
> 				}
> 				break;
> 			default:
> 				throw new UnsupportedOperationException(
> 						"Unsupported advice type on method: " + candidateAdviceMethod);
> 		}
> 
> 		// Now to configure the advice...
> 		springAdvice.setAspectName(aspectName);
> 		springAdvice.setDeclarationOrder(declarationOrder);
> 		String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
> 		if (argNames != null) {
> 			springAdvice.setArgumentNamesFromStringArray(argNames);
> 		}
> 		springAdvice.calculateArgumentBindings();
> 
> 		return springAdvice;
> 	}
> 	
> 	上面findEligibleAdvisors方法代码逻辑先说了获取全部增强，接下来看看怎么为当前的Bean匹配自己的增强
> 	protected List<Advisor> findAdvisorsThatCanApply(
> 			List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {
> 
> 		ProxyCreationContext.setCurrentProxiedBeanName(beanName);
> 		try {
> 		    //下面说
> 			return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
> 		}
> 		finally {
> 			ProxyCreationContext.setCurrentProxiedBeanName(null);
> 		}
> 	}
> 	
> 	public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
> 		if (candidateAdvisors.isEmpty()) {
> 			return candidateAdvisors;
> 		}
> 		List<Advisor> eligibleAdvisors = new ArrayList<>();
> 		for (Advisor candidate : candidateAdvisors) {
> 		    //处理引介增强，重点
> 			if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
> 				eligibleAdvisors.add(candidate);
> 			}
> 		}
> 		boolean hasIntroductions = !eligibleAdvisors.isEmpty();
> 		for (Advisor candidate : candidateAdvisors) {
> 			if (candidate instanceof IntroductionAdvisor) {
> 				// already processed
> 				continue;
> 			}
> 			//对普通bean的处理
> 			if (canApply(candidate, clazz, hasIntroductions)) {
> 				eligibleAdvisors.add(candidate);
> 			}
> 		}
> 		return eligibleAdvisors;
> 	}
> 	
> 	//引介增强与普通bean的处理最后都是进的同一个方法，只不过是引介增强的第三个参数默认使用的false
> 	public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
> 		Assert.notNull(pc, "Pointcut must not be null");
> 		//切点上是否存在排除类的配置
> 		if (!pc.getClassFilter().matches(targetClass)) {
> 			return false;
> 		}
> 
>         //验证注解的作用域是否可以作用于方法上
> 		MethodMatcher methodMatcher = pc.getMethodMatcher();
> 		if (methodMatcher == MethodMatcher.TRUE) {
> 			// No need to iterate the methods if we're matching any method anyway...
> 			return true;
> 		}
> 
> 		IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
> 		if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
> 			introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
> 		}
> 
> 		Set<Class<?>> classes = new LinkedHashSet<>();
> 		if (!Proxy.isProxyClass(targetClass)) {
> 			classes.add(ClassUtils.getUserClass(targetClass));
> 		}
> 		classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));
> 
> 		for (Class<?> clazz : classes) {
> 			Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
> 			for (Method method : methods) {
> 			    //获取类所实现的所有接口和所有类层级的方法，循环验证
> 				if (introductionAwareMethodMatcher != null ?
> 						introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
> 						methodMatcher.matches(method, targetClass)) {
> 					return true;
> 				}
> 			}
> 		}
> 
> 		return false;
> 	}
> 	
> 	//现在所有的bean对应的增强都已经获取到了，那么就可以根据类的所有增强数组创建代理
> 	//回到最初的wrapIfNecessary方法，创建代理createProxy
> 	protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
> 			@Nullable Object[] specificInterceptors, TargetSource targetSource) {
> 
> 		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
> 			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
> 		}
> 
> 		ProxyFactory proxyFactory = new ProxyFactory();
> 		proxyFactory.copyFrom(this);
> 
>         //判断是否使用Cglib动态代理
> 		if (!proxyFactory.isProxyTargetClass()) {
> 		    //如果配置开启使用则直接设置开启
> 			if (shouldProxyTargetClass(beanClass, beanName)) {
> 				proxyFactory.setProxyTargetClass(true);
> 			}
> 			else {
> 			    //如果没有配置开启则判断bean是否有合适的接口使用JDK的动态代理（JDK动态代理必须是带有接口的类，                         //如果类没有实现任何接口则只能使用Cglib动态代理）
> 				evaluateProxyInterfaces(beanClass, proxyFactory);
> 			}
> 		}
> 
>         //添加所有增强
> 		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
> 		proxyFactory.addAdvisors(advisors);
> 		//设置要代理的类
> 		proxyFactory.setTargetSource(targetSource);
> 		//Spring的一个扩展点，默认实现为空。留给我们在需要对代理进行特殊操作的时候实现
> 		customizeProxyFactory(proxyFactory);
> 
> 		proxyFactory.setFrozen(this.freezeProxy);
> 		if (advisorsPreFiltered()) {
> 			proxyFactory.setPreFiltered(true);
> 		}
> 
>         //使用代理工厂获取代理对象，下面说
> 		return proxyFactory.getProxy(getProxyClassLoader());
> 	}
> 	
> 	public Object getProxy(@Nullable ClassLoader classLoader) {
> 		return createAopProxy().getProxy(classLoader);
> 	}
> 	
> 	protected final synchronized AopProxy createAopProxy() {
> 		if (!this.active) {
> 			activate();
> 		}
> 		return getAopProxyFactory().createAopProxy(this);
> 	}
> 	
> 	@Override
> 	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
> 		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
> 			Class<?> targetClass = config.getTargetClass();
> 			if (targetClass == null) {
> 				throw new AopConfigException("TargetSource cannot determine target class: " +
> 						"Either an interface or a target is required for proxy creation.");
> 			}
> 			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
> 			    //下面说一下JDK动态代理的实现
> 				return new JdkDynamicAopProxy(config);
> 			}
> 			return new ObjenesisCglibAopProxy(config);
> 		}
> 		else {
> 			return new JdkDynamicAopProxy(config);
> 		}
> 	}
> 	
> 	//JdkDynamicAopProxy.java中
> 	@Override
> 	@Nullable
> 	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
> 		Object oldProxy = null;
> 		boolean setProxyContext = false;
> 
> 		TargetSource targetSource = this.advised.targetSource;
> 		Object target = null;
> 
> 		try {
> 		    //equals方法处理
> 			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
> 				// The target does not implement the equals(Object) method itself.
> 				return equals(args[0]);
> 			}
> 			//hash代码处理
> 			else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
> 				// The target does not implement the hashCode() method itself.
> 				return hashCode();
> 			}
> 			else if (method.getDeclaringClass() == DecoratingProxy.class) {
> 				// There is only getDecoratedClass() declared -> dispatch to proxy config.
> 				return AopProxyUtils.ultimateTargetClass(this.advised);
> 			}
> 			else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
> 					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
> 				// Service invocations on ProxyConfig with the proxy config...
> 				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
> 			}
> 
> 			Object retVal;
> 
>             //如果配置内部方法调用的增强
> 			if (this.advised.exposeProxy) {
> 				// Make invocation available if necessary.
> 				oldProxy = AopContext.setCurrentProxy(proxy);
> 				setProxyContext = true;
> 			}
> 
> 			// Get as late as possible to minimize the time we "own" the target,
> 			// in case it comes from a pool.
> 			target = targetSource.getTarget();
> 			Class<?> targetClass = (target != null ? target.getClass() : null);
> 
>             // 获取当前方法的拦截器链
> 			// Get the interception chain for this method.
> 			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
> 
> 			// Check whether we have any advice. If we don't, we can fallback on direct
> 			// reflective invocation of the target, and avoid creating a MethodInvocation.
> 			if (chain.isEmpty()) {
> 				// We can skip creating a MethodInvocation: just invoke the target directly
> 				// Note that the final invoker must be an InvokerInterceptor so we know it does
> 				// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
> 				//如果没有拦截器直接调用切点方法
> 				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
> 				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
> 			}
> 			else {
> 				// We need to create a method invocation...
> 				MethodInvocation invocation =
> 						new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
> 				// Proceed to the joinpoint through the interceptor chain.
> 				//执行拦截器链，重点，往下看
> 				retVal = invocation.proceed();
> 			}
> 
> 			// Massage return value if necessary.
> 			Class<?> returnType = method.getReturnType();
> 			//返回结果
> 			if (retVal != null && retVal == target &&
> 					returnType != Object.class && returnType.isInstance(proxy) &&
> 					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
> 				// Special case: it returned "this" and the return type of the method
> 				// is type-compatible. Note that we can't help if the target sets
> 				// a reference to itself in another returned object.
> 				retVal = proxy;
> 			}
> 			else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
> 				throw new AopInvocationException(
> 						"Null return value from advice does not match primitive return type for: " + method);
> 			}
> 			return retVal;
> 		}
> 		finally {
> 			if (target != null && !targetSource.isStatic()) {
> 				// Must have come from TargetSource.
> 				targetSource.releaseTarget(target);
> 			}
> 			if (setProxyContext) {
> 				// Restore old proxy.
> 				AopContext.setCurrentProxy(oldProxy);
> 			}
> 		}
> 	}
> 	
> 	//所有的增强都在这个拦截器里面了.
> 	@Override
> 	@Nullable
> 	public Object proceed() throws Throwable {
> 		// We start with an index of -1 and increment early.
> 		//  执行完所有的增强后执行切点方法
> 		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
> 			return invokeJoinpoint();
> 		}
> 
>         //获取下一个要执行的拦截器
> 		Object interceptorOrInterceptionAdvice =
> 				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
> 		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
> 			// Evaluate dynamic method matcher here: static part will already have
> 			// been evaluated and found to match.
> 			InterceptorAndDynamicMethodMatcher dm =
> 					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
> 			Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
> 			if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
> 				return dm.interceptor.invoke(this);
> 			}
> 			else {
> 				// Dynamic matching failed.
> 				// Skip this interceptor and invoke the next in the chain.
> 				return proceed();
> 			}
> 		}
> 		else {
> 			// It's an interceptor, so we just invoke it: The pointcut will have
> 			// been evaluated statically before this object was constructed.
> 			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
> 		}
> 	}
> ```
>

##### 总结

aop原理底层从@EnableAspectJAutoProxy注解开始，注解开始@Import(AspectJAutoProxyRegistrar.class)将这个类加入到IOC容器里被spring管理，注解**第一个属性proxyTargetClass**，默认false，尝试采用JDK动态代理织入增强(如果当前类没有实现接口则还是会使用CGLIB)，如果设置为true则强制采用CGLIB动态代理织入增强。**第二个属性exposeProxy**，通过aop框架暴露该代理对象，aopContext能够访问。为了解决类内部方法之间调用时无法增强的问题。

导入了AspectJAutoProxyRegistrar类，这个类只有registerabeanDefinitions方法，方法先注册了一个aop代理实现的Bean，AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry)，这个方法的主要功能是注册和升级。注册的底层逻辑是先根据@point注解定义的切点来自动代理与表达式匹配的类（registerOrEscalateApcAsRequired，先判断优先级，如果优先级较高则替换原先的bean，之后注册AnnotationAwareAspectJAutoProxyCreator到容器中，此类负责基于注解的aop动态代理实现）

**AnnotationAwareAspectJAutoProxyCreator这个类间接实现了BeanPostProcessor接口。我们之前在对SpringIOC的源码进行解析时提到过，Spring在实例化Bean的前后会分别调用方法postProcessBeforeInstantiation和postProcessAfterInstantiation。AOP的整体逻辑就是通过这两个方法来实现的**。

先说postProcessBeforeInstantiation逻辑。这个方法是再bean实例化之前调用的，主要针对切面类。这个方法不在AnnotationAwareAspectJAutoProxyCreator这个类中，而是在其父类AbstractAutoProxyCreator中。这个方法的第二个if条件加载所有增强（shouldSkip）,**shouldSkip里第一行代码findCandidateAdisors方法会查找所有标识了@Aspect注解的类，这个方法先获取所有Aspect类的名称集合，然后同步机制双重检查名称集合（单例模式那块），如果名称集合为空的话获取所有Bean名称，遍历所有的beanName判断是否符合条件（有时会排除一些类，不让这些类注入进spring，不符合continue），之后判断Bean的class上是否标识@Aspect注解，标识的话加入到名称集合。之后生成增强（getAdvisors）（重点），再将解析的bean名称及类上的增强缓存起来，每个bean只解析一次。最后从缓存中获取Bean的切面实例，如果不为空，则指明当前bean的class标识了@Aspect,且有切面方法**。生成增强方法getAdvisors会从@Aspect标识的类上获取@Before，@Pointcut等注解的信息的信息及其标识的方法的信息，生成增强。先检验类的合法性，获取这个类的增强方法（**获取的逻辑先定义一个Arraylist集合，往集合里添加这个类上所有@Pointcut标识的方法（包括继承父类的方法和Object类的方法），然后对所有的方法进行排序，排序的顺序按照@Around,@Before,@After,@AfterReturning,@AfterThrowing排序，没标注这些注解按照方法名称的字符串排序，有注解的方法排在没注解的前面**），给每个方法生成增强实例（getAdvisors，**再次校验类的合法性，获取切点getPointcut，根据方法、切点、AOP实例工厂、类名、序号生成切面实例new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,this, aspectInstanceFactory, declarationOrderInAspect, aspectName)**）。**获取切点**逻辑是查询方法上的切面注解，根据注解生成相应类型的AspectJAnnotation，在调用AspectJAnnotation的构造函数的同时根据注解value或pointcut属性得到切点表达式，有argNames则设置参数名称（AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod)），过滤那些不含@Before, @Around, @After, @AfterReturning, @AfterThrowing注解的方法，生成带表达式的切面切入点，设置其切入点表达式。**生成切面实例**实现类构造方法底层首先根据注解类型生成不同的通知实例，设置通知方法所属的类，设置通知的序号，同一个类中有多个切面注解标识的方法时,按上方说的排序规则来排序，其序号就是此方法在列表中的序号，第一个就是0，获取通知方法的所有参数，将通知方法上的参数设置到通知中，最后计算参数绑定工作（springAdvice.calculateArgumentBindings()），**绑定的逻辑先判断切面注解标识的方法第一个参数要求是JoinPoint,或StaticPart，若是@Around注解则也可以是ProceedingJoinPoint，结果true的话未绑定参数--，未绑定参数大于0的话然后绑定属性。绑定属性逻辑先创建一个map用来存储方法未绑定的参数名称，及参数的序号，然后添加未绑定的参数，接着如果是@AfterReturning注解的returningName 有值，验证，解析，同时得到定义返回值的类型，剩余的未绑定的参数会赋值给AspectJExpressionPointcut(表达式形式的切入点)的属性，以备后续使用。**

postProcessAfterInstantiation逻辑在bean的实例化后调用，首先一大堆的判断，之后校验此类是否应该被代理，**获取这个类的增强**，如果获取到了增强则需要针对增强创建代理，**最后创建代理**。获取这个类的增强逻辑先获取容器中所有的增强，之后验证beanClass是否该被代理，如果应该，则返回适用于这个bean的增强。其中获取全部增强的逻辑是（1.获取所有beanName 2.找出所有标记AspectJ注解的类 3.对标记Aspect的类提取增强），第二步大概逻辑是先取出类的所有方法，获取增强方法。增强方法getAdvisor逻辑是先获取切点，再对切点进行增强。切点信息的获取先初始化增强器，再根据注解的不同封装不同的增强器。**创建代理**，判断是否使用cglib增强（没开启判断如果有可用的接口用JDK增强），添加增强，创建代理工厂添加代理对象getProxy，这个方法里的createAopProxy创建增强，进入方法return new JdkDynamicAopProxy返回了一个JDK增强，方法内会先处理equals方法和hascode方法，然后获取当前方法的拦截器链，判断如果没有拦截器的话就直接调用切点方法，有的话先执行所有的增强后执行切点方法，再获取下一个拦截器。