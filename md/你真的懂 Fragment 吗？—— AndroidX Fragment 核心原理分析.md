> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6970998913754988552)

> Fragment 是一个历史悠久的组件，从 API 11 引入至今，已经成为 Android 开发中最常用的组件之一； 在这个专题里，我们将从「使用 & 核心原理 & 面试」三个层面来讨论。

> **点赞关注，不再迷路，你的支持对我意义重大！**
> 
> 🔥 **Hi，我是丑丑。本文 [GitHub · Android-NoteBook](https://github.com/pengxurui/Android-NoteBook) 已收录，这里有 Android 进阶成长路线笔记 & 博客，欢迎跟着彭丑丑一起成长。（联系方式在 GitHub）**

* * *

*   Fragment 是一个历史悠久的组件，从 API 11 引入至今，已经成为 Android 开发中最常用的组件之一；
*   在这个专题里，我们将从「使用 & 核心原理 & 面试」三个层面来讨论 Fragment。如果能帮上忙，请务必点赞加关注，这真的对我非常重要。

* * *

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a22fcbdecb7640d3a89d7a6adce70827~tplv-k3u1fbpfcp-watermark.image)

* * *

#### 1.1 Fragment 解决了什么问题？（过去）

Fragment 可以将 Activity 视图拆分为多个区块进行模块化地管理 ，避免了 Activity 视图代码过度臃肿混乱。虽然自定义 View 或 Window 也可以在一定程度拆分 Activity 界面，但事实上它们的职责不同：View / Window 的职责是封装某种视图功能，而 Fragment 是在更高的层次使用控制自定义 View。此外，Fragment 还可以更方便地管理生命周期和事务（虽然我们会通过 MVP 或 MVVM 模式分离业务逻辑，但是对于复杂页面，我们还是无法避免 Activity 视图代码演化的非常臃肿混乱）。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21f4fdc28441459ba6b891e43b33266d~tplv-k3u1fbpfcp-watermark.image)

需要注意的是，Fragment 不能脱离 Activity 独立存在，必须由 Activity 或另一个 Fragment 托管，**Fragment#onCreateView 实例化的视图最终会被嵌入到宿主的视图树中。**

<table><thead><tr><th>类</th><th>角色</th><th>MVVM 分层</th><th>生命周期感知</th></tr></thead><tbody><tr><td>Activity</td><td>视图控制器</td><td>View 层</td><td>感知</td></tr><tr><td>Fragment</td><td>视图控制器</td><td>View 层</td><td>感知</td></tr><tr><td>View</td><td>视图</td><td>View 层</td><td>不感知</td></tr><tr><td>Window</td><td>视图</td><td>View 层</td><td>不感知</td></tr></tbody></table>

#### 1.2 Fragment 存在什么问题？（现在）

Fragment 的最初设计理念是 “一个微型 Activity” 的角色，正所谓 “欲戴王冠，必受其重”，很多专门为 Activity 设计 的 API 也需要添加到 Fragment 中，比如运行时权限，多窗口模式切换等新 API。这无疑是在无限制地扩充 Fragment 的职责边界，也在增大 Fragment 设计的复杂度，要知道 Fragment 的本质思想是界面模块化而已。

#### 1.3 Fragment 2.0（未来）

