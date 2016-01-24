---
layout: post
title:  Working with overlays - Life under WindowManager
date:   2016-01-15 10:15:20
categories: Android
---

This is the start of a series of articles regarding creating and using overlays on Android.  An overlay is a view that is placed above other views.  Such views can be created on Android with several different ways and for different purposes.

There are several usages for overlays on Android. Some of them include:

<ol>
<li>Floating buttons/menus</li>
<li>Onboarding tutorials</li>
<li>Advertising</li>
<li>Augmented Reality</li>
<li>Fancy animations</li>
</ol>



<h2>Life under WindowManager</h2>

In this article we will see how we can create an overlay view that will lie on top of other applications with the help of WindowManager.

But let's see first what is a <a href="http://developer.android.com/reference/android/view/WindowManager.html" target="_blank">WindowManager.</a> WindowManager is a system service responsible for organizing the screen. It interacts with applications to create windows and decides where they will go on the screen and in which order. When creating a window, WindowManager requests a Surface for that window and on that Surface, an application will draw its user interface. Each window is a container of views and has its own view hierarchy.


<u>But let's be more specific and list what a WindowManager is and what it does:</u>
<ul>
<li>A system service </li>
<li>It requests the creation and allocation of surfaces for the clients (applications)</li>
<li>Is responsible of which windows are visible, their z-order and how they are laid on the screen </li>
<li>It deals and dispatches input and focus events to clients </li>
<li>It <a href="https://source.android.com/devices/graphics/" target="_blank">oversees</a> transitions, screen orientation and animations </li>
<li>It sends all of the window metadata to SurfaceFlinger so SurfaceFlinger can use that data to composite surfaces on the display.</li>
<li>Each instance of WindowManager is bound to a particular Display </li>
<li>WindowManager never deals with the bits of the created surfaces, it is up to the clients to do so, and they do it directly without going through the WindowManager </li>
</ul>


During the windows creation process and the management of their lifecycle we will meet the term of Window Token a special type of Binder object. A Binder is an Android-specific interprocess communication mechanism and also remote method invocation system. In Android framework Window Tokens, and Binder objects in general, are used extensively for security reasons. A Window Token uniquely identifies a window, and is used by WindowManager to decide whether or not a window is allowed to be placed at a requested position. Applications can obtain their own window tokens, but they do not have access to the tokens of other applications. This way, the framework can constrain an app to its own sandbox and does not allow applications to draw windows above other applications.

For example, during an application launch through ActivityMangerService, a Window Token (that is called Application Window Token) is created and is passed both to the application and the WindowManager. This token identifies uniquely the top window view of the application. When the application wants to add or remove windows it passes this token to WindowManager which decides if they can be created at the requested place and where they should go. This way when the ActivityManager wants to close or hide all windows created by the activity, it can easily request this by providing the relevant token to the WindowManager.


<u>In general we can say that we have two types of Window Tokens:</u>
<ul>
<li><b>AppWindowToken</b> which is the handle for an Activity that it uses to display windows. It’s a specific version of WindowToken relevant to  a particular application or activity that is displaying windows</li>
<li><b>WindowToken</b> which is the container of a set of related windows in the WindowManager.  For nested windows, there is a WindowToken created for  the parent window to manage its children. </li>
</ul>


You can read more about window tokens and how they are handled, in the following articles: <a heref="http://www.androiddesignpatterns.com/2013/07/binders-window-tokens.html" >"Binders & Window Tokens" </a> and <a heref="http://blog.csdn.net/luoshengyang/article/details/8275938" >"Android application window (Activity) view objects (View) creation process analysis" </a>.


<u>There are 3 main classes of windows:</u>
<ul>
<li><b>Application windows</b> are normal top level windows. For these types of windows, their token must be set to the token of the activity they are a part of. </li>
<li><b>Sub-windows</b> are windows that are associated with another top-level window. These kind of windows have the token of the window they are attached to. </li>
<li><b>System windows</b> are special types of windows for use by the system for specific purposes like status bar, search bar, toast notifications and others. They are not normally used by applications and a special permission is required to use them. This kind of windows can overlay above other applications. For creating and using system windows, prior the release of Android Marshmallow it was only necessary to include in Android Manifest the permission SYSTEM\_ALERT\_WINDOW </li>
</ul>
```
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
```

