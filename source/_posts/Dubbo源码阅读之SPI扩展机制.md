---
title: Dubbo源码阅读之SPI扩展机制
date: 2018-07-31 09:10:44
tags:
    - dubbo
---
>本文主要分析Dubbo的SPI扩展机制。

* [Java的SPI机制](https://docs.oracle.com/javase/1.5.0/docs/guide/jar/jar.html#Service%20Provider)
* Dubbo的SPI机制

### Java的SPI机制
SPI全称为(Service Provider Interface)，是JDK内置的一种服务提供发现机制。通过SPI服务加载机制进行服务的注册和发现，可以实现基于接口的编程，避免在Java代码中写死服务的提供者，实现多个模块的解耦。
根据Java的SPI规范，我们可以定义一个服务接口，具体的实现由对应的实现者（Service Provider服务提供者）提供，然后在使用的时候只要根据SPI的规范去获取对应的服务提供者的服务实现即可。我们看一个简单的例子：
```java
package java.spi;
public interface Developer {
    public String getPrograme();
}

package java.spi;
public class JavaDeveloper implements Developer {
    @Override
    public String getPrograme() {
        return "Java";
    }
}

package java.spi;
public class PerlDeveloper implements Developer {
    @Override
    public String getPrograme() {
        return "Perl";
    }
}
```
然后在META-INF\services目录下创建名为java.spi.Developer的文件，文件内容是接口实现类的全限定名：
```java
java.spi.JavaDeveloper
java.spi.PerlDeveloper
```
将文件导出为jar包，新建一个项目，在项目中导入该jar,然后新建一个测试类：
```java
import java.util.ServiceLoader;
import cn.edu.knowledge.spi.Developer;
public class Test {
    public static void main(String[] arg) {
         ServiceLoader<Developer> serviceLoader = ServiceLoader.load(Developer.class);
	 for (Developer developer : Developer) {
	        //将会输出: I use Java; I use Perl;
		System.out.println("I use "+developer.getPrograme());
	 }
    }
}
```

### Dubbo的SPI机制
Dubbo的扩展点加载是对JDK内置的SPI机制的一种加强。继续使用上面的例子介绍，
META-INF\services\java.spi.Developer文件中的内容将会变为key=value形式：
```java
java=java.spi.JavaDeveloper
perl=java.spi.PerlDeveloper
```
其中，key为扩展名称，value为扩展实例。扩展配置文件修改为这样子是因为，如果扩展实现中的方法或者静态字段引用了第三方库，而第三方库不存在时，那么这个类将会初始化失败，在这种情况下，如果使用以前的格式，Dubbo不可以弄清楚扩展的id，因此不可以映射这个扩展的异常信息。
例如：
无法加载扩展("mina"),当用户配置使用mina，Dubbo将会抱怨无法加载扩展，而不是报告提取哪一个扩展实现失败及原因。

与Java标准SPI相比，Dubbo SPI又增加了如下功能：
* Dubbo可以通过getExtension（String key）只加载某一个想要的扩展，Java的SPI机制则需要加载全部的实现类。
* 对于扩展实现IOC依赖注入功能，如果扩展实现A含有setB()方法，而接口B有B1和B2两个具体的实现，此时Dubbo并不会具体注入B1或者B2，而是会注入一个自动生成的接口B实现：B$Adpative，该实现者能够根据参数的不同，自动引用B1或者B2来完成相应的功能。
* 对扩展采用装饰器模式进行功能增强。

### Dubbo SPI相关接口
#### SPI接口
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface SPI {
    //默认扩展名称
    String value() default "";
}
```
当接口上表明了@SPI注解时，Dubbo将会依次从如下目录中读取相应的扩展点：
META-INF/dubbo/internal/
META-INF/dubbo/
META-INF/services/

#### Adaptive接口
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {
    String[] value() default {};
}
```
ExtensionLoader注入依赖扩展实例时，该接口为其提供了有用的信息。
value方法决定要注入哪个目标扩展。目标扩展由URL中传递的参数决定，value方法提供了url上的参数名称。如果在URL中没有找到指定的参数名称，则将使用默认的扩展（在其接口上的@SPI注解中进行指定）。
例如,给定字符串：value={"key1", "key2"}，如果在URL中发现参数key1，则使用参数key1的值作为扩展名称，如果在URL中没有发现参数key1或者参数key1的值为空，则尝试使用参数key2的值作为扩展名称，如果key2也不存在，则使用默认的扩展，否则抛异常。
如果默认的扩展名称在@SPI注解中没有提供，则使用规则生成一个name，这个name将用作URL中的搜索参数。规则为：将接口类名从大写字符开始分成几个部分, 并将各部分用点 "." 分开。
例如：com.alibaba.dubbo.xxx.YyyInvokerWrapper，则生成的name为yyy.invoker.wrapper，将会使用该name从url中进行搜索。

#### Activate接口
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Activate {
    /**
     * 当group数组中有一个group匹配时，就激活当前扩展。
     * 将使用传递给ExtensionLoader#getActivateExtension(URL, String, String)方法的group(第三个参数)来和该注解的group进行匹配
     */
    String[] group() default {};

    /**
     * 当指定的key出现在URL的参数中时，则激活当前扩展 
     * 例如：给定@Activate("cache, validation")时，只有url中出现cache或者validation参数时，才会激活当前扩展
     */
    String[] value() default {};
    
    //相对顺序 扩展列表中的哪些扩展将要放到当前扩展前面
    String[] before() default {};

    //相对顺序 扩展列表中的哪些扩展将要放到当前扩展后面
    String[] after() default {};
    
    //绝对顺序
    int order() default 0;
}
```
用于激活扩展。此注解对于使用给定条件自动激活某些扩展是非常有用的，比如：如果有多个Filter实现，@Activate可以用来加载某些Filter。
SPI提供者可以调用ExtensionLoader#getActivateExtension(URL, String, String)方法找到所有符合条件的activated扩展。

#### ExtensionFactory接口
```
@SPI
public interface ExtensionFactory {
    /**
     * 获取扩展实现类实例
     * Get extension.
     * @param type object type. 扩展接口类型
     * @param name object name. 扩展名称
     * @return object instance. 返回扩展实例
     */
    <T> T getExtension(Class<T> type, String name);
}
```
该接口根据扩展类型和扩展名称获取扩展实例。可以看到该接口声明上也加上了@SPI注解，说明存在多种实现，Dubbo提供了三种实现，分别为AdaptiveExtensionFactory、SpiExtensionFactory、SpringExtensionFactory，后面会详细介绍具体实现。


### ExtensionLoader类
ExtensionLoader类用来处理Dubbo扩展,里面定义了大量的实用方法，该类支持以下主要功能：

* 自动注入依赖扩展
* 在wrapper中，自动包装扩展
* 默认扩展是一个adaptive实例

ExtensionLoader类代码量比较多，我们先来看下ExtensionLoader类的getExtensionLoader方法，通过该方法可以得到指定扩展类型接口的扩展加载器，
例如我们想要得到ProxyFactory类型的ExtensionLoader，可以这样做：
```java
ExtensionLoader<ProxyFactory> loader = ExtensionLoader.getExtensionLoader(ProxyFactory.class);
```
#### getExtensionLoader方法
我们来看getExtensionLoader方法:
```java
/**
 * 获取扩展类型接口的扩展加载器
 * 1、校验扩展类型
 * 2、从缓存中获取扩展类型对应的扩展加载器
 * 3、缓存中不存在，则新建一个并放入缓存
 */
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        //扩展类型不可以为空
        if (type == null) {
            throw new IllegalArgumentException("Extension type == null");
        }
	//扩展类型必须为接口
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
        }
	//检测扩展类型接口是否是一个扩展点，判断依据是：接口必须有@SPI注解
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type(" + type +
                    ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
        }
        //首先从缓存中获取该扩展类型的ExtensionLoader
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            //缓存中不存在，则为扩展类型type新建一个ExtensionLoader，并放入缓存
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
	//返回扩展类型的ExtensionLoader
        return loader;
}

