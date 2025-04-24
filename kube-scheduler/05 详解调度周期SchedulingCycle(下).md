# 详解调度周期SchedulingCycle(下)

在上篇中详细介绍了调度周期的上半段，也就是`Predicates`过滤阶段，在其过程中返回了一个重要的对象`feasibleNodes`，它是一个`NodeInfo`类型的切片，保存了在条件过滤后符合Pod调度要求的节点信息。

回到`Predicates`结束的位置，也就是`schedulePod()`方法中

```Go
func (sched *Scheduler) schedulePod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
    ......
    // Predicates阶段 返回预选Node和诊断结果
    feasibleNodes, diagnosis, err := sched.findNodesThatFitPod(ctx, fwk, state, pod)
    ......
    // Case1:节点列表为空 返回失败
    if len(feasibleNodes) == 0 {
        // 上层函数只会关注err是否为空 result未被赋值
        return result, &framework.FitError{
            Pod:         pod,
            NumAllNodes: sched.nodeInfoSnapshot.NumNodes(),
            Diagnosis:   diagnosis,
        }
    }

    // Case2:列表长度为1 不需要Priorities阶段 直接走后续流程
    if len(feasibleNodes) == 1 {
        // 在Predicates阶段成功时会根据feasibleNodes填充result对象
        return ScheduleResult{
            SuggestedHost:  feasibleNodes[0].Node().Name,
            EvaluatedNodes: 1 + diagnosis.NodeToStatus.Len(),
            FeasibleNodes:  1,
        }, nil
    }
    // Case3:列表长度>1 需要Priorities阶段选出最合适的节点再返回
    priorityList, err := prioritizeNodes(ctx, sched.Extenders, fwk, state, pod, feasibleNodes)
    if err != nil {
        return result, err
    }

    host, _, err := selectHost(priorityList, numberOfHighestScoredNodesToReport)
    trace.Step("Prioritizing done")

    return ScheduleResult{
        SuggestedHost:  host,
        EvaluatedNodes: len(feasibleNodes) + diagnosis.NodeToStatus.Len(),
        FeasibleNodes:  len(feasibleNodes),
    }, err
}
```

### Priorities阶段

`Priorities`阶段的入口函数是`prioritizeNodes()`，对调度流程有过基本了解的一定都知道Pod的调度有`预选`和`优选`两个阶段，很明显在这个阶段要做的事情就是对上一步中过滤出来的节点进行排序，然后选择最合适的一个。

