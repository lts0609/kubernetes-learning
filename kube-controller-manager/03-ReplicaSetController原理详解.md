# ReplicaSetController原理详解

## 实例创建

实例创建的部分属于通用基本逻辑，创建`Pod`和`ReplicaSet`对象的Informer，GVK标识为{Group: apps, Version: v1, Kind: ReplicaSet}。

```Go
// 创建ReplicaSetController的Descriptor
func newReplicaSetControllerDescriptor() *ControllerDescriptor {
    return &ControllerDescriptor{
        name:     names.ReplicaSetController,
        aliases:  []string{"replicaset"},
        initFunc: startReplicaSetController,
    }
}

// ReplicaSetController的初始化函数
func startReplicaSetController(ctx context.Context, controllerContext ControllerContext, controllerName string) (controller.Interface, bool, error) {
    go replicaset.NewReplicaSetController(
        ctx,
        controllerContext.InformerFactory.Apps().V1().ReplicaSets(),
        controllerContext.InformerFactory.Core().V1().Pods(),
        controllerContext.ClientBuilder.ClientOrDie("replicaset-controller"),
        replicaset.BurstReplicas,
    ).Run(ctx, int(controllerContext.ComponentConfig.ReplicaSetController.ConcurrentRSSyncs))
    return nil, true, nil
}

// 创建ReplicaSetController实例
func NewReplicaSetController(ctx context.Context, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int) *ReplicaSetController {
    logger := klog.FromContext(ctx)
    eventBroadcaster := record.NewBroadcaster(record.WithContext(ctx))
    if err := metrics.Register(legacyregistry.Register); err != nil {
        logger.Error(err, "unable to register metrics")
    }
    return NewBaseController(logger, rsInformer, podInformer, kubeClient, burstReplicas,
        apps.SchemeGroupVersion.WithKind("ReplicaSet"),
        "replicaset_controller",
        "replicaset",
        controller.RealPodControl{
            KubeClient: kubeClient,
            Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "replicaset-controller"}),
        },
        eventBroadcaster,
    )
}

// 启动ReplicaSetController
func (rsc *ReplicaSetController) Run(ctx context.Context, workers int) {
    defer utilruntime.HandleCrash()

    // Start events processing pipeline.
    rsc.eventBroadcaster.StartStructuredLogging(3)
    rsc.eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: rsc.kubeClient.CoreV1().Events("")})
    defer rsc.eventBroadcaster.Shutdown()

    defer rsc.queue.ShutDown()

    controllerName := strings.ToLower(rsc.Kind)
    logger := klog.FromContext(ctx)
    logger.Info("Starting controller", "name", controllerName)
    defer logger.Info("Shutting down controller", "name", controllerName)

    if !cache.WaitForNamedCacheSync(rsc.Kind, ctx.Done(), rsc.podListerSynced, rsc.rsListerSynced) {
        return
    }

    for i := 0; i < workers; i++ {
        // 每秒循环执行
        go wait.UntilWithContext(ctx, rsc.worker, time.Second)
    }

    <-ctx.Done()
}
```

### 启动

这部分代码是标准化的，和其他所有控制器都相同，`syncHandler()`方法在创建实例的`NewBaseController()`函数中注册，实际为`syncReplicaSet()`方法。

```Go
func (rsc *ReplicaSetController) worker(ctx context.Context) {
    for rsc.processNextWorkItem(ctx) {
    }
}

func (rsc *ReplicaSetController) processNextWorkItem(ctx context.Context) bool {
    key, quit := rsc.queue.Get()
    if quit {
        return false
    }
    defer rsc.queue.Done(key)
    // 执行调谐
    err := rsc.syncHandler(ctx, key)
    if err == nil {
        rsc.queue.Forget(key)
        return true
    }

    utilruntime.HandleError(fmt.Errorf("sync %q failed with %v", key, err))
    rsc.queue.AddRateLimited(key)

    return true
}
```

## 控制器调谐逻辑

`Informer`监测到变化时`object`对象是以key为`namespace/name`的形式加入工作队列的，调谐逻辑处理时要通过`key`再把命名空间和对象名称分离。

