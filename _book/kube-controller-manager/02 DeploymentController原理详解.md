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

`DeploymentController`的实现逻辑都在`pkg/controller/deployment`路径下，`NewDeploymentController()`方法创建了一个控制器实例，根据函数签名来看，它接收上下文参数`ctx`，三种`Informer`对象用来监测`Deployment/ReplicaSet/Pod`资源的变化，以及客户端`client`。照惯例先创建事件广播器和日志记录器，然后初始化`DeploymentController`对象，其中包括客户端、事件广播器、事件记录器、限速工作队列、和用于对`ReplicaSet`对象进行`Patch`的操作器`rsControl`。再通过`AddEventHandler()`注册事件的处理函数，并初始化各种资源的`Lister`

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
    // 注册Deployment资源变化处理函数
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
    // 注册ReplicaSet资源变化处理函数
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
    // 注册Pod资源变化处理函数
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
    // 用于操作ReplicaSet对象
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
    // Deployments资源的Lister
    dLister appslisters.DeploymentLister
    // ReplicaSet资源的Lister
    rsLister appslisters.ReplicaSetLister
    // Pod资源的Lister
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

运行控制器实例的代码如下，另起一个协程，传入上下文控制生命周期，还有一个参数表示允许并发同步的`Deployment`对象数量。

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

来看元素是如何被处理的，首先从工作队列中取出一个元素，返回的是一个字符串类型的对象名称，然后交给调谐方法也就是`syncHandler()`去处理。如果获取元素时发现队列已经关闭了就返回一个`false`，`worker`协程也随之关闭。

```Go
func (dc *DeploymentController) processNextWorkItem(ctx context.Context) bool {
    // 取出一个元素
    key, quit := dc.queue.Get()
    // 如果队列为空且已经调用过shutdown关闭 quit会返回true
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
    // 通过Lister获取Deployment
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
    // 获取命名空间下所有ReplicaSet
    rsList, err := dc.rsLister.ReplicaSets(d.Namespace).List(labels.Everything())
    if err != nil {
        return nil, err
    }
    // Deployment标签转换
    deploymentSelector, err := metav1.LabelSelectorAsSelector(d.Spec.Selector)
    if err != nil {
        return nil, fmt.Errorf("deployment %s/%s has invalid label selector: %v", d.Namespace, d.Name, err)
    }
    // 认领ReplicaSet前会再次检查 避免List和Adopt之间的Deployment对象的变更
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
    // 认领ReplicaSet
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
            // 对象已经不存在了 忽略
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
        // 对象已经被删除 忽略
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
    // 获取新就版本的ReplicaSet
    newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, false)
    if err != nil {
        return err
    }
    // 合并ReplicaSet
    allRSs := append(oldRSs, newRS)
    // 同步Deployment的status
    return dc.syncDeploymentStatus(ctx, allRSs, newRS, d)
}
```

其中`getAllReplicaSetsAndSyncRevision()`方法用于获取所有新旧版本的`ReplicaSet`对象，是`Deployment Controller`调谐过程中的一个通用方法，在`rolling\rollback\recreate`过程中也被使用。

```Go
func (dc *DeploymentController) getAllReplicaSetsAndSyncRevision(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet, createIfNotExisted bool) (*apps.ReplicaSet, []*apps.ReplicaSet, error) {
   // 找到所有旧的ReplicaSet
    _, allOldRSs := deploymentutil.FindOldReplicaSets(d, rsList)

    // 获取新的ReplicaSet并更新版本号
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
  // 按创建时间升序排列ReplicaSet
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
    // 获取最新ReplicaSet
    existingNewRS := deploymentutil.FindNewReplicaSet(d, rsList)

    // 获取旧ReplicaSet的最大版本号
    maxOldRevision := deploymentutil.MaxRevision(logger, oldRSs)
    // 新ReplicaSet的版本号设置为maxOldRevision+1
    newRevision := strconv.FormatInt(maxOldRevision+1, 10)

    // 最新的ReplicaSet已经存在时
    if existingNewRS != nil {
        rsCopy := existingNewRS.DeepCopy()

        // ReplicaSet对象的注解是否更新
        annotationsUpdated := deploymentutil.SetNewReplicaSetAnnotations(ctx, d, rsCopy, newRevision, true, maxRevHistoryLengthInChars)
        // MinReadySeconds字段是否更新
        minReadySecondsNeedsUpdate := rsCopy.Spec.MinReadySeconds != d.Spec.MinReadySeconds
        // 如果需要更新 向ApiServer发送更新请求
        if annotationsUpdated || minReadySecondsNeedsUpdate {
            rsCopy.Spec.MinReadySeconds = d.Spec.MinReadySeconds
            return dc.client.AppsV1().ReplicaSets(rsCopy.ObjectMeta.Namespace).Update(ctx, rsCopy, metav1.UpdateOptions{})
        }

        // Deployment对象的版本号是否要更新
        needsUpdate := deploymentutil.SetDeploymentRevision(d, rsCopy.Annotations[deploymentutil.RevisionAnnotation])
        // Deployment对象是否有进度状态条件信息
        cond := deploymentutil.GetDeploymentCondition(d.Status, apps.DeploymentProgressing)
        // 如果设置了进度截止时间但没有状态条件信息
        if deploymentutil.HasProgressDeadline(d) && cond == nil {
            msg := fmt.Sprintf("Found new replica set %q", rsCopy.Name)
            // 更新Deployment状态条件信息字段和标识位needsUpdate
            condition := deploymentutil.NewDeploymentCondition(apps.DeploymentProgressing, v1.ConditionTrue, deploymentutil.FoundNewRSReason, msg)
            deploymentutil.SetDeploymentCondition(&d.Status, *condition)
            needsUpdate = true
        }
        // 如果Deployment需要更新 同样向ApiServer发送更新请求
        if needsUpdate {
            var err error
            if _, err = dc.client.AppsV1().Deployments(d.Namespace).UpdateStatus(ctx, d, metav1.UpdateOptions{}); err != nil {
                return nil, err
            }
        }
        // 返回最终的新ReplicaSet对象
        return rsCopy, nil
    }
    // 如果最新ReplicaSet不存在 但是不允许创建
    if !createIfNotExisted {
        return nil, nil
    }

    // 如果最新ReplicaSet不存在 需要创建
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
    // 基于Deployment对象更新注解
    annotationChanged := copyDeploymentAnnotationsToReplicaSet(deployment, newRS)
    // 更新注解部分的版本号Revision
    if newRS.Annotations == nil {
        newRS.Annotations = make(map[string]string)
    }
    // 获取ReplicaSet对象当前的版本号
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
    // 比较ReplicaSet对象当前版本号和目标版本号是否相等
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
    // 如果新ReplicaSet不存在(本次传入的事false) 需要创建新的对象 此时标识位也直为true
    if !exists && SetReplicasAnnotations(newRS, *(deployment.Spec.Replicas), *(deployment.Spec.Replicas)+MaxSurge(*deployment)) {
        annotationChanged = true
    }
    return annotationChanged
}
```

