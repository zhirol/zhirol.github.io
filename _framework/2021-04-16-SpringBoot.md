---
title: SpringBoot源码阅读笔记
author: Irol Zhou
date: 2021-04-16
category: framework
layout: post
---

## 自动装配原理

@EnableAutoConfiguration

AutoConfigurationImportSelector 自动装载配置类

selectImports() 获取所有符合条件的类的全限定类名，这些类需要被加载到 IoC 容器中

getAutoConfigurationEntry()

getAttributes() 获取exclude和excludeName

getCandidateConfigurations() 读取META-INF/spring.factories，包括下面的spring.factories和所有starter的spring.factories

```
spring-boot/spring-boot-project/spring-boot-autoconfigure/src/main/resources/META-INF/spring.factories
```

去重

去除exclude

run方法中通过反射实例化类,按@Conditonal条件装载bean



## @SpringBootApplication注解

`@SpringBootApplication` 包含`@EnableAutoConfiguration`

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    ...
}
```

`@EnableAutoConfiguration` 包含 `@Import(AutoConfigurationImportSelector.class)`

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	String[] excludeName() default {};

}
```

`AutoConfigurationImportSelector`中的`selectImports`方法

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {

	// 读取spring.boot.enableautoconfiguration配置，若为false则返回空数组
	if (!isEnabled(annotationMetadata)) {
		return NO_IMPORTS;
	}
	// getAutoConfigurationEntry读取配置列表
	AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
	return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}

// getAutoConfigurationEntry
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return EMPTY_ENTRY;
	}
	// 1.读取@EnableAutoConfiguration注解中exclude, excludeName两个属性，利用了反射
	AnnotationAttributes attributes = getAttributes(annotationMetadata);
	// 2.读取所有META-INF/spring.factories文件
	List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
	// 3.通过HashSet去重
	configurations = removeDuplicates(configurations);
	// 4.获取attributes中排除的类Set集合
	Set<String> exclusions = getExclusions(annotationMetadata, attributes);
	// 5.检查exclusions中是否有spring.factories未指定的类，有则抛出异常
	checkExcludedClasses(configurations, exclusions);
    // 6.移除显式指定的排除类
	configurations.removeAll(exclusions);
    // 7.过滤掉不满足@ConditionalXXX的类
	configurations = getConfigurationClassFilter().filter(configurations);
    // 8.将configurations和exclusions绑定到AutoConfigurationImportListener
	fireAutoConfigurationImportEvents(configurations, exclusions);
    // 9.返回configurations和exclusions信息
	return new AutoConfigurationEntry(configurations, exclusions);
}
```

所有的META-INF/spring.factories文件包括spring-boot-autoconfigure包下的spring.factories `spring-boot/spring-boot-project/spring-boot-autoconfigure/src/main/resources/META-INF/spring.factories`，以及各种引入的starter中的META-INF/spring.factories文件



## SpringApplication.run方法

### 1.new SpringApplication()

```java
@SpringBootApplication
@ServletComponentScan
public class StudyApplication {

    public static void main(String[] args) {
        SpringApplication.run(StudyApplication.class, args);
    }

}

public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
	return run(new Class<?>[] { primarySource }, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
	return new SpringApplication(primarySources).run(args);
}

public SpringApplication(Class<?>... primarySources) {
	this(null, primarySources);
}

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
	this.resourceLoader = resourceLoader;
	Assert.notNull(primarySources, "PrimarySources must not be null");
	this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 1.判断当前web环境
	this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 2.实例化spring.factories中定义的ApplicationContextInitializer配置类
    // 并将其设置为当前应用的ApplicationContextInitializer
	setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 3.实例化spring.factories中定义的ApplicationListener配置类
    // 并将其设置为当前应用的ApplicationListener
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 4.推导出当前应用启动的main方法所在的类，因为不强制在默认生成的Application中启动应用
	this.mainApplicationClass = deduceMainApplicationClass();
}
```

#### 1.1 判断当前web环境

Springboot定义了三种web环境

none：表示当前的应用即不是一个web应用也不是一个reactive应用，是一个纯后台的应用。

servlet：表示当前应用是一个标准的web应用。

reactive：表示是一个响应式的web应用,是spring5当中的新特性。

判断环境的依据就是根据Classloader中加载的类，如果是servlet，则表示是web，如果是DispatcherHandler，则表示是一个reactive应用，如果两者都不存在，则表示是一个非web环境的应用。

```java
// 1.判断当前web环境，
static WebApplicationType deduceFromClasspath() {
    // org.springframework.web.reactive.DispatcherHandler
	if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && 
        // org.springframework.web.servlet.DispatcherServlet
        !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
        // org.glassfish.jersey.servlet.ServletContainer
			&& !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
		return WebApplicationType.REACTIVE;
	}
    // {"javax.servlet.Servlet",
    // "org.springframework.web.context.ConfigurableWebApplicationContext"}
	for (String className : SERVLET_INDICATOR_CLASSES) {
		if (!ClassUtils.isPresent(className, null)) {
			return WebApplicationType.NONE;
		}
	}
	return WebApplicationType.SERVLET;
}
```

#### 1.2 实例化ApplicationContextInitializer配置类

getSpringFactoriesInstances方法，对spring.factories中的配置类进行实例化，随后根据传入的type参数进行筛选，并对结果进行排序

```java
// 2.实例化配置类后按type筛选，并对结果进行排序
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
	return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    // 2.1获取类加载器
	ClassLoader classLoader = getClassLoader();
	// 2.2调用loadFactoryNames方法，获取指定type的类名，并用hashset去重
	Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 2.3依次实例化配置类
	List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    // 2.4对实例化结果进行排序
	AnnotationAwareOrderComparator.sort(instances);
	return instances;
```



```java
// 2.2调用loadFactoryNames方法，获取指定type的类名，并用hashset去重
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    String factoryTypeName = factoryType.getName();
    // 2.2.1加载spring.factories中的配置类信息，并筛选出key为factoryTypeName的类名
    return (List)loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}

// 2.2.1加载spring.factories中的配置类信息，在这里读取并处理spring.factories
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    // 查找缓存，若缓存已经有了，则直接返回
    MultiValueMap<String, String> result = (MultiValueMap)cache.get(classLoader);
    if (result != null) {
        return result;
    } else {
        try {
            // 获取包含META-INF/spring.factories的资源路径
            Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");
            // 将spring.factories中的配置类信息组装到LinkedMultiValueMap
            LinkedMultiValueMap result = new LinkedMultiValueMap();

            while(urls.hasMoreElements()) {
                URL url = (URL)urls.nextElement();
                UrlResource resource = new UrlResource(url);
                Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                Iterator var6 = properties.entrySet().iterator();

                while(var6.hasNext()) {
                    Entry<?, ?> entry = (Entry)var6.next();
                    String factoryTypeName = ((String)entry.getKey()).trim();
                    String[] var9 = StringUtils.commaDelimitedListToStringArray((String)entry.getValue());
                    int var10 = var9.length;

                    for(int var11 = 0; var11 < var10; ++var11) {
                        String factoryImplementationName = var9[var11];
                        result.add(factoryTypeName, factoryImplementationName.trim());
                    }
                }
            }
			// LinkedMultiValueMap写入缓存
            cache.put(classLoader, result);
            return result;
        } catch (IOException var13) {
            throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var13);
        }
    }
}
```



```java
// 2.3依次实例化配置类
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
		ClassLoader classLoader, Object[] args, Set<String> names) {
	List<T> instances = new ArrayList<>(names.size());
	for (String name : names) {
		try {
            // 利用反射获取name参数对应的类对象
			Class<?> instanceClass = ClassUtils.forName(name, classLoader);
			Assert.isAssignable(type, instanceClass);
            // 获取默认构造函数，parameterTypes为空值
			Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            // 实例化
			T instance = (T) BeanUtils.instantiateClass(constructor, args);
			instances.add(instance);
		}
		catch (Throwable ex) {
			throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
		}
	}
	return instances;
}
```

#### 1.3 实例化ApplicationListener配置类

过程与1.2相同

```java
getSpringFactoriesInstances() {
    loadFactoryNames() 获取限定类名;
    createSpringFactoriesInstances() 实例化对象;
}
```

#### 1.4 设置当前main方法所在类

```java
// 4.推导出当前应用启动的main方法所在的类
private Class<?> deduceMainApplicationClass() {
	try {
        // 新建一个RuntimeException，在堆栈中找到main函数的入口
		StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
		for (StackTraceElement stackTraceElement : stackTrace) {
			if ("main".equals(stackTraceElement.getMethodName())) {
				return Class.forName(stackTraceElement.getClassName());
			}
		}
	}
	catch (ClassNotFoundException ex) {
		// Swallow and continue
	}
	return null;
}
```

### 2. run()

```java
public ConfigurableApplicationContext run(String... args) {
    // 1.开启一个计时器
	StopWatch stopWatch = new StopWatch();
	stopWatch.start();
    // spring应用上下文，也就是我们所说的spring根容器
	ConfigurableApplicationContext context = null;
    // 自定义SpringApplication启动错误的回调接口
	Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    // 2.设置java.awt.headless模式，默认为true，Headless模式是系统的一种配置模式
    // 在系统可能缺少显示设备、键盘或鼠标这些外设的情况下可以使用该模式
	configureHeadlessProperty();
    // 3.获取SpringApplicationRunListeners类实例
	SpringApplicationRunListeners listeners = getRunListeners(args);
    // 4.启动SpringApplicationRunListeners，触发启动事件，调用相应的监听器
	listeners.starting();
	try {
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 5.准备运行时环境
		ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        // 6.获取系统属性spring.beaninfo.ignore
		configureIgnoreBeanInfo(environment);
        // 7.打印banner
		Banner printedBanner = printBanner(environment);
        // 8.根据当前环境创建ApplicationContext
		context = createApplicationContext();
        // 9.加载SpringBootExceptionReporter
		exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
				new Class[] { ConfigurableApplicationContext.class }, context);
        // 10.准备上下文prepareContext
		prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 11.容器初始化refreshContext
		refreshContext(context);
        // 12.afterRefresh() 空方法，spring留的扩展点
		afterRefresh(context, applicationArguments);
        // 13.停止计时器
		stopWatch.stop();
		if (this.logStartupInfo) {
			new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
		}
        // 14.发送ApplicationStartedEvent事件
		listeners.started(context);
        // 15.调用系统中ApplicationRunner以及CommandLineRunner接口的实现类
		callRunners(context, applicationArguments);
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, listeners);
		throw new IllegalStateException(ex);
	}
	try {
        // 16.发送ApplicationReadyEvent事件
		listeners.running(context);
	}
	catch (Throwable ex) {
		handleRunFailure(context, ex, exceptionReporters, null);
		throw new IllegalStateException(ex);
	}
	return context;
}
```

#### 2.3 获取SpringApplicationRunListeners类实例

```java
// 3.获取SpringApplicationRunListeners类实例
private SpringApplicationRunListeners getRunListeners(String[] args) {
	Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    // 同样是通过getSpringFactoriesInstances方法获取SpringApplicationRunListener类实例
	return new SpringApplicationRunListeners(logger,
			getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}

// SpringApplicationRunListeners是SpringApplicationRunListener的集合，
// SpringApplicationRunListener的实现类EventPublishingRunListener，其构造函数
public EventPublishingRunListener(SpringApplication application, String[] args) {
	this.application = application;
	this.args = args;
    // 创建了一个SimpleApplicationEventMulticaster实例initialMulticaster
	this.initialMulticaster = new SimpleApplicationEventMulticaster();
    // 并将在new SpringAllication()中设置的ApplicationListener逐个添加到initialMulticaster中
    // 实际是添加到initialMulticaster的defaultRetriever.applicationListeners中
	for (ApplicationListener<?> listener : application.getListeners()) {
		this.initialMulticaster.addApplicationListener(listener);
	}
}

// 将在new SpringApplication()中设置的ApplicationListener逐个添加到initialMulticaster中
// 实际是添加到initialMulticaster的defaultRetriever.applicationListeners中
@Override
public void addApplicationListener(ApplicationListener<?> listener) {
	synchronized (this.retrievalMutex) {
		// Explicitly remove target for a proxy, if registered already,
		// in order to avoid double invocations of the same listener.
		Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
		if (singletonTarget instanceof ApplicationListener) {
			this.defaultRetriever.applicationListeners.remove(singletonTarget);
		}
		this.defaultRetriever.applicationListeners.add(listener);
		this.retrieverCache.clear();
	}
}
```

#### 2.4 启动SpringApplicationRunListeners

**Listener与Event实现的观察者模式**

```java
// 4.启动SpringApplicationRunListeners
void starting() {
	for (SpringApplicationRunListener listener : this.listeners) {
		listener.starting();
	}
}

@Override
public void starting() {
    // 广播事件
	this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
}

// 广播事件
public void multicastEvent(ApplicationEvent event) {
    this.multicastEvent(event, this.resolveDefaultEventType(event));
}

public void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = eventType != null ? eventType : this.resolveDefaultEventType(event);
    Executor executor = this.getTaskExecutor();
    // 4.1获取ApplicationListener
    Iterator var5 = this.getApplicationListeners(event, type).iterator();
    while(var5.hasNext()) {
        ApplicationListener<?> listener = (ApplicationListener)var5.next();
        if (executor != null) {
            executor.execute(() -> {
                // 4.2 依次调用ApplicationListener
                this.invokeListener(listener, event);
            });
        } else {
            this.invokeListener(listener, event);
        }
    }
}

