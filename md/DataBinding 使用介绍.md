> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6844904191656214536)

> DataBinding 是 2015 年谷歌 I/O 大会上发布的一个数据绑定框架，也就是把数据绑定到 UI 上，DataBinding 可以让 Activity 和 Fragment 减少很多逻辑，使其更容易维护、方便。

一 概述
----

DataBinding 是 2015 年谷歌 I/O 大会上发布的一个数据绑定框架，也就是把数据绑定到 UI 上，DataBinding 可以让 Activity 和 Fragment 减少很多逻辑，使其更容易维护、方便。同时也能提高性能，避免内存泄漏 以及 空指针 异常 ，同时 DataBinding 也可以双向绑定，使 UI 的改变同时同步到数据上，DataBinding 不是 MVVM 架构的必需品，但是官方已经为咱们提供了这个库，为啥我们不好好利用呢。

### 具体的 demo 地址 [github](https://github.com/niezhiyang/MVVMSimple)

二 DataBinding 优缺点
-----------------

### 2.1 优点

1.  不用再 findViewById 了 (当然 kotlin 也可以不用喽)
2.  减少了 Avtivity 和 Fragment 的逻辑处理，使 Activity 和 Fragment 逻辑更加清晰，容易维护
3.  提高性能，避免内存泄漏 以及 空指针
4.  双向绑定，当 View 改变的时候会通知 Model，当 Model 改变的时候会通知 View

### 2.2 缺点

1.  很难定位 bug，当有个界面展示不对的时候，你不知道是 View 的问题，还是 Model 的问题，还是编写逻辑的问题，
2.  xml 中 不能 Debug
3.  在 xml 中写代码，这个可能是我们 Android 开发有点反感的，同时 xml 里面是 Java 代码，不能使用 kotlin 的简洁代码
4.  双向绑定技术，不利于 View 的复用，因为一个 xml 里面绑定的一个 Model，有可能另一个界面 Model 就不一样了，所以无法复用了。除非你再手动转一下这个 Model

三 DataBinding 使用
----------------

### 3.1 引入

在 app 的 build.gradle 中加上以下代码即可，不用引用其他的依赖

```
android {
    ...
    dataBinding {
        enabled = true
    }
复制代码

```

