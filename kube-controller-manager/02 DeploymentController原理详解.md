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

## 调谐和事件处理

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

### 调谐过程

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

#### 

