---
layout: post
comments: true
title: "Spring之ApplicationListener"
date: 2019-09-04 02:25:16
image: '/assets/img/'
description: 'spring源码解析'
main-class: 'Spring'
color: '#7AAB13'
tags:
- Spring
categories:
twitter_text: 'spring源码解析'
introduction: 'spring源码解析'
---

### Spring之ApplicationListener



#### 背景

spring内置的一种观察者模式实现。涉及到的身份有三种：Event、publisher、Listener。当发布者发布一个事件，那么listener会观察到这个事件的发布，并执行自己相应的逻辑。

在实际的业务开发中，是一种非常好的解耦方式，有点消息的味道。比如我们应用中存在一个”新用户注册发放积分“的需求，可以做一个用户注册事件，当新用户注册时，发布一个新用户注册事件，由监听器去做积分发放的逻辑。



#### 身份源码简介

前面我们有聊到，主要涉及到三种身份，接下来我们分别看看这些身份下都有些什么能力（方法）。

*Publisher

```java
public interface ApplicationEventPublisher {

	/**
	 * Notify all <strong>matching</strong> listeners registered with this
	 * application of an application event. Events may be framework events
	 * (such as RequestHandledEvent) or application-specific events.
	 * @param event the event to publish
	 * @see org.springframework.web.context.support.RequestHandledEvent
	 */
	void publishEvent(ApplicationEvent event);

	/**
	 * Notify all <strong>matching</strong> listeners registered with this
	 * application of an event.
	 * <p>If the specified {@code event} is not an {@link ApplicationEvent},
	 * it is wrapped in a {@link PayloadApplicationEvent}.
	 * @param event the event to publish
	 * @since 4.2
	 * @see PayloadApplicationEvent
	 */
	void publishEvent(Object event);

}
```

发布者的能力非常精准，就是发布一个事件。同时重载了一个方法，可以发布一个对象，不过在过程中这个对象会被包装到PayloadApplicationEvent当中

*Event

```java
public abstract class ApplicationEvent extends EventObject {

	/** use serialVersionUID from Spring 1.2 for interoperability */
	private static final long serialVersionUID = 7099057708183571937L;

	/** System time when the event happened */
	private final long timestamp;


	/**
	 * Create a new ApplicationEvent.
	 * @param source the object on which the event initially occurred (never {@code null})
	 */
	public ApplicationEvent(Object source) {
		super(source);
		this.timestamp = System.currentTimeMillis();
	}


	/**
	 * Return the system time in milliseconds when the event happened.
	 */
	public final long getTimestamp() {
		return this.timestamp;
	}

}

----------------EventObject---------------
  public class EventObject implements java.io.Serializable {

    private static final long serialVersionUID = 5516075349620653480L;

    /**
     * The object on which the Event initially occurred.
     */
    protected transient Object source;

    /**
     * Constructs a prototypical Event.
     *
     * @param source the object on which the Event initially occurred
     * @throws IllegalArgumentException if source is null
     */
    public EventObject(Object source) {
        if (source == null)
            throw new IllegalArgumentException("null source");

        this.source = source;
    }
    
  }
```

Event其实也非常简单，他就是包含了一个不可被序列化的Object。

*Listener

```java
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

	/**
	 * Handle an application event.
	 * @param event the event to respond to
	 */
	void onApplicationEvent(E event);

}
```

EventListener是一个标志接口，标志接口一般都是某一类身份的判断，不具备任何属性和方法。在监听器中，只有一个onApplicationEvent方法，这个方法其实就是时间发生时被调用的服务。后续我们会分析一整套的调用逻辑。

至此，我们对今天要聊的时间发布监听有个大致的了解，对每个身份是什么，做什么也有了一定的了解。接下来，我们深入spring的源码，分析一个事件发布到监听执行到底是如何实现的。



#### 以ContextRefreshedEvent为例介绍事件发布监听

ContextRefreshedEvent是ApplicationEvent的具体实现，这个事件表示当一个应用容器，spring中容器我们一般称之为Context，这个容器被初始化或者刷新时触发。一下是它的源码，比较简单：

```java
public class ContextRefreshedEvent extends ApplicationContextEvent {

	/**
	 * Create a new ContextRefreshedEvent.
	 * @param source the {@code ApplicationContext} that has been initialized
	 * or refreshed (must not be {@code null})
	 */
	public ContextRefreshedEvent(ApplicationContext source) {
		super(source);
	}

}
```



我们前面介绍了很多，我们知道整体流程的入口就在Publisher中，我们定位到ApplicationEventPublisher的骨架实现，我们发现其实是通过Spring容器Context来实现的发布接口，代码如下：

```java
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {
		
		// 事件发布
protected void publishEvent(Object event, ResolvableType eventType) {
		Assert.notNull(event, "Event must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Publishing event in " + getDisplayName() + ": " + event);
		}

		// Decorate event as an ApplicationEvent if necessary
		// 装饰事件
		ApplicationEvent applicationEvent;
		if (event instanceof ApplicationEvent) {
			applicationEvent = (ApplicationEvent) event;
		}
		else {
		// 如果发布的是一个Object，那么用PayloadApplicationEvent来封装它
			applicationEvent = new PayloadApplicationEvent<Object>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent) applicationEvent).getResolvableType();
			}
		}

		// Multicast right now if possible - or lazily once the multicaster is initialized
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		}
		else {
		// 获取时间多播器，并且广播事件
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}
	}
	
	}
```

上面这部分代码核心就是”获取时间多播器，并进行广播“。我们进入#getApplicationEventMulticaster()的详细代码：

