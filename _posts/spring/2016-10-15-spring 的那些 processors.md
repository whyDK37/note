---
layout: default
title: spring 的那些 processors
excerpt_separator: <!--more-->
---

<!--more-->
# spring笔记
## spring 的那些 processors
**BeanFactoryPostProcessor**
Allows for custom modification of an application context's bean definitions,
adapting the bean property values of the context's underlying bean factory.

Application contexts can auto-detect BeanFactoryPostProcessor beans in
their bean definitions and apply them before any other beans get created.

Useful for custom config files targeted at system administrators that
override bean properties configured in the application context.

eg.
`PropertyPlaceholderConfigurer`
`PropertySourcesPlaceholderConfigurer`
This class is designed as a general replacement for {[@code](https://my.oschina.net/codeo)
PropertyPlaceholderConfigurer} in Spring 3.1 applications. It is used by default to
support the {[@code](https://my.oschina.net/codeo) property-placeholder} element in working against the
spring-context-3.1 XSD, whereas spring-context versions &lt;= 3.0 default to
{[@code](https://my.oschina.net/codeo) PropertyPlaceholderConfigurer} to ensure backward compatibility. See
spring-context XSD documentation for complete details.

As of Spring 3.1, PropertySourcesPlaceholderConfigurer should be used preferentially over this implementation;

**BeanPostProcessor**
Factory hook that allows for custom modification of new **bean instances**,
e.g. checking for marker interfaces or wrapping them with proxies.

ApplicationContexts can autodetect BeanPostProcessor beans in their
bean definitions and apply them to any beans subsequently created.
Plain bean factories allow for programmatic registration of post-processors,
applying to all beans created through this factory.

<p>Typically, post-processors that populate beans via marker interfaces
or the like will implement {[@link](https://my.oschina.net/u/393) #postProcessBeforeInitialization},
while post-processors that wrap beans with proxies will normally
implement {[@link](https://my.oschina.net/u/393) #postProcessAfterInitialization}.

**DestructionAwareBeanPostProcessor**
Subinterface of {@link BeanPostProcessor} that adds a before-destruction callback.

he typical usage will be to invoke custom destruction callbacks on
specific bean types, matching corresponding initialization callbacks.

**InstantiationAwareBeanPostProcessor**
Subinterface of {@link BeanPostProcessor} that adds a before-instantiation callback,
and a callback after instantiation but before explicit properties are set or
autowiring occurs.

Typically used to suppress default instantiation for specific target beans,
for example to create proxies with special TargetSources (pooling targets,
lazily initializing targets, etc), or to implement additional injection strategies
such as field injection.

**BeanDefinitionRegistryPostProcessor**
```
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor
```
Extension to the standard {@link BeanFactoryPostProcessor} SPI, allowing for
the registration of further bean definitions <i>before</i> regular
BeanFactoryPostProcessor detection kicks in. In particular,
BeanDefinitionRegistryPostProcessor may register further bean definitions
which in turn define BeanFactoryPostProcessor instances.