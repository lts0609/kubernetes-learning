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

## 抢占调度流程

### 抢占事件发生的时机

在代码中搜索关键字`Preempt`会找到路径`pkg/scheduler/framework/preemption`下的`Preempt()`方法，向上寻找调用关系可以到路径`pkg/scheduler/framework/plugins/defaultpreemption`，被其中`default_preemption.go`中的`PostFilter()`方法直接调用，可以说明抢占流程作为调度插件之一存在于`PostFilter`扩展点。`RunPostFilterPlugins()`方法的调用发生在`SchedulePod()`方法返回的错误信息不为空时，也就是`Predicates`与`Priorities`两个阶段存在失败，没有能找到一个符合条件的节点运行Pod，方法调用如下方代码内容所示。

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
    scheduleResult, err := sched.SchedulePod(ctx, fwk, state, pod)
    if err != nil{
        ......
        fitError, ok := err.(*framework.FitError)
        ......
        result, status := fwk.RunPostFilterPlugins(ctx, state, pod, fitError.Diagnosis.NodeToStatus)
        ......
    }
    ......
}
```

`RunPostFilterPlugins()`方法与`PostFilter()`方法之间的逻辑与其他种类插件一致，下面从分析`DefaultPreemption`实现的`PostFilter()`方法开始分析抢占的流程。

```Go
func (pl *DefaultPreemption) PostFilter(ctx context.Context, state *framework.CycleState, pod *v1.Pod, m framework.NodeToStatusReader) (*framework.PostFilterResult, *framework.Status) {
    defer func() {
        metrics.PreemptionAttempts.Inc()
    }()

    result, status := pl.Evaluator.Preempt(ctx, state, pod, m)
    msg := status.Message()
    if len(msg) > 0 {
        return result, framework.NewStatus(status.Code(), "preemption: "+msg)
    }
    return result, status
}
```

`PostFilter()`中调用了`Preempt()`方法，它是实际的抢占逻辑入口，在简单了解抢占的相关组件后，再正式开始深入分析抢占逻辑的实现。

### 关键组件

#### 评估器

在`kube-scheduler`中，`Evaluator `评估器是负责抢占流程的核心组件。在抢占中`Evaluator`负责流程控制和协调，具体的策略逻辑由实现了`Interface`接口的插件控制，`PreemptPod()`方法是抢占过程中的核心动作之一。

```Go
type Evaluator struct {
    // 插件名称
    PluginName string
    // 调度框架句柄
    Handler    framework.Handle
    // Pod Lister接口
    PodLister  corelisters.PodLister
    // PDB LIster接口
    PdbLister  policylisters.PodDisruptionBudgetLister
    // 异步抢占开关
    enableAsyncPreemption bool
    // 读写锁
    mu                    sync.RWMutex
    // 抢占流程中的Pod集合
    preempting sets.Set[types.UID]
    // 抢占逻辑函数
    PreemptPod func(ctx context.Context, c Candidate, preemptor, victim *v1.Pod, pluginName string) error
    // 插件接口
    Interface
}
```

`Interface`接口实现的方法如下，其中的方法实现在抢占流程中分析，此处不做说明。

```Go
type Interface interface {
    // 获取候选节点偏移量
    GetOffsetAndNumCandidates(nodes int32) (int32, int32)
    // 候选节点到受害Pod的映射
    CandidatesToVictimsMap(candidates []Candidate) map[string]*extenderv1.Victims
    // 判断Pod是否有抢占资格
    PodEligibleToPreemptOthers(ctx context.Context, pod *v1.Pod, nominatedNodeStatus *framework.Status) (bool, string)
    // 在候选节点上选择受害Pod
    SelectVictimsOnNode(ctx context.Context, state *framework.CycleState,
        pod *v1.Pod, nodeInfo *framework.NodeInfo, pdbs []*policy.PodDisruptionBudget) ([]*v1.Pod, int, *framework.Status)
    // 节点评分排序函数
    OrderedScoreFuncs(ctx context.Context, nodesToVictims map[string]*extenderv1.Victims) []func(node string) int64
}
```

##### 创建评估器

评估器实例的创建最早可以追溯到`NewInTreeRegistry()`函数，它实现了内置插件的创建和注册。插件结构包括调度框架句柄`fh`，特性开关集合`fts`，插件参数`args`，`Pod`和`PDB`的`Lister`以及评估器实例`Evaluator`。

```GO
func NewInTreeRegistry() runtime.Registry {
    ......
    registry := runtime.Registry{
        // 注册默认抢占插件
        defaultpreemption.Name:               runtime.FactoryAdapter(fts, defaultpreemption.New),
    }

    return registry
}

