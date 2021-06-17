> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6913512989916135432)

> 在相当长的一段时间里，kotlin 一直都没有自己专属的序列化 / 反序列化库。

在相当长的一段时间里，kotlin 一直都没有自己专属的序列化 / 反序列化库。于是只能拿 Java 的库来将就一下，最常用的大概就是 Gson 了。但是这样一来 Kt 的很多强大特性就用不了，比如参数默认值，属性委托等，就这样被迫退化为 Javaer 了（没错，在下正是 kotlin 吹，Java 叛徒）。 虽然社区也维护了支持 Kt 特性的第三方序列化库，比如 moshi，but 并不好用，Gson 用习惯了就喜欢这种简洁直白的女孩子（bushi）。想了解 Moshi 的自己去查吧，个人认为官方库出来后 Moshi 离完蛋不远了。

Gson 在开始介绍今天的主角之前，先来回顾一下 Gson 在 kt 中的用法，与 Java 没啥区别：

```
data class Student(val name: String, val score: Int = 80)

fun main(){
    val gosn = Gson()
    val Icarus = Student("Icarus", 99)
    println(gson.toJson(Icarus))
    val Icarus2 = gson.fromJson("""{"name":"Icarus","score":99}""", Student::class.java)
    println(Icarus2)
    println(Icarus == Icarus2)
    
    
    val SoharaMitsuki = Student("SoharaMitsuki")
    println(gson.toJson(SoharaMitsuki ))
    val SoharaMitsuki2 = gson.fromJson（"""{"name":"SoharaMitsuki"}""", Student::class.java）
    println(SoharaMitsuki2)
}
复制代码

```

注意到，我们定义的 score 属性有默认值，我就直接说结论了，使用默认值生成的对象序列化成 Json 字符串一切正常，但是 Gson 使用未给出属性值的 Json 字符串反序列化为 Student 对象，score 属性值是 0，不会用到默认值的，因为 Gson 会先去类定义里面找对应的构造函数，就是参数列表不带这个属性的构造函数，没找到就会用到 Java 黑魔法 Unsafe 类，直接创建对象。 data class Student WithInits(val name: String, val score: Int){

```
val firstName by lazy {
    name.split(" ")[0]
}

/**
*用by lazy跟随的属性没有幕后字段，初始化时不会再内存中给它开一个存储值的空间
*初次使用该属性时才会lazy后面的代码，把引用指向代码返回的那块内存
*专业名称是延迟初始化
*/
val lastName by lazy {
    name.split(" ")[1]
}
复制代码

```

}

正是因为 by lazy 跟随的属性可以在运行时算出来，所以序列化的时候他们会被忽略从而减小 Json 长度。因为延迟初始化属性在对象生成的时候只是一个空引用，Gson 从 json 字符串取回的对象相应属性也是 null，Gson 把 KClass 当作 JavaClass 对待，延迟执行的代码信息也丢了。 如果你一定要既用 Gson 又要延迟初始化， 百度搜索 “@Poko“了解详情。

主角是 kotlinx.serialization 首先配置 Gradle：

```
//build.gradle
plugins {
    id 'org.jetbrains.kotlin.jvm' version '1.4.20'
    id 'org.jetbrains.kotlin.plugin.serialization' version '1.4.20'
}

repositories {
    // Artifacts are also available on Maven Central
    jcenter()
}

dependencies {
    implementation "org.jetbrains.kotlinx:kotlinx-serialization-json:1.0.1"
}
复制代码

```

对于 JSON，我们使用 Json.encodeToString 扩展功能对数据进行编码。它将可序列化的对象作为其参数在后台进行序列化，并将其编码为 JSON 字符串。 让我们从描述项目的类开始，并尝试获取其 JSON 表示形式。

```
@Serializable 
data class Project(val name: String, val language: String)

fun main() {
    val data = Project("kotlinx.serialization", "Kotlin")
    println(Json.encodeToString(data))
    //打印{"name":"kotlinx.serialization","language":"Kotlin"}
    
    val data = Json.decodeFromString<Project>("""
        {"name":"kotlinx.serialization","language":"Kotlin"}    
    """)
    println(data)//Project(name=kotlinx.serialization, language=Kotlin)
}
复制代码

```