However since Android Marshmallow a special request is required with a direct action by the user in order to allow an app to create and add or remove windows above other apps (System windows). This permission is not included in the new request permission structure that was introduced in Marshmallow and follows a completely different approach. One of its major differences is that it cannot be granted through the app like all the other permissions.

Since Android 6.0 developers can call <b><a href="http://developer.android.com/reference/android/provider/Settings.html#canDrawOverlays(android.content.Context)" target="_blank">Settings.canDrawOverlays() </a></b> to check if the specific context was granted permission in order to draw on top of other apps. If permission hasn't been granted yet, user can create and fire an Intent with destination set to <b>Settings.ACTION\_MANAGE\_OVERLAY\_PERMISSION</b> accompanied with URI of the package name of the app to send users directly to give permission to your app yo draw above others.

```java
public static int OVERLAY_PERMISSION_CODE = 2525;  

public void addOverlay() {     
	if (!Settings.canDrawOverlays(this)) {         
			Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION,Uri.parse("package:" + getPackageName()));
        	startActivityForResult(intent, OVERLAY_PERMISSION_REQ_CODE);     
	} 
}
```

and then when the user returns to the app a check is made if permission was granted:

```java

@Override 
protected void onActivityResult(int requestCode, int resultCode, Intent data) {    
 
 if (requestCode == OVERLAY_PERMISSION_CODE) {         
 		if (!Settings.canDrawOverlays(this)) {             
 		// SYSTEM_ALERT_WINDOW permission not granted...         
 		}     
 	} 
 }
```

Now that we have listed some important information on how are windows handled through WindowMananger and the window types available, let’s see how we can create an overlay that will exist above other apps through WindowMananger.

In general when an app is in the foreground, users can interact with it and they loose this ability when it goes to the background and is paused. With WindowManager you can easily add an overlay view in a System Window on top of everything, that users can still interact with, even if the application is paused and is hidden.

Depending on how you want this overlay view to be placed and the lifetime that you want to achieve, you can obtain reference to WindowManager with different ways and add a view through an Activity or a Service.



<h4>1. Overlay through the context of an activity</h4>

A WindowManager can be easily obtained:

```java
WindowManager wm = (WindowManager) context.getSystemService(WINDOW_SERVICE);
```

You need then to define WindowManager layout params.

```java
WindowManager.LayoutParams params = new WindowManager.LayoutParams();

Point point = OverlayUtil.getScreenDimensions(context);

params.height = point.y* 7/10;
params.width = LayoutParams.MATCH_PARENT;.

params.type = WindowManager.LayoutParams.TYPE_PRIORITY_PHONE;
params.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE |
         WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS |
         WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL;
params.format = PixelFormat.TRANSLUCENT;
params.gravity = Gravity.CENTER | Gravity.LEFT;

wm.addView(overlayView, params);
```

and then we try to add our Overlay view  method

```java
addView(View, WindowManager.LayoutParams)
```

<u>This methods performs three major steps:</u>

<ul>
<li>Checks whether the window has been already been added</li>
<li>Determines the type of the window and if is a child window, and if yes, it attaches it to its parent window</li> 
<li>It creates a new ViewRootImpl object and calls its setView method.</li>
</ul>


As you may see, we specified as window type TYPE\_PRIORITY\_PHONE which is one of System Window types. This type is a non-application window type that is normally placed above all applications but resides behind the status bar. Another type that could be used would be TYPE\_SYSTEM_ALERT that is a system window like low power alert and is also placed above other applications.

If we try to add this window without being granted the permission SYSTEM\_ALERT\_WINDOW that enables adding or removing system windows, a BadToken Exception will occur since WindowManager understands that the application is trying to add a Security Window that can render above other apps without having the right token.