func New(_ context.Context, dpArgs runtime.Object, fh framework.Handle, fts feature.Features) (framework.Plugin, error) {
    args, ok := dpArgs.(*config.DefaultPreemptionArgs)
    if !ok {
        return nil, fmt.Errorf("got args of type %T, want *DefaultPreemptionArgs", dpArgs)
    }
    if err := validation.ValidateDefaultPreemptionArgs(nil, args); err != nil {
        return nil, err
    }

    podLister := fh.SharedInformerFactory().Core().V1().Pods().Lister()
    pdbLister := getPDBLister(fh.SharedInformerFactory())
    // 创建插件实例
    pl := DefaultPreemption{
        fh:        fh,
        fts:       fts,
        args:      *args,
        podLister: podLister,
        pdbLister: pdbLister,
    }
    pl.Evaluator = preemption.NewEvaluator(Name, fh, &pl, fts.EnableAsyncPreemption)

    return &pl, nil
}

// 创建评估器
func NewEvaluator(pluginName string, fh framework.Handle, i Interface, enableAsyncPreemption bool) *Evaluator {
    podLister := fh.SharedInformerFactory().Core().V1().Pods().Lister()
    pdbLister := fh.SharedInformerFactory().Policy().V1().PodDisruptionBudgets().Lister()

    ev := &Evaluator{
        PluginName:            names.DefaultPreemption,
        Handler:               fh,
        PodLister:             podLister,
        PdbLister:             pdbLister,
        Interface:             i,
        enableAsyncPreemption: enableAsyncPreemption,
        preempting:            sets.New[types.UID](),
    }
    // 注册PreemptPod()方法
    ev.PreemptPod = func(ctx context.Context, c Candidate, preemptor, victim *v1.Pod, pluginName string) error {
        ......
    }

    return ev
}
```

#### 提名器

在前面的学习中了解过调度队列的组件`Nominator`提名器，虽然它是`SchedulingQueue`的组件，但是和抢占流程也有一些关系。因为抢占插件返回的结构`framework.PostFilterResult`实际是一个`NominatingInfo`的指针，该结构的使用一般出现在提名器中。并且在抢占流程的最后，会修改Pod对象的`Status.NominatedNodeName`字段，在后面的调度周期中使用，如在`Predicates`阶段`findNodesThatFitPod()`方法的逻辑中，如果`Pod.Status.NominatedNodeName`不为空，会优先单独评估`NominatedNodeName`是否满足条件，如果不满足才会走后续标准流程。

```Go
type PostFilterResult struct {
    *NominatingInfo
}

