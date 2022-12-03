---
title: Spring源码解析：Bean加载（一）
date: 2022-11-26 15:18:44
tags: Spring源码解析
categories: Spring
description: doGetBean 方法主流程梳理。
---

当我们要从一个 BeanFactory 中拿到某个 bean 时，我们会调用 `#getBean()` 系列（之所以是系列，是因为方法重载了好几次）方法，而 `#getBean()` 方法会调用 `#doGetBean(String name, Class<T> requiredType, Object[] args, boolean typeCheckOnly)` 方法，该方法有 4 个参数：

* name

  要获取的 bean 的名称。

* requireType

  要获取的 bean 的 Java 类型。

* args 

  构造 bean 时需要的参数。

* typeCheckOnly

  是否仅仅只是做类型检查。

虽然看方法名称，是用来获取 bean 实例对象的，但是在 Spring 初始化阶段，这个方法主要是用来做 bean 的初始化。

> **Q&A**
>
> Q: 初始化的入口在哪里？
>
> A: 在 `DefaultListableBeanFactory#preInstantiateSingletons()` 方法中，虽然在加载了 BeanDefinition 后、调用该方法前， Spring 也做了很多事，但是都是为了支持一些扩展功能的，不影响对 bean 初始化的理解。

# doGetBean() 方法主流程

在容器初始化 bean 时，调用 doGetBean() 方法的参数如下：

```java
	public Object getBean(String name) throws BeansException {
		// 真正执行获取实例的方法
		return doGetBean(name, null, null, false);
	}
```

接下来看看 doGetBean() 方法的实现：

```java
	protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {
		// 1. 处理别名或者传入的是 FactoryBean 的名字的情况
		String beanName = transformedBeanName(name);
		Object beanInstance;

		// Eagerly check singleton cache for manually registered singletons.
		// 2. 尝试从缓存中加载单例 bean
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
			// 3. 对 FactoryBean 的处理
			beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			// 4. 从单例缓存中没有找到 bean，检查是不是还在创建中，如果是，抛异常
			// 如果不是单例模式的 bean，但是又是在创建中，说明是二次创建，直接抛异常
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}
			// Check if bean definition exists in this factory.
			// 5. 如果有父 BeanFactory，当前 BeanFactory 中又没有 beanName 对应的 BeanDefinition 对象，就尝试从父容器获取
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// 当前 beanFactory 中未找到，从父 BeanFactory 中查找
				// 获取 beanName
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
			// 6. 如果不是只检查 bean 类型，那说明是要创建 bean，这里标签为已创建
			if (!typeCheckOnly) {
				// 这里将 BeanDefinition 的 stale 属性设置为了 true
				markBeanAsCreated(beanName);
			}

			StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
					.tag("beanName", name);
			try {
				if (requiredType != null) {
					beanCreation.tag("beanType", requiredType::toString);
				}
				// 7. 获取 merge 之后的 BeanDefinition，在最开始加载的时候，都是 GenericBeanDefinition
				// merge 其实就是转换成 RootBeanDefinition, 后续工作都基于 RootBeanDefinition 进行
				// 合并的时候，其实主要就是将原来的 GenericBeanDefinition 内容转移到 RootBeanDefinition，
				// 如果有父 BeanDefinition，也要合并到 RootBeanDefinition 中
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				// 如果当前 BeanDefinition 是抽象的 bean，抛异常
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				// 8. 依赖  bean 处理
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						// 这里是为了避免出现 A 和 B 得初始化顺寻互相依赖，和循环依赖得注入不一样
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						// 注册依赖关系, 容器中维护了 dependentBeanMap, key 是 beanName，
						// value 是依赖 key 所表示的 bean 的所有 bean 的集合
						registerDependentBean(dep, beanName);
						try {
							// 递归调用，获取依赖的 bean，这里其实主要是为了在依赖的 bean 没有创建时进行创建
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// 9. 创建 bean 实例
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
		// 10. 返回类型适配
		return adaptBeanInstance(name, beanInstance, requiredType);
	}
```

大体上分为 10 步：