/**
 * 私有构造方法
 */
private ExtensionLoader(Class<?> type) {
        //保存扩展类型接口
        this.type = type;
        //ExtensionFactory接口的扩展实现类不需要进行注入依赖，因此这里将objectFactory设置成null
	//其他扩展接口可能依赖其他扩展接口，因此需要进行依赖注入，通过objectFactory可以获取到某个扩展类型的扩展实例
        objectFactory = (type == ExtensionFactory.class ? null :
                    ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension()
        );
}
```
其中type和objectFactory是ExtensionLoader类中定义的实例变量,EXTENSION_LOADERS缓存是静态常量：
```java
/**
 * 扩展接口的类型
 */
private final Class<?> type;

/**
 * 扩展工厂，通过扩展工厂可以获取到某个扩展类型的扩展实例
 */
private final ExtensionFactory objectFactory;

/**
 * 每个扩展类型接口(即带有@SPI注解的接口)，都有一个相对应的ExtensionLoader实例
 * <扩展类型接口，ExtensionLoader实例>
 */
private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS =
		new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();
```
#### getExtension方法
有了ExtensionLoader实例后，就可以通过该实例的getExtension(String name)方法加载扩展类型的指定实例：
```java
/**
 * 通过扩展名称name获取扩展实例
 */
public T getExtension(String name) {
	//扩展名称不可以为空
        if (name == null || name.length() == 0) {
            throw new IllegalArgumentException("Extension name == null");
        }
        if ("true".equals(name)) {
            //name = true 返回默认扩展实例（后面会分析该方法）
            return getDefaultExtension();
        }
	//根据扩展名称从cachedInstances实例变量中获取扩展实例
	//扩展实例被放入到Holder对象中，Holder类是一个持有值的助手类，提供了get/set方法
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            //缓存中不存在的话，则为扩展名称name新建一个Holder实例。
            cachedInstances.putIfAbsent(name, new Holder<Object>());
            holder = cachedInstances.get(name);
        }
	//从holder对象中获取到扩展名称name对应的扩展实例
        Object instance = holder.get();
        if (instance == null) {
	    //扩展实例为空，则为扩展名称name新建一个扩展实例并放入Holder对象中。
	    //这里可能会存在多个线程同时访问，因此需要同步创建扩展实例
            synchronized (holder) {
	        //再次判断holder对象中的扩展实例是否为空
                instance = holder.get();
                if (instance == null) {
                    //根据扩展名称name创建扩展实例（后面会分析该方法）
                    instance = createExtension(name);
		    //放入缓存
                    holder.set(instance);
                }
            }
        }
	//返回扩展实例
        return (T) instance;
}
```
在看getDefaultExtension方法和createExtension方法之前，我们先看下上面getExtension方法中出现的变量定义：
```java
/**
 * 缓存 <扩展名称，扩展实例>
 */
private final ConcurrentMap<String, Holder<Object>> cachedInstances =
    new ConcurrentHashMap<String, Holder<Object>>();

/**
 * 持有一个值的助手类
 */
public class Holder<T> {

    private volatile T value;

    public void set(T value) {
        this.value = value;
    }
    public T get() {
        return value;
    }
}
```
#### getDefaultExtension方法
接下来我们来看getDefaultExtension方法，该方法用来获取默认扩展实例。在该方法内部又调用了多个方法，我们一步步来分析，分析完整个流程后，我们在看createExtension方法.
```java
/**
 * 返回默认扩展实例，如果没有配置默认扩展名称(@SPI注解上配置的)，则返回null
 */
public T getDefaultExtension() {
        //获取扩展接口type对应的扩展实现类集合(后面会分析该方法)
	getExtensionClasses();
	//判断cachedDefaultName变量是否为空
	//cachedDefaultName变量是在getExtensionClasses()方法中进行赋值的，我们稍后去看
	if (null == cachedDefaultName || cachedDefaultName.length() == 0 || "true".equals(cachedDefaultName)) {
		return null;
	}
	//获取默认扩展名称对应的扩展实例
	return getExtension(cachedDefaultName);
}
```
#### getExtensionClasses方法(加载扩展实现类)
```java
/**
 * 获取扩展接口type对应的扩展实现类集合，先从缓存获取，缓存没有的话，再去重新加载
 * @return
 */
private Map<String, Class<?>> getExtensionClasses() {
	//先从cachedClasses缓存中获取
	Map<String, Class<?>> classes = cachedClasses.get();
	if (classes == null) {
		//缓存中为空，同步加载
		synchronized (cachedClasses) {
			//再次判断
			classes = cachedClasses.get();
			if (classes == null) {
				//重新加载扩展实现类（后面会分析该方法）
				classes = loadExtensionClasses();
				//放入缓存
				cachedClasses.set(classes);
			}
		}
	}
	return classes;
}

/**
 * 默认扩展名称,@SPI注解上配置的值
 */
private String cachedDefaultName;

/**
 * 用来缓存扩展接口type对应的所有扩展实现类
 * <扩展名称，扩展实现类>
 */
private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<Map<String, Class<?>>>();


/**
 * 加载扩展接口type对应的所有扩展实现类
 * 在getExtensionClasses中会进行同步
 * synchronized in getExtensionClasses
 * @return
 */
