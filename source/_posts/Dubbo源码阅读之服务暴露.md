---
title: Dubbo源码阅读之服务暴露
date: 2018-08-16 21:17:19
tags:
    - dubbo
---
>本小节详细介绍dubbo服务的暴露，可以先看下之前的文章《Dubbo源码阅读之集成Spring(0201)》

### ServiceConfig类变量
在ServiceConfig类中定义了如下变量，下文中会用到，我们先简单看下：
```java
//代理工厂
private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();

//Protocol实例
private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();

/**
 * dubbo协议服务URL
 * 记录暴露的服务URL
 * dubbo://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=192.168.99.60&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=4328&qos.port=22222&side=provider&timestamp=1528278313225
 */
private final List<URL> urls = new ArrayList<URL>();

/**
 * 记录暴露的服务端点
 * subscribeUrl = provider://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&category=configurators&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=4328&side=provider&timestamp=1528278313225
 * registerUrl = dubbo://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=192.168.99.60&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=4328&qos.port=22222&side=provider&timestamp=1528278313225
 */
private final List<Exporter<?>> exporters = new ArrayList<Exporter<?>>();
```


首先我们先来了解下ProxyFactory接口

### ProxyFactory接口

#### getInvoker方法

getInvoker方法是在ProxyFactory接口中定义的。ProxyFactory接口相关类图如下：
![](img/ProxyFactory.png)

```java
/**
 * 代理工厂
 */
@SPI("javassist")
public interface ProxyFactory {
    /**
     * 创建代理
     * @param invoker
     * @return proxy
     */
    @Adaptive({Constants.PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;

    /**
     * 创建invoker
     * @param <T>
     * @param proxy 接口实现类或者代理类
     * @param type 接口类型
     * @param url 注册中心url，本地为injvm
     * @return invoker
     */
    @Adaptive({Constants.PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;
}

/**
 * 抽象ProxyFactory
 */
public abstract class AbstractProxyFactory implements ProxyFactory {
    @Override
    public <T> T getProxy(Invoker<T> invoker) throws RpcException {
        //接口数组
        Class<?>[] interfaces = null;
        //获取url的interfaces参数的值
        String config = invoker.getUrl().getParameter("interfaces");
        if (config != null && config.length() > 0) {
            //使用逗号","分隔interfaces参数值
            String[] types = Constants.COMMA_SPLIT_PATTERN.split(config);
            if (types != null && types.length > 0) {
                //这里会将"远程服务接口类、EchoService类"放入到接口数组中
                interfaces = new Class<?>[types.length + 2];
                interfaces[0] = invoker.getInterface();
                interfaces[1] = EchoService.class;
                //然后将"interfaces参数的值"放入到接口数组
                for (int i = 0; i < types.length; i++) {
                    interfaces[i + 1] = ReflectUtils.forName(types[i]);
                }
            }
        }
        if (interfaces == null) {
            //interfaces数组为空的话，则将远程服务接口类、EchoService类放入到接口数组中
            interfaces = new Class<?>[]{invoker.getInterface(), EchoService.class};
        }
        //然后调用子类体实现来获取代理类
        return getProxy(invoker, interfaces);
    }

    /**
     * 获取代理类
     * @param invoker
     * @param types  远程服务接口类数组：
     *               invoker.getInterface()、
     *               EchoService.class、
     *               invoker.getUrl().getParameter("interfaces")
     * @param <T>
     * @return
     */
    public abstract <T> T getProxy(Invoker<T> invoker, Class<?>[] types);
}

/**
 * jdk动态代理
 */
public class JdkProxyFactory extends AbstractProxyFactory {

    @Override
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        //为interfaces这些接口生成代理类
        return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                interfaces, new InvokerInvocationHandler(invoker));
    }

    @Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
	//返回AbstractProxyInvoker抽象类（该类后面的小节会详细介绍）
        return new AbstractProxyInvoker<T>(proxy, type, url) {

            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {

                //根据方法名称methodName和参数类型parameterTypes获取到指定方法
                Method method = proxy.getClass().getMethod(methodName, parameterTypes);

                //使用参数arguments调用该代理类的method方法
                return method.invoke(proxy, arguments);
            }
        };
    }
}

/**
 * 使用Javassist
 */
public class JavassistProxyFactory extends AbstractProxyFactory {

    @Override
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        //为interfaces这些接口生成代理类(后面会分析Proxy类，这是dubbo自己定义的类)
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }

    @Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        
	// TODO Wrapper类不能正确处理带$的类名
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        
	//返回AbstractProxyInvoker抽象类（该类后面的小节会详细介绍）
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {

                //使用Wrapper包装类类调用远程服务方法
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }
}
```
在JavassistProxyFactory类的getInvoker方法实现中，会为相应的接口类(例如：com.alibaba.dubbo.demo.DemoService)生成包装类，生成的逻辑会在单独的小节介绍，这里只简单看下大概生成的类。
#### Wrapper包装类
```java
public class com.alibaba.dubbo.common.bytecode.Wrapper0 extends Wrapper{
	public Wrapper0(){}
	//属性名称数组
	public static String[] pns;

	//<属性名称，属性类型>
	public static java.util.Map pts;
	
	//所有方法名称数组
	public static String[] mns;
	
	//已声明的方法名称数组
	public static String[] dmns;
	
	//针对每个方法都会定义一个mts数组变量
	public static Class[] mts0;

	public String[] getPropertyNames(){ 
		return pns; 
	}
	public boolean hasProperty(String n){
		return pts.containsKey($1); 
	}
	public Class getPropertyType(String n){ 
		return (Class)pts.get($1); 
	}
	public String[] getMethodNames(){ 
		return mns; 
	}
	public String[] getDeclaredMethodNames(){ 
		return dmns; 
	}
	public void setPropertyValue(Object o, String n, Object v){ 
		com.alibaba.dubbo.demo.DemoService w; 
		try{ 
			w = ((com.alibaba.dubbo.demo.DemoService)$1); 
		}catch(Throwable e){ 
			throw new IllegalArgumentException(e); 
		} 
		throw new com.alibaba.dubbo.common.bytecode.NoSuchPropertyException("Not found property \""+$2+"\" filed or setter method in class com.alibaba.dubbo.demo.DemoService."); 
	}
	public Object getPropertyValue(Object o, String n){ 
		com.alibaba.dubbo.demo.DemoService w; 
		try{ 
			w = ((com.alibaba.dubbo.demo.DemoService)$1); 
		}catch(Throwable e){ 
			throw new IllegalArgumentException(e); 
		} 
		throw new com.alibaba.dubbo.common.bytecode.NoSuchPropertyException("Not found property \""+$2+"\" filed or setter method in class com.alibaba.dubbo.demo.DemoService."); 
	}
	/**
	 * 执行服务接口的方法
	 * $1/$2/$3/$4分别对应相应下标的参数
	 * @param o 服务接口实例
	 * @param n 服务接口方法名
	 * @param p 服务接口方法参数
	 * @param v 服务接口方法参数值
	 * @return 服务接口调用返回值
	 */
	public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws java.lang.reflect.InvocationTargetException{
		com.alibaba.dubbo.demo.DemoService w; 
		try{ 
			w = ((com.alibaba.dubbo.demo.DemoService)$1); 
		}catch(Throwable e){ 
			throw new IllegalArgumentException(e); 
		} 
		try{ 
			//方法名称为sayHello 并且 只有一个参数
			if("sayHello".equals($2) && $3.length == 1){  
				//调用接口方法sayHello
				return ($w)w.sayHello((java.lang.String)$4[0]);
			} 
		} catch(Throwable e) {      
			throw new java.lang.reflect.InvocationTargetException(e);  
		} 
		throw new com.alibaba.dubbo.common.bytecode.NoSuchMethodException("Not found method \""+$2+"\" in class com.alibaba.dubbo.demo.DemoService."); 
	}
}
```

