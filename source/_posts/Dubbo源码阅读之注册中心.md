---
title: Dubbo源码阅读之注册中心
date: 2018-08-17 22:52:59
tags:
    - dubbo
---
>注册中心是Dubbo实现服务化管理的核心组件,类似于目录服务的作用,主要用来存储Dubbo发布的服务信息(譬如提供者url串、路由信息等),Dubbo框架支持zookeeper、redis、multicast等注册中心,下面我们就详细看下Dubbo的注册中心是如何实现的。

先来看下注册中心相关的类图
![](img/registry.png)

### Registry接口
```java
public interface Registry extends Node, RegistryService {
}
```
#### Node接口
```java
public interface Node {
    /**
     * 获取url
     * @return url.
     */
    URL getUrl();

    /**
     * 是否可用
     * @return available.
     */
    boolean isAvailable();

    /**
     * 销毁
     */
    void destroy();
}
```

#### RegistryService接口
```java
public interface RegistryService {
    /**
     * 注册数据，例如：提供者服务，消费者服务，路由规则，覆盖规则和其他数据
     * 注册时需要满足以下邀约：
     * 1、当Url设置check = false参数时，注册失败时，异常不会抛出，并且会在后台重试，否则，异常将会抛出
     * 2、当url设置dynamic=false参数时，它需要被永久存储，否则，当注册有异常退出时，它应该被自动删除掉
     * 3、当url设置category=routers参数时，这意味着分类存储，默认的分类是提供者，并且数据可以通过分类部分得到通知
     * 4、当注册中心重启，网络抖动，数据不可以丢失，包括自动从虚线中删除数据
     * 5、允许具有相同URL但是参数不同的URL共存，他们不可以互相覆盖
     *
     * @param url 注册信息，不允许为空
     * 例如：dubbo://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     */
    void register(URL url);

    /**
     * 取消注册
     * 1、如果它是dynamic=false的持久化存储数据，注册信息不可以发现时，会抛出IllegalStateException异常，
     *    否则它是忽略的。
     * 2、根据完整的url匹配进行取消注册
     * @param url 注册信息,不可以为空 
     * 例如： dubbo://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     */
    void unregister(URL url);

    /**
     * 订阅符合条件的注册数据，并在注册数据发生改变时自动推送
     *
     * 1、当URL设置check=false参数时，当注册失败时，异常不会抛出，并在后台重试。
     * 2、当url设置category=routers时，它只会通知指定的分类数据，多个分类用逗号分隔，
     *      并允许使用*号匹配，这表明所有分类数据都已经订阅。
     * 3、允许interface, group, version,classifier作为一个条件查询，
     *      例如：interface=com.alibaba.foo.BarService&version=1.0.0
     * 4、查询条件允许*号匹配，订阅所有接口的所有数据包的所有版本,
     *      例如：interface=*&group=*&version=*&classifier=*
     * 5、当注册中心重启、网络抖动时，有必要自动恢复订阅请求。
     * 6、允许具有相同的url但是参数不同的URL共存,它们不可以互相覆盖
     * 7、订阅的进程必须是阻塞的，当第一条通知完成后返回。
     * @param url 订阅条件，不允许为空
     * 例如: consumer://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     * @param listener 变更事件的监听者，不可以为空
     */
    void subscribe(URL url, NotifyListener listener);

    /**
     * 取消订阅
     *  1、如果没有订阅，直接忽略
     *  2、取消订阅，需要URL全匹配
     * @param url  订阅条件，不可以为空
     * 例如： consumer://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     * @param listener 变更事件的监听者,不允许为空
     */
    void unsubscribe(URL url, NotifyListener listener);

    /**
     * 查询符合条件的注册数据，对应于订阅的push模式，这是pull模式并只返回一个结果
     * @param url 查询条件，不允许为空
     *   e.g. consumer://10.20.153.10/com.alibaba.foo.BarService?version=1.0.0&application=kylin
     * @return 已注册的信息列表, 可能为空, 意义与{NotifyListener#notify(List<URL>)}的参数相同.
     */
    List<URL> lookup(URL url);
}
```

### NotifyListener接口
监听器，监听服务的变更
```java
public interface NotifyListener {
    /**
     * 当收到服务更改的通知时触发该方法
     * 1、始终是在服务接口和数据类型的纬度上通知。也就是说，不会通知属于一个服务的部分相同类型的数据，用户无需比较先前通知的结果
     * 2、订阅时的第一个通知必须是服务所有类型的完整通知
     * 3、在变更时，允许单独通知不同类型的数据，例如：providers, consumers, routers, overrides,它只允许通知其中一种类型，
     *    但此类型的数据必须是完整的，而不是增量的
     * 4、如果数据类型为空，则需要通过url数据的类别参数标识空协议
     * 5、notifications保证通知的顺序(即registry的实现),例如：单线程推送、队列序列化、版本比较
     * @param urls 已注册的信息列表,非空,这意味着，它和RegistryService#lookup(URL)方法的返回值相同.
     */
    void notify(List<URL> urls);
}
```

