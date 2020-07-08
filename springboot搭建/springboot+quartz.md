# Springboot整合Quartz实现动态定时任务
>作者：高帅帅 
 时间：2019/07/22
## 简介
Quartz是一款功能强大的任务调度器，可以实现较为复杂的调度功能，如每月一号执行、每天凌晨执行、每周五执行等等，还支持分布式调度。本文使用Springboot+Mybatis+Quartz实现对定时任务的增、删、改、查、启用、停用等功能。并把定时任务持久化到数据库以及支持集群。
## Quartz的3个基本要素
1. Scheduler：调度器。所有的调度都是由它控制。  
2. Trigger： 触发器。决定什么时候来执行任务。  
3. JobDetail & Job： JobDetail定义的是任务数据，而真正的执行逻辑是在Job中。使用JobDetail + Job而不是Job，这是因为任务是有可能并发执行，如果Scheduler直接使用Job，就会存在对同一个Job实例并发访问的问题。而JobDetail & Job 方式，sheduler每次执行，都会根据JobDetail创建一个新的Job实例，这样就可以规避并发访问的问题。
## 需要的基础包：pom.xml
```
 <dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.2.1</version>
 </dependency>
 <dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-quartz</artifactId>
    <version>1.2.2</version>
    <exclusions>
        <exclusion>
            <groupId>org.opensymphony.quartz</groupId>
            <artifactId>quartz</artifactId>
        </exclusion>
    </exclusions>
 </dependency>
```
## Quartz配置
```
/**
 * 创建job 实例工厂，解决spring注入问题，如果使用默认会导致spring的@Autowired 无法注入问题
 * @author LLQ
 *
 */
@Component
public class MyJobFactory extends AdaptableJobFactory {
    private AutowireCapableBeanFactory factory;

    public MyJobFactory(AutowireCapableBeanFactory factory) {
        this.factory = factory;
    }

    /**
     * 创建Job实例
     */
    @Override
    protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {

        // 实例化对象
        Object job = super.createJobInstance(bundle);
        // 进行注入（Spring管理该Bean）
        factory.autowireBean(job);
        //返回对象
        return job;
    }
}
```
```
@Configuration
public class QuartzConfig {

    private MyJobFactory jobFactory;

    public QuartzConfig(MyJobFactory jobFactory){
        this.jobFactory = jobFactory;
    }

    @Bean
    public SchedulerFactoryBean schedulerFactoryBean() {
        // Spring提供SchedulerFactoryBean为Scheduler提供配置信息,并被Spring容器管理其生命周期
        SchedulerFactoryBean factory = new SchedulerFactoryBean();
        // 设置自定义Job Factory，用于Spring管理Job bean
        factory.setJobFactory(jobFactory);
        return factory;
    }

    @Bean(name = "scheduler")
    public Scheduler scheduler() {
        return schedulerFactoryBean().getScheduler();
    }

}
```
## 创建quartz配置表
```
create table task_info(
    Id int primary key auto_increment,
    jobName varchar(100),
    jobGroup varchar(100),
	jobDescription varchar(100),
	jobStatus varchar(1),
	cronExpression varchar(100),
	createTime varchar(100),
	milliSeconds varchar(100),
	repeatCount varchar(100),
	startDate varchar(100),
	endDate varchar(100),
	del_flag varchar(1)
 )COMMENT = '批处理表';

alter TABLE task_info MODIFY COLUMN Id int(11) COMMENT 'id，自增';
alter TABLE task_info MODIFY COLUMN jobName varchar(100) COMMENT '批处理方法名';
alter TABLE task_info MODIFY COLUMN jobGroup varchar(100) COMMENT '任务组';
alter TABLE task_info MODIFY COLUMN jobDescription varchar(100) COMMENT '批处理描述';
alter TABLE task_info MODIFY COLUMN jobStatus varchar(1) COMMENT '状态，1：启动，2：停止';
alter TABLE task_info MODIFY COLUMN cronExpression varchar(100) COMMENT '定时表达式';
alter TABLE task_info MODIFY COLUMN milliSeconds varchar(100) COMMENT '任务间隔时间';
alter TABLE task_info MODIFY COLUMN repeatCount varchar(100) COMMENT '任务执行次数';
alter TABLE task_info MODIFY COLUMN startDate varchar(100) COMMENT '任务开始时间';
alter TABLE task_info MODIFY COLUMN endDate varchar(100) COMMENT '任务结束时间';
alter TABLE task_info MODIFY COLUMN del_flag varchar(1) COMMENT '是否删除标志，2：删除';
```
## cronExpression表达式
| 字段      |    允许值            | 允许的特殊字符    |  
| :-------  | --------:            | :--:              |  
| 秒        |    0-59              |  , - * /          |  
| 分        |    0-59              |  , - * /          |  
| 小时      |    0-23              |  , - * /          | 
| 日期      |    1-31              |  , - *   / L W C  |  
| 月份      |    1-12 或者 JAN-DEC |  , - * /          |  
| 星期      |    1-7 或者 SUN-SAT  |  , - *   / L C #  |
| 年（可选）|    留空, 1970-2099   |  , - * /          |

