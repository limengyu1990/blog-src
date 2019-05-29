---
title: Dubbo源码阅读之集成Spring-标签解析(02)
date: 2018-08-04 17:34:52
tags: 
     - dubbo
---
>本小节将会介绍dubbo标签的解析.

上节讲到Dubbo解析标签的类是DubboBeanDefinitionParser,现在我们就来分析下该类,该类实现了Spring的BeanDefinitionParser接口，我们需要实现它的parse方法:

### DubboBeanDefinitionParser解析类
```java
public class DubboBeanDefinitionParser implements BeanDefinitionParser{
 
    private static final Pattern GROUP_AND_VERION = Pattern.compile("^[\\-.0-9_a-zA-Z]+(\\:[\\-.0-9_a-zA-Z]+)?$");
    
    //带解析实例化的bean类
    private final Class<?> beanClass;
    
    //是否必填
    private final boolean required;

    public DubboBeanDefinitionParser(Class<?> beanClass, boolean required) {
        this.beanClass = beanClass;
        this.required = required;
    }

    @Override
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        //调用了内部的parse方法
        return parse(element, parserContext, beanClass, required);
    }
   
    private static BeanDefinition parse(Element element, ParserContext parserContext,
                                        Class<?> beanClass, boolean required) {
        //创建RootBeanDefinition实例
        RootBeanDefinition beanDefinition = new RootBeanDefinition();
        //设置要实例化的类
        beanDefinition.setBeanClass(beanClass);
        //是否延迟初始化
        beanDefinition.setLazyInit(false);
        
        //获取元素的id属性作为bean的id
        String id = element.getAttribute("id");
        if ((id == null || id.length() == 0) && required) {
            //如果id属性为空，并且是必填的，则取元素的name属性
            String generatedBeanName = element.getAttribute("name");
            if (generatedBeanName == null || generatedBeanName.length() == 0) {
                //如果name属性为空，则生成一个bean名称
                //判断当前要实例化的bean类是否是ProtocolConfig类（即当前是否为<dubbo:protocol>标签）
                if (ProtocolConfig.class.equals(beanClass)) {
                    //当前是ProtocolConfig类，则设置bean名称为dubbo
                    generatedBeanName = "dubbo";
                } else {
                    //不是ProtocolConfig类,则使用interface属性做为bean的名称
                    generatedBeanName = element.getAttribute("interface");
                }
            }
            //如果bean名称为空，则使用当前实例化的类的名称
            if (generatedBeanName == null || generatedBeanName.length() == 0) {
                generatedBeanName = beanClass.getName();
            }
            //同时将bean的id属性设置为bean的名称
            id = generatedBeanName;
            int counter = 2;
            //如果上下文中已经存在该id，则对该bean的id做防重处理
            while (parserContext.getRegistry().containsBeanDefinition(id)) {
                id = generatedBeanName + (counter++);
            }
        }
        if (id != null && id.length() > 0) {
            //如果id属性不为空，则校验Spring中是否已存在该id
            if (parserContext.getRegistry().containsBeanDefinition(id)) {
                throw new IllegalStateException("Duplicate spring bean id " + id);
            }
            //通过id注册该bean
            parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
            //设置bean的id属性值
            beanDefinition.getPropertyValues().addPropertyValue("id", id);
        }
        //进一步解析<dubbo:protocol/>标签
        //如：<dubbo:protocol name="dubbo" port="20880"/>
        if (ProtocolConfig.class.equals(beanClass)) {
            //如果当前要实例化的bean是ProtocolConfig类，则遍历Spring所有已经注册的bean的名称
            for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
                //根据bean名称获取已注册的bean定义
                BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
                //查看已注册的bean定义中是否存在protocol属性
                PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
                if (property != null) {
                    //存在protocol属性，则获取protocol属性值
                    Object value = property.getValue();
                    //如果protocol属性值为ProtocolConfig类型，并且当前实例化的bean的id等于protocol属性值的name属性值的话，
                    //则将当前实例化的bean包装成RuntimeBeanReference类型
                    //然后将已注册的bean的protocol属性的值设为刚才新创建的RuntimeBeanReference
                    //也就是说将当前解析的bean（ProtocolConfig）设置到其他已注册的bean的protocol属性中
                    if (value instanceof ProtocolConfig && id.equals(((ProtocolConfig) value).getName())) {
                        definition.getPropertyValues().addPropertyValue("protocol", new RuntimeBeanReference(id));
                    }
                }
            }
        } else if (ServiceBean.class.equals(beanClass)) {
            //进一步解析<dubbo:service/>标签，这里的逻辑就是处理下面的第二种配置
            //如：<dubbo:service protocol="dubbo" ref="userService" interface="org.dubbo.service.UserInterface" retries="0" />
            //   <bean id="userService" class="org.dubbo.service.impl.UserServiceImpl" />
            //和如下配置相同
            //<dubbo:service interface="org.dubbo.service.UserInterface" class="org.dubbo.service.impl.UserServiceImpl" protocol="dubbo" retries="0">
            //   <property name="name" value="dubbo" ref=""/>
            //   <property name="age" value="5" ref=""/>
            //</dubbo:service>

            //如果当前要实例化的bean是ServiceBean类,则获取class属性并解析
            String className = element.getAttribute("class");
            if (className != null && className.length() > 0) {
                //如果class属性不为空，则实例化该class属性指定的类
                RootBeanDefinition classDefinition = new RootBeanDefinition();
                classDefinition.setBeanClass(ReflectUtils.forName(className));
                classDefinition.setLazyInit(false);
                //解析子标签元素(即上面例子中的property属性)(后面会介绍parseProperties方法)
                parseProperties(element.getChildNodes(), classDefinition);
                //为当前待实例化的bean定义增加ref属性值（即class属性对应的bean，id为 beanId + impl）
                beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
            }
        } else if (ProviderConfig.class.equals(beanClass)) {
            //进一步处理<dubbo:provider/>标签(后面会分析parseNested方法)
            parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
        } else if (ConsumerConfig.class.equals(beanClass)) {
            //进一步处理<dubbo:consumer/>标签的子标签<dubbo:reference/>
            parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);
        }
        
	//当前待实例化的bean的所有的属性(通过set方法获取到)
        Set<String> props = new HashSet<String>();
        ManagedMap parameters = null;
        //遍历待实例化类的所有方法,找到set方法
        for (Method setter : beanClass.getMethods()) {
            
            String name = setter.getName();
            
            //找到set方法，条件如下：方法名长度大于3，以set开头，是public修饰符，参数类型长度为1
            if (name.length() > 3 && name.startsWith("set")
                    && Modifier.isPublic(setter.getModifiers())
                    && setter.getParameterTypes().length == 1) {
               
                 //获取参数类型
                Class<?> type = setter.getParameterTypes()[0];

                //如setFirstName(String firstName)方法
                //将会得到property=first-name
                String property = StringUtils.camelToSplitName(name.substring(3, 4).toLowerCase() + name.substring(4), "-");
          
	        //将属性名称添加到props集合中
                props.add(property);
               
                //获取到属性的get方法
                Method getter = null;
               
                try {
                    getter = beanClass.getMethod("get" + name.substring(3), new Class<?>[0]);
                } catch (NoSuchMethodException e) {
                    try {
                        //如果没有找到get方法，则尝试找到isXXX方法
                        getter = beanClass.getMethod("is" + name.substring(3), new Class<?>[0]);
                    } catch (NoSuchMethodException e2) {
                    }
                }
                if (getter == null
                        || !Modifier.isPublic(getter.getModifiers())
                        || !type.equals(getter.getReturnType())) {
                    //如果get方法为空、不是public限定符、或者get方法返回值和set方法参数类型不一致的话，
                    //则跳过该property的处理，进行下一个property的处理
                    continue;
                }
                if ("parameters".equals(property)) {
                    //如果当前property为parameters，则解析当前元素的子元素(<dubbo:parameter>标签)(后面会分析parseParameters方法)
                    parameters = parseParameters(element.getChildNodes(), beanDefinition);
                } else if ("methods".equals(property)) {
                    //如果当前property为methods，则解析当前元素的子元素(<dubbo:method>标签)(后面会分析parseMethods方法)
                    parseMethods(id, element.getChildNodes(), beanDefinition, parserContext);
                } else if ("arguments".equals(property)) {
                    //如果当前property为arguments，则解析当前元素的子元素(<dubbo:argument>标签)(后面会分析parseArguments方法)
                    parseArguments(id, element.getChildNodes(), beanDefinition, parserContext);
                } else {
                    //从xml定义中获取property属性值
                    String value = element.getAttribute(property);
                    if (value != null) {
                        value = value.trim();
                        if (value.length() > 0) {
                            if ("registry".equals(property) && RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(value)) {
                                //如果当前属性为registry，并且属性值为N/A
                                //则为当前bean定义添加属性registry，并且属性值为RegistryConfig对象(该对象的address属性值为不可用)
                                RegistryConfig registryConfig = new RegistryConfig();
                                registryConfig.setAddress(RegistryConfig.NO_AVAILABLE);
                                beanDefinition.getPropertyValues().addPropertyValue(property, registryConfig);
                            } else if ("registry".equals(property) && value.indexOf(',') != -1) {
                                //如果当前属性为registry，并且属性值包含逗号",",即有多个值，则进一步解析(后面会分析parseMultiRef方法)
                                parseMultiRef("registries", value, beanDefinition, parserContext);
                            } else if ("provider".equals(property) && value.indexOf(',') != -1) {
                                //如果当前属性为providers，并且属性值包含逗号",",即有多个值，则进一步解析
                                parseMultiRef("providers", value, beanDefinition, parserContext);
                            } else if ("protocol".equals(property) && value.indexOf(',') != -1) {
                                //如果当前属性为protocol，并且属性值包含逗号",",即有多个值，则进一步解析
                                parseMultiRef("protocols", value, beanDefinition, parserContext);
                            } else {
                                Object reference;
                                //判断set方法的参数类型是否是原始类型
                                if (isPrimitive(type)) {
                                     if ("async".equals(property) && "false".equals(value)
                                            || "timeout".equals(property) && "0".equals(value)
                                            || "delay".equals(property) && "0".equals(value)
                                            || "version".equals(property) && "0.0.0".equals(value)
                                            || "stat".equals(property) && "-1".equals(value)
                                            || "reliable".equals(property) && "false".equals(value)) {
                                        // backward compatibility for the default value in old version's xsd
                                        //兼容老版本
					value = null;
                                    }
				    //原始类型，直接只用xml中配置的属性值
                                    reference = value;
                                } else if ("protocol".equals(property)
                                        //属性名为protocol，并且当前存在该属性值对应的扩展
                                        && ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(value)
                                        //当前上下文不包含扩展的bean定义
                                        //该扩展bean的类名称不等于ProtocolConfig类名称
                                        && (!parserContext.getRegistry().containsBeanDefinition(value)
                                        || !ProtocolConfig.class.getName().equals(parserContext.getRegistry().getBeanDefinition(value).getBeanClassName()))) {
                                    //如果元素标签名称为dubbo:provider，则提示使用dubbo:protocol进行替换
                                    if ("dubbo:provider".equals(element.getTagName())) {
                                        logger.warn("Recommended replace <dubbo:provider protocol=\"" + value + "\" ... /> to <dubbo:protocol name=\"" + value + "\" ... />");
                                    }
                                    // backward compatibility 向后兼容
                                    // 设置协议名称为value
                                    ProtocolConfig protocol = new ProtocolConfig();
                                    protocol.setName(value);
                                    //设置reference值为protocol
                                    reference = protocol;
                                } else if ("onreturn".equals(property)) {
                                    //获取"."在属性值中最后出现的位置
                                    int index = value.lastIndexOf(".");
                                    
 				     //截取value从首字符到index字符
                                    String returnRef = value.substring(0, index);
                                    
				    //从index+1字符开始进行截取
                                    String returnMethod = value.substring(index + 1);
					
				    //包装成RuntimeBeanReference
                                    reference = new RuntimeBeanReference(returnRef);
                                    
 				    //为当前bean定义增加属性onreturnMethod
                                    beanDefinition.getPropertyValues().addPropertyValue("onreturnMethod", returnMethod);
                                } else if ("onthrow".equals(property)) {
                                   //处理onthrow属性 
				   int index = value.lastIndexOf(".");
                                   String throwRef = value.substring(0, index);
                                   String throwMethod = value.substring(index + 1);
                                    
				   reference = new RuntimeBeanReference(throwRef);
                                   
				    //为当前bean定义增加属性onthrowMethod
                                    beanDefinition.getPropertyValues().addPropertyValue("onthrowMethod", throwMethod);
                                } else if ("oninvoke".equals(property)) {
                                    int index = value.lastIndexOf(".");
                                    String invokeRef = value.substring(0, index);
                                    String invokeRefMethod = value.substring(index + 1);
                                    
				    reference = new RuntimeBeanReference(invokeRef);
                                    //为当前bean定义增加属性oninvokeMethod
                                    beanDefinition.getPropertyValues().addPropertyValue("oninvokeMethod", invokeRefMethod);
                                }else {
                                    //如果当前属性为ref，并且当前上下文中包含属性值value对应的bean定义
                                    if ("ref".equals(property) && parserContext.getRegistry().containsBeanDefinition(value)) {
                                        //获取属性值value对应的bean定义
                                        BeanDefinition refBean = parserContext.getRegistry().getBeanDefinition(value);
                                        //检测是否是单例
                                        if (!refBean.isSingleton()) {
                                            //暴露的服务ref值必须是单例
                                            throw new IllegalStateException("The exported service ref " + value + " must be singleton! Please set the " + value + " bean scope to singleton, eg: <bean id=\"" + value + "\" scope=\"singleton\" ...>");
                                        }
                                    }
                                    reference = new RuntimeBeanReference(value);
                                }
                                //为当前bean定义添加属性property，属性值为reference
                                beanDefinition.getPropertyValues().addPropertyValue(property, reference);
                            }
                        }
                    }
                }
            }
        }
        //获取元素的所有属性
        NamedNodeMap attributes = element.getAttributes();
	//属性数量
        int len = attributes.getLength();
        //遍历所有属性，看看是否有不再set属性集合中的元素，如果有，则将他们添加到自定义参数map中，然后设置当前待实例化bean的parameters参数
        for (int i = 0; i < len; i++) {
            Node node = attributes.item(i);
            String name = node.getLocalName();
            //如果当前set属性集合中不包含该属性，则将该属性以及属性值添加到自定义parameters中
            if (!props.contains(name)) {
                if (parameters == null) {
                    parameters = new ManagedMap();
                }
                //获取该节点属性值
                String value = node.getNodeValue();
                //将值包装成TypedStringValue类型，并放入自定义参数map中
                parameters.put(name, new TypedStringValue(value, String.class));
            }
        }
        if (parameters != null) {
            //为当前bean定义添加属性parameters
            beanDefinition.getPropertyValues().addPropertyValue("parameters", parameters);
        }
	//返回bean定义
        return beanDefinition;
    }

}
```
#### parseProperties方法

