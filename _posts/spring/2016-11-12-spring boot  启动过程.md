---
layout: article
title: spring boot 启动过程
excerpt_separator: <!--more-->
---
简要的梳理了spring boot启动过程。
<!--more-->
# spring boot
## 启动
spring boot 的启动代码很简单，最精简的代码如下。

```
@Configuration
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
   }
}
```

**注意：spring启动的main方法必须包含在包内**

启动代码很简单，但在初始化过程中，spring做了很多工作，打开run方法，我们可以看到如下代码

```
public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
    return new SpringApplication(sources).run(args);
}
```

大致分为两部分。
1. 加载配置信息
2. 启动

## 加载配置信息
初始化的主要工作都在这个方法中完成。

```
private void initialize(Object[] sources) {
    if (sources != null && sources.length > 0) {
        this.sources.addAll(Arrays.asList(sources));
    }
    this.webEnvironment = deduceWebEnvironment();
    setInitializers((Collection) getSpringFactoriesInstances(
            ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

通过判断是否包含WEB_ENVIRONMENT_CLASSES中定义的类来决定是否是web项目。

我们应用的是compile("org.springframework.boot:spring-boot-starter-web")， 所以间接引用了org.springframework.web.context.ConfigurableWebApplicationContext，推论为web环境。

如果只引用org.springframework.boot:spring-boot-starter，就不是web项目，不会启动内置的tomcat。
```
private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };

private boolean deduceWebEnvironment() {
    for (String className : WEB_ENVIRONMENT_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return false;
        }
    }
    return true;
}
```

下面这段代码比较长，目的是通过META-INF/spring.factories中的org.springframework.context.ApplicationContextInitializer
读取配置并初始化，排序。

```
public void setInitializers(
        Collection<? extends ApplicationContextInitializer<?>> initializers) {
    this.initializers = new ArrayList<ApplicationContextInitializer<?>>();
    this.initializers.addAll(initializers);
}

private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

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

