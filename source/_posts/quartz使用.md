---
title: quartz使用
date: 2020-11-11 23:10:56
tags: quartz
categories: 定时任务
---

优先使用xxljob;条件不允许再考虑quartz

<!--more-->

## 背景

​    有一个ka项目的定时任务服务是单节点,而这个明显是不合理的,准备搞成支持分布式的定时任务,第一个想到的xxljob;但是由于项目是运行在别人公司的服务器上,身不由己,最终选择使用quartz来进行定时任务的管理

## 效果

​    因为quartz是没有界面的; 之前的公司甚至见过通过查库来增加定时任务,因此希望实现的效果就是能够通过接口来对定时任务进行增删改查(界面是不可能的,只能接口了),功能如下:

- 新增定时任务
- 修改定时任务的corn表达式
- 暂停一个定时任务
- 删除一个定时任务
- 查询所有定时任务的列表

除此之外,我并不想太多的动之前的代码,因此尽量调整量小一些



## 流程

- 1.引包

  ```java
      <dependency>
              <groupId>org.springframework.boot</groupId>
              <artifactId>spring-boot-starter-quartz</artifactId>
          </dependency>
  
  ```

  

- 2.建表语句:

  **不要百度**,因为quartz1.0和2.0建表不一样; 不同数据库也可能有一些差别,  

  建表语句在github上quartz.core包的resource下,可以先看下项目引用的版本再到github上切换对应的版本上找自己使用的数据库初始化语句,[最新master分支建表的位置](https://github.com/quartz-scheduler/quartz/tree/master/quartz-core/src/main/resources/org/quartz/impl/jdbcjobstore)

- 3.代码编写

  - 1.所有定时任务的枚举(后期使用这个枚举来进行定时任务的增删改查)

  ```java
  package com.freemud.enums;
  
  import com.freemud.extend.QuartzJobExtend;
  import com.freemud.quartz.CancelOrderSyncQuartz;
  import com.freemud.quartz.HrLinkSyncQuartz;
  import com.freemud.quartz.OrderSyncQuartz;
  import lombok.Getter;
  import org.springframework.util.StringUtils;
  
  @Getter
  public enum QuartzJobEnum {
  
      PULL_ORDER(1, OrderSyncQuartz.class),
      CANCEL_ORDER(2, CancelOrderSyncQuartz.class),
      HR_SYNC(3, HrLinkSyncQuartz.class),
      ;
      private Integer code;
  
      private Class<? extends QuartzJobExtend> clazz;
  
      QuartzJobEnum(Integer code, Class clazz) {
          this.code = code;
          this.clazz = clazz;
      }
  
      public static QuartzJobEnum getByCodeOrClassName(Integer code, String className) {
          if (code != null) {
              for (QuartzJobEnum value : values()) {
                  if (value.getCode() == code) {
                      return value;
                  }
              }
          }
          if (!StringUtils.isEmpty(className)) {
              for (QuartzJobEnum value : values()) {
                  if (className.equals(value.getClazz().getSimpleName())) {
                      return value;
                  }
              }
          }
          return null;
      }
  
  }
  
  ```

  - 2.扩展下定时任务,由于没有特别的规范要求,因此这里假设
    - JobName默认是类的简称
    - GroupName默认是全称
    - 任务描述由实现类进行说明

  ```java
  package com.freemud.extend;
  
  import org.springframework.scheduling.quartz.QuartzJobBean;
  
  public abstract class QuartzJobExtend  extends QuartzJobBean {
  
      public String getJobName() {
          return getClass().getSimpleName();
      }
  
      public String getJobGroupName() {
          return getClass().getName();
      }
  
      public abstract String getJobDescription();
  
  
      public String getTriggerDescription() {
          return getJobDescription() + "trigger";
      }
  }
  ```

  - 3.定时任务

    ```java
    @Component
    @Configuration
    @EnableScheduling
    @Slf4j
    public class CancelOrderSyncQuartz extends QuartzJobExtend {
    
        @Scheduled(cron = "0 0/1 * * * ? ")
        protected void executeQuartz() {
            //do something
        }    
    }
    	
    ```

    改为:

    ```java
    @Component
    @Configuration
    @EnableScheduling
    @Slf4j
    public class CancelOrderSyncQuartz extends QuartzJobExtend {
    
        protected void executeQuartz() {
            //do something
        } 
        
        @Override
        public String getJobDescription() {
            return "取消时同步定时任务";
        }
    
        @Override
        protected void executeInternal(JobExecutionContext jobExecutionContext) throws JobExecutionException {
            executeQuartz();
        }
    }
    ```

    ---

    至此定时任务已经正常在使用了; 但是如何初始化定时任务,如何在项目运行期间暂停/修改定时任务执行频率需要优化一下:

    ---

    

  - 4.定时任务的增删改查

    直接上代码; 修改定时任务同新增放在一个地方:

    ```java
    package com.freemud.service;
    
    import com.freemud.commonbase.utils.SpringUtils;
    import com.freemud.commonbase.vo.ScheduleAddVo;
    import com.freemud.commonbase.vo.ScheduleDeleteVo;
    import com.freemud.entity.response.ScheduleResponseVo;
    import com.freemud.enums.QuartzJobEnum;
    import com.freemud.extend.QuartzJobExtend;
    import com.google.common.collect.Sets;
    import lombok.extern.slf4j.Slf4j;
    import org.quartz.*;
    import org.quartz.impl.matchers.GroupMatcher;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.stereotype.Service;
    import org.springframework.util.CollectionUtils;
    
    import java.util.ArrayList;
    import java.util.List;
    import java.util.Set;
    
    
    @Slf4j
    @Service
    public class ScheduleService {
    
        @Autowired
        private Scheduler scheduler;
    
    
        public void addSchedule(ScheduleAddVo scheduleAddVo) throws SchedulerException {
            QuartzJobEnum jobEnum = QuartzJobEnum.getByCodeOrClassName(
                    scheduleAddVo.getCode(),
                    scheduleAddVo.getClassName());
            if (jobEnum == null) {
                return;
            }
            QuartzJobExtend jobExtend = SpringUtils.getBean(jobEnum.getClazz());
    
            TriggerKey triggerKey = TriggerKey.triggerKey(jobExtend.getJobName(), jobExtend.getJobGroupName());
    
            CronTrigger cronTrigger = TriggerBuilder.newTrigger()
                    .withIdentity(triggerKey)
                    .withDescription(jobExtend.getJobDescription())
                    .withSchedule(
                            CronScheduleBuilder.cronSchedule(scheduleAddVo.getCornExpressoin())
                                    .withMisfireHandlingInstructionDoNothing()
                    ).build();
    
            //进行更新操作
            if (scheduler.checkExists(triggerKey)) {
                JobKey jobKey = new JobKey(jobExtend.getJobName(), jobExtend.getJobGroupName());
                JobDetail jobDetail = scheduler.getJobDetail(jobKey);
    
                jobDetail.getJobBuilder().withDescription(jobExtend.getJobDescription());
                scheduler.scheduleJob(jobDetail, Sets.newHashSet(cronTrigger), true);
            }else{
                //执行新增操作
                JobDetail jobDetail = JobBuilder.newJob(jobExtend.getClass())
                        .withDescription(jobExtend.getJobDescription())
                        .withIdentity(jobExtend.getJobName(), jobExtend.getJobGroupName())
                        .build();
                scheduler.scheduleJob(jobDetail, cronTrigger);
            }
    
        }
    
    
        public void deleteSchedule(ScheduleDeleteVo scheduleDeleteVo) throws SchedulerException {
            QuartzJobEnum jobEnum = QuartzJobEnum.getByCodeOrClassName(
                    scheduleDeleteVo.getCode(),
                    scheduleDeleteVo.getClassName());
            if (jobEnum == null) {
                return;
            }
            QuartzJobExtend jobExtend = SpringUtils.getBean(jobEnum.getClazz());
            TriggerKey triggerKey = TriggerKey.triggerKey(jobExtend.getJobName(), jobExtend.getJobGroupName());
            if (scheduler.checkExists(triggerKey)) {
                scheduler.pauseTrigger(triggerKey);
                scheduler.unscheduleJob(triggerKey);
            }
        }
    
        public List<ScheduleResponseVo> listAllSchedule() throws SchedulerException {
            List<ScheduleResponseVo> result = new ArrayList<>();
            List<String> jobGroupNames = scheduler.getJobGroupNames();
            if (CollectionUtils.isEmpty(jobGroupNames)) {
                return result;
            }
            for (String jobGroupName : jobGroupNames) {
                Set<JobKey> jobKeySet = scheduler.getJobKeys(GroupMatcher.jobGroupEquals(jobGroupName));
                if (CollectionUtils.isEmpty(jobKeySet)) {
                    continue;
                }
                for (JobKey jobKey : jobKeySet) {
                    ScheduleResponseVo responseVo = new ScheduleResponseVo();
    
                    JobDetail jobDetail = scheduler.getJobDetail(jobKey);
                    TriggerKey triggerKey = TriggerKey.triggerKey(jobKey.getName(), jobGroupName);
                    Trigger trigger = scheduler.getTrigger(triggerKey);
    
                    ScheduleResponseVo.JobDetail jobDetailResponse = responseVo.new JobDetail();
                    jobDetailResponse.setJobName(jobKey.getName());
                    jobDetailResponse.setJobClass(jobDetail.getJobClass().getName());
                    jobDetailResponse.setJobGroupName(jobGroupName);
                    jobDetailResponse.setDescription(jobDetail.getDescription());
                    responseVo.setJobDetail(jobDetailResponse);
    
                    ScheduleResponseVo.Trigger triggerResponse = responseVo.new Trigger();
                    triggerResponse.setDescription(trigger.getDescription());
                    triggerResponse.setStartTime(trigger.getStartTime());
                    triggerResponse.setNextTime(trigger.getNextFireTime());
                    if (trigger instanceof CronTrigger) {
                        CronTrigger cronTrigger = (CronTrigger) trigger;
                        triggerResponse.setCornExpression(cronTrigger.getCronExpression());
                        triggerResponse.setExpressionSummary(cronTrigger.getExpressionSummary());
                    }
                    Trigger.TriggerState triggerState = scheduler.getTriggerState(triggerKey);
                    triggerResponse.setStatus(triggerState.toString());
                    responseVo.setTrigger(triggerResponse);
    
                    result.add(responseVo);
                }
            }
            return result;
        }
    
        public void pauseSchedule(Integer code) throws SchedulerException {
            QuartzJobEnum quartzJobEnum = QuartzJobEnum.getByCodeOrClassName(code, null);
            QuartzJobExtend jobExtend = SpringUtils.getBean(quartzJobEnum.getClazz());
            TriggerKey triggerKey = TriggerKey.triggerKey(jobExtend.getJobName(), jobExtend.getJobGroupName());
            if (scheduler.checkExists(triggerKey)) {
                scheduler.pauseTrigger(triggerKey);
            }
           
        }
    }
    
    ```

    查询定时任务时返回的实体类如下:

    ```java
    package com.freemud.entity.response;
    
    
    import lombok.Data;
    
    import java.util.Date;
    
    @Data
    public class ScheduleResponseVo {
    
        private JobDetail jobDetail;
    
        private Trigger trigger;
    
        @Data
        public class JobDetail {
    
            private String jobName;
    
            private String jobGroupName;
    
            private String jobClass;
    
            private String description;
        }
    
        @Data
        public class Trigger {
    
            private String cornExpression;
    
            private String expressionSummary;
    
            private String description;
    
            private Date startTime;
    
            private Date nextTime;
    
            private String status;
        }
    }
    
    
    ```

    

    然后在controller中进行调用即可;

    ```java
    
    @RestController
    @RequestMapping("/schedule")
    public class ScheduleController {
    
        @Autowired
        private ScheduleService scheduleService;
    
        @PostMapping("/add")
        public BaseResponse addSchedule(@RequestBody ScheduleAddVo scheduleAddVo) throws SchedulerException {
            scheduleService.addSchedule(scheduleAddVo);
            return ResponseUtil.success();
        }
    
        @DeleteMapping
        public BaseResponse deleteSchedule(@RequestBody ScheduleDeleteVo scheduleDeleteVo) throws SchedulerException {
            scheduleService.deleteSchedule(scheduleDeleteVo);
            return ResponseUtil.success();
        }
    
        @GetMapping
        public BaseResponse getSchedule() throws SchedulerException {
            return ResponseUtil.success(scheduleService.listAllSchedule());
        }
    
        @PostMapping("/pause/{code}")
        public BaseResponse pauseSchedule(@PathVariable Integer code) throws SchedulerException {
            scheduleService.pauseSchedule(code);
            return ResponseUtil.success();
        }
    }
    
    ```

    

    - 新增(同修改)时通过传入code或者类名来进行设置定时任务的执行如:

    ```json
    {
        "code": 2,
        "cornExpressoin" : "0 */1 * * * ?"
    }
    ```

    - 查询的效果:

      ```json
      {
          "code": 100,
          "msg": "success",
          "data": [
              {
                  "jobDetail": {
                      "jobName": "CancelOrderSyncQuartz$$EnhancerBySpringCGLIB$$df76ceae",
                      "jobGroupName": "com.freemud.quartz.CancelOrderSyncQuartz$$EnhancerBySpringCGLIB$$df76ceae",
                      "jobClass": "com.freemud.quartz.CancelOrderSyncQuartz$$EnhancerBySpringCGLIB$$df76ceae",
                      "description": "取消同步"
                  },
                  "trigger": {
                      "cornExpression": "*/5 * * * * ?",
                      "expressionSummary": "seconds: 0,5,10,15,20,25,30,35,40,45,50,55\nminutes: *\nhours: *\ndaysOfMonth: *\nmonths: *\ndaysOfWeek: ?\nlastdayOfWeek: false\nnearestWeekday: false\nNthDayOfWeek: 0\nlastdayOfMonth: false\nyears: *\n",
                      "description": "取消同步",
                      "startTime": "2020-11-11T05:41:16.000+0000",
                      "nextTime": "2020-11-11T05:42:15.000+0000",
                      "status": "NORMAL"
                  }
              },
              {
                  "jobDetail": {
                      "jobName": "OrderSyncQuartz",
                      "jobGroupName": "com.freemud.quartz.OrderSyncQuartz",
                      "jobClass": "com.freemud.quartz.OrderSyncQuartz",
                      "description": "拉单定时任务"
                  },
                  "trigger": {
                      "cornExpression": "0 */1 * * * ?",
                      "expressionSummary": "seconds: 0\nminutes: 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50,51,52,53,54,55,56,57,58,59\nhours: *\ndaysOfMonth: *\nmonths: *\ndaysOfWeek: ?\nlastdayOfWeek: false\nnearestWeekday: false\nNthDayOfWeek: 0\nlastdayOfMonth: false\nyears: *\n",
                      "description": "拉单定时任务",
                      "startTime": "2020-11-11T05:42:11.000+0000",
                      "nextTime": "2020-11-11T05:43:00.000+0000",
                      "status": "PAUSE"
                  }
              }
          ]
      }
      ```

      