```Go
func prioritizeNodes(
    ctx context.Context,
    extenders []framework.Extender,
    fwk framework.Framework,
    state *framework.CycleState,
    pod *v1.Pod,
    nodes []*framework.NodeInfo,
) ([]framework.NodePluginScores, error) {
    logger := klog.FromContext(ctx)

    if len(extenders) == 0 && !fwk.HasScorePlugins() {
        result := make([]framework.NodePluginScores, 0, len(nodes))
        for i := range nodes {
            result = append(result, framework.NodePluginScores{
                Name:       nodes[i].Node().Name,
                TotalScore: 1,
            })
        }
        return result, nil
    }

    // Run PreScore plugins.
    preScoreStatus := fwk.RunPreScorePlugins(ctx, state, pod, nodes)
    if !preScoreStatus.IsSuccess() {
        return nil, preScoreStatus.AsError()
    }

    // Run the Score plugins.
    nodesScores, scoreStatus := fwk.RunScorePlugins(ctx, state, pod, nodes)
    if !scoreStatus.IsSuccess() {
        return nil, scoreStatus.AsError()
    }

    // Additional details logged at level 10 if enabled.
    loggerVTen := logger.V(10)
    if loggerVTen.Enabled() {
        for _, nodeScore := range nodesScores {
            for _, pluginScore := range nodeScore.Scores {
                loggerVTen.Info("Plugin scored node for pod", "pod", klog.KObj(pod), "plugin", pluginScore.Name, "node", nodeScore.Name, "score", pluginScore.Score)
            }
        }
    }

    if len(extenders) != 0 && nodes != nil {
        // allNodeExtendersScores has all extenders scores for all nodes.
        // It is keyed with node name.
        allNodeExtendersScores := make(map[string]*framework.NodePluginScores, len(nodes))
        var mu sync.Mutex
        var wg sync.WaitGroup
        for i := range extenders {
            if !extenders[i].IsInterested(pod) {
                continue
            }
            wg.Add(1)
            go func(extIndex int) {
                metrics.Goroutines.WithLabelValues(metrics.PrioritizingExtender).Inc()
                defer func() {
                    metrics.Goroutines.WithLabelValues(metrics.PrioritizingExtender).Dec()
                    wg.Done()
                }()
                prioritizedList, weight, err := extenders[extIndex].Prioritize(pod, nodes)
                if err != nil {
                    // Prioritization errors from extender can be ignored, let k8s/other extenders determine the priorities
                    logger.V(5).Info("Failed to run extender's priority function. No score given by this extender.", "error", err, "pod", klog.KObj(pod), "extender", extenders[extIndex].Name())
                    return
                }
                mu.Lock()
                defer mu.Unlock()
                for i := range *prioritizedList {
                    nodename := (*prioritizedList)[i].Host
                    score := (*prioritizedList)[i].Score
                    if loggerVTen.Enabled() {
                        loggerVTen.Info("Extender scored node for pod", "pod", klog.KObj(pod), "extender", extenders[extIndex].Name(), "node", nodename, "score", score)
                    }

                    // MaxExtenderPriority may diverge from the max priority used in the scheduler and defined by MaxNodeScore,
                    // therefore we need to scale the score returned by extenders to the score range used by the scheduler.
                    finalscore := score * weight * (framework.MaxNodeScore / extenderv1.MaxExtenderPriority)

                    if allNodeExtendersScores[nodename] == nil {
                        allNodeExtendersScores[nodename] = &framework.NodePluginScores{
                            Name:   nodename,
                            Scores: make([]framework.PluginScore, 0, len(extenders)),
                        }
                    }
                    allNodeExtendersScores[nodename].Scores = append(allNodeExtendersScores[nodename].Scores, framework.PluginScore{
                        Name:  extenders[extIndex].Name(),
                        Score: finalscore,
                    })
                    allNodeExtendersScores[nodename].TotalScore += finalscore
                }
            }(i)
        }
        // wait for all go routines to finish
        wg.Wait()
        for i := range nodesScores {
            if score, ok := allNodeExtendersScores[nodes[i].Node().Name]; ok {
                nodesScores[i].Scores = append(nodesScores[i].Scores, score.Scores...)
                nodesScores[i].TotalScore += score.TotalScore
            }
        }
    }

    if loggerVTen.Enabled() {
        for i := range nodesScores {
            loggerVTen.Info("Calculated node's final score for pod", "pod", klog.KObj(pod), "node", nodesScores[i].Name, "score", nodesScores[i].TotalScore)
        }
    }
    return nodesScores, nil
}
```

通过对`Kubernetes源码`的学习，可以感觉到它的代码是很结构化的，`prioritizeNodes()`函数整体较长，我们分块来理清它的逻辑。首先根据函数签名，它的入参包括：用于控制生命周期的上下文参数`ctx`，调度扩展器`extenders`，调度框架实例`fwk`，存储调度状态的`state`，Pod信息对象`pod`，以及上一步返回的节点列表`nodes`。返回值是一个`NodePluginScores`切片类型的节点评分结果，后续会经过`selectHost()`函数，最终敲定的目标节点并返回节点名称。

```Go
func prioritizeNodes(
    ctx context.Context,
    extenders []framework.Extender,
    fwk framework.Framework,
    state *framework.CycleState,
    pod *v1.Pod,
    nodes []*framework.NodeInfo,
) ([]framework.NodePluginScores, error)
```

先来看不包括调度扩展器的逻辑部分，如果没有调度扩展器和`Score`插件，就把所有的节点都打`1分`然后返回。

```Go
func prioritizeNodes(
    ctx context.Context,
    extenders []framework.Extender,
    fwk framework.Framework,
    state *framework.CycleState,
    pod *v1.Pod,
    nodes []*framework.NodeInfo,
) ([]framework.NodePluginScores, error) {
    logger := klog.FromContext(ctx)
    // 没有扩展器也没有评分插件 统一给一分并返回
    if len(extenders) == 0 && !fwk.HasScorePlugins() {
        result := make([]framework.NodePluginScores, 0, len(nodes))
        for i := range nodes {
            result = append(result, framework.NodePluginScores{
                Name:       nodes[i].Node().Name,
                TotalScore: 1,
            })
        }
        return result, nil
    }

    // PreScore扩展点
    preScoreStatus := fwk.RunPreScorePlugins(ctx, state, pod, nodes)
    if !preScoreStatus.IsSuccess() {
        return nil, preScoreStatus.AsError()
    }

    // Score扩展点
    nodesScores, scoreStatus := fwk.RunScorePlugins(ctx, state, pod, nodes)
    if !scoreStatus.IsSuccess() {
        return nil, scoreStatus.AsError()
    }

    // 详细日志输出
    loggerVTen := logger.V(10)
    if loggerVTen.Enabled() {
        for _, nodeScore := range nodesScores {
            for _, pluginScore := range nodeScore.Scores {
                loggerVTen.Info("Plugin scored node for pod", "pod", klog.KObj(pod), "plugin", pluginScore.Name, "node", nodeScore.Name, "score", pluginScore.Score)
            }
        }
    }
    ......
}
```