Google 正在重新构思 Fragment 的定位，随着 [AndroidX Fragment 版本](https://developer.android.google.cn/jetpack/androidx/releases/fragment) 陆续更新，新版 Fragment 正在渐渐走进我们的视野，我们称新版 Fragment 为 Fragment 2.0，已知的新特性包括：

*   FragmentScenario：Fragment 的测试框架；
*   FragmentFactory：统一的 Fragment 实例化组件；
*   FragmentContainerView：Fragment 专属视图容器；
*   OnBackPressedDispatcher：在 Fragment 或其他组件中处理返回按钮事件。

具体分析见：[Android | 上车！AndroidX Fragment 新姿势！](https://www.jianshu.com/p/4c8ebe671f13)

* * *

现在，我们正式开始讨论 Fragment 的核心工作原理，分析的过程中我会结合读源码，帮助你更清晰地、全面地理解 Fragment 的工作原理。不过，在开始之前我们有必要先梳理 Fragment 源码的整体结构设计，为下文深入阅读源码打下基础。

#### 2.1 代码框架

`FragmentActivity.java`

```
final FragmentController mFragments = FragmentController.createController(new HostCallbacks());

protected void onCreate(@Nullable Bundle savedInstanceState) {
    mFragments.attachHost(null /*parent*/);
    super.onCreate(savedInstanceState);

    ...
    // FragmentController 中定义了很多 dispatchXX() 方法
    mFragments.dispatchCreate();
}
复制代码

```

如下 UML 类图描述了 Framgent 整体的代码框架：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb7551f416f143d99aeb8223af3bd776~tplv-k3u1fbpfcp-watermark.image)

要点如下：

*   FragmentActivity 是 Activity 支持 Fragment 的基础，其中持有一个 FragmentController 中间类，它是 FragmentActivity 和 FragmentManager 的中间桥接者，对 Fragment 的操作最终是分发到 FragmentManager 来处理；
    
*   FragmentManager 承载了 Fragment 的核心逻辑，负责对 Fragment 执行添加、移除或替换等操作，以及添加到返回堆栈。它的实现类 FragmentManagerImpl 是我们主要的分析对象；
    
*   FragmentHostCallback 是 FragmentManager 向 Fragment 宿主的回调接口，Activity 和 Fragment 中都有内部类实现该接口，所以 Activity 和 Fragment 都可以作为另一个 Fragment 的宿主（Fragment 宿主和 FragmentManager 是 1 : 1 的关系）；
    
*   FragmentTransaction 是 Fragment 事务抽象类，它的实现类 BackStackRecord 是事务管理的主要分析对象。
    

下图描述了每个宿主与关联的 FragmentManager 的关系：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dee2ec0071684b53bc6a36b369f88605~tplv-k3u1fbpfcp-watermark.image)

> **提示（必看）：在后面的讨论中，我们不会再感知 FragmentActivity 与 Fragment 中间的 FragmentController，因为这属于软件设计模式的实现细节，而不是 Fragment 的核心源码。阅读源码我们一定要拨开表面看本质，抓流程，不要拘泥细枝末节。**

#### 2.2 说一下 Fragment 生命周期？

生命周期感知是 Fragment 的最基础的功能，也是面试的重灾区，我认为在 “背诵” 生命周期之前，如果你先向面试官阐述你对以下问题的理解，或许是更棒的回答：

*   **问题 1：什么是生命周期，生命周期回调方法（比如 onCreateView()）是生命周期的本质吗？**

答：不然。状态转移才是生命周期的本质（Activity 同理）。生命周期方法的本质是 Fragment 状态转移，当生命周期方法被调用，说明 Fragment 从一个状态转移到另一个状态，而所谓的 “生命周期回调” 只是 Framework 提供给开发者使用的 Hook 点，用于在状态转移时执行自定义逻辑。

*   **问题 2：你提到状态转移，那你说下 Fragment 有哪几种状态？**

答：从源码看，AndroidX Fragment 定义了以下五种状态，相对于早期的 Support Fragment 版本少了 STOPPED 等状态，这是因为 Google 认为这些状态是可以对称使用的，例如 STOPPED 状态和 STARTED 状态其实没有本质区别。

`Fragment.java`

```
static final int INITIALIZING = 0;     初始状态，Fragment 未创建
static final int CREATED = 1;          已创建状态，Fragment 视图未创建
static final int ACTIVITY_CREATED = 2; 已视图创建状态，Fragment 不可见
static final int STARTED = 3;          可见状态，Fragment 不处于前台
static final int RESUMED = 4;          前台状态，可接受用户交互
复制代码

```

*   **问题 3：Fragment 生命周期与宿主同步的吗，如果不是，是独立的吗？**

答：不然。Fragment 的生命周期主要受「宿主」、「事务」、「setRetainInstance() API」三个因素影响：当宿主生命周期发生变化时，会触发 Fragment 状态转移到 宿主的最新状态。不过，使用事务和 setRetainInstance() API 也可以使 Fragment 在一定程度上与宿主状态不同步（需要注意：宿主依然在一定程度上形成约束）。

*   宿主：【见第 3 节】
*   事务：【见第 4 节】
*   setRetainInstance() API : 【见第 6 节】

下面这张图完整描绘了 Fragment 生命周期：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/153c8d103557464a9737ca4152778e8f~tplv-k3u1fbpfcp-watermark.image)