```



```java
// 4.1获取ApplicationListener
protected Collection<ApplicationListener<?>> getApplicationListeners(
		ApplicationEvent event, ResolvableType eventType) {
	Object source = event.getSource();
	Class<?> sourceType = (source != null ? source.getClass() : null);
	ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);
    // 先从retrieverCache中找，找到则直接返回
	ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
	if (retriever != null) {
		return retriever.getApplicationListeners();
	}
    // retrieverCache中找不到
	if (this.beanClassLoader == null ||
			(ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
					(sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
		// Fully synchronized building and caching of a ListenerRetriever
        // 锁住当前实例对象，再从retrieverCache缓存中查一次，有点像单例模式的写法
		synchronized (this.retrievalMutex) {
			retriever = this.retrieverCache.get(cacheKey);
			if (retriever != null) {
				return retriever.getApplicationListeners();
			}
			retriever = new ListenerRetriever(true);
            // retrieverCache中还是没有，从defaultRetriever.applicationListeners中取
            // defaultRetriever.applicationListeners在上一步中设置过了
            // 拿到listeners列表后还调用supportsEvent()进行筛选，supportsEvent()暂时不往下深究
			Collection<ApplicationListener<?>> listeners =
					retrieveApplicationListeners(eventType, sourceType, retriever);
            // 设置retrieverCache
			this.retrieverCache.put(cacheKey, retriever);
            // 返回ApplicationListener
			return listeners;
		}
	}
	else {
		// No ListenerRetriever caching -> no synchronization necessary
		return retrieveApplicationListeners(eventType, sourceType, null);
	}
```



```java
// 4.2 依次调用ApplicationListener
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
	ErrorHandler errorHandler = getErrorHandler();
	if (errorHandler != null) {
		try {
			doInvokeListener(listener, event);
		}
		catch (Throwable err) {
			errorHandler.handleError(err);
		}
	}
	else {
		doInvokeListener(listener, event);
	}
}

@SuppressWarnings({"rawtypes", "unchecked"})
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
	try {
        // 主动调用监听器收到广播后的方法
		listener.onApplicationEvent(event);
	}
	catch (ClassCastException ex) {
		String msg = ex.getMessage();
		if (msg == null || matchesClassCastMessage(msg, event.getClass())) {
			// Possibly a lambda-defined listener which we could not resolve the generic event type for
			// -> let's suppress the exception and just log a debug message.
			Log logger = LogFactory.getLog(getClass());
			if (logger.isTraceEnabled()) {
				logger.trace("Non-matching event type for listener: " + listener, ex);
			}
		}
		else {
			throw ex;
		}
	}
```

getApplicationListeners方法过滤出的监听器都会被调用，过滤出来的监听器包括LoggingApplicationListener、BackgroundPreinitializer、DelegatingApplicationListener、LiquibaseServiceLocatorApplicationListener、EnableEncryptablePropertiesBeanFactoryPostProcessor五种类型的对象。这五个对象的onApplicationEvent都会被调用。

那么这五个监听器的onApplicationEvent都做了些什么了，我这里大概说下，细节的话大家自行去跟源码

`LoggingApplicationListener`：检测正在使用的日志系统，默认是logback，支持3种，优先级从高到低：logback > log4j > javalog。此时日志系统还没有初始化

`BackgroundPreinitializer`：另起一个线程实例化Initializer并调用其run方法，包括验证器、消息转换器等等

`DelegatingApplicationListener`：此时什么也没做

`LiquibaseServiceLocatorApplicationListener`：此时什么也没做

`EnableEncryptablePropertiesBeanFactoryPostProcessor`：此时仅仅打印了一句日志，其他什么也没做

#### 2.5 prepareEnvironment准备运行时环境

```java
// 5.准备运行时环境
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
		ApplicationArguments applicationArguments) {
	// 5.1 获取或创建环境
	ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 5.2 配置环境
	configureEnvironment(environment, applicationArguments.getSourceArgs());
    // 5.3 增加ConfigurationPropertySources属性源，这个方法没看懂
	ConfigurationPropertySources.attach(environment);
    // 5.4 告知监听器，环境准备好了
	listeners.environmentPrepared(environment);
    // 5.5 将environment里的spring.main开头的配置信息绑定到SpringApplication的source中去
	bindToSpringApplication(environment);
    // 5.6 如果不是自定义的环境，将环境转换为与application能够匹配的类型
	if (!this.isCustomEnvironment) {
		environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
				deduceEnvironmentClass());
	}
    // 5.7 增加ConfigurationPropertySources属性源
	ConfigurationPropertySources.attach(environment);
	return environment;
}
```

##### 2.5.1 获取或创建环境

```java
// 5.1获取或创建环境，因为是servlet环境，所以这里返回的是StandardServletEnvironment对象
private ConfigurableEnvironment getOrCreateEnvironment() {
	if (this.environment != null) {
		return this.environment;
	}
	switch (this.webApplicationType) {
	case SERVLET:
        // 5.1.1 new StandardServletEnvironment()
		return new StandardServletEnvironment();
	case REACTIVE:
		return new StandardReactiveWebEnvironment();
	default:
		return new StandardEnvironment();
	}
}

// 5.1.1 new StandardServletEnvironment()
// StandardServletEnvironment在初始化的时候会调用父类的父类的customizePropertySources方法，
// StandardServletEnvironment对该方法进行了重载
// 调用该方法在propertySources中添加了几个配置
protected void customizePropertySources(MutablePropertySources propertySources) {
    // StubPropertySource占位配置源，没有具体引用，只用来占位，后面初始化再替换
    propertySources.addLast(new StubPropertySource("servletConfigInitParams"));
    propertySources.addLast(new StubPropertySource("servletContextInitParams"));
    // /如果jndi配置可用，添加一个名称为jndiProperties的空配置
    if (JndiLocatorDelegate.isDefaultJndiEnvironmentAvailable()) {
        propertySources.addLast(new JndiPropertySource("jndiProperties"));
    }
    // 父类customizePropertySources()
    super.customizePropertySources(propertySources);
}

