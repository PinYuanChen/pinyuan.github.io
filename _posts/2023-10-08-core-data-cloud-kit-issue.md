---
layout: post
title: 'Data Sync Problem After Support Cloud Kit for Core Data'
categories: [iOS dev]
tag: [swift, core data, cloud kit]
---

I recently utilized Core Data as the local persistence layer for my side project and discovered its compatibility with CloudKit for remote storage. Given that both Core Data and CloudKit are offerings from Apple, I was eager to test their integration. However, after following the [WWDC demo](https://developer.apple.com/videos/play/wwdc2019/202/) on migrating from `NSPersistentContainer` to `NSPersistentCloudKitContainer`, I encountered unexpected challenges. The most significant, and surprisingly unmentioned, issue was the all my existing data did not sync to the cloud after transitioning to the CloudKit container.

Imagine how disturbing it is when you find your data not sync properly across devices. Upon researching online, I found others facing the [same problem](https://developer.apple.com/forums/thread/120328). One key step is, you have to set `NSPersistentHistoryTrackingKey` to `true` when you assign the description to your `NSPersistentContainer`. In that way, CoreData can track persistent history and migrate your existing data to the CloudKit container.

However, according to the forum discussion posted above, this modification is not enough to ensure your data intact. `@andrewbuilder` listed multiple [scenarios](https://developer.apple.com/forums/thread/120328?answerId=398351022#398351022) incurred by various settings and he manifested that:

>
Records entered prior to setting NSPersistentHistoryTrackingKey = true are never synced to users CloudKit account and remain only on the device on which they are created.
>

That really sucks... 🤮

Dmitry Deplov addressed [this problem](https://medium.com/@dmitrydeplov/coredata-cloudkit-integration-for-a-live-app-57b6cfda84ad) by copying all current data to a new context, saved them to the new cloud container, and then deleted the old one. This is a workaround that ensured you transform data properly.
It sets me thinking that when implementing a new feature, even it is provided by Apple, don't believe it can integrate smoothly. In fact, there are still many missing pieces in official documents and we developers should find solutions on our own.
That's it! If you have any questions or recommendations, please leave a comment down below. See you at the top! 🪣

