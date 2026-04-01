# Quatrz 简介
{docsify-updated}

Quartz是一个开源的作业调度框架，它完全由Java写成，并设计用于J2SE和J2EE应用中。它提供了巨大的灵 活性而不牺牲简单性。你能够用它来为执行一个作业而创建简单的或复杂的调度。它有很多特征，如：数据库支持，集群，插件，EJB作业预构 建，JavaMail及其它，支持cron-like表达式等等。

### Quartz 核心概念
Quartz 的几个核心概念，这样理解起 Quartz 的原理就会变得简单了:
1. `Job` 表示一个工作，要执行的具体内容。此接口中只有一个方法，如下：
   ```
   void execute(JobExecutionContext context)
   ```
2. `JobDetail` 表示一个具体的可执行的调度程序， `Job` 是这个可执行程调度程序所要执行的内容，另外 `JobDetail` 还包含了这个任务调度的方案和策略。
3. `Trigger` 代表一个调度参数的配置，什么时候去调。通常一个 `Trigger` 只能用于一个 `Job` ，但是一个 `Job` 可以被多个 `Trigger` 触发。
4. `Scheduler` 代表一个调度容器，一个调度容器中可以注册多个 `JobDetail` 和 `Trigger` 。当 `Trigger` 与 `JobDetail` 组合，就可以被 `Scheduler` 容器调度了。调度器使用一个线程池作为任务执行的基础设施，也就是每个任务都是用线程池来执行的。

```
try {
	// Grab the Scheduler instance from the Factory
	Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();
	// and start it off
	scheduler.start();

	JobDetail jobDetail = new JobDetailImpl("demo", PersonalChatRecordFetchJob.class);
	Trigger trigger = new CronTriggerImpl("demo", "default", "0 0/1 * * * ?");
	scheduler.scheduleJob(jobDetail, trigger);
    scheduler.shutdown();
} catch (SchedulerException | ParseException se) {
	se.printStackTrace();
}
```

### 集成 springboot
pom 添加相关依赖：
```
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.7.2</version>
	<relativePath/>
</parent>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

SpringBoot提供了一些便捷的方法来和 Quartz 协同工作，这些方法里面包括`spring-boot-starter-quartz`这个启动器。如果Quartz可用，Scheduler会通过 `SchedulerFactoryBean` 这个工厂bean自动配置到SpringBoot里。
`JobDetail` 、 `Calendar` 、 `Trigger` 这些类型的 bean 也会被自动采集并关联到Scheduler上。

```
@Configuration
public class QuatrzConfiguration {

    @Bean
    public JobDetail jobDetail() {
        JobDetail jobDetail = JobBuilder.newJob(PersonalChatRecordFetchJob.class)
                .withIdentity("start_of_day", "start_of_day")
                .storeDurably()
                .build();
        return jobDetail;
    }

    @Bean
    public Trigger trigger() {
        Trigger trigger = TriggerBuilder.newTrigger()
                .forJob(jobDetail())
                .withIdentity("start_of_day", "start_of_day")
                .startNow()
                // 每天0点执行
                .withSchedule(CronScheduleBuilder.cronSchedule("0 0/1 * * * ?"))
                .build();
        return trigger;
    }
}
```