---
title: Dubbo源码阅读之集成Spring标签配置类(01)
date: 2018-08-04 09:26:06
tags: 
    - dubbo
---

>本小节将会介绍下如何在Spring环境中使用Dubbo，以及Dubbo标签实体类

* Spring中使用Dubbo的Demo
* Dubbo标签解析


### Spring中使用Dubbo的Demo
我们先来看下Dubbo源码中dubbo-demo包中的例子。

#### dubbo-demo-api
首先在dubbo-demo-api模块中定义了一个接口：
```java
public interface DemoService {
    String sayHello(String name);
}
```

然后在dubbo-demo-provider模块和dubbo-demo-consumer模块中分别引入dubbo-demo-api模块。

#### dubbo-demo-provider
我们先来看下dubbo-demo-provider服务提供端,新建一个DemoService接口的实现类DemoServiceImpl：
```java
public class DemoServiceImpl implements DemoService {
    @Override
    public String sayHello(String name) {
        System.out.println("[" + new SimpleDateFormat("HH:mm:ss").format(new Date()) + "] Hello " + name + ", request from consumer: " + RpcContext.getContext().getRemoteAddress());
        return "Hello " + name + ", response from provider: " + RpcContext.getContext().getLocalAddress();
    }
}
```
然后在Spring配置文件中进行配置：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
       http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    
    <dubbo:application name="demo-provider"/>
    
    <dubbo:registry address="multicast://224.5.6.7:1234"/>
    
    <dubbo:protocol name="dubbo" port="20880"/>
    
    <bean id="demoService" class="com.alibaba.dubbo.demo.provider.DemoServiceImpl"/>
    
    <!--暴露服务-->
    <dubbo:service interface="com.alibaba.dubbo.demo.DemoService" ref="demoService"/>
</beans>
```
可以看到我们首先在配置文件中引入了dubbo自定义的schema文件，然后我们就可以在配置文件中使用dubbo自定义的标签了。这些标签我们后面会讲解。
接下来我们新建一个Provider类，该类用来加载Spring配置文件，同时也启动了我们的服务器：
```java
public class Provider {

    public static void main(String[] args) throws Exception {
        System.setProperty("java.net.preferIPv4Stack", "true");
        //加载Spring配置
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/dubbo-demo-provider.xml"});
        context.start();
        System.in.read(); // press any key to exit
    }
}
```

#### dubbo-demo-consumer

我们在消费者端同样需要配置Spring配置文件：
```xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
       http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">

    <dubbo:application name="demo-consumer"/>

    <dubbo:registry address="multicast://224.5.6.7:1234"/>
    
    <!--引用DemoService服务-->
    <dubbo:reference id="demoService" check="false" interface="com.alibaba.dubbo.demo.DemoService"/>

</beans>
```
接着我们新建一个Consumer类：
```java
public class Consumer {

    public static void main(String[] args) {
        System.setProperty("java.net.preferIPv4Stack", "true");
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/dubbo-demo-consumer.xml"});
        context.start();

        //获取远程服务代理
        DemoService demoService = (DemoService) context.getBean("demoService");
       
        while (true) {
            try {
                Thread.sleep(1000);
                //调用远程服务方法
                String hello = demoService.sayHello("world");
                //输出调用结果
                System.out.println(hello);
            } catch (Throwable throwable) {
                throwable.printStackTrace();
            }
        }
    }
}
```
启动起来服务器和客户端，我们就可以看到输出结果了。经过上面的例子，我们可以看到使用Dubbo调用远程服务就像调用本地的服务一样简单。后面我们将详细介绍Dubbo是如何做到这些的。


### Dubbo标签解析

总体上Dubbo是通过自定义Spring标签，然后解析这些标签，将这些标签属性映射成一个个配置类，接着将这些类配置到Spring工厂，交给Spring容器来管理的方式来和Spring整合到一起的。
这部分源码主要在dubbo-config包中，我们先看下Spring自定义标签的机制以及Dubbo标签对应的实体类，下一小节在具体分析如何解析标签。

#### Spring自定义标签
在dubbo-config-spring模块的resources目录下有三个文件，分别为：dubbo.xsd、spring.handlers、spring.schemas。
dubbo.xsd文件中定义了所有支持的dubbo标签，spring.schemas文件中则指定了dubbo.xsd文件：
```java
http\://dubbo.apache.org/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd
```
因此，我们需要分别在服务器、客户端的spring配置文件中引入该xsd文件，才可以使用dubbo标签。
而spring.handlers文件中则指定了解析dubbo标签的类:
```java
http\://dubbo.apache.org/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler
```
我们就以com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler类作为入口，看看dubbo是如何解析标签的。

我们来看下DubboNamespaceHandler类，该类继承自Spring的NamespaceHandlerSupport类，通过实现init方法来将dubbo标签节点和解析类进行关联：
```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
    static {
        //检测是否有重复的DubboNamespaceHandler类，这里不会抛异常，只会打印错误日志
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    /**
     * 将节点名和解析类关联起来，NamespaceHandler通过节点名会找到相应的解析类
     */
    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
}
```
可以看到这里定义了dubbo标签的解析类DubboBeanDefinitionParser，同时也定义了该标签对应的bean(即xxxConfig类、xxxBean类)，这些bean最终交由Spring容器来管理。在介绍它们之前，我们先看一些通用的注解类,这些注解可以添加在bean类中的方法上，在解析标签的时候会用上这些注解。
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Parameter {

    String key() default "";

    boolean required() default false;

    /**
     * 是否排除该方法
     * @return
     */
    boolean excluded() default false;

    /**
     * 是否编码
     * @return
     */
    boolean escaped() default false;

    /**
     * 属性为false的话，则不执行附加属性方法
     * @return
     */
    boolean attribute() default false;

    boolean append() default false;
}
```