#### 同步Deployment对象的状态

该流程的最后一步是同步`Deployment`对象的状态，逻辑很简单，首先计算一个预期的`Status`，然后和原始数据做比较，如果不同就向`ApiServer`发送一个更新请求。

```Go
func (dc *DeploymentController) syncDeploymentStatus(ctx context.Context, allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, d *apps.Deployment) error {
    newStatus := calculateStatus(allRSs, newRS, d)

    if reflect.DeepEqual(d.Status, newStatus) {
        return nil
    }

    newDeployment := d
    newDeployment.Status = newStatus
    _, err := dc.client.AppsV1().Deployments(newDeployment.Namespace).UpdateStatus(ctx, newDeployment, metav1.UpdateOptions{})
    return err
}
```

### Deployment对象需要扩缩容或被手动暂停

在`d.Spec.Paused`的值为`true`时，表示`Deployment`对象被手动暂停，`isScalingEvent()`方法根据`Deployment`对象的`Spec.Replicas`与注释信息`"desired-replicas"`的值是否一致来判断是否要进行扩缩容操作。两种情况的结果都是直接调用`sync()`方法。
扩缩容的入口`sync()`方法对比`syncStatusOnly()`方法多了两个步骤，一个是执行扩缩容操作`scale()`，另一个差别是判断如果是暂停状态且没有回滚目标，就需要清理旧的`ReplicaSet`对象。

```Go
func (dc *DeploymentController) sync(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet) error {
    newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, false)
    if err != nil {
        return err
    }
    if err := dc.scale(ctx, d, newRS, oldRSs); err != nil {
        // If we get an error while trying to scale, the deployment will be requeued
        // so we can abort this resync
        return err
    }

    // Clean up the deployment when it's paused and no rollback is in flight.
    if d.Spec.Paused && getRollbackTo(d) == nil {
        if err := dc.cleanupDeployment(ctx, oldRSs, d); err != nil {
            return err
        }
    }

    allRSs := append(oldRSs, newRS)
    return dc.syncDeploymentStatus(ctx, allRSs, newRS, d)
}
```

#### 扩缩容逻辑

`scale()`是管理`ReplicaSet`副本数的核心方法，其中包含三个实际场景以及处理逻辑。

##### 场景一

```Go
func (dc *DeploymentController) scale(ctx context.Context, deployment *apps.Deployment, newRS *apps.ReplicaSet, oldRSs []*apps.ReplicaSet) error {
    // 场景一：单ReplicaSet活跃
    if activeOrLatest := deploymentutil.FindActiveOrLatest(newRS, oldRSs); activeOrLatest != nil {
        if *(activeOrLatest.Spec.Replicas) == *(deployment.Spec.Replicas) {
            return nil
        }
        // 实际副本数和期望副本数不一致 对其进行调节
        _, _, err := dc.scaleReplicaSetAndRecordEvent(ctx, activeOrLatest, *(deployment.Spec.Replicas), deployment)
        return err
    }
    ......
}
```

只有一个`ReplicaSet`实例存在`Pod`，具体情况包括：1.新的`ReplicaSet`被创建；2.回滚后只剩下旧`ReplicaSet`；3.滚动更新时旧`ReplicaSet`副本已经归零。

首先获取活跃的`ReplicaSet`对象，所谓活跃就是副本数大于0。其内部逻辑是在所有的`ReplicaSet`中查找副本数大于0的对象，如果结果是1个直接返回；如果是0个，先看最新的`ReplicaSet`对象是否存在，如果存在就返回它，不存在就返回列表中第一个旧的`ReplicaSet`对象；如果超过1个表示正在滚动更新过程中，返回`nil`。如果`activeOrLatest`不为空，对比活跃`ReplicaSet`对象的副本数和`Deployment`对象中是否是一致的，如果一致则不做处理。数量不一致则调用`scaleReplicaSetAndRecordEvent()`方法调整副本数，保证在后续逻辑开始前`ReplicaSet`副本实际状态和预期状态的一致性。

##### 场景二

```Go
func (dc *DeploymentController) scale(ctx context.Context, deployment *apps.Deployment, newRS *apps.ReplicaSet, oldRSs []*apps.ReplicaSet) error {
    ......
    // 场景二：新ReplicaSet已经饱和 需要缩容旧ReplicaSet
    if deploymentutil.IsSaturated(deployment, newRS) {
        // 找到旧ReplicaSet中实例不为0的
        for _, old := range controller.FilterActiveReplicaSets(oldRSs) {
            // 以0为期望值进行更新
            if _, _, err := dc.scaleReplicaSetAndRecordEvent(ctx, old, 0, deployment); err != nil {
                return err
            }
        }
        return nil
    }
  ......
}
```

