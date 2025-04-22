# 详解调度周期SchedulingCycle(上)

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

#### 核心函数findNodesThatFitPod

```Go
func (sched *Scheduler) findNodesThatFitPod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) ([]*framework.NodeInfo, framework.Diagnosis, error) {
    logger := klog.FromContext(ctx)
    // 初始化节点级的诊断字典
    diagnosis := framework.Diagnosis{
        NodeToStatus: framework.NewDefaultNodeToStatus(),
    }
    // 获取全量节点列表
    allNodes, err := sched.nodeInfoSnapshot.NodeInfos().List()
    if err != nil {
        return nil, diagnosis, err
    }
    // 运行PreFilter扩展点的插件
    // 返回值为 过滤后的节点(全量为nil)、插件运行结果(Success/Error/Unschedulable/UnschedulableAndUnresolvable/Wait/Skip/Pending)、导致不可调度的插件集合
    preRes, s, unscheduledPlugins := fwk.RunPreFilterPlugins(ctx, state, pod)
    diagnosis.UnschedulablePlugins = unscheduledPlugins
    if !s.IsSuccess() {
        if !s.IsRejected() {
            // 如果是Error/Skip直接返回
            return nil, diagnosis, s.AsError()
        }
        // 更新节点不可用原因Unschedulable/UnschedulableAndUnresolvable 抢占相关
        diagnosis.NodeToStatus.SetAbsentNodesStatus(s)

        // 返回失败 组装错误信息
        msg := s.Message()
        diagnosis.PreFilterMsg = msg
        logger.V(5).Info("Status after running PreFilter plugins for pod", "pod", klog.KObj(pod), "status", msg)
        diagnosis.AddPluginStatus(s)
        return nil, diagnosis, nil
    }

    // 逻辑1 如果存在被提名节点字段 尝试处理被提名节点
    if len(pod.Status.NominatedNodeName) > 0 {
        // evaluateNominatedNode()方法内部调用了findNodesThatPassFilters()和findNodesThatPassExtenders()
        // 和逻辑2中实际逻辑一致
        feasibleNodes, err := sched.evaluateNominatedNode(ctx, pod, fwk, state, diagnosis)
        if err != nil {
            logger.Error(err, "Evaluation failed on nominated node", "pod", klog.KObj(pod), "node", pod.Status.NominatedNodeName)
        }
        // 有被提名节点切通过插件的情况返回成功结果
        if len(feasibleNodes) != 0 {
            return feasibleNodes, diagnosis, nil
        }
    }
  
    // 逻辑2 Pod信息不存在被提名节点字段 正常处理
    nodes := allNodes
    if !preRes.AllNodes() {
        nodes = make([]*framework.NodeInfo, 0, len(preRes.NodeNames))
        for nodeName := range preRes.NodeNames {
            if nodeInfo, err := sched.nodeInfoSnapshot.Get(nodeName); err == nil {
                nodes = append(nodes, nodeInfo)
            }
        }
        diagnosis.NodeToStatus.SetAbsentNodesStatus(framework.NewStatus(framework.UnschedulableAndUnresolvable, fmt.Sprintf("node(s) didn't satisfy plugin(s) %v", sets.List(unscheduledPlugins))))
    }
    // 运行Filter扩展点的插件
    feasibleNodes, err := sched.findNodesThatPassFilters(ctx, fwk, state, pod, &diagnosis, nodes)
    processedNodes := len(feasibleNodes) + diagnosis.NodeToStatus.Len()
    sched.nextStartNodeIndex = (sched.nextStartNodeIndex + processedNodes) % len(allNodes)
    if err != nil {
        return nil, diagnosis, err
    }

    feasibleNodesAfterExtender, err := findNodesThatPassExtenders(ctx, sched.Extenders, pod, feasibleNodes, diagnosis.NodeToStatus)
    if err != nil {
        return nil, diagnosis, err
    }
    if len(feasibleNodesAfterExtender) != len(feasibleNodes) {
        if diagnosis.UnschedulablePlugins == nil {
            diagnosis.UnschedulablePlugins = sets.New[string]()
        }
        diagnosis.UnschedulablePlugins.Insert(framework.ExtenderName)
    }

    return feasibleNodesAfterExtender, diagnosis, nil
}
```

