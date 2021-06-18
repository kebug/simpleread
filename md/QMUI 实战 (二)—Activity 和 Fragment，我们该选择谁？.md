> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6844903822733623303)

> 在一开始， 官方只提供了 Activity 来作为 UI 界面的载体，因此我们也别无选择，只能用它。

在一开始， 官方只提供了 `Activity` 来作为 UI 界面的载体，因此我们也别无选择，只能用它。而在 Android 3.0 后，`Fragment` 也面世了，它一开始是用于适配平板的，以邮件列表与详情的适配为例，手机端够小，因此开始展示列表，点击进入详情，而平板够大，则可以列表显示在左侧，详情显示在右侧，点击列表只是切换详情。对于这种适配场景，列表页和详情页必须在同一个 `Activity` 里了，而这便是我所知道的 `Fragment` 诞生的场景了。但是随着 `Fragment` 框架的不断完善， 单 `Activity` 多 `Fragment` 的架构也被提出了，甚至现在官方的 navigation 库都是这种模式， 可见官方对 `Fragment` 是多么的青睐了。

当然，现在我们写 UI 也不只是这两种选择了， Reative Native， Flutter，以及将来的 compose 为我们写 UI 提供了很多很多的选择，同时我们学习的东西也越来越多了。今天我们只讨论一些 `Activity` 和 `Fragment` 相关的话题，对于另外的几个，我只是期待 compose 时代的早点到来。

Activity
--------

提到 `Activity`，可能最容易犯的一个错误就是忘记在 AndroidManifest 里注册了吧，直到运行出错，才想起要去注册，然后再编译，时间就是这样没了的。对于 `Activity` 的使用，我们需要关注生命周期、启动模式、`setContentView` 外，我们还需要掌握的一个关键知识点就是 `SaveState` 了。

iOS 端、 Web 端都是不存在 `SaveState` 的，因而这几乎是 Android 独有的。因为 Android 最初的年代是比较看轻 `View` 的。 在 Android 眼里， `View` 是廉价的，数据、状态是宝贵的，`View` 是可以随时销毁，然后根据状态进行恢复的。例如在横竖屏旋转、Dark Mode 切换这些场景中，Android 就选择销毁掉当前 `Activity`，然后再重新创建，重新创建时会去 resource 里读当前配置下的资源，例如字体大小、颜色等等，这些资源都是可以根据不同状态在不同文件件里配置成不同的值，因而我们就不需要去写各种判断。但是问题来了，当 UI 显示出来后，大多数情况是有数据渲染或状态变更的，那么销毁重建后，我们的数据该如何恢复呢？ 这就要靠 `SaveState` 机制了：在销毁时，我们把数据保存下来，然后在重建时，我们再把数据恢复出来。

当然，你也可以选择不要让系统重建你 `Activity`，而是自己处理配置的变更。做法就是在 `AndroidManifest` 文件里为 `Activity` 的 `configChanges` 属性里加上你想要自己处理的配置。例如：

```
android:configChanges="orientation|keyboardHidden|screenSize|smallestScreenSize|screenLayout|uiMode"
复制代码

```

