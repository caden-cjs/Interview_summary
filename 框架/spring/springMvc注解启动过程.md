# SpringMvc启动过程

## Servlet3.0新特性

### ServletContainerInitializer

> Tomcat容器的ServletContainerInitializer机制的实现，主要交由Context容器和ContextConfig监听器共同实现。
>
> ContextConfig监听器负责在容器启动时读取每个web应用的WEB-INF/lib目录下包含的jar包的META-INF/services/javax.servlet.ServletContainerInitializer，以及web根目录下的META-INF/services/javax.servlet.ServletContainerInitializer(即直接在src下建立的META-INF/services/javax.servlet.ServletContainerInitializer)，通过反射完成这些ServletContainerInitializer的实例化，然后再设置到Context容器中。
>
> Context容器启动时就会分别调用每个ServletContainerInitializer的**onStartup**方法，并将感兴趣的类作为参数传入。
>
> 自己可以编写META-INF/services/javax.servlet.ServletContainerInitializer文件中的实现类
>
> javax.servlet.ServletContainerInitializer文件中的内容
>
> ```bash
> top.cjsx.MyServletContainerInitializer
> ```
>
> @**HandlesTypes**(value = {xx.class})
>
> 该注解主要是在context启动的时候扫描xx这个类的子类子接口,子接口实现类,等所有相关的类,并设置到
> public void onStartup(Set<Class<?>> c, ServletContext ctx)中的Set<Class<?>> c这个参数中。注册三大组件的方法在下面的代码中。可以通过观看代码知晓

```java
package top.cjsx;

import org.springframework.web.filter.CharacterEncodingFilter;
import top.cjsx.controller.HelloServlet;
import top.cjsx.service.HelloService;

import javax.servlet.*;
import javax.servlet.annotation.HandlesTypes;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.nio.charset.StandardCharsets;
import java.util.EnumSet;
import java.util.Set;

//容器启动的时候会将@HandlesTypes指定的这个类型下面的子类（实现类，子接口等）传递过来；
@HandlesTypes(value = {WebApplicationInitializer.class})
public class MyServletContainerInitializer implements ServletContainerInitializer {

    /**
     * 应用启动的时候，会运行onStartup方法；
     * <p>
     * Set<Class<?>> c：感兴趣的类型的所有子类型；
     * ServletContext ctx:代表当前Web应用的ServletContext；一个Web应用一个ServletContext；
     * <p>
     */
    @Override
    public void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException {

        //这里的c会把所有我们感兴趣的类型都拿到
        System.out.println("感兴趣的类型：");
        if (c != null)
            c.forEach(System.out::println);

        //==========================编码形式注册三大组件============================
        ////注册组件  ServletRegistration
        ServletRegistration.Dynamic servlet = ctx.addServlet("hello", new HelloServlet());
        ////配置servlet的映射信息
        servlet.addMapping("/user");
        //
        ////注册Listener
        //ctx.addListener(UserListener.class);
        //
        ////注册Filter  FilterRegistration
        FilterRegistration.Dynamic filter = ctx.addFilter("userFilter", new Filter() {
            @Override
            public void init(FilterConfig filterConfig) throws ServletException {

            }

            public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws ServletException, IOException {
                HttpServletRequest request = (HttpServletRequest) req;
                //处理post请求中文乱码
                request.setCharacterEncoding("utf-8");
                //处理get请求中文乱码
                //使用动态代理增强 HttpServletRequest的 getParameter
                HttpServletRequest requestProxy = (HttpServletRequest) Proxy.newProxyInstance(request.getClass().getClassLoader(), request.getClass().getInterfaces(), new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        //拦截Get请的getParameter方法
                        if ("GET".equalsIgnoreCase(request.getMethod()) && "getParameter".equalsIgnoreCase(method.getName())) {
                            String value = (String) method.invoke(req, args);
                            if (value != null && value.length() > 0) {
                                value = new String(value.getBytes(StandardCharsets.ISO_8859_1), StandardCharsets.UTF_8);
                            }
                            return value;
                        }
                        return method.invoke(req, args);
                    }
                });

                chain.doFilter(requestProxy, resp);
            }

            @Override
            public void destroy() {

            }
        });
        ////配置Filter的映射信息
        filter.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST), true, "/*");
        CharacterEncodingFilter characterEncodingFilter = new CharacterEncodingFilter("UTF-8");
        FilterRegistration.Dynamic encodingFilter = ctx.addFilter("encodingFilter", characterEncodingFilter);
        encodingFilter.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST),true,"/*");
    }
}
```