```Go
func (rsc *ReplicaSetController) syncReplicaSet(ctx context.Context, key string) error {
    logger := klog.FromContext(ctx)
    startTime := time.Now()
    defer func() {
        logger.Info("Finished syncing", "kind", rsc.Kind, "key", key, "duration", time.Since(startTime))
    }()
    // 分割获取命名空间和名称
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
        return err
    }
    // 根据信息获取RS对象
    rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
    if apierrors.IsNotFound(err) {
        logger.V(4).Info("deleted", "kind", rsc.Kind, "key", key)
        rsc.expectations.DeleteExpectations(logger, key)
        return nil
    }
    if err != nil {
        return err
    }
    // 检查期望状态是否已经达成
    rsNeedsSync := rsc.expectations.SatisfiedExpectations(logger, key)
    // 获取对象的标签选择器
    selector, err := metav1.LabelSelectorAsSelector(rs.Spec.Selector)
    if err != nil {
        utilruntime.HandleError(fmt.Errorf("error converting pod selector to selector for rs %v/%v: %v", namespace, name, err))
        return nil
    }

    // 获取命名空间下的所有Pod
    allPods, err := rsc.podLister.Pods(rs.Namespace).List(labels.Everything())
    if err != nil {
        return err
    }
    // 过滤出Running的Pod
    filteredPods := controller.FilterActivePods(logger, allPods)
    // 在此过滤出由当前控制器管理的Pod
    filteredPods, err = rsc.claimPods(ctx, rs, selector, filteredPods)
    if err != nil {
        return err
    }

    var manageReplicasErr error
    // 需要进行副本管理操作
    if rsNeedsSync && rs.DeletionTimestamp == nil {
        manageReplicasErr = rsc.manageReplicas(ctx, filteredPods, rs)
    }
    // 更新ReplicaSet对象状态
    rs = rs.DeepCopy()
    newStatus := calculateStatus(rs, filteredPods, manageReplicasErr)

    // Always updates status as pods come up or die.
    updatedRS, err := updateReplicaSetStatus(logger, rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace), rs, newStatus)
    if err != nil {
        // Multiple things could lead to this update failing. Requeuing the replica set ensures
        // Returning an error causes a requeue without forcing a hotloop
        return err
    }
    // Resync the ReplicaSet after MinReadySeconds as a last line of defense to guard against clock-skew.
    if manageReplicasErr == nil && updatedRS.Spec.MinReadySeconds > 0 &&
        updatedRS.Status.ReadyReplicas == *(updatedRS.Spec.Replicas) &&
        updatedRS.Status.AvailableReplicas != *(updatedRS.Spec.Replicas) {
        rsc.queue.AddAfter(key, time.Duration(updatedRS.Spec.MinReadySeconds)*time.Second)
    }
    return manageReplicasErr
}
```

### 期望副本状态判断

`SatisfiedExpectations()`方法用来检查对象的当前状态和期望状态是否需要进行同步，用于`ReplicaSetController`、`DaemonSetController`、`JobController`这几个存在副本期望的控制器中。

```Go
func (r *ControllerExpectations) SatisfiedExpectations(logger klog.Logger, controllerKey string) bool {
    if exp, exists, err := r.GetExpectations(controllerKey); exists {
        // 检查期望数量是否满足
        if exp.Fulfilled() {
            logger.V(4).Info("Controller expectations fulfilled", "expectations", exp)
            return true
        // 检查期望是否过期
        } else if exp.isExpired() {
            logger.V(4).Info("Controller expectations expired", "expectations", exp)
            return true
        } else {
            logger.V(4).Info("Controller still waiting on expectations", "expectations", exp)
            return false
        }
    } else if err != nil {
        logger.V(2).Info("Error encountered while checking expectations, forcing sync", "err", err)
    } else {
        logger.V(4).Info("Controller either never recorded expectations, or the ttl expired", "controller", controllerKey)
    }
    return true
}

func (e *ControlleeExpectations) Fulfilled() bool {
    // 期望创建和删除的副本数都不大于0表示满足
    return atomic.LoadInt64(&e.add) <= 0 && atomic.LoadInt64(&e.del) <= 0
}
```

期望状态的检查发生在每次调谐的开始，如果`SatisfiedExpectations()`方法返回了`false`，那就意味着`manageReplicas()`方法不会被执行，会导致该对象在控制器中逻辑被阻塞。

### 过滤控制器管理Pod

