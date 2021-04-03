---
layout: post
title:  "[ios] What's the App Store file size of my app?"
categories: ios app-store
redirect_from:
  - /ios/app-store/2017/02/18/ios-app-store-file-size/
---

I was evaluating some libraries to include in the project and their size was considerably big. So I wanted to know how would that new library affect the size of my app. But I didn’t even know what’s the current size of my app. To find that out, there were a couple of things I checked:
* size of the project folder: 41MB
* size of the archive that is uploaded to the App Store: 184MB
* size of the app stated in the app details in the App Store: 10 MB

Well, that’s kinda weird. What is going on here?

According to the Apple documentation, different artifacts contain different bits of data in different formats:

* An **App Store submission .ipa** is created from an Xcode archive when uploading to the App Store or by exporting the archive for iOS App Store Deployment. This .ipa is a compressed directory containing the app bundle and additional resources needed for App Store services, such as .dSYM files for crash reporting and asset packs for On Demand Resources.
* A **universal .ipa** is a compressed app bundle that contains all of the resources to run the app on any device. Bitcode has been recompiled, and additional resources needed by the App Store, such as .dSYM files and On Demand Resources, are removed. For App Store apps, this .ipa is downloaded to devices running iOS 8 or earlier.
* A **thinned .ipa** is a compressed app bundle that contains only the resources needed to run the app on a specific device. Bitcode has been recompiled, and additional resources needed by the App Store, such as .dSYM files and On Demand Resources, are removed. For App Store apps, this .ipa is downloaded to devices running iOS 9 or later.

In my case, the App Store submission .ipa is 184MB and the thinned .ipa is 10MB. If you want to find out what would be the thinned .ipa size for your app, here is what you need to do:

1. Export your archive for testing outside the store.
2. Select “Export for specific devices” and choose “All compatible device variants” from the pop-up menu.
3. Select "Rebuild from bitcode.”
4. In the output folder, you will find App Thinning Size Report.txt, which breaks down the compressed and uncompressed file sizes, plus the size of any On Demand Resources, for each device type.

References
----------
* [https://developer.apple.com/library/content/qa/qa1795/_index.html](https://developer.apple.com/library/content/qa/qa1795/_index.html){:target="_blank"}