### AbstractRegistry抽象类
```java
public abstract class AbstractRegistry implements Registry {

    /**
     * URL地址分隔符，用来文件缓存，服务提供者URL分隔
     * URL address separator, used in file cache, service provider URL separation
     */
    private static final char URL_SEPARATOR = ' ';
    /**
     * URL地址正则表达式分隔器，用来解析文件缓存中的服务提供者的URL列表
     * 这里是空格分隔
     * URL address separated regular expression for parsing the service provider URL list in the file cache
     */
    private static final String URL_SPLIT = "\\s+";
    /**
     * 日志输出
     * Log output
     */
    protected final Logger logger = LoggerFactory.getLogger(getClass());
    /**
     * 本地磁盘缓存，其中key“value.registies”用来记录注册中心的列表。
     * 其他的是通知服务提供者的列表
     * Local disk cache, where the special key value.registies records the list of registry centers,
     * and the others are the list of notified service providers
     */
    private final Properties properties = new Properties();
    /**
     * 文件缓存定时写线程
     * File cache timing writing
     */
    private final ExecutorService registryCacheExecutor = Executors.newFixedThreadPool(1, new NamedThreadFactory("DubboSaveRegistryCache", true));
    /**
     * 是否同步保存文件
     * Is it synchronized to save the file
     */
    private final boolean syncSaveFile;
    /**
     * 每次更新缓存文件时，都会自增，作为版本号
     */
    private final AtomicLong lastCacheChanged = new AtomicLong();
    /**
     * 已注册的地址
     * 暴露的服务的URL集合，即export参数指定的URL
     */
    private final Set<URL> registered = new ConcurrentHashSet<URL>();
    /**
     * 已订阅的记录
     * URL变化时，触发NotifyListener
     * 例如：<服务提供者URL，OverrideListener>
     */
    private final ConcurrentMap<URL, Set<NotifyListener>> subscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();
    /**
     * 已通知的url
     * 会将 <订阅url的服务唯一名称,待通知类别Url列表>写入注册中心缓存文件
     * <订阅url,<待通知类别, 待通知类别Url列表>>
     */
    private final ConcurrentMap<URL, Map<String, List<URL>>> notified = new ConcurrentHashMap<URL, Map<String, List<URL>>>();

    /**
     * 注册中心URL
     */
    private URL registryUrl;
    /**
     * 本地磁盘缓存文件(dubbo注册中心缓存)
     * Local disk cache file
     */
    private File file;

    /**
     *
     * @param url multicast://224.5.6.7:1234/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&interface=com.alibaba.dubbo.registry.RegistryService&pid=3000&qos.port=22222&timestamp=1528800181027
     */
    public AbstractRegistry(URL url) {
        //校验url不为空，并设置registryUrl = url
        setUrl(url);
        // Start file save timer
        //是否同步保存文件，默认是异步
        syncSaveFile = url.getParameter(Constants.REGISTRY_FILESAVE_SYNC_KEY, false);
        //文件名，默认值是：C:\Users\Administrator/.dubbo/dubbo-registry-demo-provider-224.5.6.7:1234.cache
        String filename = url.getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + url.getParameter(Constants.APPLICATION_KEY) + "-" + url.getAddress() + ".cache");
        File file = null;
        if (ConfigUtils.isNotEmpty(filename)) {
            //创建文件目录
            file = new File(filename);
            if (!file.exists() && file.getParentFile() != null && !file.getParentFile().exists()) {
                if (!file.getParentFile().mkdirs()) {
                    throw new IllegalArgumentException("Invalid registry store file " + file + ", cause: Failed to create directory " + file.getParentFile() + "!");
                }
            }
        }
        this.file = file;
        //加载注册中心缓存file
        loadProperties();
        //使用所有的url，进行通知
        notify(url.getBackupUrls());
    }


    /**
     * urls 为空的话，会根据url生成一个protocol = empty的URL放入urls并返回（即NotifyListener接口的第4条邀约）
     * @param url 订阅url
     * @param urls 注册中心url列表
     * @return
     */
    protected static List<URL> filterEmpty(URL url, List<URL> urls) {
        if (urls == null || urls.isEmpty()) {
            //urls为空的话，则设置url的protocol = empty
            List<URL> result = new ArrayList<URL>(1);
            //设置protocol = empty
            result.add(url.setProtocol(Constants.EMPTY_PROTOCOL));
            return result;
        }
        return urls;
    }

    @Override
    public URL getUrl() {
        return registryUrl;
    }

    protected void setUrl(URL url) {
        if (url == null) {
            throw new IllegalArgumentException("registry url == null");
        }
	//设置注册中心url
        this.registryUrl = url;
    }

    public Set<URL> getRegistered() {
        return registered;
    }

    public Map<URL, Set<NotifyListener>> getSubscribed() {
        return subscribed;
    }

    public Map<URL, Map<String, List<URL>>> getNotified() {
        return notified;
    }

    public File getCacheFile() {
        return file;
    }

    public Properties getCacheProperties() {
        return properties;
    }

    public AtomicLong getLastCacheChanged() {
        return lastCacheChanged;
    }

    /**
     * 保存注册中心缓存文件
     * @param version 版本
     */
    public void doSaveProperties(long version) {
        //如果version小与当前的版本号，说明在执行该方法时，lastCacheChanged又被更新了
        //因此这里只需要直接返回,等待下一次执行
        if (version < lastCacheChanged.get()) {
            return;
        }
        if (file == null) {
            return;
        }
        // Save
        try {
            //创建一个文件锁
            File lockfile = new File(file.getAbsolutePath() + ".lock");
            if (!lockfile.exists()) {
                lockfile.createNewFile();
            }
            RandomAccessFile raf = new RandomAccessFile(lockfile, "rw");
            try {
                FileChannel channel = raf.getChannel();
                try {
                    //获取排它锁
                    FileLock lock = channel.tryLock();
                    if (lock == null) {
                        //不可以锁定注册中心缓存文件,忽略并稍后重试
                        //可能多个java进程使用该文件，请配置：dubbo.registry.file=xxx.properties
                        throw new IOException("Can not lock the registry cache file " + file.getAbsolutePath() + ", ignore and retry later, maybe multi java process use the file, please config: dubbo.registry.file=xxx.properties");
                    }
                    // Save
                    try {
                        //缓存文件不存在的话，新创建
                        if (!file.exists()) {
                            file.createNewFile();
                        }
                        FileOutputStream outputFile = new FileOutputStream(file);
                        try {
                            //保存properties文件
                            properties.store(outputFile, "Dubbo Registry Cache");
                        } finally {
                            outputFile.close();
                        }
                    } finally {
                        //释放锁
                        lock.release();
                    }
                } finally {
                    channel.close();
                }
            } finally {
                raf.close();
            }
        } catch (Throwable e) {
            if (version < lastCacheChanged.get()) {
                return;
            } else {
                //如果version >= 当前版本的话，则执行异步保存properties文件
                registryCacheExecutor.execute(new SaveProperties(lastCacheChanged.incrementAndGet()));
            }
            logger.warn("Failed to save registry store file, cause: " + e.getMessage(), e);
        }
    }

    /**
     * 加载注册中心缓存文件
     */
    private void loadProperties() {
        if (file != null && file.exists()) {
            InputStream in = null;
            try {
                in = new FileInputStream(file);
                properties.load(in);
                if (logger.isInfoEnabled()) {
                    logger.info("Load registry store file " + file + ", data: " + properties);
                }
            } catch (Throwable e) {
                logger.warn("Failed to load registry store file " + file, e);
            } finally {
                if (in != null) {
                    try {
                        in.close();
                    } catch (IOException e) {
                        logger.warn(e.getMessage(), e);
                    }
                }
            }
        }
    }

    /**
     * 根据订阅url获取已缓存的待通知url列表
     * 即从配置文件中找到属性key等于url.getServiceKey()的属性值
     * @param url
     * @return
     */
    public List<URL> getCacheUrls(URL url) {
        for (Map.Entry<Object, Object> entry : properties.entrySet()) {
            //订阅url的服务唯一名称
            String key = (String) entry.getKey();
            //类别待通知url列表，空格分隔
            String value = (String) entry.getValue();

            if (key != null && key.length() > 0 && key.equals(url.getServiceKey())
                    //key的第一个字节是字母或者是下划线
                    && (Character.isLetter(key.charAt(0)) || key.charAt(0) == '_')
                    && value != null && value.length() > 0) {
                //使用空格分隔value，拿到每一个url并放到List中
                String[] arr = value.trim().split(URL_SPLIT);
                List<URL> urls = new ArrayList<URL>();
                for (String u : arr) {
                    urls.add(URL.valueOf(u));
                }
                return urls;
            }
        }
        return null;
    }


    @Override
    public List<URL> lookup(URL url) {
        List<URL> result = new ArrayList<URL>();
        //根据订阅url获取已通知的url列表
        Map<String, List<URL>> notifiedUrls = getNotified().get(url);
        if (notifiedUrls != null && notifiedUrls.size() > 0) {
            //notifiedUrls不为空
            for (List<URL> urls : notifiedUrls.values()) {
                for (URL u : urls) {
                    //过滤掉protocol = empty的url
                    if (!Constants.EMPTY_PROTOCOL.equals(u.getProtocol())) {
                        result.add(u);
                    }
                }
            }
        } else {
            final AtomicReference<List<URL>> reference = new AtomicReference<List<URL>>();
            NotifyListener listener = new NotifyListener() {
                @Override
                public void notify(List<URL> urls) {
                    reference.set(urls);
                }
            };
            // 订阅逻辑保证第一次notify后再返回
            subscribe(url, listener);
            List<URL> urls = reference.get();
            if (urls != null && !urls.isEmpty()) {
                for (URL u : urls) {
                    if (!Constants.EMPTY_PROTOCOL.equals(u.getProtocol())) {
                        result.add(u);
                    }
                }
            }
        }
        return result;
    }

    @Override
    public void register(URL url) {
        if (url == null) {
            throw new IllegalArgumentException("register url == null");
        }
        if (logger.isInfoEnabled()) {
            logger.info("Register: " + url);
        }
        //保存注册url
        registered.add(url);
    }

    @Override
    public void unregister(URL url) {
        if (url == null) {
            throw new IllegalArgumentException("unregister url == null");
        }
        if (logger.isInfoEnabled()) {
            logger.info("Unregister: " + url);
        }
        //取消注册url
        registered.remove(url);
    }

    @Override
    public void subscribe(URL url, NotifyListener listener) {
        if (url == null) {
            throw new IllegalArgumentException("subscribe url == null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("subscribe listener == null");
        }
        if (logger.isInfoEnabled()) {
            logger.info("Subscribe: " + url);
        }
        //保存订阅url
        Set<NotifyListener> listeners = subscribed.get(url);
        if (listeners == null) {
            subscribed.putIfAbsent(url, new ConcurrentHashSet<NotifyListener>());
            listeners = subscribed.get(url);
        }
        listeners.add(listener);
    }

    @Override
    public void unsubscribe(URL url, NotifyListener listener) {
        if (url == null) {
            throw new IllegalArgumentException("unsubscribe url == null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("unsubscribe listener == null");
        }
        if (logger.isInfoEnabled()) {
            logger.info("Unsubscribe: " + url);
        }
        //取消订阅url的listener
        Set<NotifyListener> listeners = subscribed.get(url);
        if (listeners != null) {
            listeners.remove(listener);
        }
    }

    /**
     * 恢复注册和订阅
     * @throws Exception
     */
    protected void recover() throws Exception {
        //获取已注册的地址
        Set<URL> recoverRegistered = new HashSet<URL>(getRegistered());
        if (!recoverRegistered.isEmpty()) {
            if (logger.isInfoEnabled()) {
                logger.info("Recover register url " + recoverRegistered);
            }
            for (URL url : recoverRegistered) {
                //重新注册url
                register(url);
            }
        }
        //获取已订阅的记录
        Map<URL, Set<NotifyListener>> recoverSubscribed = new HashMap<URL, Set<NotifyListener>>(getSubscribed());
        if (!recoverSubscribed.isEmpty()) {
            if (logger.isInfoEnabled()) {
                logger.info("Recover subscribe url " + recoverSubscribed.keySet());
            }
            for (Map.Entry<URL, Set<NotifyListener>> entry : recoverSubscribed.entrySet()) {
                URL url = entry.getKey();
                for (NotifyListener listener : entry.getValue()) {
                    //重新订阅
                    subscribe(url, listener);
                }
            }
        }
    }

    /**
     * 通知
     * @param urls 注册中心url
     */
    protected void notify(List<URL> urls) {
        if (urls == null || urls.isEmpty()) {
            return;
        }
        //遍历已订阅的记录（例如服务暴露时那里会订阅事件）
        for (Map.Entry<URL, Set<NotifyListener>> entry : getSubscribed().entrySet()) {
            //订阅url
            URL url = entry.getKey();
            //检测是否匹配
            if (!UrlUtils.isMatch(url, urls.get(0))) {
                continue;
            }
            Set<NotifyListener> listeners = entry.getValue();
            if (listeners != null) {
                for (NotifyListener listener : listeners) {
                    try {
                        //通知注册中心
                        notify(url, listener, filterEmpty(url, urls));
                    } catch (Throwable t) {
                        logger.error("Failed to notify registry event, urls: " + urls + ", cause: " + t.getMessage(), t);
                    }
                }
            }
        }
    }

    /**
     * @param url 订阅url
     * provider://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&category=configurators&check=false&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=7444&side=provider&timestamp=1528870218728
     * @param listener
     * @param urls 待通知的urls
     */
    protected void notify(URL url, NotifyListener listener, List<URL> urls) {
        if (url == null) {
            throw new IllegalArgumentException("notify url == null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("notify listener == null");
        }
        if ((urls == null || urls.isEmpty()) && !Constants.ANY_VALUE.equals(url.getServiceInterface())) {
            logger.warn("Ignore empty notify urls for subscribe url " + url);
            return;
        }
        if (logger.isInfoEnabled()) {
            logger.info("Notify urls for subscribe url " + url + ", urls: " + urls);
        }
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
            //触发通知(类别待通知url列表)
            listener.notify(categoryList);
        }
    }

    /**
     * 保存注册中心缓存文件
     * @param url 订阅url
     */
    private void saveProperties(URL url) {
        if (file == null) {
            return;
        }
        try {
            //保存待通知url
            StringBuilder buf = new StringBuilder();
            //根据订阅url获取类别map
            Map<String, List<URL>> categoryNotified = notified.get(url);
            if (categoryNotified != null) {
                //遍历类别待通知url列表
                for (List<URL> us : categoryNotified.values()) {
                    for (URL u : us) {
                        //将待通知url添加到buf中，多个地址使用空格分隔
                        if (buf.length() > 0) {
                            buf.append(URL_SEPARATOR);
                        }
                        buf.append(u.toFullString());
                    }
                }
            }
            //保存配置到properties文件<订阅url的服务唯一名称标识,类别待通知url列表>
            properties.setProperty(url.getServiceKey(), buf.toString());
            //获取版本号
            long version = lastCacheChanged.incrementAndGet();
            if (syncSaveFile) {
                //同步保存缓存文件
                doSaveProperties(version);
            } else {
                //异步保存缓存文件
                registryCacheExecutor.execute(new SaveProperties(version));
            }
        } catch (Throwable t) {
            logger.warn(t.getMessage(), t);
        }
    }

    @Override
    public void destroy() {
        if (logger.isInfoEnabled()) {
            //销毁注册中心
            logger.info("Destroy registry:" + getUrl());
        }
        Set<URL> destroyRegistered = new HashSet<URL>(getRegistered());
        if (!destroyRegistered.isEmpty()) {
            //遍历已注册的url
            for (URL url : new HashSet<URL>(getRegistered())) {
                if (url.getParameter(Constants.DYNAMIC_KEY, true)) {
                    try {
                        //url的dynamic = true，取消注册url
                        unregister(url);
                        if (logger.isInfoEnabled()) {
                            logger.info("Destroy unregister url " + url);
                        }
                    } catch (Throwable t) {
                        logger.warn("Failed to unregister url " + url + " to registry " + getUrl() + " on destroy, cause: " + t.getMessage(), t);
                    }
                }
            }
        }
        Map<URL, Set<NotifyListener>> destroySubscribed = new HashMap<URL, Set<NotifyListener>>(getSubscribed());
        if (!destroySubscribed.isEmpty()) {
            //遍历已订阅的，挨个取消订阅
            for (Map.Entry<URL, Set<NotifyListener>> entry : destroySubscribed.entrySet()) {
                //订阅的url
                URL url = entry.getKey();
                for (NotifyListener listener : entry.getValue()) {
                    try {
                        //取消订阅url
                        unsubscribe(url, listener);
                        if (logger.isInfoEnabled()) {
                            logger.info("Destroy unsubscribe url " + url);
                        }
                    } catch (Throwable t) {
                        logger.warn("Failed to unsubscribe url " + url + " to registry " + getUrl() + " on destroy, cause: " + t.getMessage(), t);
                    }
                }
            }
        }
    }

    @Override
    public String toString() {
        return getUrl().toString();
    }

    /**
     * 异步保存缓存文件的线程
     */
    private class SaveProperties implements Runnable {
        private long version;
        private SaveProperties(long version) {
            this.version = version;
        }
        @Override
        public void run() {
            doSaveProperties(version);
        }
    }
}
```

