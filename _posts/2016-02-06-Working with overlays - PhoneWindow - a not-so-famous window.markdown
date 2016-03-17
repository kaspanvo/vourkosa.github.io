---
layout: post
title: Working with overlays - PhoneWindow, a not-so-famous window
date:  2016-02-06 10:16:20
categories: Android 
tag: android
comments: true
---

In this article we will discuss how we can create an overlay view above an Activity with the help of PhoneWindow and its inner class DecorView.

As we discussed in previous post, when an Activity is launched, a window is created in order to render the views and interact with the user. Usually this is a full screen window but developers have also the option to display a non full screen activity with the relevant Theme (like floating windows or render activity through a group of other activities like in an ActivityGroup).

But what happens under the hood? If we dive deeper in Android source code we will see that most of all these, happen around an implementation of Window’s abstract class, which is called <a href="https://android.googlesource.com/platform/frameworks/base/+/7d276c3/policy/src/com/android/internal/policy/impl/PhoneWindow.java" target="_blank">PhoneWindow.</a> 


When an activity starts, the ActivityManager along with WindowManager request the creation of a PhoneWindow. PhoneWindow  is responsible for holding the activity’s view hierarchy. Creation of PhoneWindow is requested by activity’s attach internal method:


Activity.java

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor) {
    attachBaseContext(context);
	
	...
    
    mWindow = new PhoneWindow(this);
	
	...
 }
```


You can read a really insightful article on PhoneWindow and the whole window creation process <a href="http://blog.csdn.net/luoshengyang/article/details/8223770" target="_blank">here.</a> 

<u>PhoneWindow has two important members:</u>
<ul>
<li><b>DecorView mDecor</b> – which is the top-level view of the window, containing the window decor (like the activity's window background) </li>
<li><b>ViewGroup mContentParent</b> – which is the view in which the window contents are placed (for example the user's view layout) and is either the DecorView itself, or a child of DecorView where the contents go</li>
</ul>

DecorView is an inner class of  PhoneWindow 

```java
private final class DecorView extends FrameLayout implements RootViewSurfaceTaker { ... }
```

and is nothing more than a a FrameLayout. DecorView is the root view in an activity’s window view hierarchy. Based on the theme specified for an activity or for the application in general (or any Window flags and features requested), the layout of DecorView is created. 

For example if developer has not requested any specific feature or flags, window's DecorView is installed and then its clild view is inflated from a layout resource xml file and added to the DecorView. The default basic layout is stored in a file called screen_simple.xml. Depending on different settings a different layout resource is inflated.


```java

...

if(..){}
else if(..){}
else{
	layoutResource = com.android.internal.R.layout.screen_simple;
}
...
  	  
View in = mLayoutInflater.inflate(layoutResource, null);
	  
decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
	
...
```

If we look into <a href="https://github.com/android/platform_frameworks_base/blob/master/core/res/res/layout/screen_simple.xml" target="_blank">screen_simple.xml</a> file, we will see that it includes a simple LinearLayout with a ViewStub and a FrameLayout as child views.

```java
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

If we do not explicity set our own content view when creating the Activity this is how our view hierarchy will be structured on a Nexus 4 running Android 4.3:

<img src="{{ site.baseurl }}/images/android/simple.png">

However, something that is worth mentioning here is the radical change of the view hierarchy structure under DecorView since Android Lollipop. As we may see in screenshot below, if we do not set any special flags nor we set our content view (like the previous example on Nexus 4) this is how the view hierarchy will be structured on a Nexus 5 running Android 5.1

<img src="{{ site.baseurl }}/images/android/simple2.png">

We can easily identify here that DecorView does not have a single LinearLayout as a child view anymore, but it also includes another two views, our status and navigation bar! The order of these views have an impact on several aspects of our view tree but we will cover this in a future article.

<h3>Setting activity's content view</h3>

Usually in onCreate(Bundle) function of an Activity, users are advised to use <a href="http://developer.android.com/reference/android/app/Activity.html#setContentView(android.view.View)" target="_blank">setContentView</a> to set an Activity’s content to a specific view layout. This view can either be inflated from a specific resource or through a programmatically created view.

