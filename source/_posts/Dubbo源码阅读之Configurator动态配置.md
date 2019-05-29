---
title: Dubbo源码阅读之Configurator动态配置
date: 2018-09-05 12:20:32
tags:
    - dubbo
---
>我们可以编写动态配置来配置服务提供者。这个操作通常在监控中心完成。

### 官网文档
我们先看下官网给的文档说明
```java
RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();
//获取注册中心
Registry registry = registryFactory.getRegistry(URL.valueOf("zookeeper://10.20.153.10:2181"));
//注册配置
registry.register(URL.valueOf("override://0.0.0.0/com.foo.BarService?category=configurators&dynamic=false&application=foo&timeout=1000"));
```
在这个配置override url中：
* override:// 表示数据将会被覆盖，当前dubbo支持override和absent，可以自行扩展，必填参数.
* 0.0.0.0 表示该配置对所有IP地址都有效，如果只想覆盖指定的ip数据，则可以替换指定的ip地址，必填参数.
* com.foo.BarService 表示对指定的服务有效，必填参数.
* category=configurators 表示数据是动态配置的，必填参数.
* dynamic=false 表示数据是持久化的，当注册方撤销时，数据仍存储在注册表中。
* enabled=true 启用覆盖策略，可以不传，不传的话，默认值为启用
* application=foo 表示对指定的application有效，可以不传，不传的话则对所有应用程序有效。
* timeout=1000 表示满足上述条件的timeout参数的值将会被1000覆盖，如果想要覆盖其他参数，则直接添加到override URL参数上。

#### 例子
* 禁用服务提供者(通常用于临时踢掉提供者机器，类似于禁止消费者访问，请使用路由规则)
```java
override://10.20.153.10/com.foo.BarService?category=configurators&dynamic=false&disbaled=true
```

* 调整权重:(通常用于容量评估，默认为100)
```java
override://10.20.153.10/com.foo.BarService?category=configurators&dynamic=false&weight=200
```

* 调整负载均衡策略(默认策略为随机)
```java
override://10.20.153.10/com.foo.BarService?category=configurators&dynamic=false&loadbalance=leastactive
```