```java
/**
* 解析子标签元素
* @param nodeList
* @param beanDefinition 待实例化的bean
*/
private static void parseProperties(NodeList nodeList, RootBeanDefinition beanDefinition) {
	
	//<dubbo:service interface="org.dubbo.service.UserInterface" class="org.dubbo.service.impl.UserService" protocol="dubbo" retries="0">
   	//	<property name="name" value="dubbo" ref=""/>
   	//	<property name="age" value="5" ref=""/>
	//</dubbo:service>
	if (nodeList != null && nodeList.getLength() > 0) {
	   //遍历所有property节点 
	   for (int i = 0; i < nodeList.getLength(); i++) {
		//当前节点
		Node node = nodeList.item(i);
		if (node instanceof Element) {
		    //如果节点名称为property或者节点限定名称为property
		    if ("property".equals(node.getNodeName())
			    || "property".equals(node.getLocalName())) {
			
			//获取property节点的name属性
			String name = ((Element) node).getAttribute("name");
			if (name != null && name.length() > 0) {
			    //获取property节点的value属性
			    String value = ((Element) node).getAttribute("value");
			   
			    //获取property节点的ref属性
			    String ref = ((Element) node).getAttribute("ref");
			    
			    if (value != null && value.length() > 0) {
				//为bean定义设置属性以及属性值
				beanDefinition.getPropertyValues().addPropertyValue(name, value);
			    } else if (ref != null && ref.length() > 0) {
				//为bean定义设置属性以及属性值（引用）
				beanDefinition.getPropertyValues().addPropertyValue(name, new RuntimeBeanReference(ref));
			    } else {
				//不支持的子标签
				throw new UnsupportedOperationException("Unsupported <property name=\"" + name + "\"> sub tag, Only supported <property name=\"" + name + "\" ref=\"...\" /> or <property name=\"" + name + "\" value=\"...\" />");
			    }
			}
		    }
		}
	    }
	}
}
```