// 父类customizePropertySources()
protected void customizePropertySources(MutablePropertySources propertySources) {
    // java相关属性与配置
    propertySources.addLast(new PropertiesPropertySource("systemProperties", this.getSystemProperties()));
    // 操作系统环境变量配置
    propertySources.addLast(new SystemEnvironmentPropertySource("systemEnvironment", this.getSystemEnvironment()));
}
```

##### 2.5.2 配置环境

Environment主要是负责解析properties和profile

```java
// 5.2 配置环境
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
	// 用于类型转换，从environment中getProperty获取属性时会用到，默认为true
    if (this.addConversionService) {
        // 获取ApplicationConversionService对象，这里用了双重校验锁的单例模式
		ConversionService conversionService = ApplicationConversionService.getSharedInstance();
        // 设置转换类
		environment.setConversionService((ConfigurableConversionService) conversionService);
	}
    // 5.2.1 配置PropertySources
	configurePropertySources(environment, args);
    // 5.2.2 配置Profiles
	configureProfiles(environment, args);
}

// 5.2.1 配置PropertySources
protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
	MutablePropertySources sources = environment.getPropertySources();
    // 如果存在defaultProperties默认配置，将其放到PropertySources最后
	if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
		sources.addLast(new MapPropertySource("defaultProperties", this.defaultProperties));
	}
    // 如果命令行参数不为空，则设置命令行参数，并将旧的命令行参数替换掉
	if (this.addCommandLineProperties && args.length > 0) {
		String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
        // 如果配置中已经有命令行参数，则替换
		if (sources.contains(name)) {
			PropertySource<?> source = sources.get(name);
			CompositePropertySource composite = new CompositePropertySource(name);
			composite.addPropertySource(
					new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
			composite.addPropertySource(source);
			sources.replace(name, composite);
		}
        // 如果没有，将其放在最前面
		else {
			sources.addFirst(new SimpleCommandLinePropertySource(args));
		}
	}
}

// 5.2.2 配置Profiles，这个方法暂时不细看，debug下来有点看不明白
protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
	Set<String> profiles = new LinkedHashSet<>(this.additionalProfiles);
    // 读取spring.profiles.active参数
	profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
    // 将spring.profiles.active参数写入environment
	environment.setActiveProfiles(StringUtils.toStringArray(profiles));
}

```

上面配置PropertySources体现了命令行参数的优先级最高，优先级高的会覆盖优先级低的

SpringBoot加载配置的优先级：

- 命令行参数
- Servlet初始参数
- ServletContext初始化参数
- JVM系统属性
- 操作系统环境变量
- 随机生成的带random.*前缀的属性
- 应用程序以外的application.yml或者appliaction.properties文件
- classpath:/config/application.yml或者classpath:/config/application.properties
- 通过@PropertySource标注的属性源
- 默认属性

##### 2.5.4 告知监听器，环境准备好了

```java
// 5.4 告知监听器，环境准备好了
void environmentPrepared(ConfigurableEnvironment environment) {
	for (SpringApplicationRunListener listener : this.listeners) {
		listener.environmentPrepared(environment);
	}
}

// 广播一个ApplicationEnvironmentPreparedEvent事件，广播事件的代码前面介绍过
@Override
public void environmentPrepared(ConfigurableEnvironment environment) {
	this.initialMulticaster
			.multicastEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args, environment));
}
```

这里广播时响应的listener有8个：

`RestartApplicationListener`  什么也没干
`ConfigFileApplicationListener` 加载配置文件
`AnsiOutputApplicationListener ` 字符输出监听器, 用于调整控制台显示的打印字符的各种颜色 
`LoggingApplicationListener` 日志监听器, 初始化日志配置
`ClasspathLoggingApplicationListener` 打印debug日志, 记录当前classpath
`BackgroundPreinitializer `扩展点, 目前只关注Starting, Ready和Failed事件, EnvironmentPrepared事件不做处理
`DelegatingApplicationListener `扩展点, 将当前的EnvironmentPrepared事件, 广播给其他关注该事件的Environment监听器, 目前没有做任何操作 

`FileEncodingApplicationListener` 如果指定了spring.mandatory-file-encoding属性, 系统属性file.encoding和spring.mandatory-file-encoding不一致的话, 抛出异常

```java
// ConfigFileApplicationListener

// 获取EnvironmentPostProcessor的实现类，按order排序后依次执行其postProcessEnvironment()
private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
    // 加载EnvironmentPostProcessor实现类，这里调用了SpringFactoriesLoader.loadFactories方法，该方法与上文中的getSpringFactoriesInstances类似，都是先获取classLoader，然后调用loadFactoryName，再实例化
	List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
    // postProcessors列表中把自己也加上，不知道啥意思
	postProcessors.add(this);
    // 排序
	AnnotationAwareOrderComparator.sort(postProcessors);
    // 依次调用postProcessEnvironment方法
	for (EnvironmentPostProcessor postProcessor : postProcessors) {
		postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
	}
}

// 这里的EnvironmentPostProcessor实现类有：
SystemEnvironmentPropertySourceEnvironmentPostProcessor
SpringApplicationJsonEnvironmentPostProcessor
CloudFoundryVcapEnvironmentPostProcessor
DevToolsHomePropertiesPostProcessor
DevToolsPropertyDefaultsPostProcessor
DebugAgentEnvironmentPostProcessor
ConfigFileApplicationListener

// ConfigFileApplicationListener的postProcessEnvironment方法
@Override
public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
	addPropertySources(environment, application.getResourceLoader());
}

protected void addPropertySources(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
	// 添加一个RandomValuePropertySource属性源，RandomValuePropertySource可以通过environment.getProperty("random.*")返回各种随机值
	RandomValuePropertySource.addToEnvironment(environment);
	// 加载配置文件
	(new ConfigFileApplicationListener.Loader(environment, resourceLoader)).load();
}

// 添加一个RandomValuePropertySource属性源
public static void addToEnvironment(ConfigurableEnvironment environment) {
environment.getPropertySources().addAfter(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME,
			new RandomValuePropertySource(RANDOM_PROPERTY_SOURCE_NAME));
	logger.trace("RandomValuePropertySource add to Environment");
}


// 加载配置文件
// 初始化loader
Loader(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
	this.environment = environment;
	this.placeholdersResolver = new PropertySourcesPlaceholdersResolver(this.environment);
	this.resourceLoader = (resourceLoader != null) ? resourceLoader : new DefaultResourceLoader(null);
	// 获取PropertySourceLoader类实例，分别是：
	// PropertiesPropertySourceLoader 读取.properties 和.xml文件
	// YamlPropertySourceLoader 读取.yml 和.yaml文件
	this.propertySourceLoaders = SpringFactoriesLoader.loadFactories(PropertySourceLoader.class,
			getClass().getClassLoader());
}

// 加载配置文件
void load() {
	FilteredPropertySource.apply(this.environment, DEFAULT_PROPERTIES, LOAD_FILTERED_PROPERTY,
			(defaultProperties) -> {
				this.profiles = new LinkedList<>();
				this.processedProfiles = new LinkedList<>();
				this.activatedProfiles = false;
				this.loaded = new LinkedHashMap<>();
				// 获取profile属性，结束后this.profiles结果为{null, default}
				initializeProfiles();
				// 遍历profile
                // 加载文件，default中读到指定了active的配置，就会将其放入profiles，并移除default
				while (!this.profiles.isEmpty()) {
					Profile profile = this.profiles.poll();
					if (isDefaultProfile(profile)) {
						addProfileToEnvironment(profile.getName());
					}
					load(profile, this::getPositiveProfileFilter,
							addToLoaded(MutablePropertySources::addLast, false));
					this.processedProfiles.add(profile);
				}
				//获取配置, 加入到this.loaded中
				load(null, this::getNegativeProfileFilter, addToLoaded(MutablePropertySources::addFirst, true));
				//将this.loaded按顺序添加到environment的propertySources中
            	//如果存在defaultProperties,放在defaultProperties之前
            	//如果不存在defaultProperties,直接添加到最后
				addLoadedPropertySources();
				// 添加defaultProperties
				applyActiveProfiles(defaultProperties);
			});
}
```

*// TODO* 加载配置文件这里有点复杂，后面再看吧，参考资料： 

https://www.jianshu.com/p/e26d377cd96f?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation

https://mingyang.blog.csdn.net/article/details/109259884

https://blog.csdn.net/wangwei19871103/article/details/105723450

#### 2.6 获取系统属性spring.beaninfo.ignore

```java
private void configureIgnoreBeanInfo(ConfigurableEnvironment environment) {
	if (System.getProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME) == null) {
        // 返回true，不搜索beaninfo类
		Boolean ignore = environment.getProperty("spring.beaninfo.ignore", Boolean.class, Boolean.TRUE);
		System.setProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME, ignore.toString());
	}
}
```

#### 2.7 打印banner

```java
private Banner printBanner(ConfigurableEnvironment environment) {
    // 判断banner模式，OFF直接返回
	if (this.bannerMode == Banner.Mode.OFF) {
		return null;
	}
    // 获取resourceLoader
	ResourceLoader resourceLoader = (this.resourceLoader != null) ? this.resourceLoader
			: new DefaultResourceLoader(null);
    // 实例化SpringApplicationBannerPrinter
	SpringApplicationBannerPrinter bannerPrinter = new SpringApplicationBannerPrinter(resourceLoader, this.banner);
    // 是否输出到日志文件
	if (this.bannerMode == Mode.LOG) {
		return bannerPrinter.print(environment, this.mainApplicationClass, logger);
	}
    // 打印到控制台
	return bannerPrinter.print(environment, this.mainApplicationClass, System.out);
}
```

Banner.Mode有三种

- `OFF` 不打印
- `CONSOLE` 打印到控制台
- `LOG` 打印到日志文件

#### 2.8 根据当前环境创建ApplicationContext

```java
protected ConfigurableApplicationContext createApplicationContext() {
	Class<?> contextClass = this.applicationContextClass;
    // 根据当前环境创建ApplicationContext
	if (contextClass == null) {
		try {
			switch (this.webApplicationType) {
            // 返回AnnotationConfigServletWebServerApplicationContext对象，利用了反射
			case SERVLET:
				contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
				break;
			case REACTIVE:
				contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
				break;
			default:
				contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
			}
		}
		catch (ClassNotFoundException ex) {
			throw new IllegalStateException(
					"Unable create a default ApplicationContext, please specify an ApplicationContextClass", ex);
		}
	}
	return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}