### 暴露到注册中心

我们先看下DelegateProviderMetaDataInvoker类。
```java
/**
 * DelegateProviderMetaDataInvoker类实现了invoker接口
 * 并且持有一个Invoker变量，操作都会委托给该变量
 */
public class DelegateProviderMetaDataInvoker<T> implements Invoker {
    protected final Invoker<T> invoker;
    private ServiceConfig metadata;

    public DelegateProviderMetaDataInvoker(Invoker<T> invoker,ServiceConfig metadata) {
        this.invoker = invoker;
        this.metadata = metadata;
    }

    @Override
    public Class<T> getInterface() {
        return invoker.getInterface();
    }

    @Override
    public URL getUrl() {
        return invoker.getUrl();
    }

    @Override
    public boolean isAvailable() {
        return invoker.isAvailable();
    }

    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        return invoker.invoke(invocation);
    }

    @Override
    public void destroy() {
        invoker.destroy();
    }

    public ServiceConfig getMetadata() {
        return metadata;
    }
}
```
#### Protocol接口的export方法

先来看下Protocol接口的定义，其export方法和refer方法存在@Adaptive注解。
```java
/**
 * 协议
 * Protocol. (API/SPI, Singleton, ThreadSafe)
 */
@SPI("dubbo")
public interface Protocol {

    /**
     * 当用户没有配置端口时，获取默认端口
     * @return default port
     */
    int getDefaultPort();

    /**
     * 为远程调用暴露服务
     * 1、接收到请求后协议应该记录请求源地址，通过: RpcContext.getContext().setRemoteAddress()
     * 2、export()方法必须是幂等的，也就是说，暴露同一个URL的invoker两次时，调用1次和2次没有什么不同
     * 3、传入的Invoker实例由框架实现并传入，协议不需要关心
     *
     * @param <T>     服务类型
     * @param invoker 服务invoker
     * @return 暴露服务的exporter引用，用来以后取消暴露服务
     * @throws RpcException 当暴露服务出错时抛出，比如端口已占用
     */
    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    /**
     * 引用一个远程服务
     * 1、当用户调用 refer()所返回的 Invoker对象的invoke()时，协议需相应执行同URL远端export()传入的Invoker对象的invoke()方法.
     * 2、refer()返回的Invoker由协议实现，一般来说，协议需要在此Invoker中发送远程请求。
     * 3、当URL中设置了check=false时，这个实现不可以抛出异常，而是尝试着从连接失败中恢复
     * @param T 远程服务类型
     * @param type 远程服务接口类型
     * @param url 远程服务的URL地址
     * @return 执行服务的本地代理
     * @throws RpcException when there's any error while connecting to the service provider
     */
    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    /**
     * 销毁协议
     * 1、取消该协议所有已经暴露和引用的服务
     * 2、释放协议所有占用的资源，如：链接、端口等
     * 3、协议在释放后，依然能暴露和引用新的服务
     */
    void destroy();
}

可以看到export方法是定义在Protocol接口中的，我们是通过ExtensionLoader类获取到Protocol的自适应扩展实例的，因此我们在调用export方法进行服务暴露时，实际上调用的是自适应扩展实例的export方法：
```java
//自适应扩展实现类
Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();

