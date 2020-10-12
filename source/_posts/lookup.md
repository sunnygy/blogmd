---
title: Dependence looking up
date: 2020-09-23 22:17:06
tags: Dependence looking up
categories: spring
---



## 1.The Bean will be built up by  inner AbstractApplicationContext when looking up

<!--more-->

|                          Bean Name                           |                 Bean Instance                  |                      Usage case                       |
| :----------------------------------------------------------: | :--------------------------------------------: | :---------------------------------------------------: |
|                         environment                          |                Environment 对象                |                外部化配置以及 Profiles                |
|                       systemProperties                       |           java.util.Properties 对象            |                     Java 系统属性                     |
|                      systemEnvironment                       |               java.util.Map对象                |                   操作系统环境变量                    |
|                        messageSource                         |               MessageSource 对象               |                      国际化文案                       |
|                      lifecycleProcessor                      |            LifecycleProcessor 对象             |                 Lifecycle Bean 处理器                 |
|                 applicationEventMulticaster                  |        ApplicationEventMulticaster 对象        |                   Spring 事件广播器                   |
| org.springframework.context.annotation. <br/>internalConfigurationAnnotationProcessor |      ConfigurationClassPostProcessor 对象      |                  处理 Spring 配置类                   |
| org.springframework.context.annotation<br/>        InternalAutowiredAnnotationProcessor | AutowiredAnnotationBeanPostP<br/>rocessor 对象 |           处理 @Autowired 以及 @Value 注解            |
| org.springframework.context.annotation.<br/>internalCommonAnnotationProcessor |  CommonAnnotationBeanPostPr<br/>ocessor 对象   |  （条件激活）处理 JSR-250 注解，如 @PostConstruct 等  |
| org.springframework.context.event.<br/>internalEventListenerProcessor |       EventListenerMethodProcessor 对象        |     处理标注 @EventListener 的Spring 事件监听方法     |
| org.springframework.context.event.<br/>internalEventListenerFactory |        DefaultEventListenerFactory 对象        | @EventListener 事件监听方法适配为 ApplicationListener |
| org.springframework.context.annotation.<br/>internalPersistenceAnnotationProcessor<br/> |  PersistenceAnnotationBeanPostProcessor 对象   |             （条件激活）处理 JPA 注解场景             |

##  2. The difference between ObjectFactory and  BeanFactory

> Both of ObjectFactory and  BeanFactory  provide the ability of looking up beans。不过 ObjectFactory 仅关注一个或一种类型的 Bean 依赖查找，并且
> 自身不具备依赖查找的能力，能力则由 BeanFactory 输出。BeanFactory 则提供了单一类型、集合类型以及层次性等多种依赖查找方式。

## 3.The Source Code that how create bean In Looking  up process

