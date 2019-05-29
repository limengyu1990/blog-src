---
title: Dubbo源码阅读之服务引用
date: 2018-08-23 21:16:24
tags:
    - dubbo
---
>本小节讲解上小节遗留的ref = createProxy(map)方法，不了解的可以先看下上篇文章《Dubbo源码阅读之集成Spring-0202注解解析》

该方法定义在ReferenceConfig类中，在调用init方法进行初始化时，会调用createProxy方法来生成代理对象。

### createProxy创建代理
```java
/**
 * 创建代理
 */
private T createProxy(Map<String, String> map) {
	URL tmpUrl = new URL("temp", "localhost", 0, map);
	final boolean isJvmRefer;
	//是否引用本地服务，新版使用： scope=local来判断
	if (isInjvm() == null) {
	    if (url != null && url.length() > 0) {
		//如果配置中指定了url属性，则不使用本地引用
		isJvmRefer = false;
	    } else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
		//默认情况下，如果有本地服务，则引用本地服务
		isJvmRefer = true;
	    } else {
		isJvmRefer = false;
	    }
	} else {
	    isJvmRefer = isInjvm().booleanValue();
	}
	if (isJvmRefer) {
	    //构建本地协议
	    URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
	    //引用本地服务(后面会分析该方法)
	    invoker = refprotocol.refer(interfaceClass, url);
	    if (logger.isInfoEnabled()) {
		logger.info("Using injvm service " + interfaceClass.getName());
	    }
	} else {
	    if (url != null && url.length() > 0) {
		//用户指定了URL属性，可以是p2p地址或者注册中心的地址，多个地址使用";"分隔
		String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
		if (us != null && us.length > 0) {
		    for (String u : us) {
			URL url = URL.valueOf(u);
			if (url.getPath() == null || url.getPath().length() == 0) {
			    //path为空时，使用服务接口名称作为path
			    url = url.setPath(interfaceName);
			}
			if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
			    //注册中心协议，添加引用标识refer参数
			    urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
			} else {
			    //p2p地址(后面会分析该方法)
			    urls.add(ClusterUtils.mergeUrl(url, map));
			}
		    }
		}
	    } else {
		//从注册中心配置中组装URL(后面会分析该方法)
		List<URL> us = loadRegistries(false);
		if (us != null && !us.isEmpty()) {
		    //遍历注册中心us(后面会分析该方法)
		    for (URL u : us) {
			//加载监控url(后面会分析该方法)
			URL monitorUrl = loadMonitor(u);
			if (monitorUrl != null) {
			    //添加monitor参数
			    map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
			}
			//添加refer参数
			urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
		    }
		}
		if (urls == null || urls.isEmpty()) {
		    //在消费者端，没有配置注册中心，因此无法引用相应服务接口
		    throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
		}
	    }
	    if (urls.size() == 1) {
		//引用一个远程服务(后面会分析该方法)
		//registry://172.172.172.47:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-consumer&dubbo=2.0.0&pid=1172&qos.port=33333&refer=application%3Ddemo-consumer%26check%3Dfalse%26dubbo%3D2.0.0%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D1172%26qos.port%3D33333%26register.ip%3D192.168.99.60%26side%3Dconsumer%26timestamp%3D1535094943004&registry=zookeeper&timestamp=1535095022853
		invoker = refprotocol.refer(interfaceClass, urls.get(0));
	    } else {
		//如果有多个url
		List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
		URL registryURL = null;
		for (URL url : urls) {
		    //记录"远程引用"
		    invokers.add(refprotocol.refer(interfaceClass, url));
		    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
			//使用最后一个注册中心url
			registryURL = url;
		    }
		}
		if (registryURL != null) {
		    //有注册中心协议的URL，使用AvailableCluster
		    URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
		    invoker = cluster.join(new StaticDirectory(u, invokers));
		} else { 
		    //不是注册中心的url
		    invoker = cluster.join(new StaticDirectory(invokers));
		}
	    }
	}
	//检测服务提供者是否存在
	Boolean c = check;
	if (c == null && consumer != null) {
	    c = consumer.isCheck();
	}
	if (c == null) {
	    //默认需要检测服务提供者是否存在
	    c = true;
	}
	//检测服务提供者是否存在
	if (c && !invoker.isAvailable()) {
	    throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
	}
	if (logger.isInfoEnabled()) {
	    logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
	}
	//创建服务代理
	return (T) proxyFactory.getProxy(invoker);
}

/**
 * 构造注册中心URL
 * @param provider
 * @return
 */
protected List<URL> loadRegistries(boolean provider) {
	//检测是否配置了RegistryConfig，并配置
	checkRegistry();
	List<URL> registryList = new ArrayList<URL>();
	if (registries != null && !registries.isEmpty()) {
	    //遍历注册中心配置
	    for (RegistryConfig config : registries) {
		//获取当前注册中心地址，例如： zookeeper://172.172.172.47:2181
		String address = config.getAddress();
		if (address == null || address.length() == 0) {
		    //设置address = 0.0.0.0
		    address = Constants.ANYHOST_VALUE;
		}
		//从系统配置中获取注册中心地址
		String sysaddress = System.getProperty("dubbo.registry.address");
		if (sysaddress != null && sysaddress.length() > 0) {
		    //如果系统配置的注册中心地址不为空，则优先使用系统配置
		    address = sysaddress;
		}
		if (address != null && address.length() > 0
			&& !RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(address)) {
		    //注册中心地址可用
		    Map<String, String> map = new HashMap<String, String>();
		    //附加参数，即找到application、config类中的属性，并添加进来
		    appendParameters(map, application);
		    appendParameters(map, config);
		    map.put("path", RegistryService.class.getName());
		    map.put("dubbo", Version.getVersion());
		    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
		    if (ConfigUtils.getPid() > 0) {
			map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
		    }
		    if (!map.containsKey("protocol")) {
			//添加protocol参数
			if (ExtensionLoader.getExtensionLoader(RegistryFactory.class).hasExtension("remote")) {
			    map.put("protocol", "remote");
			} else {
			    map.put("protocol", "dubbo");
			}
		    }
		    //根据address和map生成注册中心url
		    List<URL> urls = UrlUtils.parseURLs(address, map);
		    
		    //遍历注册中心url，添加registry参数,并设置protocol属性，然后保存起来
		    for (URL url : urls) {
			//url = zookeeper://172.172.172.47:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-consumer&dubbo=2.0.0&pid=1172&qos.port=33333&timestamp=1535095022853
			//添加registry参数 = zookeeper
			url = url.addParameter(Constants.REGISTRY_KEY, url.getProtocol());
			
			//重新设置协议为registry
			url = url.setProtocol(Constants.REGISTRY_PROTOCOL);
			if ((provider && url.getParameter(Constants.REGISTER_KEY, true))
				|| (!provider && url.getParameter(Constants.SUBSCRIBE_KEY, true))) {
			    //registry://172.172.172.47:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-consumer&dubbo=2.0.0&pid=1172&qos.port=33333&registry=zookeeper&timestamp=1535095022853
			    registryList.add(url);
			}
		    }
		}
	    }
	}
	return registryList;
}

/**
 * 检测RegistryConfig配置
 */
protected void checkRegistry() {
	//向后兼容
	if (registries == null || registries.isEmpty()) {
	    //获取到注册中心属性配置值
	    String address = ConfigUtils.getProperty("dubbo.registry.address");
	    if (address != null && address.length() > 0) {
		registries = new ArrayList<RegistryConfig>();
		//使用"|"分隔注册中心，然后构造RegistryConfig对象
		String[] as = address.split("\\s*[|]+\\s*");
		for (String a : as) {
		    RegistryConfig registryConfig = new RegistryConfig();
		    registryConfig.setAddress(a);
		    registries.add(registryConfig);
		}
	    }
	}
	if ((registries == null || registries.isEmpty())) {
	    throw new IllegalStateException((getClass().getSimpleName().startsWith("Reference")
		    ? "No such any registry to refer service in consumer "
		    : "No such any registry to export service in provider ")
		    + NetUtils.getLocalHost()
		    + " use dubbo version "
		    + Version.getVersion()
		    + ", Please add <dubbo:registry address=\"...\" /> to your spring config. If you want unregister, please set <dubbo:service registry=\"N/A\" />");
	}
	for (RegistryConfig registryConfig : registries) {
	    //添加属性
	    appendProperties(registryConfig);
	}
}
```