private <T> List<T> createSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args,
			Set<String> names) {
    List<T> instances = new ArrayList<T>(names.size());
    for (String name : names) {
        try {
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = instanceClass
                    .getDeclaredConstructor(parameterTypes);
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException(
                    "Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}


//SpringFactoriesLoader
/**
 * The location to look for factories.
 * <p>Can be present in multiple JAR files.
 */
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
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

初始化ApplicationListener的过程和ApplicationContextInitializer一样，就不在赘述了。

```
setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
```

通过调用栈获取启动main函数

```
this.mainApplicationClass = deduceMainApplicationClass();

private Class<?> deduceMainApplicationClass() {
    try {
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

上面初始化的配置信息在启动的时候会在相应的位置起作用，下面我们分解启动。


### 启动
经过上面的初始化过程，我们已经有了一个SpringApplication对象，根据SpringApplication类的静态run方法分析，
接下来会调用SpringApplication对象的run方法。我们接下来就分析这个对象的run方法。

```
public ConfigurableApplicationContext run(String... args) {
    // 启动计时器
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    FailureAnalyzers analyzers = null;
    configureHeadlessProperty();
    //
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.started();
    try {
        //创建并刷新ApplicationContext
        ApplicationArguments applicationArguments = new DefaultApplicationArguments( args);
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        // print spring boot banner and version
        Banner printedBanner = printBanner(environment);
        context = createApplicationContext();
        analyzers = new FailureAnalyzers(context);
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        refreshContext(context);
        afterRefresh(context, applicationArguments);
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

上面的方法通过名字很明确它实现的功能。

有关java.awt.headless的内容参考：
 [Using Headless Mode in the Java SE Platform](http://www.oracle.com/technetwork/articles/javase/headless-136834.html)


我们来详细的观察一下第一个事件，启动事件。

```
listeners.started();

//EventPublishingRunListener.java
public void started() {
    this.initialMulticaster
            .multicastEvent(new ApplicationStartedEvent(this.application, this.args));
}
public void multicastEvent(ApplicationEvent event) {
    multicastEvent(event, resolveDefaultEventType(event));
}
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

protected Collection<ApplicationListener<?>> getApplicationListeners(
			ApplicationEvent event, ResolvableType eventType) {

    Object source = event.getSource();
    Class<?> sourceType = (source != null ? source.getClass() : null);
    ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);

    // Quick check for existing entry on ConcurrentHashMap...
    ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
    if (retriever != null) {
        return retriever.getApplicationListeners();
    }

    if (this.beanClassLoader == null ||
            (ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
                    (sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
        // Fully synchronized building and caching of a ListenerRetriever
        synchronized (this.retrievalMutex) {
            retriever = this.retrieverCache.get(cacheKey);
            if (retriever != null) {
                return retriever.getApplicationListeners();
            }
            retriever = new ListenerRetriever(true);
            Collection<ApplicationListener<?>> listeners =
                    retrieveApplicationListeners(eventType, sourceType, retriever);
            this.retrieverCache.put(cacheKey, retriever);
            return listeners;
        }
    }
    else {
        // No ListenerRetriever caching -> no synchronization necessary
        return retrieveApplicationListeners(eventType, sourceType, null);
    }
}

private Collection<ApplicationListener<?>> retrieveApplicationListeners(
			ResolvableType eventType, Class<?> sourceType, ListenerRetriever retriever) {

    LinkedList<ApplicationListener<?>> allListeners = new LinkedList<ApplicationListener<?>>();
    Set<ApplicationListener<?>> listeners;
    Set<String> listenerBeans;
    synchronized (this.retrievalMutex) {
        listeners = new LinkedHashSet<ApplicationListener<?>>(this.defaultRetriever.applicationListeners);
        listenerBeans = new LinkedHashSet<String>(this.defaultRetriever.applicationListenerBeans);
    }
    for (ApplicationListener<?> listener : listeners) {
        if (supportsEvent(listener, eventType, sourceType)) {
            if (retriever != null) {
                retriever.applicationListeners.add(listener);
            }
            allListeners.add(listener);
        }
    }
    if (!listenerBeans.isEmpty()) {
        BeanFactory beanFactory = getBeanFactory();
        for (String listenerBeanName : listenerBeans) {
            try {
                Class<?> listenerType = beanFactory.getType(listenerBeanName);
                if (listenerType == null || supportsEvent(listenerType, eventType)) {
                    ApplicationListener<?> listener =
                            beanFactory.getBean(listenerBeanName, ApplicationListener.class);
                    if (!allListeners.contains(listener) && supportsEvent(listener, eventType, sourceType)) {
                        if (retriever != null) {
                            retriever.applicationListenerBeans.add(listenerBeanName);
                        }
                        allListeners.add(listener);
                    }
                }
            }
            catch (NoSuchBeanDefinitionException ex) {
                // Singleton listener instance (without backing bean definition) disappeared -
                // probably in the middle of the destruction phase
            }
        }
    }
    AnnotationAwareOrderComparator.sort(allListeners);
    return allListeners;
}

protected boolean supportsEvent(ApplicationListener<?> listener, ResolvableType eventType, Class<?> sourceType) {
		GenericApplicationListener smartListener = (listener instanceof GenericApplicationListener ?
            (GenericApplicationListener) listener : new GenericApplicationListenerAdapter(listener));
    return (smartListener.supportsEventType(eventType) && smartListener.supportsSourceType(sourceType));
}
```

方法比较多，我们一点一点的分析。

首先我们注意到事件被封装成了 ResolvableType 类，在通过 getApplicationListeners 方法过滤支持此事件的listener（初始化时加载的），
当中还增加了缓存处理。

在继续往下跟踪，发现最后是通过 supportsEvent 方法判断是否支持事件，继承关系如下。

```
public interface GenericApplicationListener extends ApplicationListener<ApplicationEvent>, Ordered
ApplicationListener<E extends ApplicationEvent>
```

继承自 GenericApplicationListener 的 GenericApplicationListenerAdapter是一个适配器，
他代理真正的 Listener，并实现了扩展方法来判断是否支持此事件。如下图：
![](https://github.com/whyDK37/note/blob/master/_posts/spring/uml/event-listener.png?raw=true)

*这里的判断是个亮点！* 首先在此看 ResolvableType ，他是4.0新加入的类。从doc上可以看出，它是一个工具类，能够匹配父类，接口，甚至泛型，功能还是相当强大的。
事件的判断全靠它来完成关键的判断。我们看两个初始化时加载的 Listener：

- 0 = {ConfigFileApplicationListener@2018}
public class ConfigFileApplicationListener implements EnvironmentPostProcessor, ApplicationListener<ApplicationEvent>, Ordered
- 1 = {AnsiOutputApplicationListener@1322}
public class AnsiOutputApplicationListener implements ApplicationListener<ApplicationEnvironmentPreparedEvent>, Ordered
- 2 = {LoggingApplicationListener@1323}
public class LoggingApplicationListener implements GenericApplicationListener {
- 3 = {ClasspathLoggingApplicationListener@2019}
public final class ClasspathLoggingApplicationListener implements GenericApplicationListener {
- 4 = {BackgroundPreinitializer@2020}
public class BackgroundPreinitializer implements ApplicationListener<ApplicationEnvironmentPreparedEvent> {
- 5 = {DelegatingApplicationListener@2021}
public class DelegatingApplicationListener implements ApplicationListener<ApplicationEvent>, Ordered {
- 6 = {ParentContextCloserApplicationListener@1327}
public class ParentContextCloserApplicationListener implements ApplicationListener<ParentContextAvailableEvent>, ApplicationContextAware, Ordered {
- 7 = {ClearCachesApplicationListener@2022}
class ClearCachesApplicationListener implements ApplicationListener<ContextRefreshedEvent> {
- 8 = {FileEncodingApplicationListener@2023}
public class FileEncodingApplicationListener implements ApplicationListener<ApplicationEnvironmentPreparedEvent>, Ordered {
- 9 = {LiquibaseServiceLocatorApplicationListener@2024}
public class LiquibaseServiceLocatorApplicationListener implements ApplicationListener<ApplicationStartedEvent> {

从类声明可以看出Event泛型是不一样的，这就决定了该Listener支持什么事件。
启动时的事件是 ApplicationStartedEvent ，根据 Listener 可以很清楚 ApplicationStartedEvent，ApplicationEvent 会被调用。
其他的的事件也是相同原理。

接下来是配置，由下面代码方法实现。

```
ApplicationArguments applicationArguments = new DefaultApplicationArguments( args);
ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);

private ConfigurableEnvironment prepareEnvironment( SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
    // Create and configure the environment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    listeners.environmentPrepared(environment);
    if (isWebEnvironment(environment) && !this.webEnvironment) {
        environment = convertToStandardEnvironment(environment);
    }
    return environment;
}

// configureEnvironment 方法先是调用configurePropertySources来配置properties，
// 然后调用 configureProfiles 来配置profiles
protected void configureEnvironment(ConfigurableEnvironment environment,
        String[] args) {
    configurePropertySources(environment, args);
    configureProfiles(environment, args);
}

protected void configurePropertySources(ConfigurableEnvironment environment,
        String[] args) {
    MutablePropertySources sources = environment.getPropertySources();
    if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
        sources.addLast(
                new MapPropertySource("defaultProperties", this.defaultProperties));
    }
    if (this.addCommandLineProperties && args.length > 0) {
        String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
        if (sources.contains(name)) {
            PropertySource<?> source = sources.get(name);
            CompositePropertySource composite = new CompositePropertySource(name);
            composite.addPropertySource(new SimpleCommandLinePropertySource(
                    name + "-" + args.hashCode(), args));
            composite.addPropertySource(source);
            sources.replace(name, composite);
        }
        else {
            sources.addFirst(new SimpleCommandLinePropertySource(args));
        }
    }
}

protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
    environment.getActiveProfiles(); // ensure they are initialized
    // But these ones should go first (last wins in a property key clash)
    Set<String> profiles = new LinkedHashSet<String>(this.additionalProfiles);
    profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
    environment.setActiveProfiles(profiles.toArray(new String[profiles.size()]));
}
```

listeners.environmentPrepared(environment); 广播 ApplicationEnvironmentPreparedEvent 事件。

#### 打印banner
打印的内容是spring图形和spring boot的版本。就不详细说明了，有兴趣的可以自行浏览。

#### createApplicationContext
根据 webEnvironment 创建 DEFAULT_WEB_CONTEXT_CLASS 或 DEFAULT_CONTEXT_CLASS。

```
/**
* The class name of application context that will be used by default for non-web
* environments.
*/
public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
+ "annotation.AnnotationConfigApplicationContext";

/**
* The class name of application context that will be used by default for web
* environments.
*/
public static final String DEFAULT_WEB_CONTEXT_CLASS = "org.springframework."
+ "boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext";
```

#### refreshContext
真正的启动现在才开始。我们看到最终调用的是refresh方法。

在查看代码之前，先说明一下当前 Context 对象的继承关系，请看下图：

![](https://github.com/whyDK37/note/blob/master/_posts/spring/uml/AnnotationConfigEmbeddedWebApplicationContext.png?raw=true)

上面的分析已经得知当前启动的是一个web项目，
所以在createApplicationContext()方法中创建的ApplicationContext是DEFAULT_WEB_CONTEXT_CLASS，也就是AnnotationConfigEmbeddedWebApplicationContext。
从图中我们可以得知 AnnotationConfigEmbeddedWebApplicationContext 的继承关系。由于继承自 EmbeddedWebApplicationContext ，
赋予了它内嵌 servlet container 的能力。启动容器的代码如下：

```
@Override
protected void onRefresh() {
    super.onRefresh();
    try {
        createEmbeddedServletContainer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start embedded container",
                ex);
    }
}
```

启动过程来自模板类 AbstractApplicationContext，和spring mvc 是一样的，这里就不过多解释了。

```
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
            // 初始化并启动 Servlet Container
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

启动完成后，会广播事件，停止计时，整个启动过程结束。

```
afterRefresh(context, applicationArguments);
listeners.finished(context, null);
stopWatch.stop();
```