<img src="{{ site.baseurl }}/images/android/badtoken.png">

Once you add an overlay view as a System Window with window manager through the context of an activity, a new window with its own view hierarchy is created and added above the current window of the activity. 

<img src="{{ site.baseurl }}/images/android/overlay_processes.png">

We can use Android Device Monitor and Hierarchy Viewer to see the list of available windows on the system.

<img src="{{ site.baseurl }}/images/android/windows_list1.png">

As we can see a new window is created with its own view hierarchy. This window has a null window token however is associated as we may see with the MainActivity.

<img src="{{ site.baseurl }}/images/android/windows_list2.png">


This is the same list we can get from 

```java
adb shell dumpsys window windows
```


```java
WINDOW MANAGER WINDOWS (dumpsys window windows)

.....

  Window #3 Window{1fb8d3e0 u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}:
    mDisplayId=0 mSession=Session{27d0ccae 6335:u0a10090} mClient=android.os.BinderProxy@7c1dde3
    mOwnerUid=10090 mShowToOwnerOnly=true package=com.vourkosa.overlayapp appop=SYSTEM_ALERT_WINDOW
    mAttrs=WM.LayoutParams{(0,0)(fillx1243) gr=#13 sim=#20 ty=2007 fl=#1000228 fmt=-3 surfaceInsets=Rect(0, 0 - 0, 0)}
    Requested w=1080 h=1243 mLayoutSeq=139
    mBaseLayer=81000 mSubLayer=0 mAnimLayer=81000+0=81000 mLastLayer=81000
    mToken=WindowToken{1180a699 null}
    mRootToken=WindowToken{1a074e0c null}
    mViewVisibility=0x0 mHaveFrame=true mObscured=false
    mSeq=0 mSystemUiVisibility=0x0
    mGivenContentInsets=[0,0][0,0] mGivenVisibleInsets=[0,0][0,0]
    mConfiguration={1.0 310mcc260mnc en_US ?layoutDir sw360dp w360dp h567dp 480dpi nrml port finger qwerty/v/v dpad/v s.6}
    mHasSurface=true mShownFrame=[0.0,304.0][1080.0,1547.0] isReadyForDisplay()=true
    mFrame=[0,304][1080,1547] last=[0,304][1080,1547]
    mSystemDecorRect=[0,0][1080,1243] last=[0,0][0,0]
    Frames: containing=[0,75][1080,1776] parent=[0,75][1080,1776]
        display=[-10000,-10000][10000,10000] overscan=[-10000,-10000][10000,10000]
        content=[0,304][1080,1547] visible=[0,304][1080,1547]
        decor=[0,0][1080,1920]
    Cur insets: overscan=[0,0][0,0] content=[0,0][0,0] visible=[0,0][0,0] stable=[0,0][0,0]
    Lst insets: overscan=[0,0][0,0] content=[0,0][0,0] visible=[0,0][0,0] stable=[0,0][0,0]
    WindowStateAnimator{5101a36 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}:
      mSurface=Surface(name=com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity)
      mDrawState=HAS_DRAWN mLastHidden=false
      Surface: shown=true layer=81000 alpha=1.0 rect=(0.0,304.0) 1080.0 x 1243.0
  Window #2 Window{14eaa6e5 u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}:
    mDisplayId=0 mSession=Session{27d0ccae 6335:u0a10090} mClient=android.os.BinderProxy@25bcb5dc
    mOwnerUid=10090 mShowToOwnerOnly=true package=com.vourkosa.overlayapp appop=NONE
    mAttrs=WM.LayoutParams{(0,0)(fillxfill) sim=#120 ty=1 fl=#81810100 wanim=0x1030466 vsysui=0x600 surfaceInsets=Rect(0, 0 - 0, 0) needsMenuKey=2}
    Requested w=1080 h=1776 mLayoutSeq=139
    mBaseLayer=21000 mSubLayer=0 mAnimLayer=21010+0=21010 mLastLayer=21010
    mToken=AppWindowToken{2a8c4f1 token=Token{15ff9498 ActivityRecord{398cce7b u0 com.vourkosa.overlayapp/.MainActivity t612}}}
    mRootToken=AppWindowToken{2a8c4f1 token=Token{15ff9498 ActivityRecord{398cce7b u0 com.vourkosa.overlayapp/.MainActivity t612}}}
    mAppToken=AppWindowToken{2a8c4f1 token=Token{15ff9498 ActivityRecord{398cce7b u0 com.vourkosa.overlayapp/.MainActivity t612}}}
    mViewVisibility=0x0 mHaveFrame=true mObscured=false
    mSeq=0 mSystemUiVisibility=0x600
    mGivenContentInsets=[0,0][0,0] mGivenVisibleInsets=[0,0][0,0]
    mConfiguration={1.0 310mcc260mnc en_US ?layoutDir sw360dp w360dp h567dp 480dpi nrml port finger qwerty/v/v dpad/v s.6}
    mHasSurface=true mShownFrame=[0.0,0.0][1080.0,1920.0] isReadyForDisplay()=true
    mFrame=[0,0][1080,1920] last=[0,0][1080,1920]
    mSystemDecorRect=[0,0][1080,1920] last=[0,0][0,0]
    Frames: containing=[0,0][1080,1920] parent=[0,0][1080,1920]
        display=[0,0][1080,1920] overscan=[0,0][1080,1920]
        content=[0,75][1080,1776] visible=[0,75][1080,1776]
        decor=[0,0][1080,1920]
    Cur insets: overscan=[0,0][0,0] content=[0,75][0,144] visible=[0,75][0,144] stable=[0,75][0,144]
    Lst insets: overscan=[0,0][0,0] content=[0,75][0,144] visible=[0,75][0,144] stable=[0,75][0,144]
    WindowStateAnimator{b9bcd37 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}:
      mSurface=Surface(name=com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity)
      mDrawState=HAS_DRAWN mLastHidden=false
      Surface: shown=true layer=21010 alpha=1.0 rect=(0.0,0.0) 1080.0 x 1920.0

 ....
```
and we can also get a lost of window tokens with 

