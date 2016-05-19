---
layout: post
title: Working with overlays - Giving birth to a new DecorView child
date:  2016-03-20 09:15:20
header-img: "images/android/after4.png"
image: "images/android/after4.png"
description: Following up the series of previous articles on how to create overlays on Android we will study in this post how to add an overlay over an Activity. We will do that by traversing it’s current view hierarchy and injecting directly a RelativeLayout below window’s DecorView. This RelativeLayout will hold the overlay layout, along with the original layout of the Activity that is already in place. 
categories: Android 
tag: android
comments: true
---

Following up the series of previous articles on how to create overlays on Android we will study in this post how to add an overlay over an Activity. We will do that by traversing it’s current view hierarchy and injecting directly a RelativeLayout below window’s DecorView. This RelativeLayout will hold the overlay layout, along with the original layout of the Activity that is already in place. 

<h3>The Basics</h3>

A simple way to add an overlay to a layout is to have a FrameLayout or a RelativeLayout as a parent, and then place the overlay view as a new child in the appropriate z-order in order to be on top of the current layout. Both FrameLayout and RelativeLayout can support this approach. FrameLayout is designed to hold a stack of child views with the most recent placed on top while RelativeLayout can be used to position child views either relative to each other, or to the screen boundaries. If two or more child views are relative to the same thing, they are placed in a stack structure with the most recent on top, similar to FrameLayout approach.

The exact same concept is followed by Android framework with DecorView (which is a FrameLayout) and the child view marked with id/content (which is also a FrameLayout) that usually holds the view hierarchy of the content of the Activity.

Now let’s see a quick example in order to understand better. We have a simple Activity with a RelativeLayout as a content view that holds a button aligned at the bottom. 

activity_rel.xml

```java
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/topRelLay"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <Button
        android:id="@+id/changeOrderBtn"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:layout_centerHorizontal="true"
        android:layout_marginBottom="30dp"
        android:text="Change Order"
        android:textSize="20sp"
        android:textStyle="bold" />

</RelativeLayout>
```

we go then and add programmatically 3 TextViews as child views to this RelativeLayout. These views are placed by default relative to the top left corner and we add some margins to them in order to demonstrate them better. The child view that is added last in RelativeLayout, is rendered on top.

```java
public class MainActivity extends Activity {

    private RelativeLayout topRelLayout;
    private Button changeOrderBtn;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.activity_rel);

        topRelLayout = (RelativeLayout) findViewById(R.id.topRelLay);
        changeOrderBtn = (Button) findViewById(R.id.changeOrderBtn);

        for(int i=0;i<3; i++) {

            TextView textView = new TextView(this);
            RelativeLayout.LayoutParams layoutParams = new RelativeLayout.LayoutParams(400, 600);

            textView.setId(i); // to track view later on HierarchyViewer
            textView.setGravity(Gravity.CENTER);
            textView.setBackgroundColor(randomColor());
            textView.setText("View " + i);
            textView.setTypeface(null, Typeface.BOLD);
            layoutParams.setMargins(50 * (i + 1), 50 * (i + 1), 0, 0);

            topRelLayout.addView(textView, layoutParams);
        }

        changeOrderBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                ViewGroup myViewGroup = ((ViewGroup) v.getParent());
                myViewGroup.bringChildToFront(myViewGroup.getChildAt(1));
            }
        });
   }
}
```

We then use on every click of the button, <a href="http://developer.android.com/reference/android/view/View.html#bringToFront()" target="_blank">bringToFront()</a> or <a href="http://developer.android.com/reference/android/view/ViewGroup.html#bringChildToFront(android.view.View)" target="_blank">bringChildToFront()</a> functions in order to change the order of these child views in the tree and bring the view rendered at the bottom of our view stack (position 1 in ViewGroup since position 0 is reserved by our button), to the top (position 3 in ViewGroup).

<img src="{{ site.baseurl }}/images/android/changeorder.gif" align="center" style="width: 200px; margin: 0 auto;" >

If we have a closer look in gif below we will see that each time we call bringChildToFront() to re-arrange views, child views' order changes also in Hierarchy Viewer.

<img src="{{ site.baseurl }}/images/android/view_hier.gif">

<h3>So let’s do it! Wait, but…why??</h3>

As we have seen above, if we have a RelativeLayout or a FrameLayout as a parent to our layout we can easily add a new overlay view on top of that. However if you are working in an agnostic environment, like for example through a third-party library you never know what type of layout is the parent of layouts in the app using that library. Moreover with the vast amount of libraries out there you will also meet many cases where the expected view hierarchy is altered and you cannot be 100% on what is the structure of the view tree you will have to deal with. Having that said, the only way to re-assure that you have full control on the tree view is to be the first one to have a RelativeLayout or a FrameLayout  injected directly under DecorView. This way, the app’s Activity, using your library, will always have a RelativeLayout as the parent of all in Activity's view hierarchy and then any view can be placed on top of everything.

