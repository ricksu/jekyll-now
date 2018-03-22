# PowerManageService简析 (1st edition)

## PowerManager概述

PowerManager开放API对设备的电源使用进行控制. 对Power的控制会极大的影响设备的电池使用时间, 所以对PowerManager的使用必须谨慎. <br> PowerManager主要是通过wakelock对电源及CPU使用进行控制. 应该只在必须的情况下才申请wakelock， 而且应该尽早释放.

## Wakelock

PowerManager通过wakelock对屏幕、CPU等进行控制， 进而实现睡眠, doze等行为.

__如下表格显示了不同wakelock的对CPU, Screen, Keyboard的影响. 注意申请时只能申请一个wakelock.__

| Flag Value                | CPU  | Screen | Keyboard |
| ------------------------- | ---- | ------ | -------- |
| `PARTIAL_WAKE_LOCK`       | On*  | Off    | Off      |
| `SCREEN_DIM_WAKE_LOCK`    | On   | Dim    | Off      |
| `SCREEN_BRIGHT_WAKE_LOCK` | On   | Bright | Off      |
| `FULL_WAKE_LOCK`          | On   | Bright | Bright   |

只有PARTIAL_WAKE_LOCK可以保证CPU一直运行， 即使power键被按下屏幕熄灭CPU也会运行. 申请其他wakelock的情况下， 按下power键则设备会进入睡眠状态.

另外还有2个flags， 它们只对screen有影响.

| Flag Value              | Description                              |
| ----------------------- | ---------------------------------------- |
| `ACQUIRE_CAUSES_WAKEUP` | Normal wake locks don't actually turn on the illumination. Instead, they cause the illumination to remain on once it turns on (e.g. from user activity). This flag will force the screen and/or keyboard to turn on immediately, when the WakeLock is acquired. A typical use would be for notifications which are important for the user to see immediately. |
| `ON_AFTER_RELEASE`      | If this flag is set, the user activity timer will be reset when the WakeLock is released, causing the illumination to remain on a bit longer. This can be used to reduce flicker if you are cycling between wake lock conditions. |

应用获取wakelock的sample code如下

```java
PowerManager pm = (PowerManager) getSystemService(Context.POWER_SERVICE);
PowerManager.WakeLock wl = pm.newWakeLock(PowerManager.SCREEN_DIM_WAKE_LOCK, "My Tag");
wl.acquire();
   ..screen will stay on during this section..
wl.release();
```

## PowerManager接口

PowerManager.aidl

```java
package android.os;

import android.os.WorkSource;

/** @hide */

interface IPowerManager
{
    void acquireWakeLock(IBinder lock, int flags, String tag, String packageName, in WorkSource ws,
            String historyTag);
    void acquireWakeLockWithUid(IBinder lock, int flags, String tag, String packageName,
            int uidtoblame);
    void releaseWakeLock(IBinder lock, int flags);
    void updateWakeLockUids(IBinder lock, in int[] uids);
    oneway void powerHint(int hintId, int data);

    void updateWakeLockWorkSource(IBinder lock, in WorkSource ws, String historyTag);
    boolean isWakeLockLevelSupported(int level);

    void userActivity(long time, int event, int flags);
    void wakeUp(long time, String reason, String opPackageName);
    void goToSleep(long time, int reason, int flags);

    void startBacklight(int msec);
    void stopBacklight();

    void nap(long time);
    boolean isInteractive();
    boolean isPowerSaveMode();
    boolean setPowerSaveMode(boolean mode);
    boolean isDeviceIdleMode();
    boolean isLightDeviceIdleMode();

    void reboot(boolean confirm, String reason, boolean wait);
    void rebootSafeMode(boolean confirm, boolean wait);
    void shutdown(boolean confirm, String reason, boolean wait);
    void crash(String message);

    void setStayOnSetting(int val);
    void boostScreenBrightness(long time);
    boolean isScreenBrightnessBoosted();

    // temporarily overrides the screen brightness settings to allow the user to
    // see the effect of a settings change without applying it immediately
    void setTemporaryScreenBrightnessSettingOverride(int brightness);
    void setTemporaryScreenAutoBrightnessAdjustmentSettingOverride(float adj);

    void setBacklightOffForWfd(boolean enable);
    void setButtonOffForWfd(boolean enable);
    // sets the attention light (used by phone app only)
    void setAttentionLight(boolean on, int color);
}
```

## PowerManager 初始化

在SystemServer中启动

> ```java
>  mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
> ```

Context.getSystemService() 的注册,

```java
//SystemServiceRegistry.java
//Manages all of the system services that can be returned by Context#getSystemService. 

static {
 .......
registerService(Context.POWER_SERVICE, PowerManager.class,
                new CachedServiceFetcher<PowerManager>() {
            @Override
            public PowerManager createService(ContextImpl ctx) {
                IBinder b = ServiceManager.getService(Context.POWER_SERVICE);
                IPowerManager service = IPowerManager.Stub.asInterface(b);
                if (service == null) {
                    Log.wtf(TAG, "Failed to get power manager service.");
                }
                return new PowerManager(ctx.getOuterContext(),
                        service, ctx.mMainThread.getHandler());
            }});
   }
```

## PowerManagerService