#### parseNested方法

```java
parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);



/**
* 解析嵌套标签
* 即：
*   <dubbo:provider>
*        <dubbo:service interface="">
*            <dubbo:method name=""></dubbo:method>
*        </dubbo:service>
*        <dubbo:service interface="">
*            <dubbo:method name=""></dubbo:method>
*        </dubbo:service>
*   </dubbo:provider>
* @param element  当前元素：<dubbo:provider>
* @param parserContext 上下文
* @param beanClass  ServiceBean.class/ReferenceBean.class
* @param required   true/false
* @param tag   标签      service/reference
* @param property 属性   provider/consumer
* @param ref   待实例化bean的Id
* @param beanDefinition 待实例化bean定义
*/
private static void parseNested(Element element, ParserContext parserContext, Class<?> beanClass,
			    boolean required, String tag, String property, 
			    String ref,BeanDefinition beanDefinition) {
        //获取当前元素的子节点
	NodeList nodeList = element.getChildNodes();
	if (nodeList != null && nodeList.getLength() > 0) {
	    boolean first = true;
	    //遍历子节点
	    for (int i = 0; i < nodeList.getLength(); i++) {
		//当前子节点
		Node node = nodeList.item(i);
		if (node instanceof Element) {
		    //如果当前子节点的名称等于tag
		    if (tag.equals(node.getNodeName())
			    || tag.equals(node.getLocalName())) {
			//是否第一个子节点
			if (first) {
			    first = false;
			    //获取当前元素default属性
			    String isDefault = element.getAttribute("default");
			    if (isDefault == null || isDefault.length() == 0) {
				//如果default属性为空，则为bean定义增加default属性，并将属性值设置成false
				beanDefinition.getPropertyValues().addPropertyValue("default", "false");
			    }
			}
			
		        //调用parse方法递归解析beanClass（即ServiceBean.class/ReferenceBean.class）
			BeanDefinition subDefinition = parse((Element) node, parserContext, beanClass, required);
			
			//如果子bean不为空，且ref不为空，则为子bean增加property属性(provider/consumer)(即将父bean设置进去,ref参数)
			if (subDefinition != null && ref != null && ref.length() > 0) {
			    subDefinition.getPropertyValues().addPropertyValue(property, new RuntimeBeanReference(ref));
			}
		    }
		}
	    }
	}
}
```
#### parseParameters方法
```java
/**
* 当前property为parameters，解析参数标签 <dubbo:parameter>
* @param nodeList 所有子节点列表
* @param beanDefinition 当前待实例化的bean
* @return 自定义参数集合
*/
private static ManagedMap parseParameters(NodeList nodeList, RootBeanDefinition beanDefinition) {
	if (nodeList != null && nodeList.getLength() > 0) {
	    
	    ManagedMap parameters = null;
            
	    //遍历子节点
	    for (int i = 0; i < nodeList.getLength(); i++) {
		
		Node node = nodeList.item(i);
		
		if (node instanceof Element) {
		    
		    //如果子节点名称是parameter
		    if ("parameter".equals(node.getNodeName())
			    || "parameter".equals(node.getLocalName())) {
			
			if (parameters == null) {
			    parameters = new ManagedMap();
			}
			
			//获取子节点的key属性、value属性、hide属性
			String key = ((Element) node).getAttribute("key");
			String value = ((Element) node).getAttribute("value");
			boolean hide = "true".equals(((Element) node).getAttribute("hide"));
			
			if (hide) {
			    //如果需要隐藏的话，则修改key属性的值
			    key = Constants.HIDE_KEY_PREFIX + key;
			}
			
			//保存参数信息
			parameters.put(key, new TypedStringValue(value, String.class));
		    }
		}
	    }
	    return parameters;
	}
	return null;
}
```