//获取invoker对象
Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
//将invoker和当前对象(ServiceBean)包装成DelegateProviderMetaDataInvoker对象
DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

//调用自适应扩展实现类的export方法
Exporter<?> exporter = protocol.export(wrapperInvoker);
```
那么，在获取自适应扩展实例时都进行了哪些操作呢？我们在《Dubbo源码阅读之SPI扩展机制》文章中介绍过Dubbo的SPI机制，这里就不再详细介绍了，只简单说明下是如何获取到Protocol实例的。
首先我们我们通过ExtensionLoader.getExtensionLoader(Protocol.class)调用，获取到了Protocol的扩展加载器，然后调用它的getAdaptiveExtension()方法拿到自适应扩展实例(Dubbo为它自动生成的一个类，后面介绍)，在此过程中，会先去扫描3个相关路径找到所有的Protocol扩展实现类定义，
最终会找到如下定义：
```java
#扩展名称=扩展实现类
registry=com.alibaba.dubbo.registry.integration.RegistryProtocol
mock=com.alibaba.dubbo.rpc.support.MockProtocol
injvm=com.alibaba.dubbo.rpc.protocol.injvm.InjvmProtocol
dubbo=com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol
#Protocol包装类
filter=com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper
listener=com.alibaba.dubbo.rpc.protocol.ProtocolListenerWrapper
```

然后检查每个扩展实现类，如果没有找到存在@Adaptive注解的扩展类，则Dubbo会为其生成一个自适应扩展类，生成的实现类如下：
```java
/**
 * 生成的实现类名为：Protocol$Adaptive，实现了Protocol接口
 * 只有接口方法上标有@Adaptive注解，才会为其生成实现，
 * 例如，Protocol接口的destroy()方法没有@Adaptive注解，因此下面的方法实现中直接抛了异常
 */
public class Protocol$Adaptive implements com.alibaba.dubbo.rpc.Protocol {
	
	public void destroy() {
		//提示 Protocol接口的destroy()方法没有@Adaptive注解，因此不支持
		throw new UnsupportedOperationException("method public abstract void com.alibaba.dubbo.rpc.Protocol.destroy() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
	}
	
	public int getDefaultPort() {
		//提示 Protocol接口的getDefaultPort()方法没有@Adaptive注解，因此不支持
		throw new UnsupportedOperationException("method public abstract int com.alibaba.dubbo.rpc.Protocol.getDefaultPort() of interface com.alibaba.dubbo.rpc.Protocol is not adaptive method!");
	}
	
	/**
	 * @param arg0 服务接口类
	 * @param arg1 注册中心url
	 */
	public com.alibaba.dubbo.rpc.Invoker refer(java.lang.Class arg0, com.alibaba.dubbo.common.URL arg1) throws com.alibaba.dubbo.rpc.RpcException {
		if (arg1 == null){
			//url参数不可以为空
			throw new IllegalArgumentException("url == null");
		}
		
		com.alibaba.dubbo.common.URL url = arg1;
		
		//使用url的protocol属性值(这里为registry)，作为扩展名称，如果为空，则默认使用"dubbo"
		String extName = url.getProtocol() == null ? "dubbo" : url.getProtocol();
		
		if(extName == null) {
			throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
		}
		
		//获取扩展名称为extName的Protocol扩展实例
		com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
		
		//调用该extension实例的refer方法
		return extension.refer(arg0, arg1);
	}
		
	public com.alibaba.dubbo.rpc.Exporter export(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
		if (arg0 == null){
		    throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
		} 
		
		if (arg0.getUrl() == null) {
		    throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
		}
		
		com.alibaba.dubbo.common.URL url = arg0.getUrl();
		
		//使用url的protocol属性值(这里为registry)，作为扩展名称，如果为空，则默认使用"dubbo"
		String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
		
		if(extName == null) {
			throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
		}
		
		//获取扩展名称为extName的Protocol扩展实例（此扩展实例已经被包装类包装过）
		com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
		
		//调用该extension实例的export方法(包装类的export方法)
		return extension.export(arg0);
	}
}
```
在上面我们说到，调用export方法实际上是调用自适应扩展实现类的export方法，我们来看下其实现。
首先校验invoker参数不可以为空，然后校验invoker对象的getUrl()方法返回的URL对象不可以为空，然后我们使用URL对象的protocol属性作为扩展名称extName(提示：在上一节的loadRegistries(boolean provider)加载注册中心url列表方法中，我们设置了注册中心url的protocol属性值为"registry")，然后根据扩展名称extName获取到真正的Protocol扩展实例(RegistryProtocol,然后RegistryProtocol会被包装)，最后调用该Protocol扩展实例的export方法。
通过生成的自适应扩展实现类，我们可以根据不同的扩展名称调用不同的扩展实例。
在getExtension(extName)方法中，获取到相应的扩展实例后(RegistryProtocol)，Dubbo会再去查看下该扩展是否存在相应的包装类，在上面我们扫描出来Protocol接口存在2个包装类，ProtocolFilterWrapper和ProtocolListenerWrapper，接着Dubbo会创建包装类的实例，并将刚才生成RegistryProtocol扩展实例通过构造函数参数传递到包装类中，通过包装类，Dubbo就实现了功能增强。
因此，现在我们再回过头来看下export方法的调用：首先是调用自适应扩展实现类的export方法，在export方法内部，根据扩展名称获取到真实的扩展实例RegistryProtocol，然后又通过两个包装类将真实的扩展实例进行了包装，最后调用的是包装类的export方法。
具体的调用链我们已经分析完了，现在我们来看下对应的代码。
```java
/**
 * 根据扩展名称创建扩展实例后，会获取包装类，将扩展实例进行包装
 */
private T createExtension(String name) {
	//根据扩展名称name获取扩展实现类clazz
	Class<?> clazz = getExtensionClasses().get(name);
	if (clazz == null) {
	    //扩展实现类为空，则抛出异常
	    throw findException(name);
	}
	try {
	    //先从缓存中获取扩展实现类的实例，如果没有获取到，则新建一个并放入缓存。
	    T instance = (T) EXTENSION_INSTANCES.get(clazz);
	    if (instance == null) {
		EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
		instance = (T) EXTENSION_INSTANCES.get(clazz);
	    }
	    //处理扩展实现类实例的依赖注入
	    injectExtension(instance);

	    //获取到type接口的所有包装类
	    Set<Class<?>> wrapperClasses = cachedWrapperClasses;
	    if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
		//遍历包装类
		for (Class<?> wrapperClass : wrapperClasses) {
		    //包装类通过构造方法创建实例，然后进行依赖注入
		    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
		}
	    }
	    return instance;
	} catch (Throwable t) {
	    throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
		    type + ")  could not be instantiated: " + t.getMessage(), t);
	}
}

