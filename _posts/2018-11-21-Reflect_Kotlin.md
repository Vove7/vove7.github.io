---
layout:     post
title:      "反射调用Kotlin类里的Companion函数"
subtitle:   "支持重载、检查可空类型"
date:       2018-11-20 16:00
author:     "Vove"
header-img: "img/title-bg/post-bg-18-8-25.jpg"
tags:
    - kotlin
    - reflection
---
> 支持重载、检查可空类型

此时有个类`C`
```kotlin
class C {
    companion object {
        fun a() {
            println("a")
        }
        fun b(s: String?) {
            println("b string $s")
        }
        fun b(a: Int) {
            println("b int $a")
        }
        fun c(): String {
            return "c 12345"
        }
        fun e(a: Int, b: Int, c: String?): String {
            return "$c $a + $b = ${a + b}"
        }
    }
}
```
现在反射C类中的`companion`函数
```kotlin
@Test
fun kotlinReflectTest() {
    val cls = Class.forName("cn.vassistant.repluginapi.C")
    KotlinReflectHelper.invokeCompanionMethod(cls, "a")//无参数函数
    KotlinReflectHelper.invokeCompanionMethod(cls, "b", "sss")//可空参数函数
    KotlinReflectHelper.invokeCompanionMethod(cls, "b", null)//可空参数函数
    KotlinReflectHelper.invokeCompanionMethod(cls, "b", 1)//重载函数
    val s = KotlinReflectHelper.invokeCompanionMethod(cls, "c")//返回值函数
    println(s)

    var r = KotlinReflectHelper.invokeCompanionMethod(cls, "e", 1, 2, "求和")//多参数函数
    println(r)
    r = KotlinReflectHelper.invokeCompanionMethod(cls, "e", 1, 2, null)//多参数函数
    println(r)
}
```
输出结果：
```
a
b string sss
b string null
b int 1
c 12345
求和 1 + 2 = 3
null 1 + 2 = 3
```

其中`KotlinReflectHelper`类：

```kotlin
object KotlinReflectHelper {
    /**
     * 反射调用Kotlin类里的Companion函数
     * 支持函数重载调用
     * 注：参数位置严格匹配
     *
     * Companion函数的第一个参数为[Companion]
     * @param cls Class<*> javaClass
     * @param name String 函数名
     * @param args Array<out Any?> 参数
     * @return Any? 返回值
     */
    @Throws
    fun invokeCompanionMethod(cls: Class<*>, name: String, vararg args: Any?): Any? {
        val k = Reflection.createKotlinClass(cls)
        k.companionObject?.declaredFunctions?.forEach eachFun@{ f ->
            val ps = f.parameters
            if (f.name == name && ps.size == args.size + 1) {//函数名相同, 参数数量相同
                //匹配函数参数
                ps.subList(1, ps.size).withIndex().forEach {
                    val paramType = it.value.type
                    val userParam = args[it.index]
                    val canNullAndNull = paramType.isMarkedNullable && userParam == null  //isMarkedNullable参数类型是否可空
                    if (!canNullAndNull) {
                        val typeMatch = when {//判断类型匹配
                            paramType.isMarkedNullable -> //参数可空 userParam不空 转不可空类型比较
                                paramType.withNullability(false).isSubtypeOf(userParam!!::class.defaultType)

                            userParam != null -> // 参数不可空 且用户参数不空
                                paramType.isSubtypeOf(userParam::class.defaultType)//类型匹配
                            else -> return@eachFun//参数不可空 用户参数空 不符合
                        }
                        if (!typeMatch) return@eachFun //匹配下一个函数
                    }
                    //else 参数可空 给定参数空 过
                }
                val argsWithInstance = arrayOf(k.companionObjectInstance, *args)
                return f.call(*argsWithInstance)
            }
        }
        throw Exception("no companion method named : $name")
    }

}
```

1. 添加依赖`implementation "org.jetbrains.kotlin:kotlin-reflect:$kotlin_version"
`或下载`kotlin-reflect.jar`包


#### 另一种方式

指定参数类型

```kotlin

    /**
     * 
     * @param cls Class<*> 类 可通过Class.forName(..) 获取
     * @param name String 函数名
     * @param argPairs Array<out Pair<Class<*>, Any?>> first: 值类型, second: 值
     * @return Any? 函数返回值
     */
    @Throws
    fun invokeCompanion(cls: Class<*>, name: String, vararg argPairs: Pair<Class<*>, Any?>): Any? {
//        println("调用函数：  ----> $name(${Arrays.toString(argPairs)})")
        val types = arrayOfNulls<KClass<*>>(argPairs.size)
        val args = arrayOfNulls<Any>(argPairs.size)
        var i = 0
        argPairs.forEach {
            types[i] = Reflection.createKotlinClass(it.first)
            args[i++] = it.second
        }

        val k = Reflection.createKotlinClass(cls)
        k.companionObject?.declaredFunctions?.forEach eachFun@{ f ->
//            println("invokeCompanionMethod 匹配函数 ----> ${f.name}()")
            val ps = f.parameters
            if (f.name == name && ps.size == argPairs.size + 1) {//函数名相同, 参数数量相同
                //匹配函数参数
                ps.subList(1, ps.size).withIndex().forEach {
                    val paramType = it.value.type
                    val userType = types[it.index]!!.defaultType
                    val typeMatch = paramType.withNullability(false).isSubtypeOf(userType)
                    if (!typeMatch) {
//                        println("invokeCompanionMethod  ----> 参数类型不匹配")
                        return@eachFun //匹配下一个函数
                    }
                    //else 参数可空 给定参数空 过
                }
                val argsWithInstance = arrayOf(k.companionObjectInstance, *args)
                return f.call(*argsWithInstance)
            } else {
//                println("invokeCompanionMethod  ----> 名称or参数数量不同")
            }
        }
        throw Exception("no companion method named : $name")
    }
```

测试：
```kotlin

@Test
fun kotlinReflectTest2() {
    val cls = Class.forName("cn.vassistant.repluginapi.C")
    KotlinReflectHelper.invokeCompanion(cls, "a")//无参数函数
    KotlinReflectHelper.invokeCompanion(cls, "b", Pair(String::class.java, "sss"))//可空参数函数
    KotlinReflectHelper.invokeCompanion(cls, "b", Pair(Int::class.java, null))//可空参数函数
    KotlinReflectHelper.invokeCompanion(cls, "b", Pair(Int::class.java, 1))//重载函数
    val s = KotlinReflectHelper.invokeCompanion(cls, "c")//返回值函数
    println(s)

    var r = KotlinReflectHelper.invokeCompanion(cls, "e", Pair(Int::class.java, 1),
            Pair(Int::class.java, 2), Pair(String::class.java, "求和"))//多参数函数
    println(r)
    r = KotlinReflectHelper.invokeCompanion(cls, "e", Pair(Int::class.java, 1),
            Pair(Int::class.java, 2), Pair(String::class.java, null))//多参数函数
    println(r)
}

```
