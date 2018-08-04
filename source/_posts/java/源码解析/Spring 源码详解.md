### Spring IOC

#### Spring IOC概述

1. 依赖反转：把控制权从具体的业务对象手中转交到平台或者框架中，Spring IOC就是实现这个模式的载体

2. 为什么要使用Spring IOC

   因为在面向对象程序设计中，一个对象会包含其它的对象，如果合伙对象的引用或依赖关系的管理有具体对象来完成，会导致代码的高度耦合和可测试性降低；如果把对象的管理和对象的注入都	交给Spring IOC来处理，那么解耦代码的同时还提高了可测试性

#### GenericApplicationContext的类图

![](https://ws1.sinaimg.cn/large/a67bf22fgy1ftluvkk5idj210j0bh765.jpg)

#### IOC容器的初始化过程

- 这里就会涉及到一个方法refresh，下面是它的源码：

  ```java
  	    public void refresh() throws BeansException, IllegalStateException {
          Object var1 = this.startupShutdownMonitor;
          synchronized(this.startupShutdownMonitor) {
              //初始化IOC容器之前的准备工作
              this.prepareRefresh();
              //得到一个BeanFactory,并且调用refreshBeanFactory(),这个方法会进行Bean的定位 、载入和
              //注册；但是在我看AnnotationConfigApplicationContext源码的时候，我没有发现它的载入和注
              //册
              ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
              //给BeanFactory设置一些属性
              this.prepareBeanFactory(beanFactory);
              try {
                  //设置BeanFactory的后置处理
                  this.postProcessBeanFactory(beanFactory);
                  //调用BeanFactory的后置处理器，这些处理器是在IOC容器中通过Bean注册的
                  this.invokeBeanFactoryPostProcessors(beanFactory);
                  //注册Bean的后处理器，在Bena的创建过程中调用
                  this.registerBeanPostProcessors(beanFactory);
                  //对上下文中的消息源进行初始化
                  this.initMessageSource();
                  //初始化上下文中的事件机制
                  this.initApplicationEventMulticaster();
                  //初始化其它特殊的Bean
                  this.onRefresh();
                  //检查监听Bean并且将这些Bean向容器注册
                  this.registerListeners();
                  //实例化所有的单件
                  this.finishBeanFactoryInitialization(beanFactory);
                  //发布容器事件，结束refresh过程
                  this.finishRefresh();
              } catch (BeansException var9) {
                  if (this.logger.isWarnEnabled()) {
                      this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                  }
                  //为防止Bean资源占用，在异常处理中销毁已经创建的单件Bean
                  this.destroyBeans();
                  //重置'active'标志
                  this.cancelRefresh(var9);
                  throw var9;
              } finally {
                  this.resetCommonCaches();
              }
  
          }
      }
  ```

  这个方法涉及到了BeanDefinition的定位、载入和注册三个基本过程

  一下是我看完源码后的一些感受

  1. context上下文的意义要了解，什么叫做上下文，其实所有的不管FilesystemXmlApplicationContext还是AnnotationConfigApplicationContext它们都不叫做真正的IOC容器，它们都拥有一个共同的属性DefaultListableBeanFactory，这个才是真正的IOC容器，实现了IOC容器的所有功能；但是它们却把这个IOC容器放在了不同的环境下，这个环境就叫做上下文；就比如说FileSystemXmlApplicationContext，它的环境就是Xml文件而AnnotationConfigApplicationContext的环境是注解；所以这个造成了它们不同的行为方式，一个通过解析xml文件，定位、载入和注册Bean而另外一个是通过解析注解定位、载入和注册Bean
  2. BeanDefinition的定位和载入是通过AnnotationConfigApplicationContext的AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner，所以定位和载入都是通过Context来完成的；注册是IOC容器的registerDefinition方法来完成的

#### IOC容器的依赖注入

- 涉及到的源码

  ```java
      public <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException {
          return this.doGetBean(name, requiredType, (Object[])null, false);
      }
  
      protected <T> T doGetBean(String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly) throws BeansException {
          String beanName = this.transformedBeanName(name);
          //从缓存中得到对象，避免重复创建
          Object sharedInstance = this.getSingleton(beanName);
          Object bean;
          if (sharedInstance != null && args == null) {
              if (this.logger.isDebugEnabled()) {
                  if (this.isSingletonCurrentlyInCreation(beanName)) {
                      this.logger.debug("Returning eagerly cached instance of singleton bean '" + beanName + "' that is not fully initialized yet - a consequence of a circular reference");
                  } else {
                      this.logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
                  }
              }
              //因为IOC中有FactoryBean和普通Bean两种，所以这里就是如果这个Bean是FactoryBean，
              //就取得它的真实对象
              bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, (RootBeanDefinition)null);
              //下面就是从父级容器里面得到Bean
          } else {
              if (this.isPrototypeCurrentlyInCreation(beanName)) {
                  throw new BeanCurrentlyInCreationException(beanName);
              }
  
              BeanFactory parentBeanFactory = this.getParentBeanFactory();
              if (parentBeanFactory != null && !this.containsBeanDefinition(beanName)) {
                  String nameToLookup = this.originalBeanName(name);
                  if (parentBeanFactory instanceof AbstractBeanFactory) {
                      return ((AbstractBeanFactory)parentBeanFactory).doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
                  }
  
                  if (args != null) {
                      return parentBeanFactory.getBean(nameToLookup, args);
                  }
  
                  return parentBeanFactory.getBean(nameToLookup, requiredType);
              }
  
              if (!typeCheckOnly) {
                  this.markBeanAsCreated(beanName);
              }
              //处理依赖，创建Bean
              try {
                  RootBeanDefinition mbd = this.getMergedLocalBeanDefinition(beanName);
                  this.checkMergedBeanDefinition(mbd, beanName, args);
                  String[] dependsOn = mbd.getDependsOn();
                  String[] var11;
                  if (dependsOn != null) {
                      var11 = dependsOn;
                      int var12 = dependsOn.length;
  
                      for(int var13 = 0; var13 < var12; ++var13) {
                          String dep = var11[var13];
                          if (this.isDependent(beanName, dep)) {
                              throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                          }
  
                          this.registerDependentBean(dep, beanName);
  
                          try {
                              this.getBean(dep);
                          } catch (NoSuchBeanDefinitionException var24) {
                              throw new BeanCreationException(mbd.getResourceDescription(), beanName, "'" + beanName + "' depends on missing bean '" + dep + "'", var24);
                          }
                      }
                  }
  
                  if (mbd.isSingleton()) {
                      sharedInstance = this.getSingleton(beanName, () -> {
                          try {
                              return this.createBean(beanName, mbd, args);
                          } catch (BeansException var5) {
                              this.destroySingleton(beanName);
                              throw var5;
                          }
                      });
                      bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                  } else if (mbd.isPrototype()) {
                      var11 = null;
  
                      Object prototypeInstance;
                      try {
                          this.beforePrototypeCreation(beanName);
                          prototypeInstance = this.createBean(beanName, mbd, args);
                      } finally {
                          this.afterPrototypeCreation(beanName);
                      }
  
                      bean = this.getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                  } else {
                      String scopeName = mbd.getScope();
                      Scope scope = (Scope)this.scopes.get(scopeName);
                      if (scope == null) {
                          throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                      }
  
                      try {
                          Object scopedInstance = scope.get(beanName, () -> {
                              this.beforePrototypeCreation(beanName);
  
                              Object var4;
                              try {
                                  var4 = this.createBean(beanName, mbd, args);
                              } finally {
                                  this.afterPrototypeCreation(beanName);
                              }
  
                              return var4;
                          });
                          bean = this.getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                      } catch (IllegalStateException var23) {
                          throw new BeanCreationException(beanName, "Scope '" + scopeName + "' is not active for the current thread; consider defining a scoped proxy for this bean if you intend to refer to it from a singleton", var23);
                      }
                  }
              } catch (BeansException var26) {
                  this.cleanupAfterBeanCreationFailure(beanName);
                  throw var26;
              }
          }
  
          if (requiredType != null && !requiredType.isInstance(bean)) {
              try {
                  T convertedBean = this.getTypeConverter().convertIfNecessary(bean, requiredType);
                  if (convertedBean == null) {
                      throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
                  } else {
                      return convertedBean;
                  }
              } catch (TypeMismatchException var25) {
                  if (this.logger.isDebugEnabled()) {
                      this.logger.debug("Failed to convert bean '" + name + "' to required type '" + ClassUtils.getQualifiedName(requiredType) + "'", var25);
                  }
  
                  throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
              }
          } else {
              return bean;
          }
      }
  ```

  doGetBean的流程图:

  ![](https://ws1.sinaimg.cn/large/a67bf22fgy1ftlwn5l3cij20db0lywfp.jpg)

  ```java
   protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
          BeanWrapper instanceWrapper = null;
          if (mbd.isSingleton()) {
              instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
          }
  
          if (instanceWrapper == null) {
              instanceWrapper = this.createBeanInstance(beanName, mbd, args);
          }
  
          Object bean = instanceWrapper.getWrappedInstance();
          Class<?> beanType = instanceWrapper.getWrappedClass();
          if (beanType != NullBean.class) {
              mbd.resolvedTargetType = beanType;
          }
  
          Object var7 = mbd.postProcessingLock;
          synchronized(mbd.postProcessingLock) {
              if (!mbd.postProcessed) {
                  try {
                      this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                  } catch (Throwable var17) {
                      throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Post-processing of merged bean definition failed", var17);
                  }
  
                  mbd.postProcessed = true;
              }
          }
  
          boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
          if (earlySingletonExposure) {
              if (this.logger.isDebugEnabled()) {
                  this.logger.debug("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
              }
  
              this.addSingletonFactory(beanName, () -> {
                  return this.getEarlyBeanReference(beanName, mbd, bean);
              });
          }
  
          Object exposedObject = bean;
  
          try {
              this.populateBean(beanName, mbd, instanceWrapper);
              exposedObject = this.initializeBean(beanName, exposedObject, mbd);
          } catch (Throwable var18) {
              if (var18 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var18).getBeanName())) {
                  throw (BeanCreationException)var18;
              }
  
              throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", var18);
          }
  
          if (earlySingletonExposure) {
              Object earlySingletonReference = this.getSingleton(beanName, false);
              if (earlySingletonReference != null) {
                  if (exposedObject == bean) {
                      exposedObject = earlySingletonReference;
                  } else if (!this.allowRawInjectionDespiteWrapping && this.hasDependentBean(beanName)) {
                      String[] dependentBeans = this.getDependentBeans(beanName);
                      Set<String> actualDependentBeans = new LinkedHashSet(dependentBeans.length);
                      String[] var12 = dependentBeans;
                      int var13 = dependentBeans.length;
  
                      for(int var14 = 0; var14 < var13; ++var14) {
                          String dependentBean = var12[var14];
                          if (!this.removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                              actualDependentBeans.add(dependentBean);
                          }
                      }
  
                      if (!actualDependentBeans.isEmpty()) {
                          throw new BeanCurrentlyInCreationException(beanName, "Bean with name '" + beanName + "' has been injected into other beans [" + StringUtils.collectionToCommaDelimitedString(actualDependentBeans) + "] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                      }
                  }
              }
          }
  
          try {
              this.registerDisposableBeanIfNecessary(beanName, bean, mbd);
              return exposedObject;
          } catch (BeanDefinitionValidationException var16) {
              throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", var16);
          }
      }
  ```

  这里有个BeanWapper，这个BeanWapper的作用是管理Bean的属性

#### 容器相关特性的设计与实现

##### ApplicationContext和Bean的初始化和销毁

ApplicationContext初始化:prepareBeanFactory()

ApplicationContext的销毁：doClose();

容器的实现是通过IOC管理Bean生命周期来实现的：

initailizeBean；doClose；destroy

##### lazy-init属性和预实例化

##### FactoryBean的实现

主要涉及到的源码：

```java
protected Object getObjectForBeanInstance(Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
        if (BeanFactoryUtils.isFactoryDereference(name)) {
            if (beanInstance instanceof NullBean) {
                return beanInstance;
            }

            if (!(beanInstance instanceof FactoryBean)) {
                throw new BeanIsNotAFactoryException(this.transformedBeanName(name), beanInstance.getClass());
            }
        }

        if (beanInstance instanceof FactoryBean && !BeanFactoryUtils.isFactoryDereference(name)) {
            Object object = null;
            if (mbd == null) {
                object = this.getCachedObjectForFactoryBean(beanName);
            }

            if (object == null) {
                FactoryBean<?> factory = (FactoryBean)beanInstance;
                if (mbd == null && this.containsBeanDefinition(beanName)) {
                    mbd = this.getMergedLocalBeanDefinition(beanName);
                }

                boolean synthetic = mbd != null && mbd.isSynthetic();
                object = this.getObjectFromFactoryBean(factory, beanName, !synthetic);
            }

            return object;
        } else {
            return beanInstance;
        }
    }
```

```java
    protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
        if (factory.isSingleton() && this.containsSingleton(beanName)) {
            synchronized(this.getSingletonMutex()) {
                Object object = this.factoryBeanObjectCache.get(beanName);
                if (object == null) {
                    object = this.doGetObjectFromFactoryBean(factory, beanName);
                    Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                    if (alreadyThere != null) {
                        object = alreadyThere;
                    } else {
                        if (shouldPostProcess) {
                            if (this.isSingletonCurrentlyInCreation(beanName)) {
                                return object;
                            }

                            this.beforeSingletonCreation(beanName);

                            try {
                                object = this.postProcessObjectFromFactoryBean(object, beanName);
                            } catch (Throwable var14) {
                                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's singleton object failed", var14);
                            } finally {
                                this.afterSingletonCreation(beanName);
                            }
                        }

                        if (this.containsSingleton(beanName)) {
                            this.factoryBeanObjectCache.put(beanName, object);
                        }
                    }
                }

                return object;
            }
        } else {
            Object object = this.doGetObjectFromFactoryBean(factory, beanName);
            if (shouldPostProcess) {
                try {
                    object = this.postProcessObjectFromFactoryBean(object, beanName);
                } catch (Throwable var17) {
                    throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", var17);
                }
            }

            return object;
        }
    }
```

上面两个方法解释了从FactoryBean取得真实对象

##### BeanPostProcessor的实现

这个我们看源码的initializeBean方法就行

```java
    protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
        if (System.getSecurityManager() != null) {
            AccessController.doPrivileged(() -> {
                this.invokeAwareMethods(beanName, bean);
                return null;
            }, this.getAccessControlContext());
        } else {
            this.invokeAwareMethods(beanName, bean);
        }
        //得到的是一个BeanWapper
        Object wrappedBean = bean;
        if (mbd == null || !mbd.isSynthetic()) {
            //这里就会一次调用所有Bean创建的前置处理器
            wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
        }

        try {
            //初始化Bean
            this.invokeInitMethods(beanName, wrappedBean, mbd);
        } catch (Throwable var6) {
            throw new BeanCreationException(mbd != null ? mbd.getResourceDescription() : null, beanName, "Invocation of init method failed", var6);
        }

        if (mbd == null || !mbd.isSynthetic()) {
            //调用所有Bean创建的后置处理器
            wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }
        return wrappedBean;
    }
```

这个方法会在getBean ->doGetBean() ->createBean-> doCreateBean中调用		

##### autowiring 的实现

```java
                if (mbd.getResolvedAutowireMode() == 1 || mbd.getResolvedAutowireMode() == 2) {
                    MutablePropertyValues newPvs = new MutablePropertyValues((PropertyValues)pvs);
                    if (mbd.getResolvedAutowireMode() == 1) {
                        this.autowireByName(beanName, mbd, bw, newPvs);
                    }

                    if (mbd.getResolvedAutowireMode() == 2) {
                        this.autowireByType(beanName, mbd, bw, newPvs);
                    }

                    pvs = newPvs;
                }
```

这是自动装配的实现，在AbstractAutowireCapableBeanFactory的populateBean方法中，这个populateBean 会在调用doCreateBean中调用

##### Bean对IOC容器的感知

### Spring AOP

#### Spring AOP 概述

业务逻辑的代码中不再含有针对特定领域问题代码的调用，业务逻辑同特定领域问题的关系通过切面来封装和维护，这样原本分散在整个应用程序中的变动就可以很好的管理起来

- 基础：视为待增强对象或者说目标对象；
- 切面：通常包含对于基础的增强应用；
- 配置：可以看成是一种编织，把基础和切面结合起来从而实现切面对目标对象的编织实现

在Spring AOP中有三个与上面对应：

- Advice通知：定义在切入点做什么，为切面增强提供织入接口，相当于上面的切面
- Pointcut切点：定义通知应该作用于哪个连接点，相当于上面的基础
- Advisor通知器：将切面和连接点结合起来的设计，相当于上面的配置

#### 增强对象功能的过程

首先给出AOP应用相关类的继承关系图：

![](https://ws1.sinaimg.cn/large/a67bf22fgy1fts57tjs3uj20op0dz0t2.jpg)

我们先以ProxyFactoryBean来讲解Aop

1. 配置ProxyFactoryBean

   ```xml
   <bean id="testAdvisor" class="comabc.TestAdvisor"/>
   <bean id="testAop" class="org.springframework.aop.ProxyFactoryBean">
   <property name="proxyInterfaces"><value>com.test.AbcInterface</value></property>
   <property name="target">
       <bean class="com.abc.TestTarget"/>
   </property>
   <property name="intercerptorNames">
       <list><value>testAdvisor</value></list>
   </property>
   </bean>
   ```

   - testAdvisor是定义切面
   - testAop定义ProxyFactoryBean，target属性是基础即待增强的类

2. ProxyFactoryBean**生成AopProxy代理对象**

   生成AopProxy代理对象的流程图

   ![](https://ws1.sinaimg.cn/large/a67bf22fgy1fts84yqduyj20x50gvwfd.jpg)

   ```java
   public Object getObject() throws BeansException {
       //初始化通知器链
       this.initializeAdvisorChain();
       //这里是对singleton和prototype的类型进行区分，生成对应的proxy
       if (this.isSingleton()) {
           return this.getSingletonInstance();
       } else {
           if (this.targetName == null) {
               this.logger.warn("Using non-singleton proxies with singleton targets is often undesirable. Enable prototype proxies by setting the 'targetName' property.");
           }
   
           return this.newPrototypeInstance();
       }
   }
   
    private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
           if (!this.advisorChainInitialized) {
               if (!ObjectUtils.isEmpty(this.interceptorNames)) {
                   if (this.beanFactory == null) {
                       throw new IllegalStateException("No BeanFactory available anymore (probably due to serialization) - cannot resolve interceptor names " + Arrays.asList(this.interceptorNames));
                   }
   
                   if (this.interceptorNames[this.interceptorNames.length - 1].endsWith("*") && this.targetName == null && this.targetSource == EMPTY_TARGET_SOURCE) {
                       throw new AopConfigException("Target required after globals");
                   }
   
                   String[] var1 = this.interceptorNames;
                   int var2 = var1.length;
                   //这里是添加Advisor链的调用，是通过interceprotNames属性进行设置的
                   for(int var3 = 0; var3 < var2; ++var3) {
                       String name = var1[var3];
                       if (this.logger.isTraceEnabled()) {
                           this.logger.trace("Configuring advisor or advice '" + name + "'");
                       }
   
                       if (name.endsWith("*")) {
                           if (!(this.beanFactory instanceof ListableBeanFactory)) {
                               throw new AopConfigException("Can only use global advisors or interceptors with a ListableBeanFactory");
                           }
   
                           this.addGlobalAdvisor((ListableBeanFactory)this.beanFactory, name.substring(0, name.length() - "*".length()));
                       } else {
                           Object advice;
                           if (!this.singleton && !this.beanFactory.isSingleton(name)) {
                               advice = new ProxyFactoryBean.PrototypePlaceholderAdvisor(name);
                           } else {
                               advice = this.beanFactory.getBean(name);
                           }
   
                           this.addAdvisorOnChainCreation(advice, name);
                       }
                   }
               }
   
               this.advisorChainInitialized = true;
           }
       }
   
   private synchronized Object getSingletonInstance() {
           if (this.singletonInstance == null) {
               this.targetSource = this.freshTargetSource();
               if (this.autodetectInterfaces && this.getProxiedInterfaces().length == 0 && !this.isProxyTargetClass()) {
                   Class<?> targetClass = this.getTargetClass();
                   if (targetClass == null) {
                       throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
                   }
   
                   this.setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
               }
   
               super.setFrozen(this.freezeProxy);
               this.singletonInstance = this.getProxy(this.createAopProxy());
           }
   
           return this.singletonInstance;
       }
   
   
       protected final synchronized AopProxy createAopProxy() {
           if (!this.active) {
               this.activate();
           }
   
           return this.getAopProxyFactory().createAopProxy(this);
       }
   
       public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
           if (!config.isOptimize() && !config.isProxyTargetClass() && !this.hasNoUserSuppliedProxyInterfaces(config)) {
               return new JdkDynamicAopProxy(config);
           } else {
               Class<?> targetClass = config.getTargetClass();
               if (targetClass == null) {
                   throw new AopConfigException("TargetSource cannot determine target class: Either an interface or a target is required for proxy creation.");
               } else {
                   //如果有接口使用JDK动态代理，没有接口使用CGLIB
                   return (AopProxy)(!targetClass.isInterface() && !Proxy.isProxyClass(targetClass) ? new ObjenesisCglibAopProxy(config) : new JdkDynamicAopProxy(config));
               }
           }
       }
   ```

3. Spring Aop拦截器调用实现

   我们就以JDK invoke方法讲解

   ```java
       @Nullable
       public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
           Object oldProxy = null;
           boolean setProxyContext = false;
           TargetSource targetSource = this.advised.targetSource;
           Object target = null;
   
           Boolean var9;
           try {
               if (this.equalsDefined || !AopUtils.isEqualsMethod(method)) {
                   //如果目标对象没有Object类的equals、hashCode方法
                   if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
                       Integer var19 = this.hashCode();
                       return var19;
                   }
   
                   if (method.getDeclaringClass() == DecoratingProxy.class) {
                       Class var18 = AopProxyUtils.ultimateTargetClass(this.advised);
                       return var18;
                   }
   
                   Object retVal;
                   if (!this.advised.opaque && method.getDeclaringClass().isInterface() && method.getDeclaringClass().isAssignableFrom(Advised.class)) {
                       //根据代理对象的配置(ProxyConfig)来调用服务
                       retVal = AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
                       return retVal;
                   }
   
                   if (this.advised.exposeProxy) {
                       oldProxy = AopContext.setCurrentProxy(proxy);
                       setProxyContext = true;
                   }
   
                   target = targetSource.getTarget();
                   Class<?> targetClass = target != null ? target.getClass() : null;
                   //获得经过过滤这个方法的所有拦截器
                   List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
                   if (chain.isEmpty()) {
                       Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                       retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
                   } else {
                       //沿着拦截器链执行方法
                       MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                       retVal = invocation.proceed();
                   }
   
                   Class<?> returnType = method.getReturnType();
                   if (retVal != null && retVal == target && returnType != Object.class && returnType.isInstance(proxy) && !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
                       retVal = proxy;
                   } else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
                       throw new AopInvocationException("Null return value from advice does not match primitive return type for: " + method);
                   }
   
                   Object var13 = retVal;
                   return var13;
               }
   
               var9 = this.equals(args[0]);
           } finally {
               if (target != null && !targetSource.isStatic()) {
                   targetSource.releaseTarget(target);
               }
   
               if (setProxyContext) {
                   AopContext.setCurrentProxy(oldProxy);
               }
   
           }
   
           return var9;
       }
   ```

   - AOP 拦截器链的调用

     ```java
     
     public Object proceed() throws Throwable {
             if (this.currentInterceptorIndex ==this.interceptorsAndDynamicMethodMatchers.size() - 1) {
                 return this.invokeJoinpoint();
      } else {
                 Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
                 if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
                     InterceptorAndDynamicMethodMatcher dm = (InterceptorAndDynamicMethodMatcher)interceptorOrInterceptionAdvice;
                     //如果通知器和调用的方法匹配，那么调用拦截器链的所有方法，否则继续匹配
                     return dm.methodMatcher.matches(this.method, this.targetClass, this.arguments) ? dm.interceptor.invoke(this) : this.proceed();
                 } else {
                     return ((MethodInterceptor)interceptorOrInterceptionAdvice).invoke(this);
                 }
             }
         }
     ```

- 拦截器链的产生

  ```java
   private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
          if (!this.advisorChainInitialized) {
              if (!ObjectUtils.isEmpty(this.interceptorNames)) {
                  if (this.beanFactory == null) {
                      throw new IllegalStateException("No BeanFactory available anymore (probably due to serialization) - cannot resolve interceptor names " + Arrays.asList(this.interceptorNames));
                  }
  
                  if (this.interceptorNames[this.interceptorNames.length - 1].endsWith("*") && this.targetName == null && this.targetSource == EMPTY_TARGET_SOURCE) {
                      throw new AopConfigException("Target required after globals");
                  }
  
                  String[] var1 = this.interceptorNames;
                  int var2 = var1.length;
                  //这里是添加Advisor链的调用，是通过interceprotNames属性进行设置的
                  for(int var3 = 0; var3 < var2; ++var3) {
                      String name = var1[var3];
                      if (this.logger.isTraceEnabled()) {
                          this.logger.trace("Configuring advisor or advice '" + name + "'");
                      }
  
                      if (name.endsWith("*")) {
                          if (!(this.beanFactory instanceof ListableBeanFactory)) {
                              throw new AopConfigException("Can only use global advisors or interceptors with a ListableBeanFactory");
                          }
  
                          this.addGlobalAdvisor((ListableBeanFactory)this.beanFactory, name.substring(0, name.length() - "*".length()));
                      } else {
                          Object advice;
                          if (!this.singleton && !this.beanFactory.isSingleton(name)) {
                              advice = new ProxyFactoryBean.PrototypePlaceholderAdvisor(name);
                          } else {
                              //从IOC容器中获得通知器或者切面，因为在实现中不管是通知器还是切面都会最终
                              //变成通知器Advisor
                              advice = this.beanFactory.getBean(name);
                          }
                          //添加到通知器中
                          this.addAdvisorOnChainCreation(advice, name);
                      }
                  }
              }
              this.advisorChainInitialized = true;
          }
      }
  
  //这个完成了从所有的通知器中得到拦截器链
  public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Advised config, Method method, @Nullable Class<?> targetClass) {
          List<Object> interceptorList = new ArrayList(config.getAdvisors().length);
          Class<?> actualClass = targetClass != null ? targetClass : method.getDeclaringClass();
          boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
          AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
      //从config中得到实现从xml中获得的所有通知器
          Advisor[] var8 = config.getAdvisors();
          int var9 = var8.length;
  
          for(int var10 = 0; var10 < var9; ++var10) {
              Advisor advisor = var8[var10];
              //拦截器链
              MethodInterceptor[] interceptors;
              if (advisor instanceof PointcutAdvisor) {
                  PointcutAdvisor pointcutAdvisor = (PointcutAdvisor)advisor;
                  if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
                      //全局的注册工厂获得一个通知器的拦截器(这里是同注册工厂里面的适配器将所有的)
                      //advisor变成特定类型的拦截器(MethodBeforeAdvice、AfterReturningAdvice等等)
                      interceptors = registry.getInterceptors(advisor);
                      //如果编码实现通知器，通知器中会实现切入点，只不过我们不怎么使用硬编码的形式实现
                      //切入点，所以下面再检查是否方法是否符合切入点
                      MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                      //下面是对所有的拦截器进行过滤，剩下这个方法的拦截器
                      if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
                          if (mm.isRuntime()) {
                              MethodInterceptor[] var15 = interceptors;
                              int var16 = interceptors.length;
  
                              for(int var17 = 0; var17 < var16; ++var17) {
                                  MethodInterceptor interceptor = var15[var17];
                                  //将所有方法加入拦截器链
                                  interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                              }
                          } else {
                              interceptorList.addAll(Arrays.asList(interceptors));
                          }
                      }
                  }
              } else if (advisor instanceof IntroductionAdvisor) {
                  IntroductionAdvisor ia = (IntroductionAdvisor)advisor;
                  if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
                      interceptors = registry.getInterceptors(advisor);
                      interceptorList.addAll(Arrays.asList(interceptors));
                  }
              } else {
                  Interceptor[] interceptors = registry.getInterceptors(advisor);
                  interceptorList.addAll(Arrays.asList(interceptors));
              }
          }
  
          return interceptorList;
      }
  ```

  - 方法增强的实现

    ```java
    //这个方法的意图是借助DefaultAdvisorAdapterRegistry的适配器将所有的Advisor变成特定的interceptor
    //方便在调用拦截器链时的调用
    public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
            List<MethodInterceptor> interceptors = new ArrayList(3);
            Advice advice = advisor.getAdvice();
            if (advice instanceof MethodInterceptor) {
                interceptors.add((MethodInterceptor)advice);
            }
    
            Iterator var4 = this.adapters.iterator();
    
        //根据所有的适配器来适配得到拦截器
            while(var4.hasNext()) {
                AdvisorAdapter adapter = (AdvisorAdapter)var4.next();
                if (adapter.supportsAdvice(advice)) {
                    interceptors.add(adapter.getInterceptor(advisor));
                }
            }
    
            if (interceptors.isEmpty()) {
                throw new UnknownAdviceTypeException(advisor.getAdvice());
            } else {
                return (MethodInterceptor[])interceptors.toArray(new MethodInterceptor[0]);
            }
        }
    ```

    ```java
    //我们就分析这个adapter，supportsAdvice用来适配判断能够通过这个适配器来得到特定的拦截器；
    //getInterceptor就是用来得到特定的拦截器
    class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {
        MethodBeforeAdviceAdapter() {
        }
    
        public boolean supportsAdvice(Advice advice) {
            return advice instanceof MethodBeforeAdvice;
        }
    
        public MethodInterceptor getInterceptor(Advisor advisor) {
            MethodBeforeAdvice advice = (MethodBeforeAdvice)advisor.getAdvice();
            return new MethodBeforeAdviceInterceptor(advice);
        }
    }
    ```

    ```java
    //这个就是一种拦截器，通过invoke方法，我们也可以知道，其实就是在调用方法之前先执行了增强的内容
    public class MethodBeforeAdviceInterceptor implements MethodInterceptor, Serializable {
        private MethodBeforeAdvice advice;
    
        public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
            Assert.notNull(advice, "Advice must not be null");
            this.advice = advice;
        }
    
        public Object invoke(MethodInvocation mi) throws Throwable {
            this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
            return mi.proceed();
        }
    }
    
    ```

#### 大概讲述一个Spring AOP实现的过程

1. 首先我们以getObject()为突破口，这个方法的目的是为了得到一个AopProxy，我认为这个AopProxy会传给动态代理参数来增强函数，然后用这个类的接口来接受这个代理类；getObject方法在创建代理类的之前通过initializeAdvisorChain方法将所有的通知器注册到ProxyFactoryBean的advisors中
2. 然后就是触发增强动能的AopProxy的invoke方法，这个方法会从全局的advisors中匹配得到所有有关调用方法的拦截器，invocation.proceed()通过这个方法会触发拦截器的调用，在拦截器调用过程中会把所有拦截器调用完，也会区别它们的先后关系

### 分析Spring Boot启动源码

```java
    public ConfigurableApplicationContext run(String... args) {
        //用来监听只能运行一个Spring boot
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
        this.configureHeadlessProperty();
        //这个是Spring Boot专有的监听器
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        listeners.starting();

        Collection exceptionReporters;
        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
            this.configureIgnoreBeanInfo(environment);
            Banner printedBanner = this.printBanner(environment);
            //初始化IOC容器实例
            context = this.createApplicationContext();
            exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
            //这里面完成了BeanDefinition的定位、载入、注册
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            //进行IOC容器的初始化
            this.refreshContext(context);
            this.afterRefresh(context, applicationArguments);
            stopWatch.stop();
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
            }

            listeners.started(context);
            this.callRunners(context, applicationArguments);
        } catch (Throwable var10) {
            this.handleRunFailure(context, var10, exceptionReporters, listeners);
            throw new IllegalStateException(var10);
        }

        try {
            listeners.running(context);
            return context;
        } catch (Throwable var9) {
            this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
            throw new IllegalStateException(var9);
        }
    }
```

```java
//这个方法比较有趣，在初始化完IOC容器后，注册了一个这个应用的shutdownhook
private void refreshContext(ConfigurableApplicationContext context) {
        this.refresh(context);
        if (this.registerShutdownHook) {
            try {
                context.registerShutdownHook();
            } catch (AccessControlException var3) {
                ;
            }
        }
    }
```

#### Spring 内置Tomcat启动源码解读

```java
    protected void onRefresh() {
        super.onRefresh();

        try {
            this.createWebServer();
        } catch (Throwable var2) {
            throw new ApplicationContextException("Unable to start web server", var2);
        }
    }
```

首先在初始化IOC容器的时候，通过onRefresh方法，调用creatWebServer创建指定的Web容器并启动

```java
 private void createWebServer() {
        WebServer webServer = this.webServer;
        ServletContext servletContext = this.getServletContext();
        if (webServer == null && servletContext == null) {
            ServletWebServerFactory factory = this.getWebServerFactory();
            this.webServer = factory.getWebServer(new ServletContextInitializer[]{this.getSelfInitializer()});
        } else if (servletContext != null) {
            try {
                this.getSelfInitializer().onStartup(servletContext);
            } catch (ServletException var4) {
                throw new ApplicationContextException("Cannot initialize servlet context", var4);
            }
        }

        this.initPropertySources();
    }

    public WebServer getWebServer(ServletContextInitializer... initializers) {
        Tomcat tomcat = new Tomcat();
        File baseDir = this.baseDirectory != null ? this.baseDirectory : this.createTempDir("tomcat");
        tomcat.setBaseDir(baseDir.getAbsolutePath());
        Connector connector = new Connector(this.protocol);
        tomcat.getService().addConnector(connector);
        this.customizeConnector(connector);
        tomcat.setConnector(connector);
        tomcat.getHost().setAutoDeploy(false);
        this.configureEngine(tomcat.getEngine());
        Iterator var5 = this.additionalTomcatConnectors.iterator();

        while(var5.hasNext()) {
            Connector additionalConnector = (Connector)var5.next();
            tomcat.getService().addConnector(additionalConnector);
        }

        this.prepareContext(tomcat.getHost(), initializers);
        return this.getTomcatWebServer(tomcat);
    }

    protected TomcatWebServer getTomcatWebServer(Tomcat tomcat) {
        return new TomcatWebServer(tomcat, this.getPort() >= 0);
    }

    public TomcatWebServer(Tomcat tomcat, boolean autoStart) {
        this.monitor = new Object();
        this.serviceConnectors = new HashMap();
        Assert.notNull(tomcat, "Tomcat Server must not be null");
        this.tomcat = tomcat;
        this.autoStart = autoStart;
        this.initialize();
    }

    private void initialize() throws WebServerException {
        logger.info("Tomcat initialized with port(s): " + this.getPortsDescription(false));
        Object var1 = this.monitor;
        synchronized(this.monitor) {
            try {
                this.addInstanceIdToEngineName();
                Context context = this.findContext();
                context.addLifecycleListener((event) -> {
                    if (context.equals(event.getSource()) && "start".equals(event.getType())) {
                        this.removeServiceConnectors();
                    }

                });
                this.tomcat.start();
                this.rethrowDeferredStartupExceptions();

                try {
                    ContextBindings.bindClassLoader(context, context.getNamingToken(), this.getClass().getClassLoader());
                } catch (NamingException var5) {
                    ;
                }

                this.startDaemonAwaitThread();
            } catch (Exception var6) {
                this.stopSilently();
                throw new WebServerException("Unable to start embedded Tomcat", var6);
            }

        }
    }
```

通过上面一步一步我们很容易发现Tomcat是怎么启动的

#### 关于Tomcat源码

- omcat的架构图

![](https://ws1.sinaimg.cn/large/a67bf22fgy1ftwlh8jmi0j20h90brq3m.jpg)

![](https://ws1.sinaimg.cn/large/a67bf22fgy1ftwlcws80xj20es09wt90.jpg)

Container的组成

![](https://ws1.sinaimg.cn/large/a67bf22fgy1ftxkshzdpvj21fc14wgmw.jpg)

- 一些名词

  1. Server：服务器，一个Tomcat只能有一个Server，可以看做就是Tomcat，用于控制Tomcat的生命周期

  2. Service：服务，有了Service就可以对外提供服务了，一个Server可以拥有多个Service

  3. Connector：连接器，表示接受请求的端点，并返回回复；Servlet容器处理请求，是需要Connector进行调度和控制的，Connector是Tomcat处理请求的主干，因此Connector的配置和使用对Tomcat的性能有着重要的影响 

  4. Engine：引擎，Engine下可以配置多个虚拟主机Virtual Host，每个虚拟主机都有一个域名，当Engine获得一个请求时，它把该请求匹配到某个Host上，然后把该请求交给该Host来处理，Engine有一个默认虚拟主机，当请求无法匹配到任何一个Host上的时候，将交给该默认Host来处理

  5. Host：虚拟主机，每个虚拟主机和某个网络域名Domain Name相匹配 ，每个虚拟主机下都可以部署(deploy)一个或者多个Web App ,每个Web App对应于一个Context，有一个Context path，当Host获得一个请求时，将把该请求匹配到某个Context上，然后把该请求交给该Context来处理，匹配的方法是“最长匹配”，所以一个path==""的Context将成为该Host的默认Context，所有无法和其它Context的路径名匹配的请求都将最终和该默认Context匹配 

  6. Context ：一个Context对应于一个Web Application，一个Web Application由一个或者多个Servlet组成，Context在创建的时候将根据配置文件CATALINA_HOME/conf/web.xml和WEBAPP_HOME/WEB-INF/web.xml载入Servlet类，当Context获得请求时，将在自己的映射表(mapping table)中寻找相匹配的Servlet类，如果找到，则执行该类，获得请求的回应，并返回。 一个Host可以有多个Context，就相当于多个Web Application；其中TomcatEmbeddedContext就相当于这个

  7. Wrapper：最底层的容器，是对 Servlet 的封装，负责 Servlet 实例的创 建、执行和销毁 

     需要注意的是	Engine、Host、Context、Wrapper都是实现Container接口的类，所以里面的结构就像第三张图一样一个容器嵌套一个容器

#### Spring Boot源码之内置Servlet容器

创建WebServer并且启动

```java
  protected void onRefresh() {
        super.onRefresh();

        try {
            this.createWebServer();
        } catch (Throwable var2) {
            throw new ApplicationContextException("Unable to start web server", var2);
        }
    }


private void createWebServer() {
        WebServer webServer = this.webServer;
        ServletContext servletContext = this.getServletContext();
        if (webServer == null && servletContext == null) {
            ServletWebServerFactory factory = this.getWebServerFactory();
            //参数是一些ServletContextInitializer，Servlet容器启动的时候会遍历这些ServletContextInitializer，并调用onStartup方法
            this.webServer = factory.getWebServer(new ServletContextInitializer[]{this.getSelfInitializer()});
        } else if (servletContext != null) {
            try {
                this.getSelfInitializer().onStartup(servletContext);
            } catch (ServletException var4) {
                throw new ApplicationContextException("Cannot initialize servlet context", var4);
            }
        }

        this.initPropertySources();
    }
```

ServletContextInitializer的其中一个具体内容，目的是添加Servlet、Filter、Listener，包括那些自定义的，它们最终都会通过适配器变为ServletRegistrationBean

```java

private void selfInitialize(ServletContext servletContext) throws ServletException {
        this.prepareWebApplicationContext(servletContext);
        //得到 IOC容器
        ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
        ServletWebServerApplicationContext.ExistingWebApplicationScopes existingScopes = new ServletWebServerApplicationContext.ExistingWebApplicationScopes(beanFactory);
        WebApplicationContextUtils.registerWebApplicationScopes(beanFactory, this.getServletContext());
        existingScopes.restore();
        //将ServletContext注册到IOC容器
        WebApplicationContextUtils.registerEnvironmentBeans(beanFactory, this.getServletContext());
        //得到ServletContextInitializerBean遍历器
        Iterator var4 = this.getServletContextInitializerBeans().iterator();

        while(var4.hasNext()) {
            //遍历执行所有ServletContextInitializer
            ServletContextInitializer beans = (ServletContextInitializer)var4.next();
            beans.onStartup(servletContext);
        }

    }
```



```java
//IOC容器初始化过程
protected void finishRefresh() {
        super.finishRefresh();
        WebServer webServer = this.startWebServer();
        if (webServer != null) {
            this.publishEvent(new ServletWebServerInitializedEvent(webServer, this));
        }

    }

//这里的启动WebServer其实就是注册所有的
    private WebServer startWebServer() {
        WebServer webServer = this.webServer;
        if (webServer != null) {
            webServer.start();
        }

        return webServer;
    }
//这里以Tomcat启动为例
   public void start() throws WebServerException {
        Object var1 = this.monitor;
        synchronized(this.monitor) {
            if (!this.started) {
                boolean var10 = false;

                try {
                    var10 = true;
                    this.addPreviouslyRemovedConnectors();
                    Connector var2 = this.tomcat.getConnector();
                    if (var2 != null && this.autoStart) {
                        this.performDeferredLoadOnStartup();
                    }

                    this.checkThatConnectorsHaveStarted();
                    this.started = true;
                    logger.info("Tomcat started on port(s): " + this.getPortsDescription(true) + " with context path '" + this.getContextPath() + "'");
                    var10 = false;
                } catch (ConnectorStartFailedException var11) {
                    this.stopSilently();
                    throw var11;
                } catch (Exception var12) {
                    throw new WebServerException("Unable to start embedded Tomcat server", var12);
                } finally {
                    if (var10) {
                        Context context = this.findContext();
                        ContextBindings.unbindClassLoader(context, context.getNamingToken(), this.getClass().getClassLoader());
                    }
                }

                Context context = this.findContext();
                ContextBindings.unbindClassLoader(context, context.getNamingToken(), this.getClass().getClassLoader());
            }
        }
    }

    private void performDeferredLoadOnStartup() {
        try {
            //得到该虚拟主机下的所有Context也就是WebApplication上下文
            Container[] var1 = this.tomcat.getHost().findChildren();
            int var2 = var1.length;

            for(int var3 = 0; var3 < var2; ++var3) {
                Container child = var1[var3];
                if (child instanceof TomcatEmbeddedContext) {
                    ((TomcatEmbeddedContext)child).deferredLoadOnStartup();
                }
            }

        } catch (Exception var5) {
            logger.error("Cannot start connector: ", var5);
            throw new WebServerException("Unable to start embedded Tomcat connectors", var5);
        }
    }

    public void deferredLoadOnStartup() {
        ClassLoader classLoader = this.getLoader().getClassLoader();
        ClassLoader existingLoader = null;
        if (classLoader != null) {
            existingLoader = ClassUtils.overrideThreadContextClassLoader(classLoader);
        }

        if (this.overrideLoadOnStart) {
            //加载该上下文下的所有Servlet
            super.loadOnStartup(this.findChildren());
        }

        if (existingLoader != null) {
            ClassUtils.overrideThreadContextClassLoader(existingLoader);
        }

    }

    public boolean loadOnStartup(Container[] children) {
        TreeMap<Integer, ArrayList<Wrapper>> map = new TreeMap();

        for(int i = 0; i < children.length; ++i) {
            Wrapper wrapper = (Wrapper)children[i];
            int loadOnStartup = wrapper.getLoadOnStartup();
            if (loadOnStartup >= 0) {
                Integer key = loadOnStartup;
                ArrayList<Wrapper> list = (ArrayList)map.get(key);
                if (list == null) {
                    list = new ArrayList();
                    map.put(key, list);
                }

                list.add(wrapper);
            }
        }

        Iterator i$ = map.values().iterator();

        while(i$.hasNext()) {
            ArrayList<Wrapper> list = (ArrayList)i$.next();
            Iterator i$ = list.iterator();

            while(i$.hasNext()) {
                Wrapper wrapper = (Wrapper)i$.next();

                try {
                    wrapper.load();
                } catch (ServletException var8) {
                    this.getLogger().error(sm.getString("standardContext.loadOnStartup.loadException", new Object[]{this.getName(), wrapper.getName()}), StandardWrapper.getRootCause(var8));
                    if (this.getComputedFailCtxIfServletStartFails()) {
                        return false;
                    }
                }
            }
        }

        return true;
    }
```



### 关于Spring AplicationContextEvent

```java
//AbstractApplicationContext下的
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
        Assert.notNull(event, "Event must not be null");
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Publishing event in " + this.getDisplayName() + ": " + event);
        }

        Object applicationEvent;
        if (event instanceof ApplicationEvent) {
            applicationEvent = (ApplicationEvent)event;
        } else {
            applicationEvent = new PayloadApplicationEvent(this, event);
            if (eventType == null) {
                eventType = ((PayloadApplicationEvent)applicationEvent).getResolvableType();
            }
        }
     //如果有事件没有处理完，就加入earlyApplicationEvents里面等待被处理
        if (this.earlyApplicationEvents != null) {
            this.earlyApplicationEvents.add(applicationEvent);
        } else {
           //真正开始处理事件
           this.getApplicationEventMulticaster().multicastEvent((ApplicationEvent)applicationEvent, eventType);
        }

        if (this.parent != null) {
            if (this.parent instanceof AbstractApplicationContext) {
                ((AbstractApplicationContext)this.parent).publishEvent(event, eventType);
            } else {
                this.parent.publishEvent(event);
            }
        }
    }

//SimpleApplicationEventMulticaster
   public void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType) {
        ResolvableType type = eventType != null ? eventType : this.resolveDefaultEventType(event);
       //这里取得了所有注册的相关类型的事件，至于怎么得到的，我这里简单的说一下；AbstractApplicationEventMulticaster.ListenerRetriever 这个类就是专门用于存储所有的监听器的，而且在AbstractApplicationEventMulticaster这个类下会缓存最近取的相关key的所有监听器，所以就可以从这个类里面得到监听器了
        Iterator var4 = this.getApplicationListeners(event, type).iterator();

        while(var4.hasNext()) {
            ApplicationListener<?> listener = (ApplicationListener)var4.next();
            //如果有线程池，使用线程池完成所有的监听器里面的内容
            Executor executor = this.getTaskExecutor();
            if (executor != null) {
                executor.execute(() -> {
                    this.invokeListener(listener, event);
                });
            } else {
                this.invokeListener(listener, event);
            }
        }

    }

```



### 相关面试题

http://www.importnew.com/15851.html#ioc_di

#### 其中值得学习的类

- ResolveableType

  这个类是为了解决获取泛型实际类型

  [这里具体使用方法](http://jinnianshilongnian.iteye.com/blog/1993608)

  

  

  