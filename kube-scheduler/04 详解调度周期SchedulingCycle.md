# 详解调度周期SchedulingCycle

调度周期的实现可以说是整个调度器的核心内容，所以展开说明。在`ScheduleOne()`中调度周期的入口方法是`schedulingCycl()`。

```Go
scheduleResult, assumedPodInfo, status := sched.schedulingCycle(schedulingCycleCtx, state, fwk, podInfo, start, podsToActivate)
```

根据函数签名部分，它接收六个参数，包括调度周期的上下文`ctx`，用于协程的生命周期管理；调度周期状态`state`，调度插件通过该对象读取或写入数据以便协同工作；调度框架接口对象`fwk`，调度过程中根据`Pod.Status.SchedulerName`字段调度框架获取对应`Framework`实例，对应调度流程中的配置以及扩展点插件列表；Pod的信息`podInfo`，根据其中的信息选择节点；开始调度的时间戳`start`；待激活Pod集合`podsToActivate`。

```Go
func (sched *Scheduler) schedulingCycle(
    ctx context.Context,
    state *framework.CycleState,
    fwk framework.Framework,
    podInfo *framework.QueuedPodInfo,
    start time.Time,
    podsToActivate *framework.PodsToActivate,
) (ScheduleResult, *framework.QueuedPodInfo, *framework.Status) 
```

然后来分析`schedulingCycle()`方法的实现逻辑，完整代码如下。

```Go
// schedulingCycle tries to schedule a single Pod.
func (sched *Scheduler) schedulingCycle(
    ctx context.Context,
    state *framework.CycleState,
    fwk framework.Framework,
    podInfo *framework.QueuedPodInfo,
    start time.Time,
    podsToActivate *framework.PodsToActivate,
) (ScheduleResult, *framework.QueuedPodInfo, *framework.Status) {
    logger := klog.FromContext(ctx)
    pod := podInfo.Pod
    scheduleResult, err := sched.SchedulePod(ctx, fwk, state, pod)
    if err != nil {
        defer func() {
            metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInSeconds(start))
        }()
        if err == ErrNoNodesAvailable {
            status := framework.NewStatus(framework.UnschedulableAndUnresolvable).WithError(err)
            return ScheduleResult{nominatingInfo: clearNominatedNode}, podInfo, status
        }

        fitError, ok := err.(*framework.FitError)
        if !ok {
            logger.Error(err, "Error selecting node for pod", "pod", klog.KObj(pod))
            return ScheduleResult{nominatingInfo: clearNominatedNode}, podInfo, framework.AsStatus(err)
        }

        if !fwk.HasPostFilterPlugins() {
            logger.V(3).Info("No PostFilter plugins are registered, so no preemption will be performed")
            return ScheduleResult{}, podInfo, framework.NewStatus(framework.Unschedulable).WithError(err)
        }

        // Run PostFilter plugins to attempt to make the pod schedulable in a future scheduling cycle.
        result, status := fwk.RunPostFilterPlugins(ctx, state, pod, fitError.Diagnosis.NodeToStatus)
        msg := status.Message()
        fitError.Diagnosis.PostFilterMsg = msg
        if status.Code() == framework.Error {
            logger.Error(nil, "Status after running PostFilter plugins for pod", "pod", klog.KObj(pod), "status", msg)
        } else {
            logger.V(5).Info("Status after running PostFilter plugins for pod", "pod", klog.KObj(pod), "status", msg)
        }

        var nominatingInfo *framework.NominatingInfo
        if result != nil {
            nominatingInfo = result.NominatingInfo
        }
        return ScheduleResult{nominatingInfo: nominatingInfo}, podInfo, framework.NewStatus(framework.Unschedulable).WithError(err)
    }

    metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInSeconds(start))

    assumedPodInfo := podInfo.DeepCopy()
    assumedPod := assumedPodInfo.Pod
    // assume modifies `assumedPod` by setting NodeName=scheduleResult.SuggestedHost
    err = sched.assume(logger, assumedPod, scheduleResult.SuggestedHost)
    if err != nil {
        return ScheduleResult{nominatingInfo: clearNominatedNode}, assumedPodInfo, framework.AsStatus(err)
    }

    // Run the Reserve method of reserve plugins.
    if sts := fwk.RunReservePluginsReserve(ctx, state, assumedPod, scheduleResult.SuggestedHost); !sts.IsSuccess() {
        // trigger un-reserve to clean up state associated with the reserved Pod
        fwk.RunReservePluginsUnreserve(ctx, state, assumedPod, scheduleResult.SuggestedHost)
        if forgetErr := sched.Cache.ForgetPod(logger, assumedPod); forgetErr != nil {
            logger.Error(forgetErr, "Scheduler cache ForgetPod failed")
        }

        if sts.IsRejected() {
            fitErr := &framework.FitError{
                NumAllNodes: 1,
                Pod:         pod,
                Diagnosis: framework.Diagnosis{
                    NodeToStatus: framework.NewDefaultNodeToStatus(),
                },
            }
            fitErr.Diagnosis.NodeToStatus.Set(scheduleResult.SuggestedHost, sts)
            fitErr.Diagnosis.AddPluginStatus(sts)
            return ScheduleResult{nominatingInfo: clearNominatedNode}, assumedPodInfo, framework.NewStatus(sts.Code()).WithError(fitErr)
        }
        return ScheduleResult{nominatingInfo: clearNominatedNode}, assumedPodInfo, sts
    }

    // Run "permit" plugins.
    runPermitStatus := fwk.RunPermitPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost)
    if !runPermitStatus.IsWait() && !runPermitStatus.IsSuccess() {
        // trigger un-reserve to clean up state associated with the reserved Pod
        fwk.RunReservePluginsUnreserve(ctx, state, assumedPod, scheduleResult.SuggestedHost)
        if forgetErr := sched.Cache.ForgetPod(logger, assumedPod); forgetErr != nil {
            logger.Error(forgetErr, "Scheduler cache ForgetPod failed")
        }

        if runPermitStatus.IsRejected() {
            fitErr := &framework.FitError{
                NumAllNodes: 1,
                Pod:         pod,
                Diagnosis: framework.Diagnosis{
                    NodeToStatus: framework.NewDefaultNodeToStatus(),
                },
            }
            fitErr.Diagnosis.NodeToStatus.Set(scheduleResult.SuggestedHost, runPermitStatus)
            fitErr.Diagnosis.AddPluginStatus(runPermitStatus)
            return ScheduleResult{nominatingInfo: clearNominatedNode}, assumedPodInfo, framework.NewStatus(runPermitStatus.Code()).WithError(fitErr)
        }

        return ScheduleResult{nominatingInfo: clearNominatedNode}, assumedPodInfo, runPermitStatus
    }

    // At the end of a successful scheduling cycle, pop and move up Pods if needed.
    if len(podsToActivate.Map) != 0 {
        sched.SchedulingQueue.Activate(logger, podsToActivate.Map)
        // Clear the entries after activation.
        podsToActivate.Map = make(map[string]*v1.Pod)
    }

    return scheduleResult, assumedPodInfo, nil
}
```

