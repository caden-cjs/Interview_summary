# 基于注解深入理解Spring（一）

--------------

## 一、IOC

-----------------------------

> 导入pom
>
> ```xml
>         <dependency>
>             <groupId>org.springframework</groupId>
>             <artifactId>spring-context</artifactId>
>             <version>4.3.24.RELEASE</version>
>         </dependency>
> ```

### 1）组件注册

#### 1）启动方式

> 在以前的SSM或者更高的SSH集成开发中,我们更多的会使用application.xml来管理整个项目的IOC容器。
>
> 在比较新的SpringBoot中，我们更多的会使用纯注解的方式，并且使用注解配置类来完成IOC组件的注入

##### 1）xml

> 以前我们都会编写一个这种配置文件,通过包扫描的方式和bean注入的方式将我们所需要的对象交给springIoc容器管理,虽然方便我们的开发,但是过多的配置文件,每次接触到都会特别的杂乱,我们通过对照配置来编写一个配置类。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">
    <!--注入对象-->
    <bean id="person" class="com.caden.bean.Person">
        <property name="age" value="20"/>
        <property name="name" value="小王"/>
    </bean>
    <!--包扫描-->
   <context:component-scan base-package="com.caden">
        <!--排除扫描的注解-->
        <context:exclude-filter type="annotation" expression="com.caden.annotation.TestControllerAnnotation"/>
    </context:component-scan>
</beans>
```

##### 2）annotation

```java
@Configuration
@ComponentScan(value = "com.caden",
        excludeFilters =
                {@ComponentScan.Filter //排除规则
                        (type = FilterType.ANNOTATION,//使用注解的方式
                                classes = {TestControllerAnnotation.class})})//注解类型
public class BeanConfig {
    //    @Bean//没有名称,默认会使用方法的名字
    @Bean("person01") //这时候IOC中使用的名称是person01
    public Person person() {
        return new Person("李四", 12);
    }
}
```

#### 2）scope

> 注入方式,设置当前对象是单实例还是多实例
>
> * @see ConfigurableBeanFactory#SCOPE_PROTOTYPE
> * String SCOPE_PROTOTYPE = "prototype"; 多实例
> * @see ConfigurableBeanFactory#SCOPE_SINGLETON
> * String SCOPE_SINGLETON = "singleton"; 单实例(默认)
> * web环境下
> * @see org.springframework.web.context.WebApplicationContext#SCOPE_REQUEST 每次请求创建一次
> * @see org.springframework.web.context.WebApplicationContext#SCOPE_SESSION 每个Session创建一次
> * @see #value

##### 1）xml

```xml
    <bean id="person" class="com.caden.bean.Person" scope="prototype">
        <property name="age" value="20"/>
        <property name="name" value="小王"/>
    </bean>
```

##### 2）annotation

```java
    @Bean
    @Scope(value = SCOPE_PROTOTYPE)
    public Person person1() {
        System.out.println("当前是多实例加载");
        return new Person("scope.prototype", 12);
    }
    @Bean
    @Scope(value = SCOPE_SINGLETON)
    public Person person2() {
        System.out.println("这是单实例加载");
        return new Person("scope.singleton", 13);
    }

//---------------------------------------------
    public void annotation() {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ScopeConfig.class);
        System.out.println("容器加载完成");
        Person person = (Person) context.getBean("person1");
        Person person2 = (Person) context.getBean("person1");
        Person person3 = (Person) context.getBean("person2");
        Person person4 = (Person) context.getBean("person2");
        System.out.println("获取对象完成");
        System.out.println(person == person2);
        System.out.println(person3 == person4);
    }
```

- 运行结果

```bash
这是单实例加载
容器加载完成
当前是多实例加载
当前是多实例加载
获取对象完成
false
true
```

- 通过结果可知,在ioc容器加载的时候就已经将单实例的对象加载完成了,当容器加载完成后,每次获取多实例的person的时候,都再次加载了方法,通过比对两个对象的内存地址可知,多实例是分别创建了两次对象,而单实例对象类似通过ioc容器中get的对象 两次获取到的都是同一个对象

#### 3）懒加载

> 懒加载,实际上对应的就是单例模式中的饿汉(饥汉)模式,饱汉(懒汉)模式的区别
>
> 当开启懒加载的时候对应的是饿汉模式,对象在第一次使用的时候才会被创建出来
>
> 默认情况下spring提供的是饱汉模式,在创建IOC容器的时候将需要的对象加载出来,调用的时候可以随时调用

##### 1）xml

```xml
    <bean id="person" class="com.caden.bean.Person" lazy-init="true"><!--懒加载-->
        <property name="age" value="20"/>
        <property name="name" value="小王"/>
    </bean>