—— 图片引用自网络

内容很多，别怕。跟着我的节奏一步步分析下去，也就那样了：“就这？🙇”

* * *

前面提到，Fragment 的生命周期受到三个因素影响，我们暂且讨论宿主的因素。

#### 3.1 Activity 与 Fragment 生命周期的同步关系

当宿主生命周期发生变化时，Fragment 的状态会同步到宿主的状态。从源码看，体现在宿主生命周期回调中会调用 FragmentManager 中一系列 dispatchXXX() 方法来触发 Fragment 状态转移。

`FragmentActivity`

```
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    mFragments.attachHost(null /*parent*/);
    ...
    mFragments.dispatchCreate(); // 最终调用 FragmentManager#dispatchCreate()
}
复制代码

```

下表总结了 Activity 生命周期与 Fragment 生命周期的关系：

<table><thead><tr><th>Activity 生命周期</th><th>FragmentManager</th><th>Fragment 状态转移</th><th>Fragment 生命周期回调</th></tr></thead><tbody><tr><td>onCreate</td><td>dispatchCreate</td><td>INITIALIZING<br>-&gt; CREATE</td><td>-&gt; onAttach<br>-&gt; onCreate</td></tr><tr><td>onStart（首次）</td><td>dispatchActivityCreated<br>dispatchStart</td><td>CREATE<br>-&gt; ACTIVITY_CREATED<br>-&gt; STARTED</td><td>-&gt; onCreateView<br>-&gt; onViewCreated<br>-&gt; onActivityCreated<br>-&gt; onStart</td></tr><tr><td>onStart（非首次）</td><td>dispatchStart</td><td>ACTIVITY_CREATED<br>-&gt; STARTED</td><td>-&gt; onStart</td></tr><tr><td>onResume</td><td>dispatchResume</td><td>STARTED<br>-&gt; RESUMED（Fragment 可交互）</td><td>-&gt; onResume</td></tr><tr><td>onPause</td><td>dispatchPause</td><td>RESUMED<br>-&gt; STARTED</td><td>-&gt; onPause</td></tr><tr><td>onStop</td><td>dispatchStop</td><td>STARTED<br>-&gt; ACTIVITY_CREATED</td><td>-&gt; onStop</td></tr><tr><td>onDestroy</td><td>dispatchDestroy</td><td>ACTIVITY_CREATED<br>-&gt; CREATED<br>-&gt; INITIALIZING</td><td>-&gt; onDestroyView<br>-&gt; onDestroy<br>-&gt; onDetach</td></tr></tbody></table>

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/63280ec450454d56b03be2dd6d48a7fa~tplv-k3u1fbpfcp-watermark.image)

#### 3.2 状态转移核心源码分析

FragmentManager 中一系列 dispatchXXX() 方法会触发 Fragment 状态转移，我们点进去看看：

> **提示：** 源码方法跳转太多，不利于理解核心流程。我直接帮你梳理出核心流程，跟你直接看源码会不同，但逻辑是相同的。

`FragmentManager.java`

```
（源码方法跳转太多，我直接帮你梳理出核心流程，跟你直接看源码会不同，但逻辑是相同的）
public void dispatchCreate() {
    mStateSaved = false;
    mStopped = false;
    moveToState(Fragment.CREATED, false);
    4、处理未执行的事务（见第 4 节）
    execPendingActions();
}

void moveToState(int newState, boolean always) {
    1、状态判断
    if (nextState == mCurState) {
        return;
    }
    mCurState = nextState;

    2、执行添加的 Fragment
    // Must add them in the proper order. mActive fragments may be out of order
    for (int i = 0; i < mAdded.size(); i++) {
        Fragment f = mAdded.get(i);
        // 更新 Fragment 到当前状态
        moveFragmentToExpectedState(f);
    }

    3、执行未添加，但是准备移除的 Fragment
    // Now iterate through all active fragments. These will include those that are removed and detached.
    for (int i = 0; i < mActive.size(); i++) {
        Fragment f = mActive.valueAt(i);
        if (f != null && (f.mRemoving || f.mDetached) && !f.mIsNewlyAdded) {
            // 更新 Fragment 到当前状态
            moveFragmentToExpectedState(f);
        }
    }
}
复制代码

```

其中，moveFragmentToExpectedState() 最终调用到 moveToState(Fragment, int) :