## SchedulePod阶段

因为选节点失败会触发抢占流程，先对可以成功选到节点的标准情况进行了解，也就是下面简化后的代码片段，做概述说明。首先调用`sched.SchedulePod()`方法，这个步骤可以说是整个调度周期的核心，其中包括了我们常说的预选`Predicates`和优选`Priorities`流程。回顾一下调度器实例的创建`sched.applyDefaultHandlers()`步骤中设置了`调度函数`和`调度失败handler`，没有采用硬编码的方式设置逻辑，提高了代码的灵活性。

```Go
func (sched *Scheduler) schedulingCycle(
    ctx context.Context,
    state *framework.CycleState,
    fwk framework.Framework,
    podInfo *framework.QueuedPodInfo,
    start time.Time,
    podsToActivate *framework.PodsToActivate,
) (ScheduleResult, *framework.QueuedPodInfo, *framework.Status) {
    logger := klog.FromContext(ctx)
    pod := podInfo.Pod
    // 核心逻辑 调度Pod 包含Predicates和Priorities全流程
    scheduleResult, err := sched.SchedulePod(ctx, fwk, state, pod)
    // 调度失败处理 暂且忽略
    ......

    metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInSeconds(start))
    // 避免影响到原始数据 深拷贝一个新对象 SchedulePod完成后Pod处于Assumed阶段
    assumedPodInfo := podInfo.DeepCopy()
    assumedPod := assumedPodInfo.Pod
    // 修改NodeName字段
    err = sched.assume(logger, assumedPod, scheduleResult.SuggestedHost)
    // 失败处理 暂且忽略
    ......

    // 运行资源预留插件 标准扩展点之一
    if sts := fwk.RunReservePluginsReserve(ctx, state, assumedPod, scheduleResult.SuggestedHost); !sts.IsSuccess() {
    // 资源预留插件失败处理(运行Unreserve) 暂且忽略
        ......
    }

    // 运行准入插件
    runPermitStatus := fwk.RunPermitPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost)
    if !runPermitStatus.IsWait() && !runPermitStatus.IsSuccess() {
        // 准入插件返回失败处理 完全同上
        ......
    }

    // 调度周期结束前 激活待激活Pod 
    if len(podsToActivate.Map) != 0 {
        sched.SchedulingQueue.Activate(logger, podsToActivate.Map)
        // 清理集合
        podsToActivate.Map = make(map[string]*v1.Pod)
    }
    // 返回结果
    return scheduleResult, assumedPodInfo, nil
}

```

