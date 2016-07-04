#java aidl

CalculateInterface

    1package com.example.aidl.calculate;
    2
    3 interface CalculateInterface {
    4     double doCalculate(double a, double b);
    5 }

编译发现，目录结构如图1-1和图1-2中gen/com.example.aidl.calculate中多了CalculateInterface.java文件，内容如下：


    1 package com.example.aidl.calculate;
    2
    3 interface CalculateInterface {
    4     double doCalculate(double a, double b);
    5 }

定义好接口就是要看服务端和客户端的代码啦，其中服务端主要看CalculateService代码，这个一个继承Service的类，在其中对AIDL中的接口进行赋予实际意义,如下：


    3 import com.example.aidl.calculate.CalculateInterface;
    4 import com.example.aidl.calculate.CalculateInterface.Stub;



    12 public class CalculateService extends Service {
    13     
    14     private static final String                            TAG                        =    "CalculateService";
    15     
    16     @Override
    17     public IBinder onBind(Intent arg0) {
    18         // TODO Auto-generated method stub
    19         logE("onBind()");
    20         return mBinder;
    21     }
    22     
    23     @Override
    24     public void onCreate() {
    25         // TODO Auto-generated method stub
    26         logE("onCreate()");
    27         super.onCreate();
    28     }
    29     
    30     @Override
    31     public void onStart(Intent intent, int startId) {
    32         // TODO Auto-generated method stub
    33         logE("onStart()");
    34         super.onStart(intent, startId);
    35     }
    36     
    37     @Override
    38     public boolean onUnbind(Intent intent) {
    39         // TODO Auto-generated method stub
    40         logE("onUnbind()");
    41         return super.onUnbind(intent);
    42     }
    43     
    44     @Override
    45     public void onDestroy() {
    46         // TODO Auto-generated method stub
    47         logE("onDestroy()");
    48         super.onDestroy();
    49     }
    50     
    51     private static void logE(String str) {
    52         Log.e(TAG, "--------" + str + "--------");
    53     }
    54     
    55     private final CalculateInterface.Stub mBinder = new CalculateInterface.Stub() {
    56         
    57         @Override
    58         public double doCalculate(double a, double b) throws RemoteException {
    59             // TODO Auto-generated method stub
    60             Log.e("Calculate", "远程计算中");
    61             Calculate calculate = new Calculate();
    62             double answer = calculate.calculateSum(a, b);
    63             return answer;
    64         }
    65     };
    66 }



#android AIDL 小结

1、AIDL （Android Interface Definition Language ）

2、AIDL 适用于 进程间通信，并且与Service端多个线程并发的情况，如果只是单个线程 可以使用 Messenger ，如果不需要IPC 可以使用Binder

3、AIDL语法：基础数据类型都可以适用，List Map等有限适用。static field 不适用。

4、AIDL基本用法

第一步：实现.aidl文件
复制代码

接口描述文件
1. 导入的包名
2. 如果有使用Object对象，需要该对象 implement Parcelable 接口，并且需要导入该接口包名+类名；
如果是primitive type 不需要这步。
3. 定义方法名称。
4. 所有的.aidl文件已经需要传递的对象接口需要在Service 与Client中各一份


    package com.aidl;
    import com.aidl.Data;
    interface IaidlData
    {
        int getPid();

        String getName();

        com.aidl.Data getData();
    }

2、在Service中实现.aidl 接口。实际实现的接口是在 gen中自动生成的 IaidlData.java的抽象内部类 Stub
复制代码

1. 需要在配置文件Androidmanifest.xml文件中声明Service，并且添加intent-filter 节点 的action属性，
   此属性一般可以使用 aidl的包名+文件名（因为该值在服务器与客户端一致）
