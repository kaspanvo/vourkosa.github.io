---
layout: post
title: How to build an iOS framework with bitcode support, keeping backwards compatibility with XCode 6
date:  2016-04-14 10:21:11
categories: iOS 
tag: ios
comments: true
---

When creating our own iOS framework, after the release of XCode 7 we needed to start supporting bitcode. For those not familiar with bitcode, bitcode is an intermediate representation of a compiled program and is part of the App Thinning feature introduced in XCode 7. With Bitcode, apps can be re-optimized for each device before being delivered to a user. You can read more on this on Apple docs <a href="https://developer.apple.com/library/tvos/documentation/IDEs/Conceptual/AppDistributionGuide/AppThinning/AppThinning.html" target="_blank">here.</a>

<h3>The problem</h3>

You can easily build an iOS library with bitcode support. In order to achieve that you just need to go to <b>Build settings</b>, search for <b>"custom compiler flags"</b> and add <b>-fembed-bitcode</b>. This flag informs XCode to build the library with bitcode and everything works fine under XCode 7. However by following the approach above we loose backwards compatibility with XCode 6. Having that said we need to ship 2 different library versions, one with bitcode flag and one without, since not everyone in our developers' community upgraded to XCode 7.

<h3>The solution</h3>

In order to be able to provide a single framework that would support both bitcode enabled or not apps, we needed to create a fat library that would include both configurations. 

We build therefore our configurations with xcodebuild both with ENABLE_BITCODE=NO and ENABLE_BITCODE=YES as shown below:

```
xcodebuild -configuration "Release" ENABLE_BITCODE=NO -target "${FMK_NAME}" -sdK iphoneos 
xcodebuild -configuration "Release" ENABLE_BITCODE=NO -target "${FMK_NAME}" -sdk iphonesimulator
xcodebuild -configuration "Release" ENABLE_BITCODE=YES -target "${FMK_NAME}" -sdk iphonesimulator
xcodebuild -configuration "Release" ENABLE_BITCODE=YES -target "${FMK_NAME}" -sdk iphoneos
```

and then create a fat library using <b>lipo</b> command like this:

```
lipo -create "${DEVICE_DIR}/${FMK_NAME}" "${SIMULATOR_DIR}/${FMK_NAME}" -output "${INSTALL_DIR}/Versions/${FMK_VERSION}/${FMK_NAME}"
```

You can easily check if a binary is bitcode compatible with <b>otool</b>:

```
otool -l (myLibName.o or myLibName.a file) | grep __LLVM.
```

The generated framework supports both bitcode-enabled and non bitcode-enabled apps, both on XCode 6 and XCode 7.


<h3>P.S. Tip</h3>

Since XCode 7.3, there is a major bug when running xcodebuild from a "Run Script", that affects the environment variable TOOLCHAINS. This is a known bug and the only workaround at the moment as described <a href="http://stackoverflow.com/questions/36184930/xcodebuild-7-3-cant-enable-bitcode"  target="_blank"> here </a> is to unset TOOLCHAINS at the beginning of your script:

```
unset TOOLCHAINS
```