public class ProtocolFilterWrapper implements Protocol {
	
    //根据上面的分析，这里的protocol是ProtocolListenerWrapper
    private final Protocol protocol;

    public ProtocolFilterWrapper(Protocol protocol) {
        if (protocol == null) {
            throw new IllegalArgumentException("protocol == null");
        }
        this.protocol = protocol;
    }
    
    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        
        if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
	    //如果url的protocol属性为registry，则调用protocol的export方法
            return protocol.export(invoker);
        }
        return protocol.export(buildInvokerChain(invoker, Constants.SERVICE_FILTER_KEY, Constants.PROVIDER));
    }
    //.....省略其他方法....
}


public class ProtocolListenerWrapper implements Protocol{
	
    //根据上面的分析，这里的protocol是RegistryProtocol
    private final Protocol protocol;

    public ProtocolListenerWrapper(Protocol protocol) {
        if (protocol == null) {
            throw new IllegalArgumentException("protocol == null");
        }
        this.protocol = protocol;
    }
    
     @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        if (Constants.REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
	    //如果url的protocol属性为registry，则调用protocol的export方法
            return protocol.export(invoker);
        }
        return new ListenerExporterWrapper<T>(protocol.export(invoker),
                Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(ExporterListener.class)
                        .getActivateExtension(invoker.getUrl(), Constants.EXPORTER_LISTENER_KEY)));
    }
}
```
经过我们的分析，最终会调用RegistryProtocol的export方法进行服务暴露，接下来，我们就来分析下该方法。
```java

//获取invoker对象
Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));

//将invoker和当前对象(ServiceBean)包装成DelegateProviderMetaDataInvoker对象
DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);



/**
 * 通过invoker的url(即注册中心url)获取providerUrl的地址（即服务暴露的url）
 * @param origininvoker
 * @return
 */
private URL getProviderUrl(final Invoker<?> origininvoker) {
	//获取参数export指定的服务地址
	String export = origininvoker.getUrl().getParameterAndDecoded(Constants.EXPORT_KEY);
	if (export == null || export.length() == 0) {
	    //注册中心服务暴露url为空
	    throw new IllegalArgumentException("The registry export url is null! registry: " + origininvoker.getUrl());
	}
	//根据url字符串生成URL对象
	URL providerUrl = URL.valueOf(export);
	return providerUrl;
}

/**
 * 获取originInvoker对象对应的缓存key（移除了dynamic、enabled属性的服务提供者url）
 */
private String getCacheKey(final Invoker<?> originInvoker) {
	//获取服务暴露的url，即export参数对应的url
	URL providerUrl = getProviderUrl(originInvoker);
	//移除dynamic、enabled参数，剩下的url作为缓存key
	String key = providerUrl.removeParameters("dynamic", "enabled").toFullString();
	return key;
}

/**
 * 获取注册中心url(如果是注册中心协议，则修改了protocol属性值，属性值为url的registry参数值，并移除url的registry参数)
 * @param originInvoker
 * @return
 */
private URL getRegistryUrl(Invoker<?> originInvoker) {
	//注册中心url
	URL registryUrl = originInvoker.getUrl();
	
	if (Constants.REGISTRY_PROTOCOL.equals(registryUrl.getProtocol())) {
	    //如果registryUrl的protocol属性为"registry"，则获取registryUrl的registry参数值，如果没有获取到，则默认protocol = dubbo
	    //这里获取到protocol = multicast
	    String protocol = registryUrl.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_DIRECTORY);
	    //设置registryUrl的protocol属性值，并移除registry参数
	    registryUrl = registryUrl.setProtocol(protocol).removeParameter(Constants.REGISTRY_KEY);
	}
	return registryUrl;
}

/**
 *
 * 基于invoker的地址获取一个注册中心的实例
 * @param originInvoker
 * @return
 */
private Registry getRegistry(final Invoker<?> originInvoker) {
	URL registryUrl = getRegistryUrl(originInvoker);
	//获取实例
	return registryFactory.getRegistry(registryUrl);
}

/**
 * 返回注册到注册中心的地址并过滤一次url参数(即服务提供者url)
 * @param originInvoker
 * @return
 */
