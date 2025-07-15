# 详解绑定周期BindingCycle

上一节调度周期的流程已经完全结束了，整个Pod调度的生命周期已经结束了大半，回到`ScheduleOne()`方法来看，在调度周期`schedulingCycle()`返回了结果以后，如果是失败就`return`了，如果是成功就通过`go`关键字启动一个`Goroutine`协程来做绑定周期的逻辑，也相当于是这一次的`ScheduleOne()`调用结束了，但别忘了它是通过`UntilWithContext()`启动的间隔为`0`的循环逻辑，表明调度周期结束后就立刻开始了下个Pod的调度计算，而绑定逻辑以及它的结果交给协程去处理，调度器的`ScheduleOne()`每次只能处理一个Pod，因此如果等待不确定会耗时多久的绑定动作结束才开始调度新Pod是不明智的，包括`Permit`插件返回`Wait`后的结果也是在绑定周期的一开始去等待接收的，这十分符合效率优先的原则。

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
        // 进入绑定生命周期
        status := sched.bindingCycle(bindingCycleCtx, state, fwk, scheduleResult, assumedPodInfo, start, podsToActivate)
        if !status.IsSuccess() {
            sched.handleBindingCycleError(bindingCycleCtx, state, fwk, assumedPodInfo, start, scheduleResult, status)
            return
        }
    }()
}
```

## 绑定周期逻辑

下面对绑定周期的逻辑进行分析，首先获取等待绑定Pod(`AssumedPod`)的对象，然后从等待队列的`channel`中读取数据并返回对应状态。源代码在调用`WaitOnPermit`方法时注释为`// Run "permit" plugins.`，此处注释是不准确的，并且可能会造成误导，实际的`Permit`插件调用点在`SchedulingCycle`周期的最后，个人认为此处应该视为`Permit`扩展点的状态接收点，不涉及插件逻辑的实际调用。整体上绑定周期涉及了半个扩展点的状态接收，和三个扩展点的执行，包括`PreBind`、`Bind`和`PostBind`。

在源码注释中的`PreBind`插件执行后，有一段注释值得注意，意为：任何此扩展点之后的失败都不会导致Pod被视为`Unschedulable`，只有当Pod在某些扩展点被拒绝时，才会将状态修改为`Unschedulable`，`PreBind`是这些中的最后一个扩展点。我们调用调度队列的`Done()`方法，可以尽早释放调度队列中存储的集群事件以优化内存消耗减轻集群压力。

这里相关的设计思想会在`PreBind`扩展点详细展开分析。

```Go
// Any failures after this point cannot lead to the Pod being considered unschedulable.
// We define the Pod as "unschedulable" only when Pods are rejected at specific extension points, and PreBind is the last one in the scheduling/binding cycle.
//
// We can call Done() here because
// we can free the cluster events stored in the scheduling queue sonner, which is worth for busy clusters memory consumption wise.
```

```Go
func (sched *Scheduler) bindingCycle(
    ctx context.Context,
    state *framework.CycleState,
    fwk framework.Framework,
    scheduleResult ScheduleResult,
    assumedPodInfo *framework.QueuedPodInfo,
    start time.Time,
    podsToActivate *framework.PodsToActivate) *framework.Status {
    logger := klog.FromContext(ctx)
    // 获取AssumedPod对象
    assumedPod := assumedPodInfo.Pod
    // 接收Permit插件返回状态
    if status := fwk.WaitOnPermit(ctx, assumedPod); !status.IsSuccess() {
        // 处理失败状态
        if status.IsRejected() {
            fitErr := &framework.FitError{
                NumAllNodes: 1,
                Pod:         assumedPodInfo.Pod,
                Diagnosis: framework.Diagnosis{
                    NodeToStatus:         framework.NewDefaultNodeToStatus(),
                    UnschedulablePlugins: sets.New(status.Plugin()),
                },
            }
            fitErr.Diagnosis.NodeToStatus.Set(scheduleResult.SuggestedHost, status)
            return framework.NewStatus(status.Code()).WithError(fitErr)
        }
        return status
    }

    // 执行PreBind插件
    if status := fwk.RunPreBindPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost); !status.IsSuccess() {
        // 处理失败状态
        if status.IsRejected() {
            fitErr := &framework.FitError{
                NumAllNodes: 1,
                Pod:         assumedPodInfo.Pod,
                Diagnosis: framework.Diagnosis{
                    NodeToStatus:         framework.NewDefaultNodeToStatus(),
                    UnschedulablePlugins: sets.New(status.Plugin()),
                },
            }
            fitErr.Diagnosis.NodeToStatus.Set(scheduleResult.SuggestedHost, status)
            return framework.NewStatus(status.Code()).WithError(fitErr)
        }
        return status
    }

    // Any failures after this point cannot lead to the Pod being considered unschedulable.
    // We define the Pod as "unschedulable" only when Pods are rejected at specific extension points, and PreBind is the last one in the scheduling/binding cycle.
    //
    // We can call Done() here because
    // we can free the cluster events stored in the scheduling queue sonner, which is worth for busy clusters memory consumption wise.
    sched.SchedulingQueue.Done(assumedPod.UID)

    // 执行Bind插件
    if status := sched.bind(ctx, fwk, assumedPod, scheduleResult.SuggestedHost, state); !status.IsSuccess() {
        return status
    }

    // Calculating nodeResourceString can be heavy. Avoid it if klog verbosity is below 2.
    logger.V(2).Info("Successfully bound pod to node", "pod", klog.KObj(assumedPod), "node", scheduleResult.SuggestedHost, "evaluatedNodes", scheduleResult.EvaluatedNodes, "feasibleNodes", scheduleResult.FeasibleNodes)
    metrics.PodScheduled(fwk.ProfileName(), metrics.SinceInSeconds(start))
    metrics.PodSchedulingAttempts.Observe(float64(assumedPodInfo.Attempts))
    if assumedPodInfo.InitialAttemptTimestamp != nil {
        metrics.PodSchedulingDuration.WithLabelValues(getAttemptsLabel(assumedPodInfo)).Observe(metrics.SinceInSeconds(*assumedPodInfo.InitialAttemptTimestamp))
        metrics.PodSchedulingSLIDuration.WithLabelValues(getAttemptsLabel(assumedPodInfo)).Observe(metrics.SinceInSeconds(*assumedPodInfo.InitialAttemptTimestamp))
    }
    // 执行PostBind插件
    fwk.RunPostBindPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost)

    // 调度结束前把待激活Pod加入ActiveQ
    if len(podsToActivate.Map) != 0 {
        sched.SchedulingQueue.Activate(logger, podsToActivate.Map)
    }

    return nil
}
```