1. beanName 处理。处理别名或者传入的是 FactoryBean 的名字的情况，拿到真实的 beanName。
2. 尝试从缓存中加载单例 bean。
3. 对 FactoryBean 的处理，如果拿到的 bean 是一个 FactoryBean，还需要通过 `#getObjectForBeanInstance()` 方法进一步处理，这虽然是低 3 步，但其实不管以怎样的方式拿到的 bean 实例，都需要有这一步处理，可以看见 doGetBean() 方法中多个地方都调用了 `#getObjectForBeanInstance()` 方法。
4. prototype 类型并发创建检查。
5. 如果当前 BeanFactory 中找不到目标 beanName，则尝试从父 BeanFactory 中查找。
6. 标识 beanName 对应的 bean 正在创建中。
7. 组装并检查 RootBeanDefinition。
8. 依赖 bean 处理，这里一般指使用 bean 标签的 `depends-on` 属性或 `@DependsOn` 注解指定的 bean，这种方式指定的依赖的 bean 必须比当前 bean 先创建。
9. 创建 bean 实例，这里分了 singlton、prototype、自定义 scope 3 种情况，具体的差别单独写博文讲解，但是都调用了 `#creatBean()` 来创建 bean 实例。
10. 返回 bean 类型适配。

因为这 10 步，有些比较简单，有些很复杂，所以本文对每一步做一个或浅或深的解析，复杂的步骤会单独再出一篇文章进行解析。

## beanName 处理

```java
	protected String transformedBeanName(String name) {
		return canonicalName(BeanFactoryUtils.transformedBeanName(name));
	}

	public String canonicalName(String name) {
		String canonicalName = name;
		// 别名处理，这里要拿到最终的 beanName 而不是别名
		String resolvedName;
		do {
			resolvedName = this.aliasMap.get(canonicalName);
			// 如果从 aliasMap 拿到的结果不是 null 说明，依然是别名，直到 resolvedName 为 null 时，说明拿到的是真实的 beanName
			if (resolvedName != null) {
				canonicalName = resolvedName;
			}
		}
		while (resolvedName != null);
		return canonicalName;
	}
```

beanName 的处理，主要是要做两件事：

1. 参数传入的 beanName 可能是别名，而 singletonObjects 中保存的是真实的 beanName 和 bean 实例的关系，所以这里需要拿到真实的 beanName，这段逻辑就是 `#canonicalName()` 方法，不再赘述。
2. 将 FactoryBean 的名称转换成最终生成的 bean 的名称，即去掉参数传入的 beanName 中的 `&`，这段逻辑在 `BeanFactoryUtils#transformedBeanName()` 方法中实现。

```java
	public static String transformedBeanName(String name) {
		Assert.notNull(name, "'name' must not be null");
		// 如果name不以 & 开头，说明不是 FactoryBean,直接返回即可
		if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
			return name;
		}
		// FactoryBean 的 beanName
		return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
			do {
				// 要删除 & 前缀
				beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
			}
			while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
			return beanName;
		});
```

考虑到 beanName 中 `&` 可能不止一个，这里是循环删除 beanName 中的 `&`，直到 beanName 不再是以 `&` 开始。

关于 FactoryBean 和 真实的 bean 之间的名称的关系，可以理解为名称为 `a` 的 bean，如果是通过 FactoryBean 生成，则对应的 FactoryBean 的名称为 `&a`。

##  从缓存中加载单例 bean 实例

```java
  public Object getSingleton(String beanName) {
     return getSingleton(beanName, true);
  }

	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// Quick check for existing instance without full singleton lock
		// singletonObjects，一级缓存，存储 beanName 和 bean 实例对象的映射关系
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null
				// 目标 bean 是否还在创建中
				&& isSingletonCurrentlyInCreation(beanName)) {
			// 从二级缓存中获取 bean 实例，二级缓存中的 bean 实例一般都是还没创建完成的实例
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null
					// 二级缓存中依然没有，如果允许提前创建，就访问三级缓存
					&& allowEarlyReference) {
				// 单例模式同步双重校验
				synchronized (this.singletonObjects) {
					// Consistent creation of early reference within full singleton lock
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						singletonObject = this.earlySingletonObjects.get(beanName);
						if (singletonObject == null) {
							ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
							if (singletonFactory != null) {
								// 从三级缓存中拿到 bean 的 ObjectFactory 对象，从而拿到 bean 的早期引用
								singletonObject = singletonFactory.getObject();
								// 将 bean 实例放入二级缓存
								this.earlySingletonObjects.put(beanName, singletonObject);
								// 将 beanName 和 ObjectFactory 的映射关系从三级缓存中删除
								this.singletonFactories.remove(beanName);
							}
						}
					}
				}
			}
		}
		return singletonObject;
	}

```

