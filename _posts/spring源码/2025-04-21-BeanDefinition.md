---
title: "test"
date: 2025-04-21
categories: [Spring源码]
tags: [Spring,BeanDefinition]
---

# BeanDefinition

- 在 Spring 框架中，BeanDefinition是用来描述一个 `Bean` 的配置信息的核心接口。它是 Spring IoC 容器内部用来管理 Bean 的元数据模型。
- `BeanDefinition`描述了一个 bean 实例，该实例具有属性值、构造函数参数值和具体实现提供的更多信息。 它不是具体的 Bean，而是描述怎么创建这个 Bean 的“说明书”或“蓝图”
- `BeanDefinition`是 Spring 容器中用来描述 Bean 的配置信息的对象，它包含了Bean的各种属性，如Bean的名称、类名、作用域、初始化方法、销毁方法等。

## 接口定义

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement{
  
}
```

### `AttributeAccessor`: 允许为这个 Bean 定义设置自定义属性(key-value)

```java
public interface AttributeAccessor{
       void setAttribute(String name, @Nullable Object value);
       @Nullable
       Object getAttribute(String name);
       @Nullable
       Object removeAttribute(String name);
       boolean hasAttribute(String name);
       String[] attributeNames();
}
```

- `AttributeAccessor`接口提供了一种通用的方式来为 Bean 定义设置自定义属性，这些属性可以是任何类型的对象，并且可以由 BeanFactory 或其他组件来检索和访问。
- 给 BeanDefinition存放一些额外的附加信息
- 比如：你写了个注解处理器，它想往某个 Bean 上打个标签或者加点信息，可以用这个接口 

###  `BeanMetadataElement`: 通常是用来获取来信息（如 XML 文件路径、配置类等）

```java
public interface BeanMetadataElement {
    @Nullable
    Object getSource();
}
```

- 追踪 Bean 的来源信息
  - 从哪个 XML 文件中定义的
  - 从哪个配置类、哪个注解生成的



## 常量
### Scope 常量

```java
String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;
```
- `SCOPE_SINGLETON`: 单例模式
- `SCOPE_PROTOTYPE`: 原型模式(多例模式)

```java
@Component
@Scope("singleton") // 默认值，其实可以不写   @Scope("prototype")
public class MyService {
  
}
```
- Spring 在创建 Bean 的 BeanDefinition 时，会读取它的 `@Scope` 注解，并根据注解的值来设置 Bean 的作用域。
- 后面 Spring 真正生成 Bean 时会判断
- 
```java
if ("singleton".equals(beanDefinition.getScope())) {
    // 从缓存中取
} else if ("prototype".equals(beanDefinition.getScope())) {
    // 直接 new 一个新对象
}
```


### Role 常量

```java
int ROLE_APPLICATION = 0;
int ROLE_SUPPORT = 1;
int ROLE_INFRASTRUCTURE = 2;
```

标识 Bean 的角色：
- 0: 普通用户自己定义的 Bean
- 1: 用于某些功能的辅助 Bean，例如 AOP 代理 Bean
- 2：Spring 框架内部使用的 Bean，例如 BeanPostProcessor Bean


## 核心方法分类

### 依赖结构

```java
String getParentName();
void setParentName(String parentName);
```

- 允许一个 Bean 定义继承另一个，类似 XML 中的`<bean parent="...">`

### Bean 的创建方式

```java
String getBeanClassName();
void setBeanClassName(String beanClassName);

String getFactoryBeanName();
void setFactoryBeanName(String factoryBeanName);

String getFactoryMethodName();
void setFactoryMethodName(String factoryMethodName);
```
支持以下几种创建方式：
- `getBeanClassName()`: 普通类名 + 构造函数， `new MyService()`
- `getFactoryBeanName()`: 工厂 Bean + 工厂方法，调用一个工厂对象，再调用它的方法，`factoryBean.create()`
- `getFactoryMethodName()`: 静态工厂方法(无工厂 Bean),调用一个静态方法返回 bean, `MyFactory.createService()`


### 懒加载
```java
boolean isLazyInit();
void setLazyInit(boolean lazyInit);
```

- `true` 表示懒加载，只有真正请求这个 Bean 的时候，才会创建它
- `false` Spring 启动时就会创建这个 Bean

#### 什么时候用懒加载？
懒加载常用于 性能优化，尤其是不确定某些 Bean 是否一定会用到时。懒加载能避免 Spring 容器启动时加载不必要的 Bean，提高启动速度。

```java
@Bean
@Lazy
public MyService myService() {
    return new MyService();
}
```


### 依赖关系 dependsOn

```java
String[] getDependsOn();
void setDependsOn(String[] dependsOn);
```

- `dependsOn` 属性允许指定当前 Bean 依赖于其他 Bean 的初始化顺序
- Spring 会确保在当前 Bean 之前，先初始化它所依赖的 Bean
- 可以指定一个或多个 Bean 名字

```java
@Bean
@DependsOn("myDependency")
public MyService myService() {
    return new MyService();
}
```

#### 为什么要用 dependsOn？
- 在某些情况下，某些 Bean 必须在其他 Bean 之前初始化（例如，一些需要提前配置的资源或连接池）。
- 通过 dependsOn 可以明确指定这个依赖关系，确保初始化顺序正确。

### autowiredCandidate 自动注入候选

```java
boolean isAutowireCandidate();
void setAutowireCandidate(boolean autowireCandidate);
```

#### 自动注入候选是什么
- `autowireCandidate` 属性控制当前 Bean 是否可以自动注入到其他 Bean 中。默认情况下，Spring 会将每个 Bean 都作为自动注入候选。 
  - `true`：当前 Bean 可以作为自动注入的候选对象。 
  - `false`：当前 Bean 不会作为自动注入候选对象。

#### 什么时候设置为 false
- 在某些情况下，你可能希望将某个 Bean 从自动注入的候选中排除。
- 尤其是当有多个 Bean 可以注入时（比如多个 DataSource），你可以通过设置 autowireCandidate = false 来排除它。

```java
@Bean
@Primary
public DataSource dataSource1() {
    return new DataSource();
}

@Bean
@AutowireCandidate(false) // 不作为自动注入候选
public DataSource dataSource2() {
    return new DataSource();
}
```


### primary 首选 Bean

```java
boolean isPrimary();
void setPrimary(boolean primary);
```

#### 什么是首选 Bean？
- 当多个候选 Bean 都能满足自动注入要求时，Spring 会选一个 Bean 注入。通过 primary 属性，你可以指定哪个 Bean 是首选的。 
  - true：这个 Bean 是首选，Spring 会优先注入这个 Bean。 
  - false：不是首选。

```java
@Bean
@Primary
public MyService myService1() {
    return new MyService();
}

@Bean
public MyService myService2() {
    return new MyService();
}
```
- Spring 会优先注入 myService1，因为它被标记为 @Primary。