// servlet
public static final String DEFAULT_SERVLET_WEB_CONTEXT_CLASS = "org.springframework.boot."
			+ "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";

// reactive
public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework."
			+ "boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";

// none
public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
```

#### 2.9 加载SpringBootExceptionReporter

```java
exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);

// 跟上面一样，使用getSpringFactoriesInstances获取实例
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
	ClassLoader classLoader = getClassLoader();
	// Use names and ensure unique to protect against duplicates
	Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
	List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
	AnnotationAwareOrderComparator.sort(instances);
	return instances;
}
```

#### 2.10 prepareContext

```java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
		SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    // 10.1 设置上下文的environment
	context.setEnvironment(environment);
    // 10.2 上下文后置处理
	postProcessApplicationContext(context);
    // 10.3 在context refresh之前，对其应用ApplicationContextInitializer
	applyInitializers(context);
    // 10.4 发送上下文准备事件，实际啥也没干
	listeners.contextPrepared(context);
    // 10.5 打印启动日志和启动应用的Profile
	if (this.logStartupInfo) {
		logStartupInfo(context.getParent() == null);
		logStartupProfileInfo(context);
	}
	// 10.6 向beanFactory注册单例bean：命令行参数bean
	ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
	beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    // 10.7 向beanFactory注册单例bean：printedBanner
	if (printedBanner != null) {
		beanFactory.registerSingleton("springBootBanner", printedBanner);
	}
    // 10.8 允许具有相同名称的Bean来覆盖之前的Bean
	if (beanFactory instanceof DefaultListableBeanFactory) {
		((DefaultListableBeanFactory) beanFactory)
				.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
	}
    // 10.9 若设置了延迟初始化，设置后置处理器
	if (this.lazyInitialization) {
		context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
	}
	// 10.10 获取全部资源，其实就一个：SpringApplication的primarySources属性
	Set<Object> sources = getAllSources();
	Assert.notEmpty(sources, "Sources must not be empty");
    // 10.11 加载主类，将其注册到beanFactory的BeanDefinitionMap中
	load(context, sources.toArray(new Object[0]));
    // 10.12 向上下文中添加ApplicationListener，并广播ApplicationPreparedEvent事件
	listeners.contextLoaded(context);
}
```

```java
// 10.1 设置上下文的environment
@Override
public void setEnvironment(ConfigurableEnvironment environment) {
    // 将context中相关的environment全部替换成SpringApplication中创建的environment
	super.setEnvironment(environment);
	this.reader.setEnvironment(environment);
	this.scanner.setEnvironment(environment);
}
```

```java
// 10.2 上下文后置处理
protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
	// false
    if (this.beanNameGenerator != null) {
		context.getBeanFactory().registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
				this.beanNameGenerator);
	}
    // false
	if (this.resourceLoader != null) {
		if (context instanceof GenericApplicationContext) {
			((GenericApplicationContext) context).setResourceLoader(this.resourceLoader);
		}
		if (context instanceof DefaultResourceLoader) {
			((DefaultResourceLoader) context).setClassLoader(this.resourceLoader.getClassLoader());
		}
	}
    // 设置转换服务
	if (this.addConversionService) {
		context.getBeanFactory().setConversionService(ApplicationConversionService.getSharedInstance());
	}
}
```

```java
// 10.3 在context refresh之前，对其应用ApplicationContextInitializer
protected void applyInitializers(ConfigurableApplicationContext context) {
    // 循环处理
	for (ApplicationContextInitializer initializer : getInitializers()) {
        // 获取initializer的泛型参数具体是什么类型
		Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(),
				ApplicationContextInitializer.class);
		Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        // 向context应用初始化器
		initializer.initialize(context);
	}
}
```

总共8个initializer，依次执行其initialize方法

`DelegatingApplicationContextInitializer` 不做任何事

`SharedMetadataReaderFactoryContextInitializer` 向context注册了一个BeanFactoryPostProcessor: CachingMetadataReaderFactoryPostProcessor实例。

`ContextIdApplicationContextInitializer` 从environment中获取spring.application.name配置项的值，并把设置成application id，若没有配置spring.application.name，则取默认值"application"，随后将application id封装成ContextId对象，注册到beanFactory中。

`RestartScopeInitializer` 向beanFactory注册了一个restart的作用域scope

`ConfigurationWarningsApplicationContextInitializer` 向上下文注册了一个BeanFactoryPostProcessor: ConfigurationWarningsPostProcessor实例，实例化ConfigurationWarningsPostProcessor的时候，也实例化了它的属性Check[] checks，check中只有一个类型是ComponentScanPackageCheck的实例。

`RSocketPortInfoApplicationContextInitializer`向上下文注册了一个内部实现的listener，用于获取RSocker服务端口。

`ServerPortInfoApplicationContextInitializer` 向上下文注册了一个ApplicationListener: ServerPortInfoApplicationContextInitializer对象自己；ServerPortInfoApplicationContextInitializer实现了ApplicationListener\<WebServerInitializedEvent>，所以他本身就是一个ApplicationListener。

`ConditionEvaluationReportLoggingListener` 将上下文赋值给自己的属性applicationContext；向上下文注册了一个ApplicationListener: ConditionEvaluationReportListener实例；从beanFactory中获取名为autoConfigurationReport的bean赋值给自己的属性report。

```java
// 10.12 向上下文中添加ApplicationListener，并广播ApplicationPreparedEvent事件
@Override
public void contextLoaded(ConfigurableApplicationContext context) {
	for (ApplicationListener<?> listener : this.application.getListeners()) {
		if (listener instanceof ApplicationContextAware) {
			((ApplicationContextAware) listener).setApplicationContext(context);
		}
        // 添加listener
		context.addApplicationListener(listener);
	}
    // 广播ApplicationPreparedEvent事件
	this.initialMulticaster.multicastEvent(new ApplicationPreparedEvent(this.application, this.args, context));
}
```

RestartApplicationListener 配置了一个ResourceLoader
CloudFoundryVcapEnvironmentPostProcessor 切换内部日志打印模式
ConfigFileApplicationListener 向context注册了一个BeanFactoryPostProcessor：PropertySourceOrderingPostProcessor实例；该实例后面会对我们的property sources进行重排序，另外该实例拥有上下文的引用。
LoggingApplicationListener 向beanFactory中注册了一个名叫springBootLoggingSystem的单例bean，也就是我们的日志系统bean。
BackgroundPreinitializer 什么也没做
DelegatingApplicationListener 什么也没做
DevToolsLogFactory$Listener 仅仅打印了一句debug日志，相当于什么也没做



#### 2.11 容器初始化refreshContext（重点）

参考资料：https://mp.weixin.qq.com/s?__biz=MzU5MDgzOTYzMw==&mid=2247484561&idx=1&sn=a7281dae7aaaa3499d59dec730464e63&scene=21#wechat_redirect

https://mp.weixin.qq.com/s?__biz=MzU5MDgzOTYzMw==&mid=2247484564&idx=1&sn=84bd8fee210c0d00687c3094431482a7&scene=21#wechat_redirect

```java
refreshContext(context);

