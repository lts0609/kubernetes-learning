# 初识Informer

## 调度器中的Informer

`Informer`是`Kubernetes`中最重要的机制之一，在前面我们了解过`kube-scheduler`的代码，现在可以简单回顾，在其中找到`Informer`的影子。

`cmd/kube-scheduler/app/server.go`文件中的`Run`函数完成了调度器实例的启动，看一下其中的`Informer`使用

```Go
startInformersAndWaitForSync := func(ctx context.Context) {
  // Start all informers.
  cc.InformerFactory.Start(ctx.Done())
  // DynInformerFactory can be nil in tests.
  if cc.DynInformerFactory != nil {
    cc.DynInformerFactory.Start(ctx.Done())
  }

  // Wait for all caches to sync before scheduling.
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

if !cc.ComponentConfig.DelayCacheUntilActive || cc.LeaderElection == nil {
  startInformersAndWaitForSync(ctx)
}
```

其中的`InformerFactory`往前找来自于`cmd/kube-scheduler/app/options/options.go`中的`Config`方法

```Go
c.InformerFactory = scheduler.NewInformerFactory(client, 0)
```

其中`scheduler.NewInformerFactory`的作用就是创建了一个`PodInformer`

```Go
func NewInformerFactory(cs clientset.Interface, resyncPeriod time.Duration) informers.SharedInformerFactory {
    informerFactory := informers.NewSharedInformerFactory(cs, resyncPeriod)
    informerFactory.InformerFor(&v1.Pod{}, newPodInformer)
    return informerFactory
}
```

在`pkg/scheduler/scheduler.go`的`New`函数中，创建了两个`Lister`。通过代码可以看出，创建`Lister`会使用到`Informer`的`Indexer`，从而间接创建`Informer`。

```Go
podLister := informerFactory.Core().V1().Pods().Lister()
nodeLister := informerFactory.Core().V1().Nodes().Lister()

func (f *nodeInformer) Informer() cache.SharedIndexInformer {
    return f.factory.InformerFor(&apicorev1.Node{}, f.defaultInformer)
}

func (f *nodeInformer) Lister() corev1.NodeLister {
    return corev1.NewNodeLister(f.Informer().GetIndexer())
}
```

以及注册了所有的事件处理方法

```Go
if err = addAllEventHandlers(sched, informerFactory, dynInformerFactory, resourceClaimCache, unionedGVKs(queueingHintsPerProfile)); err != nil {
  return nil, fmt.Errorf("adding event handlers: %w", err)
}

func addAllEventHandlers(
    sched *Scheduler,
    informerFactory informers.SharedInformerFactory,
    dynInformerFactory dynamicinformer.DynamicSharedInformerFactory,
    resourceClaimCache *assumecache.AssumeCache,
    gvkMap map[framework.EventResource]framework.ActionType,
) error {
    var (
        handlerRegistration cache.ResourceEventHandlerRegistration
        err                 error
        handlers            []cache.ResourceEventHandlerRegistration
    )
    // scheduled pod cache
    if handlerRegistration, err = informerFactory.Core().V1().Pods().Informer().AddEventHandler(
        cache.FilteringResourceEventHandler{
            FilterFunc: func(obj interface{}) bool {
                switch t := obj.(type) {
                case *v1.Pod:
                    return assignedPod(t)
                case cache.DeletedFinalStateUnknown:
                    if _, ok := t.Obj.(*v1.Pod); ok {
                        / The carried object may be stale, so we don't use it to check if
                        // it's assigned or not. Attempting to cleanup anyways.
                        return true
                    }
                    utilruntime.HandleError(fmt.Errorf("unable to convert object %T to *v1.Pod in %T", obj, sched))
                    return false
                default:
                    utilruntime.HandleError(fmt.Errorf("unable to handle object in %T: %T", sched, obj))
                    return false
                }
            },
            Handler: cache.ResourceEventHandlerFuncs{
                AddFunc:    sched.addPodToCache,
                UpdateFunc: sched.updatePodInCache,
                DeleteFunc: sched.deletePodFromCache,
            },
        },
    ); err != nil {
        return err
    }
    handlers = append(handlers, handlerRegistration)
    ......
}
```

下面重新按照正序梳理一遍：首先我们在生成`Config`对象时创建了一个`Informer`工厂，创建了`PodInformer`、`PodLister`和`NodeLister`，然后在`Run`时启动了工厂中的所有`Informer`并注册了相应的事件处理方法。