## SpringMvc

> spring-web5.0之后的版本中已经在jar包中的META-INF/services/java.servlet.ServletContainerInitializer中写入了
>
> ```bash
> org.springframework.web.SpringServletContainerInitializer
> ```
>
> 我们可以通过观察这个类的onstarter方法

```java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {}
```

### onStartup

```java
	@Override
	public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {
//用来存放启动类的list
		List<WebApplicationInitializer> initializers = new LinkedList<>();

		if (webAppInitializerClasses != null) {
            //获取WebApplicationInitializer所有的子类以及实现类
			for (Class<?> waiClass : webAppInitializerClasses) {
				// Be defensive: Some servlet containers provide us with invalid classes,
				// no matter what @HandlesTypes says...
				if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
						WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
					try {
                        //只会将不是接口以及不是抽象类的类增加到list中
						initializers.add((WebApplicationInitializer)
								ReflectionUtils.accessibleConstructor(waiClass).newInstance());
					}
					catch (Throwable ex) {
						throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
					}
				}
			}
		}

		if (initializers.isEmpty()) {
            //如果list是空的则抛出信息
			servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
			return;
		}

		servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
        //对所有的启动类执行排序
		AnnotationAwareOrderComparator.sort(initializers);
		for (WebApplicationInitializer initializer : initializers) {
            //调用启动类的onstartup方法
			initializer.onStartup(servletContext);
		}
	}
```

![SpringMvc启动的相关接口](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20200107163347.png)

#### AbstractContextLoaderInitializer

- 我们通过AbstractContextLoaderInitializer

![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20200108132301.png)

> 通过接口实现了WebApplicationInitializer的**onSrartup**方法,在onStartup方法中调用了**registerContextLoaderListener**方法,在registerContextLoaderListener方法中又调用了一个**抽象**方法**createRootApplicationContext**//主要是用于创建父IOC空间的方法

```java
@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		registerContextLoaderListener(servletContext);
	}
//用于注册一个ContextLoaderListener
	protected void registerContextLoaderListener(ServletContext servletContext) {
		WebApplicationContext rootAppContext = createRootApplicationContext();//在当前类中是一个抽象方法,并且加了注解,这个方法可以返回一个空的对象,用于创建一个父空间IOC容器,在springMvc中,官方建议使用父子容器的概念,父空间可以管理子空间的对象,子空间不可以管理父空间的对象,这样子IOC容器可以专注的负责视图解析以及controller层的处理.
		if (rootAppContext != null) {
            //如果存在父空间IOC容器,那么则设置一个监听器
			ContextLoaderListener listener = new ContextLoaderListener(rootAppContext);
            //ContextLoaderListener继承了ServletContextListener,在springIoc容器刷新的时候会被获取并执行,
			listener.setContextInitializers(getRootApplicationContextInitializers());
			servletContext.addListener(listener);
		}
		else {
			logger.debug("No ContextLoaderListener registered, as " +
					"createRootApplicationContext() did not return an application context");
		}
	}
	@Nullable
	protected abstract WebApplicationContext createRootApplicationContext();
	@Nullable
	protected ApplicationContextInitializer<?>[] getRootApplicationContextInitializers() {
		return null;
	}
```



#### **AbstractDispatcherServletInitializer**

- 再通过观察子抽象类**AbstractDispatcherServletInitializer**