清理阶段，新的`ReplicaSet`已经饱和(副本数量达到`Deployment`的期望)，需要把剩余旧的`ReplicaSet`管理的副本数量缩至0。判断依据是`ReplicaSet`中三个字段的值要和`Deployment`的`Spec.Replicas`中设置相同：1.`Spec.Replicas`；2.`Annotations`中`desired-replicas`的值；3.`Status.AvailableReplicas`。此处可能会有疑问，为什么要同时确认`Spec`和`Annotations`中的值，其实他们表示的语义是不同的，`Spec`只能表示一个当前的期望状态，可能会动态变化，而`Annotations`是在创建`ReplicaSet`对象注入的`Deployment`最终期望。

执行的动作就是找出旧的`ReplicaSet`中副本数不为0的对象，然后调用`scaleReplicaSetAndRecordEvent()`方法把它们的期望值更新为0，和场景一的处理方式基本相同。

##### 场景三(核心场景)

此为多`ReplicaSet`共存的滚动更新中间场景，首先确定策略是否为滚动更新，然后获取所有当前副本数大于0的`ReplicaSet`对象，根据`Replicas`和`MaxSurge`计算本次进行调整的副本总数。需要注意的是，在该滚动更新操作中，扩/缩容的动作是**单向**的，不会有一个对象扩容的同时另一个对象缩容的情况。通过反复`扩容-缩容`的动作，再经过场景二的收尾，最终实际的副本数与期望值相同，并且由新`ReplicaSet`替换了旧的对象。

```Go
func (dc *DeploymentController) scale(ctx context.Context, deployment *apps.Deployment, newRS *apps.ReplicaSet, oldRSs []*apps.ReplicaSet) error {
    ......
    // 场景三：滚动更新进行中
    // 判断策略是否为RollingUpdate
    if deploymentutil.IsRollingUpdate(deployment) {
        // 获取所有存在副本的ReplicaSet以及对应的副本数量
        allRSs := controller.FilterActiveReplicaSets(append(oldRSs, newRS))
        allRSsReplicas := deploymentutil.GetReplicaCountForReplicaSets(allRSs)

        allowedSize := int32(0)
        // 计算允许最大存在副本数
        if *(deployment.Spec.Replicas) > 0 {
            allowedSize = *(deployment.Spec.Replicas) + deploymentutil.MaxSurge(*deployment)
        }
        // 根据当前总副本数判断下一步是扩容新ReplicaSet还是缩容旧ReplicaSet
        deploymentReplicasToAdd := allowedSize - allRSsReplicas

        switch {
        case deploymentReplicasToAdd > 0:
            // 扩容优先处理新的ReplicaSet对象
            sort.Sort(controller.ReplicaSetsBySizeNewer(allRSs))
        case deploymentReplicasToAdd < 0:
            // 缩容优先处理旧的ReplicaSet对象
            sort.Sort(controller.ReplicaSetsBySizeOlder(allRSs))
        }
        // 初始化变量 表示要调整的副本总数
        deploymentReplicasAdded := int32(0)
        nameToSize := make(map[string]int32)
        logger := klog.FromContext(ctx)
        // 遍历所有有副本数的ReplicaSet
        for i := range allRSs {
            rs := allRSs[i]
            if deploymentReplicasToAdd != 0 {
                // 计算调整的数量
                proportion := deploymentutil.GetReplicaSetProportion(logger, rs, *deployment, deploymentReplicasToAdd, deploymentReplicasAdded)
                // 记录每个RepicaSet的新期望副本数
                nameToSize[rs.Name] = *(rs.Spec.Replicas) + proportion
                // 累加调整的副本数 影响下一个对象变化量的上限
                // 如deploymentReplicasToAdd=5 当前ReplicaSet计算调整的数量是3 下一个ReplicaSet调整的上限为5-3=2
                deploymentReplicasAdded += proportion
            } else {
                nameToSize[rs.Name] = *(rs.Spec.Replicas)
            }
        }

        // 再次遍历ReplicaSet并执行扩缩容
        for i := range allRSs {
            rs := allRSs[i]
            // Deployment的最大调整数量和计算后的调整总数可能不一致
            // 对第一个ReplicaSet对象进行差额处理
            if i == 0 && deploymentReplicasToAdd != 0 {
                // 以deploymentReplicasToAdd的值为调整的数量标准
                leftover := deploymentReplicasToAdd - deploymentReplicasAdded
                nameToSize[rs.Name] = nameToSize[rs.Name] + leftover
                if nameToSize[rs.Name] < 0 {
                    nameToSize[rs.Name] = 0
                }
            }

            // 更新副本数
            if _, _, err := dc.scaleReplicaSet(ctx, rs, nameToSize[rs.Name], deployment); err != nil {
                // 如果更新失败Deployment对象会重新入队
                return err
            }
        }
    }
    return nil
}
```

##### 扩缩容调整ReplicaSet对象的比例

先根据`Deployment`对象检查副本数和注释信息是否有需要调整的，如果需要调整就深拷贝一份最新`ReplicaSet`对象，然后先向`ApiServer`发注释信息的更新请求，然后判断是否有扩/缩容的需要，记录并将标识位返回给上层。

