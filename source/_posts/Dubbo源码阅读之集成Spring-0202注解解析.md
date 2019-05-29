---
title: Dubbo源码阅读之集成Spring注解解析(0202)
date: 2018-08-07 19:46:49
tags: 
     - dubbo
---
>本小节将会介绍ReferenceAnnotationBeanPostProcessor类的实现

### ReferenceAnnotationBeanPostProcessor类
该类用来处理@Reference注解，它实现了BeanPostProcessor接口,实现postProcessPropertyValues方法可以处理每一个属性
```java
public class ReferenceAnnotationBeanPostProcessor extends InstantiationAwareBeanPostProcessorAdapter
        implements MergedBeanDefinitionPostProcessor, PriorityOrdered, ApplicationContextAware,BeanClassLoaderAware, DisposableBean {

    /**
     * ReferenceAnnotationBeanPostProcessor的bean-name
     */
    public static final String BEAN_NAME = "referenceAnnotationBeanPostProcessor";

    private ApplicationContext applicationContext;

    private ClassLoader classLoader;

    /**
     * beanName/className,ReferenceInjectionMetadata
     */
    private final ConcurrentMap<String, ReferenceInjectionMetadata> injectionMetadataCache =
            new ConcurrentHashMap<String, ReferenceInjectionMetadata>(256);

    /**
     * cacheKey,referenceBean
     */
    private final ConcurrentMap<String, ReferenceBean<?>> referenceBeansCache =
            new ConcurrentHashMap<String, ReferenceBean<?>>();

    /**
     * 设置某个属性时调用
     * @param pvs
     * @param pds
     * @param bean
     * @param beanName
     * @return
     * @throws BeanCreationException
     */
    @Override
    public PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
        //获取bean的@Reference元数据信息
        InjectionMetadata metadata = findReferenceMetadata(beanName, bean.getClass(), pvs);
        try {
            //对Bean的属性进行自动注入
            //最终会调用内部类ReferenceFieldElement和ReferenceMethodElement的inject方法(后面会介绍)
            metadata.inject(bean, beanName, pvs);
        } catch (BeanCreationException ex) {
            throw ex;
        } catch (Throwable ex) {
            throw new BeanCreationException(beanName, "Injection of @Reference dependencies failed", ex);
        }
        return pvs;
    }

    /**
     * 获取clazz的@Reference元数据信息
     * @param beanName 当前bean的名称
     * @param clazz    当前bean的class
     * @param pvs      当前bean的属性
     * @return
     */
    private InjectionMetadata findReferenceMetadata(String beanName, Class<?> clazz, PropertyValues pvs) {
        // Fall back to class name as cache key, for backwards compatibility with custom callers.
        //使用beanName或者当前bean的类名称作为cacheKey
        String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
        // 先从缓存中查询该cacheKey
        ReferenceInjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
        //是否需要刷新(metadata == null || metadata.targetClass != clazz;则需要刷新)
        if (InjectionMetadata.needsRefresh(metadata, clazz)) {
            synchronized (this.injectionMetadataCache) {
                metadata = this.injectionMetadataCache.get(cacheKey);
                if (InjectionMetadata.needsRefresh(metadata, clazz)) {
                    if (metadata != null) {
                        metadata.clear(pvs);
                    }
                    try {
                        //获取clazz中被@Reference注解标注的元数据信息(字段、方法)
                        metadata = buildReferenceMetadata(clazz);
                        //放入缓存
                        this.injectionMetadataCache.put(cacheKey, metadata);
                    } catch (NoClassDefFoundError err) {
                        throw new IllegalStateException("Failed to introspect bean class [" + clazz.getName() +
                                "] for reference metadata: could not find class that it depends on", err);
                    }
                }
            }
        }
        return metadata;
    }

    /**
     * 获取beanClass类中被@Reference注解标注的方法和字段
     * @param beanClass
     * @return
     */
    private ReferenceInjectionMetadata buildReferenceMetadata(final Class<?> beanClass) {
        //查到beanClass中存在@Reference注解的字段
        Collection<ReferenceFieldElement> fieldElements = findFieldReferenceMetadata(beanClass);
        //查到beanClass中存在@Reference注解的方法
        Collection<ReferenceMethodElement> methodElements = findMethodReferenceMetadata(beanClass);
        //创建新的元数据信息
        return new ReferenceInjectionMetadata(beanClass, fieldElements, methodElements);
    }


    /**
     * 从给定的类中查询到所有标注@Reference注解的字段
     * 然后根据该字段和@Reference注解
     * 创建ReferenceFieldElement类(InjectionMetadata.InjectedElement的子类)
     * @param beanClass 当前bean的class
     * @return non-null
     */
    private List<ReferenceFieldElement> findFieldReferenceMetadata(final Class<?> beanClass) {

        final List<ReferenceFieldElement> elements = new LinkedList<ReferenceFieldElement>();
        //操作字段时执行的回调
        ReflectionUtils.doWithFields(beanClass, new ReflectionUtils.FieldCallback() {
            @Override
            public void doWith(Field field) throws IllegalArgumentException, IllegalAccessException {
                //获取字段上的@Reference注解
                Reference reference = getAnnotation(field, Reference.class);
                //是否存在@Reference注解
                if (reference != null) {
                    if (Modifier.isStatic(field.getModifiers())) {
                        if (logger.isWarnEnabled()) {
                            //@Reference不支持静态字段字段
                            logger.warn("@Reference annotation is not supported on static fields: " + field);
                        }
                        return;
                    }
                    //根据字段和@Reference注解创建ReferenceFieldElement对象,并保存
                    elements.add(new ReferenceFieldElement(field, reference));
                }
            }
        });
        return elements;
    }

    /**
     * 从标注@Reference注解的方法上找到（InjectionMetadata.InjectedElement）元数据
     * @param beanClass 目标bean的class
     * @return non-null {@link List}
     */
    private List<ReferenceMethodElement> findMethodReferenceMetadata(final Class<?> beanClass) {

        final List<ReferenceMethodElement> elements = new LinkedList<ReferenceMethodElement>();

        //操作方法时调用
        ReflectionUtils.doWithMethods(beanClass, new ReflectionUtils.MethodCallback() {
            @Override
            public void doWith(Method method) throws IllegalArgumentException, IllegalAccessException {

                //获取桥接方法(https://blog.csdn.net/mhmyqn/article/details/47342577)
                Method bridgedMethod = findBridgedMethod(method);

                //参数和返回类型签名相同返回true
                if (!isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                    return;
                }
                //从桥接方法上找到@Reference注解
                Reference reference = findAnnotation(bridgedMethod, Reference.class);

                //ClassUtils.getMostSpecificMethod
                //通过给定的方法(可能来自接口)和当前反射调用中使用的目标类,找到相应的目标方法
                if (reference != null && method.equals(ClassUtils.getMostSpecificMethod(method, beanClass))) {
                    if (Modifier.isStatic(method.getModifiers())) {
                        if (logger.isWarnEnabled()) {
                            //@Reference不支持static方法
                            logger.warn("@Reference annotation is not supported on static methods: " + method);
                        }
                        return;
                    }
                    if (method.getParameterTypes().length == 0) {
                        //@Reference注解只能用于带参数的方法
                        if (logger.isWarnEnabled()) {
                            logger.warn("@Reference  annotation should only be used on methods with parameters: " +
                                    method);
                        }
                    }
                    //获取bridgedMethod方法的属性描述
                    PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, beanClass);
                    //创建ReferenceMethodElement对象，并保存
                    elements.add(new ReferenceMethodElement(method, pd, reference));
                }
            }
        });
        return elements;
    }


    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Override
    public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
        if (beanType != null) {
            InjectionMetadata metadata = findReferenceMetadata(beanName, beanType, null);
            metadata.checkConfigMembers(beanDefinition);
        }
    }

    @Override
    public int getOrder() {
        return LOWEST_PRECEDENCE;
    }

    @Override
    public void destroy() throws Exception {

        for (ReferenceBean referenceBean : referenceBeansCache.values()) {
            if (logger.isInfoEnabled()) {
                logger.info(referenceBean + " was destroying!");
            }
            //销毁referenceBean
            referenceBean.destroy();
        }
        //清空injectionMetadataCache/referenceBeansCache
        injectionMetadataCache.clear();
        referenceBeansCache.clear();

        if (logger.isInfoEnabled()) {
            logger.info(getClass() + " was destroying!");
        }
    }

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        this.classLoader = classLoader;
    }

    /**
     * 获取所有的ReferenceBean
     * @return non-null {@link Collection}
     * @since 2.5.9
     */
    public Collection<ReferenceBean<?>> getReferenceBeans() {
        return this.referenceBeansCache.values();
    }
```
##### ReferenceInjectionMetadata内部类
```java
    /**
     * {@link Reference} {@link InjectionMetadata} implementation
     *
     * @since 2.5.11
     */
    private static class ReferenceInjectionMetadata extends InjectionMetadata {

        private final Collection<ReferenceFieldElement> fieldElements;

        private final Collection<ReferenceMethodElement> methodElements;

        /**
         * @param targetClass 目标类
         * @param fieldElements 存在@Reference注解的字段
         * @param methodElements 存在@Reference注解的方法
         */
        public ReferenceInjectionMetadata(Class<?> targetClass, Collection<ReferenceFieldElement> fieldElements,
                                          Collection<ReferenceMethodElement> methodElements) {
            super(targetClass, combine(fieldElements, methodElements));
            this.fieldElements = fieldElements;
            this.methodElements = methodElements;
        }

        /**
         * 将fieldElements和methodElements进行合并
         * @param elements
         * @param <T>
         * @return
         */
        private static <T> Collection<T> combine(Collection<? extends T>... elements) {
            List<T> allElements = new ArrayList<T>();
            for (Collection<? extends T> e : elements) {
                allElements.addAll(e);
            }
            return allElements;
        }

        public Collection<ReferenceFieldElement> getFieldElements() {
            return fieldElements;
        }

        public Collection<ReferenceMethodElement> getMethodElements() {
            return methodElements;
        }
    }
```

