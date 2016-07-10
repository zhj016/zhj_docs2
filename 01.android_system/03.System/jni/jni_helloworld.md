

##Mac os x下配置 Android ndk 开发环境

1. Download NDK package and extract to /Users/zhj016/soft/ndk/android-ndk-r10d
2. modify .bash_profile

	export PATH="/Users/zhj016/soft/ndk/android-ndk-r10d:$PATH"
	export ANDROID_HOME="/Users/zhj016/Library/Android/sdk"
	export ANDROID_SDK="/Users/zhj016/Library/Android/sdk"
	export NDK="/Users/zhj016/soft/ndk/android-ndk-r10d"

##native helloworld

source code 
		
		.
		├── jni
		│   ├── Android.mk
		│   └── hello.c

Android.mk

		
		LOCAL_PATH:= $(call my-dir)  
		include $(CLEAR_VARS)  
		LOCAL_SRC_FILES:=hello.c  
		LOCAL_MODULE := helloworld  
		LOCAL_MODULE_TAGS := optional  
		
		LOCAL_CFLAGS += -pie -fPIE
		LOCAL_LDFLAGS += -pie -fPIE
		
		include $(BUILD_EXECUTABLE)


hello.c

		
		#include <stdio.h>  
		int main()  
		{  
		    printf("Hello World!\n");  
		    return 0;  
		  
		}  

	export NDK_PROJECT_PATH=/Users/zhj016/test/ndk_helloworld
	ndk-build
	adb push libs/armeabi/helloworld /system/bin/
	adb shell
	helloworld

			Hello World!

##在源码中编译

