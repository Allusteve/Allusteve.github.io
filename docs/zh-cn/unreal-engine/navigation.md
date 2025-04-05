# 项目中寻路模块定制化开发的一些记录

## 自动导入寻路数据的流程

1.  编写CommandLet命令行，配合QA的自动化流程去加载预先定义好路径的Map， 触发LoadPackage流程
2.  将地图加载进内存后，进行World初始化，并触发委托回调到我们写的导入外部寻路数据函数中
3.  CheckOut地图资源并将修改后的Package数据序列化到本地，最后通过SourceControl接口上传到仓库中

### 遇到一些的问题

#### 关卡流送后寻路数据丢失

1.  寻路数据在关卡流送的过程中，是会自动Discard子关卡中保存的NavDataActor，而去从子关卡Level里保存的NavDataChunks里读取并合并寻路数据，而数据丢失的原因在于，子关卡的NavDataChunks是空的。
2.  在加载外部寻路数据的过程中，会在反序列化数据后手动调用OnNavMeshGenerationFinished()去生成NavDataChunk，这一步骤和引擎原生的BuildPath是一致的
3.  而调试过程中发现，此时NavigationSystem的RegisteredNavBounds是空的，引擎认为在NavBoudns外的寻路数据都是过期的数据，所以直接丢弃了NavDataChunk

#### 问题原因和解决方法

NavigationSystem的RegisteredNavBounds是在World初始化过程中的GatherNavigationBounds()里去完成的。最早的开发同学没用深思，直接将导入寻路数据的时机绑定到了OnNavigationInitStartStaticDelegate()，此时OnWorldInitDone才刚开始，还没有调用GatherNavigationBounds()，从而导致在加载后的NavDatChunk因为空的NavBounds而被剔除调了

解决办法非常简单，直接绑定OnNavigationInitDoneStaticDelegate委托即可。

这也提醒我们开发人员，要清楚地了解引擎中定义好的各个接口。不要像刚开始写UE代码时，喜欢把一切初始化逻辑都放到BeginPlay一样。要去深刻地理解各个接口的调用时机和上下文


## 程序化生成NavLink
施工中