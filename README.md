###前言：
记得好像是16年年初的时候[蒸米](https://github.com/zhengmin1989)就发过一篇 [iOS冰与火之歌(作者:蒸米)](https://github.com/zhengmin1989/iOS_ICE_AND_FIRE)讲通过hook微信的红包相关方法来实现自动抢红包。当时就跟着试了不过由于手上没有越狱机和开发者证书（主要是经验不够），就一直卡在红包相关方法和重签名那里了，最近突然又想起来并且在看了几篇博客后又跟着复习了一遍，终于把自动抢红包功能实现了。

###准备：
建议先看以下博客以及博客内的相关链接文章和github地址，对相关术语及流程有一个大致的了解：
[一步一步实现iOS微信自动抢红包(非越狱)](http://www.jianshu.com/p/189afbe3b429)
[非越狱环境下从应用重签名到微信上加载Cycript](http://www.jianshu.com/p/262b9849fa10)

###环境：
我当前设备环境是：
- Xcode 8.1
- iPhone6s 10.1.1 (未越狱)
 
###需要下载的工具：

**必要：**
- [yololib](https://github.com/KJCracks/yololib) 用来实现动态注入
- [iOSOpenDev](http://iosopendev.com/download/) 因为Xcode默认不支持生成dylib，所以我们需要下载iOSOpenDev
- 苹果开发者证书或企业证书 用来重签名hook后的微信
- [CaptainHook](https://github.com/rpetrich/CaptainHook) 钩子
- [AppResign](https://github.com/Urinx/iOSAppHook/releases) 重签名脚本

**非必要：**
- [class-dump](http://stevenygard.com/projects/class-dump/) 用来导出头文件（[一步一步实现iOS微信自动抢红包(非越狱)](http://www.jianshu.com/p/189afbe3b429)这篇文章中已经告诉了我们需要hook的方法，所以不需要dump头文件来分析了）
- [dumpdecrypted](https://github.com/stefanesser/dumpdecrypted) 用来砸壳 （可以直接从PP助手上下载越狱微信ipa就可以，所以不再需要自己通过越狱机来砸壳）

###步骤：
####1.直接通过PP助手下载越狱版的微信（省去自己通过越狱机砸壳步骤）

![通过PP助手下载越狱微信](http://upload-images.jianshu.io/upload_images/870531-95f5b798216b65f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下载后解压缩：

![解压缩](http://upload-images.jianshu.io/upload_images/870531-db0b5a8d50a27601.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

拷贝出 **微信-6.5.4(越狱应用) -> Payload -> WeChat ** 就是得到砸壳后的文件：

![微信-6.5.4(越狱应用) -> Payload -> WeChat](http://upload-images.jianshu.io/upload_images/870531-8d21db04de0ac28f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####2.配置环境dylib

Xcode8 中是不支持创建dylib工程的，所以需要下载安装iOSOpenDev，安装完成后Xcode8环境会提示安装iOSOpenDev失败:

[一步一步实现iOS微信自动抢红包(非越狱)](http://www.jianshu.com/p/189afbe3b429) 中的解决办法是参考[iOSOpenDev安装问题](http://www.tqcto.com/article/software/14553.html)，重新打开Xcode，在新建项目的选项中即可看到iOSOpenDev选项了。

其中有一点需要注意的是目标目录的完整路径应该是：

````
/Applications/Xcode.app/Contents/Developer/Platforms
````

由于Xcode8下的 ** /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/Xcode ** 目录中并没有 **Specifications Target**、 **Templates Project** 、 **Templates** 这三个文件，所以我的做法是参考[ Xcode3创建和使用iOS的dylib动态库](http://blog.csdn.net/hursing/article/details/8688861) 将里面提到的 [Xcode创建和使用iOS的dylib动态库](http://download.csdn.net/detail/hursing/5159352)下载下来解压缩，将 ** Xcode创建和使用iOS的dylib动态库/Developer/iPhoneOS.platform/Developer/Library/Xcode ** 下的 **Specifications Target**、 **Templates Project** 、 **Templates** 三个文件夹拷贝到  ** /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/Xcode **目录下：

![拷贝后的 /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/Xcode 目录](http://upload-images.jianshu.io/upload_images/870531-4c101c024d719898.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

将上述步骤执行完后，重启Xcode，新建工程就可以看到多了iOSOpenDev选项了，选择Cocoa Touch Library 创建就可以了

![选中Cocoa Touch Librayry](http://upload-images.jianshu.io/upload_images/870531-ebc1b1ab3ba2acab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###3.制作dylib，hook 掉红包消息方法:
这里完全照搬 [一步一步实现iOS微信自动抢红包(非越狱)](http://www.jianshu.com/p/189afbe3b429) 中的内容
其中代码在该作者的[Github](https://github.com/east520/AutoGetRedEnv)上

````
选择Cocoa Touch Library，这样我们就新建了一个dylib工程了，我们命名为autoGetRedEnv。
删除autoGetRedEnv.h文件，修改autoGetRedEnv.m为autoGetRedEnv.mm，然后在项目中加入CaptainHook.h

因为微信不会主动来加载我们的hook代码，所以我们需要把hook逻辑写到构造函数中。

__attribute__((constructor)) static void entry()
{
  //具体hook方法
}
hook微信的AsyncOnAddMsg: MsgWrap:方法，实现方法如下：

//声明CMessageMgr类
CHDeclareClass(CMessageMgr);
CHMethod(2, void, CMessageMgr, AsyncOnAddMsg, id, arg1, MsgWrap, id, arg2)
{
  //调用原来的AsyncOnAddMsg:MsgWrap:方法
  CHSuper(2, CMessageMgr, AsyncOnAddMsg, arg1, MsgWrap, arg2);
  //具体抢红包逻辑
  //...
  //调用原生的打开红包的方法
  //注意这里必须为给objc_msgSend的第三个参数声明为NSMutableDictionary,不然调用objc_msgSend时，不会触发打开红包的方法
  ((void (*)(id, SEL, NSMutableDictionary*))objc_msgSend)(logicMgr, @selector(OpenRedEnvelopesRequest:), params);
}
__attribute__((constructor)) static void entry()
{
  //加载CMessageMgr类
  CHLoadLateClass(CMessageMgr);
  //hook AsyncOnAddMsg:MsgWrap:方法
  CHClassHook(2, CMessageMgr, AsyncOnAddMsg, MsgWrap);
}
项目的全部代码，笔者已放入Github中。
完成好具体实现逻辑后，就可以顺利生成dylib了。
````

写好代码后，run一下，会在工程目录的Product目录下生成我们制作的libautogetRedEnv.dylib文件
![Product目录下生成的libautogetRedEnv.dylib](http://upload-images.jianshu.io/upload_images/870531-18c211424b2078e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####4.注入libautogetRedEnv.dylib
我们把之前步骤下载和生成得到的 **WeChat、 yololib、libautoGetRedEnv.dylib、AppResign** 放到同一个目录下：

![需要用到的文件](http://upload-images.jianshu.io/upload_images/870531-9037b5506bba000b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

切换到该目录下执行：**./yololib WeChat.app/WeChat libautoGetRedEnv.dylib**
````
dsa:Demo xxx$ ./yololib WeChat.app/WeChat libautoGetRedEnv.dylib
2017-02-03 17:22:25.685 yololib[5070:410411] dylib path @executable_path/libautoGetRedEnv.dylib
2017-02-03 17:22:25.686 yololib[5070:410411] dylib path @executable_path/libautoGetRedEnv.dylib
Reading binary: WeChat.app/WeChat

2017-02-03 17:22:25.686 yololib[5070:410411] FAT binary!
2017-02-03 17:22:25.686 yololib[5070:410411] Injecting to arch 9
2017-02-03 17:22:25.686 yololib[5070:410411] Patching mach_header..
2017-02-03 17:22:25.686 yololib[5070:410411] Attaching dylib..

2017-02-03 17:22:25.687 yololib[5070:410411] Injecting to arch 0
2017-02-03 17:22:25.687 yololib[5070:410411] 64bit arch wow
2017-02-03 17:22:25.687 yololib[5070:410411] dylib size wow 64
2017-02-03 17:22:25.687 yololib[5070:410411] mach.ncmds 76
2017-02-03 17:22:25.687 yololib[5070:410411] mach.ncmds 77
2017-02-03 17:22:25.687 yololib[5070:410411] Patching mach_header..
2017-02-03 17:22:25.687 yololib[5070:410411] Attaching dylib..

2017-02-03 17:22:25.687 yololib[5070:410411] size 63
2017-02-03 17:22:25.687 yololib[5070:410411] complete!
````
完成libautoGetRedEnv.dylib 的注入

####5.重签名

**重签名之前，需要将WeChat.app里面的Watch文件夹、PlugIns文件夹一起删去，同时将libautoGetRedEnv.dylib 拷贝到WeChat文件中**

重签名部分参考 [非越狱环境下从应用重签名到微信上加载Cycript](http://www.jianshu.com/p/262b9849fa10)，直接使用作者提供的 AppResign脚本进行签名：（如果提示Permission denied  ，需要执行一下 chmod 777 AppResign ）

````
dsa:Demo xxx$ ./AppResign WeChat.app DYCRedWechat.ipa
=============================
[*] Configure Resigning
Choose Signing Ceritificate:
[0] iPhone Developer: xxxxx(xxxxx)
[1] iPhone Distribution: xxxxxxx(xxxxxx)
[2] iPhone Distribution: xxxxxxx(xxxxxx)
[3] iPhone Developer: xxxxxxx(xxxxxx)
[4] iPhone Distribution: xxxxxxx(xxxxxx)
[5] iPhone Distribution: xxxxxxx(xxxxxx)
> 3
Use Certificate: iPhone Developer: xxxxxxx(xxxxxx)

Choose Provisioning Profile:
[0] xxxxxxx(xxxxxx)
[1] xxxxxxx(xxxxxx)
[2] xxxxxxx(xxxxxx)
[3] xxxxxxx(xxxxxx)
> 1
Use Profile: xxxxxxx(xxxxxx)
Position: /Users/xxxxxxx/Library/MobileDevice/Provisioning Profiles/xxxxxx.mobileprovision

Use default bundle ID: com.xxx.xxx

Set App Display Name: DYCWeChat
Delete url schemes (y/n): y
=============================
[*] Start Resigning App
Copying app to payload directory
Copying provisioning profile to app bundle
Parsing entitlements
Changing App ID to com.xxx.xxx
Changing Display Name to DYCWeChat
Codesigning /WeChat.app/libautoGetRedEnv.dylib with entitlements
Codesigning /WeChat.app with entitlements
Packaging IPA
Done, output at DYCRedWechat.ipa
````
生成的ipa文件通过iTunes装到手机上就OK了：
![DYCRedWechat.ipa](http://upload-images.jianshu.io/upload_images/870531-26844b4ffda3c728.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###最后
再次鸣谢链接中的相关作者

**友情提示:**
此博客仅作为技术实践，不能用于非法途径！