// 最终调用AbstractApplicationContext的refresh方法
@Override
public void refresh() throws BeansException, IllegalStateException {
    // 上锁
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
        // 11.1 准备工作，记录容器的启动时间、标记“已启动”状态、检查环境变量等
		prepareRefresh();
		// Tell the subclass to refresh the internal bean factory.
        // 11.2 创建bean容器
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
		// Prepare the bean factory for use in this context.
        // 11.3 准备BeanFactory
		prepareBeanFactory(beanFactory);
		try {
			// Allows post-processing of the bean factory in context subclasses.
            // 扩展点，空方法
			postProcessBeanFactory(beanFactory);
			// Invoke factory processors registered as beans in the context.
         	// 11.4 调用BeanFactoryPostProcessor各个实现类的postProcessBeanFactory()方法
			invokeBeanFactoryPostProcessors(beanFactory);
			// Register bean processors that intercept bean creation.
            // 11.5 注册 BeanPostProcessor 的实现类
			registerBeanPostProcessors(beanFactory);
			// Initialize message source for this context.
            // 11.6 初始化MessageSource，与国际化相关
			initMessageSource();
			// Initialize event multicaster for this context.
            // 11.7 初始化事件广播器
			initApplicationEventMulticaster();
			// Initialize other special beans in specific context subclasses.
            // 扩展点，空方法
			onRefresh();
			// Check for listener beans and register them.
            // 11.8 注册事件监听器
			registerListeners();
			// Instantiate all remaining (non-lazy-init) singletons.
            // 11.9 初始化所有的 singleton beans
			finishBeanFactoryInitialization(beanFactory);
			// Last step: publish corresponding event.
            // 11.10 结束初始化
			finishRefresh();
		}
		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}
			// Destroy already created singletons to avoid dangling resources.
            // 销毁已经初始化的的Bean
			destroyBeans();
			// Reset 'active' flag.
            // 设置 'active' 状态
			cancelRefresh(ex);
			// Propagate exception to caller.
			throw ex;
		}
		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
            // 11.11清除缓存
			resetCommonCaches();
		}
	}
}
```

##### 2.11.1 准备工作

```java
// 11.1 准备工作，记录容器的启动时间、标记“已启动”状态、检查环境变量等
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
	// Initialize any placeholder property sources in the context environment.
	initPropertySources();
	// Validate that all properties marked as required are resolvable:
	// see ConfigurablePropertyResolver#setRequiredProperties
    // 检查环境变量，如果出现value为空的环境变量，直接抛异常结束启动流程
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
```

##### 2.11.2 创建bean容器

```java
// 11.2 创建bean容器
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 创建bean容器
	refreshBeanFactory();
    // 返回bean容器
	return getBeanFactory();
}
```

**BeanFactory**

在Spring中有许多的IOC容器的实现供用户选择和使用，其中BeanFactory作为最顶层的一个接口类，它定义了IOC容器的基本功能规范，**BeanFactory**有三个子类：ListableBeanFactory、HierarchicalBeanFactory和；AutowireCapableBeanFactory。但是最终的默认实现类是**DefaultListableBeanFactory**。

**BeanDefinition**

BeanFactory是一个Bean容器，而BeanDefinition就是Bean的一种形式（它里面包含了Bean指向的类、是否单例、是否懒加载、Bean的依赖关系等相关的属性）。BeanFactory中就是保存的BeanDefinition。

```java
// BeanDefinition的定义
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
   // Bean的生命周期，默认只提供sington和prototype两种，在WebApplicationContext中还会有request, session, globalSession, application, websocket 等
   String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
   String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;
 
   // 设置父Bean
   void setParentName(String parentName);

   // 获取父Bean
   String getParentName();

   // 设置Bean的类名称
   void setBeanClassName(String beanClassName);

   // 获取Bean的类名称
   String getBeanClassName();


   // 设置bean的scope
   void setScope(String scope);

   String getScope();

   // 设置是否懒加载
   void setLazyInit(boolean lazyInit);

   boolean isLazyInit();

   // 设置该Bean依赖的所有Bean
   void setDependsOn(String... dependsOn);

   // 返回该Bean的所有依赖
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
   void setFactoryBeanName(String factoryBeanName);
   // 获取工厂名称
   String getFactoryBeanName();
   // 指定工厂类中的工厂方法名称
   void setFactoryMethodName(String factoryMethodName);
   // 获取工厂类中的工厂方法名称
   String getFactoryMethodName();

   // 构造器参数
   ConstructorArgumentValues getConstructorArgumentValues();

   // Bean 中的属性值，后面给 bean 注入属性值的时候会说到
   MutablePropertyValues getPropertyValues();

   // 是否 singleton
   boolean isSingleton();

   // 是否 prototype
   boolean isPrototype();

   // 如果这个 Bean 是被设置为 abstract，那么不能实例化，常用于作为 父bean 用于继承
   boolean isAbstract();

   int getRole();
   String getDescription();
   String getResourceDescription();
   BeanDefinition getOriginatingBeanDefinition();
}
```

###### 基于XML的容器创建

```java
// 基于XML的容器创建
@Override
protected final void refreshBeanFactory() throws BeansException {
    // 判断当前ApplicationContext是否存在BeanFactory，如果存在的话就销毁所有Bean，关闭BeanFactory
    // 一个应用可以存在多个BeanFactory，这里判断的是当前ApplicationContext是否存在BeanFactory
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
        // 1.初始化DefaultListableBeanFactory
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
        // 2.设置BeanFactory的两个配置属性：是否允许Bean覆盖、是否允许循环引用
		customizeBeanFactory(beanFactory);
        // 3.加载Bean到BeanFactory中
		loadBeanDefinitions(beanFactory);
		this.beanFactory = beanFactory;
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}

// 3.加载Bean到BeanFactory中
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	// Create a new XmlBeanDefinitionReader for the given BeanFactory.
    // 实例化XmlBeanDefinitionReader
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
	// Configure the bean definition reader with this context's
	// resource loading environment.
	beanDefinitionReader.setEnvironment(this.getEnvironment());
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
	// Allow a subclass to provide custom initialization of the reader,
	// then proceed with actually loading the bean definitions.
    // 初始化BeanDefinitionReader
	initBeanDefinitionReader(beanDefinitionReader);
    // 加载BeanDefinition
	loadBeanDefinitions(beanDefinitionReader);
}

// AbstractBeanDefinitionReader 加载BeanDefinition
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    // 获取系统指定的bean配置文件位置，一般为空
	Resource[] configResources = getConfigResources();
	if (configResources != null) {
		reader.loadBeanDefinitions(configResources);
	}
    // 获取自定义的bean配置文件位置
	String[] configLocations = getConfigLocations();
	if (configLocations != null) {
        // ↓ 加载bean，里面用到了委托模式
		reader.loadBeanDefinitions(configLocations);
	}
}

// ↓ 加载bean，里面用到了委托模式
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
    Assert.notNull(locations, "Location array must not be null");
    int count = 0;
    String[] var3 = locations;
    int var4 = locations.length;
    for(int var5 = 0; var5 < var4; ++var5) {
        String location = var3[var5];
        // 依次加载bean配置文件
        count += this.loadBeanDefinitions(location);
    }
    return count;
}

public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
    return this.loadBeanDefinitions(location, (Set)null);
}

public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
    ResourceLoader resourceLoader = this.getResourceLoader();
    if (resourceLoader == null) {
        throw new BeanDefinitionStoreException("Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
    } else {
        int count;
        // ResourcePatternResolver支持路径中带有通配符
        if (resourceLoader instanceof ResourcePatternResolver) {
            try {
                // location转化为Resource对象
                Resource[] resources = ((ResourcePatternResolver)resourceLoader).getResources(location);
                // ↓ 加载BeanDefinition
                count = this.loadBeanDefinitions(resources);
                if (actualResources != null) {
                    Collections.addAll(actualResources, resources);
                }

                if (this.logger.isTraceEnabled()) {
                    this.logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
                }

                return count;
            } catch (IOException var6) {
                throw new BeanDefinitionStoreException("Could not resolve bean definition resource pattern [" + location + "]", var6);
            }
        } else {
            // 用于加载绝对路径的配置文件
            Resource resource = resourceLoader.getResource(location);
            count = this.loadBeanDefinitions((Resource)resource);
            if (actualResources != null) {
                actualResources.add(resource);
            }

            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
            }

            return count;
        }
    }
}

// ↓ loadBeanDefinitions方法重载
public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
    Assert.notNull(resources, "Resource array must not be null");
    int count = 0;
    Resource[] var3 = resources;
    int var4 = resources.length;
    for(int var5 = 0; var5 < var4; ++var5) {
        Resource resource = var3[var5];
        // ↓ 调用XmlBeanDefinitionReader的loadBeanDefinitions实现
        count += this.loadBeanDefinitions((Resource)resource);
    }
    return count;
}

// ↓ XmlBeanDefinitionReader的loadBeanDefinitions实现
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    return this.loadBeanDefinitions(new EncodedResource(resource));
}

public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    Assert.notNull(encodedResource, "EncodedResource must not be null");
    if (this.logger.isTraceEnabled()) {
        this.logger.trace("Loading XML bean definitions from " + encodedResource);
    }

    Set<EncodedResource> currentResources = (Set)this.resourcesCurrentlyBeingLoaded.get();
    if (!currentResources.add(encodedResource)) {
        throw new BeanDefinitionStoreException("Detected cyclic loading of " + encodedResource + " - check your import definitions!");
    } else {
        int var6;
        try {
            // 获取文件流
            InputStream inputStream = encodedResource.getResource().getInputStream();
            Throwable var4 = null;

            try {
                InputSource inputSource = new InputSource(inputStream);
                if (encodedResource.getEncoding() != null) {
                    inputSource.setEncoding(encodedResource.getEncoding());
                }

                // ↓ 实际加载BeanDefinition的方法
                var6 = this.doLoadBeanDefinitions(inputSource, encodedResource.getResource());
            } catch (Throwable var24) {
                var4 = var24;
                throw var24;
            } finally {
                if (inputStream != null) {
                    if (var4 != null) {
                        try {
                            inputStream.close();
                        } catch (Throwable var23) {
                            var4.addSuppressed(var23);
                        }
                    } else {
                        inputStream.close();
                    }
                }

            }
        } catch (IOException var26) {
            throw new BeanDefinitionStoreException("IOException parsing XML document from " + encodedResource.getResource(), var26);
        } finally {
            currentResources.remove(encodedResource);
            if (currentResources.isEmpty()) {
                this.resourcesCurrentlyBeingLoaded.remove();
            }

        }

        return var6;
    }
}

// ↓ 实际加载BeanDefinition的方法
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource) throws BeanDefinitionStoreException {
    try {
        // 加载为Document对象
        Document doc = this.doLoadDocument(inputSource, resource);
        // ↓ 根据Document对象注册bean对象
        int count = this.registerBeanDefinitions(doc, resource);
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Loaded " + count + " bean definitions from " + resource);
        }

        return count;
    } catch (BeanDefinitionStoreException var5) {
        throw var5;
    } catch (SAXParseException var6) {
        throw new XmlBeanDefinitionStoreException(resource.getDescription(), "Line " + var6.getLineNumber() + " in XML document from " + resource + " is invalid", var6);
    } catch (SAXException var7) {
        throw new XmlBeanDefinitionStoreException(resource.getDescription(), "XML document from " + resource + " is invalid", var7);
    } catch (ParserConfigurationException var8) {
        throw new BeanDefinitionStoreException(resource.getDescription(), "Parser configuration exception parsing XML from " + resource, var8);
    } catch (IOException var9) {
        throw new BeanDefinitionStoreException(resource.getDescription(), "IOException parsing XML document from " + resource, var9);
    } catch (Throwable var10) {
        throw new BeanDefinitionStoreException(resource.getDescription(), "Unexpected exception parsing XML document from " + resource, var10);
    }
}

public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    // 构建读取Document的工具类
    BeanDefinitionDocumentReader documentReader = this.createBeanDefinitionDocumentReader();
    // 获取已注册的bean数量
    int countBefore = this.getRegistry().getBeanDefinitionCount();
    // ↓ 注册bean
    documentReader.registerBeanDefinitions(doc, this.createReaderContext(resource));
    // 总注册的bean减去之前注册的bean就是本次注册的bean
    return this.getRegistry().getBeanDefinitionCount() - countBefore;
}

// ↓  注册bean，注册即是将beanDefinition添加到BeanDefinitionMap中
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    // ↓ 注册bean
    this.doRegisterBeanDefinitions(doc.getDocumentElement());
}

