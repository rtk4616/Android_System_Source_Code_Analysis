在[Android](http://lib.csdn.net/base/android)系统中，Activity组件在启动之后，并且在它的窗口显示出来之前，可以显示一个启动窗口。这个启动窗口可以看作是Activity组件的预览窗口，是由WindowManagerService服务统一管理的，即由WindowManagerService服务负责启动和结束。在本文中，我们就详细分析WindowManagerService服务启动和结束Activity组件的启动窗口的过程。

《[android](http://lib.csdn.net/base/android)系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

Activity组件的启动窗口是由ActivityManagerService服务来决定是否要显示的。如果需要显示，那么ActivityManagerService服务就会通知WindowManagerService服务来为正在启动的Activity组件显示一个启动窗口，而WindowManagerService服务又是通过窗口管理策略类PhoneWindowManager来创建这个启动窗口的。这个过程如图1所示。

![img](http://img.my.csdn.net/uploads/201302/10/1360501355_1966.jpg)

图1 Activity窗口的启动窗品的创建过程

窗口管理策略类PhoneWindowManager创建完成Activity组件的启动窗口之后，就会请求WindowManagerService服务将该启动窗口显示出来。当Activity组件启动完成，并且它的窗口也显示出来的时候，WindowManagerService服务就会结束显示它的启动窗口。

注意，Activity组件的启动窗口是由ActivityManagerService服务来控制是否显示的，也就是说，Android应用程序是无法决定是否要要Activity组件显示启动窗口的。接下来，我们就分别分析Activity组件的启动窗口的显示和结束过程。

## Activity组件的启动窗口的显示过程

从前面[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文可以知道，Activity组件在启动的过程中，会调用ActivityStack类的成员函数startActivityLocked。注意，在调用ActivityStack类的成员函数startActivityLocked的时候，Actvitiy组件还处于启动的过程，即它的窗口尚未显示出来，不过这时候ActivityManagerService服务会检查是否需要为正在启动的Activity组件显示一个启动窗口。如果需要的话，那么ActivityManagerService服务就会请求WindowManagerService服务为正在启动的Activity组件设置一个启动窗口。这个过程如图2所示。

![img](http://img.my.csdn.net/uploads/201302/12/1360664786_7420.jpg)

图2 Activity组件的启动窗口的显示过程

这个过程可以分为6个步骤，接下来我们就详细分析每一个步骤。

### Step 1. ActivityStack.startActivityLocked

```java
public class ActivityStack {  
    ......  

    // Set to false to disable the preview that is shown while a new activity  
    // is being started.  
    static final boolean SHOW_APP_STARTING_PREVIEW = true;  
    ......  

    private final void startActivityLocked(ActivityRecord r, boolean newTask,  
    boolean doResume) {  
final int NH = mHistory.size();  
......  

int addPos = -1;  
......  

// Place a new activity at top of stack, so it is next to interact  
// with the user.  
if (addPos < 0) {  
    addPos = NH;  
}  
......  

// Slot the activity into the history stack and proceed  
mHistory.add(addPos, r);  
......  

if (NH > 0) {  
    // We want to show the starting preview window if we are  
    // switching to a new task, or the next activity's process is  
    // not currently running.  
    boolean showStartingIcon = newTask;  
    ProcessRecord proc = r.app;  
    if (proc == null) {  
        proc = mService.mProcessNames.get(r.processName, r.info.applicationInfo.uid);  
    }  
    if (proc == null || proc.thread == null) {  
        showStartingIcon = true;  
    }  
    ......  

    mService.mWindowManager.addAppToken(  
            addPos, r, r.task.taskId, r.info.screenOrientation, r.fullscreen);  
    boolean doShow = true;  
    if (newTask) {  
        // Even though this activity is starting fresh, we still need  
        // to reset it to make sure we apply affinities to move any  
        // existing activities from other tasks in to it.  
        // If the caller has requested that the target task be  
        // reset, then do so.  
        if ((r.intent.getFlags()  
                &Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {  
            resetTaskIfNeededLocked(r, r);  
            doShow = topRunningNonDelayedActivityLocked(null) == r;  
        }  
    }  
    if (SHOW_APP_STARTING_PREVIEW && doShow) {  
        // Figure out if we are transitioning from another activity that is  
        // "has the same starting icon" as the next one.  This allows the  
        // window manager to keep the previous window it had previously  
        // created, if it still had one.  
        ActivityRecord prev = mResumedActivity;  
        if (prev != null) {  
            // We don't want to reuse the previous starting preview if:  
            // (1) The current activity is in a different task.  
            if (prev.task != r.task) prev = null;  
            // (2) The current activity is already displayed.  
            else if (prev.nowVisible) prev = null;  
        }  
        mService.mWindowManager.setAppStartingWindow(  
                r, r.packageName, r.theme, r.nonLocalizedLabel,  
                r.labelRes, r.icon, prev, showStartingIcon);  
    }  
} else {  
    // If this is the first activity, don't do any fancy animations,  
    // because there is nothing for it to animate on top of.  
    mService.mWindowManager.addAppToken(addPos, r, r.task.taskId,  
            r.info.screenOrientation, r.fullscreen);  
}  

......  

if (doResume) {  
    resumeTopActivityLocked(null);  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/[Java](http://lib.csdn.net/base/java)/com/android/server/am/ActivityStack.java中。

参数r描述的就是正在启动的Activity组件，而参数newTask和doResume描述的是是否要将该Activity组件放在一个新的任务中启动，以及是否要马上将该Activity组件启动起来。

ActivityStack类的成员变量mHistory指向的是一个ArrayList，它描述的便是系统的Activity组件堆栈。ActivityStack类的成员函数startActivityLocked首先找到正在启动的Activity组件r在系统的Activity组件堆栈中的位置addPos，然后再将正在启动的Activity组件r保存在这个位置上。

变量NH记录的是将正在启动的Activity组件r插入到系统的Activity组件堆栈中之前系统中已经启动了的Activity组件的个数。如果变量NH的值大于0，那么就说明系统需要执行一个Activity组件切换操作，即需要在系统当前激活的Activity组件和正在启动的Activity组件r之间执行一个切换操作，使得正在启动的Activity组件r成为系统接下来要激活的Activity组件。在切换的过程，需要显示切换动画，即给系统当前激活的Activity组件显示一个退出动画，而给正在启动的Activity组件r显示一个启动动画，以及需要为正在启动的Activity组件r显示一个启动窗口。另一方面，如果变量NH的值等于0，那么系统就不需要执行Activity组件切换操作，或者为为正在启动的Activity组件r显示一个启动窗口，这时候只需要为正在启动的Activity组件r创建一个窗口令牌即可。

ActivityStack类的成员变量mService指向的是一个ActivityManagerService对象，这个ActivityManagerService对象就是系统的Activity组件管理服务，它的成员变量mWindowManager指向的是一个WindowManagerService对象，这个WindowManagerService对象也就是系统的Window管理服务。通过调用WindowManagerService类的成员函数addAppToken就可以为正在启动的Activity组件r创建一个窗口令牌，这个过程可以参考前面[Android窗口管理服务WindowManagerService对窗口的组织方式分析](http://blog.csdn.net/luoshengyang/article/details/8498908)一文。

在变量NH的值大于0的情况下，ActivityStack类的成员函数startActivityLocked首先检查用来运行Activity组件r的进程是否已经启动起来了。如果已经启动起来，那么用来描述这个进程的ProcessRecord对象proc的值就不等于null，并且这个ProcessRecord对象proc的成员变量thread的值也不等于null。如果用来运行Activity组件r的进程还没有启动起来，或者Activity组件r需要运行在一个新的任务中，那么变量showStartingIcon的值就会等于true，用来描述在系统当前处于激活状态的Activity组件没有启动窗口的情况下，要为Activity组件r创建一个新的启动窗口，否则的话，就会将系统当前处于激活状态的Activity组件的启动窗口复用为Activity组件r的启动窗口。

系统当前处于激活状态的Activity组件是通过ActivityStack类的成员变量mResumedActivity来描述的，它的启动窗口可以复用为Activity组件r的启动窗口还需要满足两个额外的条件：

Activity组件mResumedActivity与Activity组件r运行在同一个任务中，即它们的成员变量task指向的是同一个TaskRecord对象；

Activity组件mResumedActivity当前是不可见的，即它的成员变量nowVisible的值等于false。

这两个条件意味着Activity组件mResumedActivity与Activity组件r运行在同一个任务中，并且Activity组件mResumedActivity的窗口还没有显示出来就需要切换到Activity组件r去。

ActivityStack类的静态成员变量SHOW_APP_STARTING_PREVIEW是用描述系统是否可以为正在启动的Activity组件显示启动窗口，只有在它的值等于true，以及正在启动的Activity组件的窗口接下来是要显示出来的情况下，即变量doShow的值等于true，ActivityManagerService服务才会请求WindowManagerService服务为正在启动的Activity组件设置启动窗口。

一般来说，一个正在启动的Activity组件的窗口接下来是需要显示的，但是正在启动的Activity组件可能会设置一个标志位，用来通知ActivityManagerService服务在它启动的时候，对它运行在的任务进行重置。一个任务被重置之后，可能会导致其它的Activity组件转移到这个任务中来，并且位于位于这个任务的顶端。在这种情况下，系统接下来要显示的窗口就不是正在启动的Activity组件的窗口的了，而是位于正在启动的Activity组件所运行在的任务的顶端的那个Activity组件的窗口。正在启动的Activity组件所运行在的任务同时也是一个前台任务，即它顶端的Activity组件就是系统Activity组件堆栈顶端的Activity组件。

调用参数r所指向一个ActivityRecord对象的成员变量intent所描述的一个Intent对象的成员函数getFlags就可以获得正在启动的Activity组件的标志值，当这个标志值的FLAG_ACTIVITY_RESET_TASK_IF_NEEDED位等于1的时候，就说明正在启动的Activity组件通知ActivityManagerService服务对它运行在的任务进行重置。重置一个任务是通过调用ActivityStack类的成员函数resetTaskIfNeededLocked来实现的。重置了正在启动的Activity组件所运行在的任务之后，再调用ActivityStack类的成员函数topRunningNonDelayedActivityLocked来检查位于系统Activity组件堆栈顶端的Activity组件是否就是正在启动的Activity组件，就可以知道正在启动的Activity组件的窗口接下来是否是需要显示的。如果需要显示的话，那么变量doShow的值就等于true。

ActivityManagerService服务请求WindowManagerService服务为正在启动的Activity组件设置启动窗口是通过调用WindowManagerService类的成员函数setAppStartingWindow来实现的。注意，ActivityManagerService服务在请求WindowManagerService服务为正在启动的Activity组件设置启动窗口之前，同样会调用WindowManagerService类的成员函数addAppToken来创建窗口令牌。

ActivityManagerService服务请求WindowManagerService服务为正在启动的Activity组件设置启动窗口之后，如果参数doResume的值等于true，那么就会调用ActivityStack类的成员函数resumeTopActivityLocked继续执行启动参数r所描述的一个Activity组件的操作，这个过程可以参考前面[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)一文。

接下来，我们就继续分析WindowManagerService类的成员函数setAppStartingWindow的实现，以便可以了解WindowManagerService服务是如何为正在启动的Activity组件设置启动窗口的。

### Step 2. WindowManagerService.setAppStartingWindow

WindowManagerService类的成员函数setAppStartingWindow定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中，它的实现比较长，我们分段来阅读：

```java
public class WindowManagerService extends IWindowManager.Stub  
implements Watchdog.Monitor {  
    ......  

    public void setAppStartingWindow(IBinder token, String pkg,  
    int theme, CharSequence nonLocalizedLabel, int labelRes, int icon,  
    IBinder transferFrom, boolean createIfNeeded) {  
......  

synchronized(mWindowMap) {  
    ......  

    AppWindowToken wtoken = findAppWindowToken(token);  
    ......  

    // If the display is frozen, we won't do anything until the  
    // actual window is displayed so there is no reason to put in  
    // the starting window.  
    if (mDisplayFrozen || !mPolicy.isScreenOn()) {  
        return;  
    }  

    if (wtoken.startingData != null) {  
        return;  
    }  
```
参数token描述的是要设置启动窗口的Activity组件，而参数transferFrom描述的是要将启动窗口转移给Activity组件token的Activity组件。从Step 1可以知道，这两个Activity组件是运行在同一个任务中的，并且参数token描述的Activity组件Activity组件是正在启动的Activity组件，而参数transferFrom描述的Activity组件是系统当前激活的Activity组件。

这段代码首先调用WindowManagerService类的成员函数findAppWindowToken来获得与参数token对应的一个类型为AppWindowToken的窗口令牌wtoken。如果这个AppWindowToken对象的成员变量startingData的值不等于null，那么就说明参数token所描述的Activity组件已经设置过启动窗口了，因此，WindowManagerService类的成员函数setAppStartingWindow就不用往下处理了。

这段代码还会检查系统屏幕当前是否处于冻结状态，即WindowManagerService类的成员变量mDisplayFrozen的值是否等于true，或者系统屏幕当前是否处于黑屏状态，即indowManagerService类的成员变量mPolicy所指向的一个PhoneWindowManager对象的成员函数isScreenOn的返回值是否等于false。如果是处于上述两种状态的话，那么WindowManagerService类的成员函数setAppStartingWindow就不用往下处理的。因为在这两种状态下，为token所描述的Activity组件设置的启动窗口是无法显示的。

我们接着往下阅读代码：

```java
if (transferFrom != null) {  
    AppWindowToken ttoken = findAppWindowToken(transferFrom);  
    if (ttoken != null) {  
WindowState startingWindow = ttoken.startingWindow;  
if (startingWindow != null) {  
    if (mStartingIconInTransition) {  
        // In this case, the starting icon has already  
        // been displayed, so start letting windows get  
        // shown immediately without any more transitions.  
        mSkipAppTransitionAnimation = true;  
    }  
    ......  

    final long origId = Binder.clearCallingIdentity();  

    // Transfer the starting window over to the new  
    // token.  
    wtoken.startingData = ttoken.startingData;  
    wtoken.startingView = ttoken.startingView;  
    wtoken.startingWindow = startingWindow;  
    ttoken.startingData = null;  
    ttoken.startingView = null;  
    ttoken.startingWindow = null;  
    ttoken.startingMoved = true;  
    startingWindow.mToken = wtoken;  
    startingWindow.mRootToken = wtoken;  
    startingWindow.mAppToken = wtoken;  
    ......  
    mWindows.remove(startingWindow);  
    mWindowsChanged = true;  
    ttoken.windows.remove(startingWindow);  
    ttoken.allAppWindows.remove(startingWindow);  
    addWindowToListInOrderLocked(startingWindow, true);  

    // Propagate other interesting state between the  
    // tokens.  If the old token is displayed, we should  
    // immediately force the new one to be displayed.  If  
    // it is animating, we need to move that animation to  
    // the new one.  
    if (ttoken.allDrawn) {  
        wtoken.allDrawn = true;  
    }  
    if (ttoken.firstWindowDrawn) {  
        wtoken.firstWindowDrawn = true;  
    }  
    if (!ttoken.hidden) {  
        wtoken.hidden = false;  
        wtoken.hiddenRequested = false;  
        wtoken.willBeHidden = false;  
    }  
    if (wtoken.clientHidden != ttoken.clientHidden) {  
        wtoken.clientHidden = ttoken.clientHidden;  
        wtoken.sendAppVisibilityToClients();  
    }  
    if (ttoken.animation != null) {  
        wtoken.animation = ttoken.animation;  
        wtoken.animating = ttoken.animating;  
        wtoken.animLayerAdjustment = ttoken.animLayerAdjustment;  
        ttoken.animation = null;  
        ttoken.animLayerAdjustment = 0;  
        wtoken.updateLayers();  
        ttoken.updateLayers();  
    }  

    updateFocusedWindowLocked(UPDATE_FOCUS_WILL_PLACE_SURFACES);  
    mLayoutNeeded = true;  
    performLayoutAndPlaceSurfacesLocked();  
    Binder.restoreCallingIdentity(origId);  
    return;  
} else if (ttoken.startingData != null) {  
    // The previous app was getting ready to show a  
    // starting window, but hasn't yet done so.  Steal it!  
    ......  
    wtoken.startingData = ttoken.startingData;  
    ttoken.startingData = null;  
    ttoken.startingMoved = true;  
    Message m = mH.obtainMessage(H.ADD_STARTING, wtoken);  
    // Note: we really want to do sendMessageAtFrontOfQueue() because we  
    // want to process the message ASAP, before any other queued  
    // messages.  
    mH.sendMessageAtFrontOfQueue(m);  
    return;  
}  
    }  
}  
```
如果参数transferFrom的值不等于null，那么就需要检查它所描述的Activity组件是否设置有启动窗口。如果设置有的话，那么就需要将它的启动窗口设置为参数token所描述的Activity组件的启动窗口。

参数transferFrom所描述的Activity组件所设置的启动窗口保存在与它所对应的一个类型为AppWindowToken的窗口令牌的成员变量startingWindow或者startingData中，因此，这段代码首先调用WindowManagerService类的成员函数findAppWindowToken来获得与参数transferFrom对应的一个AppWindowToken对象ttoken。如果AppWindowToken对象ttoken的成员变量startingWindow的值不等于null，那么就说明参数transferFrom所描述的Activity组件的启动窗口已经创建出来了。另一方面，如果AppWindowToken对象ttoken的成员变量startingData的值不等于null，那么就说明用来描述参数transferFrom所描述的Activity组件的启动窗口的相关数据已经准备好了，但是这个启动窗口还未创建出来。接下来我们就分别分析这两种情况。

我们首先分析AppWindowToken对象ttoken的成员变量startingWindow的值不等于null的情况。

这时候如果WindowManagerService类的成员变量mStartingIconInTransition的值等于true，那么就说明参数transferFrom所描述的Activity组件所设置的启动窗口已经在启动的过程中了。在这种情况下，就需要跳过参数token所描述的Activity组件和参数transferFrom所描述的Activity组件的切换过程，即将WindowManagerService类的成员变量mSkipAppTransitionAnimation的值设置为true，这是因为接下来除了要将参数transferFrom所描述的Activity组件的启动窗口转移给参数token所描述的Activity组件之外，还需要将参数transferFrom所描述的Activity组件的窗口状态转移给参数token所描述的Activity组件的窗口。

将参数transferFrom所描述的Activity组件的启动窗口转移给参数token所描述的Activity组件需要执行以下几个操作：

将AppWindowToken对象ttoken的成员变量startingData、startingView和startingWindow的值设置到AppWindowToken对象wtoken的对应成员变量中去，其中，成员变量startingData指向的是一个StartingData对象，它描述的是用来创建启动窗口的相关数据，成员变量startingView指向的是一个View对象，它描述的是启动窗口的顶层视图，成员变量startingWindow指向的是一个WindowState对象，它描述的就是启动窗口。

将AppWindowToken对象ttoken的成员变量startingData、startingView和startingWindow的值设置为null，这是因为参数transferFrom所描述的Activity组件的启动窗口已经转移给参数token所描述的Activity组件了。

将原来属于参数transferFrom所描述的Activity组件的启动窗口startingWindow的成员变量mToken、mRootToken和mAppToken的值设置为wtoken，因为这个启动窗口现在已经属于参数token所描述的Activity组件了。

将参数transferFrom所描述的Activity组件的窗口状态转移给参数token所描述的Activity组件的窗口需要执下几个操作：

将启动窗口startingWindow从窗口堆栈中删除，即从WindowManagerService类的成员变量mWindows所描述的一个ArrayList中删除。

将启动窗口startingWindow从属于窗口令牌ttoken的窗口列表中删除，即从AppWindowToken对象ttoken的成员变量windows和allAppWindows所描述的两个ArrayList中删除。

调用WindowManagerService类的成员函数addWindowToListInOrderLocked重新将启动窗口startingWindow插入到窗口堆栈中去。注意，因为这时候启动窗口startingWindow已经被设置为参数token所描述的Activity组件了，因此，在重新将它插入到窗口堆栈中去的时候，它就会位于参数token所描述的Activity组件的窗口的上面，这一点可以参考前面[Android窗口管理服务WindowManagerService对窗口的组织方式分析](http://blog.csdn.net/luoshengyang/article/details/8498908)一文。

如果AppWindowToken对象ttoken的成员变量allDrawn和firstWindowDrawn的值等于true，那么就说明与AppWindowToken对象ttoken对应的所有窗口或者第一个窗口已经绘制好了，这时候也需要分别将AppWindowToken对象wtoken的成员变量allDrawn和firstWindowDrawn的值设置为true，以便可以迫使那些与AppWindowToken对象wtoken对应的窗口接下来可以马上显示出来。

如果AppWindowToken对象ttoken的成员变量hidden的值等于false，那么就说明参数transferFrom所描述的Activity组件是处于可见状态的，这时候就需要将AppWindowToken对象wtoken的成员变量hidden、hiddenRequested和willBeHidden的值也设置为false，以便表示参数token所描述的Activity组件也是处于可见状态的。

AppWindowToken类的成员变量clientHidden描述的是对应的Activity组件在应用程序进程这一侧的可见状态。如果AppWindowToken对象wtoken和ttoken的成员变量clientHidden的值不相等，那么就需要将AppWindowToken对象ttoken的成员变量clientHidden的值设置给AppWindowToken对象wtoken的成员变量clientHidden，并且调用AppWindowToken对象wtoken的成员函数sendAppVisibilityToClients来通知相应的应用程序进程，运行在它里面的参数token所描述的Activity组件的可见状态。

如果AppWindowToken对象ttoken的成员变量animation的值不等于null，那么就说明参数transferFrom所描述的Activity组件的窗口正在显示动画，那么就需要将该动画转移给参数token所描述的Activity组件的窗口，即将AppWindowToken对象ttoken的成员变量animation、animating和animLayerAdjustment的值设置到AppWindowToken对象wtoken的对应成员变量，并且将AppWindowToken对象ttoken的成员变量animation和animLayerAdjustment的值设置为null和0。最后还需要重新计算与AppWindowToken对象ttoken和wtoken所对应的窗口的Z轴位置。

由于前面的操作导致窗口堆栈的窗口发生了变化，因此就需要调用WindowManagerService类的成员函数updateFocusedWindowLocked来重新计算系统当前可获得焦点的窗口，以及调用WindowManagerService类的成员函数performLayoutAndPlaceSurfacesLocked来刷新系统的UI。

我们接着分析AppWindowToken对象ttoken的成员变量startingData的值不等于null的情况。

这时候由于WindowManagerService服务还没有参数transferFrom所描述的Activity组件创建启动窗口，因此，这段代码只需要将用创建这个启动窗口的相关数据转移给参数token所描述的Activity组件就可以了，即将AppWindowToken对象ttoken的成员变量startingData的值设置给AppWindowToken对象wtoken的成员变量startingData，并且将AppWindowToken对象ttoken的成员变量startingData的值设置为null。

由于这时候参数token所描述的Activity组件的启动窗口还没有创建出来，因此，接下来就会向WindowManagerService服务所运行在的线程的消息队列的头部插入一个类型ADD_STARTING的消息。当这个消息被处理的时候，WindowManagerService服务就会为参数token所描述的Activity组件创建一个启动窗口。

WindowManagerService类的成员变量mH指向的是一个类型为H的对象。H是WindowManagerService的一个内部类，它是从Handler类继承下来的，因此，调用它的成员函数sendMessageAtFrontOfQueue就可以往一个线程的消息队列的头部插入一个消息。又由于 WindowManagerService类的成员变量mH所指向的一个H对象是在WindowManagerService服务所运行在的线程中创建的，因此，调用它的成员函数sendMessageAtFrontOfQueue发送的消息是保存在WindowManagerService服务所运行在的线程的消息队列中的。

如果参数transferFrom所描述的Activity组件没有启动窗口或者启动窗口数据转移给参数token所描述的Activity组件，那么接下来就可能需要为参数token所描述的Activity组件创建一个新的启动窗口，如最后一段代码所示：

```java
// There is no existing starting window, and the caller doesn't  
// want us to create one, so that's it!  
if (!createIfNeeded) {  
    return;  
}  

// If this is a translucent or wallpaper window, then don't  
// show a starting window -- the current effect (a full-screen  
// opaque starting window that fades away to the real contents  
// when it is ready) does not work for this.  
if (theme != 0) {  
    AttributeCache.Entry ent = AttributeCache.instance().get(pkg, theme,  
            com.android.internal.R.styleable.Window);  
    if (ent.array.getBoolean(  
            com.android.internal.R.styleable.Window_windowIsTranslucent, false)) {  
        return;  
    }  
    if (ent.array.getBoolean(  
            com.android.internal.R.styleable.Window_windowIsFloating, false)) {  
        return;  
    }  
    if (ent.array.getBoolean(  
            com.android.internal.R.styleable.Window_windowShowWallpaper, false)) {  
        return;  
    }  
}  

mStartingIconInTransition = true;  
wtoken.startingData = new StartingData(  
        pkg, theme, nonLocalizedLabel,  
        labelRes, icon);  
Message m = mH.obtainMessage(H.ADD_STARTING, wtoken);  
// Note: we really want to do sendMessageAtFrontOfQueue() because we  
// want to process the message ASAP, before any other queued  
// messages.  
mH.sendMessageAtFrontOfQueue(m);  
    }  
}  

......  
```
如果参数createIfNeeded的值等于false，那么就说明不可以为参数token所描述的Activity组件创建一个新的启动窗口，因此，这时候WindowManagerService类的成员函数setAppStartingWindow就直接返回而不往下处理了。

另一方面，如果参数token所描述的Activity组件的窗口设置有一个主题，即参数theme的值不等于0，那么该窗口就有可能是：

背景是半透明的；

浮动窗口，即是一个壁纸窗口或者一个输入法窗口；

需要显示壁纸（背景也是半透明的）。

由于浮动窗口和背景半透明的窗口是不可以显示启动窗口的，因此，在上述三种情况下，WindowManagerService类的成员函数setAppStartingWindow也是直接返回而不往下处理了。

通过了上面的检查之后，这段代码就可以为参数token所描述的Activity组件创建一个启动窗口了，不过这个启动窗口不是马上就创建的，而通过一个类型为ADD_STARTING的消息来驱动创建的。这个类型为ADD_STARTING的消息是需要发送到WindowManagerService服务所运行在的线程的消息队列的头部去的。在发送这个类型为ADD_STARTING的消息之前，这段代码首先会创建一个StartingData对象，并且保存在AppWindowToken对象wtoken的成员变量startingData中，用来封装创建启动窗口所需要的数据。

如上所述，通过调用WindowManagerService类的成员变量mH的成员函数sendMessageAtFrontOfQueue可以向WindowManagerService服务所运行在的线程的消息队列的头部发送一个类型为ADD_STARTING的消息。注意，在发送这个消息之前，这段代码还会将WindowManagerService类的成员变量mStartingIconInTransition的值设置为true，以便可以表示WindowManagerService服务正在为正在启动的Activity组件创建启动窗口。

接下来，我们就继续分析定义在WindowManagerService内部的H类的成员函数sendMessageAtFrontOfQueue的实现，以便可以了解Activity组件的启动窗口的创建过程。

### Step 3. H.sendMessageAtFrontOfQueue

H类的成员函数sendMessageAtFrontOfQueue是从父类Handler继承下来的，因此，这一步调用的实际上是Handler类的成员函数sendMessageAtFrontOfQueue。Handler类的成员函数sendMessageAtFrontOfQueue用来向消息队列头部的插入一个新的消息，以便这个消息可以下一次消息循环中就能得到处理。在前面[Android应用程序线程消息循环模型分析](http://blog.csdn.net/luoshengyang/article/details/6905587)一文中，我们已经分析过往消息队列发送消息的过程了，这里不再详述。

从上面的调用过程可以知道，这一步所发送的消息的类型为ADD_STARTING，并且是向WindowManagerService服务所运行在的线程的消息队列发送的。当这个消息得到处理的时候，H类的成员函数handleMessage就会被调用，因此，接下来我们就继续分析H类的成员函数handleMessage的实现，以便可以了解Activity组件的启动窗口的创建过程。

### Step 4. H.handleMessage

```java
public class WindowManagerService extends IWindowManager.Stub  
implements Watchdog.Monitor {  
    ......  

    private final class H extends Handler {  
......  

public void handleMessage(Message msg) {  
    switch (msg.what) {   
        ......  

        case ADD_STARTING: {  
            final AppWindowToken wtoken = (AppWindowToken)msg.obj;  
            final StartingData sd = wtoken.startingData;  

            if (sd == null) {  
                // Animation has been canceled... do nothing.  
                return;  
            }  

            ......  

            View view = null;  
            try {  
                view = mPolicy.addStartingWindow(  
                    wtoken.token, sd.pkg,  
                    sd.theme, sd.nonLocalizedLabel, sd.labelRes,  
                    sd.icon);  
            } catch (Exception e) {  
                ......  
            }  

            if (view != null) {  
                boolean abort = false;  

                synchronized(mWindowMap) {  
                    if (wtoken.removed || wtoken.startingData == null) {  
                        // If the window was successfully added, then  
                        // we need to remove it.  
                        if (wtoken.startingWindow != null) {  
                            ......  
                            wtoken.startingWindow = null;  
                            wtoken.startingData = null;  
                            abort = true;  
                        }  
                    } else {  
                        wtoken.startingView = view;  
                    }  
                    ......  
                }  

                if (abort) {  
                    try {  
                        mPolicy.removeStartingWindow(wtoken.token, view);  
                    } catch (Exception e) {  
                        ......  
                    }  
                }  
            }  
        } break;  

        ......  
    }  
}  
    }  
    
    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

参数msg指向一个Message对象，从前面的Step 2可以知道，它的成员变量what的值等于ADD_STARTING，而成员变量obj指向的是一个AppWindowToken对象。这个AppWindowToken对象描述的就是要创建启动窗口的Activity组件。H类的成员函数handleMessage首先获得该AppWindowToken对象，并且保存在变量wtoken中。

有了AppWindowToken对象wtoken，H类的成员函数handleMessage就可以通过它的成员变量startingData来获得一个StartingData对象，并且保存在变量sd中。StartingData对象sd里面保存了创建启动窗口所需要的参数，因此，H类的成员函数handleMessage就可以通过这些参数来调用外部类WindowManagerService的成员变量mPolicy所指向的一个PhoneWindowManager对象的成员函数addStartingWindow来为AppWindowToken对象wtoken所描述的Activity组件创建启动窗口。

如果能够成功地为AppWindowToken对象wtoken所描述的Activity组件创建启动窗口，那么PhoneWindowManager类的成员函数addStartingWindow就返回该Activity组件的启动窗口的顶层视图。H类的成员函数handleMessage获得这个视图之后，就会将它保存在变量view中。

由于在创建这个启动窗口的过程中，AppWindowToken对象wtoken所描述的Activity组件可能已经被移除，即AppWindowToken对象wtoken的成员变量removed的值等于true，或者它的启动窗口已经被转移给另外一个Activity组件了，即AppWindowToken对象wtoken的成员变量startingData的值等于null。在这两种情况下，如果AppWindowToken对象wtoken的成员变量startingWindow的值不等于null，那么就说明前面不仅成功地为AppWindowToken对象wtoken所描述的Activity组件创建了启动窗口，并且这个启动窗口也已经成功地增加到WindowManagerService服务中去了，因此，就需要将该启动窗口从WindowManagerService服务中删除，这是通过调用外部类WindowManagerService的成员变量mPolicy所指向的一个PhoneWindowManager对象的成员函数removeStartingWindow来实现的。注意，在删除之前，还会将AppWindowToken对象wtoken的成员变量startingWindow和startingData的值均设置为null，以表示它所描述的Activity组件没有一个关联的启动窗口。

另一方面，如果AppWindowToken对象wtoken所描述的Activity组件没有被移除，并且它的启动窗口了没有转移给另外一个Activity组件，那么H类的成员函数handleMessage就会将前面得到的启动窗口的顶层视图保存在AppWindowToken对象wtoken的成员变量startingView中。注意，这时候AppWindowToken对象wtoken的成员变量startingWindow会指向一个WindowState对象，这个WindowState对象是由PhoneWindowManager类的成员函数addStartingWindow请求WindowManagerService服务创建的。

接下来，我们就继续分析PhoneWindowManager类的成员函数addStartingWindow的实现，以便可以了解它是如何为一个Activity组件创建一个启动窗口，并且将这个启动窗口增加到WindowManagerService服务中去的。

### Step 5. PhoneWindowManager.addStartingWindow

```java
public class PhoneWindowManager implements WindowManagerPolicy {  
    ......  

    static final boolean SHOW_STARTING_ANIMATIONS = true;  
    ......  

    public View addStartingWindow(IBinder appToken, String packageName,  
                          int theme, CharSequence nonLocalizedLabel,  
                          int labelRes, int icon) {  
if (!SHOW_STARTING_ANIMATIONS) {  
    return null;  
}  
if (packageName == null) {  
    return null;  
}  

try {  
    Context context = mContext;  
    boolean setTheme = false;  
    ......  
    if (theme != 0 || labelRes != 0) {  
        try {  
            context = context.createPackageContext(packageName, 0);  
            if (theme != 0) {  
                context.setTheme(theme);  
                setTheme = true;  
            }  
        } catch (PackageManager.NameNotFoundException e) {  
            // Ignore  
        }  
    }  
    if (!setTheme) {  
        context.setTheme(com.android.internal.R.style.Theme);  
    }  

    Window win = PolicyManager.makeNewWindow(context);  
    if (win.getWindowStyle().getBoolean(  
            com.android.internal.R.styleable.Window_windowDisablePreview, false)) {  
        return null;  
    }  

    Resources r = context.getResources();  
    win.setTitle(r.getText(labelRes, nonLocalizedLabel));  

    win.setType(  
        WindowManager.LayoutParams.TYPE_APPLICATION_STARTING);  
    // Force the window flags: this is a fake window, so it is not really  
    // touchable or focusable by the user.  We also add in the ALT_FOCUSABLE_IM  
    // flag because we do know that the next window will take input  
    // focus, so we want to get the IME window up on top of us right away.  
    win.setFlags(  
        WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE|  
        WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE|  
        WindowManager.LayoutParams.FLAG_ALT_FOCUSABLE_IM,  
        WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE|  
        WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE|  
        WindowManager.LayoutParams.FLAG_ALT_FOCUSABLE_IM);  

    win.setLayout(WindowManager.LayoutParams.MATCH_PARENT,  
                        WindowManager.LayoutParams.MATCH_PARENT);  

    final WindowManager.LayoutParams params = win.getAttributes();  
    params.token = appToken;  
    params.packageName = packageName;  
    params.windowAnimations = win.getWindowStyle().getResourceId(  
            com.android.internal.R.styleable.Window_windowAnimationStyle, 0);  
    params.setTitle("Starting " + packageName);  

    WindowManagerImpl wm = (WindowManagerImpl)  
            context.getSystemService(Context.WINDOW_SERVICE);  
    View view = win.getDecorView();  

    if (win.isFloating()) {  
        // Whoops, there is no way to display an animation/preview  
        // of such a thing!  After all that work...  let's skip it.  
        // (Note that we must do this here because it is in  
        // getDecorView() where the theme is evaluated...  maybe  
        // we should peek the floating attribute from the theme  
        // earlier.)  
        return null;  
    }  

    ......  

    wm.addView(view, params);  

    // Only return the view if it was successfully added to the  
    // window manager... which we can tell by it having a parent.  
    return view.getParent() != null ? view : null;  
} catch (WindowManagerImpl.BadTokenException e) {  
    // ignore  
    ......  
} catch (RuntimeException e) {  
    // don't crash if something else bad happens, for example a  
    // failure loading resources because we are loading from an app  
    // on external storage that has been unmounted.  
    ......  
}  

return null;  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindowManager.java中。

PhoneWindowManager类有一个静态成员变量SHOW_STARTING_ANIMATIONS。如果它的值等于false，那么就禁止显示Activity组件的启动窗口。此外，如果参数packageName的值等于null，那么禁止显示Activity组件的启动窗口，即调用者需要指定要显示启动窗口的Activity组件的包名称。

如果参数theme和labelRes的值不等于0，那么就说明调用者指定了启动窗口的主题和标题，这时候就需要创建一个代表了要显示启动窗口的Activity组件的运行上下文，以便可以从里面获得指定的主题和标题。否则的话，启动窗口的运行上下文就是通过PhoneWindowManager类的成员变量mContext来描述的。PhoneWindowManager类的成员变量mContext是在其构造函数中初始化的，它描述的WindowManagerService服务的运行上下文。注意，如果调用者没有指定启动窗口的主题，那么默认使用的主题就为com.android.internal.R.style.Theme。

初始化好启动窗口的运行上下文之后，即初始化好变量context所指向的一个Context对象之后，PhoneWindowManager类的成员函数addStartingWindow接下来就可以以它来参数来调用PolicyManager类的成员函数makeNewWindow来创建一个窗口了，并且将这个窗口保存在变量win中。从前面[Android应用程序窗口（Activity）的窗口对象（Window）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8223770)一文可以知道，PolicyManager类的成员函数makeNewWindow创建的是一个类型为PhoneWindow的窗口。注意，如果这个类型为PhoneWindow的窗口的com.android.internal.R.styleable.Window_windowDisablePreview属性的值等于true，那么就说明不允许为参数appToken所描述的Activity组件显示启动窗口。

PolicyManager类的成员函数makeNewWindow接下来还会继续设置前面所创建的窗口win的以下属性：

窗口类型：设置为WindowManager.LayoutParams.TYPE_APPLICATION_STARTING，即设置为启动窗口类型；

窗口标题：由参数labelRes、nonLocalizedLabel，以及窗口的运行上下文context来确定；

窗口标志：分别将indowManager.LayoutParams.FLAG_NOT_TOUCHABLE、WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE和WindowManager.LayoutParams.FLAG_ALT_FOCUSABLE_IM位设置为1，即不可接受触摸事件和不可获得焦点，但是可以接受输入法窗口；

窗口大小：设置为WindowManager.LayoutParams.MATCH_PARENT，即与父窗口一样大，但是由于这是一个顶层窗口，因此实际上是指与屏幕一样大；

布局参数：包括窗口所对应的窗口令牌（token）和包名（packageName），以及窗口所使用的动画类型（windowAnimations）和标题（title）。

从第5点可以看出，一个Activity组件的启动窗口和它本身的窗口都是对应同一个窗口令牌的，因此， 它们在窗口堆栈中就属于同一组窗口。

设置好窗口win的属性之后，接下来调用它的成员函数getDecorView就可以将它的顶层视图创建出来，并且保存在变量view中。从前面[Android应用程序窗口（Activity）的视图对象（View）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8245546)一文可以知道，这个顶层视图的类型为DecorView。

从前面[Android应用程序窗口（Activity）实现框架简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/8170307)一文可以知道，每一个进程都有一个本地窗口管理服务，这个本地窗口管理服务是由WindowManagerImpl类来实现的，负责维护进程内的所有窗口的视图对象。通过调用WindowManagerImpl类的成员函数addView就可以将一个窗口的视图增加到本地窗口管理服务中去，以便这个视图可以接受本地窗口管理服务的管理。同时，WindowManagerImpl类的成员函数addView还会请求WindowManagerService服务为其所增加的窗口视图创建一个WindowState对象，以便WindowManagerService服务可以维护对应的窗口的运行状态。

注意，如果前面所创建的窗口win是一个浮动的窗口，即它的成员函数isFloating的值等于true，那么窗口win就不能作为启动窗口来使用。这里之所以要在创建了窗口win的顶层视图之后，再来判断窗口win是否是一个浮动窗口，是因为一个窗口只有在创建了顶层视图之后，才能确定是否是浮动窗口。

如果能够成功地将前面所创建的启动窗口win的顶层视图view增加到本地窗口管理服务中，那么顶层视图view就会有一个父视图，即它的成员函数getParent的返回值不等于null，这时候PhoneWindowManager类的成员函数addStartingWindow就会将顶层视图view返回给调用者，表示已经成功地为参数appToken所描述的Activity组件创建了一个启动窗口。从前面[Android应用程序窗口（Activity）的视图对象（View）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8245546)一文可以知道，一个窗口的顶层视图的父视图就是通过一个ViewRoot对象来描述的，这个ViewRoot对象负责渲染它的子视图的UI，以及和WindowManagerService服务通信。

从上面的描述就可以知道，一个Activity组件的启动窗口在创建完成之后，就会通过调用WindowManagerImpl类的成员函数addView来将其增加到WindowManagerService服务中去，因此，接下来我们就继续分析WindowManagerImpl类的成员函数addView的实现。

### Step 6. WindowManagerImpl.addView

这个函数定义在文件frameworks/base/core/java/android/view/WindowManagerImpl.java中，它的具体实现可以参考前面[Android应用程序窗口（Activity）的视图对象（View）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8245546)一文。总的来说，这一步最终会通过调用一个实现了IWindowSession接口的Binder代理对象的成员函数add向WindowManagerService服务发送一个Binder进程间通信请求，这个Binder进程间通信请求最终是由WindowManagerService类的成员函数addWindow来处理的，即为指定的窗口创建一个WindowState对象，以便WindowManagerService服务可以维护它的状态。这个过程又可以参考前面[Android应用程序窗口（Activity）与WindowManagerService服务的连接过程分析](http://blog.csdn.net/luoshengyang/article/details/8275938)一文。

这一步执行完成之后，一个新创建的Activity组件的启动窗口就增加到WindowManagerService服务中去了，这样，WindowManagerService服务就可以下次刷新系统UI时，将该启动窗口显示出来。

接下来我们再继续分析Activity组件的启动窗口结束显示的过程。

## Activity组件的启动窗口的结束过程

Activity组件启动完成之后，它的窗口就会显示出来，这时候WindowManagerService服务就需要将它的启动窗口结束掉。我们知道，在WindowManagerService服务中，每一个窗口都对应有一个WindowState对象。每当WindowManagerService服务需要显示一个窗口的时候，就会调用一个对应的WindowState对象的成员函数performShowLocked。WindowState类的成员函数performShowLocked在执行的过程中，就会检查当前正在处理的WindowState对象所描述的窗口是否设置有启动窗口。如果有的话，那么就会将它结束掉。这个过程如图3所示。

![img](http://img.my.csdn.net/uploads/201302/14/1360774345_7357.jpg)

图3 Activity组件的启动窗口的结束过程

这个过程可以分为13个步骤，接下来我们就详细分析每一个步骤。

### Step 1. WindowState.performShowLocked

```java
public class WindowManagerService extends IWindowManager.Stub  
implements Watchdog.Monitor {  
    ......  

    private final class WindowState implements WindowManagerPolicy.WindowState {  
......  

// This must be called while inside a transaction.  
boolean performShowLocked() {  
    ......  

    if (mReadyToShow && isReadyForDisplay()) {  
        ......  

        if (!showSurfaceRobustlyLocked(this)) {  
            return false;  
        }  
        mLastAlpha = -1;  
        mHasDrawn = true;  
        mLastHidden = false;  
        mReadyToShow = false;  
        enableScreenIfNeededLocked();  

        applyEnterAnimationLocked(this);  

        int i = mChildWindows.size();  
        while (i > 0) {  
            i--;  
            WindowState c = mChildWindows.get(i);  
            if (c.mAttachedHidden) {  
                c.mAttachedHidden = false;  
                if (c.mSurface != null) {  
                    c.performShowLocked();  
                    // It hadn't been shown, which means layout not  
                    // performed on it, so now we want to make sure to  
                    // do a layout.  If called from within the transaction  
                    // loop, this will cause it to restart with a new  
                    // layout.  
                    mLayoutNeeded = true;  
                }  
            }  
        }  

        if (mAttrs.type != TYPE_APPLICATION_STARTING  
                && mAppToken != null) {  
            mAppToken.firstWindowDrawn = true;  

            if (mAppToken.startingData != null) {  
                ......  
                // If this initial window is animating, stop it -- we  
                // will do an animation to reveal it from behind the  
                // starting window, so there is no need for it to also  
                // be doing its own stuff.  
                if (mAnimation != null) {  
                    mAnimation = null;  
                    // Make sure we clean up the animation.  
                    mAnimating = true;  
                }  
                mFinishedStarting.add(mAppToken);  
                mH.sendEmptyMessage(H.FINISHED_STARTING);  
            }  
            mAppToken.updateReportedVisibilityLocked();  
        }  
    }  
    return true;  
}  

......  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

一个窗口只有在准备就绪显示之后，才能调用WindowState类的成员函数performShowLocked来显示它。一个窗口在什么情况下才是准备就绪显示的呢？一是用来描述它的一个WindowState对象的成员变量mReadyToShow的值等于true，二是调用该WindowState对象的成员函数isReadyForDisplay的返回值也等于true。WindowState类的成员函数isReadyForDisplay主要验证一个窗口中是否是处于可见状态、并且不是处于销毁的状态。一个窗口必须要处于可见状态，并且不是正在被销毁，那么它才是准备就绪显示的。此外，一个窗口如果是一个子窗口，那么只有当它以及它的父窗口都是可见的，那么它才是处于可见状态的。

通过了上面的检查之后，WindowState类的成员函数performShowLocked就会调用WindowManagerService类的成员函数showSurfaceRobustlyLocked来通知SurfaceFlinger服务来显示当前正在处理的窗口。

一旦WindowManagerService类的成员函数showSurfaceRobustlyLocked成功地通知了SurfaceFlinger服务来显示当前正在处理的窗口，那么它的返回值就会等于true，接下来WindowState类的成员函数performShowLocked还会对当前正在正在处理的窗口执行以下操作：

将对应的WindowState对象的成员变量mLastAlpha的值设置为-1，以便以后在显示窗口之前，都可以更新窗口的Alpha通道值。

将对应的WindowState对象的成员变量mHasDrawn的值设置为true，以便可以表示窗口的UI绘制出来了。

将对应的WindowState对象的成员变量mLastHidden的值设置为false，以便可以表示窗口当前是可见的。

将对应的WindowState对象的成员变量mReadyToShow的值设置为false，以便可以表示窗口已经显示过了，避免重复地请求SurfaceFlinger服务显示没有发生变化的窗口。

确保屏幕对WindowManagerService服务是可用的，这是通过调用WindowManagerService类的成员函数enableScreenIfNeededLocked来实现的。系统在启动完成之前，屏幕是用来显示开机动画的，这时候屏幕是被开机动画占用的。等到系统启动完成之后，屏幕就应该是被WindowManagerService服务占用的，这时候就需要停止显示开机动画。WindowManagerService类的成员函数enableScreenIfNeededLocked就是确保开机动画已经停止显示。

给窗口设置一个进入动画或者显示动画，这是通过调用WindowManagerService类的成员函数applyEnterAnimationLocked来实现的。默认是设置为显示动画，但是如果窗口之前是不可见的，那么就会设置为进入动画。

由于当前正在处理的窗口可能有子窗口，因此就需要在显示完成当前正在处理的窗口之后，继续将它的子窗口显示出来。如果一个窗口具有子窗口，那么这些子窗口就会保存在一个对应的WindowState对象的成员变量mChildWindows所描述的一个ArrayList中。注意，只有那些父窗口上一次是不可见的，并且具有绘图表面的子窗口才需要显示。显示子窗口是通过递归调用WindowState类的成员函数performShowLocked来实现的。

最后，如果当前正在处理的窗口是一个Acitivy组件相关的窗口，并且不是Acitivy组件的启动窗口，即当前正在处理的WindowState对象的成员变量mAppToken的值不等于null，并且成员变量mAttrs所指向的一个WindowManager.LayoutParams对象的成员变量type的值不等于TYPE_APPLICATION_STARTING，那么就需要检查该Acitivy组件是否设置有启动窗口。如果设置有启动窗口的话，那么就需要结束显示该启动窗口，因为与该Acitivy组件相关的其它窗口已经显示出来了。

从前面第一部分的内容可以知道，只要当前正在处理的WindowState对象的成员变量mAppToken不等于null，并且它所指向的一个AppWindowToken对象的成员变量startingData的值也不等于null，那么就说明当前正在处理的窗口是一个Acitivy组件相关的窗口，并且这个Acitivy组件设置有一个启动窗口。在这种情况下，WindowState类的成员函数performShowLocked就会调用WindowManagerService类的成员变量mH所指向的一个H对象的成员函数sendEmptyMessage来向WindowManagerService服务所运行在的线程发送一个类型为FINISHED_STARTING的消息，表示要结束显示一个Acitivy组件的启动窗口。在发送这个消息之前，WindowState类的成员函数performShowLocked还会将用来描述要结束显示启动窗口的Activity组件的一个AppWindowToken对象增加到WindowManagerService类的成员变量mFinishedStarting所描述的一个ArrayList中去。

注意，如果这时候当前正在处理的窗口还在显示动画，即当前正在处理的WindowState对象的成员变量mAnimation的值不等于null，那么WindowState类的成员函数performShowLocked就会同时将该动画结束掉，即将当前正在处理的WindowState对象的成员变量mAnimation的值设置为null，但是会将另外一个成员变量mAnimating的值设置为true，以便可以在其它地方对当前正在处理的窗口的动画进行清理。

还有一个地方需要注意的是，如果当前正在处理的窗口是一个Acitivy组件相关的窗口，那么WindowState类的成员函数performShowLocked还会调用当前正在处理的WindowState对象的成员变量mAppToken所指向的一个AppWindowToken对象的成员函数updateReportedVisibilityLocked来向ActivityManagerService服务报告该Acitivy组件的可见性。

接下来，我们就继续分析在WindowManagerService类内部定义的H类的成员函数sendEmptyMessage的实现，以便可以了解Acitivy组件的启动窗口的结束过程。

### Step 2. H.sendEmptyMessage

H类的成员函数sendEmptyMessage是从父类Handler继承下来的，因此，这一步调用的实际上是Handler类的成员函数sendEmptyMessage。Handler类的成员函数sendEmptyMessage用来向消息队列头部的插入一个新的消息，以便这个消息可以下一次消息循环中就能得到处理。在前面[Android应用程序线程消息循环模型分析](http://blog.csdn.net/luoshengyang/article/details/6905587)一文中，我们已经分析过往消息队列发送消息的过程了，这里不再详述。

从上面的调用过程可以知道，这一步所发送的消息的类型为FINISHED_STARTING，并且是向WindowManagerService服务所运行在的线程的消息队列发送的。当这个消息得到处理的时候，H类的成员函数handleMessage就会被调用，因此，接下来我们就继续分析H类的成员函数handleMessage的实现，以便可以了解Activity组件的启动窗口的结束过程。

Step 3. H.handleMessage

```java
public class WindowManagerService extends IWindowManager.Stub  
implements Watchdog.Monitor {  
    ......  

    private final class H extends Handler {  
......  

public void handleMessage(Message msg) {  
    switch (msg.what) {   
        ......  

        case FINISHED_STARTING: {  
            IBinder token = null;  
            View view = null;  
            while (true) {  
                synchronized (mWindowMap) {  
                    final int N = mFinishedStarting.size();  
                    if (N <= 0) {  
                        break;  
                    }  
                    AppWindowToken wtoken = mFinishedStarting.remove(N-1);  

                    ......  

                    if (wtoken.startingWindow == null) {  
                        continue;  
                    }  

                    view = wtoken.startingView;  
                    token = wtoken.token;  
                    wtoken.startingData = null;  
                    wtoken.startingView = null;  
                    wtoken.startingWindow = null;  
                }  

                try {  
                    mPolicy.removeStartingWindow(token, view);  
                } catch (Exception e) {  
                    ......  
                }  
            }  
        } break;  

        ......  
    }  
}  
    }  
    
    ......  
}  
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

从前面的Step 1可以知道，这时候用来描述需要结束启动窗口的Activity组件的AppWindowToken对象都都保存在WindowManagerService类的成员变量mFinishedStarting所描述的一个ArrayList中，因此，通过遍历保存在该ArrayList中的每一个AppWindowToken对象，就可以结束对应的启动窗口。

从前面第一部分的内容可以知道，如果一个Activity组件设置有启动窗口，那么这个启动窗口的顶层视图就会保存在用来描述该Activity组件的一个AppWindowToken对象的成员变量startingView中。获得了一个启动窗口的顶层视图之后，就可以调用WindowManagerService类的成员变量mPolicy所指向的一个PhoneWindowManager对象的成员函数removeStartingWindow来通知WindowManagerService服务删除该启动窗口，从而可以结束显示该启动窗口。

注意，在调用PhoneWindowManager类的成员函数removeStartingWindow来通知WindowManagerService服务删除一个启动窗口之前，还需要将用来描述该启动窗口的宿主Activity组件的一个AppWindowToken对象的成员变量startingData、startingView和startingWindow的值设置为null，因为这三个成员变量保存的数据都是与启动窗口相关的。

接下来，我们就继续分析PhoneWindowManager类的成员函数removeStartingWindow的实现，以便可以了解Activity组件的启动窗口的结束过程。

### Step 4. PhoneWindowManager.removeStartingWindow

```java
public class PhoneWindowManager implements WindowManagerPolicy {  
    ......  

    public void removeStartingWindow(IBinder appToken, View window) {  
......  

if (window != null) {  
    WindowManagerImpl wm = (WindowManagerImpl) mContext.getSystemService(Context.WINDOW_SERVICE);  
    wm.removeView(window);  
}  
    }  

    ......  
}  
```
这个函数定义在文件frameworks/base/policy/src/com/android/internal/policy/impl/PhoneWindowManager.java中。

PhoneWindowManager类的成员函数removeStartingWindow首先获得一个本地窗口管理服务。从前面第一部分的Step 5可以知道，这个本地窗口管理服务是由WindowManagerImpl类来实现的。获得了本地窗口管理服务之后，就可以调用它的成员函数removeView来请求WindowManagerService服务删除参数window所描述的一个启动窗口。

### Step 5. WindowManagerImpl.removeView

```java
public class WindowManagerImpl implements WindowManager {    
    ......    
    
    public void removeView(View view) {  
synchronized (this) {  
    int index = findViewLocked(view, true);  
    View curView = removeViewLocked(index);  
    if (curView == view) {  
        return;  
    }  

    ......  
}  
    }  

    ......   
    
    private View[] mViews;  
    private ViewRoot[] mRoots;  

    ......    
}  
```
这个函数定义在文件frameworks/base/core/java/android/view/WindowManagerImpl.java中。

从前面[Android应用程序窗口（Activity）的视图对象（View）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8245546)一文可以知道， 在应用程序进程中，每一个窗口的顶层视图都对应有一个ViewRoot对象，它们的对应关系是由WindowManagerImpl类来维护的。当前正在处理的进程即为WindowManagerService服务所运行在的进程，也就是System进程，它与普通的应用程序进程一样，也可以创建窗口，即它在内部也会通过WindowManagerImpl类管理窗口的顶层视图及其所对应的ViewRoot对象。

WindowManagerImpl类是如何管理进程中的窗口顶层视图与ViewRoot对象的对应关系的呢？在WindowManagerImpl类中，有两个成员变量mViews和mRoots，它们分别指向两个大小相等的View数组和ViewRoot数组，用来保存进程中的窗口顶层视图对象和ViewRoot对象，其中，第mViews[i]个窗口顶层视图与第mRoots[i]个ViewRoot对象是一一对应的。

WindowManagerImpl类的成员函数removeView首先调用另外一个成员函数findViewLocked来找到参数view所描述的一个启动窗口的顶层视图在数组mViews中的位置index，然后再以这个位置为参数来调用另外一个成员函数removeViewLocked，以便可以删除该启动窗口的顶层视图。

### Step 6. WindowManagerImpl.removeViewLocked

```java
public class WindowManagerImpl implements WindowManager {    
    ......    
    
    View removeViewLocked(int index) {  
ViewRoot root = mRoots[index];  
View view = root.getView();  
......  

root.die(false);  
......  

return view;  
    }  

    ......    
}    
```
这个函数定义在文件frameworks/base/core/java/android/view/WindowManagerImpl.java中。

从前面的调用过程可以知道，WindowManagerImpl类的成员函数removeViewLocked要删除的是在数组mViews中第index个位置的视图，同时，与这个即将要被删除的视图所对应的ViewRoot对象保存在数组mRoots中的第index个位置上。有了这个ViewRoot对象之后，就可以调用它的成员函数die来删除指定的窗口了。

### Step 7. ViewRoot.die

```java
public final class ViewRoot extends Handler implements ViewParent,    
View.AttachInfo.Callbacks {    
    ......    
    
    public void die(boolean immediate) {    
if (immediate) {    
    doDie();    
} else {    
    sendEmptyMessage(DIE);    
}    
    }    

    ......    
}    
```
这个函数定义在frameworks/base/core/java/android/view/ViewRoot.java文件中。

当参数immediate的值等于true的时候，ViewRoot类的成员函数die直接调用另外一个成员函数doDie来删除与当前正在处理的ViewRoot对象所对应的一个View对象，否则的话，ViewRoot类的成员函数die就会先向当前线程的消息队列发送一个类型为DIE消息，然后等到该消息被处理的时候，再调用ViewRoot类的成员函数doDie来删除与当前正在处理的ViewRoot对象所对应的一个View对象。

因此，无论如何，ViewRoot类的成员函数die最终都会调用到另外一个成员函数doDie来来删除与当前正在处理的ViewRoot对象所对应的一个View对象，接下来我们就继续分析它的实现。

### Step 8. ViewRoot.doDie

```java
public final class ViewRoot extends Handler implements ViewParent,    
View.AttachInfo.Callbacks {    
    ......    
    
    void doDie() {    
......    
    
synchronized (this) {    
    ......    
    
    if (mAdded) {    
        mAdded = false;    
        dispatchDetachedFromWindow();    
    }    
}    
    }    

    ......    
}    
```
这个函数定义在frameworks/base/core/java/android/view/ViewRoot.java文件中。

ViewRoot类有一个成员变量mAdded，当它的值等于true的时候，就表示当前正在处理的ViewRoot对象有一个关联的View对象，因此，这时候就可以调用另外一个成员函数dispatchDetachedFromWindow来删除这个View对象。由于删除了这个View对象之后，当前正在处理的ViewRoot对象就不再关联有View对象了，因此，ViewRoot类的成员函数doDie在调用另外一个成员函数dispatchDetachedFromWindow之前，还会将成员变量mAdded的值设置为false。

### Step 9. ViewRoot.dispatchDetachedFromWindow

```java
public final class ViewRoot extends Handler implements ViewParent,    
View.AttachInfo.Callbacks {    
    ......    

    static IWindowSession sWindowSession;  
    ......  

    final W mWindow;  
    ......  
    
    void dispatchDetachedFromWindow() {    
......    
    
try {    
    sWindowSession.remove(mWindow);    
} catch (RemoteException e) {    
}    
    
......    
    }    
    
    ......    
}      
```
这个函数定义在frameworks/base/core/java/android/view/ViewRoot.java文件中。 

从前面[Android应用程序窗口（Activity）与WindowManagerService服务的连接过程分析](http://blog.csdn.net/luoshengyang/article/details/8275938)一文可以知道，每一个与UI相关的应用程序进程，都与WindowManagerService服务建立有一个连接，这个连接是通过一个实现了IWindowSession接口的Binder代理对象来描述的，并且这个Binder代理对象就保存在ViewRoot类的静态成员变量sWindowSession中，它引用了运行在WindowManagerService服务这一侧的一个类型为Session的Binder本地对象。 

注意，由于当前进程即为WindowManagerService服务所运行在的进程，因此，这时候ViewRoot类的静态成员变量sWindowSession保存的其实不是一个实现了IWindowSession接口的Binder代理对象，而是一个实现了IWindowSession接口的类型为Session的Binder本地对象。这是因为Binder驱动发现Client和Service是位于同一个进程的时候，就会将Service的本地接口直接返回给Client，而不会将Service的代理对象返回给Client，这样就可以避免在同一进程中执行Binder进程间调用也会经过Binder驱动来中转。有关Binder进程间通信的内容，可以参考前面[Android进程间通信（IPC）机制Binder简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/6618363)这个系列的文章。

从前面[Android应用程序窗口（Activity）与WindowManagerService服务的连接过程分析](http://blog.csdn.net/luoshengyang/article/details/8275938)一文还可以知道，进程中的每一个窗口都有一个对应的W对象，这个W对象就保存在ViewRoot类的成员变量mWindow中。有了这个W对象之后，ViewRoot类的成员函数dispatchDetachedFromWindow就可以调用静态成员变量sWindowSession所描述的一个Session对象的成员函数remove来通知WindowManagerService服务删除一个对应的WindowState对象。从前面的调用过程可以知道，这个WindowState对象描述的是一个Activity组件的启动窗口，因此，WindowManagerService服务删除了这个WindowState对象之后，就相当于是将一个Activity组件的启动窗口结束掉了。

接下来，我们就继续分析Session类的成员函数remove的实现，以便可以了解Activity组件的启动窗口的结束过程。

### Step 10. Session.remove

```java
public class WindowManagerService extends IWindowManager.Stub    
implements Watchdog.Monitor {    
    ......    
    
    private final class Session extends IWindowSession.Stub    
    implements IBinder.DeathRecipient {    
......    
    
public void remove(IWindow window) {    
    removeWindow(this, window);    
}    
    
......    
    }    
    
    ......    
}    
```
这个函数定义在frameworks/base/services/java/com/android/server/WindowManagerService.java文件中。

Session类的成员函数remove的实现很简单，它只是调用外部类WindowManagerService的成员函数removeWindow来执行参数window所描述的一个窗口的删除操作。

### Step 11. WindowManagerService.removeWindow

```java
public class WindowManagerService extends IWindowManager.Stub    
implements Watchdog.Monitor {    
    ......    
    
    public void removeWindow(Session session, IWindow client) {    
synchronized(mWindowMap) {    
    WindowState win = windowForClientLocked(session, client, false);    
    if (win == null) {    
        return;    
    }    
    removeWindowLocked(session, win);    
}    
    }    
    
    ......    
}    
```
这个函数定义在frameworks/base/services/java/com/android/server/WindowManagerService.java文件中。

WindowManagerService类的成员函数removeWindow首先调用成员函数windowForClientLocked来找到与参数client所对应的一个WindowState对象，接着再调用成员函数removeWindowLocked来删除这个WindowState对象，以便可以结束掉这个WindowState对象所描述的一个启动窗口。

### Step 12. WindowManagerService.removeWindowLocked

```java
public class WindowManagerService extends IWindowManager.Stub    
implements Watchdog.Monitor {    
    ......    

    public void removeWindowLocked(Session session, WindowState win) {  
......  

// Visibility of the removed window. Will be used later to update orientation later on.  
boolean wasVisible = false;  
// First, see if we need to run an animation.  If we do, we have  
// to hold off on removing the window until the animation is done.  
// If the display is frozen, just remove immediately, since the  
// animation wouldn't be seen.  
if (win.mSurface != null && !mDisplayFrozen && mPolicy.isScreenOn()) {  
    // If we are not currently running the exit animation, we  
    // need to see about starting one.  
    if (wasVisible=win.isWinVisibleLw()) {  

        int transit = WindowManagerPolicy.TRANSIT_EXIT;  
        if (win.getAttrs().type == TYPE_APPLICATION_STARTING) {  
            transit = WindowManagerPolicy.TRANSIT_PREVIEW_DONE;  
        }  
        // Try starting an animation.  
        if (applyAnimationLocked(win, transit, false)) {  
            win.mExiting = true;  
        }  
    }  
    if (win.mExiting || win.isAnimating()) {  
        // The exit animation is running... wait for it!  
        ......  
        win.mExiting = true;  
        win.mRemoveOnExit = true;  
        mLayoutNeeded = true;  
        updateFocusedWindowLocked(UPDATE_FOCUS_WILL_PLACE_SURFACES);  
        performLayoutAndPlaceSurfacesLocked();  
        if (win.mAppToken != null) {  
            win.mAppToken.updateReportedVisibilityLocked();  
        }  
        ......  
        return;  
    }  
}  

removeWindowInnerLocked(session, win);  
......  

updateFocusedWindowLocked(UPDATE_FOCUS_NORMAL);  
......  
    }  

    ......    
}    
```
这个函数定义在frameworks/base/services/java/com/android/server/WindowManagerService.java文件中。

WindowManagerService类的成员函数removeWindowLocked在删除参数win所描述的一个窗口之前，首先检查是否需要对该窗口设置一个退出动画。只要满足以下四个条件，那么就需要对参数win所描述的一个窗口设置退出动画：

参数win所描述的一个窗口具有绘图表面，即它的成员变量mSurface的值不等于null；

系统屏幕当前没有被冻结，即WindowManagerService类的成员变量mDisplayFrozen的值等于false；

系统屏幕当前是点亮的，即WindowManagerService类的成员变量mPolicy所指向的一个PhoneWindowManager对象的成员函数isScreenOn的返回值为true；

参数win所描述的一个窗口当前是可见的，即它的成员函数isWinVisibleLw的返回值等于true。

对参数win所描述的一个窗口设置退出动画是通过调用WindowManagerService类的成员函数applyAnimationLocked来实现的。注意，如果参数win描述的是一个启动窗口，那么退出动画的类型就为WindowManagerPolicy.TRANSIT_PREVIEW_DONE，否则的话，退出动画的类型就为WindowManagerPolicy.TRANSIT_EXIT。

一旦参数win所描述的一个窗口正处于退出动画或者其它动画状态，即它的成员变量mExiting的值等于true或者成员函数isAnimating的返回值等于true，那么WindowManagerService服务就要等它的动画显示完成之后，再删除它，这是通过将它的成员变量mExiting和mRemoveOnExit的值设置为true来完成的。由于这时候还需要显示参数win所描述的一个窗口的退出动画或者其它动画，因此，WindowManagerService类的成员函数removeWindowLocked在返回之前，还需要执行以下操作：

调用WindowManagerService类的成员函数updateFocusedWindowLocked来重新计算系统当前需要获得焦点的窗口；

调用WindowManagerService类的成员函数performLayoutAndPlaceSurfacesLocked来重新布局和刷新系统的UI；

如果参数win所描述的一个与Activity组件相关的窗口，即它的成员变量mAppToken的值不等于null，那么就会调用这个成员变量所指向的一个AppWindowToken对象的成员函数updateReportedVisibilityLocked来向ActivityManagerService服务报告该Activity组件的可见性。

如果不需要对参数win所描述的一个窗口设置退出动画，那么WindowManagerService类的成员函数removeWindowLocked就会直接调用成员函数removeWindowInnerLocked来删除该窗口，并且在删除了该窗口之后，调用成员函数updateFocusedWindowLocked来重新计算系统当前需要获得焦点的窗口以及重新布局和刷新系统的UI。

接下来，我们就继续分析WindowManagerService类的成员函数removeWindowLocked的实现，以便可以了解Activity组件的启动窗口的结束过程。

### Step 13. WindowManagerService.removeWindowLocked

```java
public class WindowManagerService extends IWindowManager.Stub    
implements Watchdog.Monitor {    
    ......    

    private void removeWindowInnerLocked(Session session, WindowState win) {  
win.mRemoved = true;  

if (mInputMethodTarget == win) {  
    moveInputMethodWindowsIfNeededLocked(false);  
}  
......  

mPolicy.removeWindowLw(win);  
win.removeLocked();  

mWindowMap.remove(win.mClient.asBinder());  
mWindows.remove(win);  
mWindowsChanged = true;  
......  

if (mInputMethodWindow == win) {  
    mInputMethodWindow = null;  
} else if (win.mAttrs.type == TYPE_INPUT_METHOD_DIALOG) {  
    mInputMethodDialogs.remove(win);  
}  

final WindowToken token = win.mToken;  
final AppWindowToken atoken = win.mAppToken;  
token.windows.remove(win);  
if (atoken != null) {  
    atoken.allAppWindows.remove(win);  
}  
......  

if (token.windows.size() == 0) {  
    if (!token.explicit) {  
        mTokenMap.remove(token.token);  
        mTokenList.remove(token);  
    } else if (atoken != null) {  
        atoken.firstWindowDrawn = false;  
    }  
}  

if (atoken != null) {  
    if (atoken.startingWindow == win) {  
        atoken.startingWindow = null;  
    } else if (atoken.allAppWindows.size() == 0 && atoken.startingData != null) {  
        // If this is the last window and we had requested a starting  
        // transition window, well there is no point now.  
        atoken.startingData = null;  
    } else if (atoken.allAppWindows.size() == 1 && atoken.startingView != null) {  
        // If this is the last window except for a starting transition  
        // window, we need to get rid of the starting transition.  
        ......  
        Message m = mH.obtainMessage(H.REMOVE_STARTING, atoken);  
        mH.sendMessage(m);  
    }  
}  

if (win.mAttrs.type == TYPE_WALLPAPER) {  
    ......  
    adjustWallpaperWindowsLocked();  
} else if ((win.mAttrs.flags&FLAG_SHOW_WALLPAPER) != 0) {  
    adjustWallpaperWindowsLocked();  
}  

if (!mInLayout) {  
    assignLayersLocked();  
    mLayoutNeeded = true;  
    performLayoutAndPlaceSurfacesLocked();  
    if (win.mAppToken != null) {  
        win.mAppToken.updateReportedVisibilityLocked();  
    }  
}  

......  
    }  

    ......    
}      
```
这个函数定义在文件frameworks/base/services/java/com/android/server/WindowManagerService.java中。

由于参数win所描述的一个窗口马上就要被删除了，因此，WindowManagerService类的成员函数removeWindowLocked首先就将它的成员变量mRemoved的值设置为true。此外，如果参数win所描述的窗口是系统输入法的目标窗口，那么还需要调用WindowManagerService类的成员函数moveInputMethodWindowsIfNeededLocked来重新移动动系统输入法窗口到其它可能需要输入法的窗口的上面去。

执行完成以上两个操作之后，WindowManagerService类的成员函数removeWindowLocked接下来就可以对参数win所描述的一个窗口进行清理了，包括：

调用WindowManagerService类的成员变量mPolicy的成员函数removeWindowLw来通知窗口管理策略类PhoneWindowManager，参数win所描述的一个窗口被删除了；

调用参数win所指向的一个WindowState对象的成员函数removeLocked来执行自身的清理工作；

将参数win所指向的一个WindowState对象从WindowManagerService类的成员变量mWindowMap和mWindows中删除，即将参数win所描述的一个窗口从窗口堆栈中删除。

执行完成以上三个清理工作之后，窗口堆栈就发生变化了，因此，就需要将WindowManagerService类的成员变量mWindowsChanged的值设置为true。

接下来，WindowManagerService类的成员函数removeWindowLocked还会检查前面被删除的窗口是否是一个输入法窗口或者一个输入法对话框。如果是一个输入法窗口，那么就会将WindowManagerService类的成员变量mInputMethodWindow的值设置为true；如果是一个输入法对话框，那么就会它从WindowManagerService类的成员变量mInputMethodDialogs所描述的一个输入法对话框列表中删除。

WindowManagerService类的成员函数removeWindowLocked的任务还没有完成，它还需要继续从参数win所描述的一个窗口从它的窗口令牌的窗口列表中删除。参数win所描述的一个窗口的窗口令牌保存在它的成员变量mToken中，这个成员变量指向的是一个WindowToken对象。这个WindowToken对象有一个成员变量windows，它指向的是一个ArrayList中。这个ArrayList即为参数win所描述的一个窗口从它的窗口令牌的窗口列表，因此，将参数win所描述的一个窗口从这个窗口列表中删除即可。

如果参数win描述的一个是与Activity组件有关的窗口，那么它的成员变量mAppToken就会指向一个AppWindowToken对象。这个AppWindowToken对象的成员变量allAppWindows所指向的一个ArrayList也会保存有参数win所描述的窗口。因此，这时候也需要将参数win所描述的一个窗口从这个ArrayList中删除。

参数win所描述的一个窗口被删除了以后，与它所对应的窗口令牌的窗口数量就会减少1。如果一个窗口令牌的窗口数量减少1之后变成0，那么就需要考虑将这个窗口令牌从WindowManagerService服务的窗口令牌列表中删除了，即从WindowManagerService类的成员变量mTokenMap和mTokenList中删除，前提是这个窗口令牌不是显式地被增加到WindowManagerService服务中去的，即用来描述这个窗口令牌的一个WindowToken对象的成员变量explicit的值等于false。

另一方面，如果参数win描述的一个是与Activity组件有关的窗口，并且当它被删除之后，与该Activity组件有关的窗口的数量变为0，那么就需要将用来描述该Activity组件的一个AppWindowToken对象的成员变量firstWindowDrawn的值设置为false，以表示该Activity组件的第一个窗口还没有被显示出来，事实上也是表示目前没有窗口与该Activity组件对应。

当参数win描述的一个是与Activity组件有关的窗口的时候，WindowManagerService类的成员函数removeWindowLocked还需要检查该Activity组件是否设置有启动窗口。如果该Activity组件设置有启动窗口的话，那么就需要对它的相应成员变量进行清理。这些检查以及清理工作包括：

如果参数win所描述的窗口即为一个Activity组件的窗口，即它的值等于用来描述与它的宿主Activity组件的一个AppWindowToken对象的成员变量startingWindow的值，那么就需要将AppWindowToken对象的成员变量startingWindow的值设置为null，以便可以表示它所描述的Activity组件的启动窗口已经结束了；

如果删除了参数win所描述的窗口之后，它的宿主Activity组件的窗品数量为0，但是该Activity组件又正在准备显示启动窗口，即用来描述该Activity组件的一个AppWindowToken对象的成员变量startingData的值不等于null，那么就说明这个启动窗口接下来也没有必要显示了，因此，就需要将该AppWindowToken对象的成员变量startingData的值设置为null；

如果删除了参数win所描述的窗口之后，它的宿主Activity组件的窗品数量为1，并且用来描述该Activity组件的一个AppWindowToken对象的成员变量startingView的值不等于null，那么就说明该Activity组件剩下的最后一个窗口即为它的启动窗口，这时候就需要请求WindowManagerService服务结束掉这个启动窗口，因为已经没有必要显示了。

当一个Activity组件剩下的窗口只有一个，并且用来描述该Activity组件的一个AppWindowToken对象的成员变量startingView的值不等于null时，我们是如何知道这个剩下的窗口就是该Activity组件的启动窗口的呢？从前面第一个部分的内容可以知道，当一个Activity组件的启动窗口被创建出来之后，它的顶层视图就会保存在用来描述该Activity组件的一个AppWindowToken对象的成员变量startingView中。因此，如果Activity组件满足上述两个条件，我们就可以判断出它所剩下的一个窗口即为它的启动窗口。注意，在这种情况下，WindowManagerService类的成员函数removeWindowLocked不是马上删除这个启动窗口的，而是通过向WindowManagerService服务所运行在的线程发送一个类型为REMOVE_STARTING的消息，等到该消息被处理时再来删除这个启动窗口。

清理了窗口win的宿主Activity组件的启动窗口相关的数据之后，WindowManagerService类的成员函数removeWindowLocked又继续检查窗口win是否是一个壁纸窗口或者一个显示壁纸的窗口。如果是的话，那么就需要调用WindowManagerService类的成员函数adjustWallpaperWindowsLocked来重新调整系统中的壁纸窗口在窗口堆栈中的位置，即将它们移动到下一个可能需要显示壁纸窗口的其它窗口的下面去。

WindowManagerService类的成员函数removeWindowLocked的最后一个任务是检查WindowManagerService服务当前是否正处于重新布局窗口的状态，即判断WindowManagerService类的成员变量mInLayout的值是否等于true。如果不等于true的话，那么就需要调用WindowManagerService类的成员函数performLayoutAndPlaceSurfacesLocked来重新布局窗口，实际上就是刷新系统的UI。

注意，WindowManagerService类的成员函数removeWindowLocked在重新布局系统中的窗口之前，还需要调用另外一个成员函数assignLayersLocked来重新计算系统中的所有窗口的Z轴位置了。此外，WindowManagerService类的成员函数removeWindowLocked在重新布局了系统中的窗口之后，如果发现前面被删除的窗口win是一个与Activity组件相关的窗口，即它的成员变量mAppToken的值不等于null，那么还会调用这个成员变量所指向的一个AppWindowToken对象的成员函数updateReportedVisibilityLocked来向ActivityManagerService服务报告该Activity组件的可见性。

这一步执行完成之后，一个的Activity组件的启动窗口结束掉了。至此，我们就分析完成Activity组件的启动窗口的启动过程和结束过程了。事实上，一个Activity组件在启动的过程中，除了可能需要显示启动窗口之外，还需要与系统当前激活的Activity组件执行一个切换操作，然后才可以将自己的窗口显示出来。在接下来的一篇文章中，我们就将继续分析Activity组件的切换过程，敬请关注！