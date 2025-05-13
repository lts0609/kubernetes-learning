# DeploymentController原理详解

在上一章节中，我们简单了解了ControllerManager的创建，本篇文章中深入学习Kubernetes中最重要的控制器之一`DeploymentController`。

当一个`Deployment`创建的时候，其实总共创建了三种资源对象，分别是`Deployment`、`ReplicaSet`、`Pod`，这是非常重要的，因为`Deployment`资源并不会直接管理`Pod`，而是通过管理`ReplicaSet`对象来间接地管理`Pod`，所以一个无状态负载的`Pod`的直接归属是`ReplicaSet`。这种设计和滚动更新相关，当一个`Deployment`中定义的`Pod`模板发生变化时，会创建出一个新的`ReplicaSet`，再根据一定的规则去替换旧的`ReplicaSet`对象。

## 实例创建

下面先从`DeploymentController`的创建开始学习，它的初始化函数如下，其中包含创建控制器实例和启动控制器两个逻辑。

```Go
func startDeploymentController(ctx context.Context, controllerContext ControllerContext, controllerName string) (controller.Interface, bool, error) {
    dc, err := deployment.NewDeploymentController(
        ctx,
        controllerContext.InformerFactory.Apps().V1().Deployments(),
        controllerContext.InformerFactory.Apps().V1().ReplicaSets(),
        controllerContext.InformerFactory.Core().V1().Pods(),
        controllerContext.ClientBuilder.ClientOrDie("deployment-controller"),
    )
    if err != nil {
        return nil, true, fmt.Errorf("error creating Deployment controller: %v", err)
    }
    go dc.Run(ctx, int(controllerContext.ComponentConfig.DeploymentController.ConcurrentDeploymentSyncs))
    return nil, true, nil
}
```

`DeploymentController`的实现逻辑都在`pkg/controller/deployment`路径下，`NewDeploymentController()`方法创建了一个控制器实例，根据函数签名来看，它接收上下文参数`ctx`，三种`Informer`对象用来监测`deployment/replicaset/pod`资源的变化，以及客户端`client`。照惯例先创建事件广播器和日志记录器，然后初始化`DeploymentController`对象，其中包括客户端、事件广播器、事件记录器、限速工作队列、和用于对`replicaset`对象进行`Patch`的操作器`rsControl`。再通过`AddEventHandler()`注册事件的处理函数，并初始化各种资源的`Lister`

```Go
func NewDeploymentController(ctx context.Context, dInformer appsinformers.DeploymentInformer, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, client clientset.Interface) (*DeploymentController, error) {
    // 初始化事件处理器和日志记录器
    eventBroadcaster := record.NewBroadcaster(record.WithContext(ctx))
    logger := klog.FromContext(ctx)
    // 初始化DeploymentController实例
    dc := &DeploymentController{
        client:           client,
        eventBroadcaster: eventBroadcaster,
        eventRecorder:    eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "deployment-controller"}),
        queue: workqueue.NewTypedRateLimitingQueueWithConfig(
            workqueue.DefaultTypedControllerRateLimiter[string](),
            workqueue.TypedRateLimitingQueueConfig[string]{
                Name: "deployment",
            },
        ),
    }
    dc.rsControl = controller.RealRSControl{
        KubeClient: client,
        Recorder:   dc.eventRecorder,
    }
    // 注册deployment资源变化处理函数
    dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            dc.addDeployment(logger, obj)
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            dc.updateDeployment(logger, oldObj, newObj)
        },
        // This will enter the sync loop and no-op, because the deployment has been deleted from the store.
        DeleteFunc: func(obj interface{}) {
            dc.deleteDeployment(logger, obj)
        },
    })
    // 注册replicaset资源变化处理函数
    rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            dc.addReplicaSet(logger, obj)
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            dc.updateReplicaSet(logger, oldObj, newObj)
        },
        DeleteFunc: func(obj interface{}) {
            dc.deleteReplicaSet(logger, obj)
        },
    })
    // 注册pod资源变化处理函数
    podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        DeleteFunc: func(obj interface{}) {
            dc.deletePod(logger, obj)
        },
    })
    // 注册调谐函数
    dc.syncHandler = dc.syncDeployment
    // 注册事件入队函数
    dc.enqueueDeployment = dc.enqueue
    // 初始化资源对象Lister
    dc.dLister = dInformer.Lister()
    dc.rsLister = rsInformer.Lister()
    dc.podLister = podInformer.Lister()
    // 初始化缓存状态检查函数
    dc.dListerSynced = dInformer.Informer().HasSynced
    dc.rsListerSynced = rsInformer.Informer().HasSynced
    dc.podListerSynced = podInformer.Informer().HasSynced
    return dc, nil
}
```

最后就返回了一个完整的`DeploymentController`对象，其结构如下。

```Go
type DeploymentController struct {
    // 用于操作replicaset对象
    rsControl controller.RSControlInterface
    // Kubernetes客户端
    client    clientset.Interface
    // 事件广播器
    eventBroadcaster record.EventBroadcaster
    // 事件记录器
    eventRecorder    record.EventRecorder
    // 同步函数
    syncHandler func(ctx context.Context, dKey string) error
    // 单测使用 入队函数
    enqueueDeployment func(deployment *apps.Deployment)
    // deployments资源的Lister
    dLister appslisters.DeploymentLister
    // replicaset资源的Lister
    rsLister appslisters.ReplicaSetLister
    // pod资源的Lister
    podLister corelisters.PodLister
    // 缓存状态检查函数
    dListerSynced cache.InformerSynced
    rsListerSynced cache.InformerSynced
    podListerSynced cache.InformerSynced
    // 限速队列
    queue workqueue.TypedRateLimitingInterface[string]
}
```