private Map<String, Class<?>> loadExtensionClasses() {
	//获取扩展接口type上的@SPI注解
	final SPI defaultAnnotation = type.getAnnotation(SPI.class);
	if (defaultAnnotation != null) {
		//存在@SPI注解，则获取注解值(即默认扩展名称)
		String value = defaultAnnotation.value();
		if ((value = value.trim()).length() > 0) {
			//使用逗号分隔扩展名称
			String[] names = NAME_SEPARATOR.split(value);
			if (names.length > 1) {
			    //存在多个默认扩展名称，则抛异常
			    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
				    + ": " + Arrays.toString(names));
			}
			if (names.length == 1) {
			    //保存默认扩展名称到cachedDefaultName
			    cachedDefaultName = names[0];
			}
		}
	}
	//开始加载扩展类，依次从3个目录进行加载,保存到extensionClasses中
	Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
	//目录看下面的常量定义(后面会分析该方法)
	loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
	loadDirectory(extensionClasses, DUBBO_DIRECTORY);
	loadDirectory(extensionClasses, SERVICES_DIRECTORY);
	return extensionClasses;
}

/**
 * 扩展所在目录
 */
private static final String SERVICES_DIRECTORY = "META-INF/services/";

private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";

private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/";

/**
 * 获取ExtensionLoader的类加载器
 * @return
 */
private static ClassLoader findClassLoader() {
	return ExtensionLoader.class.getClassLoader();
}

/**
 * 扫描相应目录，加载扩展并保存到extensionClasses集合
 * @param extensionClasses 保存扩展类的Map集合
 * @param dir  扫描目录
 */
private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir) {
	//待加载的文件名，如：fileName = META-INF/dubbo/internal/com.alibaba.dubbo.rpc.ProxyFactory
	String fileName = dir + type.getName();
	try {
	    Enumeration<java.net.URL> urls;
	    //获取ExtensionLoader类加载器(后面会分析该方法)
	    ClassLoader classLoader = findClassLoader();
	    if (classLoader != null) {
		//当前类加载器不为空，加载该文件
		urls = classLoader.getResources(fileName);
	    } else {
		//当前类加载器为空，则从用来加载类的搜索路径中查找改文件
		urls = ClassLoader.getSystemResources(fileName);
	    }
	    if (urls != null) {
		while (urls.hasMoreElements()) {
		    //获取到资源定位符
		    java.net.URL resourceURL = urls.nextElement();
		    //加载资源,即读取并解析配置文件，然后加载类(后面会分析该方法)
		    loadResource(extensionClasses, classLoader, resourceURL);
		}
	    }
	} catch (Throwable t) {
	    logger.error("Exception when load extension class(interface: " +
		    type + ", description file: " + fileName + ").", t);
	}
}

/**
 * 解析配置文件，然后加载扩展类
 * @param extensionClasses 保存扩展类的集合
 * @param classLoader  类加载器
 * @param resourceURL  资源定位符
 */
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, java.net.URL resourceURL) {
	try {
	    BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), "utf-8"));
	    try {
		String line;
		//读取配置文件每一行
		while ((line = reader.readLine()) != null) {
		    //判断当前行是否包含#号，#号后面的内容都是注释，需要去掉
		    final int ci = line.indexOf('#');
		    if (ci >= 0) {
			//截取#号之前的内容，并重新设置line
			line = line.substring(0, ci);
		    }
		    line = line.trim();
		    if (line.length() > 0) {
			try {
			    String name = null;
			    //当前行是否包含=号
			    int i = line.indexOf('=');
			    if (i > 0) {
				//当前行存在=号，则使用=号分隔line，获取到key-value(line)
				//name就是key，即定义的扩展名称
				//line就是value，即定义的扩展实现类全限定名
				name = line.substring(0, i).trim();
				line = line.substring(i + 1).trim();
			    }
			    //如果line不等于空，则需要加载该扩展实现类
			    if (line.length() > 0) {
				//加载扩展实现类(后面会分析该方法)
				loadClass(
					extensionClasses,
					resourceURL,
					//加载扩展实现类
					Class.forName(line, true, classLoader),
					name
				);
			    }
			} catch (Throwable t) {
			    IllegalStateException e = new IllegalStateException("Failed to load extension class(interface: " + type + ", class line: " + line + ") in " + resourceURL + ", cause: " + t.getMessage(), t);
			    //解析行出错，记录错误行到exceptions实例变量中
			    exceptions.put(line, e);
			}
		    }
		}
	    } finally {
		reader.close();
	    }
	} catch (Throwable t) {
	    logger.error("Exception when load extension class(interface: " +
		    type + ", class file: " + resourceURL + ") in " + resourceURL, t);
	}
}

/**
 * 保存解析配置文件发生的异常信息
 * <当前key-value行，异常信息>
 */
private Map<String, IllegalStateException> exceptions = new ConcurrentHashMap<String, IllegalStateException>();

/**
 * 当前扩展类型接口type对应的扩展自适应类
 * 即存在@Adaptive注解的扩展实现类
 * 每个扩展类型接口type只能有一个实现类有@Adaptive注解，如果多个扩展实现类都有@Adaptive，则会抛异常
 */
private volatile Class<?> cachedAdaptiveClass = null;

/**
 * 如果扩展实现类是一个包装类，
 * 则会将该该实现类添加到cachedWrapperClasses集合中
 */
private Set<Class<?>> cachedWrapperClasses;

/**
 * 扩展名称正则表达式，将会使用该正则表达式切割扩展名称name，即使用逗号分隔
 */
private static final Pattern NAME_SEPARATOR = Pattern.compile("\\s*[,]+\\s*");

/**
 * 缓存<扩展名称，Activate注解>
 * 如果是逗号分隔的多个name，则取第1个扩展名称name
 * Activate为当前扩展实现类上的@Activate注解
 */
private final Map<String, Activate> cachedActivates = new ConcurrentHashMap<String, Activate>();

/**
 * 缓存<扩展实现类，扩展名称>
 */
private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<Class<?>, String>();


/**
 * 加载扩展实现类
 * @param extensionClasses 保存扩展实现类的map
 * @param resourceURL 资源文件定位符
 * @param clazz 扩展实现类，即实现了type接口的类
 * @param name  扩展名称，即配置文件中的key
 * @throws NoSuchMethodException
 */