##### ReferenceMethodElement内部类
```java
   /**
     * 内部类，方法元数据，最终会调用该类的inject方法进行注入
     */
    private class ReferenceMethodElement extends InjectionMetadata.InjectedElement {

        /**
         * 标注@Reference注解的方法
         */
        private final Method method;

        private final Reference reference;
	
	/**
	 * 新生成的referenceBean
	 */
        private volatile ReferenceBean<?> referenceBean;

        /**
         * @param method
         * @param pd 方法属性描述符
         * @param reference
         */
        protected ReferenceMethodElement(Method method, PropertyDescriptor pd, Reference reference) {
            super(method, pd);
            this.method = method;
            this.reference = reference;
        }

        @Override
        protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
            //获取referenceClass
            Class<?> referenceClass = pd.getPropertyType();
            //获取referenceClass对应的referenceBean
            referenceBean = buildReferenceBean(reference, referenceClass);

            ReflectionUtils.makeAccessible(method);
            //调用bean的method，注入依赖
            method.invoke(bean, referenceBean.getObject());
        }
    }
```
##### ReferenceFieldElement内部类
```java
    /**
     * 内部类，字段元数据，最终会调用该类的inject方法进行注入
     */
    private class ReferenceFieldElement extends InjectionMetadata.InjectedElement {
        /**
         * 标注@Reference注解的字段
         */
        private final Field field;

        private final Reference reference;

        private volatile ReferenceBean<?> referenceBean;

        protected ReferenceFieldElement(Field field, Reference reference) {
            super(field, null);
            this.field = field;
            this.reference = reference;
        }

        /**
         * 注入
         * @param bean  目标bean
         * @param beanName 目标bean-name
         * @param pvs
         * @throws Throwable
         */
        @Override
        protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {

            //被@Reference注解标识的字段的类型
            Class<?> referenceClass = field.getType();
            //获取referenceClass对应的referenceBean
            referenceBean = buildReferenceBean(reference, referenceClass);

            ReflectionUtils.makeAccessible(field);
            //注入依赖到目标bean中
            field.set(bean, referenceBean.getObject());
        }
    }
```
##### buildReferenceBean方法
```java
    /**
     * 根据@Reference注解和referenceClass生成ReferenceBean
     * @param reference
     * @param referenceClass 被@Reference注解标注的字段类型
     * @return
     * @throws Exception
     */
    private ReferenceBean<?> buildReferenceBean(Reference reference, Class<?> referenceClass) throws Exception {

        //根据@Reference注解和referenceClass 生成缓存key
        String referenceBeanCacheKey = generateReferenceBeanCacheKey(reference, referenceClass);

        //先从缓存中获取ReferenceBean
        ReferenceBean<?> referenceBean = referenceBeansCache.get(referenceBeanCacheKey);

        if (referenceBean == null) {
            //生成一个referenceBean并放入缓存(后面会分析该方法)
            ReferenceBeanBuilder beanBuilder = ReferenceBeanBuilder
                    .create(reference, classLoader, applicationContext)
                    .interfaceClass(referenceClass);
            referenceBean = beanBuilder.build();
            referenceBeansCache.putIfAbsent(referenceBeanCacheKey, referenceBean);
        }
        return referenceBean;

    }
```
```java
    /**
     * 为ReferenceBean创建一个缓存key
     * @param reference {@link Reference}
     * @param beanClass {@link Class}
     * @return
     */
    private String generateReferenceBeanCacheKey(Reference reference, Class<?> beanClass) {
        //获取接口名称(后面会分析该方法)
        String interfaceName = resolveInterfaceName(reference, beanClass);
        //生成缓存key
        String key = reference.url() + "/" + interfaceName +
                "/" + reference.version() +
                "/" + reference.group();

        Environment environment = applicationContext.getEnvironment();
        //处理占位符
        key = environment.resolvePlaceholders(key);
        return key;
    }

    /**
     * 获取接口名称interfaceName
     * 先从@Reference注解属性找，找不到则取被注解的接口变量名称
     * @param reference @Reference注解
     * @param beanClass 被@Reference注解标注的类
     * @return
     * @throws IllegalStateException
     */
    private static String resolveInterfaceName(Reference reference, Class<?> beanClass)
            throws IllegalStateException {

        String interfaceName;
        if (!"".equals(reference.interfaceName())) {
            // @Reference注解的interfaceName属性不为空
            interfaceName = reference.interfaceName();
        } else if (!void.class.equals(reference.interfaceClass())) {
            // @Reference注解的interfaceClass属性不为void
            interfaceName = reference.interfaceClass().getName();
        } else if (beanClass.isInterface()) {
            // 被@Reference注解标注的类是接口类型
            interfaceName = beanClass.getName();
        } else {
            //@Reference没有定义interfaceClass/interfaceName属性，beanClass不是一个接口
            throw new IllegalStateException(
                    "The @Reference undefined interfaceClass or interfaceName, and the property type "
                            + beanClass.getName() + " is not a interface.");
        }
        return interfaceName;
    }


    /**
     * 获取<field,ReferenceBean>
     *
     * @return non-null {@link Map}
     * @since 2.5.11
     */
    public Map<InjectionMetadata.InjectedElement, ReferenceBean<?>> getInjectedFieldReferenceBeanMap() {

        Map<InjectionMetadata.InjectedElement, ReferenceBean<?>> injectedElementReferenceBeanMap =
                new LinkedHashMap<InjectionMetadata.InjectedElement, ReferenceBean<?>>();
        for (ReferenceInjectionMetadata metadata : injectionMetadataCache.values()) {
            Collection<ReferenceFieldElement> fieldElements = metadata.getFieldElements();
            for (ReferenceFieldElement fieldElement : fieldElements) {
                injectedElementReferenceBeanMap.put(fieldElement, fieldElement.referenceBean);
            }
        }
        return injectedElementReferenceBeanMap;
    }

    /**
     * 获取<方法，ReferenceBean>
     * @return non-null {@link Map}
     * @since 2.5.11
     */
    public Map<InjectionMetadata.InjectedElement, ReferenceBean<?>> getInjectedMethodReferenceBeanMap() {

        Map<InjectionMetadata.InjectedElement, ReferenceBean<?>> injectedElementReferenceBeanMap =
                new LinkedHashMap<InjectionMetadata.InjectedElement, ReferenceBean<?>>();

        for (ReferenceInjectionMetadata metadata : injectionMetadataCache.values()) {
            Collection<ReferenceMethodElement> methodElements = metadata.getMethodElements();
            for (ReferenceMethodElement methodElement : methodElements) {
                injectedElementReferenceBeanMap.put(methodElement, methodElement.referenceBean);
            }
        }
        return injectedElementReferenceBeanMap;
    }
}
```

##### AbstractAnnotationConfigBeanBuilder类
接下来我们看下ReferenceBean的创建

```java
//创建ReferenceBean
ReferenceBeanBuilder beanBuilder = ReferenceBeanBuilder
          //create方法创建了ReferenceBeanBuilder实例
	  .create(reference, classLoader, applicationContext)
	  //设置referenceClass
          .interfaceClass(referenceClass);
//build方法创建ReferenceBean类
ReferenceBean<?> referenceBean = beanBuilder.build();
```

