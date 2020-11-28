---
title: log打印时增加链路id
date: 2020-10-21 18:34:15
tags: aop
categories: spring
---

为mq服务生成一个trackingNo

<!--more-->

# 前提

      在使用分布式项目中,一个用户的一次请求应该是一条链路,当需要查找日志时,
    可以根据一个id来将用户在所有子模块中的流程都获取到,
    这时候在两个服务之间需要传递这个id(后面称它为tid).
    然后在日志打印中将tid输出,elk搜集到后,通过tid就可以查到所有日志;
      
    现在在查看问题的时候发现mq是没有这个tid的,因此自己加入到项目中,便于之后日志查询
    ----
    实际中如果发送mq的地方有tid可以加入到mq的header中;在aop中获取该tid



# 熟悉下aop

## springboot的版本
```
-- springboot的版本为
id 'org.springframework.boot' version '2.3.2.RELEASE'

-- aop需要引入
implementation 'org.springframework.boot:spring-boot-starter-aop'
```

开始测试aop各个注解对应的执行顺序
## 验证的注解:
- @Before
- @After
- @Around
- @AfterThrowing
- @AfterReturning

## 测试代码
## 切面
```

@Aspect
@Component
public class TrackingNoAop {

//    在项目中实际使用注解时启动报错,发现需要使用全路径,可能是公司项目包版本低的原因,不多深究
//    @Pointcut("@annotation(TIDLog)")
    @Pointcut("execution(* com.wyj.daily.test_project.controller..*.*(..))")
    public void aspTrackingNo(){
    }
    //2
    @Before("aspTrackingNo()")
    public void before(JoinPoint joinPoint) {
        System.out.println("before");
    }
    //4
    @After("aspTrackingNo()")
    public void after() {
        System.out.println("after");
    }
    //1
    @Around("aspTrackingNo()")
    public void around(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        System.out.println("around");
        Object proceed = proceedingJoinPoint.proceed(proceedingJoinPoint.getArgs());
        System.out.println("proceed = " + proceed);
    }
    //3
    @AfterThrowing(pointcut = "aspTrackingNo()", throwing = "e")
    public void afterThrowing(JoinPoint joinPoint, Exception e) {
        System.out.println("afterThrowing");
    }
    //3
    @AfterReturning(pointcut = "aspTrackingNo()")
    public void afterReturning() {
        System.out.println("afterReturning ");
    }
}
```
### 被切的类
```
    @GetMapping("/testAop/{type}")
    public ResponseEntity<String> testAop(@PathVariable Integer type) {
        if (type == 1) {
            throw new RuntimeException();
        }
        return ResponseEntity.ok("哈哈");
    }
```

### 正常时打印
```
around
before
afterReturning 
after
proceed = <200 OK OK,哈哈,[]>
```
### 抛异常时打印
```
around
before
afterThrowing
after
```

## 结论:
请求-> around开始  -> before -> 执行方法(proceed)-> afterThrowing/afterReturning ->after  -> around 结束


# 代码

## 非mq的时候(公司之前封装的)

### tid的本地线程变量:
```
public class LogTreadLocal {
    private static final ThreadLocal<String> trackingNoThreadLocal = new ThreadLocal();

    public LogTreadLocal() {
    }

    public static void setTrackingNo(String trackingNo) {
        trackingNoThreadLocal.set(trackingNo);
    }

    public static String getTrackingNo() {
        String trackNo = (String)trackingNoThreadLocal.get();
        return trackNo;
    }

    public static void removeTrackingNo() {
        trackingNoThreadLocal.remove();
    }
}

```

### 拦截器

```

public class LogTrackNoInterceptor implements HandlerInterceptor {
    public LogTrackNoInterceptor() {
    }

    //假设两个服务之前会传递的这个值放在header中,name为:x-transaction-id
    public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o) {
        LogTreadLocal.setTrackingNo(StringUtils.isEmpty(httpServletRequest.getHeader("x-transaction-id")) ? UUID.randomUUID().toString().replaceAll("-", "") : httpServletRequest.getHeader("x-transaction-id"));
        return true;
    }

    public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView) {
    }

    public void afterCompletion(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) {
        LogTreadLocal.removeTrackingNo();
    }
}

```
### 日志打印时输入tid即可

