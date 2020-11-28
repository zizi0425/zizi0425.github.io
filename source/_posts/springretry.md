---
title: springretry
date: 2020-10-28 13:42:09
tags: retry
categories: spring
---

重试有很多方式,但是spring的肯定会比较香吧

<!--more-->

# 参考文章/连接

- [Java实现几种简单的重试机制](https://my.oschina.net/u/566591/blog/1526551)
- [spring-retry 源码README.md](https://github.com/spring-projects/spring-retry)
- [spring-retry（1.概念和基本用法）](https://www.jianshu.com/p/58e753ca0151)



# 背景
 项目之前有一段代码
 ```
 	public void syncPosByDelivery(DeliveryStatusDto dto, int retryNum) {
		try {
			if (retryNum < 2) {
				Response response = retryThree(dto, System.currentTimeMillis());
				if (!Objects.equals(response.code(), 204)) {
					Thread.sleep(5000L);
					retryNum++;
					syncPosByDelivery(dto, retryNum);
				}
			}
		} catch (InterruptedException e) {
			e.printStackTrace();
		}

	}
 ```
 代码实现了重试三次的功能;  
 现在需求是三方接口可能会有超时;如果超时就重试.上述代码虽然可以实现;但是我觉得实现方法不应该这么粗暴.上网搜到了Java实现几种简单的重试机制这篇文章后,准备选用spring-retry来实现我的功能  
 原因无它,因为这是spring出产的...  


 **官方文档快速上手学习下**
  # 非注解使用
 - 主要学习下基本概念; 主要是:RetryContext/*RetryPolicy/RecoveryCallback/RetryCallback

 - 重试时间策略:
  ```
    @Test
    public void springRetry1() throws Throwable {
        RetryTemplate retryTemplate = new RetryTemplate();

        TimeoutRetryPolicy policy = new TimeoutRetryPolicy();
        policy.setTimeout(3000L);

        retryTemplate.setRetryPolicy(policy);

        Object execute = retryTemplate.execute((RetryCallback<Object, Throwable>) context -> {
            TimeUnit.SECONDS.sleep(1);
            throw new RuntimeException();
        });
        System.out.println("execute = " + execute);
    }

  ```
- 增加重试后补偿
```
    @Test
    public void springRetry2() throws Throwable {
        RetryTemplate retryTemplate = new RetryTemplate();

        TimeoutRetryPolicy policy = new TimeoutRetryPolicy();
        policy.setTimeout(3000L);

        retryTemplate.setRetryPolicy(policy);

        Object execute = retryTemplate.execute((RetryCallback<Object, Throwable>) context -> {
                    TimeUnit.SECONDS.sleep(1);
                    throw new RuntimeException();
                },
                context -> {
                    return "recoveryCallBack";
                });
        System.out.println("execute = " + execute);
    }

```

 - 重试次数策略
 ```
     @Test
    public void springRetry3() throws Throwable {
        RetryTemplate build = RetryTemplate.builder()
                .maxAttempts(3)
                //执行的时间执行
                .fixedBackoff(1000)
                .retryOn(IllegalArgumentException.class)
                .build();

        Object execute = build.execute(context -> {
            // business logic here
            throw new RuntimeException();
        },
                (RecoveryCallback<Object>) context -> {
                    // recover logic here
                    return "111";
                });
        System.out.println("execute = " + execute);

    }
 ```
 - 重试次数策略2:
 ```
     @Test
    public void springRetry4(){
        RetryTemplate retryTemplate = new RetryTemplate();

        SimpleRetryPolicy simpleRetryPolicy = new SimpleRetryPolicy(3, Collections.singletonMap(SocketTimeoutException.class, true));
        retryTemplate.setRetryPolicy(simpleRetryPolicy);

        FixedBackOffPolicy backOffPolicy = new FixedBackOffPolicy();
        backOffPolicy.setBackOffPeriod(1000L);
        retryTemplate.setBackOffPolicy(backOffPolicy);


        RetryContext execute = null;
        try {
            execute = retryTemplate.execute(context -> {
                RetryContextCache retryContextCache = new MapRetryContextCache();
                retryContextCache.put("eee", context);
                throw new RuntimeException("");
            });
        } catch (RuntimeException e) {
            System.out.println("ExceptionUtils.getFullExceptionLine(e) = " + ExceptionUtils.getFullExceptionLine(e));
        }

        System.out.println("execute = " + execute);
    }
 ```

 # 注解使用
 - 和非注解使用特别相同;然后按照文档上又额外学习下listener的使用
 ```
    //自定义的listener
     @Bean
    public RetryListener retryListerner1() {
        return new RetryListener() {
            @Override
            public <T, E extends Throwable> boolean open(RetryContext context, RetryCallback<T, E> callback) {
                log.info("open ");
                logAttributeName("open", context);
                return true;
            }

            @Override
            public <T, E extends Throwable> void close(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
                log.info("close");
                logAttributeName("close",context);
            }

            @Override
            public <T, E extends Throwable> void onError(RetryContext context, RetryCallback<T, E> callback, Throwable throwable) {
                log.info("onError");
                logAttributeName("onError", context);
            }
        };
    }
    

    public static void logAttributeName(String methodName,RetryContext retryContext) {
        log.info("methodName : {} start", methodName);
        log.info("retryName : " + retryContext.getAttribute(RetryContext.NAME));
        for (String attributeName : retryContext.attributeNames()) {
            log.info(attributeName + ": " + retryContext.getAttribute(attributeName));
        }
        log.info("methodName : {} end", methodName);
    }
    
    @Bean
    public RetryListener retryListerner2(){
        return new StatisticsListener(new DefaultStatisticsRepository());
    }

 ```
- listener是用于接收到每次重试不同状态的通知;源码中默认应该是实例中的retryListerner2
- 注解使用

```
  @Retryable(listeners = "retryListerner1",
            maxAttempts = 2,
            backoff = @Backoff(delay = 100, maxDelay = 500))
    public void retry() throws BaseException {
        throw new BaseException("retry");
    }
```
 看一下listerner的调用顺序以及retryContext有哪些属性
 ```
 23:08:19.928 test_project [] WARN  o.s.r.policy.ExpressionRetryPolicy - #{...} syntax is not required for this run-time expression and is deprecated in favor of a simple expression string
23:08:19.959 test_project [] INFO  c.w.d.t.config.RetryConfig - open 
23:08:19.959 test_project [] INFO  c.w.d.t.config.RetryConfig - methodName : open start
23:08:19.960 test_project [] INFO  c.w.d.t.config.RetryConfig - retryName : null
23:08:19.960 test_project [] INFO  c.w.d.t.config.RetryConfig - methodName : open end
23:08:19.970 test_project [] INFO  c.w.d.t.config.RetryConfig - onError
23:08:19.971 test_project [] INFO  c.w.d.t.config.RetryConfig - methodName : onError start
23:08:19.971 test_project [] INFO  c.w.d.t.config.RetryConfig - retryName : public void com.wyj.daily.test_project.service.RetryService.retryException() throws com.wyj.daily.commonbase.exception.BaseException
23:08:19.971 test_project [] INFO  c.w.d.t.config.RetryConfig - context.name: public void com.wyj.daily.test_project.service.RetryService.retryException() throws com.wyj.daily.commonbase.exception.BaseException
23:08:19.971 test_project [] INFO  c.w.d.t.config.RetryConfig - methodName : onError end
23:08:20.988 test_project [] INFO  c.w.d.t.config.RetryConfig - onError
23:08:20.988 test_project [] INFO  c.w.d.t.config.RetryConfig - methodName : onError start
23:08:20.988 test_project [] INFO  c.w.d.t.config.RetryConfig - retryName : public void com.wyj.daily.test_project.service.RetryService.retryException() throws com.wyj.daily.commonbase.exception.BaseException
23:08:20.988 test_project [] INFO  c.w.d.t.config.RetryConfig - context.name: public void com.wyj.daily.test_project.service.RetryService.retryException() throws com.wyj.daily.commonbase.exception.BaseException
23:08:20.988 test_project [] INFO  c.w.d.t.config.RetryConfig - methodName : onError end
23:08:21.990 test_project [] INFO  c.w.d.t.config.RetryConfig - onError
23:08:21.990 test_project [] INFO  c.w.d.t.config.RetryConfig - methodName : onError start
23:08:21.990 test_project [] INFO  c.w.d.t.config.RetryConfig - retryName : public void com.wyj.daily.test_project.service.RetryService.retryException() throws com.wyj.daily.commonbase.exception.BaseException
23:08:21.990 test_project [] INFO  c.w.d.t.config.RetryConfig - context.name: public void com.wyj.daily.test_project.service.RetryService.retryException() throws com.wyj.daily.commonbase.exception.BaseException
23:08:21.990 test_project [] INFO  c.w.d.t.config.RetryConfig - methodName : onError end
23:08:21.990 test_project [] INFO  c.w.d.t.service.RetryService - -----------------------
23:08:21.990 test_project [] INFO  c.w.d.t.config.RetryConfig - close
23:08:21.990 test_project [] INFO  c.w.d.t.config.RetryConfig - methodName : close start
23:08:21.990 test_project [] INFO  c.w.d.t.config.RetryConfig - retryName : public void com.wyj.daily.test_project.service.RetryService.retryException() throws com.wyj.daily.commonbase.exception.BaseException
23:08:21.990 test_project [] INFO  c.w.d.t.config.RetryConfig - context.name: public void com.wyj.daily.test_project.service.RetryService.retryException() throws com.wyj.daily.commonbase.exception.BaseException
23:08:21.990 test_project [] INFO  c.w.d.t.config.RetryConfig - context.exhausted: true
23:08:21.990 test_project [] INFO  c.w.d.t.config.RetryConfig - context.recovered: true
23:08:21.990 test_project [] INFO  c.w.d.t.config.RetryConfig - context.closed: true
23:08:21.990 test_project [] INFO  c.w.d.t.config.RetryConfig - methodName : close end
 ```

 

 - 加入recover使用
 ```
    @Retryable(recover = "recover", value = BaseException.class)
    public void retryRecover() {
        throw new BaseException("retry");
    }

    @Recover
    public void recover(BaseException e) {
        log.info("-----------------------");
    }
 ```
- spel表达式:
官方文档中指出在注解中可以使用spel表达式; 如下:
```
@Retryable(exceptionExpression="message.contains('this can be retried')")
public void service1() {
  ...
}

@Retryable(exceptionExpression="message.contains('this can be retried')")
public void service2() {
  ...
}

@Retryable(exceptionExpression="@exceptionChecker.shouldRetry(#root)",
    maxAttemptsExpression = "#{@integerFiveBean}",
  backoff = @Backoff(delayExpression = "#{1}", maxDelayExpression = "#{5}", multiplierExpression = "#{1.1}"))
public void service3() {
  ...
}
```

之前对表达式没有深入了解到,因此下面exceptionChecker.shouldRetry(#root)这一个看不懂; 不知道是不是大多数人都知道...总之我百度不出来,
然后看了下[spel表达式](http://itmyhome.com/spring/expressions.html#expressions-operator-safe-navigation)后简单解释下上面的用例:
```
message.contains('this can be retried') -> 假设抛出异常为e;那么e.getMessage().contains('this can be retried')

@exceptionChecker.shouldRetry(#root) ->
有一个name为exceptionChecker的bean; bean中有一个方法为shoudRetry;入参是是这个异常本身
```

- **注意** : 其他代码可以cv; 如果上面那个代码直接cv是会抛出spel相关Exception;如果坚持使用;
请加入下面代码
```
	public static class ExceptionChecker {

		public boolean shouldRetry(Throwable t) {
			return true;
		}

	}
    //并在项目中引入该bean;
	@Bean
	public ExceptionChecker exceptionChecker() {
		return new ExceptionChecker();
	}
    //同理可得 integerFiveBean也是一个bean
	@Bean
	public Integer integerFiveBean() {
		return Integer.valueOf(5);
	}
```
配置文件中加入其他变量