`FragmentManager.java`

```
-> moveFragmentToExpectedState 最终调用到 
-> 更新 Fragment 到当前状态
void moveToState(Fragment f, int newState) {
    1、准备 Detatch Fragment 的情况，不再与宿主同步，进入 CREATED 状态
    if ((!f.mAdded || f.mDetached) && newState > Fragment.CREATED) {
        newState = Fragment.CREATED;
    }
    
    2、移除 Fragment 的情况，Fragment 不再与宿主同步
    if (f.mRemoving && newState > f.mState) {
        if (f.isInBackStack()) {
            2.1 移除动作添加到返回栈，则进入 CREATED 状态
            newState = Math.min(nextState, Fragment.CREATED);
        } else {
            2.1 移除动作添加到返回栈，则进入 DESTROY 状态
            newState = Math.min(nextState, Fragment.INITIALIZING);
        }
    }

    3、真正执行状态转移
    if (f.mState <= newState ) {
        switch (f.mState) {
            case Fragment.INITIALIZING:
                if (nextState> Fragment.INITIALIZING) {
                    ...
                }
            // fall through
            case Fragment.CREATED:
                ...
                // fall through
            case Fragment.ACTIVITY_CREATED:
                ...
                // fall through
            case Fragment.STARTED:
                ...
        }
    } else {
         switch (f.mState) {
            case Fragment.RESUMED:
                if (newState < Fragment.RESUMED) {
                    ...
                }
            // fall through
            case Fragment.STARTED:
            ...
            // fall through
            case Fragment.ACTIVITY_CREATED:
            ...
            // fall through
            case Fragment.CREATED:
            ...
        }
    }
    ...
}
复制代码

```

以上代码已经非常简化了，总结一下：

触发状态转移时，首先会判断 Fragment，如果已经处于目标状态 newState，则会跳过状态转移。然而，并不是 FragmentManager 里所有的 Fragment 都会执行状态转移，只有 **「mAdded 为真 && mDetached 为假」** 的 Fragment 才会更新到目标状态，其他 Fragment 会脱离宿主状态。最后，状态转移完成后会处理未执行的事务`execPendingActions();`，可见每次 dispatchXXX() 都会提供一次事务执行的窗口。

不同 Fragment 标志位（Detach / Remove / 返回栈）与最终状态的关系总结如下表：

<table><thead><tr><th>情况</th><th>判断</th><th>描述</th><th>最终状态</th></tr></thead><tbody><tr><td>1</td><td>f.mRemoving</td><td>移除</td><td>Fragment.INITIALIZING</td></tr><tr><td>2</td><td>f.mRemoving &amp;&amp; f.isInBackStack()</td><td>移除，但添加进返回栈</td><td>Fragment.CREATED</td></tr><tr><td>3</td><td>!f.mAdded || f.mDetached</td><td>未添加</td><td>Fragment.CREATED</td></tr><tr><td>4</td><td>f.mAdded</td><td>已添加</td><td>newState（同步到宿主状态）</td></tr></tbody></table>

> **提示：** 这些标志位可以通过事务进行干涉。

#### 3.3 典型场景生命周期

基本规律：Activity 状态转移触发 Fragment 状态转移。

```
首次启动：
Activity - onCreate
Fragment - onAttach
Fragment - onCreate
Fragment - onCreateView
Fragment - onViewCreated
Activity - onStart
Fragment - onActivityCreated
Fragment - onStart
Activity - onResume
Fragment - onResume
-------------------------------------------------
退出：
Activity - onPause
Fragment - onPause
Activity - onStop
Fragment - onStop
Activity - onDestroy
Fragment - onDestroyView
Fragment - onDestroy
Fragment - onDetach
-------------------------------------------------
回到桌面：
Activity - onPause
Fragment - onPause
Activity - onStop
Fragment - onStop
-------------------------------------------------
返回：
Activity - onStart
Fragment - onStart
Activity - onResume
Fragment - onResume
复制代码

```

* * *

现在，我们来讨论影响 Fragment 状态转移的第二个因素：事务。

#### 4.1 事务概述

