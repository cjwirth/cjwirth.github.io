---
layout: post
title: Cleaning Up Old Xcode Files
---

Today I freed up over 200GB of space on my hard drive without losing anything of value. A _large_ chunk of this was files created by and for Xcode. Finding and deleting them was pretty easy to do.

<!--excerpt-->

As time goes on, support files for Xcode accumulate and take up more and more space. It's good to take a look around to see if you really need what's in there.

## Simulator Runtimes

If you've dropped support for an old version of iOS, then you don't really need its runtime sticking around taking up space. Xcode's preferences give you an option to download more, but it doesn't give you an option to _delete_ them. You can still delete them manually, though.

```shell
> ls /Library/Developer/CoreSimulator/Profiles/Runtimes

'iOS 10.3.simruntime'
```

I only have 10.3 left, but I also had iOS 8 and 9 support! That alone freed up a fair amount of gigabytes! Plus, should I ever need to re-run an iOS 8 simulator, I can download it again from Xcode.

## Simulators

Speaking of old simulator runtimes, you probably have old simulators that you don't need either.

The first thing to do is delete simulators that are unsupported by the current Xcode SDK

```shell
> xcrun simctl delete unavailable
```

There are a lot more commands you can do from the command line to view, create, or delete simulators that currently exist.

You can also manage simulators in Xcode, but it doesn't give you the option to delete all unavailable simulators, so the command line might be nice in this case.

## Device Support

In order for Xcode to know how to deal with the OS on your phone, it needs Device Support files. If you've plugged your phone into your computer after upgrading your OS, you've seen it processing before you're able to debug on it.

These files are roughly 2.5GB for each version of iOS that your computer needs to talk to. If you update frequently (e.g. you install _all_ the betas), then you most likely have a lot of Device Support files for iOS versions you will never need again. I had 6 versions of iOS 12.0, 4 versions of 12.1.X, and 4 versions of 12.2!

Feel free deleting any old Device Support files that you don't need. Xcode will regenerate them the next time you plug in an unrecognized iOS anyway.

```shell
> rm -rf "~/Library/Developer/Xcode/iOS DeviceSupport/10.3.3 (14G60)"
```

## Derived Data

Xcode creates a lot of data from your projects and stores it in Derived Data. This is helpful and useful for projects that you are working on, but not so much for projects you aren't ever going to open again. Unfortunately, it all remains there.

You can gain back another few gigabytes by deleting everything in it, or you can be selective by only deleting the ones you don't need anymore.

```shell
> rm -rf ~/Library/Developer/Xcode/DerivedData
```

## ⚠️ Archives

This one might be a bit controversial. Archives are the finished products of apps you have made in the past, and include the debug symbols which are needed to symbolicate crash logs. It is probably a good idea to have them backed up somewhere, but I don't think it's necessary to keep them locally for all time.

Archives can be deleted through Xcode's Organizer window, but it doesn't allow multiple selection, so you'll have to delete them all one-by-one. 

You can delete them in bulk in Finder or on the command line. They are all stored in `~/Library/Developer/Xcode/Archives/`, but are organized by the date they were made. Take care not to delete archives that you actually do care about before deleting them all!

## Other Opportunities

### Caches

Another place to look for files to delete would be in `~/Library/Caches`. It's possible you might see directories for apps you have have deleted long ago.

I found that I had a 20GB directory at `~/Library/Cache/org.carthage.CarthageKit` even though I currently use CocoaPods for dependency management!

### Logs

It [seems like](https://twitter.com/mjtsai/status/1022838770494832643) the iOS simulator logs can also accumulate. They are stored in `~/Library/Logs/CoreSimulator`, so maybe take a look there. Mine were less than 1GB, so I didn't look into what they were or if they were needed. It might be worth checking though!

## Fin

I was starting to get a little nervous as I saw my available hard drive space dwindling away day by day. I knew that there must have been some kinds of unneeded files or caches that I could get rid of. Little did I know it would add up to over 200GB! I'll have to check up on it periodically to make sure I can keep my disk usage low.