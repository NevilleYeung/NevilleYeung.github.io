---
layout: post
title: "ServiceComb代码分析——服务端如何监听ip和端口"
date: 2018-08-12 
tags: ServiceComb 代码分析
---

之前的文章有分析过 [Java Chassis的启动流程](https://nevilleyeung.github.io/2018/07/ServiceComb-CodeAnalysis-JavaChassis-startup/) 。       
如前文所言，我们把ServiceComb的通信协议称为transport，有highway、rest两种。其中，rest有Vertx和Servlet。而这几种协议，都是通过vertx开源组件实现的。   
 

### VertxRestTransport的初始化       

直接开始看VertxRestTransport类的代码。  
```     
@Component
public class VertxRestTransport extends AbstractTransport {

  // 其他方法略……

  @Override
  public boolean init() throws Exception {
    restClient = RestTransportClientManager.INSTANCE.getRestClient();

    // 部署transport server
    // 从配置中读取服务端线程数，默认为1
    DeploymentOptions options = new DeploymentOptions().setInstances(TransportConfig.getThreadCount());
    SimpleJsonObject json = new SimpleJsonObject();
    // 设置服务端需要绑定的ip和端口
    json.put(ENDPOINT_KEY, getEndpoint());
    json.put(RestTransportClient.class.getName(), restClient);
    options.setConfig(json);
    // vertx会调用TransportConfig.getRestServerVerticle()取出的类进行服务端的初始化，默认是RestServerVerticle类
    return VertxUtils.blockDeploy(transportVertx, TransportConfig.getRestServerVerticle(), options);
  }

  // 其他方法略……
}
``` 
需要注意的是，RestServerVerticle继承了vertx的AbstractVerticle类。  
下面看一下RestServerVerticle的start()方法。  
```     
public class RestServerVerticle extends AbstractVerticle {

  // 其他方法略……

  @Override
  public void start(Future<Void> startFuture) throws Exception {
    try {
      super.start();
      // 如果本地未配置地址，则表示不必监听，只需要作为客户端使用即可
      if (endpointObject == null) {
        LOGGER.warn("rest listen address is not configured, will not start.");
        startFuture.complete();
        return;
      }
      // 通过vertx的router分发请求到对应的Handler（处理器）
      Router mainRouter = Router.router(vertx);
      // 如果有配置accesslog，则绑定相关的处理器
      mountAccessLogHandler(mainRouter);
      // 绑定业务请求的处理器
      initDispatcher(mainRouter);
      // 启动一个http或https服务端
      HttpServer httpServer = createHttpServer();
      httpServer.requestHandler(mainRouter::accept);
      startListen(httpServer, startFuture);
    } catch (Throwable e) {
      // vert.x got some states that not print error and execute call back in VertexUtils.blockDeploy, we add a log our self.
      LOGGER.error("", e);
      throw e;
    }
  }

  private void mountAccessLogHandler(Router mainRouter) {
    if (AccessLogConfiguration.INSTANCE.getAccessLogEnabled()) {
      String pattern = AccessLogConfiguration.INSTANCE.getAccesslogPattern();
      LOGGER.info("access log enabled, pattern = {}", pattern);
      mainRouter.route()
          .handler(new AccessLogHandler(
              pattern
          ));
    }
  }

  private void initDispatcher(Router mainRouter) {
    // 从配置文件org.apache.servicecomb.transport.rest.vertx.VertxRestDispatcher中读取到VertxRestDispatcher类的配置
    List<VertxHttpDispatcher> dispatchers = SPIServiceUtils.getSortedService(VertxHttpDispatcher.class);
    for (VertxHttpDispatcher dispatcher : dispatchers) {
      if (dispatcher.enabled()) {
        // 调用VertxRestDispatcher类的init方法，绑定接收到请求后的处理器
        dispatcher.init(mainRouter);
      }
    }
  }

  // 其他方法略……
}

``` 
当服务端接收到消费端的请求时，需要将请求分发给对应的Handler（处理器），而处理器则会找到相关的业务方法。因此需要把处理器绑定到server端。  
上文的initDispatcher()代码显示，是VertxRestDispatcher类的init()方法来完成这个操作的。  
```     
public class VertxRestDispatcher extends AbstractVertxHttpDispatcher {
  // 其他方法略……

  @Override
  public void init(Router router) {
    // 绑定cookie相关的Handler
    router.route().handler(CookieHandler.create());
    // body相关的Handler
    router.route().handler(createBodyHandler());
    // onRequest则是业务相关的Handler
    router.route().failureHandler(this::failureHandler).handler(this::onRequest);
  }

  /**
   * 服务端接收到请求后，onRequest会被调用。
   * 该方法会将请求分发给对应的业务方法，比如自己写的Hello world
   */
  private void onRequest(RoutingContext context) {
    if (transport == null) {
      transport = CseContext.getInstance().getTransportManager().findTransport(Const.RESTFUL);
    }
    HttpServletRequestEx requestEx = new VertxServerRequestToHttpServletRequest(context);
    HttpServletResponseEx responseEx = new VertxServerResponseToHttpServletResponse(context.response());

    VertxRestInvocation vertxRestInvocation = new VertxRestInvocation();
    context.put(RestConst.REST_PRODUCER_INVOCATION, vertxRestInvocation);
    vertxRestInvocation.invoke(transport, requestEx, responseEx, httpServerFilters);
  }

  // 其他方法略……
}
``` 


### ServletRestTransport的初始化       

// TODO


### HighwayTransport的初始化       

直接开始看HighwayTransport类的代码。  
```     
@Component
public class HighwayTransport extends AbstractTransport {
  public static final String NAME = "highway";

  private HighwayClient highwayClient = new HighwayClient();

  @Override
  public String getName() {
    return NAME;
  }

  public boolean init() throws Exception {
    // TODO 初始化一个tcp连接的客户端
    highwayClient.init(transportVertx);

    // 从配置中读取服务端线程数，默认为1
    DeploymentOptions deployOptions = new DeploymentOptions().setInstances(HighwayConfig.getServerThreadCount());
    // 从配置中读取服务ip和端口，构造出来的endpoint通常是这样： highway://0.0.0.0:7070?login=true
    // 至于为何会多出一个login=true，后面的highway通信的文章再写吧
    setListenAddressWithoutSchema(HighwayConfig.getAddress(), Collections.singletonMap(TcpConst.LOGIN, "true"));
    SimpleJsonObject json = new SimpleJsonObject();
    // 设置服务端需要绑定的ip和端口，即endpoint
    json.put(ENDPOINT_KEY, getEndpoint());
    deployOptions.setConfig(json);
    // vertx会调用HighwayServerVerticle类进行服务端的初始化
    return VertxUtils.blockDeploy(transportVertx, HighwayServerVerticle.class, deployOptions);
  }

  @Override
  public void send(Invocation invocation, AsyncResponse asyncResp) throws Exception {
    highwayClient.send(invocation, asyncResp);
  }
}
``` 
与前面的rest协议类似，highway协议会调用HighwayServerVerticle的start方法进行服务端的初始化。  
```     
public class HighwayServerVerticle extends AbstractVerticle {
  
  // 其它方法略

  @Override
  public void start(Future<Void> startFuture) throws Exception {
    try {
      super.start();
      // 拉起服务端口的监听
      startListen(startFuture);
    } catch (Throwable e) {
      // vert.x got some states that not print error and execute call back in VertexUtils.blockDeploy, we add a log our self.
      LOGGER.error("", e);
      throw e;
    }
  }

  protected void startListen(Future<Void> startFuture) {
    // 如果本地未配置地址，则表示不必监听，只需要作为客户端使用即可
    if (endpointObject == null) {
      LOGGER.warn("highway listen address is not configured, will not listen.");
      startFuture.complete();
      return;
    }

    // HighwayServer继承了TcpServer类，init方法是在TcpServer类中实现的
    HighwayServer server = new HighwayServer(endpoint);
    // 参数ar是服务启动后的处理函数
    server.init(vertx, SSL_KEY, ar -> {
      if (ar.succeeded()) {
        InetSocketAddress socketAddress = ar.result();
        LOGGER.info("highway listen success. address={}:{}",
            socketAddress.getHostString(),
            socketAddress.getPort());
        startFuture.complete();
        return;
      }

      LOGGER.error(HighwayTransport.NAME, ar.cause());
      startFuture.fail(ar.cause());
    });
  }
}
``` 
从上面的代码可以看出，highway协议的服务端初始化操作是在TcpServer中实现的，如下。  


```     
public class TcpServer {
  // 其它方法略

  public void init(Vertx vertx, String sslKey, AsyncResultCallback<InetSocketAddress> callback) {
    NetServer netServer;
    if (endpointObject.isSslEnabled()) {
      SSLOptionFactory factory =
          SSLOptionFactory.createSSLOptionFactory(sslKey, null);
      SSLOption sslOption;
      if (factory == null) {
        sslOption = SSLOption.buildFromYaml(sslKey);
      } else {
        sslOption = factory.createSSLOption();
      }
      SSLCustom sslCustom = SSLCustom.createSSLCustom(sslOption.getSslCustomClass());
      NetServerOptions serverOptions = new NetServerOptions();
      VertxTLSBuilder.buildNetServerOptions(sslOption, sslCustom, serverOptions);
      netServer = vertx.createNetServer(serverOptions);
    } else {
      netServer = vertx.createNetServer();
    }

    netServer.connectHandler(netSocket -> {
      TcpServerConnection connection = createTcpServerConnection();
      connection.init(netSocket);
    });

    InetSocketAddress socketAddress = endpointObject.getSocketAddress();
    netServer.listen(socketAddress.getPort(), socketAddress.getHostString(), ar -> {
      if (ar.succeeded()) {
        callback.success(socketAddress);
        return;
      }

      // 监听失败
      String msg = String.format("listen failed, address=%s", socketAddress.toString());
      callback.fail(new Exception(msg, ar.cause()));
    });
  }

  protected TcpServerConnection createTcpServerConnection() {
    return new TcpServerConnection();
  }
}
``` 

// to be continued


<br>

转载请注明：[蚊帐的博客](https://nevilleyeung.github.io) » [点击阅读原文](https://nevilleyeung.github.io/2018/08/ServiceComb-CodeAnalysis-Transport-startup/) 

 



