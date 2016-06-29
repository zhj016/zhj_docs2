#Android系统默认Home应用程序（Launcher）的启动过程源代码分析


 Android系统的Home应用程序Launcher是由ActivityManagerService启动的，而ActivityManagerService和PackageManagerService一样，都是在开机时由SystemServer组件启动的，SystemServer组件首先是启动ePackageManagerServic，由它来负责安装系统的应用程序，具体可以参考前面一篇文章Android应用程序安装过程源代码分析，系统中的应用程序安装好了以后，SystemServer组件接下来就要通过ActivityManagerService来启动Home应用程序Launcher了，Launcher在启动的时候便会通过PackageManagerServic把系统中已经安装好的应用程序以快捷图标的形式展示在桌面上，这样用户就可以使用这些应用程序了，整个过程如下图所示：





##ActivityStack.java

resumeTopActivityLocked, 这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityStack.java文件中：

    public class ActivityStack {  
        ......  

        final boolean resumeTopActivityLocked(ActivityRecord prev) {  
            // Find the first activity that is not finishing.  
            ActivityRecord next = topRunningActivityLocked(null);  

            ......  

            if (next == null) {  
                // There are no more activities!  Let's just start up the  
                // Launcher...  
                if (mMainStack) {  
                    return mService.startHomeActivityLocked();  
                }  
            }  

            ......  
        }  

        ......  
    }  



这里调用函数topRunningActivityLocked返回的是当前系统Activity堆栈最顶端的Activity，由于此时还没有Activity被启动过，因此，返回值为null，即next变量的值为null，于是就调用mService.startHomeActivityLocked语句，这里的mService就是前面在Step 7中创建的ActivityManagerService实例了。



 ##Step 12. ActivityManagerService.startHomeActivityLocked

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityManagerServcie.java文件中：


      public final class ActivityManagerService extends ActivityManagerNative  
              implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {  
          ......  

          boolean startHomeActivityLocked() {  
              ......  

              Intent intent = new Intent(  
                  mTopAction,  
                  mTopData != null ? Uri.parse(mTopData) : null);  
              intent.setComponent(mTopComponent);  
              if (mFactoryTest != SystemServer.FACTORY_TEST_LOW_LEVEL) {  
                  intent.addCategory(Intent.CATEGORY_HOME);  
              }  
              ActivityInfo aInfo =  
                  intent.resolveActivityInfo(mContext.getPackageManager(),  
                  STOCK_PM_FLAGS);  
              if (aInfo != null) {  
                  intent.setComponent(new ComponentName(  
                      aInfo.applicationInfo.packageName, aInfo.name));  
                  // Don't do this if the home app is currently being  
                  // instrumented.  
                  ProcessRecord app = getProcessRecordLocked(aInfo.processName,  
                      aInfo.applicationInfo.uid);  
                  if (app == null || app.instrumentationClass == null) {  
                      intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);  
                      mMainStack.startActivityLocked(null, intent, null, null, 0, aInfo,  
                          null, null, 0, 0, 0, false, false);  
                  }  
              }  

              return true;  
          }  

          ......  
      }  


函数首先创建一个CATEGORY_HOME类型的Intent，然后通过Intent.resolveActivityInfo函数向PackageManagerService查询Category类型为HOME的Activity，这里我们假设只有系统自带的Launcher应用程序注册了HOME类型的Activity（见packages/apps/Launcher2/AndroidManifest.xml文件）：


      <manifest  
          xmlns:android="http://schemas.android.com/apk/res/android"  
          package="com.android.launcher"  
          android:sharedUserId="@string/sharedUserId"  
          >  

          ......  

          <application  
              android:name="com.android.launcher2.LauncherApplication"  
              android:process="@string/process"  
              android:label="@string/application_name"  
              android:icon="@drawable/ic_launcher_home">  

              <activity  
                  android:name="com.android.launcher2.Launcher"  
                  android:launchMode="singleTask"  
                  android:clearTaskOnLaunch="true"  
                  android:stateNotNeeded="true"  
                  android:theme="@style/Theme"  
                  android:screenOrientation="nosensor"  
                  android:windowSoftInputMode="stateUnspecified|adjustPan">  
                  <intent-filter>  
                      <action android:name="android.intent.action.MAIN" />  
                      <category android:name="android.intent.category.HOME" />  
                      <category android:name="android.intent.category.DEFAULT" />  
                      <category android:name="android.intent.category.MONKEY"/>  
                      </intent-filter>  
              </activity>  

              ......  
          </application>  
      </manifest>  



  因此，这里就返回com.android.launcher2.Launcher这个Activity了。由于是第一次启动这个Activity，接下来调用函数getProcessRecordLocked返回来的ProcessRecord值为null，于是，就调用mMainStack.startActivityLocked函数启动com.android.launcher2.Launcher这个Activity了，这里的mMainStack是一个ActivityStack类型的成员变量。


