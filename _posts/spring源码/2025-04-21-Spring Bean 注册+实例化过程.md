---
title: "Spring Bean 注册+实例化过程"
date: 2025-04-21
categories: [ Spring源码 ]
tags: [ Spring, Bean实例化 ]
---

| 概念   | 注册 Bean                              | 实例化 Bean              |
|------|--------------------------------------|-----------------------|
| 是什么  | 告诉 Spring 容器：有这么个 Bean，它叫什么，用哪个类来创建等 | Spring 真的去创建对象的实例     |
| 发生时间 | Spring 容器启动阶段，注册所有定义                 | 根据作用域：启动时（单例），使用时（原型） |
| 技术对象 | `BeanDefinition`                     | 真实的 Java 对象           |

# 📝 Spring Bean 注册

在 Spring 框架中，Spring Bean 的注册本质上是将 BeanDefinition 信息存储到 IoC 容器中，使得容器能够对
Bean 进行管理和实例化。目前 Spring 提供了多种灵活的 Bean 注册方式

## XML 配置元信息

通过 `<bean>` 标签来配置 Bean 的元数据

```xml

<bean id="myService" class="com.example.MyService">
  <property name="username" value="张三"/>
</bean>
```

## 注解配置元信息

通过 `@Bean`、`@Component`、`@Service`、`@Repository`、`@Controller` 等注解来配置 Bean 的元数据
`@Bean` 注解：用在配置类中 `@Configuration`

```java

@Configuration
public class MyConfiguration {
  @Bean
  public MyService myService() {
    return new MyService();
  }
}
```

`@Component` 注解

```java

@Component
public class UserService {
  private String username;
}
```

`@Import` 注解
导入其他配置类或者普通类，将其注册为 Bean

```java

@Configuration
@Import(MyService.class)
public class MyConfig {

}
```

## Java API 配置元信息

### 通过 Spring 的 ApplicationContext 编程方式注册（动态注册）

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
context.

register(MyService .class);
context.

refresh();

MyService myService = context.getBean(MyService.class);
```

### BeanDefinitionRegistry 来动态注册 Bean

#### ✳️ 实现 ImportBeanDefinitionRegistrar（适合 SDK 封装）

这种方式更适合做成 SDK，用户只需要加一个注解就能触发 Bean 的动态注册。

1. 创建一个 Registrar 类：

```java
public class MyBeanRegistrar implements ImportBeanDefinitionRegistrar {
  @Override
  public void registerBeanDefinitions(
    AnnotationMetadata importingClassMetadata,
    BeanDefinitionRegistry registry) {

    // 创建 BeanDefinition（方式1：用 GenericBeanDefinition）
    GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
    beanDefinition.setBeanClass(MyService.class);
    beanDefinition.setScope(BeanDefinition.SCOPE_SINGLETON);

    // 注册到容器中
    registry.registerBeanDefinition("myService", beanDefinition);
  }
}
```

2. 创建一个自定义注解，触发这个注册器

```java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(MyBeanRegistrar.class)
public @interface EnableMyService {
}

```

3. 在主工程中使用：

```java

@SpringBootApplication
@EnableMyService
public class MyApp {
  public static void main(String[] args) {
    SpringApplication.run(MyApp.class, args);
  }
}
```

### ✳️ 通过 BeanDefinitionRegistryPostProcessor（适合框架扩展）

如果你想在 Bean 加载阶段做更多控制，可以实现这个接口：

```java

@Component
public class MyBeanRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {

  @Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    RootBeanDefinition beanDefinition = new RootBeanDefinition(MyService.class);
    registry.registerBeanDefinition("myService", beanDefinition);
  }

  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    // 可选：你可以在这里对 BeanFactory 做进一步处理
  }
}

```

# Spring Bean 实例化

✅ 构造器实例化（默认方式）

```java

@Component
public class MyService {
  public MyService() {
    System.out.println("构造器被调用");
  }
}
```

Spring 默认用无参构造函数实例化（或自动选择构造函数）。

✅ 静态工厂方法实例化

```java
public class MyServiceFactory {
  public static MyService createInstance() {
    return new MyService();
  }
}
```

```java

