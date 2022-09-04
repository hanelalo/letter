---
title: SPI 在 Java 中的应用
date: 2021-08-14 18:36:00
tags: 
    - Java
categories: Java
description: SPI 技术在 Java 的世界中应用广泛，本文对 SPI 的原理进行探究。
---

<img src='http://image.hanelalo.cn/image/202208141841577.png'/>

<!--more-->

# SPI 在 Java 中的应用

SPI，全称 `Service Provider Interface`，是 Java 提供的一套用于接入第三方扩展的技术。

## SPI 使用方式

```java
public interface EatService {
  void eat();
}


public class CatServiceImpl implements EatService{

  public CatServiceImpl(){
    System.out.println("初始化：猫");
  }

  @Override
  public void eat() {
    System.out.println("猫吃鱼");
  }
}


public class DogServiceImpl implements EatService{

  public DogServiceImpl(){
    System.out.println("初始化：狗");
  }

  @Override
  public void eat() {
    System.out.println("狗啃骨头");
  }
}


public class Main {

  public static void main(String[] args) {
    ServiceLoader<EatService> serviceLoader = ServiceLoader.load(EatService.class);
    Iterator<EatService> iterator = serviceLoader.iterator();
    while(iterator.hasNext()){
      iterator.next().eat();
    }
  }
}
```

上面定义了一个 EatService 接口，提供一个 eat() 的能力，EatService 接口又有 CatServiceImpl、DogServiceImpl 2 个实现类，每个实现类都各自实现了 eat() 方法，且显示的定义了无参构造。

最后，在 main() 方法中，通过 ServiceLoader 加载了 EatService 的实现类，并遍历实现类，执行 eat() 方法。

执行 main() 方法，没有任何输出，这是为何？唯一能想到的的结论是：Java并不知道 EatService 有哪些实现类。

虽然在开发者的视角，EatService 有 2 个实现类，那 Java 在运行时怎么知道有哪些实现类呢？Java 要求在 META-INF/services 目录下面进行接口实现类的配置，配置文件名为带包路径的接口名，内容为要加载的实现类。

```
// 文件: rg.hanelalo.spi.EatService
org.hanelalo.spi.CatServiceImpl
org.hanelalo.spi.DogServiceImpl
```

加上上述配置之后，再运行 main() 方法，得到如下结果：

```
初始化：猫
猫吃鱼
初始化：狗
狗啃骨头
```

可以看见，确实和配置所预想的一样，初始化了一个 CatServiceImpl 对象并调用了 eat() 方法，又初始化了一个 DogServiceImpl 对象并调用了 eat() 方法。

## SPI 原理

这里再次将前面遍历 EatService 实现类的代码搬过来：

```java
  public static void main(String[] args) {
    ServiceLoader<EatService> serviceLoader = ServiceLoader.load(EatService.class);
    Iterator<EatService> iterator = serviceLoader.iterator();
    while(iterator.hasNext()){
      iterator.next().eat();
    }
  }
```

结合实际场景，真正做开发的时候，更多是需要在众多实现类中找到需要的那一个，前面的例子也已经说明，每当遍历一个实现类，才会初始化一个对象出来，所以合理猜测初始化对象是在调用 iterator.eat() 时进行的，那么就需要从 ServiceLoader 提供的迭代器入手。

```java
    // ServiceLoader#iterator()
    public Iterator<S> iterator() {
        return new Iterator<S>() {

            Iterator<Map.Entry<String,S>> knownProviders
                = providers.entrySet().iterator();

            public boolean hasNext() {
                if (knownProviders.hasNext())
                    return true;
                return lookupIterator.hasNext();
            }

            public S next() {
                if (knownProviders.hasNext())
                    return knownProviders.next().getValue();
                return lookupIterator.next();
            }

            public void remove() {
                throw new UnsupportedOperationException();
            }

        };
    }
```

