
##Android系统的开机画面显示过程分析


第一个开机画面是在内核启动的过程中出现的，它是一个静态的画面。第二个开机画面是在init进程启动的过程中出现的，它也是一个静态的画面。第三个开机画面是在系统服务启动的过程中出现的，它是一个动态的画面。无论是哪一个画面，它们都是在一个称为帧缓冲区（frame buffer，简称fb）的硬件设备上进行渲染的。接下来，我们就分别分析这三个画面是如何在fb上显示的。


##  1. 第一个开机画面的显示过程


Android系统的第一个开机画面其实是Linux内核的启动画面。在默认情况下，这个画面是不会出现的，除非我们在编译内核的时候，启用以下两个编译选项：

        CONFIG_FRAMEBUFFER_CONSOLE

        CONFIG_LOGO

第一个编译选项表示内核支持帧缓冲区控制台，它对应的配置菜单项为：Device Drivers ---> Graphics support ---> Console display driver support ---> Framebuffer Console support。第二个编译选项表示内核在启动的过程中，需要显示LOGO，它对应的配置菜单项为：Device Drivers ---> Graphics support ---> Bootup logo。配置Android内核编译选项可以参考在Ubuntu上下载、编译和安装Android最新内核源代码（Linux Kernel）一文。


帧缓冲区硬件设备在内核中有一个对应的驱动程序模块fbmem，它实现在文件kernel/goldfish/drivers/video/fbmem.c中，它的初始化函数如下所示：

      /**
       *      fbmem_init - init frame buffer subsystem
       *
       *      Initialize the frame buffer subsystem.
       *
       *      NOTE: This function is _only_ to be called by drivers/char/mem.c.
       *
       */  

      static int __init  
      fbmem_init(void)  
      {  
              proc_create("fb", 0, NULL, &fb_proc_fops);  

              if (register_chrdev(FB_MAJOR,"fb",&fb_fops))  
                      printk("unable to get major %d for fb devs\n", FB_MAJOR);  

              fb_class = class_create(THIS_MODULE, "graphics");  
              if (IS_ERR(fb_class)) {  
                      printk(KERN_WARNING "Unable to create fb class; errno = %ld\n", PTR_ERR(fb_class));  
                      fb_class = NULL;  
              }  
              return 0;  
      }  


这个函数首先调用函数proc_create在/proc目录下创建了一个fb文件，接着又调用函数register_chrdev来注册了一个名称为fb的字符设备，最后调用函数class_create在/sys/class目录下创建了一个graphics目录，用来描述内核的图形系统。


##   2. 第二个开机画面的显示过程

由于第二个开机画面是在init进程启动的过程中显示的，因此，我们就从init进程的入口函数main开始分析第二个开机画面的显示过程。

init进程的入口函数main实现在文件system/core/init/init.c中，如下所示：


    int main(int argc, char **argv)  
    {  

        ......  

        queue_builtin_action(console_init_action, "console_init");  

        ......  


        return 0;  
    }  

接下来我们就重点分析函数console_init_action的实现，以便可以了解第二个开机画面的显示过程：


    static int console_init_action(int nargs, char **args)  
    {  
        int fd;  
        char tmp[PROP_VALUE_MAX];  

        if (console[0]) {  
            snprintf(tmp, sizeof(tmp), "/dev/%s", console);  
            console_name = strdup(tmp);  
        }  

        fd = open(console_name, O_RDWR);  
        if (fd >= 0)  
            have_console = 1;  
        close(fd);  

        if( load_565rle_image(INIT_IMAGE_FILE) ) {  
        ......
            }  
        }  
        return 0;  
    }  


load_565rle_image

    int load_565rle_image(char *fn)  
    {  
        struct FB fb;  
        struct stat s;  
        unsigned short *data, *bits, *ptr;  
        unsigned count, max;  
        int fd;  

        if (vt_set_mode(1))  
            return -1;  

        fd = open(fn, O_RDONLY);  
        if (fd < 0) {  
            ERROR("cannot open '%s'\n", fn);  
            goto fail_restore_text;  
        }  

        if (fstat(fd, &s) < 0) {  
            goto fail_close_file;  
        }  

        data = mmap(0, s.st_size, PROT_READ, MAP_SHARED, fd, 0);  
        if (data == MAP_FAILED)  
            goto fail_close_file;  

        if (fb_open(&fb))  
            goto fail_unmap_data;  

        max = fb_width(&fb) * fb_height(&fb);  
        ptr = data;  
        count = s.st_size;  
        bits = fb.bits;  
        while (count > 3) {  
            unsigned n = ptr[0];  
            if (n > max)  
                break;  
            android_memset16(bits, ptr[1], n << 1);  
            bits += n;  
            max -= n;  
            ptr += 2;  
            count -= 4;  
        }  

        munmap(data, s.st_size);  
        fb_update(&fb);  
        fb_close(&fb);  
        close(fd);  
        unlink(fn);  
        return 0;  

    fail_unmap_data:  
        munmap(data, s.st_size);  
    fail_close_file:  
        close(fd);  
    fail_restore_text:  
        vt_set_mode(0);  
        return -1;  
    }  


 将文件/initlogo.rle映射到init进程的地址空间之后，接下来再调用函数fb_open来打开设备文件/dev/graphics/fb0。前面在介绍第一个开机画面的显示过程中提到，设备文件/dev/graphics/fb0是用来访问系统的帧缓冲区硬件设备的，因此，打开了设备文件/dev/graphics/fb0之后，我们就可以将文件/initlogo.rle的内容输出到帧缓冲区硬件设备中去了。


##3. 第三个开机画面的显示过程


第三个开机画面是由应用程序bootanimation来负责显示的。应用程序bootanimation在启动脚本init.rc中被配置成了一个服务，如下所示：

      service bootanim /system/bin/bootanimation  
          user graphics  
          group graphics  
          disabled  
          oneshot  


应用程序bootanimation的用户和用户组名称分别被设置为graphics。注意， 用来启动应用程序bootanimation的服务是disable的，即init进程在启动的时候，不会主动将应用程序bootanimation启动起来。当SurfaceFlinger服务启动的时候，它会通过修改系统属性ctl.start的值来通知init进程启动应用程序bootanimation，以便可以显示第三个开机画面，而当System进程将系统中的关键服务都启动起来之后，ActivityManagerService服务就会通知SurfaceFlinger服务来修改系统属性ctl.stop的值，以便可以通知init进程停止执行应用程序bootanimation，即停止显示第三个开机画面。接下来我们就分别分析第三个开机画面的显示过程和停止过程。


