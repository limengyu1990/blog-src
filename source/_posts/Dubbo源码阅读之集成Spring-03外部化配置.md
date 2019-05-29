---
title: Dubbo源码阅读之集成Spring外部化配置(03)
date: 2018-08-08 14:24:15
tags:
     - dubbo
---
>这一小节，我们来看下支持外部化配置的相关类，因为本系列文章主要关注源码，具体的映射规则就不在介绍了，可以参考: https://zhuanlan.zhihu.com/p/32557951

### 支持的注解

#### DubboComponentScan注解
@Import是Spring提供的注解，可以通过导入的方式将实例添加到BeanFactory中，不了解的童鞋可以去我的博客里面找一下，有单独的文章介绍。
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(DubboComponentScanRegistrar.class)
public @interface DubboComponentScan {

    /**
     * basePackages() 别名，例如，可以使用 @DubboComponentScan("org.my.pkg") 替代 @DubboComponentScan(basePackages="org.my.pkg")
     * Alias for the {@link #basePackages()} attribute. Allows for more concise annotation
     * declarations e.g.: {@code @DubboComponentScan("org.my.pkg")} instead of
     * {@code @DubboComponentScan(basePackages="org.my.pkg")}.
     *
     * @return the base packages to scan
     */
    String[] value() default {};

    /**
     * 被标注@Service注解的类所在基础包路径
     * @return the base packages to scan
     */
    String[] basePackages() default {};