*   **问题 1：事务的特性是什么？** 事务是恢复和并发的基本单位，具备 4 个基本特性：
    
    *   原子性：事务不可分割，要么全部完成，要么全部失败回滚；
    *   一致性：事务执行前后数据都具有一致性；
    *   隔离性：事务执行过程中，不受其他事务干扰；
    *   持久性：事务一旦完成，对数据的改变就是永久的。在 Android 中体现为 Fragment 状态保存后，commit() 提交事务会抛异常，因为这部分新提交的事务影响的状态无法保存。
*   **Fragment 事务的作用：** 使用事务 FragmentTransaction 可以动态改变 Fragment 状态，使得 Fragment 在一定程度脱离宿主的状态。不过，事务依然受到宿主状态约束，例如：当前 Activity 处于 STARTED 状态，那么 addFragment 不会使得 Fragment 进入 RESUME 状态。只有将来 Activity 进入 RESUME 状态时，才会同步 Fragment 到最新状态。
    

#### 4.2 你知道不同事务操作的区别吗？

*   **add & remove**：Fragment 状态在 INITIALIZING 与 RESUMED 之间转移；
*   **detach & attach：** Fragment 状态在 CREATE 与 RESUMED 之间转移；
*   **replace：** 先移除所有 containerId 中的实例，再 add 一个 Fragment；
*   **show & hide：** 只控制 Fragment 隐藏或显示，不会触发状态转移，也不会销毁 Fragment 视图或实例；
*   **hide & detach & remove 的区别：** hide 不会销毁视图和实例、detach 只销毁视图不销毁实例、remove 会销毁实例（自然也销毁视图）。不过，如果 remove 的时候将事务添加到回退栈，那么 Fragment 实例就不会被销毁，只会销毁视图。

下图描述了 Fragment 状态转移与宿主和事务的简单关系：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0554a3da35ec4684a364d5d8c6c1d899~tplv-k3u1fbpfcp-watermark.image)

这里有一个让人摸不着头脑的问题，detach Fragment 并不会回调 onDetach()，因为 detach 只会转移到 CREATE 状态，而回调 onDetach() 需要转移到 INITIALIZING。不知道 Google 为什么要采用这么有歧义的命名。

```
detach Fragment：
Fragment - onPause
Fragment - onStop
Fragment - onDestroyView
复制代码

```

#### 4.3 说说看不同事务提交方式的区别？

FragmentTransaction 定义了 5 种提交方式：

<table><thead><tr><th>API</th><th>描述</th><th>是否同步</th></tr></thead><tbody><tr><td><strong>commit()</strong></td><td>异步提交事务，不允许状态丢失</td><td>异步</td></tr><tr><td><strong>commitAllowingStateLoss()</strong></td><td>异步提交事务，允许状态丢失</td><td>异步</td></tr><tr><td><strong>commitNow()</strong></td><td>同步提交事务，不允许状态丢失</td><td>同步</td></tr><tr><td><strong>commitNowAllowingStateLoss()</strong></td><td>同步提交事务，允许状态丢失</td><td>同步</td></tr><tr><td><strong>executePendingTransactions()</strong></td><td>同步执行事务队列中的全部事务</td><td>同步</td></tr></tbody></table>

需要注意的地方：

*   **onSaveInstanceState() 保存状态后，事务形成的新状态是不会被保存的。在状态保存之后调用 commit() 或 commitNow() 会抛异常**

`FragmentManagerImpl.java`

```
private void checkStateLoss() {
    if (mStateSaved || mStopped) {
        throw new IllegalStateException("Can not perform this action after onSaveInstanceState");
    }
}
复制代码

```

*   **使用 commitNow() 或 commitNowAllowingStateLoss() 提交的事务不允许加入回退栈**

为什么有这个设计呢？可能是 Google 考虑到同时存在同步提交和异步提交的事务，并且两个事务都要加入回退栈时，无法确定哪个在上哪个在下是符合预期的，所以干脆禁止 commitNow() 加入回退栈。如果确实有需要同步执行 + 回退栈的应用场景，可以采用`commit() + executePendingTransactions()`的取巧方法。相关源码体现如下：

`BackStackRecord.java`

```
@Override
public void commitNow() {
    disallowAddToBackStack();
    mManager.execSingleAction(this, false);
}

@Override
public void commitNowAllowingStateLoss() {
    disallowAddToBackStack();
    mManager.execSingleAction(this, true);
}
@NonNull
public FragmentTransaction disallowAddToBackStack() {
    if (mAddToBackStack) {
        throw new IllegalStateException("This transaction is already being added to the back stack");
    }
    mAllowAddToBackStack = false;
    return this;
}    
复制代码

```

