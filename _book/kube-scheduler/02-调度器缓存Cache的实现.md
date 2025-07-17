# 调度器缓存Cache的实现

在正式进入调度器的工作流程之前，先再来了解一下`Cache`概念。 如果对`Informer`机制有一定的了解，会知道其中`Indexer`组件作为集群信息的`Local Storage`，但是在Pod调度过程中，其实使用的是调度器实现的由`Cache`专门提供的集群`Node-Pod`快照信息，下面一起看`Cache`的设计和实现。

需要注意的是，`client-go`中的`Cache`和此处提到的调度器`Cache`并不是一个。`client-go`中的缓存依赖`List-Watch`机制，虽然调度器中的缓存也依赖`Informer`，但缓存中的数据是不同的，如`client-go`中的缓存的是Kubernetes的`API`对象，而调度器的缓存主要保存了`Node`和`Pod`的映射和聚合信息。

## 定义与实现

`Cache`接口定义位于代码路径`pkg/scheduler/backend/cache/interface.go`下，此处有一个小的概念，当Pod的调度周期成功完成后，异步的绑定周期完成前，调度器会假定这个Pod会正常完成绑定，此时该Pod成为`AssumePod`，它所使用的资源通过`reserve`插件预留，在缓存中也认为该Pod会成功上线，并在`cacheImpl`中使用`assumedPods`集合存储这个状态的Pod。

```go
type Cache interface {
    // 单测使用
    NodeCount() int
    // 单测使用
    PodCount() (int, error)

    // AssumePod是处于scheduling结束但还没binding完成状态的Pod
    // 调度器对调度的Pod做出乐观预期 把成功找到调度节点Pod将占用的资源也纳入计算范围
    AssumePod(logger klog.Logger, pod *v1.Pod) error

    // AssumePod绑定成功后使用次方法通知Cache
    FinishBinding(logger klog.Logger, pod *v1.Pod) error

    // AssumePod绑定失败后使用次方法通知Cache
    ForgetPod(logger klog.Logger, pod *v1.Pod) error

    // 如果是AssumePod则确认该Pod 实际是执行UpdatePod操作
    // 如果没有在已有信息中找到这个Pod 就把它添加到缓存
    AddPod(logger klog.Logger, pod *v1.Pod) error

    // 更新Pod信息 先删除后添加
    UpdatePod(logger klog.Logger, oldPod, newPod *v1.Pod) error

    // 把Pod从Cache的podStates、assumedPods以及cache.nodes中移除
    RemovePod(logger klog.Logger, pod *v1.Pod) error

    // 获取Pod对象
    GetPod(pod *v1.Pod) (*v1.Pod, error)

    // 判断Pod是否假定调度
    IsAssumedPod(pod *v1.Pod) (bool, error)

    // 添加Node 返回Node克隆对象
    AddNode(logger klog.Logger, node *v1.Node) *framework.NodeInfo

    // 更新Node 返回Node克隆对象
    UpdateNode(logger klog.Logger, oldNode, newNode *v1.Node) *framework.NodeInfo

    // 删除Node
    RemoveNode(logger klog.Logger, node *v1.Node) error

    // 更新快照
    UpdateSnapshot(logger klog.Logger, nodeSnapshot *Snapshot) error

    // 根据当前缓存生成快照
    Dump() *Dump
}
```

其中的`AddPod`/`UpdatePod`/`RemovePod`都与`Informer`相关，由其事件回调函数中使用，如下部分代码所示。

```go
    // scheduled pod cache
    if handlerRegistration, err = informerFactory.Core().V1().Pods().Informer().AddEventHandler(
        cache.FilteringResourceEventHandler{
            FilterFunc: func(obj interface{}) bool {
                switch t := obj.(type) {
                case *v1.Pod:
                    return assignedPod(t)
                case cache.DeletedFinalStateUnknown:
                    if _, ok := t.Obj.(*v1.Pod); ok {
                        return true
                    }
                    utilruntime.HandleError(fmt.Errorf("unable to convert object %T to *v1.Pod in %T", obj, sched))
                    return false
                default:
                    utilruntime.HandleError(fmt.Errorf("unable to handle object in %T: %T", sched, obj))
                    return false
                }
            },
            // 注册回调函数
            Handler: cache.ResourceEventHandlerFuncs{
                AddFunc:    sched.addPodToCache,
                UpdateFunc: sched.updatePodInCache,
                DeleteFunc: sched.deletePodFromCache,
            },
        },
    ); err != nil {
        return err
    }
```

`cacheImpl`是`Cache`接口的实现，在代码路径`pkg/scheduler/backend/cache/cache.go`中定义。

