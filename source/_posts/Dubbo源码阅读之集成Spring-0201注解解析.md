---
title: Dubbo源码阅读之集成Spring注解解析(0201)
date: 2018-08-05 09:29:48
tags:
     - dubbo
---
>本小节介绍Annotation的解析

### AnnotationBeanDefinitionParser解析类

```java
该类继承自AbstractSingleBeanDefinitionParser抽象类，该抽象类规范了注册bean的骨架(即模板方法)，我们只需要实现getBeanClass方法指定待注册的bean，以及实现doParse方法表明如何解析类。
public class AnnotationBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {

    /**
     * 解析：<dubbo:annotation package=""/>
     * @param element
     * @param parserContext
     * @param builder
     */
    @Override
    protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
        //获取配置的package属性
        String packageToScan = element.getAttribute("package");
        //逗号分隔package
        String[] packagesToScan = trimArrayElements(commaDelimitedListToStringArray(packageToScan));

        //通过构造函数设置ServiceAnnotationBeanPostProcessor类的packagesToScan属性
        builder.addConstructorArgValue(packagesToScan);
        //标识为基础设施类，防止该bean被代理
        builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);

        //注册@ReferenceAnnotationBeanPostProcessor
        registerReferenceAnnotationBeanPostProcessor(parserContext.getRegistry());

    }

    @Override
    protected boolean shouldGenerateIdAsFallback() {
        return true;
    }

    /**
     * 注册ReferenceAnnotationBeanPostProcessor类
     * 该类用来处理@Reference
     * Registers {@link ReferenceAnnotationBeanPostProcessor} into {@link BeanFactory}
     * @param registry {@link BeanDefinitionRegistry}
     */
    private void registerReferenceAnnotationBeanPostProcessor(BeanDefinitionRegistry registry) {
        // Register @Reference Annotation Bean Processor
        BeanRegistrar.registerInfrastructureBean(registry,
                ReferenceAnnotationBeanPostProcessor.BEAN_NAME,
                ReferenceAnnotationBeanPostProcessor.class);
    }

    @Override
    protected Class<?> getBeanClass(Element element) {
        //解析注册ServiceAnnotationBeanPostProcessor类
        return ServiceAnnotationBeanPostProcessor.class;
    }
}
```
我们将ServiceAnnotationBeanPostProcessor类和ReferenceAnnotationBeanPostProcessor类注册到了Bean工厂中了，接下来我们将用两小节(0201/0202)来介绍这两个类的实现，先来看ServiceAnnotationBeanPostProcessor类的实现(0201)，ReferenceAnnotationBeanPostProcessor类放到(0202)小节介绍