    /**
     * 类型安全，用来替代 basePackages()
     * @return the base packages to scan
     */
    Class<?>[] basePackageClasses() default {};
}
```
然后我们看下DubboComponentScanRegistrar类，该类负责注册ServiceAnnotationBeanPostProcessor和ReferenceAnnotationBeanPostProcessor到BeanFactory
```java
/**
 * 实现了ImportBeanDefinitionRegistrar接口，允许我们注册特定的bean到Spring中
 * Dubbo {@link DubboComponentScan} Bean Registrar
 *
 * @see Service
 * @see DubboComponentScan
 * @see ImportBeanDefinitionRegistrar
 * @see ServiceAnnotationBeanPostProcessor
 * @see ReferenceAnnotationBeanPostProcessor
 * @since 2.5.7
 */
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        //获取待扫描的基础包路径
        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);

        //注册ServiceAnnotationBeanPostProcessor
        registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);
        //注册ReferenceAnnotationBeanPostProcessor
        registerReferenceAnnotationBeanPostProcessor(registry);

    }

    /**
     * 注册ServiceAnnotationBeanPostProcessor类
     *
     * @param packagesToScan 待扫描的包路径，没有处理placeholders
     * @param registry       {@link BeanDefinitionRegistry}
     * @since 2.5.8
     */
    private void registerServiceAnnotationBeanPostProcessor(Set<String> packagesToScan,
                                                            BeanDefinitionRegistry registry) {

        BeanDefinitionBuilder builder = rootBeanDefinition(ServiceAnnotationBeanPostProcessor.class);
        builder.addConstructorArgValue(packagesToScan);
        builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        //生成beanName并执行注册
        BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinition, registry);

    }

    /**
     * 注册ReferenceAnnotationBeanPostProcessor类
     * @param registry {@link BeanDefinitionRegistry}
     */
    private void registerReferenceAnnotationBeanPostProcessor(BeanDefinitionRegistry registry) {

        // Register @Reference Annotation Bean Processor
        BeanRegistrar.registerInfrastructureBean(registry,
                ReferenceAnnotationBeanPostProcessor.BEAN_NAME,
                ReferenceAnnotationBeanPostProcessor.class);

    }

    /**
     * 获取待扫描的基础包路径
     * @param metadata
     * @return
     */
    private Set<String> getPackagesToScan(AnnotationMetadata metadata) {
        //获取@DubboComponentScan注解属性
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
                metadata.getAnnotationAttributes(DubboComponentScan.class.getName())
        );
        String[] basePackages = attributes.getStringArray("basePackages");
        Class<?>[] basePackageClasses = attributes.getClassArray("basePackageClasses");
        String[] value = attributes.getStringArray("value");
        //将basePackages、basePackageClasses、value属性指定的值合并起来
        Set<String> packagesToScan = new LinkedHashSet<String>(Arrays.asList(value));
        packagesToScan.addAll(Arrays.asList(basePackages));
        for (Class<?> basePackageClass : basePackageClasses) {
            //通过basePackageClass获取包名
            packagesToScan.add(ClassUtils.getPackageName(basePackageClass));
        }
        if (packagesToScan.isEmpty()) {
            //如果没有配置基础包路径的话，则取包名作为基础包路径
            return Collections.singleton(ClassUtils.getPackageName(metadata.getClassName()));
        }
        return packagesToScan;
    }
}
```

#### EnableDubboConfig注解
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@Import(DubboConfigConfigurationSelector.class)
public @interface EnableDubboConfig {
    /**
     * 表明是否绑定到多个Spring-Bean,默认是false
     */
    boolean multiple() default false;
}

/**
 * 该类实现了ImportSelector接口，此接口是Spring中导入外部配置的核心接口，有兴趣的读者可以去了解下，这里就不多做介绍了。
 * Dubbo {@link AbstractConfig Config} Registrar
 * @see EnableDubboConfig
 * @see DubboConfigConfiguration
 * @since 2.5.8
 */
public class DubboConfigConfigurationSelector implements ImportSelector, Ordered {

    /**
     * 要导入到Spring容器中的组件全类名
     * @param importingClassMetadata
     * @return
     */
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {

        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
                importingClassMetadata
                        .getAnnotationAttributes(EnableDubboConfig.class.getName())
        );

        //是否绑定到多个Spring-Bean
        boolean multiple = attributes.getBoolean("multiple");

        if (multiple) {
            //返回多dubbo配置bean绑定
            return of(DubboConfigConfiguration.Multiple.class.getName());
        } else {
            return of(DubboConfigConfiguration.Single.class.getName());
        }
    }

    private static <T> T[] of(T... values) {
        return values;
    }

    @Override
    public int getOrder() {
        return HIGHEST_PRECEDENCE;
    }
}


public class DubboConfigConfiguration {

    /**
     * 单Dubbo配置Bean绑定
     * Single Dubbo {@link AbstractConfig Config} Bean Binding
     */
    @EnableDubboConfigBindings({
            @EnableDubboConfigBinding(prefix = "dubbo.application", type = ApplicationConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.module", type = ModuleConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.registry", type = RegistryConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.protocol", type = ProtocolConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.monitor", type = MonitorConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.provider", type = ProviderConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.consumer", type = ConsumerConfig.class)
    })
    public static class Single {

    }

    /**
     * 多Dubbo配置Bean绑定
     * Multiple Dubbo {@link AbstractConfig Config} Bean Binding
     */
    @EnableDubboConfigBindings({
            @EnableDubboConfigBinding(prefix = "dubbo.applications", type = ApplicationConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.modules", type = ModuleConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.registries", type = RegistryConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.protocols", type = ProtocolConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.monitors", type = MonitorConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.providers", type = ProviderConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.consumers", type = ConsumerConfig.class, multiple = true)
    })
    public static class Multiple {

    }
}
```
使用方式:
```java
//将以下内容的外部化配置文件物理路径为：classpath:/META-INF/multiple-config.properties:
#多Dubbo配置Bean绑定
## dubbo.applications
dubbo.applications.applicationBean.name = dubbo-demo-application
dubbo.applications.applicationBean2.name = dubbo-demo-application2
dubbo.applications.applicationBean3.name = dubbo-demo-application3

//新建配置类DubboMultipleConfiguration
@EnableDubboConfig(multiple = true)
@PropertySource("META-INF/multiple-config.properties")
private static class DubboMultipleConfiguration {

} 


//测试类
public class DubboConfigurationBootstrap {
	public static void main(String[] args) {
		//创建配置上下文
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
		//注册当前配置 Bean
       		context.register(DubboMultipleConfiguration.class);
       		context.refresh();

		//获取ApplicationConfig Bean："applicationBean"、"applicationBean2" 和 "applicationBean3"
		ApplicationConfig applicationBean = context.getBean("applicationBean", ApplicationConfig.class);
		ApplicationConfig applicationBean2 = context.getBean("applicationBean2", ApplicationConfig.class);
		ApplicationConfig applicationBean3 = context.getBean("applicationBean3", ApplicationConfig.class);

		System.out.printf("applicationBean.name = %s \n", applicationBean.getName());
		System.out.printf("applicationBean2.name = %s \n", applicationBean2.getName());
		System.out.printf("applicationBean3.name = %s \n", applicationBean3.getName());
	}
}
```