// ↓ 注册bean
protected void doRegisterBeanDefinitions(Element root) {
    // 当前根节点
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = this.createDelegate(this.getReaderContext(), root, parent);
    if (this.delegate.isDefaultNamespace(root)) {
        // 获取 <beans ... profile="***" />中的profile参数与当前环境是否匹配，如果不匹配则不再解析
        String profileSpec = root.getAttribute("profile");
        if (StringUtils.hasText(profileSpec)) {
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(profileSpec, ",; ");
            if (!this.getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec + "] not matching: " + this.getReaderContext().getResource());
                }

                return;
            }
        }
    }

    // 前置扩展点，空方法
    this.preProcessXml(root);
    // ↓ 解析BeanDefinitions
    this.parseBeanDefinitions(root, this.delegate);
    // 后置扩展点，空方法
    this.postProcessXml(root);
    this.delegate = parent;
}

// ↓ 解析BeanDefinitions
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    // xml文件中的默认namespace包含"http://www.springframework.org/schema/beans"
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for(int i = 0; i < nl.getLength(); ++i) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element)node;
                if (delegate.isDefaultNamespace(ele)) {
                    // ↓ 解析默认namespace中的元素，包括import, alia, bean, beans
                    this.parseDefaultElement(ele, delegate);
                } else {
                    // 解析其他namespace中的元素
                    delegate.parseCustomElement(ele);
                }
            }
        }
    } else {
        // 解析其他namespace中的元素
        delegate.parseCustomElement(root);
    }
}

// ↓ 解析默认namespace中的元素，包括import, alia, bean, beans
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    if (delegate.nodeNameEquals(ele, "import")) {
        this.importBeanDefinitionResource(ele);
    } else if (delegate.nodeNameEquals(ele, "alias")) {
        this.processAliasRegistration(ele);
    } else if (delegate.nodeNameEquals(ele, "bean")) {
        // ↓ 处理bean标签
        this.processBeanDefinition(ele, delegate);
    } else if (delegate.nodeNameEquals(ele, "beans")) {
        this.doRegisterBeanDefinitions(ele);
    }
}

// ↓ 处理bean标签
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    //↓ 1.创建BeanDefinition
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);

        try {
            // ↓ 2.注册BeanDefinition
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, this.getReaderContext().getRegistry());
        } catch (BeanDefinitionStoreException var5) {
            this.getReaderContext().error("Failed to register bean definition with name '" + bdHolder.getBeanName() + "'", ele, var5);
        }

        // 发送已注册事件，空方法
        this.getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }

}
```

```java
//↓ 1.创建BeanDefinition
@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
	return parseBeanDefinitionElement(ele, null);
}

@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
	String id = ele.getAttribute(ID_ATTRIBUTE);
	String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

	List<String> aliases = new ArrayList<>();
    // 处理bean的name属性，别名
	if (StringUtils.hasLength(nameAttr)) {
		String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
		aliases.addAll(Arrays.asList(nameArr));
	}

	String beanName = id;
    // 如果没有指定bean的id, 那么用别名列表的第一个名字作为beanName
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

    // ↓ 根据<bean ...>...</bean>中的配置创建BeanDefinition，然后把配置中的信息都设置到实例中,
    // ↓ 这行执行完毕就会返回一个BeanDefinition实例
	AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
	if (beanDefinition != null) {
        // 如果没有设置id和name，那么此时的beanName就会为 null，按以下方式设置beanName
		if (!StringUtils.hasText(beanName)) {
			try {
				if (containingBean != null) {
					beanName = BeanDefinitionReaderUtils.generateBeanName(
							beanDefinition, this.readerContext.getRegistry(), true);
				}
				else {
					beanName = this.readerContext.generateBeanName(beanDefinition);
					// Register an alias for the plain bean class name, if still possible,
					// if the generator returned the class name plus a suffix.
					// This is expected for Spring 1.2/2.0 backwards compatibility.
					String beanClassName = beanDefinition.getBeanClassName();
					if (beanClassName != null &&
							beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
							!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
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
		return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
	}

	return null;
}

// ↓ 根据<bean ...>...</bean>中的配置创建BeanDefinition
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
        // 先创建实例
		AbstractBeanDefinition bd = createBeanDefinition(className, parent);
        // 设置属性
		parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
		bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
        // 解析<bean/>中的子元素，并将属性设置到db实例中
        // 解析<meta/>
		parseMetaElements(ele, bd);
        // 解析<lookup-method/>
		parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        // 解析<replaced-method/>
		parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
        // 解析<constructor-arg/>
		parseConstructorArgElements(ele, bd);
        // 解析<property/>
		parsePropertyElements(ele, bd);
        // 解析<qualifier/>
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

```java
// ↓ 2.注册BeanDefinition
public static void registerBeanDefinition(
		BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
		throws BeanDefinitionStoreException {
	// 注册bean
	String beanName = definitionHolder.getBeanName();
	registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
	// 如果bean设置了别名，也要根据别名全部注册一遍
	String[] aliases = definitionHolder.getAliases();
	if (aliases != null) {
		for (String alias : aliases) {
			registry.registerAlias(beanName, alias);
		}
	}
}

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
    // 所有的 Bean 注册后都会被放入到这个beanDefinitionMap中，查看是否已存在这个bean
	BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    // 处理Bean名称重复的情况
	if (existingDefinition != null) {
		if (!isAllowBeanDefinitionOverriding()) {
            // 如果不允许覆盖，抛异常
			throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
		}
        // 打印日志说明是用框架定义的Bean覆盖用户自定义的Bean 
		else if (existingDefinition.getRole() < beanDefinition.getRole()) {
			// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
			if (logger.isInfoEnabled()) {
				logger.info("Overriding user-defined bean definition for bean '" + beanName +
						"' with a framework-generated bean definition: replacing [" +
						existingDefinition + "] with [" + beanDefinition + "]");
			}
		}
        // 说明两个bean不一样，用新的Bean覆盖旧的Bean
		else if (!beanDefinition.equals(existingDefinition)) {
			if (logger.isDebugEnabled()) {
				logger.debug("Overriding bean definition for bean '" + beanName +
						"' with a different definition: replacing [" + existingDefinition +
						"] with [" + beanDefinition + "]");
			}
		}
        // 说明两个bean一样，用新的Bean覆盖旧的Bean
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("Overriding bean definition for bean '" + beanName +
						"' with an equivalent definition: replacing [" + existingDefinition +
						"] with [" + beanDefinition + "]");
			}
		}
        // 覆盖bean
		this.beanDefinitionMap.put(beanName, beanDefinition);
	}
	else {
              // 判断是否已经有其他的 Bean 开始初始化了，注意"注册Bean" 这个动作结束，Bean 依然还没有初始化，在Spring容器启动的最后，会预初始化所有的singleton beans
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
            // 这是个ArrayList，所以会按照bean配置的顺序保存每一个注册的Bean的名字
			this.beanDefinitionNames.add(beanName);
            // 移除手动注册的singleton bean
			removeManualSingletonName(beanName);
		}
		this.frozenBeanDefinitionNames = null;
	}
	if (existingDefinition != null || containsSingleton(beanName)) {
		resetBeanDefinition(beanName);
	}
	else if (isConfigurationFrozen()) {
		clearByTypeCache();
	}
}
```

###### 基于注解的容器创建

Spring中，管理注解Bean定义的容器有两个：`AnnotationConfigApplicationContext`和`AnnotationConfigWebApplicationContex`。这两个类是专门处理Spring注解方式配置的容器，直接依赖于注解作为容器配置信息来源的IOC容器。AnnotationConfigWebApplicationContext是AnnotationConfigApplicationContext的web版本，两者的用法以及对注解的处理方式几乎没有什么差别。

**Debug下来发现实现类是AnnotationConfigServletWebServerApplicationContext，但方法上没啥区别**

**AnnotationConfigApplicationContext**

两种方式（显式调用带参数的构造方法才会调用register或scan方法）

- 根据配置类进行注册 `AnnotatedBeanDefinitionReader`
- 扫描包路径进行注册 `ClassPathBeanDefinitionScanner`

**根据配置类进行注册**

```java
// 构造方法
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
    // 0.先执行父类的构造方法
    // 1.注册beanDefinition reader和scanner
	this();
    // 2.注册bean
	register(componentClasses);
    // 3.刷新容器
	refresh();
}

// 0.父类的构造方法
public GenericApplicationContext() {      
    // DefaultListableBeanFactory为xml创建容器的实现类
    this.beanFactory = new DefaultListableBeanFactory();
}

// 1.注册beanDefinition reader和scanner
public AnnotationConfigApplicationContext() {
    // 分别用于读取配置类、扫描包路径
	this.reader = new AnnotatedBeanDefinitionReader(this);
	this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

```java
// 2.注册bean
@Override
public void register(Class<?>... componentClasses) {
	Assert.notEmpty(componentClasses, "At least one component class must be specified");
	this.reader.register(componentClasses);
}

public void register(Class<?>... componentClasses) {
    // 遍历配置类，一般就一个，即启动类，如：class com.irol.study.StudyApplication
	for (Class<?> componentClass : componentClasses) {
		registerBean(componentClass);
	}
}

public void registerBean(Class<?> beanClass) {
	doRegisterBean(beanClass, null, null, null, null);
}

// AnnotatedBeanDefinitionReader注册bean
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
		@Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
		@Nullable BeanDefinitionCustomizer[] customizers) {
    // 注解bean的封装结构
	AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
	if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
		return;
	}
	abd.setInstanceSupplier(supplier);
    // 2.1 解析作用域scope
	ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
	abd.setScope(scopeMetadata.getScopeName());
    // 设置bean名称
	String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
    // 2.2 处理通用注解
	AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    // 如果使用了额外的限定注解，如@Qualifier，则解析限定符注解，若无限定符则默认是按类型装配
	if (qualifiers != null) {
		for (Class<? extends Annotation> qualifier : qualifiers) {
            // 若配置了@Primary注解，该bean为自动注入时的首选
			if (Primary.class == qualifier) {
				abd.setPrimary(true);
			}
            // 若配置了@Lazy，该bean为延迟初始化，即使用时才初始化
			else if (Lazy.class == qualifier) {
				abd.setLazyInit(true);
			}
            // 使用了@Primary与@Lazy之外的注解，则添加该限定符，在自动注入的时候会按名称装配
			else {
				abd.addQualifier(new AutowireCandidateQualifier(qualifier));
			}
		}
	}
	if (customizers != null) {
		for (BeanDefinitionCustomizer customizer : customizers) {
			customizer.customize(abd);
		}
	}
	BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    // 2.3 根据bean中定义的作用域scope，创建对应的代理对象
	definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    // 2.4 注册bean
	BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}
```

```java
// 2.1 解析作用域scope
@Override
public ScopeMetadata resolveScopeMetadata(BeanDefinition definition) {
	ScopeMetadata metadata = new ScopeMetadata();
	if (definition instanceof AnnotatedBeanDefinition) {
        // 获取bean的scope属性，即@Scope注解的值
		AnnotatedBeanDefinition annDef = (AnnotatedBeanDefinition) definition;
		AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(
				annDef.getMetadata(), this.scopeAnnotationType);
		if (attributes != null) {
            // 将获取到的scope设置到返回值中
			metadata.setScopeName(attributes.getString("value"));
            // 获取@Scope红的proxyMode属性，创建代理对象时会使用
			ScopedProxyMode proxyMode = attributes.getEnum("proxyMode");
            // proxyMode为DEFAULT或NO时，proxyMode设置为NO
			if (proxyMode == ScopedProxyMode.DEFAULT) {
				proxyMode = this.defaultProxyMode;
			}
            // 将proxyMode设置到返回值中
			metadata.setScopedProxyMode(proxyMode);
		}
	}
	return metadata;
}
```

```java
// 2.2 处理通用注解
public static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd) {
	processCommonDefinitionAnnotations(abd, abd.getMetadata());
}

