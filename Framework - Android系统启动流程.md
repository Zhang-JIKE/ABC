# Android 系统启动流程
## 1. BootROM
> 按下电源键，引导芯片代码会在预定地方（固化在ROM的地方）开始执行，加载引导程序BootLoader到RAM，并执行BootLoader程序
## 2. BootLoader
> 引导操作系统启动，启动第一个进程idle（pid = 0）
## 3. idle（pid = 0）进程
> idle进程启动两个进程：1.init（pid = 1）进程 2.kthread（pid = 2）进程
## 4. init（pid = 1）进程
> init进程是用户空间的鼻祖，它会fork出zygote进程
> 用户空间：应用层，执行app
> 内核空间：内核层，内核代码
## 5. zygote进程
> zygote进程启动app_main.cpp - main()
```c++
class AppRuntime : public AndroidRuntime
int main(){
  //创建AppRuntime
  AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
  //...
  //AppRuntime启动ZygoteInit进程
  runtime.start("com.android.internal.os.ZygoteInit",args, zygote);
}
```

> runtime.start() ZygoteInit进程
```c++
start(){
  //创建虚拟机，设置内存大小
  startVm(&mJavaVM, &env, zygote, primary_zygote);
  //注册JNI方法
  startReg(env);
  
  char* slashClassName = toSlashClassName(className != NULL ? className : "")
  
  //startClass即为"com.android.internal.os.ZygoteInit"的class
  jclass startClass = env->FindClass(slashClassName);
  
  //获取ZygoteInit.main()方法的ID
  jmethodID startMeth = env->GetStaticMethodID(startClass, "main", "([Ljava/lang/String;)V");
  
  //JNI调用ZygoteInit的main()，进入Java世界
  env->callStaticVoidMethod(startClass, startMeth, strArray);
}
```

> JNI调用并执行执行ZygoteInit的Java main()方法
```java
class ZygoteInit{
  public static void main(){
    //预加载信息，加载了一部分framework的资源，以及常用的java类，加快了App进程的启动
    preload(bootTimingsTraceLog);

    //创建socket，进程通信机制，为什么不用Binder？ 1.Binder还没有完成初始化 2.Binder为多线程机制，fork是写实拷贝，容易导致死锁
    zygoteServer = new ZygoteServer(isPrimaryZygote);

    //fork SystemServer进程
    Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
    
    //pid为0时，runnable被创建，fork出两个进程后由SystemServer进程执行该过程，而Zygote进程pid不为0，创建runnable为null，直接跳过此部分
    if(r != null){
      r.run();
      return;
    }

    //进入死循环，等待AMS的消息来创建进程
    caller = zygoteServer.runSelectLoop(abiList);
  }
}
```
> 思考：为什么使用Zygote去fork App的进程而不是init和SystemServer
> 1. init创建了很多进程，很多是App进程不需要的
> 2. 虚拟机的创建在Zygote
> 3. SystemServer需要启动近100个服务AMS WMS PMS等，这些服务并非是App进程必须的，App可以利用进程通信来访问这些服务

## 6. 创建SystemServer进程
```java
class ZygoteInit{
  private static Runnable forkSystemServer(String abiList, String socketName, ZygoteServer zygoteServer){
    String args[] = {...,"com.android.server.SystemServer"};
    ZygoteArguments parsedArgs = null;
    parsedArgs = new ZygoteArguments(args);
    
    int pid = Zygote.forkSystemServer(parsedArgs.mUid,...);
    
    //pid == 0 表示当前为子进程，若当前为父进程返回值为子进程pid
    if(pid == 0){
      return handleSystemServerProcess(parsedArgs);
    }
    return null;
  }
  
  private static Runnable handleSystemServerProcess(ZygoteArguments parsedArgs){
    final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
    
    //创建类加载器，设置当前线程的类加载器
    ClassLoader cl = createPathClassLoader(systemServerClasspath, parsedArgs.mTargetSdkVersion);
    Thread.currentThread().setContextClassLoader(cl);
    
    return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion, parsedArgs.mDisabledCompatChanges, parsedArgs.mRemainingArgs, cl);
    
    public static final Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges, String[] argv, ClassLoader classLoader){
      return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv, classLoader);
    }
  }
}
```

