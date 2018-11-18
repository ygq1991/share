## RPC trace实践

### 1.创建代理类

需要追踪的接口 主要分为3类

* 直接拦截一个确定名称的类

  这类方法可以直接在xml中配置一个org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator

  然后配置好指定的拦截器和需要拦截的类名称

  ```java
  <bean id="cacheDigestLogInterceptor"
            class="cn.com.ygq.xqy.framework.log.interceptor.CacheDigestLogInterceptor"/>
      <bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
          <property name="beanNames">
              <list>
                  <value>cacheClient</value>
                  <value>redisCacheClient</value>
              </list>
          </property>
          <property name="interceptorNames">
              <list>
                  <value>cacheDigestLogInterceptor</value>
              </list>
          </property>
      </bean>
  ```


* 不能简单通过名称定义

  比如，JMS的类，代码中只是定义了通用的发消息的通用类，但是名字都是任意的。

  这个时候就要自定义实现一个AutoProxyCreator，通过继承BeanNameAutoProxyCreator的父类AbstractAutoProxyCreator。只要实现父类的获取拦截器的具体实现。resolveInterceptorNames 取自父类，自定义拦截器

  ```java
  public class JmsAutoProxyCreator extends AbstractAutoProxyCreator {
  
      @Override
      public Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName,
                                                   TargetSource customTargetSource) throws BeansException {
          if (自定义的类.isAssignableFrom(beanClass)) {
          	return resolveInterceptorNames(jmsInterceptorNames);
          }
          return DO_NOT_PROXY;
      }
      
      /**
       * 构建拦截器
       * 
       * @param interceptorNames
       * @return
       */
      private Advisor[] resolveInterceptorNames(String[] interceptorNames) {
          BeanFactory beanFactory = getBeanFactory();
          ConfigurableBeanFactory cbf = (beanFactory instanceof ConfigurableBeanFactory ? (ConfigurableBeanFactory) beanFactory
              : null);
          List<Advisor> advisors = new ArrayList<Advisor>();
          for (String beanName : interceptorNames) {
              if (cbf == null || !cbf.isCurrentlyInCreation(beanName)) {
                  Object next = beanFactory.getBean(beanName);
                  advisors.add(GlobalAdvisorAdapterRegistry.getInstance().wrap(next));
              }
          }
          return advisors.toArray(new Advisor[advisors.size()]);
      }
  }
  ```

* 需要自定义创建代理类，比如拦截dubbo的provider和consumer类。

  需要自定义实现一个AbstractAutoProxyCreator 级别的代理创建类

  主要实现一个createProxy方法

  ```java
  /**
       * 创建代理对象
       *
       * @param beanClass
       * @param beanName
       * @param targetSource
       * @return
       */
      protected Object createProxy(Class<?> beanClass, String beanName,
                                   String[] specificInterceptorNames, TargetSource targetSource) {
  
          ProxyFactory proxyFactory = new ProxyFactory();
          proxyFactory.copyFrom(this);
  
          if (!proxyFactory.isProxyTargetClass()) {
              evaluateProxyInterfaces(beanClass, proxyFactory);
          }
  
          Advisor[] advisors = buildAdvisors(beanName, specificInterceptorNames);
          for (Advisor advisor : advisors) {
              proxyFactory.addAdvisor(advisor);
          }
  
          proxyFactory.setTargetSource(targetSource);
          proxyFactory.setFrozen(false);
  
          return proxyFactory.getProxy(getProxyClassLoader());
      }
  ```



​	自定义一个AbstractServiceAutoProxyCreator 继承AbstractSimpleAutoProxyCreator 主要实现BeanPostProcessor的两个方法，

```java
/**
     * 在调用afterPropertiesSet和init-method之前调用
     *
     * @param bean
     * @param beanName
     * @return
     * @throws BeansException
     */
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
                                                                               throws BeansException {
        if (bean != null) {
            try {
                bean = wrapProviderService(bean, beanName);
            } catch (Exception e) {
                LoggerUtils.error(LOGGER, e, "创建服务发布代理失败,bean={0},beanName={1}", bean, beanName);
            }
        }
        return bean;
    }

    /**
     * 在调用afterPropertiesSet和init-method之后调用
     *
     * @param bean
     * @param beanName
     * @return
     * @throws BeansException
     */
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName)
                                                                              throws BeansException {
        if (bean != null) {
            try {
                bean = wrapConsumerService(bean, beanName);
            } catch (Exception e) {
                LoggerUtils.error(LOGGER, e, "创建服务引用代理失败,bean={0},beanName={1}", bean, beanName);
            }
        }
        return bean;
    }	
```



实现具体的代理封装在具体的子类中，DubboServiceAutoProxyCreator或者HsfServiceAutoProxyCreator。

主要实现wrapProviderService和wrapConsumerService方法