static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
	AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
    // 若配置了@Lazy，设置@Lazy的值
	if (lazy != null) {
		abd.setLazyInit(lazy.getBoolean("value"));
	}
	else if (abd.getMetadata() != metadata) {
		lazy = attributesFor(abd.getMetadata(), Lazy.class);
		if (lazy != null) {
			abd.setLazyInit(lazy.getBoolean("value"));
		}
	}
    // 若配置了@Primary注解，该bean为自动注入时的首选
	if (metadata.isAnnotated(Primary.class.getName())) {
		abd.setPrimary(true);
	}
    // 若配置了@DependsOn注解，则设置依赖的bean名称，容器在实例化bean之前会先实例化所依赖的bean
	AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
	if (dependsOn != null) {
		abd.setDependsOn(dependsOn.getStringArray("value"));
	}
    // 若配置了@Role注解，则设置role的值，这个注解很少用
	AnnotationAttributes role = attributesFor(metadata, Role.class);
	if (role != null) {
		abd.setRole(role.getNumber("value").intValue());
	}
    // 若配置了@Description注解，则设置Description的值
	AnnotationAttributes description = attributesFor(metadata, Description.class);
	if (description != null) {
		abd.setDescription(description.getString("value"));
	}
}
```

```java
// 2.3 根据bean中定义的作用域scope，创建对应的代理对象
static BeanDefinitionHolder applyScopedProxyMode(
		ScopeMetadata metadata, BeanDefinitionHolder definition, BeanDefinitionRegistry registry) {
    // 获取proxyMode属性
	ScopedProxyMode scopedProxyMode = metadata.getScopedProxyMode();
    // 若proxyMode属性设置为NO，则不使用代理模式
	if (scopedProxyMode.equals(ScopedProxyMode.NO)) {
		return definition;
	}
    // 判断proxyMode是否为TARGET_CLASS
	boolean proxyTargetClass = scopedProxyMode.equals(ScopedProxyMode.TARGET_CLASS);
    // 创建代理对象
	return ScopedProxyCreator.createScopedProxy(definition, registry, proxyTargetClass);
}
```

```java
// 2.4 注册bean
protected void registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry) {
	BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, registry);
}

public static void registerBeanDefinition(
		BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
		throws BeanDefinitionStoreException {
	// Register bean definition under primary name.
	String beanName = definitionHolder.getBeanName();
    // 注册beanDefinition，这里调用的是DefaultListableBeanFactory的registerBeanDefinition方法
	registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
	// Register aliases for bean name, if any.
	String[] aliases = definitionHolder.getAliases();
    // 如果存在别名，则注册别名，逻辑与上面类似
	if (aliases != null) {
		for (String alias : aliases) {
			registry.registerAlias(beanName, alias);
		}
	}
}
```

```java
// 3.刷新容器
// 跟上面2.11容器初始化refreshContext调用的refresh方法一样，只是在2.11.2创建bean容器这一步的实现不同
// 由于在scan方法已经进行过容器的初始化了，所以obtainFreshBeanFactory()的具体实现是调用GenericApplicationContext的refreshBeanFactory方法

protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // refreshBeanFactory这里调用的是GenericApplicationContext里的实现
	refreshBeanFactory();
	return getBeanFactory();
}

// refreshBeanFactory，基本没干啥
@Override
protected final void refreshBeanFactory() throws IllegalStateException {
	if (!this.refreshed.compareAndSet(false, true)) {
		throw new IllegalStateException(
				"GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
	}
	this.beanFactory.setSerializationId(getId());
}
```

**根据包路径进行注册**

```java
// 构造方法
public AnnotationConfigApplicationContext(String... basePackages) {
    // 0.先执行父类的构造方法
    // 1.注册beanDefinition reader和scanner
	this();
    // 2.扫描指定路径，注册bean
	scan(basePackages);
    // 3.刷新容器
	refresh();
}
```

```java
// 2.扫描指定路径
@Override
public void scan(String... basePackages) {
	Assert.notEmpty(basePackages, "At least one base package must be specified");
	this.scanner.scan(basePackages);
}

public int scan(String... basePackages) {
    // 2.1 获取已注册bean的数量
	int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
    // 2.2 扫描指定路径，注册bean
	doScan(basePackages);
	// Register annotation config processors, if necessary.
    // 2.3 注册配置处理器
	if (this.includeAnnotationConfig) {
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}
    // 2.4 返回此次扫描注册的bean数量
	return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}

// 2.2 扫描指定路径，注册bean
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
	Assert.notEmpty(basePackages, "At least one base package must be specified");
	Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    // 遍历路径
	for (String basePackage : basePackages) {
        // 2.2.1 获取所有符合条件的BeanDefinition
		Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        // 绑定BeanDefinition与Scope
		for (BeanDefinition candidate : candidates) {
            // 获取scope
			ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            // 设置bean的作用域scope
			candidate.setScope(scopeMetadata.getScopeName());
            // 设置bean名称
			String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
             // 下面两个if是处理lazy、Autowire、DependencyOn、initMethod、enforceInitMethod、destroyMethod、enforceDestroyMethod、Primary、Role、Description这些逻辑的
			if (candidate instanceof AbstractBeanDefinition) {
				postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
			}
			if (candidate instanceof AnnotatedBeanDefinition) {
				AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
			}
            // 检查bean是否已存在
			if (checkCandidate(beanName, candidate)) {
                // 把BeanDefinition又封装了一层
				BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                // 根据注解中的作用域，为bean设置相应的代理模式
				definitionHolder =
						AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
				beanDefinitions.add(definitionHolder);
                // 2.2.2 注册beanDefinition
				registerBeanDefinition(definitionHolder, this.registry);
			}
		}
	}
	return beanDefinitions;
}
```

```java
// 2.2.1 获取所有符合条件的BeanDefinition
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
	if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
		return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
	}
	else {
        // 扫描
		return scanCandidateComponents(basePackage);
	}
}

