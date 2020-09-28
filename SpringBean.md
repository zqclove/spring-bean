# Spring Bean

​	全文涉及bean的生命周期、重要参数和创建过程。

# bean的加载过程

主要以图文的形式描述bean的加载过程，以少量主要源码为辅。

## 概述

​	Spring作为IOC框架，实现了依赖注入和控制反转，由一个中心化的bean工厂来负责各个bean的实例化和依赖管理。各个bean可以不需要关注各自复杂的创建过程，达到了很好的解耦效果。

​	对Spring的工作流进行一个粗略的概括，主要分为两大环节：

- **解析：**读xml配置、扫描类文件，从配置或者注解中获取bean的定义信息，注册一些扩展功能。
- **加载：**通过解析完的定义信息获取bean实例。

![img](https://upload-images.jianshu.io/upload_images/460263-7da93f65daed431a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

我们假设所有的配置和扩展类都已经装载到了 ApplicationContext 中，然后具体的分析一下 Bean 的加载流程。

思考一个问题，抛开 Spring 框架的实现，假设我们手头上已经有一套完整的 Bean Definition Map，然后指定一个 beanName 要进行实例化，需要关心什么？即使我们没有 Spring 框架，也需要了解这两方面的知识：

- **作用域**。单例作用域或者原型作用域，单例的话需要全局实例化一次，原型每次创建都需要重新实例化。
- **依赖关系**。一个 Bean 如果有依赖，我们需要初始化依赖，然后进行关联。如果多个 Bean 之间存在着循环依赖，A 依赖 B，B 依赖 C，C 又依赖 A，需要解这种循环依赖问题。

Spring 进行了抽象和封装，使得作用域和依赖关系的配置对开发者透明，我们只需要知道当初在配置里已经明确指定了它的生命周期和依赖了谁，至于是怎么实现的，依赖如何注入，托付给了 Spring 工厂来管理。

Spring 只暴露了很简单的接口给调用者，比如 `getBean` ：

```java
ApplicationContext context = new ClassPathXmlApplicationContext("hello.xml");
HelloBean helloBean = (HelloBean) context.getBean("hello");
helloBean.sayHello();
```

那我们就从 `getBean` 方法作为入口，去理解 Spring 加载的流程是怎样的，以及内部对创建信息、作用域、依赖关系等等的处理细节。

## 总体流程

![img](https:////upload-images.jianshu.io/upload_images/460263-74d88a767a80843a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

​															 		**Bean 加载流程图**

上面是跟踪了 getBean 的调用链创建的流程图，为了能够很好地理解 Bean 加载流程，省略一些异常、日志和分支处理和一些特殊条件的判断。

从上面的流程图中，可以看到一个 Bean 加载会经历这么几个阶段（用绿色标记）：

- **获取 BeanName**：对传入的 name 进行解析，转化为可以从 Map 中获取到 BeanDefinition 的 bean name。
- **合并 Bean 定义**：对父类的定义进行合并和覆盖，如果父类还有父类，会进行递归合并，以获取完整的 Bean 定义信息。
- **实例化**：使用构造或者工厂方法创建 Bean 实例。
- **属性填充**：寻找并且注入依赖，依赖的 Bean 还会递归调用 `getBean` 方法获取。
- **初始化**：调用自定义的初始化方法。
- **获取最终的 Bean**：如果是 FactoryBean 需要调用 getObject 方法，如果需要类型转换调用 TypeConverter 进行转化。

整个流程最为复杂的是对循环依赖的解决方案，后续会进行重点分析。

## 细节分析

### 转化(获取)BeanName

​	在解析完配置后创建的Map，使用的是beanName作为key。

​	见DefaultListableBeanFactory：

```java
/** Map of bean definition objects, keyed by bean name */
private final Map<String, BeanDefinition> beanDefinitionMap = 
    new ConcurrentHashMap<String, BeanDefinition>(256);
```

​	**BeanFactory**.getBean中传入的name，有可能有这几种情况：

- **bean name**：可以直接获取到定义 BeanDefinition
- **alias name**：别名，需要转化。
- **factorybean nam** ：带&前缀，通过它获取BeanDefinition的时候需要去除&前缀。

为了能够获取到正确的 BeanDefinition，需要先对 name 做一个转换，得到 beanName。

![img](https://upload-images.jianshu.io/upload_images/460263-ffc69491d898b4e4.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

​																**name转beanName**

​	见 **AbstractBeanFactory.doGetBean**：

```java
protected <T> T doGetBean ... {
    ...
    
    // 转化工作 
    final String beanName = transformedBeanName(name);
    ...
}
```

​	如果是 **alias name**，在解析阶段，alias name 和 bean name 的映射关系（aliasMap）被注册到 SimpleAliasRegistry 中。可从该注册器中取到 beanName。

​	见 **SimpleAliasRegistry.canonicalName**：

​	该方法有循环过程，不断循环获取别名（key）的映射值（value），直至映射值不存在aliasMap的别名中。

```java
public String canonicalName(String name) {
    String canonicalName = name;
    ...
    resolvedName = this.aliasMap.get(canonicalName);
    ... 
}
```

​	如果是 **factorybean name**，表示这是个工厂 bean，有携带前缀修饰符 `&` 的，直接把前缀去掉。

​	见 **BeanFactoryUtils.transformedBeanName** ：

```java
public static String transformedBeanName(String name) {
    Assert.notNull(name, "'name' must not be null");
    String beanName = name;
    while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
        beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
    }
    return beanName;
}
```

### 合并 RootBeanDefinition

​	我们从配置文件读取到的 BeanDefinition 是 **GenericBeanDefinition**。它的记录了一些当前类声明的属性或构造参数，但是对于父类只用了一个 `parentName` 来记录。

```java
public class GenericBeanDefinition extends AbstractBeanDefinition {
    ...
    private String parentName;
    ...
}
```

接下来会发现一个问题，在后续实例化 Bean 的时候，使用的 BeanDefinition 是 **RootBeanDefinition** 类型而非 **GenericBeanDefinition**。这是为什么？

答案很明显，GenericBeanDefinition 在有继承关系的情况下，定义的信息不足：

- 如果不存在继承关系，GenericBeanDefinition 存储的信息是完整的，可以直接转化为 RootBeanDefinition。
- 如果存在继承关系，GenericBeanDefinition 存储的是 **增量信息** 而不是 **全量信息**。

**为了能够正确初始化对象，需要完整的信息才行**。需要递归 **合并父类的定义**：

![img](https://upload-images.jianshu.io/upload_images/460263-802f258b1dba771d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

见 **AbstractBeanFactory.doGetBean** ：

```java
protected <T> T doGetBean ... {
    ...
    
    // 合并父类定义
    final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
        
    ...
        
    // 使用合并后的定义进行实例化
    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
        
    ...
}
```

在判断 `parentName` 存在的情况下，说明存在父类定义，启动合并。如果父类还有父类怎么办？递归调用，继续合并。

见**AbstractBeanFactory.getMergedBeanDefinition** 方法：

```java
    protected RootBeanDefinition getMergedBeanDefinition(
            String beanName, BeanDefinition bd, BeanDefinition containingBd)
            throws BeanDefinitionStoreException {

        ...
        
        String parentBeanName = transformedBeanName(bd.getParentName());

        ...
        
        // 递归调用，继续合并父类定义
        pbd = getMergedBeanDefinition(parentBeanName);
        
        ...

        // 使用合并后的完整定义，创建 RootBeanDefinition
        mbd = new RootBeanDefinition(pbd);
        
        // 使用当前定义，对 RootBeanDefinition 进行覆盖
        mbd.overrideFrom(bd);

        ...
        return mbd;
    
    }
```

​	每次合并完父类定义后，都会调用 **RootBeanDefinition.overrideFrom** 对父类的定义进行覆盖，获取到当前类能够正确实例化的 **全量信息**。

### 处理循环依赖

什么是循环依赖？

举个例子，这里有三个类 A、B、C，然后 A 关联 B，B 关联 C，C 又关联 A，这就形成了一个循环依赖。如果是方法调用是不算循环依赖的，循环依赖必须要持有引用。

![img](https:////upload-images.jianshu.io/upload_images/460263-e8616f453f6715b5.png?imageMogr2/auto-orient/strip|imageView2/2/w/478)

循环依赖

循环依赖根据注入的时机分成两种类型：

- **构造器循环依赖**。依赖的对象是通过构造器传入的，发生在实例化 Bean 的时候。
- **设值循环依赖**。依赖的对象是通过 setter 方法传入的，对象已经实例化，发生属性填充和依赖注入的时候。

**如果是构造器循环依赖，本质上是无法解决的**。比如我们准调用 A 的构造器，发现依赖 B，于是去调用 B 的构造器进行实例化，发现又依赖 C，于是调用 C 的构造器去初始化，结果依赖 A，整个形成一个死结，导致 A 无法创建。

**如果是设值循环依赖，Spring 框架只支持单例下的设值循环依赖**。Spring 通过对还在创建过程中的单例，缓存并提前暴露该单例，使得其他实例可以引用该依赖。

#### 原型模式循环依赖

**Spring 不支持原型模式的任何循环依赖**。检测到循环依赖会直接抛出 BeanCurrentlyInCreationException 异常。

下面讲述如何检测出原型模式循环依赖：

使用了一个 ThreadLocal 变量 **prototypesCurrentlyInCreation** 来记录当前线程正在创建中的 Bean 对象，见 **AbtractBeanFactory#prototypesCurrentlyInCreation**：

```java
/** Names of beans that are currently in creation */
private final ThreadLocal<Object> prototypesCurrentlyInCreation =
            new NamedThreadLocal<Object>("Prototype beans currently in creation");
```

在 Bean 创建前进行记录，在 Bean 创建后删除记录。见 **AbstractBeanFactory.doGetBean**：

```java
...
if (mbd.isPrototype()) {
    // It's a prototype -> create a new instance.
    Object prototypeInstance = null;
    try {
    
        // 添加记录
        beforePrototypeCreation(beanName);
        prototypeInstance = createBean(beanName, mbd, args);
    }
    finally {
        // 删除记录
        afterPrototypeCreation(beanName);
    }
    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
}
...     
```

见 **AbtractBeanFactory.beforePrototypeCreation** 的记录操作：

```java
    protected void beforePrototypeCreation(String beanName) {
        Object curVal = this.prototypesCurrentlyInCreation.get();
        if (curVal == null) {
            this.prototypesCurrentlyInCreation.set(beanName);
        }
        else if (curVal instanceof String) {
            Set<String> beanNameSet = new HashSet<String>(2);
            beanNameSet.add((String) curVal);
            beanNameSet.add(beanName);
            this.prototypesCurrentlyInCreation.set(beanNameSet);
        }
        else {
            Set<String> beanNameSet = (Set<String>) curVal;
            beanNameSet.add(beanName);
        }
    }
```

见 **AbtractBeanFactory.afterPrototypeCreation** 的删除操作：

```java
    protected void afterPrototypeCreation(String beanName) {
        Object curVal = this.prototypesCurrentlyInCreation.get();
        if (curVal instanceof String) {
            this.prototypesCurrentlyInCreation.remove();
        }
        else if (curVal instanceof Set) {
            Set<String> beanNameSet = (Set<String>) curVal;
            beanNameSet.remove(beanName);
            if (beanNameSet.isEmpty()) {
                this.prototypesCurrentlyInCreation.remove();
            }
        }
    }
```

为了节省内存空间，在单个元素时 **prototypesCurrentlyInCreation** 只记录 String 对象，在多个依赖元素后改用 Set 集合。这里是 Spring 使用的一个节约内存的小技巧。

了解了记录的写入和删除过程好了，再来看看读取以及判断循环的方式。这里要分两种情况讨论。

- 构造函数循环依赖。
- 设置循环依赖。

这两个地方的实现略有不同。

如果是构造函数依赖的，比如 A 的构造函数依赖了 B，会有这样的情况。实例化 A 的阶段中，匹配到要使用的构造函数，发现构造函数有参数 B，会使用 **BeanDefinitionValueResolver** 来检索 B 的实例。见 **BeanDefinitionValueResolver.resolveReference**：

```java
private Object resolveReference(Object argName, RuntimeBeanReference ref) {

    ...
    Object bean = this.beanFactory.getBean(refName);
    ...
}
```

我们发现这里继续调用 **beanFactory.getBean** 去加载 B。

如果是设值循环依赖的的，比如我们这里不提供构造函数，并且使用了 @Autowire 的方式注解依赖（还有其他方式不举例了）：

```java
public class A {
    @Autowired
    private B b;
    ...
}
```

加载过程中，找到无参数构造函数，不需要检索构造参数的引用，实例化成功。接着执行下去，进入到属性填充阶段  **AbtractBeanFactory.populateBean** ，在这里会进行 B 的依赖注入。

为了能够获取到 B 的实例化后的引用，最终会通过检索类 **DependencyDescriptor** 中去把依赖读取出来，见 **DependencyDescriptor.resolveCandidate** ：

```java
public Object resolveCandidate(String beanName, Class<?> requiredType, BeanFactory beanFactory)
            throws BeansException {
    return beanFactory.getBean(beanName, requiredType);
}
```

发现 **beanFactory.getBean** 方法又被调用到了。

**在这里，两种循环依赖达成了同一**。无论是构造函数的循环依赖还是设置循环依赖，在需要注入依赖的对象时，会继续调用 **beanFactory.getBean** 去加载对象，形成一个递归操作。

而每次调用 **beanFactory.getBean** 进行实例化前后，都使用了 **prototypesCurrentlyInCreation** 这个变量做记录。按照这里的思路走，整体效果等同于 **建立依赖对象的构造链**。

**prototypesCurrentlyInCreation** 中的值的变化如下：

![img](https:////upload-images.jianshu.io/upload_images/460263-dcc386d144f5dad0.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

原型模式的循环依赖

调用判定的地方在 **AbstractBeanFactory.doGetBean** 中，所有对象的实例化均会从这里启动。

```java
// Fail if we're already creating this bean instance:
// We're assumably within a circular reference.
if (isPrototypeCurrentlyInCreation(beanName)) {
    throw new BeanCurrentlyInCreationException(beanName);
}
```

判定的实现方法为 **AbstractBeanFactory.isPrototypeCurrentlyInCreation** ：

```java
protected boolean isPrototypeCurrentlyInCreation(String beanName) {
    Object curVal = this.prototypesCurrentlyInCreation.get();
    return (curVal != null &&
        (curVal.equals(beanName) || (curVal instanceof Set && ((Set<?>) curVal).contains(beanName))));
}
```

所以在原型模式下，构造函数循环依赖和设值循环依赖，本质上使用同一种方式检测出来。Spring 无法解决，直接抛出 BeanCurrentlyInCreationException 异常。

#### 单例模式循环依赖

**Spring不支持单例模式的构造方法循环依赖，但支持设置循环依赖。**检测到构造循环依赖也会抛出BeanCurrentlyInCreationException异常。

和原型模式相似，单例模式也用了一个数据结构来记录正在创建中的 beanName。见 **DefaultSingletonBeanRegistry**:

```java
/** Names of beans that are currently in creation */
private final Set<String> singletonsCurrentlyInCreation =
            Collections.newSetFromMap(new ConcurrentHashMap<String, Boolean>(16));
```

会在创建前进行记录，创建化后删除记录。

见 **DefaultSingletonBeanRegistry.getSingleton**

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        ...
        
        // 记录正在加载中的 beanName
        beforeSingletonCreation(beanName);
        ...
        // 通过 singletonFactory 创建 bean
        singletonObject = singletonFactory.getObject();
        ...
        // 删除正在加载中的 beanName
        afterSingletonCreation(beanName);
        
}
```

记录和判定的方式见  **DefaultSingletonBeanRegistry.beforeSingletonCreation** ：

```java
    protected void beforeSingletonCreation(String beanName) {
        if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
    }
```

这里会尝试往 **singletonsCurrentlyInCreation** 记录当前实例化的 bean。我们知道 singletonsCurrentlyInCreation 的数据结构是 Set，是不允许重复元素的，**所以一旦前面记录了，这里的 add 操作将会返回失败**。

比如加载 A 的单例，和原型模式类似，单例模式也会调用匹配到要使用的构造函数，发现构造函数有参数 B，然后使用 **BeanDefinitionValueResolver** 来检索 B 的实例，根据上面的分析，继续调用 **beanFactory.getBean** 方法。

所以拿 A，B，C 的例子来举例 **singletonsCurrentlyInCreation** 的变化，这里可以看到和原型模式的循环依赖判断方式的算法是一样：

![img](https:////upload-images.jianshu.io/upload_images/460263-f0da20b9880c6fd9.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

单例模式的构造循环依赖

- 加载 A。记录 singletonsCurrentlyInCreation = **[a]**，构造依赖 B，开始加载 B。
- 加载 B，记录 singletonsCurrentlyInCreation = **[a, b]**，构造依赖 C，开始加载 C。
- 加载 C，记录 singletonsCurrentlyInCreation = **[a, b, c]**，构造依赖 A，又开始加载 A。
- 加载 A，执行到 **DefaultSingletonBeanRegistry.beforeSingletonCreation** ，singletonsCurrentlyInCreation 中 a 已经存在了，检测到构造循环依赖，直接抛出异常结束操作。

# AbstractApplicationContext

## refresh()源码解析

```java
	public void refresh() throws BeansException, IllegalStateException {
        // 该锁对象主要用于spring环境刷新和销毁，确保刷新和销毁不会同时进行；
		synchronized (this.startupShutdownMonitor) {
			// 准备工作，例如记录事件，设置标志，检查环境变量等；
			prepareRefresh();

			// 创建beanFactory，这个对象作为applicationContext的成员变量，可以被		  					// applicationContext拿来用,并且解析资源（例如xml文件），
            // 取得bean的定义，放在beanFactory中
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 对beanFactory做一些设置，例如类加载器、spel解析器、指定bean的某些类型的成员变量对应某些			对象等
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " 				+ "cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```



# 参考资料

[图文并茂，揭秘 Spring 的 Bean 的加载过程]: https://www.jianshu.com/p/9ea61d204559

