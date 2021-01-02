## pixel 刷机
### 一、 BL 解锁

1. 打开设置-->关于手机，在版本号处连续点击7次，调出开发者选项。

2. 打开设置-->系统-->高级-->开发者选项，打开USB调试开关和OEM解锁。

如 `adb devices` 出现设置，则正式 USB 调试已打开

```bash
$ adb devices
List of devices attached
FA75P0306887	device
```

3. 若OEM解锁开关无法点击，在通知栏中将USB连接模式由充电模式改为文件传输模式。
4. 打开命令行工具，分两次输入以下两条指令：

```bash
adb reboot bootloader

# Pixel 2 XL 之后的版本
fastboot flashing unlock

# Pixel 2 XL 及之前的版本
fastboot oem unlock
```

注意：解锁只需要解一次，解锁之后千万不要随意锁上；锁上之前最好是把手机重刷一遍，保证不安装其他东西，然后再上锁。

### 二、刷入系统

根据twrp工具的作者发的一篇文章：

https://twrp.me/site/update/2019/10/23/twrp-and-android-10.html

目前twrp工具还不支持Android 10.0系统，所以如果想在pixel 上root，需要先安装9.0版本的系统，下载地址：

https://developers.google.com/android/images#crosshatch

下载pixel 相关机型的9.0系统，解压，然后进入解压后的目录

在USB调试打开的情况下，键入命令，进入 bootloader 模式：

```
adb reboot bootloader
```

也可在开机时，同时按住电源键+音量减，进入bootloader (不同的手机进入方式稍微有所差别)

查看序列号

```bash
fastboot devices -l
```

若出来一串序列号，说明安卓设备已连接

 ```bash
adb reboot bootloader   

# Linux/Mac（如果是 window，则执行 flash-all.bat 这个文件）
sh flash-all.sh
 ```
特别注意：官方脚本线刷默认是清除数据的，若想保留数据线刷，请使用Notepad++编辑脚本，将脚本中的-w删掉并保存脚本，再双击刷入即可保留数据。
```bash
fastboot -w update image-sailfish-opm4.171019.021.p1.zip
```

等待刷入后，手机重启，并会自动安装9.0系统。

### 三、刷入 twrp

Recovery 是安卓的恢复系统，类似 Windows 的 PE 和 macOS 的恢复功能，可以用来系统升级和重置手机

刷入第三方的 Recovery 可以获得更多的功能，比如 Root 和 刷入第三方 ROM

到官方下载对应的 twrp，https://twrp.me/Devices/

搜索对应的型号：

![twrp](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/android/twrp.png)

如 pixel 1，则选择 `Google Pixel（salifish）`，因为 pixel 比较特殊，还有欧版跟美版的区别，因此进去之后，还要选择欧版或者美版。

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/android/salifish.png)

