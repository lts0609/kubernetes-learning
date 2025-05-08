# DeploymentController原理详解

在上一章节中，我们简单了解了ControllerManager的创建，本篇文章中深入学习Kubernetes中最重要的控制器之一`DeploymentController`。

当一个`Deployment`创建的时候，其实总共创建了三种资源对象，分别是`Deployment`、`ReplicaSet`、`Pod`，这是非常重要的，因为`Deployment`资源并不会直接管理`Pod`，而是通过管理`ReplicaSet`对象来间接地管理`Pod`，所以一个无状态负载的`Pod`的直接归属是`ReplicaSet`。这种设计和滚动更新相关，当一个`Deployment`中定义的`Pod`模板发生变化时，会创建出一个新的`ReplicaSet`，再根据一定的规则去替换旧的`ReplicaSet`对象。

### 实例创建

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

### 启动逻辑