*   **commitNow() 和 executePendingTransactions() 都是同步执行，有区别吗？**

commitNow() 是同步执行当前事务，而 executePendingTransactions() 是同步执行事务队列中的全部事务。

* * *

#### 5.1 添加方法

有两种方式可以将 Fragment 添加到 Activity 视图上：静态加载 + 动态加载

*   **静态加载：** 静态加载是指在布局文件中使用 标签添加 Fragment 的方式，要点总结如下：

<table><thead><tr><th>属性</th><th>描述</th></tr></thead><tbody><tr><td>class</td><td>Fragment 的全限定类名</td></tr><tr><td>android:name</td><td>Fragment 的全限定类名（与 class 没有差别，但 class 优先）</td></tr><tr><td>android:id</td><td>Fragment 唯一标识</td></tr><tr><td>android:tag</td><td>Fragment 唯一标识（id 和 tag 至少设置一个）</td></tr></tbody></table>

举例：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal"
    android:layout_width="match_parent"
    android:layout_height="match_parent">
    <fragment android:
            android:id="@+id/list"
            android:layout_weight="1"
            android:layout_width="0dp"
            android:layout_height="match_parent" />
</LinearLayout>
复制代码

```

*   **动态加载：** 动态加载是指在代码中使用事务 FragmentTransaction 添加 Fragment 的方式。例如：

```
TextFragment fragment = new TextFragment();
fragmentTransaction.add(R.id.containerId, fragment);
fragmentTransaction.commit();
复制代码

```

#### 5.2 Fragment 静态加载源码分析

从布局文件添加 Fragment 本质上是 xml 解析为视图树的过程，这个过程由 LayoutInflater 完成。最终， 标签的解析工作最终是交给 FragmentManager#onCreateView(...) 处理的，让我们来看看具体是如何处理的，源码如下：

`FragmentManagerImpl.java`

```
（已简化）
public View onCreateView(@Nullable View parent, @NonNull String name, @NonNull Context context, @NonNull AttributeSet attrs) {
    1、解析属性
    String fname = 解析 class 属性
    if (fname == null) {
        fname    = 解析 android:name 属性
    }
    int id       = 解析 android:id 属性
    String tag   = 解析 android:tag 属性
    
    2、根据 id 或 tag 重用已经创建的 Fragment
    Fragment fragment = id != View.NO_ID ? findFragmentById(id) : null;
    if (fragment == null && tag != null) {
        fragment = findFragmentByTag(tag);
    }
    if (fragment == null && containerId != View.NO_ID) {
        fragment = findFragmentById(containerId);
    }

    3、新建 Fragment
    if (fragment == null) {
        3.1 反射创建 Fragment 实例
        fragment = getFragmentFactory().instantiate(context.getClassLoader(), fname);
        3.2 mFromLayout 设置为 true
        fragment.mFromLayout = true;
        fragment.mFragmentId = id != 0 ? id : containerId;
        fragment.mContainerId = containerId;
        fragment.mTag = tag;
        fragment.mInLayout = true;
        fragment.mFragmentManager = this;
        fragment.mHost = mHost;
        fragment.onInflate(mHost.getContext(), attrs, fragment.mSavedFragmentState);
        3.3 添加 Fragment，立即状态转移
        addFragment(fragment, true);
    } else if (fragment.mInLayout) {
        ...
    } else {
        ...
    }

    4.1 将 id 设置给 Fragment 根布局
    if (id != 0) {
        fragment.mView.setId(id);
    }
    4.2 将 tag 设置给 Fragment 根布局
    if (fragment.mView.getTag() == null) {
        fragment.mView.setTag(tag);
    }
    
    5、返回 Fragment 根布局
    return fragment.mView;
}