```

##### 2）annotation

```java
    @Bean
    public Person person2() {
        System.out.println("这是单实例加载");
        return new Person("scope.singleton", 13);
    }

    @Bean
    @Lazy
    public Person person3() {
        return new Person("lazy", 14);
    }
//---------------------
    @Test
    public void annotation() {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ScopeConfig.class);
        System.out.println("容器加载完成");
        Person person = (Person) context.getBean("person1");
        Person person2 = (Person) context.getBean("person3");
        System.out.println("获取对象完成");
    }
```

- 运行结果

- ```bash
    这是单实例加载
    容器加载完成
    当前是多实例加载
    获取对象完成
    ```

#### 4）条件注入

> spring4.0之后提供的功能,根据条件注入
>
> ```java
> @Target({ElementType.TYPE, ElementType.METHOD})
> @Retention(RetentionPolicy.RUNTIME)
> @Documented
> public @interface Conditional {
>     /**    * All {@link Condition}s that must {@linkplain Condition#matches match}    * in order for the component to be registered.    */  
>     Class<? extends Condition>[] value(); //需要多个Condition
> }
> ```
>
> 模拟一个需求,注入两个对象,根据需求,如果当前系统是windows,注入第一个对象,如果当前系统是linux注入第二个对象。

##### 1）实现接口

```java
public class LinuxCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        //获取对象工厂
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        //获取类加载器
        ClassLoader classLoader = context.getClassLoader();
        //获取bean定义的注册类
        BeanDefinitionRegistry registry = context.getRegistry();
        //获取当前系统环境
        Environment environment = context.getEnvironment();
        String system = environment.getProperty("os.name");
        return system.contains("linux");
    }
}
//------------------------------------------
public class WindowsCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Environment environment = context.getEnvironment();
        String system = environment.getProperty("os.name");
        return system.contains("Windows");
    }
}
```

##### 2）注解配置

```java
@Configuration
public class ConditionConfig {
    @Conditional({
            WindowsCondition.class //设置外面编写的对象
    })
    @Bean("bill")
    public Person person1() {
        return new Person("比尔盖茨", 62);
    }

    @Conditional({
            LinuxCondition.class
    })
    @Bean("linus")
    public Person person2() {
        return new Person("林纳斯", 55);
    }
}
//@Conditional还可以标注到配置类上,当标注到配置类上的时候,则当前类中所有的bean都需要根据这个条件判断
```

##### 3）运行

```java
    @Test
    public void annotation() {
        /*根据当前操作系统
         *  当前系统是windows时注入比尔盖茨
         *  当前系统是linux注入林纳斯
         * */
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ConditionConfig.class);
        String[] beanNamesForType = context.getBeanNamesForType(Person.class);
        for (String s : beanNamesForType) {
            Object bean = context.getBean(s);
            System.out.println(bean);
        }
    }
```

> 运行结果：
>
> Person(name=比尔盖茨, age=62)
>
> 修改参数：
>
> ![](https://mypic-1252529543.cos.ap-shanghai.myqcloud.com/20190729151513.png)
>
> Person(name=林纳斯, age=55)

##### 4）判断条件

```java
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) { 
        //获取bean定义的注册类
        BeanDefinitionRegistry registry = context.getRegistry();
        boolean person = registry.containsBeanDefinition("person");//判断容器中是否有这个类
    }