```java
	/**
	 * Return the internal ApplicationEventMulticaster used by the context.
	 * @return the internal ApplicationEventMulticaster (never {@code null})
	 * @throws IllegalStateException if the context has not been initialized yet
	 */
	ApplicationEventMulticaster getApplicationEventMulticaster() throws IllegalStateException {
		if (this.applicationEventMulticaster == null) {
			throw new IllegalStateException("ApplicationEventMulticaster not initialized - " +
					"call 'refresh' before multicasting events via the context: " + this);
		}
		return this.applicationEventMulticaster;
	}
```

applicationEventMulticaster这个东西的初始换是伴随着容器初始化完成的，并为容器所使用。获取了ApplicationEventMuticaster，接下来就可以进行广播了，我们先看下这个接口所支持的方法：

```java
public interface ApplicationEventMulticaster {

	/**
	 * Add a listener to be notified of all events.
	 * @param listener the listener to add
	 */
	void addApplicationListener(ApplicationListener<?> listener);

	/**
	 * Add a listener bean to be notified of all events.
	 * @param listenerBeanName the name of the listener bean to add
	 */
	void addApplicationListenerBean(String listenerBeanName);

	/**
	 * Remove a listener from the notification list.
	 * @param listener the listener to remove
	 */
	void removeApplicationListener(ApplicationListener<?> listener);

	/**
	 * Remove a listener bean from the notification list.
	 * @param listenerBeanName the name of the listener bean to add
	 */
	void removeApplicationListenerBean(String listenerBeanName);

	/**
	 * Remove all listeners registered with this multicaster.
	 * <p>After a remove call, the multicaster will perform no action
	 * on event notification until new listeners are being registered.
	 */
	void removeAllListeners();

	/**
	 * Multicast the given application event to appropriate listeners.
	 * <p>Consider using {@link #multicastEvent(ApplicationEvent, ResolvableType)}
	 * if possible as it provides a better support for generics-based events.
	 * @param event the event to multicast
	 */
	void multicastEvent(ApplicationEvent event);

	/**
	 * Multicast the given application event to appropriate listeners.
	 * <p>If the {@code eventType} is {@code null}, a default type is built
	 * based on the {@code event} instance.
	 * @param event the event to multicast
	 * @param eventType the type of event (can be null)
	 * @since 4.2
	 */
	void multicastEvent(ApplicationEvent event, ResolvableType eventType);

}
```

从这个接口来看，我们大致知道这个接口功能大致是：

1、维护所有的监听器Listener

2、广播事件到对应的监听器

接下来看它的multicastEvent的具体实现

```java
public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
	// 类型转换相关	
  ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
  // 获取所有该事件相关的所有监听器
		for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
      // 支持异步消息推送
			Executor executor = getTaskExecutor();
			if (executor != null) {
				executor.execute(new Runnable() {
					@Override
					public void run() {
						invokeListener(listener, event);
					}
				});
			}
			else {
        // 同步推送
				invokeListener(listener, event);
			}
		}
	}
```

广播事件的时候主要做了以下几件事情：

1、获取该事件相关的监听器

2、判断是否需要异步执行

3、把事件推送给listenner

接下来我们看最后一步，#invokeListener方法

```java
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
  // 是否有配置错误处理
		ErrorHandler errorHandler = getErrorHandler();
		if (errorHandler != null) {
			try {
				doInvokeListener(listener, event);
			}
			catch (Throwable err) {
				errorHandler.handleError(err);
			}
		}
		else {
			doInvokeListener(listener, event);
		}
	}

----------#doInvokeListener----------
  private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
		try {
      // 执行listener的onApplicationEvent方法
			listener.onApplicationEvent(event);
		}
		catch (ClassCastException ex) {
			String msg = ex.getMessage();
			if (msg == null || matchesClassCastMessage(msg, event.getClass().getName())) {
				// Possibly a lambda-defined listener which we could not resolve the generic event type for
				// -> let's suppress the exception and just log a debug message.
				Log logger = LogFactory.getLog(getClass());
				if (logger.isDebugEnabled()) {
					logger.debug("Non-matching event type for listener: " + listener, ex);
				}
			}
			else {
				throw ex;
			}
		}
	}
```

这个方法非常简单：

1、判断是否有异常处理的Handler，这个handler可以自己配置，主要是针对异步情况下的异常处理

2、执行listenner的onApplicationEvent方法

至此，整套事件发布监听的逻辑就结束了。下面演示一下实战：

每一个用户注册，我们需要给用户发放100优惠券，我们通过事件发布监听的方式去实现：

1、设置一个用户注册事件

```java
public class UserRegisterEvent extends ApplicationEvent {

    /**
     * 表示一个用户
     */
    private String userId;
     public UserRegisterEvent(Object source, String userId) {
        super(source);
        this.userId = userId;
    }
    /**
     * Create a new ApplicationEvent.
     *
     * @param source the object on which the event initially occurred (never {@code null})
     */
    public UserRegisterEvent(Object source) {
        super(source);
    }

    public String getUserId() {
        return userId;
    }

    public void setUserId(String userId) {
        this.userId = userId;
    }
}
```

2、设置一个监听器

```java
// 需要把listener交给spring容器进行管理
@Component
public class UserRegisterListener implements ApplicationListener<UserRegisterEvent> {

    /**
     * Handle an application event.
     *
     * @param event the event to respond to
     */
    @Override
    public void onApplicationEvent(UserRegisterEvent event) {
        System.out.printf("user %s get 100 ticket", event.getUserId());
    }
}

```

3、事件发布，我们用spring的publisher发布消息

``` java
    public static void main(String[] args) throws Exception {
      // 主动启动容器
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[]{"META-INF/spring/dubbo-demo-provider.xml"});
        context.start();
        System.out.println("有用户注册");
        context.publishEvent(new UserRegisterEvent(new Object(), "297988405"));

    }
```

4、运行结果

```java
有用户注册
user 297988405 get 100 ticket
```