private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
	Set<BeanDefinition> candidates = new LinkedHashSet<>();
	try {
        //组装扫描路径（组装完成后是这种格式： classpath*:cn/shiyujun/config/**/*.class）
		String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
				resolveBasePackage(basePackage) + '/' + this.resourcePattern;
        //根据路径获取资源对象
		Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
		boolean traceEnabled = logger.isTraceEnabled();
		boolean debugEnabled = logger.isDebugEnabled();
        // 遍历指定包
		for (Resource resource : resources) {
			if (traceEnabled) {
				logger.trace("Scanning " + resource);
			}
			if (resource.isReadable()) {
				try {
                    // 根据资源对象通过反射获取资源对象的MetadataReader
					MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                    // 查看配置类是否有@Conditional一系列的注解，是否满足注册Bean的条件
					if (isCandidateComponent(metadataReader)) {
						ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
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

// MetadataReader
public interface MetadataReader {
    // 配置类的资源对象
    Resource getResource();
    // 类元数据
    ClassMetadata getClassMetadata();
    // 类注解数据
    AnnotationMetadata getAnnotationMetadata();
}
```

```java
// 2.2.2 注册beanDefinition
// 与上面根据配置类注册bean的过程相同
```

```java
// 3.刷新容器
// 根据配置类进行注册的过程相同
```

##### 2.11.3 准备BeanFactory

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 设置类加载器
	beanFactory.setBeanClassLoader(getClassLoader());
    // 设置EL表达式解析器
	beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    // 设置属性注册解析器PropertyEditor
	beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
    
    // 在所有实现了Aware接口的bean在初始化的时候，这个 processor负责回调，
    // 这个我们很常用，如我们会为了获取 ApplicationContext 而 implement ApplicationContextAware
    // 它不仅仅回调ApplicationContextAware，还会回调EnvironmentAware、ResourceLoaderAware等
	beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    
    // 下面几行的意思就是，如果某个bean依赖于以下几个接口的实现类，在自动装配的时候忽略它们，Spring会通过其他方式来处理这些依赖。
	beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
	beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
	beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
	beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    
    //下面几行就是为特殊的几个 bean 赋值，如果有 bean 依赖了以下几个，会注入相应的值
	beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
	beanFactory.registerResolvableDependency(ResourceLoader.class, this);
	beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
	beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // 注册事件监听器
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

##### 2.11.4 调用postProcessBeanFactory()

invokeBeanFactoryPostProcessors(beanFactory)调用BeanFactoryPostProcessor各个实现类的postProcessBeanFactory()方法，其实就一个类`ConfigurationClassPostProcessor`处理@Configuration、@Bean等注解

扫描并处理@PropertySource、@ComponetScan、@Import、@ImportResource、@Bean注解

处理@Import注解时包括主类的@Import注解

参考：https://www.cnblogs.com/toby-xu/p/11332666.html

##### 2.11.9 初始化所有的 singleton beans

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	// Initialize conversion service for this context.
	if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
			beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        // 设置beanFactory类型转换服务
		beanFactory.setConversionService(
				beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
	}
	// Register a default embedded value resolver if no bean post-processor
	// (such as a PropertyPlaceholderConfigurer bean) registered any before:
	// at this point, primarily for resolution in annotation attribute values.
    // EmbeddedValueResolver用于读取配置文件属性
	if (!beanFactory.hasEmbeddedValueResolver()) {
		beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
	}
	// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
    // 先初始化 LoadTimeWeaverAware 类型的 Bean
	String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
	for (String weaverAwareName : weaverAwareNames) {
		getBean(weaverAwareName);
	}
	// Stop using the temporary ClassLoader for type matching.
    // 停止使用用于类型匹配的临时类加载器
	beanFactory.setTempClassLoader(null);
	// Allow for caching all bean definition metadata, not expecting further changes.
    // 冻结所有的bean定义，即已注册的bean定义将不会被修改或后处理
	beanFactory.freezeConfiguration();
	// Instantiate all remaining (non-lazy-init) singletons.
    // ↓ 初始化单例bean
	beanFactory.preInstantiateSingletons();
}
```

```java
// ↓ 初始化单例bean
@Override
public void preInstantiateSingletons() throws BeansException {
	if (logger.isTraceEnabled()) {
		logger.trace("Pre-instantiating singletons in " + this);
	}
	// Iterate over a copy to allow for init methods which in turn register new bean definitions.
	// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
    // this.beanDefinitionNames 保存了所有的 beanNames
	List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
	// Trigger initialization of all non-lazy singleton beans...
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
					FactoryBean<?> factory = (FactoryBean<?>) bean;
                    // 判断当前 FactoryBean 是否是 SmartFactoryBean 的实现
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged(
								(PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
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
                // ↓ 不是FactoryBean的直接使用此方法进行初始化
				getBean(beanName);
			}
		}
	}
	// Trigger post-initialization callback for all applicable beans...
    // 如果bean实现了 SmartInitializingSingleton 接口，那么在这里回调
	for (String beanName : beanNames) {
		Object singletonInstance = getSingleton(beanName);
		if (singletonInstance instanceof SmartInitializingSingleton) {
			SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
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
```

不管是不是FactoryBean，最后都调用了`getBean(beanName)`

```java
@Override
public Object getBean(String name) throws BeansException {
	return doGetBean(name, null, null, false);
}

protected <T> T doGetBean(
		String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
		throws BeansException {
    // 获取beanName，处理两种情况，一个是前面说的 FactoryBean(前面带 ‘&’)，再一个这个方法是可以根据别名来获取Bean的，所以在这里是要转换成最正统的BeanName
    //主要逻辑就是如果是FactoryBean就把&去掉如果是别名就把根据别名获取真实名称
	String beanName = transformedBeanName(name);
    //最后的返回值
	Object bean;
	// Eagerly check singleton cache for manually registered singletons.
    // ↓ 检查是否已初始化，在缓存中查找
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
        // 这里如果是普通Bean 的话，直接返回，如果是 FactoryBean 的话，返回它创建的那个实例对象
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}
	else {
		// Fail if we're already creating this bean instance:
		// We're assumably within a circular reference.
        // 如果存在prototype类型的这个bean
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}
		// Check if bean definition exists in this factory.
        // 如果当前BeanDefinition不存在这个bean且具有父BeanFactory
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			String nameToLookup = originalBeanName(name);
            // 返回父容器的查询结果
			if (parentBeanFactory instanceof AbstractBeanFactory) {
				return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
						nameToLookup, requiredType, args, typeCheckOnly);
			}
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
        // typeCheckOnly 为 false，将当前 beanName 放入一个 alreadyCreated 的 Set 集合中。
		if (!typeCheckOnly) {
			markBeanAsCreated(beanName);
		}
        /**
        ** 到这就要创建bean了
        **/
		try {
			RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);
			// Guarantee initialization of beans that the current bean depends on.
            // 先初始化依赖的所有 Bean， depends-on 中定义的依赖
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dep : dependsOn) {
                    // 检查是不是有DependsOn循环依赖
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
                        // 执行创建 Bean
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
				if (!StringUtils.hasLength(scopeName)) {
					throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
				}
				Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
				}
				try {
					Object scopedInstance = scope.get(beanName, () -> {
						beforePrototypeCreation(beanName);
						try {
                            // 执行创建 Bean
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
```

以上：Spring本身只定义了两种Scope，Singleton和Prototype，一开始会先判断bean存不存在，如果存在会从缓存中取，如果不存在调用`createBean`创建bean

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        // 在一级缓存中查找
		Object singletonObject = this.singletonObjects.get(beanName);
        // 一级缓存中找不到，且bean正在创建中
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
                // 在二级缓存中查找
				singletonObject = this.earlySingletonObjects.get(beanName);
                // 二级缓存中找不到，单例的bean allowEarlyReference都是true
				if (singletonObject == null && allowEarlyReference) {
                    // 在三级缓存中查找
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    // 找到了返回bean，并将bean放入二级缓存，从三级缓存中移除
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

```java
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
    // 准备方法重载，如果bean中定义了 <lookup-method /> 和 <replaced-method />
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
        // ↓ 创建 bean
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
```

```java
// ↓ 创建 bean
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
		throws BeanCreationException {

	// Instantiate the bean.
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) {
        //如果是factoryBean则从缓存删除
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
        // ↓ 1.实例化 Bean，这个方法里面才是终点
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
    //bean实例
	Object bean = instanceWrapper.getWrappedInstance();
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
    // 解决循环依赖问题，判断是否需要提前暴露
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		if (logger.isTraceEnabled()) {
			logger.trace("Eagerly caching bean '" + beanName +
					"' to allow for resolving potential circular references");
		}
            //当正在创建A时，A依赖B，此时通过将A作为ObjectFactory放入单例工厂中进行提前暴露，此处B需要引用A，但A正在创建，从单例工厂拿到ObjectFactory，从而允许循环依赖
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
	}

	// Initialize the bean instance.
	Object exposedObject = bean;
	try {
        // ↓ 2.负责属性装配，很重要
		populateBean(beanName, mbd, instanceWrapper);
        // ↓ 3.初始化bean
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

    // 如果提前暴露
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
							"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
				}
			}
		}
	}

	// Register bean as disposable.
	try {
        // 把bean注册到相应的Scope中
		registerDisposableBeanIfNecessary(beanName, bean, mbd);
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
	}

	return exposedObject;
}
```

```java
// ↓ 1.实例化 Bean，这个方法里面才是终点
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
	// Make sure bean class is actually resolved at this point.
    // 确保已经加载了此 class
	Class<?> beanClass = resolveBeanClass(mbd, beanName);
    // 校验类的访问权限
	if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
	}
	Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
	if (instanceSupplier != null) {
		return obtainFromSupplier(instanceSupplier, beanName);
	}
    // 采用工厂方法实例化
	if (mbd.getFactoryMethodName() != null) {
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}
	// Shortcut when re-creating the same bean...
    //是否第一次
	boolean resolved = false;
    //是否采用构造函数注入
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
			return autowireConstructor(beanName, mbd, null, null);
		}
		else {
            // 无参构造函数
			return instantiateBean(beanName, mbd);
		}
	}
	// Candidate constructors for autowiring?
    // 判断是否采用有参构造函数
	Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
	if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
			mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        // 构造函数依赖注入
		return autowireConstructor(beanName, mbd, ctors, args);
	}
	// Preferred constructors for default construction?
	ctors = mbd.getPreferredConstructors();
	if (ctors != null) {
		return autowireConstructor(beanName, mbd, ctors, null);
	}
	// No special handling: simply use no-arg constructor.
    // 调用无参构造函数
	return instantiateBean(beanName, mbd);
}
```

```java
// ↓ 2.负责属性装配，很重要
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
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                            // 如果返回 false，代表不需要进行后续的属性设值，也不需要再经过其他的 BeanPostProcessor 的处理
				if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
					return;
				}
			}
		}
	}

    // bean的所有属性
	PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

	int resolvedAutowireMode = mbd.getResolvedAutowireMode();
	if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
		MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
		// Add property values based on autowire by name if applicable
        // 通过名字找到所有属性值，如果是 bean 依赖，先初始化依赖的 bean。记录依赖关系
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
			autowireByName(beanName, mbd, bw, newPvs);
		}
		// Add property values based on autowire by type if applicable.
        // 通过类型装配，复杂一些
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
               		// 处理完bean的成员属性后，检查和修改属性
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
        // 前面处理的属性都是PropertyValues形式的，在这里应用到实际的bean中
		applyPropertyValues(beanName, mbd, bw, pvs);
	}
}
```

```java
// ↓ 3.初始化bean
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                // 调用aware方法
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
            // 调用BeanPostProcessor的前置方法，在初始化bean前调用
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
            // 调用bean自定义的init方法
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
            // 调用BeanPostProcessor的后置方法，在初始化bean后调用
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```

##### 2.11.10 结束初始化

```java
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