```

#### 5）注册组件的方式

##### 1）包扫描

- @Controller/@Service/@Repository/@Component

##### 2）@Bean

- 导入第三方包里面的组件

##### 3）@Import

> 查看Import的源码 
>
> ```java
> /** * 
> {@link Configuration}, 
> {@link ImportSelector}, 
> {@link ImportBeanDefinitionRegistrar} 
> * or regular component classes to import. */
> ```
>
> 可以在import中写Configuration对象ImportSelector对象以及ImportBeanDefinitionRegistrar或者直接需要注入的对象

- 快速给容器中导入一个组件

```java
@Configuration
@Import(Person.class)
public class ImportConfig {
}
```

> 配置文件这个的时候我们获取组件可以获取到一个名称是com.caden.bean.Person的组件

###### 1）ImportSelector

```java
public class MyImportSelector implements ImportSelector {//我们定义一个选择器
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        //返回一个数组,数组中包含多个类,这些类则会自动注入
        return new String[]{"com.caden.bean.Red","com.caden.bean.Blue"};
    }
}
//配置类
@Configuration
@Import({Person.class, MyImportSelector.class})
public class ImportConfig {
}
```

> com.caden.bean.Person
> com.caden.bean.Red
> com.caden.bean.Blue
>
> 打印容器中所有的类可知,通过选择器,import了red以及blue

###### 2）ImportBeanDefinitionRegistrar

> 条件注入

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    //定义了一个MyImportBeanDefinitionRegistrar
    /**
     * @param importingClassMetadata 当前类的注解信息
     * @param registry               当前IOC容器的注册信息
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        boolean red = registry.containsBeanDefinition("com.caden.bean.Red");
        boolean yellow = registry.containsBeanDefinition("com.caden.bean.yellow");
        boolean person = registry.containsBeanDefinition("com.caden.bean.Person");
        if (red && person) {
            //如果同时注入了red和person
            //指定bean定义信息(Bean的类型)
            RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(Object.class);
            //注册一个bean,指定名称
            registry.registerBeanDefinition("rootBeanDefinition", rootBeanDefinition);
        }
        if (red && yellow) {
            //如果同时注入了red和yellow,则注入
            RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(Object.class);
            //注册一个bean,指定名称
            registry.registerBeanDefinition("redYellowBeanDefinition", rootBeanDefinition);
        }
    }
}
//-------------------配置类
@Configuration
@Import({Person.class, MyImportSelector.class, MyImportBeanDefinitionRegistrar.class})
public class ImportConfig {
}
```

> 结果
>
> importConfig
> com.caden.bean.Person
> com.caden.bean.Red
> com.caden.bean.Blue
> rootBeanDefinition
>
> 因为yello并不存在,所以没有注入redYellowBeanDefinition

##### 4）FactoryBean

> 通过创建工厂bean创建对象

```java
public class PersonFactoryBean implements FactoryBean<Person> {
    @Override
    public Person getObject() throws Exception {
        System.out.println("创建factoryPerson");
        return new Person("factory", 100);
    }

    @Override
    public Class<?> getObjectType() {
        return Person.class;//返回对象的类型
    }

    @Override
    public boolean isSingleton() {
        return true;//返回true是单例,否则是多例
    }
}
//------------------------配置类
@Configuration
public class FactoryConfig {
    @Bean
    public PersonFactoryBean personFactoryBean() {
        return new PersonFactoryBean();
    }
}
//-----------------主程序
    @Test
    public void annocation() {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(FactoryConfig.class);
        Class<?> personFactoryBean = context.getType("personFactoryBean");
        Class<?> factoryConfig = context.getType("&personFactoryBean");
        context.getBean("personFactoryBean");//只会创建一次
        context.getBean("personFactoryBean");
        System.out.println(personFactoryBean);
        System.out.println(factoryConfig);
    }
```

> 运行结果
>
> 创建factoryPerson
> class com.caden.bean.Person
> class com.caden.factorybean.PersonFactoryBean

### 2）Bean的生命周期

> bean创建 --> bean初始化 --> bean销毁

#### 1）指定初始化和销毁方法

##### 1）xml

```xml
<bean id="person" class="com.caden.bean.Person" lazy-init="true"
           <!--初始化方法-->		 
          init-method="toString" 
		  <!--销毁方法-->
          destroy-method="getAge">
</bean>
```

##### 2）annotation

```java
@Configuration
public class InitDestoryConfig {
    @Bean(initMethod = "init", destroyMethod = "destory")
    public Person person() {
        return new Person("init", 12);
    }
}
```

##### 3）运行结果

```java
@Configuration
public class InitDestoryConfig {
    @Bean(initMethod = "init", destroyMethod = "destory")
    public Person person() {
        return new Person("init", 12);
    }
}
```

> person创建了
>
> <spring log>
>
> person销毁了

- **单实例**：init方法在bean创建成功且成功赋值之后调用,销毁方法在ioc容器销毁之前进行销毁,主要用来销毁连接池等

