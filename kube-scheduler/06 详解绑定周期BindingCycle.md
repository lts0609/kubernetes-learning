# 详解绑定周期BindingCycle

上一节调度周期的流程已经完全结束了，整个Pod调度的生命周期已经结束了大半，回到`ScheduleOne()`方法来看，在调度周期`schedulingCycle()`返回了结果以后，如果是失败就`return`了，如果是成功就通过`go`关键字启动一个`Goroutine`协程来做绑定周期的逻辑，也相当于是这一次的`ScheduleOne()`调用结束了，但别忘了它是通过`UntilWithContext()`启动的间隔为`0s`的循环逻辑，表明调度周期结束后就立刻开始了下个Pod的调度计算，而绑定逻辑以及它的结果交给协程去处理，调度器的`ScheduleOne()`每次只能处理一个Pod，不会在不确定会耗时多久的绑定动作上去等待，包括`Permit`插件返回`Wait`后的结果也是在绑定周期的一开始去等待接收的，这十分符合效率优先的原则。

```Go
func (sched *Scheduler) ScheduleOne(ctx context.Context) {
    ......
    // 调度周期
    scheduleResult, assumedPodInfo, status := sched.schedulingCycle(schedulingCycleCtx, state, fwk, podInfo, start, podsToActivate)
    // 失败退出
    if !status.IsSuccess() {
        sched.FailureHandler(schedulingCycleCtx, fwk, assumedPodInfo, status, scheduleResult.nominatingInfo, start)
        return
    }

    // 或者通过协程处理绑定逻辑 不影响下一次调度的开始
    go func() {
        bindingCycleCtx, cancel := context.WithCancel(ctx)
        defer cancel()

        metrics.Goroutines.WithLabelValues(metrics.Binding).Inc()
        defer metrics.Goroutines.WithLabelValues(metrics.Binding).Dec()

        status := sched.bindingCycle(bindingCycleCtx, state, fwk, scheduleResult, assumedPodInfo, start, podsToActivate)
        if !status.IsSuccess() {
            sched.handleBindingCycleError(bindingCycleCtx, state, fwk, assumedPodInfo, start, scheduleResult, status)
            return
        }
    }()
}

```

