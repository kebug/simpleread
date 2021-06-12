> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6850037271714856968)

> 不知道大家有没有发现，Android 版的掘金有下面这个小小动画：点击作者头像跳转到作者的详情页，而作者头像会从当前界面通过动画过渡到详情页界面。

**不知道大家有没有发现，Android 版的掘金有下面这个小小动画：点击作者头像跳转到作者的详情页，而作者头像会从当前界面通过动画过渡到详情页界面。**

**![](https://user-gold-cdn.xitu.io/2020/7/10/1733940d938fe766?imageslim)****

知识贫乏限制了我的视野，真心想不到这怎么实现的？

最近在写动画方面文章时候，从网上找到了答案：**「Activity 过渡动画中的共享元素过渡」**。

本文的初衷，是和大家一起扫盲，如果对你有用，**「欢迎点赞」**，让更多的小伙伴多学点知识。小小的动画，隐藏着巨大的知识点；怪不得面试造火箭，工作拧螺丝，这是知识储备，虽然可能一辈子也用不上。

【**「系列好文推荐」**】

[Android 属性动画, 看完这篇够用了吧](https://juejin.im/post/6846687601118691341)

[Android 矢量图动画：每人送一辆掘金牌小黄车](https://juejin.im/post/6847902224396484621)

### 一、Activity 切换过渡动画

`Activity`过渡动画包含**「进入过渡」**和**「退出过渡」**、**「共享元素过渡」**三个动画，它们同样仅支持`Android 5.0+`版本。

#### 一)、共享元素过渡动画

共享元素过渡指的两个`Activity`共享的视图如何在两个`Activity`之间进行过渡。例如上面的`Gif`图，共享视图就是`ImageView`。

共享元素也分一个元素和多个元素。

定义共享元素过渡效果**「步骤」**如下：

1.  在两个`Activity`定义两个相同类型的 View；
2.  给两个`View`设置相同的`transitionName`属性；
3.  通过`ActivityOptions.makeSceneTransitionAnimation()`函数生成`Bundle`对象；
4.  `startActivity()`函数传递`bundle`对象。

**「栗子讲解，清晰易懂：」**

1.  分别在`activity_first.xml`和`activity_second.xml`布局文件定义`ImageView`组件，并将`transitionName`属性设为`activityTransform`。

```
<!--activity_first.xml文件内容-->

<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/white"
    android:orientation="vertical">

    <ImageView
        android:id="@+id/ivImage"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:src="@mipmap/ic_one"
        android:transition />

    <TextView
        android:id="@+id/tvText"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="10dp"
        android:gravity="center"
        android:text="我是第一个Activity"
        android:textColor="@color/c_333"
        android:textSize="18sp" />
</LinearLayout>

<!--activity_second.xml文件内容-->
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <ImageView
        android:id="@+id/ivImage"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:adjustViewBounds="true"
        android:src="@mipmap/ic_one"
        android:transition />

    <TextView
        android:id="@+id/tvText"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_above="@id/ivImage"
        android:layout_marginBottom="10dp"
        android:gravity="center"
        android:text="我是第2个Activity"
        android:textColor="@color/c_333"
        android:textSize="18sp" />
</RelativeLayout>
复制代码

```

**「预览图」** ![](https://user-gold-cdn.xitu.io/2020/7/11/173398cdb47bb032?imageView2/0/w/1280/h/960/ignore-error/1) `activityTransform`属性也可以通过代码设置。

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    ivImage.transition
}
复制代码

```

2.  在`FirstActivity`中给`ImageView`设置点击事件，跳转到第二个 Activity。

```
ivImage.setOnClickListener {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {//判断Android版本
        val bundle =
            ActivityOptions.makeSceneTransitionAnimation(this, ivImage, "activityTransform")
                .toBundle()
        startActivity(Intent(this, SecondActivity::class.java), bundle)
    } else {
        startActivity(Intent(this, SecondActivity::class.java))
    }
}
复制代码

```

代码中，先判断当前`Android`版本是否大于等于 5.0，大于或等于`Android 5.0`的话就设置共享元素动画, 小于 5.0 就正常启动第二个`Activity`。

通过`ActivityOptions.makeSceneTransitionAnimation()`创建启动`Activity`过渡的一些参数，`makeSceneTransitionAnimation()`函数第一个参数为`Activity`对象; 第二个参数为共享元素组件，这里设置为`id`是`ivImage`的`ImageView`视图；第三个参数为`transitionName`属性的值，这里是`activityTransform`。在调用`AcivityOptions`对象`toBundle`函数，包装成`Bundle`对象。

**「效果图：」**

![](https://user-gold-cdn.xitu.io/2020/7/11/173399a99a8948a3?imageslim) **「多个共享元素过渡」**

多个共享元素过渡也很简单，只需要调用`makeSceneTransitionAnimation()`函数的另外一个重载函数即可。

1.  在前面 XML 布局的基础上，给`TextView`增加`transitionName`属性:`textTransform`。

```
<!--activity_first.xml文件内容-->
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/white"
    android:orientation="vertical">

    <ImageView
        android:id="@+id/ivImage"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:src="@mipmap/ic_one"
        android:transition />

    <TextView
        android:id="@+id/tvText"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="10dp"
        android:gravity="center"
        android:transition
        android:text="我是第一个Activity"
        android:textColor="@color/c_333"
        android:textSize="18sp" />
</LinearLayout>

<!--activity_second.xml文件内容-->
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <ImageView
        android:id="@+id/ivSecondImage"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:adjustViewBounds="true"
        android:src="@mipmap/ic_one"
        android:transition />

    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:transition
        android:layout_above="@id/ivSecondImage"
        android:layout_marginBottom="10dp"
        android:gravity="center"
        android:text="我是第2个Activity"
        android:textColor="@color/c_333"
        android:textSize="18sp" />
</RelativeLayout>
复制代码

```

2.  构建多个`Pair`对象，并传递给`makeSceneTransitionAnimation()`函数，启动`Activity`。

```
ivImage.setOnClickListener {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {             
    
    val imagePair=Pair<View,String>(ivImage,"activityTransform")
    val textPair=Pair<View,String>(ivImage,"textTransform")
    
    val bundle =
        ActivityOptions.makeSceneTransitionAnimation(this,
                imagePair,textPair).toBundle()
        
        startActivity(Intent(this, SecondActivity::class.java), bundle) 
    } else {
        startActivity(Intent(this, SecondActivity::class.java))
    }
}
复制代码

```

这里主要是通过将共享视图和`transitionName`属性的值包装到`Pair`对象，其他操作和一个共享元素的操作步骤并无区别。

**「效果图：」**

![](https://user-gold-cdn.xitu.io/2020/7/11/1733cfa764aa491f?imageslim)

**「深坑提醒」**

有时从`RecyclerView`界面进入到详情页，由于详情页加载延迟，可能出现没有效果。例如`ImageView`从网络加载图片，可能 A 界面到 B 界面没效果，B 回到 A 界面有效果。

![](https://user-gold-cdn.xitu.io/2020/7/13/173473efe2c8292e?imageslim) 解决步骤：

1.  在`setContentView`后添加下面代码，延迟加载过渡动画。

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    postponeEnterTransition()
}
复制代码

```

2.  在共享元素视图加载完毕，或者图片加载完毕后调用下面代码, 开始加载过渡动画。

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    startPostponedEnterTransition()
}
复制代码

```

![](https://user-gold-cdn.xitu.io/2020/7/13/173473cc8f694a47?imageslim) 例如我是在 Glide 加载完再调用：

```
 Glide.with(mContext)
                    .asBitmap()
                    .load(value?.avatar ?: "")
                    .listener(object : RequestListener<Bitmap> {
                        override fun onResourceReady(resource: Bitmap?, model: Any?, target: Target<Bitmap>?, dataSource: DataSource?, isFirstResource: Boolean): Boolean {
                            animatorCallback?.invoke()//回调开始加载过渡动画
                            return false
                        }

                        override fun onLoadFailed(e: GlideException?, model: Any?, target: Target<Bitmap>?, isFirstResource: Boolean): Boolean {
                            animatorCallback?.invoke()//回调开始加载过渡动画
                            return false
                        }
                    })
                    .apply(RequestOptions.circleCropTransform())
                    .placeholder(R.mipmap.ic_default)
                    .error(R.mipmap.ic_default)
                    .into(authorBinding!!.ivAvatar)
复制代码

```

大家也可以考虑下面代码：

```
shareElement.viewTreeObserver.addOnPreDrawListener(object : ViewTreeObserver.OnPreDrawListener {
                override fun onPreDraw(): Boolean {
                    shareElement!!.viewTreeObserver.removeOnPreDrawListener(this)
                    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                        animatorCallback?.invoke()
                    }
                    return true
                }
            })
复制代码

```

#### 二)、进入过渡与退出过渡动画

与共享元素相反的，就是 Activity 进入与退出过渡动画，两个 Activity 之间在没有共享的视图情况下进行动画切换。下面先看三种动画效果图：**「爆炸式效果」**和**「淡入淡出式效果」**、**「滑动式效果」**。

*   **「爆炸式」**：将视图移入场景中心或从中移出；
*   **「滑动式」**：将视图从场景的其中一个边缘移入或移出；
*   **「爆炸式」**：通过更改视图的不透明度，在场景中添加视图或从中移除视图；

第一个界面采用`Fade`淡入淡出效果，第二个界面采用了`Explode`爆炸效果。

![](https://user-gold-cdn.xitu.io/2020/7/12/17340f22a8a3dea0?imageslim) 前后两个界面都采用了`Slide`滑入滑出效果。

![](https://user-gold-cdn.xitu.io/2020/7/12/17340fc926f651e5?imageslim) 利用 Android 现有的过渡框架, 实现起来是很简单的，步骤如下：

1.  在`Activity`的`onCreate()`方法中调用 `setContentView()`前设置启用窗口过渡属性;

```
window.requestFeature(Window.FEATURE_CONTENT_TRANSITIONS)
复制代码

```

2.  创建过渡效果对象`Slide`、`Explode`、`Fade`;

```
val slide=Slide()
slide.slideEdge=Gravity.START
slide.duration=300//效果时长，一般Activity切换时间很短，不建议设置过长
复制代码

```

如果是`Slide`效果，可以设置`slideEdge`属性来指定滑动方向，默认是`Gravity.BOTTOM`。

3.  将过渡效果设置给 window 相关属性，设置；

```
//退出当前界面的过渡动画
window.exitTransition = slide
//进入当前界面的过渡动画
window.enterTransition = slide
//重新进入界面的过渡动画
window.reenterTransition = slide
复制代码

```

4.  调用第二个`Activity`界面，使用过渡效果。

```
 startActivity(
        Intent(this, SecondActivity::class.java),
        ActivityOptions.makeSceneTransitionAnimation(this).toBundle())
复制代码

```

那么`Activity`的`OnCreate()`方法看起来是这样子的。

```
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            window.requestFeature(Window.FEATURE_CONTENT_TRANSITIONS)
            window.allowEnterTransitionOverlap=false
            Slide().apply {
                duration = 300
                excludeTarget(android.R.id.statusBarBackground, true)
                excludeTarget(android.R.id.navigationBarBackground, true)
            }.also {
                window.exitTransition = it
                window.enterTransition = it
                window.reenterTransition = it
            }
        }
        
        setContentView(R.layout.activity_first)

        ivContent.setOnClickListener {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                startActivity(
                    Intent(this, SecondActivity::class.java),
                    ActivityOptions.makeSceneTransitionAnimation(this).toBundle()
                )
            }
        }
    }