```java
adb shell dumpsys window tokens
```


```java
WINDOW MANAGER TOKENS (dumpsys window tokens)
  All tokens:
  WindowToken{1180a699 null}:
    windows=[Window{1fb8d3e0 u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}]
    windowType=-1 hidden=false hasVisible=true
  AppWindowToken{2a8c4f1 token=Token{15ff9498 ActivityRecord{398cce7b u0 com.vourkosa.overlayapp/.MainActivity t612}}}:
    windows=[Window{14eaa6e5 u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}]
    windowType=2 hidden=false hasVisible=true
    app=true voiceInteraction=false
    allAppWindows=[Window{14eaa6e5 u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}]
    groupId=612 appFullscreen=true requestedOrientation=-1
    hiddenRequested=false clientHidden=false willBeHidden=false reportedDrawn=true reportedVisible=true
    numInterestingWindows=1 numDrawnWindows=1 inPendingTransaction=false allDrawn=true (animator=true)
    startingData=null removed=false firstWindowDrawn=true mDeferRemoval=false
    
...
```

We can see from the list of available tokens that we have the top level AppWindowToken for MainActivity and WindowToken for our overlay window is also listed but has a null value.

If we start a second activity, without destroying the first activity, we can see that a new window and a new AppWindowToken has been created for that activity while the overlay view still resides on top of everything.


<img src="{{ site.baseurl }}/images/android/second_activity.png">


