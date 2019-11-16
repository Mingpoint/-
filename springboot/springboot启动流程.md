##### 启动流程

1、创建SpringApplication

``` java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //判断应用类型
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //从类路径(META-IFN/spring.factories)下加载ApplicationContextInitializer的实现类
setInitializers((Collection)getSpringFactoriesInstances(ApplicationContextInitializer.class));
     //从类路径(META-IFN/spring.factories)下加载ApplicationListener的实现类
setListeners((Collection)getSpringFactoriesInstances(ApplicationListener.class));
    //判断主类
		this.mainApplicationClass = deduceMainApplicationClass();
}

```



2、运行run方法



```java
	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
        //ioc容器
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
        //获取SpringApplicationRunListeners，从类路径META-INF/spring.factories获取
		SpringApplicationRunListeners listeners = getRunListeners(args);
        //回调所有SpringApplicationRunListeners的starting方法
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      	//准备环境
            //1.创建获取环境
            //2.创建获取环境完成后，回调SpringApplicationRunListeners.environmentPrepared
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
            //创建ConfigurableApplicationContext ioc容器
			context = createApplicationContext();
            //异常处理报告
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
            //1.准备上下文，把环境装配到上下文中，
            //2.调用applyInitializers方法，	     
            //applyInitializers方法中回调所有的ApplicationContextInitializer.initialize
            //ApplicationContextInitializer是上面创建SpringApplication加载了的
            //3.回调所有SpringApplicationRunListeners.contextPrepared方法
            //4.回调所有SpringApplicationRunListeners.contextLoaded方法
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            //刷新容器，初始化ioc容器
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
            //1.从ioc容器中获取ApplicationRunner，并回调run 
            //2.从ioc容器中获取CommandLineRunner，并回调run
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