```go
type cacheImpl struct {
    // 停止channel
    stop   <-chan struct{}
    // time to live 指缓存过期时间
    ttl    time.Duration
    // 定时周期
    period time.Duration
    // 读写锁
    mu sync.RWMutex
    // AssumedPod集合
    assumedPods sets.Set[string]
    // 所有Pod集合
    podStates map[string]*podState
    // 所有Node集合
    nodes     map[string]*nodeInfoListItem
    // 最新更新Node的指针
    headNode *nodeInfoListItem
    // 节点树
    nodeTree *nodeTree
    // 镜像状态
    imageStates map[string]*framework.ImageStateSummary
}
```

## 相关数据结构

### podState

Pod在缓存中是以`namespace-podname`为key，`podState`为value的形式存储的。

```go
type podState struct {
    // Pod对象
    pod *v1.Pod
    // 用于AssumedPod 如果在ddl之前没有完成bind则认为该AssumedPod过期
    deadline *time.Time
    // 绑定成功标志
    bindingFinished bool
}
```

### nodeInfoListItem

`nodeInfoListItem`是存储节点信息的双向链表。

```go
type nodeInfoListItem struct {
    // NodeInfo
    info *framework.NodeInfo
    // 双向链表指针
    next *nodeInfoListItem
    // 双向链表指针
    prev *nodeInfoListItem
}
```

### nodeTree

`nodeTree`是根据`zone`划分的节点集合。

```go
type nodeTree struct {
    // key是zone value是NodeName
    tree     map[string][]string
    // key的列表
    zones    []string
    // 总节点数
    numNodes int
}
```

## 相关方法

代码逻辑都比较清晰，故仅在代码片段中简单注释。

### New

启动一个`Cache`实例，代码实现如下。

```go
func New(ctx context.Context, ttl time.Duration) Cache {
    logger := klog.FromContext(ctx)
    // 创建Cache cleanAssumedPeriod
    cache := newCache(ctx, ttl, cleanAssumedPeriod)
    cache.run(logger)
    return cache
}

func newCache(ctx context.Context, ttl, period time.Duration) *cacheImpl {
    logger := klog.FromContext(ctx)
    return &cacheImpl{
        ttl:    ttl,
        period: period,
        stop:   ctx.Done(),

        nodes:       make(map[string]*nodeInfoListItem),
        nodeTree:    newNodeTree(logger, nil),
        assumedPods: sets.New[string](),
        podStates:   make(map[string]*podState),
        imageStates: make(map[string]*framework.ImageStateSummary),
    }
}

func (cache *cacheImpl) run(logger klog.Logger) {
    // 起一个goroutine 定期清理AssumedPod 
    go wait.Until(func() {
        cache.cleanupAssumedPods(logger, time.Now())
    }, cache.period, cache.stop)
}

func (cache *cacheImpl) cleanupAssumedPods(logger klog.Logger, now time.Time) {
    cache.mu.Lock()
    defer cache.mu.Unlock()
    defer cache.updateMetrics()

    // 遍历AssumedPod
    for key := range cache.assumedPods {
        // 获取podStatus
        ps, ok := cache.podStates[key]
        if !ok {
            logger.Error(nil, "Key found in assumed set but not in podStates, potentially a logical error")
            klog.FlushAndExit(klog.ExitFlushTimeout, 1)
        }
        // 还在binding处理中则跳过当前Pod
        if !ps.bindingFinished {
            logger.V(5).Info("Could not expire cache for pod as binding is still in progress", "podKey", key, "pod", klog.KObj(ps.pod))
            continue
        }
        // 不是常驻Pod且当前时间超过了AssumedPod的ddl
        if cache.ttl != 0 && now.After(*ps.deadline) {
            logger.Info("Pod expired", "podKey", key, "pod", klog.KObj(ps.pod))
            // 移除过期的AssumedPod
            if err := cache.removePod(logger, ps.pod); err != nil {
                logger.Error(err, "ExpirePod failed", "podKey", key, "pod", klog.KObj(ps.pod))
            }
        }
    }
}

func (cache *cacheImpl) removePod(logger klog.Logger, pod *v1.Pod) error {
    key, err := framework.GetPodKey(pod)
    if err != nil {
        return err
    }

    n, ok := cache.nodes[pod.Spec.NodeName]
    if !ok {
        logger.Error(nil, "Node not found when trying to remove pod", "node", klog.KRef("", pod.Spec.NodeName), "podKey", key, "pod", klog.KObj(pod))
    } else {
        // 从NodeInfo中移除过期AssumedPod
        if err := n.info.RemovePod(logger, pod); err != nil {
            return err
        }
        // 如果移除后 对应NodeInfo已经没有Pod了
        // 顺便把这个Node从缓存中删除
        if len(n.info.Pods) == 0 && n.info.Node() == nil {
            cache.removeNodeInfoFromList(logger, pod.Spec.NodeName)
        } else {
            // 删除后Node中还有Pod 移动这个节点到Head
            cache.moveNodeInfoToHead(logger, pod.Spec.NodeName)
        }
    }
    // 从缓存中删除过期AssumedPod
    delete(cache.podStates, key)
    delete(cache.assumedPods, key)
    return nil
}
```

