---
tags:
- spring
date: 2015-11-07T11:52:15+08:00
title: Spring Task Execution and Scheduling
url: 2015/11/07/spring-doc-scheduling/
---

#####28.1 介绍

&#160;&#160;&#160;&#160;
Spring为异步执行和任务调度提供了抽象接口，分别使用TaskExecutor和TaskScheduler。另外，Spring也提供了对线程池和CommonJ的支持。

&#160;&#160;&#160;&#160;
Spring集成了Timer和Quartz，这两种调度器都是使用FacotryBean分别地通过引用Timer或Trigger实例。此外，为了方便，可以直接调用一个目标对象的方法。

#####28.2 TaskExecutor

&#160;&#160;&#160;&#160;
Spring的TaskExecutor接口和JDK中的Executor是完全相同的，事实上，TaskExecutor只是对JDK5中线程池的抽象。当其他的组建需要使用线程池是，可以创建TaskExecutor对象来使用。

######28.2.1 TaskExecutor types

&#160;&#160;&#160;&#160;
Spring为TaskExecutor接口提供了很多实现类，大多数情况下，我们不用自己去实现。
SimpleAsyncTaskExecutor  不会重复使用线程，而是会为每一个调度启动新的线程。然而，它支持并发数限制，当超过最大并发数时会阻塞其他调度。
SyncTaskExecutor  不会异步执行，调度发生在调用线程中（也就是说没有新启线程）。
ConcurrentTaskExecutor  为了适配JDK中的Executor对象。

######28.2.2 Using a TaskExecutor



#####28.3 TaskScheduler

TaskScheduler提供了如下几个方法，我们只要关注下schedule(Runnable task, Trigger trigger)这个方法，关注Trigger这个对象。其他的几个方法和JDK中基本类似。

    public interface TaskScheduler {
        ScheduledFuture schedule(Runnable task, Trigger trigger);
        ScheduledFuture schedule(Runnable task, Date startTime);  
        ScheduledFuture scheduleAtFixedRate(Runnable task, Date startTime, long period);
        ScheduledFuture scheduleAtFixedRate(Runnable task, long period);
        ScheduledFuture scheduleWithFixedDelay(Runnable task, Date startTime, long delay);
        ScheduledFuture scheduleWithFixedDelay(Runnable task, long delay);
    }

下面我们看看Trigger接口

    public interface Trigger {
        Date nextExecutionTime(TriggerContext triggerContext);
    }