接下来我们看下Protocol的refer方法，在《Dubbo源码阅读之服务暴露》小节我们介绍过，将会生成Protocol$Adaptive类：
```java
public class Protocol$Adaptive implements com.alibaba.dubbo.rpc.Protocol {
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
		
		//获取扩展名称为extName的Protocol扩展实例(这里可能是被包装过的类)
		com.alibaba.dubbo.rpc.Protocol extension = (com.alibaba.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.Protocol.class).getExtension(extName);
		
		//调用该extension实例的refer方法
		return extension.refer(arg0, arg1);
	}
}
```
这里和服务暴露时的流程一样，将会调用两个包装类ProtocolFilterWrapper和ProtocolListenerWrapper的refer方法。
```java
public class ProtocolFilterWrapper implements Protocol {
	@Override
	public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
		if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
		    //这里将会调用ProtocolListenerWrapper类的refer方法
		    return protocol.refer(type, url);
		}
		return buildInvokerChain(protocol.refer(type, url), Constants.REFERENCE_FILTER_KEY, Constants.CONSUMER);
	}
}

public class ProtocolListenerWrapper implements Protocol {
	/**
	 * @param type interface com.alibaba.dubbo.demo.DemoService
	 * @param url  registry://224.5.6.7:1234/com.alibaba.dubbo.registry.RegistryService?application=demo-consumer&dubbo=2.0.0&pid=1512&qos.port=33333&refer=application%3Ddemo-consumer%26check%3Dfalse%26dubbo%3D2.0.0%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D1512%26qos.port%3D33333%26register.ip%3D192.168.99.60%26side%3Dconsumer%26timestamp%3D1528340394760&registry=multicast&timestamp=1528340417091
	 * @param <T>
	 * @return
	 * @throws RpcException
	 */
	@Override
	public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
		if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
		    //这里将会调用RegistryProtocol类的refer方法
		    return protocol.refer(type, url);
		}
		return new ListenerInvokerWrapper<T>(protocol.refer(type, url),
			Collections.unmodifiableList(
				ExtensionLoader.getExtensionLoader(InvokerListener.class)
					.getActivateExtension(url, Constants.INVOKER_LISTENER_KEY)));
	}
}
```
RegistryProtocol的refer方法：
```java
/**
 * @param type远程服务接口类型
 * @param url 远程服务的URL地址
 * registry://172.172.172.47:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-consumer&dubbo=2.0.0&pid=1172&qos.port=33333&refer=application%3Ddemo-consumer%26check%3Dfalse%26dubbo%3D2.0.0%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D1172%26qos.port%3D33333%26register.ip%3D192.168.99.60%26side%3Dconsumer%26timestamp%3D1535094943004&registry=zookeeper&timestamp=1535095022853
 * @return
 * @throws RpcException
 */
@Override
@SuppressWarnings("unchecked")
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
	//生成注册中心url
	url = url.setProtocol(
		//设置协议，从url中的registry参数中获取注册时的协议(zookeeper)，没有获取到的话，则默认为dubbo协议
		url.getParameter(Constants.REGISTRY_KEY, Constants.DEFAULT_REGISTRY)
		//然后移除registry参数
	).removeParameter(Constants.REGISTRY_KEY);

	//根据url获取注册中心
	//zookeeper://172.172.172.47:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-consumer&dubbo=2.0.0&pid=1172&qos.port=33333&refer=application%3Ddemo-consumer%26check%3Dfalse%26dubbo%3D2.0.0%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D1172%26qos.port%3D33333%26register.ip%3D192.168.99.60%26side%3Dconsumer%26timestamp%3D1535094943004&timestamp=1535095022853
	Registry registry = registryFactory.getRegistry(url);
	if (RegistryService.class.equals(type)) {
	    return proxyFactory.getInvoker((T) registry, type, url);
	}

	//获取url中的refer参数，并转换成map
	Map<String, String> qs = StringUtils.parseQueryString(
		url.getParameterAndDecoded(Constants.REFER_KEY)
	);
	//获取map中的group参数
	String group = qs.get(Constants.GROUP_KEY);
	if (group != null && group.length() > 0) {
	    if ((Constants.COMMA_SPLIT_PATTERN.split(group)).length > 1
		    || "*".equals(group)) {
		//group="a,b" or group="*"
		return doRefer(getMergeableCluster(), registry, type, url);
	    }
	}
	return doRefer(cluster, registry, type, url);
}

/**
 *
 * @param cluster
 * @param registry 注册中心
 * @param type 远程服务接口类型
 * @param url 注册中心url
 * @return
 */
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
	//根据服务接口type和注册中心url创建RegistryDirectory对象(后面会分析该方法)
	//Directory代表多个Invoker,可以把它看成List<Invoker>，
        //但与List不同的是,它的值可能是动态变化的,比如注册中心推送变更
	RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
	//设置注册中心实例
	directory.setRegistry(registry);
	//设置协议
	directory.setProtocol(protocol);
	
	//refer参数指定的url的所有属性
	Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
	
	//生成消费者url
	//consumer://192.168.99.60/com.alibaba.dubbo.demo.DemoService?application=demo-consumer&check=false&dubbo=2.0.0&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=1172&qos.port=33333&side=consumer&timestamp=1535094943004
	URL subscribeUrl = new URL(Constants.CONSUMER_PROTOCOL, parameters.remove(Constants.REGISTER_IP_KEY), 0, type.getName(), parameters);
	
	if (!Constants.ANY_VALUE.equals(url.getServiceInterface())
		&& url.getParameter(Constants.REGISTER_KEY, true)) {
	    //注册消费者，添加category=consumers,check=false参数
	    registry.register(subscribeUrl.addParameters(Constants.CATEGORY_KEY, Constants.CONSUMERS_CATEGORY,
		    Constants.CHECK_KEY, String.valueOf(false)));
	}
	
	//订阅此url
	directory.subscribe(subscribeUrl.addParameter(Constants.CATEGORY_KEY,
		//提供者 /dubbo/interfaceClass/providers
		//配置  /dubbo/interfaceClass/configurators
		//路由  /dubbo/interfaceClass/routers
		Constants.PROVIDERS_CATEGORY
			+ "," + Constants.CONFIGURATORS_CATEGORY
			+ "," + Constants.ROUTERS_CATEGORY));
	
	//Cluster将Directory中的多个Invoker伪装成一个Invoker,对上层透明,伪装过程包含了容错逻辑,调用失败后,重试另一个(后面会分析该方法)
	Invoker invoker = cluster.join(directory);
	
	//注册消费者
	ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
	return invoker;
}
```
这里首先注册了消费者，然后订阅了相关目录(providers、configurations、routers)，当有相应服务提供者提供服务时，注册中心会通过notify通知到消费者，接着消费者会通过RegistryDirectory类异步更新本地缓存。
注册消费者时，首先会调用AbstractRegistry类的register方法，将注册url保存起来，然后会调用FailbackRegistry类的register方法，在该方法中会调用doRegister方法(该方法调用失败的话，会稍后进行重试),最后会调用ZookeeperRegistry的doRegister方法，进行最终的注册。
```java
//ZookeeperRegistry类的doRegister方法
@Override
protected void doRegister(URL url) {
	try {
	    //执行注册 dubbo://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=5312&side=provider&timestamp=1534994264738
	    //根据url确定节点路径；根据url的dynamic参数确定是否是临时节点 /dubbo/com.alibaba.dubbo.demo.DemoService/providers/dubbo%3A%2F%2F192.168.99.60%3A20880%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26dubbo%3D2.0.0%26generic%3Dfalse%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D5312%26side%3Dprovider%26timestamp%3D1534994264738
	    zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
	} catch (Throwable e) {
	    throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
	}
}
```
订阅时，会调用RegistryDirectory类的subscribe方法
```java
public class RegistryDirectory<T> extends AbstractDirectory<T> implements NotifyListener {

       /**
	* 订阅url
	* @param url 消费者url 
	* consumer://192.168.99.60/com.alibaba.dubbo.demo.DemoService?application=demo-consumer&category=providers,configurators,routers&check=false&dubbo=2.0.0&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=1880&qos.port=33333&side=consumer&timestamp=1535449640995
	*/
	public void subscribe(URL url) {
		//设置消费者url(将url保存到AbstractDirectory父类中)
		setConsumerUrl(url);
		
		//订阅该消费者url(this即为当前RegistryDirectory对象,它实现了NotifyListener接口)
		//首先调用AbstractRegistry的subscribe方法(保存订阅信息)
		//接着调用FailbackRegistry的subscribe方法(调用ZookeeperRegistry的doSubscribe方法，如果发生异常了会捕获到，然后保存下来稍后重试)
		//然后调用ZookeeperRegistry的doSubscribe方法(获取待通知urls，最后调用notify方法进行通知)
		registry.subscribe(url, this);
	}
}
```
我们简单看下ZookeeperRegistry类的doSubscribe方法，其中省略掉了一些代码
```java
//此listener参数就是上文创建的RegistryDirectory类
protected void doSubscribe(final URL url, final NotifyListener listener) {
	List<URL> urls = new ArrayList<URL>();
	//根据toCategoriesPath(url)方法获取到类别列表，然后遍历该列表
	//如：/dubbo/com.alibaba.dubbo.demo.DemoService/providers
	for (String path : toCategoriesPath(url)) {
	    ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
	    if (listeners == null) {
		zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
		listeners = zkListeners.get(url);
	    }
	    //根据dubbo的监听获取zk的监听
	    ChildListener zkListener = listeners.get(listener);
	    if (zkListener == null) {
	        //创建节点监听
		listeners.putIfAbsent(listener, new ChildListener() {
		    @Override
		    public void childChanged(String parentPath, List<String> currentChilds) {
			//执行通知
			ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
		    }
		});
		zkListener = listeners.get(listener);
	    }
	    //创建path节点,
	    //即：/dubbo/com.xxx.demoService/providers
	    //即：/dubbo/consumers  /dubbo/com.alibaba.dubbo.demo.DemoService/configurators
	    zkClient.create(path, false);
	    //为path节点添加zkListener监听(children变量即为该path节点的子节点数据)
	    List<String> children = zkClient.addChildListener(path, zkListener);
	    if (children != null) {
	        //节点数据(待通知列表)
		//empty://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&category=configurators&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=6504&side=provider&timestamp=1534927826842
		urls.addAll(toUrlsWithEmpty(url, path, children));
	    }
	}
	//执行通知(urls即为待通知的消息列表)
	notify(url, listener, urls);
}
```
调用notify方法进行通知时,会先调用FailbackRegistry类的notify方法,内部会调用doNotify方法(调用失败的话,会保存失败信息,稍后重试)
```java
//FailbackRegistry类的notify
//省略了一些不重要的代码
@Override
protected void notify(URL url, NotifyListener listener, List<URL> urls) {
	try {
	    //执行通知(将会调用父类AbstractRegistry中的notify方法)
	    doNotify(url, listener, urls);
	} catch (Exception t) {
	    //通知失败，添加到失败列表，定期重试
	    Map<NotifyListener, List<URL>> listeners = failedNotified.get(url);
	    if (listeners == null) {
		failedNotified.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, List<URL>>());
		listeners = failedNotified.get(url);
	    }
	    listeners.put(listener, urls);
	    logger.error("Failed to notify for subscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
	}
}
```
而doNotify方法最终会调用AbstractRegistry类的notify方法,在AbstractRegistry类的notify方法中,会根据category将待通知urls进行分组,然后挨个处理每一个category,并按照<订阅url服务唯一标识,空格分隔的多个待通知url>的形式，保存到本地properties文件中，最后调用RegistryDirectory类的notify方法将待通知列表categoryList下发给消费者。
```java
//AbstractRegistry类的notify方法
//省略了一些不重要的代码
protected void notify(URL url, NotifyListener listener, List<URL> urls) {
	//<category,List<URL>>
	Map<String, List<URL>> result = new HashMap<String, List<URL>>();
	//遍历待通知urls，根据url中的category参数进行分组，保存到result中
	for (URL u : urls) {
	    //url和u是否匹配(url的范围是否比u大)
	    if (UrlUtils.isMatch(url, u)) {
		//获取待通知url的category，默认值是providers
		String category = u.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
		List<URL> categoryList = result.get(category);
		if (categoryList == null) {
		    categoryList = new ArrayList<URL>();
		    result.put(category, categoryList);
		}
		categoryList.add(u);
	    }
	}
	if (result.size() == 0) {
	    //类别待通知url列表为空，直接返回
	    return;
	}
	//下面会将notified中的url及其values.values中的URL保存到缓存文件中
	Map<String, List<URL>> categoryNotified = notified.get(url);
	if (categoryNotified == null) {
	    notified.putIfAbsent(url, new ConcurrentHashMap<String, List<URL>>());
	    categoryNotified = notified.get(url);
	}
	for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
	    //类别
	    String category = entry.getKey();
	    //类别待通知url列表
	    List<URL> categoryList = entry.getValue();
	    //保存<类别,类别待通知url列表>
	    categoryNotified.put(category, categoryList);
	    //保存注册中心缓存文件(将订阅url、待通知url列表保存到缓存文件)
	    saveProperties(url);
	    //触发通知给消费者(类别待通知url列表)，此listener就是RegistryDirectory类
	    listener.notify(categoryList);
	}
}
```
然后我们来看RegistryDirectory类的notify方法
```java
public synchronized void notify(List<URL> urls) {
	List<URL> invokerUrls = new ArrayList<URL>();
	List<URL> routerUrls = new ArrayList<URL>();
	List<URL> configuratorUrls = new ArrayList<URL>();
	//遍历待通知列表，根据category进行分组，并分别保存到上面定义的list列表中
	for (URL url : urls) {
	    //获取url协议
	    String protocol = url.getProtocol();
	    //获取url分类
	    String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
	    if (Constants.ROUTERS_CATEGORY.equals(category)
		    || Constants.ROUTE_PROTOCOL.equals(protocol)) {
		//添加路由url
		routerUrls.add(url);
	    } else if (Constants.CONFIGURATORS_CATEGORY.equals(category)
		    || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
		//添加配置url
		configuratorUrls.add(url);
	    } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
		//添加服务提供者url
		invokerUrls.add(url);
	    } else {
		//不支持的分类
		logger.warn("Unsupported category " + category + " in notified url: " + url + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost());
	    }
	}
	if (configuratorUrls != null && !configuratorUrls.isEmpty()) {
	    //将url转换成Configurator类并保存(后面会分析该方法)
	    this.configurators = toConfigurators(configuratorUrls);
	}
	if (routerUrls != null && !routerUrls.isEmpty()) {
	    //将路由url转换成路由对象(后面会分析该方法)
	    List<Router> routers = toRouters(routerUrls);
	    if (routers != null) {
		//保存路由(后面会分析该方法)
		setRouters(routers);
	    }
	}
	List<Configurator> localConfigurators = this.configurators;
	//合并override参数
	this.overrideDirectoryUrl = directoryUrl;
	if (localConfigurators != null && !localConfigurators.isEmpty()) {
	    for (Configurator configurator : localConfigurators) {
		//根据configurator覆盖overrideDirectoryUrl中的属性，
		//或者将configurator中的属性添加到overrideDirectoryUrl中
		this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
	    }
	}
	//刷新Invoker(后面会分析该方法)
	refreshInvoker(invokerUrls);
}

```
接下来，我们来挨个看下上面用到的方法