private void loadClass(Map<String, Class<?>> extensionClasses,
		   java.net.URL resourceURL,
		   Class<?> clazz,
		   String name) throws NoSuchMethodException {
	//检测实现类clazz是否是type的子类型，即clazz是否实现了接口type
	if (!type.isAssignableFrom(clazz)) {
	    throw new IllegalStateException("Error when load extension class(interface: " +
		    type + ", class line: " + clazz.getName() + "), class "
		    + clazz.getName() + "is not subtype of interface.");
	}
	//检测实现类clazz是否有@Adaptive注解
	if (clazz.isAnnotationPresent(Adaptive.class)) {
	    if (cachedAdaptiveClass == null) {
	        //cachedAdaptiveClass变量为空，则将cachedAdaptiveClass赋值为clazz
		cachedAdaptiveClass = clazz;
	    } else if (!cachedAdaptiveClass.equals(clazz)) {
	        //cachedAdaptiveClass变量不为空，则判断cachedAdaptiveClass是否和当前实现类class是否是同一个
		//如果不是同一个，则抛异常：每个扩展类型type不可以存在多个自适应扩展类
		throw new IllegalStateException("More than 1 adaptive class found: "
			+ cachedAdaptiveClass.getClass().getName()
			+ ", " + clazz.getClass().getName());
	    }
	} else if (isWrapperClass(clazz)) {
	    //实现类clazz是包装类(后面会分析该方法)
	    Set<Class<?>> wrappers = cachedWrapperClasses;
	    if (wrappers == null) {
		cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
		wrappers = cachedWrapperClasses;
	    }
	    //将该扩展实现类添加到包装类集合中
	    wrappers.add(clazz);
	} else {
	    //检测该实现类是否有默认构造方法
	    clazz.getConstructor();
	    if (name == null || name.length() == 0) {
		//扩展名称name为空时，则根据规则重新设置name(后面会分析该方法)
		name = findAnnotationName(clazz);
		if (name == null || name.length() == 0) {
		    //name仍然为空
		    if (clazz.getSimpleName().length() > type.getSimpleName().length()
			    && clazz.getSimpleName().endsWith(type.getSimpleName())) {
			//如果当前扩展实现类的类名以扩展接口type的名称结尾，则截取接口类名称之前的部分作为name，并转换成小写
			//例如：当clazz = JavassistProxyFactory，type = ProxyFactory时，则name = javassist
			name = clazz.getSimpleName().substring(0, clazz.getSimpleName().length() - type.getSimpleName().length()).toLowerCase();
		    } else {
			//扩展实现类class没有对应的扩展名称，抛异常
			throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
		    }
		}
	    }
	    //使用逗号分隔扩展名称name
	    String[] names = NAME_SEPARATOR.split(name);
	    if (names != null && names.length > 0) {
	        //获取扩展实现类上的@Activate注解
		Activate activate = clazz.getAnnotation(Activate.class);
		if (activate != null) {
		    //存在@Activate注解
		    //则将第1个扩展名称和@Activate注解缓存起来
		    cachedActivates.put(names[0], activate);
		}
		//遍历扩展名称
		for (String n : names) {
		    if (!cachedNames.containsKey(clazz)) {
			//如果<扩展实现类,name>缓存中不包含该扩展实现类,则保存到缓存
			cachedNames.put(clazz, n);
		    }
		    //通过扩展名称n从扩展实现类集合中获取扩展实现类c
		    Class<?> c = extensionClasses.get(n);
		    if (c == null) {
			//扩展实现类c不存在，则保存<扩展名称n，扩展实现类clazz>到缓存
			extensionClasses.put(n, clazz);
		    } else if (c != clazz) {
			//一个扩展名称对应多个扩展实现类，则抛异常
			throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
		    }
		}
	    }
	}
}

/**
 * 检测扩展实例类clazz是否是包装类。
 * 判断依据是：扩展实现类clazz中有一个接收扩展接口(type接口)作为参数的构造方法
 * @param clazz 扩展实现类
 * @return
 */
private boolean isWrapperClass(Class<?> clazz) {
	try {
	    clazz.getConstructor(type);
	    return true;
	} catch (NoSuchMethodException e) {
	    return false;
	}
}
```
加载扩展类型接口type对应的扩展实现类流程到此就全部结束了(即上文中介绍的getExtensionClasses()方法流程)。

#### createExtension方法(创建扩展实现类实例)
我们回到上文中遗留的createExtension(name)方法，看看是如何根据扩展名称name创建扩展实现类实例的。
```java
/**
 * 根据扩展名称name创建扩展实现类实例
 * @param name 扩展名称
 * @return 扩展实现类实例
 */
@SuppressWarnings("unchecked")
private T createExtension(String name) {
	//根据扩展名称name获取扩展实现类clazz
	//getExtensionClasses()方法我们上文已经介绍过了，拿到扩展类型接口type对应的所有扩展实现类
	Class<?> clazz = getExtensionClasses().get(name);
	if (clazz == null) {
	    //根据扩展名称没有获取到扩展实现类，则抛出异常
	    throw findException(name);
	}
	try {
	    //先从缓存中获取扩展实现类的实例
	    T instance = (T) EXTENSION_INSTANCES.get(clazz);
	    if (instance == null) {
	        //如果没有获取到实例，则新建一个实现类clazz的实例并放入缓存。
		EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
		instance = (T) EXTENSION_INSTANCES.get(clazz);
	    }
	    //处理实现类实例的依赖注入(后面会分析该方法)
	    injectExtension(instance);
	    //处理包装类
	    Set<Class<?>> wrapperClasses = cachedWrapperClasses;
	    if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
	        //当前扩展接口type存在包装类扩展实现类
		//遍历包装类列表
		for (Class<?> wrapperClass : wrapperClasses) {
		    //通过参数为type接口的构造方法创建包装类实例(将当前实例instance传递进去)，然后处理包装类的依赖注入
		    //这里可能会有多个包装类，依次进行包装
		    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
		}
	    }
	    //返回扩展实例
	    return instance;
	} catch (Throwable t) {
	    throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
		    type + ")  could not be instantiated: " + t.getMessage(), t);
	}
}

/**
 * 处理扩展实现类实例instance的依赖项，进行依赖注入
 * 通过set方法注入依赖项
 * @param instance 扩展实现类的实例
 * @return
 */