分析一下`schedulePod()`方法的逻辑，

```Go
func (sched *Scheduler) schedulePod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
    // 创建一个调度过程中的跟踪器
    trace := utiltrace.New("Scheduling", utiltrace.Field{Key: "namespace", Value: pod.Namespace}, utiltrace.Field{Key: "name", Value: pod.Name})
    // 超过100ms打印日志
    defer trace.LogIfLong(100 * time.Millisecond)
    // 每次执行调度的开始会更新调度缓存
    // 对象的更新依据是Generation 缓存中节点是双向链表 当遍历到节点Generation小于当前快照Generation时退出能够提高效率
    if err := sched.Cache.UpdateSnapshot(klog.FromContext(ctx), sched.nodeInfoSnapshot); err != nil {
        return result, err
    }
    trace.Step("Snapshotting scheduler cache and node infos done")
    // 快照中没有节点时返回错误
    if sched.nodeInfoSnapshot.NumNodes() == 0 {
        return result, ErrNoNodesAvailable
    }
    // 核心函数 Predicates流程
    feasibleNodes, diagnosis, err := sched.findNodesThatFitPod(ctx, fwk, state, pod)
    if err != nil {
        return result, err
    }
    trace.Step("Computing predicates done")
    // 没找到可用节点返回错误
    if len(feasibleNodes) == 0 {
        return result, &framework.FitError{
            Pod:         pod,
            NumAllNodes: sched.nodeInfoSnapshot.NumNodes(),
            Diagnosis:   diagnosis,
        }
    }

    // 找到的可用节点只有一个直接选用 不需要后续Priorities流程
    if len(feasibleNodes) == 1 {
        return ScheduleResult{
            SuggestedHost:  feasibleNodes[0].Node().Name,
            EvaluatedNodes: 1 + diagnosis.NodeToStatus.Len(),
            FeasibleNodes:  1,
        }, nil
    }
  
    // 核心函数 Priorities流程
    priorityList, err := prioritizeNodes(ctx, sched.Extenders, fwk, state, pod, feasibleNodes)
    if err != nil {
        return result, err
    }
    // 节点打分后的最终选择
    host, _, err := selectHost(priorityList, numberOfHighestScoredNodesToReport)
    trace.Step("Prioritizing done")
    // 返回节点选择结果
    return ScheduleResult{
        SuggestedHost:  host,
        EvaluatedNodes: len(feasibleNodes) + diagnosis.NodeToStatus.Len(),
        FeasibleNodes:  len(feasibleNodes),
    }, err
}
```

### Predicates阶段

在`schedulePod()`方法中，`Predicates`阶段的入口代码如下，其中返回结果为`PodInfo`类型的列表`feasibleNodes`和节点不符合条件的原因`diagnosis`。

```Go
feasibleNodes, diagnosis, err := sched.findNodesThatFitPod(ctx, fwk, state, pod)
```

其中`Diagnosis`类型组成如下，记录全局的`Predicates`节点诊断信息。

```Go
type Diagnosis struct {
    // PreFilter/Filter阶段不可用节点的信息集合
    NodeToStatus *NodeToStatus
    // 使其返回UnschedulablePlugins或UnschedulableAndUnresolvable状态的插件集合信息
    UnschedulablePlugins sets.Set[string]
    // 使其返回Pending状态的插件集合信息
    PendingPlugins sets.Set[string]
    // PreFilter插件返回消息
    PreFilterMsg string
    // PostFilter插件返回消息
    PostFilterMsg string
}

type NodeToStatus struct {
    nodeToStatus map[string]*Status 
    // 插件返回失败后标记节点状态为Unschedulable/UnschedulableAndUnresolvable
    // 与后续抢占逻辑有关 抢占不会去尝试UnschedulableAndUnresolvable的节点
    absentNodesStatus *Status
}

type Status struct {
    code    Code
    reasons []string
    err     error
    plugin string
}
```

补充说明一下插件返回的内部状态，为枚举类型：

`Success`：插件执行成功；

`Error`：内部错误，立即入队重试；

`Unschedulable`：表示临时的不可调度，是动态(资源)的条件不满足，比如节点CPU资源不足，这种失败后Pod会重新入队等待调度，有退避时间；

`UnschedulableAndUnresolvable`是静态(配置)条件不满足，如要求节点上存在某个标签但实际不存在，调度器不会重试，对应事件如节点更新可能会触发重新调度；

`Wait`：仅和`Permit`插件有关，要求Pod进入等待状态；

`Skip`：跳过当前插件检查；

`Pending `是外部依赖条件不满足导致的等待，如存储卷未准备好或有依赖Pod的处理项未完成；