* 服务降级:(通常用于暂时屏蔽非关键服务的错误）
```java
override://0.0.0.0/com.foo.BarService?category=configurators&dynamic=false&application=foo&mock=force:return+null
```

现在我们开始看源码实现
### Configurator接口
```java
public interface Configurator extends Comparable<Configurator> {

    /**
     * 获取配置url
     */
    URL getUrl();

    /**
     * 配置服务提供者url
     * 向url中添加新属性(absent) 或者 覆盖url中的属性(override).(新属性来源于配置url)
     * @param url 旧的提供者url
     * @return  新的提供者url
     */
    URL configure(URL url);
}
```

#### AbstractConfigurator抽象类
该抽象类实现了Configurator接口。并实现了configure方法，在该方法中，会判断当前url是否满足覆盖url的条件。如果满足的话，会调用抽象方法doConfigure执行相应的配置。doConfigure抽象方法由两个子类AbsentConfigurator(absent)和OverrideConfigurator(override)进行实现。
```java
public abstract class AbstractConfigurator implements Configurator {

    /**
     * 配置url
     */
    private final URL configuratorUrl;

    public AbstractConfigurator(URL url) {
        if (url == null) {
            throw new IllegalArgumentException("configurator url == null");
        }
        this.configuratorUrl = url;
    }

    @Override
    public URL getUrl() {
        return configuratorUrl;
    }

    /**
     * @param url 旧的url
     * @return
     */
    @Override
    public URL configure(URL url) {
        if (configuratorUrl == null || configuratorUrl.getHost() == null
                || url == null || url.getHost() == null) {
            return url;
        }
        //如果override url存在端口，则意味着它是服务提供者地址
        //我们想要这个override url控制一个指定的服务提供者
        //该覆盖url规则可能对特定的服务提供者实例生效，或者对持有这个服务提供者实例的消费者生效
        if (configuratorUrl.getPort() != 0) {
            if (url.getPort() == configuratorUrl.getPort()) {
                //服务提供者
                return configureIfMatch(url.getHost(), url);
            }
        } else {
            //configuratorUrl没有端口，意味着这个url的ip指定的是一个消费者地址或者0.0.0.0
            //如果它是一个消费者ip地址，目的是控制一个特定的消费者实例，
            //它必须在消费者端生效，任何提供者都将会忽略该url
            //如果ip是0.0.0.0，则该配置url对消费者、生产者都生效
            if (url.getParameter(Constants.SIDE_KEY, Constants.PROVIDER).equals(Constants.CONSUMER)) {
                //消费者
                //NetUtils.getLocalHost is the ip address consumer registered to registry.
                //NetUtils.getLocalHost是一个注册到注册中心的消费者ip地址
                return configureIfMatch(NetUtils.getLocalHost(), url);
            } else if (url.getParameter(Constants.SIDE_KEY, Constants.CONSUMER).equals(Constants.PROVIDER)) {
                //生产者
                // take effect on all providers, so address must be 0.0.0.0,
                // otherwise it won't flow to this if branch
                //对所有生产者生效，因此地址必须是0.0.0.0，否则，它将不会进入这个分支
                return configureIfMatch(Constants.ANYHOST_VALUE, url);
            }
        }
        return url;
    }

    /**
     * @param host 受影响的主机地址
     * @param url
     * @return
     */
    private URL configureIfMatch(String host, URL url) {
        if (Constants.ANYHOST_VALUE.equals(configuratorUrl.getHost())
                || host.equals(configuratorUrl.getHost())) {
            //配置url的host等于0.0.0.0，或者等于host
            
            //获取配置url的application参数值，如果为空，则获取username属性
            String configApplication = configuratorUrl.getParameter(Constants.APPLICATION_KEY,configuratorUrl.getUsername());
            //获取当前url的application参数值，如果为空，则获取username属性
            String currentApplication = url.getParameter(Constants.APPLICATION_KEY, url.getUsername());

            if (configApplication == null || Constants.ANY_VALUE.equals(configApplication)
                    || configApplication.equals(currentApplication)) {
                //配置url的configApplication = (null || * || currentApplication)
                Set<String> condtionKeys = new HashSet<String>();
                
		//添加category、check、dynamic、enabled参数
                condtionKeys.add(Constants.CATEGORY_KEY);
                condtionKeys.add(Constants.CHECK_KEY);
                condtionKeys.add(Constants.DYNAMIC_KEY);
                condtionKeys.add(Constants.ENABLED_KEY);
                
		//遍历配置url的参数列表
                for (Map.Entry<String, String> entry : configuratorUrl.getParameters().entrySet()) {
                    //参数key
                    String key = entry.getKey();
                    //参数value
                    String value = entry.getValue();
                    
		    if (key.startsWith("~") || Constants.APPLICATION_KEY.equals(key) || Constants.SIDE_KEY.equals(key)) {
			//参数key = (application || side || ^~ )
                        
			//添加该key参数
                        condtionKeys.add(key);
                        
			if (value != null && !Constants.ANY_VALUE.equals(value)
                                && !value.equals(url.getParameter(key.startsWith("~") ? key.substring(1) : key))) {
                            //如果 当前url的key参数的值 != * 并且 不等于 配置url的key参数的值，则直接返回当前url
                            return url;
                        }
                    }
                }
                //从配置url中移除condtionKeys参数，然后执行配置
                return doConfigure(url, configuratorUrl.removeParameters(condtionKeys));
            }
        }
        return url;
    }

    /**
     * 1、具有特定主机IP的URL比0.0.0.0的优先级高
     * 2、如果两个url有相同的host，比较priority字段的值
     */
    @Override
    public int compareTo(Configurator o) {
        if (o == null) {
            return -1;
        }

        int ipCompare = getUrl().getHost().compareTo(o.getUrl().getHost());
        if (ipCompare == 0) {
            int i = getUrl().getParameter(Constants.PRIORITY_KEY, 0),
                    j = o.getUrl().getParameter(Constants.PRIORITY_KEY, 0);
            if (i < j) {
                return -1;
            } else if (i > j) {
                return 1;
            } else {
                return 0;
            }
        } else {
            return ipCompare;
        }
    }
    
    //抽象方法，执行相应配置策略
    protected abstract URL doConfigure(URL currentUrl, URL configUrl);

}
```
#### AbsentConfigurator实现类
```java
public class AbsentConfigurator extends AbstractConfigurator {

    public AbsentConfigurator(URL url) {
        super(url);
    }
    
    /**
     * @currentUrl 服务提供者url
     * @configUrl  配置url
     */
    @Override
    public URL doConfigure(URL currentUrl, URL configUrl) {
        //将配置url中的参数添加到服务提供者url参数中(只会添加不存在的，不会覆盖已存在的参数)
        return currentUrl.addParametersIfAbsent(configUrl.getParameters());
    }
}

/**
 * 只有原服务提供者url中不包含该参数时，才会添加，不会覆盖
 */
public URL addParametersIfAbsent(Map<String, String> parameters) {
        if (parameters == null || parameters.size() == 0) {
            return this;
        }
        //A-1,B-2,C-2 覆盖url
        //A-2,B-2     原服务提供者url参数
        //A-2,B-2,C-2 新的服务提供者url参数
        Map<String, String> map = new HashMap<String, String>(parameters);
        map.putAll(getParameters());
        return new URL(protocol, username, password, host, port, path, map);
}
```

#### OverrideConfigurator实现类
```java
public class OverrideConfigurator extends AbstractConfigurator {

    public OverrideConfigurator(URL url) {
        super(url);
    }
 
    /**
     * @currentUrl 服务提供者url
     * @configUrl  配置url
     */
    @Override
    public URL doConfigure(URL currentUrl, URL configUrl) {
        //将配置url中的参数添加到服务提供者url参数中
        return currentUrl.addParameters(configUrl.getParameters());
    }
}

/**
 * 将配置url中的参数添加到服务提供者url参数中(会覆盖)
 */
public URL addParameters(Map<String, String> parameters) {
	if (parameters == null || parameters.size() == 0) {
	    return this;
	}
	//参数值没有发生变化
	boolean hasAndEqual = true;
	for (Map.Entry<String, String> entry : parameters.entrySet()) {
	    //获取当前服务提供者url的参数值
	    String value = getParameters().get(entry.getKey());
	    if (value == null) {
		//当前服务提供者url中的参数值为空，并且覆盖url中的参数值不为空
		if (entry.getValue() != null) {
		    hasAndEqual = false;
		    break;
		}
	    } else {
		//当前服务提供者url中的参数值不为空，并且和覆盖url中的参数值不相等
		if (!value.equals(entry.getValue())) {
		    hasAndEqual = false;
		    break;
		}
	    }
	}
	if (hasAndEqual) {
	    //没有发生变化。立即返回
	    return this;
	}
	Map<String, String> map = new HashMap<String, String>(getParameters());
	//使用配置url中的参数值覆盖当前服务提供者url中的参数值
	map.putAll(parameters);
	return new URL(protocol, username, password, host, port, path, map);
}
```

#### 配置工厂类
工厂类用来创建相应的配置策略实现类
```java
public class AbsentConfiguratorFactory implements ConfiguratorFactory {
    @Override
    public Configurator getConfigurator(URL url) {
        return new AbsentConfigurator(url);
    }
}

public class OverrideConfiguratorFactory implements ConfiguratorFactory {
    @Override
    public Configurator getConfigurator(URL url) {
        return new OverrideConfigurator(url);
    }
}
```
配置扩展
```java
override=com.alibaba.dubbo.rpc.cluster.configurator.override.OverrideConfiguratorFactory
absent=com.alibaba.dubbo.rpc.cluster.configurator.absent.AbsentConfiguratorFactory
```

自定义的配置策略实现，可以自行参照AbsentConfigurator类进行实现，本小节就先介绍到这里了。