#### 更新Configurator
```java
//将待通知url转换成Configurator
public static List<Configurator> toConfigurators(List<URL> urls) {
	if (urls == null || urls.isEmpty()) {
	    return Collections.emptyList();
	}
	List<Configurator> configurators = new ArrayList<Configurator>(urls.size());
	//遍历待通知url列表
	for (URL url : urls) {
	    if (Constants.EMPTY_PROTOCOL.equals(url.getProtocol())) {
		//协议为empty，则清空configurators，并跳出循环
		configurators.clear();
		break;
	    }
	    //获取url所有参数
	    Map<String, String> override = new HashMap<String, String>(url.getParameters());
	    
	    //override的anyhost参数可以被自动添加
	    //它不可以改变对变化中的url的判断，因此需要移除掉anyhost参数
	    override.remove(Constants.ANYHOST_KEY);
	    if (override.size() == 0) {
		configurators.clear();
		continue;
	    }
	    //添加配置
	    configurators.add(configuratorFactory.getConfigurator(url));
	}
	//排序
	Collections.sort(configurators);
	return configurators;
}
```

#### 更新Router
```java
//将待通知url转换成Router对象
private List<Router> toRouters(List<URL> urls) {
	List<Router> routers = new ArrayList<Router>();
	if (urls == null || urls.isEmpty()) {
	    return routers;
	}
	if (urls != null && !urls.isEmpty()) {
	    //遍历待通知url列表
	    for (URL url : urls) {
		if (Constants.EMPTY_PROTOCOL.equals(url.getProtocol())) {
		    //协议为empty的话，直接返回
		    continue;
		}
		//获取url的router参数，该参数标识了路由协议
		String routerType = url.getParameter(Constants.ROUTER_KEY);
		if (routerType != null && routerType.length() > 0) {
		    //设置路由协议
		    url = url.setProtocol(routerType);
		}
		try {
		    //根据url获取路由实例
		    Router router = routerFactory.getRouter(url);
		    if (!routers.contains(router)) {
			//添加路由
			routers.add(router);
		    }
		} catch (Throwable t) {
		    logger.error("convert router url to router error, url: " + url, t);
		}
	    }
	}
	return routers;
}

protected void setRouters(List<Router> routers) {
	//复制routers列表
	routers = routers == null ? new ArrayList<Router>() : new ArrayList<Router>(routers);
	
	//获取路由工厂的扩展名称
	String routerkey = url.getParameter(Constants.ROUTER_KEY);
	if (routerkey != null && routerkey.length() > 0) {
	    //根据 路由工厂扩展名 获取 路由工厂扩展实例
	    RouterFactory routerFactory = ExtensionLoader.getExtensionLoader(RouterFactory.class).getExtension(routerkey);
	    //根据url获取路由实例，并放入routers
	    routers.add(routerFactory.getRouter(url));
	}
	//添加支持mock协议的invoker选择器
	routers.add(new MockInvokersSelector());
	//排序
	Collections.sort(routers);
	//保存最新的路由信息
	this.routers = routers;
}
```
#### 更新Invoker
然后我们来看refreshInvoker方法
```java
/**
 * 将待通知invoker url转换成invoker Map，转换规则如下：
 * 1、如果url已经转换成invoker，则不再重新引用并直接从缓存中获取，并注意url中的任何参数更改都将被重新引用。
 * 2、如果传入的invoker列表不为空，则意味着这是最新的调用者列表
 * 3、如果传入的invoker列表为空，则意味着该规则只是一个override规则或者route规则，需要重新对比以决定是否需要重新引用
 */
private void refreshInvoker(List<URL> invokerUrls) {
	if (invokerUrls != null && invokerUrls.size() == 1 && invokerUrls.get(0) != null
		&& Constants.EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
	    //empty协议
	    //禁止访问
	    this.forbidden = true;
	    this.methodInvokerMap = null;
	    //销毁所有的invokers
	    destroyAllInvokers();
	} else {
	    //允许访问
	    this.forbidden = false;
	    //记录当前的 urlInvokerMap
	    Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap;
	    if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
		//收到的invokerUrls为空，但是缓存中的不为空，则使用当前缓存中的invoker
		invokerUrls.addAll(this.cachedInvokerUrls);
	    } else {
		this.cachedInvokerUrls = new HashSet<URL>();
		//缓存invokerUrls列表，方便比较
		this.cachedInvokerUrls.addAll(invokerUrls);
	    }
	    if (invokerUrls.isEmpty()) {
		//invoker url列表为空，直接返回
		return;
	    }
	    //将invoker-url转换成invoker-map(后面会分析该方法)
	    Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);
	    
	    //建立 方法名与invoker的映射(后面会分析该方法)
	    Map<String, List<Invoker<T>>> newMethodInvokerMap = toMethodInvokers(newUrlInvokerMap);
	   
	    //状态改变
	    if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0) {
		//转换发生错误
		logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls.toString()));
		return;
	    }
	    //保存最新的 methodInvokerMap(后面会分析该方法)
	    this.methodInvokerMap = multiGroup ? toMergeMethodInvokerMap(newMethodInvokerMap) : newMethodInvokerMap;
	    //保存最新的 urlInvokerMap
	    this.urlInvokerMap = newUrlInvokerMap;
	    try {
		//关闭未使用的invoker(后面会分析该方法)
		destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap);
	    } catch (Exception e) {
		logger.warn("destroyUnusedInvokers error. ", e);
	    }
	}
}
```