再次提醒，并不是只有数据类才能序列化，只是为了反序列化时能把类直接打印出来。 敲黑板，@Serializable，可以加参数指定我们自定义的序列化器，无参数时使用系统给的 Serializer。 幕后字段序列化 仅对有后备字段的类的属性进行序列化，因此具有 getter / setter 但却没有幕后字段的属性不会被序列化，被委托的属性也不会被序列化。

```
@Serializable 
class Project(
    // name is a property with backing field -- serialized
    var name: String
) {
    var stars: Int = 0 // property with a backing field -- serializedval 
    path: String // getter only, no backing field -- not serialized
        get() = "kotlin/$name"
    var id by ::name // delegated property -- not serialized
}

fun main() {
    val data = Project("kotlinx.serialization").apply { stars = 9000 }
    println(Json.encodeToString(data))
    //{"name":"kotlinx.serialization","stars":9000}
}
复制代码

```

如果我们想定义 Project 类，使其采用路径字符串，然后将其解构为相应的属性，则我们可能很想编写类似以下代码的内容:

```
@Serializable 
class Project(path: String) {
    val owner: String = path.substringBefore('/')    
    val name: String = path.substringAfter('/')    
}
复制代码

```

此类无法编译，因为 @Serializable 注解要求该类的主构造函数的所有参数均为属性。一个简单的解决方法是使用类的属性定义一个私有的主构造函数，然后将所需的构造函数转换为辅助构造函数。

```
@Serializable 
class Project private constructor(val owner: String, val name: String) {
    constructor(path: String) : this(
        owner = path.substringBefore('/'),    
        name = path.substringAfter('/')
    )                        

    val path: String
        get() = "$owner/$name"  
}

fun main() {
    println(Json.encodeToString(Project("kotlin/kotlinx.serialization")))
    //{"owner":"kotlin","name":"kotlinx.serialization"}
}
复制代码

```

path 不具有幕后字段，不会被序列化。 数据验证 另一种情况是你可能想引入不带属性的主构造函数参数，在将其值存储到属性之前对其进行验证。为了使其可序列化，应该在主构造函数中将其替换为属性，然后将验证移至 init {...} 块中：

```
@Serializable
class Project(val name: String) {
    init {
        require(name.isNotEmpty()) { "name cannot be empty" }
    }
}

fun main() {
    val data = Json.decodeFromString<Project>("""
        {"name":""}
    """)//Exception in thread "main" java.lang.IllegalArgumentException: name cannot be empty
    println(data)
}
复制代码

```

### 默认值

默认值属性反序列化时会被自动填充，序列化时不会被写入 json，目的还是节省空间和带宽，在大多数实际场景中，此类配置可以减少视觉混乱，并节省要序列化的数据量。

```
0@Serializable 
data class Project(val name: String, val language: String = "Kotlin")

fun main() {
    val data = Json.decodeFromString<Project>("""
        {"name":"kotlinx.serialization"}
    """)
    println(data)//Project(name=kotlinx.serialization, language=Kotlin)
    
    val data1 = Project("kotlinx.serialization")
    println(Json.encodeToString(data1))//{"name":"kotlinx.serialization"}
}
复制代码

```

另一种类似情况是可空属性默认值为 null

```
@Serializable
class Project(val name: String, val renamedTo: String? = null)

fun main() {
    val data = Project("kotlinx.serialization")
    println(Json.encodeToString(data))//{"name":"kotlinx.serialization"}
}
复制代码

```

当输入中存在可选属性时，该属性的相应初始化器不会调用。此功能是为提高性能而设计的，因此请注意不要依赖初始化程序中的副作用。

```
fun computeLanguage(): String {
    println("Computing")
    return "Kotlin"
}

@Serializable 
data class Project(val name: String, val language: String = computeLanguage())
 
fun main() {
    val data = Json.decodeFromString<Project>("""
        {"name":"kotlinx.serialization","language":"Kotlin"}
    """)
    println(data)//Project(name=kotlinx.serialization, language=Kotlin)
}
复制代码

```