private T injectExtension(T instance) {
	try {
	    //判断加载器是否为空
	    if (objectFactory != null) {
	        //遍历实现类中的方法
		for (Method method : instance.getClass().getMethods()) {
		    //找到参数长度为1、public修饰符的set方法
		    if (method.getName().startsWith("set")
			    && method.getParameterTypes().length == 1
			    && Modifier.isPublic(method.getModifiers())) {
			//获取set方法的参数类型
			Class<?> pt = method.getParameterTypes()[0];
			try {
			    //截取set方法的方法名，去掉"set"字符，并将余下的字符转换成小写字母，作为属性
			    String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
			    //从扩展工厂中找到该依赖项实例(pt为依赖项的类型，property为依赖项的名称)
			    Object object = objectFactory.getExtension(pt, property);
			    if (object != null) {
				//依赖项实例存在的话，通过该set方法则将该依赖项实例注入到扩展实现类实例instance中
				method.invoke(instance, object);
			    }
			} catch (Exception e) {
			    logger.error("fail to inject via method " + method.getName()
				    + " of interface " + type.getName() + ": " + e.getMessage(), e);
			}
		    }
		}
	    }
	} catch (Exception e) {
	    logger.error(e.getMessage(), e);
	}
	//返回扩展实现类的实例
	return instance;
}
```
以上便是获取扩展实例的过程，最终会将扩展实例放入到缓存变量cachedInstances中,即<扩展名称，扩展实例>。

#### getActivateExtension方法
接下来我们在看下ExtensionLoader类中的getActivateExtension方法，通过该方法可以得到已启用的扩展列表
```java
/**
 * key为url中标识扩展名称的参数
 */
public List<T> getActivateExtension(URL url, String key) {
	//调用了下面的重载方法
	return getActivateExtension(url, key, null);
}

public List<T> getActivateExtension(URL url, String key, String group) {
	//从url中获取参数key对应的值(即获取扩展名称)
	String value = url.getParameter(key);
	//调用了下面的重载方法
	return getActivateExtension(
		url, 
		value == null || value.length() == 0 ? null : Constants.COMMA_SPLIT_PATTERN.split(value), 
		group
	);
}

/**
 * 获取已启用的扩展列表
 * url 
 * values为扩展名称列表
 * group为配置的组
 */
public List<T> getActivateExtension(URL url, String[] values, String group) {
	List<T> exts = new ArrayList<T>();
	//扩展名称数组
	List<String> names = values == null ? new ArrayList<String>(0) : Arrays.asList(values);
	//names是否包含-default
	if (!names.contains(Constants.REMOVE_VALUE_PREFIX + Constants.DEFAULT_KEY)) {
	    //先获取下扩展接口type的扩展实现类集合(上文介绍过该方法)
	    //执行getExtensionClasses方法时，会将存在@Activate注解的实现类缓存起来：<扩展名称，Activate注解>
	    getExtensionClasses();
	    //遍历cachedActivates集合
	    for (Map.Entry<String, Activate> entry : cachedActivates.entrySet()) {
		//扩展名称
		String name = entry.getKey();
		//@Activate注解
		Activate activate = entry.getValue();
		//group是否匹配(后面会分析该方法)
		if (isMatchGroup(group, activate.group())) {
		    //根据扩展名称name获取扩展实例
		    T ext = getExtension(name);
		    if (!names.contains(name)
			    && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)
			    && isActive(activate, url)) {
			//当扩展名称数组不包含该扩展名name，并且不包含-name，并且已激活时，
			//将当前扩展实例添加到返回结果集中
			exts.add(ext);
		    }
		}
	    }
	    //按照配置的before()和after()/order()进行排序
	    Collections.sort(exts, ActivateComparator.COMPARATOR);
	}
	List<T> usrs = new ArrayList<T>();
	for (int i = 0; i < names.size(); i++) {
	    //当前扩展名称name
	    String name = names.get(i);
	    if (!name.startsWith(Constants.REMOVE_VALUE_PREFIX)
		    //当前扩展名称name不是以-开头，并且扩展名称数组names不包含-name时，
		    && !names.contains(Constants.REMOVE_VALUE_PREFIX + name)) {
		if (Constants.DEFAULT_KEY.equals(name)) {
		    //当前扩展名称name=default
		    if (!usrs.isEmpty()) {
		        //usrs不为空，将usrs集合添加到exts头部
			exts.addAll(0, usrs);
			//清空usrs
			usrs.clear();
		    }
		} else {
		    //当前扩展名称name != default时，获取该name的扩展实例，并存起来
		    T ext = getExtension(name);
		    usrs.add(ext);
		}
	    }
	}
	if (!usrs.isEmpty()) {
	    exts.addAll(usrs);
	}
	return exts;
}

/**
 * group是否匹配
 * 1、group为空，则匹配
 * 2、group在groups中存在，则匹配
 * @param group
 * @param groups @Activate中配置的group数组
 * @return
 */
private boolean isMatchGroup(String group, String[] groups) {
	if (group == null || group.length() == 0) {
	    return true;
	}
	if (groups != null && groups.length > 0) {
	    for (String g : groups) {
		if (group.equals(g)) {
		    return true;
		}
	    }
	}
	return false;
}
/**
 * 是否已激活
 * 1、@activate没有配置value()
 * 2、@Activate中配置的value()在url的参数列表中存在,且url参数对应的value不为空
 * @param activate
 * @param url
 * @return
 */
private boolean isActive(Activate activate, URL url) {
        //获取@Activate注解中配置的keys
	String[] keys = activate.value();
	if (keys.length == 0) {
	    return true;
	}
	for (String key : keys) {
	    for (Map.Entry<String, String> entry : url.getParameters().entrySet()) {
		String k = entry.getKey();
		String v = entry.getValue();
		if ((k.equals(key) || k.endsWith("." + key))
			&& ConfigUtils.isNotEmpty(v)) {
		    return true;
		}
	    }
	}
	return false;
}
```

#### getAdaptiveExtension方法
接下来看下getAdaptiveExtension方法，该方法用来获取扩展接口type对应的自适应扩展实例。
```java

/**
 * 缓存的自适应实例
 */
private final Holder<Object> cachedAdaptiveInstance = new Holder<Object>();

/**
 * 记录创建自适应扩展实例时发生的异常
 */
private volatile Throwable createAdaptiveInstanceError;


/**
 * 获取扩展接口type的自适应扩展实例
 */
public T getAdaptiveExtension() {
	//先从缓存中获取
	Object instance = cachedAdaptiveInstance.get();
	if (instance == null) {
	    //缓存中不存在
	    //判断createAdaptiveInstanceError变量是否为空，不为空说明之前创建自适应实例时发生了异常
	    if (createAdaptiveInstanceError == null) {
	        //没有发生异常，同步创建自适应扩展实例
		synchronized (cachedAdaptiveInstance) {
		    //再次验证缓存中是否存在该实例
		    instance = cachedAdaptiveInstance.get();
		    if (instance == null) {
			try {
			    //创建自适应扩展实例(后面会分析该方法)
			    instance = createAdaptiveExtension();
			    //将创建好的实例保存进缓存
			    cachedAdaptiveInstance.set(instance);
			} catch (Throwable t) {
			    //创建实例时发生异常，记录异常
			    createAdaptiveInstanceError = t;
			    throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
			}
		    }
		}
	    } else {
		throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
	    }
	}
	return (T) instance;
}

