---
layout:     post
title:      "反射调用Kotlin类里的Companion函数"
subtitle:   ""
date:       2018-11-20 16:00
author:     "Hux"
header-img: "img/title-bg/post-bg-18-8-25.jpg"
tags:
    - kotlin
    - reflection
---


此时有个类`C`
```kotlin
class C {
    companion object {
        fun a() {
            println("a")
        }
        fun b(s: String) {
            println("b $s")
        }
        fun c(): String {
            return "12345"
        }
    }
}
```
现在反射C类中的`companion`函数
```kotlin
fun kotlinReflectTest() {
    val cls = Class.forName("cn.vassistant.repluginapi.C")
    KotlinReflectHelper.invokeCompanionMethod(cls, "a")
    KotlinReflectHelper.invokeCompanionMethod(cls, "b", "sss")
    val s = KotlinReflectHelper.invokeCompanionMethod(cls, "c")
    println(s)
}
```
输出：
```
a
b sss
12345
```

其中`KotlinReflectHelper`类：
```kotlin
object KotlinReflectHelper {
    /**
     * 反射调用Kotlin类里的Companion函数
     * @param cls Class<*> javaClass
     * @param name String 函数名
     * @param args Array<out Any?> 参数
     * @return Any? 返回值
     */
    @Throws
    fun invokeCompanionMethod(cls: Class<*>, name: String, vararg args: Any?): Any? {
        val k = Reflection.createKotlinClass(cls)
        k.companionObject?.declaredFunctions?.forEach {
            if (it.name == name) {
                val argsWithInstance = arrayOf(k.companionObjectInstance, *args)
                return it.call(*argsWithInstance)
            }
        }
        throw Exception("no companion method named : $name")
    }

}

```

#### 注意

1. 添加依赖`implementation "org.jetbrains.kotlin:kotlin-reflect:$kotlin_version"
`或下载`kotlin-reflect.jar`包