然后我们再分别下载某个 img 和 zip，例如在这里，我下载的是 [twrp-pixel-installer-sailfish-3.3.0-0.zip](https://eu.dl.twrp.me/sailfish/twrp-pixel-installer-sailfish-3.3.0-0.zip.html) 和 [twrp-3.3.0-0-sailfish.img](https://eu.dl.twrp.me/sailfish/twrp-3.3.0-0-sailfish.img.html)

img是用来刷临时twrp recovery，zip刷的是正式twrp recovery

```bash
adb reboot bootloader

# 刷入临时 twrp recovery
fastboot boot twrp-3.3.0-0-sailfish.img

# 直接进入 recovery
fastboot reboot twrp-3.3.0-0-sailfish.img

# 或者先重启后再进入 recovery
fastboot reboot
# 进入 recovery 模式
fastboot reboot recovery
```

在 `recovery 模式` 下刷入`zip 包` 有两种方式

**a. 手动刷入**：

1. 拷贝 zip 压缩包到手机存储卡中：`adb push twrp-pixel-installer-sailfish-3.3.0-0.zip /sdcard`
2. 点击第一个选项 `install`
3. 拉到最下面，找到要刷的模块，例如 `twrp-pixel-installer-sailfish-3.3.0-0.zip`，点击即可
4. 接着会跳到 `install zip` 页面，这时候在下方把 `Swipe to confirm Flash` 拉到最右即可
5. 最后 `Reboot System` 重启手机

**b. 自动刷入**:

```bash
# sideload 准备模式
adb shell twrp sideload
# 刷入 zip 包，如 twrp-pixel-installer-sailfish-3.3.0-0.zip 
adb sideload twrp-pixel-installer-sailfish-3.3.0-0.zip 
 
# 刷完之后重启
adb reboot
```



如果有些机型没有 `zip` 包，如红米 note3，只有`img` ，那直接刷入`img` 即可

```bash
fastboot flash recovery twrp-3.3.1-0-hennessy.img
# 重启
fastboot reboot
# 或者重启到 recovery
fastboot boot twrp-3.3.1-0-hennessy.img
```

### 四、刷入 Magisk

推荐阅读：[Magisk 是如何工作的？](https://sspai.com/post/53043)

Magisk 是一个兼具稳定性和可玩性的神器：作为一个 Root 方案，它能不破坏系统实现无痛 OTA，作为一个插件扩展平台，它又能提供丰富的自定义模块来满足多样化的定制需求



安装 [Magisk](https://github.com/topjohnwu/Magisk/releases)， 版本不低于 20.4。安装方式有两种

##### a. 用三方 REC 卡刷 [Magisk-v21.1.zip](https://github.com/topjohnwu/Magisk/releases/tag/v21.1)

```bash
# 进入 recovery 模式
adb reboot recovery

# sideload 准备模式
adb shell twrp sideload
# 刷入 magisk
adb sideload Magisk-v20.2.zip

# sideload 准备模式
adb shell twrp sideload
# 刷入 zip 包，如 magisk 下的模块
adb sideload Magisk/Move_Certificates-v1.9.zip

# 重启手机
adb reboot
```

当然也可以手动刷，可参考步骤三中刷入 `twrp-pixel-installer-sailfish-3.3.0-0.zip`

##### b. 通过  [Magisk Manager v8.0.3.apk](https://github.com/topjohnwu/Magisk/releases/tag/manager-v8.0.3) 安装

安装完 Magisk Manager apk后， 选择安装（需要手机可以 fanqiang），安装后重启手机

卸载 Magisk 最为彻底的方式就是在 Magisk Manager 中点击「卸载」、「完全卸载」，应用会自动下载刷完 uninstall.zip 卸载包、自动卸载它自己、自动重启。如果你无法进入系统，在 TWRP 中手动刷入 uninstall.zip 卸载包即可。

### 五、刷入 Magisk 模块

在安装完 magisk 之后，我们的手机就已经是 root 状态了。此时我们还可以安装 magisk 的一些模块来实现某些功能。例如Android 7以上的 Android 系统，App默认不信任用户证书,只信任系统证书。例如`Move_Certificates.zip `的作用就是帮我们把证书从用户目录移动系统目录。安装 `Magisk 模块` 的方式也有几种

**1. 通过 magisk 在线安装**

这种最为简单，但是需要手机可以 fanqiang。

**2. [Magisk 官方仓库](https://github.com/Magisk-Modules-Repo?q=&type=&language=) 下载 zip 包，然后 adb push 推到手机里，再通过 magisk 安装**

```bash
adb push Magisk/Move_Certificates-v1.9.zip /sdcard
```

选择`从本地安装`，选择好刚刚推到手机的模块，点击要安装的模块，安装完后重启手机即可

![](https://img-1257127044.cos.ap-guangzhou.myqcloud.com/android/magis_install.png)



**3. [Magisk 官方仓库](https://github.com/Magisk-Modules-Repo?q=&type=&language=) 下载 zip 包，在 recovery 模式下安装**

```bash
# 进入 recovery 模式
adb reboot recovery

# sideload 准备模式
adb shell twrp sideload
# 刷入 zip 包，如 magisk 下的模块
adb sideload Magisk/Move_Certificates-v1.9.zip

# 重启手机
adb reboot
```


### 六、刷入 EdXpose

-  [Magisk 官方仓库](https://github.com/Magisk-Modules-Repo?q=&type=&language=) 下载 zip 包，然后adb push 推到手机里，再通过 magisk 安装，参考`步骤五 - 刷入 Magisk 模块`
-  Magisk 在线安装（需要手机可以fanqiang)

刷入 EdXpose 分为三步，分别是 magisk 安装 `Riru-Core`，magisk 安装 `edXposed`，安装 EdXposed 的app

1. Magisk 安装 `Riru-Core`，可在线搜索 `Riru-Core` 或者 官方仓库找到 [riru-core ](https://github.com/Magisk-Modules-Repo/riru-core)安装
2. Magisk 安装 `edXposed`，可在线搜索 `edXposed` （yahfa和sandhook版本二选一）或者 官方仓库找到 [edxpose](https://github.com/Magisk-Modules-Repo/riru_edxposed)安装，安装完后记得重启手机
3. 下载EdXposed 的app：https://github.com/ElderDrivers/EdXposedManager/releases， 然后安装即可: `adb install EdXposedManager-4.5.7-45700-org.meowcat.edxposed.manager-release.apk`



参考: 

[Pixel 3 XL 刷入EdXposed框架](https://blog.csdn.net/u010838555/article/details/105655660)
[EdXposed安装教程-支持Pie的Xposed构架](https://www.jianshu.com/p/e4485cbaf3cd)