由于在输入中指定了 language 属性，因此在输出中看不到 “计算” 字符串。

@Required 修饰的属性其值必须显式指明。

```
@Serializable 
data class Project(val name: String, @Required val language: String = "Kotlin")

fun main() {
    val data = Json.decodeFromString<Project>("""    
        {"name":"kotlinx.serialization"}    
    """)
    println(data)//Exception in thread "main" kotlinx.serialization.MissingFieldException: Field 'language' is required, but it was missing
}
复制代码

```

@Transient 修饰的属性不会被序列化，反序列化时也不能被指定。

```
@Serializable 
data class Project(val name: String, @Transient val language: String = "Kotlin")

fun main() {
    val data = Json.decodeFromString<Project>("""        
        {"name":"kotlinx.serialization","language":"Kotlin"}    
    """)/**
    *Exception in thread "main" kotlinx.serialization.json.internal.JsonDecodingException:
    *Unexpected JSON token at offset 60: Encountered an unknown key 'language'.
    *Use 'ignoreUnknownKeys = true' in 'Json {}' builder to ignore unknown keys.
    */
    println(data)
}
复制代码

```

Kt 的序列化框架是严格支持 Kt 的类型系统的，所以下面的代码有异常：

```
@Serializable 
data class Project(val name: String, val language: String = "Kotlin")

fun main() {
    val data = Json.decodeFromString<Project>("""
        {"name":"kotlinx.serialization","language":null}
    """)//Exception in thread "main" kotlinx.serialization.json.internal.JsonDecodingException: Unexpected JSON token at offset 52: Expected string literal but 'null' literal was found.
//Use 'coerceInputValues = true' in 'Json {}` builder to coerce nulls to default values.
    println(data)
}
复制代码

```

### 套娃序列化

可序列化的类可以在其可序列化属性中引用其他类。被引用的类也必须标记为 @Serializable

```
@Serializable
class Project(val name: String, val owner: User)

@Serializable
class User(val name: String)

fun main() {
    val owner = User("kotlin")
    val data = Project("kotlinx.serialization", owner)
    println(Json.encodeToString(data))//{"name":"kotlinx.serialization","owner":{"name":"kotlin"}}
}
复制代码

```

### 不压缩重复引用

```
@Serializable
class Project(val name: String, val owner: User, val maintainer: User)

@Serializable
class User(val name: String)

fun main() {
    val owner = User("kotlin")
    val data = Project("kotlinx.serialization", owner, owner)
    println(Json.encodeToString(data))
    //{"name":"kotlinx.serialization","owner":{"name":"kotlin"},"maintainer":{"name":"kotlin"}}
}
复制代码

```

Kotlin 序列化设计用于纯数据的编码和解码。它不支持使用重复的对象引用重建任意对象图。尝试序列化两次引用同一对象的实例, 就写入字符串两次。所以不要出现实例的环状引用，那就爆栈了。

### 泛型类

Kotlin 中的泛型类提供类型多态行为，由 Kotlin 序列化在编译时强制执行。例如，考虑一个泛型可序列化类 Box 。 我们在 JSON 中获得的实际类型取决于为 Box 指定的实际编译时类型参数。

```
@Serializable
class Box<T>(val contents: T)

@Serializable
class Data(
    val a: Box<Int>,
    val b: Box<Project>
)

fun main() {
    val data = Data(Box(42), Box(Project("kotlinx.serialization", "Kotlin")))
    println(Json.encodeToString(data))
    //{"a":{"contents":42},"b":{"contents":{"name":"kotlinx.serialization","language":"Kotlin"}}}
}
复制代码

```

### 键别名

默认情况下，在编码表示形式中使用的属性名称（在我们的示例中为 JSON）与源代码中的名称相同。用于序列化的名称称为序列名称，可以使用 @SerialName 进行更改。例如，我们可以在源代码中使用语言属性，并使用缩写的序列名。

```
@Serializable
class Project(val name: String, @SerialName("lang") val language: String)

fun main() {
    val data = Project("kotlinx.serialization", "Kotlin")
    println(Json.encodeToString(data))
    //{"name":"kotlinx.serialization","lang":"Kotlin"}
}
复制代码

```