Someone would argue that this is a really bad approach since you are interfering with the provided hierarchy of the framework. This is so true.. I remember last November discussing this with Adam Powel from Android team at Android Dev Summit in San Francisco. His reaction when I was describing how we were traversing the view hierarchy of an app and how we were injecting an overlay as a new DecorView child was stunning. Long story short, he was just saying “no, no, no you shouldn’t do that!!”.  Messing up with the provided view hierarchy of an app, can reveal several issues, that’s also true. However, at <a href="http://www.pollfish.com" target="_blank">Pollfish</a> we had been successfully following this approach for more than 2 years, injecting overlay layouts under DecorView in thousands of mobile apps on more than 200m mobile devices out there. During that period we had to resolve a lot of corner cases in order to have a robust solution, until we moved on to a new approach.

Below we can see the view hierarchy of Android Studio's default empty activity template project running on a Nexus 5 before and after re-arranging the hierarchy to add a new DecorView child

<b>Before:</b>

<img src="{{ site.baseurl }}/images/android/before2.png" >


<b>After:</b>

<img src="{{ site.baseurl }}/images/android/after4.png" >

<h3>The flow</h3>

Here we will list the necessary steps involved, when initializing a third party library, an SDK, through an Activity and trying to inject a RelativeLayout as a new DecorView child in order to later on, add an OverlayLayout on top of current Activity’s layout.


<h4>1) Check if OverlayLayout already exists in view hierarchy</h4>

During the first step, we try to see if we have already injected an OverlayLayout to our Activity’s view hierarchy, and if yes get a reference to it. We need to do this step since our library may get initialized several times during an Activity’s lifecycle. This way we can ensure that no more than one overlays will be placed to the Activity.


```java
ViewGroup decor = (ViewGroup) activity.getWindow().getDecorView();
OverlayLayout overlayLayout = getExistingOverlayInView(decor); 
```

```java
public OverlayLayout getExistingOverlayInView(ViewGroup parent){

 for (int i = 0; i < parent.getChildCount(); i++) {

    View child = parent.getChildAt(i);

    if (child instanceof OverlayLayout)
	  return (OverlayLayout) child;

    if (child instanceof ViewGroup) {

	  View v = getExistingOverlayInView((ViewGroup) child);

	  if (v != null)
	    return (OverlayLayout) v;
    }
  } 
  return null;
}
```

<h4>2) Remove OverlayLayout (if found) and restore to Activity's initial view hierarchy</h4>

If OverlayLayout was found in view hierarchy we try to remove it and restore view tree of the Activity, to it’s initial state.

We initially get reference to the parent of the retrieved OverlayLayout. This parent should be our injected RelativeLayout.

```java
ViewGroup ourRelativeLayout= (ViewGroup) overlayLayout.getParent(); 
```

Then we get reference to the parent of our injected RelativeLayout which should be the initial parent of our Activity’s view hierarchy and in particular our DecorView.

```java
ViewGroup parentRelativeLayout = (ViewGroup) ourRelativeLayout.getParent();
```

We remove then our own RelativeLayout (ourRelativeLayout) from parentRelativeLayout and then we try to get reference to the initial child of DecorView (which we have tagged with a specific tag prior changing view hierarchy)

Once we get reference to our initial child view we remove all views from our RelativeLayout and add back the original view to its initial parent (at the specific child position that we kept reference when we removed it). After that, we remove our RelativeLayout from our DecorView and voilà, our app is back to its initial view state.

```java
public void bringAppViewsToPriorOverlayState(OverlayLayout overlayLayout){

ViewGroup ourRelativeLayout = (ViewGroup) overlayLayout.getParent();

  if (ourRelativeLayout != null) {

	ViewGroup parentRelativeLayout = (ViewGroup) ourRelativeLayout.getParent();
					
	if (parentRelativeLayout != null) {

		parentRelativeLayout.removeView(ourRelativeLayout);

		if (ourRelativeLayout.getChildCount() > 1) { // our RelativeLayout should hold both OverlayLayout & initial DevorView child

			ourRelativeLayout.removeView(overlayLayout);

			View originalView = (View) ourRelativeLayout.findViewWithTag("ORIGINAL_VIEW_TAG");

			if (originalView != null) {

				Log.d(TAG,"Found previous original app's view - reordering view tree");
										
				ourRelativeLayout.removeAllViews();				
				parentRelativeLayout.addView(originalView, OverlayLayout.getIndexOfParentInInitialHierarchy());
				parentRelativeLayout.removeView(ourRelativeLayout);
			}
		}
	}
    }
}
```