PowerManager是PowerManagerService的代理， 它们之间通过Binder进行通信.<br> PowerManagerService主要负责wakelock的申请、释放， 屏幕亮、暗的管理，CPU睡眠的管理等. 与之交互的service包括ActivityManagerService, WMS, displayService, BatteryService等等. 是Android系统的非常核心的service.

1. service初始化

构造函数

> ```java
> public PowerManagerService(Context context) {
>         super(context);
>         mContext = context;
>         mHandlerThread = new ServiceThread(TAG,
>                 Process.THREAD_PRIORITY_DISPLAY, false /*allowIo*/);
>         mHandlerThread.start();
>         mHandler = new PowerManagerHandler(mHandlerThread.getLooper());
>
>         synchronized (mLock) {
>             mWakeLockSuspendBlocker = createSuspendBlockerLocked("PowerManagerService.WakeLocks");
>             mDisplaySuspendBlocker = createSuspendBlockerLocked("PowerManagerService.Display");
>             mDisplaySuspendBlocker.acquire();
>             mHoldingDisplaySuspendBlocker = true;
>             mHalAutoSuspendModeEnabled = false;
>             mHalInteractiveModeEnabled = true;
>
>             mWakefulness = WAKEFULNESS_AWAKE;
>
>             nativeInit(); # navtive层初始化
>             nativeSetAutoSuspend(false);
>             nativeSetInteractive(true);
>             nativeSetFeature(POWER_FEATURE_DOUBLE_TAP_TO_WAKE, 0);
>         }
>     }
> ```

BinderService extends IPowerManager.Stub， 用于Binder通信， 开放API给系统. 上层可以通过getSystemService的方式获取PowerManager服务.

> ```java
>     @Override
>     public void onStart() {
>         publishBinderService(Context.POWER_SERVICE, new BinderService());
>         publishLocalService(PowerManagerInternal.class, new LocalService());
>
>         Watchdog.getInstance().addMonitor(this);
>         Watchdog.getInstance().addThread(mHandler);
>     }
> ```

systemReady

```
public void systemReady(IAppOpsService appOps) {
       获取需要交互的service
       注册相关settings的observer
       调用updatePowerStateLocked更新初始电源状态
    }
```

2. wakelock管理

   ```java
   private final class BinderService extends IPowerManager.Stub {
           @Override // Binder call
           public void acquireWakeLockWithUid(IBinder lock, int flags, String tag,
                   String packageName, int uid) {
               ....
           }

           @Override // Binder call
           public void acquireWakeLock(IBinder lock, int flags, String tag, String packageName,
                   WorkSource ws, String historyTag) {
               ....
               try {
                   acquireWakeLockInternal(lock, flags, tag, packageName, ws, historyTag, uid, pid);
               } finally {
                   Binder.restoreCallingIdentity(ident);
               }
               ....
           }

           @Override // Binder call
           public void releaseWakeLock(IBinder lock, int flags) {
               ...
               try {
                   releaseWakeLockInternal(lock, flags);
               } finally {
                   Binder.restoreCallingIdentity(ident);
               }
               .....
           }      
   }
   ```
3. 核心函数

      当wakelock, useractivity等状态发生变化时， 都会调用updatePowerStateLocked对Power state进行更新. 这是PowerManagerService中最核心的函数.

   ```
   /**
        * Updates the global power state based on dirty bits recorded in mDirty.
        *
        * This is the main function that performs power state transitions.
        * We centralize them here so that we can recompute the power state completely
        * each time something important changes, and ensure that we do it the same
        * way each time.  The point is to gather all of the transition logic here.
        */
       private void updatePowerStateLocked() {
           if (!mSystemReady || mDirty == 0) {
               return;
           }
           if (!Thread.holdsLock(mLock)) {
               Slog.wtf(TAG, "Power manager lock was not held when calling updatePowerStateLocked");
           }

           Trace.traceBegin(Trace.TRACE_TAG_POWER, "updatePowerState");
           try {
               // Phase 0: Basic state updates.
               updateIsPoweredLocked(mDirty);
               updateStayOnLocked(mDirty);
               updateScreenBrightnessBoostLocked(mDirty);

               // Phase 1: Update wakefulness.
               // Loop because the wake lock and user activity computations are influenced
               // by changes in wakefulness.
               final long now = SystemClock.uptimeMillis();
               int dirtyPhase2 = 0;
               for (;;) {
                   int dirtyPhase1 = mDirty;
                   dirtyPhase2 |= dirtyPhase1;
                   mDirty = 0;

                   updateWakeLockSummaryLocked(dirtyPhase1);
                   updateUserActivitySummaryLocked(now, dirtyPhase1);
                   if (!updateWakefulnessLocked(dirtyPhase1)) {
                       break;
                   }
               }

               // Phase 2: Update display power state.
               boolean displayBecameReady = updateDisplayPowerStateLocked(dirtyPhase2);

               // Phase 3: Update dream state (depends on display ready signal).
               updateDreamLocked(dirtyPhase2, displayBecameReady);

               // Phase 4: Send notifications, if needed.
               finishWakefulnessChangeIfNeededLocked();

               // Phase 5: Update suspend blocker.
               // Because we might release the last suspend blocker here, we need to make sure
               // we finished everything else first!
               updateSuspendBlockerLocked();
           } finally {
               Trace.traceEnd(Trace.TRACE_TAG_POWER);
           }
       }
   ```

   亮、灭屏都会在这个函数中进行处理及设置.

   ​

   ​