`RunPreScorePlugins()`的实现和`RunPreFilterPlugins()`非常类似，同样是做了两大类事情：在遍历执行`PreScore`插件的过程中，调用`cycleState.Write()`记录信息到`cycleState`中，如污点容忍和亲和性等，并在遍历结束后把没有相关条件后续不需要执行的`Score`插件也记录到`cycleState`。

```Go
func (f *frameworkImpl) RunPreScorePlugins(
    ctx context.Context,
    state *framework.CycleState,
    pod *v1.Pod,
    nodes []*framework.NodeInfo,
) (status *framework.Status) {
    startTime := time.Now()
    skipPlugins := sets.New[string]()
    // 最后把Score阶段不需要执行的插件记录到cycleState
    defer func() {
        state.SkipScorePlugins = skipPlugins
        metrics.FrameworkExtensionPointDuration.WithLabelValues(metrics.PreScore, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
    }()
    logger := klog.FromContext(ctx)
    verboseLogs := logger.V(4).Enabled()
    if verboseLogs {
        logger = klog.LoggerWithName(logger, "PreScore")
    }
    // 遍历执行PreScore插件逻辑
    for _, pl := range f.preScorePlugins {
        ctx := ctx
        if verboseLogs {
            logger := klog.LoggerWithName(logger, pl.Name())
            ctx = klog.NewContext(ctx, logger)
        }
        status = f.runPreScorePlugin(ctx, pl, state, pod, nodes)
        if status.IsSkip() {
            skipPlugins.Insert(pl.Name())
            continue
        }
        if !status.IsSuccess() {
            return framework.AsStatus(fmt.Errorf("running PreScore plugin %q: %w", pl.Name(), status.AsError()))
        }
    }
    return nil
}
```

以Kubernetes的默认插件配置为例，分析`PreScore`和`Score`插件的行为，由于`Weight`权重字段一定是作用于`Score`相关的扩展点，所以选择`TaintToleration`插件作为分析对象。

```Go
func getDefaultPlugins() *v1.Plugins {
    plugins := &v1.Plugins{
        MultiPoint: v1.PluginSet{
            Enabled: []v1.Plugin{
                {Name: names.SchedulingGates},
                {Name: names.PrioritySort},
                {Name: names.NodeUnschedulable},
                {Name: names.NodeName},
                {Name: names.TaintToleration, Weight: ptr.To[int32](3)},
                {Name: names.NodeAffinity, Weight: ptr.To[int32](2)},
                {Name: names.NodePorts},
                {Name: names.NodeResourcesFit, Weight: ptr.To[int32](1)},
                {Name: names.VolumeRestrictions},
                {Name: names.NodeVolumeLimits},
                {Name: names.VolumeBinding},
                {Name: names.VolumeZone},
                {Name: names.PodTopologySpread, Weight: ptr.To[int32](2)},
                {Name: names.InterPodAffinity, Weight: ptr.To[int32](2)},
                {Name: names.DefaultPreemption},
                {Name: names.NodeResourcesBalancedAllocation, Weight: ptr.To[int32](1)},
                {Name: names.ImageLocality, Weight: ptr.To[int32](1)},
                {Name: names.DefaultBinder},
            },
        },
    }
    applyFeatureGates(plugins)

    return plugins
}
```

在经过从`算法硬编码`到`Scheduler Framework`的重构后，插件都在路径`pkg/scheduler/framework/plugins`下定义。`TaintToleration`插件可以在该路径下找到，一般每个插件目录下都是由算法实现和单元测试两个文件组成。

在`taint_toleration.go`文件中，可以看到该插件实现了`Filter、PreScore、Score、NormalizeScore`接口，其中的逻辑比较简单，`PreScore`扩展点时使用`cycleState.Write()`记录污点容忍信息到`cycleState`，到`Score`扩展点时`TaintToleration`根据写入时的key使用`cycleState.Read()`读出`PreScore`阶段记录的数据，如果不能容忍软性污点就计数加一，最后返回结果。`NormalizeScore`扩展点调用了默认的归一化逻辑。