#### Dubbo标签对应的实体类
在dubbo-config-api模块下的com.alibaba.dubbo.config包中定义了dubbo标签对应的实体类，我们先了解下这些实体类，然后再看如何解析它们。

##### ApplicationConfig
我们对照着使用方法一起看下：
```xml
 <!--dubbo:application中可配置的属性都定义在ApplicationConfig实体类中-->
 <dubbo:application name="demo-provider"/>
```
ApplicationConfig类继承自AbstractConfig抽象类，该抽象类提供了一些解析配置的公共方法。
```java
public class ApplicationConfig extends AbstractConfig {

    private static final long serialVersionUID = 5508512956753757169L;

    /**
     * 服务治理（必填）
     * 当前应用名称，用于注册中心计算应用间依赖关系，注意：消费者和提供者应用名不要一样，
     * 此参数不是匹配条件，你当前项目叫什么名字就填什么，和提供者消费者角色无关
     * application
     */
    private String name;

    /**
     * 服务治理
     * 当前应用模块版本
     * application.version
     */
    private String version;

    /**
     * 服务治理
     * 应用负责人，用于服务治理，请填写负责人公司邮箱前缀
     * owner
     */
    private String owner;

    /**
     * 服务治理
     * 组织名称(BU或部门)，用于注册中心区分服务来源，此配置项建议不要使用autoconfig，直接写死在配 置中，比如 china,intl,itu,crm,asc,dw,aliexpress 等
     * organization
     */
    private String organization;

    /**
     * 服务治理
     * 用于服务分层对应的架构。如，intl、china。不同的架构使用不同的分层
     * architecture
     */
    private String architecture;

    /**
     * 服务治理
     * 环境，例如：dev、test、production
     * environment
     */
    private String environment;

    /**
     * 性能优化（javassist）
     * Java字节码编译器，用于动态类的生成，可选：jdk或javassist
     * compiler 
     */
    private String compiler;

    /**
     * 性能优化（slf4j）
     * 日志输出方式，可选： slf4j,jcl,log4j,jdk
     * logger
     */
    private String logger;

    /**
     * 注册中心列表
     */
    private List<RegistryConfig> registries;

    /**
     * 监控中心
     */
    private MonitorConfig monitor;

    /**
     * 是否是默认的
     */
    private Boolean isDefault;

    /**
     * 保存线程转储的目录
     */
    private String dumpDirectory;

    /**
     * 是否启用qos
     */
    private Boolean qosEnable;

    /**
     * qos端口
     */
    private Integer qosPort;
    /**
     * 是否可以接受国外ip
     */
    private Boolean qosAcceptForeignIp;

    /**
     *  自定义参数
     */
    private Map<String, String> parameters;

    public ApplicationConfig() {
    }

    public ApplicationConfig(String name) {
        setName(name);
    }

    @Parameter(key = Constants.APPLICATION_KEY, required = true)
    public String getName() {
        return name;
    }

    public void setName(String name) {
        checkName("name", name);
        this.name = name;
        if (id == null || id.length() == 0) {
            id = name;
        }
    }

    @Parameter(key = "application.version")
    public String getVersion() {
        return version;
    }
    public void setCompiler(String compiler) {
        this.compiler = compiler;
        AdaptiveCompiler.setDefaultCompiler(compiler);
    }
    public void setLogger(String logger) {
        this.logger = logger;
        LoggerFactory.setLoggerAdapter(logger);
    }
    // ...省略其他相似方法
}
```
##### ModuleConfig
```java
public class ModuleConfig extends AbstractConfig {

    private static final long serialVersionUID = 5508512956753757169L;

    /**
     * 服务治理（必填）
     * 当前模块名称，用于注册中心计算模块间依赖关系
     * module
     */
    private String name;

    /**
     * 服务治理
     * 当前模块的版本
     * module.version
     */
    private String version;

    /**
     * 服务治理
     * 模块负责人，用于服务治理，请填 写负责人公司邮箱前缀
     * owner
     */
    private String owner;

    /**
     * 服务治理
     * 组织名称(BU或部门)，用于注册中心区分服务来源，此配置项建议不要使用autoconfig，直接写死在配 置中，比如 china,intl,itu,crm,asc,dw,aliexpress 等
     * organization
     */
    private String organization;

    /**
     * 注册中心列表
     * registry centers
     */
    private List<RegistryConfig> registries;

    /**
     * 监控中心
     * monitor center
     */
    private MonitorConfig monitor;

    /**
     * 是否是默认
     */
    private Boolean isDefault;

    public ModuleConfig(String name) {
        setName(name);
    }

    @Parameter(key = "module", required = true)
    public String getName() {
        return name;
    }

    public void setName(String name) {
        checkName("name", name);
        this.name = name;
        //id为空的话，使用name的值
        if (id == null || id.length() == 0) {
            id = name;
        }
    }

    @Parameter(key = "module.version")
    public String getVersion() {
        return version;
    }
    
    public void setOwner(String owner) {
        checkName("owner", owner);
        this.owner = owner;
    }
    
    public void setOrganization(String organization) {
        checkName("organization", organization);
        this.organization = organization;
    }

    public RegistryConfig getRegistry() {
        return registries == null || registries.isEmpty() ? null : registries.get(0);
    }

    public void setRegistry(RegistryConfig registry) {
        List<RegistryConfig> registries = new ArrayList<RegistryConfig>(1);
        registries.add(registry);
        this.registries = registries;
    }

    @SuppressWarnings({"unchecked"})
    public void setRegistries(List<? extends RegistryConfig> registries) {
        this.registries = (List<RegistryConfig>) registries;
    }
    // ...省略其他类似get、set方法
}
```