### Dump

主要用于调试，属于`CacheDebugger`的一部分，通常把`Cache`中的信息记录到日志，从而定位调度过程中的问题。

```go
func (cache *cacheImpl) Dump() *Dump {
    cache.mu.RLock()
    defer cache.mu.RUnlock()

    nodes := make(map[string]*framework.NodeInfo, len(cache.nodes))
    for k, v := range cache.nodes {
       // Snapshot返回一个NodeInfo的克隆对象
       nodes[k] = v.info.Snapshot()
    }

    return &Dump{
       Nodes:       nodes,
       AssumedPods: cache.assumedPods.Union(nil),
    }
}

// CacheDebugger provides ways to check and write cache information for debugging.
type CacheDebugger struct {
    Comparer CacheComparer
    Dumper   CacheDumper
}
```

### UpdateSnapshot

把`Cache`的信息更新到一个`Snapshot`快照结构，每一轮调度Pod时都会使用此方法更新快照。

```go
func (cache *cacheImpl) UpdateSnapshot(logger klog.Logger, nodeSnapshot *Snapshot) error {
    cache.mu.Lock()
    defer cache.mu.Unlock()
    // 获取上次的快照编号
    snapshotGeneration := nodeSnapshot.generation
    // 初始化更新标志位
    updateAllLists := false
    updateNodesHavePodsWithAffinity := false
    updateNodesHavePodsWithRequiredAntiAffinity := false
    updateUsedPVCSet := false

    // 从headNode开始遍历节点信息并更新快照
    for node := cache.headNode; node != nil; node = node.next {
        // 如果当次节点编号不大于快照编号就退出遍历
        // 因为Cache中的节点是一个双向链表 最近被更新过的节点信息在前面
        // 如果前面的节点都没有比快照更新 那后面也不会有更新
       if node.info.Generation <= snapshotGeneration {
          break
       }
       if np := node.info.Node(); np != nil {
          existing, ok := nodeSnapshot.nodeInfoMap[np.Name]
          // 如果节点信息在快照中不存在 updateAllLists标志位置为True
          if !ok {
             updateAllLists = true
             existing = &framework.NodeInfo{}
             nodeSnapshot.nodeInfoMap[np.Name] = existing
          }
          // 根据NodeInfo创建快照副本
          clone := node.info.Snapshot()
          // 判断亲和性信息是否需要更新 如果缓存和快照中的列表长度不一致 updateNodesHavePodsWithAffinity标志位置为True
          if (len(existing.PodsWithAffinity) > 0) != (len(clone.PodsWithAffinity) > 0) {
             updateNodesHavePodsWithAffinity = true
          }
          // 判断亲和性信息是否需要更新 如果缓存和快照中的列表长度不一致 updateNodesHavePodsWithRequiredAntiAffinity标志位置为True
          if (len(existing.PodsWithRequiredAntiAffinity) > 0) != (len(clone.PodsWithRequiredAntiAffinity) > 0) {
             updateNodesHavePodsWithRequiredAntiAffinity = true
          }
          // 判断PVC集合是否需要更新 如果有就把updateUsedPVCSet标志位置为True
          if !updateUsedPVCSet {
             if len(existing.PVCRefCounts) != len(clone.PVCRefCounts) {
                updateUsedPVCSet = true
             } else {
                for pvcKey := range clone.PVCRefCounts {
                   if _, found := existing.PVCRefCounts[pvcKey]; !found {
                      updateUsedPVCSet = true
                      break
                   }
                }
             }
          }
          // 覆盖原有信息
          *existing = *clone
       }
    }
    // 用最新的编号给快照编号赋值
    if cache.headNode != nil {
       nodeSnapshot.generation = cache.headNode.info.Generation
    }

    // 如果快照中的节点数量大于当前缓存中的节点数量 移除已被删除的节点
    if len(nodeSnapshot.nodeInfoMap) > cache.nodeTree.numNodes {
       cache.removeDeletedNodesFromSnapshot(nodeSnapshot)
       updateAllLists = true
    }
    // 如果表示有更新项 执行具体函数更新
    if updateAllLists || updateNodesHavePodsWithAffinity || updateNodesHavePodsWithRequiredAntiAffinity || updateUsedPVCSet {
       // 更新快照信息
       cache.updateNodeInfoSnapshotList(logger, nodeSnapshot, updateAllLists)
    }
    // 检查结果的一致性 如果不一致再次执行updateNodeInfoSnapshotList函数更新
    if len(nodeSnapshot.nodeInfoList) != cache.nodeTree.numNodes {
       errMsg := fmt.Sprintf("snapshot state is not consistent, length of NodeInfoList=%v not equal to length of nodes in tree=%v "+
          ", length of NodeInfoMap=%v, length of nodes in cache=%v"+
          ", trying to recover",
          len(nodeSnapshot.nodeInfoList), cache.nodeTree.numNodes,
          len(nodeSnapshot.nodeInfoMap), len(cache.nodes))
       logger.Error(nil, errMsg)
       // 更新快照信息
       cache.updateNodeInfoSnapshotList(logger, nodeSnapshot, true)
       return errors.New(errMsg)
    }

    return nil
}

func (cache *cacheImpl) updateNodeInfoSnapshotList(logger klog.Logger, snapshot *Snapshot, updateAll bool) {
    // 初始化列表和集合
    snapshot.havePodsWithAffinityNodeInfoList = make([]*framework.NodeInfo, 0, cache.nodeTree.numNodes)
    snapshot.havePodsWithRequiredAntiAffinityNodeInfoList = make([]*framework.NodeInfo, 0, cache.nodeTree.numNodes)
    snapshot.usedPVCSet = sets.New[string]()
    // updateAll标志位为True时重新构建nodeInfoList 在节点树结构发生变化(添加/删除)时被采用 更新成本更高
    // 如果是False 不重新构建nodeInfoList而是基于现有的nodeInfoList进行更新 此时只涉及亲和/反亲和/PVC发生变化的情况
    if updateAll {
        snapshot.nodeInfoList = make([]*framework.NodeInfo, 0, cache.nodeTree.numNodes)
        nodesList, err := cache.nodeTree.list()
        if err != nil {
            logger.Error(err, "Error occurred while retrieving the list of names of the nodes from node tree")
        }
        for _, nodeName := range nodesList {
            if nodeInfo := snapshot.nodeInfoMap[nodeName]; nodeInfo != nil {
                snapshot.nodeInfoList = append(snapshot.nodeInfoList, nodeInfo)
                if len(nodeInfo.PodsWithAffinity) > 0 {
                    snapshot.havePodsWithAffinityNodeInfoList = append(snapshot.havePodsWithAffinityNodeInfoList, nodeInfo)
                }
                if len(nodeInfo.PodsWithRequiredAntiAffinity) > 0 {
                    snapshot.havePodsWithRequiredAntiAffinityNodeInfoList = append(snapshot.havePodsWithRequiredAntiAffinityNodeInfoList, nodeInfo)
                }
                for key := range nodeInfo.PVCRefCounts {
                    snapshot.usedPVCSet.Insert(key)
                }
            } else {
                logger.Error(nil, "Node exists in nodeTree but not in NodeInfoMap, this should not happen", "node", klog.KRef("", nodeName))
            }
        }
    } else {
        for _, nodeInfo := range snapshot.nodeInfoList {
            if len(nodeInfo.PodsWithAffinity) > 0 {
                snapshot.havePodsWithAffinityNodeInfoList = append(snapshot.havePodsWithAffinityNodeInfoList, nodeInfo)
            }
            if len(nodeInfo.PodsWithRequiredAntiAffinity) > 0 {
                snapshot.havePodsWithRequiredAntiAffinityNodeInfoList = append(snapshot.havePodsWithRequiredAntiAffinityNodeInfoList, nodeInfo)
            }
            for key := range nodeInfo.PVCRefCounts {
                snapshot.usedPVCSet.Insert(key)
            }
        }
    }
}
```