```Go
func (dc *DeploymentController) scaleReplicaSet(ctx context.Context, rs *apps.ReplicaSet, newScale int32, deployment *apps.Deployment) (bool, *apps.ReplicaSet, error) {
    // 检查副本数是否需要调整
    sizeNeedsUpdate := *(rs.Spec.Replicas) != newScale
    // 检查注释信息是否需要调整
    annotationsNeedUpdate := deploymentutil.ReplicasAnnotationsNeedUpdate(rs, *(deployment.Spec.Replicas), *(deployment.Spec.Replicas)+deploymentutil.MaxSurge(*deployment))

    scaled := false
    var err error
    // 需要更新时
    if sizeNeedsUpdate || annotationsNeedUpdate {
        oldScale := *(rs.Spec.Replicas)
        // 深拷贝保证对象信息最新
        rsCopy := rs.DeepCopy()
        *(rsCopy.Spec.Replicas) = newScale
        // 设置ReplicaSet对象注释信息
        deploymentutil.SetReplicasAnnotations(rsCopy, *(deployment.Spec.Replicas), *(deployment.Spec.Replicas)+deploymentutil.MaxSurge(*deployment))
        // 发起更新请求
        rs, err = dc.client.AppsV1().ReplicaSets(rsCopy.Namespace).Update(ctx, rsCopy, metav1.UpdateOptions{})
        if err == nil && sizeNeedsUpdate {
            var scalingOperation string
            // 判断后续动作是扩容还是缩容
            if oldScale < newScale {
                scalingOperation = "up"
            } else {
                scalingOperation = "down"
            }
            scaled = true
            // 记录事件和缩放类型
            dc.eventRecorder.Eventf(deployment, v1.EventTypeNormal, "ScalingReplicaSet", "Scaled %s replica set %s from %d to %d", scalingOperation, rs.Name, oldScale, newScale)
        }
    }
    // 返回是否需要扩缩容
    return scaled, rs, err
}
```

##### 滚动更新数量计算规则

由`GetReplicaSetProportion()`函数返回给外层一个整数，这个数值的绝对值不超过允许值。

```Go
func GetReplicaSetProportion(logger klog.Logger, rs *apps.ReplicaSet, d apps.Deployment, deploymentReplicasToAdd, deploymentReplicasAdded int32) int32 {
    if rs == nil || *(rs.Spec.Replicas) == 0 || deploymentReplicasToAdd == 0 || deploymentReplicasToAdd == deploymentReplicasAdded {
        return int32(0)
    }
    // 计算调整比例
    rsFraction := getReplicaSetFraction(logger, *rs, d)
    allowed := deploymentReplicasToAdd - deploymentReplicasAdded
    // 限制调整的绝对值不超过allowed限制
    if deploymentReplicasToAdd > 0 {
        return min(rsFraction, allowed)
    }
    return max(rsFraction, allowed)
}
```

期望更新副本的差额计算逻辑在`getReplicaSetFraction()`函数中实现，如果`Deployment`要把副本数缩容到0，就直接返回当前`ReplicaSet`副本数作为差额。然后检查`ReplicaSet`注释中的最大容量，然后根据公式`期望容量=当前副本数*当前最大容量/上一轮最大容量`，返回本轮要扩/缩容的数量。

```Go
func getReplicaSetFraction(logger klog.Logger, rs apps.ReplicaSet, d apps.Deployment) int32 {
    // 如果想要缩容至0 直接返回当前的副本数
    if *(d.Spec.Replicas) == int32(0) {
        return -*(rs.Spec.Replicas)
    }
    // 获取Deployment的当前最大容量和上一轮为ReplicaSet对象注入的最大容量
    deploymentMaxReplicas := *(d.Spec.Replicas) + MaxSurge(d)
    deploymentMaxReplicasBeforeScale, ok := getMaxReplicasAnnotation(logger, &rs)
    // 如果ReplicaSet对象的最大容量注解缺失或值为0
    if !ok || deploymentMaxReplicasBeforeScale == 0 {
        // 用当前Deployment的副本数重新写入
        deploymentMaxReplicasBeforeScale = d.Status.Replicas
        // 异常情况 返回0给上层 避免后续计算出无效比例0
        if deploymentMaxReplicasBeforeScale == 0 {
            return 0
        }
    }
    // 获取ReplicaSet当前副本数
    scaleBase := *(rs.Spec.Replicas)
    // 计算规则为 当前副本数*当前最大容量/上一轮最大容量
    newRSsize := (float64(scaleBase * deploymentMaxReplicas)) / float64(deploymentMaxReplicasBeforeScale)
    // 返回期望调整的副本数量
    return integer.RoundToInt32(newRSsize) - *(rs.Spec.Replicas)
}
```

### Deployment对象需要回滚

根据`Deployment`对象`Annotation`中`"deprecated.deployment.rollback.to"`的值来显式指定回滚的版本，会在未来被逐渐弃用并使用`kubectl rollback`命令控制回滚，修改资源对象是不被推荐的行为，该回滚逻辑的代码在`rollback()`方法中实现。

首先获取`ReplicaSet`的信息，然后从注解信息中找出期望回滚的`Revision`版本号，如果是0尝试回滚到最近的一个版本。正常情况下遍历所有的`ReplicaSet`对象，并尝试根据`Revision`进行匹配，然后用`ReplicaSet`的Pod描述也就是`Template`字段更新当前`Deployment`中的内容，同时也更新注释信息，最后向`ApiServer`发送对`Deployment`对象的更新请求并请求回滚注解。

