---
title: "Spring IOC源码解析"
date: 2025-02-21
categories: [Spring源码]
tags: [Spring,IOC,三级缓存, 源码]
---

```java
// class DefaultListableBeanFactory

/**
** 提前实例化单例 Bean。在容器启动时，确保所有非懒加载的单例 Bean 都被创建和初始化。
**/
public void preInstantiateSingletons() throws BeansException {
    // 标记正在进行单例 Bean 的预实例化过程
    if (logger.isTraceEnabled()) {
       logger.trace("Pre-instantiating singletons in " + this);
    }

    // 这里将 beanDefinitionNames（包含所有 Bean 定义的名称）转为一个新的 ArrayList，
    // 这是为了避免在迭代过程中修改原始集合，因为 Bean 初始化过程可能会修改容器中的定义。
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // 遍历 Bean 定义，实例化非懒加载的单例 Bean
    for (String beanName : beanNames) {
       // 首先获取该 Bean 的合并定义
       RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
       // 判断 Bean 是否是单例（isSingleton()）、是否为抽象 Bean（isAbstract()，跳过）
       // 以及是否是懒加载（isLazyInit()，懒加载的 Bean 会被跳过）。
       if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
          // 如果 Bean 是一个 FactoryBean，那么会特别处理。FactoryBean 用于创建其他类型的 Bean，
          // 而不是直接返回其自身对象。会检查 SmartFactoryBean 接口的 isEagerInit() 方法，
          // 判断是否需要提前初始化。
          if (isFactoryBean(beanName)) {
             Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
             if (bean instanceof FactoryBean) {
                FactoryBean<?> factory = (FactoryBean<?>) bean;
                boolean isEagerInit;
                if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                   isEagerInit = AccessController.doPrivileged(
                         (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
                         getAccessControlContext());
                }
                else {
                   isEagerInit = (factory instanceof SmartFactoryBean &&
                         ((SmartFactoryBean<?>) factory).isEagerInit());
                }
                if (isEagerInit) {
                   getBean(beanName);
                }
             }
          }
          else {
             getBean(beanName);
          }
       }
    }

    //  触发所有适用的 Bean 的初始化后回调
    for (String beanName : beanNames) {
       // 遍历所有的 Bean 名称，调用 getSingleton(beanName) 获取 Bean 实例。
       Object singletonInstance = getSingleton(beanName);
       // 如果该实例是 SmartInitializingSingleton 类型，说明它需要在所有单例 Bean 初始化完成后执行某些额外的初始化逻辑
       if (singletonInstance instanceof SmartInitializingSingleton) {
          StartupStep smartInitialize = this.getApplicationStartup().start("spring.beans.smart-initialize")
                .tag("beanName", beanName);
          SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
          if (System.getSecurityManager() != null) {
             AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                smartSingleton.afterSingletonsInstantiated();
                return null;
             }, getAccessControlContext());
          }
          else {
             smartSingleton.afterSingletonsInstantiated();
          }
          smartInitialize.end();
       }
    }
}
```