type NominatingInfo struct {
    NominatedNodeName string
    NominatingMode    NominatingMode
}
```

### 抢占流程详解

前面已经做好了铺垫，在`PostFilter()`方法中执行`Preempt()`方法直接返回抢占的结果，所以从`pkg/scheduler/framework/preemption/preemption.go`路径的`Preempt()`方法开始详细的抢占流程分析。

根据源码注释可以看出，抢占大致分为六步：

* 第一步：通过`PodLister`获取Pod的最新信息状态；

* 第二步：确认这个Pod是否有资格进行抢占；

* 第三步：获取所有可发生抢占的候选节点；

* 第四步：调用注册的扩展器进一步缩小候选节点范围；

* 第五步：根据各种条件选择出最优的候选节点；

* 第六步：执行抢占，驱逐低优先级Pod；

这六个步骤执行后会返回抢占的最终结果给调度器，整体来看和标准的调度流程很类似，都会包括预选和优选的过程，实际上这些逻辑都紧紧围绕着调度器的本职工作：为Pod选择合适的`Node`，然后把`Node`的名称告诉Pod。

```Go
func (ev *Evaluator) Preempt(ctx context.Context, state *framework.CycleState, pod *v1.Pod, m framework.NodeToStatusReader) (*framework.PostFilterResult, *framework.Status) {
    logger := klog.FromContext(ctx)

    // 0) Fetch the latest version of <pod>.
    // It's safe to directly fetch pod here. Because the informer cache has already been
    // initialized when creating the Scheduler obj.
    // However, tests may need to manually initialize the shared pod informer.
    podNamespace, podName := pod.Namespace, pod.Name
    pod, err := ev.PodLister.Pods(pod.Namespace).Get(pod.Name)
    if err != nil {
        logger.Error(err, "Could not get the updated preemptor pod object", "pod", klog.KRef(podNamespace, podName))
        return nil, framework.AsStatus(err)
    }

    // 1) Ensure the preemptor is eligible to preempt other pods.
    nominatedNodeStatus := m.Get(pod.Status.NominatedNodeName)
    if ok, msg := ev.PodEligibleToPreemptOthers(ctx, pod, nominatedNodeStatus); !ok {
        logger.V(5).Info("Pod is not eligible for preemption", "pod", klog.KObj(pod), "reason", msg)
        return nil, framework.NewStatus(framework.Unschedulable, msg)
    }

    // 2) Find all preemption candidates.
    allNodes, err := ev.Handler.SnapshotSharedLister().NodeInfos().List()
    if err != nil {
        return nil, framework.AsStatus(err)
    }
    candidates, nodeToStatusMap, err := ev.findCandidates(ctx, state, allNodes, pod, m)
    if err != nil && len(candidates) == 0 {
        return nil, framework.AsStatus(err)
    }

    // Return a FitError only when there are no candidates that fit the pod.
    if len(candidates) == 0 {
        fitError := &framework.FitError{
            Pod:         pod,
            NumAllNodes: len(allNodes),
            Diagnosis: framework.Diagnosis{
                NodeToStatus: nodeToStatusMap,
                // Leave UnschedulablePlugins or PendingPlugins as nil as it won't be used on moving Pods.
            },
        }
        fitError.Diagnosis.NodeToStatus.SetAbsentNodesStatus(framework.NewStatus(framework.UnschedulableAndUnresolvable, "Preemption is not helpful for scheduling"))
        // Specify nominatedNodeName to clear the pod's nominatedNodeName status, if applicable.
        return framework.NewPostFilterResultWithNominatedNode(""), framework.NewStatus(framework.Unschedulable, fitError.Error())
    }

    // 3) Interact with registered Extenders to filter out some candidates if needed.
    candidates, status := ev.callExtenders(logger, pod, candidates)
    if !status.IsSuccess() {
        return nil, status
    }

    // 4) Find the best candidate.
    bestCandidate := ev.SelectCandidate(ctx, candidates)
    if bestCandidate == nil || len(bestCandidate.Name()) == 0 {
        return nil, framework.NewStatus(framework.Unschedulable, "no candidate node for preemption")
    }

    logger.V(2).Info("the target node for the preemption is determined", "node", bestCandidate.Name(), "pod", klog.KObj(pod))

    // 5) Perform preparation work before nominating the selected candidate.
    if ev.enableAsyncPreemption {
        ev.prepareCandidateAsync(bestCandidate, pod, ev.PluginName)
    } else {
        if status := ev.prepareCandidate(ctx, bestCandidate, pod, ev.PluginName); !status.IsSuccess() {
            return nil, status
        }
    }

    return framework.NewPostFilterResultWithNominatedNode(bestCandidate.Name()), framework.NewStatus(framework.Success)
}
```

下面对每一个阶段分别进行分析，Pod信息更新作为常规操作不多说明。

#### 抢占资格判断

通过`NodeToStatusReader`的`Get()`方法获取提名节点的状态信息记录，传递给`PodEligibleToPreemptOthers()`方法判断Pod是否有资格进行抢占操作。首先判断`Pod.Spec.PreemptionPolicy`中的抢占策略是否存在且不为`Never`，然后获取节点快照和Pod的`NominatedNodeName`，如果字段存在表示已经在之前进行过抢占流程了，现在又出现在`PostFilter`阶段表示抢占失败。此时需要判断这个Pod是否重新进行抢占操作，如果是`UnschedulableAndUnresolvable`表示提名节点因为某些问题出现了永久不可用的情况，允许开始重新抢占；如果是其他失败原因如`Unschedulable`则需要没有低于当前优先级的Pod正因抢占而退出才可以重新抢占，在这种因为临时资源导致抢占失败的情况下，为了避免资源浪费Pod还可以重试抢占尝试调度到其他节点。一般来说，抢占解决的是`Unschedulable`的问题，而`UnschedulableAndUnresolvable`的重试是上一次调度到当前调度周期之间发生了预期以外的变化，所以允许重新抢占。

回顾一下`Filter`扩展点的错误状态，`Unschedulable`表示临时的调度失败，如`CPU`资源不足、Pod间亲和性不满足等情况，可能下一轮调度情况变化就会可以调度了，这种情况不用人工干预只需要调度器重试；`UnschedulableAndUnresolvable`表示硬性条件的不满足，比如`NodeSelector`节点标签不满足、`PV`绑定失败等情况，其中临时条件和永久条件的关键区别在于**资源是否随着Pod的生命周期而发生变化**。

```Go
// 1) Ensure the preemptor is eligible to preempt other pods.
nominatedNodeStatus := m.Get(pod.Status.NominatedNodeName)
if ok, msg := ev.PodEligibleToPreemptOthers(ctx, pod, nominatedNodeStatus); !ok {
    logger.V(5).Info("Pod is not eligible for preemption", "pod", klog.KObj(pod), "reason", msg)
    return nil, framework.NewStatus(framework.Unschedulable, msg)
}

