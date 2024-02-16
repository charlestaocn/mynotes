

**一、server.xml**

本质上tomcat的启动流程和总体架构都离不开server.xml。在Server.xml中我们可以看到一些我们比较熟悉的配置。

Listener结点配置：

![[tomcat/_resources/tomcat/1b67cdaa34442123657ad9ca4c7a8e26_MD5.png]]

service结点配置：

```javascript
<Service name="Catalina">
```



connector结点配置：

```javascript
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443"/>
```



这一篇会以源码的方式分析server.xml中的这些结点是在哪里被解析的，以及tomcat核心的启动流程。

**二、Bootstrap**

（1）tomcat启动第一步，是根据catalina.sh脚本来进行启动，在下方图画圈部分会发现，实际上是调用了Bootstrap类。

![[tomcat/_resources/tomcat/8c2d78931b82916b47a7dca721bd49a5_MD5.png]]

（2）追溯到Bootstrap类，发现在Bootstrap中有main方法。main方法中的源码如下：

```javascript
public static void main(String[] args) {
if (daemon == null) {//1、构造Bootstrap类        Bootstrap bootstrap = new Bootstrap();
        try {            //2、调用初始化方法            bootstrap.init();
        } catch (Throwable var3) {
            handleThrowable(var3);
            var3.printStackTrace();
            return;
        }
        daemon = bootstrap;
    } else {
        Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
    }    ...
}
```



（3）在调用初始化方法中，我们可以看到利用反射加载了Catalina类，并调用了setParentClassLoader方法，最后daemon被赋值为Catalina类

```javascript
public void init() throws Exception {
    this.initClassLoaders();
    Thread.currentThread().setContextClassLoader(this.catalinaLoader);
    SecurityClassLoad.securityClassLoad(this.catalinaLoader);
    if (log.isDebugEnabled()) {
        log.debug("Loading startup class");
    }

    Class<?> startupClass = this.catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
    Object startupInstance = startupClass.newInstance();
    if (log.isDebugEnabled()) {
        log.debug("Setting startup class properties");
    }

    String methodName = "setParentClassLoader";
    Class<?>[] paramTypes = new Class[]{Class.forName("java.lang.ClassLoader")};
    Object[] paramValues = new Object[]{this.sharedLoader};
    Method method = startupInstance.getClass().getMethod(methodName, paramTypes);
    method.invoke(startupInstance, paramValues);
    this.catalinaDaemon = startupInstance;
}
```



（4）可以理解为在（2）中创建了Catalina对象。重新回到main方法，继续往下看真正的主流程。

此时command为start，因此都会走进对应的start的if分支。主要作了三件事情。此时的daemon实际为catalina。

```javascript
try {
    String command = "start";
    if (args.length > 0) {
        command = args[args.length - 1];
    }

    if (command.equals("startd")) {
        args[args.length - 1] = "start";
        daemon.load(args);
        daemon.start();
    } else if (command.equals("stopd")) {
        args[args.length - 1] = "stop";
        daemon.stop();
    } else if (command.equals("start")) { //都会走进该if分支
        daemon.setAwait(true);
        daemon.load(args);
        daemon.start();
    } else if (command.equals("stop")) {
        daemon.stopServer(args);
    } else if (command.equals("configtest")) {
        daemon.load(args);
        if (null == daemon.getServer()) {
            System.exit(1);
        }

        System.exit(0);
    } else {
        log.warn("Bootstrap: command \"" + command + "\" does not exist.");
    }
} catch (Throwable var4) {
    Throwable t = var4;
    if (var4 instanceof InvocationTargetException && var4.getCause() != null) {
        t = var4.getCause();
    }

    handleThrowable(t);
    t.printStackTrace();
    System.exit(1);
}
```



（5）daemon.setAwait(true)，我的理解是是设置主线程一直等待来保证主线程不挂掉。

```javascript
daemon.setAwait(true);
```



（6）执行daemon.load(args)。这段源码的作用是通过反射调用catalina中的load方法。将catalina组件load出来。

```javascript
private void load(String[] arguments) throws Exception {
    String methodName = "load";
    Object[] param;
    Class[] paramTypes;
    if (arguments != null && arguments.length != 0) {
        paramTypes = new Class[]{arguments.getClass()};
        param = new Object[]{arguments};
    } else {
        paramTypes = null;
        param = null;
    }

    Method method = this.catalinaDaemon.getClass().getMethod(methodName, paramTypes);
    if (log.isDebugEnabled()) {
        log.debug("Calling startup class " + method);
    }

    method.invoke(this.catalinaDaemon, param);
}
```



**三、Catalina.load()**