##### toInvokers方法
```java
/**
 * 将invoker-url转换成invoker，如果url已经被引用，将不会重新引用
 * @param urls 服务提供者url
 * dubbo://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=6640&side=provider&timestamp=1535449604077
 * @return invokers
 */
private Map<String, Invoker<T>> toInvokers(List<URL> urls) {
	Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<String, Invoker<T>>();
	if (urls == null || urls.isEmpty()) {
	    return newUrlInvokerMap;
	}
	Set<String> keys = new HashSet<String>();
	//获取当前url支持的协议
	String queryProtocols = this.queryMap.get(Constants.PROTOCOL_KEY);
	//遍历urls，检测每一个providerUrl是否满足当前调用者url的协议要求(queryProtocols)
	for (URL providerUrl : urls) {
	    //如果在reference端配置了协议protocol，则只选择匹配的协议
	    if (queryProtocols != null && queryProtocols.length() > 0) {
		boolean accept = false;
		//可接受的协议数组
		String[] acceptProtocols = queryProtocols.split(",");
		for (String acceptProtocol : acceptProtocols) {
		    if (providerUrl.getProtocol().equals(acceptProtocol)) {
			//如果该提供者url的协议在可接受的协议范围内，则跳出循环
			accept = true;
			break;
		    }
		}
		if (!accept) {
		    //没找到，则判断下一个服务提供者
		    continue;
		}
	    }
	    if (Constants.EMPTY_PROTOCOL.equals(providerUrl.getProtocol())) {
	        //服务提供者协议为empty，则进行下次循环
		continue;
	    }
	    if (!ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(providerUrl.getProtocol())) {
		//不支持的协议
		logger.error(new IllegalStateException("Unsupported protocol " + providerUrl.getProtocol() + " in notified url: " + providerUrl + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost()
			+ ", supported protocol: " + ExtensionLoader.getExtensionLoader(Protocol.class).getSupportedExtensions()));
		continue;
	    }
	    //合并url参数(后面会分析该方法)
	    URL url = mergeUrl(providerUrl);
	    
	    //url的参数已经排过序
	    //dubbo://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-consumer&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=1880&qos.port=33333&register.ip=192.168.99.60&remote.timestamp=1535449604077&side=consumer&timestamp=1535449640995
	    String key = url.toFullString();
	    if (keys.contains(key)) {
		//重复的url
		continue;
	    }
	    keys.add(key);
	    //缓存key是url，不与消费者端的参数合并，无论消费者如何组合参数，
            //如果服务器url发生改变，则重新进行引用
	    Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap;
	    //根据缓存key获取invoker
	    Invoker<T> invoker = localUrlInvokerMap == null ? null : localUrlInvokerMap.get(key);
	    if (invoker == null) {
		//不在缓存中，重新引用
		try {
		    boolean enabled = true;
		    //获取disabled参数、enabled参数，来判断是否可用
		    if (url.hasParameter(Constants.DISABLED_KEY)) {
			enabled = !url.getParameter(Constants.DISABLED_KEY, false);
		    } else {
			enabled = url.getParameter(Constants.ENABLED_KEY, true);
		    }
		    if (enabled) {
			//可以引用，重新引用(后面会介绍)
			invoker = new InvokerDelegate<T>(
			        //根据服务接口、远程服务url重新引用
				protocol.refer(serviceType, url),
				url,
				providerUrl
			);
		    }
		} catch (Throwable t) {
		    logger.error("Failed to refer invoker for interface:" + serviceType + ",url:(" + url + ")" + t.getMessage(), t);
		}
		if (invoker != null) {
		    //保存新的invoker到缓存
		    newUrlInvokerMap.put(key, invoker);
		}
	    } else {
		newUrlInvokerMap.put(key, invoker);
	    }
	}
	keys.clear();
	return newUrlInvokerMap;
}

```
##### toMethodInvokers方法

