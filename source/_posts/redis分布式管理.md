---
title: try-finally分布式锁统一管理
date: 2021-02-04 22:48:09
categories: 代码小技巧
tags: 分布式锁
toc: true
---

每一个分布式锁都要自己try-catch一下?

<!--more-->

# 普通分布式锁:

## 代码

```java
 @Override
    public viod create(CreateDeliveryVo createDeliveryVo) {
        String lockKey = RedisLockConstants.DELIVERY_CREATE_LOCK_PREFIX + createDeliveryVo.getOrderId();
        boolean lockSuccess = false;
        try {
            if (lockSuccess = redisLockManager.lock(lockKey, RedisLockConstants.DEFAULT_REDIS_WAIT_TIME_SECOND_10)) {
                createDelivery(createDeliveryVo);
            }
        } catch (Exception e) {
            log.error("create delivery error, e: {}", ThrowableUtil.getStackTrace(e));
        } finally {
            if (lockSuccess) {
                redisLockManager.unLock(lockKey);
            }
        }
    }
```

## 功能

上述代码就是给createDelivery(createDeliveryVo)这个方法加一个锁; 

## 缺点

- 每一次使用分布式锁都要这么一坨及其相似的地方臃肿
- 博主当前接手的项目并发及其严重场景较多,因此无奈使用分布式锁进行管控,原本可读性差的代码加上分布式锁后更是惨不忍睹
- finally块中出现过因为判断反导致锁未释放的情况

# 优化方向

- 统一管理分布式锁的代码;业务代码不关注lock 和unlock
- 简化代码,使用分布式锁尽量简单

# 优化<下面代码以void方法为demo的例子>

## 定义一个自定义异常类

```java

@Getter
public class LockException extends RuntimeException{

    public static Integer LOCK_ERROR = 10001;

    private Integer code;

    public LockException() {
        this(LOCK_ERROR);
    }
    public LockException(Integer code) {
        super();
        this.code = code;
    }

    public LockException(String message, Integer code) {
        super(message);
        this.code = code;
    }

}
```

## 定义一个执行代码的接口

- 对于当前demo无返回值的直接使用Runnable也是可行的

```java
public interface RedisLockExtend {
    void execute();
}
```

## 定义一个与redis交互的实体类

```java
@Data
public class RedisLockEntity {

    private String lockKey;
    private Long expire;
    private TimeUnit timeUnit;

    public RedisLockEntity(String lockKey) {
        this(lockKey, 60L, TimeUnit.SECONDS);
    }

    public RedisLockEntity(String lockKey, Long expire, TimeUnit timeUnit) {
        this.lockKey = lockKey;
        this.expire = expire;
        this.timeUnit = timeUnit;
    }

}
```

## **定义最核心的类**

```java
@Component
public class RedisLockHelper {

    @Autowired
    protected StringRedisTemplate redisTemplate;

    public void execute(RedisLockEntity entity, RedisLockExtend redisLockExtend) {
        Boolean tryLock = false;
        try {
            tryLock = redisTemplate.opsForValue().setIfAbsent(entity.getLockKey(), "1",
                    entity.getExpire(), entity.getTimeUnit());
            if (tryLock) {
                redisLockExtend.execute();
            }else{
                throw new LockException("获取锁失败了");
            }
        } finally {
            if(tryLock) {
                redisTemplate.delete(entity.getLockKey());
            }
        }
    }
}
```

## 使用的代码

```java
  	@Autowired
    RedisLockHelper redisLockHelper;

    public void tryLock(Boolean flag) {
        redisLockHelper.execute(new RedisLockEntity("study.test.lock"), () -> {
        	//这里是业务代码

            //业务代码结束
        });
    }

```

# over