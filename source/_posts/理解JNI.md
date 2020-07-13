---
title: 理解 JNI
date: 2019-04-09 16:28:01
tags: [JNI,AIDL,深入理解 Android 读书笔记]
---

_2020.06.15 更新_

### JNI 概述

JNI(Java Native Interface),意为 **Java 本地调用**,是连接 Java 和 native 的桥梁。

JNI 推出的原因：

1. Java 的平台无关性不能迁移到虚拟机上, Java 虚拟机是使用 native 编写的，虚拟机运行在具体的平台上(Linux、Windows等),由于平台的特性，所以虚拟机无法实现平台无关性。Java 使用 JNI 技术可以作为桥梁，可以实现 Java 调用虚拟机的 native 层，实现了Java 的平台无关性。
2. 执行效率和速度。

### JNI 之 Java 层操作

Java 层主要有两个关键：

1. 加载 native 动态库
2. 声明 Java 的 native 方法

此方式为动态加载注册方式，即在运行时加载 jni 库
```
public class MediaScanner implements AutoCloseable {
    static {
        System.loadLibrary("media_jni");// 加载 so 库
        native_init();//调用 native 方法
    }
    ...
    // 在 Java 中声明 native 方法
    private static native final void native_init();
    private native final void native_setup();
    private native final void native_finalize();

    ...

}    

```
<!-- more -->

### JNI 之 native 层操作 (一)

实例代码：
MediaScanner.cpp 代码片段
```
// MediaScanner.java 的 native 的 JNI 实现
static void
android_media_MediaScanner_native_init(JNIEnv *env)
{
    ALOGV("native_init");
    jclass clazz = env->FindClass(kClassMediaScanner);
    if (clazz == NULL) {
        return;
    }

    fields.context = env->GetFieldID(clazz, "mNativeContext", "J");
    if (fields.context == NULL) {
        return;
    }
}

static void
android_media_MediaScanner_native_setup(JNIEnv *env, jobject thiz)
{
    ALOGV("native_setup");
    MediaScanner *mp = new StagefrightMediaScanner;

    if (mp == NULL) {
        jniThrowException(env, kRunTimeException, "Out of memory");
        return;
    }

    env->SetLongField(thiz, fields.context, (jlong)mp);
}
```

### JNI 之 native 层操作 (二) -- 注册 JNI 函数

如何知道 Java 层的 MediaScanner 中的 `native_init` 函数对应 JNI 层的 `android_media_MediaScanner_native_init` 函数,这时就需要 JNI 注册，将两个层面的函数关联起来。注册后，有了这层关联关系，Java 层调取 JNI 层函数就可以实现。

#### 静态注册