```java
// class AbstractBeanFactory

protected <T> T doGetBean(
       String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
       throws BeansException {
    // 将 name 转换为标准化的 beanName
    String beanName = transformedBeanName(name);
    Object beanInstance;

    // 尝试从单例缓存中获取 Bean
    // 如果是单例模式，且 Bean 已经存在于缓存中，直接返回缓存中的实例。
    // 如果该 Bean 尚未完全初始化（isSingletonCurrentlyInCreation），可能是由于循环依赖，此时会返回部分初始化的实例。
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
       if (logger.isTraceEnabled()) {
          if (isSingletonCurrentlyInCreation(beanName)) {
             logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                   "' that is not fully initialized yet - a consequence of a circular reference");
          }
          else {
             logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
          }
       }
       beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    else {
       // 检查当前 Bean 是否正在创建（原型模式不支持循环依赖）。
       // 如果检测到循环依赖，抛出异常。
       if (isPrototypeCurrentlyInCreation(beanName)) {
          throw new BeanCurrentlyInCreationException(beanName);
       }

       // Check if bean definition exists in this factory.
       BeanFactory parentBeanFactory = getParentBeanFactory();
       if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
          // Not found -> check parent.
          String nameToLookup = originalBeanName(name);
          if (parentBeanFactory instanceof AbstractBeanFactory) {
             return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                   nameToLookup, requiredType, args, typeCheckOnly);
          }
          else if (args != null) {
             // Delegation to parent with explicit args.
             return (T) parentBeanFactory.getBean(nameToLookup, args);
          }
          else if (requiredType != null) {
             // No args -> delegate to standard getBean method.
             return parentBeanFactory.getBean(nameToLookup, requiredType);
          }
          else {
             return (T) parentBeanFactory.getBean(nameToLookup);
          }
       }
        
       // 如果不是仅进行类型检查（typeCheckOnly），会将该 Bean 标记为已创建。
       if (!typeCheckOnly) {
          markBeanAsCreated(beanName);
       }

       StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
             .tag("beanName", name);
       try {
          if (requiredType != null) {
             beanCreation.tag("beanType", requiredType::toString);
          }
          // 获取并合并 Bean 的本地定义，确保定义完整有效
          RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
          checkMergedBeanDefinition(mbd, beanName, args);

          // 如果当前 Bean 定义中声明了 depends-on，需要先初始化这些依赖。
          String[] dependsOn = mbd.getDependsOn();
          if (dependsOn != null) {
             for (String dep : dependsOn) {
                if (isDependent(beanName, dep)) {
                   throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                         "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                }
                registerDependentBean(dep, beanName);
                try {
                   getBean(dep);
                }
                catch (NoSuchBeanDefinitionException ex) {
                   throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                         "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                }
             }
          }

          //  创建实例
          // 单例
          if (mbd.isSingleton()) {
             // 标记创建中
             sharedInstance = getSingleton(beanName, () -> {
                try {
                   return createBean(beanName, mbd, args);
                }
                catch (BeansException ex) {
                   // Explicitly remove instance from singleton cache: It might have been put there
                   // eagerly by the creation process, to allow for circular reference resolution.
                   // Also remove any beans that received a temporary reference to the bean.
                   destroySingleton(beanName);
                   throw ex;
                }
             });
             beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
          }
          // 原型
          else if (mbd.isPrototype()) {
             // It's a prototype -> create a new instance.
             Object prototypeInstance = null;
             try {
                beforePrototypeCreation(beanName);
                prototypeInstance = createBean(beanName, mbd, args);
             }
             finally {
                afterPrototypeCreation(beanName);
             }
             beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
          }
          // 自定义作用域，通过 Scope 接口管理
          else {
             String scopeName = mbd.getScope();
             if (!StringUtils.hasLength(scopeName)) {
                throw new IllegalStateException("No scope name defined for bean '" + beanName + "'");
             }
             Scope scope = this.scopes.get(scopeName);
             if (scope == null) {
                throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
             }
             try {
                Object scopedInstance = scope.get(beanName, () -> {
                   beforePrototypeCreation(beanName);
                   try {
                      return createBean(beanName, mbd, args);
                   }
                   finally {
                      afterPrototypeCreation(beanName);
                   }
                });
                beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
             }
             catch (IllegalStateException ex) {
                throw new ScopeNotActiveException(beanName, scopeName, ex);
             }
          }
       }
       catch (BeansException ex) {
          beanCreation.tag("exception", ex.getClass().toString());
          beanCreation.tag("message", String.valueOf(ex.getMessage()));
          cleanupAfterBeanCreationFailure(beanName);
          throw ex;
       }
       finally {
          beanCreation.end();
       }
    }

    return adaptBeanInstance(name, beanInstance, requiredType);
}
```

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "Bean name must not be null");
    synchronized (this.singletonObjects) {
       // 从一级缓存取
       Object singletonObject = this.singletonObjects.get(beanName);
       if (singletonObject == null) {
          // 一级缓存中不存在
          
          // 如果当前容器正在销毁单例实例，禁止创建新的单例，避免破坏销毁流程
          if (this.singletonsCurrentlyInDestruction) {
             throw new BeanCreationNotAllowedException(beanName,
                   "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                   "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
          }
          if (logger.isDebugEnabled()) {
             logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
          }
          
          // 标记当前 Bean 正在创建，防止循环依赖。
          beforeSingletonCreation(beanName);
          boolean newSingleton = false;
          boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
          if (recordSuppressedExceptions) {
             this.suppressedExceptions = new LinkedHashSet<>();
          }
          // 通过工厂创建单例对象   进入到 createBean 方法
          try {
             singletonObject = singletonFactory.getObject();
             newSingleton = true;
          }
          catch (IllegalStateException ex) {
             // Has the singleton object implicitly appeared in the meantime ->
             // if yes, proceed with it since the exception indicates that state.
             singletonObject = this.singletonObjects.get(beanName);
             if (singletonObject == null) {
                throw ex;
             }
          }
          catch (BeanCreationException ex) {
             if (recordSuppressedExceptions) {
                for (Exception suppressedException : this.suppressedExceptions) {
                   ex.addRelatedCause(suppressedException);
                }
             }
             throw ex;
          }
          finally {
             if (recordSuppressedExceptions) {
                this.suppressedExceptions = null;
             }
             afterSingletonCreation(beanName);
          }
          if (newSingleton) {
             addSingleton(beanName, singletonObject);
          }
       }
       return singletonObject;
    }
}
```

```java
// AbstractAutowireCapableBeanFactory

protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
       throws BeanCreationException {

    if (logger.isTraceEnabled()) {
       logger.trace("Creating instance of bean '" + beanName + "'");
    }
    RootBeanDefinition mbdToUse = mbd;

    // Make sure bean class is actually resolved at this point, and
    // clone the bean definition in case of a dynamically resolved Class
    // which cannot be stored in the shared merged bean definition.
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
       mbdToUse = new RootBeanDefinition(mbd);
       mbdToUse.setBeanClass(resolvedClass);
    }

    // Prepare method overrides.
    try {
       mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
       throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
             beanName, "Validation of method overrides failed", ex);
    }

    try {
       // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
       Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
       if (bean != null) {
          return bean;
       }
    }
    catch (Throwable ex) {
       throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
             "BeanPostProcessor before instantiation of bean failed", ex);
    }

    try {
       Object beanInstance = doCreateBean(beanName, mbdToUse, args);
       if (logger.isTraceEnabled()) {
          logger.trace("Finished creating instance of bean '" + beanName + "'");
       }
       return beanInstance;
    }
    catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
       // A previously detected exception with proper bean creation context already,
       // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
       throw ex;
    }
    catch (Throwable ex) {
       throw new BeanCreationException(
             mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
    }
}
```

```java
class AbstractAutowireCapableBeanFactory

protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
       throws BeanCreationException {

    // 实例化 bean   BeanWrapper：用于封装实例对象及其类型元信息。
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
       instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
       // 创建新实例
       instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
       mbd.resolvedTargetType = beanType;
    }

    // Allow post-processors to modify the merged bean definition.
    synchronized (mbd.postProcessingLock) {
       if (!mbd.postProcessed) {
          try {
             applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
          }
          catch (Throwable ex) {
             throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                   "Post-processing of merged bean definition failed", ex);
          }
          mbd.postProcessed = true;
       }
    }

    // isSingletonCurrentlyInCreation(beanName)  判断bean 是否在创建中
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
          isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
       if (logger.isTraceEnabled()) {
          logger.trace("Eagerly caching bean '" + beanName +
                "' to allow for resolving potential circular references");
       }
       
       // 放入第三级缓存中
       // getEarlyBeanReference：在 AOP 或代理场景下返回提前暴露的 Bean 引用。
       addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
       // 属性填充
       populateBean(beanName, mbd, instanceWrapper);
       exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
       if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
          throw (BeanCreationException) ex;
       }
       else {
          throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
       }
    }

    if (earlySingletonExposure) {
       Object earlySingletonReference = getSingleton(beanName, false);
       if (earlySingletonReference != null) {
          if (exposedObject == bean) {
             exposedObject = earlySingletonReference;
          }
          else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
             String[] dependentBeans = getDependentBeans(beanName);
             Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
             for (String dependentBean : dependentBeans) {
                if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                   actualDependentBeans.add(dependentBean);
                }
             }
             if (!actualDependentBeans.isEmpty()) {
                throw new BeanCurrentlyInCreationException(beanName,
                      "Bean with name '" + beanName + "' has been injected into other beans [" +
                      StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                      "] in its raw version as part of a circular reference, but has eventually been " +
                      "wrapped. This means that said other beans do not use the final version of the " +
                      "bean. This is often the result of over-eager type matching - consider using " +
                      "'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
             }
          }
       }
    }

    // Register bean as disposable.
    try {
       registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
       throw new BeanCreationException(
             mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}
```

```java

// populateBean 是 Spring 容器中用于给 Bean 填充属性的核心方法之一。
// 它通过读取 BeanDefinition 中定义的属性和依赖，结合 BeanWrapper 对象，完成对 Bean 属性的注入。
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    // 如果 BeanWrapper 为 null 且 BeanDefinition 中有属性需要设置，则抛出异常。
    if (bw == null) {
       if (mbd.hasPropertyValues()) {
          throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
       }
       else {
          // Skip property population phase for null instance.
          return;
       }
    }

    // 调用 InstantiationAwareBeanPostProcessor 的 postProcessAfterInstantiation 方法，
    // 允许用户在 Bean 属性设置之前对 Bean 进行进一步的定制。
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
       for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
          if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
             return;
          }
       }
    }

    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
       MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
       // Add property values based on autowire by name if applicable.
       if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
          autowireByName(beanName, mbd, bw, newPvs);
       }
       // Add property values based on autowire by type if applicable.
       if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
          autowireByType(beanName, mbd, bw, newPvs);
       }
       pvs = newPvs;
    }

    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
       if (pvs == null) {
          pvs = mbd.getPropertyValues();
       }

       // 如果有 InstantiationAwareBeanPostProcessor，
       // 调用其 postProcessProperties 和 postProcessPropertyValues 方法，允许修改或验证属性值。
       for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
          // bp 如果是 AutowiredAnnotationBeanPostProcessor，会处理 @Autowired 引入的依赖
          // bp.postProcessProperties 会执行getBean()，获取依赖的bean 
          PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
          if (pvsToUse == null) {
             if (filteredPds == null) {
                filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
             }
             pvsToUse = bp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
             if (pvsToUse == null) {
                return;
             }
          }
          pvs = pvsToUse;
       }
    }
    if (needsDepCheck) {
       if (filteredPds == null) {
          filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
       }
       checkDependencies(beanName, mbd, filteredPds, pvs);
    }
       
    // 将最终计算的 PropertyValues 应用于目标 Bean。
    // 由 applyPropertyValues 方法完成实际的属性填充。
    if (pvs != null) {
       applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```
