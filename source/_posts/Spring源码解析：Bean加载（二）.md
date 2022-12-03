---
title: Spring源码解析：Bean加载（二）
date: 2022-11-26 21:01:38
tags: Spring源码解析
categories: Spring
description: createBean() 方法主流程梳理。
---

## Spring 对 Scope 扩展的支持 

在[Spring源码解析：Bean加载（一）](/posts/9e6d785ab2c8)中，讲到了 `#doGetBean()` 方法中的 10 步，其中不同 Scope 创建 bean 的逻辑未深入解释，同时，也提到它们的区别主要在于 bean 实例的生命周期，但是创建 bean 都是调用的 `#createBean()` 方法。

```java
				if (mbd.isSingleton()) {
					// 9.1. 创建单例 bean
					// 根据 beanName 获取单例 bean, 并传入一个 BeanFactory 函数时接口的回调
					// 当 bean 不存在时，需要调用这个回调函数来创建 bean
					sharedInstance = getSingleton(beanName, () -> {
						try {
							// 当 bean 不存在时，最终会调用到这里的 creatBean 方法来创建单例 bean
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
				else if (mbd.isPrototype()) {
					// 9.2. 创建 prototype 模式 bean
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						// 设置 bean 正在创建中
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					// FactoryBean 处理
					beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					// 9.3. 其他自定义 Scope bean 创建
					String scopeName = mbd.getScope();
					if (!StringUtils.hasLength(scopeName)) {
						throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
					}
					Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						// 指定 Scope 进行初始化
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
```

本文将开始对 `#createBean()` 方法深入解析，但是该方法实现比较复杂，所以也需要分成几篇博文进行讲解。

## createBean()

```java
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;
		// 1. 加载 class
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}
    
		try {
			// 2. 准备覆盖的方法，整理的 MethodOverride 是前面解析 BeanDefinition 时的 replaced-method、lookup-method 两个标签的解析结果
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			//3. 给BeanPostProcessor一个返回代理而不是目标 bean 实例的机会，AOP 就是在这一步实现
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
			// 4. 通过 beanName BeanDefinition 创建 bean 实例
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```

主要分为以下 4 步：

1. 加载 bean 对应的的 Class。
2. 方法重写准备。
3. bean 实例化前处理，主要是调用 BeanPostProcessor。
4. 调用 `#doCreateBean()` 方法创建 bean。

### 加载 bean 对应 Class

```java
	protected Class<?> resolveBeanClass(RootBeanDefinition mbd, String beanName, Class<?>... typesToMatch)
			throws CannotLoadBeanClassException {

		try {
			// 如果 BeanDefinition 中已经有了 bean 的 Class 类型，则直接返回
			if (mbd.hasBeanClass()) {
				return mbd.getBeanClass();
			}
			if (System.getSecurityManager() != null) {
				return AccessController.doPrivileged((PrivilegedExceptionAction<Class<?>>)
						() -> doResolveBeanClass(mbd, typesToMatch), getAccessControlContext());
			}
			else {
				// 加载 bean Class
				return doResolveBeanClass(mbd, typesToMatch);
			}
		}
		catch (PrivilegedActionException pae) {
			ClassNotFoundException ex = (ClassNotFoundException) pae.getException();
			throw new CannotLoadBeanClassException(mbd.getResourceDescription(), beanName, mbd.getBeanClassName(), ex);
		}
		catch (ClassNotFoundException ex) {
			throw new CannotLoadBeanClassException(mbd.getResourceDescription(), beanName, mbd.getBeanClassName(), ex);
		}
		catch (LinkageError err) {
			throw new CannotLoadBeanClassException(mbd.getResourceDescription(), beanName, mbd.getBeanClassName(), err);
		}
	}
```

如果 RootBeanDefinition 中已经指定了 Class，则直接返回，如果未指定，则加载相应的 Class，这个逻辑比较简单，其实也不用关心，不赘述。

### 方法重写准备

```java
	private MethodOverrides methodOverrides = new MethodOverrides();

	public void prepareMethodOverrides() throws BeanDefinitionValidationException {
		// Check that lookup methods exist and determine their overloaded status.
		if (hasMethodOverrides()) {
			getMethodOverrides().getOverrides().forEach(this::prepareMethodOverride);
		}
	}

	public boolean hasMethodOverrides() {
		return !this.methodOverrides.isEmpty();
	}

	public MethodOverrides getMethodOverrides() {
		return this.methodOverrides;
	}
```