![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20200108134149.png)

```java
public abstract class AbstractDispatcherServletInitializer extends AbstractContextLoaderInitializer {
    public static final String DEFAULT_SERVLET_NAME = "dispatcher";
    @Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		super.onStartup(servletContext);//首先调用了父类的onStartup方法
		registerDispatcherServlet(servletContext);//用于注册DispatcherServlet
	}
    //这个类的主要核心方法
    protected void registerDispatcherServlet(ServletContext servletContext) {
		String servletName = getServletName();//获取一个servletName,我们实现类可以重写这个方法,用于修改servlet方法
		Assert.hasLength(servletName, "getServletName() must not return null or empty");

		WebApplicationContext servletAppContext = createServletApplicationContext();//这个方法在当前类中使一个抽象方法,用于创建一个子IOC容器
		Assert.notNull(servletAppContext, "createServletApplicationContext() must not return null");

		FrameworkServlet dispatcherServlet = createDispatcherServlet(servletAppContext);
        //这个方法在当前抽象类中定义,直接new了一个DispatcherServlet,我们可以重写这个方法实现自定义的DispatcherServlet
		Assert.notNull(dispatcherServlet, "createDispatcherServlet(WebApplicationContext) must not return null");
		dispatcherServlet.setContextInitializers(getServletApplicationContextInitializers());
        //设置一些启动器

		ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);//将创建好的dispatcherServlet设置到servlet容器中,名称就是上文中通过getServletName()获取的名称
		if (registration == null) {
			throw new IllegalStateException("Failed to register servlet with name '" + servletName + "'. " +
					"Check if there is another servlet registered under the same name.");
		}

		registration.setLoadOnStartup(1);//设置启动顺序,1表示立即启动
		registration.addMapping(getServletMappings());//设置servlet映射的目录,可以通过getServletMappings()抽象方法自定义多个映射的目录
		registration.setAsyncSupported(isAsyncSupported());//设置是否支持异步,当前方法是返回true,也可以重写该方法设置

		Filter[] filters = getServletFilters();//当前是一个抽象方法,可以自定义一些Filter
		if (!ObjectUtils.isEmpty(filters)) {
			for (Filter filter : filters) {
				registerServletFilter(servletContext, filter);//当前方法主要是将这些Filter注册到ServletContext中,并且匹配dispatcherServlet
			}
		}

		customizeRegistration(registration);//当前类中是一个空方法,可以重写对注册完成的dispatcherServlet做一些自定义处理
	}
    
    protected FrameworkServlet createDispatcherServlet(WebApplicationContext servletAppContext) {
		return new DispatcherServlet(servletAppContext);
	}
    
    protected String getServletName() {
		return DEFAULT_SERVLET_NAME;
	}
    
    protected abstract WebApplicationContext createServletApplicationContext();
    
    protected FrameworkServlet createDispatcherServlet(WebApplicationContext servletAppContext) {
		return new DispatcherServlet(servletAppContext);
	}
    
    @Nullable
	protected ApplicationContextInitializer<?>[] getServletApplicationContextInitializers() {
		return null;
	}
    protected abstract String[] getServletMappings();
    
    @Nullable
	protected Filter[] getServletFilters() {
		return null;
	}
    
   protected FilterRegistration.Dynamic registerServletFilter(ServletContext servletContext, Filter filter) {
		String filterName = Conventions.getVariableName(filter);
		Dynamic registration = servletContext.addFilter(filterName, filter);

		if (registration == null) {
			int counter = 0;
			while (registration == null) {
				if (counter == 100) {
					throw new IllegalStateException("Failed to register filter with name '" + filterName + "'. " +
							"Check if there is another filter registered under the same name.");
				}
				registration = servletContext.addFilter(filterName + "#" + counter, filter);
				counter++;
			}
		}

		registration.setAsyncSupported(isAsyncSupported());
		registration.addMappingForServletNames(getDispatcherTypes(), false, getServletName());
		return registration;
	}

	private EnumSet<DispatcherType> getDispatcherTypes() {
		return (isAsyncSupported() ?
				EnumSet.of(DispatcherType.REQUEST, DispatcherType.FORWARD, DispatcherType.INCLUDE, DispatcherType.ASYNC) :
				EnumSet.of(DispatcherType.REQUEST, DispatcherType.FORWARD, DispatcherType.INCLUDE));
	}
    
    protected boolean isAsyncSupported() {
		return true;
	}
    
    protected void customizeRegistration(ServletRegistration.Dynamic registration) {
	}
}
```