### PreBind扩展点

`PreBind`插件调用入口和其他扩展点没有任何区别，查看`PreBind()`方法主要有`DynamicResources`和`VolumeBinding`两个插件实现，简单分析这两个插件在该阶段都实现了什么逻辑。首先`VolumeBinding`插件通过`CycleState`对象获取状态信息，然后判断是否所有需要的卷都已经和Pod进行了绑定，如果都已绑定则跳过。否则获取目标节点上的卷，然后调用`BindPodVolumes()`方法绑定Pod和卷。

```Go
func (pl *VolumeBinding) PreBind(ctx context.Context, cs *framework.CycleState, pod *v1.Pod, nodeName string) *framework.Status {
    // 获取状态信息
    s, err := getStateData(cs)
    if err != nil {
        return framework.AsStatus(err)
    }
    if s.allBound {
        // 所有卷都绑定直接返回
        return nil
    }
    // 获取节点上要与Pod绑定的卷
    podVolumes, ok := s.podVolumesByNode[nodeName]
    if !ok {
        return framework.AsStatus(fmt.Errorf("no pod volumes found for node %q", nodeName))
    }
    logger := klog.FromContext(ctx)
    logger.V(5).Info("Trying to bind volumes for pod", "pod", klog.KObj(pod))
    // 进行绑定
    err = pl.Binder.BindPodVolumes(ctx, pod, podVolumes)
    if err != nil {
        logger.V(5).Info("Failed to bind volumes for pod", "pod", klog.KObj(pod), "err", err)
        return framework.AsStatus(err)
    }
    logger.V(5).Info("Success binding volumes for pod", "pod", klog.KObj(pod))
    return nil
}
```

第二个插件`DynamicResources`处理动态资源如GPU在Pod绑定到节点前的分配，首先还是从`CycleState`获取状态信息，然后判断声明的资源是否已经在节点上被预留，如果没有被预留则执行`bindClaim()`方法，`bindClaim()`返回更新后的资源对象，实际上是更新了它的`ReservedFor`、`Allocation`和`Finalizers`字段，把返回的对象更新到调度上下文`CycleState`中。