### AssumePod

假定一个Pod调度成功，把它以`AssumedPod`的形式添加到节点缓存中。

```go
func (cache *cacheImpl) AssumePod(logger klog.Logger, pod *v1.Pod) error {
    key, err := framework.GetPodKey(pod)
    if err != nil {
       return err
    }

    cache.mu.Lock()
    defer cache.mu.Unlock()
    if _, ok := cache.podStates[key]; ok {
       return fmt.Errorf("pod %v(%v) is in the cache, so can't be assumed", key, klog.KObj(pod))
    }

    return cache.addPod(logger, pod, true)
}

func (cache *cacheImpl) addPod(logger klog.Logger, pod *v1.Pod, assumePod bool) error {
    key, err := framework.GetPodKey(pod)
    if err != nil {
        eturn err
    }
    n, ok := cache.nodes[pod.Spec.NodeName] 
    // 如果缓存中不存在对应节点就新创建一个
    if !ok {
        n = newNodeInfoListItem(framework.NewNodeInfo())
        cache.nodes[pod.Spec.NodeName] = n
    }
    // 缓存的NodeInfo中添加Pod信息
    n.info.AddPod(pod)
    // 移动节点到头部
    cache.moveNodeInfoToHead(logger, pod.Spec.NodeName)
    ps := &podState{
        pod: pod,
    }
    // podStates中添加Pod信息
    cache.podStates[key] = ps
    if assumePod {
        // 如果是AssumePod在assumedPods列表中也添加
        cache.assumedPods.Insert(key)
    }
    return nil
}
```

