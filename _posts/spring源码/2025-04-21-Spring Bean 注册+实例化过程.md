---
title: "Spring Bean 注册+实例化过程"
date: 2025-04-21
categories: [Spring源码]
tags: [Spring, Bean实例化]
---

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
bd.setBeanClass(UserService.class);

// 设置构造器参数：new UserService("admin")
ConstructorArgumentValues cav = new ConstructorArgumentValues();
cav.addGenericArgumentValue("admin");
bd.setConstructorArgumentValues(cav);

// 设置 setter 注入：setPort(8080)
MutablePropertyValues mpv = new MutablePropertyValues();
mpv.add("port", 8080);
bd.setPropertyValues(mpv);
```

# 🧠 第二步：注册到 BeanFactory

我们手动用 DefaultListableBeanFactory，Spring 最常用的容器实现。

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();

// 注册 BeanDefinition 到容器
factory.registerBeanDefinition("userService", bd);
```

# 🚀 第三步：获取 Bean（触发实例化 + 注入）

```java
UserService userService = (UserService) factory.getBean("userService");
userService.print(); // 输出：username = admin, port = 8080
```


- Spring 通过 ConstructorArgumentValues 找到了 UserService(String) 构造器，并调用它 
- 然后通过 MutablePropertyValues 找到 setPort() 方法，并注入了 8080 
- 最后你拿到的是已经 fully-initialized 的 Bean！


# 📚 如果有多个构造器，Spring 是怎么选择的？
- 如果注册了一个 BeanDefinition，里面设置了 ConstructorArgumentValues，Spring 在实例化 Bean 时就会用这些信息去找一个 匹配的构造器。

比如有一个类：

```java
public class MyService {
    public MyService() {}
    public MyService(String name) {}
    public MyService(String name, int port) {}
}
```

Spring 是如何知道用哪一个构造器的？
- 它会匹配参数数量 + 类型。

Spring 经典容器 DefaultListableBeanFactory#getBean() 最终会走到：

```java
AbstractAutowireCapableBeanFactory#doCreateBean()
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
cav.addGenericArgumentValue("admin");
cav.addGenericArgumentValue(8080);
```

-   那 Spring 会在所有构造器中找出：
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


