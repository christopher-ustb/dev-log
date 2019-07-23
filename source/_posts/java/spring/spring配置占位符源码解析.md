---
title: spring配置占位符替换源码解析
date: 2019-7-23 14:06:39
categories:
- Java
tags:
- Java
- 读源码
---

今天搭建spring-boot项目启动时遇到一个问题：TDDL（淘宝内部使用的一款分库分表框架）DataSource无法找到MySQL配置，仔细观察日志发现是XML文件中spring bean的占位符，如`${za.castle.bastion.tddl.appname}`未被正确替换，运行时还是原来的带$符号的原字符串。

不得不承认，很多时候，我们的代码都是从其他同类项目复制过来的，遇到问题时，最常见的疑问大概就是：“在X项目中是好的啊？”。其实这种时候，问题不是不知道为什么我的代码不行，而是不知道，为什么X项目的代码可行。我的老师曾经说过：“当你不知道什么是正确，你如何能够纠正错误？”。

下面，我就从源码出发，揭示spring替换配置占位符的详细过程。

## 替换占位符

spring-boot启动入口`SpringApplication.run()`：

```java
public class SpringApplication {
    public ConfigurableApplicationContext run(String... args) {
		...
        context = createApplicationContext();
        analyzers = new FailureAnalyzers(context);
        prepareContext(context, environment, listeners, applicationArguments,
                printedBanner);
        refreshContext(context);
        afterRefresh(context, applicationArguments);
        ...
	}
}
```

`refreshContext`是spring bean体系最重要的方法之一，包含了大部分对bean的处理，其中就包含了：

```java
private static void invokeBeanFactoryPostProcessors(
        Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {
    for (BeanFactoryPostProcessor postProcessor : postProcessors) {
        postProcessor.postProcessBeanFactory(beanFactory);
    }
}
```

BeanFactoryPostProcessor是spring中与bean定义交互的接口，可以修改其定义。其中，就包含了`PropertySourcesPlaceholderConfigurer`这个实现(这个实现由PropertyPlaceholderAutoConfiguration配置到Spring ApplicationContext)，转换发现的所有placeholders。

```java
public class BeanDefinitionVisitor {
    public void visitBeanDefinition(BeanDefinition beanDefinition) {
		visitParentName(beanDefinition);
		visitBeanClassName(beanDefinition);
		visitFactoryBeanName(beanDefinition);
		visitFactoryMethodName(beanDefinition);
		visitScope(beanDefinition);
		visitPropertyValues(beanDefinition.getPropertyValues());
		ConstructorArgumentValues cas = beanDefinition.getConstructorArgumentValues();
		visitIndexedArgumentValues(cas.getIndexedArgumentValues());
		visitGenericArgumentValues(cas.getGenericArgumentValues());
	}
    ...
    protected String resolveStringValue(String strVal) {
		if (this.valueResolver == null) {
			throw new IllegalStateException("No StringValueResolver specified - pass a resolver " +
					"object into the constructor or override the 'resolveStringValue' method");
		}
		String resolvedValue = this.valueResolver.resolveStringValue(strVal);
		// Return original String if not modified.
		return (strVal.equals(resolvedValue) ? strVal : resolvedValue);
	}
}
```

`resolveStringValue`方法会将placeholder替换为对应的值。

## Environment

上面解释了什么时候替换占位符，那么占位符的值从哪里来的呢？

spring中所有的property都来源于一个统一的抽象`Environment`，在`PropertySourcesPlaceholderConfigurer`初始化之时，会将applicationContext中的`Environment`注入。

```java
class ApplicationContextAwareProcessor implements BeanPostProcessor {
    private void invokeAwareInterfaces(Object bean) {
        if (bean instanceof Aware) {
            if (bean instanceof EnvironmentAware) {
                ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
            }
        ...
        }
    }
}
```

## spring-cloud-config

所以，属性来自`Environment`，那么*spring-cloud-config-client*又是如何将*spring-cloud-config-server*上的属性设置到`Environment`中的呢？

spring-cloud-config的启动配置类`PropertySourceBootstrapConfiguration`在初始化时，会试图获取property，并将其设置进`applicationContext.getEnvironment()`中：

```java
public class PropertySourceBootstrapConfiguration implements
		ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {
    public void initialize(ConfigurableApplicationContext applicationContext) {
		...
		ConfigurableEnvironment environment = applicationContext.getEnvironment();
		for (PropertySourceLocator locator : this.propertySourceLocators) {
			PropertySource<?> source = null;
			source = locator.locate(environment);
			if (source == null) {
				continue;
			}
			logger.info("Located property source: " + source);
			composite.addPropertySource(source);
			empty = false;
		}
		...
	}
}
```

*spring-cloud-config-client*的`locate()`方法实现就是从server获取配置并注入。

所以*spring-cloud-config-client*就这样获取到配置，并保存在`Environment`中，并在bean定义处理时替代了对应的占位符。