private URL getRegistedProviderUrl(final Invoker<?> originInvoker) {
	//服务暴露的url，即export参数指定的url
	URL providerUrl = getProviderUrl(originInvoker);

	//在注册中心中看到的服务地址
	final URL registedProviderUrl = providerUrl.removeParameters(
			//删除不需要输出的参数key
			getFilteredKeys(providerUrl)
		)
		//移除监控
		.removeParameter(Constants.MONITOR_KEY)
		//移除绑定ip、port
		.removeParameter(Constants.BIND_IP_KEY)
		.removeParameter(Constants.BIND_PORT_KEY)
		//移除qos
		.removeParameter(QOS_ENABLE)
		.removeParameter(QOS_PORT)
		.removeParameter(ACCEPT_FOREIGN_IP);
	
	return registedProviderUrl;
}

/**
 * 服务暴露
 * originInvoker参数就是上面我们创建的wrapperInvoker
 */
@Override
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
	//暴露invoker(后面会分析该方法)
	final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
	
	//获取注册中心地址,如：multicast://224.5.6.7:1234/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&export=dubbo%3A%2F%2F192.168.99.60%3A20880%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26bind.ip%3D192.168.99.60%26bind.port%3D20880%26dubbo%3D2.0.0%26generic%3Dfalse%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D7520%26qos.port%3D22222%26side%3Dprovider%26timestamp%3D1528784828723&pid=7520&qos.port=22222&timestamp=1528784779675
	URL registryUrl = getRegistryUrl(originInvoker);

	//获取注册中心实例
	final Registry registry = getRegistry(originInvoker);
	
	//注册中心url的export参数指定的url(服务提供者url)，去掉了一些key
	final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);
	
	//判断是否延迟发布，默认是true
	boolean register = registedProviderUrl.getParameter("register", true);
	
	//注册服务提供者(保存到map缓存中)
	ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registedProviderUrl);

	if (register) {
	    //registryUrl注册中心url，registedProviderUrl服务提供者url
	    //获取注册中心，并将服务提供者注册到注册中心
	    register(registryUrl, registedProviderUrl);
	    
	    //根据originInvoker获取到ProviderInvokerWrapper并标识isReg = true
	    ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
	}

	// 当提供者订阅时，会影响一个场景，即同一JVM即暴露服务，又引用同一服务。
	// 因为subscribed使用服务的名称作为缓存key，它会导致订阅信息被覆盖
	//如：provider://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&category=configurators&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=5452&side=provider&timestamp=1528789631708
	final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
	
	//创建OverrideListener对象(后面小节会介绍)
	final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
	
	//保存overrideSubscribeUrl和overrideSubscribeListener
	overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
	
	//订阅服务(后面注册中心小节会介绍)
	registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
	
	//确保每次发布时都返回一个新的exporter实例
	return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registedProviderUrl);
}

/**
 * subscribedOverrideUrl
 * @param registedProviderUrl 服务提供者url
 * @return
 */
private URL getSubscribedOverrideUrl(URL registedProviderUrl) {
	//设置protocol = provider,修改前 protocol = dubbo
	//设置category = configurators
	//设置check = false
	return registedProviderUrl.setProtocol(Constants.PROVIDER_PROTOCOL)
		.addParameters(Constants.CATEGORY_KEY, Constants.CONFIGURATORS_CATEGORY,
			Constants.CHECK_KEY, String.valueOf(false));
}

/**
 * 通过registryUrl获取注册中心，然后将服务提供者url注册到注册中心
 * @param registryUrl 注册中心url
 * @param registedProviderUrl 服务提供者url
 */
public void register(URL registryUrl, URL registedProviderUrl) {
	//根据注册中心url 获取注册中心实例
	Registry registry = registryFactory.getRegistry(registryUrl);
	//注册 服务提供者 到注册中心
	registry.register(registedProviderUrl);
}

/**
 * 为了解决RMI反复暴露端口冲突的问题，已经暴露的服务不需要再次暴露
 * <providerurl,exporter>
 */
private final Map<String, ExporterChangeableWrapper<?>> bounds = new ConcurrentHashMap<String, ExporterChangeableWrapper<?>>();

/**
 * <服务提供者url,NotifyListener>
 */
private final Map<URL, NotifyListener> overrideListeners = new ConcurrentHashMap<URL, NotifyListener>();

/**
 * 暴露originInvoker
 */
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker) {
	//获取缓存key
	String key = getCacheKey(originInvoker);
	
	//先检查是否暴露过，没有暴露过的话，则进行暴露
	ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
	if (exporter == null) {
	    synchronized (bounds) {
		exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
		if (exporter == null) {
		    //根据originInvoker，及其服务提供者url，创建InvokerDelegete对象
		    final Invoker<?> invokerDelegete = new InvokerDelegete<T>(originInvoker, getProviderUrl(originInvoker));
		   
		    //1、因为invokerDelegete对象的getUrl方法将会返回服务提供者url，其protocol属性将会返回扩展名称dubbo，因此这里会调用DubboProtocol的export方法进行暴露
		    //2、创建ExporterChangeableWrapper对象
		    exporter = new ExporterChangeableWrapper<T>((Exporter<T>) protocol.export(invokerDelegete), originInvoker);
		    
		    //dubbo://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=192.168.99.60&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=6440&qos.port=22222&side=provider&timestamp=1528789363848
		    //URL url = exporter.getInvoker().getUrl();
		    //放入缓存
		    bounds.put(key, exporter);
		}
	    }
	}
	return exporter;
}

/**
 * Invoker委托类
 */
public static class InvokerDelegete<T> extends InvokerWrapper<T> {

	private final Invoker<T> invoker;