会发现这里是针对 MethodOverride 的逻辑，这个接口在[Spring源码解析：BeanDefinition](posts/d13dfeffc92a)中解析 [lookup-method](/posts/d13dfeffc92a#解析-lookup-method-标签) 和 [replaced-method](/posts/d13dfeffc92a#%E8%A7%A3%E6%9E%90-replaced-method-%E6%A0%87%E7%AD%BE) 两个标签时提到，解析完成后，分别生成了 LookupOverride、ReplaceOverride 两个类型的对象放到了 BeanDefinition 中，这里的 methodOverrides 其实就是这两个对象了。

然后再看看 `#prepareMethodPverride()` 方法的逻辑

```java
	protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
		// 获取 class 中指定方法名的方法个数
		int count = ClassUtils.getMethodCountForName(getBeanClass(), mo.getMethodName());
		if (count == 0) {
			throw new BeanDefinitionValidationException(
					"Invalid method override: no method with name '" + mo.getMethodName() +
					"' on class [" + getBeanClassName() + "]");
		}
		else if (count == 1) {
			// 标记为未 Overload，避免检查 arg 类型时多余的开销
			// Mark override as not overloaded, to avoid the overhead of arg type checking.
			mo.setOverloaded(false);
		}
	}
```

会发现这里是拿到了 Class 中和要做 override 操作的方法同名的方法（即方法重载）的数量，如果为 0，说明方法名不存在，抛异常，如果数量为 1，则说明没有被重载，将 overloaded 置为 false，否则保持为默认的 true。

> 这可以认为是一个前置的小优化。

### bean 实例化前处理

```java
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
	// ......
    	try {
			//3. 给BeanPostProcessor一个返回代理而不是目标 bean 实例的机会，AOP 就是在这一步实现
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
  // ......
  }

	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			// Make sure bean class is actually resolved at this point.
			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
				Class<?> targetType = determineTargetType(beanName, mbd);
				if (targetType != null) {
					// 执行 BeanPostProcessor 的 before
					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
					if (bean != null) {
						// 执行 BeanPostProcessor 的 after 方法，到这里说明已经创建了 bean 实例，猜测这里拿到的是动态代理生成的 bean 实例
						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		return bean;
	}
```

这一步处理得到的 bean 实例如果不为空，则整个 createBean() 方法就直接返回了，即这一步处理得到的非空结果就是最终的 bean 实例对象了。

其实这里就是做一些前置处理，如果 bean 需要代理，那么这里会返回 bean 的代理对象。

> 先记住这里返回的是代理对象，后面讲循环依赖时，会再次讲到这里。

### doCreateBean() 方法

doCreateBean() 方法主要做 bean 实例化、属性填充、初始化，同时还有循环依赖的一些逻辑。

循环依赖问题的解决并不是在某一个地方的逻辑解决的，而是加载 bean 的很多个环节都有相关逻辑，各个地方相互配合才解决了循环依赖问题。

```java
	protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// 1. 实例化 bean 对象
		// BeanWrapper 提供的是操作 bean 实例的基础能力
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			// 创建 bean 实例
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
					// 2. mergeBeanDefinitionPostProcessor，@Autowired 注解的预解析就在这里
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		// 3. 解决循环依赖
		// 是否支持单例 bean 实例提前暴露，用于解决循环依赖问题，Spring 目前只解决了单例 bean 的循环依赖
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			// 为解决循环依赖做准备，在这里将 beanName 和 bean 的对象工厂放进缓存中
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			// 4. 填充 bean，注入各个属性值，如果要注入的 bean 没有初始化，那也会初始化要注入的 bean
			populateBean(beanName, mbd, instanceWrapper);
			// 5. 初始化
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
			// 6. 这里只会从一级、二级缓存里面拿 bean 实例，但是因为本身就是还在创建，所以这里其实是从二级缓存里面拿，但是二级缓存里面不一定有
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				// 不为空，已经从三级缓存进入到了二级缓存
				// 是同一个对象，说明拿到的是没有被增强的实例
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
					// 到这里如果 actualDependentBeans 不为空，说明这个 bean 所依赖的 bean 还没有创建完，
					// 存在初始化顺序上的循环依赖
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
			// 7. 根据 scope 注册 bean 的 destroy-method
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}
		return exposedObject;
	}
```

`#doCreateBean()` 方法分以下 7 步：

1. 实例化 bean 对象，这里实例化，其实就是使用构造函数构建一个对象，但是构建出来的对象的各个属性可能是 null。这取决于 bean 的注入方式，这一步会讲通过构造函数注入的依赖先完成，对于使用 set 方法注入和直接使用 @Autowired 注解注入的，则是后续填充属性时注入。
2. 执行 mergeBeanDefinitionPostProcessor，一些前置处理，`@Autowired` 注解的解析就在这里。
3. 解决循环依赖的逻辑，这俩主要是 beanName 和 ObjectFactory 方法第三级缓存中。循环依赖整体的解决方案，后面单独写博文讲解。
4. 填充 bean 属性。这一步主要是将 @Autowired 注解和 set 方法进行注入依赖做填充操作。
5. 执行 bean 初始化方法，执行 init-method 标签、@PostConstruct 标记的方法。
6. 调用 `#getSingleton()` 方法，需要注意这里调用时，传入的第二个参数为 false，所以这里其实是尝试从二级缓存中获取 bean 实例，不会调用第三级缓存。
7. 注册 bean 的销毁方法。