Sytem进程在启动SurfaceFlinger服务的过程中，首先会创建一个SurfaceFlinger实例，然后再将这个实例注册到Service Manager中去。在注册的过程，前面创建的SurfaceFlinger实例会被一个sp指针引用。从前面Android系统的智能指针（轻量级指针、强指针和弱指针）的实现原理分析一文可以知道，当一个对象第一次被智能指针引用的时候，这个对象的成员函数onFirstRef就会被调用。由于SurfaceFlinger重写了父类RefBase的成员函数onFirstRef，因此，在注册SurfaceFlinger服务的过程中，将会调用SurfaceFlinger类的成员函数onFirstRef。在调用的过程，就会创建一个线程来启动第三个开机画面。

SurfaceFlinger类实现在文件frameworks/base/services/surfaceflinger/SurfaceFlinger.cpp 中，它的成员函数onFirstRef的实现如下所示：

      void SurfaceFlinger::onFirstRef()  
      {  
          run("SurfaceFlinger", PRIORITY_URGENT_DISPLAY);  

          // Wait for the main thread to be done with its initialization  
          mReadyToRunBarrier.wait();  
      }  


      status_t SurfaceFlinger::readyToRun()  
      {  
          LOGI(   "SurfaceFlinger's main thread ready to run. "  
                  "Initializing graphics H/W...");  

          ......  

          mReadyToRunBarrier.open();  

          /*
           *  We're now ready to accept clients...
           */  

          // start boot animation  
          property_set("ctl.start", "bootanim");  

          return NO_ERROR;  
      }  


前面在介绍第二个开机画面的时候提到，当系统属性发生改变时，init进程就会接收到一个系统属性变化通知，这个通知最终是由在init进程中的函数handle_property_set_fd来处理的。

函数handle_property_set_fd实现在文件system/core/init/property_service.c中，如下所示：

        void handle_property_set_fd()  
        {  
            prop_msg msg;  
            int s;  
            int r;  
            int res;  
            struct ucred cr;  
            struct sockaddr_un addr;  
            socklen_t addr_size = sizeof(addr);  
            socklen_t cr_size = sizeof(cr);  

            if ((s = accept(property_set_fd, (struct sockaddr *) &addr, &addr_size)) < 0) {  
                return;  
            }  

            /* Check socket options here */  
            if (getsockopt(s, SOL_SOCKET, SO_PEERCRED, &cr, &cr_size) < 0) {  
                close(s);  
                ERROR("Unable to recieve socket options\n");  
                return;  
            }  

            r = recv(s, &msg, sizeof(msg), 0);  
            close(s);  
            if(r != sizeof(prop_msg)) {  
                ERROR("sys_prop: mis-match msg size recieved: %d expected: %d\n",  
                      r, sizeof(prop_msg));  
                return;  
            }  

            switch(msg.cmd) {  
            case PROP_MSG_SETPROP:  
                msg.name[PROP_NAME_MAX-1] = 0;  
                msg.value[PROP_VALUE_MAX-1] = 0;  

                if(memcmp(msg.name,"ctl.",4) == 0) {  
                    if (check_control_perms(msg.value, cr.uid, cr.gid)) {  
                        handle_control_message((char*) msg.name + 4, (char*) msg.value);  
                    } else {  
                        ERROR("sys_prop: Unable to %s service ctl [%s] uid: %d pid:%d\n",  
                                msg.name + 4, msg.value, cr.uid, cr.pid);  
                    }  
                } else {  
                    if (check_perms(msg.name, cr.uid, cr.gid)) {  
                        property_set((char*) msg.name, (char*) msg.value);  
                    } else {  
                        ERROR("sys_prop: permission denied uid:%d  name:%s\n",  
                              cr.uid, msg.name);  
                    }  
                }  
                break;  

            default:  
                break;  
            }  
        }  

init进程是通过一个socket来接收系统属性变化事件的。每一个系统属性变化事件的内容都是通过一个prop_msg对象来描述的。在prop_msg对象对，成员变量name用来描述发生变化的系统属性的名称，而成员变量value用来描述发生变化的系统属性的值。系统属性分为两种类型，一种是普通类型的系统属性，另一种是控制类型的系统属性（属性名称以“ctl.”开头）。控制类型的系统属性在发生变化时，会触发init进程执行一个命令，而普通类型的系统属性就不具有这个特性。注意，改变系统属性是需要权限，因此，函数handle_property_set_fd在处理一个系统属性变化事件之前，首先会检查修改系统属性的进程是否具有相应的权限，这是通过调用函数check_control_perms或者check_perms来实现的。

从前面的调用过程可以知道，当前发生变化的系统属性的名称为“ctl.start”，它的值被设置为“bootanim”。由于这是一个控制类型的系统属性，因此，在通过了权限检查之后，另外一个函数handle_control_message就会被调用，以便可以执行一个名称为“bootanim”的命令。


 函数handle_control_message实现在system/core/init/init.c中，如下所示：


      void handle_control_message(const char *msg, const char *arg)  
      {  
          if (!strcmp(msg,"start")) {  
              msg_start(arg);  
          } else if (!strcmp(msg,"stop")) {  
              msg_stop(arg);  
          } else {  
              ERROR("unknown control msg '%s'\n", msg);  
          }  
      }  

 控制类型的系统属性的名称是以"ctl."开头，并且是以“start”或者“stop”结尾的，其中，“start”表示要启动某一个服务，而“stop”表示要停止某一个服务，它们是分别通过函数msg_start和msg_stop来实现的。由于当前发生变化的系统属性是以“start”来结尾的，因此，接下来就会调用函数msg_start来启动一个名称为“bootanim”的服务。

 函数msg_start实现在文件system/core/init/init.c中，如下所示：

    static void msg_start(const char *name)  
    {  
        struct service *svc;  
        char *tmp = NULL;  
        char *args = NULL;  

        if (!strchr(name, ':'))  
            svc = service_find_by_name(name);  
        else {  
            tmp = strdup(name);  
            args = strchr(tmp, ':');  
            *args = '\0';  
            args++;  

            svc = service_find_by_name(tmp);  
        }  

        if (svc) {  
            service_start(svc, args);  
        } else {  
            ERROR("no such service '%s'\n", name);  
        }  
        if (tmp)  
            free(tmp);  
    }  

 参数name的值等于“bootanim”，它用来描述一个服务名称。这个函数首先调用函数service_find_by_name来找到名称等于“bootanim”的服务的信息，这些信息保存在一个service结构体svc中，接着再调用另外一个函数service_start来将对应的应用程序启动起来。

