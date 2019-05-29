---
title: Dubbo源码阅读之注册中心-Zookeeper注册中心
date: 2018-08-23 20:24:06
tags:
    - dubbo
---
>本小节讲解Zookeeper注册中心的实现


### ZookeeperRegistry注册中心
先来看下后面会用到的接口：
```java
/**
 * 节点监听
 */
public interface ChildListener {
    /**
     * 节点发生改变
     * @param path
     * @param children
     */
    void childChanged(String path, List<String> children);
}
/**
 * 连接状态监听器
 */
public interface StateListener {
    //断开连接
    int DISCONNECTED = 0;
    //已连接
    int CONNECTED = 1;
    //重新连接
    int RECONNECTED = 2;

    /**
     * 连接状态改变
     * @param connected 状态值
     */
    void stateChanged(int connected);
}
```
然后我们来看ZookeeperRegistry的实现,该类继承自FailbackRegistry,因此具备了发生异常时自动重试的能力
```java
public class ZookeeperRegistry extends FailbackRegistry {

    private final static Logger logger = LoggerFactory.getLogger(ZookeeperRegistry.class);

    /**
     * 默认的zk端口
     */
    private final static int DEFAULT_ZOOKEEPER_PORT = 2181;

    /**
     * 默认的zk根节点
     */
    private final static String DEFAULT_ROOT = "dubbo";

    /**
     * zk根节点，如：/dubbo
     */
    private final String root;

    private final Set<String> anyServices = new ConcurrentHashSet<String>();

    /**
     * URL：订阅url,<节点监听事件>
     */
    private final ConcurrentMap<URL, ConcurrentMap<NotifyListener, ChildListener>>
            zkListeners = new ConcurrentHashMap<URL, ConcurrentMap<NotifyListener, ChildListener>>();

    /**
     * zk客户端接口(支持多种客户端实现)
     */
    private final ZookeeperClient zkClient;

    public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
        //设置注册中心url
        super(url);
        if (url.isAnyHost()) {
            //host = 0.0.0.0 || anyhost = true
            throw new IllegalStateException("registry address == null");
        }
        //获取url参数group,标识zk根节点名称,默认为dubbo
        String group = url.getParameter(Constants.GROUP_KEY, DEFAULT_ROOT);
        if (!group.startsWith(Constants.PATH_SEPARATOR)) {
            //添加前缀，如：/dubbo
            group = Constants.PATH_SEPARATOR + group;
        }
        this.root = group;

        //调用zookeeperTransporter接口的connect方法连接到zk客户端(后面会分析该方法)
        zkClient = zookeeperTransporter.connect(url);

        //添加节点变更事件到AbstractZookeeperClient父类中的缓存集合变量中
        zkClient.addStateListener(new StateListener() {
            @Override
            public void stateChanged(int state) {
                if (state == RECONNECTED) {
                    try {
                        //重连事件，进行恢复
                        recover();
                    } catch (Exception e) {
                        logger.error(e.getMessage(), e);
                    }
                }
            }
        });
    }

    /**
     * 给zk地址附加默认端口号
     * @param address
     * @return
     */
    static String appendDefaultPort(String address) {
        if (address != null && address.length() > 0) {
            int i = address.indexOf(':');
            if (i < 0) {
                //添加zk默认端口2181，即：address:2181
                return address + ":" + DEFAULT_ZOOKEEPER_PORT;
            } else if (Integer.parseInt(address.substring(i + 1)) == 0) {
                //端口为0的话，则使用默认端口
                return address.substring(0, i + 1) + DEFAULT_ZOOKEEPER_PORT;
            }
        }
        return address;
    }

    @Override
    public boolean isAvailable() {
        //连接是否可用
        return zkClient.isConnected();
    }

    @Override
    public void destroy() {
        super.destroy();
        try {
            //销毁客户端
            zkClient.close();
        } catch (Exception e) {
            logger.warn("Failed to close zookeeper client " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

    @Override
    protected void doRegister(URL url) {
        try {
            //执行注册 dubbo://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=5312&side=provider&timestamp=1534994264738
            //根据url确定节点路径；根据url的dynamic参数确定是否是临时节点
	    //如：/dubbo/com.alibaba.dubbo.demo.DemoService/providers/dubbo%3A%2F%2F192.168.99.60%3A20880%2Fcom.alibaba.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddemo-provider%26dubbo%3D2.0.0%26generic%3Dfalse%26interface%3Dcom.alibaba.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D5312%26side%3Dprovider%26timestamp%3D1534994264738
            //在zk上创建节点
	    zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

    @Override
    protected void doUnregister(URL url) {
        try {
            //执行取消注册
            //删除url节点路径
            zkClient.delete(toUrlPath(url));
        } catch (Throwable e) {
            throw new RpcException("Failed to unregister " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

    @Override
    protected void doUnsubscribe(URL url, NotifyListener listener) {
        //执行取消订阅
        //根据订阅url获取监听
        ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
        if (listeners != null) {
            //根据NotifyListener获取zk监听
            ChildListener zkListener = listeners.get(listener);
            if (zkListener != null) {
                //从zk上移除节点监听
                zkClient.removeChildListener(toUrlPath(url), zkListener);
            }
        }
    }

    /**
     * 执行订阅
     * @param url
     * 例如： provider://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&category=configurators&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=6504&side=provider&timestamp=1534927826842
     *       consumer://192.168.99.60/com.alibaba.dubbo.demo.DemoService?application=demo-consumer&category=providers,configurators,routers&check=false&dubbo=2.0.0&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=1880&qos.port=33333&side=consumer&timestamp=1535449640995
     * @param listener 例如：OverrideListener
     */
    @Override
    protected void doSubscribe(final URL url, final NotifyListener listener) {
        try {
            //执行订阅
            //查看订阅url的interface属性是否=*
            if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {
                String root = toRootPath();
                //获取订阅url的监听列表
                ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                if (listeners == null) {
                    zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                    listeners = zkListeners.get(url);
                }
                //获取listener对应的zk监听器
                ChildListener zkListener = listeners.get(listener);
                if (zkListener == null) {
                    //如果zk监听器为空，则新建一个，并放入zkListeners集合中
                    listeners.putIfAbsent(listener, new ChildListener() {
                        @Override
                        public void childChanged(String parentPath, List<String> currentChilds) {
                            for (String child : currentChilds) {
                                child = URL.decode(child);
                                if (!anyServices.contains(child)) {
                                    //记录节点child
                                    anyServices.add(child);
                                    //订阅child
                                    //添加interface = child
                                    //添加check = false
                                    subscribe(url.setPath(child).addParameters(Constants.INTERFACE_KEY,child,
                                            Constants.CHECK_KEY, String.valueOf(false)), listener);
                                }
                            }
                        }
                    });
                    zkListener = listeners.get(listener);
                }
                //创建root持久化节点
                zkClient.create(root, false);
                //添加root节点监听，services为root节点的子节点列表
                List<String> services = zkClient.addChildListener(root, zkListener);
                if (services != null && !services.isEmpty()) {
                    for (String service : services) {
                        //子节点
                        service = URL.decode(service);
                        anyServices.add(service);
                        //订阅子节点
                        subscribe(url.setPath(service).addParameters(Constants.INTERFACE_KEY, service,
                                Constants.CHECK_KEY, String.valueOf(false)), listener);
                    }
                }
            } else {
                List<URL> urls = new ArrayList<URL>();
                //遍历url的类别列表 /dubbo/com.alibaba.dubbo.demo.DemoService/configurators
                for (String path : toCategoriesPath(url)) {
		    //根据订阅url获取监听
                    ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                    if (listeners == null) {
                        zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                        listeners = zkListeners.get(url);
                    }
                    ChildListener zkListener = listeners.get(listener);
                    if (zkListener == null) {
		        //添加为空，则新建一个节点监听
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
                    //添加path节点监听zkListener
                    List<String> children = zkClient.addChildListener(path, zkListener);
                    if (children != null) {
			//empty://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&category=configurators&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=6504&side=provider&timestamp=1534927826842
                        urls.addAll(toUrlsWithEmpty(url, path, children));
                    }
                }
                //通知消费者(urls即为待通知的消息)
                notify(url, listener, urls);
            }
        } catch (Throwable e) {
            throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

    @Override
    public List<URL> lookup(URL url) {
        if (url == null) {
            throw new IllegalArgumentException("lookup url == null");
        }
        try {
            //找到的所有的子节点
            List<String> providers = new ArrayList<String>();
            //遍历url类别数组 /dubbo/com.alibaba.dubbo.demo.DemoService/providers
            for (String path : toCategoriesPath(url)) {
                //获取path节点的子节点
                List<String> children = zkClient.getChildren(path);
                if (children != null) {
                    providers.addAll(children);
                }
            }
            return toUrlsWithoutEmpty(url, providers);
        } catch (Throwable e) {
            throw new RpcException("Failed to lookup " + url + " from zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

    /**
     * 如果root = /,则返回 / ,否则返回 /dubbo/
     * @return
     */
    private String toRootDir() {
        if (root.equals(Constants.PATH_SEPARATOR)) {
            return root;
        }
        return root + Constants.PATH_SEPARATOR;
    }

    private String toRootPath() {
        return root;
    }

    /**
     * 获取服务地址
     * @param url
     * @return  /dubbo/  或者 /dubbo/编码后的interface
     */
    private String toServicePath(URL url) {
        String name = url.getServiceInterface();
        if (Constants.ANY_VALUE.equals(name)) {
            //interface = *，则返回: /dubbo/
            return toRootPath();
        }
        //返回： /dubbo/编码后的interface
        return toRootDir() + URL.encode(name);
    }

    /**
     * 根据url获取类别数组
     * @param url
     * @return
     */
    private String[] toCategoriesPath(URL url) {
        String[] categories;
        if (Constants.ANY_VALUE.equals(url.getParameter(Constants.CATEGORY_KEY))) {
            //url的category参数=*，则取所有的类别，即 categories = providers、consumers、routers、configurators
            categories = new String[]{
			Constants.PROVIDERS_CATEGORY, 
			Constants.CONSUMERS_CATEGORY,
			Constants.ROUTERS_CATEGORY, 
			Constants.CONFIGURATORS_CATEGORY
	    };
        } else {
            //如果category参数为空，则取默认类别：providers
            categories = url.getParameter(Constants.CATEGORY_KEY, new String[]{Constants.DEFAULT_CATEGORY});
        }
        String[] paths = new String[categories.length];
        for (int i = 0; i < categories.length; i++) {
            //例如：/dubbo/com.xxx.demoService/providers (interface != *)
            //例如：/dubbo/providers (interface = *)
            paths[i] = toServicePath(url) + Constants.PATH_SEPARATOR + categories[i];
        }
        return paths;
    }

    /**
     * 获取url对应的类别地址
     * /dubbo/com.alibaba.dubbo.demo.demoService/providers
     * @param url
     * @return
     */
    private String toCategoryPath(URL url) {
        return toServicePath(url) + Constants.PATH_SEPARATOR + url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
    }

    /**
     * 获取url对应的url地址(将会在zk上创建该地址)
     * @param url
     * @return /dubbo/com.alibaba.dubbo.demo.DemoService/providers/dubbo://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=7112&side=provider&timestamp=1534931882323
     */
    private String toUrlPath(URL url) {
        return toCategoryPath(url) + Constants.PATH_SEPARATOR + URL.encode(url.toFullString());
    }

    /**
     * 返回值为空的话，不会返回默认值
     * @param consumer
     * @param providers
     * @return
     */
    private List<URL> toUrlsWithoutEmpty(URL consumer, List<String> providers) {
        List<URL> urls = new ArrayList<URL>();
        if (providers != null && !providers.isEmpty()) {
            for (String provider : providers) {
                //解码
                provider = URL.decode(provider);
                if (provider.contains("://")) {
                    URL url = URL.valueOf(provider);
                    //是否匹配
                    if (UrlUtils.isMatch(consumer, url)) {
                        urls.add(url);
                    }
                }
            }
        }
        return urls;
    }

    /**
     * 如果为空的话，则返回一个默认值
     * @param consumer provider://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&category=configurators&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=6504&side=provider&timestamp=1534927826842
     * @param path /dubbo/com.alibaba.dubbo.demo.DemoService/configurators
     * @param providers
     * @return
     */
    private List<URL> toUrlsWithEmpty(URL consumer, String path, List<String> providers) {
        List<URL> urls = toUrlsWithoutEmpty(consumer, providers);
        if (urls == null || urls.isEmpty()) {
            int i = path.lastIndexOf('/');
            //获取类别
            String category = i < 0 ? path : path.substring(i + 1);
            //设置empty协议，并设置category属性
            URL empty = consumer.setProtocol(Constants.EMPTY_PROTOCOL).addParameter(Constants.CATEGORY_KEY, category);
            urls.add(empty);
        }
        return urls;
    }
}
```
可以看到zk注册中心根据NotifyListener接口与RegistryDirectory类进行通信，通过notify方法通知消费者有更新。通过ZookeeperClient与Zookeeper进行交互。
接下来，我们来看下是dubbo是如何通过工厂创建ZookeeperRegistry实例的。