```java
/**
 * 获取Invoker和method之间的映射关系
 * @param invokersMap Invoker Map
 * @return Mapping relation between Invoker and method
 */
private Map<String, List<Invoker<T>>> toMethodInvokers(Map<String, Invoker<T>> invokersMap) {
	//method与invoker列表的映射
	Map<String, List<Invoker<T>>> newMethodInvokerMap = new HashMap<String, List<Invoker<T>>>();
	
	// According to the methods classification declared by the provider URL,
	// the methods is compatible with the registry to execute the filtered methods
	List<Invoker<T>> invokersList = new ArrayList<Invoker<T>>();
	
	if (invokersMap != null && invokersMap.size() > 0) {
	    //遍历invokers
	    for (Invoker<T> invoker : invokersMap.values()) {
		//获取服务提供者方法列表
		String parameter = invoker.getUrl().getParameter(Constants.METHODS_KEY);
		if (parameter != null && parameter.length() > 0) {
		    //逗号分隔方法
		    String[] methods = Constants.COMMA_SPLIT_PATTERN.split(parameter);
		    if (methods != null && methods.length > 0) {
			for (String method : methods) {
			    if (method != null && method.length() > 0 && !Constants.ANY_VALUE.equals(method)) {
				//通过method获取invoker列表
				List<Invoker<T>> methodInvokers = newMethodInvokerMap.get(method);
				if (methodInvokers == null) {
				    methodInvokers = new ArrayList<Invoker<T>>();
				    //添加方法、invoker映射
				    newMethodInvokerMap.put(method, methodInvokers);
				}
				methodInvokers.add(invoker);
			    }
			}
		    }
		}
		//添加invoker
		invokersList.add(invoker);
	    }
	}
	//根据路由筛选InvokersList(后面会分析该方法)
	List<Invoker<T>> newInvokersList = route(invokersList, null);
	
	//保存<*,newInvokersList>
	newMethodInvokerMap.put(Constants.ANY_VALUE, newInvokersList);
	
	if (serviceMethods != null && serviceMethods.length > 0) {
	    //遍历服务method列表,并从newMethodInvokerMap中获取该method对应的Invoker列表
	    //如果没有获取到，则将该method映射到newInvokersList列表上
	    for (String method : serviceMethods) {
		List<Invoker<T>> methodInvokers = newMethodInvokerMap.get(method);
		if (methodInvokers == null || methodInvokers.isEmpty()) {
		    methodInvokers = newInvokersList;
		}
		//根据路由筛选invoker列表，然后保存method、invokerList映射
		newMethodInvokerMap.put(method, route(methodInvokers, method));
	    }
	}
	for (String method : new HashSet<String>(newMethodInvokerMap.keySet())) {
	    List<Invoker<T>> methodInvokers = newMethodInvokerMap.get(method);
	    //排序invoker
	    Collections.sort(methodInvokers, InvokerComparator.getComparator());
	    //不可变集合
	    newMethodInvokerMap.put(method, Collections.unmodifiableList(methodInvokers));
	}
	return Collections.unmodifiableMap(newMethodInvokerMap);
}
```
###### route方法
```java
/**
 * 根据路由筛选InvokersList
 * Router 负责从多个 Invoker 中按路由规则选出子集，比如读写分离，应用隔离等
 * @param invokers
 * @param method
 * @return
 */
private List<Invoker<T>> route(List<Invoker<T>> invokers, String method) {
	//创建Invocation对象
	Invocation invocation = new RpcInvocation(method, new Class<?>[0], new Object[0]);
	//获取路由列表
	List<Router> routers = getRouters();
	if (routers != null) {
	    for (Router router : routers) {
		if (router.getUrl() != null) {
		    //执行路由
		    invokers = router.route(invokers, getConsumerUrl(), invocation);
		}
	    }
	}
	return invokers;
}
```