#### ServiceAnnotationBeanPostProcessor类实现
接下来我们来分析下ServiceAnnotationBeanPostProcessor类,该类用来处理@Service注解，它实现了BeanDefinitionRegistryPostProcessor接口,实现它的postProcessBeanDefinitionRegistry方法允许我们实现自定义的注册bean定义的逻辑。同时也实现了几个以Aware为结尾的接口，例如BeanClassLoaderAware，实现了这些接口后，则ServiceAnnotationBeanPostProcessor类被实例化后将会获取相对应的资源。
```java
public class ServiceAnnotationBeanPostProcessor implements BeanDefinitionRegistryPostProcessor, EnvironmentAware,
        ResourceLoaderAware, BeanClassLoaderAware {

    //分隔符	
    private static final String SEPARATOR = ":";

    //扫描的包路径(上面的小节介绍过，注册该bean时，会扫描包路径)
    private final Set<String> packagesToScan;
    
    //实现EnvironmentAware接口，实例化后会自动注入
    private Environment environment;

    private ResourceLoader resourceLoader;

    private ClassLoader classLoader;
 
    //构造方法，参数是需要扫描的包路径
    public ServiceAnnotationBeanPostProcessor(String... packagesToScan) {
        this(Arrays.asList(packagesToScan));
    }

    public ServiceAnnotationBeanPostProcessor(Collection<String> packagesToScan) {
        this(new LinkedHashSet<String>(packagesToScan));
    }

    public ServiceAnnotationBeanPostProcessor(Set<String> packagesToScan) {
        //保存包路径
	this.packagesToScan = packagesToScan;
    }
	
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
	//处理待扫描包里面的占位符(后面会分析该方法)
        Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);
        
        if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
            //扫描包，找到存在@Service注解的类，然后注册它(后面会分析该方法)
	    registerServiceBeans(resolvedPackagesToScan, registry);
        } else {
            if (logger.isWarnEnabled()) {
                //没有配置带扫描的包路径，将会忽略ServiceBean的注册
                logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
            }
        }
    }
   
    /**
     * 处理带扫描包里面的占位符
     * @param packagesToScan
     * @return
     */
    private Set<String> resolvePackagesToScan(Set<String> packagesToScan) {
        Set<String> resolvedPackagesToScan = new LinkedHashSet<String>(packagesToScan.size());
        //遍历带扫描的包
        for (String packageToScan : packagesToScan) {
            if (StringUtils.hasText(packageToScan)) {
                //通过environment解决占位符
                String resolvedPackageToScan = environment.resolvePlaceholders(packageToScan.trim());
                resolvedPackagesToScan.add(resolvedPackageToScan);
            }
        }
        return resolvedPackagesToScan;
    }
   
    /**
     * 注册存在@Service注解的类
     * @param packagesToScan 待扫描的基础包路径
     * @param registry       {@link BeanDefinitionRegistry}
     */
    private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

	//创建扫描器，用来扫描指定的包路径(后面会介绍该类)
        DubboClassPathBeanDefinitionScanner scanner =
                new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);

	//获取BeanNameGenerator实例用来生成bean name(后面会介绍该方法)
        BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);
        
	//将BeanNameGenerator设置到扫描器
        scanner.setBeanNameGenerator(beanNameGenerator);
        
        //设置过滤器，只扫描存在@Service注解的
        scanner.addIncludeFilter(new AnnotationTypeFilter(Service.class));

	//遍历基础包
        for (String packageToScan : packagesToScan) {

            // 首先注册@Service bean
            scanner.scan(packageToScan);

            // 找到该包下所有存在@Service的BeanDefinition(不论是否是@ComponentScan扫描的)，
	    //然后为该bean生成beanName，并将该beanName和BeanDefinition包装成BeanDefinitionHolder（后面会分析该方法）
            Set<BeanDefinitionHolder> beanDefinitionHolders =
                    findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

            if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {
                
		//遍历beanDefinitionHolders
                for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
		    //根据@Service注解 和 beanDefinition  注册ServiceBean类(后面会分析该方法)
                    registerServiceBean(beanDefinitionHolder, registry, scanner);
                }

                if (logger.isInfoEnabled()) {
		    //输出扫描出来的@Service bean的数量，以及扫描的包路径
                    logger.info(beanDefinitionHolders.size() + " annotated Dubbo's @Service Components { " +
                            beanDefinitionHolders +
                            " } were scanned under package[" + packageToScan + "]");
                }
            } else {
                //没有扫描到
                if (logger.isWarnEnabled()) {
                    logger.warn("No Spring Bean annotating Dubbo's @Service was found under package["
                            + packageToScan + "]");
                }
            }
        }
    }

    /**
     * 获取BeanNameGenerator
     * @since 2.5.8
     */
    private BeanNameGenerator resolveBeanNameGenerator(BeanDefinitionRegistry registry) {

        BeanNameGenerator beanNameGenerator = null;

        if (registry instanceof SingletonBeanRegistry) {
	    //单例bean
            SingletonBeanRegistry singletonBeanRegistry = SingletonBeanRegistry.class.cast(registry);
	    //获取beanName生成器
            beanNameGenerator = (BeanNameGenerator) singletonBeanRegistry.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
        }
        if (beanNameGenerator == null) {
            if (logger.isInfoEnabled()) {

                logger.info("BeanNameGenerator bean can't be found in BeanFactory with name ["
                        + CONFIGURATION_BEAN_NAME_GENERATOR + "]");
                logger.info("BeanNameGenerator will be a instance of " +
                        AnnotationBeanNameGenerator.class.getName() +
                        " , it maybe a potential problem on bean name generation.");
            }
	    //使用AnnotationBeanNameGenerator
            beanNameGenerator = new AnnotationBeanNameGenerator();
        }
        return beanNameGenerator;
    }

    /**
     * 扫描包路径，通过过滤器找到存在@Service注解的bean，然后为该bean生成beanName，
     * 然后将该beanName和beanDefinition封装成BeanDefinitionHolder对象，返回BeanDefinitionHolder列表
     */
    private Set<BeanDefinitionHolder> findServiceBeanDefinitionHolders(
            ClassPathBeanDefinitionScanner scanner, String packageToScan, BeanDefinitionRegistry registry,
            BeanNameGenerator beanNameGenerator) {
 	
	//扫描基础包，查询候选组件（通过过滤器过滤出来的，即存在@Service注解的bean）
        Set<BeanDefinition> beanDefinitions = scanner.findCandidateComponents(packageToScan);

        Set<BeanDefinitionHolder> beanDefinitionHolders = new LinkedHashSet<BeanDefinitionHolder>(beanDefinitions.size());

        for (BeanDefinition beanDefinition : beanDefinitions) {
	    //根据beanDefinition生成bean名称
            String beanName = beanNameGenerator.generateBeanName(beanDefinition, registry);
           
	    //根据beanName和beanDefinition创建BeanDefinitionHolder对象
	    BeanDefinitionHolder beanDefinitionHolder = new BeanDefinitionHolder(beanDefinition, beanName);
           
	    //保存到列表	
	    beanDefinitionHolders.add(beanDefinitionHolder);
        }
        return beanDefinitionHolders;
    }

    /**
     * 根据@Service注解 和 beanDefinition  注册ServiceBean类
     * Registers {@link ServiceBean} from new annotated {@link Service} {@link BeanDefinition}
     */
    private void registerServiceBean(BeanDefinitionHolder beanDefinitionHolder, BeanDefinitionRegistry registry,
                                     DubboClassPathBeanDefinitionScanner scanner) {
	
	//从beanDefinitionHolder中获取beanName，然后加载该类(后面会分析该方法)
        Class<?> beanClass = resolveClass(beanDefinitionHolder);

	//查询该bean的@Service注解(后面会分析该方法)
        Service service = findAnnotation(beanClass, Service.class);
		
	//获取interfaceClass接口(后面会分析该方法)
        Class<?> interfaceClass = resolveServiceInterfaceClass(beanClass, service);
	
	//获取bean的名称
        String annotatedServiceBeanName = beanDefinitionHolder.getBeanName();
	
	//构建ServiceBean定义(后面会分析该方法)
        AbstractBeanDefinition serviceBeanDefinition =
                buildServiceBeanDefinition(service, interfaceClass, annotatedServiceBeanName);
	
        //生成ServiceBean的beanName(后面会分析该方法)
        String beanName = generateServiceBeanName(service, interfaceClass, annotatedServiceBeanName);

	//检测重复的候选bean
        if (scanner.checkCandidate(beanName, serviceBeanDefinition)) {
	    //注解ServiceBean
            registry.registerBeanDefinition(beanName, serviceBeanDefinition);

            if (logger.isInfoEnabled()) {
                logger.warn("The BeanDefinition[" + serviceBeanDefinition +
                        "] of ServiceBean has been registered with name : " + beanName);
            }
        } else {
	    //发现重复的bean定义，@DubboComponentScan多次扫描到同一个包？
            if (logger.isWarnEnabled()) {
                logger.warn("The Duplicated BeanDefinition[" + serviceBeanDefinition +
                        "] of ServiceBean[ bean name : " + beanName +
                        "] was be found , Did @DubboComponentScan scan to same package in many times?");
            }
        }
    }

    /**
     * 生成ServiceBean的bean name
     *
     * @param service  @Service注解
     * @param interfaceClass 存在@Service注解的类的接口
     * @param annotatedServiceBeanName 存在@Service注解的Bean name
     * @return ServiceBean@interfaceClassName#annotatedServiceBeanName
     * @since 2.5.9
     */
    private String generateServiceBeanName(Service service, Class<?> interfaceClass, String annotatedServiceBeanName) {
	//添加"ServiceBean"
        StringBuilder beanNameBuilder = new StringBuilder(ServiceBean.class.getSimpleName());

	//添加分隔符":" 和 annotatedServiceBeanName 
        beanNameBuilder.append(SEPARATOR).append(annotatedServiceBeanName);

        //获取接口的全限定名
        String interfaceClassName = interfaceClass.getName();
  
        //添加分隔符":" 和 interfaceClassName
        beanNameBuilder.append(SEPARATOR).append(interfaceClassName);
       
  	//获取@Service注解的version属性
        String version = service.version();
	
        if (StringUtils.hasText(version)) {
            //添加分隔符":" 和 version
	    beanNameBuilder.append(SEPARATOR).append(version);
        }
	
	//获取Service的group属性
        String group = service.group();
        if (StringUtils.hasText(group)) {
	    //添加分隔符":" 和 group
            beanNameBuilder.append(SEPARATOR).append(group);
        }
	//返回生成的bean名称
        return beanNameBuilder.toString();
    }

    /**
     * 处理@Service注解里的interfaceClass()，该interfaceClass不可以为空，并且必须是接口类型
     * 1、获取@Service注解中的interfaceClass()
     * 2、获取@Service注解中的interfaceName()，并加载
     * 3、获取@Service注解类的第1个接口
     * @param annotatedServiceBeanClass 存在@Service注解的类
     * @param service @Service注解
     * @return
     */
    private Class<?> resolveServiceInterfaceClass(Class<?> annotatedServiceBeanClass, Service service) {
        
	//interfaceClass默认为@Service注解的interfaceClass属性
        Class<?> interfaceClass = service.interfaceClass();
	
	
        if (void.class.equals(interfaceClass)) {
	    //@Service注解的interfaceClass属性为空
            interfaceClass = null;
		
	    //获取@Service注解的interfaceName属性
            String interfaceClassName = service.interfaceName();
            if (StringUtils.hasText(interfaceClassName)) {
		//判断是否存在interfaceClassName类
                if (ClassUtils.isPresent(interfaceClassName, classLoader)) {
		    //加载interfaceClassName类，并赋值给interfaceClass
                    interfaceClass = resolveClassName(interfaceClassName, classLoader);
                }
            }
        }

        if (interfaceClass == null) {
	    //获取该类的所有接口
            Class<?>[] allInterfaces = annotatedServiceBeanClass.getInterfaces();
            if (allInterfaces.length > 0) {
		//取第一个接口，赋值给interfaceClass
                interfaceClass = allInterfaces[0];
            }

        }
	//验证不为空，且interfaceClass为接口
        Assert.notNull(interfaceClass,
                "@Service interfaceClass() or interfaceName() or interface class must be present!");

        Assert.isTrue(interfaceClass.isInterface(),
                "The type that was annotated @Service is not an interface!");

        return interfaceClass;
    }
	
    private Class<?> resolveClass(BeanDefinitionHolder beanDefinitionHolder) {
	//获取bean定义
        BeanDefinition beanDefinition = beanDefinitionHolder.getBeanDefinition();
        return resolveClass(beanDefinition);

    }
	
    /**
     * 加载类
     */
    private Class<?> resolveClass(BeanDefinition beanDefinition) {
	//获取bean的类名
        String beanClassName = beanDefinition.getBeanClassName();
 	//加载该类	
        return resolveClassName(beanClassName, classLoader);
    }

    /**
     * 构建ServiceBean定义
     * @param service  @Service注解
     * @param interfaceClass 接口类
     * @param annotatedServiceBeanName 存在@Service注解的类的bean name
     */
    private AbstractBeanDefinition buildServiceBeanDefinition(Service service, Class<?> interfaceClass,
                                                              String annotatedServiceBeanName) {
	
	//获取ServiceBean的BeanDefinition构造器
        BeanDefinitionBuilder builder = rootBeanDefinition(ServiceBean.class);
	
	//获取BeanDefinition
        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
	
	//获取ServiceBean的属性
        MutablePropertyValues propertyValues = beanDefinition.getPropertyValues();

	//忽略的属性名称,这些属性的值后面会设置，从@Service注解中获取
        String[] ignoreAttributeNames = of("provider", "monitor", "application", "module", "registry", "protocol", "interface");
	
	//添加属性值(后面会介绍该类)
        propertyValues.addPropertyValues(new AnnotationPropertyValuesAdapter(service, environment, ignoreAttributeNames));

        //为ServiceBean添加ref属性，属性值为annotatedServiceBeanName(即存在@Service注解的bean name)(后面会分析该方法)	
        addPropertyReference(builder, "ref", annotatedServiceBeanName);
        //为ServiceBean添加interface属性，属性值为interfaceClass接口的名称
        builder.addPropertyValue("interface", interfaceClass.getName());

	//获取@Service注解的provider属性(即com.alibaba.dubbo.config.ProviderConfig)
        String providerConfigBeanName = service.provider();
        if (StringUtils.hasText(providerConfigBeanName)) {
	    //为ServiceBean添加provider属性，属性值为providerConfigBeanName
            addPropertyReference(builder, "provider", providerConfigBeanName);
        }

        //获取@Service注解的monitor属性(即com.alibaba.dubbo.config.MonitorConfig)
	String monitorConfigBeanName = service.monitor();
        if (StringUtils.hasText(monitorConfigBeanName)) {
	    //为ServiceBean添加monitor属性，属性值为monitorConfigBeanName
            addPropertyReference(builder, "monitor", monitorConfigBeanName);
        }

	//获取@Service注解的application属性(即com.alibaba.dubbo.config.ApplicationConfig)
        String applicationConfigBeanName = service.application();
        if (StringUtils.hasText(applicationConfigBeanName)) {
	    //为ServiceBean添加application属性，属性值为applicationConfigBeanName
            addPropertyReference(builder, "application", applicationConfigBeanName);
        }

	//获取@Service注解的module属性(即com.alibaba.dubbo.config.ModuleConfig)
	String moduleConfigBeanName = service.module();
        if (StringUtils.hasText(moduleConfigBeanName)) {
	    //为ServiceBean添加module属性，属性值为moduleConfigBeanName
            addPropertyReference(builder, "module", moduleConfigBeanName);
        }


	//获取@Service注解的registry属性(即com.alibaba.dubbo.config.RegistryConfig)
        String[] registryConfigBeanNames = service.registry();
	//遍历registryConfigBeanNames，处理占位符，然后包装成RuntimeBeanReference并返回(后面会分析该方法)
        List<RuntimeBeanReference> registryRuntimeBeanReferences = toRuntimeBeanReferences(registryConfigBeanNames);

        if (!registryRuntimeBeanReferences.isEmpty()) {
            //为ServiceBean添加registries属性
	    builder.addPropertyValue("registries", registryRuntimeBeanReferences);
        }

        //获取@Service的protocol属性(即com.alibaba.dubbo.config.ProtocolConfig)
	String[] protocolConfigBeanNames = service.protocol();
	//处理占位符
        List<RuntimeBeanReference> protocolRuntimeBeanReferences = toRuntimeBeanReferences(protocolConfigBeanNames);

        if (!protocolRuntimeBeanReferences.isEmpty()) {
            //为ServiceBean添加protocols属性
	    builder.addPropertyValue("protocols", protocolRuntimeBeanReferences);
        }
	//返回@ServiceBean定义
        return builder.getBeanDefinition();
    }

	
    /**
     * 处理占位符，并包装成包装成RuntimeBeanReference对象
     * @param beanNames
     * @return
     */
    private ManagedList<RuntimeBeanReference> toRuntimeBeanReferences(String... beanNames) {
        ManagedList<RuntimeBeanReference> runtimeBeanReferences = new ManagedList<RuntimeBeanReference>();
        if (!ObjectUtils.isEmpty(beanNames)) {
            //遍历bean names
            for (String beanName : beanNames) {
		//解决占位符
                String resolvedBeanName = environment.resolvePlaceholders(beanName);
		//将beanName包装成RuntimeBeanReference对象
                runtimeBeanReferences.add(new RuntimeBeanReference(resolvedBeanName));
            }
        }
        return runtimeBeanReferences;
    }
    
    /**
     * 为ServiceBean添加propertyName属性，属性值为beanName
     * @param builder ServiceBean定义构造器
     * @param propertyName 属性名称
     * @param beanName 存在@Service直接的bean name
     */
    private void addPropertyReference(BeanDefinitionBuilder builder, String propertyName, String beanName) {
        //处理占位符
	String resolvedBeanName = environment.resolvePlaceholders(beanName);
        //为ServiceBean添加propertyName属性，属性值为resolvedBeanName
	builder.addPropertyReference(propertyName, resolvedBeanName);
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        this.classLoader = classLoader;
    }
}
```
我们再看下用到的其他类
##### DubboClassPathBeanDefinitionScanner
该类继承自ClassPathBeanDefinitionScanner,它提供自动扫描功能,根据提供的基础包路径,扫描classpath下该基础包路径,找到符合条件的类并注册为Spring的一个Bean,
默认情况下,ClassPathBeanDefinitionScanner将会扫描所有用Spring指定了的注解标识的类,包括@Component、@Service、@Repository、@Controller,
也可以对扫描的机制进行配置,设置一些Filter,只有满足Filter的类才能被注册为Bean.
```java
public class DubboClassPathBeanDefinitionScanner extends ClassPathBeanDefinitionScanner {

    public DubboClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry,
                                               boolean useDefaultFilters,
                                               Environment environment,
                                               ResourceLoader resourceLoader) {
        super(registry, useDefaultFilters);
        
	setEnvironment(environment);

        setResourceLoader(resourceLoader);

	//会调用AnnotationConfigUtils类的registerAnnotationConfigProcessors方法
        registerAnnotationConfigProcessors(registry);
    }
    public DubboClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry,
                                               Environment environment,
                                               ResourceLoader resourceLoader) {
        this(registry, false, environment, resourceLoader);
    }

    @Override
    public Set<BeanDefinitionHolder> doScan(String... basePackages) {
        return super.doScan(basePackages);
    }

    @Override
    public boolean checkCandidate(String beanName, BeanDefinition beanDefinition) throws IllegalStateException {
        //检测beanName是否已经存在
        return super.checkCandidate(beanName, beanDefinition);
    }
}
```
接下来我们看下AnnotationConfigUtils类的registerAnnotationConfigProcessors方法
```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, Object source) {

	//···省略···
	Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet(4);
	RootBeanDefinition def;

	//注册ConfigurationClassPostProcessor
	if (!registry.containsBeanDefinition("org.springframework.context.annotation.internalConfigurationAnnotationProcessor")) {
	    //1.在spring使用AnnotationConfigBeanDefinitionParser解析xml文件的时候  也就是配置annotation-config的时候
            //2.启动在AnnotationConfigApplicationContext容器的时候
	    def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
	    def.setSource(source);
	    beanDefs.add(registerPostProcessor(registry, def, "org.springframework.context.annotation.internalConfigurationAnnotationProcessor"));
	}
	
	//注册AutowiredAnnotationBeanPostProcessor 
	if (!registry.containsBeanDefinition("org.springframework.context.annotation.internalAutowiredAnnotationProcessor")) {
	    def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
	    def.setSource(source);
	    beanDefs.add(registerPostProcessor(registry, def, "org.springframework.context.annotation.internalAutowiredAnnotationProcessor"));
	}

	//···省略···
	return beanDefs;
}
```
##### AnnotationPropertyValuesAdapter
该类实现了Spring的PropertyValues接口，此接口是PropertyValue的集合管理类,用来储存键值对.MutablePropertyValues是其常用实现类
在该适配器内部就是用了MutablePropertyValues实现类来进行操作。
```java
//我们在上文构建ServiceBean定义的时候会创建AnnotationPropertyValuesAdapter类，相关代码如下：
//String[] ignoreAttributeNames = of("provider", "monitor", "application", "module", "registry", "protocol", "interface");
//propertyValues.addPropertyValues(new AnnotationPropertyValuesAdapter(service, environment, ignoreAttributeNames));
//其中service参数是@Service注解，而environment参数类实现了PropertyResolver接口


class AnnotationPropertyValuesAdapter implements PropertyValues {
     
    //传过来的注解
    private final Annotation annotation;
    
    //属性解决器,规范了解析底层任意property资源的接口
    private final PropertyResolver propertyResolver;

    //是否忽略默认值
    private final boolean ignoreDefaultValue;
	
    //委托类,即MutablePropertyValues
    private final PropertyValues delegate;

    public AnnotationPropertyValuesAdapter(Annotation annotation, PropertyResolver propertyResolver, boolean ignoreDefaultValue, String... ignoreAttributeNames) {
        this.annotation = annotation;
        this.propertyResolver = propertyResolver;
        this.ignoreDefaultValue = ignoreDefaultValue;
        //生成MutablePropertyValues
        this.delegate = adapt(annotation, ignoreDefaultValue, ignoreAttributeNames);
    }

    public AnnotationPropertyValuesAdapter(Annotation annotation, PropertyResolver propertyResolver, String... ignoreAttributeNames) {
        this(annotation, propertyResolver, true, ignoreAttributeNames);
    }

    private PropertyValues adapt(Annotation annotation, boolean ignoreDefaultValue, String... ignoreAttributeNames) {
        //创建MutablePropertyValues类(后面会分析该方法getAttributes()
	return new MutablePropertyValues(getAttributes(annotation, propertyResolver, ignoreDefaultValue, ignoreAttributeNames));
    }

    public Annotation getAnnotation() {
        return annotation;
    }

    public boolean isIgnoreDefaultValue() {
        return ignoreDefaultValue;
    }

    @Override
    public PropertyValue[] getPropertyValues() {
	//调用委托类的方法
        return delegate.getPropertyValues();
    }

    @Override
    public PropertyValue getPropertyValue(String propertyName) {
        //调用委托类的方法获取属性值
        return delegate.getPropertyValue(propertyName);
    }

    @Override
    public PropertyValues changesSince(PropertyValues old) {
        return delegate.changesSince(old);
    }

    @Override
    public boolean contains(String propertyName) {
        return delegate.contains(propertyName);
    }

    @Override
    public boolean isEmpty() {
        return delegate.isEmpty();
    }
}
```
下面我们看下AnnotationUtils.getAttributes方法
```java
/**
 * 获取注解的属性集合
 * @param annotation   注解
 * @param propertyResolver 属性解决器
 *    1、根据属性名称获取属性值（替换后的）
 *    2、替换${propertyName:defaultValue}格式的占位符为实际值
 * @param ignoreDefaultValue 是否忽略默认值
 * @param ignoreAttributeNames 需要忽略的属性名
 * @return <属性名，属性值>
 */
public static Map<String, Object> getAttributes(Annotation annotation,
					    PropertyResolver propertyResolver,
					    boolean ignoreDefaultValue,
					    String... ignoreAttributeNames) {
	
	//需要忽略的属性名
	Set<String> ignoreAttributeNamesSet = new HashSet<String>(arrayToList(ignoreAttributeNames));

	//获取annotation注解的属性map(调用的Spring的AnnotationUtils.getAnnotationAttributes方法)
	Map<String, Object> attributes = getAnnotationAttributes(annotation);
	//属性名,真实属性值
	Map<String, Object> actualAttributes = new LinkedHashMap<String, Object>();
	
	//是否需要处理占位符
	boolean requiredResolve = propertyResolver != null;

	for (Map.Entry<String, Object> entry : attributes.entrySet()) {
	    //属性名称
	    String attributeName = entry.getKey();
	    //属性值
	    Object attributeValue = entry.getValue();

	    //忽略默认属性值
	    if (ignoreDefaultValue && nullSafeEquals(attributeValue, getDefaultValue(annotation, attributeName))) {
		//属性值和默认值相等，则跳过该属性
		continue;
	    }

	    //忽略属性名
	    if (ignoreAttributeNamesSet.contains(attributeName)) {
		//如果待忽略的属性名列表包含该属性名，则跳过该属性
		continue;
	    }

	    // 处理占位符,属性值为字符串类型
	    if (requiredResolve && attributeValue instanceof String) {
		//获取真实属性值
		String resolvedValue = propertyResolver.resolvePlaceholders(valueOf(attributeValue));
		//格式化真实属性值
		attributeValue = trimAllWhitespace(resolvedValue);
	    }
	    //保存属性名和真实属性值
	    actualAttributes.put(attributeName, attributeValue);
	}
	return actualAttributes;
}
```