## 启动逻辑

运行控制器实例的代码如下，另起一个协程，传入上下文控制生命周期，还有一个参数表示允许并发同步的`deployment`对象数量。

```Go
go dc.Run(ctx, int(controllerContext.ComponentConfig.DeploymentController.ConcurrentDeploymentSyncs))
```

来看`Run()`方法的具体实现逻辑，首先还是标准的初始化流程和日志打印，然后会通过`WaitForNamedCacheSync()`方法确认`Informer`监听的资源是否同步成功，内部会调用`PollImmediateUntil()`函数阻塞等待`InformerSynced`返回的结果。然后根据传入的`worker`数值启动对应数量的协程去处理事件，最后通过接收`Done`信号的方式阻塞主线程。

```Go
func (dc *DeploymentController) Run(ctx context.Context, workers int) {
  // 异常处理 用于捕获panic
    defer utilruntime.HandleCrash()

    // 启动事件广播器
    dc.eventBroadcaster.StartStructuredLogging(3)
    dc.eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: dc.client.CoreV1().Events("")})
    // 退出时停止事件广播器和控制器的工作队列
  defer dc.eventBroadcaster.Shutdown()
    defer dc.queue.ShutDown()
    // 日志记录
    logger := klog.FromContext(ctx)
    logger.Info("Starting controller", "controller", "deployment")
    defer logger.Info("Shutting down controller", "controller", "deployment")
    // 确认缓存同步成功
    if !cache.WaitForNamedCacheSync("deployment", ctx.Done(), dc.dListerSynced, dc.rsListerSynced, dc.podListerSynced) {
        return
    }
    // 启动worker线程
    for i := 0; i < workers; i++ {
        go wait.UntilWithContext(ctx, dc.worker, time.Second)
    }
    // 阻塞主进程
    <-ctx.Done()
}
```

## 调谐基本流程

循环执行的逻辑是`worker()`方法，根据其中方法的命名，很明显它要做的就是不停地处理下一个元素。

```Go
func (dc *DeploymentController) worker(ctx context.Context) {
    for dc.processNextWorkItem(ctx) {
    }
}
```

来看元素是如何被处理的，首先从工作队列中取出一个元素，返回的是一个字符串类型的对象名称，然后交给调谐方法也就是`syncHandler()`去处理。如果获取元素时发现队列以及关闭了就返回一个`false`，`worker`协程也随之关闭。

```Go
func (dc *DeploymentController) processNextWorkItem(ctx context.Context) bool {
    // 取出一个元素
    key, quit := dc.queue.Get()
    // 如果队列为空且以及调用过shutdown关闭 quit会返回true
    if quit {
        return false
    }
    // 结束时通知队列处理完成(成功/失败重新入队)
    defer dc.queue.Done(key)
    // 通过调谐方法处理
    err := dc.syncHandler(ctx, key)
    // 错误处理
    dc.handleErr(ctx, err, key)

    return true
}
```

下面就是控制器中最核心的逻辑了，一般来说会叫做`reconciler()`，此处仅命名不同。在队列中取出`key`的格式为`namespcae/deploymentname`，调谐时会先切分出`namespace`和`name`，然后通过`Lister`从缓存中获取到具体的`Deployment`对象并拷贝，在调度器的学习过程中对于Pod的处理也是要拷贝的，因为缓存中是反映系统实际状态的信息，避免在处理过程中影响原始内容，所以后续操作都要用深拷贝的对象。在开始调谐逻辑之前会先检查`Deployment`对象的`Selector`字段是否为空，如果是则记录错误并跳过当前对象的调谐。

```Go
func (dc *DeploymentController) syncDeployment(ctx context.Context, key string) error {
    logger := klog.FromContext(ctx)
    // 获取命名空间和对象名称
    namespace, name, err := cache.SplitMetaNamespaceKey(key)
    if err != nil {
        logger.Error(err, "Failed to split meta namespace cache key", "cacheKey", key)
        return err
    }
    // 记录开始时间
    startTime := time.Now()
    logger.V(4).Info("Started syncing deployment", "deployment", klog.KRef(namespace, name), "startTime", startTime)
    // 延迟打印结束日志
    defer func() {
        logger.V(4).Info("Finished syncing deployment", "deployment", klog.KRef(namespace, name), "duration", time.Since(startTime))
    }()
    // 通过Lister获取deployment
    deployment, err := dc.dLister.Deployments(namespace).Get(name)
    if errors.IsNotFound(err) {
        logger.V(2).Info("Deployment has been deleted", "deployment", klog.KRef(namespace, name))
        return nil
    }
    if err != nil {
        return err
    }

    // 缓存中的对象只读 深拷贝对象以避免影响缓存内容
    d := deployment.DeepCopy()
    
    everything := metav1.LabelSelector{}
    // 保护逻辑 Selector为空时意味着选择所有Pod 这是一个错误事件
    if reflect.DeepEqual(d.Spec.Selector, &everything) {
        dc.eventRecorder.Eventf(d, v1.EventTypeWarning, "SelectingAll", "This deployment is selecting all pods. A non-empty selector is required.")
        // 直接更新Generation并返回 不执行后续调谐动作
        if d.Status.ObservedGeneration < d.Generation {
            d.Status.ObservedGeneration = d.Generation
            dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{})
        }
        return nil
    }

    // 获取属于该Deployment对象的ReplicaSet对象
    rsList, err := dc.getReplicaSetsForDeployment(ctx, d)
    if err != nil {
        return err
    }
    // 根据rsList获取该Deployment下的所有Pod
    podMap, err := dc.getPodMapForDeployment(d, rsList)
    if err != nil {
        return err
    }
    // 如果Deployment对象正在删除中 只更新状态并返回
    if d.DeletionTimestamp != nil {
        return dc.syncStatusOnly(ctx, d, rsList)
    }

    // 暂停或恢复时用unknown更新状态
    if err = dc.checkPausedConditions(ctx, d); err != nil {
        return err
    }
    // 暂停的处理逻辑
    if d.Spec.Paused {
        return dc.sync(ctx, d, rsList)
    }

    // 回滚的处理逻辑
    if getRollbackTo(d) != nil {
        return dc.rollback(ctx, d, rsList)
    }
    // 扩缩容的处理逻辑
    scalingEvent, err := dc.isScalingEvent(ctx, d, rsList)
    if err != nil {
        return err
    }
  
    if scalingEvent {
        return dc.sync(ctx, d, rsList)
    }
    // 根据滚动更新的策略处理
    switch d.Spec.Strategy.Type {
    case apps.RecreateDeploymentStrategyType:
        return dc.rolloutRecreate(ctx, d, rsList, podMap)
    case apps.RollingUpdateDeploymentStrategyType:
        return dc.rolloutRolling(ctx, d, rsList)
    }
    return fmt.Errorf("unexpected deployment strategy type: %s", d.Spec.Strategy.Type)
}
```

