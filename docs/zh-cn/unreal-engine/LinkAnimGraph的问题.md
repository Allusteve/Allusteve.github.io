# SeamlessTravel中子动画蓝图的生命周期问题

之前项目在开发过程中，关卡流程那边调用了SeamlessTravel让玩家无缝传送，同时保留了玩家的Actor到下一个World里。但是传送后过去后，玩家角色动画一部分正常的逻辑却失效了，经过逐一排查后，发现Bug是因为使用了LinkAnimGraph所间接导致的，这里简单记录一下问题起因和解决方法。

## Link Anim Graph
在UE4和UE5的工作环境下，出于快速迭代和模块化的需求，角色的动画蓝图通常都会根据功能拆分为一个个子动画蓝图，在主动画蓝图的AnimGraph里用LinkAnimGraph节点的形式链接进来。SkeletalMeshComponent会在InitAnim流程中调用AnimGraph里所有动画节点FAnimNode的OnInitializeAnimInstance接口，而Link节点就会在这时候通过NewObject动态创建子动画蓝图实例，然后保存在Outer也就是SkeletalMeshComponent的UAnimInstance的数组里。
```C++
FAnimNode_LinkedAnimGraph::ReinitializeLinkedAnimInstance(const UAnimInstance* InOwningAnimInstance, UAnimInstance* InNewAnimInstance)
```
创建出来的子动画蓝图实例会跟着主动画蓝图实例一起更新计算
![linkanimgraph](LinkAnimGraph/initanim.png ':size=75%')

## SeamlessTravel的相关流程
UE中大部分GamePlay相关的Actor的生命周期都是依赖于World的，当发生关卡切换时，新的World被加载进来，而旧的World会Cleanup。除了部分关键Actor（PlayerController, PlayerState），大部分Actor都会跟着CurrentWorld一起销毁。所以引擎提供了一个ActorList让开发者保留一些Actor避免重复创建导致不必要的开销，比如玩家的Character。具体的执行流程可以看引擎源码，名称已经贴在下面。
```C++
FSeamlessTravelHandler::Tick()
```
在处理要保留的KeptActors，会调用Actor的Rename接口来修改Outer，里面会调用UnregisterAllComponents，然后再RegisterAllComponents
![rename_1](LinkAnimGraph/rename.png ':size=70%')
![rename_2](LinkAnimGraph/rename_2.png ':size=70%')

而在该流程中，SkeletalMeshComp会清空动态创建的子AnimInstance。然后在InitAnim中通过初始化动画节点创建新的LinkAnimstances。而作为主动画蓝图的AnimScriptInstance是不会受此过程影响的。
![reset](LinkAnimGraph/reset.png ':size=70%')

## 问题和解决方法
梳理完整个流程，问题的起因也就很清楚了。子动画蓝图实例在地图切换前后被清空然后再创建，而一些初始化的逻辑在切换前的关卡被调用了，切换后没有。最终通过排查，发现是子动画蓝图在BeginPlay里做了一些逻辑绑定，而动画蓝图作为一个UObject，其BeginPlay接口是伴随着外层Actor也就是玩家的Character。而SeamlessTravel中保留的KeptActor，是不会在新的关卡调用BeginPlay的！

而解决问题的方法很简单，将BeginPlay的逻辑迁移到InitializeAnimation即可。题外说，在刚开始UE开发时候，喜欢将一些初始化逻辑放在BeginPlay里，随着阅历和项目深度地提高，发现BeginPlay是一个“很不可靠”的方法，建议大家以后写逻辑时，当要在BeginPlay里写东西时，多思考一下对象是不是有更好的时机去做这些事情。