# springboot启动原理

---------------

## 几个比较重要的事件回调机制

- **ApplicationContextInitializer**
- **SpringApplicationRunListener** =>1.2.2
- **ApplicationRunner** =>1.2.8
- **CommandLineRunner** =>1.2.8

--------------------

## 一、通过run启动springboot程序

```java
    public static void main(String[] args) {
        SpringApplication.run(SpringbootJpaApplication.class, args);
    }
```


## 1.run方法


```java
public static ConfigurableApplicationContext run(Object source, String... args) {
		return run(new Object[] { source }, args);
	}
//==>
public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
		return new SpringApplication(sources).run(args);
	}//实际是springboot启动的时候首先创建一个springApplication对象,再run启动我们的springboot容器
```

### 1.1创建springApplication

```java
public SpringApplication(Object... sources) {
		initialize(sources);
	}
//===>
	private void initialize(Object[] sources) {
        
		if (sources != null && sources.length > 0) {
             //this.sources对象中将所有需要加载的class记录下来
			this.sources.addAll(Arrays.asList(sources));
		}
       //通过this.webEnvironment记录当前应用是否是web应用
		this.webEnvironment = deduceWebEnvironment();//判断是否是web对象=>1.2.1
        
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));//设置初始化程序=>1.2.2
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

- [1.2.1](#1.2.1.监听器启动)	
- [1.2.2](#1.2.2.getRunListeners()方法)

#### 1.1.1.deduceWebEnvironment()方法

```java
	private boolean deduceWebEnvironment() {
		for (String className : WEB_ENVIRONMENT_CLASSES) {//常量见下方
			if (!ClassUtils.isPresent(className, null)) {//通过遍历,判断这两个class是否存在,若有一个不存在,则判断不是web应用
				return false;
			}
		}
		return true;
	}	

private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };//需要判断的class
```

#### 1.1.2.setInitializers()方法

```java
setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
//==>
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type) {
		return getSpringFactoriesInstances(type, new Class<?>[] {});
	}