#### @EnableDubboConfigBinding/@EnableDubboConfigBindings注解
@EnableDubboConfig适合绝大多数外部化配置场景,然而无论是单Bean绑定,还是多Bean绑定,其外部化配置属性前缀是固化的,如dubbo.application以及dubbo.applications.
当应用需要自定义外部化配置属性前缀,@EnableDubboConfigBinding能提供更大的弹性，支持单个外部化配置属性前缀(prefix)与Dubbo配置Bean类型(AbstractConfig子类)绑定,如果需要多次绑定时,可使用@EnableDubboConfigBindings.
@EnableDubboConfigBinding在支持外部化配置属性与Dubbo配置类绑定时,与Dubbo过去的映射行为不同,被绑定的Dubbo配置类将会提升为Spring Bean,无需提前装配Dubbo配置类.同时,支持多Dubbo配置Bean装配.其Bean的绑定规则与@EnableDubboConfig一致.

##### @EnableDubboConfigBinding注解

```java
@Target({ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(DubboConfigBindingRegistrar.class)
public @interface EnableDubboConfigBinding {

    /**
     * 指定待绑定Dubbo配置类的外部化配置属性的前缀,比如dubbo.application为 ApplicationConfig的外部化配置属性的前缀.
     * prefix()支持占位符(Placeholder),
     * 并且其关联前缀值是否以"."作为结尾字符是可选的,即"dubbo.application"与"dubbo.application."效果相同
     * 例如："dubbo.application." 或 "dubbo.application"
     * The name prefix of the properties that are valid to bind to {@link AbstractConfig Dubbo Config}.
     *
     * @return the name prefix of the properties to bind
     */
    String prefix();

    /**
     * 绑定的Dubbo配置类型,即Dubbo配置类,所有AbstractConfig的子类都可以
     * @return The binding type of {@link AbstractConfig Dubbo Config}.
     * @see AbstractConfig
     * @see ApplicationConfig
     * @see ModuleConfig
     * @see RegistryConfig
     */
    Class<? extends AbstractConfig> type();

    /**
     * 是否需要将prefix()作为多个type()类型的Spring Bean外部化配置属性,默认值为false
     * It indicates whether {@link #prefix()} binding to multiple Spring Beans.
     *
     * @return the default value is <code>false</code>
     */
    boolean multiple() default false;
}


/**
 * 实现了ImportBeanDefinitionRegistrar接口，允许我们注册特定的bean到Spring中
 * {@link AbstractConfig Dubbo Config} binding Bean registrar
 *
 * @see EnableDubboConfigBinding
 * @see DubboConfigBindingBeanPostProcessor
 * @since 2.5.8
 */
public class DubboConfigBindingRegistrar implements ImportBeanDefinitionRegistrar, EnvironmentAware {

    private final Log log = LogFactory.getLog(getClass());

    private ConfigurableEnvironment environment;

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        //获取@EnableDubboConfigBinding注解
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
                importingClassMetadata.getAnnotationAttributes(EnableDubboConfigBinding.class.getName()));
        //处理@EnableDubboConfigBinding注解属性并注册bean
        registerBeanDefinitions(attributes, registry);
    }

    /**
     * 读取@EnableDubboConfigBinding注解属性值，然后注册bean
     * @param attributes
     * @param registry
     */
    protected void registerBeanDefinitions(AnnotationAttributes attributes, BeanDefinitionRegistry registry) {

        //获取@EnableDubboConfigBinding注解的prefix属性，并处理占位符
        String prefix = environment.resolvePlaceholders(attributes.getString("prefix"));
        //获取@EnableDubboConfigBinding注解的type属性
        Class<? extends AbstractConfig> configClass = attributes.getClass("type");
        //获取@EnableDubboConfigBinding注解的multiple属性
        boolean multiple = attributes.getBoolean("multiple");
        //注册Dubbo配置bean
        registerDubboConfigBeans(prefix, configClass, multiple, registry);
    }

    /**
     * 注册bean
     * @param prefix 属性前缀
     * @param configClass 待配置的config类
     * @param multiple
     * @param registry
     */
    private void registerDubboConfigBeans(String prefix,
                                          Class<? extends AbstractConfig> configClass,
                                          boolean multiple,
                                          BeanDefinitionRegistry registry) {

        /**
         * @Configuration
           @PropertySource(value = "classpath:resources.properties", ignoreResourceNotFound = false)
           public class AppConfig { }
         */
        //根据prefix从配置文件中找到相关的属性值
        //多bean绑定：
        //prefix:
        //  applications.prefix = dubbo.apps.
        //properties:
        //  applicationBean.name = dubbo-demo-application
        //  applicationBean2.name = dubbo-demo-application2
        //单bean绑定：prefix = dubbo.module.
        //  dubbo.module.id = moduleBean  =>  id = moduleBean
        //  dubbo.module.name = dubbo-demo-module => name = dubbo-demo-module
        Map<String, String> properties = getSubProperties(environment.getPropertySources(), prefix);

        //如果没有配置属性的话，则直接返回
        if (CollectionUtils.isEmpty(properties)) {
            if (log.isDebugEnabled()) {
                log.debug("There is no property for binding to dubbo config class [" + configClass.getName()
                        + "] within prefix [" + prefix + "]");
            }
            return;
        }

        //获取bean名称列表(即外部配置文件中配置的)
        Set<String> beanNames = multiple ? resolveMultipleBeanNames(properties) :
                Collections.singleton(resolveSingleBeanName(properties, configClass, registry));
        //遍历bean名称列表
        for (String beanName : beanNames) {
            //最终注册bean的地方
            registerDubboConfigBean(beanName, configClass, registry);
            //注册DubboConfigBindingBeanPostProcessor类
            registerDubboConfigBindingBeanPostProcessor(prefix, beanName, multiple, registry);
        }
    }

    /**
     * 使用beanName注册bean：configClass
     * @param beanName
     * @param configClass 配置类
     * @param registry
     */
    private void registerDubboConfigBean(String beanName, Class<? extends AbstractConfig> configClass,
                                         BeanDefinitionRegistry registry) {

        BeanDefinitionBuilder builder = rootBeanDefinition(configClass);
        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        //使用beanName注册bean：configClass
        registry.registerBeanDefinition(beanName, beanDefinition);
        if (log.isInfoEnabled()) {
            log.info("The dubbo config bean definition [name : " + beanName + ", class : " + configClass.getName() +
                    "] has been registered.");
        }
    }

    /**
     * 注册DubboConfigBindingBeanPostProcessor类
     * 用来处理dubbo config bean
     * @param prefix  属性前缀
     * @param beanName 已注册的config配置类的beanName
     * @param multiple 
     * @param registry
     */
    private void registerDubboConfigBindingBeanPostProcessor(String prefix, String beanName, boolean multiple,
                                                             BeanDefinitionRegistry registry) {

        Class<?> processorClass = DubboConfigBindingBeanPostProcessor.class;
        
        BeanDefinitionBuilder builder = rootBeanDefinition(processorClass);
        
        //如果是multiple的话，则最终前缀：prefix+"."+beanName
        //否则的话，最终前缀：prefix
        String actualPrefix = multiple ? normalizePrefix(prefix) + beanName : prefix;
        
        //通过构造函数注入属性：prefix、beanName
        builder.addConstructorArgValue(actualPrefix).addConstructorArgValue(beanName);
        //获取beanDefinition
        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        //设置不可代理
        beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        //注册DubboConfigBindingBeanPostProcessor
        registerWithGeneratedName(beanDefinition, registry);

        if (log.isInfoEnabled()) {
            log.info("The BeanPostProcessor bean definition [" + processorClass.getName()
                    + "] for dubbo config bean [name : " + beanName + "] has been registered.");
        }
    }

    @Override
    public void setEnvironment(Environment environment) {
        Assert.isInstanceOf(ConfigurableEnvironment.class, environment);
        this.environment = (ConfigurableEnvironment) environment;
    }

    /**
     * 处理多个bean名称
     * @param properties 如：
     *          applicationBean.name = dubbo-demo-application
     *          applicationBean2.name = dubbo-demo-application2
     * @return applicationBean、applicationBean2
     */
    private Set<String> resolveMultipleBeanNames(Map<String, String> properties) {
        //保存已解决的beanNames
        Set<String> beanNames = new LinkedHashSet<String>();
        //遍历所有的属性名：applicationBean.name、applicationBean2.name
        for (String propertyName : properties.keySet()) {
            //获取"."符号在属性名中第一次出现的位置
            //即判断是否存在"."
            int index = propertyName.indexOf(".");
            if (index > 0) {
                //获取bean名称：applicationBean、applicationBean2
                String beanName = propertyName.substring(0, index);
                //保存bean名称
                beanNames.add(beanName);
            }
        }
        return beanNames;
    }

    /**
     * 处理单个bean名称
     * @param properties 如：
     *      dubbo.module.id = moduleBean  =>  id = moduleBean
     *      dubbo.module.name = dubbo-demo-module => name = dubbo-demo-module
     * @param configClass dubbo配置类
     * @param registry
     * @return
     */
    private String resolveSingleBeanName(Map<String, String> properties,
                                         Class<? extends AbstractConfig> configClass,
                                         BeanDefinitionRegistry registry) {
        //获取id属性作为bean的名称
        String beanName = properties.get("id");
        if (!StringUtils.hasText(beanName)) {
            //id属性为空的话
            BeanDefinitionBuilder builder = rootBeanDefinition(configClass);
            //则使用工具类生成bean名称
            beanName = BeanDefinitionReaderUtils.generateBeanName(
                    builder.getRawBeanDefinition(), registry
            );
        }
        return beanName;
    }
}
```
可以看到，为每个注册的Config-bean都注册了一个DubboConfigBindingBeanPostProcessor类，现在我们看下DubboConfigBindingBeanPostProcessor类，该类实现了BeanPostProcessor接口,
实现该接口，可以在bean实例化前后，增加一些自己的逻辑处理，同时也实现了InitializingBean接口.
```java
/**
 * Dubbo Config Binding {@link BeanPostProcessor}
 *
 * @see EnableDubboConfigBinding
 * @see DubboConfigBindingRegistrar
 * @since 2.5.8
 */
public class DubboConfigBindingBeanPostProcessor implements BeanPostProcessor, ApplicationContextAware, InitializingBean {

    /**
     * 配置属性的前缀
     */
    private final String prefix;

    /**
     * 绑定的Config配置类的beanName
     */
    private final String beanName;

    private DubboConfigBinder dubboConfigBinder;

    private ApplicationContext applicationContext;

    /**
     * 是否忽略未知字段
     */
    private boolean ignoreUnknownFields = true;

    /**
     * 是否忽略无效字段
     */
    private boolean ignoreInvalidFields = true;

    /**
     * @param prefix   配置属性的前缀
     * @param beanName 绑定的Config配置类的beanName
     */
    public DubboConfigBindingBeanPostProcessor(String prefix, String beanName) {
        Assert.notNull(prefix, "The prefix of Configuration Properties must not be null");
        Assert.notNull(beanName, "The name of bean must not be null");
        this.prefix = prefix;
        this.beanName = beanName;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        //如果当前实例化的bean的beanName和该处理器保存的beanName相同的话,说明是该实例化bean对应的处理器
        //这个在之前介绍DubboConfigBindingRegistrar类时有介绍
        if (beanName.equals(this.beanName) && bean instanceof AbstractConfig) {
            AbstractConfig dubboConfig = (AbstractConfig) bean;
            //通过prefix绑定beanName的属性
            dubboConfigBinder.bind(prefix, dubboConfig);
            if (log.isInfoEnabled()) {
                log.info("The properties of bean [name : " + beanName + "] have been binding by prefix of " +
                        "configuration properties : " + prefix);
            }
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        if (dubboConfigBinder == null) {
            try {
                //先从BeanFactory中获取DubboConfigBinder实例
                dubboConfigBinder = applicationContext.getBean(DubboConfigBinder.class);
            } catch (BeansException ignored) {
                if (log.isDebugEnabled()) {
                    log.debug("DubboConfigBinder Bean can't be found in ApplicationContext.");
                }
                //使用默认实现
                dubboConfigBinder = createDubboConfigBinder(applicationContext.getEnvironment());
            }
        }
        //设置是否忽略未知字段、无效字段
        dubboConfigBinder.setIgnoreUnknownFields(ignoreUnknownFields);
        dubboConfigBinder.setIgnoreInvalidFields(ignoreInvalidFields);
    }

    /**
     * 创建DubboConfigBinder实例(默认实现)
     * @param environment
     * @return {@link DefaultDubboConfigBinder}
     */
    protected DubboConfigBinder createDubboConfigBinder(Environment environment) {
        DefaultDubboConfigBinder defaultDubboConfigBinder = new DefaultDubboConfigBinder();
        defaultDubboConfigBinder.setEnvironment(environment);
        return defaultDubboConfigBinder;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```