从前面的内容可以知道，名称等于“bootanim”的服务所对应的应用程序为/system/bin/bootanimation，这个应用程序实现在frameworks/base/cmds/bootanimation目录中，其中，应用程序入口函数main是实现在frameworks/base/cmds/bootanimation/bootanimation_main.cpp中的，如下所示：

      int main(int argc, char** argv)  
      {  
      #if defined(HAVE_PTHREADS)  
          setpriority(PRIO_PROCESS, 0, ANDROID_PRIORITY_DISPLAY);  
      #endif  

          char value[PROPERTY_VALUE_MAX];  
          property_get("debug.sf.nobootanimation", value, "0");  
          int noBootAnimation = atoi(value);  
          LOGI_IF(noBootAnimation,  "boot animation disabled");  
          if (!noBootAnimation) {  

              sp<ProcessState> proc(ProcessState::self());  
              ProcessState::self()->startThreadPool();  

              // create the boot animation object  
              sp<BootAnimation> boot = new BootAnimation();  

              IPCThreadState::self()->joinThreadPool();  

          }  
          return 0;  
      }  


这个函数首先检查系统属性“debug.sf.nobootnimaition”的值是否不等于0。如果不等于的话，那么接下来就会启动一个Binder线程池，并且创建一个BootAnimation对象。这个BootAnimation对象就是用来显示第三个开机画面的。由于BootAnimation对象在显示第三个开机画面的过程中，需要与SurfaceFlinger服务通信，因此，应用程序bootanimation就需要启动一个Binder线程池。

BootAnimation类间接地继承了RefBase类，并且重写了RefBase类的成员函数onFirstRef，因此，当一个BootAnimation对象第一次被智能指针引用的时，这个BootAnimation对象的成员函数onFirstRef就会被调用。

BootAnimation类的成员函数onFirstRef实现在文件frameworks/base/cmds/bootanimation/BootAnimation.cpp中，如下所示：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

    void BootAnimation::onFirstRef() {  
        status_t err = mSession->linkToComposerDeath(this);  
        LOGE_IF(err, "linkToComposerDeath failed (%s) ", strerror(-err));  
        if (err == NO_ERROR) {  
            run("BootAnimation", PRIORITY_DISPLAY);  
        }  
    }  


mSession是BootAnimation类的一个成员变量，它的类型为SurfaceComposerClient，是用来和SurfaceFlinger执行Binder进程间通信的，它是在BootAnimation类的构造函数中创建的，如下所示：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

    BootAnimation::BootAnimation() : Thread(false)  
    {  
        mSession = new SurfaceComposerClient();  
    }  

SurfaceComposerClient类内部有一个实现了ISurfaceComposerClient接口的Binder代理对象mClient，这个Binder代理对象引用了SurfaceFlinger服务，SurfaceComposerClient类就是通过它来和SurfaceFlinger服务通信的。

回到BootAnimation类的成员函数onFirstRef中，由于BootAnimation类引用了SurfaceFlinger服务，因此，当SurfaceFlinger服务意外死亡时，BootAnimation类就需要得到通知，这是通过调用成员变量mSession的成员函数linkToComposerDeath来注册SurfaceFlinger服务的死亡接收通知来实现的。

BootAnimation类继承了Thread类，因此，当BootAnimation类的成员函数onFirstRef调用了父类Thread的成员函数run之后，系统就会创建一个线程，这个线程在第一次运行之前，会调用BootAnimation类的成员函数readyToRun来执行一些初始化工作，后面再调用BootAnimation类的成员函数htreadLoop来显示第三个开机画面。

BootAnimation类的成员函数readyToRun的实现如下所示：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

    status_t BootAnimation::readyToRun() {  
        mAssets.addDefaultAssets();  

        DisplayInfo dinfo;  
        status_t status = session()->getDisplayInfo(0, &dinfo);  
        if (status)  
            return -1;  

        // create the native surface  
        sp<SurfaceControl> control = session()->createSurface(  
                getpid(), 0, dinfo.w, dinfo.h, PIXEL_FORMAT_RGB_565);  
        session()->openTransaction();  
        control->setLayer(0x40000000);  
        session()->closeTransaction();  

        sp<Surface> s = control->getSurface();  

        // initialize opengl and egl  
        const EGLint attribs[] = {  
                EGL_DEPTH_SIZE, 0,  
                EGL_NONE  
        };  
        EGLint w, h, dummy;  
        EGLint numConfigs;  
        EGLConfig config;  
        EGLSurface surface;  
        EGLContext context;  

        EGLDisplay display = eglGetDisplay(EGL_DEFAULT_DISPLAY);  

        eglInitialize(display, 0, 0);  
        EGLUtils::selectConfigForNativeWindow(display, attribs, s.get(), &config);  
        surface = eglCreateWindowSurface(display, config, s.get(), NULL);  
        context = eglCreateContext(display, config, NULL, NULL);  
        eglQuerySurface(display, surface, EGL_WIDTH, &w);  
        eglQuerySurface(display, surface, EGL_HEIGHT, &h);  

        if (eglMakeCurrent(display, surface, surface, context) == EGL_FALSE)  
            return NO_INIT;  

        mDisplay = display;  
        mContext = context;  
        mSurface = surface;  
        mWidth = w;  
        mHeight = h;  
        mFlingerSurfaceControl = control;  
        mFlingerSurface = s;  

        mAndroidAnimation = true;  
        if ((access(USER_BOOTANIMATION_FILE, R_OK) == 0) &&  
                (mZip.open(USER_BOOTANIMATION_FILE) == NO_ERROR) ||  
                (access(SYSTEM_BOOTANIMATION_FILE, R_OK) == 0) &&  
                (mZip.open(SYSTEM_BOOTANIMATION_FILE) == NO_ERROR))  
            mAndroidAnimation = false;  

        return NO_ERROR;  
    }  