	/**
	 * @param invoker
	 * @param url   服务提供者url
	 */
	public InvokerDelegete(Invoker<T> invoker, URL url) {
	    super(invoker, url);
	    this.invoker = invoker;
	}

	public Invoker<T> getInvoker() {
	    if (invoker instanceof InvokerDelegete) {
		return ((InvokerDelegete<T>) invoker).getInvoker();
	    } else {
		return invoker;
	    }
	}
}

/**
 * exporter代理
 * 建立通过protocol已暴露的exporter和已返回的exporter的对应关系
 * 并且可以在覆盖时修改关系
 */
private class ExporterChangeableWrapper<T> implements Exporter<T> {

	private final Invoker<T> originInvoker;

	//protocol.export(invokerDelegete)返回的Exporter
	private Exporter<T> exporter;

	public ExporterChangeableWrapper(Exporter<T> exporter, Invoker<T> originInvoker) {
	    this.exporter = exporter;
	    this.originInvoker = originInvoker;
	}

	public Invoker<T> getOriginInvoker() {
	    return originInvoker;
	}

	@Override
	public Invoker<T> getInvoker() {
	    return exporter.getInvoker();
	}

	public void setExporter(Exporter<T> exporter) {
	    this.exporter = exporter;
	}

	@Override
	public void unexport() {
	    String key = getCacheKey(this.originInvoker);
	    bounds.remove(key);
	    exporter.unexport();
	}
}

/**
 * 可销毁的Exporter
 */
static private class DestroyableExporter<T> implements Exporter<T> {
	
	public static final ExecutorService executor = Executors.newSingleThreadExecutor(new NamedThreadFactory("Exporter-Unexport", true));

	private Exporter<T> exporter;
	private Invoker<T> originInvoker;

	//overrideSubscribeUrl 订阅的url
	private URL subscribeUrl; 
	
	//registedProviderUrl 注册的服务提供者url
	private URL registerUrl;

	public DestroyableExporter(Exporter<T> exporter, Invoker<T> originInvoker, URL subscribeUrl, URL registerUrl) {
	    this.exporter = exporter;
	    this.originInvoker = originInvoker;
	    this.subscribeUrl = subscribeUrl;
	    this.registerUrl = registerUrl;
	}

	@Override
	public Invoker<T> getInvoker() {
	    return exporter.getInvoker();
	}

	@Override
	public void unexport() {
	    //获取注册中心实例
	    Registry registry = RegistryProtocol.INSTANCE.getRegistry(originInvoker);
	    try {
	        //取消注册registerUrl
		registry.unregister(registerUrl);
	    } catch (Throwable t) {
		logger.warn(t.getMessage(), t);
	    }
	    try {
	        //移除监听
		NotifyListener listener = RegistryProtocol.INSTANCE.overrideListeners.remove(subscribeUrl);
		//取消订阅subscribeUrl
		registry.unsubscribe(subscribeUrl, listener);
	    } catch (Throwable t) {
		logger.warn(t.getMessage(), t);
	    }
	    //提交线程，取消暴露服务
	    executor.submit(new Runnable() {
		@Override
		public void run() {
		    try {
		        //取消暴露服务时，等待注册中心通知所有的消费者的超时时间
			int timeout = ConfigUtils.getServerShutdownTimeout();
			if (timeout > 0) {
			    logger.info("Waiting " + timeout + "ms for registry to notify all consumers before unexport. Usually, this is called when you use dubbo API");
			    Thread.sleep(timeout);
			}
			//取消暴露服务
			exporter.unexport();
		    } catch (Throwable t) {
			logger.warn(t.getMessage(), t);
		    }
		}
	    });
	}
}
```

ProviderConsumerRegTable类用来保存注册的服务提供者和消费者
```java
public class ProviderConsumerRegTable {
    /**
     * <服务唯一名称,Set<服务提供者执行器>>
     */
    public static ConcurrentHashMap<String, Set<ProviderInvokerWrapper>> providerInvokers = new ConcurrentHashMap<String, Set<ProviderInvokerWrapper>>();
    public static ConcurrentHashMap<String, Set<ConsumerInvokerWrapper>> consumerInvokers = new ConcurrentHashMap<String, Set<ConsumerInvokerWrapper>>();

    /**
     * 注册服务提供者
     * @param invoker
     * @param registryUrl 注册中心地址
     * @param providerUrl 服务提供者url
     */
    public static void registerProvider(Invoker invoker, URL registryUrl, URL providerUrl) {
	//创建ProviderInvokerWrapper实例
        ProviderInvokerWrapper wrapperInvoker = new ProviderInvokerWrapper(invoker, registryUrl, providerUrl);
        
	//获取服务唯一名称标识
        String serviceUniqueName = providerUrl.getServiceKey();
	
	//根据serviceUniqueName从缓存中获取
	Set<ProviderInvokerWrapper> invokers = providerInvokers.get(serviceUniqueName);
        
	if (invokers == null) {
            providerInvokers.putIfAbsent(serviceUniqueName, new ConcurrentHashSet<ProviderInvokerWrapper>());
            invokers = providerInvokers.get(serviceUniqueName);
        }
        invokers.add(wrapperInvoker);
    }
}

/**
 * 通过invoker 获取相应的ProviderInvokerWrapper
 * @param invoker
 * @return
 */
