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

> ZygoteInit进程
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

> ZygoteInit.main()
```c++
//预加载信息，加载了一部分framework的资源，以及常用的java类，加快了App进程的启动
preload(bootTimingsTraceLog);

//创建socket，进程通信机制，为什么不用Binder？ 1.Binder还没有完成初始化 2.Binder为多线程机制，fork是写实拷贝，容易导致死锁
zygoteServer = new ZygoteServer(isPrimaryZygote);

//fork SystemServer进程
Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);
r.run();

//进入死循环，等待AMS的消息来创建进程
caller = zygoteServer.runSelectLoop(abiList);
```
> 思考：为什么使用Zygote去fork App的进程而不是init和SystemServer
> 1. init创建了很多进程，很多是App进程不需要的
> 2. 虚拟机的创建在Zygote
> 3. SystemServer需要启动近100个服务AMS WMS PMS等，这些服务并非是App进程必须的，App可以利用进程通信来访问这些服务

## 6. SystemServer
> AMS WMS PMS均由SystemServer启动
```c++

```
## 7. Apps
> zygote启动App进程

# Android App安装流程

# Android App启动流程

# Android Activity启动流程

# Android Service启动流程