#### PreFilter扩展点

`PreFilter`的作用主要是缩小集群范围，可能会收集集群/节点信息，一般会通过`cycleState.Write()`方法，以`扩展点+插件名`为key写入`CycleState`对象中。

```Go
func (f *frameworkImpl) RunPreFilterPlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod) (_ *framework.PreFilterResult, status *framework.Status, _ sets.Set[string]) {
    // 开始时间戳
    startTime := time.Now()
    // 已经在PreFilter扩展点处理过的插件跳过Filter扩展点的处理
    skipPlugins := sets.New[string]()
    defer func() {
        state.SkipFilterPlugins = skipPlugins
        metrics.FrameworkExtensionPointDuration.WithLabelValues(metrics.PreFilter, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
    }()
    // 初始化变量
    var result *framework.PreFilterResult
    pluginsWithNodes := sets.New[string]()
    logger := klog.FromContext(ctx)
    verboseLogs := logger.V(4).Enabled()
    if verboseLogs {
        logger = klog.LoggerWithName(logger, "PreFilter")
    }
    var returnStatus *framework.Status
    // 遍历当前扩展点的插件列表
    for _, pl := range f.preFilterPlugins {
        ctx := ctx
        if verboseLogs {
            logger := klog.LoggerWithName(logger, pl.Name())
            ctx = klog.NewContext(ctx, logger)
        }
        // 运行单个插件 返回结果和状态
        // PreFilter一般都是准备工作 所以基本上返回值PreFilterResult都是nil
        r, s := f.runPreFilterPlugin(ctx, pl, state, pod)
        if s.IsSkip() {
            skipPlugins.Insert(pl.Name())
            continue
        }
        // status不是Success时 返回或记录状态
        if !s.IsSuccess() {
            s.SetPlugin(pl.Name())
            if s.Code() == framework.UnschedulableAndUnresolvable {
                // 静态条件不会被满足 UnschedulableAndUnresolvable状态直接退出
                return nil, s, nil
            }
            if s.Code() == framework.Unschedulable {
                // 动态条件可能后面会被满足 Unschedulable状态可能触发抢占流程
                returnStatus = s
                continue
            }
            return nil, framework.AsStatus(fmt.Errorf("running PreFilter plugin %q: %w", pl.Name(), s.AsError())).WithPlugin(pl.Name()), nil
        }
        // 只有在PreFilter阶段缩小了节点范围 即返回不是全量的节点集合时
        if !r.AllNodes() {
            // 记录使节点范围缩小的插件
            pluginsWithNodes.Insert(pl.Name())
        }
        // 合并PreFilter结果
        result = result.Merge(r)
        // 取交集后节点集合为空 也就是PreFilter阶段筛掉了所有的节点
        if !result.AllNodes() && len(result.NodeNames) == 0 {
            msg := fmt.Sprintf("node(s) didn't satisfy plugin(s) %v simultaneously", sets.List(pluginsWithNodes))
            if len(pluginsWithNodes) == 1 {
                msg = fmt.Sprintf("node(s) didn't satisfy plugin %v", sets.List(pluginsWithNodes)[0])
            }
            return result, framework.NewStatus(framework.UnschedulableAndUnresolvable, msg), pluginsWithNodes
        }
    }
    return result, returnStatus, pluginsWithNodes
}
```

一个插件可以实现多个扩展点的接口，如`Fit`插件就同时实现了`PreFilter`、`Filter`和`Score`。

#### Filter扩展点