复制代码

```

上面代码中调用 了`excludeTarget()`方法将状态栏和导航栏排除在过渡动画效果之外。否则会跟着一起起动画效果，不是很美观。

正常情况，退出与进入过渡动画会有一小段交叉的过程，而`window.allowEnterTransitionOverlap=false`就是禁止交叉，只有退出过渡动画结束后才会再显示进入过渡动画。

如果第二个`Activity`在`finish`掉后，回到第一个`Activity`界面也想有过渡效果，就不要手动调用`finish()`, 可以调用`finishAfterTransition ()`方法。

#### 三)、兼容 Android 5.0 前

如果 Android 5.0 前也想要有切换动画怎么办？

1.  在`res/anim`文件夹下创建想要的效果：

```
<alpha 
    xmlns:android="http://schemas.android.com/apk/res/android"
        android:interpolator="@interpolator/decelerate_quad"
        android:fromAlpha="0.0"
        android:toAlpha="1.0"
        android:duration="@android:integer/config_longAnimTime" />
复制代码

```

2.  在启动`Activity`后调用`overridePendingTransition()`方法。

```
val intent = Intent(this, TestActivity2::class.java)
startActivity(intent)
overridePendingTransition(R.anim.fade_in, R.anim.fade_out)
复制代码

```

`overridePendingTransition()`方法第一个参数为下一个界面进入动画，第二个参数为当前界面退出动画。

到这里，`Activity`的切换使用过渡动画基本就结束了。有朋友可能会问，只有 Activity 切换才能应用过渡效果么？

### 二、布局变化过渡动画

在上一节要理解一个概念：场景。布局的显示与隐藏可以理解分别为一个场景，过渡动画就是解决场景切换带来的生硬视觉感受。Activity 切换过渡动画指在两个 Activity 之间，而布局变化过渡动画，是指同个 Activity 之间 View 的变化过渡动画。

#### 一)、手动创建 Scene

手动创建场景的话，需要我们自己创建起始和结束场景，利用现有的过渡效果来达到两个场景的切换。默认情况下，当前界面就是起始场景。

1.  创建起始场景和结束场景的`xml`布局。起始场景和结束场景需要有相同根元素，例如下面代码`id`为`flConatent`的`FrameLayout`布局。

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:id="@+id/tvText"
        android:text="内容过渡动画"
        android:gravity="center"
        android:textSize="18sp"
        android:layout_width="match_parent"
        android:layout_height="50dp"/>

    <FrameLayout
        android:id="@+id/flContent"
        android:layout_weight="1"
        android:layout_width="match_parent"
        android:layout_height="0dp">
      <include layout="@layout/layout_first_scene"/>
    </FrameLayout>

</LinearLayout>
复制代码

```

