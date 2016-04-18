---
layout: post
title: Sharing Android libraries as JARs - Including and reading image resources
date:  2016-04-06 10:16:20
categories: Android 
tag: android
comments: true
---


When deciding to create a library that will be distributed publicly among the developer ecosystem you need to consider in which format you will expose it to the community.
<br/>
<h3>Initial considerations</h3>

There are several things that affect this decision and there are several formats to choose from. Does your library need to include Android resources to perform, is this an open or a closed source project? What is your major target audience, what are the main tools this audience uses and which format are they more familiar with? Where will you host this library and many other factors should be taken under consideration.

<h3>Major formats</h3>

Currently, there are four major formats out there:

<ul>
<li><b>1) Library module - </b>

<a href="http://developer.android.com/tools/projects/index.html#LibraryProjects" target="_blank">Library modules</a> contain shareable source code and resources. Developers can reference them in their Android projects and, at build time, include its compiled sources in their apks.  Library modules have a structure similar to Android projects but cannot include raw assets. Distributing a library this way, requires providing source code publicly available and maintaining it among the community with any implications this might have
</li>

<li><b>2) JAR - </b>

Jar files are used widely among the Android and Java community in general.  It is a Java archive format that contains all classes of the library (*class files).  The contents of a jar file can be extracted easily with any unzip software or with jar command of JVM.  Jar files can be obfuscated, however they do not support Android resources and this is one of the main reasons that AAR format was introduced in the new build system.
</li>
<li><b>3) apklib - </b>

apklib is a way to bundle an Android project and a way to share both code and resources among Android projects.  It is <a href="https://plus.google.com/+ChristopherBroadfoot/posts/7uyipf8DTau" target="_blank">nothing more</a> than a zipped format of src/ and res/ directories along with AndroidManifest.xml file. apklib  cannot contain any compiled classes or jars. This approach is nowadays obsolete since AAR format is a much better solution.
</li>
<li><b>4) AAR - </b>

<a href="http://tools.android.com/tech-docs/new-build-system/aar-format" target="_blank">AAR</a> format was introduced by Android team, at Google I/O 2013, to overcome the other formats’ weaknesses. An AAR file is the binary distribution of a library project. It is a standard zip archive and its main differentiation from jar files is that it can include Android resources such as layouts and drawables.  The major difference with apklib format is that it contains the compiled class files of the library project. AAR files are now the default way of distributing Android libraries.
</li>
</ul>

<br/>
<h3>The problem</h3>

When we initially started four years ago, our main target audience for our library was small indie developers. The majority of them were using Eclipse, with no deep knowledge on Android ecosystem and the format they were mostly familiar with, for library projects, was JAR format. Our library was a complete product containing server side logic, encryption mechanisms and other features. Having that said, we decided that we should distribute it as a closed-source and obfuscated solution. In addition, in order to reduce network bandwidth we wanted to include some image and file resources that would be extensively used in our library. 

Library project was not an option for us, since we wanted to distribute our library as a closed source solution and aar files did not exist back then. Having that said we had to choose between JAR and apklib format but since JAR format was the most widely used and easily adapted solution, we had to go that way. After the first years and the introduction of AAR format we moved to that format, keeping though a distribution channel in JAR format too.

The main problem that we had to deal with was that jar files could not support Android resources. We had therefore to include resources in the JAR file and then read them somehow through our library code.

Retrieving resources through R class could not be used since it would be replaced in compile time. Having that said we needed a way to retrieve our resources during runtime.

<h3>The solution</h3>

In order to workaround this problem, when creating our library's JAR file, we created a folder (images/) and  placed there the icons we wanted to use.

<img src="{{ site.baseurl }}/images/android/mylibrary.png" align="center" style="width: 250px; margin: 0 auto;" >


```java
task makeJar(type: Jar) {
 ...
    into(images) {
        from 'drawables’
           include '*.png'
    }
 ...
}
```

and then when deciding to retrieve an image during runtime, we used ClassLoader class to access it: 

```java
URL url = OurClass.class.getClassLoader().getResource("images/my_image.png");
```		

if url is not null we decode to Bitmap with decodeStream:


```java
Bitmap btmp = BitmapFactory.decodeStream(url.openStream(), null, null)
```	

Note that another alternative to this approach is with <a href="http://developer.android.com/reference/java/lang/ClassLoader.html#getResourceAsStream(java.lang.String)" target="_blank">getResourceAsStream()</a> function of ClassLoader class.


However, since we wanted our library code to work also when working on our library, prior creating our jar, we used the following approach to retrieve our image resource id:

```java
if(url==null)
{
  int resourceID = context.getResources().getIdentifier("my_image", "drawable", context.getPackageName());
}
```			

and then decode our resource to a bitmap.

```java
Bitmap btmp = BitmapFactory.decodeResource(context.getResources(), resourceID, null);
	
```	
<h3>Conclusion</h3>

The approach described above is just a workaround on how to include and retrieve images from a JAR library. AAR format was introduced to resolve all these and provide a solid solution on how to include resources in a library respecting Android resources structure, which is not the case in the example above. At <a href="http://www.pollfish.com" target="_blank">Pollfish</a>, we still distribute our SDK both on JAR and AAR formats. Developer community does not always follow the latest advances of the ecosystem and more than 40% of partner apps still prefer to integrate through a JAR file.


<br/>
<h4>P.S. Tips</h4>

1. JAR solution that includes resources can be problematic in several cases. For those that work with other platforms and tools a good example can be how Unity handled JAR files with resources prior Unity 5. In Unity 4 for example when building a plugin for a JAR file you needed to include library's resources under Assets/Plugins/Android/res/raw in order for the library to be able to read them. The suggested solution above could not work and all library resources should be extracted outside the jar file. Since Unity 5 and the introduction of <a href="http://docs.unity3d.com/Manual/PluginInspector.html" target="_blank"> PluginInspector</a> even the aforementioned approach would not work any more and AAR format was somehow forced for libraries with resources.

2. This is definitely not the way to go. You can read on why, in an interesting blog <a href="http://blog.danlew.net/2013/08/20/joda_time_s_memory_issue_in_android/" target="_blank">post</a> by Dan Lew.
