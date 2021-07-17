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
> zygote进程是Java进程的鼻祖，它会fork出SystemServer进程

> zygote进程启动app_main.cpp - main()

> 启动ZygoteInit

> ZygoteInit.main() -- preload() // 预加载信息
> ZygoteInit.main() -- new ZygoteServer() // 创建zygote的socket服务
> ZygoteInit.main() -- r = forkSystemServer() // fork创建SystemServer进程
> ZygoteInit.main() -- r.run() // 执行SystemServer的main方法
> ZygoteInit.main() -- zygoteServer.runSelectLoop() // zygote进入无限循环
> 
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
AndroidRuntime.start(){
  //创建虚拟机
  startVm(&mJavaVM, &env, zygote, primary_zygote);
  //注册JNI方法
  startReg(env);
  //JNI调用ZygoteInit的main()，进入Java世界
  jmethodID startMeth = env->GetStaticMethodID(startClass, "main", "([Ljava/lang/String;)V");
  env->callStaticVoidMethod();
}
```

## 6. SystemServer
> AMS WMS PMS均由SystemServer启动
## 7. Apps
> zygote启动App进程

# Android App安装流程

# Android App启动流程

# Android Activity启动流程

# Android Service启动流程
