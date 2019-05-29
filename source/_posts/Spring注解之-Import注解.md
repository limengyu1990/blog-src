---
title: Spring注解之@Import注解
date: 2018-07-30 20:53:39
tags: spring
---
>本小节介绍Spring提供的@Import注解

* @Import注解的作用
* @Import注解的定义
* @Import注解的使用

### @Import注解的作用
表示要导入1个或多个@Configuration配置类。提供了与Spring XML中的<import/>元素等效的作用.
@Import注解允许我们使用导入的方式将实例添加到Spring BeanFactory中.允许我们导入@Configuration类、ImportSelector接口和ImportBeanDefinitionRegistrar接口的实现类,类似于AnnotationConfigApplicationContext类的register方法。

### @Import注解的定义
可以看到该注解只可以使用在类上.
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {
	/**
	 * {@link Configuration}, {@link ImportSelector}, {@link ImportBeanDefinitionRegistrar}
	 * or regular component classes to import.
	 */
	Class<?>[] value();
}
```
### @Import注解的使用

接下来，我们分别介绍这三种使用方式,我们先创建一些测试类和测试方法:
```java
package org.dubbo.import;
public class A{}

public class B{}

public class C{}

public class D{}

//测试方法
public static void main(String args[]){
   //创建配置上下文
   AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
   //注册当前配置 Bean
   context.register(DubboConfiguration.class);
   context.refresh();
   
   //获取所有已注册的beanNames
   String[] beanNames = context.getBeanDefinitionNames();
   for(String beanName : beanNames){
       //输出beanNames
       System.out.println("beanName -> "+ beanName);
   }
}
```

#### Configuration方式
```java
@Import({A.class,B.class})
@Configuration
public class DubboConfig{} 
```
执行测试方法，将会输出：
```java
beanName -> DubboConfig
beanName -> org.dubbo.import.A
beanName -> org.dubbo.import.B
```

#### 实现ImportSelector接口方式
```java
/**
 * ImportSelector实现类
 */
public class DubboImportSelector implements ImportSelector{
    public String[] selectImports(AnnotationMetadata metadata) {
        return new String[]{"org.dubbo.import.C"};
    }
}

/**
 * 修改配置
 */
@Import({A.class,B.class,DubboImportSelector.class})
@Configuration
public class DubboConfig{}
```

执行测试方法，将会输出：
```java
beanName -> DubboConfig
beanName -> org.dubbo.import.A
beanName -> org.dubbo.import.B
beanName -> org.dubbo.import.C
```

#### 实现ImportBeanDefinitionRegistrar接口方式

```java
/**
 *
 * ImportBeanDefinitionRegistrar实现类
 */
public class DubboImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar{
    
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,BeanDefinitionRegistry registry) {
        //创建一个D类的rootBeanDefinition对象
        RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(D.class);
        //注册beanName=d的bean
        registry.registerBeanDefinition("d", rootBeanDefinition);
    }
}

/**
 * 修改配置
 */
@Import({A.class,B.class,DubboImportSelector.class,DubboImportBeanDefinitionRegistrar.class})
@Configuration
public class DubboConfig{}
```

然后执行测试方法，将会输出：
```java
beanName -> DubboConfig
beanName -> org.dubbo.import.A
beanName -> org.dubbo.import.B
beanName -> org.dubbo.import.C
beanName -> org.dubbo.import.D
```

可以看到，我们成功将A/B/C/D添加到了BeanFactory中.
