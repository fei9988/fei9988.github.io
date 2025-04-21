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
