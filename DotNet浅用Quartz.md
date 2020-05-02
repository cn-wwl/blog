一、使用Quartz.NET背景

最近在做一个抄表的项目，由于公司的抄表产品是个半成品，很多功能很简陋并且也不是那么的完善，但是既然接到手里了，自然想把功能做的更完整，代码能优化就优化。然后就偶遇代码`timer定时器`里写了一堆定时的逻辑处理，感觉有点乱；哈哈，然后就想着怎么把这一块的代码给优化一下。后来偶然看到`Quartz`，于是开始了研究啦。

二、Quartz.NET关键词介绍

- **Job**  表示要执行的任务内容，Job任务需要继承`IJob`接口，因此要需要实现IJob接口的`Execute(IJobExecutionContext context)`方法。
- **IJobDetail**  表示一个具体可执行的调度作业程序，要执行的 任务的内容定义好了，我们还需要通过`JobBuilder`创建并标识这个任务内容。
- **Trigger**   表示一个触发器，在什么时间去执行任务内容。
- **Scheduler** 代表一个调度容器，一个调度容器中可以注册多个 `JobDetail` 和 `Trigger`。当 `Trigger` 与 `JobDetail` 组合，就可以被 `Scheduler` 容器调度了。

三、使用Quartz.NET

1. 使用`Nuget`安装 Quartz 包，不同Quartz版本对`.NET Framework`框架的要求是不一样的，因此，我们在使用Quartz的时候，请注意自己的`.NET Framework`框架版本。

2. 创建Job任务作业

   ```C#
   public class RunTask : IJob
       {
           public void Execute(IJobExecutionContext context)
           {
               Console.WriteLine($"{DateTime.Now} 跑步");
           }
       }
   ```


3. 创建IJobDetail调度作业程序

   ```c#
   public class TaskJob
       {
       	/// <summary>
           /// 创建任务作业
           /// </summary>
           /// <typeparam name="T">Job</typeparam>
           /// <param name="jobName">任务名称</param>
           /// <param name="group">分组名称</param>
           /// <returns></returns>
           public IJobDetail CreateReadingMtrJob<T>(string jobName, string group) where T : IJob
           {
               IJobDetail job = JobBuilder.Create<T>().WithIdentity(jobName, group).Build();
               return job;
           }
       }
   ```

   where T ：泛型用法，此处用法为约束T的类型必须是继承IJob接口

   

4. 创建Trigger触发器，此处我们直接用的是cron表达式作为触发条件

   ```c#
   	/// <summary>
       /// Corn触发器
       /// </summary>
       public class CronTrigger
       {
           /// <summary>
           /// 创建 Corn触发器
           /// </summary>
           /// <param name="triggerName">触发器名称</param>
           /// <param name="group">分组名称</param>
           /// <param name="cron">触发条件</param>
           /// <returns></returns>
           public ICronTrigger CreateCronTrigger(string triggerName, string group, string cron)
           {
               ICronTrigger trigger = (ICronTrigger)TriggerBuilder.Create()
                      .WithIdentity(triggerName, group)
                      .WithCronSchedule(cron)
                      .Build();
               return trigger;
           }
   
       }
   ```

   [cron](https://cron.qqe2.com/ ) 是一种任务定时执行的时间表达式 ，程序可通过该公式 定时执行相应的任务作业。

5. 创建调度器工厂

   ```c#
   public class QuartzFactory
       {
           private StdSchedulerFactory factory = null;
   
           //调度器集合
           public Dictionary<string, IScheduler> schedulers = new Dictionary<string, IScheduler>();
   
           public QuartzFactory()
           {
               BuildSchedulerFactory();
           }
           private void BuildSchedulerFactory()
           {
               // 实例化调度器工厂
               factory = new StdSchedulerFactory();
           }
   
           /// <summary>
           /// 新增调度器
           /// </summary>
           /// <param name="name">调度器名称</param>
           public void AddScheduler(string name)
           {
               if (!schedulers.Any(a => a.Key.Equals(name)))
               {
                   schedulers.Add(name, factory.GetScheduler());
               }
           }
   
           /// <summary>
           /// 移除调度器
           /// </summary>
           /// <param name="name">调度器名称</param>
           public void RemoveScheduler(string name)
           {
               if (schedulers.Any(a => a.Key.Equals(name)))
               {
                   schedulers.Remove(name);
               }
           }
   
   
   
           /// <summary>
           /// 创建任务
           /// </summary>
           /// <typeparam name="T">Job包</typeparam>
           /// <param name="scheduler">调度器名称</param>
           /// <param name="name">任务名称</param>
           /// <param name="group">分组名称</param>
           /// <param name="cron">触发条件</param>
           public void AddTask<T>(string scheduler, string name, string group, string cron) where T : IJob
           {
               CronTrigger cronTrigger = new CronTrigger();
               var trigger = cronTrigger.CreateCronTrigger(name, group, cron);
   
               TaskJob mtrJob = new TaskJob();
               var job = mtrJob.CreateReadingMtrJob<T>(name, group);
   
               schedulers[scheduler].ScheduleJob(job, trigger);
           }
   
       }
   ```

   

6. 使用调度任务

   ```c#
   static void Main(string[] args)
           {
               //创建调度工厂
               QuartzFactory quartzFactory = new QuartzFactory();
   
               //添加调度器
               quartzFactory.AddScheduler("Life");
   
               //为指定调度器添加任务
               quartzFactory.AddTask<RunTask>("Life", "RunTask", "Bodybuilding", "0 0 6 1/1 * ? *");
               quartzFactory.AddTask<EyeExercisesTask>("Life", "EyeExercisesTask", "Bodybuilding", "0 0 19 1/1 * ? *");
   
               //启动调度器
               quartzFactory.schedulers["Life"].Start();
           }
   ```

   设计调度工厂的目的在于 一个项目可能由多个不同的系统组成，每个系统都可以使用独立的调度器；系统中调度任务的职责一样就可以把任务归到同一组，这样就可以通过调度工厂查看每个系统的调度任务啦。

四、总结

1. 遇到一个功能所使用的方式 比较繁琐时，想一想、查一查有没有好的方案去优化。
2. 尽量避免写一下面向过程的代码，多使用设计模式，否则你的code将会变成 `Shit mountain`。

五、源码

[下载](https://github.com/cn-wwl/Quartz.git)