```Go
func (pl *DynamicResources) PreBind(ctx context.Context, cs *framework.CycleState, pod *v1.Pod, nodeName string) *framework.Status {
    if !pl.enabled {
        return nil
    }
    // 获取状态信息
    state, err := getStateData(cs)
    if err != nil {
        return statusError(klog.FromContext(ctx), err)
    }
    if len(state.claims) == 0 {
        return nil
    }

    logger := klog.FromContext(ctx)
    // 遍历生命资源是否都已经被预留
    for index, claim := range state.claims {
        // 如果没有被预留执行处理逻辑
        if !resourceclaim.IsReservedForPod(pod, claim) {
            claim, err := pl.bindClaim(ctx, state, index, pod, nodeName)
            if err != nil {
                return statusError(logger, err)
            }
            // 更新资源状态
            state.claims[index] = claim
        }
    }
    return nil
}

func (pl *DynamicResources) bindClaim(ctx context.Context, state *stateData, index int, pod *v1.Pod, nodeName string) (patchedClaim *resourceapi.ResourceClaim, finalErr error) {
    logger := klog.FromContext(ctx)
    // 深拷贝动态资源
    claim := state.claims[index].DeepCopy()
    // 获取动态资源分配信息
    allocation := state.informationsForClaim[index].allocation
    defer func() {
        // 结束前清理分配状态
        if allocation != nil {
            if finalErr == nil {
                if err := pl.draManager.ResourceClaims().AssumeClaimAfterAPICall(claim); err != nil {
                    logger.V(5).Info("Claim not stored in assume cache", "err", finalErr)
                }
            }
            pl.draManager.ResourceClaims().RemoveClaimPendingAllocation(claim.UID)
        }
    }()

    logger.V(5).Info("preparing claim status update", "claim", klog.KObj(state.claims[index]), "allocation", klog.Format(allocation))
    // 资源版本比较
    refreshClaim := false
    // RetryOnConflict方法接收一个重试次数和匿名函数
    retryErr := retry.RetryOnConflict(retry.DefaultRetry, func() error {
        // 后续重试时如果标识位为true
        if refreshClaim {
            // 通过API获取最新资源对象
            updatedClaim, err := pl.clientset.ResourceV1beta1().ResourceClaims(claim.Namespace).Get(ctx, claim.Name, metav1.GetOptions{})
            if err != nil {
                return fmt.Errorf("get updated claim %s after conflict: %w", klog.KObj(claim), err)
            }
            logger.V(5).Info("retrying update after conflict", "claim", klog.KObj(claim))
            // 覆盖旧版本资源对象
            claim = updatedClaim
        } else {
            // 第一次执行先设置标识位为true
            refreshClaim = true
        }
        // 检查资源是否已经标记为删除
        if claim.DeletionTimestamp != nil {
            return fmt.Errorf("claim %s got deleted in the meantime", klog.KObj(claim))
        }
        // 处理资源分配结果
        if allocation != nil {
            if claim.Status.Allocation != nil {
                return fmt.Errorf("claim %s got allocated elsewhere in the meantime", klog.KObj(claim))
            }

            // 如果没有Finalizer
            if !slices.Contains(claim.Finalizers, resourceapi.Finalizer) {
                // 给资源对象添加Finalizer避免被意外删除
                claim.Finalizers = append(claim.Finalizers, resourceapi.Finalizer)
                // 通过API更新资源对象
                updatedClaim, err := pl.clientset.ResourceV1beta1().ResourceClaims(claim.Namespace).Update(ctx, claim, metav1.UpdateOptions{})
                if err != nil {
                    return fmt.Errorf("add finalizer to claim %s: %w", klog.KObj(claim), err)
                }
                // 更新资源对象
                claim = updatedClaim
            }
            // 把从CycleState中临时存储的分配决策赋给资源对象
            claim.Status.Allocation = allocation
        }

        // 添加到资源预留列表
        claim.Status.ReservedFor = append(claim.Status.ReservedFor, resourceapi.ResourceClaimConsumerReference{Resource: "pods", Name: pod.Name, UID: pod.UID})
        // 通过API更新资源对象
        updatedClaim, err := pl.clientset.ResourceV1beta1().ResourceClaims(claim.Namespace).UpdateStatus(ctx, claim, metav1.UpdateOptions{})
        if err != nil {
            if allocation != nil {
                return fmt.Errorf("add allocation and reservation to claim %s: %w", klog.KObj(claim), err)
            }
            return fmt.Errorf("add reservation to claim %s: %w", klog.KObj(claim), err)
        }
        // 更新资源对象
        claim = updatedClaim
        return nil
    })

    if retryErr != nil {
        return nil, retryErr
    }

    logger.V(5).Info("reserved", "pod", klog.KObj(pod), "node", klog.ObjectRef{Name: nodeName}, "resourceclaim", klog.Format(claim))
    return claim, nil
}
```

根据以上的了解，可以感受到几个扩展点之间的协作关系：

* `Reserve`阶段在内存中锁定资源并写入`CycleState`；
* `Permit`阶段验证资源的依赖是否就绪、资源是否未被占用；
* `PreBind`阶段把分配结果持久化到资源对象API；

`PreBind`与`Reserve`和`Permit`共同完成了从资源**临时锁定**到**许可绑定**再到**最终确认**的过程，与更早的`Filter/Score`完成了**静态匹配**到**动态确认**的过程。`PreBind`是绑定执行前的最终确认者，是最后一个失败后会导致Pod进入`Unschedulable`的扩展点，如果在其后的`Bind`阶段失败，通常会进行重试绑定的操作，而不会标记为`Unschedulable`而重新进行调度计算。

### Bind扩展点