然后我们看下使用到的DubboConfigBinder接口
```java
public interface DubboConfigBinder extends EnvironmentAware {
    /**
     * 设置是否忽略未知字段，即是否忽略在目标对象中没有相应字段的绑定参数。默认值是“true”
     * 设置为false，可以强制所有的绑定的参数在目标对象中必须有一个相应的字段
     *
     * Set whether to ignore unknown fields, that is, whether to ignore bind
     * parameters that do not have corresponding fields in the target object.
     * <p>Default is "true". Turn this off to enforce that all bind parameters
     * must have a matching field in the target object.
     * @see #bind
     */
    void setIgnoreUnknownFields(boolean ignoreUnknownFields);

    /**
     * 是否忽略无效字段，即是否忽略在目标对象中存在相应字段，但是不可访问的绑定参数
     * 默认为“false”
     * Set whether to ignore invalid fields, that is, whether to ignore bind
     * parameters that have corresponding fields in the target object which are
     * not accessible (for example because of null values in the nested path).
     * <p>Default is "false".
     *
     * @see #bind
     */
    void setIgnoreInvalidFields(boolean ignoreInvalidFields);

    /**
     * 以指定的前缀绑定相关属性到Dubbo配置对象中
     * Bind the properties to Dubbo Config Object under specified prefix.
     *
     * @param prefix 属性前缀
     * @param dubboConfig dubboConfig配置类实例
     */
    <C extends AbstractConfig> void bind(String prefix, C dubboConfig);
}

//抽象类
public abstract class AbstractDubboConfigBinder implements DubboConfigBinder {

    /**
     * 属性配置
     */
    private Iterable<PropertySource<?>> propertySources;

    private boolean ignoreUnknownFields = true;

    private boolean ignoreInvalidFields = false;

    /**
     * 获取多个PropertySource
     */
    protected Iterable<PropertySource<?>> getPropertySources() {
        return propertySources;
    }

    public boolean isIgnoreUnknownFields() {
        return ignoreUnknownFields;
    }

    @Override
    public void setIgnoreUnknownFields(boolean ignoreUnknownFields) {
        this.ignoreUnknownFields = ignoreUnknownFields;
    }

    public boolean isIgnoreInvalidFields() {
        return ignoreInvalidFields;
    }

    @Override
    public void setIgnoreInvalidFields(boolean ignoreInvalidFields) {
        this.ignoreInvalidFields = ignoreInvalidFields;
    }

    @Override
    public final void setEnvironment(Environment environment) {
        if (environment instanceof ConfigurableEnvironment) {
            //从environment中获取propertySources
            this.propertySources = ((ConfigurableEnvironment) environment).getPropertySources();
        }
    }
}


/**
 * 默认实现 基于Spring的DataBinder
 */
public class DefaultDubboConfigBinder extends AbstractDubboConfigBinder {

    @Override
    public <C extends AbstractConfig> void bind(String prefix, C dubboConfig) {
        //创建DataBinder实例，绑定dubboConfig配置类
        DataBinder dataBinder = new DataBinder(dubboConfig);
        // Set ignored*
        //设置是否忽略无效字段、未知字段
        dataBinder.setIgnoreInvalidFields(isIgnoreInvalidFields());
        dataBinder.setIgnoreUnknownFields(isIgnoreUnknownFields());
        //从PropertySources中获取指定前缀的属性
        Map<String, String> properties = getSubProperties(getPropertySources(), prefix);
        //将Map转换成MutablePropertyValues对象
        MutablePropertyValues propertyValues = new MutablePropertyValues(properties);
        // 进行绑定
        dataBinder.bind(propertyValues);
    }
}
```
然后看下工具类PropertySourcesUtils中的getSubProperties方法：
```java
/**
 * 根据属性前缀，获取所有相关的属性和属性值
 * Get Sub {@link Properties}
 * @param propertySources {@link PropertySource} Iterable
 * @param prefix          the prefix of property name
 * @return Map<String,String>
 * @see Properties
*/
public static Map<String, String> getSubProperties(Iterable<PropertySource<?>> propertySources, String prefix) {
	Map<String, String> subProperties = new LinkedHashMap<String, String>();
	//规范化前缀,即前缀结尾处补上"."符号
	String normalizedPrefix = normalizePrefix(prefix);
	//遍历每一个PropertySource
	for (PropertySource<?> source : propertySources) {
	    if (source instanceof EnumerablePropertySource) {
		//遍历属性名称列表，找到以normalizedPrefix开头的属性名称
		//applications.prefix = dubbo.apps.
		//dubbo.apps.applicationBean.name = dubbo-demo-application
		//dubbo.apps.applicationBean2.name = dubbo-demo-application2

		for (String name : ((EnumerablePropertySource<?>) source).getPropertyNames()) {
		    if (name.startsWith(normalizedPrefix)) {
			//截取子名称，如：subName = applicationBean.name/applicationBean2.name
			String subName = name.substring(normalizedPrefix.length());
			//更加属性名称name获取属性值value
			Object value = source.getProperty(name);
			//保存applicationBean.name = dubbo-demo-application
			//保存applicationBean2.name = dubbo-demo-application2
			subProperties.put(subName, String.valueOf(value));
		    }
		}
	    }
	}
	return subProperties;
}
```

