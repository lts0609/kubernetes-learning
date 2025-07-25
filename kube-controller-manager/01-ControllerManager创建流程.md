# ControllerManager创建流程

## 控制器创建入口函数

根据在之前调度器学习过程中对`Cobra`框架构建组件的了解，首先就会想到`kube-controller- manager`的创建入口也在`cmd/kube-controller-manager/controller-manager.go`中，其中同样也只包含简单的三行代码。

```Go
func main() {
    command := app.NewControllerManagerCommand()
    code := cli.Run(command)
    os.Exit(code)
}
```

这和调度器中是完全相同的，下面进入`cmd/kube-controller-manager/app/controllermanager.go`路径下去看具体逻辑。还是关注`RunE()`中return的`Run()`函数。

```Go
// NewControllerManagerCommand creates a *cobra.Command object with default parameters
func NewControllerManagerCommand() *cobra.Command {
    // 初始化特性门控
    _, _ = featuregate.DefaultComponentGlobalsRegistry.ComponentGlobalsOrRegister(
        featuregate.DefaultKubeComponent, utilversion.DefaultBuildEffectiveVersion(), utilfeature.DefaultMutableFeatureGate)
    // 初始化配置信息KubeControllerManagerOptions对象
    s, err := options.NewKubeControllerManagerOptions()
    if err != nil {
        klog.Background().Error(err, "Unable to initialize command options")
        klog.FlushAndExit(klog.ExitFlushTimeout, 1)
    }

    cmd := &cobra.Command{
        ......
        // 核心逻辑
        RunE: func(cmd *cobra.Command, args []string) error {
            verflag.PrintAndExitIfRequested()

            // Activate logging as soon as possible, after that
            // show flags with the final logging configuration.
            if err := logsapi.ValidateAndApply(s.Logs, utilfeature.DefaultFeatureGate); err != nil {
                return err
            }
            cliflag.PrintFlags(cmd.Flags())
            // KubeControllerManagerOptions-->Config
            c, err := s.Config(KnownControllers(), ControllersDisabledByDefault(), ControllerAliases())
            if err != nil {
                return err
            }

            // add feature enablement metrics
            fg := s.ComponentGlobalsRegistry.FeatureGateFor(featuregate.DefaultKubeComponent)
            fg.(featuregate.MutableFeatureGate).AddMetrics()
            // 传入CompletedConfig创建控制器实例
            return Run(context.Background(), c.Complete())
        },
        Args: func(cmd *cobra.Command, args []string) error {
            for _, arg := range args {
                if len(arg) > 0 {
                    return fmt.Errorf("%q does not take any arguments, got %q", cmd.CommandPath(), args)
                }
            }
            return nil
        },
    }

    fs := cmd.Flags()
    namedFlagSets := s.Flags(KnownControllers(), ControllersDisabledByDefault(), ControllerAliases())
    verflag.AddFlags(namedFlagSets.FlagSet("global"))
    globalflag.AddGlobalFlags(namedFlagSets.FlagSet("global"), cmd.Name(), logs.SkipLoggingConfigurationFlags())
    for _, f := range namedFlagSets.FlagSets {
        fs.AddFlagSet(f)
    }

    cols, _, _ := term.TerminalSize(cmd.OutOrStdout())
    cliflag.SetUsageAndHelpFunc(cmd, namedFlagSets, cols)

    return cmd
}
```

配置信息不是我们需要关注的，在控制器中配置的创建实际和调度器中是基本一致的的，都是`Options -> Config -> CompletedConfig`，把完整配置传入核心逻辑`Run()`函数，

## 控制器核心创建逻辑

`Run()`函数的实现也和调度器十分相似，首先初始化日志记录器，打印基本环境信息，然后初始化事件广播器，注册配置和健康检查设置，启动Server并创建两个不同权限的客户端。这里涉及了一个重要的闭包函数`run()`。