```Go
func (pl *TaintToleration) PreScore(ctx context.Context, cycleState *framework.CycleState, pod *v1.Pod, nodes []*framework.NodeInfo) *framework.Status {
    if len(nodes) == 0 {
        return nil
    }
    tolerationsPreferNoSchedule := getAllTolerationPreferNoSchedule(pod.Spec.Tolerations)
    state := &preScoreState{
        tolerationsPreferNoSchedule: tolerationsPreferNoSchedule,
    }
    cycleState.Write(preScoreStateKey, state)
    return nil
}

func (pl *TaintToleration) Score(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeName string) (int64, *framework.Status) {
    nodeInfo, err := pl.handle.SnapshotSharedLister().NodeInfos().Get(nodeName)
    if err != nil {
        return 0, framework.AsStatus(fmt.Errorf("getting node %q from Snapshot: %w", nodeName, err))
    }
    node := nodeInfo.Node()

    s, err := getPreScoreState(state)
    if err != nil {
        return 0, framework.AsStatus(err)
    }

    score := int64(countIntolerableTaintsPreferNoSchedule(node.Spec.Taints, s.tolerationsPreferNoSchedule))
    return score, nil
}

func (pl *TaintToleration) NormalizeScore(ctx context.Context, _ *framework.CycleState, pod *v1.Pod, scores framework.NodeScoreList) *framework.Status {
    return helper.DefaultNormalizeScore(framework.MaxNodeScore, true, scores)
}

func countIntolerableTaintsPreferNoSchedule(taints []v1.Taint, tolerations []v1.Toleration) (intolerableTaints int) {
    for _, taint := range taints {
        // 仅处理软性污点
        if taint.Effect != v1.TaintEffectPreferNoSchedule {
            continue
        }
        // 不能容忍该污点时计数+1
        if !v1helper.TolerationsTolerateTaint(tolerations, &taint) {
            intolerableTaints++
        }
    }
    // 返回不能容忍污点计数结果
    return
}
```

#### Score扩展点与NormalizeScore评分归一化

`RunScorePlugins()`的实现和`RunFilterPlugins()`类似。首先，`Filter`插件执行的过程中，状态被存储在`CycleState`对象并读写，过滤的行为属于责任链模式，如果一处不通过就直接失败退出，所以过滤阶段节点层面并行但插件层面是串行的。评分阶段也是节点层面串行和插件层面并行，如有疑问可以对比`Predicates`阶段的`findNodesThatPassFilters()`与`Priorities`阶段的`RunScorePlugins()`方法并加以详细对比。

