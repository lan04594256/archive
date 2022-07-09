#quartz 定时任务
## 参考项目:FEBS-Shiro
## [定时任务初始化SQL](quartz.sql)

##maven
```
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
     <version>2.3.2</version>
</dependency>
```
## 配置代码
```
package cc.mrbird.febs.job.configure;

import com.baomidou.dynamic.datasource.DynamicRoutingDataSource;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.quartz.SchedulerFactoryBean;

import javax.sql.DataSource;
import java.util.Properties;

/**
 * 定时任务配置
 *
 * @author MrBird
 */
@Configuration
@RequiredArgsConstructor
public class ScheduleConfigure {

    private final DynamicRoutingDataSource dynamicRoutingDataSource;

    @Bean
    public SchedulerFactoryBean schedulerFactoryBean() {
        SchedulerFactoryBean factory = new SchedulerFactoryBean();
        // 手动从多数据源中获取 quartz数据源
        DataSource quartz = dynamicRoutingDataSource.getDataSource("quartz");
        factory.setDataSource(quartz);

        // quartz参数
        Properties prop = new Properties();
        prop.put("org.quartz.scheduler.instanceName", "MyScheduler");
        prop.put("org.quartz.scheduler.instanceId", "AUTO");
        // 线程池配置
        prop.put("org.quartz.threadPool.class", "org.quartz.simpl.SimpleThreadPool");
        prop.put("org.quartz.threadPool.threadCount", "20");
        prop.put("org.quartz.threadPool.threadPriority", "5");
        // JobStore配置
        prop.put("org.quartz.jobStore.class", "org.quartz.impl.jdbcjobstore.JobStoreTX");
        // 集群配置
        prop.put("org.quartz.jobStore.isClustered", "true");
        prop.put("org.quartz.jobStore.clusterCheckinInterval", "15000");
        prop.put("org.quartz.jobStore.maxMisfiresToHandleAtATime", "1");

        prop.put("org.quartz.jobStore.misfireThreshold", "12000");
        prop.put("org.quartz.jobStore.tablePrefix", "QRTZ_");
        factory.setQuartzProperties(prop);

        factory.setSchedulerName("FEBS_Scheduler");
        // 延时启动
        factory.setStartupDelay(1);
        factory.setApplicationContextSchedulerContextKey("applicationContextKey");
        // 启动时更新己存在的 Job
        factory.setOverwriteExistingJobs(true);
        // 设置自动启动，默认为 true
        factory.setAutoStartup(true);

        return factory;
    }
}


```