```Go
func (dc *DeploymentController) rollback(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet) error {
    logger := klog.FromContext(ctx)
    // 获取ReplicaSet对象
    newRS, allOldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, true)
    if err != nil {
        return err
    }

    allRSs := append(allOldRSs, newRS)
    // 获取目标版本号
    rollbackTo := getRollbackTo(d)
    // 特殊情况处理
    if rollbackTo.Revision == 0 {
    // Revision为0时尝试回滚到上一个版本
        if rollbackTo.Revision = deploymentutil.LastRevision(logger, allRSs); rollbackTo.Revision == 0 {
            dc.emitRollbackWarningEvent(d, deploymentutil.RollbackRevisionNotFound, "Unable to find last revision.")
            // 清除rollbackto注解
            return dc.updateDeploymentAndClearRollbackTo(ctx, d)
        }
    }
    for _, rs := range allRSs {
        v, err := deploymentutil.Revision(rs)
        if err != nil {
            logger.V(4).Info("Unable to extract revision from deployment's replica set", "replicaSet", klog.KObj(rs), "err", err)
            continue
        }
        // 匹配Revision
        if v == rollbackTo.Revision {
            logger.V(4).Info("Found replica set with desired revision", "replicaSet", klog.KObj(rs), "revision", v)
            // 更新Template和注解并向ApiServer发送更新请求
            performedRollback, err := dc.rollbackToTemplate(ctx, d, rs)
            if performedRollback && err == nil {
                dc.emitRollbackNormalEvent(d, fmt.Sprintf("Rolled back deployment %q to revision %d", d.Name, rollbackTo.Revision))
            }
            return err
        }
    }
    dc.emitRollbackWarningEvent(d, deploymentutil.RollbackRevisionNotFound, "Unable to find the revision to rollback to.")
    // 清理rollbackto注解
    return dc.updateDeploymentAndClearRollbackTo(ctx, d)
}
```

### Deployment对象滚动更新

#### Recreate策略

如果经过判断，滚动更新的策略为`Recreate`，其更新的处理方式为先终止旧的`Pod`，再启动新的`Pod`，在代码中由`rolloutRecreate()`方法为入口进入后续逻辑，一些核心的逻辑在之前的扩缩容部分已经有所涉及，下面分析该部分代码。

在一开始先获取新旧`ReplicaSet`对象，值得注意的是`getAllReplicaSetsAndSyncRevision()`方法传入一个`false`，因为`Recreate`逻辑严格要求先把旧的实例删掉才能创建新的，如果缩容操作前创建新`ReplicaSet`会导致新旧版本实例共存。然后获取旧版本中有`Pod`实例存在的`ReplicaSet`对象，并修改它们的`Spec.Replicas`为0，然后在直到没有旧版本的`Pod`运行前都对`Deployment`的状态进行同步，缩容操作完成后如果新的`ReplicaSet`不存在，再次调用`getAllReplicaSetsAndSyncRevision()`方法传入`true`，创建该对象。扩容新`ReplicaSet`，扩容完成后清理旧版本对象并同步`Deployment`状态。

```Go
func (dc *DeploymentController) rolloutRecreate(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet, podMap map[types.UID][]*v1.Pod) error {
    // 第四个入参表示如果ReplicaSet不存在是否创建
    // 在缩容阶段避免新旧Pod共存 此处不直接创建
    newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, false)
    if err != nil {
        return err
    }
    allRSs := append(oldRSs, newRS)
    // 获取活跃的旧ReplicaSet对象
    activeOldRSs := controller.FilterActiveReplicaSets(oldRSs)
    // 把所有活跃的旧ReplicaSet副本数设置为0
    scaledDown, err := dc.scaleDownOldReplicaSetsForRecreate(ctx, activeOldRSs, d)
    if err != nil {
        return err
    }
    // 缩容旧的ReplicaSet
    if scaledDown {
        // 同步Deployment状态
        return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
    }
    // 等待缩容结束
    if oldPodsRunning(newRS, oldRSs, podMap) {
        // 同步Deployment状态
        return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
    }
    // 如果新的ReplicaSet对象不存在 自动创建它
    if newRS == nil {
        newRS, oldRSs, err = dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, true)
        if err != nil {
            return err
        }
        // 记录对象
        allRSs = append(oldRSs, newRS)
    }
    // 扩容新的ReplicaSet
    if _, err := dc.scaleUpNewReplicaSetForRecreate(ctx, newRS, d); err != nil {
        return err
    }
    // 清理旧的ReplicaSet
    if util.DeploymentComplete(d, &d.Status) {
        if err := dc.cleanupDeployment(ctx, oldRSs, d); err != nil {
            return err
        }
    }
    // 同步Deployment状态
    return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
}
```

`oldPodsRunning()`函数用来检查是否有`Pod`实例在运行中，首先获取所有的`Pod`集合，然后检查其`Status.Phase`字段，该字段表示`Pod`的状态，包括`Pending/Running/Running/Failed/Unknown`，对应各生命周期状态。对于属于新版本`ReplicaSet`管理的跳过处理，该逻辑只确认旧版本的`Pod`是否有仍处于或可能处于运行状态的。

```Go
func oldPodsRunning(newRS *apps.ReplicaSet, oldRSs []*apps.ReplicaSet, podMap map[types.UID][]*v1.Pod) bool {
    if oldPods := util.GetActualReplicaCountForReplicaSets(oldRSs); oldPods > 0 {
        return true
    }
    // 遍历Pod
    for rsUID, podList := range podMap {
        // 跳过属于新ReplicaSet管理的Pod
        if newRS != nil && newRS.UID == rsUID {
            continue
        }
        for _, pod := range podList {
            switch pod.Status.Phase {
            case v1.PodFailed, v1.PodSucceeded:
                // 退出状态
                continue
            case v1.PodUnknown:
                // 异常状态
                return true
            default:
                // 其他运行状态
                return true
            }
        }
    }
    return false
}
```

`syncRolloutStatus()`方法用来同步缩容的状态，首先会根据当前观测到的`Generation`、`Replicas`、`UpdatedReplicas`、`ReadyReplicas`、`AvailableReplicas`等副本数量信息，判断当前`Deployment`对象的状态是否达成了最低的可用条件，然后更新`CondType`为`DeploymentAvailable`的状态并组装一个最新的`DeploymentStatus`类型的状态信息。然后尝试获取`DeploymentProgressing`的状态信息，并对`Deployment`状态进行判断，如果副本数和新版本副本数相等且状态信息为新`ReplicaSet`可用(`NewReplicaSetAvailable`)表示部署完成。如果结果表示未完成部署，则对结果进行确认并向`ApiServer`发送更新`Deployment`的请求。