```Go
func (f *frameworkImpl) RunScorePlugins(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodes []*framework.NodeInfo) (ns []framework.NodePluginScores, status *framework.Status) {
    startTime := time.Now()
    defer func() {
        metrics.FrameworkExtensionPointDuration.WithLabelValues(metrics.Score, status.Code().String(), f.profileName).Observe(metrics.SinceInSeconds(startTime))
    }()
    allNodePluginScores := make([]framework.NodePluginScores, len(nodes))
    numPlugins := len(f.scorePlugins)
    plugins := make([]framework.ScorePlugin, 0, numPlugins)
    pluginToNodeScores := make(map[string]framework.NodeScoreList, numPlugins)
    // 初始化Score扩展点使用的插件列表
    for _, pl := range f.scorePlugins {
        if state.SkipScorePlugins.Has(pl.Name()) {
            continue
        }
        plugins = append(plugins, pl)
        pluginToNodeScores[pl.Name()] = make(framework.NodeScoreList, len(nodes))
    }
    // 创建上下文对象
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    errCh := parallelize.NewErrorChannel()
    // Score插件列表不为空时
    if len(plugins) > 0 {
        logger := klog.FromContext(ctx)
        verboseLogs := logger.V(4).Enabled()
        if verboseLogs {
            logger = klog.LoggerWithName(logger, "Score")
        }
        // 为每个节点并发执行评分操作
        f.Parallelizer().Until(ctx, len(nodes), func(index int) {
            nodeName := nodes[index].Node().Name
            logger := logger
            if verboseLogs {
                logger = klog.LoggerWithValues(logger, "node", klog.ObjectRef{Name: nodeName})
            }
            // Score插件级别串行
            for _, pl := range plugins {
                ctx := ctx
                if verboseLogs {
                    logger := klog.LoggerWithName(logger, pl.Name())
                    ctx = klog.NewContext(ctx, logger)
                }
                s, status := f.runScorePlugin(ctx, pl, state, pod, nodeName)
                if !status.IsSuccess() {
                    err := fmt.Errorf("plugin %q failed with: %w", pl.Name(), status.AsError())
                    errCh.SendErrorWithCancel(err, cancel)
                    return
                }
                // 记录评分信息到pluginToNodeScores
                pluginToNodeScores[pl.Name()][index] = framework.NodeScore{
                    Name:  nodeName,
                    Score: s,
                }
            }
        }, metrics.Score)
        if err := errCh.ReceiveError(); err != nil {
            return nil, framework.AsStatus(fmt.Errorf("running Score plugins: %w", err))
        }
    }

    // NormalizeScore扩展点
    // 为每个插件并发执行评分归一化
    f.Parallelizer().Until(ctx, len(plugins), func(index int) {
        pl := plugins[index]
        // Score插件必须实现ScoreExtensions()方法 如果不需要归一化就在该方法中返回nil
        if pl.ScoreExtensions() == nil {
            return
        }
        nodeScoreList := pluginToNodeScores[pl.Name()]
        // 有归一化需要的执行插件的NormalizeScore()方法并返回结果
        status := f.runScoreExtension(ctx, pl, state, pod, nodeScoreList)
        if !status.IsSuccess() {
            err := fmt.Errorf("plugin %q failed with: %w", pl.Name(), status.AsError())
            errCh.SendErrorWithCancel(err, cancel)
            return
        }
    }, metrics.Score)
    if err := errCh.ReceiveError(); err != nil {
        return nil, framework.AsStatus(fmt.Errorf("running Normalize on Score plugins: %w", err))
    }

    // 按节点粒度并发执行 根据权重调整最终评分
    f.Parallelizer().Until(ctx, len(nodes), func(index int) {
        nodePluginScores := framework.NodePluginScores{
            Name:   nodes[index].Node().Name,
            Scores: make([]framework.PluginScore, len(plugins)),
        }

        for i, pl := range plugins {
            weight := f.scorePluginWeight[pl.Name()]
            nodeScoreList := pluginToNodeScores[pl.Name()]
            score := nodeScoreList[index].Score
            // 评分的范围如果不在1-100之间返回错误
            if score > framework.MaxNodeScore || score < framework.MinNodeScore {
                err := fmt.Errorf("plugin %q returns an invalid score %v, it should in the range of [%v, %v] after normalizing", pl.Name(), score, framework.MinNodeScore, framework.MaxNodeScore)
                errCh.SendErrorWithCancel(err, cancel)
                return
            }
            weightedScore := score * int64(weight)
            // 记录单个插件最终评分到nodePluginScores
            nodePluginScores.Scores[i] = framework.PluginScore{
                Name:  pl.Name(),
                Score: weightedScore,
            }
            // 累加记录节点总分
            nodePluginScores.TotalScore += weightedScore
        }
        allNodePluginScores[index] = nodePluginScores
    }, metrics.Score)
    if err := errCh.ReceiveError(); err != nil {
        return nil, framework.AsStatus(fmt.Errorf("applying score defaultWeights on Score plugins: %w", err))
    }
    // 返回节点评分结果列表
    return allNodePluginScores, nil
}

// 节点评分结构
type NodePluginScores struct {
    // 节点名称
    Name string
    // 插件-评分 列表
    Scores []PluginScore
    // 总分
    TotalScore int64
}
```

##### NormalizeScore插件

一些插件的`NormalizeScore()`实现是直接调用了默认的归一化评分方法`DefaultNormalizeScore()`，在文件`pkg/scheduler/framework/plugins/helper/normalize_score.go`中定义，仅注释不做过多说明。

```Go
func DefaultNormalizeScore(maxPriority int64, reverse bool, scores framework.NodeScoreList) *framework.Status {
    var maxCount int64
    // 获取所有节点中的最高分
    for i := range scores {
        if scores[i].Score > maxCount {
            maxCount = scores[i].Score
        }
    }
    // 如果所有节点评分都是0的情况
    if maxCount == 0 {
        if reverse {
            for i := range scores {
                scores[i].Score = maxPriority
            }
        }
        return nil
    }
    // 正常情况
    for i := range scores {
        score := scores[i].Score
        // 100*评分/最高分
        score = maxPriority * score / maxCount
        // reverse用于Score评分低表示更高优先级的情况 如TaintToleration插件
        if reverse {
            // 低分反转变成高分
            score = maxPriority - score
        }
        // 记录归一化后的评分
        scores[i].Score = score
    }
    return nil
}
```