我们是通过ReferenceBeanBuilder类创建ReferenceBean的，而ReferenceBeanBuilder继承自AbstractAnnotationConfigBeanBuilder,
AbstractAnnotationConfigBeanBuilder定义了创建配置bean的算法骨架(即build模板方法)，其他步骤都由子类ReferenceBeanBuilder负责实现。
```java
/**
 * Annotation bean配置构造器
 * Abstract Configurable {@link Annotation} Bean Builder
 * @since 2.5.7
 */
abstract class AbstractAnnotationConfigBeanBuilder<A extends Annotation, B extends AbstractInterfaceConfig> {

    protected final Log logger = LogFactory.getLog(getClass());

    /**
     * 注解
     */
    protected final A annotation;

    protected final ApplicationContext applicationContext;

    protected final ClassLoader classLoader;

    protected Object bean;

    /**
     * 接口类
     */
    protected Class<?> interfaceClass;

    protected AbstractAnnotationConfigBeanBuilder(A annotation, ClassLoader classLoader,
                                                  ApplicationContext applicationContext) {
        Assert.notNull(annotation, "The Annotation must not be null!");
        Assert.notNull(classLoader, "The ClassLoader must not be null!");
        Assert.notNull(applicationContext, "The ApplicationContext must not be null!");
        this.annotation = annotation;
        this.applicationContext = applicationContext;
        this.classLoader = classLoader;
    }

    /**
     * Build {@link B}
     * @return non-null
     * @throws Exception
     */
    public final B build() throws Exception {
        //检测依赖
        checkDependencies();
        //构建B（子类实现）
        B bean = doBuild();
        //配置B
        configureBean(bean);

        if (logger.isInfoEnabled()) {
            logger.info(bean + " has been built.");
        }
        return bean;

    }

    private void checkDependencies() {

    }

    /**
     * Builds {@link B Bean}
     *
     * @return {@link B Bean}
     */
    protected abstract B doBuild();


    protected void configureBean(B bean) throws Exception {

        //子类实现
        preConfigureBean(annotation, bean);

        //配置RegistryConfig(即将RegistryConfig实例注入到B对象中)
        configureRegistryConfigs(bean);

        //配置MonitorConfig
        configureMonitorConfig(bean);

        //配置ApplicationConfig
        configureApplicationConfig(bean);

        //配置ModuleConfig
        configureModuleConfig(bean);

        //子类实现
        postConfigureBean(annotation, bean);
    }

    protected abstract void preConfigureBean(A annotation, B bean) throws Exception;


    /**
     * 配置RegistryConfig
     * @param bean
     */
    private void configureRegistryConfigs(B bean) {
        //获取RegistryConfig的bean-names
        String[] registryConfigBeanIds = resolveRegistryConfigBeanNames(annotation);
        //根据bean-names从工厂中获取RegistryConfig实例
        List<RegistryConfig> registryConfigs = getBeans(applicationContext, registryConfigBeanIds, RegistryConfig.class);
        //将registryConfigs注入到B中
        bean.setRegistries(registryConfigs);
    }

    /**
     * 配置MonitorConfig
     * @param bean
     */
    private void configureMonitorConfig(B bean) {
        //获取MonitorConfig的bean-names
        String monitorBeanName = resolveMonitorConfigBeanName(annotation);
        //获取monitorConfig实例
        MonitorConfig monitorConfig = getOptionalBean(applicationContext, monitorBeanName, MonitorConfig.class);
        //将registryConfigs注入到B中
        bean.setMonitor(monitorConfig);

    }

    /**
     * 配置ApplicationConfig
     * @param bean
     */
    private void configureApplicationConfig(B bean) {
        //获取ApplicationConfig的bean-names
        String applicationConfigBeanName = resolveApplicationConfigBeanName(annotation);

        ApplicationConfig applicationConfig =
                getOptionalBean(applicationContext, applicationConfigBeanName, ApplicationConfig.class);

        bean.setApplication(applicationConfig);
    }

    /**
     * 配置ModuleConfig
     * @param bean
     */
    private void configureModuleConfig(B bean) {
        //获取ModuleConfig的bean-names
        String moduleConfigBeanName = resolveModuleConfigBeanName(annotation);

        ModuleConfig moduleConfig =
                getOptionalBean(applicationContext, moduleConfigBeanName, ModuleConfig.class);

        bean.setModule(moduleConfig);

    }

    /**
     * Resolves the bean name of {@link ModuleConfig}
     *
     * @param annotation {@link A}
     * @return
     */
    protected abstract String resolveModuleConfigBeanName(A annotation);

    /**
     * Resolves the bean name of {@link ApplicationConfig}
     *
     * @param annotation {@link A}
     * @return
     */
    protected abstract String resolveApplicationConfigBeanName(A annotation);


    /**
     * Resolves the bean ids of {@link com.alibaba.dubbo.config.RegistryConfig}
     *
     * @param annotation {@link A}
     * @return non-empty array
     */
    protected abstract String[] resolveRegistryConfigBeanNames(A annotation);

    /**
     * Resolves the bean name of {@link MonitorConfig}
     *
     * @param annotation {@link A}
     * @return
     */
    protected abstract String resolveMonitorConfigBeanName(A annotation);

    /**
     * Configures Bean
     *
     * @param annotation
     * @param bean
     */
    protected abstract void postConfigureBean(A annotation, B bean) throws Exception;


    public <T extends AbstractAnnotationConfigBeanBuilder<A, B>> T bean(Object bean) {
        this.bean = bean;
        return (T) this;
    }

    public <T extends AbstractAnnotationConfigBeanBuilder<A, B>> T interfaceClass(Class<?> interfaceClass) {
        this.interfaceClass = interfaceClass;
        return (T) this;
    }
}
```
##### ReferenceBeanBuilder实现类

我们接下来看ReferenceBeanBuilder实现类
```java
/**
 * ReferenceBean 构造器
 */
class ReferenceBeanBuilder extends AbstractAnnotationConfigBeanBuilder<Reference, ReferenceBean> {
	
    private ReferenceBeanBuilder(Reference annotation, ClassLoader classLoader, ApplicationContext applicationContext) {
        super(annotation, classLoader, applicationContext);
    }

    /**
     * 设置referenceBean实例的interfaceClass属性
     * @param reference
     * @param referenceBean
     */
    private void configureInterface(Reference reference, ReferenceBean referenceBean) {

        //获取@Reference注解的interfaceClass属性
        Class<?> interfaceClass = reference.interfaceClass();

        if (void.class.equals(interfaceClass)) {
            interfaceClass = null;
            //interfaceClass属性没有配置,则取interfaceName属性
            String interfaceClassName = reference.interfaceName();
            if (StringUtils.hasText(interfaceClassName)) {
                if (ClassUtils.isPresent(interfaceClassName, classLoader)) {
                    //加载interfaceClassName类，作为interfaceClass
                    interfaceClass = ClassUtils.resolveClassName(interfaceClassName, classLoader);
                }
            }
        }
        if (interfaceClass == null) {
            //如果@Reference注解没有配置interfaceClass属性和interfaceName属性
            //则使用创建对象时使用的interfaceClass
            interfaceClass = this.interfaceClass;
        }
        //校验interfaceClass是接口类型
        Assert.isTrue(interfaceClass.isInterface(),
                "The class of field or method that was annotated @Reference is not an interface!");
        //设置referenceBean对象的interfaceClass
        referenceBean.setInterface(interfaceClass);
    }


    /**
     * 设置referenceBean实例的consumer属性
     * @param reference
     * @param referenceBean
     */
    private void configureConsumerConfig(Reference reference, ReferenceBean<?> referenceBean) {
        //获取@Reference注解的consumer属性
        String consumerBeanName = reference.consumer();
        //从工厂中获取ConsumerConfig实例
        ConsumerConfig consumerConfig = getOptionalBean(applicationContext, consumerBeanName, ConsumerConfig.class);
        //设置referenceBean实例的consumer属性
        referenceBean.setConsumer(consumerConfig);
    }

    @Override
    protected ReferenceBean doBuild() {
        //创建ReferenceBean对象
        return new ReferenceBean<Object>();
    }

    @Override
    protected void preConfigureBean(Reference reference, ReferenceBean referenceBean) {
        Assert.notNull(interfaceClass, "The interface class must set first!");

        //根据referenceBean创建数据绑定对象
        DataBinder dataBinder = new DataBinder(referenceBean);
        //设置转换器
        dataBinder.setConversionService(getConversionService());
        //忽略的属性名称
        String[] ignoreAttributeNames = of("application", "module", "consumer", "monitor", "registry");
        //dataBinder.setDisallowedFields(ignoreAttributeNames)
        //绑定注解属性
        dataBinder.bind(new AnnotationPropertyValuesAdapter(reference, applicationContext.getEnvironment(), ignoreAttributeNames));
    }

    /**
     * 获取ConversionService
     * @return
     */
    private ConversionService getConversionService() {
        //创建默认转换器
        DefaultConversionService conversionService = new DefaultConversionService();
        //添加StringArray到String的转换
        conversionService.addConverter(new StringArrayToStringConverter());
        //添加StringArray到Map的转换
        conversionService.addConverter(new StringArrayToMapConverter());
        return conversionService;
    }


    @Override
    protected String resolveModuleConfigBeanName(Reference annotation) {
        //获取@Reference注解的module属性
        return annotation.module();
    }

    @Override
    protected String resolveApplicationConfigBeanName(Reference annotation) {
        //获取@Reference注解的application属性
        return annotation.application();
    }

    @Override
    protected String[] resolveRegistryConfigBeanNames(Reference annotation) {
        //获取@Reference注解的registry属性
        return annotation.registry();
    }

    @Override
    protected String resolveMonitorConfigBeanName(Reference annotation) {
        //获取@Reference注解的monitor属性
        return annotation.monitor();
    }

    @Override
    protected void postConfigureBean(Reference annotation, ReferenceBean bean) throws Exception {

        //设置applicationContext属性
        bean.setApplicationContext(applicationContext);
        //设置interfaceClass属性
        configureInterface(annotation, bean);
        //设置consumer属性
        configureConsumerConfig(annotation, bean);
        //属性都设置完了，开始调用bean的afterPropertiesSet方法(可见ReferenceBean实现了InitializingBean接口)
        //在afterPropertiesSet()方法中会校验consumer/application/module/registry/monitor等属性是否为空，
	//为空的话,会从Spring容器中再次获取下,并重新赋值,然后会根据配置是否立即初始化该bean
	bean.afterPropertiesSet();
    }

    /**
     * 创建ReferenceBeanBuilder实例
     * @param annotation
     * @param classLoader
     * @param applicationContext
     * @return
     */
    public static ReferenceBeanBuilder create(Reference annotation, ClassLoader classLoader,
                                              ApplicationContext applicationContext) {
        return new ReferenceBeanBuilder(annotation, classLoader, applicationContext);
    }
}
```