可以发现核心在于 knownProviders、lookupIterator 这 2 个变量，这里先聚焦于 lookupIterator，明白其作用后，也就知道了 knownProviders 的用处。

### lookupIterator#hasNext()

```java
        public boolean hasNext() {
            if (acc == null) {
                return hasNextService();
            } else {
                PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
                    public Boolean run() { return hasNextService(); }
                };
                return AccessController.doPrivileged(action, acc);
            }
        }
```

不管怎样，最终都是调用了 hasNextService() 方法进行了判断，再深入查看 hasNextService() 方法：

```java
        private boolean hasNextService() {
            if (nextName != null) {
                return true;
            }
            if (configs == null) {
                try {
                    String fullName = PREFIX + service.getName();
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
            while ((pending == null) || !pending.hasNext()) {
                if (!configs.hasMoreElements()) {
                    return false;
                }
                pending = parse(service, configs.nextElement());
            }
            nextName = pending.next();
            return true;
        }
```

1. 其中 PREFIX 常量值为 `META-INF/services/`，那么 fullName 就恰好是前面讲到的 SPI 配置文件的地址。

2. loader.getResource() 加载了配置文件。

3. 遍历配置文件中的实现类配置，调用 parse 方法进行解析。

4. nextName 赋值。

在 parse() 方法中仅仅只是将实现类名解析了出来。

总之，在hasNext()、hasNextService() 两个方法配合下，读取了 SPI 的配置文件，并获取到接口实现类的类名。

### lookupIterator#next()

```java
        public S next() {
            if (acc == null) {
                return nextService();
            } else {
                PrivilegedAction<S> action = new PrivilegedAction<S>() {
                    public S run() { return nextService(); }
                };
                return AccessController.doPrivileged(action, acc);
            }
        }
```

和 hasNext() 方法类似，关注 nextService() 方法即可。

```java
        private S nextService() {
            if (!hasNextService())
                throw new NoSuchElementException();
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            try {
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
                fail(service,
                     "Provider " + cn + " not found");
            }
            if (!service.isAssignableFrom(c)) {
                fail(service,
                     "Provider " + cn  + " not a subtype");
            }
            try {
                S p = service.cast(c.newInstance());
                providers.put(cn, p);
                return p;
            } catch (Throwable x) {
                fail(service,
                     "Provider " + cn + " could not be instantiated",
                     x);
            }
            throw new Error();          // This cannot happen
        }
```

1. 首先还是调用 hashNextService() 方法做了二次检查，避免开发者没吊用 hasNext() 而直接调用 next() 方法，前面讲过 hashNextService() 会给 nextName 进行赋值。

2. 获取当前遍历的实现类名称，并将 nextName 置为空，为下次遍历做准备。

3. 通过 Class.forName() 的方式加载实现类。

4. 通过 class.newInstance() 的方式实例化一个实现类的对象，并尝试转换为接口类型，这里其实是一次检查是否真的是目标接口的实现类的行为。

5. 以类名为 key，类对象为 value，放入 providers 中，并返回。

### knownProviders

前面讲了 lookupIterator 的方法，发现最后都会把实现类名和类对象放进 ServiceLoader 的 providers 中，再观察 ServiceLoader.iterator() 方法发现，knownProviders 其实就是从 providers 中生成的迭代器，所以后续再调用 ServiceLoader.iterator() 时，就不用再重新加载配置文件、初始化类了。

## 缺点

上一节有提到，更多时候是在众多实现类中找一个，而找的方式只能是遍历，最终会出现只使用其中一个实现类，但却初始化所有实现类的情况，这是一种资源的浪费。

像 Dubbo 针对 SPI 进行了更灵活的优化，能够按需加载，为每个实现类定义一个名称，使用时，可以直接通过名称先找到实现类名，再进行初始化，而不是每个实现了都初始化之后再判断是否是需要的实现类，具体的可翻阅 dubbo 项目 org.apache.dubbo.common.extension 包下面的源码。


