```javascript
public void load() {

    if (loaded) {
        return;
    }
    loaded = true;
    long t1 = System.nanoTime();
    initDirs();
    initNaming();
    ConfigFileLoader.setSource(new CatalinaBaseConfigurationSource(Bootstrap.getCatalinaBaseFile(), getConfigFile()));
        // 获取server.xmlFile file = configFile();
        // 注意该方法    Digester digester = createStartDigester();
        try (ConfigurationSource.Resource resource = ConfigFileLoader.getSource().getServerXml()) {
    InputStream inputStream = resource.getInputStream();
    InputSource inputSource = new InputSource(resource.getURI().toURL().toString());
    inputSource.setByteStream(inputStream);    // 将server 放到栈顶 即下一个load的是server对象    digester.push(this);
    digester.parse(inputSource);
} catch (Exception e) {
    log.warn(sm.getString("catalina.configFail", file.getAbsolutePath()), e);
    if (file.exists() && !file.canRead()) {
        log.warn(sm.getString("catalina.incorrectPermissions"));
    }
    return;
}
//...}
```


## （1）进入createStartDigester，在这个方法中就会解析server.xml。

```javascript
/** * 解析server.xml */
protected Digester createStartDigester() {
    long t1=System.currentTimeMillis();
    // Initialize the digester
    Digester digester = new Digester();
    digester.setValidating(false);
    digester.setRulesValidation(true);
    Map<Class<?>, List<String>> fakeAttributes = new HashMap<>();
    // Ignore className on all elements
    List<String> objectAttrs = new ArrayList<>();
    objectAttrs.add("className");
    fakeAttributes.put(Object.class, objectAttrs);
    // Ignore attribute added by Eclipse for its internal tracking
    List<String> contextAttrs = new ArrayList<>();
    contextAttrs.add("source");
    fakeAttributes.put(StandardContext.class, contextAttrs);
    // Ignore Connector attribute used internally but set on Server
    List<String> connectorAttrs = new ArrayList<>();
    connectorAttrs.add("portOffset");
    fakeAttributes.put(Connector.class, connectorAttrs);
    digester.setFakeAttributes(fakeAttributes);
    digester.setUseContextClassLoader(true);
```



解析器主要做了三件事，addObjectCraeate-创建结点对应的对象，addSetProperties-添加结点属性，addSetNext-设置结点的调用规则，这里理解即可。知道server.xml是在catalina.load()方法中被解析的，并且不同的结点被发现需要解析时，都有各自对应的类来进行解析。

```javascript
digester.addObjectCreate("Server",
                         "org.apache.catalina.core.StandardServer",
                         "className");
digester.addSetProperties("Server");
digester.addSetNext("Server",
                    "setServer",
                    "org.apache.catalina.Server");
digester.addObjectCreate("Server/GlobalNamingResources",
                         "org.apache.catalina.deploy.NamingResourcesImpl");
digester.addSetProperties("Server/GlobalNamingResources");
digester.addSetNext("Server/GlobalNamingResources",
                    "setGlobalNamingResources",
                    "org.apache.catalina.deploy.NamingResourcesImpl");
digester.addRule("Server/Listener",
        new ListenerCreateRule(null, "className"));
digester.addSetProperties("Server/Listener");
digester.addSetNext("Server/Listener",
                    "addLifecycleListener",
                    "org.apache.catalina.LifecycleListener");   ....
```



在这个解析器方法中，解析的方法有很多，但实际上有些方法已经不需要使用了，比如Executor结点，在server.xml中可以看到已经被注释掉了，这是因为在之前的版本tomcat启动流程使用的是tomcat自带的线程池，而现在被注释掉了是因为使用了java自带的线程池，后面会讲到在启动流程时是如何使用线程池的。

```javascript
digester.addObjectCreate("Server/Service/Executor",
                 "org.apache.catalina.core.StandardThreadExecutor",
                 "className");
digester.addSetProperties("Server/Service/Executor");
digester.addSetNext("Server/Service/Executor",
                    "addExecutor",
                    "org.apache.catalina.Executor");
```



![[tomcat/_resources/tomcat/53ae31b2aec28fbf018d2030a403becc_MD5.png]]

（2）继续回到catalina.load()中，这里的getServer()根据刚才的解析器实际上就是StandarServer。

```javascript
// Start the new server
try {
    getServer().init();
} catch (LifecycleException e) {
    if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
        throw new java.lang.Error(e);
    } else {
        log.error(sm.getString("catalina.initError"), e);
    }
}

long t2 = System.nanoTime();
if(log.isInfoEnabled()) {
    log.info(sm.getString("catalina.init", Long.valueOf((t2 - t1) / 1000000)));
}
```



**四、StandarServer.init()**

进入init方法，发现实际上进入了LifecycleBase类，并且实现了init方法。在父类的init方法中设置了状态为INITIALIZING和INITIALIZED，实际上可以理解使用了LigecycleBase作为父类来管理了子类的生命周期，达到很好的复用-即如果不抽出来放到父类的话每个子类都需要多出这几行代码。