### Zookeeper注册中心工厂
先来看下ZookeeperRegistryFactory类，该类继承自AbstractRegistryFactory，用来创建具体的ZookeeperRegistry实例
```java
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {

    private ZookeeperTransporter zookeeperTransporter;

    public void setZookeeperTransporter(ZookeeperTransporter zookeeperTransporter) {
        this.zookeeperTransporter = zookeeperTransporter;
    }
   
    /**
     * @param url 注册中心url
     */
    @Override
    public Registry createRegistry(URL url) {
        //创建zk注册中心
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }
}
```
ZookeeperTransporter是一个接口，dubbo支持多个zk客户端实现，例如Curator、ZkClient，该接口就是用来创建具体的客户端实现的，可以看到默认是使用curator客户端。
```java
@SPI("curator")
public interface ZookeeperTransporter {
    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    ZookeeperClient connect(URL url);
}

public class CuratorZookeeperTransporter implements ZookeeperTransporter {
    @Override
    public ZookeeperClient connect(URL url) {
        //返回Curator实现
        return new CuratorZookeeperClient(url);
    }
}

public class ZkclientZookeeperTransporter implements ZookeeperTransporter {
    @Override
    public ZookeeperClient connect(URL url) {
        //返回Zkclient实现
        return new ZkclientZookeeperClient(url);
    }
}
```
下面我们就来看看各个ZookeeperClient的实现。