##### toMergeMethodInvokerMap方法
```java
private Map<String, List<Invoker<T>>> toMergeMethodInvokerMap(Map<String, List<Invoker<T>>> methodMap) {
	Map<String, List<Invoker<T>>> result = new HashMap<String, List<Invoker<T>>>();
	for (Map.Entry<String, List<Invoker<T>>> entry : methodMap.entrySet()) {
	    //当前方法名
	    String method = entry.getKey();
	    //当前invokers
	    List<Invoker<T>> invokers = entry.getValue();
	    //将invokers按照group进行分组，放入到groupMap中(<group,List<invokers>>)
	    Map<String, List<Invoker<T>>> groupMap = new HashMap<String, List<Invoker<T>>>();
	    for (Invoker<T> invoker : invokers) {
		//当前url的group
		String group = invoker.getUrl().getParameter(Constants.GROUP_KEY, "");
		List<Invoker<T>> groupInvokers = groupMap.get(group);
		if (groupInvokers == null) {
		    groupInvokers = new ArrayList<Invoker<T>>();
		    groupMap.put(group, groupInvokers);
		}
		groupInvokers.add(invoker);
	    }
	    if (groupMap.size() == 1) {
		//只有一个group
		result.put(method, groupMap.values().iterator().next());
	    } else if (groupMap.size() > 1) {
		//多个group的情况
		List<Invoker<T>> groupInvokers = new ArrayList<Invoker<T>>();
		for (List<Invoker<T>> groupList : groupMap.values()) {
		    //针对每一个groupList创建一个StaticDirectory，然后生成一个invoker并放入groupInvokers中
		    groupInvokers.add(cluster.join(new StaticDirectory<T>(groupList)));
		}
		result.put(method, groupInvokers);
	    } else {
		//没有group
		result.put(method, invokers);
	    }
	}
	return result;
}
```
![](img/toMergeMethodInvokerMap.png)