具体逻辑不赘述，因为代码中注释已经说明了。

需要注意的是，这里终于出现了常说的 Spring 三级缓存：

```java
	/* 一级缓存，存储单例 bean 名称和对应的最终版的 Bean 实例对象 */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/* 二级缓存，存储单例 bean 名称和对应的 Bean 的最初版本 */
	private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

	/* 三级缓存，存储单例 bean 名称和对应的 Bean 的函数式工厂对象 */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

* singletonObjects

  一级缓存，保存了 beanName 和最终可用的 bean 实例对象的映射关系。

* earlySingletonObjects

  二级缓存，保存了 beanName 和实例化的 bean 对象的映射关系。

  > 这里暂且认为实例化阶段是指对象已经被 new 出来了，但是其依赖的 bean 还未注入，所以是个半成品。

* singletonFactories

  三级缓存，保存的是 beanName 和 ObjectFactory 的映射关系，ObjectFactory 能拿到最终 bean 实例的引用，作用讲起来比较复杂，后面新博文讲三级缓存和循环依赖时，再来聊聊它的作用比较合适。

从 `#getSingleton()` 方法逻辑可以知道，bean 实例一开始是在三级缓存的 ObjectFactory 中，调用方法时，会从三级缓存拿到 ObjectFactory，拿到 bean 实例对象，再放入二级缓存，同时将 singltonFactories 中 beanName 对应的缓存删除。

## FactoryBean 的处理

在此之前，需要先了解一下 FactoryBean，Spring 对 FactoryBean 的定位是用来构建一些比较复杂的 bean，即通过简单的依赖注入并不能得到想要的对象实例时，可以使用 FactoryBean 来自定义实例化的逻辑。

```java
public interface FactoryBean<T> {

	String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";
	
  /* 获取 bean 实例 */
	@Nullable
	T getObject() throws Exception;
  
  /* 获取 bean 的 Java 类型 */
	@Nullable
	Class<?> getObjectType();
	
  /* 是否为单例 bean */
	default boolean isSingleton() {
		return true;
	}
}
```

换言之，FactoryBean 本身并不是我们想要的 bean 实例，调用它的 getObject() 方法，才能拿到我们想要的 bean 实例。

然后，再看看 `#getObjectForBeanInstance()` 方法逻辑：

```java
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.
		// 1. 如果 name 是有 & 前缀的，说明要的就是 FactoryBean 的实例
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			// name有&前缀表示是 FactoryBean，但是获取到的 bean 并不是 FactoryBean 的实例，直接抛异常
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
			if (mbd != null) {
				mbd.isFactoryBean = true;
			}
			return beanInstance;
		}

		// Now we have the bean instance, which may be a normal bean or a FactoryBean.
		// If it's a FactoryBean, we use it to create a bean instance, unless the
		// caller actually wants a reference to the factory.
		if (!(beanInstance instanceof FactoryBean)) {
			// 2. name 不是以 & 开头，所以要获取的是 FactoryBean 最终生成的 Bean 对象
			return beanInstance;
		}

		Object object = null;
		if (mbd != null) {
			mbd.isFactoryBean = true;
		}
		else {
			// 3. 从缓存中获取 bean 对象
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
      // 4. 从 FactoryBean 中获取 bean 实例
			// Return bean instance from factory.
			// 此时 beanInstance 肯定是 FactoryBean
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```

上面代码逻辑主要分 4 步：

1. 对 name 参数有 `&` 前缀时的处理。

   如果 bean 实例是 FactoryBean 类型的，或者是 NullBean，直接返回，如果不是则 bean 实例和 name 实际想要的对象实例不符，直接抛异常。

   回看一下前面对 beanName 的处理逻辑中，有提到 FactoryBean 的名称是实际的 beanName 前加上一个 `&` 前缀，倒不是说容器中是这样存的，而是对 beanName 入参的规范是：如果要获取的是 FactoryBean 的实例，而不是最终构建出的 bean 实例，则 beanName 前需要加 `&` 前缀。