初始视图，第一个场景, 布局`layout_first_scene.xml`：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">
    <TextView
        android:id="@+id/tvFirst"
        android:textSize="18sp"
        android:layout_marginTop="100dp"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:gravity="center_horizontal|top"
        android:text="感谢大家阅读文章" />
</LinearLayout>
复制代码

```

第二个场景, 布局`layout_second_scene.xml`：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:textSize="18sp"
        android:layout_marginTop="100dp"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:gravity="center_horizontal|top"
        android:text="我是新小梦\n欢迎大家点赞支持一下" />
</LinearLayout>
复制代码

```

2.  创建起始场景和结束场景。

```
val firstScene = Scene.getSceneForLayout(flContent, R.layout.layout_first_scene, this)
val secondScene = Scene.getSceneForLayout(flContent, R.layout.layout_sencod_scene, this)
复制代码

```

默认情况下，过渡动画应用整个场景，如果场景某个`View`不参加, 可以通过`过渡效果对象`的`removeTarget()`方法进行移除。

```
Slide(Gravity.TOP).removeTarget(tvNoJoin)
复制代码

```

3.  点击时，进行场景过渡。

```
tvText.setOnClickListener {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        if (isFirst) {
            TransitionManager.go(secondScene, Slide(Gravity.TOP))
        }else{
            TransitionManager.go(firstScene, Slide(Gravity.TOP))
        }
        isFirst=!isFirst
    }
}
复制代码

```

