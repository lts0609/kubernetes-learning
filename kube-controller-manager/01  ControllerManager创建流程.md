# ControllerManager创建流程

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

`NewControllerDescriptors()`函数返回了一个key是控制器名称，value是`ControllerDescriptor`的映射，其中有一个需要注意的地方，就是`ServiceAccountTokenControllerDescriptor`，它是唯一特殊的控制器，需要最先启动而且使用具有**根权限**的客户端初始化，在之前的代码中以及创建了对象`saTokenControllerDescriptor`，那么为什么在下面这段函数中还要注册呢？主要有两个原因：`register()`校验了控制器描述符的合法性，以及保证和其他控制器元数据创建时的一致性。虽然后面会被`saTokenControllerDescriptor`替换，但是不影响和其他控制器描述符一起初始化一次。

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