@Configuration
public class Config {
  @Bean
  public MyService myService() {
    return MyServiceFactory.createInstance();
  }
}
```

✅ 实例工厂方法实例化

```java
public class MyServiceFactory {
  public MyService createService() {
    return new MyService();
  }
}
```

```java

@Configuration
public class Config {
  @Bean
  public MyServiceFactory myServiceFactory() {
    return new MyServiceFactory();
  }

  @Bean
  public MyService myService(MyServiceFactory factory) {
    return factory.createService();
  }
}
```

假设有这么一个类：

```java
public class UserService {
  private String username;
  private int port;

  public UserService(String username) {
    this.username = username;
  }

  public void setPort(int port) {
    this.port = port;
  }

  public void print() {
    System.out.println("username = " + username + ", port = " + port);
  }
}
```

# 🛠️ 第一步：创建 BeanDefinition 并设置构造参数和属性

```java
GenericBeanDefinition bd = new GenericBeanDefinition();
bd.

setBeanClass(UserService .class);

// 设置构造器参数：new UserService("admin")
ConstructorArgumentValues cav = new ConstructorArgumentValues();
cav.

addGenericArgumentValue("admin");
bd.

setConstructorArgumentValues(cav);

// 设置 setter 注入：setPort(8080)
MutablePropertyValues mpv = new MutablePropertyValues();
mpv.

add("port",8080);
bd.

setPropertyValues(mpv);
```

# 🧠 第二步：注册到 BeanFactory

我们手动用 DefaultListableBeanFactory，Spring 最常用的容器实现。

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();

// 注册 BeanDefinition 到容器
factory.

registerBeanDefinition("userService",bd);
```

# 🚀 第三步：获取 Bean（触发实例化 + 注入）

```java
UserService userService = (UserService) factory.getBean("userService");
userService.

print(); // 输出：username = admin, port = 8080
```

- Spring 通过 ConstructorArgumentValues 找到了 UserService(String) 构造器，并调用它
- 然后通过 MutablePropertyValues 找到 setPort() 方法，并注入了 8080
- 最后你拿到的是已经 fully-initialized 的 Bean！

# 📚 如果有多个构造器，Spring 是怎么选择的？

- 如果注册了一个 BeanDefinition，里面设置了 ConstructorArgumentValues，Spring 在实例化 Bean
  时就会用这些信息去找一个 匹配的构造器。

比如有一个类：

```java
public class MyService {
  public MyService() {
  }

  public MyService(String name) {
  }

  public MyService(String name, int port) {
  }
}
```

Spring 是如何知道用哪一个构造器的？

- 它会匹配参数数量 + 类型。

Spring 经典容器 DefaultListableBeanFactory#getBean() 最终会走到：

```java
AbstractAutowireCapableBeanFactory#

doCreateBean()
```

然后调用：

```java
instantiateBean(beanName, mbd);
```

接着再走进：

```java
instantiateUsingFactoryMethod(); // 如果你用了工厂方法

// 否则走：
autowireConstructor(); // 👉 我们重点关注这个
```

方法全名：

```java
org.springframework.beans.factory.support.ConstructorResolver#autowireConstructor
```

- 🔍 第一步：获取类的所有构造器

```java
Constructor<?>[] candidates = clazz.getDeclaredConstructors();
```

- 🔍 第二步：遍历所有构造器，尝试匹配 ConstructorArgumentValues 里提供的参数
  - 比如写了

```java
ConstructorArgumentValues cav = new ConstructorArgumentValues();
cav.

addGenericArgumentValue("admin");
cav.

addGenericArgumentValue(8080);
```

- 那 Spring 会在所有构造器中找出：

```java
MyService(String, int);
```

匹配得最精确的优先用。

- 🔍 第三步：支持自动装配（如果你没有手动指定 constructor 参数）
  - 如果你没指定 ConstructorArgumentValues，Spring 还可以自动尝试装配参数（基于 @Autowired、类型匹配等）。

## Spring 构造器选择的优先级：

| 优先级 | 情况                                            |
|-----|-----------------------------------------------|
| 最高	 | 显式指定了 ConstructorArgumentValues，按类型+个数精确匹配构造器 |
| 中等	 | 没指定参数，但类上用 @Autowired 指定了构造器                  |
| 最低	 | 自动推断（构造器参数最少的那个，或唯一的那个）                       |