### ForgetPod

`AssumePod`过期，将其从调度缓存中移除。

```go
func (cache *cacheImpl) ForgetPod(logger klog.Logger, pod *v1.Pod) error {
    key, err := framework.GetPodKey(pod)
    if err != nil {
       return err
    }

    cache.mu.Lock()
    defer cache.mu.Unlock()

    currState, ok := cache.podStates[key]
    if ok && currState.pod.Spec.NodeName != pod.Spec.NodeName {
       return fmt.Errorf("pod %v(%v) was assumed on %v but assigned to %v", key, klog.KObj(pod), pod.Spec.NodeName, currState.pod.Spec.NodeName)
    }

    // Only assumed pod can be forgotten.
    if ok && cache.assumedPods.Has(key) {
       return cache.removePod(logger, pod)
    }
    return fmt.Errorf("pod %v(%v) wasn't assumed so cannot be forgotten", key, klog.KObj(pod))
}
```

### updatePod

更新缓存中Pod信息，先删除后添加能够保证缓存信息的一致性。

```go
func (cache *cacheImpl) updatePod(logger klog.Logger, oldPod, newPod *v1.Pod) error {
    // 先删除
    if err := cache.removePod(logger, oldPod); err != nil {
       return err
    }
    // 再添加
    return cache.addPod(logger, newPod, false)
}
```

## Cache中数据的同步

在调度器启动过程中，代码位于`cmd/kube-scheduler/app/server.go`，`Cache`同步的重要代码就在`Run()`函数中，此处会涉及到`Informer`机制。但可以提前了解到，在调度器启动之前，通过和`Informer`的协同，保证了`Cache`中数据和集群信息的一致性。

```go
startInformersAndWaitForSync := func(ctx context.Context) {
    // 启动所有Informer
    cc.InformerFactory.Start(ctx.Done())
    // DynInformerFactory can be nil in tests.
    if cc.DynInformerFactory != nil {
       cc.DynInformerFactory.Start(ctx.Done())
    }

    // 等待Cache中的数据同步完成
    cc.InformerFactory.WaitForCacheSync(ctx.Done())
    // DynInformerFactory can be nil in tests.
    if cc.DynInformerFactory != nil {
       cc.DynInformerFactory.WaitForCacheSync(ctx.Done())
    }

    // Wait for all handlers to sync (all items in the initial list delivered) before scheduling.
    if err := sched.WaitForHandlersSync(ctx); err != nil {
       logger.Error(err, "waiting for handlers to sync")
    }

    close(handlerSyncReadyCh)
    logger.V(3).Info("Handlers synced")
}
```

等待缓存同步用到下面的这个函数，但在未来版本会被其他函数替代。

```go
func WaitForCacheSync(stopCh <-chan struct{}, cacheSyncs ...InformerSynced) bool {
    err := wait.PollImmediateUntil(syncedPollPeriod,
       func() (bool, error) {
          for _, syncFunc := range cacheSyncs {
             if !syncFunc() {
                return false, nil
             }
          }
          return true, nil
       },
       stopCh)
    if err != nil {
       return false
    }

    return true
}
```

## Pod在缓存中的流程图

```go
// State Machine of a pod's events in scheduler's cache:
//
//
//   +-------------------------------------------+   +----+
//   |                            Add            |   |    |
//   |                                           |   |    | Update
//   +      Assume                Add            v   v    |
// Initial +--------> Assumed +------------+---> Added <--+
//   ^                +    +               |       +
//   |                |    |               |       |
//   |                |    |           Add |       | Remove
//   |                |    |               |       |
//   |                |    |               +       |
//   +----------------+    +-----------> Expired   +------> Deleted
//         Forget             Expire
//
```