/**
 * 为扩展接口type创建自适应扩展实例，并处理依赖注入
 * @return
 */
private T createAdaptiveExtension() {
	try {
	    //在这里调用了getAdaptiveExtensionClass方法获取到自适应扩展类，然后生成实例
	    return injectExtension((T) getAdaptiveExtensionClass().newInstance());
	} catch (Exception e) {
	    throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
	}
}

/**
 * 获取扩展接口type的自适应扩展类
 * @return
 */
private Class<?> getAdaptiveExtensionClass() {
        //加载扩展接口type对应的所有扩展实现类(上文介绍过该方法)
	getExtensionClasses();
	//查看缓存中是否已经存在扩展接口type对应的自适应扩展类，如果存在，则直接返回(调用getExtensionClasses方法时，会去设置cachedAdaptiveClass值)
	if (cachedAdaptiveClass != null) {
	    return cachedAdaptiveClass;
	}
	//不存在的话，为该扩展接口type创建一个自适应扩展类(后面会分析该方法)
	return cachedAdaptiveClass = createAdaptiveExtensionClass();
}

/**
 * 扩展接口type不存在自适应扩展类的时候，dubbo默认会为其创建
 * 创建条件：type接口的方法中必须有一个带@Adaptive注解的方法dubbo才会创建自适应扩展类
 * @return
 */
private Class<?> createAdaptiveExtensionClass() {
	//生成自适应扩展实现类的代码(后面会分析该方法)
	String code = createAdaptiveExtensionClassCode();
	//获取类加载器
	ClassLoader classLoader = findClassLoader();
	//获取Compiler自适应扩展实例
	com.alibaba.dubbo.common.compiler.Compiler compiler =
		ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class)
			.getAdaptiveExtension();
	//编译code(后面会分析该方法)
	return compiler.compile(code, classLoader);
}
```

接下来的createAdaptiveExtensionClassCode方法比较长，在分析之前，我们先来看下ProxyFactory接口，我们将使用该接口作为例子(即假设当前扩展接口type为ProxyFactory),配合着该方法一起看：
```java
@SPI("javassist")
public interface ProxyFactory {
    