##Step 13.  ActivityStack.startActivityLocked

这个函数定义在frameworks/base/services/java/com/android/server/am/ActivityStack.java文件中，具体可以参考Android应用程序启动过程源代码分析一文，这里就不详述了，在我们这个场景中，调用这个函数的最后结果就是把com.android.launcher2.Launcher启动起来，接着调用它的onCreate函数。



## Step 14. Launcher.onCreate
这个函数定义在packages/apps/Launcher2/src/com/android/launcher2/Launcher.java文件中：

在CODE上查看代码片派生到我的代码片

    public final class Launcher extends Activity  
            implements View.OnClickListener, OnLongClickListener, LauncherModel.Callbacks, AllAppsView.Watcher {  
        ......  

        @Override  
        protected void onCreate(Bundle savedInstanceState) {  
            ......  

            if (!mRestoring) {  
                mModel.startLoader(this, true);  
            }  

            ......  
        }  

        ......  
    }  

这里的mModel是一个LauncherModel类型的成员变量，这里通过调用它的startLoader成员函数来执行加应用程序的操作。

##Step 15. LauncherModel.startLoader

这个函数定义在packages/apps/Launcher2/src/com/android/launcher2/LauncherModel.java文件中：

      public class LauncherModel extends BroadcastReceiver {  
          ......  

          public void startLoader(Context context, boolean isLaunching) {  
              ......  

                      synchronized (mLock) {  
                           ......  

                           // Don't bother to start the thread if we know it's not going to do anything  
                           if (mCallbacks != null && mCallbacks.get() != null) {  
                               // If there is already one running, tell it to stop.  
                               LoaderTask oldTask = mLoaderTask;  
                               if (oldTask != null) {  
                                   if (oldTask.isLaunching()) {  
                                       // don't downgrade isLaunching if we're already running  
                                       isLaunching = true;  
                                   }  
                                   oldTask.stopLocked();  
                       }  
                       mLoaderTask = new LoaderTask(context, isLaunching);  
                       sWorker.post(mLoaderTask);  
                      }  
                 }  
          }  

          ......  
      }  

这里不是直接加载应用程序，而是把加载应用程序的操作作为一个消息来处理。这里的sWorker是一个Handler，通过它的post方式把一个消息放在消息队列中去，然后系统就会调用传进去的参数mLoaderTask的run函数来处理这个消息，这个mLoaderTask是LoaderTask类型的实例，于是，下面就会执行LoaderTask类的run函数了。



    private static final HandlerThread sWorkerThread = new HandlerThread("launcher-loader");
    static {
        sWorkerThread.start();
    }
    private static final Handler sWorker = new Handler(sWorkerThread.getLooper());


##Step 16. LoaderTask.run

这个函数定义在packages/apps/Launcher2/src/com/android/launcher2/LauncherModel.java文件中：



    public class LauncherModel extends BroadcastReceiver {  
        ......  

        private class LoaderTask implements Runnable {  
            ......  

            public void run() {  
                ......  

                keep_running: {  
                    ......  

                    // second step  
                    if (loadWorkspaceFirst) {  
                        ......  
                        loadAndBindAllApps();  
                    } else {  
                        ......  
                    }  

                    ......  
                }  

                ......  
            }  

            ......  
        }  

        ......  
    }  

这里调用loadAndBindAllApps成员函数来进一步操作。

##Step 17. LoaderTask.loadAndBindAllApps

这个函数定义在packages/apps/Launcher2/src/com/android/launcher2/LauncherModel.java文件中：

    public class LauncherModel extends BroadcastReceiver {  
        ......  

        private class LoaderTask implements Runnable {  
            ......  

            private void loadAndBindAllApps() {  
                ......  

                if (!mAllAppsLoaded) {  
                    loadAllAppsByBatch();  
                    if (mStopped) {  
                        return;  
                    }  
                    mAllAppsLoaded = true;  
                } else {  
                    onlyBindAllApps();  
                }  
            }  


            ......  
        }  

        ......  
    }  