func (pl *DefaultPreemption) PodEligibleToPreemptOthers(_ context.Context, pod *v1.Pod, nominatedNodeStatus *framework.Status) (bool, string) {
    // 确认抢占策略开启
    if pod.Spec.PreemptionPolicy != nil && *pod.Spec.PreemptionPolicy == v1.PreemptNever {
        return false, "not eligible due to preemptionPolicy=Never."
    }
    // 从快照获取节点信息
    nodeInfos := pl.fh.SnapshotSharedLister().NodeInfos()
    nomNodeName := pod.Status.NominatedNodeName
    // 如果Pod已经经过一轮抢占计算并且有NominatedNodeName
    // 但是还在本轮调度的Filter阶段失败而走到了抢占流程
    if len(nomNodeName) > 0 {
        // 抢占失败 允许重试
        if nominatedNodeStatus.Code() == framework.UnschedulableAndUnresolvable {
            return true, ""
        }
        // 遍历提名节点的Pod
        if nodeInfo, _ := nodeInfos.Get(nomNodeName); nodeInfo != nil {
            podPriority := corev1helpers.PodPriority(pod)
            for _, p := range nodeInfo.Pods {
                if corev1helpers.PodPriority(p.Pod) < podPriority && podTerminatingByPreemption(p.Pod) {
                    // 如果有低于当前优先级的Pod且正因为抢占处于退出状态则不允许进行抢占
                    return false, "not eligible due to a terminating pod on the nominated node."
                }
            }
        }
    }
    return true, ""
}

```

