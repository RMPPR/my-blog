## 一、遇到的一些大坑
### 坑一: 使用.git下载会自带依赖libxxx.so
使用.git下载[zygisk的github项目](https://github.com/topjohnwu/zygisk-module-sample)和在网页下载github项目完全不一样，前者涉及一个作者自己编写的依赖（libxxx.so），使用作者这个依赖比较好编译通过但后者对zygisk不太支持，后面删掉了
### 坑二：自动构建的模块与zygisk的要求不同
我是使用android studio来进行构建的，编译输出zygisk模块（.zip时）很容易出错，最后是使用AI调试了几次写的自己编写的脚本才能通过（后面会详细描述）

### 坑三：原版magisk APK对zygisk的支持有限
3.使用原始的magisk APK似乎只能支持26.2，即使更新到30.6的APK，内核仍然不支持，最后放弃了使用了[magisk delta](https://magisk-delta.en.uptodown.com/android)

## 二、简单步骤（由我个人能跑通的流程）
### 1.编译zygisk模块
- 首先使用.git或者直接download去下载[zygisk-module](https://github.com/topjohnwu/zygisk-module-sample)

- 使用android studio打开，使用gradle进行构建，这个过程倒没啥问题

- 原本他自带了一个example.cpp但是我试了会在加载进去的时候报错，所以这里写入我们的example.cpp，他的功能是在app启动的时候插入一个日志
```
#include <stdint.h>  
#include <cstdlib>  
#include <unistd.h>  
#include <fcntl.h>  
#include <android/log.h>  
#include <string>  
  
#include "zygisk.hpp"  
  
using zygisk::Api;  
using zygisk::AppSpecializeArgs;  
using zygisk::ServerSpecializeArgs;  
  
// 将 DEBUG 改为 INFO，确保日志输出  
#define LOGD(...) __android_log_print(ANDROID_LOG_INFO, "MyModule", __VA_ARGS__)  
  
class MyModule : public zygisk::ModuleBase {  
public:  
    void onLoad(Api *api, JNIEnv *env) override {  
        this->api = api;  
        this->env = env;  
    }  
  
    void preAppSpecialize(AppSpecializeArgs *args) override {  
        // 1. 防御空指针 (沙盒进程没有名字)  
        if (args->nice_name == nullptr) {  
            api->setOption(zygisk::Option::DLCLOSE_MODULE_LIBRARY);  
            return;  
        }  
  
        // 2. 【核心过滤 1】：UID 物理隔离  
        // 凡是 UID < 10000 的全部是系统核心服务，直接放过，避免底层崩溃  
        if (args->uid < 10000) {  
            api->setOption(zygisk::Option::DLCLOSE_MODULE_LIBRARY);  
            return;  
        }  
  
        // 获取包名  
        const char *process = env->GetStringUTFChars(args->nice_name, nullptr);  
        std::string pkgName(process);  
        env->ReleaseStringUTFChars(args->nice_name, process);  
  
        // 3. 【核心过滤 2】：高危系统级 App 黑名单  
        // 过滤掉桌面、系统UI、谷歌底层服务等容易因为硬件渲染崩溃的 App        if (pkgName.find("com.android.systemui") != std::string::npos ||  
            pkgName.find("nexuslauncher") != std::string::npos ||  
            pkgName.find("com.google.android.gms") != std::string::npos ||  
            pkgName.find("com.google.android.googlequicksearchbox") != std::string::npos ||  
            pkgName.find("com.android.phone") != std::string::npos) {  
  
            // 命中黑名单，立刻自我卸载  
            api->setOption(zygisk::Option::DLCLOSE_MODULE_LIBRARY);  
            return;  
        }  
  
        // 4. 放行剩下的所有通用 App！(包括你自己编写和安装的各种第三方 App)        preSpecialize(pkgName.c_str());  
    }  
  
    void preServerSpecialize(ServerSpecializeArgs *args) override {  
        // 【核心防御】：绝对不碰 system_server        api->setOption(zygisk::Option::DLCLOSE_MODULE_LIBRARY);  
    }  
  
private:  
    Api *api;  
    JNIEnv *env;  
  
    void preSpecialize(const char *process) {  
        unsigned r = 0;  
        int fd = api->connectCompanion();  
  
        if (fd >= 0) {  
            read(fd, &r, sizeof(r));  
            close(fd);  
            // 成功连接并获取随机数，打印通用 App 的包名  
            LOGD("App Hooked! process=[%s], r=[%u]\n", process, r);  
        } else {  
            LOGD("process=[%s], Failed to connect companion!\n", process);  
        }  
    }  
};  
  
static int urandom = -1;  
  
static void companion_handler(int i) {  
    if (urandom < 0) {  
        urandom = open("/dev/urandom", O_RDONLY);  
    }  
  
    if (urandom >= 0) {  
        unsigned r = 0;  
        read(urandom, &r, sizeof(r));  
        // 注释掉 companion 的日志，避免日志过于杂乱，只看 App 端即可  
        // LOGD("companion r=[%u]\n", r);  
        write(i, &r, sizeof(r));  
    }  
}  
  
// 注册模块  
REGISTER_ZYGISK_MODULE(MyModule)  
REGISTER_ZYGISK_COMPANION(companion_handler)
```


- 然后这里遇到了第一二个坑：项目直接构建和assemble出来会是一个aar格式，不是zygisk模块所需要的.zip，而且zygisk要求的.zip样式如下：
![[Pasted image 20260602120308.png]]
- 他要包含一个名为zygisk的文件夹，其中的so以架构命名，一个.prop文件，因此我对应编写了构建文件build.gradle(module)：
```
plugins {  
    id 'com.android.library'  
}  
  
// =======================================================  
// 恢复自动打包 Magisk/Zygisk .zip 模块的神奇脚本  
// =======================================================  
// =======================================================  
// 终极版：全自动生成 module.prop 并打包 Zygisk 模块  
// =======================================================  
task buildMagiskZip(type: Zip) {  
    // 1. 确保在 C++ 和 Java 编译完成后再执行打包  
    dependsOn 'assembleRelease'  
  
    // 2. 指定输出路径和最终的 Zip 文件名  
    destinationDirectory = file("${rootProject.rootDir}/out")  
    archiveFileName = "EmuHook-v1.zip"  
  
    // 3. 动态生成你需要的 module.prop 文件！  
    def propFile = file("${buildDir}/intermediates/module.prop")  
    doFirst {  
        propFile.parentFile.mkdirs()  
        // 这里就是你自定义的模块信息，打包时会自动写入文件  
        propFile.text = """id=my_emuhook  
name=My EmuHook Module  
version=v1.0  
versionCode=1  
author=YourName  
description=A test Zygisk module for Android emulator.  
"""  
    }  
    // 将刚才生成的 module.prop 放入 zip 根目录  
    from(propFile)  
  
    // 4. (可选) 如果你项目根目录有 template 文件夹(包含 META-INF 刷机脚本)，一并打包  
    def templateDir = file("${rootProject.rootDir}/template")  
    if (templateDir.exists()) {  
        from(templateDir) {  
            exclude 'module.prop' // 排除旧的，防止和我们动态生成的冲突  
        }  
    }  
  
    // 5. 提取底层 .so 灵魂，并自动创建 zygisk 文件夹  
    // 5. 提取底层 .so 灵魂，并自动重命名为 Zygisk 官方严苛要求的格式  
    from(zipTree("${buildDir}/outputs/aar/module-release.aar")) {  
        include 'jni/**/*.so'  
        eachFile { file ->  
            // 文件在 AAR 里的原始路径类似于: jni/x86_64/libexample.so  
            // Magisk 要求的路径必须是: zygisk/x86_64.so  
  
            // 拿到倒数第二级的目录名 (比如 x86_64)            def abi = file.relativePath.segments[-2]  
  
            // 强行改变它的输出路径和文件名！  
            file.path = "zygisk/${abi}.so"  
        }  
        includeEmptyDirs = false  
    }  
}  
  
// 绑定触发器：在 assembleRelease 结束时自动执行上述任务  
afterEvaluate {  
    tasks.named("assembleRelease").configure {  
        finalizedBy buildMagiskZip  
    }  
}  
  
android {  
    namespace 'com.emuhook.module'  
    compileSdkVersion 31  
  
    defaultConfig {  
        minSdkVersion 26  
        targetSdkVersion 28  
        ndkVersion "26.1.10909125"  
  
        // 【致命修复】：必须把参数传给 ndkBuild，而不是 cmake！  
        externalNativeBuild {  
            ndkBuild {  
                // 确保参数以正确的列表形式传递给编译环境  
                arguments "APP_STL=c++_static",  
                        "APP_CPPFLAGS+=-std=c++17" // <-- 核心：强制指定 C++17 标准，彻底解开 lldiv 报错  
                abiFilters "x86_64", "x86", "arm64-v8a", "armeabi-v7a"  
            }  
        }    }  
    compileOptions {  
        sourceCompatibility JavaVersion.VERSION_1_8  
        targetCompatibility JavaVersion.VERSION_1_8  
    }  
  
    // 指定 Android.mk 脚本的路径  
    externalNativeBuild {  
        ndkBuild {  
            path("jni/Android.mk")  
        }  
    }}

```
- 上述文件会自动将输出文件打包为符合要求的形式

- 接着同步项目，最后编译输出，我采用的是命令行：
```
.\gradlew assembleRelease
```
- 直接使用按钮我之前试过报错了，暂没找到原因，就考虑一直用命令行了

- 最后在和.gradle平行的目录文件夹可以找到一个out，里面有我们的标准zygisk模块，我这里命名为EmuHook-v1.zip
![[Pasted image 20260602121036.png]]

### 2.安装magisk delta APK
这里我使用rootAVD来实现，但在实现的时候发现由于坑三，他自带的magisk APK不好用，于是我开始尝试使用[magisk delta](https://magisk-delta.en.uptodown.com/android)，我把这个apk放进了[rootAVD](https://github.com/newbit1/rootAVD)这个项目中改名并替换了原本的magisk的APK
- 然后启动AVD，版本选择了android11，这里注意构建模拟器不要使用带play的就行，![[Pasted image 20260602121843.png]]- 启动模拟器以后，切换到下载下来的rootAVD主目录，输入命令
```
rootAVD.bat system-images\android-30\google_apis\x86_64\ramdisk.img
```

这里注意，后面是你构建这个模拟器时自动下载的版本镜像的位置

- 执行完成后，他会自动关闭模拟器，请执行**Cold Boot Now (冷启动)**

- 然后打开模拟器，应用中有一个magisk delta APK，请点击，此时可能会弹出需要额外安装信息，点击安装即可

### 3.安装zygisk模块

- 点击 App 右上角的 **齿轮（设置）**。
    
- 往下滑动，找到 **Zygisk** 选项，**直接将其勾选开启**。
    
- **⚠️ 极其重要：** 开启后，**绝对不要**点击软件下方弹出的重启按钮。
    
- 回到 Android Studio 的 Device Manager。
    
- 手动彻底关闭该模拟器。
    
- 点击右侧菜单，执行 **Cold Boot Now（冷启动）**。

- 冷启动完成，以后Zygisk这里显示：yes，意味着ygisk环境已经完成：
![[Pasted image 20260602122730.png]]
- 此时再将之前编译好的EmuHook.zip拖进模拟器屏幕
- 点击右下角的**Modules**
- 点击 **Install from storage（从存储安装）**，找到你的模块压缩包并刷入。
- 刷入显示 `Success` 后，回到 Android Studio，再执行最后一次 **Cold Boot Now（冷启动）
这里也提供另一种方案（我的常用方案）
- 在个人电脑上进入cmd，依次输入：
```
abd shell
# 进入shell
su
# 进入root
magisk --install-module /sdcard/Download/EmuHook-v1.zip
#将zip拖入模拟器的默认路径是这个
exit
#退出shell
exit
#退出adb
adb logcat -s MyModule
#可以看到MyModule的运行日志，如下图所示
```
![[Pasted image 20260602123205.png]]
模块有问题的时候可以手动卸载或者进入adb shell以后输入
```
rm -rf /data/adb/modules/my_emuhook
```

todo：尝试编写hook逻辑