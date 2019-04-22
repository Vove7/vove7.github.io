---
layout:     post
title:      "Okhttp网络封装"
subtitle:   "依赖于Kotlin的特性实现简洁的请求"
date:       2019-04-22 16:01
author:     "Vove"
header-img: "img/title-bg/post-bg-18-8-25.jpg"
tags:
    - kotlin
    - okhttp
---


> 还记得在进行网络请求时，先将返回内容转为 model。每一步都要判断空，每个请求。编写出的代码冗余非常高。
> 这里希望能利用kotlin的特性，完成代码的简化。
> 当然目前有众多优秀的开源库，例如：Retrofit，这里仅提供又一选择。

### 简单看几个例子

- 获取百度网页

```kotlin
NetHelper.get<String>("https://www.baidu.com/") {
    //请求成功
    success { _, s ->
        println(s)
    }
    //请求失败
    fail { _, e ->
        e.printStackTrace()
    }
} 
```

- 自动解析 json

```kotlin
@Test
fun getModel() {
    val l = CountDownLatch(1)
    
    //1.json文件内容：{ a : 10, b : "string",c : [1, 2, 3] }
    NetHelper.get<ResponseModelDemo>("http://127.0.0.1:4000/1.json") {
        success { _, model -> 
            //model 为 ResponseModelDemo 类型
            println(model)
        }
        fail { _, e ->
            e.printStackTrace()
        }
        end {
            l.countDown()
        }
    }
    l.await() //等待请求结束
    print("结束")
}
/**
 * 返回的model
 */
data class ResponseModelDemo(
        var a: Int? = null,
        var b: String? = null,
        var c: Array<Int>? = null
)

```
### 引入

见[VTP]([![](https://jitpack.io/v/Vove7/VTP.svg)](https://jitpack.io/#Vove7/VTP))

源码亦在此仓库

### 扩展

一般来说我们的 json 返回格式一般为：
```json
{
   code:0,
   msg:"success",
   data: {
       ...
   }
}

```

此时根据需求建立一个`ResponseMessage`

```kotlin
class ResponseMessage<T> {
    var code: Int = -1

    var message: String = "null"

    var err: String? = null
    var data: T? = null

    fun isOk(): Boolean {
        return code == 0
    }
}
```

另外封装自己的 NetHelper

```kotlin

object MyNetHelper {
     inline fun <reified T> postJson(
            url: String, model: Any? = null, requestCode: Int = -1, arg1: String? = null,
            crossinline callback: WrappedRequestCallback<ResponseMessage<T>>.() -> Unit
     ): Call {

        return NetHelper.postJson(url, RequestModel(model, arg1), requestCode, callback)
    }

    inline fun <reified T> get(
            url: String, params: Map<String, String>? = null, requestCode: Int = 0,
            callback: WrappedRequestCallback<ResponseMessage<T>>.() -> Unit
    ): Call {
        return NetHelper.get(url, params, requestCode, callback)
    }

    //.....
}
```

使用：
```kotlin

//2.json 内容：{ code:0, msg:"success", data: { a : 10, b : "string",c : [1, 2, 3] } }
MyNetHelper.get<ResponseModelDemo>("http://127.0.0.1:4000/2.json") {
    success { _, responseModel ->
        //检查请求结果
        if (responseModel.isOk()) {
            //data 为 ResponseModelDemo类型
            println(responseModel.data) 
        } else {
            print(responseModel.message)
        }
    }
    fail { _, e ->
        e.printStackTrace()
    }
}

```

### 使用的 Kotlin 特性

可根据 kotlin + 以下关键词 搜索相关内容

- inline 函数

- reified 

- 扩展函数 可参考：[Kotlin扩展函数和扩展属性笔记](https://www.jianshu.com/p/7291c9a1ec1e)