对上面例如`TaintToleration`插件的了解，不难发现其实调度器的算法实现并不复杂，重点在于整体流程的设计，实际上在了解了`Scheduler Framework`后，Pod的整个调度流程就已经非常清晰了。到目前为止总共接触到了`PreFilter`、`Filter`、`PreScore`、`Score`这四个扩展点的插件(其中`NormalizeScore`在`Score`扩展点内部，不属于12个标准扩展点之一)。在结合流程图中，前面有三个扩展点没有看到，分别是`PreEnqueue`、`QueueSort`和`PostFilter`，其中`PreEnqueue`和`QueueSort`是调度队列相关的两个插件，所以没有出现在调度周期内，如果感兴趣可以回到调度队列的`runPreEnqueuePlugins()`方法中，在Pod添加到`ActiveQ`时调用。`QueueSort`调用点较为隐蔽，可以以`func (aq *activeQueue) update()`为入口，调用关系如下方所示，调度队列实例创建之初，就向其中注册了`Less()`方法，它的调用点不像其他的插件是`runXXXPlugin()`而是`Less()`，Pod的入队和出队都会调用`Less()`方法。`PostFilter`插件只与抢占流程有关，在后面会单独介绍。

```Go
// 入队的调用链 activeQueue.update->queue.AddOrUpdate->heap.Push->up
func (aq *activeQueue) update(newPod *v1.Pod, oldPodInfo *framework.QueuedPodInfo) *framework.QueuedPodInfo {
    aq.lock.Lock()
    defer aq.lock.Unlock()

    if pInfo, exists := aq.queue.Get(oldPodInfo); exists {
        _ = pInfo.Update(newPod)
        aq.queue.AddOrUpdate(pInfo)
        return pInfo
    }
    return nil
}

func (h *Heap[T]) AddOrUpdate(obj T) {
    key := h.data.keyFunc(obj)
    if _, exists := h.data.items[key]; exists {
        h.data.items[key].obj = obj
        heap.Fix(h.data, h.data.items[key].index)
    } else {
        heap.Push(h.data, &itemKeyValue[T]{key, obj})
        if h.metricRecorder != nil {
            h.metricRecorder.Inc()
        }
    }
}

// golang的container/heap包
func Push(h Interface, x any) {
    h.Push(x)
    up(h, h.Len()-1)
}

func up(h Interface, j int) {
    for {
        i := (j - 1) / 2 // parent
        // 直接调用点
        if i == j || !h.Less(j, i) {
            break
        }
        h.Swap(i, j)
        j = i
    }
}

// 出队的调用链 activeQueue.pop->activeQueue.unlockedPop->queue.Pop->heap.Pop->down
func (aq *activeQueue) pop(logger klog.Logger) (*framework.QueuedPodInfo, error) {
    aq.lock.Lock()
    defer aq.lock.Unlock()

    return aq.unlockedPop(logger)
}

func (aq *activeQueue) unlockedPop(logger klog.Logger) (*framework.QueuedPodInfo, error) {
    for aq.queue.Len() == 0 {
        if aq.closed {
            logger.V(2).Info("Scheduling queue is closed")
            return nil, nil
        }
        aq.cond.Wait()
    }
    
    pInfo, err := aq.queue.Pop()
    ......
}

func (h *Heap[T]) Pop() (T, error) {
    obj := heap.Pop(h.data)
    if obj != nil {
        if h.metricRecorder != nil {
            h.metricRecorder.Dec()
        }
        return obj.(T), nil
    }
    var zero T
    return zero, fmt.Errorf("heap is empty")
}

// golang的container/heap包
func Pop(h Interface) any {
    n := h.Len() - 1
    h.Swap(0, n)
    down(h, 0, n)
    return h.Pop()
}

func down(h Interface, i0, n int) bool {
    i := i0
    for {
        j1 := 2*i + 1
        if j1 >= n || j1 < 0 { // j1 < 0 after int overflow
            break
        }
        j := j1 // left child
        // 直接调用点
        if j2 := j1 + 1; j2 < n && h.Less(j2, j1) {
            j = j2 // = 2*i + 2  // right child
        }
        if !h.Less(j, i) {
            break
        }
        h.Swap(i, j)
        i = j
    }
    return i > i0
}
```