具体的配置项可以看[官方文档](https://developer.android.com/guide/topics/manifest/activity-element)。 例如 orientation 便是横竖屏旋转， uiMode 可以用于接收 Dark Mode 的切换。当这些配置好后。 我们在 `Activity` 里可以通过重写 onConfigurationChanged 来接受配置的变更。

我们说回 `SaveState`。 当发生配置变更、或者因为内存等原因被系统杀死的 `Activity`, 会调用 `onSaveInstanceState` 来保存 UI 状态，默认情况下会遍历 `View`, 然后以 `View` 的 id 为键来收集 `View` 的状态。当然每个 View 都是可以重写 `onSaveInstanceState` 来更改默认行为， 例如 `RecyclerView`/`ListView` 就阻止了子 `View` 状态的保存。那么这里就有一个问题了。 **如果同一个 Activity 里出现了两个 id 相同的 `View`， 那会怎样呢？ 那样就会发生状态的覆盖，状态恢复也会混乱**。不要以为这种事情不会发生，我来列举下有可能出现的场景，看看你有没有遇到呢？

1.  现在界面越来越复杂了，同一个 `Activity` 里可能会使用 `ViewPager` 套 `ViewPager` 的现象，而很多人起 id 也很随意，不会业务化，都叫 `viewPager`，然后就会发现，我原本选中的是第二个 tab，怎么屏幕一旋转，就变成第三个了？这就是因为 `ViewPager` 会保存当前选中的是哪个 tab，多个相同 id 的 `ViewPager`, 在状态保存是后者覆盖前者，恢复过来后永远是最后一个 ViewPager 选中的状态了。
2.  有许多人切换 `Fragment` 时, 使用的是 add 而不是 replace。这样就造成同一时刻可能有很多 `Fragment` 共存，每一个 `Fragment` 在界面上的表现就是一个 `View`, 这样无形增加了系统布局、渲染的时间外，也是有可能状态保存于恢复出错的， 例如每个 `Fragment` 都有一个 `RecyclerView`，然后它的 id 是 `recyclerView`， 这样界面上就有很多个同 id 的 `RecyclerView` 了，而 `RecyclerView` 在状态保存时会保存滚动位置的，那么当屏幕旋转后，你所有的 `RecyclerView` 状态都恢复成了最后一个保存的 `RecyclerView` 的了，就会出现滚动到了开始或者某个奇奇怪怪的位置了。
3.  在 `RecyclerView` 没成熟之前，我们做横向的翻页滚动，基本都是用 `ViewPager` 做的（又是它），如果我们 `ViewPager` 的每一个 Pager 都采用相同的 xml， 那又可能出现这种问题了，假想一下，你每个 Pager 都有一个进度条，结果屏幕一旋转，进度全都乱套了。 那么大家现在就知道原因了。 `RecyclerView` 不会出现这种问题，因为它拦截了 `SaveState` 的派发，完全由 Adapter 的数据决定 UI 的渲染。

在复杂的 UI 下，各种奇奇怪怪的状态问题总是容易出现，但问题的本质都是相同的，因此熟悉 UI 的人，在遇到这样的场景，会快速反应过来问题出现在哪里了，进而进行修复。而如果能够在写 UI 时就考虑到这种问题，起 id 名不随意，那就更妙了。

这其实也暴露了另外一点，我们在做自定义 View 时，是否有考虑这些问题，坦白点讲， QMUI 在这方面做得就不够好，我们更多的强调自适应，而推荐用 configChange 来规避了 `StateSave` 的问题。

最后一点，如果我们有大量的数据，这种 `StateSave` 的肯定是不满足需求的，这个时候，`Activity` 提供了 `onRetainNonConfigurationInstance` 的方式来保存状态无关的数据，如果你使用 `ViewModel`, 你就根本不需要关系这些了， 在 `Activity` 销毁与重建后，使用的是同一个 `ViewModel`， 因此还是早点用上 `ViewModel` 吧，它不止你表面上看到的那些东西。

Fragment
--------

`Fragment` 虽然比 `Activity` 轻量，有很多特性，但是使用起来是比 `Activity` 复杂的， 相应的坑点也比 `Activity` 多。

第一点就是它的生命周期有两个，一个是 `Fragment` 的生命周期, 另外一个就是 `Fragment` 管理的 `View` 的生命周期。为什么会有两个呢？ 前面已经说过了，Android 是轻 UI 的，那么从 FragmentA 切换到 FragmentB 时，就会去释放掉 FragmentA 管理的 `View`，从 FragmentB 返回到 FragmentA 时，再重新创建 `View`。 所以 `Fragment.onCreateView(三参数)` 、`Fragment.onDestoryView()`会被多次调用，这或许是很多人都会疑惑的点。

我们在切换 `Fragment` 的时候就会发生 `View` 的销毁，那么同样需要做 `StateSave` 了，因此，`Fragment` 的 `onSaveInstanceState` 与 `onViewStateRestored` 基本上是与 `View` 的生命周期挂钩的，而不仅仅是屏幕旋转、Dark Mode 切换时才调用了。当然，一般情况下，在 `Activity` 销毁与重建时，Fragment 也会销毁与重建。但如果你调用了 `Fragment.setRetainInstance(true)` ，它就会走 `Activity` 的 `onRetainNonConfigurationInstance`，而不会销毁重建了，但是 View 的生命周期依旧会走销毁重建。

`Fragment` 存在 `View` 这个生命周期的另外一点就是我们不能像在 `Activity`里一样随意操作 UI。 如果你在 `onDestoryView` 之后操作了 UI，那么当 `View` 重建后，之前的操作就白费了。这个时候 `ViewModel` 和 `LiveData` 就显得特别重要了， 它可以让你在数据在正确的生命周期里渲染到 UI 上。但是需要注意的是我们对 `LifeCircleOwner` 的选择， `Fragment` 是有两个 `LifeCircleOwner` 的，如果与 UI 相关， 我们应该选用 `viewLifeCircleOwner`, 并且应该在 `onActivityCreated` 里调用 `LiveData` 的 `observe` 方法。

`Fragment` 存在另外一个问题就是转场动画的问题了。如果在转场动画过程中发生了数据渲染，那么就会产生动画卡顿现象，因为数据渲染造成了 UI 重新 layout，而 `Activity` 是 window 动画，不受这个影响。对比起来，可能 `Activity` 的动画更加流畅了。 而我们也没办法解决 `Fragment` 的这个问题，只能避免在动画过程中进行数据渲染了，因此 QMUI 提供了 `runAfterAnimation` 方法。

`Fragment` 是用 `FragmentManager` 来管理的，然后通过 `FragmentTransaction` 来实现 `Fragment` 的添加、删除与切换等。 当然 `Fragment` 提供了 `childFragmentManager`，这使得我们可以一层一层的添加 `Child Fragment`。 而当 `Activity` 切换到不同生命周期状态时，会沿着 `FragmentManager` 派发给所有的 `Fragment`。 这是很有用的，例如在做 `ViewPager` 的 `Pager` 时，我们可以采用自定义 `View`， 也可以采用 `Fragment`。但是当 `Pager` 需要在 `onResume`、`onPause` 等时机做一些事情时，如果用自定义 `View` 时，那就需要我们写一堆的代码来控制这一切，而采用 `Fragment` 时，则可以方便的运用这个特性来实现。因此官方也提供了 `FragmentPagerAdapter`。 如果我们有一些公用的业务 UI 大组件， 不妨也通过它来封装，方便生命周期的管理。

QMUI 为 Activity 和 Fragment 添加的功能
--------------------------------

QMUI 的 arch 库，为 `Activity` 和 `Fragment` 添加了一些新的功能。

首先就是手势返回，这个在 iOS 上是自带的，而 Android 却需要开发者自己去实现。一个主要的原因就是 Android 是主张 View 可以随时创建与销毁。 以 `Fragment` 为例，当我们从 FragmentA 切换到 FragmentB 后， FragmentA 的 View 已经销毁了，这个时候手势返回时肯定无法获取到之前那个 View 的，因此默认是无法实现类似 iOS 那种手势返回的。但产品们就是希望看到这种手势返回效果， 所以我们也必须做，对于 `Fragment` 而言， QMUI 就是在 QMUI 层用成员变量保存了 `Fragment` 创建的 View， 然后在下次创建的时候直接传入缓存的 `View`， 这直接打破了原本的官方的逻辑，也就会引入新的翔， 比如我们可能会在 `onActivityCreated` 为 TopBar 添加子 View，但因为 View 缓存后，就会出现重复添加的问题，因此 QMUI 也提供了 `onViewCreated(一参数)` 规避这个问题。`Fragment` 的手势返回大量的运用了发射来更改 `FragmentManager` 里记录的数据信息，这是基于阅读 `FragmentManager` 源码后找到的插入点，一般而言，这种实现是有版本兼容问题的：如果官方更新了实现，那我的反射也就会失败，但目前 `Fragment` 框架比较稳定了，也不会轻易去改核心功能的，所以影响其实还好。

相比于 `Fragment` 的手势返回。 `Activity` 的手势返回实现会简单很多，但是会存在一些无法解决的问题。目前主流的有三种实现方式：

1.  手势返回将 `Activity` 透明掉， 这样就可以看到背后的 `Activity` 了。但是调用透明 `Activity` 的方法是反射，而且比较耗时，并且背后的 `Activity` 没办法移动, 做不了视差。
2.  手势返回时将前一个 `Activity` 的 View 取下来，然后添加到当前 `Activity` 的 View 的下层。 手势返回后再将 View 放回去。这个方案存在 `View` 跨 `Window` 的添加与移除，而且无法处理背后的 `Activity`有显示 `Dialog` 的场景。
3.  手势返回时将前一个 `Activity` 的 View 以及 Dialog 的 View 绘制到当前 `Activity` 的背后，这个是 QMUI 提供的方案，性能可能是最优的， 因为只是多了绘制，但问题是如果前一个 `Activity` 在 `onPause` 之后更改了 View 的显示，那就可能在手势返回时有错误的表现（例如 FaceBook 提供的 DraweeView），另外它对 SurfaceView 等支持也不太好，对于这种场景，直接禁用掉手势返回比较好，其它的手势返回其实也不能很好的支持 SurfaceView 这些。

当 Android 10 出来后，它提供了新的 Navigation Gesture，很多国产机已经开始这么做了，这这种场景下，系统的手势返回会优先于我们的手势返回，这个时候这套手势返回就有点鸡肋了（尽管它的交互必系统的更优雅，更贴合 iOS）。

QMUI 提供的另外一些功能就只是一些 util 性质的函数了。例如 `startFragment`、`startFragmentAndDestroyCurrent`等，后一个是一个特殊化的实现，对于官方设计而言，他们觉得应该遵循用户行为，我从 A 点击进入 B，然后进入 C， 那么我返回是就应该从 C 返回 B，再返回到 A。但产品需求可能不是这样，例如我把某篇文章分享给某个用户，那么我要先打开用户列表，选择某个用户，再进入用户的聊天界面。当我从聊天界面返回时，应该是返回的文章界面，而不是中间的用户列表界面。而我们的 `Fragment` 并没有 `Activity` 的 `finish` 方法来结束自己，这就是 `startFragmentAndDestroyCurrent` 存在的价值了。

另外需要提一下的是 `FragmentManager.popBackStack()` 这个方法，它的意思是将当前 `Fragment` 从堆栈中移除，我们的返回就是通过它做的，但是它却不是任何时候都能调用的，一般在用户主动的点击行为下调用是没有问题的，但如果你想在 `Fragment.onCreate` 里调用，是不行的，即使是在 `onResume` 里也是不行的，对于上面那个分享的需求场景，如果你想以返回到用户列表时立刻执行 `popBackStack` 来取代 `startFragmentAndDestroyCurrent`，那是会失败的，如果有兴趣，你可以自己试试，官方的 `Fragment` 会直接 crash 掉，而 `QMUIFragment` 则会忽略这次执行。其原因是因为 `FragmentManager` 里的状态机要在 `onResume` 之后才赋值当前状态。 这是一个很常见的重入报错的问题，我简单写个类似的例子，看完后你估计就能猜到是怎么回事了：

```
interface Observer{
    fun doSomething()
}

class Observable{
    private val observers = arrayListOf<Observer>()
    
    fun addObserver(observer: Observer){
        observers.add(observer)
    }
    
    fun removeObserver(observer: Observer){
        observers.remove(observer)
    }
    
    fun dispatch(){
        val count = observers.size
        for(i in 0 until count){
            observers[i].doSomething()
        }
    }
}
复制代码

```

上面这个代码很简单，就是观察者模式的运用，大家能否看出其中错误呢？

假设我的调用是这样：

```
val observable = Observable()
val secondObserver = object : Observer {
    override fun doSomething() {
        //....
    }
}
val firstObserver = object : Observer {
    override fun doSomething() {
        observable.removeObserver(secondObserver)
    }
}
observable.addObserver(firstObserver)
observable.addObserver(secondObserver)
observable.dispatch()
复制代码

```

如果运行这段代码，你会发现报 `IndexOutOfBoundsException`，这是因为我们在第一个 Observer 将第二个 Observer 移除了，从而造成 Observable.dispatch 里的 for 循环的 count 错误。 我们的 `Fragment.onResume` 也处于这种阶段，如果在这里面去移除 `Fragment`，可能导致 `FragmentManager` 里的某些变量值出错。当然， `FragmentManager` 里是有保护的，是不允许你的某些操作的，你操作了就抛出错误，终止整个执行。

QMUI arch 框架目前也在尝试引入一些注解来简化代码，例如 `FirstFragment` 注解，例如 `LatestVisitRecord` 注解。在之后的实战过程中，我会详细介绍这些注解使用的场景。

#总结 今天主要介绍了一些 `Activity` 和 `Fragment` 的知识。知识点是比较零散的，需要我们在使用过程中不断地学习与掌握。而且我们不是做选择题，而是两者都要会，在适合的场景引入 `Fragment`，可以极大的减少工作量。例如在插件化场景中，我们就不需要解决 `Activity` 的注册问题了。 而 `SaveState`、`ViewModel`、手势返回等都埋藏了很多细节的知识点，我们要想熟练的驾驭它，就需要熟悉它的各种坑点以及坑点产生的原因。新手往往写点 UI 就去看看效果，这是对各个控件极其不熟悉的原因。如果能做到写半天的代码，一次编译，结果效果全是自己想要的，那编写 UI 的能力就是真正的上了一个台阶了。

GankWithQmui 会采用单 Activity 多 Fragment 的架构，这是我比较喜欢的架构。下一次我们就开始构建我们的第一个 `Fragment` 了。

下期博文：QMUI 实战 (三)——你是如何启动你的第一个 Fragment 的？