##### ReferenceBean类

接下来来看ReferenceBean类,该类继承自ReferenceConfig类,并实现了FactoryBean接口。
```java
/**
 * ReferenceFactoryBean
 * @export
 */
public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean,
        ApplicationContextAware, InitializingBean, DisposableBean {

    private transient ApplicationContext applicationContext;

    public ReferenceBean() {
        super();
    }

    public ReferenceBean(Reference reference) {
        //调用父类构造方法，在父类构造方法中会调用appendAnnotation方法处理@Reference注解的属性
        super(reference);
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
        //将applicationContext添加到SpringExtensionFactory中
        //SPI那一章节，我们介绍过SpringExtensionFactory类
        SpringExtensionFactory.addApplicationContext(applicationContext);
    }

    @Override
    public Object getObject() throws Exception {
        //获取对象实例，调用父类中的get方法进行初始化
        return get();
    }

    @Override
    public Class<?> getObjectType() {
        //调用父类中的getInterfaceClass方法获取对象类型(后面会分析该方法)
        return getInterfaceClass();
    }

    @Override
    @Parameter(excluded = true)
    public boolean isSingleton() {
        return true;
    }

    @Override
    @SuppressWarnings({"unchecked"})
    public void afterPropertiesSet() throws Exception {
        //如果没有配置Consumer
        if (getConsumer() == null) {
            //获取IOC容器中的ConsumerConfig（只获取单例类型的）
            Map<String, ConsumerConfig> consumerConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ConsumerConfig.class, false, false);
            if (consumerConfigMap != null && consumerConfigMap.size() > 0) {
                ConsumerConfig consumerConfig = null;
                for (ConsumerConfig config : consumerConfigMap.values()) {
                    //检查是否有重复的consumer配置(default属性)
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        if (consumerConfig != null) {
                            throw new IllegalStateException("Duplicate consumer configs: " + consumerConfig + " and " + config);
                        }
                        consumerConfig = config;
                    }
                }
                if (consumerConfig != null) {
                    //配置bean的consumer属性
                    setConsumer(consumerConfig);
                }
            }
        }
        if (getApplication() == null
                && (getConsumer() == null || getConsumer().getApplication() == null)) {
            //从IOC容器中获取ApplicationConfig
            Map<String, ApplicationConfig> applicationConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ApplicationConfig.class, false, false);
            if (applicationConfigMap != null && applicationConfigMap.size() > 0) {
                ApplicationConfig applicationConfig = null;
                for (ApplicationConfig config : applicationConfigMap.values()) {
                    //检测是否有重复的配置(default属性)
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        if (applicationConfig != null) {
                            throw new IllegalStateException("Duplicate application configs: " + applicationConfig + " and " + config);
                        }
                        applicationConfig = config;
                    }
                }
                if (applicationConfig != null) {
                    //设置bean的ApplicationConfig属性
                    setApplication(applicationConfig);
                }
            }
        }
        if (getModule() == null
                && (getConsumer() == null || getConsumer().getModule() == null)) {
            //从IOC容器中获取ModuleConfig
            Map<String, ModuleConfig> moduleConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ModuleConfig.class, false, false);
            if (moduleConfigMap != null && moduleConfigMap.size() > 0) {
                ModuleConfig moduleConfig = null;
                for (ModuleConfig config : moduleConfigMap.values()) {
                    //检测是否有重复的moduleConfig
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        if (moduleConfig != null) {
                            throw new IllegalStateException("Duplicate module configs: " + moduleConfig + " and " + config);
                        }
                        moduleConfig = config;
                    }
                }
                if (moduleConfig != null) {
                    //设置module属性
                    setModule(moduleConfig);
                }
            }
        }
        if ((getRegistries() == null || getRegistries().isEmpty())
                && (getConsumer() == null || getConsumer().getRegistries() == null || getConsumer().getRegistries().isEmpty())
                && (getApplication() == null || getApplication().getRegistries() == null || getApplication().getRegistries().isEmpty())) {
            //从IOC容器中获取RegistryConfig
            Map<String, RegistryConfig> registryConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, RegistryConfig.class, false, false);
            if (registryConfigMap != null && registryConfigMap.size() > 0) {
                List<RegistryConfig> registryConfigs = new ArrayList<RegistryConfig>();
                        registryConfigs.add(config);
                    }
                }
                if (registryConfigs != null && !registryConfigs.isEmpty()) {
                    //设置registries属性
                    super.setRegistries(registryConfigs);
                }
            }
        }
        if (getMonitor() == null
                && (getConsumer() == null || getConsumer().getMonitor() == null)
                && (getApplication() == null || getApplication().getMonitor() == null)) {
            //从IOC容器中获取MonitorConfig
            Map<String, MonitorConfig> monitorConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, MonitorConfig.class, false, false);
            if (monitorConfigMap != null && monitorConfigMap.size() > 0) {
                MonitorConfig monitorConfig = null;
                for (MonitorConfig config : monitorConfigMap.values()) {
                    //检测是否有重复的MonitorConfig
                    if (config.isDefault() == null || config.isDefault().booleanValue()) {
                        if (monitorConfig != null) {
                            throw new IllegalStateException("Duplicate monitor configs: " + monitorConfig + " and " + config);
                        }
                        monitorConfig = config;
                    }
                }
                if (monitorConfig != null) {
                    //设置MonitorConfig属性
                    setMonitor(monitorConfig);
                }
            }
        }
        //获取是否 立即初始化 属性init
        Boolean b = isInit();
        if (b == null && getConsumer() != null) {
            //从ConsumerConfig属性中获取init
            b = getConsumer().isInit();
        }
        if (b != null && b.booleanValue()) {
            //执行初始化
            getObject();
        }
    }

    @Override
    public void destroy() {
        // do nothing
    }
}
```
接下来，我们来看下父类ReferenceConfig中的方法
```java
/**
* 获取服务接口类
* interfaceClass > GenericService.class > interfaceName
* @return
*/
public Class<?> getInterfaceClass() {
	//interfaceClass属性不为空，直接返回
	if (interfaceClass != null) {
	    return interfaceClass;
	}
	//Reference或者Consumer的generic属性为true，则使用GenericService类
	if (isGeneric()
		|| (getConsumer() != null && getConsumer().isGeneric())) {
	    return GenericService.class;
	}
	try {
	    if (interfaceName != null && interfaceName.length() > 0) {
		//interfaceName属性不为空，加载该类
		this.interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
			.getContextClassLoader());
	    }
	} catch (ClassNotFoundException t) {
	    throw new IllegalStateException(t.getMessage(), t);
	}
	return interfaceClass;
}


/**
 * 是否已销毁
 */
private transient volatile boolean destroyed;

/**
 * 接口代理引用
 * interface proxy reference
 */
private transient volatile T ref;


/**
 * 获取接口代理引用
 */
public synchronized T get() {
        //判断是否已经销毁
        if (destroyed) {
            throw new IllegalStateException("Already destroyed!");
        }
        if (ref == null) {
            //初始化
            init();
        }
        return ref;
}
```

