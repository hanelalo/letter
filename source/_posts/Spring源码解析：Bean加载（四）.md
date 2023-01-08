---
title: Spring源码解析：Bean加载（四）
date: 2023-01-01 21:59:41
tags: Spring源码解析
categories: Spring
description: bean 实例化后，还需要对依赖的 bean 进行填充。
---

在前面几篇博文中，已经详细介绍了 bean 的实例化。在实例化完成后，bean 并没有真正的完全可用，此时还需要进行一些必要的依赖属性填充。

换言之，只是通过构造函数将 bean 实例构建出来了，但是里面一些并未通过构造函数进行初始化的属性，还需要单独再填充一次。

填充属性的在[Spring源码解析：Bean加载（二）](/posts/54dce9e15d4c#doCreateBean-%E6%96%B9%E6%B3%95)中就已经提到过，位于 doCreateBean() 方法中。

```java
	protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		if (bw == null) {
			// 如果 BeanDefinition 中判定是有属性值的，但是 BeanWrapper 为 null，直接抛异常
			if (mbd.hasPropertyValues()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// 不需要进行属性填充，直接return
				return;
			}
		}

		// InstantiationAwareBeanPostProcessors 用于提供在 bean 的属性设置前修改 bean，比如用来支持自定义的属性注入。
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
				if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
					// 如果后处理器一旦有一个返回 false, 就不填充属性值
					return;
				}
			}
		}

		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

		// 1. 获取自动注入类型
		int resolvedAutowireMode = mbd.getResolvedAutowireMode();
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
			// Add property values based on autowire by name if applicable.
			if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
				// 按 beanName 注入
				autowireByName(beanName, mbd, bw, newPvs);
			}
			// Add property values based on autowire by type if applicable.
			if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
				// 按照 bean 的类型注入
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
			for (InstantiationAwareBeanPostProcessor bp : getBeanPostProcessorCache().instantiationAware) {
				PropertyValues pvsToUse = bp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
				if (pvsToUse == null) {
					if (filteredPds == null) {
						filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
					}
					// 后处理
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
			// 依赖检查
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}

		if (pvs != null) {
			// 2. 将属性应用到 bean 中, 前面只是找到了每一个依赖的 bean 而已
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
	}
```

这里核心逻辑主要有 2 个点：

1. 获取注入类型，根据自动注入类型，解析注入的 bean。
2. 将上一步对应的解析值，设置到 bean 中，对应 `applyPropertyValues()` 方法。

## 自动注入

```java
		// 获取自动注入类型
		int resolvedAutowireMode = mbd.getResolvedAutowireMode();
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
			// Add property values based on autowire by name if applicable.
			if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
				// 按 beanName 注入
				autowireByName(beanName, mbd, bw, newPvs);
			}
			// Add property values based on autowire by type if applicable.
			if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
				// 按照 bean 的类型注入
				autowireByType(beanName, mbd, bw, newPvs);
			}
			pvs = newPvs;
		}
```

注入类型从 `AbstractBeanDefinition#getResolvedAutowireMode()` 中得到。

```java
	public int getResolvedAutowireMode() {
		if (this.autowireMode == AUTOWIRE_AUTODETECT) {
			// 如果自动注入类型是 autodetect, 就根据构造函数判断最终的注入类型
			Constructor<?>[] constructors = getBeanClass().getConstructors();
			for (Constructor<?> constructor : constructors) {
				if (constructor.getParameterCount() == 0) {
					// parameterCount==0说明有无参构造，直接返回通过 type注入
					return AUTOWIRE_BY_TYPE;
				}
			}
			// 通过构造函数注入
			return AUTOWIRE_CONSTRUCTOR;
		}
		else {
			// 就以当前值返回
			return this.autowireMode;
		}
	}
```

然后根据注入类型的不同，又调用了 `#autowireByName`、`#autowireByType` 方法。

### autowireByName

```java
	protected void autowireByName(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
//		寻找非简单类型的需要注入的属性，像基础类型和Date就数据简单类型
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
		for (String propertyName : propertyNames) {
			// 因为是按照名称注入，所以这里直接将属性名称作为beanName进行搜索
			if (containsBean(propertyName)) {
				// 递归初始化相关 bean
				Object bean = getBean(propertyName);
				pvs.add(propertyName, bean);
				// 注册 bean 依赖关系
				registerDependentBean(propertyName, beanName);
				if (logger.isTraceEnabled()) {
					logger.trace("Added autowiring by name from bean name '" + beanName +
							"' via property '" + propertyName + "' to bean named '" + propertyName + "'");
				}
			}
			else {
				if (logger.isTraceEnabled()) {
					logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
							"' by name: no matching bean found");
				}
			}
		}
	}
```

1. 这里首先通过`#unsatisfiedNonSimpleProperties()` 方法剔除了一些简单类型的字段，其实就是一些 Java 的基础类型和 Date 类型；
2. 然后就判断属性名对应的 bean 是否存在，存在就通过 `#getBean()` 方法获取 bean 实例，这里如果依赖的 bean 还没创建，则会开始创建依赖的 bean；
3. 注册 bean 的依赖关系，其实就是容器维护了一个 map 用于记录各个 bean 之间的依赖关系。

### autowireByType

```java
	protected void autowireByType(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}

		Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
		// 1. 获取非简单属性
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
		for (String propertyName : propertyNames) {
			try {
				// 2. 寻找需要注入的属性
				PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
				// Don't try autowiring by type for type Object: never makes sense,
				// even if it technically is a unsatisfied, non-simple property.
				if (Object.class != pd.getPropertyType()) {
					// 获取写入属性值的方法
					MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
					// Do not allow eager init for type matching in case of a prioritized post-processor.
					boolean eager = !(bw.getWrappedInstance() instanceof PriorityOrdered);
					DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
					// 3. 解析指定 beanName 的属性匹配的值并存放在 autowiredBeanNames 中
					// 如果注入的是一个 list，将会找到所有匹配的类型注入
					Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
					if (autowiredArgument != null) {
						pvs.add(propertyName, autowiredArgument);
					}
					for (String autowiredBeanName : autowiredBeanNames) {
						// 4. 注册依赖关系
						registerDependentBean(autowiredBeanName, beanName);
						if (logger.isTraceEnabled()) {
							logger.trace("Autowiring by type from bean name '" + beanName + "' via property '" +
									propertyName + "' to bean named '" + autowiredBeanName + "'");
						}
					}
					autowiredBeanNames.clear();
				}
			}
			catch (BeansException ex) {
				throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
			}
		}
	}
```

1. 通过 `#unsatisfiedNonSimpleProperties()` 方法获取非简单属性，然后开始遍历得到的每一个属性。
2. 根据属性名得到相应的 PropertyDescriptor。PropertyDescriptor 是 JDK 中定义的类，持有一个属性的 read、write 方法引用。
3. `#resolveDependency()` 方法解析依赖。
4. 注册依赖关系。

下面再深入看看 `#resolveDependency()` 方法。

#### resolveDependency

```java
	public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {
		// 初始化参数名称发现
		descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
		// 对注入 Optional 类型处理
		if (Optional.class == descriptor.getDependencyType()) {
			return createOptionalDependency(descriptor, requestingBeanName);
		}
		// ObjectFactory 注入类型处理
		else if (ObjectFactory.class == descriptor.getDependencyType() ||
				ObjectProvider.class == descriptor.getDependencyType()) {
			return new DependencyObjectProvider(descriptor, requestingBeanName);
		}
		// javaxInjectProviderClass 类型处理
		else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
			return new Jsr330Factory().createDependencyProvider(descriptor, requestingBeanName);
		}
		else {
			// 普通的 bean 注入处理，构建惰性加载的代理
			Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
					descriptor, requestingBeanName);
			if (result == null) {
				// 通用处理逻辑
				result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
			}
			return result;
		}
	}
```

这里是对普通 bean 的注入和一些特殊类型的 bean 的注入分开进行处理，但主要关注 `#doResolveDependency()` 方法对普通注入的解析。

同时需要注意的是，对于普通注入的处理，如果注入时选择了延迟加载，则需要在这里以代理的方式对延迟加载进行支持，如果不是延迟加载，则会调用 `#doResolveDependency()` 方法。

```java
	public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {
		// 设置注入点
		InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
		try {
			// 针对给定工厂解决此依赖关系的快捷方式
			Object shortcut = descriptor.resolveShortcut(this);
			if (shortcut != null) {
				return shortcut;
			}
			// 依赖 bean 类型
			Class<?> type = descriptor.getDependencyType();
			// 处理 @Value 注解
			Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
			if (value != null) {
				if (value instanceof String) {
					String strVal = resolveEmbeddedValue((String) value);
					BeanDefinition bd = (beanName != null && containsBean(beanName) ?
							getMergedBeanDefinition(beanName) : null);
					value = evaluateBeanDefinitionString(strVal, bd);
				}
				TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
				try {
					return converter.convertIfNecessary(value, type, descriptor.getTypeDescriptor());
				}
				catch (UnsupportedOperationException ex) {
					// A custom TypeConverter which does not support TypeDescriptor resolution...
					return (descriptor.getField() != null ?
							converter.convertIfNecessary(value, type, descriptor.getField()) :
							converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
				}
			}
			// 处理多种类型的 bean，这里主要处理 Stream、Array、Collection、Map 这几种类型的依赖注入
			Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
			if (multipleBeans != null) {
				return multipleBeans;
			}
			// 到这里基本可以确定这里要注入的 bean 只有一个
			Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
			if (matchingBeans.isEmpty()) {
				// 最终如果没有找到依赖的 bean, 并且 @Autowired 上的 required 是 true, 就抛异常
				if (isRequired(descriptor)) {
					raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
				}
				return null;
			}

			String autowiredBeanName;
			Object instanceCandidate;
			// 如果匹配到了多个 bean
			if (matchingBeans.size() > 1) {
				autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
				// 有多个 bean，但是通过 @Primary、@Priority、fieldName 都没有找到
				if (autowiredBeanName == null) {
					// 但是 @Autowired 的 required 属性为 true，且注入的类型不是多个 bean 一起注入，
					// 抛异常，找到了多个 bean, 但是一个都没匹配上
					if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
						return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
					}
					// 如果不是多个 bean 一起注入，并且 @Autowired 的 required 属性为 false，就返回 null
					else {
						// In case of an optional Collection/Map, silently ignore a non-unique case:
						// possibly it was meant to be an empty collection of multiple regular beans
						// (before 4.3 in particular when we didn't even look for collection beans).
						return null;
					}
				}
				instanceCandidate = matchingBeans.get(autowiredBeanName);
			}
			// 恰好只匹配上了一个 bean
			else {
				// We have exactly one match.
				Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
				autowiredBeanName = entry.getKey();
				instanceCandidate = entry.getValue();
			}

			if (autowiredBeanNames != null) {
				autowiredBeanNames.add(autowiredBeanName);
			}
			if (instanceCandidate instanceof Class) {
				// 如果拿到的是依赖的类型的 Class，从 beanFactory 中获取实例
				instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
			}
			Object result = instanceCandidate;
			if (result instanceof NullBean) {
				// 处理每找到 bean 的情况
				if (isRequired(descriptor)) {
					raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
				}
				result = null;
			}
			// 判断拿到的 bean 实例和实际依赖的类型是否匹配，不匹配则抛异常
			if (!ClassUtils.isAssignableValue(type, result)) {
				throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
			}
			return result;
		}
		finally {
			ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
		}
	}
```

上面的注释已经很详细，就不再赘述了。

到这里，注入 bean 的解析就基本完结了，接下来看看 `#applyPropertyValues()` 方法对属性值的设置。

## applyPropertyValues() 方法

```java
	protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
		if (pvs.isEmpty()) {
			return;
		}

		if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
			((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
		}

		MutablePropertyValues mpvs = null;
		List<PropertyValue> original;

		if (pvs instanceof MutablePropertyValues) {
			mpvs = (MutablePropertyValues) pvs;
			// 如果已经转换为对应类型，直接设置进 beanWrapper 中即可
			if (mpvs.isConverted()) {
				// Shortcut: use the pre-converted values as-is.
				try {
					// 真正将属性值设置到 bean 实例中
					bw.setPropertyValues(mpvs);
					return;
				}
				catch (BeansException ex) {
					throw new BeanCreationException(
							mbd.getResourceDescription(), beanName, "Error setting property values", ex);
				}
			}
			original = mpvs.getPropertyValueList();
		}
		else {
			// 如果不是 MutablePropertyValues 封装类型，直接使用原始属性值列表
			original = Arrays.asList(pvs.getPropertyValues());
		}

		// 类型转换器
		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}
		BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

		// Create a deep copy, resolving any references for values.
		List<PropertyValue> deepCopy = new ArrayList<>(original.size());
		boolean resolveNecessary = false;
		// 转换成对应类型
		for (PropertyValue pv : original) {
			if (pv.isConverted()) {
				// 如果已经做了类型转换，则直接添加到 deepCopy
				deepCopy.add(pv);
			}
			else {
				// 属性名称
				String propertyName = pv.getName();
				// 原始值
				Object originalValue = pv.getValue();
				// AutowiredPropertyMarker.INSTANCE 表示这个属性是需要注入的属性
				if (originalValue == AutowiredPropertyMarker.INSTANCE) {
					Method writeMethod = bw.getPropertyDescriptor(propertyName).getWriteMethod();
					if (writeMethod == null) {
						throw new IllegalArgumentException("Autowire marker for property without write method: " + pv);
					}
					// 将 originalValue 重新赋值为 DependencyDescriptor
					originalValue = new DependencyDescriptor(new MethodParameter(writeMethod, 0), true);
				}
				// 得到 bean 实例
				Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
				// convertedValue 记录转换后的值
				Object convertedValue = resolvedValue;

				boolean convertible = bw.isWritableProperty(propertyName) &&
						!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
				// 使用自定义类型转换器进行转换
				if (convertible) {
					convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
				}
				// Possibly store converted value in merged bean definition,
				// in order to avoid re-conversion for every created bean instance.
				if (resolvedValue == originalValue) {
					// 缓存转换后的值，避免每次注入都要重新做一次转换
					if (convertible) {
						pv.setConvertedValue(convertedValue);
					}
					deepCopy.add(pv);
				}
				else if (convertible // 可转换的
						&& originalValue instanceof TypedStringValue // 值类型是字符串
						&& !((TypedStringValue) originalValue).isDynamic() // 不是动态生成的字符串
						&& !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) { // 不是Collection或者数组
					pv.setConvertedValue(convertedValue);
					deepCopy.add(pv);
				}
				else {
					resolveNecessary = true;
					deepCopy.add(new PropertyValue(pv, convertedValue));
				}
			}
		}
		if (mpvs != null && !resolveNecessary) {
			mpvs.setConverted();
		}

		// Set our (possibly massaged) deep copy.
		try {
			// 设置属性值
			bw.setPropertyValues(new MutablePropertyValues(deepCopy));
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Error setting property values", ex);
		}
	}
```

总而言之，这里就是对不需要转换的值，直接注入到 bean 中，对需要转换的则先转换过后，注入到 bean 中。

## 总结

在填充属性时：

1. 解析属性值，会初始化依赖的 bean。
2. 如果属性值需要转换，则会转换后进行注入，如果有自定义的类型转换器就使用自定义，否则直接使用 BeanWrapper 作为类型转换器。