<h4>3) Getting reference to the child of DecorView which holds the Activity’s view hierarchy</h4>

As we have discussed in previous article, since Android Lollipop DecorView has more than one child (navigation bar and status bar backgrounds). Having that said we need to find the child that holds the Activity’s view hierarchy. This child usually lies at:

```java
View view = (View) decor.getChildAt(0);
```

However, since other libs or even the app developer can interfere with DecorView in an agnostic environment we need to follow a different approach to get reference to the appropriate child. We use a loop looking for a child of DecorView which is an instance of a ViewGroup. Obviously this is a solution that can easily break in future releases of Android or even if someone else had played around with DecorView. If no child is found, we fall back and get reference at child view at position 0.  It is important here to keep a reference to the position of the referenced child below DecorView. We will need that (Step 2), if we try to restore the view hierarchy to its initial state, prior injecting our overlay view. Have in mind that changing the order of the child of DecorView can mess up the whole Activity’s layout structure.

```java
ViewGroup decor = (ViewGroup) act.getWindow().getDecorView();

if (decor != null) {
  
   View originalView = null;

   for (int i = 0; i < decor.getChildCount(); i++) {

      if (decor.getChildAt(i) instanceof ViewGroup) {

         originalView = (ViewGroup) decor.getChildAt(i);

         OverlayLayout.setIndexOfParentInInitialHierarchy(i); // (used in step 2)

         break;
      }
   }

  if (originalView == null) {			
    originalView = (View) decor.getChildAt(0);
  }
 
  originalView.setTag("ORIGINAL_VIEW_TAG");
}
```

<h4>4) Creating parent RelativeLayout</h4>

We create a parent RelativeLayout which will be the new parent/root of the Activity’s view hierarchy.

```java
RelativeLayout ourRelativeLayout = new RelativeLayout(act);
```

<h4>5) Removing DecorView's child view from parent</h4>

We initially remove the original DecorView's child from its parent

```java
ViewGroup parent = (ViewGroup) originalView.getParent(); // in default solution this is our DecorView

parent.removeView(originalView);

```
           

<h4>6) Adding the original DecorView child view to our parent RelativeLayout</h4>

We add back the original DecorView child view to our RelativeLayout which will be the parent of all.

```java
ourRelativeLayout.addView(originalView, new RelativeLayout.LayoutParams(RelativeLayout.LayoutParams.MATCH_PARENT,RelativeLayout.LayoutParams.MATCH_PARENT)); 
```

<h4>7) Adding our parent RelativeLayout to DecorView</h4>

If this is the first time we re-arrange the views we just add our RelativeLayout to the parent view, otherwise we find use the original position of DecorView's child view in the tree and we use it in order to add our RelativeLayout in the appropriate index/order in the tree:

```java
RelativeLayout.LayoutParams rootParam = new RelativeLayout.LayoutParams(
                     RelativeLayout.LayoutParams.MATCH_PARENT,
                     RelativeLayout.LayoutParams.MATCH_PARENT);

parent.addView(ourRelativeLayout, rootParam);

```

or (if this is not the first time we re-arrange)

```java
parent.addView(ourRelativeLayout, indexOfParentInInitialHierarchy, rootParam); 
```

<h4>8) Adding our OverlayLayout to parent RelativeLayout</h4>

We then add our OverlayLayout to our parent RelativeLayout and we are ready! OverlayLayout will be placed on top of the content of the original DecorView's child as explained in the introduction of this article.

```java
RelativeLayout.LayoutParams overlayParam = new RelativeLayout.LayoutParams(RelativeLayout.LayoutParams.MATCH_PARENT,RelativeLayout.LayoutParams.MATCH_PARENT);
overlayParam.addRule(RelativeLayout.ALIGN_PARENT_BOTTOM);  					

// create our overlay layout				
OverlayLayout overlayLayout = new OverlayLayout(act);

ourRelativeLayout.addView(overlayLayout, overlayParam);
```

<h3>Corner cases</h3>

Following this approach, several corner cases can occur that will need special handling. This is due to fragmentation of Android and the agnostic environment that the code runs. Flags applied to Activity's window, soft input events or orientation changes can really mess up this solution so you should act proactively. Moreover, status and navigation bars as appearing on different devices and versions of the framework can deliver wrong heights for the actual visible part of the screen and this solution can end up in a real bad visual result and a broken view hierarchy.

<h3>Conclusion</h3>

To conclude, messing around with the view hierarchy of an Activity on the DecorView level is wrong. There so many things that can go bad. Parameters inherited by child view layouts or flags that affect the structure of the view tree can easily go away when you brake the consistency of the structured view tree. During the last 2 years we had to identify, track and solve such issues in an agnostic environment.

I will try to find some time later on this week to publish an open source example of this implementation in order for anyone interested to look into this further.