以有被提名节点的处理流程为例，与标准流程相同，`Filter`阶段的两个重要方法为`findNodesThatPassFilters()`和`findNodesThatPassExtenders`。

```Go
func (sched *Scheduler) evaluateNominatedNode(ctx context.Context, pod *v1.Pod, fwk framework.Framework, state *framework.CycleState, diagnosis framework.Diagnosis) ([]*framework.NodeInfo, error) {
    // 获取被提名Node的信息
    nnn := pod.Status.NominatedNodeName
    nodeInfo, err := sched.nodeInfoSnapshot.Get(nnn)
    if err != nil {
        return nil, err
    }
    // 准备节点列表
    node := []*framework.NodeInfo{nodeInfo}
    // 阶段1 运行Filter插件返回
    feasibleNodes, err := sched.findNodesThatPassFilters(ctx, fwk, state, pod, &diagnosis, node)
    if err != nil {
        return nil, err
    }
    // 阶段2 仍属于Filter扩展点 运行自定义扩展过滤插件
    feasibleNodes, err = findNodesThatPassExtenders(ctx, sched.Extenders, pod, feasibleNodes, diagnosis.NodeToStatus)
    if err != nil {
        return nil, err
    }
    // 返回feasibleNodes列表
    return feasibleNodes, nil
}
```

`findNodesThatPassFilters()`方法过滤出了通过`Filter`插件的节点列表。

```Go
func (sched *Scheduler) findNodesThatPassFilters(
    ctx context.Context,
    fwk framework.Framework,
    state *framework.CycleState,
    pod *v1.Pod,
    diagnosis *framework.Diagnosis,
    nodes []*framework.NodeInfo) ([]*framework.NodeInfo, error) {
    numAllNodes := len(nodes)
    // 提前确定要初步过滤出的节点数量
    numNodesToFind := sched.numFeasibleNodesToFind(fwk.PercentageOfNodesToScore(), int32(numAllNodes))
    // 如果调度周期没有其他扩展点 那节点数量必须是1
    if !sched.hasExtenderFilters() && !sched.hasScoring(fwk) {
        numNodesToFind = 1
    }

    // 初始化变量
    feasibleNodes := make([]*framework.NodeInfo, numNodesToFind)
    // 如果Filter扩展点没有插件 避免头部节点成为热点 从索引点拿够数量的Node就返回
    if !fwk.HasFilterPlugins() {
        for i := range feasibleNodes {
            feasibleNodes[i] = nodes[(sched.nextStartNodeIndex+i)%numAllNodes]
        }
        return feasibleNodes, nil
    }
    // 有Filter插件的标准流程
    errCh := parallelize.NewErrorChannel()
    var feasibleNodesLen int32
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    type nodeStatus struct {
        node   string
        status *framework.Status
    }
    result := make([]*nodeStatus, numAllNodes)
    // 定义checkNode函数 即每个节点的Filter逻辑
    checkNode := func(i int) {
        nodeInfo := nodes[(sched.nextStartNodeIndex+i)%numAllNodes]
        // 运行Filter插件
        status := fwk.RunFilterPluginsWithNominatedPods(ctx, state, pod, nodeInfo)
        if status.Code() == framework.Error {
            errCh.SendErrorWithCancel(status.AsError(), cancel)
            return
        }
        if status.IsSuccess() {
           // 计数先加一再判断
            length := atomic.AddInt32(&feasibleNodesLen, 1)
            if length > numNodesToFind {
                // 如果已经超过目标值 停止所有剩余节点的检查并会退计数
                cancel()
                atomic.AddInt32(&feasibleNodesLen, -1)
            } else {
                // 在范围内就正常添加节点
                feasibleNodes[length-1] = nodeInfo
            }
        } else {
            // 插件执行失败记录错误信息
            result[i] = &nodeStatus{node: nodeInfo.Node().Name, status: status}
        }
    }

    // 记录开始时间
    beginCheckNode := time.Now()
    statusCode := framework.Success
    defer func() {
        metrics.FrameworkExtensionPointDuration.WithLabelValues(metrics.Filter, statusCode.String(), fwk.ProfileName()).Observe(metrics.SinceInSeconds(beginCheckNode))
    }()

    // 使用并行器检查节点
    fwk.Parallelizer().Until(ctx, numAllNodes, checkNode, metrics.Filter)
    // 避免并发条件下的超额计数问题
    feasibleNodes = feasibleNodes[:feasibleNodesLen]
    // 聚合节点结果到diagnosis对象
    for _, item := range result {
        if item == nil {
            continue
        }
        diagnosis.NodeToStatus.Set(item.node, item.status)
        diagnosis.AddPluginStatus(item.status)
    }
    // 如过errCh收到错误返回筛选结果和错误
    if err := errCh.ReceiveError(); err != nil {
        statusCode = framework.Error
        return feasibleNodes, err
    }
    return feasibleNodes, nil
}
```

