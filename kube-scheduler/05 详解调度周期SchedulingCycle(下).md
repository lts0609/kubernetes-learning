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
