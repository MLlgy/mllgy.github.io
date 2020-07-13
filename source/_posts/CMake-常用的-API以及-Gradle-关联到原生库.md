---
title: CMake 常用的 API以及 Gradle 关联到原生库
date: 2020-06-08 16:08:43
tags: [CMake]
---

### 1. 创建一个静态/动态库

* **add_library 命令：** 创建一个动态库/静态库
```
add_library(hello-libs // 指定库名
		 SHARED // 指定构建出来库的属性 -- SHARED：动态库、STATIC：静态库
            	  hello-libs.cpp // 指定源码文件
                   )
```
根据源码构建动态/静态库。
<!-- more -->
* **find_library() 命令：** 添加到您的 CMake 构建脚本中以定位 NDK 库，并将其路径存储为一个变量
```
find_library( # Defines the name of the path variable that stores the
              # location of the NDK library.
              log-lib // 为 NDK 中库指定本地名称

              # Specifies the name of the NDK library that
              # CMake needs to locate. // 指定链接 NDK 中的库
              log )
```
这个 Api 是为了引入 NDK 中的库，需要注意的是本部操作只是单纯的将库引入到项目中，但是现在并不能合法的使用，如果想要合法的使用，需要v下面的一个 API 。

* **target_link_libraries() 命令：** 关联库
```
# Links your native library against one or more other native libraries.
target_link_libraries( // Specifies the target library. 指定本地库
					  hello-libs
						
					// 指定一个/多个关联到上面指定本地库的库
                      android
                      lib_gmath
                      lib_gperf
                      log)
```
* **target_include_directories 命令：** 指定编译给定目标时要使用的包含目录或目标 。[^1] 
```
target_include_directories(hello-libs PRIVATE
                           ${distribution_DIR}/gmath/include
                           ${distribution_DIR}/gperf/include)
```
以上命令表明 `hello-libs` 库中会使用到  `${distribution_DIR}/gmath/include` 、`${distribution_DIR}/gperf/include` 中的源文件。

### 2. 添加其他预构建库

由于想要添加的库已经事先完成了构建，那么为了实现使用它们的目的需要使用 `IMPORTED` 标志告知 CMake 您只希望将库导入到项目中

```
// 添加其他预构建库
add_library(# 要添加的库
			lib_gperf
			# 库的属性(静态/动态)
		    SHARED 
		    # IMPORT 标识
		    IMPORTED // import 标识)
		    
// 指定库的路径				   
set_target_properties(
				   # 指定目标库
				    lib_gperf 

					# Specifies the parameter you want to define.
				    PROPERTIES IMPORTED_LOCATION
				    
				    # Provides the path to the library you want to import
				    // 提供想要 import 的库的路径
				    // 在 Cmake 构建时添加依赖库的多个 ABI 版本，而不必为每个版本编写相同的命令，使用 ANDROID_ABI 路径变量，此变量为 NDK 默认的一组 ABI，或在 Gradle 手动配置
  				    ${distribution_DIR}/gperf/lib/${ANDROID_ABI}/libgperf.so)
  				    
// 为了确保 CMake 可以在编译时定位您的标头文件，您需要使用 include_directories() 命令，并包含标头文件的路径：
include_directories( imported-lib/include/ )

// 同上边描述，使用以下命令关联到自己的原生库
target_link_libraries( native-lib imported-lib app-glue ${log-lib} )
```
### 3. CMake 关联到 Gradle 
#### 3.1 配置 Gradle 关联到本地库

```
android {
  ...
  defaultConfig {...}
  buildTypes {...}

  // 外部本地构建配置闭包
  externalNativeBuild {

    // CMake 构建配置闭包.
    cmake {

      // 提供 CMakeLists.txt 的路径
      path "CMakeLists.txt"
    }
  }
}
```

#### 3.2 指定可选配置
* 您可以在模块级 build.gradle 文件的 defaultConfig {} 块中配置另一个 externalNativeBuild {} 块，为 CMake  指定可选参数和标志。
* 构建配置中为每个产品风味重写这些属性
```
android {
  ...
  defaultConfig {
    ...
    externalNativeBuild {
      cmake {    
        arguments "-DANDROID_ARM_NEON=TRUE", "-DANDROID_TOOLCHAIN=clang"

        // Sets optional flags for the C compiler.
        cFlags "-D_EXAMPLE_C_FLAG1", "-D_EXAMPLE_C_FLAG2"

        // Sets a flag to enable format macro constants for the C++ compiler.
        cppFlags "-D__STDC_FORMAT_MACROS"
      }
    }
  }

  buildTypes {...}

// 每个版本针对的本地库的版本不同，构建多个产品
  productFlavors {
    ...
    demo {
      ...
      externalNativeBuild {
        cmake {
         ...
          targets "native-lib-demo"
        }
      }
    }

    paid {
      ...
      externalNativeBuild {
        cmake {
          ...
          targets "native-lib-paid"
        }
      }
    }
  }

  // Use this block to link Gradle to your CMake or ndk-build script.
  externalNativeBuild {
    cmake {...}
    // or ndkBuild {...}
  }
}
```

#### 3.3 指定 ABI 打包到 APK 中
配置 `defaultConfig.ndk.abiFilters` 属性
```
android {
  ...
  defaultConfig {
    ...
    externalNativeBuild {
      cmake {...}
      // or ndkBuild {...}
    }
    ndk {
      abiFilters 'x86', 'x86_64', 'armeabi', 'armeabi-v7a',
                   'arm64-v8a'
    }
  }
  buildTypes {...}
  externalNativeBuild {...}
}

```
#### 参考资料
[Developer -- JNI](https://developer.android.com/studio/projects/add-native-code)
以上实例来自官方 demo：[hello-libs](https://github.com/googlesamples/android-ndk/tree/master/hello-libs)
[^1]: 具体解释参见 ： [CMake  官网](https://cmake.org/cmake/help/v3.0/command/target_include_directories.html)
