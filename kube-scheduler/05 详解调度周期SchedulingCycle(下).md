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