    //@Adaptive({"proxy"})
    @Adaptive({Constants.PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;
   
    @Adaptive({Constants.PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;
```
可以看到该接口上添加了@SPI("javassist")注解、接口方法上添加了@Adaptive({Constants.PROXY_KEY})

```java
/**
* 为扩展接口type生成自适应扩展类的类代码
* 假设 type = com.alibaba.dubbo.rpc.ProxyFactory
* @return
*/
private String createAdaptiveExtensionClassCode() {
	StringBuilder codeBuilder = new StringBuilder();
	//获取扩展接口type的方法列表，查看方法上是否存在@Adaptive注解，
	//如果不存在@Adaptive注解，则不用生成自适应扩展类
	Method[] methods = type.getMethods();
	boolean hasAdaptiveAnnotation = false;
	for (Method m : methods) {
	    if (m.isAnnotationPresent(Adaptive.class)) {
		hasAdaptiveAnnotation = true;
		break;
	    }
	}
	// 不存在带有@Adaptive注解的方法，因此不需要生成自适应扩展类
	if (!hasAdaptiveAnnotation) {
	    throw new IllegalStateException("No adaptive method on extension " + type.getName() + ", refuse to create the adaptive class!");
	}
	//添加包声明：type扩展接口所在包包名
	//package com.alibaba.dubbo.rpc;
	codeBuilder.append("package ").append(type.getPackage().getName()).append(";");

	//import com.alibaba.dubbo.common.extension.ExtensionLoader;
	codeBuilder.append("\nimport ").append(ExtensionLoader.class.getName()).append(";");
	
	//可以看到生成的实现类名称为ProxyFactory$Adaptive，实现了ProxyFactory接口
	//public class com.alibaba.dubbo.rpc.ProxyFactory$Adaptive implements com.alibaba.dubbo.rpc.ProxyFactory{
	codeBuilder.append("\npublic class ")
		.append(type.getSimpleName())
		.append("$Adaptive")
		.append(" implements ")
		.append(type.getCanonicalName()).append(" {");

	//遍历type接口的方法
	for (Method method : methods) {
	    //接口方法返回类型
	    Class<?> rt = method.getReturnType();
	    //接口方法参数类型数组
	    Class<?>[] pts = method.getParameterTypes();
	    //接口方法异常类型数组
	    Class<?>[] ets = method.getExceptionTypes();
	   
	    //判断当前方法上是否存在@Adaptive注解
	    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
            //方法内容
	    StringBuilder code = new StringBuilder(512);
	    if (adaptiveAnnotation == null) {
		//添加异常：type接口的当前方法不是一个adaptive方法
		code.append("throw new UnsupportedOperationException(\"method ")
			.append(method.toString()).append(" of interface ")
			.append(type.getName()).append(" is not adaptive method!\");");
	    } else {
		//获取到的当前方法的"URL类型参数/返回URL类型的get方法的参数"的下标（在当前方法参数中的下标）
		int urlTypeIndex = -1;
		//遍历当前方法参数
		for (int i = 0; i < pts.length; ++i) {
		    if (pts[i].equals(URL.class)) {
		        //第i个下标是URL类型的参数
			urlTypeIndex = i;
			break;
		    }
		}
		if (urlTypeIndex != -1) {
		    //从当前方法参数中找到了URL类型的参数
		    //添加"校验Url类型参数是否为空"的code
		    String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"url == null\");",
			    urlTypeIndex);
		    code.append(s);
		    
		    //添加"声明url类型的变量"的code
		    //URL url = arg{$urlTypeIndex}
		    s = String.format("\n%s url = arg%d;", URL.class.getName(), urlTypeIndex);
		    code.append(s);
		}else {
		    //从当前方法参数中没有找到URL类型的参数
		    //返回URL类型的get方法的名称
		    String attribMethod = null;

		    //遍历参数列表，查询每个参数的方法列表，查找返回URL类型的get方法
		    LBL_PTS:
		    //遍历当前方法参数
		    for (int i = 0; i < pts.length; ++i) {
			//遍历方法第i个参数的所有方法
			Method[] ms = pts[i].getMethods();
			for (Method m : ms) {
			    //当前方法名
			    String name = m.getName();
			    //当前方法没有参数，且返回值是URL类型，以get开头或者长度>3,且是public修饰，没有static修饰
			    if ((name.startsWith("get") || name.length() > 3)
				    && Modifier.isPublic(m.getModifiers())
				    && !Modifier.isStatic(m.getModifiers())
				    && m.getParameterTypes().length == 0
				    && m.getReturnType() == URL.class) {
				//方法的第i个参数是"返回URL类型的get方法"
				urlTypeIndex = i;
				//“返回URL类型的get方法的名称”
				attribMethod = name;
				//跳转到LBL_PTS
				break LBL_PTS;
			    }
			}
		    }
		    if (attribMethod == null) {
			//没有找到返回URL类型的方法，抛异常
			throw new IllegalStateException("fail to create adaptive class for interface " + type.getName()
				+ ": not found url parameter or url attribute in parameters of method " + method.getName());
		    }
			
		    //添加“方法第urlTypeIndex个参数为空的判断”
		    String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"%s argument == null\");",
			    urlTypeIndex, pts[urlTypeIndex].getName());
		    code.append(s);
			
		    //添加“方法第urlTypeIndex个参数的attribMethod方法的返回值为空的判断”
		    s = String.format("\nif (arg%d.%s() == null) throw new IllegalArgumentException(\"%s argument %s() == null\");",
			    urlTypeIndex, attribMethod, pts[urlTypeIndex].getName(), attribMethod);
		    code.append(s);

		    //添加：com.alibaba.dubbo.common.URL url = arg{$urlTypeIndex}.attribMethod();
		    s = String.format("%s url = arg%d.%s();", URL.class.getName(), urlTypeIndex, attribMethod);
		    code.append(s);
		}
		//@Adaptive注解的value值
		String[] value = adaptiveAnnotation.value();
		if (value.length == 0) {
		    //没有设置value，则根据扩展接口名称按照一定规则生成value
		    //如：ProxyFactory 将生成value：proxy.factory
		    //则charArray = [P,r,o,x,y,F,a,c,t,o,r,y]
		    char[] charArray = type.getSimpleName().toCharArray();
		    StringBuilder sb = new StringBuilder(128);
		    for (int i = 0; i < charArray.length; i++) {
			if (Character.isUpperCase(charArray[i])) {
			    //当前字符是大写的
			    if (i != 0) {
			        //当前字符不是第1个字符，则添加“.”到sb中
				sb.append(".");
			    }
			    //将当前字符转为小写，并添加到sb中
			    sb.append(Character.toLowerCase(charArray[i]));
			} else {
			    //当前字符是小写的，直接添加到sb中
			    sb.append(charArray[i]);
			}
		    }
		    //生成的value = [{"proxy.factory"}]
		    value = new String[]{sb.toString()};
		}
		//当前方法中是否存在Invocation类型的参数
		boolean hasInvocation = false;
		for (int i = 0; i < pts.length; ++i) {
		    //当前方法的第i个参数名称是否为“com.alibaba.dubbo.rpc.Invocation”
		    if (pts[i].getName().equals("com.alibaba.dubbo.rpc.Invocation")) {
			//添加"校验参数Invocation是否为空"的code
			String s = String.format("\nif (arg%d == null) throw new IllegalArgumentException(\"invocation == null\");", i);
			code.append(s);
			//添加"获取方法名"的code
			s = String.format("\nString methodName = arg%d.getMethodName();", i);
			code.append(s);
			//标识 存在Invocation类型的参数
			hasInvocation = true;
			break;
		    }
		}
		//@SPI注解上配置的值，默认扩展名称
		String defaultExtName = cachedDefaultName;
		String getNameCode = null;
		//遍历@Adaptive注解的value数组
		for (int i = value.length - 1; i >= 0; --i) {
		    if (i == value.length - 1) {
		        //当前i为value的最后1个元素
			if (null != defaultExtName) {
			    //默认扩展名称不为空，如果没有获取到值，则会使用默认值
			    //第i个元素是否等于protocol
			    if (!"protocol".equals(value[i])) {
				if (hasInvocation) {
				    //存在Invocation类型的参数
				    //调用url的getMethodParameter方法，获取value[i]对应的扩展名，如果为空，则取默认扩展名defaultExtName
				    getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
				} else {
				    //不存在Invocation类型的参数
				    //从url的value[i]参数中(当前例子为proxy)获取扩展名，如果没有获取到，则使用默认扩展名javassist
				    //url.getParameter("proxy","javassist")
				    getNameCode = String.format("url.getParameter(\"%s\", \"%s\")", value[i], defaultExtName);
				}
			    } else {
			        //第i个元素等于protocol
				//从url中获取protocol属性值，如果没有获取到，则使用默认扩展名defaultExtName
				getNameCode = String.format("( url.getProtocol() == null ? \"%s\" : url.getProtocol() )", defaultExtName);
			    }
			} else {
			    //默认扩展名称为空，如果没有获取到值，则不会使用默认值
			    if (!"protocol".equals(value[i])) {
				if (hasInvocation) {
				    getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
				} else {
				    //从url中获取参数value[i]对应的值
				    getNameCode = String.format("url.getParameter(\"%s\")", value[i]);
				}
			    } else {
			        //从url中获取protocol属性值
				getNameCode = "url.getProtocol()";
			    }
			}
		    } else {
		        //当前第i个元素不是value最后一个元素
			if (!"protocol".equals(value[i])) {
			    if (hasInvocation) {
				getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
			    } else {
				getNameCode = String.format("url.getParameter(\"%s\", %s)", value[i], getNameCode);
			    }
			} else {
			    getNameCode = String.format("url.getProtocol() == null ? (%s) : url.getProtocol()", getNameCode);
			}
		    }
		}
		
		//添加：String extName = url.getParameter("proxy", "javassist");
		//获取扩展名
		code.append("\nString extName = ").append(getNameCode).append(";");
		
		//添加“校验扩展名不为空”的code
		String s = String.format("\nif(extName == null) " +
				"throw new IllegalStateException(\"Fail to get extension(%s) name from url(\" + url.toString() + \") use keys(%s)\");",
			type.getName(), Arrays.toString(value));
		code.append(s);

		//添加"根据扩展名获取扩展实例"的code
		//com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName) ;
		s = String.format("\n%s extension = (%<s)%s.getExtensionLoader(%s.class).getExtension(extName);",
			type.getName(), ExtensionLoader.class.getSimpleName(), type.getName());
		code.append(s);

		//处理返回值
		if (!rt.equals(void.class)) {
		    //返回值不是void类型
		    code.append("\nreturn ");
		}

		//例如：return extension.getInvoker(arg0);
		//这里的逻辑就是：根据扩展名称的不同获取相应的扩展实现，然后调用方法，并返回
		s = String.format("extension.%s(", method.getName());
		//先填写调用信息
		code.append(s);
		
		//遍历当前方法参数，添加方法参数
		for (int i = 0; i < pts.length; i++) {
		    if (i != 0) {
			code.append(", ");
		    }
		    code.append("arg").append(i);
		}
		code.append(");");
	    }

	    //生成方法签名code
	    //如：public java.lang.Object getProxy(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException
	    codeBuilder.append("\npublic ")
		    .append(rt.getCanonicalName())
		    .append(" ")
		    .append(method.getName())
		    .append("(");

            //添加参数
	    for (int i = 0; i < pts.length; i++) {
		if (i > 0) {
		    codeBuilder.append(", ");
		}
		codeBuilder.append(pts[i].getCanonicalName());
		codeBuilder.append(" ");
		codeBuilder.append("arg").append(i);
	    }
	    codeBuilder.append(")");

	    //添加异常
	    if (ets.length > 0) {
		codeBuilder.append(" throws ");
		for (int i = 0; i < ets.length; i++) {
		    if (i > 0) {
			codeBuilder.append(", ");
		    }
		    codeBuilder.append(ets[i].getCanonicalName());
		}
	    }
	   
	    codeBuilder.append(" {");
	     //将方法code添加到方法代码块中
	    codeBuilder.append(code.toString());
	    codeBuilder.append("\n}");
	}
	codeBuilder.append("\n}");
	if (logger.isDebugEnabled()) {
	    logger.debug(codeBuilder.toString());
	}
	//返回生产的code
	return codeBuilder.toString();
}
```
以type = com.alibaba.dubbo.rpc.ProxyFactory为例子，生成的代码大概如下所示:
```java
package com.alibaba.dubbo.rpc;
import com.alibaba.dubbo.common.extension.ExtensionLoader;

public class ProxyFactory$Adaptive implements com.alibaba.dubbo.rpc.ProxyFactory {

    public java.lang.Object getProxy(com.alibaba.dubbo.rpc.Invoker arg0) throws com.alibaba.dubbo.rpc.RpcException {
	 if (arg0 == null){
	    throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument == null");
	 }
	 if (arg0.getUrl() == null) {
	    throw new IllegalArgumentException("com.alibaba.dubbo.rpc.Invoker argument getUrl() == null");
	 }
	 com.alibaba.dubbo.common.URL url = arg0.getUrl();
	 String extName = url.getParameter("proxy", "javassist");
	 if(extName == null) {
	    throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
	 }
	 com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
	 return extension.getProxy(arg0);
    }

    public com.alibaba.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, com.alibaba.dubbo.common.URL arg2) throws com.alibaba.dubbo.rpc.RpcException {
	if (arg2 == null) {
	    throw new IllegalArgumentException("url == null");
	}
	com.alibaba.dubbo.common.URL url = arg2;
	String extName = url.getParameter("proxy", "javassist");
	if(extName == null){
	    throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.rpc.ProxyFactory) name from url(" + url.toString() + ") use keys([proxy])");
	}
	com.alibaba.dubbo.rpc.ProxyFactory extension = (com.alibaba.dubbo.rpc.ProxyFactory)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.rpc.ProxyFactory.class).getExtension(extName);
	return extension.getInvoker(arg0, arg1, arg2);
    }
}
```
以上就是生成自适应实现类的全部内容，生成好代码后，会交由Compiler扩展进行编译得到Class对象，然后在本小节开头介绍的createAdaptiveExtension()方法中进行实例化，并进行依赖注入。
本文内容过多，Compiler编译的部分就放到后面的博文中在介绍了。另外ExtensionLoader类中也提供了很多辅助方法，内容比较简单，在这里就不详细介绍了。

### ExtensionFactory工厂实现
```
@SPI
public interface ExtensionFactory {
    /**
     * 获取扩展实现类实例
     * Get extension.
     * @param type object type. 扩展接口类型
     * @param name object name. 扩展名称
     * @return object instance. 返回扩展实例
     */
    <T> T getExtension(Class<T> type, String name);
}
```
该接口根据扩展类型和扩展名称获取扩展实例。可以看到该接口声明上也加上了@SPI注解，说明存在多种实现，Dubbo提供了三种实现，分别为AdaptiveExtensionFactory、SpiExtensionFactory、SpringExtensionFactory.

#### SpringExtensionFactory
```java
public class SpringExtensionFactory implements ExtensionFactory {