##### ServiceBean类
在上面的内容中，我们完成了ServiceBean类的注册，现在我们详细看看ServiceBean类.该类继承自ServiceConfig类，并且实现了众多Spring接口
```java
/**
 * ServiceFactoryBean
 * InitializingBean接口在bean实例化完成后将会自动调用afterPropertiesSet方法
 * DisposableBean接口在bean销毁之前调用destroy方法
 * ApplicationContextAware接口在bean实例化完成后将会注入ApplicationContext属性
 * ApplicationListener接口Spring容器初始化完成后会回调onApplicationEvent方法
 * BeanNameAware接口注入bean名称
 *
 * @export
 */
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean, ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware {

    private static final long serialVersionUID = 213195494150089726L;


    private static transient ApplicationContext SPRING_CONTEXT;

    /**
     * @Service注解
     */
    private final transient Service service;

    private transient ApplicationContext applicationContext;

    private transient String beanName;

    /**
     * 是否支持ApplicationListener
     */
    private transient boolean supportedApplicationListener;

    public ServiceBean() {
        super();
        this.service = null;
    }

    public ServiceBean(Service service) {
        super(service);
        this.service = service;
    }

    /**
     * 获取Spring ApplicationContext
     * @return
     */
    public static ApplicationContext getSpringContext() {
        return SPRING_CONTEXT;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        //设置ApplicationContext
        this.applicationContext = applicationContext;
        SpringExtensionFactory.addApplicationContext(applicationContext);
        if (applicationContext != null) {
            SPRING_CONTEXT = applicationContext;
            try {
                //backward compatibility to spring 2.0.1
                //从applicationContext中获取到addApplicationListener方法
                Method method = applicationContext.getClass().getMethod("addApplicationListener", new Class<?>[]{ApplicationListener.class});
                //调用applicationContext的addApplicationListener方法，将当前对象添加进去
                method.invoke(applicationContext, new Object[]{this});
                //设置支持ApplicationListener
                supportedApplicationListener = true;
            } catch (Throwable t) {
                if (applicationContext instanceof AbstractApplicationContext) {
                    try {
                        // 向后兼容
                        // backward compatibility to spring 2.0.1
                        Method method = AbstractApplicationContext.class.getDeclaredMethod("addListener", new Class<?>[]{ApplicationListener.class});
                        if (!method.isAccessible()) {
                            method.setAccessible(true);
                        }
                        method.invoke(applicationContext, new Object[]{this});
                        supportedApplicationListener = true;
                    } catch (Throwable t2) {
                    }
                }
            }
        }
    }

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
    }

    public Service getService() {
        return service;
    }

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
	//没有配置delay，或者delay = -1
        if (isDelay() && !isExported() && !isUnexported()) {
            if (logger.isInfoEnabled()) {
                logger.info("The service ready on spring started. service: " + getInterface());
            }
            //暴露服务(后面会分析该方法)
            export();
        }
    }

    /**
     * 两种情况：
     * 1、设置了延迟暴露(delay != null && delay != -1)，dubbo在Spring实例化bean的时候会对实现了InitializingBean的类进行回调，
     * 回调方法是afterPropertySet()，如果设置了延迟暴露，dubbo在这个方法中进行服务的发布。
     * 2、没有设置延迟或者延迟为-1，dubbo会在Spring实例化完bean之后，
     * 在刷新容器最后一步发布ContextRefreshEvent事件的时候，
     * 通知实现了ApplicationListener的类进行回调onApplicationEvent，dubbo会在这个方法中发布服务
     * @return
     */
    private boolean isDelay() {
        //获取delay属性
        Integer delay = getDelay();
        //获取服务提供者
        ProviderConfig provider = getProvider();
        if (delay == null && provider != null) {
            //delay属性为空，则取服务提供者的delay属性
            delay = provider.getDelay();
        }
        //(支持spring监听事件 && 没有设置延迟或者延迟为-1) 则返回true
        return supportedApplicationListener && (delay == null || delay == -1);
    }

	
    /**
     * 1、会判断ServiceBean的ProviderConfig、ApplicationConfig、ModuleConfig、List<RegistryConfig>、MonitorConfig、List<ProtocolConfig>属性是否为空，为空的话，从Spring容器中获取，然后进行赋值，其中还会检测是否存在重复的配置(default属性)
     * 2、接着会判断并设置path属性(使用beanName)
     * 3、根据delay属性判断是否需要暴露服务
     */
    @Override
    @SuppressWarnings({"unchecked", "deprecation"})
    public void afterPropertiesSet() throws Exception {
        //没有配置Provider
        if (getProvider() == null) {
            //从IOC容器中获取到所有的Provider
            Map<String, ProviderConfig> providerConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProviderConfig.class, false, false);
            if (providerConfigMap != null && providerConfigMap.size() > 0) {
                //从IOC容器中获取到所有的Protocol
                Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
                if ((protocolConfigMap == null || protocolConfigMap.size() == 0)
                        && providerConfigMap.size() > 1) {
                    // backward compatibility
                    //如果没有配置Protocol，但是存在Provider的话，遍历Provider列表
                    //从Provider列表中找到isDefault属性为true的Provider，并保存起来
                    List<ProviderConfig> providerConfigs = new ArrayList<ProviderConfig>();
                    for (ProviderConfig config : providerConfigMap.values()) {
                        if (config.isDefault() != null && config.isDefault().booleanValue()) {
                            //找到default为true的Provider
                            providerConfigs.add(config);
                        }
                    }
                    if (!providerConfigs.isEmpty()) {
                        //根据Provider构造Protocol(废弃方法)
                        setProviders(providerConfigs);
                    }
                } else {
                    ProviderConfig providerConfig = null;
                    //从provider列表中找到default属性为null或者为true的provider，如果找到多个则抛异常
                    for (ProviderConfig config : providerConfigMap.values()) {
                        //没有配置isDefault属性或者isDefault = true
                        //检测是否存在重复的Provider配置
                        if (config.isDefault() == null || config.isDefault().booleanValue()) {
                            if (providerConfig != null) {
                                throw new IllegalStateException("Duplicate provider configs: " + providerConfig + " and " + config);
                            }
                            providerConfig = config;
                        }
                    }
                    if (providerConfig != null) {
                        //设置Provider
                        setProvider(providerConfig);
                    }
                }
            }
        }
        if (getApplication() == null
                && (getProvider() == null || getProvider().getApplication() == null)) {
            //从IOC容器中找到所有的ApplicationConfig
            Map<String, ApplicationConfig> applicationConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ApplicationConfig.class, false, false);
            if (applicationConfigMap != null && applicationConfigMap.size() > 0) {
                ApplicationConfig applicationConfig = null;
                //检测是否存在重复的Application配置(default属性为null或者为true)
                for (ApplicationConfig config : applicationConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        if (applicationConfig != null) {
                            throw new IllegalStateException("Duplicate application configs: " + applicationConfig + " and " + config);
                        }
                        applicationConfig = config;
                    }
                }
                if (applicationConfig != null) {
                    setApplication(applicationConfig);
                }
            }
        }
        if (getModule() == null
                && (getProvider() == null || getProvider().getModule() == null)) {
            //从IOC容器中找到所有的ModuleConfig
            Map<String, ModuleConfig> moduleConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ModuleConfig.class, false, false);
            if (moduleConfigMap != null && moduleConfigMap.size() > 0) {
                ModuleConfig moduleConfig = null;
                //检测是否存在重复的Module配置(default属性为null或者为true)
                for (ModuleConfig config : moduleConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        if (moduleConfig != null) {
                            throw new IllegalStateException("Duplicate module configs: " + moduleConfig + " and " + config);
                        }
                        moduleConfig = config;
                    }
                }
                if (moduleConfig != null) {
                    setModule(moduleConfig);
                }
            }
        }
        //register为空或者Provider中的register为空或者Application中的register为空
        if ((getRegistries() == null || getRegistries().isEmpty())
                && (getProvider() == null || getProvider().getRegistries() == null || getProvider().getRegistries().isEmpty())
                && (getApplication() == null || getApplication().getRegistries() == null || getApplication().getRegistries().isEmpty())) {
            //从IOC容器中获取RegistryConfig
            Map<String, RegistryConfig> registryConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, RegistryConfig.class, false, false);
            if (registryConfigMap != null && registryConfigMap.size() > 0) {
                List<RegistryConfig> registryConfigs = new ArrayList<RegistryConfig>();
                //遍历registry列表，从中找到default属性为null或者为true的registry，并保存起来
                for (RegistryConfig config : registryConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        registryConfigs.add(config);
                    }
                }
                if (registryConfigs != null && !registryConfigs.isEmpty()) {
                    super.setRegistries(registryConfigs);
                }
            }
        }
        if (getMonitor() == null
                && (getProvider() == null || getProvider().getMonitor() == null)
                && (getApplication() == null || getApplication().getMonitor() == null)) {
            Map<String, MonitorConfig> monitorConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, MonitorConfig.class, false, false);
            if (monitorConfigMap != null && monitorConfigMap.size() > 0) {
                MonitorConfig monitorConfig = null;
                //检测是否存在重复的monitor配置(default属性为空或者为true)
                for (MonitorConfig config : monitorConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        if (monitorConfig != null) {
                            throw new IllegalStateException("Duplicate monitor configs: " + monitorConfig + " and " + config);
                        }
                        monitorConfig = config;
                    }
                }
                if (monitorConfig != null) {
		    //设置monitor
                    setMonitor(monitorConfig);
                }
            }
        }
        if ((getProtocols() == null || getProtocols().isEmpty())
                && (getProvider() == null || getProvider().getProtocols() == null || getProvider().getProtocols().isEmpty())) {
            //从IOC容器中获取ProtocolConfig
            Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
            if (protocolConfigMap != null && protocolConfigMap.size() > 0) {
                List<ProtocolConfig> protocolConfigs = new ArrayList<ProtocolConfig>();
                //遍历Protocol列表，从中找到default属性为空，或者为true的Protocol，并保存起来
                for (ProtocolConfig config : protocolConfigMap.values()) {
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        protocolConfigs.add(config);
                    }
                }
                if (protocolConfigs != null && !protocolConfigs.isEmpty()) {
                    super.setProtocols(protocolConfigs);
                }
            }
        }
        if (getPath() == null || getPath().length() == 0) {
            //path服务名称为空的话
            if (beanName != null && beanName.length() > 0
                    && getInterface() != null && getInterface().length() > 0
                    && beanName.startsWith(getInterface())) {
                //使用beanName作为服务名称
                setPath(beanName);
            }
        }
        if (!isDelay()) {
            //配置了delay属性，暴露服务(后面会分析该方法)
            export();
        }
    }

    @Override
    public void destroy() throws Exception {
        // This will only be called for singleton scope bean, and expected to be called by spring shutdown hook when BeanFactory/ApplicationContext destroys.
        // We will guarantee dubbo related resources being released with dubbo shutdown hook.
        //unexport();
    }

    // merged from dubbox
    @Override
    protected Class getServiceClass(T ref) {
        if (AopUtils.isAopProxy(ref)) {
            //从Aop代理类中获取到目标对象
            return AopUtils.getTargetClass(ref);
        }
        return super.getServiceClass(ref);
    }
}
```
###### export方法
export方法是定义在ServiceBean的父类ServiceConfig中的
```java
public synchronized void export() {
	if (provider != null) {
	    if (export == null) {
		//export属性为空的话，则获取provider中的export属性
		export = provider.getExport();
	    }
	    if (delay == null) {
		//delay属性为空的话，则获取provider中的delay属性
		delay = provider.getDelay();
	    }
	}
	if (export != null && !export) {
	    //如果不暴露服务，直接返回
	    return;
	}

	if (delay != null && delay > 0) {
	    //delay大于0的话，会启动线程，延迟暴露服务
	    delayExportExecutor.schedule(new Runnable() {
		@Override
		public void run() {
		    doExport();
		}
	    }, delay, TimeUnit.MILLISECONDS);
	} else {
	    //直接暴露服务
	    doExport();
	}
}
```
可以看到最终会调用doExport方法进行服务暴露
```java
protected synchronized void doExport() {
	if (unexported) {
	    //检测是否已经取消服务暴露
	    throw new IllegalStateException("Already unexported!");
	}
	if (exported) {
	    //已经暴露过服务，直接返回
	    return;
	}
	exported = true;
	//检查服务接口interface属性是否配置
	if (interfaceName == null || interfaceName.length() == 0) {
	    throw new IllegalStateException("<dubbo:service interface=\"\" /> interface not allow null!");
	}
	//检测ProviderConfig是否已配置，没有配置则进行配置(此方法在"Dubbo源码阅读之集成Spring(03)"中介绍过)
	checkDefault();
	if (provider != null) {
	    //从provider中获取缺失的配置
	    if (application == null) {
		application = provider.getApplication();
	    }
	    if (module == null) {
		module = provider.getModule();
	    }
	    if (registries == null) {
		registries = provider.getRegistries();
	    }
	    if (monitor == null) {
		monitor = provider.getMonitor();
	    }
	    if (protocols == null) {
		protocols = provider.getProtocols();
	    }
	}
	if (module != null) {
	    //从module中获取注册中心和监控中心
	    if (registries == null) {
		registries = module.getRegistries();
	    }
	    if (monitor == null) {
		monitor = module.getMonitor();
	    }
	}
	if (application != null) {
	    //从application中获取注册中心和监控中心
	    if (registries == null) {
		registries = application.getRegistries();
	    }
	    if (monitor == null) {
		monitor = application.getMonitor();
	    }
	}
	//获取服务接口类interfaceClass
	if (ref instanceof GenericService) {
	    //服务接口为GenericService类型
	    interfaceClass = GenericService.class;
	    if (StringUtils.isEmpty(generic)) {
		//generic属性为空的话，则设置为true
		generic = Boolean.TRUE.toString();
	    }
	} else {
	    try {
		//加载interfaceName类
		interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
			.getContextClassLoader());
	    } catch (ClassNotFoundException e) {
		throw new IllegalStateException(e.getMessage(), e);
	    }
	    //检测方法列表methods是否都在接口interfaceClass中存在
	    checkInterfaceAndMethods(interfaceClass, methods);
	    //校验ref类(ref不可以为空，并且实现了interfaceClass接口)
	    checkRef();
	    //将generic属性设置成false
	    generic = Boolean.FALSE.toString();
	}
	//处理服务接口本地实现类
	if (local != null) {
	    if ("true".equals(local)) {
		//默认情况下，实现类名称为：interfaceName + "Local"
		local = interfaceName + "Local";
	    }
	    Class<?> localClass;
	    try {
		//加载实现类
		localClass = ClassHelper.forNameWithThreadContextClassLoader(local);
	    } catch (ClassNotFoundException e) {	
		//没有找到该实现类
		throw new IllegalStateException(e.getMessage(), e);
	    }
	    //校验服务接口本地实现类是否实现了interfaceClass接口
	    if (!interfaceClass.isAssignableFrom(localClass)) {
		throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceName);
	    }
	}
	//处理服务接口本地存根实现类
	if (stub != null) {
	    if ("true".equals(stub)) {
		//默认情况下，实现类名称为：interfaceName + "Stub"
		stub = interfaceName + "Stub";
	    }
	    Class<?> stubClass;
	    try {
		//加载实现类
		stubClass = ClassHelper.forNameWithThreadContextClassLoader(stub);
	    } catch (ClassNotFoundException e) {
		throw new IllegalStateException(e.getMessage(), e);
	    }
	    //校验该存根类是否实现了interfaceClass接口
	    if (!interfaceClass.isAssignableFrom(stubClass)) {
		throw new IllegalStateException("The stub implementation class " + stubClass.getName() + " not implement interface " + interfaceName);
	    }
	}
	//检测ApplicationConfig是否为空(会调用appendProperties方法添加属性，该方法在之前博文介绍过)
	checkApplication();
	//检测注册中心list是否为空(从配置文件中获取dubbo.registry.address属性，然后创建RegistryConfig对象，添加属性，放入到list中)
	checkRegistry();
	//检测协议protocols是否为空，如果为空且provider不为空，则先从provider对象中获取
	//然后遍历protocols，处理name(name为空，则设置为dubbo)并添加属性。
	checkProtocol();
	//为ServiceBean对象添加属性
	appendProperties(this);
	//检测local/stub/mock配置是否正确
	//local/stub/mock(mock 可配置为"return ")应该为interfaceClass或者为interfaceClass子类
	checkStubAndMock(interfaceClass);
	//服务名称path为空的话，则设置为interfaceName
	if (path == null || path.length() == 0) {
	    path = interfaceName;
	}
	//暴露url(后面会分析该方法)
	doExportUrls();
	//根据服务唯一名称、当前ServiceBean实例、ref 创建ProviderModel实例(后面会分析)
	ProviderModel providerModel = new ProviderModel(getUniqueServiceName(), this, ref);
	//根据服务唯一名称，注册提供者服务，即放到类ApplicationModel中的变量名为providedServices的map中
	ApplicationModel.initProviderModel(getUniqueServiceName(), providerModel);
}
```

