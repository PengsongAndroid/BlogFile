---
title: 坑爹的Android 6.0 startDiscovery蓝牙搜索不到设备问题
date: 2017-03-27 20:50:44
categories: Android日常问题
tags: Android, Android权限机制
---

最近在做重构项目的过程中，重构蓝牙扫描模块的时候。发现调用扫描方法的时候，根本没有返回搜索到的设备。开始以为是厂商sdk的bug，或者是自己新写的代码问题。然后没有用厂商sdk，自己写代码来扫描，琢磨了半天看日志，debug，依然没找到问题。后来突然想到是不是权限的问题，之前项目的targetSdkVersion 22，新的项目是23，然后Android 6.0有一套新的权限机制，敏感权限需要申请，感觉可能是权限问题导致的。
<!-- more -->
搜索了一下，下面是具体的权限列表

#### Normal Permission
```java
ACCESS_LOCATION_EXTRA_COMMANDS
ACCESS_NETWORK_STATE
ACCESS_NOTIFICATION_POLICY
ACCESS_WIFI_STATE
BLUETOOTH
BLUETOOTH_ADMIN
BROADCAST_STICKY
CHANGE_NETWORK_STATE
CHANGE_WIFI_MULTICAST_STATE
CHANGE_WIFI_STATE
DISABLE_KEYGUARD
EXPAND_STATUS_BAR
GET_PACKAGE_SIZE
INSTALL_SHORTCUT
INTERNET
KILL_BACKGROUND_PROCESSES
MODIFY_AUDIO_SETTINGS
NFC
READ_SYNC_SETTINGS
READ_SYNC_STATS
RECEIVE_BOOT_COMPLETED
REORDER_TASKS
REQUEST_INSTALL_PACKAGES
SET_ALARM
SET_TIME_ZONE
SET_WALLPAPER
SET_WALLPAPER_HINTS
TRANSMIT_IR
UNINSTALL_SHORTCUT
USE_FINGERPRINT
VIBRATE
WAKE_LOCK
WRITE_SYNC_SETTINGS
```

#### Dangerous Permission
```java
group:android.permission-group.CONTACTS
  permission:android.permission.WRITE_CONTACTS
  permission:android.permission.GET_ACCOUNTS
  permission:android.permission.READ_CONTACTS

group:android.permission-group.PHONE
  permission:android.permission.READ_CALL_LOG
  permission:android.permission.READ_PHONE_STATE
  permission:android.permission.CALL_PHONE
  permission:android.permission.WRITE_CALL_LOG
  permission:android.permission.USE_SIP
  permission:android.permission.PROCESS_OUTGOING_CALLS
  permission:com.android.voicemail.permission.ADD_VOICEMAIL

group:android.permission-group.CALENDAR
  permission:android.permission.READ_CALENDAR
  permission:android.permission.WRITE_CALENDAR

group:android.permission-group.CAMERA
  permission:android.permission.CAMERA

group:android.permission-group.SENSORS
  permission:android.permission.BODY_SENSORS

group:android.permission-group.LOCATION
  permission:android.permission.ACCESS_FINE_LOCATION
  permission:android.permission.ACCESS_COARSE_LOCATION

group:android.permission-group.STORAGE
  permission:android.permission.READ_EXTERNAL_STORAGE
  permission:android.permission.WRITE_EXTERNAL_STORAGE

group:android.permission-group.MICROPHONE
  permission:android.permission.RECORD_AUDIO

group:android.permission-group.SMS
  permission:android.permission.READ_SMS
  permission:android.permission.RECEIVE_WAP_PUSH
  permission:android.permission.RECEIVE_MMS
  permission:android.permission.RECEIVE_SMS
  permission:android.permission.SEND_SMS
  permission:android.permission.READ_CELL_BROADCASTS
```

看了一下蓝牙权限不是敏感权限，应该不需要在运行时获取啊。然后查了下[官方文档](https://developer.android.com/about/versions/marshmallow/android-6.0-changes.html#behavior-power)。
有这么一段内容：
![](http://7xrxl6.com1.z0.glb.clouddn.com/permission1.png)

所以在通过蓝牙扫描附近外部设备时，需要获取这两个权限。
```
ACCESS_FINE_LOCATION
ACCESS_COARSE_LOCATION
```
检查和申请权限可以调用这两个方法，具体如何使用可以自行搜索。
```
checkSelfPermission()
requestPermissions()
```

处理权限的代码并不复杂，但是需要自己去封装。
我自己是使用的一个开源库：https://github.com/lovedise/PermissionGen
使用起来比较方便，感兴趣的可以看看。