<u>There are 3 options on how to set ContentView:</u>

<b>1. setContentView (View view)</b> - Set the activity content to an explicit view.

```java
@Override
public void setContentView(View view) {
	setContentView(view, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
}
```

<b>2 .setContentView (int layoutResID)</b> - Set the activity content from a layout resource.

```java
@Override
public void setContentView(int layoutResID) {
	if (mContentParent == null) {
	    installDecor();
	} else {
	    mContentParent.removeAllViews();
    }     
	mLayoutInflater.inflate(layoutResID, mContentParent);
	final Callback cb = getCallback();
	    
	if (cb != null && !isDestroyed()) {
	    cb.onContentChanged();
	}
}
```
<b>3. setContentView (View view, ViewGroup.LayoutParams params)</b> - Set the activity content to an explicit view with specified layout params.

```java	
@Override
public void setContentView(View view, ViewGroup.LayoutParams params) {	
	if (mContentParent == null) {
	     installDecor();
	} else {
	     mContentParent.removeAllViews();
    }       
	mContentParent.addView(view, params);
	final Callback cb = getCallback();
	        
	if (cb != null && !isDestroyed()) {
	    cb.onContentChanged();
	}
}
```

The view provided as an argument in setContentView is placed directly into the view hierarchy of the activity. Regardless the layout params of the view, if not specified explicitly, the view layout params will change to MATCH_PARENT and fit the window of the activity (as we can see on the first option).

If we look deeper into the code in PhoneWindow.java, we will see that when we call setContentView and we do not have already a DecorView, a DecorView will be installed and a layout will be generated. Otherwise, all views under mContentParent (which is our FrameLayout with id/content) will be removed and our view/layout will be added or inflated to the view hierarchy of the activity under mContentParent.


In our previous example if we do not explicitly ask to setContentView  we will get an empty screen with a view hierarchy as seen above. If we set our own simple_layout.xml through setContentView (a LinearLayout with a TextView), our layout will be rendered below the FrameLayout with id/content as you may see below:

<img src="{{ site.baseurl }}/images/android/with_content.png">


</br>
<h3>Adding an overlay view layout</h3>

So can we add or remove an overlay layout view in our Activity's window using content views? 

<img src="{{ site.baseurl }}/images/android/overlay_view.png">


We will see that aside from setContentView options, there is another function called <b>addContentView</b> which allows us to set a layout on top of the layout we already added with setContentView. As we may see from the implementation in PhoneWindow.java of addContentView function:

```java	
@Override
public void addContentView(View view, ViewGroup.LayoutParams params) {
    if (mContentParent == null) {
        installDecor();
    }
    mContentParent.addView(view, params);
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
}
```

the difference with setContentView is that it does not remove any previously added views to mContentParent. It just adds the new view as another child of mContentParent and placed in an order to be rendered above the previous layout added by setContentView.

```java	
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    
    setContentView(R.layout.simple_layout);
    
	OverlayLayout overlayLayout = new OverlayLayout(this); // our overlay view
    
    addContentView(overlayLayout, new RelativeLayout.LayoutParams(RelativeLayout.LayoutParams.MATCH_PARENT,RelativeLayout.LayoutParams.MATCH_PARENT));   
}
```
 Having that said we can see below the visualized result of addContentView when trying to add an overlay view. 

<img src="{{ site.baseurl }}/images/android/overlay_hierarchy.png">

As we may see OverlayLayout is another child of FrameLayout with id/content.

<img src="{{ site.baseurl }}/images/android/with_overlay_content.png">


Removing the overlay view can be as easy as removing it from its parent:


```java	
((ViewGroup) overlayLayout.getParent()).removeView(overlayLayout);
```


If we do not have reference to the view we can use tag mechanisms to assign and locate the view later on and follow the same approach as above.


Adding overlays on activities with addContentView is an exceptional way to add overlays to activities since it inherits all the characteristics and behaviour/state from activity's window and there is no need for special handling of different cases like orientation issues, special flags and others. We will dive further into the benefits of this approach in comparison with other options as we move on to this series of articles regarding creating overlays.  