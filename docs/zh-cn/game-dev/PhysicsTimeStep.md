# 翻译|Gaffer On Games - Fix Your TimeStep

> 原文地址： [Fix Your Timestep!](https://gafferongames.com/post/fix_your_timestep/)

## Introduction
你好，我是Glenn Fiedler，欢迎来到游戏物理的世界。

在之前的文章里我们讨论了如何使用数值积分器来对运动学方程进行积分。积分听起来比较复杂，但这只是一种将物理模拟向前推进一小段时间的方法，我们把这一小段时间称为“delta时间”（简称dt）。

但是如何选择这个delta时间值呢？这似乎是一个微不足道的话题，但事实上有很多不同的方法可以做到这一点，每种方法都有自己的优点和缺点——所以请继续阅读！

## Fixed Delta Time
最简单的前进方式是使用固定的Delta时间，比如使用一秒的1/60
```
double t = 0.0;
double dt = 1.0 / 60.0;

while(!quit)
{
    intergate(state, t, dt);
    render(state);
    t += dt;
}
```
在许多方面，这段代码都是理想的。如果你足够幸运，你的delta时间与显示器刷新率匹配，并且你可以确保你的更新循环占用的实时时间少于一帧，那么你已经有了更新物理模拟的完美解决方案，你可以停止阅读这篇文章了。

但在现实世界中，你可能不会提前知道显示器的刷新率。VSYNC可能被关闭，或者你可能在一台速度较慢的计算机上运行，该计算机无法以60帧的速率来完成物理更新和渲染帧。

在这种情况下，你的模拟会比你预期的跑得更快或更慢。

## Variable Delta Time
解决这个问题似乎很简单。只需记录上一帧所需的时间，然后将该值反馈为下一帧的delta time。当然，这是有道理的，因为如果计算机速度太慢，无法在60HZ下更新，并且必须降至30fps，你会自动通过1/30作为delta time。对于75HZ而不是60HZ的显示器刷新率，甚至在快速计算机上关闭VSYNC的情况下也是如此：
```
double t = 0.0;

double currentTime = hires_time_in_seconds();

while ( !quit )
{
    double newTime = hires_time_in_seconds();
    double frameTime = newTime - currentTime;
    currentTime = newTime;

    integrate( state, t, frameTime );
    t += frameTime;

    render( state );
}
```
但这种方法存在一个巨大的问题，我现在将对此进行解释。问题的关键是，你的物理模拟的行为取决于你所传入的delta time。这种影响可能很微妙，轻微的情况是，根据你实际游戏的运行帧率不同，会感觉物理模拟不太正确；极端的情况就比如，弹簧模拟爆炸到无穷大，快速移动的物体穿过墙壁，玩家从地板上掉下来！


但有一点是肯定的，那就是期望你的模拟能够正确处理传递给它的任意delta time是完全不现实的。为了理解原因，考虑一下如果你用十分之一秒作为delta time传入给物理模拟系统？一秒钟怎么样？10秒？100？最终你会找到一个转折点。

## Semi-fixed timestep
更现实的说法是，只有当delta time小于或等于某个一个最大值时，物理模拟才能表现良好。在实践中，这通常比试图让delta在取值范围很广的条件下还要保证模拟防弹（Bulletproof，这里的含义应该是物理模拟更稳定和鲁棒）要容易得多。

掌握了这些知识，这里有一个简单的技巧，可以确保你在不同的机器上以正确的速度运行的同时，永远不会传入一个超过最大值的delta time：
```
    double t = 0.0;
    double dt = 1 / 60.0;

    double currentTime = hires_time_in_seconds();

    while ( !quit )
    {
        double newTime = hires_time_in_seconds();
        double frameTime = newTime - currentTime;
        currentTime = newTime;
              
        while ( frameTime > 0.0 )
        {
            float deltaTime = min( frameTime, dt );
            integrate( state, t, deltaTime );
            frameTime -= deltaTime;
            t += deltaTime;
        }

        render( state );
    }
```
这种方法的好处是，我们现在有了delta时间的上限。它永远不会大于这个值，因为如果超过了最大值，我们会细分时间步长。缺点是，我们现在每次显示更新都要采取多个步骤，包括一个额外的步骤来消耗不可被dt整除的帧时间的剩余部分。如果你的程序性能瓶颈是渲染模块，这是没有问题的，但如果你的模拟是一帧中最昂贵的部分（耗费时间多），你可能会遇到所谓的“死亡螺旋”。

什么是死亡螺旋？当你的物理模拟无法跟上要求的时间步长时，就会发生这种情况。例如，如果你的模拟被告知：“好吧，请模拟X秒的物理”，如果在Y>X的情况下需要Y秒的时间，那么不需要爱因斯坦也能意识到随着时间的推移，你的模拟会落后。这被称为死亡螺旋，因为落后会导致你的更新模拟在一帧内花费更多的substep来追赶，这会导致你进一步落后，并会让你模拟更多的substep…

那么该如何避免这种情况呢？为了确保稳定的更新，我建议留出一些空间。您确实需要确保更新X秒物理模拟所需的时长明显少于X秒。如果你能做到这一点，那么你的物理引擎可以通过模拟更多的帧来“赶上”任何临时的性能峰值。或者，您可以以限制在一帧内可以模拟的最大Steps，虽然会导致模拟在性能高负载下会减慢。但可以说，这比螺旋上升到死亡要好，尤其是如果沉重的性能负载只是一个临时情况。

## Free the physics
现在，让我们对这个问题更进一步讨论。如果你想要在相同输入的情况下，从一次运行到下一次运行的结果能够保证同样的精度，该怎么办？当尝试使用确定性帧同步（deterministic lockstep）对物理模拟进行网络化时，这很有用，但通常也很好的一点是，知道模拟从一次运行到下一次运行的行为完全相同，而不可能根据渲染帧率产生任何不同的行为。

但你会问，为什么必须有完全固定的delta时间才能做到这一点？带有小余数步长的半固定delta时间肯定“足够好”吗？是的，你是对的。它在大多数情况下都足够好，但由于浮点运算的精度有限，它并不完全相同。

我们想要的是两全其美：用于模拟的固定delta time加上以不同帧率渲染的能力。这两件事似乎完全不一致，而且事实确实如此——除非我们能找到一种方法来解耦模拟和渲染帧率。

以下是操作方法。以固定的dt时间步长提前进行物理模拟，同时确保它跟上来自渲染器的计时器值，以便模拟以正确的速率进行。例如，如果显示帧速率为50fps，模拟以100fps运行，则每帧显示更新都需要执行两次物理模拟更新。非常简单。

如果显示帧速率为200fps怎么办？在这种情况下，每次渲染更新我们需要花半个物理时长，但我们不能这样做，我们必须以常量dt去更新。因此，我们每两帧渲染更新就进行一次物理模拟更新。

更棘手的是，如果显示帧速率是60fps，但我们希望模拟以100fps运行呢？没有简单的整型倍数。如果VSYNC（垂直同步）被禁用，并且显示帧速率在帧与帧之间波动，该怎么办呢？

如果你感觉到头脑发热，别担心，解决这个问题所需要的就是改变你的观点。不要认为在渲染之前必须模拟一定量的帧时间，而是将视点倒置，这样想：渲染器产生时间，模拟以离散dt大小的步长去消耗时间。

例如：
```
   double t = 0.0;
    const double dt = 0.01;

    double currentTime = hires_time_in_seconds();
    double accumulator = 0.0;

    while ( !quit )
    {
        double newTime = hires_time_in_seconds();
        double frameTime = newTime - currentTime;
        currentTime = newTime;

        accumulator += frameTime;

        while ( accumulator >= dt )
        {
            integrate( state, t, dt );
            accumulator -= dt;
            t += dt;
        }

        render( state );
    }
```
请注意与半固定时间时长不同，我们只对时长大小为dt的时间步长进行积分，因此在常见情况下，我们在每帧结束时都会留下一些未模拟的时间。剩余的时间通过accumulator变量传递到下一帧，不会被丢弃。

## The final touch
但是每次计算剩下的时间怎么办？这似乎不正确，不是吗？

为了理解发生了什么，请考虑一个显示帧速率为60fps，物理运行速度为50fps的场景。因为两个速率大小没有一个“好的”倍数，所以当每帧剩余的时间“累积”到accumulator并超过dt以上时，会导致物理模拟在每帧主要计算一个物理TimeStep和偶尔计算两个物理TimeStep之间交替。

现在考虑一下，大部分的渲染帧将在accumulator中剩下一些无法模拟的帧时间，因为它小于dt。这意味着我们在与渲染时间略有不同的时间显示物理模拟的状态，导致屏幕上物理模拟出现微妙但视觉上令人不快的停顿。

这个问题的一个解决方案是基于Accumulator中剩余的时间在先前和当前物理状态之间进行插值：
```   
    double t = 0.0;
    double dt = 0.01;

    double currentTime = hires_time_in_seconds();
    double accumulator = 0.0;

    State previous;
    State current;

    while ( !quit )
    {
        double newTime = time();
        double frameTime = newTime - currentTime;
        if ( frameTime > 0.25 )
            frameTime = 0.25;
        currentTime = newTime;

        accumulator += frameTime;

        while ( accumulator >= dt )
        {
            previousState = currentState;
            integrate( currentState, t, dt );
            t += dt;
            accumulator -= dt;
        }

        const double alpha = accumulator / dt;

        State state = currentState * alpha + 
            previousState * ( 1.0 - alpha );

        render( state );
    }
```
这看起来比较复杂但是这有一个简单的方式去解释它。在accumulator里任何剩下的时间，实际上都是衡量在计算另一个完整的物理步骤之前还需要多少时间。例如，accumulator里剩下了dt/2 时间，意味着我们目前处于当前物理步骤和下一个物理步骤之间的中间。dt*0.1的剩余时间意味着物理更新是在当前状态和下一状态之间的1/10。

我们可以使用这个剩余时间来获得之前和当前物理状态之间的混合系数，只需简单地除以dt。这给出了一个在[0,1]范围内的Alpha值，该值用于在两个物理状态之间执行线性插值，以获得用来渲染的当前物理状态。
对于单个值和矢量状态值，这种插值很容易实现。如果将方向存储为四元数并使用球面线性插值（slerp）在之前的方向和当前的方向之间混合，则甚至可以将其用于完整的三维刚体动力学。