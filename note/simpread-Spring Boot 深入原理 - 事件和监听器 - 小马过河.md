> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 http://majunwei.com/view/201708262210563675.html

在 [Spring Boot 深入原理 - SpringApplication 启动原理](http://majunwei.com/view/201708231840127244.html)里，我们概括性的介绍了 SpringApplication 的启动流程。

看过的同学一定知道，第一步就是初始化监听器并触发 ApplicationStartedEvent 事件。本文就从 SpringApplication 的第一步开始，深入聊聊 Spring Boot 的事件和监听器。

![](http://static.majunwei.com/userfiles/201610292143075020/images/cms/article/2017/08/event1.png)

![](http://static.majunwei.com/userfiles/201610292143075020/images/cms/article/2017/08/event2.png)

分三个部分来说吧：

*   第一部分是监听器，这里有个 ApplicationListener 接口，所有监听器都要实现这个接口，包括 Spring Boot 内置的监听器以及自定义监听器。
*   第二部分是事件广播器 ApplicationEventMulticaster，用于广播事件，触发事件监听器。
*   第三部分是 SpringApplicationRunListener，用于包装事件广播器，并调用事件广播器的广播方法。

其实还有一个比较重要的接口，没在类图中体现：ApplicationEvent，它是所有内置事件和自定义事件的父接口, 如 ApplicationStartedEvent。

初始化监听器的入口在 SpringApplication.run() 方法里：

```
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        FailureAnalyzers analyzers = null;
        configureHeadlessProperty();
 //初始化监听器
        SpringApplicationRunListeners listeners = getRunListeners(args);
//发布ApplicationStartedEvent
        listeners.starting();
        try {
 //装配参数和环境
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(
                    args);
            ConfigurableEnvironment environment = prepareEnvironment(listeners,
                    applicationArguments);
  //打印Banner
            Banner printedBanner = printBanner(environment);
 //创建ApplicationContext()
            context = createApplicationContext();
            analyzers = new FailureAnalyzers(context);
 //装配Context
            prepareContext(context, environment, listeners, applicationArguments,
                    printedBanner);
  //refreshContext
            refreshContext(context);
  //afterRefresh
            afterRefresh(context, applicationArguments);
            //发布ApplicationReadyEvent
  listeners.finished(context, null);
            stopWatch.stop();
            if (this.logStartupInfo) {
                new StartupInfoLogger(this.mainApplicationClass)
                        .logStarted(getApplicationLog(), stopWatch);
            }
            return context;
        }
        catch (Throwable ex) {
            handleRunFailure(context, listeners, analyzers, ex);
            throw new IllegalStateException(ex);
        }
}

```

进入 getRunListeners() 方法：

```
private SpringApplicationRunListeners getRunListeners(String[] args) {
		Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
		return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
				SpringApplicationRunListener.class, types, this, args));
}

```

这里直接实例化了一个 SpringApplicationRunListeners 对象，这里不是重点，重点是里面的 getSpringFactoriesInstances()：

```
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
		// Use names and ensure unique to protect against duplicates
		Set<String> names = new LinkedHashSet<String>(
				SpringFactoriesLoader.loadFactoryNames(type, classLoader));
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
}

```

这里的核心代码是 SpringFactoriesLoader.loadFactoryNames(type, classLoader)：

```
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		try {
			Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			List<String> result = new ArrayList<String>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
				String factoryClassNames = properties.getProperty(factoryClassName);
				result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
			}
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
					"] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}

```

这块是加载监听器的事件广播器的核心逻辑，注意看这里的常量：

```
/**
	 * The location to look for factories.
	 * <p>Can be present in multiple JAR files.
	 */
	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

```

进入 spring-boot-xxx.RELEASE.jar 的 META-INF 目录下有个 spring.factories 文件：

![](http://static.majunwei.com/userfiles/201610292143075020/images/cms/article/2017/08/event3.png)

这里面定义了大量的接口实现类配置，其中就有本文中重要的 org.springframework.boot.context.event.EventPublishingRunListener：

```
# Run Listeners

org.springframework.boot.SpringApplicationRunListener=\

org.springframework.boot.context.event.EventPublishingRunListener

```

回到 loadFactoryNames 方法，这里会加载此文件中的 org.springframework.boot.context.event.EventPublishingRunListener 并在 getSpringFactoriesInstances 方法中进行实例化。

通过前面的类图可以看出，EventPublishingRunListener 就是事件广播器的包装类。具体广播过程稍后介绍。

到这里好像还没跟 SpringApplication 的 Listeners 关联上，不要着急，马上就到了：

```
public EventPublishingRunListener(SpringApplication application, String[] args) {
		this.application = application;
		this.args = args;
		this.initialMulticaster = new SimpleApplicationEventMulticaster();
		for (ApplicationListener<?> listener : application.getListeners()) {
			this.initialMulticaster.addApplicationListener(listener);
		}
}

```

看到了吧，EventPublishingRunListener 的构造函数里，首先创建一个事件广播器对象，然后再将 SpringApplication 的 listeners 全部添加到 initialMulticaster 里。

至此，监听器的初始化就完成了，接下来就看事件是怎么广播出去的。

监听器的初始化完成以后，就做好了接受事件的准备，那这些事件是怎么广播出去的呢，咱们接着看：

这里拿 ApplicationStartedEvent 举例（其他事件的广播过程都一样）：

首先，在 SpringApplication 的 run() 方法里：

```
listeners.starting();

```

这个方法里就是遍历所有 SpringApplicationRunListener 接口的实现类（EventPublishingRunListener）并调用它们的 starting 方法：

```
public void starting() {
		for (SpringApplicationRunListener listener : this.listeners) {
			listener.starting();
		}
}

```

在 EventPublishingRunListener 里，starting 方法将会通过事件广播器广播 ApplicationStartedEvent 事件：

```
@Override
	@SuppressWarnings("deprecation")
	public void starting() {
		this.initialMulticaster
				.multicastEvent(new ApplicationStartedEvent(this.application, this.args));
	}

```

广播之后，根据事件类型反向调用对应监听器的 onApplicationEvent 方法。

```
    @Override
	public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			Executor executor = getTaskExecutor();
			if (executor != null) {
				executor.execute(new Runnable() {
					@Override
					public void run() {
						invokeListener(listener, event);
					}
				});
			}
			else {
				invokeListener(listener, event);
			}
		}
	}

```

```
    protected void invokeListener(ApplicationListener listener, ApplicationEvent event) {
		ErrorHandler errorHandler = getErrorHandler();
		if (errorHandler != null) {
			try {
				listener.onApplicationEvent(event);
			}
			catch (Throwable err) {
				errorHandler.handleError(err);
			}
		}
		else {
			try {
				listener.onApplicationEvent(event);
			}
			catch (ClassCastException ex) {
				String msg = ex.getMessage();
				if (msg == null || msg.startsWith(event.getClass().getName())) {
					// Possibly a lambda-defined listener which we could not resolve the generic event type for
					Log logger = LogFactory.getLog(getClass());
					if (logger.isDebugEnabled()) {
						logger.debug("Non-matching event type for listener: " + listener, ex);
					}
				}
				else {
					throw ex;
				}
			}
		}
	}


```

这个时候如果你实现了 ApplicationStartedEvent 类型的监听接口:

```
public class ApplicationStartedListener implements ApplicationListener<ApplicationStartedEvent>

```

就会触发这个实现类 onApplicationEvent 方法。其他事件类型的调用过程是一样的，不同的是监听事件类型不一样。

至此，Spring Boot 广播事件的全过程就完成了。

本文介绍了 Spring Boot 事件监听器从初始化到事件广播的全过程。个人在学习的时候，初始化的时候多花了一些时间。但这个过程非常有用，对后续其他部分的理解都非常有帮助。主要还是要沉下心来仔细看看，有个好工具就是 debug。

注：本文的目的是介绍事件和监听器的原理，并没有介绍到底有多少监听器。感兴趣的同学可以查阅 Spring Boot 的 docs。