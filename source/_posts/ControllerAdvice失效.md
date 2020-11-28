---
title: ControllerAdvice失效
date: 2020-11-06 17:19:27
tags: aop
categories: spring
---

jar包和本地冲突

<!--more-->

接手了一个项目, 整个项目没有使用过自定义异常,

自己加入了全局异常无效


自己加入的全局异常如下:

```java
package com.freemud.delivery.aop;

import com.freemud.delivery.entity.enums.ResultCodeEnum;
import com.freemud.delivery.entity.exception.DeliveryException;
import com.freemud.delivery.entity.util.ExceptionUtils;
import com.freemud.delivery.entity.vo.ApiResult;
import lombok.extern.slf4j.Slf4j;
import org.springframework.validation.ObjectError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.List;
import java.util.stream.Collectors;


@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler({MethodArgumentNotValidException.class})
    public ApiResult handler(MethodArgumentNotValidException e) {
        List<String> errorList = e.getBindingResult().getAllErrors()
                .stream()
                .map(ObjectError::getDefaultMessage)
                .map(String::valueOf)
                .collect(Collectors.toList());
        return new ApiResult(ResultCodeEnum.PARAM_ERROR, errorList);
    }


    @ExceptionHandler({DeliveryException.class})
    public ApiResult handler(DeliveryException e) {
        return new ApiResult(e.getCode(), e.getMessage(), null);
    }

    @ExceptionHandler({Exception.class})
    public ApiResult handler(Exception e) {
        log.error(ExceptionUtils.getFullStackTrace(e));
        return new ApiResult(ResultCodeEnum.SYSTEM_ERROR);
    }

}

```



使用时发现并没有生效,当我故意入参错误时,报错如下:

```tex
2020-11-06 17:51:51,113 WARN deliverycenter (AbstractHandlerExceptionResolver.java:140) Resolved [org.springframework.web.bind.MethodArgumentNotValidException: Validation failed for argument at index 0 in method: public com.freemud.delivery.entity.vo.ApiResult<com.freemud.delivery.entity.vo.PageResult<com.freemud.delivery.entity.vo.service.delivery.DeliveryVO>> com.freemud.delivery.service.controller.QueryDeliveryController.queryListByDeliveryStatus(com.freemud.delivery.entity.vo.service.delivery.QueryDeliveryListReqVO), with 1 error(s): [Field error in object 'queryDeliveryListReqVO' on field 'partnerId': rejected value []; codes [NotBlank.queryDeliveryListReqVO.partnerId,NotBlank.partnerId,NotBlank.java.lang.String,NotBlank]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [queryDeliveryListReqVO.partnerId,partnerId]; arguments []; default message [partnerId]]; default message [partnerId不能为空]] ]

```

我在console中搜索我的这个bean时发现这个提示:

```tex
2020-11-06 17:55:52,459 INFO deliverycenter (ExceptionHandlerExceptionResolver.java:288) Detected @ExceptionHandler methods in platformExceptionHandler
2020-11-06 17:55:52,459 INFO deliverycenter (ExceptionHandlerExceptionResolver.java:288) Detected @ExceptionHandler methods in globalExceptionHandler
```

原来这个服务引入了公司的一个基础包; 这个包里已经有了全局异常; 这个类如下:

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.freemud.framework.exception;

import com.freemud.framework.constants.SysStatusCode;
import com.freemud.framework.result.ApiResult;
import com.freemud.framework.util.Utils;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;

@ControllerAdvice
@ResponseBody
public class PlatformExceptionHandler {
    public PlatformExceptionHandler() {
    }

    @ExceptionHandler({Exception.class})
    public ApiResult handleException() {
        return new ApiResult(SysStatusCode.SYSTEM_ERROR);
    }

    @ExceptionHandler({PlatformException.class})
    public ApiResult handlePlatformException(PlatformException platformException) {
        return Utils.notNull(platformException.getCode()) ? new ApiResult(platformException.getCode(), platformException.getMsg()) : new ApiResult(SysStatusCode.SYSTEM_ERROR);
    }
}
```

我认为这个全局异常处理太差劲了; 完全没办法满足我( 同时我有点不理解了,原来是有自定义异常类的; 但是整个项目从来没有见到过一个使用的地方)

所以我现在需要做的事情就是让基础jar包的失效,使用我自己的全局异常处理

这时候在SpringBoot启动类中的排除这一个bean的注入,代码如下:

```java
@ComponentScan(value = "com.free.*",
        excludeFilters = @ComponentScan.Filter(
                type = FilterType.ASSIGNABLE_TYPE,
                classes = {PlatformExceptionHandler.class}
        ))
```

重启服务,调试成功