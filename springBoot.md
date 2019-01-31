# springBoot启动流程
![avatar](\src\main\resources\static\images\springapplicationflow.png)
```java
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
* 第一步，初始化监听器

    * 这里会初始化Spring Boot自带的监听器，以及添加到SpringApplication的自定义监听器。
* 第二步，发布ApplicationStartingEvent事件
    * 到这一步，Spring Boot会发布一个ApplicationStartingEvent事件。如果你想在这个时候执行一些代码可以通过实现ApplicationListener接口实现；
    * 下面是ApplicationListener接口的定义，注意这里有个<E extends ApplicationEvent>
    > public interface ApplicationListener<E extends ApplicationEvent> extends EventListener
    * 例如，你想监听ApplicationStartingEvent事件，你可以这样定义：
    > public class ApplicationStartingListener implements ApplicationListener<ApplicationStartingEvent>

    * 然后通过SpringApplication.addListener(..)添加进去即可
* 第三步，装配参数和环境

    * 在这一步，首先会初始化参数，然后装配环境，确定是web环境还是非web环境
* 第四步，发布ApplicationEnvironmentPreparedEvent事件

    * 准确的说，这个应该属于第三步，在装配完环境后，就触发ApplicationEnvironmentPreparedEvent事件。如果想在这个时候执行一些代码，可以订阅这个事件的监听器，方法同第二步
* 第五步，打印Banner

    * 看过Spring Boot实例教程 - 自定义Banner的同学会很熟悉，启动的Banner就是在这一步打印出来的
* 第六步，创建ApplicationContext

    * 这里会根据是否是web环境，来决定创建什么类型的ApplicationContext,ApplicationContext不要多说了吧，不知道ApplicationContext是啥的同学，出门左转补下Spring基础知识吧。
* 第七步，装配Context

    * 这里会设置Context的环境变量、注册Initializers、beanNameGenerator等
* 第八步，发布ApplicationPreparedEvent事件
    * 这里放在第七步会更准确，因为这个是在装配Context的时候发布的。

    * 值得注意的是：这里是假的，假的，假的，源码中是空的，并没有真正发布ApplicationPreparedEvent事件。不知道作者这么想的
* 第九步，注册、加载等

    * 注册springApplicationArguments、springBootBanner，加载资源等
* 第十步，发布ApplicationPreparedEvent事件

    * 注意，到这里才是真正发布了ApplicationPreparedEvent事件。这里和第八步好让人误解
* 第十一步，refreshContext

    * 装配context beanfactory等非常重要的核心组件
* 第十二步，afterRefreshContext

    * 这里会调用自定义的Runners，不知道Runners是什么的同学，请参考Spring Boot官方文档 - SpringApplication

* 第十三步，发布ApplicationStartedEvent事件

* 第十四步，发布ApplicationReadyEvent事件
    * 最后一步，发布ApplicationReadyEvent事件，启动完毕，表示服务已经可以开始正常提供服务了。通常我们这里会监听这个事件来打印一些监控性质的日志，表示应用正常启动了。添加方法同第二步。

    * 注意：如果启动失败，这一步会发布ApplicationFailedEvent事件。

    * 到这里，Spring Boot启动的一些关键动作就介绍完了

# SpringApplicationRunListener And ApplicationListener
## SpringApplicationRunListener
* 监听spring boot应用生命周期事件
* 可以做一些参数设置,应用程序事件发布等处理
* 需在spring.factories文件中配置,启动时SpringFactoriesLoader会进行扫描加载对应的bean
```java
org.springframework.boot.SpringApplicationRunListener=\
com.alibaba.nacos.core.listener.LoggingSpringApplicationRunListener
```
```java
public class LoggingSpringApplicationRunListener implements SpringApplicationRunListener, Ordered{}
```
## ApplicationListener
* 监听应用程序事件,可以监听指定类型的事件
```java
@Component
public class MyListener 
        implements ApplicationListener<ContextRefreshedEvent> {
  
    public void onApplicationEvent(ContextRefreshedEvent event) {
        ...
    }
}
```
```java
@Component
public class MyListener {
  
    @EventListener
    public void handleContextRefresh(ContextRefreshedEvent event) {
        ...
    }
}
```
# EnvironmentPostProcessor
* 动态管理配置文件扩展接口,需要在spring.factories中进行配置,SpringFactoriesLoader会扫描并加载bean
```java
org.springframework.boot.env.EnvironmentPostProcessor=\
com.alibaba.nacos.core.env.NacosDefaultPropertySourceEnvironmentPostProcessor

@Component
public class MyEnvironmentPostProcessor implements EnvironmentPostProcessor {

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        try{
            InputStream inputStream = new FileInputStream("/Users/naeshihiroshi/study/studySummarize/SpringBoot/springboot.properties");
            Properties properties = new Properties();
            properties.load(inputStream);
            PropertiesPropertySource propertiesPropertySource = new PropertiesPropertySource("my",properties);
            environment.getPropertySources().addLast(propertiesPropertySource);
        }catch (IOException e){
            e.printStackTrace();
        }
    }
}
```
* 在spring boot发布ApplicationEnvironmentPreparedEvent环境配置准备事件后,进行初始化扩展的环境配置,在spring上下文构建之前进行设置

## CommandLineRunner接口和ApplicationRunner接口
* 在spring boot启动流程的第十二步，afterRefreshContext
* 此扩展点可以在容器启动完成后执行一些内容。比如读取配置文件，数据库连接之类的
```java
@Component
public class TestImplCommandLineRunner implements CommandLineRunner {
 
 
    @Override
    public void run(String... args) throws Exception {
 
        System.out.println("<<<<<<<<<<<<这个是测试CommandLineRunn接口>>>>>>>>>>>>>>");
    }
}
@Component
public class TestImplApplicationRunner implements ApplicationRunner {
 
 
    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println(args);
        System.out.println("这个是测试ApplicationRunner接口");
    }
}
```