```Go
func (dc *DeploymentController) syncRolloutStatus(ctx context.Context, allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, d *apps.Deployment) error {
    // 计算Deployment最新状态
    newStatus := calculateStatus(allRSs, newRS, d)
    // 如果没有配置截止时间 就删除CondType为DeploymentProgressing为状态信息
    if !util.HasProgressDeadline(d) {
        util.RemoveDeploymentCondition(&newStatus, apps.DeploymentProgressing)
    }
    // 获取CondType为DeploymentProgressing为状态信息
    currentCond := util.GetDeploymentCondition(d.Status, apps.DeploymentProgressing)
    // Deployment状态判断
    isCompleteDeployment := newStatus.Replicas == newStatus.UpdatedReplicas && currentCond != nil && currentCond.Reason == util.NewRSAvailableReason
    // 未部署完成 进行状态判断并更新
    if util.HasProgressDeadline(d) && !isCompleteDeployment {
        switch {
        // 已完成
        case util.DeploymentComplete(d, &newStatus):
            msg := fmt.Sprintf("Deployment %q has successfully progressed.", d.Name)
            if newRS != nil {
                msg = fmt.Sprintf("ReplicaSet %q has successfully progressed.", newRS.Name)
            }
            condition := util.NewDeploymentCondition(apps.DeploymentProgressing, v1.ConditionTrue, util.NewRSAvailableReason, msg)
            util.SetDeploymentCondition(&newStatus, *condition)
        // 处理中
        case util.DeploymentProgressing(d, &newStatus):
            msg := fmt.Sprintf("Deployment %q is progressing.", d.Name)
            if newRS != nil {
                msg = fmt.Sprintf("ReplicaSet %q is progressing.", newRS.Name)
            }
            condition := util.NewDeploymentCondition(apps.DeploymentProgressing, v1.ConditionTrue, util.ReplicaSetUpdatedReason, msg)
            if currentCond != nil {
                if currentCond.Status == v1.ConditionTrue {
                    condition.LastTransitionTime = currentCond.LastTransitionTime
                }
                util.RemoveDeploymentCondition(&newStatus, apps.DeploymentProgressing)
            }
            util.SetDeploymentCondition(&newStatus, *condition)
        // 已超时
        case util.DeploymentTimedOut(ctx, d, &newStatus):
            msg := fmt.Sprintf("Deployment %q has timed out progressing.", d.Name)
            if newRS != nil {
                msg = fmt.Sprintf("ReplicaSet %q has timed out progressing.", newRS.Name)
            }
            condition := util.NewDeploymentCondition(apps.DeploymentProgressing, v1.ConditionFalse, util.TimedOutReason, msg)
            util.SetDeploymentCondition(&newStatus, *condition)
        }
    }

    // 处理失败状态Condition
    if replicaFailureCond := dc.getReplicaFailures(allRSs, newRS); len(replicaFailureCond) > 0 {
        // 只会返回一条信息
        util.SetDeploymentCondition(&newStatus, replicaFailureCond[0])
    } else {
        // 没有失败信息就从Map中删除该key
        util.RemoveDeploymentCondition(&newStatus, apps.DeploymentReplicaFailure)
    }

    // 新旧状态是否一致
    if reflect.DeepEqual(d.Status, newStatus) {
        // 如果状态一致 把Deployment对象重新入队并返回
        dc.requeueStuckDeployment(ctx, d, newStatus)
        return nil
    }

    newDeployment := d
    newDeployment.Status = newStatus
    // 更新对象
    _, err := dc.client.AppsV1().Deployments(newDeployment.Namespace).UpdateStatus(ctx, newDeployment, metav1.UpdateOptions{})
    return err
}
```

#### RollingUpdate策略

如果经过判断，更新策略为`RollingUpdate`，则采用滚动更新方式，逻辑入口为`rolloutRolling()`方法。从外层的逻辑来看很清晰，首先获取对象的信息，然后有限尝试扩容新`ReplicaSet`对象，如果扩容则本次调谐返回并更新状态，如果无法进行扩容动作，则对旧`ReplicaSet`进行缩容操作，如果缩容也返回并更新`Deployment`状态，如果两者都没有就根据`Spec`和`Status`的一致性检查`Deployment`对象是否为部署成功的状态，如果是就清理旧`ReplicaSet`对象，最后更新状态。

```Go
func (dc *DeploymentController) rolloutRolling(ctx context.Context, d *apps.Deployment, rsList []*apps.ReplicaSet) error {
    newRS, oldRSs, err := dc.getAllReplicaSetsAndSyncRevision(ctx, d, rsList, true)
    if err != nil {
        return err
    }
    allRSs := append(oldRSs, newRS)
    // 尝试扩容新版本
    scaledUp, err := dc.reconcileNewReplicaSet(ctx, allRSs, newRS, d)
    if err != nil {
        return err
    }
    if scaledUp {
        // 结束本次调谐
        return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
    }

    // 尝试缩容旧版本
    scaledDown, err := dc.reconcileOldReplicaSets(ctx, allRSs, controller.FilterActiveReplicaSets(oldRSs), newRS, d)
    if err != nil {
        return err
    }
    if scaledDown {
        // 结束本次调谐
        return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
    }
    // 检查是否完成部署
    if deploymentutil.DeploymentComplete(d, &d.Status) {
        if err := dc.cleanupDeployment(ctx, oldRSs, d); err != nil {
            return err
        }
    }

    // 同步状态
    return dc.syncRolloutStatus(ctx, allRSs, newRS, d)
}
```