-> 3.3 添加 Fragment，立即状态转移
public void addFragment(Fragment fragment, boolean moveToStateNow) {
    ...
    if (moveToStateNow) {
        moveToState(fragment); // 状态转移
    }
}
-> 状态转移
void moveToState(Fragment f, int newState, ...) {
    ...
    if (f.mState <= newState ) {
        switch (f.mState) {
            case Fragment.INITIALIZING:
                if (nextState> Fragment.INITIALIZING) {
                    ...
                }
            // fall through
            case Fragment.CREATED:
                if (f.mFromLayout && !f.mPerformedCreateView) { // 如果来自布局，并且未执行过 onCreateView
                    最终调用：
                    mView = onCreateView(inflater, container, savedInstanceState);
                    f.onViewCreated(f.mView, f.mSavedFragmentState);
                    提示：最终在 LayoutInflater 中执行 viewGroup.addView(view, params);
                }
                if (!f.mFromLayout) { // 不是来自布局
                    最终调用：
                    mView = onCreateView(inflater, container, savedInstanceState);
                    f.onViewCreated(f.mView, f.mSavedFragmentState);
                    if (container != null) {
                        container.addView(f.mView);
                    }
                }
                ...
                // fall through
            case Fragment.ACTIVITY_CREATED:
                ...
                // fall through
            case Fragment.STARTED:
                ...
        }
    } else {
         ...
    }
}
复制代码

```

以上代码已经非常简化了，代码虽然长但是流程很清楚：

*   1、FragmentManager 根据布局中的 id 属性或 tag 属性来重用 Fragment，如果不存在则通过反射来创建 Fragment 实例。
*   2、设置 mFromLayout 为 true，并立即执行状态转移。在 moveToState() 的 CREATE 分支会根据 mFromLayout 判断：如果来自布局，并且未执行过 onCreateView，才会回调 Fragment#onCreateView 创建 View 实例。
*   3、最终回溯到 LayoutInflater 中，执行 ViewGroup#addView(mView)，将 Fragment 根布局添加到父布局中（所以，我们不用在 Fragment 里创建的视图时调用 addView() ）。

在我之前写的一篇文章里已经详细讨论过布局解析的全过程：[Android | 带你探究 LayoutInflater 布局解析原理](https://juejin.cn/post/6886052422260228103)，关于 的部分在第 4.2 节，记得去看看。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/559a60b02e374e4a8bf2d4055fdddc62~tplv-k3u1fbpfcp-watermark.image)

#### 5.3 Fragment 动态加载源码分析

Fragment 事务的源码在 **第 4.4 节** 已经讨论过了，我们知道了通过事务添加 / 移除的 Fragment 最终还是会走到 moveToState(...) 来执行状态转移。在创建 View 实例后，mView 也会直接添加到 containerId 容器上。

#### 5.4 静态加载和动态加载的区别体现在哪里？

静态加载和动态加载的主要区别体现在 **执行加载操作的消息周期不同**：静态加载和布局解析是在同一个 Handler 消息周期中，而动态加载和事务提交不一定在一个 Handler 消息周期中（取决于调用 commit() 还是 commitNow()）。

#### 5.5 静态加载和动态加载的优缺点？

这个问题看似合理，但其实经不起推敲，是个伪命题。~较常见的说法是静态加载简单直接，而动态加载灵活性跟高。~ 提出这个说法的人其实忽略了一点：从布局文件中静态加载的 Fragment 也可以使用事务进行动态操作，静态加载也是具有灵活性的。

比较合理的问法是：静态加载和动态加载各适合什么场景？静态加载适合于界面初始化时就确定显示位置和时机的 Fragment，从布局文件中加载可以方便预览。相反地，动态加载适用于初始化时无法确定显示位置和时机的 Fragment，需要依赖代码中的判断条件动态判断。

```
演示：使用事务操作从布局文件中静态加载的 Fragment
with(supportFragmentManager.beginTransaction()) {
    val fragmentA = supportFragmentManager.findFragmentById(R.id.FragmentA)
    if (null != fragmentA) {
        hide(fragmentA)
        commit()
    }
}
复制代码