#### parseMethods方法
```java

/**
 *
 * 当前property为methods，解析<dubbo:method>标签
 * @param id 当前待实例化的bean-id
 * @param nodeList 所有子节点列表
 * @param beanDefinition 待实例化bean
 * @param parserContext
 */
private static void parseMethods(String id, NodeList nodeList, 
				RootBeanDefinition beanDefinition,
				ParserContext parserContext) {

	if (nodeList != null && nodeList.getLength() > 0) {
	    ManagedList methods = null;
	    
	    //遍历子节点列表
	    for (int i = 0; i < nodeList.getLength(); i++) {
		
		Node node = nodeList.item(i);
		
		if (node instanceof Element) {
        if (ProtocolConfig.class.equals(beanClass)) {
            //如果当前要实例化的bean是ProtocolConfig类，则遍历Spring所有已经注册的bean的名称
            for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
                //根据bean名称获取已注册的bean定义
                BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
                //查看已注册的bean定义中是否存在protocol属性
                PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
                if (property != null) {
                    //存在protocol属性，则获取protocol属性值
                    Object value = property.getValue();
                    //如果protocol属性值为ProtocolConfig类型，并且当前实例化的bean的id等于protocol属性值的name属性值的话，
                    //则将当前实例化的bean包装成RuntimeBeanReference类型
                    //然后将已注册的bean的protocol属性的值设为刚才新创建的RuntimeBeanReference
                    //也就是说将当前解析的bean（ProtocolConfig）设置到其他已注册的bean的protocol属性中
                    if (value instanceof ProtocolConfig && id.equals(((ProtocolConfig) value).getName())) {
                        definition.getPropertyValues().addPropertyValue("protocol", new RuntimeBeanReference(id));
                    }
                }
            }
        } else if (ServiceBean.class.equals(beanClass)) {
            //进一步解析<dubbo:service/>标签，这里的逻辑就是处理下面的第二种配置
            //如：<dubbo:service protocol="dubbo" ref="userService" interface="org.dubbo.service.UserInterface" retries="0" />
            //   <bean id="userService" class="org.dubbo.service.impl.UserServiceImpl" />
            //和如下配置相同
            //<dubbo:service interface="org.dubbo.service.UserInterface" class="org.dubbo.service.impl.UserServiceImpl" protocol="dubbo" retries="0">
            //   <property name="name" value="dubbo" ref=""/>
            //   <property name="age" value="5" ref=""/>
            //</dubbo:service>

            //如果当前要实例化的bean是ServiceBean类,则获取class属性并解析
            String className = element.getAttribute("class");
            if (className != null && className.length() > 0) {
                //如果class属性不为空，则实例化该class属性指定的类
                RootBeanDefinition classDefinition = new RootBeanDefinition();
                classDefinition.setBeanClass(ReflectUtils.forName(className));
                classDefinition.setLazyInit(false);
                //解析子标签元素(即上面例子中的property属性)(后面会介绍parseProperties方法)
                parseProperties(element.getChildNodes(), classDefinition);
                //为当前待实例化的bean定义增加ref属性值（即class属性对应的bean，id为 beanId + impl）
                beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
            }
        } else if (ProviderConfig.class.equals(beanClass)) {
            //进一步处理<dubbo:provider/>标签(后面会分析parseNested方法)
            parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
        } else if (ConsumerConfig.class.equals(beanClass)) {
            //进一步处理<dubbo:consumer/>标签的子标签<dubbo:reference/>
            parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);
        }
        
	//当前待实例化的bean的所有的属性(通过set方法获取到)
        Set<String> props = new HashSet<String>();
        ManagedMap parameters = null;
        //遍历待实例化类的所有方法,找到set方法
        for (Method setter : beanClass.getMethods()) {
            
            String name = setter.getName();
            
            //找到set方法，条件如下：方法名长度大于3，以set开头，是public修饰符，参数类型长度为1
            if (name.length() > 3 && name.startsWith("set")
                    && Modifier.isPublic(setter.getModifiers())
                    && setter.getParameterTypes().length == 1) {
               
                 //获取参数类型
                Class<?> type = setter.getParameterTypes()[0];

                //如setFirstName(String firstName)方法
                //将会得到property=first-name
                String property = StringUtils.camelToSplitName(name.substring(3, 4).toLowerCase() + name.substring(4), "-");
          
	        //将属性名称添加到props集合中
                props.add(property);
               
                //获取到属性的get方法
                Method getter = null;
               
                try {
                    getter = beanClass.getMethod("get" + name.substring(3), new Class<?>[0]);
                } catch (NoSuchMethodException e) {
                    try {
                        //如果没有找到get方法，则尝试找到isXXX方法
                        getter = beanClass.getMethod("is" + name.substring(3), new Class<?>[0]);
                    } catch (NoSuchMethodException e2) {
                    }
                }
                if (getter == null
                        || !Modifier.isPublic(getter.getModifiers())
                        || !type.equals(getter.getReturnType())) {
                    //如果get方法为空、不是public限定符、或者get方法返回值和set方法参数类型不一致的话，
                    //则跳过该property的处理，进行下一个property的处理
                    continue;
                }
                if ("parameters".equals(property)) {
                    //如果当前property为parameters，则解析当前元素的子元素(<dubbo:parameter>标签)(后面会分析parseParameters方法)
                    parameters = parseParameters(element.getChildNodes(), beanDefinition);
                } else if ("methods".equals(property)) {
                    //如果当前property为methods，则解析当前元素的子元素(<dubbo:method>标签)(后面会分析parseMethods方法)
                    parseMethods(id, element.getChildNodes(), beanDefinition, parserContext);
                } else if ("arguments".equals(property)) {
                    //如果当前property为arguments，则解析当前元素的子元素(<dubbo:argument>标签)(后面会分析parseArguments方法)
                    parseArguments(id, element.getChildNodes(), beanDefinition, parserContext);
                } else {
                    //从xml定义中获取property属性值
                    String value = element.getAttribute(property);
                    if (value != null) {
                        value = value.trim();
                        if (value.length() > 0) {
                            if ("registry".equals(property) && RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(value)) {
                                //如果当前属性为registry，并且属性值为N/A
                                //则为当前bean定义添加属性registry，并且属性值为RegistryConfig对象(该对象的address属性值为不可用)
                                RegistryConfig registryConfig = new RegistryConfig();
                                registryConfig.setAddress(RegistryConfig.NO_AVAILABLE);
                                beanDefinition.getPropertyValues().addPropertyValue(property, registryConfig);
                            } else if ("registry".equals(property) && value.indexOf(',') != -1) {
                                //如果当前属性为registry，并且属性值包含逗号",",即有多个值，则进一步解析(后面会分析parseMultiRef方法)
                                parseMultiRef("registries", value, beanDefinition, parserContext);
                            } else if ("provider".equals(property) && value.indexOf(',') != -1) {
                                //如果当前属性为providers，并且属性值包含逗号",",即有多个值，则进一步解析
                                parseMultiRef("providers", value, beanDefinition, parserContext);
                            } else if ("protocol".equals(property) && value.indexOf(',') != -1) {
                                //如果当前属性为protocol，并且属性值包含逗号",",即有多个值，则进一步解析
                                parseMultiRef("protocols", value, beanDefinition, parserContext);
                            } else {
                                Object reference;
                                //判断set方法的参数类型是否是原始类型
                                if (isPrimitive(type)) {
                                     if ("async".equals(property) && "false".equals(value)
                                            || "timeout".equals(property) && "0".equals(value)
			    
		    //当前节点的名称为"method"
		    if ("method".equals(node.getNodeName()) || "method".equals(node.getLocalName())) {
			
			//获取当前节点的name属性
			String methodName = element.getAttribute("name");
			
			if (methodName == null || methodName.length() == 0) {
			    //name属性不可以为空
			    throw new IllegalStateException("<dubbo:method> name attribute == null");
			}
			
			if (methods == null) {
			    methods = new ManagedList();
			}
			
			//调用parse方法递归解析MethodConfig.class
			BeanDefinition methodBeanDefinition = parse(((Element) node),
				parserContext, 
				MethodConfig.class, 
				false
			);
			
			//新生成的bean的名称
			String name = id + "." + methodName;
			
			//将bean名称和bean定义关联起来
			BeanDefinitionHolder methodBeanDefinitionHolder = new BeanDefinitionHolder(
				methodBeanDefinition, name);
			//保存新生成的bean
			methods.add(methodBeanDefinitionHolder);
		    }
		}
	    }
	    if (methods != null) {
		//为待实例化的bean添加methods属性
		beanDefinition.getPropertyValues().addPropertyValue("methods", methods);
	    }
	}
}

```