首先通过`PodLister`获取命名空间下的所有Pod，然后调用`claimPods()`方法进行处理。该方法会先创建一个`PodControllerRefManager`对象，也就是说在每次调谐时都会给目标`ReplicaSet`创建一个`PodControllerRefManager`用来认领Pod，它作为局部变量生命周期随`claimPods()`方法一同结束。

```Go
func (rsc *ReplicaSetController) claimPods(ctx context.Context, rs *apps.ReplicaSet, selector labels.Selector, filteredPods []*v1.Pod) ([]*v1.Pod, error) {
    // Pod认领判断函数
    canAdoptFunc := controller.RecheckDeletionTimestamp(func(ctx context.Context) (metav1.Object, error) {
        fresh, err := rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace).Get(ctx, rs.Name, metav1.GetOptions{})
        if err != nil {
            return nil, err
        }
        if fresh.UID != rs.UID {
            return nil, fmt.Errorf("original %v %v/%v is gone: got uid %v, wanted %v", rsc.Kind, rs.Namespace, rs.Name, fresh.UID, rs.UID)
        }
        return fresh, nil
    })
    // 创建PodControllerRefManager
    cm := controller.NewPodControllerRefManager(rsc.podControl, rs, selector, rsc.GroupVersionKind, canAdoptFunc)
    // 处理Pod
    return cm.ClaimPods(ctx, filteredPods)
}
```

对象创建后只有一个任务，调用`ClaimPods()`方法过滤属于这个`ReplicaSet`管理的Pod。在这个过程中要匹配Pod标签和`ReplicaSet`的标签选择器，所以输入参数除了上下文信息只传入了Pod列表，实际上还可以根据需要传递过滤函数`filters`。其中定义了`match`、`adopt`、`release`三种方法用于在对应的情况下处理Pod。

```Go
func (m *PodControllerRefManager) ClaimPods(ctx context.Context, pods []*v1.Pod, filters ...func(*v1.Pod) bool) ([]*v1.Pod, error) {
    var claimed []*v1.Pod
    var errlist []error
    // 筛选函数
    match := func(obj metav1.Object) bool {
        pod := obj.(*v1.Pod)
        // 匹配标签选择器
        if !m.Selector.Matches(labels.Set(pod.Labels)) {
            return false
        }
        // 匹配自定义过滤函数
        for _, filter := range filters {
            if !filter(pod) {
                return false
            }
        }
        return true
    }
    // 认领函数
    adopt := func(ctx context.Context, obj metav1.Object) error {
        return m.AdoptPod(ctx, obj.(*v1.Pod))
    }
    // 释放函数
    release := func(ctx context.Context, obj metav1.Object) error {
        return m.ReleasePod(ctx, obj.(*v1.Pod))
    }
    // 遍历Pod列表
    for _, pod := range pods {
        ok, err := m.ClaimObject(ctx, pod, match, adopt, release)
        if err != nil {
            errlist = append(errlist, err)
            continue
        }
        // 收集认领成功的Pod
        if ok {
            claimed = append(claimed, pod)
        }
    }
    return claimed, utilerrors.NewAggregate(errlist)
}
```

处理单个Pod的具体动作由`ClaimObject()`方法完成，首先检查Pod是否存在`OwnerReference`，如果不存在那么当前它是一个孤儿对象，检查Pod是否正处于删除状态以及它的命名空间和当前控制器的是否相同，然后尝试认领该Pod;如果存在，检查`OwnerReference`的`UID`和当前控制器是否相同，`UID`如果相同还需要确认标签选择器是否满足，然后返回结果。