### FailbackRegistry抽象类
FailbackRegistry抽象类增加了失败重试功能，MulticastRegistry、ZookeeperRegistry等都继承自它.
```java
public abstract class FailbackRegistry extends AbstractRegistry {

    /**
     * 注册中心失败重试线程
     */
    private final ScheduledExecutorService retryExecutor =
            Executors.newScheduledThreadPool(1,
                    new NamedThreadFactory("DubboRegistryFailedRetryTimer", true));

    /**
     * 用于失败重试的定时器，定期检查是有失败的请求，如果有，则无限重试
     */
    private final ScheduledFuture<?> retryFuture;

    /**
     * 注册失败的URL列表
     */
    private final Set<URL> failedRegistered = new ConcurrentHashSet<URL>();

    /**
     * 取消注册失败的URL列表
     */
    private final Set<URL> failedUnregistered = new ConcurrentHashSet<URL>();

    /**
     * 订阅失败的记录
     * <订阅url，监听>
     */
    private final ConcurrentMap<URL, Set<NotifyListener>> failedSubscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();

    /**
     * 取消订阅失败的记录
     * <订阅url，监听>
     */
    private final ConcurrentMap<URL, Set<NotifyListener>> failedUnsubscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();

    /**
     * 通知失败的URL
     */
    private final ConcurrentMap<URL, Map<NotifyListener, List<URL>>> failedNotified = new ConcurrentHashMap<URL, Map<NotifyListener, List<URL>>>();

    public FailbackRegistry(URL url) {
        //设置注册中心url
        super(url);
        //获取url对应的重试间隔时间，默认值是5秒
        int retryPeriod = url.getParameter(Constants.REGISTRY_RETRY_PERIOD_KEY, Constants.DEFAULT_REGISTRY_RETRY_PERIOD);
        this.retryFuture = retryExecutor.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                //检测并连接到注册中心
                try {
		    //执行重试
                    retry();
                } catch (Throwable t) {
                    logger.error("Unexpected error occur at failed retry, cause: " + t.getMessage(), t);
                }
            }
        }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
    }

    public Future<?> getRetryFuture() {
        return retryFuture;
    }

    public Set<URL> getFailedRegistered() {
        return failedRegistered;
    }

    public Set<URL> getFailedUnregistered() {
        return failedUnregistered;
    }

    public Map<URL, Set<NotifyListener>> getFailedSubscribed() {
        return failedSubscribed;
    }

    public Map<URL, Set<NotifyListener>> getFailedUnsubscribed() {
        return failedUnsubscribed;
    }

    public Map<URL, Map<NotifyListener, List<URL>>> getFailedNotified() {
        return failedNotified;
    }

    /**
     * 添加订阅失败的记录
     * @param url 订阅url
     * @param listener
     */
    private void addFailedSubscribed(URL url, NotifyListener listener) {
        Set<NotifyListener> listeners = failedSubscribed.get(url);
        if (listeners == null) {
            failedSubscribed.putIfAbsent(url, new ConcurrentHashSet<NotifyListener>());
            listeners = failedSubscribed.get(url);
        }
        listeners.add(listener);
    }

    /**
     * 移除订阅失败的记录（从三个列表中都移除）
     * @param url 订阅url
     * @param listener
     */
    private void removeFailedSubscribed(URL url, NotifyListener listener) {
        //订阅失败的记录
        Set<NotifyListener> listeners = failedSubscribed.get(url);
        if (listeners != null) {
            listeners.remove(listener);
        }
        //取消订阅失败的记录
        listeners = failedUnsubscribed.get(url);
        if (listeners != null) {
            listeners.remove(listener);
        }
        //通知失败的记录
        Map<NotifyListener, List<URL>> notified = failedNotified.get(url);
        if (notified != null) {
            notified.remove(listener);
        }
    }

    @Override
    public void register(URL url) {
        //保存url到集合缓存中
        super.register(url);
        //从已失败的记录中移除该url
        failedRegistered.remove(url);
        failedUnregistered.remove(url);
        try {
            //向服务端发送注册请求
            doRegister(url);
        } catch (Exception e) {
            Throwable t = e;

            //注册中心url以及服务提供者url中的check = true且url不是消费端
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true)
                    && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
            //判断是否需要跳过故障恢复(SkipFailbackWrapperException异常只是作为标记)
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                //如果启动检测或者跳过故障恢复的话，则直接抛出异常
                if (skipFailback) {
                    t = t.getCause();
                }
                //注册url到注册中心发生失败
                throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
            } else {
                logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }
            //注册失败的话，将url保存到注册失败列表中，定期重试
            failedRegistered.add(url);
        }
    }

    @Override
    public void unregister(URL url) {
        //从缓存集合中移除该url
        super.unregister(url);
        //从失败列表中移除该url
        failedRegistered.remove(url);
        failedUnregistered.remove(url);
        try {
            //向服务端发送取消注册请求
            doUnregister(url);
        } catch (Exception e) {
            Throwable t = e;
            //判断是否启动检测
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true)
                    //非消费者
                    && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
            //是否跳过故障恢复
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if (skipFailback) {
                    t = t.getCause();
                }
                //直接抛出异常
                throw new IllegalStateException("Failed to unregister " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
            } else {
                //取消注册url执行失败，等待重试
                logger.error("Failed to unregister " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }
            //将失败的url保存到失败列表，等待定期重试
            failedUnregistered.add(url);
        }
    }

    @Override
    public void subscribe(URL url, NotifyListener listener) {
        //将订阅保存到集合缓存
        super.subscribe(url, listener);
        //从失败列表中移除该订阅记录
        removeFailedSubscribed(url, listener);
        try {
            //向服务端发送订阅请求
            doSubscribe(url, listener);
        } catch (Exception e) {
            Throwable t = e;
            //订阅失败的话，则从缓存文件中获取该订阅url对应的注册中心url列表(即类别待通知url列表)
            List<URL> urls = getCacheUrls(url);
            if (urls != null && !urls.isEmpty()) {
                //触发通知
                notify(url, listener, urls);
                //订阅失败，将使用缓存列表
                logger.error("Failed to subscribe " + url + ", Using cached list: " + urls + " from cache file: " + getUrl().getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/dubbo-registry-" + url.getHost() + ".cache") + ", cause: " + t.getMessage(), t);
            } else {
                //是否启动检测
                boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                        && url.getParameter(Constants.CHECK_KEY, true);
                //是否跳过故障恢复
                boolean skipFailback = t instanceof SkipFailbackWrapperException;
                if (check || skipFailback) {
                    if (skipFailback) {
                        t = t.getCause();
                    }
                    throw new IllegalStateException("Failed to subscribe " + url + ", cause: " + t.getMessage(), t);
                } else {
                    //订阅失败，等待重试
                    logger.error("Failed to subscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
                }
            }
            //添加到订阅失败列表，请求重试
            addFailedSubscribed(url, listener);
        }
    }

    @Override
    public void unsubscribe(URL url, NotifyListener listener) {
        //将订阅从集合缓存中移除
        super.unsubscribe(url, listener);
        //从失败列表中移除订阅
        removeFailedSubscribed(url, listener);
        try {
            //向服务端发送取消订阅请求
            doUnsubscribe(url, listener);
        } catch (Exception e) {
            Throwable t = e;
            //是否启动检测
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true);
            //是否跳过故障恢复
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if (skipFailback) {
                    t = t.getCause();
                }
                throw new IllegalStateException("Failed to unsubscribe " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
            } else {
                //取消订阅失败，等待重试
                logger.error("Failed to unsubscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }
            //添加到失败列表中，定期重试
            Set<NotifyListener> listeners = failedUnsubscribed.get(url);
            if (listeners == null) {
                failedUnsubscribed.putIfAbsent(url, new ConcurrentHashSet<NotifyListener>());
                listeners = failedUnsubscribed.get(url);
            }
            listeners.add(listener);
        }
    }

    @Override
    protected void notify(URL url, NotifyListener listener, List<URL> urls) {
        if (url == null) {
            throw new IllegalArgumentException("notify url == null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("notify listener == null");
        }
        try {
            //执行通知(调用父类中的notify方法)
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

    protected void doNotify(URL url, NotifyListener listener, List<URL> urls) {
        super.notify(url, listener, urls);
    }

    @Override
    protected void recover() throws Exception {
        //获取已注册列表
        Set<URL> recoverRegistered = new HashSet<URL>(getRegistered());
        if (!recoverRegistered.isEmpty()) {
            if (logger.isInfoEnabled()) {
                logger.info("Recover register url " + recoverRegistered);
            }
            //将已注册的添加到失败列表中，等待恢复
            for (URL url : recoverRegistered) {
                failedRegistered.add(url);
            }
        }
        //获取已订阅的列表
        Map<URL, Set<NotifyListener>> recoverSubscribed = new HashMap<URL, Set<NotifyListener>>(getSubscribed());
        if (!recoverSubscribed.isEmpty()) {
            if (logger.isInfoEnabled()) {
                logger.info("Recover subscribe url " + recoverSubscribed.keySet());
            }
            for (Map.Entry<URL, Set<NotifyListener>> entry : recoverSubscribed.entrySet()) {
                URL url = entry.getKey();
                for (NotifyListener listener : entry.getValue()) {
                    //将已订阅的添加到失败列表中，等待恢复
                    addFailedSubscribed(url, listener);
                }
            }
        }
    }

    /**
     * 重试 之前操作失败的 记录
     */
    protected void retry() {
        if (!failedRegistered.isEmpty()) {
            //处理注册失败的数据(重新注册)
            Set<URL> failed = new HashSet<URL>(failedRegistered);
            if (failed.size() > 0) {
                if (logger.isInfoEnabled()) {
                    logger.info("Retry register " + failed);
                }
                try {
                    for (URL url : failed) {
                        try {
                            //重试注册
                            doRegister(url);
                            //重新注册成功，则将其从失败列表中移除
                            failedRegistered.remove(url);
                        } catch (Throwable t) {
                            // Ignore all the exceptions and wait for the next retry
                            //忽略所有的异常，等待下次重试
                            logger.warn("Failed to retry register " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                        }
                    }
                } catch (Throwable t) {
                    logger.warn("Failed to retry register " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                }
            }
        }
        if (!failedUnregistered.isEmpty()) {
            //处理取消注册失败的数据(重新执行取消注册)
            Set<URL> failed = new HashSet<URL>(failedUnregistered);
            if (!failed.isEmpty()) {
                if (logger.isInfoEnabled()) {
                    logger.info("Retry unregister " + failed);
                }
                try {
                    for (URL url : failed) {
                        try {
                            //重试取消注册
                            doUnregister(url);
                            //重试成功，则从失败列表中移除出去
                            failedUnregistered.remove(url);
                        } catch (Throwable t) { // Ignore all the exceptions and wait for the next retry
                            logger.warn("Failed to retry unregister  " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                        }
                    }
                } catch (Throwable t) { // Ignore all the exceptions and wait for the next retry
                    logger.warn("Failed to retry unregister  " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                }
            }
        }
        if (!failedSubscribed.isEmpty()) {
            //处理订阅失败的数据
            Map<URL, Set<NotifyListener>> failed = new HashMap<URL, Set<NotifyListener>>(failedSubscribed);
            for (Map.Entry<URL, Set<NotifyListener>> entry : new HashMap<URL, Set<NotifyListener>>(failed).entrySet()) {
                if (entry.getValue() == null || entry.getValue().size() == 0) {
                    //将待通知url列表为空的数据移除出去
                    failed.remove(entry.getKey());
                }
            }
            if (failed.size() > 0) {
                if (logger.isInfoEnabled()) {
                    logger.info("Retry subscribe " + failed);
                }
                try {
                    for (Map.Entry<URL, Set<NotifyListener>> entry : failed.entrySet()) {
                        URL url = entry.getKey();
                        Set<NotifyListener> listeners = entry.getValue();
                        for (NotifyListener listener : listeners) {
                            try {
                                //重试订阅
                                doSubscribe(url, listener);
                                //订阅成功，从失败列表中移除出去
                                listeners.remove(listener);
                            } catch (Throwable t) {
                                logger.warn("Failed to retry subscribe " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                            }
                        }
                    }
                } catch (Throwable t) { // Ignore all the exceptions and wait for the next retry
                    logger.warn("Failed to retry subscribe " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                }
            }
        }
        if (!failedUnsubscribed.isEmpty()) {
            //处理取消订阅失败的数据
            Map<URL, Set<NotifyListener>> failed = new HashMap<URL, Set<NotifyListener>>(failedUnsubscribed);
            for (Map.Entry<URL, Set<NotifyListener>> entry : new HashMap<URL, Set<NotifyListener>>(failed).entrySet()) {
                if (entry.getValue() == null || entry.getValue().isEmpty()) {
                    //将待通知url列表为空的数据移除出去
                    failed.remove(entry.getKey());
                }
            }
            if (failed.size() > 0) {
                if (logger.isInfoEnabled()) {
                    logger.info("Retry unsubscribe " + failed);
                }
                try {
                    for (Map.Entry<URL, Set<NotifyListener>> entry : failed.entrySet()) {
                        URL url = entry.getKey();
                        Set<NotifyListener> listeners = entry.getValue();
                        for (NotifyListener listener : listeners) {
                            try {
                                //重试取消订阅
                                doUnsubscribe(url, listener);
                                //重试成功，从失败列表中移除
                                listeners.remove(listener);
                            } catch (Throwable t) { // Ignore all the exceptions and wait for the next retry
                                logger.warn("Failed to retry unsubscribe " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                            }
                        }
                    }
                } catch (Throwable t) { // Ignore all the exceptions and wait for the next retry
                    logger.warn("Failed to retry unsubscribe " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                }
            }
        }
        if (!failedNotified.isEmpty()) {
            //处理通知失败的数据
            Map<URL, Map<NotifyListener, List<URL>>> failed = new HashMap<URL, Map<NotifyListener, List<URL>>>(failedNotified);
            for (Map.Entry<URL, Map<NotifyListener, List<URL>>> entry : new HashMap<URL, Map<NotifyListener, List<URL>>>(failed).entrySet()) {
                if (entry.getValue() == null || entry.getValue().size() == 0) {
                    failed.remove(entry.getKey());
                }
            }
            if (failed.size() > 0) {
                if (logger.isInfoEnabled()) {
                    logger.info("Retry notify " + failed);
                }
                try {
                    for (Map<NotifyListener, List<URL>> values : failed.values()) {
                        for (Map.Entry<NotifyListener, List<URL>> entry : values.entrySet()) {
                            try {
                                NotifyListener listener = entry.getKey();
                                List<URL> urls = entry.getValue();
                                //重试通知
                                listener.notify(urls);
                                //重试成功，从失败列表中移除
                                values.remove(listener);
                            } catch (Throwable t) { // Ignore all the exceptions and wait for the next retry
                                logger.warn("Failed to retry notify " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                            }
                        }
                    }
                } catch (Throwable t) { // Ignore all the exceptions and wait for the next retry
                    logger.warn("Failed to retry notify " + failed + ", waiting for again, cause: " + t.getMessage(), t);
                }
            }
        }
    }

    @Override
    public void destroy() {
        //调用父类的销毁逻辑,取消注册等
        super.destroy();
        try {
            //停止定时重试线程
            retryFuture.cancel(true);
        } catch (Throwable t) {
            logger.warn(t.getMessage(), t);
        }
    }

    // ==== Template method ====

    /**
     * 注册
     * @param url
     */
    protected abstract void doRegister(URL url);

    /**
     * 取消注册
     * @param url
     */
    protected abstract void doUnregister(URL url);

    /**
     * 订阅
     * @param url
     * @param listener
     */
    protected abstract void doSubscribe(URL url, NotifyListener listener);

    /**
     * 取消订阅
     * @param url
     * @param listener
     */
    protected abstract void doUnsubscribe(URL url, NotifyListener listener);
}
```