#### ZookeeperClient接口
该接口定义了一些针对zk的基本操作
```java
public interface ZookeeperClient {
    /**
     * 创建一个节点
     * @param path 节点路径
     * @param ephemeral 是否临时节点
     */
    void create(String path, boolean ephemeral);

    /**
     * 删除一个节点
     * @param path
     */
    void delete(String path);

    /**
     * 获取某个节点的子节点路径
     * @param path
     * @return
     */
    List<String> getChildren(String path);

    /**
     * 添加节点监听器
     * @param path 节点路径
     * @param listener 监听器
     * @return
     */
    List<String> addChildListener(String path, ChildListener listener);

    /**
     * 移除节点监听器
     * @param path
     * @param listener
     */
    void removeChildListener(String path, ChildListener listener);

    /**
     * 添加变更事件监听
     * @param listener
     */
    void addStateListener(StateListener listener);
    /**
     * 移除变更事件监听
     */
    void removeStateListener(StateListener listener);
    /**
     * 是否已连接
     * @return
     */
    boolean isConnected();
    /**
     * 关闭
     */
    void close();
    /**
     * 获取注册中心url
     */
    URL getUrl();
}
```
##### AbstractZookeeperClient抽象类
```java
public abstract class AbstractZookeeperClient<TargetChildListener> implements ZookeeperClient {

    protected static final Logger logger = LoggerFactory.getLogger(AbstractZookeeperClient.class);

    /**
     * 注册中心url
     */
    private final URL url;

    /**
     * 缓存监听
     */
    private final Set<StateListener> stateListeners = new CopyOnWriteArraySet<StateListener>();

    /**
     * 节点监听缓存
     */
    private final ConcurrentMap<String, ConcurrentMap<ChildListener, TargetChildListener>>
            childListeners = new ConcurrentHashMap<String, ConcurrentMap<ChildListener, TargetChildListener>>();

    /**
     * 客户端是否已停止
     */
    private volatile boolean closed = false;

    public AbstractZookeeperClient(URL url) {
        this.url = url;
    }

    @Override
    public URL getUrl() {
        return url;
    }

    @Override
    public void create(String path, boolean ephemeral) {
        int i = path.lastIndexOf('/');
        if (i > 0) {
            //获取到父路径
            String parentPath = path.substring(0, i);
            if (!checkExists(parentPath)) {
                //父路径不存在的话，进行递归创建
                create(parentPath, false);
            }
        }
        if (ephemeral) {
            //创建临时节点
            createEphemeral(path);
        } else {
            //创建持久节点
            createPersistent(path);
        }
    }

    @Override
    public void addStateListener(StateListener listener) {
        stateListeners.add(listener);
    }
    @Override
    public void removeStateListener(StateListener listener) {
        stateListeners.remove(listener);
    }
    public Set<StateListener> getSessionListeners() {
        return stateListeners;
    }
    @Override
    public List<String> addChildListener(String path, final ChildListener listener) {
        //从缓存中获取path节点的监听
        ConcurrentMap<ChildListener, TargetChildListener> listeners = childListeners.get(path);
        if (listeners == null) {
            childListeners.putIfAbsent(path, new ConcurrentHashMap<ChildListener, TargetChildListener>());
            listeners = childListeners.get(path);
        }
        TargetChildListener targetListener = listeners.get(listener);
        if (targetListener == null) {
            //为节点path新创建一个监听，并放入listeners中
            listeners.putIfAbsent(listener, createTargetChildListener(path, listener));
            targetListener = listeners.get(listener);
        }
        //添加目标节点监听
        return addTargetChildListener(path, targetListener);
    }

    @Override
    public void removeChildListener(String path, ChildListener listener) {
        //从缓存中获取path节点的监听
        ConcurrentMap<ChildListener, TargetChildListener> listeners = childListeners.get(path);
        if (listeners != null) {
            //从缓存中移除监听listener
            TargetChildListener targetListener = listeners.remove(listener);
            if (targetListener != null) {
                //移除目标节点的监听targetListener
                removeTargetChildListener(path, targetListener);
            }
        }
    }

    /**
     * 状态变更事件
     * @param state
     */
    protected void stateChanged(int state) {
        //遍历所有的监听器
        for (StateListener sessionListener : getSessionListeners()) {
	    //执行变更事件
            sessionListener.stateChanged(state);
        }
    }

    @Override
    public void close() {
        if (closed) {
	    //已关闭
            return;
        }
        closed = true;
        try {
            doClose();
        } catch (Throwable t) {
            logger.warn(t.getMessage(), t);
        }
    }
    //************模板方法************
    //关闭
    protected abstract void doClose();
    //创建持久节点
    protected abstract void createPersistent(String path);
    //创建临时节点
    protected abstract void createEphemeral(String path);
    //检测节点是否存在
    protected abstract boolean checkExists(String path);
    //创建目标节点监听
    protected abstract TargetChildListener createTargetChildListener(String path, ChildListener listener);
    //添加目标节点监听
    protected abstract List<String> addTargetChildListener(String path, TargetChildListener listener);
    //移除目标节点监听
    protected abstract void removeTargetChildListener(String path, TargetChildListener listener);
}
```
##### CuratorZookeeperClient实现
```java
public class CuratorZookeeperClient extends AbstractZookeeperClient<CuratorWatcher> {
    
    //Curator客户端，对zk的操作都是由它来完成
    private final CuratorFramework client;

    public CuratorZookeeperClient(URL url) {
        //设置注册中心url
        super(url);
        try {
            CuratorFrameworkFactory.Builder builder = CuratorFrameworkFactory.builder()
                    //指定注册中心url地址,多个地址使用逗号分隔
                    .connectString(url.getBackupAddress())
                    //设置重试策略，最大重试1次，重试间隔1000毫秒
                    .retryPolicy(new RetryNTimes(1, 1000))
                    //设置连接超时,单位ms,默认1500ms,这里设置5秒
                    .connectionTimeoutMs(5000);
            //获取url的username:password
            String authority = url.getAuthority();
            if (authority != null && authority.length() > 0) {
                //添加授权
                builder = builder.authorization("digest", authority.getBytes());
            }
            //生成curator客户端
            client = builder.build();
            //监听客户端链接状态
            client.getConnectionStateListenable().addListener(new ConnectionStateListener() {
                @Override
                public void stateChanged(CuratorFramework client, ConnectionState state) {
                    if (state == ConnectionState.LOST) {
                        //断开状态，则触发链接断开事件
                        CuratorZookeeperClient.this.stateChanged(StateListener.DISCONNECTED);
                    } else if (state == ConnectionState.CONNECTED) {
                        //已连接状态，则触发链接事件
                        CuratorZookeeperClient.this.stateChanged(StateListener.CONNECTED);
                    } else if (state == ConnectionState.RECONNECTED) {
                        //重连状态，则触发重连事件
                        CuratorZookeeperClient.this.stateChanged(StateListener.RECONNECTED);
                    }
                }
            });
            //启动客户端
            client.start();
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }

    @Override
    public void createPersistent(String path) {
        try {
            //创建持久化节点path
            client.create().forPath(path);
        } catch (NodeExistsException e) {
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }

    @Override
    public void createEphemeral(String path) {
        try {
            //创建临时节点path
            client.create().withMode(CreateMode.EPHEMERAL).forPath(path);
        } catch (NodeExistsException e) {
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }

    @Override
    public void delete(String path) {
        try {
            //删除节点path
            client.delete().forPath(path);
        } catch (NoNodeException e) {
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }

    @Override
    public List<String> getChildren(String path) {
        try {
            //获取path节点的子节点列表
            return client.getChildren().forPath(path);
        } catch (NoNodeException e) {
            return null;
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }

    @Override
    public boolean checkExists(String path) {
        try {
            //检测path节点是否已存在
            if (client.checkExists().forPath(path) != null) {
                return true;
            }
        } catch (Exception e) {
        }
        return false;
    }
    @Override
    public boolean isConnected() {
        //是否已连接状态
        return client.getZookeeperClient().isConnected();
    }

    @Override
    public void doClose() {
        //关闭客户端
        client.close();
    }

    @Override
    public CuratorWatcher createTargetChildListener(String path, ChildListener listener) {
        //新创建一个目标节点监听
        return new CuratorWatcherImpl(listener);
    }

    @Override
    public List<String> addTargetChildListener(String path, CuratorWatcher listener) {
        try {
            //获取path子节点，并添加监听
            return client.getChildren().usingWatcher(listener).forPath(path);
        } catch (NoNodeException e) {
            return null;
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }

    @Override
    public void removeTargetChildListener(String path, CuratorWatcher listener) {
        //移除监听
        ((CuratorWatcherImpl) listener).unwatch();
    }
   
    /**
     * 该内部类实现了CuratorWatcher接口
     */
    private class CuratorWatcherImpl implements CuratorWatcher {
        //监听
        private volatile ChildListener listener;
        public CuratorWatcherImpl(ChildListener listener) {
            this.listener = listener;
        }
        /**
         * 取消监听
         */
        public void unwatch() {
            this.listener = null;
        }
        @Override
        public void process(WatchedEvent event) throws Exception {
            if (listener != null) {
                //变更节点
                String path = event.getPath() == null ? "" : event.getPath();
                //触发变更事件
                listener.childChanged(path,
                        //如果path为空，curator使用watcher将会抛出异常
                        //如果客户端连接、断开连接服务器，zookeeper将会排队watched event事件
                        StringUtils.isNotEmpty(path)
                                //为path的子节点增加监听(当前CuratorWatcherImpl,只能使用1次)
                                ? client.getChildren().usingWatcher(this).forPath(path)
                                : Collections.<String>emptyList());
            }
        }
    }
}
```
##### ZkclientZookeeperClient实现
```java
public class ZkclientZookeeperClient extends AbstractZookeeperClient<IZkChildListener> {
    /**
     * ZkClient包装类，下面的操作都会交给它来执行
     */
    private final ZkClientWrapper client;

    private volatile KeeperState state = KeeperState.SyncConnected;

    public ZkclientZookeeperClient(URL url) {
        //设置注册中心地址
        super(url);
        //创建zkClient客户端
        client = new ZkClientWrapper(url.getBackupAddress(), 30000);
        //添加监听
        client.addListener(new IZkStateListener() {
            @Override
            public void handleStateChanged(KeeperState state) throws Exception {
		//设置当前状态
		ZkclientZookeeperClient.this.state = state;
                if (state == KeeperState.Disconnected) {
                    //触发断开连接事件
                    stateChanged(StateListener.DISCONNECTED);
                } else if (state == KeeperState.SyncConnected) {
                    //触发连接成功事件
                    stateChanged(StateListener.CONNECTED);
                }
            }
            @Override
            public void handleNewSession() throws Exception {
                //触发重新连接事件
                stateChanged(StateListener.RECONNECTED);
            }
        });
        client.start();
    }
    @Override
    public void createPersistent(String path) {
        try {
            client.createPersistent(path);
        } catch (ZkNodeExistsException e) {
        }
    }
    @Override
    public void createEphemeral(String path) {
        try {
            client.createEphemeral(path);
        } catch (ZkNodeExistsException e) {
        }
    }
    @Override
    public void delete(String path) {
        try {
            client.delete(path);
        } catch (ZkNoNodeException e) {
        }
    }
    @Override
    public List<String> getChildren(String path) {
        try {
            return client.getChildren(path);
        } catch (ZkNoNodeException e) {
            return null;
        }
    }
    @Override
    public boolean checkExists(String path) {
        try {
            return client.exists(path);
        } catch (Throwable t) {
        }
        return false;
    }
    @Override
    public boolean isConnected() {
        return state == KeeperState.SyncConnected;
    }
    @Override
    public void doClose() {
        client.close();
    }
    @Override
    public IZkChildListener createTargetChildListener(String path, final ChildListener listener) {
        return new IZkChildListener() {
            @Override
            public void handleChildChange(String parentPath, List<String> currentChilds) throws Exception {
                //触发节点变更事件
                listener.childChanged(parentPath, currentChilds);
            }
        };
    }
    @Override
    public List<String> addTargetChildListener(String path, final IZkChildListener listener) {
        return client.subscribeChildChanges(path, listener);
    }
    @Override
    public void removeTargetChildListener(String path, IZkChildListener listener) {
        client.unsubscribeChildChanges(path, listener);
    }
}
```
###### ZkClientWrapper包装类
Zkclient包装类可以在连接失效后自动监控连接的状态，使用方式与curator一致
```java
public class ZkClientWrapper {

    /**
     * 获取客户端超时时间
     */
    private long timeout;
    /**
     * ZkClient客户端
     */
    private ZkClient client;
    /**
     * 当前状态
     */
    private volatile KeeperState state;

    /**
     * 可监听的FutureTask
     */
    private ListenableFutureTask<ZkClient> listenableFutureTask;
    /**
     * 客户端是否已启动
     */
    private volatile boolean started = false;

    /**
     * @param serverAddr 注册中心url
     */
    public ZkClientWrapper(final String serverAddr, long timeout) {
        //设置超时
        this.timeout = timeout;

        //创建一个FutureTask，用来创建zkClient客户端
        listenableFutureTask = ListenableFutureTask.create(new Callable<ZkClient>() {
            @Override
            public ZkClient call() throws Exception {
	        //创建ZkClient客户端
                return new ZkClient(serverAddr, Integer.MAX_VALUE);
            }
        });
    }
   
    /**
     * 启动zkclient
     */
    public void start() {
        if (!started) {
            //新建守护线程，创建zkClient客户端
            Thread connectThread = new Thread(listenableFutureTask);
            connectThread.setName("DubboZkclientConnector");
            connectThread.setDaemon(true);
            connectThread.start();
            try {
                //获取新创建的客户端
                client = listenableFutureTask.get(timeout, TimeUnit.MILLISECONDS);
            } catch (Throwable t) {
	        //获取client超时
                logger.error("Timeout! zookeeper server can not be connected in : " + timeout + "ms!", t);
            }
            started = true;
        } else {
            logger.warn("Zkclient has already been started!");
        }
    }
  
    /**
     * 添加监听listener
     */
    public void addListener(final IZkStateListener listener) {
        //使用listenableFutureTask监听client的创建，client创建成功后在订阅该listener
        listenableFutureTask.addListener(new Runnable() {
            @Override
            public void run() {
                try {
                    client = listenableFutureTask.get();
                    //获取到客户端后，订阅listener
                    client.subscribeStateChanges(listener);
                } catch (InterruptedException e) {
                    logger.warn(Thread.currentThread().getName() + " was interrupted unexpectedly, which may cause unpredictable exception!");
                } catch (ExecutionException e) {
                    logger.error("Got an exception when trying to create zkclient instance, can not connect to zookeeper server, please check!", e);
                }
            }
        });
    }

    public boolean isConnected() {
        //是否已连接
        return client != null && state == KeeperState.SyncConnected;
    }

    public void createPersistent(String path) {
        Assert.notNull(client, new IllegalStateException("Zookeeper is not connected yet!"));
        //创建持久化节点
        client.createPersistent(path, true);
    }

    public void createEphemeral(String path) {
        Assert.notNull(client, new IllegalStateException("Zookeeper is not connected yet!"));
        //创建临时节点
        client.createEphemeral(path);
    }

    public void delete(String path) {
        Assert.notNull(client, new IllegalStateException("Zookeeper is not connected yet!"));
        //删除节点
        client.delete(path);
    }

    public List<String> getChildren(String path) {
        Assert.notNull(client, new IllegalStateException("Zookeeper is not connected yet!"));
        //获取path节点的子节点
        return client.getChildren(path);
    }

    public boolean exists(String path) {
        Assert.notNull(client, new IllegalStateException("Zookeeper is not connected yet!"));
        //判断path节点是否存在
        return client.exists(path);
    }

    public void close() {
        Assert.notNull(client, new IllegalStateException("Zookeeper is not connected yet!"));
        client.close();
    }

    public List<String> subscribeChildChanges(String path, final IZkChildListener listener) {
        Assert.notNull(client, new IllegalStateException("Zookeeper is not connected yet!"));
        //订阅path节点变更事件
        return client.subscribeChildChanges(path, listener);
    }

    public void unsubscribeChildChanges(String path, IZkChildListener listener) {
        Assert.notNull(client, new IllegalStateException("Zookeeper is not connected yet!"));
        //取消订阅path节点变更事件
        client.unsubscribeChildChanges(path, listener);
    }
}
```

关于zk注册中心的内容就介绍到这里，下一小节介绍其他注册中心的实现。