##### 销毁无用的invoker
```java
/**
 * 检测缓存中的invoker是否需要被销毁
 * 如果设置url属性：refer.autodestroy=false
 * invokers将会在不减少的情况下增加，可能会有一个引用泄露
 * @param oldUrlInvokerMap
 * @param newUrlInvokerMap
 */
private void destroyUnusedInvokers(Map<String, Invoker<T>> oldUrlInvokerMap,Map<String, Invoker<T>> newUrlInvokerMap) {
	if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0) {
	    //关闭所有的invoker
	    destroyAllInvokers();
	    return;
	}
	List<String> deleted = null;
	if (oldUrlInvokerMap != null) {
	    Collection<Invoker<T>> newInvokers = newUrlInvokerMap.values();
	    for (Map.Entry<String, Invoker<T>> entry : oldUrlInvokerMap.entrySet()) {
		if (!newInvokers.contains(entry.getValue())) {
		    //老的invoker在新的invoker列表中不存在
		    if (deleted == null) {
			deleted = new ArrayList<String>();
		    }
		    //标记该老的url，后面会进行删除
		    deleted.add(entry.getKey());
		}
	    }
	}
	if (deleted != null) {
	    for (String url : deleted) {
		if (url != null) {
		    //从老的invoker-map中移除该url，并关闭该老的invoker
		    Invoker<T> invoker = oldUrlInvokerMap.remove(url);
		    if (invoker != null) {
			try {
			    //销毁该invoker
			    invoker.destroy();
			} catch (Exception e) {
			    logger.warn("destory invoker[" + invoker.getUrl() + "] faild. " + e.getMessage(), e);
			}
		    }
		}
	    }
	}
}
```
#### 重新引用invoker