#### DubboRegistry类
```java
public class DubboRegistry extends FailbackRegistry {

    private final static Logger logger = LoggerFactory.getLogger(DubboRegistry.class);

    /**
     * 重新连接检测周期：3秒
     */
    private static final int RECONNECT_PERIOD_DEFAULT = 3 * 1000;

    /**
     * 重新连接定时线程
     */
    private final ScheduledExecutorService scheduledExecutorService =
            Executors.newScheduledThreadPool(1, new NamedThreadFactory("DubboRegistryReconnectTimer", true));

    /**
     * 重新连接定时器，定期检查连接是否可用，如果不可用，无限重连
     */
    private final ScheduledFuture<?> reconnectFuture;

    /**
     * 客户端获取处理的锁
     * 锁定客户端实例的创建过程，防止重复客户端
     */
    private final ReentrantLock clientLock = new ReentrantLock();

    private final Invoker<RegistryService> registryInvoker;
    
    /**
     * 注册中心
     */
    private final RegistryService registryService;

    public DubboRegistry(Invoker<RegistryService> registryInvoker, RegistryService registryService) {
        //设置注册中心url（registryInvoker.getUrl()）
        super(registryInvoker.getUrl());
        this.registryInvoker = registryInvoker;
        this.registryService = registryService;
        // 启动重连线程
        int reconnectPeriod = registryInvoker.getUrl()
                .getParameter(Constants.REGISTRY_RECONNECT_PERIOD_KEY, RECONNECT_PERIOD_DEFAULT);
        reconnectFuture = scheduledExecutorService.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                //检测并连接到注册中心
                try {
                    connect();
                } catch (Throwable t) { 
                    logger.error("Unexpected error occur at reconnect, cause: " + t.getMessage(), t);
                }
            }
        }, reconnectPeriod, reconnectPeriod, TimeUnit.MILLISECONDS);
    }
    
    /**
     * 连接到注册中心
     */
    protected final void connect() {
        try {
            //检测是否是已连接的
            if (isAvailable()) {
                return;
            }
            if (logger.isInfoEnabled()) {
                logger.info("Reconnect to registry " + getUrl());
            }
            //连接前先上锁
            clientLock.lock();
            try {
                // Double check whether or not it is connected
                //再次检测是否已连接
                if (isAvailable()) {
                    return;
                }
                //执行恢复
                recover();
            } finally {
                //释放锁
                clientLock.unlock();
            }
        } catch (Throwable t) {
            // 如果设置了check = true，则直接抛异常，否则忽略异常，等待下次重试
            if (getUrl().getParameter(Constants.CHECK_KEY, true)) {
                if (t instanceof RuntimeException) {
                    throw (RuntimeException) t;
                }
                throw new RuntimeException(t.getMessage(), t);
            }
            logger.error("Failed to connect to registry " + getUrl().getAddress() + " from provider/consumer " + NetUtils.getLocalHost() + " use dubbo " + Version.getVersion() + ", cause: " + t.getMessage(), t);
        }
    }

    @Override
    public boolean isAvailable() {
        //registryInvoker为空的话说明不可用，直接返回
        if (registryInvoker == null) {
            return false;
        }
        return registryInvoker.isAvailable();
    }

    @Override
    public void destroy() {
        super.destroy();
        try {
            if (!reconnectFuture.isCancelled()) {
                //取消重连定时器
                reconnectFuture.cancel(true);
            }
        } catch (Throwable t) {
            logger.warn("Failed to cancel reconnect timer", t);
        }
        //销毁registryInvoker
        registryInvoker.destroy();
    }

    @Override
    protected void doRegister(URL url) {
        //交由registryService来完成
        registryService.register(url);
    }

    @Override
    protected void doUnregister(URL url) {
        registryService.unregister(url);
    }

    @Override
    protected void doSubscribe(URL url, NotifyListener listener) {
        registryService.subscribe(url, listener);
    }

    @Override
    protected void doUnsubscribe(URL url, NotifyListener listener) {
        registryService.unsubscribe(url, listener);
    }

    @Override
    public List<URL> lookup(URL url) {
        return registryService.lookup(url);
    }
}
```
FailbackRegistry的注册中心实现类，如ZookeeperRegistry类等，将会在单独的小节进行详细讲解，这里就先不介绍了。