#### init方法
该init方法会生成接口代理引用ref。
```java
public synchronized T get() {
        //判断是否已经销毁
        if (destroyed) {
            throw new IllegalStateException("Already destroyed!");
        }
        if (ref == null) {
            //初始化
            init();
        }
        return ref;
}

private void init() {
	//已经初始化过的话，则返回
	if (initialized) {
	    return;
	}
	initialized = true;
	//检测是否配置了interface属性
	if (interfaceName == null || interfaceName.length() == 0) {
	    throw new IllegalStateException("<dubbo:reference interface=\"\" /> interface not allow null!");
	}
	//获取Consumer的全局配置(设置ConsumerConfig对象的属性)(后面会分析该方法)
	checkDefault();
	//加载当前ReferenceConfig对象(即ReferenceBean)的属性信息(后面会分析该方法)
	appendProperties(this);
	//当前ReferenceConfig没有配置generic属性的话，则使用Consumer的generic属性配置
	if (getGeneric() == null && getConsumer() != null) {
	    //设置generic属性
	    setGeneric(getConsumer().getGeneric());
	}
	//(generic ！= null) && (generic = “true” || "nativejava" || "bean") ProtocolUtils.isGeneric返回true
	if (ProtocolUtils.isGeneric(getGeneric())) {
	    //如果配置了使用通用接口，则设置服务接口类为GenericService类
	    interfaceClass = GenericService.class;
	} else {
	    //否则根据interfaceName加载服务接口类
	    try {
		interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
			.getContextClassLoader());
	    } catch (ClassNotFoundException e) {
		throw new IllegalStateException(e.getMessage(), e);
	    }
	    //检测methods方法是否都存在于接口服务类interfaceClass中(后面会分析该方法)
	    checkInterfaceAndMethods(interfaceClass, methods);
	}
	//1、根据服务接口名称从系统配置中查找resolve：即通过-DinterfaceName=resolve配置
	//2、resolve为空，则加载dubbo.resolve.file文件,从resolveFile中加载resolve
	//   a、从系统配置文件中加载resolve文件：即-Ddubbo.resolve.file=xxx配置
	//   b、使用dubbo的默认resolve文件：${user.home}/dubbo-resolve.properties
	String resolve = System.getProperty(interfaceName);
	String resolveFile = null;
	if (resolve == null || resolve.length() == 0) {
	    resolveFile = System.getProperty("dubbo.resolve.file");
	    if (resolveFile == null || resolveFile.length() == 0) {
		//没有配置resolveFile的话，则使用dubbo默认的resolve文件
		//如：System.getProperty("user.home") = C:\Users\Administrator
		//resolveFile = userResolveFile = C:\Users\Administrator\dubbo-resolve.properties
		File userResolveFile = new File(new File(System.getProperty("user.home")), "dubbo-resolve.properties");
		if (userResolveFile.exists()) {
		    resolveFile = userResolveFile.getAbsolutePath();
		}
	    }
	    //加载resolveFile文件
	    if (resolveFile != null && resolveFile.length() > 0) {
		Properties properties = new Properties();
		FileInputStream fis = null;
		try {
		    fis = new FileInputStream(new File(resolveFile));
		    properties.load(fis);
		} catch (IOException e) {
		    throw new IllegalStateException("Unload " + resolveFile + ", cause: " + e.getMessage(), e);
		} finally {
		    try {
			if (null != fis) {
			    fis.close();
			}
		    } catch (IOException e) {
			logger.warn(e.getMessage(), e);
		    }
		}
		//从resolveFile配置文件中加载resolve
		resolve = properties.getProperty(interfaceName);
	    }
	}
	if (resolve != null && resolve.length() > 0) {
	    url = resolve;
	    if (logger.isWarnEnabled()) {
		if (resolveFile != null && resolveFile.length() > 0) {
		    //使用默认dubbo resolve file
		    logger.warn("Using default dubbo resolve file " + resolveFile + " replace " + interfaceName + "" + resolve + " to p2p invoke remote service.");
		} else {
		    //可以使用-DinterfaceName=resolve配置p2p
		    logger.warn("Using -D" + interfaceName + "=" + resolve + " to p2p invoke remote service.");
		}
	    }
	}
	//加载注册中心、监控中心优先级
	//consumer > module > application
	if (consumer != null) {
	    if (application == null) {
		application = consumer.getApplication();
	    }
	    if (module == null) {
		module = consumer.getModule();
	    }
	    if (registries == null) {
		registries = consumer.getRegistries();
	    }
	    if (monitor == null) {
		monitor = consumer.getMonitor();
	    }
	}
	if (module != null) {
	    if (registries == null) {
		registries = module.getRegistries();
	    }
	    if (monitor == null) {
		monitor = module.getMonitor();
	    }
	}
	if (application != null) {
	    //<dubbo:application name="demo-consumer" qosPort="33333" id="demo-consumer" />
	    if (registries == null) {
		registries = application.getRegistries();
	    }
	    if (monitor == null) {
		monitor = application.getMonitor();
	    }
	}
	//检测Application配置(后面会分析该方法)
	checkApplication();
	//检测Stub和Mock(后面会分析该方法)
	checkStubAndMock(interfaceClass);

	//记录Consumer端的服务接口相关信息
	Map<String, String> map = new HashMap<String, String>();
	//保存方法属性的attributes
	Map<Object, Object> attributes = new HashMap<Object, Object>();
	//设置消费者端
	map.put(Constants.SIDE_KEY, Constants.CONSUMER_SIDE);
	//设置dubbo版本
	map.put(Constants.DUBBO_VERSION_KEY, Version.getVersion());
	//设置时间戳
	map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
	if (ConfigUtils.getPid() > 0) {
	    //设置父进程id
	    map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
	}
	if (!isGeneric()) {
	    //不是通用泛型接口，则获取revision属性,默认等于version属性
	    String revision = Version.getVersion(interfaceClass, version);
	    if (revision != null && revision.length() > 0) {
		map.put("revision", revision);
	    }
	    //获取包装类，然后获取服务接口中的方法名称数组(后面会分析该方法)
	    String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
	    if (methods.length == 0) {
		//在服务接口中，没有找到方法，则设置属性：methods = *
		logger.warn("NO method found in service interface " + interfaceClass.getName());
		map.put("methods", Constants.ANY_VALUE);
	    } else {
		//设置methods=逗号分隔的方法name
		map.put("methods", StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
	    }
	}
	//设置interface属性
	map.put(Constants.INTERFACE_KEY, interfaceName);
	//添加附加参数(后面会分析该方法)
	appendParameters(map, application);
	appendParameters(map, module);
	appendParameters(map, consumer, Constants.DEFAULT_KEY);
	appendParameters(map, this);
	//获取服务前缀：group/interface:version
	String prefix = StringUtils.getServiceKey(map);
	//处理引用的方法列表
	if (methods != null && !methods.isEmpty()) {
	    for (MethodConfig method : methods) {
		//附加参数(后面会分析该方法)
		appendParameters(map, method, method.getName());
		//处理方法重试
		String retryKey = method.getName() + ".retry";
		if (map.containsKey(retryKey)) {
		    //将属性retryKey从map中移除
		    String retryValue = map.remove(retryKey);
		    if ("false".equals(retryValue)) {
			//属性retryKey的值为false,则添加新的属性到map中(即禁用方法重试)
			map.put(method.getName() + ".retries", "0");
		    }
		}
		//附加属性(后面会分析该方法)
		appendAttributes(attributes, method, prefix + "." + method.getName());
		//attributes保存的是方法名，调用checkAndConvertImplicitConfig方法后将会保存Method实例
		checkAndConvertImplicitConfig(method, map, attributes);
	    }
	}
	//从系统配置属性中获取：注册到注册中心的ip
	String hostToRegistry = ConfigUtils.getSystemProperty(Constants.DUBBO_IP_TO_REGISTRY);
	if (hostToRegistry == null || hostToRegistry.length() == 0) {
	    //获取本地地址,如：192.168.99.60
	    hostToRegistry = NetUtils.getLocalHost();
	} else if (isInvalidLocalHost(hostToRegistry)) {
	    //无效的地址
	    throw new IllegalArgumentException("Specified invalid registry ip from property:" + Constants.DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
	}
	//设置属性：注册到注册中心的ip
	map.put(Constants.REGISTER_IP_KEY, hostToRegistry);

	//保存方法属性到系统上下文中
	StaticContext.getSystemContext().putAll(attributes);
	
	//根据服务接口及其属性创建代理(后面会分析该方法)
	ref = createProxy(map);
	
	//构造ConsumerModel对象,getUniqueServiceName()生成唯一服务名称
	ConsumerModel consumerModel = new ConsumerModel(getUniqueServiceName(), this, ref, interfaceClass.getMethods());
	
	//根据服务名称注册消费者(后面会分析该方法)
	ApplicationModel.initConsumerModel(getUniqueServiceName(), consumerModel);
}
```
接下来，我们看上面使用到的方法