BootAnimation类的成员函数session用来返回BootAnimation类的成员变量mSession所描述的一个SurfaceComposerClient对象。通过调用SurfaceComposerClient对象mSession的成员函数createSurface可以获得一个SurfaceControl对象control。

SurfaceComposerClient类的成员函数createSurface首先调用内部的Binder代理对象mClient来请求SurfaceFlinger返回一个类型为SurfaceLayer的Binder代理对象，接着再使用这个Binder代理对象来创建一个SurfaceControl对象。创建出来的SurfaceControl对象的成员变量mSurface就指向了从SurfaceFlinger返回来的类型为SurfaceLayer的Binder代理对象。有了这个Binder代理对象之后，SurfaceControl对象就可以和SurfaceFlinger服务通信了。

调用SurfaceControl对象control的成员函数getSurface会返回一个Surface对象s。这个Surface对象s内部也有一个类型为SurfaceLayer的Binder代理对象mSurface，这个Binder代理对象与前面所创建的SurfaceControl对象control的内部的Binder代理对象mSurface引用的是同一个SurfaceLayer对象。这样，Surface对象s也可以通过其内部的Binder代理对象mSurface来和SurfaceFlinger服务通信。

Surface类继承了ANativeWindow类。ANativeWindow类是连接OpenGL和Android窗口系统的桥梁，即OpenGL需要通过ANativeWindow类来间接地操作Android窗口系统。这种桥梁关系是通过EGL库来建立的，所有以egl为前缀的函数名均为EGL库提供的接口。

为了能够在OpenGL和Android窗口系统之间的建立一个桥梁，我们需要一个EGLDisplay对象display，一个EGLConfig对象config，一个EGLSurface对象surface，以及一个EGLContext对象context，其中，EGLDisplay对象display用来描述一个EGL显示屏，EGLConfig对象config用来描述一个EGL帧缓冲区配置参数，EGLSurface对象surface用来描述一个EGL绘图表面，EGLContext对象context用来描述一个EGL绘图上下文（状态），它们是分别通过调用egl库函数eglGetDisplay、EGLUtils::selectConfigForNativeWindow、eglCreateWindowSurface和eglCreateContext来获得的。注意，EGLConfig对象config、EGLSurface对象surface和EGLContext对象context都是用来描述EGLDisplay对象display的。有了这些对象之后，就可以调用函数eglMakeCurrent来设置当前EGL库所使用的绘图表面以及绘图上下文。

还有另外一个地方需要注意的是，每一个EGLSurface对象surface有一个关联的ANativeWindow对象。这个ANativeWindow对象是通过函数eglCreateWindowSurface的第三个参数来指定的。在我们这个场景中，这个ANativeWindow对象正好对应于前面所创建的 Surface对象s。每当OpenGL需要绘图的时候，它就会找到前面所设置的绘图表面，即EGLSurface对象surface。有了EGLSurface对象surface之后，就可以找到与它关联的ANativeWindow对象，即Surface对象s。有了Surface对象s之后，就可以通过其内部的Binder代理对象mSurface来请求SurfaceFlinger服务返回帧缓冲区硬件设备的一个图形访问接口。这样，OpenGL最终就可以将要绘制的图形渲染到帧缓冲区硬件设备中去，即显示在实际屏幕上。屏幕的大小，即宽度和高度，可以通过函数eglQuerySurface来获得。

BootAnimation类的成员变量mAndroidAnimation是一个布尔变量。当它的值等于true的时候，那么就说明需要显示的第三个开机画面是Android系统默认的开机动画，否则的话，第三个开机画面就是由用户自定义的开机动画。

自定义的开机动画是由文件USER_BOOTANIMATION_FILE或者文件SYSTEM_BOOTANIMATION_FILE来描述的。只要其中的一个文件存在，那么第三个开机画面就会使用用户自定义的开机动画。USER_BOOTANIMATION_FILE和SYSTEM_BOOTANIMATION_FILE均是一个宏，它们的定义如下所示：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

    #define USER_BOOTANIMATION_FILE "/data/local/bootanimation.zip"  
    #define SYSTEM_BOOTANIMATION_FILE "/system/media/bootanimation.zip"  


这一步执行完成之后，用来显示第三个开机画面的线程的初始化工作就执行完成了，接下来，就会执行这个线程的主体函数，即BootAnimation类的成员函数threadLoop。

BootAnimation类的成员函数threadLoop的实现如下所示：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

    bool BootAnimation::threadLoop()  
    {  
        bool r;  
        if (mAndroidAnimation) {  
            r = android();  
        } else {  
            r = movie();  
        }  

        eglMakeCurrent(mDisplay, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT);  
        eglDestroyContext(mDisplay, mContext);  
        eglDestroySurface(mDisplay, mSurface);  
        mFlingerSurface.clear();  
        mFlingerSurfaceControl.clear();  
        eglTerminate(mDisplay);  
        IPCThreadState::self()->stopProcess();  
        return r;  
    }  

如果BootAnimation类的成员变量mAndroidAnimation的值等于true，那么接下来就会调用BootAnimation类的成员函数android来显示系统默认的开机动画，否则的话，就会调用BootAnimation类的成员函数movie来显示用户自定义的开机动画。显示完成之后，就会销毁前面所创建的EGLContext对象mContext、EGLSurface对象mSurface，以及EGLDisplay对象mDisplay等。

接下来，我们就分别分析BootAnimation类的成员函数android和movie的实现。