##### RegistryConfig
注册中心配置，如果有多个不同的注册中心，可以声明多个 <dubbo:registry> 标签，并在 <dubbo:service> 或 <dubbo:reference> 的 registry 属性指定使用的注册中心.
```java
public class RegistryConfig extends AbstractConfig {

    /**
     * 不可用标识
     */
    public static final String NO_AVAILABLE = "N/A";

    /**
     * 服务发现（必填）
     * 注册中心服务器地址，如果地址没有端口缺省为9090，同一集群内的多个地址用逗号分隔，如：ip:port,ip:port
     * 不同集群的注册中心，请配置多个 <dubbo:registry> 标签
     * <host:port>
     */
    private String address;

    /**
     * 服务治理
     * 登录注册中心用户名，如果注册中心不需要验证可不填
     * <username>
     */
    private String username;

    /**
     * 服务治理
     * 登录注册中心密码，如果注册中心不需要验证可不填
     * <password>
     */
    private String password;

    /**
     * 服务发现
     * 注册中心默认端口,当address没有带端口时使用此端口做为缺省值
     * <port>
     */
    private Integer port;

    /**
     * 服务发现(dubbo)
     * 注册中心地址协议,支持dubbo,http,local三种协议,分别表示: dubbo地址,http地址,本地注册中心
     * <protocol>
     */
    private String protocol;

    /**
     * 性能调优(netty)
     * 网络传输方式，可选mina,netty
     * registry.transporter
     */
    private String transporter;

    private String server;

    private String client;
    
    private String cluster;

    private String group;

    private String version;

    /**
     * 性能调优(5000)
     * 注册中心请求超时时间(毫秒)
     * registry.timeout	
     */
    private Integer timeout;

    /** 
     * 性能调优(60000)
     * 注册中心会话超时时间(毫秒)，用于检测提供者非正常断线后的脏数据，比如用心跳检测的实现，此时间就是心跳间隔，不同注册中心实现不一样
     * registry.session
     */
    private Integer session;

    /**
     * 服务治理
     * 使用文件缓存注 册中心地址列表及服务提供者列表，应用重启时将基于此文件恢复，注意：两个注册中心不能使用同一文件存储
     * registry.file
     */
    private String file;

    /**
     * 性能调优(0)
     * 停止时等待通知完成时间(毫秒)
     * registry.wait
     */
    private Integer wait;

    /**
     * 服务治理(true)
     * 在启动时，是否检测注册中心是否可用
     * check
     */
    private Boolean check;

    /**
     * 服务治理(true)
     * 服务是否动态注册，如果设为 false，注册后将显示后disable状态，需人工启用，并且服务提供者停止时，也不会自动取消册，需人工禁用
     * dynamic
     */
    private Boolean dynamic;

    /**
     * 服务治理（true）
     * 是否向此注册中心注册服务，如果设为false，将只订阅，不注册
     * register
     */
    private Boolean register;

    /**
     * 服务治理（true）
     * 是否向此注册中心订阅服务，如果设为false，将只注册，不订阅
     * subscribe 
     */
    private Boolean subscribe;

    /**
     * 自定义参数
     */
    private Map<String, String> parameters;

    /**
     * 是否是默认
     */
    private Boolean isDefault;

    public RegistryConfig() {
    }

    public static void destroyAll() {
        AbstractRegistryFactory.destroyAll();
    }

    public void setProtocol(String protocol) {
        checkName("protocol", protocol);
        this.protocol = protocol;
    }

    @Parameter(excluded = true)
    public String getAddress() {
        return address;
    }

    public void setUsername(String username) {
        checkName("username", username);
        this.username = username;
    }

    public void setFile(String file) {
        checkPathLength("file", file);
        this.file = file;
    }

    public void setTransporter(String transporter) { 
        checkName("transporter", transporter);
        this.transporter = transporter;
    }

    public void setServer(String server) {
        checkName("server", server);
        this.server = server;
    }

    public void setClient(String client) {
        checkName("client", client);
        this.client = client;
    }

}
```
##### MonitorConfig
```java
public class MonitorConfig extends AbstractConfig {

    /**
     * 服务治理(dubbo)
     * 监控中心协议，如果为protocol=”registry”，表示从注册中心发现监控中心地址，否则直连监控中心
     * protocol
     */
    private String protocol;
    
    /**
     * 服务治理(N/A)
     * 直连监控中心服务器地址，address=”10.20.130.230:12080”
     * <url>
     */
    private String address;

    private String username;

    private String password;

    private String group;

    private String version;

    private String interval;

    private Map<String, String> parameters;

    private Boolean isDefault;

    public MonitorConfig(String address) {
        this.address = address;
    }

    @Parameter(excluded = true)
    public String getAddress() {
        return address;
    }

    @Parameter(excluded = true)
    public String getProtocol() {
        return protocol;
    }

    @Parameter(excluded = true)
    public String getUsername() {
        return username;
    }
    @Parameter(excluded = true)
    public String getPassword() {
        return password;
    }

    public void setParameters(Map<String, String> parameters) {
        checkParameterName(parameters);
        this.parameters = parameters;
    }

}
```
##### ProviderConfig
该类继承自AbstractServiceConfig类
```java
public class ProviderConfig extends AbstractServiceConfig {

    //如果没有设置协议的属性值，那么默认值将会生效
    /**
     * 服务ip地址（当有多个网卡可用时使用）(自动查找本机IP)
     * <host>
     */
    private String host;

    /**
     * 服务端口
     * port
     */
    private Integer port;

    /**
     * 上下文路径
     * contextpath
     */
    private String contextpath;

    /**
     * 线程池(fixed)
     * threadpool
     */
    private String threadpool;

    /**
     * 线程池大小（固定大小）
     */
    private Integer threads;

    /**
     * IO线程池大小（CPU数+1）
     * iothreads
     */
    private Integer iothreads;

    /**
     * 线程池队列长度(0)
     * queues
     */
    private Integer queues;

    /**
     * 最大可接受的连接(0)
     * accepts
     */
    private Integer accepts;

    /**
     * 协议编解码器（dubbo）
     * codec
     */
    private String codec;

    /**
     * 编码（UTF-8）
     * charset
     */
    private String charset;

    /**
     * payload最大长度(88388608(=8M))
     * payload
     */
    private Integer payload;

    /**
     * buffer大小(8192)
     * buffer
     */
    private Integer buffer;

    /**
     * transporter
     */
    private String transporter;

    /**
     * how information gets exchanged
     */
    private String exchanger;

    /**
     * 性能调优
     * 线程调度模式
     * 协议的消息派发方式，用于指定线程模型，比如：dubbo协议的all,direct,message,execution,connection等
     */
    private String dispatcher;

    /**
     * networker
     */
    private String networker;

    /**
     * 服务器实现
     * dubbo协议缺省 为netty，http协议缺省为servlet
     * server
     */
    private String server;

    /**
     * 客户端实现
     * dubbo协议缺省 为netty
     * client
     */
    private String client;

    /**
     * 支持telnet命令,逗号分割
     */
    private String telnet;

    /**
     * 命令行提示符
     */
    private String prompt;

    /**
     * 状态检测
     */
    private String status;

    /**
     * 停止的时候等待多久
     */
    private Integer wait;

    private Boolean isDefault;

    @Parameter(excluded = true)
    public Boolean isDefault() {
        return isDefault;
    }

    @Parameter(excluded = true)
    public String getHost() {
        return host;
    }

    @Parameter(excluded = true)
    public Integer getPort() {
        return port;
    }

    @Parameter(excluded = true)
    public String getContextpath() {
        return contextpath;
    }

    public void setContextpath(String contextpath) {
        checkPathName("contextpath", contextpath);
        this.contextpath = contextpath;
    }

    public void setThreadpool(String threadpool) {
        checkExtension(ThreadPool.class, "threadpool", threadpool);
        this.threadpool = threadpool;
    }

    public void setTelnet(String telnet) {
        checkMultiExtension(TelnetHandler.class, "telnet", telnet);
        this.telnet = telnet;
    }

    @Parameter(escaped = true)
    public String getPrompt() {
        return prompt;
    }
    public void setStatus(String status) {
        checkMultiExtension(StatusChecker.class, "status", status);
        this.status = status;
    }

    @Override
    public String getCluster() {
        return super.getCluster();
    }

    @Override
    public Integer getConnections() {
        return super.getConnections();
    }

    @Override
    public Integer getTimeout() {
        return super.getTimeout();
    }

    @Override
    public Integer getRetries() {
        return super.getRetries();
    }

    @Override
    public String getLoadbalance() {
        return super.getLoadbalance();
    }

    @Override
    public Boolean isAsync() {
        return super.isAsync();
    }

    @Override
    public Integer getActives() {
        return super.getActives();
    }

    public void setTransporter(String transporter) {
        checkExtension(Transporter.class, "transporter", transporter);
        this.transporter = transporter;
    }

    public void setExchanger(String exchanger) {
        checkExtension(Exchanger.class, "exchanger", exchanger);
        this.exchanger = exchanger;
    }

    public void setDispatcher(String dispatcher) {
        checkExtension(Dispatcher.class, Constants.DISPATCHER_KEY, exchanger);
        checkExtension(Dispatcher.class, "dispather", exchanger);
        this.dispatcher = dispatcher;
    }
}
```
##### AbstractMethodConfig
该抽象类继承自AbstractConfig
```java
public abstract class AbstractMethodConfig extends AbstractConfig {

    /**
     * 远程调用超时(1000毫秒)
     * timeout for remote invocation in milliseconds
     */
    protected Integer timeout;

    /**
     * 重试次数（2）
     */
    protected Integer retries;

    /**
     * 最大并发调用(0)
     * max concurrent invocations
     */
    protected Integer actives;

    /**
     * 负载均衡（random）
     */
    protected String loadbalance;

    /**
     * 是否异步
     */
    protected Boolean async;

    /**
     * 是否确认异步发送
     */
    protected Boolean sent;

    /**
     * 当服务调用失败时，被调用的mack类的名字
     * mock
     */
    protected String mock;

    /**
     * 合并
     */
    protected String merger;

    /**
     * 缓存
     */
    protected String cache;

    /**
     * 验证
     */
    protected String validation;

    /**
     * 自定义参数
     */
    protected Map<String, String> parameters;

    public void setLoadbalance(String loadbalance) {
        checkExtension(LoadBalance.class, "loadbalance", loadbalance);
        this.loadbalance = loadbalance;
    }

    @Parameter(escaped = true)
    public String getMock() {
        return mock;
    }

    public void setMock(Boolean mock) {
        if (mock == null) {
            setMock((String) null);
        } else {
            setMock(String.valueOf(mock));
        }
    }

    public void setMock(String mock) {
        if (mock != null && mock.startsWith(Constants.RETURN_PREFIX)) {
            checkLength("mock", mock);
        } else {
            checkName("mock", mock);
        }
        this.mock = mock;
    }
}
```