2. 需要实现IaidlData.aidl文件中定义的接口。
   aidl文件是一个接口描述文件，会在gen中自动生成一个同名的IaidlData.java接口文件，该接口文件包含一个抽象类stub，其继承了android.os.Binder、实现IaidlData接口

   故，我们实际需要实现的是Stub抽象类。


    public class AidlService extends Service
    {
        public void onCreate()
        {
            Log.d("aidl", "aidlService--------------onCreate");
        }

        public IBinder onBind(Intent intent)
        {
            return mBinder;
        }

        private final IaidlData.Stub mBinder = new IaidlData.Stub()
        {
            public int getPid()
            {
                return Process.myPid();
            }

            public String getName() throws RemoteException
            {
                return "go or not go is a problem";
            }

            public Data getData() throws RemoteException
            {
                Data data = new Data();
                data.id = Process.myPid();
                data.name = "go or not go is a problem";
                return data;
            }
        };
    }

3、绑定Service ，并且获取IaidlService 对象

1. 建立连接，使用Action属性定位需要的Service
   actoin的属性的采用aidl文件的类名+包名（与服务一致），之前需要在服务中设置相同的action属性，否则找不到服务。

2. 获取服务返回的stub对象，mIaidlData = IaidlData.Stub.asInterface(service);



    package com.clent;

    import com.aidl.IaidlData;


    import android.app.Activity;
    import android.content.ComponentName;
    import android.content.Intent;
    import android.content.ServiceConnection;
    import android.os.Bundle;
    import android.os.IBinder;
    import android.os.RemoteException;
    import android.util.Log;
    import android.view.View;

    public class AidlClientActivity extends Activity
    {
        IaidlData mIaidlData;

        public void onCreate(Bundle savedInstanceState)
        {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);        
        }    

        protected void onStart()
        {
            super.onStart();
            Log.d("aidl" , "onstart ----------bindservice-----"+IaidlData.class.getName());
            Intent intent = new Intent(IaidlData.class.getName());
            bindService(intent, mSecondaryConnection, BIND_AUTO_CREATE);
        }

        private ServiceConnection mSecondaryConnection = new ServiceConnection()
        {
            public void onServiceConnected(ComponentName className, IBinder service)
            {
                Log.d("aidl", "onServiceConnected----------------");
                mIaidlData = IaidlData.Stub.asInterface(service);
            }

            public void onServiceDisconnected(ComponentName className)
            {
                mIaidlData = null;
            }
        };


        public void onClick(View view)
        {
            System.out.println( " onclick--------------- : ");
            if(mIaidlData != null)
            {
                try
                {
                    System.out.println( " name : "+mIaidlData.getName());

                    System.out.println( " id   : "+mIaidlData.getPid());

                    System.out.println( " data : "+mIaidlData.getData().id +"   "+mIaidlData.getData().name);
                }
                catch (RemoteException e)
                {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }

            }
        }


        protected void onDestroy()
        {
            super.onDestroy();
            unbindService(mSecondaryConnection);
        }
    }

4、如果aidl文件中需要传递Object对象，需要添加对应的.aidl文件
复制代码

1. 定义该对象Data，并实现Parcelable
2. 添加Data.aidl文件，并采用如下方式编写导入Data
3. 需要在引用到Data的.aidl文件中 import com.aidl.Data
4. 需要在服务端和 客户端都添加 Data.aidl与 Data.java文件 并且需要一致。

    Data.aidl
    aidl文件：
    package com.aidl;

        parcelable Data;

复制代码

5、添加 对应的Object类，并且实现Parcelable接口
复制代码

    public class Data implements Parcelable
    {
        public int id;
        public String name;

        public static final Parcelable.Creator<Data> CREATOR = new Parcelable.Creator<Data>()
        {
            public Data createFromParcel(Parcel in)
            {
                return new Data(in);
            }

            public Data[] newArray(int size)
            {
                return new Data[size];
            }
        };

        public Data()
        {
        }

        private Data(Parcel in)
        {
            readFromParcel(in);
        }    

        public void readFromParcel(Parcel in)
        {
            id = in.readInt();
            name = in.readString();        
        }

        public int describeContents()
        {
            return 0;
        }

        public void writeToParcel(Parcel dest, int flags)
        {
            dest.writeInt(id);
            dest.writeString(name);        
        }
    }