## 页面相关接口代码
```
import cc.mrbird.febs.common.annotation.ControllerEndpoint;
import cc.mrbird.febs.common.controller.BaseController;
import cc.mrbird.febs.common.entity.FebsResponse;
import cc.mrbird.febs.common.entity.QueryRequest;
import cc.mrbird.febs.common.entity.Strings;
import cc.mrbird.febs.job.entity.Job;
import cc.mrbird.febs.job.service.IJobService;
import com.wuwenze.poi.ExcelKit;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.apache.shiro.authz.annotation.RequiresPermissions;
import org.quartz.CronExpression;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

import javax.servlet.http.HttpServletResponse;
import javax.validation.Valid;
import javax.validation.constraints.NotBlank;

/**
 * @author MrBird
 */
@Slf4j
@Validated
@RestController
@RequestMapping("job")
@RequiredArgsConstructor
public class JobController extends BaseController {

    private final IJobService jobService;

    @GetMapping
    @RequiresPermissions("job:view")
    public FebsResponse jobList(QueryRequest request, Job job) {
        return new FebsResponse().success()
                .data(getDataTable(jobService.findJobs(request, job)));
    }

    @GetMapping("cron/check")
    public boolean checkCron(String cron) {
        try {
            return CronExpression.isValidExpression(cron);
        } catch (Exception e) {
            return false;
        }
    }

    @PostMapping
    @RequiresPermissions("job:add")
    @ControllerEndpoint(operation = "新增定时任务", exceptionMessage = "新增定时任务失败")
    public FebsResponse addJob(@Valid Job job) {
        jobService.createJob(job);
        return new FebsResponse().success();
    }

    @GetMapping("delete/{jobIds}")
    @RequiresPermissions("job:delete")
    @ControllerEndpoint(operation = "删除定时任务", exceptionMessage = "删除定时任务失败")
    public FebsResponse deleteJob(@NotBlank(message = "{required}") @PathVariable String jobIds) {
        jobService.deleteJobs(StringUtils.split(jobIds, Strings.COMMA));
        return new FebsResponse().success();
    }

    @PostMapping("update")
    @ControllerEndpoint(operation = "修改定时任务", exceptionMessage = "修改定时任务失败")
    public FebsResponse updateJob(@Valid Job job) {
        jobService.updateJob(job);
        return new FebsResponse().success();
    }

    @GetMapping("run/{jobIds}")
    @RequiresPermissions("job:run")
    @ControllerEndpoint(operation = "执行定时任务", exceptionMessage = "执行定时任务失败")
    public FebsResponse runJob(@NotBlank(message = "{required}") @PathVariable String jobIds) {
        jobService.run(jobIds);
        return new FebsResponse().success();
    }

    @GetMapping("pause/{jobIds}")
    @RequiresPermissions("job:pause")
    @ControllerEndpoint(operation = "暂停定时任务", exceptionMessage = "暂停定时任务失败")
    public FebsResponse pauseJob(@NotBlank(message = "{required}") @PathVariable String jobIds) {
        jobService.pause(jobIds);
        return new FebsResponse().success();
    }

    @GetMapping("resume/{jobIds}")
    @RequiresPermissions("job:resume")
    @ControllerEndpoint(operation = "恢复定时任务", exceptionMessage = "恢复定时任务失败")
    public FebsResponse resumeJob(@NotBlank(message = "{required}") @PathVariable String jobIds) {
        jobService.resume(jobIds);
        return new FebsResponse().success();
    }

    @GetMapping("excel")
    @RequiresPermissions("job:export")
    @ControllerEndpoint(exceptionMessage = "导出Excel失败")
    public void export(QueryRequest request, Job job, HttpServletResponse response) {
        ExcelKit.$Export(Job.class, response)
                .downXlsx(jobService.findJobs(request, job).getRecords(), false);
    }
}

```
## 定时任务调用代码
``` 
package cc.mrbird.febs.job.utils;

import cc.mrbird.febs.common.entity.FebsConstant;
import cc.mrbird.febs.common.utils.SpringContextUtil;
import cc.mrbird.febs.job.entity.Job;
import cc.mrbird.febs.job.entity.JobLog;
import cc.mrbird.febs.job.service.IJobLogService;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.quartz.JobExecutionContext;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.scheduling.quartz.QuartzJobBean;

import java.util.Date;
import java.util.concurrent.Future;

/**
 * 定时任务
 *
 * @author MrBird
 */
@Slf4j
public class ScheduleJob extends QuartzJobBean {

    private final ThreadPoolTaskExecutor scheduleJobExecutorService = SpringContextUtil.getBean(FebsConstant.FEBS_SHIRO_THREAD_POOL, ThreadPoolTaskExecutor.class);

    @Override
    protected void executeInternal(JobExecutionContext context) {
        Job scheduleJob = (Job) context.getMergedJobDataMap().get(Job.JOB_PARAM_KEY);

        // 获取spring bean
        IJobLogService scheduleJobLogService = SpringContextUtil.getBean(IJobLogService.class);

        JobLog jobLog = new JobLog();
        jobLog.setJobId(scheduleJob.getJobId());
        jobLog.setBeanName(scheduleJob.getBeanName());
        jobLog.setMethodName(scheduleJob.getMethodName());
        jobLog.setParams(scheduleJob.getParams());
        jobLog.setCreateTime(new Date());

        long startTime = System.currentTimeMillis();

        try {
            // 执行任务
            log.info("任务准备执行，任务ID：{}", scheduleJob.getJobId());
            ScheduleRunnable task = new ScheduleRunnable(scheduleJob.getBeanName(), scheduleJob.getMethodName(),
                    scheduleJob.getParams());
            Future<?> future = scheduleJobExecutorService.submit(task);
            future.get();
            long times = System.currentTimeMillis() - startTime;
            jobLog.setTimes(times);
            // 任务状态 0：成功 1：失败
            jobLog.setStatus(JobLog.JOB_SUCCESS);

            log.info("任务执行完毕，任务ID：{} 总共耗时：{} 毫秒", scheduleJob.getJobId(), times);
        } catch (Exception e) {
            log.error("任务执行失败，任务ID：" + scheduleJob.getJobId(), e);
            long times = System.currentTimeMillis() - startTime;
            jobLog.setTimes(times);
            // 任务状态 0：成功 1：失败
            jobLog.setStatus(JobLog.JOB_FAIL);
            jobLog.setError(StringUtils.substring(e.toString(), 0, 2000));
        } finally {
            scheduleJobLogService.saveJobLog(jobLog);
        }
    }
}

```