##### @EnableDubboConfigBindings注解
该注解支持配置多个@EnableDubboConfigBinding注解
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(DubboConfigBindingsRegistrar.class)
public @interface EnableDubboConfigBindings {
    /**
     * The value of {@link EnableDubboConfigBindings}
     * @return non-null
     */
    EnableDubboConfigBinding[] value();
}


public class DubboConfigBindingsRegistrar implements ImportBeanDefinitionRegistrar, EnvironmentAware {

    private ConfigurableEnvironment environment;

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        //获取@EnableDubboConfigBindings注解的属性值
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
                importingClassMetadata.getAnnotationAttributes(EnableDubboConfigBindings.class.getName()));

        AnnotationAttributes[] annotationAttributes = attributes.getAnnotationArray("value");

        //构造DubboConfigBindingRegistrar对象
        DubboConfigBindingRegistrar registrar = new DubboConfigBindingRegistrar();
        registrar.setEnvironment(environment);
        //遍历@EnableDubboConfigBinding列表,处理每一个@EnableDubboConfigBinding注解
        for (AnnotationAttributes element : annotationAttributes) {
            //注册DubboConfigBindingRegistrar类
            registrar.registerBeanDefinitions(element, registry);
        }
    }

    @Override
    public void setEnvironment(Environment environment) {
        Assert.isInstanceOf(ConfigurableEnvironment.class, environment);
        this.environment = (ConfigurableEnvironment) environment;
    }
}
```
简单看下使用:
```java
//将以下内容的外部化配置文件物理路径为：classpath:/META-INF/bindings.properties
# classpath:/META-INF/bindings.properties
## 占位符值 : ApplicationConfig 外部配置属性前缀
applications.prefix = dubbo.apps.