BootAnimation类的成员函数android的实现如下所示：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

    bool BootAnimation::android()  
    {  
        initTexture(&mAndroid[0], mAssets, "images/android-logo-mask.png");  
        initTexture(&mAndroid[1], mAssets, "images/android-logo-shine.png");  

        // clear screen  
        glShadeModel(GL_FLAT);  
        glDisable(GL_DITHER);  
        glDisable(GL_SCISSOR_TEST);  
        glClear(GL_COLOR_BUFFER_BIT);  
        eglSwapBuffers(mDisplay, mSurface);  

        glEnable(GL_TEXTURE_2D);  
        glTexEnvx(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_REPLACE);  

        const GLint xc = (mWidth  - mAndroid[0].w) / 2;  
        const GLint yc = (mHeight - mAndroid[0].h) / 2;  
        const Rect updateRect(xc, yc, xc + mAndroid[0].w, yc + mAndroid[0].h);  

        // draw and update only what we need  
        mFlingerSurface->setSwapRectangle(updateRect);  

        glScissor(updateRect.left, mHeight - updateRect.bottom, updateRect.width(),  
                updateRect.height());  

        // Blend state  
        glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);  
        glTexEnvx(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_REPLACE);  

        const nsecs_t startTime = systemTime();  
        do {  
            nsecs_t now = systemTime();  
            double time = now - startTime;  
            float t = 4.0f * float(time / us2ns(16667)) / mAndroid[1].w;  
            GLint offset = (1 - (t - floorf(t))) * mAndroid[1].w;  
            GLint x = xc - offset;  

            glDisable(GL_SCISSOR_TEST);  
            glClear(GL_COLOR_BUFFER_BIT);  

            glEnable(GL_SCISSOR_TEST);  
            glDisable(GL_BLEND);  
            glBindTexture(GL_TEXTURE_2D, mAndroid[1].name);  
            glDrawTexiOES(x,                 yc, 0, mAndroid[1].w, mAndroid[1].h);  
            glDrawTexiOES(x + mAndroid[1].w, yc, 0, mAndroid[1].w, mAndroid[1].h);  

            glEnable(GL_BLEND);  
            glBindTexture(GL_TEXTURE_2D, mAndroid[0].name);  
            glDrawTexiOES(xc, yc, 0, mAndroid[0].w, mAndroid[0].h);  

            EGLBoolean res = eglSwapBuffers(mDisplay, mSurface);  
            if (res == EGL_FALSE) {  
                break;  
            }  

            // 12fps: don't animate too fast to preserve CPU  
            const nsecs_t sleepTime = 83333 - ns2us(systemTime() - now);  
            if (sleepTime > 0)  
                usleep(sleepTime);  
        } while (!exitPending());  

        glDeleteTextures(1, &mAndroid[0].name);  
        glDeleteTextures(1, &mAndroid[1].name);  
        return false;  
    }  

Android系统默认的开机动画是由两张图片android-logo-mask.png和android-logo-shine.png中。这两张图片保存在frameworks/base/core/res/assets/images目录中，它们最终会被编译在framework-res模块（frameworks/base/core/res）中，即编译在framework-res.apk文件中。编译在framework-res模块中的资源文件可以通过AssetManager类来访问。

BootAnimation类的成员函数android首先调用另外一个成员函数initTexture来将根据图片android-logo-mask.png和android-logo-shine.png的内容来分别创建两个纹理对象，这两个纹理对象就分别保存在BootAnimation类的成员变量mAndroid所描述的一个数组中。通过混合渲染这两个纹理对象，我们就可以得到一个开机动画，这是通过中间的while循环语句来实现的。

图片android-logo-mask.png用作动画前景，它是一个镂空的“ANDROID”图像。图片android-logo-shine.png用作动画背景，它的中间包含有一个高亮的呈45度角的条纹。在每一次循环中，图片android-logo-shine.png被划分成左右两部分内容来显示。左右两个部分的图像宽度随着时间的推移而此消彼长，这样就可以使得图片android-logo-shine.png中间高亮的条纹好像在移动一样。另一方面，在每一次循环中，图片android-logo-shine.png都作为一个整体来渲染，而且它的位置是恒定不变的。由于它是一个镂空的“ANDROID”图像，因此，我们就可以通过它的镂空来看到它背后的图片android-logo-shine.png的条纹一闪一闪地划过。

这个while循环语句会一直被执行，直到应用程序/system/bin/bootanimation被结束为止，后面我们再分析。