静态注册实现方法参见 [Android Studio 配置 javah 生成 C/C++ 头文件，完成 JNI 调用](https://blog.csdn.net/Strange_Monkey/article/details/84028290) 中相关内容，需要使用 Java 工具 javah 。

当 Java 层调用 `native_init` 函数时，就会去 JNI 库中寻找 `android_media_MediaScanner_native_init` 函数，如果没有，就会报错，如果存在该函数，就会建立关联，**此关联其实就是保存的 JNI 层函数的函数指针**。以后 Java 层再调用 `native_init` 方法时 ，直接 **调用该函数指针就可以了**，**这部分的工作是在 Java 虚拟机中完成的。**

**Java native 方法是通过 `函数指针` 来与 _JNI 层的函数_ 建立联系的。**

#### 动态注册

JNI 的静态注册步骤繁琐，需要配合 javap 工具、生成相应的 .h 文件等操作，由静态注册知：Java native 函数是通过函数指针来和 JNI 层函数建立关联关系的。如果直接让 native 函数知道 JNI 层对应函数的函数指针，是不是很方便，这就是 动态注册。


在静态注册中可知，Java 层和 JNI 层的函数是一一对应的，那么可以 **使用结构体来保存这种关联关系**。同时 JNI 中可以使用 `JNINativeMethod` 这种结构体来实现以上功能，这就是动态注册方法。

关于 JNINativeMethod 的定义：

```
typedef struct{
    // Java 中 native 函数的名字，不用带包路径
    const char* name;
    // Java 层函数的签名信息
    const char* signature;
    // JNI 层对应函数的函数指针，其为 void* 类型
    void* fnPtr:
} JNINativeMethod
```

MediaScanner.cpp  中的具体使用：

```
static const JNINativeMethod gMethods[] = {
    ...
    ...

    {
        "native_init",// Java 层方法名
        "()V",// Java 层方法签名信息
        (void *)android_media_MediaScanner_native_init// JNI 层对应的函数指针
    },

    {
        "native_setup",
        "()V",
        (void *)android_media_MediaScanner_native_setup
    },

    ...
    ...

};
```

AndroidRuntime.cpp 类中提供了 `registerNativeMethod` 来完成注册工作：

```
/*
 * Register native methods using JNI.
 */
/*static*/ int AndroidRuntime::registerNativeMethods(JNIEnv* env,
    const char* className, const JNINativeMethod* gMethods, int numMethods)// 将上文中的 jMethods 传入
{
    return jniRegisterNativeMethods(env, className, gMethods, numMethods);
}
```

`jniRegisterNativeMethods` 为 JNIHelper 中提供的方法：

```
extern "C" int jniRegisterNativeMethods(C_JNIEnv* env, const char* className,
    const JNINativeMethod* gMethods, int numMethods)
{
    JNIEnv* e = reinterpret_cast<JNIEnv*>(env);

    ALOGV("Registering %s's %d native methods...", className, numMethods);

    scoped_local_ref<jclass> c(env, findClass(env, className));
    ...
    // 真正执行注册的函数
    if ((*env)->RegisterNatives(e, c.get(), gMethods, numMethods) < 0) {
        ....
    }

    return 0;
}

```

重要的工作只要两步：

```

// 找到对应的类
scoped_local_ref<jclass> c(env, findClass(env, className));

// 这句话其实是调用 JINEnv 的 RegisterNatives方法，将 JNI 类中结构体注册进来，从而完成注册关系
(*env)->RegisterNatives(e, c.get(), gMethods, numMethods)

```

_注册的函数在什么地方以及什么时候执行？_


当 Java 层通过 `System.loadLibrary()` 加载完 JNI 动态库后，接着会查找库中的 `JNI_Onload` 的函数，如果有的话，就会调用他，而动态注册的工作就是在此处完成的。所以，如果想使用动态注册方法，就必须实现JNI_OnLoad函数，只有在这个函数中才有机会完成动态注册的工作。

MediaScanner 相应的 libmedia_jni.so 库的 JNI_OnLoad 函数的具体实现在 android_media_MediaPlayer.cpp 中，具体如下：

```
// 该函数的第一个参数类型为 JavaVM，为 Java 虚拟机在 JNI 层的代表，每个 Java 进程只有一个
jint JNI_OnLoad(JavaVM* vm, void* reserved){
    JNIEnv* env = NULL;
    jint result = -1;
    if(vm->GetEnv((void*) &env, JNI_VERSION_1_4) != JNI_OK){
        goto bail;
    }
    // 动态注册 MediaScanner 的 JNI 函数
    if(register_android_media_MediaScaner(env) < 0){
        goto bail;
    }
    return JNI_VERSION_1_4;

}
```

### native 函数的参数含义

```
/**
* Java 层的 processFile 只有 3 个参数，而 JNI 中的方法有 5 个参数。
* JNIEnv *env 为 代表 JNI 环境的结构体(JNI 可以调用的方法的结构体，比如 env->GetFieldID，其中 GetFieldID 方法即为 JNIEnv 结构体中的一员)
* jobject thiz：代表 Java 层的 MediaScanner 对象，如果方法为 static，参数为 jclass ,代表在调用 Java 的哪一个 Class 中的函数
* 剩下的为 Java 层中该方法的参数
*/
static jboolean android_media_MediaScanner_processFile(
        JNIEnv *env, jobject thiz, jstring path,
        jstring mimeType, jobject client){
    .....
}  
```

### JNIEnv 介绍

JNIEnv 是一个 **线程相关** 的 **代表 JNI 环境** 的 **结构体**。

![JNIEnv 内部结构简图](/images/2019_04_10_1.jpg)

JNIEnv 实际上是提供了一系列 JNI 系统函数，通过这些函数可以做到：
1. 调用 Java 函数
2. 操作 jobject 对象

### JavaVM 和 JNIEnv 的关系

* 调用 JavaVM 的 AttachCurrentThread 函数，就可得到 **这个线程** 的 JNIEnv 结构体。这样就可以在后台线程中回调 Java 函数了
* 在后台线程退出前，需要调用 JavaVM的DetachCurrentThread 函数来释放对应的资源

### JNIEnv 的使用

获得 FiledID 和 MethodID
```
static void
android_media_MediaPlayer_native_init(JNIEnv *env)
{
    jclass clazz;

    // 获得 jclass
    clazz = env->FindClass("android/media/MediaPlayer");
    // 获得 clazz 中的 FileID
    fields.context = env->GetFieldID(clazz, "mNativeContext", "J");
    // 获得 clazz 的 MethodID
    fields.post_event = env->GetStaticMethodID(clazz, "postEventFromNative",
                                               "(Ljava/lang/Object;IIILjava/lang/Object;)V");

    fields.surface_texture = env->GetFieldID(clazz, "mNativeSurfaceTexture", "J");
    env->DeleteLocalRef(clazz);

}
```

调用 Field 和 Method

```
//调用 JNIEnv 的 CallVoidMethod 函数
// 参数含义：mClient 为 MediaScannerClient 对象
// 第二个参数为函数 scanFile 的 jmedthodid ,后面为 scanFile 的参数
eEnv -> CallVoidMethod(mClient,mScanFileMethod,pathStr,lastModified,fileSize)
```

JNIEnv 有一系列类似 CallVoidMethod 的函数，形式如下：

```
NativeType Call<type>Method(JNIEnv *env,jobject obj,jmethodId methodId,....)
```

关于 JNIEnv 类型中方法的使用可以查看[Android 应用的安全防护和逆向分析: JNIEnv 类型中方法的使用](https://book.douban.com/subject/27617785/)


### jstring

jstring 对象可以看成 Java 中 String 对象在 JNI 层的代表。

1. JNIEnv 调用 NewString(JNIEnv *env,const jchar *unicodeChars,jsize len):从 Native 的字符得到 jstring 对象(Unicode)。
2. JNIEnv 的 NewStringUTF 将 Native 的一个 UTF 字符串得到一个 jstring 对象(UTF)。
3. JNIEnv 提供 GetStringChar 函数，将 Java String对象转换成本地 Unicode 字符串。
4. JNIEnv 提供 GetStringUTFChars 函数 ，将 Java String 对象转换为本地 UTF 字符串。
5. 调用上面四个函数后需要调用 ReleaseStringChars 或 ReleaseStringUTFChars 函数来释放相应资源。

### JNI 中的三种引用

1. Local Reference：本地引用。JNI 函数执行完成后，这些 jobject 可能被回收。
2. Global Reference：这种方式的引用，不主动释放，永远不会被回收。
3. Weak Global Reference：  在使用过程中，可能会被回收。