## 下属资源对象的获取

我们知道资源对象归属关系的匹配是基于标签选择的，在一个`yaml`文件的声明中，上层资源如`Deployment`、

`StatefulSet`等对下层资源如`Pod`的标签选择常有以下的表示形式：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      component: redis
    matchExpressions:
      - { key: tier, operator: In, values: [cache] }
      - { key: environment, operator: NotIn, values: [dev] }
  template:
    metadata:
      labels:
        component: redis
        tier: cache
        environment: test
    spec:
      containers:
      ......
```

标签的选择规则定义在字段`spec.selector`下，在和下层资源匹配时必须全部满足，所以在内部匹配时会进行一个非常重要的阶段，也就是把规则或一组规则的集合转换为统一的标识方法，然后在所有下层资源中过滤符合所有条件的，即认为两者具有从属关系。

### API资源的描述

根据`Deployment`类型为开始层层分析，首先`Deployment`结构体中包含`DeploymentSpec`类型的描述信息，后面如果学习`Operator`开发会了解到，一般定义一个`API`对象，通常会包含`metav1.TypeMeta`、`metav1.ObjectMeta`、`Spec`以及`Status`四个字段。

```Go
type Deployment struct {
    metav1.TypeMeta
    // +optional
    metav1.ObjectMeta

    // Specification of the desired behavior of the Deployment.
    // +optional
    Spec DeploymentSpec

    // Most recently observed status of the Deployment.
    // +optional
    Status DeploymentStatus
}
```

在一个`yaml`文件中，`apiVersion`和`kind`这两个字段属于`TypeMeta`，说明了`API`的版本和类型信息(GVK)，`metadata`字段属于`ObjectMeta`，描述了对象元数据，包括名称、命名空间、标签和注解，`spec`字段属于类型的`Spec`，表示该资源对象的期望状态，包括副本数量和容器配置等。

```yaml
# TypeMeta
apiVersion: apps/v1
kind: Deployment
----------------------------------------------------------
# ObjectMeta
metadata:
  name: my-deployment
  namespace: default
  labels:
    app: my-app
    tier: frontend
  annotations:
    description: This is my deployment