#### AbstractAnnotationConfigDispatcherServletInitializer

![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20200108143829.png)

- 接着再观察AbstractAnnotationConfigDispatcherServletInitializer抽象类,当前类一个4个方法,当中有两个抽象方法,实现了父类的两个抽象方法,主要用于创建父IOC容器和子IOC容器

```java
public abstract class AbstractAnnotationConfigDispatcherServletInitializer
		extends AbstractDispatcherServletInitializer {
    @Override
	@Nullable
	protected WebApplicationContext createRootApplicationContext() {
        //实现了AbstractContextLoaderInitializer抽象类的方法,在父类中registerContextLoaderListener方法中调用,用于创建父IOC容器
		Class<?>[] configClasses = getRootConfigClasses();//调用了当前类的抽象方法,用于设置父容器的配置信息
		if (!ObjectUtils.isEmpty(configClasses)) {//如果获取到了RootConfigClasse则创建一个IOC容器,并且将所有配置类注册到新的IOC容器中
			AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
			context.register(configClasses);
			return context;//
		}
		else {
			return null;
		}
	}
    
    @Override
   //实现了AbstractDispatcherServletInitializer的抽象方法,这个类的registerDispatcherServlet中调用了该方法,主要用于创建子IOC容器
	protected WebApplicationContext createServletApplicationContext() {
		AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();//因为子容器必须创建,否则Mvc启动没有意义,所以当前容器没有判断是否为空
		Class<?>[] configClasses = getServletConfigClasses();//当前类的抽象方法,获取关与子容器的配置文件
		if (!ObjectUtils.isEmpty(configClasses)) {
			context.register(configClasses);//注册配置文件
		}
		return context;
	}

}
```