```Go
func Run(ctx context.Context, c *config.CompletedConfig) error {
    // 初始化日志记录器
    logger := klog.FromContext(ctx)
    stopCh := ctx.Done()

    // 打印版本信息
    logger.Info("Starting", "version", utilversion.Get())
    // 打印Golang环境变量
    logger.Info("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))

    // 初始化事件广播器
    c.EventBroadcaster.StartStructuredLogging(0)
    c.EventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: c.Client.CoreV1().Events("")})
    defer c.EventBroadcaster.Shutdown()
    // 注册配置信息
    if cfgz, err := configz.New(ConfigzName); err == nil {
        cfgz.Set(c.ComponentConfig)
    } else {
        logger.Error(err, "Unable to register configz")
    }

    // 健康检查设置
    var checks []healthz.HealthChecker
    var electionChecker *leaderelection.HealthzAdaptor
    if c.ComponentConfig.Generic.LeaderElection.LeaderElect {
        electionChecker = leaderelection.NewLeaderHealthzAdaptor(time.Second * 20)
        checks = append(checks, electionChecker)
    }
    healthzHandler := controllerhealthz.NewMutableHealthzHandler(checks...)

    // 启动http服务器
    var unsecuredMux *mux.PathRecorderMux
    if c.SecureServing != nil {
        unsecuredMux = genericcontrollermanager.NewBaseHandler(&c.ComponentConfig.Generic.Debugging, healthzHandler)
        slis.SLIMetricsWithReset{}.Install(unsecuredMux)

        handler := genericcontrollermanager.BuildHandlerChain(unsecuredMux, &c.Authorization, &c.Authentication)
        if _, _, err := c.SecureServing.Serve(handler, 0, stopCh); err != nil {
            return err
        }
    }
    // 创建root权限客户端和普通客户端
    clientBuilder, rootClientBuilder := createClientBuilders(c)

    saTokenControllerDescriptor := newServiceAccountTokenControllerDescriptor(rootClientBuilder)
    // 闭包函数
    run := func(ctx context.Context, controllerDescriptors map[string]*ControllerDescriptor) {
        controllerContext, err := CreateControllerContext(ctx, c, rootClientBuilder, clientBuilder)
        if err != nil {
            logger.Error(err, "Error building controller context")
            klog.FlushAndExit(klog.ExitFlushTimeout, 1)
        }

        if err := StartControllers(ctx, controllerContext, controllerDescriptors, unsecuredMux, healthzHandler); err != nil {
            logger.Error(err, "Error starting controllers")
            klog.FlushAndExit(klog.ExitFlushTimeout, 1)
        }

        controllerContext.InformerFactory.Start(stopCh)
        controllerContext.ObjectOrMetadataInformerFactory.Start(stopCh)
        close(controllerContext.InformersStarted)

        <-ctx.Done()
    }

    // 如果没开启选举 直接运行Controller并返回
    if !c.ComponentConfig.Generic.LeaderElection.LeaderElect {
        // 初始化控制器描述符集合
        controllerDescriptors := NewControllerDescriptors()
        controllerDescriptors[names.ServiceAccountTokenController] = saTokenControllerDescriptor
        // 启动控制器
        run(ctx, controllerDescriptors)
        return nil
    }
    // 开启选举的情况
    ......
}
```

在对`run()`闭包函数以及内部逻辑做解释之前，先了解一个数据结构`ControllerDescriptor`，也就是该函数的入参类型，它用于描述和管理控制器的信息。

```Go
type InitFunc func(ctx context.Context, controllerContext ControllerContext, controllerName string) (controller controller.Interface, enabled bool, err error)

type ControllerDescriptor struct {
    // 控制器名称
    name                      string
    // 初始化函数
    initFunc                  InitFunc
    // 特性门控列表
    requiredFeatureGates      []featuregate.Feature
    // 别名
    aliases                   []string
    // 是否默认禁用
    isDisabledByDefault       bool
    // 是否和云供应商有关
    isCloudProviderController bool
    // 是否有特殊处理逻辑
    requiresSpecialHandling   bool
}
```

以不开启选举的流程为例，不涉及选主逻辑会直接启动控制器，首先会初始化`ControllerDescriptor`集合，然后传递给`run()`。

```Go
    if !c.ComponentConfig.Generic.LeaderElection.LeaderElect {
        controllerDescriptors := NewControllerDescriptors()
        controllerDescriptors[names.ServiceAccountTokenController] = saTokenControllerDescriptor
        run(ctx, controllerDescriptors)
        return nil
    }
```