----------------------------------------------------------
# Spec
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: my-image
```

回到`DeploymentSpec`类型中，其`Selector`字段为`metav1.LabelSelector`类型的指针。

```Go
type DeploymentSpec struct {
    Replicas int32
    Selector *metav1.LabelSelector
    Template api.PodTemplateSpec
    Strategy DeploymentStrategy
    MinReadySeconds int32
    RevisionHistoryLimit *int32
    Paused bool
    RollbackTo *RollbackConfig
    ProgressDeadlineSeconds *int32
}
```

继续看`LabelSelector`类型的定义，它正符合在一个`yaml`文件中对于标签选择的定义规范，即：1.选择标签与某个值是匹配的；2.标签和某些值存在`In/NotIn/Exists/DoesNotExist`的关系。

```Go
type LabelSelector struct {
    // matchLabels is a map of {key,value} pairs. A single {key,value} in the matchLabels
    // map is equivalent to an element of matchExpressions, whose key field is "key", the
    // operator is "In", and the values array contains only "value". The requirements are ANDed.
    // +optional
    MatchLabels map[string]string `json:"matchLabels,omitempty" protobuf:"bytes,1,rep,name=matchLabels"`
    // matchExpressions is a list of label selector requirements. The requirements are ANDed.
    // +optional
    // +listType=atomic
    MatchExpressions []LabelSelectorRequirement `json:"matchExpressions,omitempty" protobuf:"bytes,2,rep,name=matchExpressions"`
}
```

### 标签选择的转换

上面说到过，在控制器内部进行从属资源选择时，会对上层资源进行标签的转换以匹配所属资，`metav1.LabelSelectorAsSelector()`方法实现了这一逻辑，把`metav1.LabelSelector`类型转换为`labels.Selector`对象，下面来看它的实现。

首先对传入的`LabelSelector`对象进行检查，如果是空则表示不匹配标签，不为空但长度是0表示匹配所有标签。首先处理`MatchLabels`字段，这一部分都是期望标签与目标值一致的，所以操作符使用`Equals`。然后遍历处理`MatchExpressions`字段，根据其中`Operator`的值进行转换，然后初始化一个`labels.Selector`接口，然后调用`Add()`方法添加之前处理好的标签，最终`Api`对象中的标签会以`labelkey--operator--labelvalue`切片的内部标签形式统一存在。

```Go
func LabelSelectorAsSelector(ps *LabelSelector) (labels.Selector, error) {
    // 对象检查
    if ps == nil {
        return labels.Nothing(), nil
    }
    if len(ps.MatchLabels)+len(ps.MatchExpressions) == 0 {
        return labels.Everything(), nil
    }
    requirements := make([]labels.Requirement, 0, len(ps.MatchLabels)+len(ps.MatchExpressions))
    // 处理MatchLabels字段
    for k, v := range ps.MatchLabels {
        r, err := labels.NewRequirement(k, selection.Equals, []string{v})
        if err != nil {
            return nil, err
        }
        requirements = append(requirements, *r)
    }
    // 处理MatchExpressions字段
    for _, expr := range ps.MatchExpressions {
        var op selection.Operator
        switch expr.Operator {
        case LabelSelectorOpIn:
            op = selection.In
        case LabelSelectorOpNotIn:
            op = selection.NotIn
        case LabelSelectorOpExists:
            op = selection.Exists
        case LabelSelectorOpDoesNotExist:
            op = selection.DoesNotExist
        default:
            return nil, fmt.Errorf("%q is not a valid label selector operator", expr.Operator)
        }
        r, err := labels.NewRequirement(expr.Key, op, append([]string(nil), expr.Values...))
        if err != nil {
            return nil, err
        }
        requirements = append(requirements, *r)
    }
    // 初始化一个internalSelector类型
    selector := labels.NewSelector()
    // 添加requirements
    selector = selector.Add(requirements...)
    return selector, nil
}
```

### Deployment下属资源的获取

#### ReplicaSet的获取

`getReplicaSetsForDeployment()`方法用于获取`Deployment`下属的`ReplicaSet`实例，首先获取命名空间下的所有`ReplocaSet`对象，然后把`Deployment`对象的标签解析为内部形式。基于`rsControl`、`Selector`等封装出一个`ReplicaSetControllerRefManager`结构对象用于处理该`Deployment`与`ReplicaSet`之间的从属关系，最后调用其`ClaimReplicaSets()`方法认领属于当前`Deployment`的`ReplicaSet`对象。

```Go
func (dc *DeploymentController) getReplicaSetsForDeployment(ctx context.Context, d *apps.Deployment) ([]*apps.ReplicaSet, error) {
    // 获取命名空间下所有replicaset
    rsList, err := dc.rsLister.ReplicaSets(d.Namespace).List(labels.Everything())
    if err != nil {
        return nil, err
    }
    // deployment标签转换
    deploymentSelector, err := metav1.LabelSelectorAsSelector(d.Spec.Selector)
    if err != nil {
        return nil, fmt.Errorf("deployment %s/%s has invalid label selector: %v", d.Namespace, d.Name, err)
    }
    // 认领replicaset前会再次检查 避免List和Adopt之间的deployment对象的变更
    canAdoptFunc := controller.RecheckDeletionTimestamp(func(ctx context.Context) (metav1.Object, error) {
        //直接从ApiServer获取最新对象 并通过UID进行一致性确认
        fresh, err := dc.client.AppsV1().Deployments(d.Namespace).Get(ctx, d.Name, metav1.GetOptions{})
        if err != nil {
            return nil, err
        }
        if fresh.UID != d.UID {
            return nil, fmt.Errorf("original Deployment %v/%v is gone: got uid %v, wanted %v", d.Namespace, d.Name, fresh.UID, d.UID)
        }
        return fresh, nil
    })
    // 创建Replicaset对象的引用管理器
    cm := controller.NewReplicaSetControllerRefManager(dc.rsControl, d, deploymentSelector, controllerKind, canAdoptFunc)
    // 认领replicaset
    return cm.ClaimReplicaSets(ctx, rsList)
}
```

`ReplicaSet`的认领逻辑在`ClaimReplicaSets()`方法中实现，其中定义了三个函数，分别对应`标签选择`、`认领`和`释放`。遍历`ReplicaSet`列表，然后把认领的对象加入`claimed`变量并返回给上层。

```Go
func (m *ReplicaSetControllerRefManager) ClaimReplicaSets(ctx context.Context, sets []*apps.ReplicaSet) ([]*apps.ReplicaSet, error) {
    var claimed []*apps.ReplicaSet
    var errlist []error
    // 三个辅助函数
    match := func(obj metav1.Object) bool {
        return m.Selector.Matches(labels.Set(obj.GetLabels()))
    }
    adopt := func(ctx context.Context, obj metav1.Object) error {
        return m.AdoptReplicaSet(ctx, obj.(*apps.ReplicaSet))
    }
    release := func(ctx context.Context, obj metav1.Object) error {
        return m.ReleaseReplicaSet(ctx, obj.(*apps.ReplicaSet))
    }
    // 遍历处理
    for _, rs := range sets {
        ok, err := m.ClaimObject(ctx, rs, match, adopt, release)
        if err != nil {
            errlist = append(errlist, err)
            continue
        }
        if ok {
            claimed = append(claimed, rs)
        }
    }
    return claimed, utilerrors.NewAggregate(errlist)
}
```

具体的处理逻辑体现在`ClaimObkect()`方法中，其中包含很多的`if-else`，下面进行分析。

拿到`RepicaSet`对象的第一步是获取它的`OwnerReferences`对象，判断逻辑如下：

* 第一种情况：所属控制器存在
    * 其所属控制器存在，但不是当前的控制器，跳过处理；
    * 其所属控制器存在，是当前的控制器，还需要检查一次标签选择，避免由于`Selector`动态修改导致的不匹配；
    * 其所属控制器存在，是当前的控制器，但标签不匹配，如果当前控制器正在删除中，也跳过处理；
    * 其所属控制器存在，是当前的控制器，但标签不匹配，控制器正常，尝试释放对象；
* 第二种情况：所属控制器不存在，孤儿对象
    * 控制器被删除或标签不匹配，跳过处理；
    * 控制器被删除或标签匹配，`ReplicaSet`对象正在被删除，跳过处理；
    * 控制器被删除或标签匹配，`ReplicaSet`对象正常，命名空间不匹配，跳过处理；
    * 控制器被删除或标签匹配，`ReplicaSet`对象正常，命名空间匹配，尝试认领；

```Go
func (m *BaseControllerRefManager) ClaimObject(ctx context.Context, obj metav1.Object, match func(metav1.Object) bool, adopt, release func(context.Context, metav1.Object) error) (bool, error) {
    controllerRef := metav1.GetControllerOfNoCopy(obj)
    // 有所属控制器
    if controllerRef != nil {
        // 不属于当前控制器
        if controllerRef.UID != m.Controller.GetUID() {
            // 忽略
            return false, nil
        }
        // 属于当前控制器
        if match(obj) {
            // 标签匹配 返回
            return true, nil
        }
        // 属于当前控制器 标签不匹配
        if m.Controller.GetDeletionTimestamp() != nil {
            // 控制器在被删除 忽略
            return false, nil
        }
        // 控制器没被删除 释放
        if err := release(ctx, obj); err != nil {
            // 对象以及不存在了 忽略
            if errors.IsNotFound(err) {
                return false, nil
            }
            // 可能被其他人释放 忽略
            return false, err
        }
        // 成功释放
        return false, nil
    }

    // 另一种情况 没有所属控制器：孤儿对象
    if m.Controller.GetDeletionTimestamp() != nil || !match(obj) {
        // 控制器正在被删除或标签不匹配 忽略
        return false, nil
    }
    // 控制器没被删除 标签也匹配
    if obj.GetDeletionTimestamp() != nil {
        // 目标对象正在被删除 忽略
        return false, nil
    }

    if len(m.Controller.GetNamespace()) > 0 && m.Controller.GetNamespace() != obj.GetNamespace() {
        // 命名空间不匹配 忽略
        return false, nil
    }

    // 控制器正常 标签匹配 命名空间匹配 尝试认领
    if err := adopt(ctx, obj); err != nil {
        // 对象以及被删除 忽略
        if errors.IsNotFound(err) {
            return false, nil
        }
        // 已被其他人认领 忽略
        return false, err
    }
    // 认领成功
    return true, nil
}
```

#### Pod的获取

确认了`Deployment`下属的`ReplicaSet`列表后，使用`getPodMapForDeployment()`方法获取`Pod`的列表。根据函数签名，入参是`Deployment`和`ReplicaSet`列表，返回的是一个以`ReplicaSet`的`UID`为key，`Pod`对象为value的列表。

首先进行控制器的标签转换，再获取到同一命名空间下标签匹配的`Pod`列表。在滚动更新过程中，可能存在多个`ReplicaSet`实例，并且每个实例下都还包含`Pod`，所以会先以`ReplicaSet`实例的`UID`为key初始化一个Map，然后遍历所有`Pod`，

```Go
func (dc *DeploymentController) getPodMapForDeployment(d *apps.Deployment, rsList []*apps.ReplicaSet) (map[types.UID][]*v1.Pod, error) {
    // 标签转换
    selector, err := metav1.LabelSelectorAsSelector(d.Spec.Selector)
    if err != nil {
        return nil, err
    }
    // 列出命名空间下所有标签匹配Pod
    pods, err := dc.podLister.Pods(d.Namespace).List(selector)
    if err != nil {
        return nil, err
    }
    // 以UID为key初始化空集合
    podMap := make(map[types.UID][]*v1.Pod, len(rsList))
    for _, rs := range rsList {
        podMap[rs.UID] = []*v1.Pod{}
    }
    // 遍历Pod 根据其OwnerReference的UID加入对应集合
    for _, pod := range pods {
        controllerRef := metav1.GetControllerOf(pod)
        if controllerRef == nil {
            continue
        }
        // Only append if we care about this UID.
        if _, ok := podMap[controllerRef.UID]; ok {
            podMap[controllerRef.UID] = append(podMap[controllerRef.UID], pod)
        }
    }
    return podMap, nil
}
```

## 调谐的具体动作

根据不同的场景会有不同的调谐动作，场景大概可以分为几类：

* `Deployment`对象正在删除中
* `Deployment`对象手动暂停
* `Deployment`对象需要回滚
* `Deployment`对象副本扩缩容
* `Deployment`对象滚动更新

下面根据几种场景，结合代码分别进行详细的说明。

### Deployment对象正在删除中

在这种情况下，仅会同步状态，但不做任何可能影响资源状态的操作。

```Go
func (dc *DeploymentController) syncStatusOnly(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet) error {
    // 获取新就版本的replicaset
    newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, false)
    if err != nil {
        return err
    }
    // 合并replicaset
    allRSs := append(oldRSs, newRS)
    // 同步deployment的status
    return dc.syncDeploymentStatus(ctx, allRSs, newRS, d)
}
```

其中`getAllReplicaSetsAndSyncRevision()`方法用于获取所有新旧版本的`ReplicaSet`对象，是`Deployment Controller`调谐过程中的一个通用方法，在`rolling\rollback\recreate`过程中也被使用。

```Go
func (dc *DeploymentController) getAllReplicaSetsAndSyncRevision(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet, createIfNotExisted bool) (*apps.ReplicaSet, []*apps.ReplicaSet, error) {
   // 找到所有旧的replicaset
    _, allOldRSs := deploymentutil.FindOldReplicaSets(d, rsList)

    // 获取新的replicaset并更新版本号
    newRS, err := dc.getNewReplicaSet(ctx, d, rsList, allOldRSs, createIfNotExisted)
    if err != nil {
        return nil, nil, err
    }

    return newRS, allOldRSs, nil
}
```

#### 获取旧的ReplicaSet对象

这部分的逻辑也比较简单，首先获取新的`ReplicaSet`对象，然后遍历所有`ReplicaSet`并根据`UID`判断是否是旧的对象，并且如果旧的`ReplicaSet`还关联`Pod`，单独存放一份到`requiredRSs`中，返回的两个列表分别是：有`Pod`存在的旧`ReplicaSet`和旧`ReplicaSet`全集。

```Go
func FindOldReplicaSets(deployment *apps.Deployment, rsList []*apps.ReplicaSet) ([]*apps.ReplicaSet, []*apps.ReplicaSet) {
    var requiredRSs []*apps.ReplicaSet
    var allRSs []*apps.ReplicaSet
    newRS := FindNewReplicaSet(deployment, rsList)
    for _, rs := range rsList {
        // Filter out new replica set
        if newRS != nil && rs.UID == newRS.UID {
            continue
        }
        allRSs = append(allRSs, rs)
        if *(rs.Spec.Replicas) != 0 {
            requiredRSs = append(requiredRSs, rs)
        }
    }
    return requiredRSs, allRSs
}
```

#### 获取新的ReplicaSet对象

来看新的对象是如何获取的，`ReplicaSetsByCreationTimestamp`类型是`[]*apps.ReplicaSet`类型的别名，专门为了实现`ReplicaSet`对象基于创建时间戳的排序而存在。第一步是先对所有的`ReplicaSet`进行排序，按照创建时间戳升序排列。第二步会遍历所有的对象，返回和最新`ReplicaSet`对象的`Template`描述完全一致的最早版本，这是Kubernetes中**确定性原则**的体现：避免了随机选择，并且避免了集群信息中存在多个相同`Template`的`ReplicaSet`情况下的处理异常。

```Go
func FindNewReplicaSet(deployment *apps.Deployment, rsList []*apps.ReplicaSet) *apps.ReplicaSet {
  // 按创建时间升序排列replicaset
    sort.Sort(controller.ReplicaSetsByCreationTimestamp(rsList))
    for i := range rsList {
        if EqualIgnoreHash(&rsList[i].Spec.Template, &deployment.Spec.Template) {
            // In rare cases, such as after cluster upgrades, Deployment may end up with
            // having more than one new ReplicaSets that have the same template as its template,
            // see https://github.com/kubernetes/kubernetes/issues/40415
            // We deterministically choose the oldest new ReplicaSet.
            return rsList[i]
        }
    }
    // new ReplicaSet does not exist.
    return nil
}
```

`Template`字段的比较函数如下，先对两个对象做深拷贝，然后删除`ReplicaSet`对象的`pod-template-hash`标签，该标签是在`ReplicaSet`创建时自动添加的根据Pod模板哈希而来的一个`Label`，用于帮助`ReplicaSet`选择并隔离不同版本的`Pod`，此处的一致性判断逻辑关注于用户的配置，删除该标签避免了用户配置相同但哈希结果不同的特殊情况。

```Go
func EqualIgnoreHash(template1, template2 *v1.PodTemplateSpec) bool {
    t1Copy := template1.DeepCopy()
    t2Copy := template2.DeepCopy()
    // Remove hash labels from template.Labels before comparing
    delete(t1Copy.Labels, apps.DefaultDeploymentUniqueLabelKey)
    delete(t2Copy.Labels, apps.DefaultDeploymentUniqueLabelKey)
    return apiequality.Semantic.DeepEqual(t1Copy, t2Copy)
}
```

#### 处理新的ReplicaSet对象

在`getAllReplicaSetsAndSyncRevision()`方法中，新`ReplicaSet`对象是由`getNewReplicaSet()`方法返回的，用于生成和管理滚动更新过程中的`ReplicaSet`新对象。

首先尝试获取最新的`ReplicaSet`对象和最新对象的预期版本号`Revision`，如果该`ReplicaSet`对象存在检查其是否需要更新，如果要更新就向`ApiServer`发送一个更新请求并返回，然后检查`Deployment`对象是否需要更新，如果需要更新同样向`ApiServer`请求。如果`ReplicaSet`更新使函数返回，不用担心`Deployment`对象无法被更新，因为`ReplicaSet`的更新可以触发控制器的调谐动作，如果`Deployment`对象需要更新也会在下个调谐周期被处理。

如果预期的`ReplicaSet`对象不存在，就需要去创建它，然后更新`Deployment`对象，最后返回新的`ReplicaSet`对象。

```Go
func (dc *DeploymentController) getNewReplicaSet(ctx context.Context, d *apps.Deployment, rsList, oldRSs []*apps.ReplicaSet, createIfNotExisted bool) (*apps.ReplicaSet, error) {
    logger := klog.FromContext(ctx)
    // 获取最新replicaset
    existingNewRS := deploymentutil.FindNewReplicaSet(d, rsList)

    // 获取旧replicaset的最大版本号
    maxOldRevision := deploymentutil.MaxRevision(logger, oldRSs)
    // 新replicaset的版本号设置为maxOldRevision+1
    newRevision := strconv.FormatInt(maxOldRevision+1, 10)

    // 最新的replicaset已经存在时
    if existingNewRS != nil {
        rsCopy := existingNewRS.DeepCopy()

        // replicaset对象的注解是否更新
        annotationsUpdated := deploymentutil.SetNewReplicaSetAnnotations(ctx, d, rsCopy, newRevision, true, maxRevHistoryLengthInChars)
        // MinReadySeconds字段是否更新
        minReadySecondsNeedsUpdate := rsCopy.Spec.MinReadySeconds != d.Spec.MinReadySeconds
        // 如果需要更新 向ApiServer发送更新请求
        if annotationsUpdated || minReadySecondsNeedsUpdate {
            rsCopy.Spec.MinReadySeconds = d.Spec.MinReadySeconds
            return dc.client.AppsV1().ReplicaSets(rsCopy.ObjectMeta.Namespace).Update(ctx, rsCopy, metav1.UpdateOptions{})
        }

        // deployment对象的版本号是否要更新
        needsUpdate := deploymentutil.SetDeploymentRevision(d, rsCopy.Annotations[deploymentutil.RevisionAnnotation])
        // deployment对象是否有进度状态条件信息
        cond := deploymentutil.GetDeploymentCondition(d.Status, apps.DeploymentProgressing)
        // 如果设置了进度截止时间但没有状态条件信息
        if deploymentutil.HasProgressDeadline(d) && cond == nil {
            msg := fmt.Sprintf("Found new replica set %q", rsCopy.Name)
            // 更新deployment状态条件信息字段和标识位needsUpdate
            condition := deploymentutil.NewDeploymentCondition(apps.DeploymentProgressing, v1.ConditionTrue, deploymentutil.FoundNewRSReason, msg)
            deploymentutil.SetDeploymentCondition(&d.Status, *condition)
            needsUpdate = true
        }
        // 如果deployment需要更新 同样向ApiServer发送更新请求
        if needsUpdate {
            var err error
            if _, err = dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{}); err != nil {
                return nil, err
            }
        }
        // 返回最终的新replicaset对象
        return rsCopy, nil
    }
    // 如果最新replicaset不存在 但是不允许创建
    if !createIfNotExisted {
        return nil, nil
    }

    // 如果最新replicaset不存在 需要创建
    newRSTemplate := *d.Spec.Template.DeepCopy()
    podTemplateSpecHash := controller.ComputeHash(&newRSTemplate, d.Status.CollisionCount)
    newRSTemplate.Labels = labelsutil.CloneAndAddLabel(d.Spec.Template.Labels, apps.DefaultDeploymentUniqueLabelKey, podTemplateSpecHash)
    // Selector中也需要pod-template-hash
    newRSSelector := labelsutil.CloneSelectorAndAddLabel(d.Spec.Selector, apps.DefaultDeploymentUniqueLabelKey, podTemplateSpecHash)

    // 组装ReplicaSet对象
    newRS := apps.ReplicaSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:            d.Name + "-" + podTemplateSpecHash,
            Namespace:       d.Namespace,
            OwnerReferences: []metav1.OwnerReference{*metav1.NewControllerRef(d, controllerKind)},
            Labels:          newRSTemplate.Labels,
        },
        Spec: apps.ReplicaSetSpec{
            Replicas:        new(int32),
            MinReadySeconds: d.Spec.MinReadySeconds,
            Selector:        newRSSelector,
            Template:        newRSTemplate,
        },
    }
    allRSs := append(oldRSs, &newRS)
    // 获取目标副本数
    newReplicasCount, err := deploymentutil.NewRSNewReplicas(d, allRSs, &newRS)
    if err != nil {
        return nil, err
    }
    // 更新副本数
    *(newRS.Spec.Replicas) = newReplicasCount
    // 设置ReplicaSet对象的注解
    deploymentutil.SetNewReplicaSetAnnotations(ctx, d, &newRS, newRevision, false, maxRevHistoryLengthInChars)
    // 创建ReplicaSet对象并处理异常
    alreadyExists := false
    createdRS, err := dc.client.AppsV1().ReplicaSets(d.Namespace).Create(ctx, &newRS, metav1.CreateOptions{})
    switch {
    // ReplicaSet对象已经存在(哈希冲突)
    case errors.IsAlreadyExists(err):
        alreadyExists = true
        rs, rsErr := dc.rsLister.ReplicaSets(newRS.Namespace).Get(newRS.Name)
        if rsErr != nil {
            return nil, rsErr
        }

        controllerRef := metav1.GetControllerOf(rs)
        if controllerRef != nil && controllerRef.UID == d.UID && deploymentutil.EqualIgnoreHash(&d.Spec.Template, &rs.Spec.Template) {
            createdRS = rs
            err = nil
            break
        }

        if d.Status.CollisionCount == nil {
            d.Status.CollisionCount = new(int32)
        }
        preCollisionCount := *d.Status.CollisionCount
        *d.Status.CollisionCount++
        // Update the collisionCount for the Deployment and let it requeue by returning the original
        // error.
        _, dErr := dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{})
        if dErr == nil {
            logger.V(2).Info("Found a hash collision for deployment - bumping collisionCount to resolve it", "deployment", klog.KObj(d), "oldCollisionCount", preCollisionCount, "newCollisionCount", *d.Status.CollisionCount)
        }
        return nil, err
  // 命名空间正在删除导致的异常  
    case errors.HasStatusCause(err, v1.NamespaceTerminatingCause):
        // if the namespace is terminating, all subsequent creates will fail and we can safely do nothing
        return nil, err
    case err != nil:
        msg := fmt.Sprintf("Failed to create new replica set %q: %v", newRS.Name, err)
        if deploymentutil.HasProgressDeadline(d) {
            cond := deploymentutil.NewDeploymentCondition(apps.DeploymentProgressing, v1.ConditionFalse, deploymentutil.FailedRSCreateReason, msg)
            deploymentutil.SetDeploymentCondition(&d.Status, *cond)
            _, _ = dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{})
        }
        dc.eventRecorder.Eventf(d, v1.EventTypeWarning, deploymentutil.FailedRSCreateReason, msg)
        return nil, err
    }
    // 创建成功记录事件
    if !alreadyExists && newReplicasCount > 0 {
        dc.eventRecorder.Eventf(d, v1.EventTypeNormal, "ScalingReplicaSet", "Scaled up replica set %s from 0 to %d", createdRS.Name, newReplicasCount)
    }
    // 检查Deployment是否需要更新
    needsUpdate := deploymentutil.SetDeploymentRevision(d, newRevision)
    if !alreadyExists && deploymentutil.HasProgressDeadline(d) {
        msg := fmt.Sprintf("Created new replica set %q", createdRS.Name)
        condition := deploymentutil.NewDeploymentCondition(apps.DeploymentProgressing, v1.ConditionTrue, deploymentutil.NewReplicaSetReason, msg)
        deploymentutil.SetDeploymentCondition(&d.Status, *condition)
        needsUpdate = true
    }
    // 更新Deployment对象
    if needsUpdate {
        _, err = dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{})
    }
    return createdRS, err
}
```

##### 给新的ReplicaSet对象设置注解

`SetNewReplicaSetAnnotations()`方法返回一个`bool`值来表示注解是否被修改。

```Go
func SetNewReplicaSetAnnotations(ctx context.Context, deployment *apps.Deployment, newRS *apps.ReplicaSet, newRevision string, exists bool, revHistoryLimitInChars int) bool {
    logger := klog.FromContext(ctx)
    // 基于deployment对象更新注解
    annotationChanged := copyDeploymentAnnotationsToReplicaSet(deployment, newRS)
    // 更新注解部分的版本号Revision
    if newRS.Annotations == nil {
        newRS.Annotations = make(map[string]string)
    }
    // 获取replicaset对象当前的版本号
    oldRevision, ok := newRS.Annotations[RevisionAnnotation]
    oldRevisionInt, err := strconv.ParseInt(oldRevision, 10, 64)
    if err != nil {
        if oldRevision != "" {
            logger.Info("Updating replica set revision OldRevision not int", "err", err)
            return false
        }
        //If the RS annotation is empty then initialise it to 0
        oldRevisionInt = 0
    }
    newRevisionInt, err := strconv.ParseInt(newRevision, 10, 64)
    if err != nil {
        logger.Info("Updating replica set revision NewRevision not int", "err", err)
        return false
    }
    // 比较replicaset对象当前版本号和目标版本号是否相等
    if oldRevisionInt < newRevisionInt {
        // 需要更新
        newRS.Annotations[RevisionAnnotation] = newRevision
        // 修改标识位
        annotationChanged = true
        logger.V(4).Info("Updating replica set revision", "replicaSet", klog.KObj(newRS), "newRevision", newRevision)
    }
    // 如果版本号不一致表明此处发生更新动作 需要记录信息
    if ok && oldRevisionInt < newRevisionInt {
        revisionHistoryAnnotation := newRS.Annotations[RevisionHistoryAnnotation]
        oldRevisions := strings.Split(revisionHistoryAnnotation, ",")
        // 第一个元素是空字符串表示之前没有记录过revision信息
        if len(oldRevisions[0]) == 0 {
            newRS.Annotations[RevisionHistoryAnnotation] = oldRevision
        } else {
            // 计算一个新的总长度
            // 例如revisionHistoryAnnotation的值是"1,2,3" 长度5 oldRevision是"4" 那么新的例如revisionHistoryAnnotation的值是应该是"1,2,3,4" 长度7
            totalLen := len(revisionHistoryAnnotation) + len(oldRevision) + 1
            // 避免RevisionHistoryAnnotation字符串长度超过最大限制
            start := 0
            // 如果超过限制 每次减去一个Revision数值和一个逗号
            for totalLen > revHistoryLimitInChars && start < len(oldRevisions) {
                totalLen = totalLen - len(oldRevisions[start]) - 1
                start++
            }
            // 长度符合限制时 把oldRevision加入切片 并Join新的字符串替换原有注解
            if totalLen <= revHistoryLimitInChars {
                oldRevisions = append(oldRevisions[start:], oldRevision)
                newRS.Annotations[RevisionHistoryAnnotation] = strings.Join(oldRevisions, ",")
            } else {
                logger.Info("Not appending revision due to revision history length limit reached", "revisionHistoryLimit", revHistoryLimitInChars)
            }
        }
    }
    // 如果新replicaset不存在(本次传入的事false) 需要创建新的对象 此时标识位也直为true
    if !exists && SetReplicasAnnotations(newRS, *(deployment.Spec.Replicas), *(deployment.Spec.Replicas)+MaxSurge(*deployment)) {
        annotationChanged = true
    }
    return annotationChanged
}
```