```javascript
public final synchronized void init() throws LifecycleException {
    if (!state.equals(LifecycleState.NEW)) {
        invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
    }

    try {
        setStateInternal(LifecycleState.INITIALIZING, null, false);
        initInternal();
        setStateInternal(LifecycleState.INITIALIZED, null, false);
    } catch (Throwable t) {
        handleSubClassException(t, "lifecycleBase.initFail", toString());
    }
}
```



查看initInternal()方法，发现果然是一个抽象的类，并且每个子类都重新了initInternal方法。

![[tomcat/_resources/tomcat/3ff72b938c40ea364879daf21a1507ef_MD5.png]]

![[tomcat/_resources/tomcat/58f415d2383c864c1120ffdb4bdead1e_MD5.png]]

（4）StandardServer.initInternal()

在Server中进行services的init方法。即调用了父类的init()方法管理生命周期，并调用了StandardService重写的initInternal()。

```javascript
protected void initInternal() throws LifecycleException {

    // 省去前面不重要代码 ...
    for (int i = 0; i < services.length; i++) {
        services[i].init();
    }
}
```



**五、StandardService.initInternal()**

![[tomcat/_resources/tomcat/c69fb24778541f80f286631ea9b63630_MD5.png]]

首先会启动engine，即调用StandardEngine的init方法，一样的调用流程，一样的管理周期，最终调用到StandardEngine重新的initInternal()。

```javascript
@Override
protected void initInternal() throws LifecycleException {

    super.initInternal();
    if (engine != null) {                // 1、启动engine        engine.init();
    }    //...
}
```



![[tomcat/_resources/tomcat/0546bec769417f1538af8a71c53cb5b6_MD5.png]]

**六、StandardEngine.initInternal()**

StandardEngine这里调用了父类ConainerBase的initInternal方法，设置线程池。

```javascript
@Override
protected void initInternal() throws LifecycleException {
    getRealm();
    super.initInternal();
}
```



```javascript
@Override
protected void initInternal() throws LifecycleException {    //设置线程池    reconfigureStartStopExecutor(getStartStopThreads());
    super.initInternal();
}
```



在这一步设置了tomcat启动和关闭的线程池。为了启动standardEngine下面的所有组件，通过线程池的方式来启动。

```javascript
private void reconfigureStartStopExecutor(int threads) {
    if (threads == 1) {
        if (!(startStopExecutor instanceof InlineExecutorService)) {
            startStopExecutor = new InlineExecutorService();
        }
    } else {
        Server server = Container.getService(this).getServer();
        server.setUtilityThreads(threads);
        startStopExecutor = server.getUtilityExecutor();
    }
}
```



（7）Engin的init成功后，重新回到StandardService。

```javascript
@Override
protected void initInternal() throws LifecycleException {

    super.initInternal();
    if (engine != null) {//standardEngine的启动
       engine.init();
    }

    //因为server.xml并没有设置线程池（被注释），因此不会走这段代码来创建线程池
    for (Executor executor : findExecutors()) {
        if (executor instanceof JmxEnabled) {
            ((JmxEnabled) executor).setDomain(getDomain());
        }
        executor.init();
    }

    //mapperListener的启动
    mapperListener.init();
        //最后启动connector    synchronized (connectorsLock) {
        for (Connector connector : connectors) {
            connector.init();
        }
    }
}
```



进行connector的启动，调用Connector重写的initInternal方法。

![[tomcat/_resources/tomcat/0e61d742f927dc62eade9026234f9f09_MD5.png]]

**七、Connector.initernal()**

```javascript
@Override
protected void initInternal() throws LifecycleException {

    super.initInternal();
    if (protocolHandler == null) {
        throw new LifecycleException(
                sm.getString("coyoteConnector.protocolHandlerInstantiationFailed"));
    }

    //构造适配器
    adapter = new CoyoteAdapter(this);
    //指定协议、绑定适配器    protocolHandler.setAdapter(adapter);
    if (service != null) {
        protocolHandler.setUtilityExecutor(service.getServer().getUtilityExecutor());
    }    // ...
```



在指定协议中会使用到protocolHandler，这里的protocolHandler究竟是什么样的模式呢？

下面截图可以看到protocolHandler的引用，有NIO、HttpProtocol。可以回到catalina对象中的解析器，查看Connector是如何解析的。

![[tomcat/_resources/tomcat/859af2b8dd309e251bc09a5b2d5131df_MD5.png]]

定位到解析器中的addRule方法，查看里面的ConnectorCreateRule类。发现在内部构造了Connector对象。

```javascript
digester.addRule("Server/Service/Connector",
                 new ConnectorCreateRule());
```



这里的protocal就为server.xml中Connector结点配置的protocol属性。

![[tomcat/_resources/tomcat/3773ffcc3bc3f31cc78c412fb107d953_MD5.png]]

