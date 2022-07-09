## 理解使用

想必大家都用过 **Spring** 的事件处理机制，简单说下，**Spring** 容器提供各个 **Bean** 之间的消息通信，往容器发送自定义 **Event 事件** 或 **对象（由 Spring 封装成 PayloadApplicationEvent
）**，并通知相应的处理该 **Event 事件** 的 **监听器ApplicationListener** ，使得Bean之间的依赖关系得到了解耦，便于程序的扩展，同时 **Spring** 内部实现了容器生命周期的 **Event事件** ，对  **Spring** 集成其他相关组件简直是very友好


### 1. 自定义ApplicationEvent事件

#### 继承ApplicationEvent或者其子类，编写自定义ApplicationEvent事件
```java
public abstract class SpringApplicationEvent extends ApplicaAtionEvent {
    private final String[] args;

    public SpringApplicationEvent(SpringApplication application, String[] args) {
        super(application);
        this.args = args;
    }

    public SpringApplication getSpringApplication() {
        return (SpringApplication)this.getSource();
    }

    public final String[] getArgs() {
        return this.args;
    }
}
```

#### 或者使用PayloadApplicationEvent\<T\> 

推荐使用，可以减少相应 **ApplicationEvent** 子类的定义

在第三步使用 **publishEvent** 时，可以直接 **publish** 一个任意 **Object**，Spring内部会判断是否为ApplicationEvent对象，如果不是则会帮忙封装成一个 **PayloadApplicationEvent**

```java
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
    Assert.notNull(event, "Event must not be null");
    Object applicationEvent;
    if (event instanceof ApplicationEvent) {
        applicationEvent = (ApplicationEvent)event;
    } else {
        // 当publish的对象不是ApplicationEvent以及子类时，由Spring自行封装成PayloadApplicationEvent
        applicationEvent = new PayloadApplicationEvent(this, event);
        if (eventType == null) {
            eventType = ((PayloadApplicationEvent)applicationEvent).getResolvableType();
        }
    }
    ...
    ...
 }
```


### 2. 自定义ApplicationListener监听器

#### 编写自定义监听器

```java
private class ContextRefreshListener implements ApplicationListener<ContextRefreshedEvent> {
   @Override
   public void onApplicationEvent(ContextRefreshedEvent event) {
      FrameworkServlet.this.onApplicationEvent(event);
   }
}
```

#### 注解方式@EventListener

推荐使用，减少 **ApplicationListener** 子类的编写，使用注解提高开发效率

```java
@Slf4j
@Component
public class CustomerEventListener {
    
    @EventListener
    public void xxnEventListener(PayloadApplicationEvent<User> event) {
        // your code
    }
}
```
#### 对于PayloadApplicationEvent，ApplicationListener提供静态方法构造对应的监听器
推荐使用
```jav
@Bean
public ApplicationListener<PayloadApplicationEvent<User>> customerListener() {
    return ApplicationListener.forPayload((user) -> {
        // your code
    });
}
```

### 3. 调用容器的publishEvent

```java
ApplicationContext applicationContext = SpringContextHolder.getApplicationContext();
//自定义事件
applicationContext.publishEvent(new YourEvent(this));
//使用PayloadApplicationEvent<T>
applicationContext.publishEvent(new PayloadApplicationEvent(this, "xxx"));
applicationContext.publishEvent("xxx");
```

## 源码解析

不得不说 **Spring** 将设计模式运用到了极致， **Spring事件处理** 底层原理即是我们熟悉的 **观察者设计模式** 

**观察者：ApplicationListener**

**被观察者：ApplicaAtionEvent**

**Spring容器** 使用一个 
 `Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet<>()` 
 存放不同的 **观察者ApplicationListener**

PS：以下解读基于读者熟悉Spring Bean的生命周期

### 1. 往容器里注册ApplicationListener

#### EventListenerMethodProcessor扫描@EventListener的方法，为其生成ApplicationListener并注册