###### doExportUrls方法
该方法内部会调用loadRegistries方法构造注册中心url地址，然后调用doExportUrlsFor1Protocol方法进行服务url的暴露。
```java
/**
 * 暴露url
 *
private void doExportUrls() {
	//构造注册中心地址(后面会分析该方法)
	//例如：registry://224.5.6.7:1234/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&pid=6384&qos.port=22222&registry=multicast&timestamp=1528347455956
	List<URL> registryURLs = loadRegistries(true);
	
	//例如：<dubbo:protocol name="dubbo" port="20880" id="dubbo" />
	for (ProtocolConfig protocolConfig : protocols) {
	    //暴露服务url	
	    doExportUrlsFor1Protocol(protocolConfig, registryURLs);
	}
}

/**
 * 构造注册中心URL
 * 优先使用系统配置中的注册中心列表，如果没有配置，则使用RegistryConfig中配置的
 * @param provider 是否是提供者
 * @return
 */
protected List<URL> loadRegistries(boolean provider) {
	//检测registries变量(即ArrayList<RegistryConfig>。<dubbo:registry address="..." />）
	checkRegistry();
	List<URL> registryList = new ArrayList<URL>();
	//遍历registries
	if (registries != null && !registries.isEmpty()) {
	    for (RegistryConfig config : registries) {
		//获取当前注册中心地址
		String address = config.getAddress();
		if (address == null || address.length() == 0) {
		    //将address设置为*
		    address = Constants.ANYHOST_VALUE;
		}
		//从系统属性中获取注册中心地址
		String sysaddress = System.getProperty("dubbo.registry.address");
		if (sysaddress != null && sysaddress.length() > 0) {
		    //如果系统注册中心地址不为空，则优先使用系统注册中心地址
		    address = sysaddress;
		}
		if (address != null && address.length() > 0
			&& !RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(address)) {
		    //地址可用

		    Map<String, String> map = new HashMap<String, String>();
		    //附加参数，即找到application、config类中的属性，并添加到map中
		    appendParameters(map, application);
		    appendParameters(map, config);
		    //添加服务名称
		    map.put("path", RegistryService.class.getName());
		    //添加dubbo版本
		    map.put("dubbo", Version.getVersion());
		    //添加时间戳
		    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
		    if (ConfigUtils.getPid() > 0) {
			//添加pid
			map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
		    }
		    //添加protocol
		    if (!map.containsKey("protocol")) {
			//存在扩展名称为remote的RegistryFactory，则设置protocol属性为remote，否则设置为dubbo
			if (ExtensionLoader.getExtensionLoader(RegistryFactory.class).hasExtension("remote")) {
			    map.put("protocol", "remote");
			} else {
			    map.put("protocol", "dubbo");
			}
		    }
		    //根据当前注册中心地址和map生产url列表（后面会分析该方法）
		    List<URL> urls = UrlUtils.parseURLs(address, map);
		    for (URL url : urls) {
			//向url中添加registry参数，参数值为url的protocol属性(例如：registry=multicast)
			url = url.addParameter(Constants.REGISTRY_KEY, url.getProtocol());
			//重新设置url的protocol属性值为“registry”(后面暴露服务时，将会使用该protocol属性值作为扩展名称，获取对应的Protocol实例，因此将会使用RegistryProtocol实例，下一节会详细分析)
			url = url.setProtocol(Constants.REGISTRY_PROTOCOL);
			if ((provider && url.getParameter(Constants.REGISTER_KEY, true))
				|| (!provider && url.getParameter(Constants.SUBSCRIBE_KEY, true))) {
			    //提供者 && url中的register参数值为true
			    //不是提供者 && url中的subscribe参数值为true
			    //则将该url添加到registryList列表中
			    registryList.add(url);
			}
		    }
		}
	    }
	}
	return registryList;
}

/**
 * @param address 注册中心地址列表("|"或者";"分隔)
 * @param defaults map参数
 * @return
 */
public static List<URL> parseURLs(String address, Map<String, String> defaults) {
        if (address == null || address.length() == 0) {
	    //注册中心地址为空，返回null
            return null;
        }
        //通过“|”或者“;”分隔注册中心地址
        String[] addresses = Constants.REGISTRY_SPLIT_PATTERN.split(address);
        if (addresses == null || addresses.length == 0) {
            return null;
        }
        List<URL> registries = new ArrayList<URL>();
	//遍历每一个注册中心地址
        for (String addr : addresses) {
            //通过注册中心地址addr和map参数构造注册中心URL对象（后面会分析方法）
            registries.add(parseURL(addr, defaults));
        }
	//返回注册中心地址url列表
        return registries;
}

/**
 * 生成注册中心URL
 * 1、根据注册中心地址address生成url字符串(处理备用地址)
 * 2、从defaults集合中获取属性(protocol、username、password、port、path)默认值
 * 3、根据defaults新生成集合defaultParameters，并移除(protocol、username、password、port、path)属性
 * 4、根据url字符串生成URL对象，并获取该URL对象的属性值(protocol、username、password、port、path、parameters),
 *    其中parameters属性值是新生成的一个map。
 * 5、判断URL对象的这些属性值(protocol、username、password、port、path、parameters)是否为空，如果为空，则使用属性默认值，并标识发生了改变。
 * 6、如果发生了改变。则根据新的属性值重新生成一个URL对象并返回。
 * @param address 注册中心地址(可能是多个，使用","分隔)
 * @param defaults map参数
 * @return
 */
public static URL parseURL(String address, Map<String, String> defaults) {
	if (address == null || address.length() == 0) {
	    return null;
	}
	//该url最终会被解析成URL对象
	String url;
	if (address.indexOf("://") >= 0) {
	    //该注册中心地址包含"://"
	    url = address;
	} else {
	    //使用逗号","分隔address地址
	    String[] addresses = Constants.COMMA_SPLIT_PATTERN.split(address);
	    //将addresses数组第1个元素赋值给url
	    url = addresses[0];
	    if (addresses.length > 1) {
		//遍历addresses数组后面的元素(即查看是否存在备用的注册中心地址)
		//backup记录注册中心备用地址
		StringBuilder backup = new StringBuilder();
		for (int i = 1; i < addresses.length; i++) {
		    if (i > 1) {
			backup.append(",");
		    }
		    backup.append(addresses[i]);
		}
		//将注册中心备用地址添加到url的backup参数中
		url += "?" + Constants.BACKUP_KEY + "=" + backup.toString();
	    }
	}
	//从defaults中获取protocol属性
	String defaultProtocol = defaults == null ? null : defaults.get("protocol");
	if (defaultProtocol == null || defaultProtocol.length() == 0) {
	    //如果protocol属性值为空，则设置为dubbo
	    defaultProtocol = "dubbo";
	}
	//从defaults中获取username属性和password属性
	String defaultUsername = defaults == null ? null : defaults.get("username");
	String defaultPassword = defaults == null ? null : defaults.get("password");
	//从defaults中获取port属性
	int defaultPort = StringUtils.parseInteger(defaults == null ? null : defaults.get("port"));
	//从defaults中获取path属性
	String defaultPath = defaults == null ? null : defaults.get("path");
	//根据defaults新生成一个map集合defaultParameters
	Map<String, String> defaultParameters = defaults == null ? null : new HashMap<String, String>(defaults);
	if (defaultParameters != null) {
	    //移除defaultParameters集合中的以下属性
	    defaultParameters.remove("protocol");
	    defaultParameters.remove("username");
	    defaultParameters.remove("password");
	    defaultParameters.remove("host");
	    defaultParameters.remove("port");
	    defaultParameters.remove("path");
	}
	//根据url字符串生成URL对象
	URL u = URL.valueOf(url);
	//如果新生成的URL中的某属性值为空，且该属性的默认值不为空，则意味着发生了改变，changed会被设置为true
	boolean changed = false;
	
	//获取URL对象中的protocol、username、password、host、port、path属性
	String protocol = u.getProtocol();
	String username = u.getUsername();
	String password = u.getPassword();
	String host = u.getHost();
	int port = u.getPort();
	String path = u.getPath();
	
	//获取URL对象的参数map
	Map<String, String> parameters = new HashMap<String, String>(u.getParameters());
	if ((protocol == null || protocol.length() == 0) && defaultProtocol != null && defaultProtocol.length() > 0) {
	    changed = true;
	    //使用protocol默认值设置URL的protocol属性
	    protocol = defaultProtocol;
	}
	if ((username == null || username.length() == 0) && defaultUsername != null && defaultUsername.length() > 0) {
	    changed = true;
	    //使用username默认值设置URL的username属性
	    username = defaultUsername;
	}
	if ((password == null || password.length() == 0) && defaultPassword != null && defaultPassword.length() > 0) {
	    changed = true;
	    //使用password默认值设置URL的password属性
	    password = defaultPassword;
	}
	/*if (u.isAnyHost() || u.isLocalHost()) {
	    changed = true;
	    host = NetUtils.getLocalHost();
	}*/
	if (port <= 0) {
	    //URL的port属性值小于0，且默认的port值大于0，则使用defaultPort设置URL的port属性
	    if (defaultPort > 0) {
		changed = true;
		port = defaultPort;
	    } else {
		//默认port如果也小于0的话，则设置URL的port属性为9090
		changed = true;
		port = 9090;
	    }
	}
	if (path == null || path.length() == 0) {
	    //使用默认服务名称设置URL的path属性
	    if (defaultPath != null && defaultPath.length() > 0) {
		changed = true;
		path = defaultPath;
	    }
	}
	//遍历默认参数集合
	if (defaultParameters != null && defaultParameters.size() > 0) {
	    for (Map.Entry<String, String> entry : defaultParameters.entrySet()) {
		//默认参数key
		String key = entry.getKey();
		//默认参数值defaultValue
		String defaultValue = entry.getValue();
		if (defaultValue != null && defaultValue.length() > 0) {
		    //查看URL参数集合中该参数key对应的值
		    String value = parameters.get(key);
		    if (value == null || value.length() == 0) {
			//如果URL中的参数值为空，则使用该参数key的默认值defaultValue
			changed = true;
			parameters.put(key, defaultValue);
		    }
		}
	    }
	}
	if (changed) {
	    //当前URL的属性值发送改变了，则重新生成一个URL对象
	    u = new URL(protocol, username, password, host, port, path, parameters);
	}
	return u;
}
```
加载注册中心URL地址的方法分析完了，我们再来看doExportUrlsFor1Protocol方法
```java
/**
 * 暴露服务Url到各个注册中心
 * 1、map装配参数
 * 2、利用map中的参数构建URL，为暴露服务做准备
 * 3、根据范围选择是暴露本地服务，还是暴露远程服务
 * 4、根据代理工厂生成服务代理对象invoker
 * 5、根据配置的协议，暴露服务代理
 * @param protocolConfig 例如: <dubbo:protocol name="dubbo" port="20880" id="dubbo" />
 * @param registryURLs 例如: registry://224.5.6.7:1234/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.0&pid=4328&qos.port=22222&registry=multicast&timestamp=1528278313174
 */
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
	//获取协议名称
	String name = protocolConfig.getName();
	if (name == null || name.length() == 0) {
	    name = "dubbo";
	}
	Map<String, String> map = new HashMap<String, String>();
	//添加side参数，值为provider，标识服务提供端
	map.put(Constants.SIDE_KEY, Constants.PROVIDER_SIDE);
	//添加dubbo版本
	map.put(Constants.DUBBO_VERSION_KEY, Version.getVersion());
	//添加时间戳
	map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
	if (ConfigUtils.getPid() > 0) {
	    //添加pid
	    map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
	}
	//添加参数到map
	appendParameters(map, application);
	appendParameters(map, module);
	appendParameters(map, provider, Constants.DEFAULT_KEY);
	appendParameters(map, protocolConfig);
	//当前ServiceConfig(ServiceBean)
	appendParameters(map, this);

	if (methods != null && !methods.isEmpty()) {
	    //遍历服务方法
	    for (MethodConfig method : methods) {
		//将method的属性添加到map中
		appendParameters(map, method, method.getName());
		//设置方法重试次数(retryKey是map的key)
		String retryKey = method.getName() + ".retry";
		if (map.containsKey(retryKey)) {
		    String retryValue = map.remove(retryKey);
		    if ("false".equals(retryValue)) {
		        //retryValue值为false的话，说明不启用重试
			map.put(method.getName() + ".retries", "0");
		    }
		}
		//获取当前服务方法的参数
		List<ArgumentConfig> arguments = method.getArguments();
		if (arguments != null && !arguments.isEmpty()) {
		    //遍历当前服务方法的参数配置
		    for (ArgumentConfig argument : arguments) {
			if (argument.getType() != null && argument.getType().length() > 0) {
			    //遍历服务接口的方法列表
			    Method[] methods = interfaceClass.getMethods();
			    if (methods != null && methods.length > 0) {
				for (int i = 0; i < methods.length; i++) {
				    //方法名
				    String methodName = methods[i].getName();
				    if (methodName.equals(method.getName())) {
					//方法名称一样,获取方法参数类型数组
					Class<?>[] argtypes = methods[i].getParameterTypes();
					// one callback in the method
					if (argument.getIndex() != -1) {
					    //校验参数的类型是否匹配
					    if (argtypes[argument.getIndex()].getName().equals(argument.getType())) {
						//将方法的参数的属性添加到map，前缀为：方法名.参数索引
						appendParameters(map, argument, method.getName() + "." + argument.getIndex());
					    } else {
						//参数配置错误，索引和类型不匹配
						throw new IllegalArgumentException("argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
					    }
					} else {
					    // multiple callbacks in the method
					    for (int j = 0; j < argtypes.length; j++) {
						//当前参数类型
						Class<?> argclazz = argtypes[j];
						//参数类型名称匹配
						if (argclazz.getName().equals(argument.getType())) {
						    appendParameters(map, argument, method.getName() + "." + j);
						    if (argument.getIndex() != -1 && argument.getIndex() != j) {
							throw new IllegalArgumentException("argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
						    }
						}
					    }
					}
				    }
				}
			    }
			} else if (argument.getIndex() != -1) {
			    appendParameters(map, argument, method.getName() + "." + argument.getIndex());
			} else {
			    //<dubbo:argument index='0' .../> or <dubbo:argument type=xxx .../>
			    throw new IllegalArgumentException("argument config must set index or type attribute.eg: <dubbo:argument index='0' .../> or <dubbo:argument type=xxx .../>");
			}

		    }
		}
	    } // end of methods for
	}
	//(generic ！= null) && (generic = “true” || "nativejava" || "bean") 返回 true
	if (ProtocolUtils.isGeneric(generic)) {
	    //添加generic参数
	    map.put(Constants.GENERIC_KEY, generic);
	    //添加methods参数，值为"*"
	    map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
	} else {
	    String revision = Version.getVersion(interfaceClass, version);
	    if (revision != null && revision.length() > 0) {
	        //添加版本
		map.put("revision", revision);
	    }
	    //生成interfaceClass类的包装类，获取方法名称
	    String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
	    if (methods.length == 0) {
		//服务接口中没有发现方法定义
		logger.warn("NO method found in service interface " + interfaceClass.getName());
		//添加methods参数，值为"*"
		map.put(Constants.METHODS_KEY, Constants.ANY_VALUE);
	    } else {
		//添加methods参数，值为逗号分隔的方法名称
		map.put(Constants.METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
	    }
	}
	if (!ConfigUtils.isEmpty(token)) {
	    //token为true或者token为default，则isDefault方法返回true
	    if (ConfigUtils.isDefault(token)) {
		//添加token参数，值为uuid
		map.put(Constants.TOKEN_KEY, UUID.randomUUID().toString());
	    } else {
		//添加token参数
		map.put(Constants.TOKEN_KEY, token);
	    }
	}
	//判断协议名称是否为"injvm"
	if (Constants.LOCAL_PROTOCOL.equals(protocolConfig.getName())) {
	    //本地协议，不注册
	    protocolConfig.setRegister(false);
	    //添加notify参数，值为false
	    map.put("notify", "false");
	}
	//暴露服务
	//获取上下文地址
	String contextPath = protocolConfig.getContextpath();
	if ((contextPath == null || contextPath.length() == 0) && provider != null) {
	    //获取provider对象配置的上下文地址
	    contextPath = provider.getContextpath();
	}
	//获取注册ip(后面会分析该方法)
	String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
	//获取注册端口(后面会分析该方法)
	Integer port = this.findConfigedPorts(protocolConfig, name, map);
	//根据协议名称name、主机host、端口port、上下文、map参数构建URL对象(服务暴露的url地址)
	//例如url：dubbo://192.168.99.60:20880/com.alibaba.dubbo.demo.DemoService?anyhost=true&application=demo-provider&bind.ip=192.168.99.60&bind.port=20880&dubbo=2.0.0&generic=false&interface=com.alibaba.dubbo.demo.DemoService&methods=sayHello&pid=7184&qos.port=22222&side=provider&timestamp=1528347825839
	URL url = new URL(name, host, port, (contextPath == null || contextPath.length() == 0 ? "" : contextPath + "/") + path, map);
	//查看ConfiguratorFactory(override/absent)是否存在url.getProtocol()扩展，存在的话，则配置该url对象
	//override会覆盖参数配置，absent只有参数不存在时才会添加(后面的章节会介绍该扩展)
	if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class).hasExtension(url.getProtocol())) {
	    url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
		    .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
	}
	//获取url的scope参数
	String scope = url.getParameter(Constants.SCOPE_KEY);
	
	//只有scope != none 时才会暴露服务，scope = null时会同时暴露本地服务和远程服务
	if (!Constants.SCOPE_NONE.toString().equalsIgnoreCase(scope)) {
	    //scope != remote 暴露本地服务
	    if (!Constants.SCOPE_REMOTE.toString().equalsIgnoreCase(scope)) {
		//本地暴露(后面小节会分析该方法)
		exportLocal(url);
	    }
	    if (!Constants.SCOPE_LOCAL.toString().equalsIgnoreCase(scope)) {
	         //scope != local 暴露到远程
		if (logger.isInfoEnabled()) {
		    //暴露dubbo服务interfaceClass到url
		    logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
		}
		if (registryURLs != null && !registryURLs.isEmpty()) {
		    //注册中心地址registryURLs不为空，则遍历注册中心地址，将服务暴露到各个注册中心
		    for (URL registryURL : registryURLs) {
			
			//添加dynamic参数到url中，参数值从当前注册中心的参数中获取
			url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
			
			//构造监控url(后面会分析该方法)
			URL monitorUrl = loadMonitor(registryURL);
			if (monitorUrl != null) {
			    //添加monitor参数(编码)到url中
			    url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
			}
			
			if (logger.isInfoEnabled()) {
			    //注册dubbo服务 interfaceClass的url 到注册中心registryURL上
			    logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
			}
			
			//registryURL地址上添加export参数，参数值为服务暴露的url(即interfaceClass的url)
			//根据ref、interfaceClass、registryURL创建服务代理对象Invoker(后面小节会分析该方法)
			Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
			
			//将invoker和当前对象this包装成DelegateProviderMetaDataInvoker对象(后面小节会分析该方法)
			DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
			
			//在此处开始暴露服务
			//调用export方法暴露服务，得到exporter对象(后面小节会分析该方法)
			Exporter<?> exporter = protocol.export(wrapperInvoker);
			//保存暴露的服务到本地变量exporters中
			exporters.add(exporter);
		    }
		} else {
		    //注册中心地址registryURLs为空
		    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
		    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
		    Exporter<?> exporter = protocol.export(wrapperInvoker);
		    exporters.add(exporter);
		}
	    }
	}
	//保存暴露的服务url
	this.urls.add(url);
}
```
我们依次看下上面用到的方法: findConfigedHosts、findConfigedPorts、loadMonitor
exportLocal、getInvoker、export方法将在后面的章节(Dubbo源码阅读之服务暴露)进行详细介绍。
```java
/**
 * 为服务提供者 获取注册ip和绑定ip，可以单独配置
 * Register & bind IP address for service provider, can be configured separately.
 * Configuration priority:
 * environment variables -> java system properties -> host property in config file ->
 * /etc/hosts -> default network address -> first available network address
 * @param protocolConfig
 * @param registryURLs
 * @param map
 * @return 注册ip
 */
private String findConfigedHosts(ProtocolConfig protocolConfig, List<URL> registryURLs, Map<String, String> map) {
	boolean anyhost = false;
	//从系统配置中获取获取绑定ip
	String hostToBind = getValueFromConfig(protocolConfig, Constants.DUBBO_IP_TO_BIND);
	if (hostToBind != null && hostToBind.length() > 0 && isInvalidLocalHost(hostToBind)) {
	    //无效的ip
	    throw new IllegalArgumentException("Specified invalid bind ip from property:" + Constants.DUBBO_IP_TO_BIND + ", value:" + hostToBind);
	}
	if (hostToBind == null || hostToBind.length() == 0) {
	    //从协议配置中获取服务ip地址
	    hostToBind = protocolConfig.getHost();
	    if (provider != null && (hostToBind == null || hostToBind.length() == 0)) {
		//从服务提供者中获取服务ip地址
		hostToBind = provider.getHost();
	    }
	    if (isInvalidLocalHost(hostToBind)) {
		//仍然是无效的地址，则设置anyhost=true
		anyhost = true;
		try {
		    //获取本地地址
		    hostToBind = InetAddress.getLocalHost().getHostAddress();
		} catch (UnknownHostException e) {
		    logger.warn(e.getMessage(), e);
		}
		if (isInvalidLocalHost(hostToBind)) {
		    //仍然是无效的地址，遍历注册中心url列表
		    if (registryURLs != null && !registryURLs.isEmpty()) {
			for (URL registryURL : registryURLs) {
			    if (Constants.MULTICAST.equalsIgnoreCase(registryURL.getParameter("registry"))) {
				// 跳过组播registry，因为我们不可以通过Socket连接到它
				continue;
			    }
			    try {
				Socket socket = new Socket();
				try {
				    //根据注册中心host和port构建SocketAddress
				    SocketAddress addr = new InetSocketAddress(registryURL.getHost(), registryURL.getPort());
				    //连接到该地址
				    socket.connect(addr, 1000);
				    //连接成功的话，则获取到绑定地址，并跳出循环
				    hostToBind = socket.getLocalAddress().getHostAddress();
				    break;
				} finally {
				    try {
					socket.close();
				    } catch (Throwable e) {
				    }
				}
			    } catch (Exception e) {
				logger.warn(e.getMessage(), e);
			    }
			}
		    }
		    if (isInvalidLocalHost(hostToBind)) {
			//仍然为无效本地地址的话，则使用本地地址
			hostToBind = getLocalHost();
		    }
		}
	    }
	}
	//添加bind.ip参数
	map.put(Constants.BIND_IP_KEY, hostToBind);
	// 默认情况下，注册ip不用于绑定ip
	// 获取注册ip
	String hostToRegistry = getValueFromConfig(protocolConfig, Constants.DUBBO_IP_TO_REGISTRY);
	if (hostToRegistry != null && hostToRegistry.length() > 0 && isInvalidLocalHost(hostToRegistry)) {
	    throw new IllegalArgumentException("Specified invalid registry ip from property:" + Constants.DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
	} else if (hostToRegistry == null || hostToRegistry.length() == 0) {
	    //默认情况下，绑定ip用于注册ip
	    hostToRegistry = hostToBind;
	}
	//添加anyhost参数
	map.put(Constants.ANYHOST_KEY, String.valueOf(anyhost));
	return hostToRegistry;
}

/**
 * 为服务提供者 获取注册端口和绑定端口
 * Register port and bind port for the provider, can be configured separately
 * Configuration priority:
 * environment variable -> java system properties -> port property in protocol config file
 * -> protocol default port
 * @param protocolConfig
 * @param name
 * @return 注册端口
 */
private Integer findConfigedPorts(ProtocolConfig protocolConfig, String name, Map<String, String> map) {
	Integer portToBind = null;

	//从系统配置中获取绑定端口
	String port = getValueFromConfig(protocolConfig, Constants.DUBBO_PORT_TO_BIND);
	portToBind = parsePort(port);

	if (portToBind == null) {
	    //从协议配置中获取绑定端口
	    portToBind = protocolConfig.getPort();
	    if (provider != null && (portToBind == null || portToBind == 0)) {
		//从服务提供者中获取绑定端口
		portToBind = provider.getPort();
	    }
	    //根据协议名称获取默认绑定端口
	    final int defaultPort = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(name).getDefaultPort();
	    if (portToBind == null || portToBind == 0) {
		portToBind = defaultPort;
	    }
	    if (portToBind == null || portToBind <= 0) {
		//获取随机端口
		portToBind = getRandomPort(name);
		if (portToBind == null || portToBind < 0) {
		    //获取可用端口
		    portToBind = getAvailablePort(defaultPort);
		    //设置随机端口(放入map缓存)
		    putRandomPort(name, portToBind);
		}
		logger.warn("Use random available port(" + portToBind + ") for protocol " + name);
	    }
	}
	//保存绑定端口，稍后用作url的key
	map.put(Constants.BIND_PORT_KEY, String.valueOf(portToBind));
	// registry port, not used as bind port by default
	String portToRegistryStr = getValueFromConfig(protocolConfig, Constants.DUBBO_PORT_TO_REGISTRY);
	Integer portToRegistry = parsePort(portToRegistryStr);
	if (portToRegistry == null) {
	    //注册端口使用绑定端口
	    portToRegistry = portToBind;
	}
	return portToRegistry;
}

/**
 * 构造监控URL
 * @param registryURL 注册中心URL
 * @return
 */
protected URL loadMonitor(URL registryURL) {
	if (monitor == null) {
	    //获取监控地址、监控协议配置
	    String monitorAddress = ConfigUtils.getProperty("dubbo.monitor.address");
	    String monitorProtocol = ConfigUtils.getProperty("dubbo.monitor.protocol");
	    if ((monitorAddress == null || monitorAddress.length() == 0) && (monitorProtocol == null || monitorProtocol.length() == 0)) {
		//如果监控地址和监控协议为空，则返回Null
		return null;
	    }
	    //构造监控配置对象(地址、协议)
	    monitor = new MonitorConfig();
	    if (monitorAddress != null && monitorAddress.length() > 0) {
		monitor.setAddress(monitorAddress);
	    }
	    if (monitorProtocol != null && monitorProtocol.length() > 0) {
		monitor.setProtocol(monitorProtocol);
	    }
	}
	//添加属性
	appendProperties(monitor);
	//参数
	Map<String, String> map = new HashMap<String, String>();
	//添加interface参数
	map.put(Constants.INTERFACE_KEY, MonitorService.class.getName());
	//添加dubbo版本
	map.put("dubbo", Version.getVersion());
	//添加时间戳
	map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
	if (ConfigUtils.getPid() > 0) {
	    //添加pid
	    map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
	}
	//添加monitor属性
	appendParameters(map, monitor);
	//获取monitor地址，优先使用系统配置值
	String address = monitor.getAddress();
	String sysaddress = System.getProperty("dubbo.monitor.address");
	if (sysaddress != null && sysaddress.length() > 0) {
	    address = sysaddress;
	}
	if (ConfigUtils.isNotEmpty(address)) {
	    if (!map.containsKey(Constants.PROTOCOL_KEY)) {
		//协议地址不为空，且map中不包含protocol属性时，则设置protocol
		if (ExtensionLoader.getExtensionLoader(MonitorFactory.class).hasExtension("logstat")) {
		    //包含logstat扩展
		    map.put(Constants.PROTOCOL_KEY, "logstat");
		} else {
		    //添加protocol属性
		    map.put(Constants.PROTOCOL_KEY, "dubbo");
		}
	    }
	    //生成URL对象
	    return UrlUtils.parseURL(address, map);
	} else if (Constants.REGISTRY_PROTOCOL.equals(monitor.getProtocol()) && registryURL != null) {
	    //监控address为空
	    //registryURL不为空，且monitor的协议为registry注册中心协议
	    return registryURL.setProtocol("dubbo")
		    //添加protocol属性为registry
		    .addParameter(Constants.PROTOCOL_KEY, "registry")
		    //添加refer属性，属性值为map参数
		    .addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map));
	}
	return null;
}
```

由于本小节主要是讲ServiceAnnotationBeanPostProcessor类实现，关于服务暴露的内容将会放到下一小节介绍。