```Go
func (m *BaseControllerRefManager) ClaimObject(ctx context.Context, obj metav1.Object, match func(metav1.Object) bool, adopt, release func(context.Context, metav1.Object) error) (bool, error) {
    // 获取Pod的OwnerReference对象
    controllerRef := metav1.GetControllerOfNoCopy(obj)
    // OwnerReference存在
    if controllerRef != nil {
        // 检查OwnerReference的UID和当前控制器是否相同
        if controllerRef.UID != m.Controller.GetUID() {
            // 属于其他控制器管理
            return false, nil
        }
        if match(obj) {
            // 只要满足条件就返回成功
            return true, nil
        }
        // OwnerReference的UID相同但不满足标签选择器的条件
        // 检查是否正在被删除
        if m.Controller.GetDeletionTimestamp() != nil {
            return false, nil
        }
        // 如果没有在删除中 调用release方法释放Pod
        if err := release(ctx, obj); err != nil {
            // If the pod no longer exists, ignore the error.
            if errors.IsNotFound(err) {
                return false, nil
            }
            // Either someone else released it, or there was a transient error.
            // The controller should requeue and try again if it's still stale.
            return false, err
        }
        // 释放成功返回false
        return false, nil
    }

    // OwnerReference不存在 孤儿对象处理分支
    if m.Controller.GetDeletionTimestamp() != nil || !match(obj) {
        // Ignore if we're being deleted or selector doesn't match.
        return false, nil
    }
    // 是否正在删除中
    if obj.GetDeletionTimestamp() != nil {
        return false, nil
    }
    // 命名空间检查
    if len(m.Controller.GetNamespace()) > 0 && m.Controller.GetNamespace() != obj.GetNamespace() {
        return false, nil
    }

    // 调用adopt方法认领Pod
    if err := adopt(ctx, obj); err != nil {
        // If the pod no longer exists, ignore the error.
        if errors.IsNotFound(err) {
            return false, nil
        }
        // Either someone else claimed it first, or there was a transient error.
        // The controller should requeue and try again if it's still orphaned.
        return false, err
    }
    // 认领成功返回true
    return true, nil
}
```

### 副本管理(核心逻辑)

如果当前状态和期望状态不一致，就会调用`manageReplicas()`方法使副本状态趋于期望。根据函数签名，输入参数包括上下文信息、控制器管理的Pod列表`filteredPods`和`ReplicaSet`对象`rs`。

```Go
func (rsc *ReplicaSetController) manageReplicas(ctx context.Context, filteredPods []*v1.Pod, rs *apps.ReplicaSet) error {
    // 计算期望副本数和实际副本数的差值
    diff := len(filteredPods) - int(*(rs.Spec.Replicas))
    // 经过KeyFunc获取ReplicaSet对象的obj key
    rsKey, err := controller.KeyFunc(rs)
    if err != nil {
        utilruntime.HandleError(fmt.Errorf("couldn't get key for %v %#v: %v", rsc.Kind, rs, err))
        return nil
    }
    logger := klog.FromContext(ctx)
    // 实际副本数小于期望副本数
    if diff < 0 {
        diff *= -1
        // 限制变化数量上限
        if diff > rsc.burstReplicas {
            diff = rsc.burstReplicas
        }
        // 记录预期创建数量 与Informer协同作用
        rsc.expectations.ExpectCreations(logger, rsKey, diff)
        logger.V(2).Info("Too few replicas", "replicaSet", klog.KObj(rs), "need", *(rs.Spec.Replicas), "creating", diff)
        // 创建期望数量的Pod
        successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
            err := rsc.podControl.CreatePods(ctx, rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
            if err != nil {
                if apierrors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
                    return nil
                }
            }
            return err
        })
        // 存在Pod创建失败
        if skippedPods := diff - successfulCreations; skippedPods > 0 {
            logger.V(2).Info("Slow-start failure. Skipping creation of pods, decrementing expectations", "podsSkipped", skippedPods, "kind", rsc.Kind, "replicaSet", klog.KObj(rs))
            for i := 0; i < skippedPods; i++ {
                // 修改期望创建数量
                rsc.expectations.CreationObserved(logger, rsKey)
            }
        }
        return err
    } else if diff > 0 {
        // 实际副本数大于期望副本数
        if diff > rsc.burstReplicas {
            diff = rsc.burstReplicas
        }
        logger.V(2).Info("Too many replicas", "replicaSet", klog.KObj(rs), "need", *(rs.Spec.Replicas), "deleting", diff)

        relatedPods, err := rsc.getIndirectlyRelatedPods(logger, rs)
        utilruntime.HandleError(err)

        // Choose which Pods to delete, preferring those in earlier phases of startup.
        podsToDelete := getPodsToDelete(filteredPods, relatedPods, diff)

        // Snapshot the UIDs (ns/name) of the pods we're expecting to see
        // deleted, so we know to record their expectations exactly once either
        // when we see it as an update of the deletion timestamp, or as a delete.
        // Note that if the labels on a pod/rs change in a way that the pod gets
        // orphaned, the rs will only wake up after the expectations have
        // expired even if other pods are deleted.
        rsc.expectations.ExpectDeletions(logger, rsKey, getPodKeys(podsToDelete))

        errCh := make(chan error, diff)
        var wg sync.WaitGroup
        wg.Add(diff)
        for _, pod := range podsToDelete {
            go func(targetPod *v1.Pod) {
                defer wg.Done()
                if err := rsc.podControl.DeletePod(ctx, rs.Namespace, targetPod.Name, rs); err != nil {
                    // Decrement the expected number of deletes because the informer won't observe this deletion
                    podKey := controller.PodKey(targetPod)
                    rsc.expectations.DeletionObserved(logger, rsKey, podKey)
                    if !apierrors.IsNotFound(err) {
                        logger.V(2).Info("Failed to delete pod, decremented expectations", "pod", podKey, "kind", rsc.Kind, "replicaSet", klog.KObj(rs))
                        errCh <- err
                    }
                }
            }(pod)
        }
        wg.Wait()

        select {
        case err := <-errCh:
            // all errors have been reported before and they're likely to be the same, so we'll only return the first one we hit.
            if err != nil {
                return err
            }
        default:
        }
    }

    return nil
}
```

