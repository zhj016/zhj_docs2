#System UI 分析


##相关资料索引

[android 6.0 SystemUI源码分析(3)-Recent Panel加载显示流程](http://blog.csdn.net/zhudaozhuan/article/details/50819499)







##System UI 包含哪些component

####Shell

+ System UI Service / System UI application

####Panel
+ System UI class
	+ Recents
	+ SystemBar
	+ VolumeUI
	+ KeyguardViewMediator
	+ TunerService
	+ PowerUI
	

####Service

+ LoadAverageService
+ KeyguardService
+ ImageWallpaper

####Activity

+ .recents.RecentsActivity
+ .usb.UsbConfirmActivity
+ .usb.UsbPermissionActivity
+ .usb.UsbResolverActivity
+ .usb.UsbAccessoryUriActivity
+ .usb.UsbDebuggingActivity
+ .usb.UsbDebuggingSecondaryUserActivity
+ .settings.BrightnessDialog

##System UI 如何启动

在SystemServer中，按intent方式启动。


##System UI 与系统的接口与交互

+ PhoneWindowManger : 处理按键，调用SBMS接口
+ SBMS : StatusBarManagerService 
	+ 提供mBar接口给系统  IStatusBarService.Stub
	+ 接受mBar注册   registerStatusBar
+ SystemUI, 注册到SBMS
	+ mBarService.registerStatusBar(mCommandQueue, iconList, switches, binders);
	+ public class CommandQueue extends IStatusBar.Stub {	

##StatusBar与Recents之间的联系

+ RecentsComponent 接口
+ SystemUIApplication 
	+     private final Map<Class<?>, Object> mComponents = new HashMap<Class<?>, Object>();  