`NewControllerDescriptors()`函数返回了一个key是控制器名称，value是`ControllerDescriptor`的映射，通过`Descriptor`的包装实现了控制器逻辑与配置的分离。其中有一个需要注意的地方，`ServiceAccountTokenControllerDescriptor`是唯一特殊的控制器，需要最先启动而且使用具有**根权限**的客户端初始化，在之前的代码中已经创建了对象`saTokenControllerDescriptor`，那么为什么在下面这段函数中还要注册呢？主要的原因是`NewControllerDescriptors()`函数没有入参而`ServiceAccountTokenControllerDescriptor`的初始化函数需要传入根权限的客户端，但是要保证和其他控制器元数据创建时的一致性，并且其中`register()`校验了控制器描述符的合法性，虽然后面会被单独创建的`saTokenControllerDescriptor`替换，但是不影响和其他控制器描述符一起初始化一次。

```Go
func NewControllerDescriptors() map[string]*ControllerDescriptor {
    // 初始化
    controllers := map[string]*ControllerDescriptor{}
    // 使用Set避免重复元素
    aliases := sets.NewString()

    // 合法性校验与集合元素添加
    register := func(controllerDesc *ControllerDescriptor) {
        if controllerDesc == nil {
            panic("received nil controller for a registration")
        }
        name := controllerDesc.Name()
        if len(name) == 0 {
            panic("received controller without a name for a registration")
        }
        if _, found := controllers[name]; found {
            panic(fmt.Sprintf("controller name %q was registered twice", name))
        }
        if controllerDesc.GetInitFunc() == nil {
            panic(fmt.Sprintf("controller %q does not have an init function", name))
        }

        for _, alias := range controllerDesc.GetAliases() {
            if aliases.Has(alias) {
                panic(fmt.Sprintf("controller %q has a duplicate alias %q", name, alias))
            }
            aliases.Insert(alias)
        }

        controllers[name] = controllerDesc
    }

    // 注册所有的ControllerDescriptor
    register(newServiceAccountTokenControllerDescriptor(nil))

    register(newEndpointsControllerDescriptor())
    register(newEndpointSliceControllerDescriptor())
    register(newEndpointSliceMirroringControllerDescriptor())
    register(newReplicationControllerDescriptor())
    register(newPodGarbageCollectorControllerDescriptor())
    register(newResourceQuotaControllerDescriptor())
    register(newNamespaceControllerDescriptor())
    register(newServiceAccountControllerDescriptor())
    register(newGarbageCollectorControllerDescriptor())
    register(newDaemonSetControllerDescriptor())
    register(newJobControllerDescriptor())
    register(newDeploymentControllerDescriptor())
    register(newReplicaSetControllerDescriptor())
    register(newHorizontalPodAutoscalerControllerDescriptor())
    register(newDisruptionControllerDescriptor())
    register(newStatefulSetControllerDescriptor())
    register(newCronJobControllerDescriptor())
    register(newCertificateSigningRequestSigningControllerDescriptor())
    register(newCertificateSigningRequestApprovingControllerDescriptor())
    register(newCertificateSigningRequestCleanerControllerDescriptor())
    register(newTTLControllerDescriptor())
    register(newBootstrapSignerControllerDescriptor())
    register(newTokenCleanerControllerDescriptor())
    register(newNodeIpamControllerDescriptor())
    register(newNodeLifecycleControllerDescriptor())

    register(newServiceLBControllerDescriptor())          // cloud provider controller
    register(newNodeRouteControllerDescriptor())          // cloud provider controller
    register(newCloudNodeLifecycleControllerDescriptor()) // cloud provider controller

    register(newPersistentVolumeBinderControllerDescriptor())
    register(newPersistentVolumeAttachDetachControllerDescriptor())
    register(newPersistentVolumeExpanderControllerDescriptor())
    register(newClusterRoleAggregrationControllerDescriptor())
    register(newPersistentVolumeClaimProtectionControllerDescriptor())
    register(newPersistentVolumeProtectionControllerDescriptor())
    register(newVolumeAttributesClassProtectionControllerDescriptor())
    register(newTTLAfterFinishedControllerDescriptor())
    register(newRootCACertificatePublisherControllerDescriptor())
    register(newKubeAPIServerSignerClusterTrustBundledPublisherDescriptor())
    register(newEphemeralVolumeControllerDescriptor())

    // feature gated
    register(newStorageVersionGarbageCollectorControllerDescriptor())
    register(newResourceClaimControllerDescriptor())
    register(newLegacyServiceAccountTokenCleanerControllerDescriptor())
    register(newValidatingAdmissionPolicyStatusControllerDescriptor())
    register(newTaintEvictionControllerDescriptor())
    register(newServiceCIDRsControllerDescriptor())
    register(newStorageVersionMigratorControllerDescriptor())
    register(newSELinuxWarningControllerDescriptor())

    for _, alias := range aliases.UnsortedList() {
        if _, ok := controllers[alias]; ok {
            panic(fmt.Sprintf("alias %q conflicts with a controller name", alias))
        }
    }
    // 返回ControllerDescriptor集合
    return controllers
}
```

