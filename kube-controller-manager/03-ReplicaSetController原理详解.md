# ReplicaSetController原理详解

## 实例创建

实例创建的部分属于通用基本逻辑，创建`Pod`和`ReplicaSet`对象的Informer，GVK标识为{Group: apps, Version: v1, Kind: ReplicaSet}。

```Go
func newReplicaSetControllerDescriptor() *ControllerDescriptor {
    return &ControllerDescriptor{
        name:     names.ReplicaSetController,
        aliases:  []string{"replicaset"},
        initFunc: startReplicaSetController,
    }
}

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
```