#### 慢启动批量创建

使用`slowStartBatch()`函数批量创建Pod，函数名为慢启动批处理，用于控制并发的速率，避免一次性发起过多请求造成的系统压力。接收处理总数`count`、初始批处理大小`initialBatchSize`和逻辑函数`fn`。循环执行输入的逻辑函数，初始批处理大小为1，通过`channel`控制并发，每一轮循环后更新计数和批处理大小，最后返回成功数量。

```Go
func slowStartBatch(count int, initialBatchSize int, fn func() error) (int, error) {
    // 剩余处理次数
    remaining := count
    successes := 0
    // 循环执行逻辑 批处理数量逐步增加
    for batchSize := min(remaining, initialBatchSize); batchSize > 0; batchSize = min(2*batchSize, remaining) {
        errCh := make(chan error, batchSize)
        var wg sync.WaitGroup
        // 并发控制
        wg.Add(batchSize)
        for i := 0; i < batchSize; i++ {
            go func() {
                defer wg.Done()
                if err := fn(); err != nil {
                    errCh <- err
                }
            }()
        }
        wg.Wait()
        // 更新计数
        curSuccesses := batchSize - len(errCh)
        successes += curSuccesses
        // 有失败事件直接返回
        if len(errCh) > 0 {
            return successes, <-errCh
        }
        remaining -= batchSize
    }
    return successes, nil
}
```

##### 创建失败处理和EventHandler

在上面的流程中曾调用`expectations.ExpectCreations()`方法设置期望创建/删除副本数量，期望值`expectations`充当缓冲计数器，并且会传递到后面的周期，控制器在启动时的`SatisfiedExpectations() `方法就是对期望值进行检查，为了保证核心逻辑的顺利执行，会期望每次检查时的`ControlleeExpectations.add`和`ControlleeExpectations.del`都不大于0。这依赖`PodInformer`和`CreationObserved()`的协同处理，在注册的`EventHandler`中，每观测到创建了一个属于该`ReplicaSet`对象的Pod副本，就会调用`CreationObserved()方法使`计数器的`add`字段值减一，也就是说如果所有的创建操作都执行成功，那下一次的期望检查就会通过，然后重新在`manageReplicas()`方法中计算差值并进行创建/删除。当创建过程中发生错误，那么就会调用`CreationObserved()`方法，执行`期望扩容数-成功扩容数`的次数，最终使计数器清零。

```Go
func (r *ControllerExpectations) CreationObserved(logger klog.Logger, controllerKey string) {
    r.LowerExpectations(logger, controllerKey, 1, 0)
}

func (r *ControllerExpectations) LowerExpectations(logger klog.Logger, controllerKey string, add, del int) {
    if exp, exists, err := r.GetExpectations(controllerKey); err == nil && exists {
        exp.Add(int64(-add), int64(-del))
        // The expectations might've been modified since the update on the previous line.
        logger.V(4).Info("Lowered expectations", "expectations", exp)
    }
}
```