### RegistryFactory接口
```java
@SPI("dubbo")
public interface RegistryFactory {
    /**
     * 根据url获取注册中心实例
     *
     * 1、当设置check=false时，链接不会检测，除此之外当断开连接时将会抛异常
     * 2、支持URL上的username:password授权认证
     * 3、支持backup=10.20.153.10配置候选注册中心集群地址
     * 4、支持file=registry.cache本地磁盘文件缓存
     * 5、支持timeout=1000请求超时配置
     * 6、支持session=60000配置session超时或者到期设置
     * @param url 注册中心地址，不允许为空
     * @return 注册中心引用，永不返回空值
     */
    @Adaptive({"protocol"})
    Registry getRegistry(URL url);
}
```
#### AbstractRegistryFactory抽象类
```java
public abstract class AbstractRegistryFactory implements RegistryFactory {

    private static final Logger LOGGER = LoggerFactory.getLogger(AbstractRegistryFactory.class);

    /**
     * 用来锁定获取注册中心实例时的过程
     */
    private static final ReentrantLock LOCK = new ReentrantLock();

    /**
     * 注册中心集合
     * Map<RegistryAddress, Registry>
     * key = multicast://224.5.6.7:1234/com.alibaba.dubbo.registry.RegistryService
     */
    private static final Map<String, Registry> REGISTRIES = new ConcurrentHashMap<String, Registry>();

    /**
     * 获取所有注册中心
     * @return all registries
     */
    public static Collection<Registry> getRegistries() {
        return Collections.unmodifiableCollection(REGISTRIES.values());
    }

    /**
     * 关闭所有创建的注册中心
     */
    public static void destroyAll() {
        //锁定注册中心关闭过程
        LOCK.lock();
        try {
            for (Registry registry : getRegistries()) {
                try {
                    //调用destroy()方法进行销毁
                    registry.destroy();
                } catch (Throwable e) {
                    LOGGER.error(e.getMessage(), e);
                }
            }
            //清理缓存
            REGISTRIES.clear();
        } finally {
            //释放锁
            LOCK.unlock();
        }
    }

    @Override
    public Registry getRegistry(URL url) {
        //设置path属性
        url = url.setPath(RegistryService.class.getName())
                //添加interface = com.alibaba.dubbo.registry.RegistryService
                .addParameter(Constants.INTERFACE_KEY, RegistryService.class.getName())
                //移除export、refer参数
                .removeParameters(Constants.EXPORT_KEY, Constants.REFER_KEY);
        //key = multicast://224.5.6.7:1234/com.alibaba.dubbo.registry.RegistryService
        String key = url.toServiceString();
        // Lock the registry access process to ensure a single instance of the registry
        //加锁,确保该key对应的注册中心为单例
        LOCK.lock();
        try {
            Registry registry = REGISTRIES.get(key);
            if (registry != null) {
                //缓存中存在，直接返回
                return registry;
            }
            //根据url创建注册中心实例，这里由子类来实现
            registry = createRegistry(url);
            if (registry == null) {
	        //创建失败
                throw new IllegalStateException("Can not create registry " + url);
            }
            //放入缓存，并返回
            REGISTRIES.put(key, registry);
            return registry;
        } finally {
            // 释放锁
            LOCK.unlock();
        }
    }

    /**
     * 创建注册中心实例
     * @param url 注册中心url
     * @return
     */
    protected abstract Registry createRegistry(URL url);
}
```
AbstractRegistryFactory的实现类，如ZookeeperRegistryFactory等类，将在相关的小节单独介绍，这里就不多介绍了。

