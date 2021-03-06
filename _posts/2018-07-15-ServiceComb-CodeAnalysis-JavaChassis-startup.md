---
layout: post
title: "ServiceComb代码分析——Java Chassis启动流程"
date: 2018-07-15 
tags: ServiceComb 代码分析
---

ServiceComb是华为开源的微服务框架，目前已发布第一个Apache孵化版本，可点击 [官网](http://servicecomb.incubator.apache.org/) 查看。       
主要模块有Java Chassis、服务中心和Saga，下文是对Java Chassis 的代码走读和分析。       
该系列博文均是基于1.0.0-mX版本的代码进行分析，可点击 [代码](https://github.com/apache/incubator-servicecomb-java-chassis/tree/1.0.0-mX) 查看。
 

### 概述       

从 [Java Chassis官方文档](https://docs.servicecomb.io/java-chassis/zh_CN/) 可以了解到，Java Chassis其实是一个用于快速构建微服务的JAVA SDK。       
如下图1，我们可以将其4大模块理解为，【编程模型】将代码抽象为【服务契约】，以不同的【通信模型】提供带有【服务治理】功能的服务。
<div style="text-align:center"><img src="/images/posts/ServiceComb-CodeAnalysis-JavaChassis-startup/architecture.png" alt="Java Chassis架构" /><p>图1</p></div>  
话不多说，接下来我们要以一个 [官方demo](https://github.com/apache/incubator-servicecomb-java-chassis/tree/1.0.0-mX/demo/demo-pojo) 为入口，走读分析其代码。


### Server程序启动入口       

这个demo由3个模块构成，pojo-server、pojo-client和pojo-tests。按照惯例，我们先从server开始。       
启动server的main函数如下所示，只有两行。不错，很spring cloud。

```     
public class PojoServer {
  public static void main(String[] args) throws Exception {
    Log4jUtils.init();
    BeanUtils.init();
  }
}
``` 
Log4jUtils.init()和BeanUtils.init()先后完成了日志和服务的初始化。       
【注意】必须按照这个顺序进行初始化。


### 日志初始化过程分析       

略


### 服务初始化过程分析       

1、实例化上下文对象       
BeanUtils.init()方法会实例化spring的ClassPathXmlApplicationContext对象，并加载所有jar包中以【.bean.xml】结尾的文件。
<div style="text-align:center"><img src="/images/posts/ServiceComb-CodeAnalysis-JavaChassis-startup/BeansUtils.png" alt="BeansUtils" /><p>图2</p></div>  
如果不了解ClassPathXmlApplicationContext的具体作用，可以Google。

2、spring容器触发事件  
上一步会导致 org.apache.servicecomb.core.CseApplicationListener 的 setApplicationContext 和 onApplicationEvent 方法被调用。而这两个方法完成了微服务的初始化和注册。

那它们是如何被触发的呢？  

1）首先，Java Chassis的core模块的cse.bean.xml中定义了CseApplicationListener，如下。  
```     
  <bean class="org.apache.servicecomb.core.CseApplicationListener">
  </bean>
``` 
2）同时，CseApplicationListener 实现了spring的 ApplicationListener\<ApplicationEvent\> 接口，如下。  
```     
public class CseApplicationListener
    implements ApplicationListener<ApplicationEvent>, Ordered, ApplicationContextAware {
    	……
}
``` 
因此，CseApplicationListener 会通过 onApplicationEvent 方法来监听 ApplicationContext 中的事件。  
至于 ApplicationContext 和 ApplicationListener 的关系，可以Google。

3、CseApplicationListener的setApplicationContext方法分析  
```     
  public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
    // 获取上下文对象，为初始化做准备
    this.applicationContext = applicationContext;
    BeanUtils.setContext(applicationContext);
    // 注册方法初始化，为向服务中心注册做准备
    RegistryUtils.init();
  }
``` 

4、 CseApplicationListener的onApplicationEvent方法分析  
从代码可以发现，listener一旦监听到ContextRefreshedEvent事件，则会触发初始化流程。  
```     
  public void onApplicationEvent(ApplicationEvent event) {
    // initEventClass就是ContextRefreshedEvent.class
    if (initEventClass.isInstance(event)) {
      if (applicationContext instanceof AbstractApplicationContext) {
      	// TODO
        ((AbstractApplicationContext) applicationContext).registerShutdownHook();
      }

      // 设置服务初始化相关的对象
      if (SCBEngine.getInstance().getBootListenerList() == null) {
        SCBEngine.getInstance().setProducerProviderManager(applicationContext.getBean(ProducerProviderManager.class));
        SCBEngine.getInstance().setConsumerProviderManager(applicationContext.getBean(ConsumerProviderManager.class));
        SCBEngine.getInstance().setTransportManager(applicationContext.getBean(TransportManager.class));
        SCBEngine.getInstance().setSchemaListenerManager(applicationContext.getBean(SchemaListenerManager.class));
        // 取出所有实现了BootListener接口的bean：ProducerProviderManager、RestEngineSchemaListener、MonitorBootListener（待确认来源）
        SCBEngine.getInstance().setBootListenerList(applicationContext.getBeansOfType(BootListener.class).values());
      }

      // 调用init方法，完成初始化
      SCBEngine.getInstance().init();
    } else if (event instanceof ContextClosedEvent) {
      SCBEngine.getInstance().destroy();
    }
  }
``` 
从上面的代码可知，是由SCBEngine来完成服务的初始化。


### SCBEngine初始化代码分析       

走读SCBEngine的代码，可以发现是由doInit()方法实现了初始化的动作。  
```     
  private void doInit() throws Exception {
    status = SCBStatus.STARTING;

    // 往eventBus注册本对象，用于监听事件
    eventBus.register(this);

    consumerProviderManager.setAppManager(RegistryUtils.getServiceRegistry().getAppManager());
    // TODO
    AbstractEndpointsCache.init(RegistryUtils.getInstanceCacheManager(), transportManager);

    // 将事件通知给所有的BootListener（ProducerProviderManager、RestEngineSchemaListener、MonitorBootListener）
    triggerEvent(EventType.BEFORE_HANDLER);
    // 加载所有的classpath*:config/cse.handler.xml，并存入ConsumerHandlerManager和ProducerHandlerManager。
    HandlerConfigUtils.init();
    triggerEvent(EventType.AFTER_HANDLER);

    triggerEvent(EventType.BEFORE_PRODUCER_PROVIDER);
    // spring自动注入ProducerProvider接口的所有实现类：PojoProducerProvider和RestProducerProvider，并初始化。
    producerProviderManager.init();
    triggerEvent(EventType.AFTER_PRODUCER_PROVIDER);

    triggerEvent(EventType.BEFORE_CONSUMER_PROVIDER);
    // spring自动注入ConsumerProvider接口的所有实现类：PojoConsumerProvider和RestConsumerProvider，并初始化。
    consumerProviderManager.init();
    triggerEvent(EventType.AFTER_CONSUMER_PROVIDER);

    triggerEvent(EventType.BEFORE_TRANSPORT);
    // spring自动注入Transport接口所有实现类：VertxRestTransport、ServletRestTransport和HighwayTransport，并初始化。
    // 初始化内容：
    // 1、从Vertx和Servlet中选一个作为rest的通信方式，默认是Vertx；
    // 2、将需要监听的ip和端口（endpoint）存入Microservice实例；
    // 3、按配置初始化各transport，拉起端口监听。
    transportManager.init();
    triggerEvent(EventType.AFTER_TRANSPORT);

    // 将各服务的契约信息（SchemaMata）取出，将契约信息加载到契约管理中。完成这步操作后，服务才能被正常访问。
    schemaListenerManager.notifySchemaListener();

    // TODO RestEngineSchemaListener的onBootEvent会被触发。
    triggerEvent(EventType.BEFORE_REGISTRY);

    // TODO
    triggerAfterRegistryEvent();

    // 调用RemoteServiceRegistry的run方法，启动心跳、注册、watch等定时任务。
    RegistryUtils.run();

    // 添加优雅关闭的方法。
    // destory方法：1、将自身的服务实例信息在服务中心中unregistry掉；
    // 2、确保所有请求处理完；
    // 3、destroy掉配置中心上的相关配置；
    // 4、通知BootListener相关的事件。
    Runtime.getRuntime().addShutdownHook(new Thread(this::destroy));
  }
``` 

由此我们可以先简单地将这些代码与各模块对应起来。  
<div style="text-align:center"><img src="/images/posts/ServiceComb-CodeAnalysis-JavaChassis-startup/architecture&code.png" alt="architecture&code" /><p>图3</p></div>  
初始化流程代码与模块的对应关系如上图；至于各模块的服务流程，后续博文再进行分析。  
而这几个模块的关系，我们可以这样理解。红框1和2中，将使用不同编程模式的代码抽象成服务契约；  
服务契约和红框3的服务治理模块，在与红框4的通信模型绑定后，就可向外部提供服务；  
服务的访问与被访问，都会经过3的服务治理模块。  


### HandlerConfigUtils初始化代码分析       

如图3所示，初始化过程中HandlerConfigUtils的init方法会被调用，用于加载servicecomb自带或者自己开发的handler。至于handler的作用，以后再说。  
```     
public final class HandlerConfigUtils {
  private HandlerConfigUtils() {
  }

  private static Config loadConfig() throws Exception {
    Config config = new Config();

    List<Resource> resList =
        PaaSResourceUtils.getSortedResources("classpath*:config/cse.handler.xml", ".handler.xml");
    for (Resource res : resList) {
      Config tmpConfig = XmlLoaderUtils.load(res, Config.class);
      config.mergeFrom(tmpConfig);
    }

    return config;
  }

  public static void init() throws Exception {
    // 1、加载所有cse.handler.xml中的配置，配置内容一般如下所示：
    // <config>
    //   <handler id="simpleLB"
    //     class="org.apache.servicecomb.core.handler.impl.SimpleLoadBalanceHandler"/>
    // </config>
    Config config = loadConfig();
    // 2、将config存入两个对象中。两个类都是继承AbstractHandlerManager，因此调用的都是同一个方法。
    ConsumerHandlerManager.INSTANCE.init(config);
    ProducerHandlerManager.INSTANCE.init(config);
  }
}
``` 

>* 1、Java Chassis自带的cse.handler.xml中，共有6个handler：  
"qps-flowcontrol-consumer" -> "class org.apache.servicecomb.qps.ConsumerQpsFlowControlHandler"  
"qps-flowcontrol-provider" -> "class org.apache.servicecomb.qps.ProviderQpsFlowControlHandler"  
"bizkeeper-consumer" -> "class org.apache.servicecomb.bizkeeper.ConsumerBizkeeperHandler"  
"loadbalance" -> "class org.apache.servicecomb.loadbalance.LoadbalanceHandler"  
"simpleLB" -> "class org.apache.servicecomb.core.handler.impl.SimpleLoadBalanceHandler"  
"bizkeeper-provider" -> "class org.apache.servicecomb.bizkeeper.ProviderBizkeeperHanlder"  
>* 2、将配置存入服务端和消费端，后续服务调用的场景将会用到。  


### ProviderManager初始化代码分析       

前面有提到producerProviderManager和consumerProviderManager的初始化操作。  
这两个类的init()方法，主要作用是：  
将整个微服务实例的信息与具体的的业务方法，抽象成microservice->scheme->operation这样一个树状结构的服务信息。  
通过microserviceId、schemeId和operationId，就能唯一确认一个业务方法。这样就能在服务调用中快速找到需要被调用的方法。  
下面是ProducerProviderManager的部分代码：  
```     
@Component
public class ProducerProviderManager implements BootListener {
  private static final Logger LOGGER = LoggerFactory.getLogger(ProducerProviderManager.class);

  // TODO spring自动注入ProducerProvider接口的所有实现类：PojoProducerProvider和RestProducerProvider
  @Autowired(required = false)
  private List<ProducerProvider> producerProviderList = Collections.emptyList();

  @Inject
  private MicroserviceMetaManager microserviceMetaManager;

  private MicroserviceMeta microserviceMeta;

  public void init() throws Exception {
    for (ProducerProvider provider : producerProviderList) {
      // 调用PojoProducerProvider和RestProducerProvider的init方法，如前面所说，完成微服务实例信息的抽象
      provider.init();
    }

    Microservice microservice = RegistryUtils.getMicroservice();
    microserviceMeta = microserviceMetaManager.getOrCreateMicroserviceMeta(microservice);
    for (SchemaMeta schemaMeta : microserviceMeta.getSchemaMetas()) {
      String content = SchemaUtils.swaggerToString(schemaMeta.getSwagger());
      microservice.addSchema(schemaMeta.getSchemaId(), content);
    }
  }

  // 其他方法略……
}
``` 

而ConsumerProviderManager的初始化代码则什么都没做。  
与producer类似，ConsumerProviderManager自动注入ConsumerProvider接口的所有实现类：PojoConsumerProvider和RestConsumerProvider，而后调用两者的init方法。  
PojoConsumerProvider和RestConsumerProvider都继承了AbstractConsumerProvider类，但两者都没有实现自己的init方法。而父类的init方法是空的。（摊手  

### transportManager通信协议初始化代码分析       

transport指的就是Java Chassis提供的通信协议，有两种：rest和highway。  
在分析这段代码前，先看一下pojo demo的microservice.yaml文件。  
```     
APPLICATION_ID: pojotest
service_description:
  name: pojo
  version: 0.0.4
servicecomb:
  service:
    registry:
      address: http://127.0.0.1:30100
  rest:
    address: 0.0.0.0:8080?protocol=http2
  highway:
    address: 0.0.0.0:7070
``` 
我们可以看到，yaml文件配置了rest和highway两种通信方式，端口则分别是8080和7070。所以，服务启动后，会监听这两个端口并对外提供服务。  

其实，通信协议初始化的操作只有2步：
>* 1、读取用户配置；  
>* 2、将配置好的协议启动，并拉起监听。 

具体细节，容我们一起看看TransportManager的代码。  
```     
@Component
public class TransportManager {
  private static final Logger LOGGER = LoggerFactory.getLogger(TransportManager.class);

  // spring注入3个实现了transport接口的类：
  // rest：VertxRestTransport、ServletRestTransport
  // highway：HighwayTransport
  @Inject
  private List<Transport> transports;

  private Map<String, Transport> transportMap = new HashMap<>();

  public void setTransports(List<Transport> transports) {
    this.transports = transports;
  }

  public void init() throws Exception {
    // 构造一个通信协议的map，key为协议名，如："rest","highway"，value为transport对象。
    // 从两个rest的transport类中，选择一个作为rest的通信协议。
    // 例："rest" -> "VertxRestTransport", "highway" -> "HighwayTransport"
    // 更多细节请查看该函数代码注释。
    buildTransportMap();

    for (Transport transport : transportMap.values()) {
      // 初始化通信协议，拉起服务监听（细节参考下文的链接）
      if (transport.init()) {
        // 服务监听成功后，将endpoint（即ip和端口）存入微服务对象中
        Endpoint endpoint = transport.getPublishEndpoint();
        if (endpoint != null && endpoint.getEndpoint() != null) {
          LOGGER.info("endpoint to publish: {}", endpoint.getEndpoint());
          Microservice microservice = RegistryUtils.getMicroservice();
          microservice.getInstance().getEndpoints().add(endpoint.getEndpoint());
        }
        continue;
      }
    }
  }

  protected void buildTransportMap() {
    // 生成一个这样的map：
    // "rest" -> "VertxRestTransport, ServletRestTransport"
    // "highway" -> "HighwayTransport"
    Map<String, List<Transport>> groups = groupByName();

    // 从rest的两个transport类中选择一个作为rest的通信协议，而highway只有一个
    for (Entry<String, List<Transport>> entry : groups.entrySet()) {
      List<Transport> group = entry.getValue();

      // 确保list中transport的优先级配置不相同
      checkTransportGroup(group);
      // 从transport list中选择一个作为通信协议，rest二选一，highway只有一个
      Transport transport = chooseOneTransport(group);
      // 生成我们需要的map
      transportMap.put(transport.getName(), transport);
    }
  }

  protected Transport chooseOneTransport(List<Transport> group) {
    // 将list中的transport对象，以getOrder方法的返回值为关键字，升序排列。
    // rest：VertxRestTransport的getOrder方法返回值是-1000，而ServletRestTransport则是默认值0，因此，排序后的list是：VertxRestTransport, ServletRestTransport
    group.sort(Comparator.comparingInt(Transport::getOrder));

    // 按顺序遍历list，无配置或配置的ip和端口合法，则使用该transport。
    // 因此，rest的默认transport是VertxRestTransport。
    for (Transport transport : group) {
      if (transport.canInit()) {
        LOGGER.info("choose {} for {}.", transport.getClass().getName(), transport.getName());
        return transport;
      }
    }

  // 其他方法略...
}
```  

从上面的代码可以看到，rest有两个transport类而highway只有一个。  
至于两个rest通信协议的区别和使用场景，可以查看其官方文档 [通信协议](https://docs.servicecomb.io/java-chassis/zh_CN/build-provider/protocol.html) 。  
各transport的服务端启动流程，可参考这篇博文 [服务端的启动](https://nevilleyeung.github.io/2018/08/ServiceComb-CodeAnalysis-Transport-startup/) 。



### schemaListenerManager初始化代码分析       

跟踪schemaListenerManager.notifySchemaListener()的代码，可以发现它最后会调用SchemaListener接口的onSchemaLoaded方法。  
该接口会将契约信息加载到契约管理中，当服务接口被调用时，则可从契约管理中找到对应的契约，从而找到对应的接口。  
```     
  public void notifySchemaListener(SchemaMeta... schemaMetas) {
    for (SchemaListener listener : schemaListenerList) {
      listener.onSchemaLoaded(schemaMetas);
    }
  }
```  
而实现了SchemaListener接口的类只有一个，便是RestEngineSchemaListener了，如下。  
```     
  @Override
  public void onSchemaLoaded(SchemaMeta... schemaMetas) {
    // 此时相应的ServicePathManager可能正在被使用，为避免太高的复杂度，使用copy on write逻辑
    Map<String, ServicePathManager> mgrMap = new HashMap<>();
    for (SchemaMeta schemaMeta : schemaMetas) {
      MicroserviceMeta microserviceMeta = schemaMeta.getMicroserviceMeta();
      ServicePathManager mgr = findPathManager(mgrMap, microserviceMeta);
      // 将契约路径加载到ServicePathManager中
      // 消费端发出请求时会先从ServicePathManager中查找
      mgr.addSchema(schemaMeta);
    }

    for (ServicePathManager mgr : mgrMap.values()) {
      // 对具有动态path operation进行排序
      mgr.sortPath();

      mgr.saveToMicroserviceMeta();
    }
  }
```  



### RegistryUtils服务注册与心跳定时任务初始化代码分析       

RegistryUtils的run方法，最终会调用到RemoteServiceRegistry类的run方法，在线程池中分配两个定时任务。  

```     
public class RemoteServiceRegistry extends AbstractServiceRegistry {

  // 其它代码略

  @Override
  public void init() {
    super.init();
    taskPool = new ScheduledThreadPoolExecutor(2, new ThreadFactory() {
      @Override
      public Thread newThread(Runnable task) {
        return new Thread(task, "Service Center Task");
      }
    }, new RejectedExecutionHandler() {
      @Override
      public void rejectedExecution(Runnable task, ThreadPoolExecutor executor) {
        LOGGER.warn("Too many pending tasks, reject " + task.getClass().getName());
      }
    });
  }

  // 其它代码略

  @Override
  public void run() {
    super.run();

    // 启动与服务中心相关的定时任务serviceCenterTask：
    // 服务注册、服务实例注册、watch和心跳
    taskPool.scheduleAtFixedRate(serviceCenterTask,
        serviceRegistryConfig.getHeartbeatInterval(),
        serviceRegistryConfig.getHeartbeatInterval(),
        TimeUnit.SECONDS);

    // 定时向进程内部发出PeriodicPullEvent事件，向服务中心拉取依赖的服务实例
    taskPool.scheduleAtFixedRate(
        () -> eventBus.post(new PeriodicPullEvent()),
        serviceRegistryConfig.getInstancePullInterval(),
        serviceRegistryConfig.getInstancePullInterval(),
        TimeUnit.SECONDS);
  }

  // 其它代码略
}
``` 
从上面的代码可以看出，这两个定时任务分别是serviceCenterTask和eventBus.post(new PeriodicPullEvent())。  
>* serviceCenterTask  

serviceCenterTask会调用MicroserviceServiceCenterTask的任务。而MicroserviceServiceCenterTask则会发起微服务注册、服务实例注册、watch和心跳这4个任务。  
```     
public class MicroserviceServiceCenterTask extends CompositeTask {
  public MicroserviceServiceCenterTask(EventBus eventBus, ServiceRegistryConfig serviceRegistryConfig,
      ServiceRegistryClient srClient, Microservice microservice) {
    // 微服务注册任务：将微服务信息注册到服务中心
    addTask(new MicroserviceRegisterTask(eventBus, srClient, microservice));
    // 微服务实例注册任务：将微服务实例信息注册到服务中心
    addTask(new MicroserviceInstanceRegisterTask(eventBus, serviceRegistryConfig, srClient, microservice));
    // 微服务实例watch任务：在调用其他服务的时候，会从服务中心查询其他服务的实例信息。
    // 服务中心支持使用PULL和PUSH两种模式通知实例变化。
    // PUSH，指的是服务中心往Java chassis push实例变化信息。
    // PULL，指的是Java chassis往服务中心pull实例变化信息。
    addTask(new MicroserviceWatchTask(eventBus, serviceRegistryConfig, srClient, microservice));
    // 微服务实例心跳任务：Java chassis定时向服务中心发送心跳信息
    addTask(new MicroserviceInstanceHeartbeatTask(eventBus, srClient, microservice));
  }
}
``` 

>* eventBus.post(new PeriodicPullEvent())  
定时从服务中心拉取该服务实例所依赖的服务提供端的实例信息。  


<br>

转载请注明：[蚊帐的博客](https://nevilleyeung.github.io) » [点击阅读原文](https://nevilleyeung.github.io/2018/07/ServiceComb-CodeAnalysis-JavaChassis-startup/) 

 



