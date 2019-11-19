## VNDK的目的

在Android9，将Framwork和Vendor(HAL)分割为两个独立的空间，想要实现Framework和Vendor开发域能够独立更新。
这样Android9的Framework可以独立升级到9.1，而Vendor开发域可以维持在9.0的适配版本。

## Concept

要想达成上述的目的，则需要完成两件事：
1. 以前Framework和Vendor_HAL之间是通过动态链接库的接口来共享的，如果要独立更新，不需要重新链接，则需要将接口独立，使用进程间通信的方式来进行
2. 以前Framework和Vendor(HAL)是共享一套动态链接库的的，为了避免更新Framework的时候，动态链接库的更新影响Vendor_HAL开发域，则需要将动态链接库独立为两套

## 现状进展：

Android9.0已经全面使能了vmdk，所以已经是两套库了。

## 需要权衡的问题

1. 原则上Vendor_HAL模块不能使用Framework空间的库，Framework不能使用Vendor_HAL空间的库，即放在vndk目录下的库只允许被vendor-hal引用
2. 从进程内函数调用改为进程间通信，会导致性能的损失，所以从平衡的视角来看，高性能需要的模块，还是要沿用进程内函数调用的方法但是要限定范围，为了实现这个功能，引入了sp-hal的概念，即标记为sp-hal的vendor_hal库，可以被frameworks引入,而且这些sp-hal的库的范围是google进行控制的
   * opengl相关的库（render、mapper、egl..）
   * 内存共享库android.hidl.memory 
3. 还有一个问题，即SP-hal的库可能还会使用framwork空间的库，这部分的管理引入一个新的概念，即sp-vndk
4. 有关sp-hal和vndk-sp的相关库的列表，请参考附件


## 最终方案，分区方案：

* vendor扩展的vndk库，需要放置在vendor/lib目录下
    * vendor扩展的vndk库要遵循google定义的接口
    * 遵循ocp原则，即修改关闭，扩展开放
* FrameworkOnly的库存放在system/lib(lib64)
* sp-hal的库存放在system/lib(lib64)
* 而sp-hal依赖的vndk-sp的库存放在system/lib(lib64)/vndk-sp
* vndk的库存放在system/lib/vndk
* vndk-扩展存放在vendor/lib/vndk
* vnd-only存放在vendor/lib下


## Build系统改变点

● Define a vendor module which must be installed to vendor partition
* LOCAL_VENDOR_MODULE := true (Android.mk)
* vendor: true (Android.bp)

● Enable full VNDK build-time support (Android 8.1)
*  BOARD_VNDK_VERSION := current (BoardConfig.mk)
*  Build two variants: vendor_available: true
*  VNDK: vndk.enabled: true
*  VNDK-SP: vndk.support_system_process: true

● Disable runtime dynamic linker isolation between framework and vendor (Android 8.1)
*  BOARD_VNDK_RUNTIME_DISABLE := true (BoardConfig.mk)