所有的`ControllerDescriptor`元数据都初始化后，替换`ServiceAccountTokenControllerDescriptor`为此前创建的内容，然后调用核心入口逻辑闭包函数`run()`，下面来看它的实现逻辑。

```Go
    run := func(ctx context.Context, controllerDescriptors map[string]*ControllerDescriptor) {
        // 创建控制器上下文
        controllerContext, err := CreateControllerContext(ctx, c, rootClientBuilder, clientBuilder)
        if err != nil {
            logger.Error(err, "Error building controller context")
            klog.FlushAndExit(klog.ExitFlushTimeout, 1)
        }
        // 启动控制器
        if err := StartControllers(ctx, controllerContext, controllerDescriptors, unsecuredMux, healthzHandler); err != nil {
            logger.Error(err, "Error starting controllers")
            klog.FlushAndExit(klog.ExitFlushTimeout, 1)
        }
        // 启动SharedInformer工厂 监控Kubernetes标准资源
        controllerContext.InformerFactory.Start(stopCh)
        // 启动MetadataInformerFactory工厂 监控类型化资源如CRD
        controllerContext.ObjectOrMetadataInformerFactory.Start(stopCh)
        close(controllerContext.InformersStarted)

        <-ctx.Done()
    }
```

关于`CreateControllerContext()`函数，它的作用是

```Go
func CreateControllerContext(ctx context.Context, s *config.CompletedConfig, rootClientBuilder, clientBuilder clientbuilder.ControllerClientBuilder) (ControllerContext, error) {
    // 闭包函数 用于裁剪obj对象的ManagedFields字段来提高内存效率
    trim := func(obj interface{}) (interface{}, error) {
        // 获取obj对象元数据
        if accessor, err := meta.Accessor(obj); err == nil {
            if accessor.GetManagedFields() != nil {
                // 裁剪ManagedFields字段
                accessor.SetManagedFields(nil)
            }
        }
        return obj, nil
    }
    // 创建SharedInformer工厂
    versionedClient := rootClientBuilder.ClientOrDie("shared-informers")
    sharedInformers := informers.NewSharedInformerFactoryWithOptions(versionedClient, ResyncPeriod(s)(), informers.WithTransform(trim))
    // 创建MetadataInformers工厂
    metadataClient := metadata.NewForConfigOrDie(rootClientBuilder.ConfigOrDie("metadata-informers"))
    metadataInformers := metadatainformer.NewSharedInformerFactoryWithOptions(metadataClient, ResyncPeriod(s)(), metadatainformer.WithTransform(trim))

    // 等待ApiServer启动 超时时间设置为10s
    if err := genericcontrollermanager.WaitForAPIServer(versionedClient, 10*time.Second); err != nil {
        return ControllerContext{}, fmt.Errorf("failed to wait for apiserver being healthy: %v", err)
    }

    // 创建Discovery客户端
    discoveryClient := rootClientBuilder.DiscoveryClientOrDie("controller-discovery")
    // 把Discovery客户端包装成一个缓存客户端
    cachedClient := cacheddiscovery.NewMemCacheClient(discoveryClient)
    // 再把缓存客户端包装成一个REST映射器
    restMapper := restmapper.NewDeferredDiscoveryRESTMapper(cachedClient)
    // 启动一个协程定时每30s刷新REST映射器缓存 确保获取最新的API信息
    go wait.Until(func() {
        restMapper.Reset()
    }, 30*time.Second, ctx.Done())
    // 组装ControllerContext对象
    controllerContext := ControllerContext{
        ClientBuilder:                   clientBuilder,
        InformerFactory:                 sharedInformers,
        ObjectOrMetadataInformerFactory: informerfactory.NewInformerFactory(sharedInformers, metadataInformers),
        ComponentConfig:                 s.ComponentConfig,
        RESTMapper:                      restMapper,
        InformersStarted:                make(chan struct{}),
        ResyncPeriod:                    ResyncPeriod(s),
        ControllerManagerMetrics:        controllersmetrics.NewControllerManagerMetrics("kube-controller-manager"),
    }
    // 如果开启了GarbageCollectorController垃圾回收控制器
    if controllerContext.ComponentConfig.GarbageCollectorController.EnableGarbageCollector &&
        controllerContext.IsControllerEnabled(NewControllerDescriptors()[names.GarbageCollectorController]) {
        ignoredResources := make(map[schema.GroupResource]struct{})
        for _, r := range controllerContext.ComponentConfig.GarbageCollectorController.GCIgnoredResources {
            // 获取忽略资源列表
            ignoredResources[schema.GroupResource{Group: r.Group, Resource: r.Resource}] = struct{}{}
        }
        // 创建GraphBuilder用来构建资源关系依赖
        controllerContext.GraphBuilder = garbagecollector.NewDependencyGraphBuilder(
            ctx,
            metadataClient,
            controllerContext.RESTMapper,
            ignoredResources,
            controllerContext.ObjectOrMetadataInformerFactory,
            controllerContext.InformersStarted,
        )
    }
    // 注册指标计数器
    controllersmetrics.Register()
    return controllerContext, nil
}
```

