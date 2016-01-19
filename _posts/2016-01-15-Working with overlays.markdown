---
layout: post
title:  Working with overlays - Life under WindowManager
date:   2016-01-15 10:15:20
categories: Android
---

This is the start of a series of articles regarding creating and using overlays on Android.  An overlay is a view that is placed above other views.  Such views can be created on Android with several different ways and for different purposes.

There are several usages for overlays on Android. Some of them include:

<ol>
<li>Onboarding tutorials</li>
<li>Floating buttons/menus</li>
<li>Advertising</li>
<li>Augmented Reality</li>
<li>Fancy animations</li>
</ol>


In this article we will see how we can create an overlay view with the help of WindowManager.

<h2>Life under WindowManager</h2>

But let's see first what is a <a href="http://developer.android.com/reference/android/view/WindowManager.html" target="_blank">WindowManager:</a>

<ul>
<li>A system service</li>
<li>Creates and allocates surfaces for the clients (applications) - (requests SurfaceManager). Is responsible of which windows are visible, their z-order and how they are laid on the screen</li>
<li>WindowManager never deals with the bits of the created surfaces, it is up to the clients to do so, and they do it directly without going through the WindowManager</li>
</ul>

A WindowManager can be easily obtained:

<code>WindowManager wm = (WindowManager) Context.getSystemService(WINDOW_SERVICE); </code>

<code> adb shell dumpsys window </code>

With WindowManager you can easily add an overlay view on top of everything. Depending on where you want this overlay view to be placed, you either have to obtain reference to the WindowManager and add the view to the window of an Activity or a Service.

<ol>
<li><b>Activity</b> - Overlay view is relevant to the activity </li>
<li><b>Service</b>  - Overlay view lays above the system, on any screen above the service window</li>
</ol>

<h4>1. Overlay above an activity</h4>

Once you add an overlay view with window manager through an activity, a new window with its own view hierarchy is created and added above the current window of the activity. As far as the activity is not destroyed there is no issue. However, once that activity is destroyed a leak occurs if the overlay view is not priorly removed. 

If we dive deeper into this, we will see that what happens is relevant to window tokens. You can read more about window tokens <a heref="http://www.androiddesignpatterns.com/2013/07/binders-window-tokens.html" >here. </a>. If we see into Android code, in <a heref="https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/view/WindowManagerImpl.java"> WindowManagerImpl</a> implementation, when an activity is destroyed WindowManagerImpl tries to close all windows associated with a specific token. This is the point where a window leak will occur.

<code>
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
</code>

So what we can do as a workaround to avoid a window loadk and keep our overlay window above the system is to define our token

<code> params.token=new Binder();</code>

This way we make the overlay view stay alive and be independent of our app's activities lifecycle. However once the app is destroyed overlay window will be removed too.

<img src="{{ site.baseurl }}/images/android/initial_state.png">

<img src="{{ site.baseurl }}/images/android/overlayed_app.png">

<img src="{{ site.baseurl }}/images/android/with_activity_overlay.png">




<code>&lt;uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/&gt;</code>