```java
EventListenerMethodProcessor.java

// Bean生命周期的钩子，在单例Bean实例化后执行
@Override
public void afterSingletonsInstantiated() {
    ...
    String[] beanNames = beanFactory.getBeanNamesForType(Object.class);
    for (String beanName : beanNames) {
        ...
        try {
           //主要方法，解析Bean的@EventListener的方法
           processBean(beanName, type);
        }
        catch (Throwable ex) {
           throw new BeanInitializationException("Failed to process @EventListener " +
                 "annotation on bean with name '" + beanName + "'", ex);
        }
        ...
    }
    ...
}


private void processBean(final String beanName, final Class<?> targetType) {
    ...
    // 扫描到Bean的@EventListener的所有方法
    Map<Method, EventListener> annotatedMethods = null;
    try {
       annotatedMethods = MethodIntrospector.selectMethods(targetType,
             (MethodIntrospector.MetadataLookup<EventListener>) method ->
                   AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class));
    }
    ...
    
    
    ...
    
    // Non-empty set of methods
    ConfigurableApplicationContext context = this.applicationContext;
    Assert.state(context != null, "No ApplicationContext set");
    List<EventListenerFactory> factories = this.eventListenerFactories;
    Assert.state(factories != null, "EventListenerFactory List not initialized");
    // 遍历@EventListener方法
    for (Method method : annotatedMethods.keySet()) {
       // DefaultEventListenerFactory 和 TransactionalEventListenerFactory (不支持@EventListener，支持@TransactionalEventListener)
       for (EventListenerFactory factory : factories) {
          // 调用DefaultEventListenerFactory工厂
          if (factory.supportsMethod(method)) {
             Method methodToUse = AopUtils.selectInvocableMethod(method, context.getType(beanName));
             // 创建了一个ApplicationListene适配器 ApplicationListenerMethodAdapter
             ApplicationListener<?> applicationListener =
                   factory.createApplicationListener(beanName, targetType, methodToUse);
             if (applicationListener instanceof ApplicationListenerMethodAdapter) {
                ((ApplicationListenerMethodAdapter) applicationListener).init(context, this.evaluator);
             }
             // 注册到容器里
             context.addApplicationListener(applicationListener);
             break;
          }
       }
    }
    ...
}

```

#### 在Bean初始化后，扫描ApplicationListener类型的Bean注册到容器


```java

ApplicationListenerDetector.java

// Bean生命周期的钩子，在单例Bean初始化化执行
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
   // 遍历扫描ApplicationListener类型的Bean
   if (bean instanceof ApplicationListener) {
      Boolean flag = this.singletonNames.get(beanName);
      if (Boolean.TRUE.equals(flag)) {
         this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
      }
      ...
   }
   return bean;
}


```

### 2. 发送事件触发

#### publishEvent往ApplicationEventMulticaster广播器广播事件

```java 

AbstractApplicationContext.java

public void publishEvent(Object event) {
    this.publishEvent(event, (ResolvableType)null);
}

protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
    Assert.notNull(event, "Event must not be null");
    Object applicationEvent;
    
    // 判断是否为ApplicationEvent，否则包装成PayloadApplicationEvent
    if (event instanceof ApplicationEvent) {
        applicationEvent = (ApplicationEvent)event;
    } else {
        applicationEvent = new PayloadApplicationEvent(this, event);
        if (eventType == null) {
            eventType = ((PayloadApplicationEvent)applicationEvent).getResolvableType();
        }
    }

    if (this.earlyApplicationEvents != null) {
        this.earlyApplicationEvents.add(applicationEvent);
    } else {
       // 使用事件广播器 广播事件
       this.getApplicationEventMulticaster().multicastEvent((ApplicationEvent)applicationEvent, eventType);
    }

    // 如有父容器，向父容器发送事件
    if (this.parent != null) {
        if (this.parent instanceof AbstractApplicationContext) {
            ((AbstractApplicationContext)this.parent).publishEvent(event, eventType);
        } else {
            this.parent.publishEvent(event);
        }
    }

}

```

#### 默认的ApplicationEventMulticaster广播器找到匹配的事件监听器，并执行监听器的onApplicationEvent

```java

// 默认的事件广播器，可自己扩展注入容器里面
SimpleApplicationEventMulticaster.java


public void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = eventType != null ? eventType : this.resolveDefaultEventType(event);
    Executor executor = this.getTaskExecutor();
    // 找到监听该事件的监听器
    Iterator var5 = this.getApplicationListeners(event, type).iterator();

    // 调用监听器的逻辑
    while(var5.hasNext()) {
        ApplicationListener<?> listener = (ApplicationListener)var5.next();
        if (executor != null) {
            // 使用线程池去执行监听器的逻辑， 默认是没有
            executor.execute(() -> {
                this.invokeListener(listener, event);
            });
        } else {
            this.invokeListener(listener, event);
        }
    }

}


protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
    ErrorHandler errorHandler = this.getErrorHandler();
    if (errorHandler != null) {
        try {
            // 直接调用listener的onApplicationEvent方法
            this.doInvokeListener(listener, event);
        } catch (Throwable var5) {
            errorHandler.handleError(var5);
        }
    } else {
        this.doInvokeListener(listener, event);
    }

}

```