BootAnimation类的成员函数movie的实现比较长，我们分段来阅读：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

    bool BootAnimation::movie()  
    {  
        ZipFileRO& zip(mZip);  

        size_t numEntries = zip.getNumEntries();  
        ZipEntryRO desc = zip.findEntryByName("desc.txt");  
        FileMap* descMap = zip.createEntryFileMap(desc);  
        LOGE_IF(!descMap, "descMap is null");  
        if (!descMap) {  
            return false;  
        }  

        String8 desString((char const*)descMap->getDataPtr(),  
                descMap->getDataLength());  
        char const* s = desString.string();  

        Animation animation;  

        // Parse the description file  
        for (;;) {  
            const char* endl = strstr(s, "\n");  
            if (!endl) break;  
            String8 line(s, endl - s);  
            const char* l = line.string();  
            int fps, width, height, count, pause;  
            char path[256];  
            if (sscanf(l, "%d %d %d", &width, &height, &fps) == 3) {  
                //LOGD("> w=%d, h=%d, fps=%d", fps, width, height);  
                animation.width = width;  
                animation.height = height;  
                animation.fps = fps;  
            }  
            if (sscanf(l, "p %d %d %s", &count, &pause, path) == 3) {  
                //LOGD("> count=%d, pause=%d, path=%s", count, pause, path);  
                Animation::Part part;  
                part.count = count;  
                part.pause = pause;  
                part.path = path;  
                animation.parts.add(part);  
            }  
            s = ++endl;  
        }  


从前面BootAnimation类的成员函数readyToRun的实现可以知道，如果目标设备上存在压缩文件/data/local/bootanimation.zip，那么BootAnimation类的成员变量mZip就会指向它，否则的话，就会指向目标设备上的压缩文件/system/media/bootanimation.zip。无论BootAnimation类的成员变量mZip指向的是哪一个压缩文件，这个压缩文件都必须包含有一个名称为“desc.txt”的文件，用来描述用户自定义的开机动画是如何显示的。

	文件desc.txt的内容格式如下面的例子所示：

[html] view plain copy
在CODE上查看代码片派生到我的代码片

    600 480 24  
    p   1   0   part1  
    p   0   10  part2  

第一行的三个数字分别表示开机动画在屏幕中的显示宽度、高度以及帧速（fps）。剩余的每一行都用来描述一个动画片断，这些行必须要以字符“p”来开头，后面紧跟着两个数字以及一个文件目录路径名称。第一个数字表示一个片断的循环显示次数，如果它的值等于0，那么就表示无限循环地显示该动画片断。第二个数字表示每一个片断在两次循环显示之间的时间间隔。这个时间间隔是以一个帧的时间为单位的。文件目录下面保存的是一系列png文件，这些png文件会被依次显示在屏幕中。

以上面这个desct.txt文件的内容为例，它描述了一个大小为600 x 480的开机动画，动画的显示速度为24帧每秒。这个开机动画包含有两个片断part1和part2。片断part1只显示一次，它对应的png图片保存在目录part1中。片断part2无限循环地显示，其中，每两次循环显示的时间间隔为10 x (1 / 24)秒，它对应的png图片保存在目录part2中。

上面的for循环语句分析完成desc.txt文件的内容后，就得到了开机动画的显示大小、速度以及片断信息。这些信息都保存在Animation对象animation中，其中，每一个动画片断都使用一个Animation::Part对象来描述，并且保存在Animation对象animation的成员变量parts所描述的一个片断列表中。

接下来，BootAnimation类的成员函数movie再断续将每一个片断所对应的png图片读取出来，如下所示：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

    // read all the data structures  
    const size_t pcount = animation.parts.size();  
    for (size_t i=0 ; i<numEntries ; i++) {  
        char name[256];  
        ZipEntryRO entry = zip.findEntryByIndex(i);  
        if (zip.getEntryFileName(entry, name, 256) == 0) {  
            const String8 entryName(name);  
            const String8 path(entryName.getPathDir());  
            const String8 leaf(entryName.getPathLeaf());  
            if (leaf.size() > 0) {  
                for (int j=0 ; j<pcount ; j++) {  
                    if (path == animation.parts[j].path) {  
                        int method;  
                        // supports only stored png files  
                        if (zip.getEntryInfo(entry, &method, 0, 0, 0, 0, 0)) {  
                            if (method == ZipFileRO::kCompressStored) {  
                                FileMap* map = zip.createEntryFileMap(entry);  
                                if (map) {  
                                    Animation::Frame frame;  
                                    frame.name = leaf;  
                                    frame.map = map;  
                                    Animation::Part& part(animation.parts.editItemAt(j));  
                                    part.frames.add(frame);  
                                }  
                            }  
                        }  
                    }  
                }  
            }  
        }  
    }  


每一个png图片都表示一个动画帧，使用一个Animation::Frame对象来描述，并且保存在对应的Animation::Part对象的成员变量frames所描述的一个帧列表中。

获得了开机动画的所有信息之后，接下来BootAnimation类的成员函数movie就准备开始显示开机动画了，如下所示：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

    // clear screen  
    glShadeModel(GL_FLAT);  
    glDisable(GL_DITHER);  
    glDisable(GL_SCISSOR_TEST);  
    glDisable(GL_BLEND);  
    glClear(GL_COLOR_BUFFER_BIT);  

    eglSwapBuffers(mDisplay, mSurface);  

    glBindTexture(GL_TEXTURE_2D, 0);  
    glEnable(GL_TEXTURE_2D);  
    glTexEnvx(GL_TEXTURE_ENV, GL_TEXTURE_ENV_MODE, GL_REPLACE);  
    glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);  
    glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);  
    glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);  
    glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);  

    const int xc = (mWidth - animation.width) / 2;  
    const int yc = ((mHeight - animation.height) / 2);  
    nsecs_t lastFrame = systemTime();  
    nsecs_t frameDuration = s2ns(1) / animation.fps;  

    Region clearReg(Rect(mWidth, mHeight));  
    clearReg.subtractSelf(Rect(xc, yc, xc+animation.width, yc+animation.height));  


前面的一系列gl函数首先用来清理屏幕，接下来的一系列gl函数用来设置OpenGL的纹理显示方式。

变量xc和yc的值用来描述开机动画的显示位置，即需要在屏幕中间显示开机动画，另外一个变量frameDuration的值用来描述每一帧的显示时间，它是以纳秒为单位的。

Region对象clearReg用来描述屏幕中除了开机动画之外的其它区域，它是用整个屏幕区域减去开机动画所点据的区域来得到的。

准备好开机动画的显示参数之后，最后就可以执行显示开机动画的操作了，如下所示：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

        for (int i=0 ; i<pcount && !exitPending() ; i++) {  
            const Animation::Part& part(animation.parts[i]);  
            const size_t fcount = part.frames.size();  
            glBindTexture(GL_TEXTURE_2D, 0);  

            for (int r=0 ; !part.count || r<part.count ; r++) {  
                for (int j=0 ; j<fcount && !exitPending(); j++) {  
                    const Animation::Frame& frame(part.frames[j]);  

                    if (r > 0) {  
                        glBindTexture(GL_TEXTURE_2D, frame.tid);  
                    } else {  
                        if (part.count != 1) {  
                            glGenTextures(1, &frame.tid);  
                            glBindTexture(GL_TEXTURE_2D, frame.tid);  
                            glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);  
                            glTexParameterx(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);  
                        }  
                        initTexture(  
                                frame.map->getDataPtr(),  
                                frame.map->getDataLength());  
                    }  

                    if (!clearReg.isEmpty()) {  
                        Region::const_iterator head(clearReg.begin());  
                        Region::const_iterator tail(clearReg.end());  
                        glEnable(GL_SCISSOR_TEST);  
                        while (head != tail) {  
                            const Rect& r(*head++);  
                            glScissor(r.left, mHeight - r.bottom,  
                                    r.width(), r.height());  
                            glClear(GL_COLOR_BUFFER_BIT);  
                        }  
                        glDisable(GL_SCISSOR_TEST);  
                    }  
                    glDrawTexiOES(xc, yc, 0, animation.width, animation.height);  
                    eglSwapBuffers(mDisplay, mSurface);  

                    nsecs_t now = systemTime();  
                    nsecs_t delay = frameDuration - (now - lastFrame);  
                    lastFrame = now;  
                    long wait = ns2us(frameDuration);  
                    if (wait > 0)  
                        usleep(wait);  
                }  
                usleep(part.pause * ns2us(frameDuration));  
            }  

            // free the textures for this part  
            if (part.count != 1) {  
                for (int j=0 ; j<fcount ; j++) {  
                    const Animation::Frame& frame(part.frames[j]);  
                    glDeleteTextures(1, &frame.tid);  
                }  
            }  
        }  

        return false;  
    }  


第一层for循环用来显示每一个动画片断，第二层的for循环用来循环显示每一个动画片断，第三层的for循环用来显示每一个动画片断所对应的png图片。这些png图片以纹理的方式来显示在屏幕中。