```“*”字符被用来指定所有的值。如：“*”在分钟的字段域里表示“每分钟”。```   
```“-”字符被用来指定一个范围。如：“10-12”在小时域意味着“10点、11点、12点”。```  
```“,”字符被用来指定另外的值。如：“MON,WED,FRI”在星期域里表示”星期一、星期三、星期五”。```  
```“?”字符只在日期域和星期域中使用。它被用来指定“非明确的值”。当你需要通过在这两个域中的一个来指定一些东西的时候，它是有用的。```  
```“L”字符指定在月或者星期中的某天（最后一天）。即“Last ”的缩写。但是在星期和月中“Ｌ”表示不同的意思，如：在月子段中“L”指月份的最后一天-1月31日，2月28日，如果在星期字段中则简单的表示为“7”或者“SAT”。如果在星期字段中在某个value值得后面，则表示“某月的最后一个星期value”,如“6L”表示某月的最后一个星期五。```  
```“W”字符只能用在月份字段中，该字段指定了离指定日期最近的那个星期日。```  
```“#”字符只能用在星期字段，该字段指定了第几个星期value在某月中。  ```  
每一个元素都可以显式地规定一个值（如6），一个区间（如9-12），一个列表（如9，11，13）或一个通配符（如*）。“月份中的日期”和“星期中的日期”这两个元素是互斥的，因此应该通过设置一个问号（？）来表明你不想设置的那个字段。   
例：
cron表达式  




| 表达式                |    意义            |   
| :-------              | --------:            |   
| "0 0 12 * * ?"        |   每天中午12点触发   |   
| "0 15 10 ? * *"        |    每天上午10:15触发             |  
| "0 15 10 * * ?"      |    每天上午10:15触发              |  
| "0 15 10 * * ? *"      |    每天上午10:15触发              |  
| "0 15 10 * * ? 2005"      |    2005年的每天上午10:15触发 |  
| "0 * 14 * * ?"      |    在每天下午2点到下午2:59期间的每1分钟触发  |  
| "0 0/5 14 * * ?"|    在每天下午2点到下午2:55期间的每5分钟触发   |  
| "0 0/5 14,18 * * ?" | 在每天下午2点到2:55期间和下午6点到6:55期间的每5分钟触发 |  
| "0 0-5 14 * * ?" | 在每天下午2点到下午2:05期间的每1分钟触发 |  
| "0 10,44 14 ? 3 WED" | 每年三月的星期三的下午2:10和2:44触发 |  
| "0 15 10 ? * MON-FRI" | 周一至周五的上午10:15触发 |  
| "0 15 10 15 * ?" | 每月15日上午10:15触发 |  
| "0 15 10 L * ?" | 	每月最后一日的上午10:15触发 |  
| "0 15 10 ? * 6L" | 每月的最后一个星期五上午10:15触发  |  
| "0 15 10 ? * 6L 2002-2005" | 2002年至2005年的每月的最后一个星期五上午10:15触发  |  
| "0 15 10 ? * 6#3" | 每月的第三个星期五上午10:15触发  |  
| "0 6 * * * " | 每天早上6点  |  
| "0 */2 * * * " | 每两个小时  |  
| "0 23-7/2，8 * * *" | 晚上11点到早上8点之间每两个小时，早上八点  |  
| "0 11 4 * 1-3" | 每个月的4号和每个礼拜的礼拜一到礼拜三的早上11点  |  
| "0 4 1 1 *" | 月1日早上4点  |  