![[tomcat/_resources/tomcat/bea2e0d693e7cf105ab457f15acc078d_MD5.png]]

这一步即在构造Connector时，指定了protocolHandler为Http11NioProtocol — NIO模式。

```javascript
public Connector(String protocol) {
    boolean aprConnector = AprLifecycleListener.isAprAvailable() &&
AprLifecycleListener.getUseAprConnector();
    //配置为HTTP/1.1 并且没有配置aprConnector    if ("HTTP/1.1".equals(protocol) || protocol == null) {
        if (aprConnector) {
            protocolHandlerClassName = "org.apache.coyote.http11.Http11AprProtocol";
        } else {
            protocolHandlerClassName = "org.apache.coyote.http11.Http11NioProtocol";
        }
    } else if ("AJP/1.3".equals(protocol)) {
        if (aprConnector) {
            protocolHandlerClassName = "org.apache.coyote.ajp.AjpAprProtocol";
        } else {
            protocolHandlerClassName = "org.apache.coyote.ajp.AjpNioProtocol";
        }
    } else {
        protocolHandlerClassName = protocol;
    }// 利用反射构造 protocolHandler
ProtocolHandler p = null;
try {
    Class<?> clazz = Class.forName(protocolHandlerClassName);
    p = (ProtocolHandler) clazz.getConstructor().newInstance();
} catch (Exception e) {
    log.error(sm.getString(
            "coyoteConnector.protocolHandlerInstantiationFailed"), e);
} finally {
    this.protocolHandler = p;
}
}
```



**七、protocolHandler.init()**

回到Connector.initInternal方法，进行protocolHandler的初始化操作。

```javascript
@Override
protected void initInternal() throws LifecycleException {
    //...
    try {
        protocolHandler.init();
    } catch (Exception e) {
        throw new LifecycleException(
                sm.getString("coyoteConnector.protocolHandlerInitializationFailed"), e);
    }
}
```



追溯到父类AbstractProtocol，会进行endPoint的初始化方法。

```javascript
@Override
public void init() throws Exception {
    //...
    String endpointName = getName();
    endpoint.setName(endpointName.substring(1, endpointName.length()-1));
    endpoint.setDomain(domain);
    //endPoint的初始化操作    endpoint.init();
}
```



而endPoint对象的构造就比较简单了，不是由server.xml解析出来的，如下图所示。

```javascript
public Http11NioProtocol() {
    super(new NioEndpoint());
}
```



**八、endPoint.init()**

```javascript
public final void init() throws Exception {
if (bindOnInit) {//绑定端口、http协议bindWithCleanup();
        bindState = BindState.BOUND_ON_INIT;
    }
    if (this.domain != null) {
        // Register endpoint (as ThreadPool - historical name)
        oname = new ObjectName(domain + ":type=ThreadPool,name=\"" + getName() + "\"");
        Registry.getRegistry(null, null).registerComponent(this, oname, null);
        ObjectName socketPropertiesOname = new ObjectName(domain +
                ":type=ThreadPool,name=\"" + getName() + "\",subType=SocketProperties");
        socketProperties.setObjectName(socketPropertiesOname);
        Registry.getRegistry(null, null).registerComponent(socketProperties, socketPropertiesOname, null);
        for (SSLHostConfig sslHostConfig : findSslHostConfigs()) {
            registerJmx(sslHostConfig);
        }
    }
}
```



（1）查看bindWithCleanUp方法是做什么的。

![[tomcat/_resources/tomcat/4b178cb9f84d3dfdeb02da4568864284_MD5.png]]

进入后查看NioEndpoint对应的bind方法。

```javascript
@Override
public void bind() throws Exception {    // 这里主要进行IP+端口的绑定    initServerSocket();
    // acceptor线程数量
    if (acceptorThreadCount == 0) {
        acceptorThreadCount = 1;
    }    // poller线程数量
```



```javascript
    if (pollerThreadCount <= 0) {
        pollerThreadCount = 1;
    }
    setStopLatch(new CountDownLatch(pollerThreadCount));
    // SSL证书的绑定
    initialiseSsl();
    selectorPool.open();
}
```



绑定后，NIOEntpoint执行结束。到这一步tomcat对应的init启动流程结束。

**九、tomcat-init总结**

tomcat启动中会先调用init方法，将需要用到的类先初始化完毕，最后绑定好ip与端口，总体的init才算完成，init顺序如下面的流程图所示。并且细心的同学可以发现，前面的类并不关心后面的类是如何init的，当自己被init结束后，传递给下一个类-即结点来进行处理或者返回处理完毕，这就是责任链模式，在tomcat中启动中就用了责任链模式。

![[tomcat/_resources/tomcat/c1d0f2828e1d267574a61adb45cddf1c_MD5.png]]

