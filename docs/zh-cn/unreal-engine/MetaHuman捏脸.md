# 关于MetaHuman的表情逻辑驱动和骨骼捏脸

> 文章写下时的工作环境：UE5.0.3 

> 22年的时候因为角色捏脸相关的需求，替美术预研了MetaHuman的表情技术。最近重温细节时发现很多都忘记了，想来还是要用文字记录下来
> 
## 前言
首先美术给到了MetaHumanCreator生成的角色资产，希望我们在其基础上对脸部的骨骼调整，达到捏脸的效果，同时还要保证原生的RigLogic表情系统能正常运作。我这边刚拿到USkeletalMesh资源后，发现在UE编辑器下对面部骨骼的transform进行操作却始终无法生效，于是找到了Epic官方发布的RigLogic白皮书，对其整个表情系统进行了学习，下面简单记录一下整个过程中遇到的技术要点

## MetaHuman的表情逻辑驱动
先说结论，MetaHuman面部表情的逻辑驱动入口在UE引擎层面上是靠一个ControlRig节点。打开一个MetaHuman的USkeletalMesh后，可以看到它自带了后处理动画蓝图(PostProcess ABP)，我们如果直接在窗口下方禁用它或者去除对它的引用，就能看到可以自由拖动面部骨骼了

在AssetUserData栏目下，可以看到保存了一个UDNAAsset，如果在开启后处理动画蓝图的同时去除DNAAsset，会发现面部骨骼也能操作了，可以理解是后处理动画蓝图和DNAAsset两个一起配合着驱动RigLogi表情系统。所以如果要做骨骼捏脸方案，对面部骨骼进行修改，那么就应该遵循RigLogic原生的流程去处理。因此，我们就需要进一步探寻后处理动画蓝图的内容和DNAAsset之间的关联和工作流。


![Face](MetaHuman/face.png ':size=90%')

### 后处理动画蓝图
打开FaceProcessABP后，可以看到AnimGraph非常简单，只有一个ControlRig动画节点，如果不熟悉ControlRig工作原理的，可以看笔者的另一篇文章。打开动画节点引用的ControlRig资产，同样很简单只定义了一个ForwardsSolve事件，虽然连接出来的节点很多，但从整个执行流上看就两个部分：
1. 根据当前动画数据设置动画曲线数值
2. 执行RigLogic节点

而且RigHierarchy的骨骼树和MetaHuman的FReferenceSkeleton是一致的，所以在ControlRig里对任何骨骼的修改都会通过数据交换传递到外界。动画曲线只是浮点数集合，本身不包含任何逻辑计算，所以只需要将注意力集中到ControlRig图表中最后一个节点RigUnit_RigLogic

## RigUnit_RigLogic数据结构
在RigLogic节点里，有一个非常核心的数据结构FRigUnit_RigLogic_Data，它是整个RigLogic节点执行过程中的上下文，它包含了以下这两个关键的数据结构，了解它们是解读MetaHuman表情非常关键的一环。
- FSharedRigRuntimeContxt
- FRigLogic
- FRigInstance

![RigLogicData](MetaHuman/RigLogicData.png ':size=80%')

### SharedRigRuntimeContxt
首先是SharedRigRuntimeContxt，RigUnit节点在初始化阶段会通过USkeletalMesh获取一个指向FSharedRigRuntimeContext的指针。MetaHuman白皮书说过DNAAsset是所有数据来源，它是一组二进制数据以AssetUserData的形式保存在USkeletalMesh中，我们在引擎里是无法直接解析数据类型。但是在UDNAsset的序列化流程里，在Loading阶段通过它们自己写的第三方库提供了解析DNA数据流的接口IBehaviorReader和IGeometryReader，并以此完成了SharedRigRuntimeContxt的接口初始化。

```C++
struct FSharedRigRuntimeContext
{
    //...
    	uint32 DNAHash;
	TSharedPtr<IBehaviorReader> BehaviorReader;
	TSharedPtr<IGeometryReader> GeometryReader;
	TSharedPtr<FRigLogic> RigLogic;

};
```

### FRigLogic
UDNAAsset是MetaHuman的数据容器，它存放了所有骨骼，控制器，BlendShapes等一系列数据。而FRigLogic是它在UE引擎下的数据表示，它的工厂构造函数通过初始化后的IBehaviorReader，将DNA里的原型数据读取出来并单独COPY出来，所以不同USkeletalMesh所对应的MetaHuman的数据都是相互隔离的，而一个USkeletalMesh对应的多个动画实例则共享同一份内存地址。

