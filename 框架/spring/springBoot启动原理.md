# SpringBoot启动源码

## 容器启动

```java
@SpringBootApplication
public class AnalysisApplication {
    public static void main(String[] args) {
        SpringApplication.run(AnalysisApplication.class, args);
    }
}
```

> 上面是一个正常的SpringBoot的启动类,追踪源码发现,SpringApplication.run()方法最终会调用这个run方法

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {//①
		return new SpringApplication(primarySources).run(args);
	}
```

> 可知,大体上SpringBoot启动分成两步, 第一步创建一个**SpringApplication**对象，第二部调用这个对象的run方法，返回一个ConfigurableApplicationContext的Ioc上下文对象。
>
> 首先看下创建SpringAllication对象做了些什么。

### 创建**SpringApplication**

```java
public SpringApplication(Class<?>... primarySources) {//②
		this(null, primarySources);
	}	
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    //③ ①=>②=>③ ,可知resourceLoader=null,primarySources=我们自己创建的Application启动类的class对象
		this.resourceLoader = resourceLoader;//null
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));//实际上,是将我们传进来所有的对象设置到这个对象中
		this.webApplicationType = WebApplicationType.deduceFromClasspath();//判断当前是否是web环境
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));//下方详解这个方法,用来获取一些定义好的ApplicationContextInitializer实现类,启动器
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));//同上,这个主要是获取一些定义好的ApplicationListener的实现类,监听器,用于SpringBoot正式启动时的回调事件
		this.mainApplicationClass = deduceMainApplicationClass();//在我们设置的启动类时,可以设置多个类,这里主要是判断main方法所在的类
	}
//这个方法执行完成之后,创建对象结束
```

### 启动SpringBoot

> 整个run方法如下

```java
	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();//设置一个计时器
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();//配置java.awt.headless系统属性的值
		SpringApplicationRunListeners listeners = getRunListeners(args);//获取之前配置好的META-INF/spring.factories文件中获取SpringApplicationRunListener,这个对象会保存SpringApplicationRunListener实现类的集合
		listeners.starting();//调用这个集合的starting方法
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);//// 参数封装，也就是在命令行下启动应用带的参数，如--server.port=9000
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);//配置环境,后面详解
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);//通过配置好的配置环境对象获取logo,并打印
			context = createApplicationContext();//根据当前环境创建对应的上下文对象,普通环境,标准的web环境,非阻塞的web环境
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
new Class[] { ConfigurableApplicationContext.class }, context);//获取异常处理器,SpringBootExceptionReporter接口
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);//准备环境
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```

### 详解

#### getSpringFactoriesInstances

> 这个方法在SpringBoot中非常常用,主要用来获取配置文件中设置的类
>
> 图片是MATE-INF/spring.factories文件中的内容

![image-20200113150823447](C:\Users\Caden\AppData\Roaming\Typora\typora-user-images\image-20200113150823447.png)

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
		return getSpringFactoriesInstances(type, new Class<?>[] {})		;
	}
// ===>
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = getClassLoader();//这个类涉及到类加载,这边获取一个类加载器,猜测SpringBoot预留的可自定义类加载器的入口
		// Use names and ensure unique to protect against duplicates
		Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));//重点
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}
// => 接重点
	public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
		String factoryTypeName = factoryType.getName();
		return loadSpringFactories(classLoader)/**重点**/.getOrDefault(factoryTypeName, Collections.emptyList());
	}
// => 接重点
	private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
		MultiValueMap<String, String> result = cache.get(classLoader);//一个map缓存,保证只会读取一次,第二次将只会从map中读数据
		if (result != null) {
			return result;
		}

		try {
            //public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
            //实际上是获取了META-INF/spring.factories文件中的Key=Value键值对,之后组装成一到一个map中
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					String factoryTypeName = ((String) entry.getKey()).trim();
					for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
						result.add(factoryTypeName, factoryImplementationName.trim());//将值加到map中
                    }
				}
			}
			cache.put(classLoader, result);//将值放到当前类的缓存中,key是ClassLoader,Value是我们需要的map
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
```

#### prepareEnvironment

> ```java
> private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
>       ApplicationArguments applicationArguments) {
>    ConfigurableEnvironment environment = getOrCreateEnvironment();//创建或获取环境
>    configureEnvironment(environment, applicationArguments.getSourceArgs());//配置环境:配置PropertySources和activeProfiles
>    ConfigurationPropertySources.attach(environment);
>    listeners.environmentPrepared(environment);
>    bindToSpringApplication(environment);
>    if (!this.isCustomEnvironment) {
>       environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
>             deduceEnvironmentClass());
>    }
>    ConfigurationPropertySources.attach(environment);
>    return environment;
> }
> ```
>
> 

## 自动配置