    /**
     * 保存所有的ApplicationContext上下文对象
     */
    private static final Set<ApplicationContext> contexts = new ConcurrentHashSet<ApplicationContext>();

    /**
     * 添加ApplicationContext
     */
    public static void addApplicationContext(ApplicationContext context) {
        contexts.add(context);
    }

    /**
     * 移除ApplicationContext
     */
    public static void removeApplicationContext(ApplicationContext context) {
        contexts.remove(context);
    }

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        //遍历所有的context
        for (ApplicationContext context : contexts) {
            //判断该context上下文是否包含此bean
            if (context.containsBean(name)) {
                //根据名称获取context中的bean
                Object bean = context.getBean(name);
                //判断该bean是否是type类型的实例，如果是的话，则返回该bean
                if (type.isInstance(bean)) {
                    return (T) bean;
                }
            }
        }
        return null;
    }
}
```

#### SpiExtensionFactory

```java
public class SpiExtensionFactory implements ExtensionFactory {

    @Override
    public <T> T getExtension(Class<T> type, String name) {
        //判断类型type是否是接口，并且存在@SPI注解
        if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
            //获取扩展接口type的扩展加载器
            ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
	    //通过扩展加载器获取接口type支持的所有扩展的名称列表
            if (!loader.getSupportedExtensions().isEmpty()) {
                //返回自适应扩展实例
                return loader.getAdaptiveExtension();
            }
        }
        return null;
    }
}
```

#### AdaptiveExtensionFactory
该ExtensionFactory实现存在@Adaptive注解，因此它是自适应实现。

```java
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    /**
     * ExtensionFactory扩展实现类实例集合
     */
    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        //获取ExtensionFactory.class对应的扩展加载器
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        
        //遍历扩展接口ExtensionFactory支持的扩展名称列表(spi、spring)
        for (String name : loader.getSupportedExtensions()) {
            //根据扩展名称name加载扩展实现类实例
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    /**
     * 自适应的扩展工厂，挨个遍历spring和spi，从中查询指定类型的实例
     * @param type object type. 扩展接口类型
     * @param name object name. 扩展名称
     * @param <T>
     * @return
     */
    @Override
    public <T> T getExtension(Class<T> type, String name) {
        //遍历ExtensionFactory实现类集合，从中挨个查找指定类型的扩展
        for (ExtensionFactory factory : factories) {
            //获取type类型的扩展实例
            T extension = factory.getExtension(type, name);
            if (extension != null) {
               //实例不为空，返回实例 
               return extension;
            }
        }
        return null;
    }
}
```

到此，关于SPI的实现就全部介绍完了，下一小节将会介绍和Sping集成相关的内容。