由于`ControllerManager`的运行所谓等待`ApiServer`成功启动，就是等待它的`/healthz`端点返回`OK`。

```Go
func WaitForAPIServer(client clientset.Interface, timeout time.Duration) error {
    var lastErr error
    // 轮询器执行目标函数
    err := wait.PollImmediate(time.Second, timeout, func() (bool, error) {
        healthStatus := 0
        result := client.Discovery().RESTClient().Get().AbsPath("/healthz").Do(context.TODO()).StatusCode(&healthStatus)
        if result.Error() != nil {
            lastErr = fmt.Errorf("failed to get apiserver /healthz status: %v", result.Error())
            return false, nil
        }
        if healthStatus != http.StatusOK {
            content, _ := result.Raw()
            lastErr = fmt.Errorf("APIServer isn't healthy: %v", string(content))
            klog.Warningf("APIServer isn't healthy yet: %v. Waiting a little while.", string(content))
            return false, nil
        }
        // 返回200OK结束轮询
        return true, nil
    })

    if err != nil {
        return fmt.Errorf("%v: %v", err, lastErr)
    }

    return nil
}
```

在`CreateControllerContext()`函数返回了控制器的公共配置后，就进入到下一个重要的步骤，也就是逐个启动控制器。

首先会启动`ServiceAccountToken`控制器，因为与`ApiServer`交互时会用到令牌验证身份，如果该控制器没有启动会影响到其他控制器的正常运行。然后再遍历`ControllerDescriptor`集合启动其他控制器。

```Go
func StartControllers(ctx context.Context, controllerCtx ControllerContext, controllerDescriptors map[string]*ControllerDescriptor,
    unsecuredMux *mux.PathRecorderMux, healthzHandler *controllerhealthz.MutableHealthzHandler) error {
    var controllerChecks []healthz.HealthChecker

    // ServiceAccountTokenController需要第一个被启动
    if serviceAccountTokenControllerDescriptor, ok := controllerDescriptors[names.ServiceAccountTokenController]; ok {
        check, err := StartController(ctx, controllerCtx, serviceAccountTokenControllerDescriptor, unsecuredMux)
        if err != nil {
            return err
        }
        if check != nil {
            // HealthChecker should be present when controller has started
            controllerChecks = append(controllerChecks, check)
        }
    }

    // 遍历启动其他控制器
    for _, controllerDesc := range controllerDescriptors {
        if controllerDesc.RequiresSpecialHandling() {
            continue
        }

        check, err := StartController(ctx, controllerCtx, controllerDesc, unsecuredMux)
        if err != nil {
            return err
        }
        if check != nil {
            // HealthChecker should be present when controller has started
            controllerChecks = append(controllerChecks, check)
        }
    }

    healthzHandler.AddHealthChecker(controllerChecks...)

    return nil
}
```