## 多 ApplicationConfig Bean 绑定
dubbo.apps.applicationBean.name = dubbo-demo-application
dubbo.apps.applicationBean2.name = dubbo-demo-application2
dubbo.apps.applicationBean3.name = dubbo-demo-application3

## 单 ModuleConfig Bean 绑定
dubbo.module.id = moduleBean
dubbo.module.name = dubbo-demo-module

## 单 RegistryConfig Bean 绑定
dubbo.registry.address = zookeeper://192.168.99.100:32770


/**
 * 新建一个DubboConfiguration类作为Dubbo配置Bean，添加@EnableDubboConfigBinding注解进行绑定
 * 然后配置@PropertySource注解指定外部配置文件
 */
@EnableDubboConfigBindings({
@EnableDubboConfigBinding(prefix = "${applications.prefix}",
               type = ApplicationConfig.class, multiple = true), //多ApplicationConfig Bean绑定
@EnableDubboConfigBinding(prefix = "dubbo.module", //不带"."后缀
               type = ModuleConfig.class), //单ModuleConfig Bean绑定
@EnableDubboConfigBinding(prefix = "dubbo.registry.", //带"."后缀
               type = RegistryConfig.class) //单RegistryConfig Bean绑定
})
@PropertySource("META-INF/bindings.properties")
@Configuration
public class DubboConfiguration {

}