#### MyWebAppInitializer(最终自定义的配置文件)

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    /**
     * 根容器的配置类；（Spring的配置文件）父容器；
     */
    @Override
    //继承自AbstractAnnotationConfigDispatcherServletInitializer抽象类,用于这个类在创建父ioc容器时可以读取到配置文件
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[]{RootConfig.class};
    }

    /**
     * web容器的配置类（SpringMVC配置文件子容器；
     */
    @Override
    //继承自AbstractAnnotationConfigDispatcherServletInitializer抽象类,用于这个类在创建子ioc容器时可以读取到配置文件
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[]{AppConfig.class};
    }

    //设置拦截路径
    @Override
    //抽象类AbstractDispatcherServletInitializer中注册dispatcherServlet时配置的匹配路径
    protected String[] getServletMappings() {
//        System.out.println("自定义getServletMappings");
        return new String[]{"/"};
    }

    // 重写父类AbstractDispatcherServletInitializer中的创建dispatcherServlet的方法,相当于自定义一个			dispatcherServlet
    // Spring MVC 推荐自定义
    @Override
    //抽象类AbstractDispatcherServletInitializer中注册dispatcherServlet时配置的匹配路径
    protected FrameworkServlet createDispatcherServlet(WebApplicationContext servletAppContext) {
        DispatcherServlet dispatcherServlet = (DispatcherServlet) 				super.createDispatcherServlet(servletAppContext);
        //dispatcherServlet,获取到这个对象,可以做一些自定义操作
        return dispatcherServlet;
    }
}
```

##### ContextLoaderListener详解

> 在分析AbstractContextLoaderInitializer类中的registerContextLoaderListener方法时,可以看到我们将ContextLoaderListener这个类作为一个servletListener添加到了servletcntext中了.因为整个启动过程中,并没有刷新IOC容器的过程,所以我猜测最终IOC容器刷新是通过这个监听器完成的

```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
    public ContextLoaderListener(WebApplicationContext context) {
		super(context);
	}


	/**
	 * Initialize the root web application context.
	 */
	@Override
	public void contextInitialized(ServletContextEvent event) {
		//当servlet启动的时候会依次调用listener的contextInitialized方法,这个时候会调用这个方法
        initWebApplicationContext(event.getServletContext());//这个方法在ContextLoader中
	}


	/**
	 * Close the root web application context.
	 */
	@Override
	public void contextDestroyed(ServletContextEvent event) {
		closeWebApplicationContext(event.getServletContext());
		ContextCleanupListener.cleanupAttributes(event.getServletContext());
	}
}
```

- initWebApplicationContext方法

![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20200108163236.png)

```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
		if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
			throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
		}
		
		Log logger = LogFactory.getLog(ContextLoader.class);
		servletContext.log("Initializing Spring root WebApplicationContext");
		if (logger.isInfoEnabled()) {
			logger.info("Root WebApplicationContext: initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {
			// Store context in local instance variable, to guarantee that
			// it is available on ServletContext shutdown.
			if (this.context == null) {
                //判断上下文context是否存在,我们在上述方法中创建rootIOC容器的时候已经将context设值了,所以这个值是存在的
				this.context = createWebApplicationContext(servletContext);
			}
            
			if (this.context instanceof ConfigurableWebApplicationContext) {
                //我们创建的context是AnnotationConfigWebApplicationContext类型,在上述继承图中确实属于ConfigurableWebApplicationContext的子类
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;//对当前IOC对象进行强转
				if (!cwac.isActive()) {
                    //是否已经激活
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {//查看当前是否存在父容器
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);//当前这个类中这个方法会返回null
						cwac.setParent(parent);//所以这个父容器依旧是null
					}
					configureAndRefreshWebApplicationContext(cwac, servletContext);//配置和刷新这个容器,主要就是对该容器执行刷新操作,真正的启动spring Ioc容器
				}
			}
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);//将注册好的root容器设置到servletContext中,在初始化SpringMvcIoc中会用到

			ClassLoader ccl = Thread.currentThread().getContextClassLoader();
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = this.context;
			}
			else if (ccl != null) {
				currentContextPerThread.put(ccl, this.context);
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
						WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
			}
			if (logger.isInfoEnabled()) {
				long elapsedTime = System.currentTimeMillis() - startTime;
				logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
			}

			return this.context;
		}
		catch (RuntimeException ex) {
			logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
			throw ex;
		}
		catch (Error err) {
			logger.error("Context initialization failed", err);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
			throw err;
		}
	}
