> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6973874488345624584)

> 有很多流行的三方库比如 GSON，Moshi 来进行序列化和反序列化对象。我的工程主要过去主要用的是 GSON。但在 Kotlin 里使用 GSON 的时候发现了很多限制。

> Kotlin JSON Serialization using kotlinx.serialization  
> 作者：mahbaleshwar hegde(maabu)

[原文链接](https://mahabaleshwarhnr.medium.com/json-serialization-using-kotlin-native-serialization-c8395bb46762)

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7581db9c665b412b84f9448e5b871c82~tplv-k3u1fbpfcp-watermark.image)

在过去的一年半时间我一直从事 Android 应用开发。我需要用到一些解析库对 JSON 对象进行编解码。有很多流行的三方库比如 GSON，Moshi 来进行序列化和反序列化对象。我的工程主要过去主要用的是 GSON。但在 Kotlin 里使用 GSON 的时候发现了很多限制。

让我们看一下下面的示例

```
import com.google.gson.Gson

data class User(val firstName: String,
                val lastName: String = "Appleased") {

    val fullName: String
        get() = "$firstName $lastName"
}

fun main() {
    val json = """
        {
           "firstName": "John"
        }
    """.trimIndent()
    val user = Gson().fromJson(json, User::class.java)
    println(user.fullName)
    print(user.lastName.isBlank())
}
Output:  John null.
         App crashes.
复制代码

```

这里失去了 Kotlin 的两个主要特性：

1.  类型安全 - lastname 属性是非空的，但是 GSON 仍能将一个 null 付给它创建了一个 user 对象。
2.  参数默认值没有效果。