- **多实例**：获取的时候调用init方法，销毁方法并不会在容器关闭的时候进行销毁

#### 2）实现接口

##### 1）InitializingBean

> 需要实例化的bean继承InitializingBean,编写**afterPropertiesSet**方法,效果类似**initMethod**
>
> 都是在bean对象创建完成并且赋值成功后调用,在**init-method**方法前执行

##### 2）DisposableBean

> 需要实例化的bean继承DisposableBean,编写destory方法,效果类似destory-method
>
> 都是在Ioc容器销毁的时候调用,但是实在destory-method前调用

```java
//创建一个对象
public class Blue implements InitializingBean, DisposableBean {
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("InitializingBean blue创建成功");
    }

    public void init() {
        System.out.println("blue的init方法");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("DisposableBean的destory方法");
    }

    public void destory1() {
        System.out.println("blue的destory1方法");
    }
}
//编写配置类
@Configuration
public class InitializingConfig {
    @Bean(initMethod = "init", destroyMethod = "destory1")
    public Blue blue() {
        return new Blue();
    }
}
//调用
    @Test
    public void annotation() {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(InitializingConfig.class);
        context.destroy();
    }
```

> 运行结果:
>
> InitializingBean blue创建成功
> blue的init方法
>
> <spring log>
>
> DisposableBean的destory方法
> blue的destory1方法

#### 3）JSR250

> 主要在JSR250规范中的两个注解,标注在bean对象的方法之上
>
> @PostConstruct:构造器执行之后执行,紧跟着构造器
>
> @PreDestroy:在对象销毁之前执行
>
> 

```java
public class Blue implements InitializingBean, DisposableBean {
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("InitializingBean blue创建成功");
    }

    public void init() {
        System.out.println("blue的init方法");
    }

    public Blue() {
        System.out.println("构造器执行");
    }
//我们对Blue对象进行了改造
    //构造器之后执行的方法
    @PostConstruct
    public void constructAfter() {
        System.out.println("postConstruct执行");
    }
	//bean对象销毁之前的
    @PreDestroy
    public void destoryAfter() {
        System.out.println("destoryAfter执行");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("DisposableBean的destory方法");
    }

    public void destory1() {
        System.out.println("blue的destory1方法");
    }
}
//其他配置类和测试执行方法和上一个例子相同就不贴出来了
//结果,在创建对象时,因为postConstruct是在构造器之后执行在赋值之前则比initmethod和IntializingBean的方法都要早,在销毁对象的时候,spring会在对象销毁之前回调需要销毁对象destoryAfter注解注释的方法
```

> 结果:
>
> 构造器执行
> postConstruct执行
> InitializingBean blue创建成功
> blue的init方法
>
> <spring log>
>
> destoryAfter执行
> DisposableBean的destory方法
> blue的destory1方法

#### 4） BeanPostProcessor

> Bean后置处理器,在初始化前和销毁之后调用,我们可以自己编写一个后置处理器注入到容器中
>
> 通过观察源码注释postProcessBeforeInitialization方法是在InitializingBean.afterPropertiesSet和init-method之前执行的,紧跟着构造器,比jsr250的postConstruct还要早一点

```java
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("后置处理器初始化执行:" + beanName);
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("后置处理器销毁执行:" + beanName);
        return bean;
    }
}
```

> 配置类中将这个后置处理器注入到容器中
>
> 执行结果:
>
> 构造器执行
> 后置处理器初始化执行:blue
> postConstruct执行
> InitializingBean blue创建成功
> blue的init方法
> 后置处理器销毁执行:blue
>
> destoryAfter执行
> DisposableBean的destory方法
> blue的destory1方法

##### 1） 阅读源码

> 通过阅读spring的启动源码了解BeanPostProcessor的具体流程