//==>
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
		// Use names and ensure unique to protect against duplicates
    	// 确保使用的名称唯一
		Set<String> names = new LinkedHashSet<String>(
				SpringFactoriesLoader.loadFactoryNames(type, classLoader));
   
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}
//loadFactoryNames==>
	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
	public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		try {
			Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) ://通过META-INF/spring.factories寻找ApplicationContextInitializer中所包含的类==> 附录一(挑选了一部分,并非全部)
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			List<String> result = new ArrayList<String>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
				String propertyValue = properties.getProperty(factoryClassName);
				for (String factoryName : StringUtils.commaDelimitedListToStringArray(propertyValue)) {
					result.add(factoryName.trim());
				}
			}
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
//最后将加载成功的类保存到this.initializers中
```

![加载成功的类](https://mypic-1252529543.cos.ap-shanghai.myqcloud.com/20190523140337.png)

#### 1.1.3.setListeners()方法

```java
setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
//getSpringFactoriesInstances(ApplicationListener.class))同上方法.从META-INF/spring.factories中查找ApplicationListener包含的类==>附录二
//保存到this.listeners对象中
```

![](https://mypic-1252529543.cos.ap-shanghai.myqcloud.com/20190523140847.png)

#### 1.1.4.deduceMainApplicationClass()方法

```java
this.mainApplicationClass = deduceMainApplicationClass();//判断主要的配置类
//==>
	private Class<?> deduceMainApplicationClass() {
		try {
			StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
			for (StackTraceElement stackTraceElement : stackTrace) {//因为run方法中可以传多个class文件导springrun方法中,通过判断哪个类中有main方法,判断哪个类是主类
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

---------------------

### 1.2 运行springApplication容器

```java
	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();//创建一个用于监听的类
		stopWatch.start();//监听器启动
		ConfigurableApplicationContext context = null;//设置一个空的ioc容器
		FailureAnalyzers analyzers = null;//异常分析对象
		configureHeadlessProperty();//用于awt.我们暂时不进行探究
		SpringApplicationRunListeners listeners = getRunListeners(args);//获取SpringApplicationRunListeners=>1.2.2
		listeners.starting();//回调所有的SpringApplicationRunListeners的starting方法=>1.2.3
		try {
            //封装命令行参数
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
            //准备环境
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);//1.2.4
			Banner printedBanner = printBanner(environment);//打印Banner图标
			context = createApplicationContext();//创建ioc容器
			analyzers = new FailureAnalyzers(context);//创建异常分析对象
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);//准备环境==>1.2.6回调之前初始化时所保存的所有的ApplicationContextInitializer的initialize,contextPrepared,contextLoaded方法,并向IOC容器中注册了打印信息对象,以及参数对象
			refreshContext(context);//刷新IOC容器,初始化IOC所有组件,如果是web应用,还会创建嵌入式的servlet
			afterRefresh(context, applicationArguments);//回调2个类 ApplicationRunner CommandLineRunner
			listeners.finished(context, null);//所有listeners调用finished方法
			stopWatch.stop();//所有监听器停止
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			return context;//启动完成
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
	}
```

#### 1.2.1.监听器启动

- [1.1](#1.1创建springApplication)

```java
	public void start(String taskName) throws IllegalStateException {
		if (this.running) {
			throw new IllegalStateException("Can't start StopWatch: it's already running");
		}
		this.running = true;
		this.currentTaskName = taskName;
		this.startTimeMillis = System.currentTimeMillis();//设置任务名称为空的监听器的启动时间
	}
```

#### 1.2.2.getRunListeners()方法

```java
SpringApplicationRunListeners listeners = getRunListeners(args);
//==>
private SpringApplicationRunListeners getRunListeners(String[] args) {
		Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
		return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
				SpringApplicationRunListener.class, types, this, args));//通过前面的探究可以知道是从MATE-INF/spring-factories中获取SpringApplicationRunListener为key所包含的对象,并加载=>附录三
	}
//=>
	SpringApplicationRunListeners(Log log,
			Collection<? extends SpringApplicationRunListener> listeners) {
		this.log = log;
		this.listeners = new ArrayList<SpringApplicationRunListener>(listeners);
	}//加载的类
```

![](https://mypic-1252529543.cos.ap-shanghai.myqcloud.com/20190523143022.png)

#### 1.2.3.listeners.starting()方法

```java
	listeners.starting();//开启监听器
//=>
public void starting() {
		for (SpringApplicationRunListener listener : this.listeners) {
            //回调所有的listener的starting方法
			listener.starting();
		}
	}
```

#### 1.2.4.prepareEnvironment()方法

```java
	private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// Create and configure the environment
		ConfigurableEnvironment environment = getOrCreateEnvironment();//获取或者创建一个配置环境
		configureEnvironment(environment, applicationArguments.getSourceArgs());//配置环境,多环境选择等
		listeners.environmentPrepared(environment);//准备环境
        //回调listeners的environmentPrepared方法
		if (!this.webEnvironment) {//判断如果不是web环境的情况下所执行的方法
			environment = new EnvironmentConverter(getClassLoader())
					.convertToStandardEnvironmentIfNecessary(environment);//将环境转换成标准环境
		}
		return environment;
	}
```

#### 1.2.5.createApplicationContext()方法

```java
context = createApplicationContext()
    //=>
    	public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
			+ "annotation.AnnotationConfigApplicationContext";
	public static final String DEFAULT_WEB_CONTEXT_CLASS = "org.springframework."
			+ "boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext";
	protected ConfigurableApplicationContext createApplicationContext() {
		Class<?> contextClass = this.applicationContextClass;
		if (contextClass == null) {
			try {
				contextClass = Class.forName(this.webEnvironment
						? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);//主要是通过判断是否是web环境,如果是则使用org.springframework.context.annotation.AnnotationConfigApplicationContext Ioc容器否则使用AnnotationConfigEmbeddedWebApplicationContext Ioc容器
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Unable create a default ApplicationContext, "
								+ "please specify an ApplicationContextClass",
						ex);
			}
		}
		return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);//通过BeanUtils反射
	}
```

#### 1.2.6.prepareContext()方法

```java
prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
//==>
	private void prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
		context.setEnvironment(environment);//将准备好的环境放入IOC容器对象中
		postProcessApplicationContext(context);//后置处理器,设置了一些组件
		applyInitializers(context);//回调initialize方法
        ////回调之前初始化时所保存的所有的ApplicationContextInitializer的initialize方法
        //==>后面1
		listeners.contextPrepared(context);//回调listener.contextPrepared方法
        //=>后面2
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}

		// Add boot specific singleton beans
		context.getBeanFactory().registerSingleton("springApplicationArguments",
				applicationArguments);//将参数信息注册到Ioc容器中
        //名为:springApplicationArguments
		if (printedBanner != null) {
			context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
		}//注册springBootBanner对象

		// Load the sources
		Set<Object> sources = getSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		load(context, sources.toArray(new Object[sources.size()]));
		listeners.contextLoaded(context);//回调listeners.contextLoaded方法
        //=>3
	}
//=>1
protected void applyInitializers(ConfigurableApplicationContext context) {
		for (ApplicationContextInitializer initializer : getInitializers()) {
			Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
					initializer.getClass(), ApplicationContextInitializer.class);
			Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
            //回调之前初始化时所保存的所有的ApplicationContextInitializer的initialize方法
			initializer.initialize(context);
		}
	}
//=>2
	public void contextPrepared(ConfigurableApplicationContext context) {
		for (SpringApplicationRunListener listener : this.listeners) {
			listener.contextPrepared(context);
		}
	}
//=>3
	public void contextLoaded(ConfigurableApplicationContext context) {
		for (SpringApplicationRunListener listener : this.listeners) {
			listener.contextLoaded(context);//表示环境准备完成
		}
	}
```

#### 1.2.7.refreshContext()方法

```java
private void refreshContext(ConfigurableApplicationContext context) {
   refresh(context);
   if (this.registerShutdownHook) {
      try {
         context.registerShutdownHook();
      }
      catch (AccessControlException ex) {
         // Not allowed in some environments.
      }
   }
}
//=>
	protected void refresh(ApplicationContext applicationContext) {
		Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
		((AbstractApplicationContext) applicationContext).refresh();
	}
//==>
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
```

#### 1.2.8.afterRefresh()方法

```java
protected void afterRefresh(ConfigurableApplicationContext context,
			ApplicationArguments args) {
		callRunners(context, args);
	}
//==>
	private void callRunners(ApplicationContext context, ApplicationArguments args) {
		List<Object> runners = new ArrayList<Object>();
		runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());//从IOC容器中获取所有ApplicationRunner对象
		runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());//从IOC容器中获取所有CommandLineRunner对象
		AnnotationAwareOrderComparator.sort(runners);
		for (Object runner : new LinkedHashSet<Object>(runners)) {
			if (runner instanceof ApplicationRunner) {
				callRunner((ApplicationRunner) runner, args);//回调这些对象的run方法,
			}
			if (runner instanceof CommandLineRunner) {
				callRunner((CommandLineRunner) runner, args);
			}
		}
        //先回调ApplicationRunner.run方法再回调CommandLineRunner.run方法
	}
//=>
	private void callRunner(ApplicationRunner runner, ApplicationArguments args) {
		try {
			(runner).run(args);
		}
		catch (Exception ex) {
			throw new IllegalStateException("Failed to execute ApplicationRunner", ex);
		}
	}
```

## 二、监听器的使用

- 配置在META-INF/spring.factories文件中的2个
  - **ApplicationContextInitializer**
  - **SpringApplicationRunListener**
- 注入到IOC容器的2个
  - **ApplicationRunner** 
  - **CommandLineRunner** 

### 1.自定义几个组件

```properties
##在classpath路径中需要创建META/INF文件夹并且创建spring.factories文件
org.springframework.context.ApplicationContextInitializer=\
com.caden.springboot.listener.MyApplicationContextInitializer
org.springframework.boot.SpringApplicationRunListener=\
com.caden.springboot.listener.MySpringApplicationRunListener
#另外两种listerner需要自己导入到Ioc容器中
```



------------

## 附录

### 一、ApplicationContextInitializer中所包含的一部分类

```properties
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.context.embedded.ServerPortInfoApplicationContextInitializer
```

### 二、ApplicationListener中包含的一部分类

```properties
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener,\
org.springframework.boot.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.logging.LoggingApplicationListener
```

### 三、SpringApplicationRunListener中所包含的一部分类

```properties
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
```

