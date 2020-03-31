---
title: clang build
date: 2020-03-31 23:57:39
tags: 编译原理
---

# Clang 源码开发环境的搭建

跟随 [教程](https://www.jianshu.com/p/e3f46d42643b) 搭建

使用下面命令下载源码搭建环境：

``` shell
git clone -b release_90 git@github.com:llvm-mirror/llvm.git llvm
git clone -b release_90 git@github.com:llvm-mirror/clang.git llvm/tools/clang
git clone -b release_90 git@github.com:llvm-mirror/clang-tools-extra.git llvm/tools/clang/tools/extra
git clone -b release_90 git@github.com:llvm-mirror/compiler-rt.git llvm/projects/compiler-rt
```
<!-- more -->

下载完成之后，进行编译（需要将近1小时时间）编译完成之后磁盘空间占用大概 37G

## 创建插件
## 

详细创建流程参考 [LLVM & Clang 入门](https://github.com/CYBoys/Blogs/blob/master/LLVM_Clang/LLVM%20%26%20Clang%20%E5%85%A5%E9%97%A8.md) 里面讲得非常详细，不再进行copy赘述。

如果出现错误：

```
Unknown CMake command "add_llvm_loadable_module".
```

使用 `add_llvm_library` 代替 `add_llvm_loadable_module` 的使用

例如一个CMakeLists.txt文件的内容为：

``` c
add_llvm_library(xxPlugin MODULE xxPlugin.cpp PLUGIN_TOOL clang)

if(LLVM_ENABLE_PLUGINS AND (WIN32 OR CYGWIN))
  target_link_libraries(xxPlugin PRIVATE
    clangAST
    clangBasic
    clangFrontend
    LLVMSupport
    )
endif()
```

关于[add_llvm_loadable_module ](https://gitlab.freedesktop.org/nh/llvm/commit/69e8318af3e969bc67f18123a6ae5e7702ad8fc8)修改文档

返回到llvm_build目录，也就是LLVM工程的目录，使用 CMake 进行再次构建

``` shell
cmake -G Xcode ../llvm -DCMAKE_BUILD_TYPE:STRING=MinSizeRel
```

在Products目录下就能看到dylib动态库文件

## 集成到 Xcode 工程

1. 在工程的 build settings中搜索 `Other C Flags` 添加如下参数

    ```
    -Xclang -load -Xclang (.dylib)动态库路径 -Xclang -add-plugin -Xclang 插件名字（namespace 的名字，名字不对则无法使用插件）
    ```
    
    例如：
    
    ```
    -Xclang -load -Xclang ${PROJECT_DIR}/cctest/CCDetector.dylib -Xclang -add-plugin -Xclang CCDetector
    ```
    
2. 在Build Settings栏目中新增两项用户定义的设置
    
    ![](http://qiniu.huyangjie.cn/mweb/15846219530904.jpg)
    
    分别是CC和CXX。
    ![](http://qiniu.huyangjie.cn/mweb/15846219659124.jpg)
    
    
    
    CC对应的是自己编译的clang的绝对路径，CXX对应的是自己编译的clang++的绝对路径。

3. 在Build Settings栏目中搜索index，将Enable Index-Wihle-Building Functionality的Default改为NO。




### 参考

[LLVM & Clang 入门](https://github.com/CYBoys/Blogs/blob/master/LLVM_Clang/LLVM%20%26%20Clang%20%E5%85%A5%E9%97%A8.md)

[Clang 之旅--使用 Xcode 开发 Clang 插件](https://www.jianshu.com/p/e3f46d42643b)