注意，如果一个动画片断的循环显示次数不等于1，那么就说明这个动画片断中的png图片需要重复地显示在屏幕中。由于每一个png图片都需要转换为一个纹理对象之后才能显示在屏幕中，因此，为了避免重复地为同一个png图片创建纹理对象，第三层的for循环在第一次显示一个png图片的时候，会调用函数glGenTextures来为这个png图片创建一个纹理对象，并且将这个纹理对象的名称保存在对应的Animation::Frame对象的成员变量tid中，这样，下次再显示相同的图片时，就可以使用前面已经创建好了的纹理对象，即调用函数glBindTexture来指定当前要操作的纹理对象。

如果Region对象clearReg所包含的区域不为空，那么在调用函数glDrawTexiOES和eglSwapBuffers来显示每一个png图片之前，首先要将它所包含的区域裁剪掉，避免开机动画可以显示在指定的位置以及大小中。

每当显示完成一个png图片之后，都要将变量frameDuration的值从纳秒转换为毫秒。如果转换后的值大小于，那么就需要调用函数usleep函数来让线程睡眠一下，以保证每一个png图片，即每一帧动画都按照预先指定好的速度来显示。注意，函数usleep指定的睡眠时间只能精确到毫秒，因此，如果预先指定的帧显示时间小于1毫秒，那么BootAnimation类的成员函数movie是无法精确地控制地每一帧的显示时间的。

还有另外一个地方需要注意的是，每当循环显示完成一个片断时，需要调用usleep函数来使得线程睡眠part.pause * ns2us(frameDuration)毫秒，以便可以按照预先设定的节奏来显示开机动画。

最后一个if语句判断一个动画片断是否是循环显示的，即循环次数不等于1。如果是的话，那么就说明前面为它所对应的每一个png图片都创建过一个纹理对象。现在既然这个片断的显示过程已经结束了，因此，就需要释放前面为它所创建的纹理对象。

至此，第三个开机画面的显示过程就分析完成了。

##接下来，我们再继续分析第三个开机画面是如何停止显示的。


 ActivityManagerService类有一个类型为ActivityStack的成员变量mMainStack，它用来描述系统的Activity组件堆栈，它的成员函数activityIdleInternal的实现如下所示：

    public class ActivityStack {  
        ......  

        final void activityIdleInternal(IBinder token, boolean fromTimeout,  
                Configuration config) {  
            ......  

            boolean enableScreen = false;  

            synchronized (mService) {  
                ......  

                // Get the activity record.  
                int index = indexOfTokenLocked(token);  
                if (index >= 0) {  
                    ActivityRecord r = (ActivityRecord)mHistory.get(index);                  
                    ......  

                    if (mMainStack) {  
                        if (!mService.mBooted && !fromTimeout) {  
                            mService.mBooted = true;  
                            enableScreen = true;  
                        }  
                    }  
                }  

                ......  
            }  

            ......  

            if (enableScreen) {  
                mService.enableScreenAfterBoot();  
            }  
        }  

        ......  
    }          



参数token用来描述刚刚启动起来的Launcher组件，通过它来调用函数indexOfTokenLocked就可以得到Launcher组件在系统Activity组件堆栈中的位置index。得到了Launcher组件在系统Activity组件堆栈中的位置index之后，就可以在ActivityStack类的成员变量mHistory中得到一个ActivityRecord对象r。这个ActivityRecord对象r同样是用来描述Launcher组件的。

ActivityStack类的成员变量mMainStack是一个布尔变量，当它的值等于true的时候，就说明当前正在处理的ActivityStack对象是用来描述系统的Activity组件堆栈的。 ActivityStack类的另外一个成员变量mService指向了系统中的ActivityManagerService服务。ActivityManagerService服务有一个类型为布尔值的成员变量mBooted，它的初始值为false，表示系统尚未启动完成。

从前面的调用过程可以知道，参数fromTimeout的值等于false。在这种情况下，如果ActivityManagerService服务的成员变量mBooted也等于false，那么就说明应用程序已经启动起来了，即说明系统已经启动完成了。这时候ActivityManagerService服务的成员变量mBooted以及变量enableScreen的值就会被设置为true。

当变量enableScreen的值等于true的时候，ActivityStack类就会调用ActivityManagerService服务的成员函数enableScreenAfterBoot停止显示开机动画，以便可以将屏幕让出来显示应用程序Launcher的界面。

ActivityManagerService类的成员函数enableScreenAfterBoot的实现如下所示：

    public final class ActivityManagerService extends ActivityManagerNative  
            implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
        ......  

        void enableScreenAfterBoot() {  
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_ENABLE_SCREEN,  
                    SystemClock.uptimeMillis());  
            mWindowManager.enableScreenAfterBoot();  
        }  

        ......  
    }  


ActivityManagerService类的成员变量mWindowManager指向了系统中的Window管理服务WindowManagerService，ActivityManagerService服务通过调用它的成员函数enableScreenAfterBoot来停止显示开机动画。

WindowManagerService类的成员函数enableScreenAfterBoot的实现如下所示：


    public class WindowManagerService extends IWindowManager.Stub  
            implements Watchdog.Monitor {  
        ......  

        public void enableScreenAfterBoot() {  
            synchronized(mWindowMap) {  
                if (mSystemBooted) {  
                    return;  
                }  
                mSystemBooted = true;  
            }  

            performEnableScreen();  
        }  

        ......  
    }  


 WindowManagerService类的成员变量mSystemBooted用来记录系统是否已经启动完成的。如果已经启动完成的话，那么这个成员变量的值就会等于true，这时候WindowManagerService类的成员函数enableScreenAfterBoot什么也不做就返回了，否则的话，WindowManagerService类的成员函数enableScreenAfterBoot首先将这个成员变量的值设置为true，接着再调用另外一个成员函数performEnableScreen来执行停止显示开机动画的操作。

WindowManagerService类的成员函数performEnableScreen的实现如下所示：


    public class WindowManagerService extends IWindowManager.Stub  
            implements Watchdog.Monitor {  
        ......  

        public void performEnableScreen() {  
            synchronized(mWindowMap) {  
                if (mDisplayEnabled) {  
                    return;  
                }  
                if (!mSystemBooted) {  
                    return;  
                }  

                ......  

                mDisplayEnabled = true;  
                ......  

                try {  
                    IBinder surfaceFlinger = ServiceManager.getService("SurfaceFlinger");  
                    if (surfaceFlinger != null) {  
                        //Slog.i(TAG, "******* TELLING SURFACE FLINGER WE ARE BOOTED!");  
                        Parcel data = Parcel.obtain();  
                        data.writeInterfaceToken("android.ui.ISurfaceComposer");  
                        surfaceFlinger.transact(IBinder.FIRST_CALL_TRANSACTION,  
                                                data, null, 0);  
                        data.recycle();  
                    }  
                } catch (RemoteException ex) {  
                    Slog.e(TAG, "Boot completed: SurfaceFlinger is dead!");  
                }  
            }  

            ......  
        }  

        ......  
    }  