## 定时任务调用核心方法
```
package com.minsheng.reinsurance.service;

import com.minsheng.reinsurance.bean.entity.TaskInfo;
import com.minsheng.reinsurance.dao.TaskInfoDao;
import com.minsheng.reinsurance.enums.DeleteFlagEnum;
import org.apache.commons.lang.StringUtils;
import org.apache.commons.lang.time.DateFormatUtils;
import org.quartz.*;
import org.quartz.impl.matchers.GroupMatcher;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.*;

@Service
public class TaskService extends Services<TaskInfo> {
    private static final Logger LOGGER = LoggerFactory.getLogger(TaskService.class);

    private static final String path = "com.minsheng.reinsurance.task.";

    @Autowired
    private Scheduler scheduler;

    @Autowired
    TaskInfoDao taskInfoDao;

    /**
     * 校验实体类中各字段值
     * @param rtn
     * @param taskInfo
     * @return
     */
    public boolean validateTaskParams(HashMap<String, Object> rtn, TaskInfo taskInfo,String dos) {
        if (taskInfo != null) {
            String jobName = taskInfo.getJobName(),
                    jobGroup = taskInfo.getJobGroup(),
                    jobDescription = taskInfo.getJobDescription(),
                    jobStatus = taskInfo.getJobStatus(),
                    cronExpression = taskInfo.getCronExpression(),
                    startDate = taskInfo.getStartDate(),
                    endDate = taskInfo.getEndDate();
            if (StringUtils.isBlank(jobName) || StringUtils.isBlank(jobDescription) || StringUtils.isBlank(jobStatus)
                    || StringUtils.isBlank(cronExpression) || StringUtils.isBlank(startDate) ) {
                rtn.put("msg", "请填写完整");
                return false;
            }
            SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            format.setLenient(false);
            try {
                format.parse(startDate);
            } catch (ParseException e) {
                rtn.put("msg", "开始日期格式不正确");
                return false;
                //e.printStackTrace();
            }
            if(!StringUtils.isBlank(endDate)){
                try {
                    format.parse(endDate);
                } catch (ParseException e) {
                    rtn.put("msg", "结束日期格式不正确");
                    return false;
                    //e.printStackTrace();
                }
            }
            //当操作为添加任务时，需判断是否已存在相同任务
            if(dos.equals("add")){
                HashMap<String, Object> params = new HashMap<>();
                params.put("jobName", jobName);
                params.put("delFlag", DeleteFlagEnum.NORMAL.getCode());
                List<TaskInfo> taskInfoList = taskInfoDao.findBy(params);
                if(taskInfoList.size()>0){
                    rtn.put("msg", "该方法名称定时任务已存在，请先删除同名服务再进行添加操作");
                    return false;
                }
            }

            return true;
        }
        return false;
    }
    /**
     *
     * @Title: list
     * @Description: 任务列表
     * @param @return    参数
     * @return List<TaskInfo>    返回类型
     * @throws
     */
    public List<TaskInfo> queryJobList() {
	LOGGER.info("TaskService--data-s-->queryJobList()");
	List<TaskInfo> list = new ArrayList<>();
	try {
            for (String groupJob : scheduler.getJobGroupNames()) {
		for (JobKey jobKey : scheduler.getJobKeys(GroupMatcher.<JobKey> groupEquals(groupJob))) {
                    List<? extends Trigger> triggers = scheduler.getTriggersOfJob(jobKey);
                    for (Trigger trigger : triggers) {
			Trigger.TriggerState triggerState = scheduler.getTriggerState(trigger.getKey());
			JobDetail jobDetail = scheduler.getJobDetail(jobKey);
			String cronExpression = "";
			String createTime = "";
			String milliSeconds = "";
			String repeatCount = "";
			String startDate = "";
			String endDate = "";
			if (trigger instanceof CronTrigger) {
			CronTrigger cronTrigger = (CronTrigger) trigger;
			cronExpression = cronTrigger.getCronExpression();
			createTime = cronTrigger.getDescription();
			} else if (trigger instanceof SimpleTrigger) {
                            SimpleTrigger simpleTrigger = (SimpleTrigger) trigger;
                            milliSeconds = simpleTrigger.getRepeatInterval()+ "";
                            repeatCount = simpleTrigger.getRepeatCount() + "";
                            startDate = "2019-01-01";
                            endDate = "2019-12-01";;
			}
                        TaskInfo info = new TaskInfo();
                        info.setJobName(jobKey.getName());
                        info.setJobGroup(jobKey.getGroup());
                        info.setJobDescription(jobDetail.getDescription());
                        info.setJobStatus(triggerState.name());
                        info.setCronExpression(cronExpression);
                        info.setCreateTime(createTime);

                        info.setRepeatCount(repeatCount);
                        info.setStartDate(startDate);
                        info.setMilliSeconds(milliSeconds);
                        info.setEndDate(endDate);
                        list.add(info);
                    }
                }
            }
            LOGGER.info("任务的数量为：---------------->" + list.size());
	} catch (SchedulerException e) {
            LOGGER.info("查询任务失败，原因是：------------------>" + e.getMessage());
            e.printStackTrace();
	}
	return list;
}

    /**
     *
     * @Title: setSimpleTrigger
     * @Description: 简单调度
     * @param @param inputMap
     * @param @return    参数
     * @return Boolean    返回类型
     * @throws
     */
    @SuppressWarnings({ "unchecked" })
    public void setSimpleTriggerJob(TaskInfo info) {
        LOGGER.info("TaskService--data-s-->setSimpleTriggerJob()" + info);
        String jobName = path+info.getJobName();
        String jobGroup = info.getJobGroup();
        String jobDescription = info.getJobDescription();
        Long milliSeconds = Long.parseLong(info.getMilliSeconds());
        Integer repeatCount = Integer.parseInt(info.getRepeatCount());
        //Date start = DateUtils.parseDate(info.getStartDate());
        Date startDate = null;
        try {
            startDate = toDate("2019-01-01 00:00:00");
        } catch (ParseException e) {
            e.printStackTrace();
        }
        Date endDate = null;
        try {
            endDate = toDate("2019-12-01 00:00:00");
        } catch (ParseException e) {
            e.printStackTrace();
        }
        try {
            TriggerKey triggerKey = TriggerKey.triggerKey(jobName, jobGroup);// 触发器的key值
            JobKey jobKey = JobKey.jobKey(jobName, jobGroup);// job的key值
            if (checkExists(jobName, jobGroup)) {
                LOGGER.info(
                        "===> AddJob fail, job already exist, jobGroup:{}, jobName:{}",
                        jobGroup, jobName);
//                throw new ServiceException(String.format(
//                        "Job已经存在, jobName:{%s},jobGroup:{%s}", jobName,
//                        jobGroup));
            }
            /* 简单调度 */
            SimpleTrigger trigger = (SimpleTrigger) TriggerBuilder
                    .newTrigger()
                    .withIdentity(triggerKey)
                    .startAt(startDate)
                    .withSchedule(
                            SimpleScheduleBuilder.simpleSchedule()
                                    .withIntervalInMilliseconds(milliSeconds)
                                    .withRepeatCount(repeatCount))
                    .endAt(endDate).build();
            Class<? extends Job> clazz = (Class<? extends Job>) Class
                    .forName(jobName);
            JobDetail jobDetail = JobBuilder.newJob(clazz).withIdentity(jobKey)
                    .withDescription(jobDescription).build();
            System.out.println(scheduler+"====="+jobDetail+"======"+trigger);
            scheduler.scheduleJob(jobDetail, trigger);
        } catch (SchedulerException | ClassNotFoundException e) {
            LOGGER.info("任务添加失败！--->简单调度" + e.getMessage());
            //throw new ServiceException("任务添加失败！--->简单调度");
        }
    }

    /**
     *
     * @Title: addJob
     * @Description: 保存定时任务
     * @param @param info    参数
     * @return void    返回类型
     * @throws
     */
    @SuppressWarnings("unchecked")
    public void addJob(HashMap<String, Object> rtn,TaskInfo info) {
        LOGGER.info("TaskService--data-s-->addJob()" + info);

        //int i = taskInfoDao.insert(info);
        //System.out.println("=========I==:"+i);
        String jobName = path+info.getJobName(),
                jobGroup = info.getJobGroup(),
                cronExpression = info
                .getCronExpression(),
                jobDescription = info.getJobDescription(),
                createTime = DateFormatUtils
                .format(new Date(), "yyyy-MM-dd HH:mm:ss");
        String startDate1 = info.getStartDate();
        Date startDate = null;
        try {
            startDate = toDate(startDate1);
        } catch (ParseException e) {
            e.printStackTrace();
        }
        Date endDate = null;
        String endDate1 = info.getEndDate();
        //没有结束日期处理为2999-12-31 00:00:00
        if(!"".equals(endDate1)){
            try {
                endDate = toDate(endDate1);
            } catch (ParseException e) {
                e.printStackTrace();
            }
        }else{
            try {
                endDate = toDate("2999-12-31 00:00:00");
            } catch (ParseException e) {
                e.printStackTrace();
            }
        }

        try {
            if (checkExists(jobName, jobGroup)) {
                LOGGER.info(
                        "===> AddJob fail, job already exist, jobGroup:{}, jobName:{}",
                        jobGroup, jobName);
                System.out.println(String.format(
                        "Job已经存在, jobName:{%s},jobGroup:{%s}", jobName,
                        jobGroup));
            }

            TriggerKey triggerKey = TriggerKey.triggerKey(jobName, jobGroup);
            JobKey jobKey = JobKey.jobKey(jobName, jobGroup);

            CronScheduleBuilder schedBuilder = CronScheduleBuilder
                    .cronSchedule(cronExpression)
                    .withMisfireHandlingInstructionDoNothing();
            CronTrigger trigger = TriggerBuilder.newTrigger()
                    .withIdentity(triggerKey).startAt(startDate).withDescription(createTime)
                    .withSchedule(schedBuilder).endAt(endDate).build();

            Class<? extends Job> clazz = (Class<? extends Job>) Class
                    .forName(jobName);
            JobDetail jobDetail = JobBuilder.newJob(clazz).withIdentity(jobKey)
                    .withDescription(jobDescription).build();
            scheduler.scheduleJob(jobDetail, trigger);
            System.out.println("定时任务 启动成功");
        } catch (SchedulerException | ClassNotFoundException e) {
            LOGGER.info("保存定时任务-->类名不存在或执行表达式错误--->复杂调度" + e.getMessage());
            //throw new ServiceException("类名不存在或执行表达式错误");
        }
    }

    /**
     *
     * @Title: edit
     * @Description: 修改定时任务
     * @param @param info    参数
     * @return void    返回类型
     * @throws
     */
    public void editJob(TaskInfo info) {
        LOGGER.info("修改批处理" + info);
        String jobName = path+info.getJobName(),
                jobGroup = info.getJobGroup(),
                cronExpression = info.getCronExpression(),
                jobDescription = info.getJobDescription(),
                createTime = DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss");
        try {
            HashMap<String, Object> rtn = new HashMap<>();
            if (!checkExists(jobName, jobGroup)) {
                addJob(rtn,info);
//                System.out.println(
//                        String.format("Job不存在, jobName:{%s},jobGroup:{%s}",
//                                jobName, jobGroup));
            }else{
                TriggerKey triggerKey = TriggerKey.triggerKey(jobName, jobGroup);
                JobKey jobKey = new JobKey(jobName, jobGroup);
                CronScheduleBuilder cronScheduleBuilder = CronScheduleBuilder
                        .cronSchedule(cronExpression)
                        .withMisfireHandlingInstructionDoNothing();
                CronTrigger cronTrigger = TriggerBuilder.newTrigger()
                        .withIdentity(triggerKey).withDescription(createTime)
                        .withSchedule(cronScheduleBuilder).build();

                JobDetail jobDetail = scheduler.getJobDetail(jobKey);
                jobDetail.getJobBuilder().withDescription(jobDescription);
                HashSet<Trigger> triggerSet = new HashSet<>();
                triggerSet.add(cronTrigger);

                scheduler.scheduleJob(jobDetail, triggerSet, true);
            }

        } catch (SchedulerException e) {
            LOGGER.info("修改定时任务-->类名不存在或执行表达式错误--->复杂调度" + e.getMessage());
            //throw new ServiceException("类名不存在或执行表达式错误");
        }
    }

    /**
     *
     * @Title: delete
     * @Description: 删除定时任务
     * @param @param jobName
     * @param @param jobGroup    参数
     * @return void    返回类型
     * @throws
     */
    public void deleteJob(String jobName, String jobGroup) {
        LOGGER.info("TaskService--data-s-->deleteJob()--jobName:" + jobName
                + "jobGroup:" + jobGroup);
        jobName= path+jobName;
        TriggerKey triggerKey = TriggerKey.triggerKey(jobName, jobGroup);
        try {
            if (checkExists(jobName, jobGroup)) {
                scheduler.pauseTrigger(triggerKey);
                scheduler.unscheduleJob(triggerKey);
                LOGGER.info("===> delete, triggerKey:{}", triggerKey);
            }
        } catch (SchedulerException e) {
            LOGGER.info("删除定时任务-->复杂调度" + e.getMessage());
            //throw new ServiceException(e.getMessage());
        }
    }

    /**
     *
     * @Title: pause
     * @Description: 暂停定时任务
     * @param @param jobName
     * @param @param jobGroup    参数
     * @return void    返回类型
     * @throws
     */
    public void pauseJob(String jobName, String jobGroup) {
        System.out.println("TaskService--data-s-->pauseJob()--jobName:" + jobName
                + "jobGroup:" + jobGroup);
        TriggerKey triggerKey = TriggerKey.triggerKey(jobName, jobGroup);
        try {
            if (checkExists(jobName, jobGroup)) {
                scheduler.pauseTrigger(triggerKey);
                System.out.println("===> Pause success, triggerKey:{}"+triggerKey);
            }
        } catch (SchedulerException e) {
            LOGGER.info("暂停定时任务-->复杂调度" + e.getMessage());
            //throw new ServiceException(e.getMessage());
        }
    }

    /**
     *
     * @Title: resume
     * @Description: 恢复暂停任务
     * @param @param jobName
     * @param @param jobGroup    参数
     * @return void    返回类型
     * @throws
     */
    public void resumeJob(String jobName, String jobGroup) {
        LOGGER.info("TaskService--data-s-->resumeJob()--jobName:" + jobName
                + "jobGroup:" + jobGroup);
        jobName = path + jobName;
        TriggerKey triggerKey = TriggerKey.triggerKey(jobName, jobGroup);
        try {
            if (checkExists(jobName, jobGroup)) {
                scheduler.resumeTrigger(triggerKey);
                LOGGER.info("===> Resume success, triggerKey:{}", triggerKey);
            }
        } catch (SchedulerException e) {
            System.out.println("重新开始任务-->复杂调度" + e.getMessage());
            e.printStackTrace();
        }
    }

    /**
     *
     * @Title: checkExists
     * @Description: 验证任务是否存在
     * @param @param jobName
     * @param @param jobGroup
     * @param @return
     * @param @throws SchedulerException    参数
     * @return boolean    返回类型
     * @throws
     */
    private boolean checkExists(String jobName, String jobGroup)
            throws SchedulerException {
        System.out.println("TaskService--data-s-->checkExists()--jobName:" + jobName
                + "jobGroup:" + jobGroup);
        TriggerKey triggerKey = TriggerKey.triggerKey(jobName, jobGroup);
        return scheduler.checkExists(triggerKey);
    }

    public Date toDate(String date) throws ParseException {
        //String string = "2016-10-24 21:59:06";
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return sdf.parse(date);
    }
}

```