```java
//1.创建IOC容器
 context = new AnnotationConfigApplicationContext(InitializingConfig.class);
//2.IOC容器中的刷新方法
	public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		this();
		register(annotatedClasses);
		refresh();//刷新方法
	}
//3.refresh方法调用了初始化单例对象
finishBeanFactoryInitialization(beanFactory);//通过一系列getBean后都没有获取到bean后,开始初始化这个bean
//4.创建bean
//4.1加载单例的IOC对象
beanFactory.preInstantiateSingletons()
	public void preInstantiateSingletons() throws BeansException {
    	//获取所有需要加载的bean的名称
    	List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);
        for (String beanName : beanNames) {//遍历这些字段
            	//获取bean定义信息
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
            //判断是否不是抽象类,并且是单例的,并且不是懒加载
            	if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
                    if (isFactoryBean(beanName)) {//判断这个对象是否是一个工厂类
                        //暂时只研究对象的加载,这里之后研究
                    }else{
                        //如果不是一个工厂类,则直接加载bean
                        getBean(beanName);
                    }
        }
	}
//4.2 获取bean对象
    if (mbd.isSingleton()) {//判断当前这个对象是否是单例的
        sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {//==>
            @Override
            public Object getObject() throws BeansException {
                    return createBean(beanName, mbd, args);
            }
        });
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    }
//4.3 返回一个对象的实例,如果这个对象没有创建,则创建之后返回
    public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        //首先会根据名称获取这个对象
		Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {//判断是否为空
            /*
            一系列确认当前状态
            */
            singletonObject = singletonFactory.getObject();//回调上面的方法,
//4.4 创建一个对象和应用后置处理
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
 //一系列的判断
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    //4.5调用构造器以及后置处理器
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)throws BeanCreationException {
    	//判断这个bean定义是否是单例的
    	if (mbd.isSingleton()) {
            //如果是单例的,获取未完成实例缓存中,如果存在则删除掉并且将实例返回出来
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
    	//判断是否有未加载完成的实例
    	if (instanceWrapper == null) {
            //如果不存在则重新加载这个实例,创建对象
			instanceWrapper = createBeanInstance(beanName, mbd, args);
//4.6加载对象的构造器
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    	//获取工厂方法名称
    	if (mbd.getFactoryMethodName() != null) {
            //使用工厂方法实例化对象,调用构造方法
			return instantiateUsingFactoryMethod(beanName, mbd, args);
						}
					}
 //4.7对象构造完成后,我们回到doCreateBean中,继续向下看
            //应用后置处理器
            if (!mbd.postProcessed) {//判断当前的bean定义是否已经包装过后置处理器
                  applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
{
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {//获取所有的bean后置处理器
			if (bp instanceof MergedBeanDefinitionPostProcessor) {
                //获取到对应的后置处理器合并到当前的bean定义对象中
				MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
				bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
			}
		}
}
}
          mbd.postProcessed = true;//包装完成后将flag设置成true
        Object exposedObject = bean;
        populateBean(beanName, mbd, instanceWrapper);//开始填充属性   
        if (exposedObject != null) {
            //初始化对象,以及回调后置处理器方法
			exposedObject = initializeBean(beanName, exposedObject, mbd);
           protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) { 
         Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
            applyBeanPostProcessorsBeforeInitialization{
                //方法中回调了Bean后置处理器
                for (BeanPostProcessor processor : getBeanPostProcessors()) {
			result = processor.postProcessBeforeInitialization(result, beanName);
			if (result == null) {
				return result;
```

##### 2）Spring底层的使用