public static ProviderInvokerWrapper getProviderWrapper(Invoker invoker) {
	//获取服务提供者url
	URL providerUrl = invoker.getUrl();
	if (Constants.REGISTRY_PROTOCOL.equals(providerUrl.getProtocol())) {
	    //获取export参数的值，即(服务提供者url)
	    providerUrl = URL.valueOf(providerUrl.getParameterAndDecoded(Constants.EXPORT_KEY));
	}
	//获取服务唯一标识
	String serviceUniqueName = providerUrl.getServiceKey();
	//根据服务唯一标识获取invokers
	Set<ProviderInvokerWrapper> invokers = providerInvokers.get(serviceUniqueName);
	if (invokers == null) {
	    return null;
	}
	for (ProviderInvokerWrapper providerWrapper : invokers) {
	    Invoker providerInvoker = providerWrapper.getInvoker();
	    //通过invoker获取providerWrapper
	    if (providerInvoker == invoker) {
		return providerWrapper;
	    }
	}
	return null;
}


/**
 * 重新暴露修改了url的invoker
 * @param originInvoker
 * @param newInvokerUrl
 */
@SuppressWarnings("unchecked")
private <T> void doChangeLocalExport(final Invoker<T> originInvoker, URL newInvokerUrl) {
        //获取originInvoker对应的缓存key
	String key = getCacheKey(originInvoker);

	//从缓存中获取该exporter
	final ExporterChangeableWrapper<T> exporter = (ExporterChangeableWrapper<T>) bounds.get(key);
	
	if (exporter == null) {
	    logger.warn(new IllegalStateException("error state, exporter should not be null"));
	} else {
	    //重新生成invokerDelegete
	    final Invoker<T> invokerDelegete = new InvokerDelegete<T>(originInvoker, newInvokerUrl);
	    //重新暴露服务
	    exporter.setExporter(protocol.export(invokerDelegete));
	}
}
```

接下来我们看下OverrideListener类
```java
/**
 * 1、确保由registryProtocol返回的exporter可以被正常销毁
 * 2、通知后，无需重新注册到注册中心
 * 3、通过暴露方法传递invoker，最好是exporter的invoker
 */
private class OverrideListener implements NotifyListener {
	/**
	 * 已订阅的url
	 * protocol = provider
	 */
	private final URL subscribeUrl;

	private final Invoker originInvoker;

	public OverrideListener(URL subscribeUrl, Invoker originalInvoker) {
	    this.subscribeUrl = subscribeUrl;
	    this.originInvoker = originalInvoker;
	}

	/**
	 * 通知
	 * @param urls 已注册的信息列表，它总是非空。这意味着与RegistryService#lookup(URL)有相同的返回值
	 */
	@Override
	public synchronized void notify(List<URL> urls) {
	    logger.debug("original override urls: " + urls);
	    
	    //获取匹配到的url
	    List<URL> matchedUrls = getMatchedUrls(urls, subscribeUrl);
	    logger.debug("subscribe url: " + subscribeUrl + ", override urls: " + matchedUrls);

	    //没有匹配到结果，直接返回
	    if (matchedUrls.isEmpty()) {
		return;
	    }
		
	    //根据匹配的url生成configurators列表(后面小节会分析该方法)
	    List<Configurator> configurators = RegistryDirectory.toConfigurators(matchedUrls);

	    final Invoker<?> invoker;
	    if (originInvoker instanceof InvokerDelegete) {
	        //委托类，调用getInvoker()方法获取invoker
		invoker = ((InvokerDelegete<?>) originInvoker).getInvoker();
	    } else {
		invoker = originInvoker;
	    }

	    //originUrl即export参数对应的url(原始服务提供者url)
	    URL originUrl = RegistryProtocol.this.getProviderUrl(invoker);
	    
	    //这里的exporter就是在上步的doLocalExport()方法中生成的
	    String key = getCacheKey(originInvoker);

	    //根据缓存key从缓存中获取exporter
	    ExporterChangeableWrapper<?> exporter = bounds.get(key);
	    if (exporter == null) {
		logger.warn(new IllegalStateException("error state, exporter should not be null"));
		return;
	    }
	    
	    //当前的url(旧的服务暴露的url)，可能已经合并了很多次
	    URL currentUrl = exporter.getInvoker().getUrl();
	   
	    //合并配置，生成新的服务暴露url
	    URL newUrl = getConfigedInvokerUrl(configurators, originUrl);
	    
	    if (!currentUrl.equals(newUrl)) {
		//已暴露的服务提供者url已经发生改变，重新暴露
		RegistryProtocol.this.doChangeLocalExport(originInvoker, newUrl);

		logger.info("exported provider url changed, origin url: " + originUrl + ", old export url: " + currentUrl + ", new export url: " + newUrl);
	    }
	}
	
	/**
	 * 获取匹配的url
	 * @param configuratorUrls
	 * @param currentSubscribe 当前已订阅的url
	 */
	private List<URL> getMatchedUrls(List<URL> configuratorUrls, URL currentSubscribe) {
	    List<URL> result = new ArrayList<URL>();
	    
	    for (URL url : configuratorUrls) {
		URL overrideUrl = url;
		//与旧版本兼容
		if (url.getParameter(Constants.CATEGORY_KEY) == null && Constants.OVERRIDE_PROTOCOL.equals(url.getProtocol())) {
		    //url的category参数为空，并且url协议为override时，则新增url的category参数 = configurators
		    overrideUrl = url.addParameter(Constants.CATEGORY_KEY, Constants.CONFIGURATORS_CATEGORY);
		}

		//检测url是否可以应用到当前服务
		//consumerUrl,providerUrl
		if (UrlUtils.isMatch(currentSubscribe, overrideUrl)) {
		    //匹配，则将该url保存
		    result.add(url);
		}
	    }
	    return result;
	}