## 本次调整关于mq消费者的日志打印
    原本熟悉切面后,也准备放入本地线程变量中,但是公司使用logback进行日志打印,
    logback中有一个MDC功能,本质上也是本地线程变量,因此准备直接使用logback的MDC,
    两者也不存在冲突


### 注解
```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface TIDLog {
}
```
### 切面
```
@Aspect
@Component
@Slf4j
public class TrackingNoAop {

    @Pointcut("@annotation(com.freemud.delivery.annotation.TIDLog)")
    public void trackingNo() {
    }

    //2
    @Before("trackingNo()")
    public void before(JoinPoint joinPoint) {
        MDC.put("tid", UUIDUtils.getUUID());
    }

    @After("trackingNo()")
    public void after() {
        MDC.remove("tid");
    }


    @AfterThrowing(pointcut = "trackingNo()", throwing = "e")
    public void afterThrowing(Exception e) {
        log.error(ExceptionUtils.getFullStackOnLine(e));
    }

}

```
### logback调整

appender->encoder->pattern下增加该tid打印,比如:

        <pattern>%d{HH:mm:ss.SSS} %contextName [%X{tid}] %-5level %logger{36} - %msg%n</pattern>


​		
### 新增日志打印工具类

```

@Slf4j
public class LogUtils {

    public static BiFunction<Long, Long, Long> takeUpTime = (startTime, endTime) -> {
        if (startTime != null && endTime != null) {
            return endTime - startTime;
        }
        return null;
    };

    public static void info(String message, Object requestData, Object responseData) {
        StackTraceElement stackTraceElement = Thread.currentThread().getStackTrace()[2];
        log.info("ClassName: {} ,MethodName: {} , MethodLine: {}, Message: {} , RequestData: {} ,ResponseData: {}",
                stackTraceElement.getClassName(),
                stackTraceElement.getMethodName(),
                stackTraceElement.getLineNumber(),
                message,
                JSONObject.toJSONString(requestData),
                JSONObject.toJSONString(responseData)
        );
    }

    public static void info(String message, Object requestData, Object responseData, Long startTime, Long endTime) {
        StackTraceElement stackTraceElement = Thread.currentThread().getStackTrace()[2];
        log.info("ClassName: {} ,MethodName: {} , MethodLine: {} , startTime: {} , takeUpTime: {}" +
                        ", Message: {} , RequestData: {} ,ResponseData: {} ",
                stackTraceElement.getClassName(),
                stackTraceElement.getMethodName(),
                stackTraceElement.getLineNumber(),
                startTime,
                takeUpTime.apply(startTime, endTime),
                message,
                JSONObject.toJSONString(requestData),
                JSONObject.toJSONString(responseData)
        );
    }


    public static void error(String message, Object requestData, Exception e) {
        StackTraceElement stackTraceElement = Thread.currentThread().getStackTrace()[2];
        log.error("ClassName: {} ,MethodName: {} , MethodLine: {}, Message: {} , RequestData: {} ,ExceptionInfo: {}",
                stackTraceElement.getClassName(),
                stackTraceElement.getMethodName(),
                stackTraceElement.getLineNumber(),
                message,
                JSONObject.toJSONString(requestData),
                ExceptionUtils.getFullStackTrace(e)
        );
    }


    public static void error(String message, Object requestData, Long startTime, Long endTime, Exception e) {
        StackTraceElement stackTraceElement = Thread.currentThread().getStackTrace()[2];
        log.error("ClassName: {} ,MethodName: {} , MethodLine: {}, startTime: {} , takeUpTime: {} ," +
                        " Message: {} , RequestData: {} , ExceptionInfo: {} ,  ",
                stackTraceElement.getClassName(),
                stackTraceElement.getMethodName(),
                stackTraceElement.getLineNumber(),
                message,
                JSONObject.toJSONString(requestData),
                startTime,
                takeUpTime.apply(startTime, endTime),
                ExceptionUtils.getFullStackTrace(e)
        );

    }

}

```


# END