![FRigLogic](MetaHuman/RigInstance.png ':size=90%')

```C++
namespace rl4 {

class RigLogicImpl : public RigLogic {
    private:
        using ControlsPtr = std::unique_ptr<Controls, std::function<void (Controls*)> >;
        using JointsPtr = std::unique_ptr<Joints, std::function<void (Joints*)> >;
        using BlendShapesPtr = std::unique_ptr<BlendShapes, std::function<void (BlendShapes*)> >;
        using AnimatedMapsPtr = std::unique_ptr<AnimatedMaps, std::function<void (AnimatedMaps*)> >;

        // ... getter ...
        void calculateControls(RigInstance* instance) const override;
        void calculateJoints(RigInstance* instance) const override;
        void calculateJoints(RigInstance* instance, std::uint16_t jointGroupIndex) const override;
        void calculateBlendShapes(RigInstance* instance) const override;
        void calculateAnimatedMaps(RigInstance* instance) const override;
        void calculate(RigInstance* instance) const override;

        MemoryResource* getMemoryResource();
    private:
        MemoryResource* memRes;
        ControlsPtr controls;
        JointsPtr joints;
        BlendShapesPtr blendShapes;
        AnimatedMapsPtr animatedMaps;
        Configuration config;
        RigMetrics metrics;
};
}  // namespace rl4

```
### FRigInstance
完成FRigLogic的初始化后，会通过FRigLogic完成FRigInstance的初始化，并将其handle保存在RigUnit的FRigUnit_RigLogic_Data的Unique指针里。所以RigInstance是每个Control节点，每个RigVM单独持有一份，计算出来的表情数据也会保存在RigInstance里，节点与节点之间相互不会影响。
```C++
void FRigUnit_RigLogic_Data::InitializeRigLogic(const URigHierarchy* InHierarchy)
{
    // ...
    if (!RigInstance.IsValid())
	{
		RigInstance = MakeUnique<FRigInstance>(SharedRigRuntimeContext->RigLogic.Get());
    }
}

FRigInstance::FRigInstance(FRigLogic* RigLogic) :
	MemoryResource{FMemoryResource::SharedInstance()},
	RigInstance{rl4::RigInstance::create(RigLogic->Unwrap(), MemoryResource.Get())}
{
}
```

## RigUnit_RigLogic执行流程
完成数据初始化后，在RigUnit_RigLogic在每帧的执行逻辑非常简单，一共就4步
1. Data.CalculateRigLogic(Hierarchy);
2. Data.UpdateJoints(Hierarchy, JointUpdateParamsTemp);
3. Data.UpdateBlendShapeCurves(Hierarchy, BlendShapeValues);
4. Data.UpdateAnimMapCurves(Hierarchy, AnimMapOutputs);

其中第3和第4步都是设置动画曲线数据，这里就先直接略过，关注第1步和第2步。

### CalculateRigLogic
从代码上看，这里在ControlRig资源里前半部分计算的RigHierarchy曲线数据，传递到RigInstance。我们注意到MetaHuman骨骼里有很多复杂名称的动画曲线，这些都是RigLogic算法库里预定义好的，曲线映射Mappings也是从DNA资源里获取的，每根曲线都对应某个特点表情特征，不要随意修改骨骼资产里的曲线定义。

![CalculateRigLogic](MetaHuman/CalculateRigLogic.png ':size=70%')

将每根曲线数值转换为表情计算的输入RawControlInput并存在每个节点独有的RigInstance里，然后将其作为参数传递到RigLogicImpl里

![calrig](MetaHuman/RigCalculate.png ':size=90%')

然后依次计算Controls，Joints，BlendShapes，AnimatedMaps。其中Controls计算出来的数据会影响，后续3个计算的结果。

### UpdateJoints
根据表情输入设定计算出来的数据，接下来就是应用表情数据。一般是表情在骨骼上，都是以叠加的形式混合在基础骨骼上。RigLogic这里也是相同的处理，首先构造出一个骨骼数据，然后在UpdateJoints里将上一步计算出来的RigInstance->GetJointOutputs()作为DeltaTransform叠加到NeutralJoint里。

