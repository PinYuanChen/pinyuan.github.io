---
layout: post
title: 'Prevent Losing All Your Data After Support Cloud Kit for Core Data'
categories: [iOS dev]
tag: [swift, core data, cloud kit]
---

I recently utilized Core Data as the local persistence layer for my side project and discovered its compatibility with CloudKit for remote storage. Given that both Core Data and CloudKit are offerings from Apple, I was eager to test their integration. However, after following the [WWDC demo](https://developer.apple.com/videos/play/wwdc2019/202/) on migrating from `NSPersistentContainer` to `NSPersistentCloudKitContainer`, I encountered unexpected challenges. The most significant, and surprisingly unmentioned, issue was the loss of all my existing data after transitioning to the CloudKit container.

Imagine how shocking it is when you find your existing data all vanished. Upon researching online, I found others facing the [same problem](https://developer.apple.com/forums/thread/120328). One key step is, you have to set `NSPersistentHistoryTrackingKey` to `true` when you assign the description to your `NSPersistentContainer`. In that way, CoreData can track persistent history and migrate your existing data to the CloudKit container.

However, according to the forum discussion posted above, this modification is not enough to ensure your data intact. `@andrewbuilder` listed all the [scenarios](https://developer.apple.com/forums/thread/120328?answerId=398351022#398351022) and it manifested that:
>
Records entered prior to setting NSPersistentHistoryTrackingKey = true are never synced to users CloudKit account and remain only on the device on which they are created.
>