##### 节点列表长度确定规则

为了平衡调度的效率，不会把所有符合条件的节点都列出并打分，所以`feasiblenodes`切片会有一个预估长度，最小长度是100。`numFeasibleNodesToFind`抽样方法接收两个参数，分别是打分抽样百分比和集群节点总数。

```Go
func (sched *Scheduler) numFeasibleNodesToFind(percentageOfNodesToScore *int32, numAllNodes int32) (numNodes int32) {
    // 节点总数<100 直接返回节点总数
    if numAllNodes < minFeasibleNodesToFind {
        return numAllNodes
    }

    // 打分抽样百分比设置
    var percentage int32
    if percentageOfNodesToScore != nil {
        percentage = *percentageOfNodesToScore
    } else {
        // DefaultPercentageOfNodesToScore=0表示动态适应 在此处重新设置值
        percentage = sched.percentageOfNodesToScore
    }
    // 动态适应
    if percentage == 0 {
        // 抽样百分比也就是 50-(num/125) 最低百分比是5
        percentage = int32(50) - numAllNodes/125
        if percentage < minFeasibleNodesPercentageToFind {
            percentage = minFeasibleNodesPercentageToFind
        }
    }
    // 计算数量 总量*百分比 不小于是100
    numNodes = numAllNodes * percentage / 100
    if numNodes < minFeasibleNodesToFind {
        return minFeasibleNodesToFind
    }

    return numNodes
}
```

##### 运行Filter插件

`RunFilterPluginsWithNominatedPods()`是真正运行插件`RunFilterPlugins()`前的重要方法，也是Filter插件的上层入口函数。在之前先说明一个关键的点，`NominatedPod`是在抢占计算后产生的，会在后续调度循环中尝试调度。

此处的巧妙设计充分表现了调度器的**保守决策**，两次插件执行必须全部通过才算做节点可用：第一次循环先设置模拟添加提名Pod标识位`podsAdded`，然后调用`RunFilterPlugins()`执行`Filter`插件，第二次时判断标识位和上一次结果，如果没有提名Pod可模拟或模拟后节点不可用，直接返回不用再执行第二次。如果考虑了提名Pod且第一次结果为成功，那么还需要执行第二次，保证如Pod间亲和性等条件在没有提名Pod在节点上运行时仍然能够满足。如果第二次评估失败，那么会覆盖第一次的评估结果，认为当前节点不可用。