2. 如果 name 参数没有 `&` 前缀，且 bean 实例不是 FactoryBean，直接返回。

3. 这里是从 FactoryBean 的缓存中获取 bean 实例。如果前面两步都没能返回，那说明要拿的不是 FactoryBean，而是 FactoryBean 构建的 bean。

4. 本质上就是要调用 `FactoryBean#getObject()` 方法获取 bean 实例。

前面 3 步都比较简单，我们深入看看滴 4 步。

```java
	protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
		// 如果构建的是单例 bean，且 singletonObjects 中已经有这个 bean
		if (factory.isSingleton() && containsSingleton(beanName)) {
			synchronized (getSingletonMutex()) {
				// 尝试从缓存中获取 bean 实例
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
					// 缓存中没有，则调用 FactoryBean.getObject() 获取
					object = doGetObjectFromFactoryBean(factory, beanName);
					// Only post-process and store if not put there already during getObject() call above
					// (e.g. because of circular reference processing triggered by custom getBean calls)
					// 再次从缓存中获取
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					if (alreadyThere != null) {
						// 如果缓存中有，则返回缓存中的 bean
						object = alreadyThere;
					}
					else {
						// 是否需要进行后置处理
						if (shouldPostProcess) {
							if (isSingletonCurrentlyInCreation(beanName)) {
								// Temporarily return non-post-processed object, not storing it yet..
								return object;
							}
							beforeSingletonCreation(beanName);
							try {
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							catch (Throwable ex) {
								throw new BeanCreationException(beanName,
										"Post-processing of FactoryBean's singleton object failed", ex);
							}
							finally {
								afterSingletonCreation(beanName);
							}
						}
						if (containsSingleton(beanName)) {
							// 放入缓存
							this.factoryBeanObjectCache.put(beanName, object);
						}
					}
				}
				return object;
			}
		}
		else {
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			if (shouldPostProcess) {
				try {
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
				}
			}
			return object;
		}
	}

```

这里同样是尝试从缓存里面拿，缓存中没有则直接创建。

这里的核心是 `#doGetObjectFromFactoryBean()` 方法创建 bean，逻辑比较简单，其实就是调用 `FactoryBean#getObject()` 方法。

## prototype 类型并发创建检查

```java
	private final ThreadLocal<Object> prototypesCurrentlyInCreation =
			new NamedThreadLocal<>("Prototype beans currently in creation");

	protected boolean isPrototypeCurrentlyInCreation(String beanName) {
		Object curVal = this.prototypesCurrentlyInCreation.get();
		return (curVal != null &&
				(curVal.equals(beanName) || (curVal instanceof Set && ((Set<?>) curVal).contains(beanName))));
	}
```

prototypesCurrentlyInCreation 是一个记录正在创建中的 prototype 类型 beanName 的线程 ThreadLocal，如果当前要创建的 beanName 在 ThreadLocal 中，说明该 bean 已经在创建中了，创建时遇见这个情况，直接抛异常。

> 这里会抛异常是因为 prototype 类型是说每次获取这个 bean，都会创建一个新的 bean 实例，prototypesCurrentlyInCreation 是一个线程上下文，线程内，代码是串行执行，会出现上面代码中的情况应是，同一个线程中，创建 prototype 类型 bean 时，和其他 bean 有循环依赖，导致循环创建 bean，所以才会报错，总结一下，其实就是，**Spring 不能解决 prototype 类型的循环依赖。**

## 从父 BeanFactory 中查找 beanName

```java
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// 当前 beanFactory 中未找到，从父 BeanFactory 中查找
				// 获取 beanName
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
```

如果父 BeanFactory 是 AbstractBeanFactory 类型的，则调用其 doGetBean() 方法，如果不是，则调用其 getBean() 方法，只不过参数得按实际的情况传。

> 是的，doGetBean() 方法是定义在 AbstractBeanFactory 类中，而不是在 BeanFactory 接口中。

## 标识 beanName 对应的 bean 正在创建中

```java
if (!typeCheckOnly) {
   // 这里将 BeanDefinition 的 stale 属性设置为了 true
   markBeanAsCreated(beanName);
}
```