- [doGetBean](#doGetBean)
- - [第1步：尝试从缓存中获取Bean](#getCachedBean)
  - [第3步：Prototype类型Bean的循环依赖检查](#checkBeanDep)
  - [第5步：Depends-on依赖检查](#dep-on)
  - [第6步：创建Singleton Bean](#singletion)
  - [第7步：创建Prototype Bean](#prototype)
  - [第9步：getObjectForBeanInstance判断是否为FactoryBean](#factoryBean) 

  ### BeanFactory#getBean as the entry，demonstrate the loading process of Bean：       

```java
public Object getBean(String name) throws BeansException {
   return doGetBean(name, null, null, false);
}
```

## <span id="doGetBean">doGetBean<span>

spring源码中真正的执行逻辑方法一般都是以do开头，看到doGetBean，那么真正的创建实例逻辑应是在此方法中:

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
      @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

   final String beanName = transformedBeanName(name);
   Object bean;

   // Eagerly check singleton cache for manually registered singletons.
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
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
   }

   else {
      // Fail if we're already creating this bean instance:
      // We're assumably within a circular reference.
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

      if (!typeCheckOnly) {
         markBeanAsCreated(beanName);
      }

      try {
         final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
         checkMergedBeanDefinition(mbd, beanName, args);

         // Guarantee initialization of beans that the current bean depends on.
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

         // Create bean instance.
         if (mbd.isSingleton()) {
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
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
         }

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
            bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
         }

         else {
            String scopeName = mbd.getScope();
            final Scope scope = this.scopes.get(scopeName);
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
               bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
            }
            catch (IllegalStateException ex) {
               throw new BeanCreationException(beanName,
                     "Scope '" + scopeName + "' is not active for the current thread; consider " +
                     "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                     ex);
            }
         }
      }
      catch (BeansException ex) {
         cleanupAfterBeanCreationFailure(beanName);
         throw ex;
      }
   }

   // Check if required type matches the type of the actual bean instance.
   if (requiredType != null && !requiredType.isInstance(bean)) {
      try {
         T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
         if (convertedBean == null) {
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
         }
         return convertedBean;
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

这个方法很长，大概功能包括了：

1. 首先检查本地缓存，如果在缓存中已经存在此Bean，则返回缓存中的实例；
2. 如果缓存中不存在，进行一些创建实例前的准备工作：
3. 如果Bean的scope是Prototype，则检查循环依赖；
4. 如果当前BeanFactory中没有此BeanDefinition，且存在父BeanFactory，则从父BeanFactory中获取Bean；
5. depends-on循环依赖检查，注意此处是值xml中配置的depends-on属性声明的依赖关系，而不是一般情况下我们所指的循环依赖问题；
6. 创建Singleton类型的Bean
7. 创建Prototype类型的Bean
8. 创建其他scope类型的Bean
9. (不管是第1步，还是第6、7、8步中，获取到实例后，都需要经过getObjectForBeanInstance方法再返回）

###  <span id="getCachedBean">第1步：尝试从缓存中获取Bean</span>

根据BeanName获取实例：

```java
/**
	 * Return the (raw) singleton object registered under the given name.
	 * <p>Checks already instantiated singletons and also allows for an early
	 * reference to a currently created singleton (resolving a circular reference).
	 * @param beanName the name of the bean to look for
	 * @param allowEarlyReference whether early references should be created or not
	 * @return the registered singleton object, or {@code null} if none found
	 */
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}

```

这里涉及到了spring中对于单例Bean的 ***三级缓存*** ，主要是用于解决 ***循环依赖*** 问题：

```java
/** Cache of singleton objects: bean name --> bean instance */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);
/** Cache of early singleton objects: bean name --> bean instance */
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
/** Cache of singleton factories: bean name --> ObjectFactory */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);
```

从缓存中获取的步骤是先访问一级缓存singletonObjects，如果存在则返回，没有则访问二级缓存earlySingletonObjects；访问二级缓存，如果存在则返回，没有则访问三级缓存singletonFactories；三级缓存中保存的是创建Bean的工厂，如果三级缓存存在，则调用其getObject方法得到Bean存入二级缓存，并且将Bean工厂从三级缓存中移除，也就是说从三级缓存移入到二级缓存中。

- 一级缓存：单例对象缓存：当Bean已经被实例化、初始化之后，也就是完全创建完成后，会放入singletonObjects。
- 二级缓存：早期单例对象缓存：它是一个中间的过渡缓存，保存已经被实例化，但没有被属性注入的早期暴露Bean。 当Bean没有完全创建完成时，一级缓存中当然没有，尝试二级缓存中获取，如果二级缓存中，但是在三级缓存中有此Bean的创建工厂，意味着可以得到这个Bean的早期暴露对象，将Bean的工厂从三级缓存中移除，放入二级缓存中。
- 三级缓存：单例工厂缓存 ：它保存的是可以创建Bean的工厂。

###   <span id="checkBeanDep">第3步：Prototype类型Bean的循环依赖检查</span>

```java
protected boolean isPrototypeCurrentlyInCreation(String beanName) {
		Object curVal = this.prototypesCurrentlyInCreation.get();
		return (curVal != null &&
				(curVal.equals(beanName) || (curVal instanceof Set && ((Set<?>) curVal).contains(beanName))));
	}
```

其中的prototypesCurrentlyInCreation是一个ThreadLocal的集合，保存了当前线程中，正在创建中的Prototype类型BeanName。结构为：

```  java
/** Names of beans that are currently in creation */
	private final ThreadLocal<Object> prototypesCurrentlyInCreation =
			new NamedThreadLocal<Object>("Prototype beans currently in creation");
```

当在此变量中得到了要创建的Bean，说明在当前线程中，此Bean正在创建中，但为什么又要创建呢，说明存在循环依赖了。如A与B互相依赖，A在创建过程中被存入prototypesCurrentlyInCreation，当检查到了依赖B，会先去创建B。当创建B的时候，发现依赖了A，由于是prototype类型的，所以A会被再次创建，再创建时发现A已经存在于prototypesCurrentlyInCreation中，表示两者循环依赖了。
所以明显，Bean加入到prototypesCurrentlyInCreation的时机，是实例创建前

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

Bean从prototypesCurrentlyInCreation移除的时机，是实例创建后

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

###  <span id="dep-on">第5步：Depends-on依赖检查</span>

depends-on标签：它的作用是一个bean实例化的过程需要依赖于另一个bean的初始化，也就是说被依赖的bean将会在需要依赖的bean初始化之前加载。
获取当前Bean所依赖的dependsOnBean，检查是否循环依赖了，保存依赖关系，并且先初始化dependsOnBean。
首先检查循环依赖，出现了一个缓存：dependentBeanMap（保存的是beanName，所有依赖此BeanName的Bean集合）；
通过这个缓存得到（依赖beanName的Bean集合），检查集合中是否有dependentBeanName（beanName依赖的），即代表循环依赖了。

```java
private boolean isDependent(String beanName, String dependentBeanName, Set<String> alreadySeen) {
   String canonicalName = canonicalName(beanName);
   if (alreadySeen != null && alreadySeen.contains(beanName)) {
      return false;
   }
   Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
   if (dependentBeans == null) {
      return false;
   }
   if (dependentBeans.contains(dependentBeanName)) {
      return true;
   }
   for (String transitiveDependency : dependentBeans) {
      if (alreadySeen == null) {
         alreadySeen = new HashSet<String>();
      }
      alreadySeen.add(beanName);
      if (isDependent(transitiveDependency, dependentBeanName, alreadySeen)) {
         return true;
      }
   }
   return false;
}
```

检查完，没有循环依赖之后，再注册依赖关系到上述的dependentBeanMap缓存中；
另外，又出现了与之相对应的缓存：dependenciesForBeanMap（beanName，BeanName所依赖的所有Bean集合）。

```java
public void registerDependentBean(String beanName, String dependentBeanName) {
   // A quick check for an existing entry upfront, avoiding synchronization...
   String canonicalName = canonicalName(beanName);
   Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
   if (dependentBeans != null && dependentBeans.contains(dependentBeanName)) {
      return;
   }

   // No entry yet -> fully synchronized manipulation of the dependentBeans Set
   synchronized (this.dependentBeanMap) {
      dependentBeans = this.dependentBeanMap.get(canonicalName);
      if (dependentBeans == null) {
         dependentBeans = new LinkedHashSet<String>(8);
         this.dependentBeanMap.put(canonicalName, dependentBeans);
      }
      dependentBeans.add(dependentBeanName);
   }
   synchronized (this.dependenciesForBeanMap) {
      Set<String> dependenciesForBean = this.dependenciesForBeanMap.get(dependentBeanName);
      if (dependenciesForBean == null) {
         dependenciesForBean = new LinkedHashSet<String>(8);
         this.dependenciesForBeanMap.put(dependentBeanName, dependenciesForBean);
      }
      dependenciesForBean.add(canonicalName);
   }
}
```