```java
WINDOW MANAGER TOKENS (dumpsys window tokens)
  All tokens:
  WindowToken{38eaa3b8 null}:
    windows=[Window{37fadf5d u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}]
    windowType=-1 hidden=false hasVisible=true
  AppWindowToken{572ee3b token=Token{2622ccca ActivityRecord{429e035 u0 com.vourkosa.overlayapp/.MainActivity t613}}}:
    windows=[Window{f186e0f u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}]
    windowType=2 hidden=true hasVisible=true
    app=true voiceInteraction=false
    allAppWindows=[Window{f186e0f u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}]
    groupId=613 appFullscreen=true requestedOrientation=-1
    hiddenRequested=true clientHidden=true willBeHidden=false reportedDrawn=false reportedVisible=false
    numInterestingWindows=2 numDrawnWindows=2 inPendingTransaction=false allDrawn=true (animator=true)
    startingData=null removed=false firstWindowDrawn=true mDeferRemoval=false
....
AppWindowToken{7d560a0 token=Token{6f9a5a3 ActivityRecord{24cda0d2 u0 com.vourkosa.overlayapp/.SecondActivity t613}}}:
    windows=[Window{bc0f81e u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.SecondActivity}]
    windowType=2 hidden=false hasVisible=true
    app=true voiceInteraction=false
    allAppWindows=[Window{bc0f81e u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.SecondActivity}]
    groupId=613 appFullscreen=true requestedOrientation=-1
    hiddenRequested=false clientHidden=false willBeHidden=false reportedDrawn=true reportedVisible=true
    numInterestingWindows=1 numDrawnWindows=1 inPendingTransaction=false allDrawn=true (animator=true)
    startingData=null removed=false firstWindowDrawn=true mDeferRemoval=false
....

```

As far as the activity is not destroyed there is no issue. However, once that activity is destroyed a Window leak occurs if the overlay view is not removed in advance. 

If we dive deeper into this, we will see that what happens is relevant to Window Tokens. If we see into Android code, in <a heref="https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/view/WindowManagerGlobal.java"> WindowManagerGlobal </a> implementation, when an activity is destroyed, it notifies WindowManagerGlobal to try and close all windows associated with that activity. The overlay window has a null window token but is still referenced from the context of the activity.  
This is the point where a window leak will occur.


WindowManagerGlobal.java

```java
public void closeAll(IBinder token, String who, String what) {
        synchronized (this) {
            if (mViews == null)
                return;
            
            int count = mViews.length;
            //Log.i("foo", "Closing all windows of " + token);
            for (int i=0; i<count; i++) {
                //Log.i("foo", "@ " + i + " token " + mParams[i].token
                //        + " view " + mRoots[i].getView());
                if (token == null || mParams[i].token == token) {
                    ViewRootImpl root = mRoots[i];
                    root.mAddNesting = 1;
                    
                    //Log.i("foo", "Force closing " + root);
                    if (who != null) {
                        WindowLeaked leak = new WindowLeaked(
                                what + " " + who + " has leaked window "
                                + root.getView() + " that was originally added here");
                        leak.setStackTrace(root.getLocation().getStackTrace());
                        Log.e("WindowManager", leak.getMessage(), leak);
                    }

                    removeViewLocked(i);
                    i--;
                    count--;
                }
            }
        }
    }
```


So what can we do as a workaround to avoid a window leak and keep our overlay window above the system even if the activity that created is destroyed? 

<u>There are two different ways to achieve that:</u>

<img src="{{ site.baseurl }}/images/android/landing.png">


a) By assigning a token to the new Security Window before calling WindowManager to add it in its WindowManager LayoutParams 

```java
params.token=new Binder();
```
From the list of tokens we can see that the Overlay window has now a token

```java
WINDOW MANAGER TOKENS (dumpsys window tokens)
  All tokens:
  WindowToken{2bdf2e70 android.os.BinderProxy@39a72622}:
    windows=[Window{1a15c8b3 u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}]
    windowType=-1 hidden=false hasVisible=true

...

  AppWindowToken{2d2fc41a token=Token{c1de2c5 ActivityRecord{39c9f53c u0 com.vourkosa.overlayapp/.MainActivity t616}}}:
    windows=[Window{219167ca u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}]
    windowType=2 hidden=false hasVisible=true
    app=true voiceInteraction=false
    allAppWindows=[Window{219167ca u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}]
    groupId=616 appFullscreen=true requestedOrientation=-1
    hiddenRequested=false clientHidden=false willBeHidden=false reportedDrawn=true reportedVisible=true
    numInterestingWindows=1 numDrawnWindows=1 inPendingTransaction=false allDrawn=true (animator=true)
    startingData=null removed=false firstWindowDrawn=true mDeferRemoval=false
    
...    
```


