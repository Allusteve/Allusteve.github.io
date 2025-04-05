# UE中的叠加动画

## 叠加混合
动画Pipeline的核心三要素是采样，混合，后处理。混合最常见的是线性混合，而叠加混合（也有称作加法混合）则是为混合带来一种新的方式。通过引入Difference Clip
形式的动画，即一个Pose到另一个Pose所需的改变，再将其加进普通动画序列，产生一些有趣的姿势和动作变化。

我们令 S= source clip, R = reference clip，那么叠加动画是D = S-R。我们再令要混合的BasePose为T，那么混合出来的结果Outpout = T + D。
个人认为，叠加动画的美妙之处在于,它得到的结果O同时保留了T和S中动画师想要表达的效果，而且T的选取可以和R没有关联。

## UE中设置叠加动画
叠加动画本身的原理很简单，无非就是一组TargetPose减去BasePose得到DeltaPose。在UE里面，通过设置UAnimSequence的AdditiveSettings来开启或者关闭。
而且UAnimSequence开启了叠加设置，那么通过下面方式得到的数据都是经过处理的Delta差值。
```C++
UAnimSequence::GetAnimationPose(OutPose, OutCurve, ExtractionContext)
```
![settings](AdditiveAnim/settings.png ':size=80%')

这是因为在动画编辑器里引擎做了后处理操作，在PostEdit流程中，如果AdditiveSettings发生了变更，会更新动画数据并重新压缩。也就是动画资源本身的叠加与否是离线
操作完成的，并不是运行时计算得到的。当然UE也提供了一个动画节点MakeDynamicAdditive来满足Runtime计算叠加动画的需求。

![PostEdit](AdditiveAnim/PostEdit.png ':size=80%')

## 使用叠加动画
在叠加动画的使用上，相比较一般的绝对动画，没有太大区别，常见的都是在LocalSpace也就是骨骼空间进行计算的。但是如果多个叠加动画混合后再叠加，最终的效果可能会与预期产生偏差，经典的例子就是制作瞄准偏移AimOffset。比如往一个BasePose叠加一个向上瞄准的AO_LookUp让上半身朝向上方。但是如果先叠加一个倾斜动画的BasePose发生倾斜，然后在骨骼空间叠加一个向上的动画，就会得到一个斜向上的结果。如果希望得到倾斜后向上的结果，就需要在MeshSpace（模型空间）中处理Rotation。
![Aimoffset](AdditiveAnim/Aimoffset.png ':size=60%')

> 可以看到在ApplyMeshSpace的时候会先将Pose转换到MeshSpace，完成叠加计算，再转换回来

![Apply](AdditiveAnim/ApplyAdditive.png ':size=80%')
![MeshSpace](AdditiveAnim/MeshSpace.png ':size=80%')

## 叠加动画和RootMotion
前面说过设置叠加动画后，引擎会重新计算动画数据并压缩。而UE在计算RootMotion的流程中，提取根骨骼的动画数据是通过GetBoneTransform得到的。也就是说如果要对叠加动画开启RootMotion，那么引擎会使用处理过后的叠加动画数据进行计算，而不是原生的RawData。对于RootBone，得到的Transform数据就是TargetPose - BasePose的差值。
```C++
FTransform UAnimSequence::ExtractRootTrackTransform(float Pos, const FBoneContainer * RequiredBones) const
{
    // ...
	if(TrackIndex != INDEX_NONE)
	{
		// if we do have root data, then return root data
		FTransform RootTransform;
		GetBoneTransform(RootTransform, TrackIndex, Pos, bUseRawDataOnly);
		return RootTransform;
	}
}
```
如果角色使用的是类似UE官方的小白人Mannequin骨架就没有任何问题，它的根骨骼Transform在ReferencePose下，位移和旋转都归零了。但有时美术会给到一些比较“奇葩”的骨骼，会带有初始的旋转数据，那么就需要注意一下。这里用商城找到的一个模型资源作为示例，演示开发过程中遇到的**两个深坑**。

![Skeleton](AdditiveAnim/LichSkeleton.png ':size=70%')

### RootMotionRootLock问题
如果使用类似上述的骨骼，在使用RootMotion的时候，首先需要将RootMotionRootLock设置为**AnimFirstFrame**。使用叠加混合时我们得到的结果是Result = AdditivePose(root) + BasePose(root)，而AdditivePose(root)经过处理已经减去BasePose的根骨骼数据，所以直接用是正常的。但是开启RootMotion后会强制锁定根骨骼，默认选项是RefPose，因为RefPose有初始旋转，导致得到的结果就是RefPose的根骨骼Transform的double。下图角色翻转90度就是因为RefPose里面的初始数据。

![Double](AdditiveAnim/DoubleRef.png ':size=80%')

### ExtractRootBone坐标系问题
在设置好正确的RootLock后，就进入RootMoiton数据提取流程了。在这个过程中会遇到一个问题，AdditiveAnim下提取出来的Transform和常规非AdditiveAnim设置下提取出来的Transform不一致的问题。下面第一张gif是NoAdditive下播放带位移蒙太奇的情况，角色攻击向前位移一段距离，通过log可以看到提取的RootMotion数据，其位移方向在ComponentSpace下是水平面沿着Y轴的。第二张gif则是播放叠加蒙太奇的情况，角色几乎原地不动，通过log可以看到Rootmotion位移方向已经是沿着Z轴，这一小段位移转化成速度后Apply到MovementMode_Walking，几乎等于没生效。

> 引擎中一些默认的LogCategory是设置为Warning，如果想输出该类型的Log级别信息，需要重新设置Category默认级别。例如这里调试RootMotion的信息使用的是LogRootMotion类型，可以想单次调整Log级别，可以直接在控制台输出Log LogRootMotion Log 
> 
> 如果想持久化设置Log级别，可以直接在项目Config目录下的DefaultEngine.ini添加下配置
>> [Core.Log]  
>> LogRootMotion = Log
> 


![NoAdditive](AdditiveAnim/NoAdditive.gif ':size=70%')
![AdditiveAttack](AdditiveAnim/Additive.gif ':size=70%')

可以看到在AdditiveAnim设置下，提取出来的根骨骼Transform发生了变化。其原因在于根骨骼附带的初始化rotation数据，经过叠加动画设置Delta处理后被抵消了，看起来是在水平面的位移，乘以RefPose下根骨骼的inversetransform后变换到根骨骼坐标系下，就发生了变化。如果按照小白人Mannequin的格式，初始旋转都清空后就不会有这些麻烦事了。

![Extract](AdditiveAnim/Extract.PNG ':size=70%')

如果想在这种格式的骨骼数据，并且有在叠加动画中启用RootMotion的需求。可以参考笔者如下的尝试，在ExtractRootMotion的过程中针对叠加动画多做一步数据处理。如果有更好的方法，也欢迎大家一起沟通讨论。

![Fix](AdditiveAnim/fix.png ':size=50%')