`TransitionManager.go()`第一个参数表示结束场景，第二个参数表示当前场景退出时过渡效果，当前场景就是初始场景。

**「效果图：」**

![](https://user-gold-cdn.xitu.io/2020/7/13/173484d6dc2be293?imageslim)

#### 二)、系统自动创建 Scene

这种情况，我们调用`TransitionManager.beginDelayedTransition(sceneRoot)`函数时，系统会自动记录当前`sceneRoot`节点下所有要进行动画的视图作为起始节点，下一帧中再次记录`sceneRoot`子节点下所有 起始场景进行动画状态的视图作为结束场景。这种一般用来改变视图的属性，然后进行动画过渡，如`View`的宽高。

**「栗子」**：

定义只有一个正方形的`View`, 通过改变正方形的宽高为原来的 2 倍，来看看动画效果。

1.  `activity_text.xml`布局文件，定义`id`为`sceneRoot`的根节点，也是场景的根节点。

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:id="@+id/sceneRoot"
    android:background="@color/colorPrimary">

    <View
        android:id="@+id/vSquare"
        android:layout_width="100dp"
        android:layout_height="100dp"
        android:layout_centerHorizontal="true"
        android:layout_marginTop="50dp"
        android:background="@color/white" />
</RelativeLayout>
复制代码

```

2.  在`TestActivity`的`OnCreate`方法中调用下面代码，将正方形的宽高设置 200dp。

```
vSquare.setOnClickListener {
   TransitionManager.beginDelayedTransition(sceneRoot)
   vSquare.layoutParams.apply {
       width = dp2px(200f, this@TestActivity)
       height = dp2px(200f, this@TestActivity)
   }.also {
       vSquare.layoutParams = it
   }
}
复制代码

```

**「效果图：」**

![](https://user-gold-cdn.xitu.io/2020/7/14/1734adada152c203?imageslim)

### 三、过渡动画效果

上面的动画效果，都是采用系统内置的，那具体有哪些动画效果，或支持自定义么？

过渡效果类都继承自`Transition`类，`Transition`类持有场景切动画的相关信息，子类的主要作用是捕获属性值（例如起始值和结束值）和如何演奏动画。从这里也可以看出，过渡动画也是属性动画的一个扩展与应用。

#### 一)、内置过渡动画

系统支持将任何扩展`Visibility`类的过渡作为进入或退出过渡，内置继承自`Visibility`的类有`Explode`、`Slide`、`Fade`；支持共享元素过渡的有：

*   `changeScroll` 为目标视图滑动添加动画效果
*   `changeBounds` 为目标视图布局边界的变化添加动画效果
*   `changeClipBounds` 为目标视图裁剪边界的变化添加动画效果
*   `changeTransform` 为目标视图缩放和旋转方面的变化添加动画效果
*   `changeImageTransform` 为目标图片尺寸和缩放方面的变化添加动画效果

代码示例：

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    TransitionSet().apply {
        addTransition(ChangeImageTransform())
        addTransition(ChangeBounds())
        addTransition(Fade(Fade.MODE_IN))
    }.also {
       window.sharedElementEnterTransition=it
    }
}
复制代码

```

`TransitionSet`对象是动画的合集，可以将多个过渡效果组织起来。

也可以通过 XML 布局来实现, 在`res/transition`文件夹创建 ``：

```
<?xml version="1.0" encoding="utf-8"?>
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="300"
    android:transitionOrdering="together">
    <changeImageTransform />
    <changeBounds />
    <fade />
</transitionSet>
复制代码

```

`transitionSet`和`fade...`等等的一些属性和 [Android 矢量图动画：每人送一辆掘金牌小黄车](https://juejin.im/post/6847902224396484621)文章 讲到的一些属性大同小异，这里不再复述。

代码调用：

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    TransitionInflater.from(this).inflateTransition(R.transition.transition_set).also {
        window.sharedElementEnterTransition=it
    }
}
复制代码

```

**「效果图：」**

![](https://user-gold-cdn.xitu.io/2020/7/14/1734c06bc1d50c4e?imageslim) 当现有的过渡效果不满足日常需求时，可以通过继承`Transition`，定制自己的动画特效。

#### 二)、自定义过渡动画

子类继承`Transition`类，并重写其三个方法。

```
class MyTransition : Transition() {
   override fun captureStartValues(transitionValues: TransitionValues?) {}

   override fun captureEndValues(transitionValues: TransitionValues?) {}

   override fun createAnimator(
       sceneRoot: ViewGroup?,
       startValues: TransitionValues?,
       endValues: TransitionValues?
   ): Animator {
       return super.createAnimator(sceneRoot, startValues, endValues)
   }
   
}
复制代码

```

`captureStartValues()`与`captureEndValues()`方法是必须实现的，捕获动画的起始值和结束值，而`createAnimator()`方法，是用来创建自定义的动画。

参数`TransitionValues`可以理解是用来存储 View 的一些属性值，参数`sceneRoot`为根视图。

自定义过渡效果感兴趣可以参考：[Android 自定义 Transition 动画](https://blog.csdn.net/qibin0506/article/details/53248597)

好啦，过渡动画就讲到这里~

参考文章：

[官网文档](https://developer.android.google.cn/training/transitions/start-activity?hl=zh-cn)

[酷炫的 Activity 切换动画，打造更好的用户体验](https://www.jianshu.com/p/37e94f8b6f59)

[我的 GitHub](https://github.com/Android-XXM)

**「 【码字不易，点个赞，日后好查看】」**

本文使用 [mdnice](https://mdnice.com/?from=juejin) 排版

**