##Step 18. LoaderTask.loadAllAppsByBatch

这个函数定义在packages/apps/Launcher2/src/com/android/launcher2/LauncherModel.java文件中：

    public class LauncherModel extends BroadcastReceiver {  
        ......  

        private class LoaderTask implements Runnable {  
            ......  

            private void loadAllAppsByBatch() {   
                ......  

                final Intent mainIntent = new Intent(Intent.ACTION_MAIN, null);  
                mainIntent.addCategory(Intent.CATEGORY_LAUNCHER);  

                final PackageManager packageManager = mContext.getPackageManager();  
                List<ResolveInfo> apps = null;  

                int N = Integer.MAX_VALUE;  

                int startIndex;  
                int i=0;  
                int batchSize = -1;  
                while (i < N && !mStopped) {  
                    if (i == 0) {  
                        mAllAppsList.clear();  
                        ......  
                        apps = packageManager.queryIntentActivities(mainIntent, 0);  

                        ......  

                        N = apps.size();  

                        ......  

                        if (mBatchSize == 0) {  
                            batchSize = N;  
                        } else {  
                            batchSize = mBatchSize;  
                        }  

                        ......  

                        Collections.sort(apps,  
                            new ResolveInfo.DisplayNameComparator(packageManager));  
                    }  

                    startIndex = i;  
                    for (int j=0; i<N && j<batchSize; j++) {  
                        // This builds the icon bitmaps.  
                        mAllAppsList.add(new ApplicationInfo(apps.get(i), mIconCache));  
                        i++;  
                    }  

                    final boolean first = i <= batchSize;  
                    final Callbacks callbacks = tryGetCallbacks(oldCallbacks);  
                    final ArrayList<ApplicationInfo> added = mAllAppsList.added;  
                    mAllAppsList.added = new ArrayList<ApplicationInfo>();  

                    mHandler.post(new Runnable() {  
                        public void run() {  
                            final long t = SystemClock.uptimeMillis();  
                            if (callbacks != null) {  
                                if (first) {  
                                    callbacks.bindAllApplications(added);  
                                } else {  
                                    callbacks.bindAppsAdded(added);  
                                }  
                                ......  
                            } else {  
                                ......  
                            }  
                        }  
                    });  

                    ......  
                }  

                ......  
            }  

            ......  
        }  

        ......  
    }  

函数首先构造一个CATEGORY_LAUNCHER类型的Intent：


    final Intent mainIntent = new Intent(Intent.ACTION_MAIN, null);  
    mainIntent.addCategory(Intent.CATEGORY_LAUNCHER);  

接着从mContext变量中获得PackageManagerService的接口：

    final PackageManager packageManager = mContext.getPackageManager();  

下一步就是通过这个PackageManagerService.queryIntentActivities接口来取回所有Action类型为Intent.ACTION_MAIN，并且Category类型为Intent.CATEGORY_LAUNCHER的Activity了。

我们先进入到PackageManagerService.queryIntentActivities函数中看看是如何获得这些Activity的，然后再回到这个函数中来看其余操作。

##Step 19. PackageManagerService.queryIntentActivities

这个函数定义在frameworks/base/services/java/com/android/server/PackageManagerService.java文件中：

    class PackageManagerService extends IPackageManager.Stub {  
        ......  

        public List<ResolveInfo> queryIntentActivities(Intent intent,  
                String resolvedType, int flags) {  
            ......  

            synchronized (mPackages) {  
                String pkgName = intent.getPackage();  
                if (pkgName == null) {  
                    return (List<ResolveInfo>)mActivities.queryIntent(intent,  
                            resolvedType, flags);  
                }  

                ......  
            }  

            ......  
        }  

        ......  
    }  

回忆前面一篇文章Android应用程序安装过程源代码分析，系统在前面的Step 8中启动PackageManagerService时，会把系统中的应用程序都解析一遍，然后把解析得到的Activity都保存在mActivities变量中，这里通过这个mActivities变量的queryIntent函数返回符合条件intent的Activity，这里要返回的便是Action类型为Intent.ACTION_MAIN，并且Category类型为Intent.CATEGORY_LAUNCHER的Activity了。