你可以在网上找到相同类型的其他问题以及一些解决办法。于是 Kotlin 团队想到了要开发一个 native 支持的库 [kotlinx.serialization](https://github.com/Kotlin/kotlinx.serialization)。这个库支持 JVM，JavaScript，Native 所有平台，同时也支持多种格式的序列化——JSON，CBOR，protocol buffers 等等。

### 开始使用 kotlinx.serialization

前面已经提到过 kotlinx.serialization 包含多种类型的序列化格式，比如 JSON，Protocol buffers，CBOR，Properties，HOCON。本文只会涉及到 JSON 序列化的一些基础内容。

#### 安装

```
plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'org.jetbrains.kotlin.plugin.serialization' version '1.4.30'   
    id 'org.jetbrains.kotlin.jvm' version '1.4.30'
}
dependencies {
  implementation "org.jetbrains.kotlinx:kotlinx-serialization- json:1.1.0"
}
复制代码

```

#### JSON 序列化

首先，通过添加 @Serializable 注解的形式给一个类进行序列化。

```
@Serializable
data class Project(val name: String, val language: String)

fun main() {
    val project = Project("kotlinx.serialization", "Kotlin")
    val jsonString = Json.encodeToString(project)
    print(jsonString) 
}
Output: {"name":"kotlinx.serialization","language":"Kotlin"}
复制代码

```

#### JSON 反序列化

```
@Serializable
data class Project(val name: String, val language: String)

fun main() {
    val jsonString = """
        {"name":"kotlinx.serialization","language":"Kotlin"}
    """.trimIndent()
    val project: Project = Json.decodeFromString(jsonString)
    print(project)
}
Output: Project(name=kotlinx.serialization, language=Kotlin)
复制代码

```

接着我们再看下 kotlinx.serialization 的几个重要的特性。

#### 1. 强制类型安全

kotlinx.serialization 的 API 确保了类型安全。如果构造函数的参数类型是非空的话，对于值是 null 的参数无法正常创建对象。

```
@Serializable
data class Project(val name: String, val language: String)

fun main() {
    val jsonString = """
        {"name":"kotlinx.serialization", "language": null}
    """.trimIndent()
    val project: Project = Json.decodeFromString(jsonString)
    print(project)
}
Output: Exception in thread "main" kotlinx.serialization.json.internal.JsonDecodingException: Unexpected JSON token at offset 45: Expected string literal but 'null' literal was found.
复制代码

```

#### 2. 在解析 JSON 的时候支持默认值

kotlinx.serialization 的 API 支持在解析 JSON 设置默认值。用 Json 的构造器 PAI 将`coreceInputValues`设置为 true 即可。

```
@Serializable
data class Project(val name: String, val language: String = "kotlin")

fun main() {
   val json = Json {
        coerceInputValues = true
    }
    val jsonString = """
        {"name":"kotlinx.serialization", "language": null}
    """.trimIndent()
    val project: Project = json.decodeFromString(jsonString)
    print(project)
}
 Output: Project(name=kotlinx.serialization, language=kotlin)
复制代码

```

#### 3. 泛型类

kotlinx.serialization 的 API 在序列化和反序列化泛型类型的时候非常简单也非常高效。 序列化：

```
@Serializable
data class Version(val major: String, 
                   val minor: String, 
                   val patch: String)
@Serializable
data class Number<T>(val value: T)
@Serializable
data class Data(val intNumber: Number<Int>, 
                val longNumber: Number<Long>, 
                val versionNumber: Number<Version>)

fun main() {
    val data = Data(Number(10),
            Number(10L),
            Number(Version("1", "0", "0")))
    val encodedString = Json.encodeToString(data)
    print(encodedString)
}
Output: {"intNumber":{"value":10},"longNumber":{"value":10},"versionNumber":{"value":{"major":"1","minor":"0","patch":"0"}}}
复制代码

```

反序列化：

```
@Serializable
data class Project(val name: String, val language: String)
fun main() {

    val jsonString = """
        [
          {
            "name": "kotlinx.serialization",
            "language": "kotlin"
           }, 
          {
            "name": "coroutines",
            "language": "kotlin"
          }
        ]
    """.trimIndent()
    val projects: List<Project> = Json.decodeFromString(jsonString)
    print(projects)
}
Output: [Project(name=kotlinx.serialization, language=kotlin), Project(name=coroutines, language=kotlin)]
复制代码

```

我们没有像 GSON 或者其他 Java 基础库那样用任何匿名的`TypeToken`对象去获取泛型的类型来转换对象。因此在 kotlinx.serialization 里面不需要任何匿名的`TypeToken`对象来反序列化泛型类型。

#### 4. 序列化字段名

经常遇到 json 的 key 和字段名不一致的问题。我们可以通过添加 @SerialName("json_key") 给字段进行序列化。

```
@Serializable
data class Project(val name: String,
                  @SerialName("lang) val language: String)
复制代码

```

#### 5. 引用对象

kotlinx.serialization 支持嵌套对象的序列化。它指挥序列化哪些添加了`@Serializable`的字段或者类，否则编译器会报错：

```
@Serializable
data class Project(val name: String,
                  val language: String,
                   val project: Version
)
@Serializable 
data class Version(val major: Int,
                   val minor: Int,
                   val patch: Int)
Output: {"name":"Jetpack","language":"Kotlin","project":{"major":1,"minor":0,"patch":0}}
复制代码

```

#### 6. 数据校验

可以在 json 反序列化的时候对数据进行校验。

```
@Serializable
data class LoginResponse(val accessToken: String) {
    init {
        require(accessToken.isNotEmpty()) { "Access token cannot be empty" }
    }
}

fun main() {

    val jsonString = """
         { "accessToken": "" }
    """.trimIndent()
    val loginResponse: LoginResponse = Json.decodeFromString(jsonString)
}
Output: Exception in thread "main" java.lang.IllegalArgumentException: Access token cannot be empty
复制代码

```

### 支持 Retrofit

具体可以看下针对 Retrofit 2 Converter.Factory 的 Kotlin 序列化的库。

### 最后思考

kotlinx.serialization 有很多优秀的特性。本文只介绍了最基本的。你可以通过[官方文档](https://github.com/Kotlin/kotlinx.serialization)探索更多的特性。

Happy Coding！！！