##### checkDefault方法
获取Consumer的全局配置
```java
private void checkDefault() {
        if (consumer == null) {
	    //consumer为空，会新创建一个ConsumerConfig对象
            consumer = new ConsumerConfig();
        }
	//调用appendProperties方法
        appendProperties(consumer);
}
```
##### appendProperties方法
该方法在init方法中大量出现，我们先看下它的实现，它是定义在AbstractConfig抽象类中的静态方法，参数是AbstractConfig类型。
```java
/**
 * 附加属性
 * 找到config的所有的setXXX方法，然后切割出属性XXX，然后从系统配置/dubbo配置文件中获取到属性的值，
 * 然后执行setXXX方法把属性值设置到config对象中
 * @param config
 */
protected static void appendProperties(AbstractConfig config) {
	if (config == null) {
	   //config为空，直接返回
	   return;
	}
	//生成前缀，如： dubbo.monitor.
	String prefix = "dubbo." + getTagName(config.getClass()) + ".";
	//获取config类的所有方法
	Method[] methods = config.getClass().getMethods();
	//遍历方法列表
	for (Method method : methods) {
	    try {
		String name = method.getName();
		//方法名长度>3，以set开头，修饰符是public，参数个数=1，参数类型是原始类型
		if (name.length() > 3 && name.startsWith("set") && Modifier.isPublic(method.getModifiers())
			&& method.getParameterTypes().length == 1 && isPrimitive(method.getParameterTypes()[0])) {
		    //转换方法名称为属性名称，如：setFirstName将会转换成first.name
		    //例如：property = default
		    String property = StringUtils.camelToSplitName(name.substring(3, 4).toLowerCase() + name.substring(4), ".");

		    //1、从系统配置中获取：prefix + id + property
		    //2、从系统配置中获取：prefix + property
		    //3、从config类中找到属性get方法或者is方法，然后执行该方法
			//3.1、从config配置中获取 prefix + id + property属性
			//3.2、从config配置中获取 prefix + property属性
			//3.3、根据prefix + property获取旧版属性legacyKey,然后根据legacyKey从config配置中获取
		    //ConfigUtils.getProperty(后面会分析该方法)
		    //如果属性值value不为空的话，则调用该set方法，将属性设置到config对象中，整个方法流程执行结束
			
		    String value = null;
		    if (config.getId() != null && config.getId().length() > 0) {
			//如果id属性不为空，则属性名称为：prefix + id + property
			//例如：pn = dubbo.monitor."id".default
			String pn = prefix + config.getId() + "." + property;
			//从系统配置中获取属性pn的值value
			value = System.getProperty(pn);
			if (!StringUtils.isBlank(value)) {
			    //如果系统中设置了pn属性，则使用该属性值
			    logger.info("Use System Property " + pn + " to config dubbo");
			}
		    }
		    if (value == null || value.length() == 0) {
			//属性名称为：prefix + property
			//例如：dubbo.monitor.default
			String pn = prefix + property;
			//获取属性pn的系统属性值value
			value = System.getProperty(pn);
			if (!StringUtils.isBlank(value)) {
			    logger.info("Use System Property " + pn + " to config dubbo");
			}
		    }
		    if (value == null || value.length() == 0) {
			Method getter;
			try {
			    //获取当前属性的get方法
			    getter = config.getClass().getMethod("get" + name.substring(3), new Class<?>[0]);
			} catch (NoSuchMethodException e) {
			    try {
				//get方法不存在的话，则获取下is方法
				getter = config.getClass().getMethod("is" + name.substring(3), new Class<?>[0]);
			    } catch (NoSuchMethodException e2) {
				getter = null;
			    }
			}
			if (getter != null) {
			    //执行config对象的属性的getter方法，结果为空的话
			    if (getter.invoke(config, new Object[0]) == null) {
				if (config.getId() != null && config.getId().length() > 0) {
				    //id属性不为空
				    //如：dubbo.monitor."id".default
				    value = ConfigUtils.getProperty(prefix + config.getId() + "." + property);
				}
				if (value == null || value.length() == 0) {
				    //如：dubbo.monitor.default
				    value = ConfigUtils.getProperty(prefix + property);
				}
				if (value == null || value.length() == 0) {
				    //从旧版属性中获取(如：dubbo.monitor.default)
				    String legacyKey = legacyProperties.get(prefix + property);
				    if (legacyKey != null && legacyKey.length() > 0) {
					//转换旧版属性值
					value = convertLegacyValue(legacyKey, ConfigUtils.getProperty(legacyKey));
				    }
				}

			    }
			}
		    }
		    if (value != null && value.length() > 0) {
			//如果属性值不为空的话，则使用属性值作为参数执行config类的method方法
			method.invoke(
				config,
				new Object[]{
					//转换原始类型
					convertPrimitive(method.getParameterTypes()[0], value)
				}

			);
		    }
		}
	    } catch (Exception e) {
		logger.error(e.getMessage(), e);
	    }
	}
}
```

##### checkInterfaceAndMethods方法
```java
/**
* 判断方法methods是否都在服务接口类中存在
* @param interfaceClass 服务接口类
* @param methods 引用的服务接口方法
*/
protected void checkInterfaceAndMethods(Class<?> interfaceClass, List<MethodConfig> methods) {
	if (interfaceClass == null) {
	    //服务接口不可以为空
	    throw new IllegalStateException("interface not allow null!");
	}
	if (!interfaceClass.isInterface()) {
	    // interfaceClass必须为接口类型
	    throw new IllegalStateException("The interface class " + interfaceClass + " is not a interface!");
	}
	// 检查methods方法是否存在于interfaceClass接口中
	if (methods != null && !methods.isEmpty()) {
	    //遍历引用的方法
	    for (MethodConfig methodBean : methods) {
		//方法名
		String methodName = methodBean.getName();
		if (methodName == null || methodName.length() == 0) {
		    //<dubbo:method>标签的name属性必须设置
		    throw new IllegalStateException("<dubbo:method> name attribute is required! Please check: <dubbo:service interface=\"" + interfaceClass.getName() + "\" ... ><dubbo:method name=\"\" ... /></<dubbo:reference>");
		}
		boolean hasMethod = false;
		for (java.lang.reflect.Method method : interfaceClass.getMethods()) {
		    //方法名一样，则认为存在该方法
		    if (method.getName().equals(methodName)) {
			hasMethod = true;
			break;
		    }
		}
		if (!hasMethod) {
		    //引用的方法在服务接口中不存在
		    throw new IllegalStateException("The interface " + interfaceClass.getName()
			    + " not found method " + methodName);
		}
	    }
	}
}
```
##### checkApplication方法
该方法用来验证application,application为空的话，会新建ApplicationConfig对象
```java
protected void checkApplication() {
	// 处理兼容
	if (application == null) {
	    //从配置文件中加载application.name属性
	    String applicationName = ConfigUtils.getProperty("dubbo.application.name");
	    if (applicationName != null && applicationName.length() > 0) {
		//新创建一个ApplicationConfig对象
		application = new ApplicationConfig();
	    }
	}
	if (application == null) {
	    //需要配置 <dubbo:application name=\"...\" />
	    throw new IllegalStateException(
		    "No such application config! Please add <dubbo:application name=\"...\" /> to your spring config.");
	}
	//添加application的属性
	appendProperties(application);
	//获取配置: 服务停止时的等待时间(优先获取SHUTDOWN_WAIT_KEY属性，在获取SHUTDOWN_WAIT_SECONDS_KEY(废弃))
	String wait = ConfigUtils.getProperty(Constants.SHUTDOWN_WAIT_KEY);
	if (wait != null && wait.trim().length() > 0) {
	    //设置到系统配置中
	    System.setProperty(Constants.SHUTDOWN_WAIT_KEY, wait.trim());
	} else {
	    wait = ConfigUtils.getProperty(Constants.SHUTDOWN_WAIT_SECONDS_KEY);
	    if (wait != null && wait.trim().length() > 0) {
		System.setProperty(Constants.SHUTDOWN_WAIT_SECONDS_KEY, wait.trim());
	    }
	}
}
```