### Directory接口
接下来我们看看那Directory接口相关的内容
```java
public interface Directory<T> extends Node {

    /**
     * 获取服务类型
     */
    Class<T> getInterface();

    /**
     * 获取invokers列表
     */
    List<Invoker<T>> list(Invocation invocation) throws RpcException;
}
```

#### AbstractDirectory抽象类
```java
public abstract class AbstractDirectory<T> implements Directory<T> {
    
    /**
     * 注册中心url
     */
    private final URL url;

    /**
     * Directory是否已销毁
     */
    private volatile boolean destroyed = false;

    /**
     * 消费者url
     */
    private volatile URL consumerUrl;

    /**
     * 路由列表
     */
    private volatile List<Router> routers;

    public AbstractDirectory(URL url, URL consumerUrl, List<Router> routers) {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        this.url = url;
        this.consumerUrl = consumerUrl;
        //设置路由
        setRouters(routers);
    }
	
    /**
     * 从此list方法返回的InvokerList，已经被Routers过滤
     */
    @Override
    public List<Invoker<T>> list(Invocation invocation) throws RpcException {
        if (destroyed) {
            throw new RpcException("Directory already destroyed .url: " + getUrl());
        }
        //根据invocation获取invokers列表(根据方法名查询缓存methodInvokerMap)
        List<Invoker<T>> invokers = doList(invocation);
        List<Router> localRouters = this.routers;
        if (localRouters != null && !localRouters.isEmpty()) {
	    //遍历路由
            for (Router router : localRouters) {
                try {
                    if (router.getUrl() == null || router.getUrl().getParameter(Constants.RUNTIME_KEY, false)) {
			//如果url的runtime配置为true,则每次都会进行route
			//执行路由，进行过滤
                        invokers = router.route(invokers, getConsumerUrl(), invocation);
                    }
                } catch (Throwable t) {
                    logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
                }
            }
        }
        return invokers;
    }

    /**
     * 设置路由
     * 1、添加：收到notify通知的routers、当前url的router参数、new MockInvokersSelector()
     * 2、将routers排序
     * 3、缓存routers
     * @param routers 收到notify通知的routers
     */
    protected void setRouters(List<Router> routers) {
        routers = routers == null ? new ArrayList<Router>() : new ArrayList<Router>(routers);
        //获取路由工厂扩展名称
        String routerkey = url.getParameter(Constants.ROUTER_KEY);
        if (routerkey != null && routerkey.length() > 0) {
            //根据路由工厂扩展名获取扩展实例
            RouterFactory routerFactory = ExtensionLoader.getExtensionLoader(RouterFactory.class).getExtension(routerkey);
            //根据url获取路由实例，并放入routers
            routers.add(routerFactory.getRouter(url));
        }
        //添加支持mock协议的invoker选择器
        routers.add(new MockInvokersSelector());
        //排序
        Collections.sort(routers);
        this.routers = routers;
    }
    /**
     * 根据invocation获取invokers列表
     * @param invocation
     * @return
     * @throws RpcException
     */
    protected abstract List<Invoker<T>> doList(Invocation invocation) throws RpcException;
}
```
#### RegistryDirectory实现类
该实现类，我们在上一小节《Dubbo源码阅读之服务引用》中已经详细介绍过，这里只简单看下剩余的方法
```java
public class RegistryDirectory<T> extends AbstractDirectory<T> implements NotifyListener {

    //缓存 <服务url, Invoker>
    private volatile Map<String, Invoker<T>> urlInvokerMap;

    //缓存 <服务方法名称,List<Invoker>>
    private volatile Map<String, List<Invoker<T>>> methodInvokerMap;

    /**
     * 通过调用方法名从本地缓存中找到invokers
     * @param invocation
     * @return
     */
    @Override
    public List<Invoker<T>> doList(Invocation invocation) {
        if (forbidden) {
            //1、没有服务提供者。 2、服务提供者被禁用
            throw new RpcException(RpcException.FORBIDDEN_EXCEPTION,
                "No provider available from registry " + getUrl().getAddress() + " for service " + getConsumerUrl().getServiceKey() + " on consumer " +  NetUtils.getLocalHost()
                        + " use dubbo version " + Version.getVersion() + ", please check status of providers(disabled, not registered or in blacklist).");
        }
        List<Invoker<T>> invokers = null;
        Map<String, List<Invoker<T>>> localMethodInvokerMap = this.methodInvokerMap;
        if (localMethodInvokerMap != null && localMethodInvokerMap.size() > 0) {
            //获取调用的方法名称
            String methodName = RpcUtils.getMethodName(invocation);
            //获取调用的方法参数
            Object[] args = RpcUtils.getArguments(invocation);
            if (args != null && args.length > 0 && args[0] != null 
			&& (args[0] instanceof String || args[0].getClass().isEnum())) {
                //第一个参数是字符串类型或者枚举类型
                //可以根据第一个参数枚举路由
                invokers = localMethodInvokerMap.get(methodName + "." + args[0]);
            }
            if (invokers == null) {
                //通过方法名称查询
                invokers = localMethodInvokerMap.get(methodName);
            }
            if (invokers == null) {
                //通过*查询
                invokers = localMethodInvokerMap.get(Constants.ANY_VALUE);
            }
            if (invokers == null) {
                //遍历本地缓存，找到最后一个invokers
                Iterator<List<Invoker<T>>> iterator = localMethodInvokerMap.values().iterator();
                if (iterator.hasNext()) {
                    invokers = iterator.next();
                }
            }
        }
        return invokers == null ? new ArrayList<Invoker<T>>(0) : invokers;
    }

    @Override
    public boolean isAvailable() {
        if (isDestroyed()) {
            return false;
        }
        Map<String, Invoker<T>> localUrlInvokerMap = urlInvokerMap;
        if (localUrlInvokerMap != null && localUrlInvokerMap.size() > 0) {
            for (Invoker<T> invoker : new ArrayList<Invoker<T>>(localUrlInvokerMap.values())) {
                if (invoker.isAvailable()) {
                    //本地缓存中的invoker，如果有一个可用，就返回可用
                    return true;
                }
            }
        }
        return false;
    }

    /**
     * 关闭所有的invokers
     */
    private void destroyAllInvokers() {
        Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap;
        if (localUrlInvokerMap != null) {
            for (Invoker<T> invoker : new ArrayList<Invoker<T>>(localUrlInvokerMap.values())) {
                try {
		    //销毁invoker
                    invoker.destroy();
                } catch (Throwable t) {
                    logger.warn("Failed to destroy service " + serviceKey + " to provider " + invoker.getUrl(), t);
                }
            }
            //清空缓存
            localUrlInvokerMap.clear();
        }
        methodInvokerMap = null;
    }
}

````
#### StaticDirectory实现类
静态目录服务,它的所有Invoker通过构造函数传入,在服务消费方引用服务的时候,服务对多注册中心进行引用时,将Invokers集合直接传入StaticDirectory构造器,再由Cluster伪装成一个Invoker
```java
public class ReferenceConfig<T> extends AbstractReferenceConfig {