> 我们可以查看BeanPostProcessor所有实现类![](https://mypic-1252529543.cos.ap-shanghai.myqcloud.com/20190730144720.png)

###### 1） ApplicationContextAwareProcessor

> 集成ApplicationContextAware等等接口,这个后置处理器会在注入IOC容器时将ApplicationContext注入到对象
>
>  AutowiredAnnotationBeanPostProcessor负责扫描@Authowired注解

```java
	@Override
	public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		AccessControlContext acc = null;

		if (System.getSecurityManager() != null &&
				(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
						bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
						bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged(new PrivilegedAction<Object>() {
				@Override
				public Object run() {
					invokeAwareInterfaces(bean);
					return null;
				}
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}
-----------------------------------------------------------------------------------------------------
    	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {//判断对象实现了哪个接口,比如实现了ApplicationContextAware
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
			if (bean instanceof ApplicationContextAware) {
                //调用ApplicationContextAware.setApplicationContext方法
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}
```



#### 总结

> 整个Bean的生命周期
>
> 初始化 -> 构造器执行
>
> -> 后置处理器(postProcessBeforeInitialization) -> JSR250@PostConstruct ->
>
> -> 赋值
>
> -> InitializingBean.afterPropertiesSet -> init-method -> 后置处理器(postProcessAfterInitialization)
>
> -> IOC容器准备销毁Bean -> JSR250@PreDestory -> DisposableBean.destoryAfter
>
> -> destory-method -> 销毁对象

### 3） 自动装配

> @Autowired注解
>
> 1. 默认会通过context.getBean(class)方法获取对象
>
> 2. 如果获取到多个值,会根据属性名称作为ID去IOC容器中查询(context.getBean(String)
>
> 3. 如果还没找到则会报错,@Qualifier("testDao01")可以指定名称,不会通过属性名去找
>
> 4. 自动注入默认一定需要容器中存在该对象,可以通过@Autowired(required = false)设置,则IOC容器中没找到也不会出错
>
> 5. @Primary注解在方法或者类上的时候,当前注入的对象一直会作为首选项,依旧可以通过@Qualifier("testDao01")指定对象ID
>
> 6. 支持JSR250(@Resource) 和JSR330(@Inject)
>
>     1. @Resource和@Autowired一样,但是不支持@Primary和@Autowired(required = false),默认是按照属性名查找,也可以@Resource(name = "testDao01")指定ID名称
>
>     2. @Inject
>
>         1. 导入依赖
>
>             ```xml
>               <dependency>
>                         <groupId>javax.inject</groupId>
>                         <artifactId>javax.inject</artifactId>
>                         <version>1</version>
>                     </dependency>
>             ```
>
>         2. 和Autowired功能相似,没有@Autowired(required = false)功能

#### 1） Autowired使用规则

##### 1） 标记在方法上

> 一般是@Bean+参数形式,参数上的Authowired可以省略,会自动从IOC容器获取

```java
    @Bean
    public Color color(@Autowired/*这个也可以省略*/Green green) {
        return new Color(green);
    }
```

##### 2） 标记在构造器上

> 当对象只有一个有参构造器的时候可以不写Autowired,并且参数会自动从IOC容器中获取

```java
    @Autowired //如果只有这一个构造器则可以省略,如果多个构造器则必须写
    public Color(@Autowired/*这个也可以省略*/Green green) {
        System.out.println(green);
        this.green = green;
    }
```

##### 3） 标记在参数上

> 最常用的方式

```java
    @Autowired
    private Color color;
```



#### 2） Aware

> xxxxAware接口,主要通过Bean后置处理器来实现,上面4.2.1已经看过ApplicationContextAwareProcess可以自动注入IOC容器对象的
>
> 每个Aware接口都有对应的xxxAwareProcess后置处理器去处理相应的回调方法

#### 3）Profile

> 多套配置文件
>
> 通过@Profile("xxx")注解可以让注入容器的bean根据环境的不同注入不同的配置
>
> 启动项目时编辑VM选项
>
> -Dspring.profiles.active=name //指定启动的模式

- 案例

```java
//1.配置
@Configuration
@PropertySource("classpath:/dbconfig.properties")//读取配置文件
public class ProfileConfig implements EmbeddedValueResolverAware {
    @Value("${db.user}")//从配置文件中注入值
    private String user;
    private StringValueResolver valueResolver;
    private String driver;

    @Profile("dev")//指定环境
    //如果不指定环境,则在任意环境都会注入,同时profile也可以注解在类上,作用和在方法上相同
    @Bean
    public DataSource dataSourceDev(@Value("${db.password}") String password) throws Exception {
        //创建数据源
        ComboPooledDataSource dataSource = new ComboPooledDataSource();
        //注入值
        dataSource.setUser(user);
        //通过入参@Value注解注入值
        dataSource.setPassword(password);
        dataSource.setJdbcUrl("jdbc:mysql://192.168.64.120:3306/test");
        //通过EmbeddedValueResolverAwareProcess后置处理器注入值
        dataSource.setDriverClass(valueResolver.resolveStringValue("${db.driver}"));
        return dataSource;
    }

    @Profile("test")
    @Bean
    public DataSource dataSourceTest(@Value("${db.password}") String password) throws Exception {
        ComboPooledDataSource dataSource = new ComboPooledDataSource();
        dataSource.setUser(user);
        dataSource.setPassword(password);
        dataSource.setJdbcUrl("jdbc:mysql://192.168.64.120:3306/test");
        dataSource.setDriverClass(valueResolver.resolveStringValue("${db.driver}"));
        return dataSource;
    }

    @Profile("prod")
    @Bean
    public DataSource dataSource03(@Value("${db.password}") String password) throws Exception {
        ComboPooledDataSource dataSource = new ComboPooledDataSource();
        dataSource.setUser(user);
        dataSource.setPassword(password);
        dataSource.setJdbcUrl("jdbc:mysql://192.168.64.120:3306/test");
        dataSource.setDriverClass(valueResolver.resolveStringValue("${db.driver}"));
        return dataSource;
    }

    @Override
    public void setEmbeddedValueResolver(StringValueResolver resolver) {
        this.valueResolver = resolver;
    }
}
//2.运行
public class ProfileMain {
    AnnotationConfigApplicationContext context;

    @Before
    public void startIOC() {
        context = new AnnotationConfigApplicationContext();
    }

    @Test
    public void annotation() {
        context.getEnvironment().setActiveProfiles("dev");//设置环境
        //设置主配置类
        context.register(ProfileConfig.class);
        //刷新容器
        context.refresh();
        String[] beanNamesForType = context.getBeanNamesForType(DataSource.class);
        for (String s : beanNamesForType) {
            System.out.println(s);
        }
    }
}
```

### IOC小结

> 组件添加:
>
> ComponentScan,Bean,Configuration,Component.Controller,Repository,**Conditional**,Primary,Lazy,Scope,**Import**,**ImportSelector**,Bean工厂
>
> 后置处理:
>
> BeanFactoryPostProcessor,BeanDefintionRegistryPostProcessor
>
> SpringIoc容器创建过程:

------------------------

## 二、AOP

------------------

> 在程序运行期间,动态的将某段代码切入到指定方法,指定位置的编程方式
>
> 动态代理
>
> 1.导入AOP
>
> ```xml
>         <dependency>
>             <groupId>org.springframework</groupId>
>             <artifactId>spring-aspects</artifactId>
>             <version>4.3.24.RELEASE</version>
>         </dependency>
> ```
>
> 2. 定义一个方法,在方法运行之前,运行之后,返回之前,出错之后打印日志
> 3. 定义一个日志切片类:切面类里面的方法需要动态感知上面方法的运行

### 1）使用步骤

#### 1）导入pom

```xml
<dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aspects</artifactId>
            <version>4.3.24.RELEASE</version>
        </dependency>
```

#### 2）编写业务

```java
public class MethodDiv {
    public double div(int a, int b) {
        System.out.println("除法执行");
        return a / b;
    }
}
```

#### 3）编写切面类

> 1. 前置通知（@Beofre）
> 2. 后置通知（@After）
> 3. 返回通知（@AfterReturning）
> 4. 异常通知（@AfterThrowing）
> 5. 环绕通知（@Around）
> 6. 定义可重用的切入点（@Pointcut）
> 7. JoinPoint参数

```java
@Aspect
public class LogAspect {
    /*
    本类引用:Pointcut()
    其他类: 方法全名
    * */
    @Pointcut("execution(public double com.caden.bean.MethodDiv.div(int,int))")
    public void pointCut() {
    }

    /*方法运行之前
    	JointPoint对象一定要在参数列表的第一位,否则会出错
    */
    @Before("pointCut()")
    public void startLog(JoinPoint joinPoint) {
        System.out.println("方法运行了,方法名是:" + joinPoint.getSignature().getName());
    }

    /*无论方法是否正常结束*/
    @After("com.caden.aspect.LogAspect.pointCut()")
    public void logEnd(JoinPoint joinPoint) {
        System.out.println(joinPoint.getSignature().getName() + "方法运行结束了");
    }

    /*方法返回之前执行*/
    @AfterReturning(value = "pointCut()", returning = "result")
    public void logReuturn(JoinPoint joinPoint, Object result) {
        System.out.println(joinPoint.getSignature().getName() + "方法的返回值是{" + result + "}");
    }

    /*方法异常执行*/
    @AfterThrowing(value = "pointCut()", throwing = "ex")
    public void logException(JoinPoint joinPoint, Exception ex) {
        System.out.println(joinPoint.getSignature().getName() + "方法异常信息是{" + ex.getMessage() + "}");
    }
}
```

#### 4）配置开启AOP

> @EnableAspectJAutoProxy
>
> 将业务类以及切面类注入到容器中，从容器中获取对象并执行方法即可

```java
@Configuration
@EnableAspectJAutoProxy
public class AspectConfig {
    @Bean
    public MethodDiv methodDiv() {
        return new MethodDiv();
    }

    @Bean
    public LogAspect logAspect() {
        return new LogAspect();
    }
}
```

###  2）Aop原理

#### 1）@EnableAspectJAutoProxy

> 通过@EnableAspectJAutoProxy注解中import了AspectJAutoProxyRegistrar这个对象
>
> 这个对象实现了ImportBeanDefinitionRegistrar接口,会自动执行registerBeanDefinitions方法
>
> //需要注册这个类
>
> AspectJ自动注释代理创建类
>
> org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator
>
> //首先判断org.springframework.aop.config.internalAutoProxyCreator是否在容器中
>
> 第一次肯定没有
>
> 创建一个AnnotationAwareAspectJAutoProxyCreator的bean定义,并且注册到IOC中

2）AnnotationAwareAspectJAutoProxyCreator

>   AspectJAwareAdvisorAutoProxyCreator
>
> ​	-> AbstractAdvisorAutoProxyCreator
>
> ​		-> AbstractAutoProxyCreator
>
> AbstractAutoProxyCreator实现了SmartInstantiationAwareBeanPostProcessor和BeanFactoryAware接口	
>
> SmartInstantiationAwareBeanPostProcessor继承了BeanPostProcessor接口,是一个后置处理器	
>
> BeanFactoryAware继承了BeanFactoryAware接口

```java
//1.创建IOC容器
AnnotationConfigApplicationContext.refresh();//刷新容器
=>AnnotationConfigApplicationContext.registerBeanPostProcessors(beanFactory);//注册后置处理器
	//创建所有的声明的后置处理器
	=>AnnotationConfigApplicationContext.registerBeanPostProcessors()
        =>PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
//<--
	//获取所有需要创建的后置处理器
	String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
	//自动给ioc容器中增加一些后置处理器
	int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));
	//之后对所有的后置处理器进行分类,实现PriorityOrdered接口的,Ordered接口的和其他的
List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
List<String> orderedPostProcessorNames = new ArrayList<String>();
List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
	for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}	
		// First, register the BeanPostProcessors that implement PriorityOrdered.
		//首先注册这些实现PriorityOrdered接口的后置处理器,
		//对这些后置处理器进行排序
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		//注册
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);
		//接着注册Ordered接口的后置处理器
List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
sortPostProcessors(orderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);	
		//最后注册其他后置处理器
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : nonOrderedPostProcessorNames) {
            //org.springframework.aop.config.internalAutoProxyCreator,当前加载的后置处理器=>
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);
//-->
		=>
            //创建bean
            Object sharedInstance = getSingleton(beanName);//首先会获取当前的单例对象,如果存在则直接获取实例对象,当前第一次创建所以不存在
if (sharedInstance != null && args == null) {}
else {
    	//之后判断当前名称是否正在创建中,和是否存在bean对象定义
    if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
		}
	//然后进行检查后获取当前对象的定义信息
    	final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
    	if (mbd.isSingleton()) {//判断是否是单例对象
            //如果是,则获取这个单例对象
            sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {//==>
                @Override
                public Object getObject() throws BeansException {
                    return createBean(beanName, mbd, args);
                }
            });
        }
}
//==>
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    synchronized (this.singletonObjects) {//同步代码块,当前其他代码不可操作这个存放单例对象的map
        //根据名称获取这个单例对象
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            //判断单例对象是否存在
            //判断其他信息
        singletonObject = singletonFactory.getObject();//调用getObject方法在上方=>
        }
       //如果获取到对象或者创建成功则返回创建的对象
        return (singletonObject != NULL_OBJECT ? singletonObject : null);//
    }
}
//=>
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
    //bean定义对象
RootBeanDefinition mbdToUse = mbd;
    //解析这个bean的class
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        //判断解析的class不为空并且mdb中定义的是一个Class对象,但是class对象存在并且bean对象定义的的名称存在则创建一个新的bean定义对象
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}
    //覆盖一些方法 TODO
    mbdToUse.prepareMethodOverrides();
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);//创建一个实体==> 
}
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)throws BeanCreationException {
    //创建实例
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        //判断当前实例是否是单例对象,如果是单例对象则从工厂bean中将这个包装对象拿出来
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {//如果没从bean工厂中获取到,则根据这个对象的名称创建一个包装对象
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
        //获取这个对象实例
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
	//获取这个对象的类型
    Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
    mbd.resolvedTargetType = beanType;
}
```