```C++
FRigUnit_RigLogic_JointUpdateParams JointUpdateParamsTemp
(
    Data.SharedRigRuntimeContext->RigLogic->GetNeutralJointValues(),
    Data.RigInstance->GetJointOutputs()
);
```

![updatejoint](MetaHuman/updatejoint.png ':size=80%')

注意到NeutralJoint是从SharedRigRuntimeContext::RigLogic里获取的，这在同一个USkeletalMesh之间是共享的，它描述了一个MetaHuman的基础脸型数据。而DeltaTransform则是保存在RigInstance里，这在不同RigUnit里都是不同的，比如同一个Face根据不同动画曲线数值而呈现出不同的表情。

## 骨骼修改实现
通过上面的流程分析，可以很清楚了解如果要实现修改面部骨骼的捏脸方案，就需要骨骼捏脸数据应用到基础脸型数据，也就是FRigLogic里的NeutralJointValues。为了不破坏RigUnit的原生流程，我们采取非侵入式的修改，在执行PostProcessABP前将数据写进FRigLogic里。首先我们要获取UDNAAsset里的FSharedRigRuntimeContext，因为源码里是用friend struct的形式，所以接口没有暴露出来。因此，需要修改源码自己添加一个。

```C++
UCLASS(NotBlueprintable, hidecategories = (Object))
class RIGLOGICMODULE_API UDNAAsset : public UAssetUserData
{
	GENERATED_BODY()
public:
    // ........
    FSharedRigRuntimeContext* GetSharedRigRuntimeContext()
    {
        return &Context;
    }
};
```

### NeutralJointValues
有了RigRuntimeContext的指针，那么修改数据的入口就有了，接下来看一下NeutralJointValues是怎么定义的。因为RigLogic是一个第三方库，它的Transform数据排列方式和UE的FTransform是不一致的，所以插件里写了一个FTransformArrayView结构将底层数据封装了一层。

![transformview](MetaHuman/transform.png ':size=70%')

同理，我们如果想将美术提供的FTransform数据写入Values内存地址里，就仿照ConvertCoordinateSystem将数据反转一下。
```C++
class RIGLOGICMODULE_API FTransformArrayView
{
//....
public:
    void SetTransformFromIndex(const FTransform& InTransform, size_t Index)
    {
        float* Source = Values + (Index * TransformationSize);
        Source[0] = InTransform.GetLocation().X;
        Source[1] = -1.0f * InTransform.GetLocation().Y;
        Source[2] = InTransform.GetLocation().Z;
        Source[3] = InTransform.Rotator().Roll;
        Source[4] = -1.0f * InTransform.Rotator().Pitch;
        Source[5] = -1.0f * InTransform.Rotator().Yaw;
        Source[6] = InTransform.GetScale3D().X;
        Source[7] = InTransform.GetScale3D().Y;
        Source[8] = InTransform.GetScale3D().Z;
    }
};
```

## 总结
有了上面两个新增的接口，我们就可以将数据写入SharedRigRuntimeContext里，从而修改MetaHuman的基础脸型，同时修改后的脸型数据也会作为表情计算的输入计算得到新的DeltaTransform，从而保证表情系统依然能正常作用于修改后的脸型。大致的修改流程可以参考下面的伪代码

```C++
{
    // impl these code in Character, Component, AnimInstance or AnimNode
    UDNAAsset* DNAAsset = Cast<UDNAAsset>(SkelMesh->GetAssetUserDataOfClass(UDNAAsset::StaticClass()));
    FSharedRigRuntimeContext* ContextPtr = DNAAsset->GetSharedRigRuntimeContext();
    FName BoneName = ...
    FTransform NewTransform = ...
    for (const uint16 JointIndex : SharedRigRuntimeContext->VariableJointIndices[CurrentLOD].Values)
	{
		if(ContextPtr->BehaviorReader->GetJointName(JointIndex) == BoneName)
        {
            ContextPtr->RigLogic->GetNeutralJointValues().SetTransformFromIndex(NewTransform, JointIndex);
        }
	}
}
```

下面的gif里，左边和右边的是原生的metahuman mesh，中间的mesh通过上面自定义的接口是将payton的脸型数据写进ettore。然后通过Sequencer设置RigLogic表情曲线，可以看到在修改后的脸型上，表情依然是能正常播放的。

![gif](MetaHuman/metahuman.gif)