##### checkStubAndMock方法
该方法用来检测: 服务接口本地实现类、服务接口本地存根类、mock
```java
/**
* 检测local/stub/mock的配置是否正确
* local/stub/mock(mock 可配置为"return ")应该为interfaceClass或者为interfaceClass子类
* @param interfaceClass 服务接口类
*/
protected void checkStubAndMock(Class<?> interfaceClass) {
	//处理服务接口本地实现类
	if (ConfigUtils.isNotEmpty(local)) {
	    //如果local = true或者local = default(即local属性配置),则本地实现类名为：接口名称+Local,否则本地实现类名为：local属性
	    //加载本地实现类
	    Class<?> localClass = ConfigUtils.isDefault(local) ? ReflectUtils.forName(interfaceClass.getName() + "Local") : ReflectUtils.forName(local);
	    //检查本地实现类是否实现了服务接口
	    if (!interfaceClass.isAssignableFrom(localClass)) {
		throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceClass.getName());
	    }
	    try {
		//检测本地实现类中是否存在相应的构造方法，即：public 本地实现类名 (服务接口名){}
		ReflectUtils.findConstructor(localClass, interfaceClass);
	    } catch (NoSuchMethodException e) {
		throw new IllegalStateException("No such constructor \"public " + localClass.getSimpleName() + "(" + interfaceClass.getName() + ")\" in local implementation class " + localClass.getName());
	    }
	}
	//处理服务接口本地存根类
	if (ConfigUtils.isNotEmpty(stub)) {
	    //如果stub = true或者stub = default(即stub属性)，则本地存根类名为：接口名称+Stub,否则本地存根类名为：stub属性
	    //加载本地存根类
	    Class<?> localClass = ConfigUtils.isDefault(stub) ? ReflectUtils.forName(interfaceClass.getName() + "Stub") : ReflectUtils.forName(stub);
	    //检测本地存根类是否实现了服务接口
	    if (!interfaceClass.isAssignableFrom(localClass)) {
		throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceClass.getName());
	    }
	    try {
		//检测本地存根类是否存在相应的构造方法, 即：public 本地存根类名 (服务接口名){}
		ReflectUtils.findConstructor(localClass, interfaceClass);
	    } catch (NoSuchMethodException e) {
		throw new IllegalStateException("No such constructor \"public " + localClass.getSimpleName() + "(" + interfaceClass.getName() + ")\" in local implementation class " + localClass.getName());
	    }
	}
	//处理mock
	if (ConfigUtils.isNotEmpty(mock)) {
	    //判断mock属性是否以"return "开头
	    if (mock.startsWith(Constants.RETURN_PREFIX)) {
		//获取到"return "之后的值
		String value = mock.substring(Constants.RETURN_PREFIX.length());
		try {
		    //解析mock值
		    MockInvoker.parseMockValue(value);
		} catch (Exception e) {
		    throw new IllegalStateException("Illegal mock json value in <dubbo:service ... mock=\"" + mock + "\" />");
		}
	    } else {
		//获取mock类
		//如果mock = true或者mock = default(即mock属性)，则本地存根类名为：接口名称+Mock,否则本地存根类名为：mock属性
		Class<?> mockClass = ConfigUtils.isDefault(mock) ? ReflectUtils.forName(interfaceClass.getName() + "Mock") : ReflectUtils.forName(mock);
		//检测mock类是否实现了服务接口
		if (!interfaceClass.isAssignableFrom(mockClass)) {
		    throw new IllegalStateException("The mock implementation class " + mockClass.getName() + " not implement interface " + interfaceClass.getName());
		}
		try {
		    //检测默认构造方法，即：pulic mock类名 () {}
		    mockClass.getConstructor(new Class<?>[0]);
		} catch (NoSuchMethodException e) {
		    throw new IllegalStateException("No such empty constructor \"public " + mockClass.getSimpleName() + "()\" in mock implementation class " + mockClass.getName());
		}
	    }
	}
}
```
##### appendParameters方法
```java
/**
* 附加参数
* 1、获取config对象的方法列表，
* 2、找到getXXX或者isXXX方法或者getParameters方法(会通过方法上的@Parameter注解判断是否需要过滤该属性)
* 3、然后执行该方法，拿到方法返回值(会通过方法上的@Parameter注解判断是否需要编码及追加)。
* 然后将属性(@Parameter注解配置或者通过getXXX获取，如果配置了prefix，则属性为: prefix.属性)
* 以及值保存到parameters中
*
* @param parameters 当前参数Map
* @param config  目标配置对象
* @param prefix 属性配置前缀
*/
protected static void appendParameters(Map<String, String> parameters, Object config, String prefix) {
	if (config == null) {
	    return;
	}
	//获取配置类config的所有方法
	Method[] methods = config.getClass().getMethods();
	for (Method method : methods) {
	    try {
		String name = method.getName();
		//方法名以get或者is开头，并且方法名不为getClass，并且存在public修饰符
		//并且不存在参数，并且方法返回类型为原始类型
		if ((name.startsWith("get") || name.startsWith("is"))
			&& !"getClass".equals(name)
			&& Modifier.isPublic(method.getModifiers())
			&& method.getParameterTypes().length == 0
			&& isPrimitive(method.getReturnType())) {
		    //获取方法上的@Parameter注解
		    Parameter parameter = method.getAnnotation(Parameter.class);
		    if (method.getReturnType() == Object.class || parameter != null && parameter.excluded()) {
			//如果当前方法的返回类型为Object，或者该方法存在@Parameter注解且excluded属性值为true
			//则跳过该方法，即忽略该属性
			continue;
		    }
		    int i = name.startsWith("get") ? 3 : 2;
		    //根据方法名获取到属性名，如：getFirstName被转换成：first.name
		    String prop = StringUtils.camelToSplitName(name.substring(i, i + 1).toLowerCase() + name.substring(i + 1), ".");
		    //key为@Parameter注解的key属性或者为根据当前方法名截取到的属性名
		    String key;
		    if (parameter != null && parameter.key() != null && parameter.key().length() > 0) {
			//如果@Parameter注解的key属性不为空，则使用key属性
			key = parameter.key();
		    } else {
			//否则使用prop作为key
			key = prop;
		    }
		    //执行config对象的method方法，获取到方法返回值
		    Object value = method.invoke(config, new Object[0]);
		    //将方法返回值转换成字符串，并去掉空格
		    String str = String.valueOf(value).trim();
		    if (value != null && str.length() > 0) {
			//方法返回值不为空
			if (parameter != null && parameter.escaped()) {
			    //根据@Parameter注解的escaped属性来决定是否需要对属性值进行编码
			    str = URL.encode(str);
			}
			if (parameter != null && parameter.append()) {
			    //如果配置了@Parameter注解的append属性为true
			    //则从参数Map中获取key对应的值pre
			    String pre = parameters.get(Constants.DEFAULT_KEY + "." + key);
			    if (pre != null && pre.length() > 0) {
				//追加方法返回值
				str = pre + "," + str;
			    }
			    pre = parameters.get(key);
			    if (pre != null && pre.length() > 0) {
				//追加方法返回值
				str = pre + "," + str;
			    }
			}
			if (prefix != null && prefix.length() > 0) {
			    //如果前缀不为空，则拼接key
			    key = prefix + "." + key;
			}
			//将key、value保存到参数Map中
			parameters.put(key, str);
		    } else if (parameter != null && parameter.required()) {
			//如果方法返回值为空，且@Parameter注解的required属性为true
			//则抛出异常，提示config类的key属性的值为空
			throw new IllegalStateException(config.getClass().getSimpleName() + "." + key + " == null");
		    }
		} else if ("getParameters".equals(name)
			&& Modifier.isPublic(method.getModifiers())
			&& method.getParameterTypes().length == 0
			&& method.getReturnType() == Map.class) {
		    //方法名称为getParameters，且方法有public修饰符，且参数为空，且方法返回值为Map
		    //则执行config对象的getParameters方法，获取到返回值Map
		    Map<String, String> map = (Map<String, String>) method.invoke(config, new Object[0]);
		    if (map != null && map.size() > 0) {
			//格式化前缀，前缀以“.”结尾
			String pre = (prefix != null && prefix.length() > 0 ? prefix + "." : "");
			//遍历getParameters方法返回值
			for (Map.Entry<String, String> entry : map.entrySet()) {
			    //将属性以及属性值保存到parameters中
			    parameters.put(pre + entry.getKey().replace('-', '.'), entry.getValue());
			}
		    }
		}
	    } catch (Exception e) {
		throw new IllegalStateException(e.getMessage(), e);
	    }
	}
}
```