This way we make the Overlay view stay alive and be independent of the activity's lifecycle. However once the app is destroyed, the Overlay window will be removed too.

b) By getting reference to the WindowManager through application's Context

```java
WindowManager wm = (WindowManager) getApplicationContext().getSystemService(WINDOW_SERVICE);
```

If we obtain reference to WindowManager through the application's context instead of the activity's context we can see now that the Overlay Window is not relevant to the activity any more but to the app.

```java
WINDOW MANAGER TOKENS (dumpsys window tokens)
  All tokens:
  WindowToken{23292f77 null}:
    windows=[Window{1cef1811 u0 com.vourkosa.overlayapp}]
    windowType=-1 hidden=false hasVisible=true
 
 ...
 
  AppWindowToken{21f5a48f token=Token{3dc7d7ee ActivityRecord{1e855169 u0 com.vourkosa.overlayapp/.MainActivity t618}}}:
    windows=[Window{39091423 u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}]
    windowType=2 hidden=false hasVisible=true
    app=true voiceInteraction=false
    allAppWindows=[Window{39091423 u0 com.vourkosa.overlayapp/com.vourkosa.overlayapp.MainActivity}]
    groupId=618 appFullscreen=true requestedOrientation=-1
    hiddenRequested=false clientHidden=false willBeHidden=false reportedDrawn=true reportedVisible=true
    numInterestingWindows=1 numDrawnWindows=1 inPendingTransaction=false allDrawn=true (animator=true)
    startingData=null removed=false firstWindowDrawn=true mDeferRemoval=false
    
...
```

<img src="{{ site.baseurl }}/images/android/overlay_app.png">

This trick allows the Overlay view to live as far as the app lives. Once the application stops the Overlay view is removed too.


<h4>2. Overlay through a service</h4>

Another way to create an Overlay as a System Window that will be added above other applications is by adding that overlay through WindowManager in the same way as described in previous section but through a service that can run in the background.

A common usage of this approach can be found in Facebook chatheads and other popular apps. You just have to declare the service in the app's AndroidManifest file.

```java
<service android:name=".OverlayService"></service>
```

and start or stop the service at any time:

```java
    Intent intent = new Intent(activity, OverlayService.class);
    activity.startService(intent);
```

```java
    Intent intent = new Intent(activity, OverlayService.class);
    activity.stopService(intent);
```

<img src="{{ site.baseurl }}/images/android/androidhead.png">


```java
public class OverlayService extends Service {

	private WindowManager wm;
    private ImageView androidHead;
  
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        
        androidHead = new ImageView(this);
        androidHead.setImageResource(R.drawable.ic_launcher);

        wm = (WindowManager) getSystemService(WINDOW_SERVICE);

        final WindowManager.LayoutParams params = new WindowManager.LayoutParams();
        
        params.width = ViewGroup.LayoutParams.WRAP_CONTENT;
        params.height = ViewGroup.LayoutParams.WRAP_CONTENT;
        params.type = WindowManager.LayoutParams.TYPE_PRIORITY_PHONE;
        params.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE |
                WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS |
                WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL;
        params.format = PixelFormat.TRANSLUCENT;
        params.gravity = Gravity.TOP | Gravity.LEFT;

        wm.addView(androidHead, params);

    }


    @Override
    public void onDestroy() {
        super.onDestroy();
        removeView();
    }

    private void removeView() {
        if (androidHead != null) {
            wm.removeView(androidHead);
        }
    }
}
```

As we have seen there are different ways to place overlays above an app or other apps by using WindowManager. However the need of requesting a special permission especially since Android Marshmallow or the need of declaring and using a service to keep the overlay independent of an app's or activity's lifecycle makes this approach less convenient.