然后打开布局的 xml，在根节点 mac 按住 command+return ，windows 按住 art+enter 就会出现这样的标记，选第一个就会出现 DataBinding 的 layout。如果没有这个选项，重启 studio 就可以了，就会生成 layout 的跟标签，data 标签下面就是咱们要绑定的实体类，比如一个 Cat 类 ![](https://user-gold-cdn.xitu.io/2020/5/18/17225e6614a38453?imageView2/0/w/1280/h/960/ignore-error/1)

```
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
     <variable
            
            type="com.nzy.mvvmsimple.databinding.model.Cat" />
    </data>
     <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".MainActivity">
         <TextView
            android:id="@+id/tv_name_ob"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@{cat.name}"
            android:textSize="20sp"
            app:layout_constraintLeft_toLeftOf="parent"
            app:layout_constraintRight_toRightOf="parent"
             />
        </androidx.constraintlayout.widget.ConstraintLayout>
</layout>
复制代码

```

然后在 Activity 中通过 DataBindingUtil 拿到刚才的 xml 的 bind 类, 生成此 bind 类默认是 xml 文件名的驼峰之后加一个 Binding 字符串，比如 xml 是 activity_binddata，那么生成的类是 ActivityBinddataBinding，这个类名也可以改，但是基本上都是用默认的、

```
var binding:ActivityBinddataBinding =
            DataBindingUtil.setContentView<ActivityBinddataBinding>(this, R.layout.activity_binddata)
            
        var cat = Cat("咖啡猫")
        binding.cat = cat
复制代码

```

在 Fragment 或者 RecyclerView 中 使用

```
val listItemBinding = ListItemBinding.inflate(layoutInflater, viewGroup, false)
    // or
    val listItemBinding = DataBindingUtil.inflate(layoutInflater, R.layout.list_item, viewGroup, false)
复制代码

```

在 xml 中 使用 @{cat.name} 就是把 Cat 的 name 绑定在 了这个 Textview 上

```
    <TextView
    android:text="@{cat.name}"
    android:textSize="20sp"
     />
复制代码

```

原理就是在 TextViewBindingAdapter 里面有个注解 上有这样的方法，最终调用的是这里，这里是通过注解设置的文本

![](https://user-gold-cdn.xitu.io/2020/6/8/17294268d8cff3e0?imageView2/0/w/1280/h/960/ignore-error/1)

### 3.2 data 中标签的含义

variable 里面就是一些需要绑定的 Model 变量

*   type 是要使用的 Model 类（全限定名字）, 直接打名字会自动提示出来的
*   name 就是在本 xml 对于这个 type 的一个使用名字
*   import 相当于咱们 android 中的导包，当一个 Model 有多个引用到这个 Xml 中时用这样的方式，比如下面 User，有两个地方在使用，所以直接导进来使用即可，此时的 type 直接就是类名字，不是全限定名了
*   默认 java.lang.* 包下的类会自动导入，所以此时 type 不用写全限定名字

```
<data>

    <variable
        
        type="com.nzy.mvvmsimple.databinding.model.Cat" />

    <import type="com.nzy.mvvmsimple.user.User" />

    <variable
        
        type="User" />

    <variable
        
        type="User" />
</data>

复制代码

```

*   alias 指定别名，也就是当用到的 Model 名字一样的时候，可以取一个别名

```
 <data>
        <import type="com.nzy.mvvmsimple.user.User" />
        <import
            alias="User2"
            type="com.nzy.mvvmsimple.databinding.model.User" />

        <variable
            
            type="User" />

        <variable
            
            type="User2" />
    </data>
复制代码

```

### 3.3 单向绑定

单向绑定是指数据源改变之后会立马通知 xml 进行赋值改变，刷新 UI。支持的有三种

1.  Obssservable 扩展的属性
2.  ViewModel + Obssservable 扩展的属性，可以不用使用 binding.lifecycleOwner = this 绑定生命周期的方法，即可实现单向绑定
3.  ViewModel + LiveData 必须使用 binding.lifecycleOwner = this 方法绑定生命周期 去 实现单向绑定

#### 3.1 Observable 扩展的属性

在创建实现 Observable 接口的类时要完成一些操作，但如果您的类只有少数几个属性，这样操作的意义不大。在这种情况下，您可以使用通用 Observable 类和以下特定于基元的类，将字段设为可观察字段，以及 ObservableField 属性

*   ObservableBoolean
*   ObservableByte
*   ObservableChar
*   ObservableShort
*   ObservableInt
*   ObservableLong
*   ObservableFloat
*   ObservableDouble
*   ObservableParcelable

#### 例子

```
class Cat {
    // 猫的名字用 ObservableField 包裹
    var name: ObservableField<String> = ObservableField<String>()
    // 是否显示猫的名字 用 ObservableBoolean
    var isShowName = ObservableBoolean()
}
复制代码

```

xml 中因为引用到 View.VISIBLE, 所以得导进来 View

```
 <variable
    
    type="com.nzy.mvvmsimple.databinding.model.Cat" />
 <import type="android.view.View" />   
 ...
   <TextView
            android:id="@+id/tv_name_ob"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@{cat.name}"
            android:textSize="20sp"
            android:visibility="@{cat.isShowName()?View.VISIBLE:View.INVISIBLE}"
            app:layout_constraintLeft_toLeftOf="parent"
            app:layout_constraintRight_toRightOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/tv_name_view_model" />

复制代码

```

绑定数据的 Activity，这样就可以实现单向绑定了，UI 会随着数据源的改变而改变

```
     var cat = Cat()
    cat.name.set("咖啡猫")
    cat.isShowName.set(true)
    binding.cat = cat
    // 点击事件
     bt_change.setOnClickListener {
        cat.name.set("ObservableField 改变的咖啡猫")
        cat.isShowName.set(!cat.isShowName.get())

    }
复制代码

```

#### 3.2 ViewModel + Obssservable 扩展的属性

其实这个相当于 Obssservable 扩展的属性 方式，只是 Cat 类 变更成 ViewModel 而已。

#### 3.3 ViewModel + LiveData

这个是用的最多的一个，也就是在咱们 android 开发中。按道理来说，xml 中避免引用过多的数据，一般引用一个 ViewModel 即可完成大部分的需求。 还是先看一下 xml

```
 <variable
    
    type="com.nzy.mvvmsimple.databinding.DataViewModel" />
    ...
<TextView
    android:id="@+id/tv_name_view_model"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="@{viewmodel.nameLiveData}"
    android:textSize="20sp"
    app:layout_constraintLeft_toLeftOf="parent"
    app:layout_constraintRight_toRightOf="parent"
    app:layout_constraintTop_toBottomOf="@+id/tv_name" />
复制代码

```

然后看看 ViewModel，就是一个用 LiveData 包裹起来

```
class DataViewModel :ViewModel(){
    var nameLiveData = MutableLiveData<String>()
}
复制代码

```

然后看一下 Activity 的代码

```
var binding: ActivityBinddataBinding =
            DataBindingUtil.setContentView<ActivityBinddataBinding>(
                this,
                R.layout.activity_binddata
            )
val viewModel: DataViewModel by viewModels()
 viewModel.nameLiveData.value = "小小猫"
 // 绑定ViewModel
 binding.viewmodel = viewModel
 // 必须绑定生命周期，否则无效果
 binding.lifecycleOwner = this
复制代码

```

### 3.4 双向绑定

双向绑定是 当 Model 改变的时候，UI 跟着改变，当 UI 改变的时候 Model 也会跟着改变，比如 EditText，CheckBox 的状态，如果自定义双向绑定的时候，要注意判断状态是否需要改变，切记不要变成递归改变，死循环了。

#### 举个例子

比如 xml 里面 Textview 和 EditText 用的是一个 Model 的 nameLiveData ，此时你会看出来，TextView 单向绑定，EditText 双向绑定，当输入内容的时候 TextView 也会改变

```
<variable
        
        type="com.nzy.mvvmsimple.databinding.DataViewModel" />
    <TextView
        android:id="@+id/tv_name_view_model"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="20dp"
        android:text="@{viewmodel.nameLiveData}"
        android:textSize="20sp"
 />

    <EditText
        android:id="@+id/et_name"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="20dp"
        android:text="@={viewmodel.nameLiveData}"
        android:textSize="20sp"
      />
复制代码

```

![](https://user-gold-cdn.xitu.io/2020/6/9/1729886ef19566cf?imageslim)

### android 官方支持双向绑定的控件

![](https://user-gold-cdn.xitu.io/2020/6/9/1729893ce38e5b12?imageView2/0/w/1280/h/960/ignore-error/1)

### 3.5 点击事件绑定

#### 3.5.1 直接传过来的 click 事件

```
<variable
    
    type="android.view.View.OnClickListener" />
    ...
 <Button
                android:onClick="@{click}"
                android:id="@+id/bt1"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_marginTop="20dp"
                android:layout_weight="1"
                android:text="绑定1" />
复制代码

```

#### 3.5.2. 通过 ViewModel(或者其他类) 的方法，不带参数

```
 <Button
    android:onClick="@{()->viewmodel.click()}"
    android:id="@+id/bt2"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    android:layout_marginTop="20dp"
    android:layout_weight="1"
    android:text="绑定2" />
复制代码

```

在 ViewModel 中

```
 fun click() {
        Toast.makeText(getApplication(), "绑定方式2", Toast.LENGTH_SHORT).show()
    }
复制代码

```

#### 3.5.3. 通过 ViewModel(或者其他类) 的方法，带 View 本身的参数

```
    <Button
        android:onClick="@{(view)->viewmodel.click(view)}"
        android:id="@+id/bt3"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginTop="20dp"
        android:layout_weight="1"
        android:text="绑定3" />

复制代码

```

ViewModel 中的方法

```
  fun click(view: View) {
        Toast.makeText(getApplication(), "绑定方式3", Toast.LENGTH_SHORT).show()
    }
复制代码

```

#### 3.5.4. 通过 ViewModel(或者其他类) 的方法，带其他的参数 不带 View 本身的参数

```
  <Button
    android:onClick="@{()->viewmodel.click(viewmodel.nameLiveData)}"
    android:id="@+id/bt4"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    android:layout_marginTop="20dp"
    android:layout_weight="1"
    android:text="绑定4" />
复制代码

```

ViewModel 里面

```
fun click(name:String){
    Toast.makeText(getApplication(), "绑定方式4$name", Toast.LENGTH_SHORT).show()
 }
复制代码

```

#### 3.5.5. 通过 ViewModel(或者其他类) 的方法，带其他的参数 且带 View 本身的参数

```
  <Button
    android:onClick="@{(view)->viewmodel.click(view,viewmodel.nameLiveData)}"
    android:id="@+id/bt5"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    android:layout_marginTop="20dp"
    android:layout_weight="1"
    android:text="绑定5" />
复制代码

```

ViewModel 里面, 方法的顺序不能变

```
fun click(view:View,name:String){
        Toast.makeText(getApplication(), "绑定方式5$name", Toast.LENGTH_SHORT).show()
    }
复制代码

```

#### 3.5.6. 直接调用某个类的方法，viewmodel::click，此时的 ViewModel 方法是必须带 View 本身的参数的

```
   <Button
    android:onClick="@{viewmodel::click}"
    android:id="@+id/bt6"
    android:layout_width="0dp"
    android:layout_height="wrap_content"
    android:layout_marginTop="20dp"
    android:layout_weight="1"
    android:text="绑定6" />
复制代码

```

ViewModel 里面，方法必须带 View 本身的参数

```
fun click(view: View) {
        Toast.makeText(getApplication(), "绑定方式3", Toast.LENGTH_SHORT).show()
    }
复制代码

```

#### 3.5.6. ObservableField<View.OnClickListener> 通过 ObservableField 包裹一个 OnClickListener

```
    <Button
        android:onClick="@{viewmodel.observableFieldClick}"
        android:id="@+id/bt7"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginTop="20dp"
        android:layout_weight="1"
        android:text="绑定7" />
复制代码

```

Actvity 里面设置给

```
viewModel.observableFieldClick.set(object :View.OnClickListener{
    override fun onClick(v: View?) {
        Toast.makeText(this@BindingActivity, "绑定方式7", Toast.LENGTH_SHORT).show()
    }
})
复制代码

```

### 3.6 xml 中尽量用简单的一些表达式

*   算术运算符 + - / * %
*   字符串连接运算符 +
*   逻辑运算符 && ||
*   二元运算符 & | ^
*   一元运算符 + - ! ~
*   移位运算符 >> >>> <<
*   比较运算符 == > < >= <=（请注意，< 需要转义为 <）
*   instanceof
*   分组运算符 ()
*   字面量运算符 - 字符、字符串、数字、null
*   类型转换
*   方法调用
*   字段访问
*   数组访问 []
*   三元运算符 ?: 比如

```
android:text="@{String.valueOf(index + 1)}"
android:visibility="@{age > 13 ? View.GONE : View.VISIBLE}"
复制代码

```

```
android:text="@{user.displayName ?? user.lastName}"
相当于
android:text="@{user.displayName != null ? user.displayName : user.lastName}"
复制代码

```

### 3.7 其他一些常见的引用

1.  使用 一些类中的常量 ，比如 View.VISEBLE 和 View.GONE

```
<data>
    <import type="android.view.View"/>
</data>
<TextView
    android:text="@{user.lastName}"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:visibility="@{viewmodel.isVisible ? View.VISIBLE : View.GONE}"/>
复制代码

```

2.  使用工具类的静态方法, 把 util 导入进来

```
 <import type="com.nzy.mvvmsimple.databinding.util.StringUtil" />
<Button
android:visibility="@{StringUtil.isEmpty(viewmodel.nameLiveData)?View.VISIBLE:View.GONE}"
android:layout_width="0dp"
android:layout_height="wrap_content"
android:layout_marginTop="20dp"
android:layout_weight="1"
android:text="button1" />
复制代码

```

3.  引用资源 比如 dimen , 同理 drawable 也是

```
android:padding="@{viewmodel.isBigPadding? @dimen/big_padding : @dimen/small_padding}"
或者是 
android:padding="@{viewmodel.padding}" // 这个设置的是px。
复制代码

```

```
android:src="@{viewmodel.isBigPadding? @drawable/hashiqi_one: @drawable/hashiqi_two}"
或者
android:src="@{viewmodel.drawable}"
复制代码

```

### 3.8 自定义绑定适配器

使用 kotlin 定义适配器 需要在 app 的 buid.gradle 中 加入 apply plugin: 'kotlin-kapt' 这个插件

#### BindingAdapter

绑定一些自定义逻辑，使用 BindingAdapter 注释的静态绑定适配器方法支持自定义特性 比如自定义一个加载网络的 ImageView，只有一个参数

```
@BindingAdapter("imageUrl")
fun setImageUrl(imageView: ImageView, url: String?) {
    if (url != null) {
        Glide.with(imageView).load(url).into(imageView)
    }
}
复制代码

```

比如定义一个有 error 的, 有展位图的图片 ImageView，requireAll==false 证明可以不用写全部

```
@BindingAdapter(value = ["imageUrl", "placeholder", "error"], requireAll = false)
fun setImageUrl(imageView: ImageView, url: String?, placeHolder: Drawable?, error: Drawable?) {
    if (url == null) {
        imageView.setImageDrawable(placeHolder);
    } else {
        Glide.with(imageView).load(url).placeholder(placeHolder).error(error).into(imageView)
    }
}
复制代码

```

#### BindingMethods 和 BindingConversion 基本上没用，暂时不要考虑用就行了

一般咱们自定义 bindadapter 的时候，xml 是没有提示的，咱们可以在 attrs.xml 里面写入咱们的 bind，这样就有提示了
-----------------------------------------------------------------------