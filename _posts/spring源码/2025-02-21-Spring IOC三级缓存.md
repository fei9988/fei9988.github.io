---
title: "Spring IOC源码解析"
date: 2025-02-21
categories: [Spring源码]
tags: [Spring,IOC,三级缓存]
---

# Spring 的三级缓存是为了解决什么问题？
- 循环依赖问题：当两个 Bean 相互依赖时，Spring 会使用三级缓存来解决循环依赖问题。具体来说，当容器创建 Bean 时，如果检测到某个 Bean 正在初始化，还没有完成，它就会从二级缓存中获取到一个半初始化的 Bean。然后将这个半初始化的 Bean 传递给另一个 Bean，实现依赖注入。这样可以避免无限递归地创建 Bean。 
- 性能优化：通过分层缓存管理，可以避免重复创建同一个 Bean，提高性能，尤其是在单例 Bean 创建过程中，避免了重复的初始化操作。

# 循环依赖
循环依赖是指在 Bean 之间存在依赖关系，并且它们相互依赖，导致在初始化过程中出现循环引用的问题。例如：A 依赖于 B，B 依赖于 C，C 依赖于 A。

# Spring 中的三级缓存
1. SingletonObjects（一级缓存）：用于缓存完全初始化的单例 Bean 实例。
2. EarlySingletonObjects（二级缓存）：用于缓存早曝光的单例 Bean 实例。这些实例还没有完全初始化。
3. SingletonFactories（三级缓存）：用于缓存用于创建早期曝光单例 Bean 的工厂。这些工厂可以生成 Bean 实例，并将它们放入二级缓存中。

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
    /** Cache of singleton objects: bean name to bean instance. */
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    /** Cache of singleton factories: bean name to ObjectFactory. */
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

    /** Cache of early singleton objects: bean name to bean instance. */
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
}
```

# 流程
1. 获取所有的 bean 的 Definition
2. 遍历
3. getBean(BeanA)、doGetBean（BeanA）
   1. getSingleton(beanA)
      1. 从一级缓存拿，没有
      2. isSingletonCurrentlyInCreation(beanName)，beanA 不在创建中，返回 null
4. 创建 beanA 实例，getSingleton(beanA, ObjectFactory<?> singletonFactory)
   1. 从一级缓存拿，没有
   2. 标记为创建中：beforeSingletonCreation：Set<String> singletonsCurrentlyInCreation 加入这个 beanName
   3. createBean
5. 创建 beanA：doCreateBean
   1. 将创建 beanA 的工厂方法放入第三级缓存中
6. addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
7. 注入依赖：polulateBean
   1. 处理InstantiationAwareBeanPostProcessor（@Autowired），执行postProcessProperties 方法
      1. 获取 beanB：getBean
         1. 从三级缓存中取 getSingleton(beanB)
            1. 一级缓存中没有
            2. 判断 beanB 是否在创建中，否，返回 null
         2. 将 beanB 标记为创建中
         3. 创建 beanB：createBean、doCreateBean
            1. 将创建 beanB 的工厂方法加入第三级缓存
            2. 注入 beanB 的依赖：populationBean
               1. getBean(beanA)
                  1. 一级缓存没有
                  2. beanA 已经在创建中，获取二级缓存，不存在
                  3. 获取第三级缓存，执行工厂方法getEarlyBeanReference(beanName, mbd, bean)
                     1. 判断是否需要代理
                  4. 将获取到的 beanA 放入第二级缓存
                  5. 删除第三级缓存中的 beanA
                  6. 将 beanA 注入 beanB
            3. 初始化 beanB
         4. 将 beanB 的创建中状态删去
         5. 将 beanB 加入第一级缓存，删除二三级的缓存
      2. 将 beanB 注入到 beanA 中
8. 初始化 beanA
   1. 将 beanA 的创建中状态删去
   2. 将 beanB 加入第一级缓存，删除二三级的缓存

# 能否只要第一级和第三级缓存
1. 可能导致生成多个不同实例（破坏单例）
   - 第三级缓存中保存的是 bean 工厂（`ObjectFactory`），每次调用`getObject()`生成一个新对象
   - 假如没有第二级缓存，如果 BeanA和多个 Bean 循环依赖，那么会导致 BeanA 被多次初始化，从而破坏单例
   - 正常流程：
     - 创建 BeanA，放入第三级缓存中
       - BeanA 依赖注入，依赖 BeanB 和 BeanC
       - 创建 BeanB，放入第三级缓存中
       - BeanB依赖注入，依赖 BeanA
         - 从第三级缓存中获取 BeanA，调用 `getObject()`，创建 BeanA，放入第二级缓存中
         - 返回 BeanA，注入到 BeanB中
       - BeanC依赖注入，依赖 BeanA
         - 从第二级缓存中获取 BeanA，返回 BeanA，注入到 BeanC中
   
  - 如果没有第二级缓存，那么每次都需要从第三级缓存中获取 BeanA，创建 BeanA，放入第二级缓存中，导致 BeanA 被多次初始化，从而破坏单例

# 能否只要第一级和第二级缓存
- 如果有代理对象，需要在初始化之前生成，要提前暴露
- 代理对象正常是在 bean 初始化之后生成的，如果没有第三级缓存延迟生成，那么就需要所有的 bean 都提前暴露，造成性能浪费。
- 在有三级缓存的情况下，只有循环引用才会调用工厂方法提前暴露，没有发生循环依赖的 bean 就是正常的创建流程，不会提前暴露。

# 三级缓存中为什么第一级和第三级是`concurrentHashMap`,第二级是`HashMap`
- 第一级和第三级缓存都是`ConcurrentHashMap`，是为了保证并发环境下的线程安全。
- 第二级缓存是`HashMap`，是为了保证循环依赖的场景下，能够快速获取到已经创建的 bean，而不需要每次都去第三级缓存中获取。
- 这样，在循环依赖的情况下，可以减少不必要的创建 bean 的次数，提高性能
