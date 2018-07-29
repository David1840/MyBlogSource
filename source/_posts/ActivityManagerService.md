---
title: AMS启动
date: 2018-07-20 11:07:39
categories: 
- Android系统
tags:
- Android
- 系统
---

## startBootstrapServices
在SystemServer.run方法中startBootstrapServices()

```
private void startBootstrapServices() {
     Installer installer = mSystemServiceManager.startService(Installer.class);
     //启动AMS服务
     mActivityManagerService = mSystemServiceManager.startService(
             ActivityManagerService.Lifecycle.class).getService();
     //设置AMS的系统服务管理器
     mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
     //设置AMS的APP安装器
     mActivityManagerService.setInstaller(installer);
   ...
     //设置SystemServer
     mActivityManagerService.setSystemProcess();
 }
```

`mSystemServiceManager.startService(ActivityManagerService.Lifecycle.class)`功能：

1. 创建ActivityManagerService.Lifecycle对象；
2. 调用Lifecycle.onStart()方法。

```
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;

    public Lifecycle(Context context) {
        super(context);
        //创建ActivityManagerService
        mService = new ActivityManagerService(context);
    }

    @Override
    public void onStart() {
        mService.start(); 
    }

    public ActivityManagerService getService() {
        return mService;
    }
}
```

在Lifecycle中创建AMS实例对象，并调用AMS.start();在new ActivityManagerService(context)创建了三个线程，分别为”ActivityManager”，”android.ui”，”CpuTracker”。


## setSystemProcess

```
public void setSystemProcess() {
    try {
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true); //AMS
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats); //进程统计
        ServiceManager.addService("meminfo", new MemBinder(this));  //内存
        ServiceManager.addService("gfxinfo", new GraphicsBinder(this));  //图像信息
        ServiceManager.addService("dbinfo", new DbBinder(this)); //数据库
        if (MONITOR_CPU_USAGE) {
            ServiceManager.addService("cpuinfo", new CpuBinder(this)); //CPU
        }
        ServiceManager.addService("permission", new PermissionController(this)); //权限
        ServiceManager.addService("processinfo", new ProcessInfoService(this)); //进程服务
        ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                "android", STOCK_PM_FLAGS);

       
        mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());
        synchronized (this) {
            //创建ProcessRecord对象
            ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
            app.persistent = true; //设置为persistent进程
            app.pid = MY_PID;
            app.maxAdj = ProcessList.SYSTEM_ADJ;
            app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.put(app.pid, app);
            }
            updateLruProcessLocked(app, false, null);//维护进程lru
            updateOomAdjLocked(); //更新adj
        }
    } catch (PackageManager.NameNotFoundException e) {
        throw new RuntimeException("", e);
    }
}
```

该方法主要就是注册各种服务到ServiceManager。

## startOtherServices()

```
private void startOtherServices() {

  //安装系统Provider
  mActivityManagerService.installSystemProviders();
  ...
  mActivityManagerService.systemReady(new Runnable() {
    public void run() {
      //phase550
      mSystemServiceManager.startBootPhase(
              SystemService.PHASE_ACTIVITY_MANAGER_READY);

      mActivityManagerService.startObservingNativeCrashes();
      //启动WebView
      WebViewFactory.prepareWebViewInSystemServer();
      //启动系统UI
      startSystemUi(context);

      // 执行一系列服务的systemReady方法
      networkScoreF.systemReady();
      networkManagementF.systemReady();
      networkStatsF.systemReady();
      networkPolicyF.systemReady();
      connectivityF.systemReady();
      audioServiceF.systemReady();
      Watchdog.getInstance().start(); //Watchdog开始工作

      //phase600
      mSystemServiceManager.startBootPhase(
              SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);

      //执行一系列服务的systemRunning方法
      wallpaper.systemRunning();
      inputMethodManager.systemRunning(statusBarF);
      location.systemRunning();
      countryDetector.systemRunning();
      networkTimeUpdater.systemRunning();
      commonTimeMgmtService.systemRunning();
      textServiceManagerService.systemRunning();
      assetAtlasService.systemRunning();
      inputManager.systemRunning();
      telephonyRegistry.systemRunning();
      mediaRouter.systemRunning();
      mmsService.systemRunning();
    }
  });
}
```

## 总结
1. 创建AMS实例对象，创建Andoid Runtime，ActivityThread和Context对象；
2. setSystemProcess：注册AMS、meminfo、cpuinfo等服务到ServiceManager；
3. installSystemProviderss，加载SettingsProvider；
4. 启动SystemUIService，再调用一系列服务的systemReady()方法；
