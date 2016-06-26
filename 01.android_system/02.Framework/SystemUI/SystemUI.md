#System UI 分析


##相关资料索引

[android 6.0 SystemUI源码分析(3)-Recent Panel加载显示流程](http://blog.csdn.net/zhudaozhuan/article/details/50819499)







##System UI 包含哪些component

####Shell

+ System UI Service 
			
			
			26public class SystemUIService extends Service {
			27
			28    @Override
			29    public void onCreate() {
			30        super.onCreate();
			31        ((SystemUIApplication) getApplication()).startServicesIfNeeded();
			32    }
			33
			34    @Override
			35    public IBinder onBind(Intent intent) {
			36        return null;
			37    }
			38
			39    @Override
			40    protected void dump(FileDescriptor fd, PrintWriter pw, String[] args) {
			41        SystemUI[] services = ((SystemUIApplication) getApplication()).getServices();
			42        if (args == null || args.length == 0) {
			43            for (SystemUI ui: services) {
			44                pw.println("dumping service: " + ui.getClass().getName());
			45                ui.dump(fd, pw, args);
			46            }
			47        } else {
			48            String svc = args[0];
			49            for (SystemUI ui: services) {
			50                String name = ui.getClass().getName();
			51                if (name.endsWith(svc)) {
			52                    ui.dump(fd, pw, args);
			53                }
			54            }
			55        }
			56    }
			57}



+ System UI application

		
		34public class SystemUIApplication extends Application {
		35
		36    private static final String TAG = "SystemUIService";
		37    private static final boolean DEBUG = false;
		38
		39    /**
		40     * The classes of the stuff to start.
		41     */
		42    private final Class<?>[] SERVICES = new Class[] {
		43            com.android.systemui.tuner.TunerService.class,
		44            com.android.systemui.keyguard.KeyguardViewMediator.class,
		45            com.android.systemui.recents.Recents.class,
		46            com.android.systemui.volume.VolumeUI.class,
		47            com.android.systemui.statusbar.SystemBars.class,
		48            com.android.systemui.usb.StorageNotification.class,
		49            com.android.systemui.power.PowerUI.class,
		50            com.android.systemui.media.RingtonePlayer.class,
		51            com.android.systemui.keyboard.KeyboardUI.class,
		52    };
		53
		54    /**
		55     * Hold a reference on the stuff we start.
		56     */
		57    private final SystemUI[] mServices = new SystemUI[SERVICES.length];
		58    private boolean mServicesStarted;
		59    private boolean mBootCompleted;
		60    private final Map<Class<?>, Object> mComponents = new HashMap<Class<?>, Object>();
		61
		62    @Override
		63    public void onCreate() {
		64        super.onCreate();
		65        // Set the application theme that is inherited by all services. Note that setting the
		66        // application theme in the manifest does only work for activities. Keep this in sync with
		67        // the theme set there.
		68        setTheme(R.style.systemui_theme);
		69
		70        IntentFilter filter = new IntentFilter(Intent.ACTION_BOOT_COMPLETED);
		71        filter.setPriority(IntentFilter.SYSTEM_HIGH_PRIORITY);
		72        registerReceiver(new BroadcastReceiver() {
		73            @Override
		74            public void onReceive(Context context, Intent intent) {
		75                if (mBootCompleted) return;
		76
		77                if (DEBUG) Log.v(TAG, "BOOT_COMPLETED received");
		78                unregisterReceiver(this);
		79                mBootCompleted = true;
		80                if (mServicesStarted) {
		81                    final int N = mServices.length;
		82                    for (int i = 0; i < N; i++) {
		83                        mServices[i].onBootCompleted();
		84                    }
		85                }
		86            }
		87        }, filter);
		88    }
		89
		90    /**
		91     * Makes sure that all the SystemUI services are running. If they are already running, this is a
		92     * no-op. This is needed to conditinally start all the services, as we only need to have it in
		93     * the main process.
		94     *
		95     * <p>This method must only be called from the main thread.</p>
		96     */
		97    public void startServicesIfNeeded() {
		98        if (mServicesStarted) {
		99            return;
		100        }
		101
		102        if (!mBootCompleted) {
		103            // check to see if maybe it was already completed long before we began
		104            // see ActivityManagerService.finishBooting()
		105            if ("1".equals(SystemProperties.get("sys.boot_completed"))) {
		106                mBootCompleted = true;
		107                if (DEBUG) Log.v(TAG, "BOOT_COMPLETED was already sent");
		108            }
		109        }
		110
		111        Log.v(TAG, "Starting SystemUI services.");
		112        final int N = SERVICES.length;
		113        for (int i=0; i<N; i++) {
		114            Class<?> cl = SERVICES[i];
		115            if (DEBUG) Log.d(TAG, "loading: " + cl);
		116            try {
		117                mServices[i] = (SystemUI)cl.newInstance();
		118            } catch (IllegalAccessException ex) {
		119                throw new RuntimeException(ex);
		120            } catch (InstantiationException ex) {
		121                throw new RuntimeException(ex);
		122            }
		123            mServices[i].mContext = this;
		124            mServices[i].mComponents = mComponents;
		125            if (DEBUG) Log.d(TAG, "running: " + mServices[i]);
		126            mServices[i].start();
		127
		128            if (mBootCompleted) {
		129                mServices[i].onBootCompleted();
		130            }
		131        }
		132        mServicesStarted = true;
		133    }
		134
		135    @Override
		136    public void onConfigurationChanged(Configuration newConfig) {
		137        if (mServicesStarted) {
		138            int len = mServices.length;
		139            for (int i = 0; i < len; i++) {
		140                mServices[i].onConfigurationChanged(newConfig);
		141            }
		142        }
		143    }
		144
		145    @SuppressWarnings("unchecked")
		146    public <T> T getComponent(Class<T> interfaceType) {
		147        return (T) mComponents.get(interfaceType);
		148    }
		149
		150    public SystemUI[] getServices() {
		151        return mServices;
		152    }
		153}


####Panel
+ System UI class
	+ Recents
	+ SystemBar
	+ VolumeUI
	+ KeyguardViewMediator
	+ TunerService
	+ PowerUI
	
	
			
		42    private final Class<?>[] SERVICES = new Class[] {
		43            com.android.systemui.tuner.TunerService.class,
		44            com.android.systemui.keyguard.KeyguardViewMediator.class,
		45            com.android.systemui.recents.Recents.class,
		46            com.android.systemui.volume.VolumeUI.class,
		47            com.android.systemui.statusbar.SystemBars.class,
		48            com.android.systemui.usb.StorageNotification.class,
		49            com.android.systemui.power.PowerUI.class,
		50            com.android.systemui.media.RingtonePlayer.class,
		51            com.android.systemui.keyboard.KeyboardUI.class,
		52    };
		53

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


	
	1223    static final void startSystemUi(Context context) {
	1224        Intent intent = new Intent();
	1225        intent.setComponent(new ComponentName("com.android.systemui",
	1226                    "com.android.systemui.SystemUIService"));
	1227        //Slog.d(TAG, "Starting service: " + intent);
	1228        context.startServiceAsUser(intent, UserHandle.OWNER);
	1229    }


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






