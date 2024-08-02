---
layout:       post
title:        "@Import 原理"
author:       "joey"
header-style: text
catalog:      true
tags:
    - Spring
---
**在项目开发的过程中，我们会遇到很多名字为 @Enablexxx 的注解，比如** `@EnableApollo-Config`、 `@EnableFeignClients`、 `@EnableAsync` 等。他们的功能都是通过这样的注解实现一个开关，决定了是否开启某个功能模块的所有组件的自动化配置，这极大的降低了我们的使用成本。

**@Import注解的三种用法主要包括：**

* **直接填class数组方式**
* **ImportSelector方式【重点】**
* **ImportBeanDefinitionRegistrar方式**

```java
@Configuration
@Import({Department.class, Employee.class})
public class PersonConfig2 {
}
```

**结果Department和Employee被导入,使用@Import导入bean时，id默认是组件的** `全类名`。

**按照默认的习惯，我们会把某个功能模块的开启注解定义为 @Enablexxx，功能的实现和名字格式其实无关，而是其内部实现，这里用 @EnableAsync 来举例子。**

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AsyncConfigurationSelector.class)
public @interface EnableAsync {

}
```

**可以看到除了3个通用注解，还有一个** `@Import(AsyncConfigurationSelector.class)`注解，显然它真正在这里发挥了关键作用，它可以往容器中注入一个配置类。

**在 Spring 容器启动的过程中，执行到调用** `invokeBeanFactoryPostProcessors(beanFactory)`方法的时候，会调用所有已经注册的 BeanFactoryPostProcessor，然后会调用实现 `BeanDefinitionRegistryPostProcessor` 接口的后置处理器 `ConfigurationClassPostProcessor` ，调用其 `postProcessBeanDefinitionRegistry()` 方法， 在这里会解析通过注解配置的类，然后调用 `ConfigurationClassParser#doProcessConfigurationClass()` 方法，最终会走到 `processImports()`方法，对 @Import 注解进行处理，具体流程如下：

![1722504651586](https://note.youdao.com/yws/api/personal/file/WEB3704b3660657e4eea615f0b9dddb2f6e?method=download&shareKey=7213275fc3486324b67ac75dbb117427)

**上述代码的核心逻辑无非就是如下几个步骤。**

1. **找到被 @Import 修饰的候选类集合，依次循环遍历。**
2. **如果该类实现了** `ImportSelector`接口，就调用 `ImportSelector` 的 `selectImports()` 方法，这个方法返回的是一批配置类的全限定名，然后递归调用 `processImports()`继续解析这些配置类，比如可以 @Import 的类里面有 @Import 注解，在这里可以递归处理。
3. **如果被修饰的类没有实现**`ImportSelector` 接口，而是实现了 `ImportBeanDefinitionRegistrar` 接口，则把对应的实例放入 `importBeanDefinitionRegistrars` 这个Map中，等到 `ConfigurationClassPostProcessor`处理 configClass 的时候，会与其他配置类一同被调用 `ImportBeanDefinitionRegistrar` 的 `registerBeanDefinitions()` 方法，以实现往 Spring 容器中注入一些 BeanDefinition。
4. **如果以上的两个接口都未实现，则进入 else 逻辑，将其作为普通的 @Configuration 配置类进行解析。**

**所以到这里，你应该明白 @Import 的作用机制了吧。对上述逻辑我总结了一张图，如下：**
![1722504651586](https://note.youdao.com/yws/api/personal/file/WEB5ee014d2ee06ce03646aea3bc7f4bb43?method=download&shareKey=a90cc6218aaab4afe8c99956e58e89d7)