	/**
	 * 合并configurators的url，返回新的url
	 * @param configurators
	 * @param url 原始服务提供者url
	 * @return
	 */
	private URL getConfigedInvokerUrl(List<Configurator> configurators, URL url) {
	    for (Configurator configurator : configurators) {
	        //配置url
		url = configurator.configure(url);
	    }
	    return url;
	}
}
```

最后我们来看下DubboProtocol的export方法

```java
/**
 * 保存发布的服务
 * <URL的serviceKey,Exporter>
 * 通过key可以获取到服务发布对象DubboExporter，
 * 然后通过DubboExporter的getInvoker方法得到服务调用对象Invoker,从而调用服务
 */
protected final Map<String, Exporter<?>> exporterMap = new ConcurrentHashMap<String, Exporter<?>>();

/**
 * 消费者端为调度事件暴露一个存根服务
 * <servicekey,stubmethods>
 */
private final ConcurrentMap<String, String> stubServiceMethodsMap = new ConcurrentHashMap<String, String>();


/**
 *
 * @param invoker 即新创建的InvokerDelegete对象
 */
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
	
	//服务提供者url
	//dubbo://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=192.168.99.60&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=7520&qos.port=22222&side=provider&timestamp=1528784828723
	URL url = invoker.getUrl();

	// 获取服务名称，格式为：group/ServiceName:version:port
	// 这里group和version没有设置，因此key = com.alibaba.dubbo.demo.DemoService:20880
	String key = serviceKey(url);
	
	//生成key值之后，结合Invoker和exportMap生成服务暴露对象exporter，
	//然后将生成的服务暴露对象exporter作为value值放入map中，从而实现服务发布
	DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
	exporterMap.put(key, exporter);

	//为调度事件暴露一个存根服务，默认false
	Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
	
	//是否是回调服务
	Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
	if (isStubSupportEvent && !isCallbackservice) {
	    //存根服务方法
	    String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
	    if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
		if (logger.isWarnEnabled()) {
		    //设置了存根代理支持事件，但是没有发现存根方法
		    logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
			    "], has set stubproxy support event ,but no stub methods founded."));
		}
	    } else {
		//保存存根方法
		stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
	    }
	}
	//开启服务
	openServer(url);
	//优化序列化(加载优化序列化类,后面小节会详细介绍)
	optimizeSerialization(url);
	//返回exporter
	return exporter;
}

/**
 * 保存已创建的服务器
 * <地址address，ExchangeServer>
 */
private final Map<String, ExchangeServer> serverMap = new ConcurrentHashMap<String, ExchangeServer>();

/**
 * 开启服务(后面会小节会详解讲解)
 * @param url
 */
private void openServer(URL url) {
	//获取服务地址
	String key = url.getAddress();
	
	//客户端可以暴露一个只能服务端调用的服务
	boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
	
	if (isServer) {
	    //先从缓存中获取下该服务地址key对应的服务器
	    ExchangeServer server = serverMap.get(key);
	    if (server == null) {
		//创建服务器，并放入缓存
		serverMap.put(key, createServer(url));
	    } else {
		//服务器支持重置，与override一起使用
		//重置服务器
		server.reset(url);
	    }
	}
}


/**
 * 创建服务器
 * @param url 服务提供者url
 * @return
 */
private ExchangeServer createServer(URL url) {
	//当服务器关闭时，发送readonly事件，默认情况下，是启用的。
	url = url.addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString());
	
	//默认情况下，启用心跳
	url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
	
	//获取Transporter扩展，默认使用netty
	String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);

	if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
	    //不支持的服务器类型
	    throw new RpcException("Unsupported server type: " + str + ", url: " + url);
	}
	//在url中添加codec=dubbo
	url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
	
	ExchangeServer server;
	try {
	    //Exchangers门面 绑定到服务器
	    server = Exchangers.bind(url, requestHandler);
	} catch (RemotingException e) {
	    throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
	}
	
	//获取客户端扩展名称
	str = url.getParameter(Constants.CLIENT_KEY);
	
	if (str != null && str.length() > 0) {
	    //获取支持的Transporter扩展名称列表
	    Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
	    if (!supportedTypes.contains(str)) {
		//不支持的客户端类型
		throw new RpcException("Unsupported client type: " + str);
	    }
	}
	return server;
}
```


### 暴露到本地

主要的方法调用protocol.export上文我们已经介绍过了，这里就不再介绍了。

```java
/**
 * 暴露服务到本地注册中心
 * @param url 服务暴露的url
 */
private void exportLocal(URL url) {
	if (!Constants.LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
	    //例如：injvm://127.0.0.1/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=192.168.99.60&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=4328&qos.port=22222&side=provider&timestamp=1528278313225
	    URL local = URL.valueOf(url.toFullString())
		    //设置injvm协议
		    .setProtocol(Constants.LOCAL_PROTOCOL)
		    //设置本地地址
		    .setHost(LOCALHOST)
		    //设置端口
		    .setPort(0);

	    //获取到实现类ref的Class对象，然后将它放入到ThreadLocal中
	    ServiceClassHolder.getInstance().pushServiceClass(getServiceClass(ref));

	    //暴露服务(后面会分析export方法)
	    Exporter<?> exporter = protocol.export(
		    proxyFactory.getInvoker(ref, (Class) interfaceClass, local)
	    );

	    //保存暴露的服务到本地变量中
	    exporters.add(exporter);
	    logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry");
	}
}
```

到此，关于服务暴露的内容，我们就介绍完毕了，下一小节，介绍如果引用服务.