##### 尝试扩容新ReplicaSet

对这段逻辑进行简单的解释，首先对`ReplicaSet`和`Deployment`其中的`Spec.Replicas`字段做比较。如果新`ReplicaSet`已经和`Deployment`的期望副本数一致了则不做处理；如果是非预期的新`Replicas`期望副本数大于`Deployment`，则调整`ReplicaSet`的期望副本数为`Deployment`的期望副本数；其他情况就只剩下新`Replicas`期望副本数小于`Deployment`了，计算一下本次调整后的新`ReplicaSet`副本数并执行更新操作。

```Go
func (dc *DeploymentController) reconcileNewReplicaSet(ctx context.Context, allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, deployment *apps.Deployment) (bool, error) {
    if *(newRS.Spec.Replicas) == *(deployment.Spec.Replicas) {
        // Scaling not required.
        return false, nil
    }
    if *(newRS.Spec.Replicas) > *(deployment.Spec.Replicas) {
        // Scale down.
        scaled, _, err := dc.scaleReplicaSetAndRecordEvent(ctx, newRS, *(deployment.Spec.Replicas), deployment)
        return scaled, err
    }
    newReplicasCount, err := deploymentutil.NewRSNewReplicas(deployment, allRSs, newRS)
    if err != nil {
        return false, err
    }
    scaled, _, err := dc.scaleReplicaSetAndRecordEvent(ctx, newRS, newReplicasCount, deployment)
    return scaled, err
}
```

###### 扩容新副本数的计算

通过`NewRSNewReplicas()`函数计算出新`ReplicaSet`调整后的副本数量，规则也很简单。如果是`Recreate`策略直接返回`Deployment`的期望值；如果是`RollingUpdate`策略，根据`MaxSurge`计算`Deployment`的副本数量上限，然后根据**Deployment副本数上限与当前总副本数的差值**和**Deployment期望副本数与新ReplicaSet期望副本数的差值**，选择其中较小的加上当前新`ReplicaSet`的期望副本数，返回给上层作为调整后的期望副本数值。

```Go
func NewRSNewReplicas(deployment *apps.Deployment, allRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet) (int32, error) {
    switch deployment.Spec.Strategy.Type {
    case apps.RollingUpdateDeploymentStrategyType:
        // Check if we can scale up.
        maxSurge, err := intstrutil.GetScaledValueFromIntOrPercent(deployment.Spec.Strategy.RollingUpdate.MaxSurge, int(*(deployment.Spec.Replicas)), true)
        if err != nil {
            return 0, err
        }
        // Find the total number of pods
        currentPodCount := GetReplicaCountForReplicaSets(allRSs)
        maxTotalPods := *(deployment.Spec.Replicas) + int32(maxSurge)
        if currentPodCount >= maxTotalPods {
            // Cannot scale up.
            return *(newRS.Spec.Replicas), nil
        }
        // Scale up.
        scaleUpCount := maxTotalPods - currentPodCount
        // Do not exceed the number of desired replicas.
        scaleUpCount = min(scaleUpCount, *(deployment.Spec.Replicas)-*(newRS.Spec.Replicas))
        return *(newRS.Spec.Replicas) + scaleUpCount, nil
    case apps.RecreateDeploymentStrategyType:
        return *(deployment.Spec.Replicas), nil
    default:
        return 0, fmt.Errorf("deployment type %v isn't supported", deployment.Spec.Strategy.Type)
    }
}
```

##### 尝试缩容旧ReplicaSet

缩容旧`ReplicaSet`的过程中首先计算最大可缩容数量，其计算公式为**当前副本数-最小可用副本数-新ReplicaSet不可用副本数**，然后根据最大缩容数量去缩容处理旧版本的`ReplicaSet`，总共会经历两轮缩容，第一次先清理旧`ReplicaSet`中的不健康副本，返回一个数量`cleanupCount`，然后再正常进行缩容，返回一个数量`scaledDownCount`，如果两者的和大于0表示进行了缩容操作。

```Go
func (dc *DeploymentController) reconcileOldReplicaSets(ctx context.Context, allRSs []*apps.ReplicaSet, oldRSs []*apps.ReplicaSet, newRS *apps.ReplicaSet, deployment *apps.Deployment) (bool, error) {
    logger := klog.FromContext(ctx)
    oldPodsCount := deploymentutil.GetReplicaCountForReplicaSets(oldRSs)
    // 没有副本可以缩容
    if oldPodsCount == 0 {
        return false, nil
    }
    allPodsCount := deploymentutil.GetReplicaCountForReplicaSets(allRSs)
    logger.V(4).Info("New replica set", "replicaSet", klog.KObj(newRS), "availableReplicas", newRS.Status.AvailableReplicas)
    maxUnavailable := deploymentutil.MaxUnavailable(*deployment)
    minAvailable := *(deployment.Spec.Replicas) - maxUnavailable
    newRSUnavailablePodCount := *(newRS.Spec.Replicas) - newRS.Status.AvailableReplicas
    // 计算最大缩容数量
    maxScaledDown := allPodsCount - minAvailable - newRSUnavailablePodCount
    if maxScaledDown <= 0 {
        return false, nil
    }
    // 第一轮缩容 清理旧ReplicaSet的不健康副本 返回缩容的不健康副本数
    oldRSs, cleanupCount, err := dc.cleanupUnhealthyReplicas(ctx, oldRSs, deployment, maxScaledDown)
    if err != nil {
        return false, nil
    }
    logger.V(4).Info("Cleaned up unhealthy replicas from old RSes", "count", cleanupCount)

    allRSs = append(oldRSs, newRS)
    // 第二轮缩容 正常缩容旧ReplicaSet 返回正常缩容的副本数
    scaledDownCount, err := dc.scaleDownOldReplicaSetsForRollingUpdate(ctx, allRSs, oldRSs, deployment)
    if err != nil {
        return false, nil
    }
    logger.V(4).Info("Scaled down old RSes", "deployment", klog.KObj(deployment), "count", scaledDownCount)
    // 返回结果表示是否进行了缩容
    totalScaledDown := cleanupCount + scaledDownCount
    return totalScaledDown > 0, nil
}
```