##### AbstractInterfaceConfig
该类继承自AbstractMethodConfig
```java
public abstract class AbstractInterfaceConfig extends AbstractMethodConfig {

    /**
     * 服务接口的本地实现类名称
     */
    protected String local;

    /**
     * 服务接口的本地存根类名称
     */
    protected String stub;

    /**
     * 服务监控
     */
    protected MonitorConfig monitor;

    /**
     * 代理类型
     */
    protected String proxy;

    /**
     * 性能调优（failover）
     * 集群方式，可选：failover/failfast/failsafe/failback/forking
     * default.cluster
     */
    protected String cluster;

    /**
     * 过滤器
     */
    protected String filter;

    /**
     * 监听器
     */
    protected String listener;

    /**
     * 所有者
     */
    protected String owner;

    /**
     * 连接限制，0标识共享连接（0）
     * 否则它定义委托给当前服务的连接
     * default.connections	
     */
    protected Integer connections;

    protected String layer;

    /**
     * application配置
     */
    protected ApplicationConfig application;

    /**
     *  module配置
     */
    protected ModuleConfig module;

    /**
     * 注册中心内地址
     * <dubbo:registry address="multicast://224.5.6.7:1234" id="com.alibaba.dubbo.config.RegistryConfig" />
     */
    protected List<RegistryConfig> registries;

    /**
     * 连接事件
     * connection events
     */
    protected String onconnect;

    /**
     * 断开连接事件
     * disconnection events
     */
    protected String ondisconnect;

    /**
     * 回调限制
     * callback limits
     */
    private Integer callbacks;

    /**
     * 引用/暴露服务的作用域
     * 如果它是local，则意味着只在当前虚拟机中进行搜索
     */
    private String scope;

    public void setStub(Boolean stub) {
        if (stub == null) {
            setStub((String) null);
        } else {
            setStub(String.valueOf(stub));
        }
    }

    public void setStub(String stub) {
        checkName("stub", stub);
        this.stub = stub;
    }

    public void setCluster(String cluster) {
        checkExtension(Cluster.class, "cluster", cluster);
        this.cluster = cluster;
    }

    public void setProxy(String proxy) {
        checkExtension(ProxyFactory.class, "proxy", proxy);
        this.proxy = proxy;
    }

    @Parameter(key = Constants.REFERENCE_FILTER_KEY, append = true)
    public String getFilter() {
        return filter;
    }

    public void setFilter(String filter) {
        checkMultiExtension(Filter.class, "filter", filter);
        this.filter = filter;
    }

    @Parameter(key = Constants.INVOKER_LISTENER_KEY, append = true)
    public String getListener() {
        return listener;
    }

    public void setListener(String listener) {
        checkMultiExtension(InvokerListener.class, "listener", listener);
        this.listener = listener;
    }
    public void setLayer(String layer) {
        checkNameHasSymbol("layer", layer);
        this.layer = layer;
    }

    public RegistryConfig getRegistry() {
        return registries == null || registries.isEmpty() ? null : registries.get(0);
    }

    public void setRegistry(RegistryConfig registry) {
        List<RegistryConfig> registries = new ArrayList<RegistryConfig>(1);
        registries.add(registry);
        this.registries = registries;
    }
    //省略其他方法...后面会介绍
}
```
##### AbstractServiceConfig
该抽象类继承自AbstractInterfaceConfig
```java
public abstract class AbstractServiceConfig extends AbstractInterfaceConfig {


    /**
     * 版本(0.0.0)
     */
    protected String version;

    /**
     * 组
     */
    protected String group;

    /**
     * 服务是否被弃用
     */
    protected Boolean deprecated;

    /**
     * 延迟暴露
     */
    protected Integer delay;

    /**
     * 是否暴露服务
     */
    protected Boolean export;

    /**
     * 权重
     */
    protected Integer weight;

    /**
     * 文档中心
     */
    protected String document;

    /**
     * 是否在注册中心注册为动态服务(true)
     */
    protected Boolean dynamic;

    /**
     * 是否使用token
     */
    protected String token;

    /**
     * 访问日志
     * access log
     */
    protected String accesslog;
    protected List<ProtocolConfig> protocols;
    /**
     * 允许最大执行时间
     * max allowed execute times
     */
    private Integer executes;
    /**
     * 是否注册
     * whether to register
     */
    private Boolean register;

    /**
     * 根据指定的稳定吞吐率和预热期来创建RateLimiter
     * warm up period
     */
    private Integer warmup;

    /**
     * dubbo协议缺省为hessian2， rmi协议缺省为java，http协议缺省为json
     * serialization
     */
    private String serialization;


    public void setVersion(String version) {
        checkKey("version", version);
        this.version = version;
    }

    public void setGroup(String group) {
        checkKey("group", group);
        this.group = group;
    }

    @Parameter(escaped = true)
    public String getDocument() {
        return document;
    }

    public void setToken(String token) {
        checkName("token", token);
        this.token = token;
    }

    public void setToken(Boolean token) {
        if (token == null) {
            setToken((String) null);
        } else {
            setToken(String.valueOf(token));
        }
    }

    @SuppressWarnings({"unchecked"})
    public void setProtocols(List<? extends ProtocolConfig> protocols) {
        this.protocols = (List<ProtocolConfig>) protocols;
    }

    public ProtocolConfig getProtocol() {
        return protocols == null || protocols.isEmpty() ? null : protocols.get(0);
    }

    public void setProtocol(ProtocolConfig protocol) {
        this.protocols = Arrays.asList(new ProtocolConfig[]{protocol});
    }

    public void setAccesslog(Boolean accesslog) {
        if (accesslog == null) {
            setAccesslog((String) null);
        } else {
            setAccesslog(String.valueOf(accesslog));
        }
    }

    @Override
    @Parameter(key = Constants.SERVICE_FILTER_KEY, append = true)
    public String getFilter() {
        return super.getFilter();
    }

    @Override
    @Parameter(key = Constants.EXPORTER_LISTENER_KEY, append = true)
    public String getListener() {
        return super.getListener();
    }

    @Override
    public void setListener(String listener) {
        checkMultiExtension(ExporterListener.class, "listener", listener);
        super.setListener(listener);
    }
}

```
##### ConsumerConfig
```java
public class ConsumerConfig extends AbstractReferenceConfig {

    private Boolean isDefault;

    /**
     * 使用的网络框架客户端：netty, mina, etc
     */
    private String client;

    @Override
    public void setTimeout(Integer timeout) {
        super.setTimeout(timeout);
        //设置rmi超时时间
        String rmiTimeout = System.getProperty("sun.rmi.transport.tcp.responseTimeout");
        if (timeout != null && timeout > 0
                && (rmiTimeout == null || rmiTimeout.length() == 0)) {
            System.setProperty("sun.rmi.transport.tcp.responseTimeout", String.valueOf(timeout));
        }
    }
}
```
##### AbstractReferenceConfig
```java
public abstract class AbstractReferenceConfig extends AbstractInterfaceConfig {

    //如果Reference的属性没有配置，则默认值将会生效

    /**
     * 检测服务提供者是否存在
     */
    protected Boolean check;

    /**
     * 1、是否立即初始化，如果为true,bean加载时会立即调用消费者初始化
     * 2、消费者bean被使用者调用时，调用getObject->get->init
     */
    protected Boolean init;

    /**
     * 是否使用缺省泛化接口
     */
    protected String generic;

    /**
     * scope = local
     * 是否从当前虚拟机中查找reference的实例
     * whether to find reference's instance from the current JVM
     */
    protected Boolean injvm;

    /**
     * 惰性创建连接
     * lazy create connection
     */
    protected Boolean lazy;

    /**
     * 重连
     */
    protected String reconnect;
	
    protected Boolean sticky;

    /**
     * stub中是否支持事件
     * Constants.DEFAULT_STUB_EVENT
     */
    protected Boolean stubevent;

    /**
     * 默认版本
     */
    protected String version;

    protected String group;

    @Parameter(excluded = true)
    public Boolean isGeneric() {
        return ProtocolUtils.isGeneric(generic);
    }

    public void setGeneric(Boolean generic) {
        if (generic != null) {
            this.generic = generic.toString();
        }
    }

    @Override
    @Parameter(key = Constants.REFERENCE_FILTER_KEY, append = true)
    public String getFilter() {
        return super.getFilter();
    }

    @Override
    @Parameter(key = Constants.INVOKER_LISTENER_KEY, append = true)
    public String getListener() {
        return super.getListener();
    }

    @Override
    public void setListener(String listener) {
        checkMultiExtension(InvokerListener.class, "listener", listener);
        super.setListener(listener);
    }

    @Parameter(key = Constants.LAZY_CONNECT_KEY)
    public Boolean getLazy() {
        return lazy;
    }

    @Override
    public void setOnconnect(String onconnect) {
        if (onconnect != null && onconnect.length() > 0) {
            this.stubevent = true;
        }
        super.setOnconnect(onconnect);
    }

    @Override
    public void setOndisconnect(String ondisconnect) {
        if (ondisconnect != null && ondisconnect.length() > 0) {
            this.stubevent = true;
        }
        super.setOndisconnect(ondisconnect);
    }

    @Parameter(key = Constants.STUB_EVENT_KEY)
    public Boolean getStubevent() {
        return stubevent;
    }

    @Parameter(key = Constants.RECONNECT_KEY)
    public String getReconnect() {
        return reconnect;
    }

    @Parameter(key = Constants.CLUSTER_STICKY_KEY)
    public Boolean getSticky() {
        return sticky;
    }
}
```