```java
class RuntimeInit{
  protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges, String[] argv, ClassLoader classLoader){
  VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
  VMRuntime.getRuntime().setDisabledCompatChanges(disabledCompatChanges);
      
  final Arguments args = new Arguments(argv);
      
  return findStaticMain(args.startClass, args.startArgs, classLoader);
  }
   
  protected static Runnable findStaticMain(String className, String[] argv, ClassLoader classLoader){
    Class<?> cl = Class.forName(className, true, classLoader);
    Method m = cl.getMethod("main", new Class[]{ String[].class });
    return new MethodAndArgsCaller(m, argv);
  }
   
  static class MethodAndArgsCaller implements Runnable{
    private final Method method;
    private final String[] mArgs;
  
    //构造方法
    public MethodAndArgsCaller(...);
    
    //反射多线程调用SystemServer的main方法
    public void run(){
      mMethod.invoke(null, new Object[]{ mArgs });
    }
  }
}
```
![fork](https://user-images.githubusercontent.com/28483207/126585573-74e2537b-4bfa-4f89-af6d-d0f636183a56.png)

## 7. SystemServer进程，执行SystemServer的main方法
```java
class SystemServer{
  
  //用于启动各项服务
  SystemServiceManager mSystemServiceManager;
  
  public SystemServer(...);
  
  public static void main{
    SystemServer().run();
  }
  
  private void run(){
    //加载动态链接库libandroid_servers.so
    System.loadLibrary("android_servers");
    //创建系统上下文
    createSystemContext();  
    ActivityThread.initializeMainlineModules();
    
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
   
    //ATMS启动,AMS启动,AMS.setSystemProcess()将AMS等服务添加到ServiceManager中,PMS启动,PKMS启动
    startBootstrapServices();
    //电池服务
    startCoreServices();
    //WMS服务,AMS.systemReady()启动桌面
    startOtherServices();
  }
  
  private void startBootstrapServices(){
    //1.通过反射创建对象 2.调用Lifecycle的onStart()方法
    ActivityTaskManagerService atm = mSystemServiceManager.startService(ActivityTaskManagerService.Lifecycle.class).getService();
    mActivityManagerService = ActivityManagerService.Lifecycle.startService(mSystemServiceManager, atm)
    
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    
    mActivityManagerService.setSystemProcess();
  }
}

class ActivityTaskManagerService{

  public ActivityTaskManagerService(Context context){
    mSystemThread = ActivityThhread.currentActivityThread();
    
    //Activity生命周期管理类
    mLifecycleManager = new ClientLifecycleManager();
  }
  
  public void initialize(IntentFirewall intentFirewall, PendingIntentController pendingIntentController, Looper looper){
    //ActivityStackSupervisor 栈管理对象
    mStackSupervisor = createStackSupervisor();
    
    //设置当前任务栈
    setRecentTasks(new RecentTasks(this, mStackSupervisor))
  }
  
  public static final class Lifecycle extends SystemService{
    
    private final ActivityTaskManagerService mService;
    
    public Lifecycle(Context context){
      mService = new ActivityTaskManagerService(context);
    }
    
    @Override
    public void onStart(){
      //把atm服务保存在ServiceManager中
      publishBinderService(Context.ACTIVITY_TASK_SERVICE, mService);
      mService.start();
    }
  }
}

class SystemService{
  protected final void publishBinderService(String name, IBinder service, boolean allowIsolated, int dumpPriority){
    ServiceManager.addService(name, service, allowIsolated, dumpPriority)
  }
}

class ActivityManagerService{

  public ActivityManagerService(Context context, ActivityManagerService atm){
    mActivityTaskManager = atm;
    mActivityTaskManager.initialize(mIntentFirewall, mPendingIntentController, DisplayThread.get().getLooper());
  }
  
  public void setSystemProcess(){
    //把当前ActivityManagerService保存到ServiceManager中
    ServiceManager.addService(Context.ACTIVITY_SERVICE, this, ...);
    
    ServiceManager.addService("meminfo", new MemBinder(this), ...);
    ServiceManager.addService("gfxinfo", new GraphicsBinder(this), ...);
    ServiceManager.addService("dbinfo", new DBBinder(this), ...);
    ServiceManager.addService("cpuinfo", new CpuBinder(this), ...);
    ServiceManager.addService("permission", new PermissionController(this), ...);
  }

  public static final class Lifecycle extends SystemService{
    
    private final ActivityManagerService mService;
    private static c sAtm;
    
    public Lifecycle(Context context){
      mService = new ActivityManagerService(context, sAtm);
    }
    
    public static ActivityManagerService startService(SystemServiceManager ssm, ActivityTaskManagerService atm){
      sAtm = atm;
      return ssm.startService(ActivityManagerService.Lifecycle.class).getService();
    }
    
    @Override
    public void onStart(){
      mService.start();
    }
  }
}
```

# Android App安装流程

# Android App启动流程

# Android Activity启动流程

# Android Service启动流程