```java
	private final Set<String> alreadyCreated = Collections.newSetFromMap(new ConcurrentHashMap<>(256));

	protected void markBeanAsCreated(String beanName) {
		if (!this.alreadyCreated.contains(beanName)) {
			synchronized (this.mergedBeanDefinitions) {
				if (!this.alreadyCreated.contains(beanName)) {
					// Let the bean definition get re-merged now that we're actually creating
					// the bean... just in case some of its metadata changed in the meantime.
					clearMergedBeanDefinition(beanName);
					this.alreadyCreated.add(beanName);
				}
			}
		}
	}
```

alreadyCreated 记录的是至少已经创建了一次的 beanName，这里是将当前的 beanName 也加入到 alreadyCreated 中。

## 组装并检查 RootBeanDefinition

```java
	// 组装 RootBeanDefinition
	RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
	// 如果当前 BeanDefinition 是抽象的 bean，抛异常
	checkMergedBeanDefinition(mbd, beanName, args);
```

这里之所以要组装 RootBeanDefinition，是因为 Spring 中 BeanDefinition 有多种，而后续流程是固定流程，所以需要有确定的统一的 BeanDefinition，所以这里需要把其他类型的 BeanDefinition 转换成 RootBeanDefinition。

在学习 Java 基础时，我们就知道抽象类型是不能实例化的，所以得到 RootBeanDefinition 后还得检查要创建的 bean 是否是一个抽象类型，如果是，则直接抛异常。

## 依赖处理

```java
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						// 这里是为了避免出现 A 和 B 得初始化顺寻互相依赖，和循环依赖得注入不一样
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						// 注册依赖关系, 容器中维护了 dependentBeanMap, key 是 beanName，
						// value 是依赖 key 所表示的 bean 的所有 bean 的集合
						registerDependentBean(dep, beanName);
						try {
							// 递归调用，获取依赖的 bean，这里其实主要是为了在依赖的 bean 没有创建时进行创建
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

```

依赖处理这里，做了 3 件事：

1. 循环依赖检查

   主要是避免 A depends on B，B depends on A 这样的循环依赖的情况。

   Spring 维护了一个 `Map<String, Set<String>>` 类型的 `dependentBeanMap`，key 是 beanName，value 是依赖这个 bean 的 beanName 列表。 

2. 注册依赖关系。

3. 创建依赖的 bean，依然是调用 getBean() 方法进行创建，这就又回到了文章最开始的 doGetBean() 的地方了。

## 根据不同的 scope 创建 bean 实例

```java
				// 9. 创建 bean 实例
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

这段逻辑比较简单，只不过从上面的代码可以看出，

1. Spring 是原生的支持了 singleton、prototype 两种类型的 bean，其他类型都是通过扩展 Scope 接口来实现的，创建 bean 也是调用的 `#Scope.get()` 方法。

2. 不管哪种 Scope，创建 bean 实例的逻辑一致，都是调用 `#createBean()` 方法进行创建，只是创建出的 bean 的生命周期不同。

## 返回值类型转换

```java
	<T> T adaptBeanInstance(String name, Object bean, @Nullable Class<?> requiredType) {
		// Check if required type matches the type of the actual bean instance.
		// 类型检查，检查得到的 bean 类型和期望的类型是否一致
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				Object convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return (T) convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

> Spring 初始化阶段，requireType 是传的 null。

1. 如果 requireType 为 null，或者 bean 实例类型和 requireType 一致，则直接使用 `(T)bean` 的方式进行强转。
2. 如果类型不一致，则使用 TypeConverter 进行转换后返回。

# 总结

1. 带 `&` 前缀的 name，获取的是 FactoryBean 的实例对象，但是业务上一般不会这样做。
2. 如果当前 BeanFactory 中不存在，则从父 BeanFactory 中查找。
3. 所有 BeanDefinition 都会转换成 RootBeanDefinition 进行后续的创建流程。
4. depends-on 建立的依赖关系不允许出现循环依赖。
5. 所有的 Scope 的 bean，虽然生命周期不同，但都是调用 `#createBean()` 方法进行创建。
6. 如果获取的 bean 类型和预期的类型不一致，还可以通过实现 TypeConverter 接口来进行转换。