##### ProtocolConfig
```java
public class ProtocolConfig extends AbstractConfig {

    private static final long serialVersionUID = 6913423882496634749L;

    /**
     * 性能调优(必填dubbo)
     * 协议名称
     * 	<protocol>
     */
    private String name;

    /**
     * 服务发现(自动查找本机IP)
     * 服务主机名，多网卡选择或指定VIP及域名时使用，为空则自动查找本机IP，-建议不要配置，让Dubbo自动获取本机IP
     * host
     */
    private String host;

    /**
     * 服务端口
     * dubbo协议缺省端口为20880,rmi协议缺省端口为1099,http 和hessian协议缺省端口为80
     * 如果配置为-1 或者没有配置port，则会分配一个没有被占用的端口
     * Dubbo 2.4.0+，分配的端口在协议缺省端口的基础上增长，确保端口段可控
     * <port>
     */
    private Integer port;

    /**
     * 服务发现
     * 提供者上下文路径，为服务path的前缀
     * <path>
     */
    private String contextpath;

    /**
     * 性能调优（fixed）
     * 线程池类型，可选：fixed/cached
     * threadpool
     */
    private String threadpool;

    /**
     * 性能调优(100)
     * 服务线程池大小(固定大小)
     * threads
     */
    private Integer threads;

    /**
     * 性能调优(cpu个数+1)
     * IO线程池大小（固定大小）
     */
    private Integer iothreads;

    /**
     * 性能调优(0)
     * 线程池队列大小，当线程池满时，排队等待执行的队列大小，
     * 建议不要设置，当线程程池时应立即失败，重试其它服务提供机器，而不是排队，除非有特殊需求
     * queues
     */
    private Integer queues;

    /**
     * 性能调优(0)
     * 服务提供方最大可接受连接数
     * accepts 
     */
    private Integer accepts;

    /**
     * 性能调优(dubbo)
     * 协议编码方式
     * codec
     */
    private String codec;

    /**
     * 性能调优(dubbo协议缺省为hessian2，rmi协议缺省为java，http协议 缺省为json)
     * 协议序列化方式，当协议支持多种序列化方式时使用，比如：dubbo协议的dubbo,hessian2,java,compactedjava，以及http协议的json等
     * serialization
     */
    private String serialization;

    /**
     * 序列化编码(UTF-8)
     * charset
     */
    private String charset;

    /**
     * 性能调优(88388608(=8M))
     * 请求及响应数据包大小限制，单位：字节
     * payload
     */
    private Integer payload;

    /**
     * 性能调优(8192)
     * 网络读写缓冲区大小
     * buffer
     */
    private Integer buffer;

    /**
     * 性能调优（0）
     * 心跳间隔，对于长连接，当物理层断开时，比如拔网线，TCP的FIN消息来不及发送，
     * 对方收不到断开事件，此时需要心跳来帮助检查连接是否已断开
     * heartbeat
     */
    private Integer heartbeat;

    /**
     * 服务治理
     * 设为true，将向logger中输出访问日志，也可填写访问日志文件路径，直接把访问日志输出到指定文件
     * accesslog
     */
    private String accesslog;
	
    /**
     * 性能调优(dubbo协议缺省为netty)
     * 协议的服务端和客户端实现类型，比如：dubbo协议的mina,netty等，可以分拆为server和client配置
     * transporter
     */
    private String transporter;

    private String exchanger;

    /**
     * 性能(dubbo协议缺省为all)
     * 协议的消息派发方式，用于指定线程模型，比如：dubbo协议的all,direct,message,execution,connection等
     * dispatcher
     */
    private String dispatcher;

    /**
     * networker
     */
    private String networker;

    /**
     * 性能调优(dubbo协议缺省为netty，http协议缺省为servlet)
     * 协议的服务器端实现类型，比如：dubbo协议的mina,netty等，http协议的jetty,servlet等
     * server
     */
    private String server;

    /**
     * 性能调优(协议缺省为netty)
     * 客户端实现
     * client
     */
    private String client;

    /**
     * 服务治理
     * 支持的telnet命令，逗号分隔
     * telnet
     */
    private String telnet;

    /**
     * 命令行提示
     */
    private String prompt;

    /**
     * 状态检测
     */
    private String status;

    /**
     * 服务治理（true）
     * 该协议的服务是否注册到注册中心
     */
    private Boolean register;

    /**
     * 是否长连接
     * TODO add this to provider config
     */
    private Boolean keepAlive;

    /**
     * TODO add this to provider config
     */
    private String optimizer;

    private String extension;

    private Map<String, String> parameters;

    private Boolean isDefault;
  
    /**
     * 是否已经销毁
     */
    private static final AtomicBoolean destroyed = new AtomicBoolean(false);

    public ProtocolConfig() {
    }

    public ProtocolConfig(String name) {
        setName(name);
    }

    public ProtocolConfig(String name, int port) {
        setName(name);
        setPort(port);
    }

    @Parameter(excluded = true)
    public String getName() {
        return name;
    }

    public void setName(String name) {
        checkName("name", name);
        this.name = name;
        if (id == null || id.length() == 0) {
            id = name;
        }
    }

    @Parameter(excluded = true)
    public String getHost() {
        return host;
    }

    @Parameter(excluded = true)
    public Integer getPort() {
        return port;
    }

    @Parameter(excluded = true)
    public String getContextpath() {
        return contextpath;
    }

    public void setContextpath(String contextpath) {
        checkPathName("contextpath", contextpath);
        this.contextpath = contextpath;
    }

    public void setThreadpool(String threadpool) {
        checkExtension(ThreadPool.class, "threadpool", threadpool);
        this.threadpool = threadpool;
    }

    public void setCodec(String codec) {
        if ("dubbo".equals(name)) {
            checkMultiExtension(Codec.class, "codec", codec);
        }
        this.codec = codec;
    }

    public void setSerialization(String serialization) {
        if ("dubbo".equals(name)) {
            checkMultiExtension(Serialization.class, "serialization", serialization);
        }
        this.serialization = serialization;
    }

    public void setServer(String server) {
        if ("dubbo".equals(name)) {
            checkMultiExtension(Transporter.class, "server", server);
        }
        this.server = server;
    }

    public void setClient(String client) {
        if ("dubbo".equals(name)) {
            checkMultiExtension(Transporter.class, "client", client);
        }
        this.client = client;
    }

    public void setTelnet(String telnet) {
        checkMultiExtension(TelnetHandler.class, "telnet", telnet);
        this.telnet = telnet;
    }

    @Parameter(escaped = true)
    public String getPrompt() {
        return prompt;
    }

    public void setStatus(String status) {
        checkMultiExtension(StatusChecker.class, "status", status);
        this.status = status;
    }

    public void setTransporter(String transporter) {
        checkExtension(Transporter.class, "transporter", transporter);
        this.transporter = transporter;
    }

    public void setExchanger(String exchanger) {
        checkExtension(Exchanger.class, "exchanger", exchanger);
        this.exchanger = exchanger;
    }

    public void setDispatcher(String dispatcher) {
        checkExtension(Dispatcher.class, "dispacther", dispatcher);
        this.dispatcher = dispatcher;
    }

    public void destory() {
        if (name != null) {
	    //获取Protocol扩展实例，然后调用destroy方法
            ExtensionLoader.getExtensionLoader(Protocol.class)
                    .getExtension(name)
                    .destroy();
        }
    }
}
```
##### ServiceConfig
```java
public class ServiceConfig<T> extends AbstractServiceConfig {

    /**
     * 服务发现
     * 服务接口名称
     */
    private String interfaceName;
   
    /**
     * 服务接口类
     */
    private Class<?> interfaceClass;
   
    /**
     * 服务发现(必填)
     * 服务对象实现引用
     */
    private T ref;
    
    /**
     * 服务发现(缺省为接口名)
     * 服务路径(注意：1.0不支持自定义路径，总是使用接口名，如果有1.0调 2.0，配置服务路径可能不兼容
     * <path>
     */
    private String path;
    
    /**
     * 服务方法配置
     */
    private List<MethodConfig> methods;
   
    /**
     * 提供者
     */
    private ProviderConfig provider;


    /**
     * 泛化接口
     */
    private volatile String generic;
}

```
##### ReferenceConfig
```java
public class ReferenceConfig<T> extends AbstractReferenceConfig {

    /**
     * 引用的接口名称
     */
    private String interfaceName;

    /**
     * 引用的接口类
     */
    private Class<?> interfaceClass;
    
    /**
     * 客户端类型
     */
    private String client;
    
    /**
     * 点对点调用url
     */
    private String url;
    
    /**
     * 引用接口方法配置
     */
    private List<MethodConfig> methods;
    
    /**
     * 默认配置
     */
    private ConsumerConfig consumer;
    
    /**
     * 协议
     */
    private String protocol;

    /**
     * 接口代理引用
     * interface proxy reference
     */
    private transient volatile T ref;

    private transient volatile Invoker<?> invoker;

    /**
     * 是否已初始化
     */
    private transient volatile boolean initialized;

    /**
     * 是否已销毁
     */
    private transient volatile boolean destroyed;
}
```

### 覆盖和优先级
以timeout为例，这里按照优先级从高到低排列(retries,loadbalance, actives也应用相同的规则)：
* 方法级别，接口级别，默认/全局级别
* 相同的级别下，消费者比提供者有更高的优先级
![](img/level.jpg)

这一小节我们就先介绍到这里，下一小节开始介绍具体的解析逻辑。