#### parseArguments方法
```java
/**
* 当前property为arguments，解析<dubbo:argument>标签
* @param id  当前待实例化的bean-id
* @param nodeList 子节点列表
* @param beanDefinition 当前待实例化的bean
* @param parserContext
*/
private static void parseArguments(String id, NodeList nodeList, RootBeanDefinition beanDefinition,
			       ParserContext parserContext) {

	if (nodeList != null && nodeList.getLength() > 0) {
	    
	    ManagedList arguments = null;

	    //遍历子节点
	    for (int i = 0; i < nodeList.getLength(); i++) {
		
		Node node = nodeList.item(i);

		if (node instanceof Element) {
		    
		    Element element = (Element) node;

		    //当前子节点名称为argument
		    if ("argument".equals(node.getNodeName()) || "argument".equals(node.getLocalName())) {
			
			//获取当前子节点的index属性
			String argumentIndex = element.getAttribute("index");
			
			if (arguments == null) {
			    arguments = new ManagedList();
			}

			//调用parse方法递归解析ArgumentConfig.class
			BeanDefinition argumentBeanDefinition = parse(((Element) node),
				parserContext, 
				ArgumentConfig.class, 
				false
			);

			//新生成的bean的名称
			String name = id + "." + argumentIndex;

			//将新生成的bean名称和bean定义关联起来
			BeanDefinitionHolder argumentBeanDefinitionHolder = new BeanDefinitionHolder(
				argumentBeanDefinition, name);
        if (!beanDefinitionRegistry.containsBeanDefinition(beanName)) {
            //该bean还没有注册的话，则进行注册
	    RootBeanDefinition beanDefinition = new RootBeanDefinition(beanType);
            
	    beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
            
	    beanDefinitionRegistry.registerBeanDefinition(beanName, beanDefinition);
        }
    }
}
```

DubboBeanDefinitionParser解析类就介绍完毕了，最后给一个的parse方法的处理流程图.
![](img/DubboBeanDefinitionParser.png)