接下来，我们来看看上面的toInvokers方法中，重新引用invoker的逻辑，即
```java
if (enabled) {
	//可以引用，重新引用
	invoker = new InvokerDelegate<T>(
		protocol.refer(serviceType, url),
		url,
		providerUrl
	);
}
```
这里会先调用ProtocolListenerWrapper类的refer方法,然后在该方法内会在调用ProtocolFilterWrapper类的refer方法
```java
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
	if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
	    return protocol.refer(type, url);
	}
	return new ListenerInvokerWrapper<T>(
	        //调用ProtocolFilterWrapper类的refer方法获取到invoker
		protocol.refer(type, url),
		Collections.unmodifiableList(
		      ExtensionLoader.getExtensionLoader(InvokerListener.class)
				     .getActivateExtension(url, Constants.INVOKER_LISTENER_KEY)
	));
}
```
我们看下ProtocolFilterWrapper类的refer方法，在该方法内部会去调用DubboProtocol类的refer方法
```java
@Override
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
	if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
		return protocol.refer(type, url);
	}
	//构建invoker链
	return buildInvokerChain(
		//调用DubboProtocol类的refer方法
		protocol.refer(type, url), 
		//reference.filter
		Constants.REFERENCE_FILTER_KEY, 
		//consumer
		Constants.CONSUMER
	);
}

/**
 * 构建Invoker链
 * @param invoker
 * @param key  
 * @param group 
 */
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker,String key, String group) {
	Invoker<T> last = invoker;
	//获取过滤器：ConsumerContextFilter、FutureFilter、MonitorFilter
	List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
	if (!filters.isEmpty()) {
	    for (int i = filters.size() - 1; i >= 0; i--) {
		//MonitorFilter、FutureFilter、ConsumerContextFilter
		final Filter filter = filters.get(i);
		final Invoker<T> next = last;
		last = new Invoker<T>() {
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
		        //调用filter的invoke方法
			return filter.invoke(next, invocation);
		    }
		    @Override
		    public void destroy() {
			invoker.destroy();
		    }
		    @Override
		    public String toString() {
			return invoker.toString();
		    }
		};
	    }
	}
	return last;
}
```
DubboProtocol类的refer方法
```java
/**
 * @param serviceType com.alibaba.dubbo.demo.DemoService
 * @param url 
 * dubbo://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-consumer&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=5624&qos.port=33333&register.ip=192.168.99.60&remote.timestamp=1535531661191&side=consumer&timestamp=1535531690333
 */
@Override
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
	//加载优化序列化类
	optimizeSerialization(url);
	
	//创建DubboInvoker
	DubboInvoker<T> invoker = new DubboInvoker<T>(
		serviceType,
		url,
		//创建客户端(后面小节会分析该方法)
		getClients(url),
		//当前对象已创建的invoker集合
		invokers
	);
	//将invoker保存到invokers集合中
	invokers.add(invoker);
	return invoker;
}
```
该invoker中持有一个client对象，默认是NettyClient，后面小节会介绍该过程，最终生成的Invoker如下图，：
![](img/InvokerChain.png)

#### registerConsumer注册消费者

最后调用ProviderConsumerRegTable类的registerConsumer方法注册消费者
```java
public static ConcurrentHashMap<String, Set<ConsumerInvokerWrapper>> consumerInvokers = new ConcurrentHashMap<String, Set<ConsumerInvokerWrapper>>();

/**
 * 注册消费者
 * @param invoker
 * @param registryUrl 注册中心url
 * @param consumerUrl 消费者url
 * @param registryDirectory
 */
public static void registerConsumer(Invoker invoker, URL registryUrl, URL consumerUrl, RegistryDirectory registryDirectory) {
	//创建ConsumerInvokerWrapper
	ConsumerInvokerWrapper wrapperInvoker = new ConsumerInvokerWrapper(invoker, registryUrl, consumerUrl, registryDirectory);
	//服务唯一标识
	String serviceUniqueName = consumerUrl.getServiceKey();
	//根据服务唯一标识获取invokers
	Set<ConsumerInvokerWrapper> invokers = consumerInvokers.get(serviceUniqueName);
	if (invokers == null) {
	    consumerInvokers.putIfAbsent(serviceUniqueName, new ConcurrentHashSet<ConsumerInvokerWrapper>());
	    invokers = consumerInvokers.get(serviceUniqueName);
	}
	//添加wrapperInvoker到缓存
	invokers.add(wrapperInvoker);
}
```

到此为止，整个流程就介绍完毕了。下面小节将会介绍一下注册中心Registry。