```

![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20200109101033.png)

> 　在SpringIoc容器启动成功后,目前没观察到SpringMvcIoc容器什么时候启动,我猜测在创建DispatcherServlet的时候启动的,观察一下源码,DispatcherServlet因为继承了**FrameworkServlet**,我在这个类中发现了**configureAndRefreshWebApplicationContext**方法中调用了wac.refresh(),可知在这里SpringMvcIoc被刷新了。上溯查询源码

![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20200109101610.png)

> 通过debug发现**GenericServlet**这个类实现了Servlet,因为DispatcherServlet已经被设置到ServletContext中,并且设置了立即启动,所以在被设置到ServletConxtext时,servlet容器就已经调用了Dispatcher的init方法,会上溯到**GenericServlet**的init方法
>
> ```java
>     
> public abstract class GenericServlet 
>     implements Servlet, ServletConfig, java.io.Serializable
> {
> public void init(ServletConfig config) throws ServletException {
> 	this.config = config;
> 	this.init();
>     }
>     public void init() throws ServletException {
> 
>     }
> }
> ```
>
> 在**GenericServlet**的init()方法是空的,我们向下寻找子类中的init()方法
>
> ```java
> public abstract class HttpServletBean extends HttpServlet implements EnvironmentCapable, EnvironmentAware {
> 	@Override
> 	public final void init() throws ServletException {//重写了父类的init()方法,并且增加了final
> 		if (logger.isDebugEnabled()) {
> 			logger.debug("Initializing servlet '" + getServletName() + "'");
> 		}
> 
> 		// Set bean properties from init parameters.
> 		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);//设置一些参数
> 		if (!pvs.isEmpty()) {
> 			try {
> 				BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
> 				ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
> 				bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
> 				initBeanWrapper(bw);
> 				bw.setPropertyValues(pvs, true);
> 			}
> 			catch (BeansException ex) {
> 				if (logger.isErrorEnabled()) {
> 					logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
> 				}
> 				throw ex;
> 			}
> 		}
> 
> 		// Let subclasses do whatever initialization they like.
> 		initServletBean();//主要在这调用了一个空方法initServletBean
> 
> 		if (logger.isDebugEnabled()) {
> 			logger.debug("Servlet '" + getServletName() + "' configured successfully");
> 		}
> 	}
>     	protected void initServletBean() throws ServletException {
> 	}
> }
> ```
>
> 接着向下寻找initServletBean方法在什么类中实现
>
> ![](https://raw.githubusercontent.com/huwd5620125/my_pic_pool/master/img/20200108163236.png)
>
> ```java
> public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
> 	@Override
> 	protected final void initServletBean() throws ServletException {
>         //和init()方法一样,在FrameworkServlet中也对父类的方法重写的同时增加了final标记
> 		getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
> 		if (this.logger.isInfoEnabled()) {
> 			this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
> 		}
> 		long startTime = System.currentTimeMillis();
> 
> 		try {
> 			this.webApplicationContext = initWebApplicationContext();//这里就是初始化springMvcIoc容器的方法了
> 			initFrameworkServlet();
> 		}
> 		catch (ServletException | RuntimeException ex) {
> 			this.logger.error("Context initialization failed", ex);
> 			throw ex;
> 		}
> 
> 		if (this.logger.isInfoEnabled()) {
> 			long elapsedTime = System.currentTimeMillis() - startTime;
> 			this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
> 					elapsedTime + " ms");
> 		}
> 	}
>     
> protected WebApplicationContext initWebApplicationContext() {
>     //上个方法中被调用
> 		WebApplicationContext rootContext =
> 				WebApplicationContextUtils.getWebApplicationContext(getServletContext());
>     //通过servletContext获取在初始化SpringIoc容器的上下文对象
> 		WebApplicationContext wac = null;
> 
> 		if (this.webApplicationContext != null) {
>             //在初始化Dispatcher时设置了的serviceContext
> 			// A context instance was injected at construction time -> use it
> 			wac = this.webApplicationContext;
> 			if (wac instanceof ConfigurableWebApplicationContext) {
>                 //设置的类型和rootcontext类型相同,所以也属于ConfigurableWebApplicationContext
> 				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
> 				if (!cwac.isActive()) {
> 					// The context has not yet been refreshed -> provide services such as
> 					// setting the parent context, setting the application context id, etc
> 					if (cwac.getParent() == null) {
> 						// The context instance was injected without an explicit parent -> set
> 						// the root application context (if any; may be null) as the parent
> 						cwac.setParent(rootContext);//将我们创建的root容器设置为service容器的父容器
> 					}
> 					configureAndRefreshWebApplicationContext(cwac);//对service容器设置一些属性值,ID,servletContext,config信息,命名空间(SpringMvc允许存在多个子容器和多个DispatcherServlet,还设置了一个刷新容器的Listener(ContextRefreshListener),设置了一些初始化器,之后对当前的容器执行刷新操作
>                     /**
>                     
> protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
> 		if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
> 			// The application context id is still set to its original default value
> 			// -> assign a more useful id based on available information
> 			if (this.contextId != null) {
> 				wac.setId(this.contextId);
> 			}
> 			else {
> 				// Generate default id...
> 				wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
> 						ObjectUtils.getDisplayString(getServletContext().getContextPath()) + '/' + getServletName());
> 			}
> 		}
> 
> 		wac.setServletContext(getServletContext());//设置ServletContext
> 		wac.setServletConfig(getServletConfig());//设置servlet的配置信息
> 		wac.setNamespace(getNamespace());//设置命名空间
> 		wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));//设置一个用于刷新的Listener,主要是调用onRefresh()方法的
> 		===>
> 	public void onApplicationEvent(ContextRefreshedEvent event) {
> 		this.refreshEventReceived = true;
> 		onRefresh(event.getApplicationContext());调用这个方法,在DispatcherServlet类中
> 	}
> <===
> 		// The wac environment's #initPropertySources will be called in any case when the context
> 		// is refreshed; do it eagerly here to ensure servlet property sources are in place for
> 		// use in any post-processing or initialization that occurs below prior to #refresh
> 		ConfigurableEnvironment env = wac.getEnvironment();
> 		if (env instanceof ConfigurableWebEnvironment) {
> 			((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
> 		}
> 
> 		postProcessWebApplicationContext(wac);//在当前方法和子类中都是一个空方法,应该是为启动启动方式准备的,用于后置处理context的
> 		applyInitializers(wac);//设置一些初始化器
> 		wac.refresh();//刷新容器
> 	}                    
>                   
>  */
>                     )
> 				}
> 			}
> 		}
> 		if (wac == null) {
> 			// No context instance was injected at construction time -> see if one
> 			// has been registered in the servlet context. If one exists, it is assumed
> 			// that the parent context (if any) has already been set and that the
> 			// user has performed any initialization such as setting the context id
> 			wac = findWebApplicationContext();
> 		}
> 		if (wac == null) {
> 			// No context instance is defined for this servlet -> create a local one
> 			wac = createWebApplicationContext(rootContext);
> 		}
> 
> 		if (!this.refreshEventReceived) {
> 			// Either the context is not a ConfigurableApplicationContext with refresh
> 			// support or the context injected at construction time had already been
> 			// refreshed -> trigger initial onRefresh manually here.
> 			onRefresh(wac);
> 		}
> 
> 		if (this.publishContext) {
> 			// Publish the context as a servlet context attribute.
> 			String attrName = getServletContextAttributeName();
> 			getServletContext().setAttribute(attrName, wac);
> 			if (this.logger.isDebugEnabled()) {
> 				this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
> 						"' as ServletContext attribute with name [" + attrName + "]");
> 			}
> 		}
> 
> 		return wac;
> 	}
>     
> }
> ```

## 小总结

> 1. springMvc利用Servlet3.0新特性在jar包下MATE-INF/services/javax.servlet.ServletContainerInitializer文件中写入了再servlet启动的时候调用的类的类名
>
> ```tex
> org.springframework.web.SpringServletContainerInitializer
> ```
>
> 2. 这个类感兴趣的是这个接口WebApplicationInitializer,servlet启动的时候会将相关的实现类通过SpringServletContainerInitializer类中的**onStartup**()载入,一般需要我们自己编写一个类,上述已经说过了
> 3. 之后会对这些实现类进行排序后依次执行这些类的**onStartup**()方法，默认情况下会最先创建一个root容器（spring+springMvc环境），再通过注册一个listener到servlet中，用于刷新root容器。紧接着会创建servletIoc容器（springMvcIoc容器），再通过这个IOC容器创建一个DespatcherServlet并且添加到ServletContext中，之后通过init方法在servlet创建成功后通过init方法刷新servletIoc容器，刷新容器的时候会通过servletContext获取rootIoc容器，最终启动完成SpringMvc+Spring