###### 缩容不健康副本

首先对所有旧版本的副本按创建时间进行排序，优先处理创建更早的副本。遍历所有旧的`ReplicaSet`对象，总共缩容数量不能超过方法中传入的`maxCleanupCount`，每个`ReplicaSet`的缩容选择缩容余额和不健康副本数两者中较小的，更新`ReplicaSet`的副本数为`Spec.Replicas-scaledDownCount`，并更新本地缓存中的`ReplicaSet`对象，每次缩容的值进行累加最终返回给上层。

```Go
func (dc *DeploymentController) cleanupUnhealthyReplicas(ctx context.Context, oldRSs []*apps.ReplicaSet, deployment *apps.Deployment, maxCleanupCount int32) ([]*apps.ReplicaSet, int32, error) {
    logger := klog.FromContext(ctx)
    // 根据创建时间从早到晚排序
    sort.Sort(controller.ReplicaSetsByCreationTimestamp(oldRSs))
    // 初始化计数
    totalScaledDown := int32(0)
    // 遍历ReplicaSet
    for i, targetRS := range oldRSs {
        // 受maxCleanupCount限制清理的最大数量
        if totalScaledDown >= maxCleanupCount {
            break
        }
        if *(targetRS.Spec.Replicas) == 0 {
            // 跳过没有副本的ReplicaSet
            continue
        }
        logger.V(4).Info("Found available pods in old RS", "replicaSet", klog.KObj(targetRS), "availableReplicas", targetRS.Status.AvailableReplicas)
        if *(targetRS.Spec.Replicas) == targetRS.Status.AvailableReplicas {
            // no unhealthy replicas found, no scaling required.
            continue
        }
        // 计算缩容数量 取缩容余额和不可用副本数中较小的
        scaledDownCount := min(maxCleanupCount-totalScaledDown, *(targetRS.Spec.Replicas)-targetRS.Status.AvailableReplicas)
        newReplicasCount := *(targetRS.Spec.Replicas) - scaledDownCount
        if newReplicasCount > *(targetRS.Spec.Replicas) {
            return nil, 0, fmt.Errorf("when cleaning up unhealthy replicas, got invalid request to scale down %s/%s %d -> %d", targetRS.Namespace, targetRS.Name, *(targetRS.Spec.Replicas), newReplicasCount)
        }
        // 更新ReplicaSet副本数
        _, updatedOldRS, err := dc.scaleReplicaSetAndRecordEvent(ctx, targetRS, newReplicasCount, deployment)
        if err != nil {
            return nil, totalScaledDown, err
        }
        // 累加计数
        totalScaledDown += scaledDownCount
        // 更新旧ReplicaSet缓存
        oldRSs[i] = updatedOldRS
    }
    return oldRSs, totalScaledDown, nil
}
```

###### 正常缩容

正常缩容的逻辑和处理不健康副本类似，先进行排序，然后相当于是重新计算了最大可缩容副本数，遍历`ReplicaSet`并选择缩容余额和期望副本数中较小的作为缩容数量，更新`ReplicaSet`对象并计数缩容的副本。

```Go
func (dc *DeploymentController) scaleDownOldReplicaSetsForRollingUpdate(ctx context.Context, allRSs []*apps.ReplicaSet, oldRSs []*apps.ReplicaSet, deployment *apps.Deployment) (int32, error) {
    logger := klog.FromContext(ctx)

    maxUnavailable := deploymentutil.MaxUnavailable(*deployment)
    minAvailable := *(deployment.Spec.Replicas) - maxUnavailable
    availablePodCount := deploymentutil.GetAvailableReplicaCountForReplicaSets(allRSs)
    if availablePodCount <= minAvailable {
        return 0, nil
    }
    logger.V(4).Info("Found available pods in deployment, scaling down old RSes", "deployment", klog.KObj(deployment), "availableReplicas", availablePodCount)
    // 按创建时间排序
    sort.Sort(controller.ReplicaSetsByCreationTimestamp(oldRSs))
    // 初始化计数
    totalScaledDown := int32(0)
    // 计算最大可缩容数量
    totalScaleDownCount := availablePodCount - minAvailable
    // 遍历ReplicaSet
    for _, targetRS := range oldRSs {
        // 最大缩容数量约束
        if totalScaledDown >= totalScaleDownCount {
            break
        }
        if *(targetRS.Spec.Replicas) == 0 {
            // 跳过没有副本的ReplicaSet
            continue
        }
        // 计算缩容数量
        scaleDownCount := min(*(targetRS.Spec.Replicas), totalScaleDownCount-totalScaledDown)
        newReplicasCount := *(targetRS.Spec.Replicas) - scaleDownCount
        if newReplicasCount > *(targetRS.Spec.Replicas) {
            return 0, fmt.Errorf("when scaling down old RS, got invalid request to scale down %s/%s %d -> %d", targetRS.Namespace, targetRS.Name, *(targetRS.Spec.Replicas), newReplicasCount)
        }
        // 更新ReplicaSet副本数
        _, _, err := dc.scaleReplicaSetAndRecordEvent(ctx, targetRS, newReplicasCount, deployment)
        if err != nil {
            return totalScaledDown, err
        }
        // 累加计数
        totalScaledDown += scaleDownCount
    }

    return totalScaledDown, nil
}
```