```

* * *

事实上，从 [androidx.fragment 1.3.0](https://developer.android.google.cn/jetpack/androidx/releases/fragment#1.3.0) 开始，setRetainInstance() 这个 API 已经废弃了。不过，考虑到这个 API 的重要性，我们还是花费一点时间来回顾一下。

#### 6.1 概述

*   **问题 1：什么时候应该使用 setRetainInstance(true)？**

答：在配置变更时（例如屏幕旋转），整个 Activity 需要销毁重建，顺带着 Activity 中的 Fragment 也需要销毁重建。而设置 setRetainInstance(true) 的 Fragment 对象在 Activity 销毁重建的过程中不会被销毁。

*   **问题 2：setRetainInstance(true) 对 Fragment 生命周期的影响？**

答：在 Activity 销毁时，**Fragment 不会回调 onDestroy()**，而是直接回调 onDestroy() + onDetach()；在 Activity 重建时，**Fragment 不会回调 onCreate()**，而是直接调用 onCreateView()。

*   **问题 3：为什么废弃 setRetainInstance()？**

答：引入 ViewModel 后，setRetainInstance() API 开始变得鸡肋。ViewModel 已经提供了在 Activity 重建等场景下保持数据的能力，虽然 setRetainInstance() 也具备相同功能，但需要利用 Fragment 来间接存储数据，使用起来不方便，存储粒度也过大。

#### 6.2 setRetainInstance() 核心源码分析

`Fragment.java`

```
@Deprecated
public void setRetainInstance(boolean retain) {
    mRetainInstance = retain;
    if (mFragmentManager != null) {
        if (retain) {
            mFragmentManager.addRetainedFragment(this);
        } else {
            mFragmentManager.removeRetainedFragment(this);
        }
    } else {
        mRetainInstanceChangedWhileDetached = true;
    }
}
复制代码

```

`FragmentManager.java`

```
void addRetainedFragment(@NonNull Fragment f) {
    mNonConfig.addRetainedFragment(f);
}
复制代码

```

`FragmentManagerViewModel.java`

```
void addRetainedFragment(@NonNull Fragment fragment) {
    if (mIsStateSaved) {
        if (FragmentManager.isLoggingEnabled(Log.VERBOSE)) {
            Log.v(TAG, "Ignoring addRetainedFragment as the state is already saved");
        }
        return;
    }
    if (mRetainedFragments.containsKey(fragment.mWho)) {
        return;
    }
    mRetainedFragments.put(fragment.mWho, fragment);
    if (FragmentManager.isLoggingEnabled(Log.VERBOSE)) {
        Log.v(TAG, "Updating retained Fragments: Added " + fragment);
    }
}
复制代码

```

这段代码并不复杂，当我们调用 Fragment#setRetainInstance(true) 时，最终会将 Fragment 添加到一个 ViewModel 中。ViewModel 是具备在 Activity 重建是恢复数据的能力的，**现在的问题转换为 ViewModel 为什么可以恢复数据？**

简单来说，在 Activity 销毁时，最终会调用 Activity#retainNonConfigurationInstances() 保存 ActivityClientRecord，并托管给 ActivityManagerService。这个过程就相当于把 Fragment 保存到更长的生命周期了。关于 ViewModel 的具体分析，我后面会专门写一篇文章，期待吗？

* * *

我们前面讲了 Fragment 一些历史问题的由来，以及它的一些核心特性，包括生命周期、事务、加载方式和已过时的 setRetainInstance(true)。关于 Fragment 的话题还有很多，今天我们只讨论了其中最核心的部分，更多内容我后续会继续发布更多文章来讨论。

* * *

#### 参考资料

*   [Fragment 的过去、现在和将来](https://zhuanlan.zhihu.com/p/149937029) —— 谷歌开发者
*   [Fragment 的过去、现在和将来（Youtube 视频版）](https://www.youtube.com/watch?v=RS1IACnZLy4&t=3s) —— Android Dev Summit '19
*   [Fragment 指南](https://developer.android.google.cn/guide/fragments) —— 官方文档
*   [Activity 都重建了，你 Fragment 凭什么活着？](https://www.wanandroid.com/wenda/show/11077) —— Wan Android
*   [今天考察下 Fragment 相关两个不常见 API](https://wanandroid.com/wenda/show/12229) —— Wan Android
*   [Fragment 是如何被存储与恢复的？](https://www.wanandroid.com/wenda/show/12574) —— Wan Android
*   [Fragment 生命周期](https://juejin.cn/post/6844903752114126855) —— 更木小八 著
*   [Fragment 不为人知的细节](https://mp.weixin.qq.com/s/y2JqIBR3mYQOzCEymv7DOQ) —— 三雒 著

* * *

> **创作不易，你的「三连」是丑丑最大的动力，我们下次见！**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74d8d9c542a04361927adfbd1cc5a090~tplv-k3u1fbpfcp-zoom-1.image)