	private T createProxy(Map<String, String> map) {
		// 省略其他代码.....
		for (URL url : urls) {
		    //记录"远程引用"
		    invokers.add(refprotocol.refer(interfaceClass, url));
		    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
			// use last registry url
			//使用最后注册的url
			registryURL = url;
		    }
		}
		if (registryURL != null) {
		    //有注册中心协议的URL
		    //使用AvailableCluster
		    URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
		    invoker = cluster.join(new StaticDirectory(u, invokers));
		} else { 
		    //不是注册中心的url
		    invoker = cluster.join(new StaticDirectory(invokers));
		}
	}
}
```
来看看实现
```java
public class StaticDirectory<T> extends AbstractDirectory<T> {

    private final List<Invoker<T>> invokers;

    public StaticDirectory(URL url, List<Invoker<T>> invokers, List<Router> routers) {
        
	super(url == null && invokers != null && !invokers.isEmpty() ? invokers.get(0).getUrl() : url, routers);
        
	if (invokers == null || invokers.isEmpty()) {
            throw new IllegalArgumentException("invokers == null");
        }
        
	this.invokers = invokers;
    }

    @Override
    public Class<T> getInterface() {
        return invokers.get(0).getInterface();
    }

    @Override
    public boolean isAvailable() {
        if (isDestroyed()) {
            return false;
        }
        for (Invoker<T> invoker : invokers) {
            //是否可用
            if (invoker.isAvailable()) {
                return true;
            }
        }
        return false;
    }

    @Override
    public void destroy() {
        if (isDestroyed()) {
	    //已经销毁，直接返回
            return;
        }
        super.destroy();
        for (Invoker<T> invoker : invokers) {
            //销毁
            invoker.destroy();
        }
        invokers.clear();
    }

    @Override
    protected List<Invoker<T>> doList(Invocation invocation) throws RpcException {
        //返回构造方法传入的invokers集合
        return invokers;
    }

}
```
这一小节就先介绍到这里，后面我们详细介绍各个注册中心实现。