## ServiceAccountToken控制器的创建

看一下`ServiceAccountTokenController`是如何启动的，`StartController()`是该控制器启动的直接步骤。经过一系列的检查后，调用此前在`ControllerDescriptor`对象中注册的`InitFunc`初始化函数创建控制器实例，并注册调试接口和创建健康检查器。

```Go
func StartController(ctx context.Context, controllerCtx ControllerContext, controllerDescriptor *ControllerDescriptor,
    unsecuredMux *mux.PathRecorderMux) (healthz.HealthChecker, error) {
    // 初始化日志记录器
    logger := klog.FromContext(ctx)
    controllerName := controllerDescriptor.Name()
    // 校验需要的特性门控是否全部开启
    for _, featureGate := range controllerDescriptor.GetRequiredFeatureGates() {
        if !utilfeature.DefaultFeatureGate.Enabled(featureGate) {
            logger.Info("Controller is disabled by a feature gate", "controller", controllerName, "requiredFeatureGates", controllerDescriptor.GetRequiredFeatureGates())
            return nil, nil
        }
    }
    // 如果是云厂商控制器则跳过
    if controllerDescriptor.IsCloudProviderController() {
        logger.Info("Skipping a cloud provider controller", "controller", controllerName)
        return nil, nil
    }
    // 校验当前控制器是否被启用
    if !controllerCtx.IsControllerEnabled(controllerDescriptor) {
        logger.Info("Warning: controller is disabled", "controller", controllerName)
        return nil, nil
    }
    // 随机延迟启动控制器 避免资源竞争
    time.Sleep(wait.Jitter(controllerCtx.ComponentConfig.Generic.ControllerStartInterval.Duration, ControllerStartJitter))

    logger.V(1).Info("Starting controller", "controller", controllerName)
    // 执行ControllerDescriptor中的InitFunc初始化控制器实例
    initFunc := controllerDescriptor.GetInitFunc()
    ctrl, started, err := initFunc(klog.NewContext(ctx, klog.LoggerWithName(logger, controllerName)), controllerCtx, controllerName)
    if err != nil {
        logger.Error(err, "Error starting controller", "controller", controllerName)
        return nil, err
    }
    if !started {
        logger.Info("Warning: skipping controller", "controller", controllerName)
        return nil, nil
    }

    check := controllerhealthz.NamedPingChecker(controllerName)
    if ctrl != nil {
        // 注册调试接口
        if debuggable, ok := ctrl.(controller.Debuggable); ok && unsecuredMux != nil {
            if debugHandler := debuggable.DebuggingHandler(); debugHandler != nil {
                basePath := "/debug/controllers/" + controllerName
                unsecuredMux.UnlistedHandle(basePath, http.StripPrefix(basePath, debugHandler))
                unsecuredMux.UnlistedHandlePrefix(basePath+"/", http.StripPrefix(basePath, debugHandler))
            }
        }
        // 创建健康检查器
        if healthCheckable, ok := ctrl.(controller.HealthCheckable); ok {
            if realCheck := healthCheckable.HealthChecker(); realCheck != nil {
                check = controllerhealthz.NamedHealthChecker(controllerName, realCheck)
            }
        }
    }

    logger.Info("Started controller", "controller", controllerName)
    return check, nil
}
```

在这里先不关注各种控制器初始化函数的具体实现，后续会在分析每种具体控制器时一并说明，其他控制器也都通过`StartController`创建出来后，继续回到闭包函数`run()`中，剩下的最后一个步骤就是通过工厂启动`ControllerContext`中的两类`Informer`，所有`Informer`实例都启动后关闭控制器上下文中的`ControllerContext.InformersStarted`通道，最后通过常见的方式`<-ctx.Done()`挂起主线程，直至收到停止信号后优雅退出。