```java

    private static final String DUBBO_SERVICE_BEAN     = "com.alibaba.dubbo.config.spring.ServiceBean";

    private static final String DUBBO_REFERENCE_BEAN   = "com.alibaba.dubbo.config.spring.ReferenceBean";

    private static final String DUBBO_METETHOD_SET_REF = "setRef";

    private static final String DUBBO_METETHOD_GET_REF = "getRef";

public DubboServiceAutoProxyCreator() throws ClassNotFoundException, NoSuchMethodException {
        Class<?> dubboServiceName = Class.forName(DUBBO_SERVICE_BEAN);
        setRefMethod = dubboServiceName.getMethod(DUBBO_METETHOD_SET_REF, Object.class);
        setRefMethod.setAccessible(true);
        getRefMethod = dubboServiceName.getMethod(DUBBO_METETHOD_GET_REF);
        getRefMethod.setAccessible(true);
    }


/**
     * dubbo服务判断以及代理,修改service bean中的ref对象
     *
     * @param bean
     * @param beanName
     * @return
     * @throws IllegalAccessException
     * @throws InvocationTargetException
     */
    @Override
    protected Object wrapProviderService(Object bean, String beanName)
                                                                      throws IllegalAccessException,
                                                                      InvocationTargetException {
        if (bean.getClass().getName().equals(DUBBO_SERVICE_BEAN)) {
            Object ref = getRefMethod.invoke(bean);
            Object proxy = createProxy(ref.getClass(), beanName, this.serviceInterceptorNames,
                new SingletonTargetSource(ref));
            setRefMethod.invoke(bean, proxy);
        }
        return bean;
    }
```



主要是要清楚，提供者接口和消费者接口被ServiceBean和ReferenceBean封装，然后通过反射调用setRef，getRef方法。把dubbo封装引用的对象取出来，创建一个代理，然后放回去来实现。





### 2.拦截器



上次三种场景都需要配置不同的拦截器，以集体的rpc层的拦截器为例。

```java
 @Override public Object invoke(MethodInvocation invocation) throws Throwable {
        Class<?> clazz = invocation.getMethod().getDeclaringClass();
        String className = clazz.getSimpleName();
        String methodName = invocation.getMethod().getName();

        String sourceAppName = RpcContext.getContext()
                .getAttachment(AppNameDubboProviderFilter.APP_NAME_OF_CONSUMER);

        long startTime = System.currentTimeMillis();
        String salResult = RESULT_SUCCESS;
        try {
            TracerFactory.getRpcTracer().startReceive(className, methodName);
            // 通过service template返回的是result
            Object result = invocation.proceed();
            if (resultClass != null && isSuccessMethod != null && result != null && resultClass
                    .isAssignableFrom(result.getClass())) {
                Boolean isSuccess = (Boolean) isSuccessMethod.invoke(result);
                if (isSuccess) {
                    salResult = RESULT_SUCCESS;
                } else {
                    Object errorContext = getErrorMethod.invoke(result);
                    if (errorContext != null) {
                        salResult = (Boolean) isBusinessErrorMethod.invoke(errorContext) ?
                                RESULT_BIZERROR :
                                RESULT_ERROR;
                    } else {
                        salResult = RESULT_ERROR;
                    }
                }
            }
            return result;
        } catch (BusinessException e) {
            salResult = RESULT_BIZERROR;
            throw e;
        } catch (Throwable e) {
            if (serviceDigestExceptionHandler != null) {
                salResult = serviceDigestExceptionHandler.getLogResultIdentifier(e);
            } else {
                salResult = RESULT_ERROR;
            }
            throw e;
        } finally {
            TracerFactory.getRpcTracer().setRemoteIPToMDC();
            long elapsed = System.currentTimeMillis() - startTime;
            if (!shouldSkip(clazz.getName(), methodName)) {
                if (SERVICE_DIGEST_LOGGER.isInfoEnabled()) {
                    StringBuilder log = new StringBuilder("[(");
                    log.append(sourceAppName).append(",").append(className).append(",")
                            .append(methodName).append(",").append(salResult).append(",")
                            .append(elapsed).append("ms)]");
                    LoggerUtils.info(SERVICE_DIGEST_LOGGER, log.toString());
                }
            }
            TracerFactory.getRpcTracer().clearRemoteIPFromMDC();
            TracerFactory.getRpcTracer().finishReceive(salResult);

        }
    }
```



这里根据具体的业务需求对方法拦截后，获取方法的调用时间，调用结果，调用类方法信息。记录下来



### 3.traceId， RpcId的生成

这里使用到TracerFactory.getRpcTracer()这个工具类 RpcTracer.

  这里有四个方法 stratSend，startReceive. finishStart, finishEnd

 在logback.xml中配置traceId

```java
<!-- 时间滚动输出 业务异常 日志 -->
	<appender name="EXCEPTION-APPENDER" class="ch.qos.logback.core.rolling.RollingFileAppender">
		<file>${logging.root}/${app.name}/common-exception.log</file>
		<filter class="ch.qos.logback.core.filter.EvaluatorFilter">
			<evaluator class="ch.qos.logback.classic.boolex.JaninoEventEvaluator">
				<expression>return throwable != null &amp;&amp; (cn.com.ygq.xqy.framework.exception.BusinessException.class.isInstance(throwable) || cn.com.ygq.xqy.framework.exception.XQYException.class.isInstance(throwable));</expression>
			</evaluator>
			<onMatch>ACCEPT</onMatch>
			<onMismatch>DENY</onMismatch>
		</filter>
		<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
			<FileNamePattern>${logging.root}/${app.name}/common-exception.log.%d{yyyy-MM-dd}</FileNamePattern>
			<MaxHistory>30</MaxHistory>
		</rollingPolicy>
		<encoder charset="UTF-8">
			<pattern>%date [%-5level] [%thread] %logger{80} - %msg%X{traceId}%n</pattern>
		</encoder>
	</appender>
```











BeanPostProcessor 需要搞清楚继承这个类后在什么时候触发方法调用

