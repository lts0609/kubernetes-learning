# 失败处理与抢占调度Preemption

## 调度的失败处理

在`ScheduleOne()`方法中可以看到调度器在两个位置进行了失败处理，不难想到这两处就是`SchedulingCycle`与`BindingCycle`的结尾，在两个生命周期结束时进行错误判断与处理。

```Go
func (sched *Scheduler) ScheduleOne(ctx context.Context) {
    ......

    scheduleResult, assumedPodInfo, status := sched.schedulingCycle(schedulingCycleCtx, state, fwk, podInfo, start, podsToActivate)
    if !status.IsSuccess() {
        sched.FailureHandler(schedulingCycleCtx, fwk, assumedPodInfo, status, scheduleResult.nominatingInfo, start)
        return
    }

    go func() {
        ......

        status := sched.bindingCycle(bindingCycleCtx, state, fwk, scheduleResult, assumedPodInfo, start, podsToActivate)
        if !status.IsSuccess() {
            sched.handleBindingCycleError(bindingCycleCtx, state, fwk, assumedPodInfo, start, scheduleResult, status)
            return
        }
    }()
}
```

失败处理接口`FailureHandler`是`FailureHandlerFn`类型的函数，在调度器创建时的`applyDefaultHandlers()`方法设置。

```Go
func (sched *Scheduler) applyDefaultHandlers() {
    sched.SchedulePod = sched.schedulePod
    sched.FailureHandler = sched.handleSchedulingFailure
}
```

下面来详细分析`handleSchedulingFailure()`方法内的逻辑。首先来看函数签名，该方法接收六个参数，分别是上下文信息`ctx`，调度配置`fwk`，Pod信息`podInfo`，返回状态`status`，提名信息`nominatingInfo`和调度起始时间`start`。整体上包括调度事件记录、Pod提名节点信息处理、Pod对象重新入队和Pod状态更新。

```Go
func (sched *Scheduler) handleSchedulingFailure(ctx context.Context, fwk framework.Framework, podInfo *framework.QueuedPodInfo, status *framework.Status, nominatingInfo *framework.NominatingInfo, start time.Time) {
    calledDone := false
    defer func() {
        if !calledDone {
            // 一般情况下AddUnschedulableIfNotPresent内部会调用SchedulingQueue.Done(pod.UID) 
            // 避免没有调用的特殊情况 正确释放Pod资源
            sched.SchedulingQueue.Done(podInfo.Pod.UID)
        }
    }()

    logger := klog.FromContext(ctx)
    // 初始化错误原因
    reason := v1.PodReasonSchedulerError
    if status.IsRejected() {
        // 如果状态是被拒绝表示不可调度
        reason = v1.PodReasonUnschedulable
    }
    // 记录指标
    switch reason {
    case v1.PodReasonUnschedulable:
        metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
    case v1.PodReasonSchedulerError:
        metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
    }

    // 获取失败信息
    pod := podInfo.Pod
    err := status.AsError()
    errMsg := status.Message()

    if err == ErrNoNodesAvailable {
        // 集群中没有Node注册
        logger.V(2).Info("Unable to schedule pod; no nodes are registered to the cluster; waiting", "pod", klog.KObj(pod))
    } else if fitError, ok := err.(*framework.FitError); ok {
        // 不符合条件被调度插件拒绝
        // 记录UnschedulablePlugins和PendingPlugins
        podInfo.UnschedulablePlugins = fitError.Diagnosis.UnschedulablePlugins
        podInfo.PendingPlugins = fitError.Diagnosis.PendingPlugins
        logger.V(2).Info("Unable to schedule pod; no fit; waiting", "pod", klog.KObj(pod), "err", errMsg)
    } else {
        // 其他内部错误
        logger.Error(err, "Error scheduling pod; retrying", "pod", klog.KObj(pod))
    }

    // 使用Lister获取最新信息并在其中查找当前Pod
    podLister := fwk.SharedInformerFactory().Core().V1().Pods().Lister()
    cachedPod, e := podLister.Pods(pod.Namespace).Get(pod.Name)
    if e != nil {
        logger.Info("Pod doesn't exist in informer cache", "pod", klog.KObj(pod), "err", e)
    } else {
        // 检查是否有NodeName信息
        if len(cachedPod.Spec.NodeName) != 0 {
            logger.Info("Pod has been assigned to node. Abort adding it back to queue.", "pod", klog.KObj(pod), "node", cachedPod.Spec.NodeName)
        } else {
            // 没有NodeName信息 把Pod深拷贝一份后重新入队
            podInfo.PodInfo, _ = framework.NewPodInfo(cachedPod.DeepCopy())
            if err := sched.SchedulingQueue.AddUnschedulableIfNotPresent(logger, podInfo, sched.SchedulingQueue.SchedulingCycle()); err != nil {
                logger.Error(err, "Error occurred")
            }
            calledDone = true
        }
    }

    // 尝试添加带有提名节点信息的Pod到Nominator
    if sched.SchedulingQueue != nil {
        sched.SchedulingQueue.AddNominatedPod(logger, podInfo.PodInfo, nominatingInfo)
    }

    if err == nil {
        return
    }
    // 记录事件
    msg := truncateMessage(errMsg)
    fwk.EventRecorder().Eventf(pod, nil, v1.EventTypeWarning, "FailedScheduling", "Scheduling", msg)
    // 更新Pod状态 包括提名节点信息和Condition
    if err := updatePod(ctx, sched.client, pod, &v1.PodCondition{
        Type:    v1.PodScheduled,
        Status:  v1.ConditionFalse,
        Reason:  reason,
        Message: errMsg,
    }, nominatingInfo); err != nil {
        logger.Error(err, "Error updating pod", "pod", klog.KObj(pod))
    }
}
```

## 抢占事件的发生