回到Step 18中的 LoaderTask.loadAllAppsByBatch函数中，从queryIntentActivities函数调用处返回所要求的Activity后，便调用函数tryGetCallbacks(oldCallbacks)得到一个返CallBack接口，这个接口是由Launcher类实现的，接着调用这个接口的.bindAllApplications函数来进一步操作。注意，这里又是通过消息来处理加载应用程序的操作的。

##Step 20. Launcher.bindAllApplications

这个函数定义在packages/apps/Launcher2/src/com/android/launcher2/Launcher.java文件中：


    public final class Launcher extends Activity  
            implements View.OnClickListener, OnLongClickListener, LauncherModel.Callbacks, AllAppsView.Watcher {  
        ......  

        private AllAppsView mAllAppsGrid;  

        ......  

        public void bindAllApplications(ArrayList<ApplicationInfo> apps) {  
            mAllAppsGrid.setApps(apps);  
        }  

        ......  
    }  


这里的mAllAppsGrid是一个AllAppsView类型的变量，它的实际类型一般就是AllApps2D了。

##Step 21. AllApps2D.setApps

这个函数定义在packages/apps/Launcher2/src/com/android/launcher2/AllApps2D.java文件中：

    public class AllApps2D  
        extends RelativeLayout  
        implements AllAppsView,  
            AdapterView.OnItemClickListener,  
            AdapterView.OnItemLongClickListener,  
            View.OnKeyListener,  
            DragSource {  

        ......  

        public void setApps(ArrayList<ApplicationInfo> list) {  
            mAllAppsList.clear();  
            addApps(list);  
        }  

        public void addApps(ArrayList<ApplicationInfo> list) {  
            final int N = list.size();  

            for (int i=0; i<N; i++) {  
                final ApplicationInfo item = list.get(i);  
                int index = Collections.binarySearch(mAllAppsList, item,  
                    LauncherModel.APP_NAME_COMPARATOR);  
                if (index < 0) {  
                    index = -(index+1);  
                }  
                mAllAppsList.add(index, item);  
            }  
            mAppsAdapter.notifyDataSetChanged();  
        }  

        ......  
    }  

函数setApps首先清空mAllAppsList列表，然后调用addApps函数来为上一步得到的每一个应用程序创建一个ApplicationInfo实例了，有了这些ApplicationInfo实例之后，就可以在桌面上展示系统中所有的应用程序了。

    public final class Launcher extends Activity  
            implements View.OnClickListener, OnLongClickListener, LauncherModel.Callbacks, AllAppsView.Watcher {  
        ......  

        public void onClick(View v) {  
            Object tag = v.getTag();  
            if (tag instanceof ShortcutInfo) {  
                ......  
            } else if (tag instanceof FolderInfo) {  
                ......  
            } else if (v == mHandleView) {  
                if (isAllAppsVisible()) {  
                    ......  
                } else {  
                    showAllApps(true);  
                }  
            }  
        }  

        ......  
    }  

接着就会调用showAllApps函数显示应用程序图标：


    public final class Launcher extends Activity  
            implements View.OnClickListener, OnLongClickListener, LauncherModel.Callbacks, AllAppsView.Watcher {  
        ......  

        void showAllApps(boolean animated) {  
            mAllAppsGrid.zoom(1.0f, animated);  

            ((View) mAllAppsGrid).setFocusable(true);  
            ((View) mAllAppsGrid).requestFocus();  

            // TODO: fade these two too  
            mDeleteZone.setVisibility(View.GONE);  
        }  

        ......  
    }  

当点击上面的这些应用程序图标时，便会响应AllApps2D.onItemClick函数：

    public class AllApps2D  
        extends RelativeLayout  
        implements AllAppsView,  
            AdapterView.OnItemClickListener,  
            AdapterView.OnItemLongClickListener,  
            View.OnKeyListener,  
            DragSource {  

        ......  

        public void onItemClick(AdapterView parent, View v, int position, long id) {  
            ApplicationInfo app = (ApplicationInfo) parent.getItemAtPosition(position);  
            mLauncher.startActivitySafely(app.intent, app);  
        }  


        ......  
    }

这里的成员变量mLauncher的类型为Launcher，于是就调用Launcher.startActivitySafely函数来启动应用程序了，这个过程具体可以参考Android应用程序启动过程源代码分析一文。
