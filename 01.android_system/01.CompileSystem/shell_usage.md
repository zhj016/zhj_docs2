#Android makefile里的shell

##一个简单的复制

    $(shell cp -rf $(LOCAL_PATH)/libs/*.so $(TARGET_OUT)/lib)


##调用脚本

    LOCAL_PATH := $(call my-dir)
    include $(CLEAR_VARS)

    $(shell ($(LOCAL_PATH)/echo_test.sh))

    LOCAL_MODULE := libecho_test
    LOCAL_MODULE_TAGS := optional
    include $(BUILD_SHARED_LIBRARY)

下面的echo_test.sh正确执行

    #!/bin/bash
    echo 'echo is working' >&2

or

    #!/bin/bash
    echo 'echo is working' >/dev/null

下面的脚本不能正确执行

  #!/bin/bash
  echo 'echo is working'

The Android NDK build system is actually GNU Make. All of the code in the Android.mk file has to be valid make.

When you run $(shell) and don't store the value in a variable, then it is as if you copied the standard output of the script into your Android.mk file. i.e. it is as if your file contained the following:

    LOCAL_PATH := $(call my-dir)
    include $(CLEAR_VARS)

    echo is working

    LOCAL_MODULE := libecho_test
    LOCAL_MODULE_TAGS := optional
    include $(BUILD_SHARED_LIBRARY)

.. which is not valid make syntax. Redirecting to >&2 in your script works because the output goes to the error output and is then shown on the console.

如果要打印信息，使用 $(info) or $(warning). Or if you really want to run a script during the build, store its output in a variable:

    ECHO_RESULT := $(shell ($(LOCAL_PATH)/echo_test.sh))

or

    $(info $(shell ($(LOCAL_PATH)/echo_test.sh)))

Here you won't see the echo output of the script, it goes into the variable.