WindowManagerService类的另外一个成员变量mDisplayEnabled用来描述WindowManagerService是否已经初始化过系统的屏幕了，只有当它的值等于false，并且系统已经完成启动，即WindowManagerService类的成员变量mSystemBooted等于true的情况下，WindowManagerService类的成员函数performEnableScreen才通知SurfaceFlinger服务停止显示开机动画。

注意，WindowManagerService类的成员函数performEnableScreen是通过一个类型为IBinder.FIRST_CALL_TRANSACTION的进程间通信请求来通知SurfaceFlinger服务停止显示开机动画的。

在SurfaceFlinger服务，类型为IBinder.FIRST_CALL_TRANSACTION的进程间通信请求被定义为停止显示开机动画的请求，如下所示：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

    class BnSurfaceComposer : public BnInterface<ISurfaceComposer>  
    {  
    public:  
        enum {  
            // Note: BOOT_FINISHED must remain this value, it is called from  
            // Java by ActivityManagerService.  
            BOOT_FINISHED = IBinder::FIRST_CALL_TRANSACTION,  
            ......  
        };  

        virtual status_t    onTransact( uint32_t code,  
                                        const Parcel& data,  
                                        Parcel* reply,  
                                        uint32_t flags = 0);  
    };  

BnSurfaceComposer类定义在文件frameworks/base/include/surfaceflinger/ISurfaceComposer.h中，它是SurfaceFlinger服务所要继承的Binder本地对象类，其中。当SurfaceFlinger服务接收到类型为IBinder::FIRST_CALL_TRANSACTION，即类型为BOOT_FINISHED的进程间通信请求时，它就会将该请求交给它的成员函数bootFinished来处理。

SurfaceFlinger服务的成员函数bootFinished实现在文件frameworks/base/services/surfaceflinger/SurfaceFlinger.cpp中，如下所示：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

    void SurfaceFlinger::bootFinished()  
    {  
        const nsecs_t now = systemTime();  
        const nsecs_t duration = now - mBootTime;  
        LOGI("Boot is finished (%ld ms)", long(ns2ms(duration)) );  
        mBootFinished = true;  
        property_set("ctl.stop", "bootanim");  
    }  

这个函数主要就是将系统属性“ctl.stop”的值设置为“bootanim”。前面提到，每当有一个系统属性发生变化时，init进程就会被唤醒，并且调用运行在它里面的函数handle_property_set_fd来处理这个系统属性变化事件。在我们这个场景中，由于被改变的系统属性的名称是以"ctl."开头的，即被改变的系统属性是一个控制类型的属性，因此，接下来函数handle_property_set_fd又会调用另外一个函数handle_control_message来处理该系统属性变化事件。

函数handle_control_message实现在文件system/core/init/init.c中，如下所示：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

    void handle_control_message(const char *msg, const char *arg)  
    {  
        if (!strcmp(msg,"start")) {  
            msg_start(arg);  
        } else if (!strcmp(msg,"stop")) {  
            msg_stop(arg);  
        } else {  
            ERROR("unknown control msg '%s'\n", msg);  
        }  
    }  

从前面的调用过程可以知道，参数msg和arg的值分别等于"stop"和“bootanim”，这表示要停止执行名称为“bootanim”的服务，这是通过调用函数msg_stop来实现的。   

函数msg_stop也是实现在文件system/core/init/init.c中，如下所示：

[cpp] view plain copy
在CODE上查看代码片派生到我的代码片

    static void msg_stop(const char *name)  
    {  
        struct service *svc = service_find_by_name(name);  

        if (svc) {  
            service_stop(svc);  
        } else {  
            ERROR("no such service '%s'\n", name);  
        }  
    }  

这个函数首先调用函数service_find_by_name来找到名称等于name，即“bootanim”的服务，然后再调用函数service_stop来停止这个服务。

前面提到，名称为“bootanim”的服务对应的应用程序即为/system/bin/bootanimation。因此，停止名称为“bootanim”的服务即为停止执行应用程序/system/bin/bootanimation，而当应用程序/system/bin/bootanimation停止执行的时候，开机动画就会停止显示了。


至此，Android系统的三个开机画面的显示过程就分析完成了。通过这个三个开机画面的显示过程分析，我们学习到：

+ 在内核层，系统屏幕是使用一个称为帧缓冲区的硬件设备来描述的，而用户空间的应用程序可以通过设备文件/dev/fb0或者/dev/graphics/fb0来操作这个硬件设备。实际上，帧缓冲区本身并不是一个真正的硬件，它只不过是对显卡的一个抽象表示，不过，我们通过访帧缓冲区就可以间接地操作显卡内存以及显卡中的其它寄存器。

+ OpenGL是通过EGL接口来渲染屏幕，而EGL接口是通过ANativeWindow类来间接地渲染屏幕的。我们可以将ANativeWindow类理解成一个Android系统的本地窗口类，即相当于是Windows系统中的窗口句柄概念，它最终是通过文件/dev/fb0或者/dev/graphics/fb0来渲染屏幕的。

+ init进程在启动的过程中，会将另外一个ueventd进程也启动起来。ueventd进程对应的可执行文件与init进程对应的可执行文件均为/init，不过ueventd进程主要负责处理内核发出的uevent事件，即负责管理系统中的设备文件。

+ 每当我们设置一个系统属性的时候，init进程都会接收到一个系统属性变化事件。当发生变化的系统属性的名称等于“ctl.start”或者“ctl.stop”，那么实际上是向init进程发出一个启动或者停止服务的命令。

前面第1点和第2点的知识是与Android系统的UI实现相关的，而后面第3点和第4点是两个额外获得的知识点。

本文的目的并不是单纯为了介绍Android系统的开机画面，而是希望能够以Android系统的开机画面来作为切入点来分析Android系统的UI实现。在后面的文章中，我们就会根据本文所涉及到的知识点，来展开分析Android系统的UI实现，敬请关注。