```Go
func (f *frameworkImpl) RunFilterPluginsWithNominatedPods(ctx context.Context, state *framework.CycleState, pod *v1.Pod, info *framework.NodeInfo) *framework.Status {
    var status *framework.Status

    podsAdded := false
    logger := klog.FromContext(ctx)
    logger = klog.LoggerWithName(logger, "FilterWithNominatedPods")
    ctx = klog.NewContext(ctx, logger)
    // 两次循环
    for i := 0; i < 2; i++ {
        stateToUse := state
        nodeInfoToUse := info
        // 第一轮循环假设带上NominatedPod一起评估
        if i == 0 {
            var err error
            podsAdded, stateToUse, nodeInfoToUse, err = addNominatedPods(ctx, f, pod, state, info)
            if err != nil {
                return framework.AsStatus(err)
            }
        } else if !podsAdded || !status.IsSuccess() {
            // i==1时才判断
            break
        }

        status = f.RunFilterPlugins(ctx, stateToUse, pod, nodeInfoToUse)
        if !status.IsSuccess() && !status.IsRejected() {
        // 两次循环中只要有一次结果是失败 就认为是失败的
            return status
        }
    }

    return status
}
```

`addNominatedPods()`函数把被提名到当前节点且优先级高于当前对象的Pod临时添加到节点信息中，以模拟成功抢占后的状态。

```Go
func addNominatedPods(ctx context.Context, fh framework.Handle, pod *v1.Pod, state *framework.CycleState, nodeInfo *framework.NodeInfo) (bool, *framework.CycleState, *framework.NodeInfo, error) {
    if fh == nil {
        return false, state, nodeInfo, nil
    }
    // 要在当前Node上抢占的Pod列表
    nominatedPodInfos := fh.NominatedPodsForNode(nodeInfo.Node().Name)
    if len(nominatedPodInfos) == 0 {
    // 如果没有就跳过这一步
        return false, state, nodeInfo, nil
    }
    // 如果有NominatedPod 拷贝一份快照和状态信息
    nodeInfoOut := nodeInfo.Snapshot()
    stateOut := state.Clone()
    podsAdded := false
    // 遍历NominatedPod列表
    for _, pi := range nominatedPodInfos {
        // 优先级比当前Pod高才会被尝试加入NodeInfo
        if corev1.PodPriority(pi.Pod) >= corev1.PodPriority(pod) && pi.Pod.UID != pod.UID {
            nodeInfoOut.AddPodInfo(pi)
            // 不是直接加入
            status := fh.RunPreFilterExtensionAddPod(ctx, stateOut, pod, pi, nodeInfoOut)
            if !status.IsSuccess() {
                // 任意提名Pod调度失败 表示节点原因抢占失败 一票否决
                return false, state, nodeInfo, status.AsError()
            }
            // 成功模拟后修改标识位
            podsAdded = true
        }
    }
    return podsAdded, stateOut, nodeInfoOut, nil
}
```

运行`Filter`插件的实际入口在`RunFilterPlugins()`方法中，和`PreFilter`的调用类似，都是循环执行集合中的插件然后返回状态，没有需要特别说明的地方。

```Go
func (f *frameworkImpl) RunFilterPlugins(
    ctx context.Context,
    state *framework.CycleState,
    pod *v1.Pod,
    nodeInfo *framework.NodeInfo,
) *framework.Status {
    logger := klog.FromContext(ctx)
    verboseLogs := logger.V(4).Enabled()
    if verboseLogs {
        logger = klog.LoggerWithName(logger, "Filter")
    }

    for _, pl := range f.filterPlugins {
        if state.SkipFilterPlugins.Has(pl.Name()) {
            continue
        }
        ctx := ctx
        if verboseLogs {
            logger := klog.LoggerWithName(logger, pl.Name())
            ctx = klog.NewContext(ctx, logger)
        }
        if status := f.runFilterPlugin(ctx, pl, state, pod, nodeInfo); !status.IsSuccess() {
            if !status.IsRejected() {
                status = framework.AsStatus(fmt.Errorf("running %q filter plugin: %w", pl.Name(), status.AsError()))
            }
            status.SetPlugin(pl.Name())
            return status
        }
    }

    return nil
}
```

回到之前的流程中，也就是`evaluateNominatedNode()`方法中，这里涉及到调度扩展器`Scheduler Extenders`，在此处先不详细说明