//新建测试类
public class DubboConfigurationBootstrap {

	public static void main(String[] args) {
		//创建配置上下文
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
		//注册当前配置 Bean
		context.register(DubboConfiguration.class);
		context.refresh();

		//获取ApplicationConfig Bean："applicationBean"、"applicationBean2" 和 "applicationBean3"
		ApplicationConfig applicationBean = context.getBean("applicationBean", ApplicationConfig.class);
		ApplicationConfig applicationBean2 = context.getBean("applicationBean2", ApplicationConfig.class);
		ApplicationConfig applicationBean3 = context.getBean("applicationBean3", ApplicationConfig.class);
		
		//applicationBean.name = dubbo-demo-application 
		//applicationBean2.name = dubbo-demo-application2 
		capplicationBean3.name = dubbo-demo-application3 
		System.out.printf("applicationBean.name = %s \n", applicationBean.getName());
		System.out.printf("applicationBean2.name = %s \n", applicationBean2.getName());
		System.out.printf("applicationBean3.name = %s \n", applicationBean3.getName());

		//获取ModuleConfig Bean："moduleBean"
		ModuleConfig moduleBean = context.getBean("moduleBean", ModuleConfig.class);
		//moduleBean.name = dubbo-demo-module 
		System.out.printf("moduleBean.name = %s \n", moduleBean.getName()); 

		//获取RegistryConfig Bean
		RegistryConfig registry = context.getBean(RegistryConfig.class);
		//registry.address = zookeeper://192.168.99.100:32770 
		System.out.printf("registry.address = %s \n", registry.getAddress());
	}
}
```

#### @EnableDubbo注解
该注解等同于组合使用@DubboComponentScan和@EnableDubboConfig，上文已经介绍过，这里就不多介绍了。
```java
/**
 * 启用Dubbo组件作为SpringBean
 * 等同于组合使用@DubboComponentScan和@EnableDubboConfig
 */
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@EnableDubboConfig
@DubboComponentScan
public @interface EnableDubbo {

    /**
     * 扫描@Service基础包路径
     * Base packages to scan for annotated @Service classes.
     * <p>
     * Use {@link #scanBasePackageClasses()} for a type-safe alternative to String-based
     * package names.
     *
     * @return the base packages to scan
     * @see DubboComponentScan#basePackages()
     */
    @AliasFor(annotation = DubboComponentScan.class, attribute = "basePackages")
    String[] scanBasePackages() default {};

    /**
     * Type-safe alternative to {@link #scanBasePackages()} for specifying the packages to
     * scan for annotated @Service classes. The package of each class specified will be
     * scanned.
     *
     * @return classes from the base packages to scan
     * @see DubboComponentScan#basePackageClasses
     */
    @AliasFor(annotation = DubboComponentScan.class, attribute = "basePackageClasses")
    Class<?>[] scanBasePackageClasses() default {};


    /**
     *
     * It indicates whether {@link AbstractConfig} binding to multiple Spring Beans.
     *
     * @return the default value is <code>false</code>
     * @see EnableDubboConfig#multiple()
     */
    @AliasFor(annotation = EnableDubboConfig.class, attribute = "multiple")
    boolean multipleConfig() default false;
}
```