##### appendAttributes方法
```java
/**
* 附加属性
* 1、获取config对象的getXXX或者isXXX方法(会通过方法上的@Parameter注解判断是否需要过滤该属性)，
* 然后调用该方法获取返回值value。
* 2、然后根据@Parameter注解的key属性或者当前方法名称(取消get/is前缀)作为key（prefix+"."+key），
* 3、最终将key和value保存到parameters中
* @param parameters 保存方法属性
* @param config MethodConfig等对象
* @param prefix
*/
protected static void appendAttributes(Map<Object, Object> parameters, Object config, String prefix) {
	if (config == null) {
	    return;
	}
	Method[] methods = config.getClass().getMethods();
	//遍历config对象的方法
	for (Method method : methods) {
	    try {
		//方法名
		String name = method.getName();
		//找到get方法或者is方法
		if ((name.startsWith("get") || name.startsWith("is"))
			&& !"getClass".equals(name)
			&& Modifier.isPublic(method.getModifiers())
			&& method.getParameterTypes().length == 0
			&& isPrimitive(method.getReturnType())) {
		    //获取方法上的@Parameter注解
		    Parameter parameter = method.getAnnotation(Parameter.class);
		    if (parameter == null || !parameter.attribute()) {
			//如果该方法没有@Parameter注解，或者@Parameter注解的attribute属性为false
			//则跳过该方法
			continue;
		    }
		    //优先获取@Parameter注解的key属性，否则获取当前方法名(去掉get或者is前缀)
		    String key;
		    if (parameter.key() != null && parameter.key().length() > 0) {
			key = parameter.key();
		    } else {
			int i = name.startsWith("get") ? 3 : 2;
			key = name.substring(i, i + 1).toLowerCase() + name.substring(i + 1);
		    }
		    //执行config对象的当前方法，获取方法返回值
		    Object value = method.invoke(config, new Object[0]);
		    if (value != null) {
			if (prefix != null && prefix.length() > 0) {
			    //前缀不为空的话，拼接上前缀
			    key = prefix + "." + key;
			}
			//将属性和属性值放到parameters中
			parameters.put(key, value);
		    }
		}
	    } catch (Exception e) {
		throw new IllegalStateException(e.getMessage(), e);
	    }
	}
}
```
##### checkAndConvertImplicitConfig方法
```java
/**
* 处理onreturn、onthrow、oninvoke属性，将attributes中的value，从"方法名称"转换成"方法对象"
* 如：将<onReturnMethodKey,onReturnMethod方法名>转换为<onReturnMethodKey,onReturnMethod方法对象>
* @param method 当前方法配置
* @param map 当前所有属性map
* @param attributes 当前方法的属性map
*/
private static void checkAndConvertImplicitConfig(MethodConfig method, Map<String, String> map, Map<Object, Object> attributes) {
	//检测配置冲突
	if (Boolean.FALSE.equals(method.isReturn()) && (method.getOnreturn() != null || method.getOnthrow() != null)) {
	    //当设置了onreturn或者onthrow时，必须同时设置isReturn为true
	    throw new IllegalStateException("method config error : return attribute must be set true when onreturn or onthrow has been setted.");
	}

	//attributes存的是<onReturnMethodKey,onReturnMethod方法名>
	//经过转换后存的是<onReturnMethodKey,onReturnMethod方法类>

	//onReturnMethodKey是属性key
	String onReturnMethodKey = StaticContext.getKey(map, method.getName(), Constants.ON_RETURN_METHOD_KEY);
	//从attributes中获取属性onReturnMethodKey对应的值(方法名)
	Object onReturnMethod = attributes.get(onReturnMethodKey);
	if (onReturnMethod != null && onReturnMethod instanceof String) {
	    //getMethodByName方法根据方法名onReturnMethod从类method.getOnreturn().getClass()中找到相应的方法
	    attributes.put(onReturnMethodKey, getMethodByName(method.getOnreturn().getClass(), onReturnMethod.toString()));
	}
	//下面的类似
	//convert onthrow methodName to Method
	String onThrowMethodKey = StaticContext.getKey(map, method.getName(), Constants.ON_THROW_METHOD_KEY);
	Object onThrowMethod = attributes.get(onThrowMethodKey);
	if (onThrowMethod != null && onThrowMethod instanceof String) {
	    attributes.put(onThrowMethodKey, getMethodByName(method.getOnthrow().getClass(), onThrowMethod.toString()));
	}
	//convert oninvoke methodName to Method
	String onInvokeMethodKey = StaticContext.getKey(map, method.getName(), Constants.ON_INVOKE_METHOD_KEY);
	Object onInvokeMethod = attributes.get(onInvokeMethodKey);
	if (onInvokeMethod != null && onInvokeMethod instanceof String) {
	    attributes.put(onInvokeMethodKey, getMethodByName(method.getOninvoke().getClass(), onInvokeMethod.toString()));
	}
}
```

##### StaticContext类
init方法中，最终会将方法的属性添加到StaticContext中
```java
StaticContext.getSystemContext().putAll(attributes);


/**
 * 系统上下文，只是框架内部使用
 * System context, for internal use only
 */
public class StaticContext extends ConcurrentHashMap<Object, Object> {
    private static final long serialVersionUID = 1L;
    private static final String SYSTEMNAME = "system";
    private static final ConcurrentMap<String, StaticContext> context_map = new ConcurrentHashMap<String, StaticContext>();
    private String name;

    private StaticContext(String name) {
        super();
        this.name = name;
    }

    /**
     * 获取系统上下文,即key为system
     * @return
     */
    public static StaticContext getSystemContext() {
        return getContext(SYSTEMNAME);
    }

    /**
     * 根据name获取上下文
     * @param name
     * @return
     */
    public static StaticContext getContext(String name) {
        //通过name从context_map中获取StaticContext
        StaticContext appContext = context_map.get(name);
        if (appContext == null) {
            //没有获取到StaticContext，则为name新建一个StaticContext，然后放入context_map中
            appContext = context_map.putIfAbsent(name, new StaticContext(name));
            if (appContext == null) {
                appContext = context_map.get(name);
            }
        }
        return appContext;
    }

    /**
     * 从context_map中移除name对应的上下文
     * @param name
     * @return
     */
    public static StaticContext remove(String name) {
        return context_map.remove(name);
    }

    public static String getKey(URL url, String methodName, String suffix) {
        return getKey(url.getServiceKey(), methodName, suffix);
    }

    public static String getKey(Map<String, String> paras, String methodName, String suffix) {
        return getKey(StringUtils.getServiceKey(paras), methodName, suffix);
    }

    /**
     * 获取唯一标识
     * @param servicekey 服务唯一标识
     * @param methodName 方法名
     * @param suffix  前缀
     * @return
     */
    private static String getKey(String servicekey, String methodName, String suffix) {
        StringBuffer sb = new StringBuffer().append(servicekey).append(".").append(methodName).append(".").append(suffix);
        return sb.toString();
    }

    public String getName() {
        return name;
    }
}
```

##### getUniqueServiceName方法
```java
/**
* 获取唯一服务名称（group和version可以为空）
* group/interfaceName:version
* @return
*/
@Parameter(excluded = true)
public String getUniqueServiceName() {
	StringBuilder buf = new StringBuilder();
	if (group != null && group.length() > 0) {
	    buf.append(group).append("/");
	}
	buf.append(interfaceName);
	if (version != null && version.length() > 0) {
	    buf.append(":").append(version);
	}
	return buf.toString();
}
```
##### ConsumerModel类
```java
public class ConsumerModel {
    /**
     * 元数据(即ReferenceBean实例)
     */
    private ReferenceConfig metadata;
    /**
     * 代理对象
     */
    private Object proxyObject;
    /**
     * 唯一的服务接口名称
     */
    private String serviceName;

    private final Map<Method, ConsumerMethodModel> methodModels = new IdentityHashMap<Method, ConsumerMethodModel>();

    public ConsumerModel(String serviceName,ReferenceConfig metadata, Object proxyObject, Method[] methods) {
        this.serviceName = serviceName;
        this.metadata = metadata;
        this.proxyObject = proxyObject;

        if (proxyObject != null) {
            //代理对象不为空,遍历服务接口方法列表，创建ConsumerMethodModel类
            for (Method method : methods) {
                //<服务接口方法，ConsumerMethodModel<服务接口方法,metadata实例>>
                methodModels.put(method, new ConsumerMethodModel(method, metadata));
            }
        }
    }

    /**
     * Return service metadata for consumer
     * @return service metadata
     */
    public ReferenceConfig getMetadata() {
        return metadata;
    }

    public Object getProxyObject() {
        return proxyObject;
    }

    /**
     * 获取消费者端的给定方法的MethodModel
     * Return method model for the given method on consumer side
     *
     * @param method method object
     * @return method model
     */
    public ConsumerMethodModel getMethodModel(Method method) {
        return methodModels.get(method);
    }

    /**
     * 获取当前服务的所有方法MethodModel
     * Return all method models for the current service
     *
     * @return method model list
     */
    public List<ConsumerMethodModel> getAllMethods() {
        return new ArrayList<ConsumerMethodModel>(methodModels.values());
    }

    public String getServiceName() {
        return serviceName;
    }
}
```

##### ApplicationModel类
接下来我们简单看下ApplicationModel.initConsumerModel(getUniqueServiceName(), consumerModel);
```java
/**
* 将服务的ConsumerModel方法保存到本地Map中
* @param serviceName
* @param consumerModel
* @return
*/
public static boolean initConsumerModel(String serviceName, ConsumerModel consumerModel) {
	if (consumedServices.putIfAbsent(serviceName, consumerModel) != null) {
	    logger.warn("Already register the same consumer:" + serviceName);
	    return false;
	}
	return true;
}
```

init方法我们还剩下两个方法没有讲解：
1、 ref = createProxy(map); 即创建代理对象
2、String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames(); 即包装类

计划这两个方法留着后面的章节(讲完注册中心等章节后)在讲解。