```Go
func findNodesThatPassExtenders(ctx context.Context, extenders []framework.Extender, pod *v1.Pod, feasibleNodes []*framework.NodeInfo, statuses *framework.NodeToStatus) ([]*framework.NodeInfo, error) {
    logger := klog.FromContext(ctx)

    // 遍历所有扩展器
    for _, extender := range extenders {
        if len(feasibleNodes) == 0 {
            break
        }
        if !extender.IsInterested(pod) {
            continue
        }

        // 调用扩展器的Filter方法
        // 返回可用Node列表、失败可重试节点列表、失败不可重试节点列表
        feasibleList, failedMap, failedAndUnresolvableMap, err := extender.Filter(pod, feasibleNodes)
        if err != nil {
            if extender.IsIgnorable() {
                logger.Info("Skipping extender as it returned error and has ignorable flag set", "extender", extender, "err", err)
                continue
            }
            return nil, err
        }
        // 状态写入
        for failedNodeName, failedMsg := range failedAndUnresolvableMap {
            statuses.Set(failedNodeName, framework.NewStatus(framework.UnschedulableAndUnresolvable, failedMsg))
        }

        for failedNodeName, failedMsg := range failedMap {
            if _, found := failedAndUnresolvableMap[failedNodeName]; found {
                // 两种失败都存在时 只记录UnschedulableAndUnresolvable状态就可以
                continue
            }
            statuses.Set(failedNodeName, framework.NewStatus(framework.Unschedulable, failedMsg))
        }
        // 更新最终节点列表
        feasibleNodes = feasibleList
    }
    return feasibleNodes, nil
}
```

至此`Predicates`阶段以返回一个`feasibleNodes`为结束，简单来看一下Pod中没有`NominatedNodeName`的标准情况，部分代码如下，并没有实际上的区别，基本相当于是未被封装的`evaluateNominatedNode()`

```Go
    // 初始化nodes为全部节点
    nodes := allNodes
    // 如果Prefilter阶段返回的不是全部节点
    // 就重新设置nodes为preRes中的节点
    if !preRes.AllNodes() {
        nodes = make([]*framework.NodeInfo, 0, len(preRes.NodeNames))
        for nodeName := range preRes.NodeNames {
            if nodeInfo, err := sched.nodeInfoSnapshot.Get(nodeName); err == nil {
                nodes = append(nodes, nodeInfo)
            }
        }
        // 记录诊断信息
        diagnosis.NodeToStatus.SetAbsentNodesStatus(framework.NewStatus(framework.UnschedulableAndUnresolvable, fmt.Sprintf("node(s) didn't satisfy plugin(s) %v", sets.List(unscheduledPlugins))))
    }
    // 同evaluateNominatedNode()中的调用流程
    feasibleNodes, err := sched.findNodesThatPassFilters(ctx, fwk, state, pod, &diagnosis, nodes)
    // 处理过的节点数量=通过数量+未通过数量
    processedNodes := len(feasibleNodes) + diagnosis.NodeToStatus.Len()
    // 更新下次选节点的起始索引
    sched.nextStartNodeIndex = (sched.nextStartNodeIndex + processedNodes) % len(allNodes)
    if err != nil {
        return nil, diagnosis, err
    }
    // 同evaluateNominatedNode()中的调用流程
    feasibleNodesAfterExtender, err := findNodesThatPassExtenders(ctx, sched.Extenders, pod, feasibleNodes, diagnosis.NodeToStatus)
    if err != nil {
        return nil, diagnosis, err
    }
    // 如果运行扩展过滤器的节点数量和之前不一样 更新不可调度插件集合
    if len(feasibleNodesAfterExtender) != len(feasibleNodes) {
        if diagnosis.UnschedulablePlugins == nil {
            diagnosis.UnschedulablePlugins = sets.New[string]()
        }
        diagnosis.UnschedulablePlugins.Insert(framework.ExtenderName)
    }

    return feasibleNodesAfterExtender, diagnosis, nil
```