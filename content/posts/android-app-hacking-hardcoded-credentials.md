+++
title = "Android App Hacking: Hardcoded Credentials"
date = "2020-05-17T00:00:00+11:00"
author = "bad5ect0r"
authorTwitter = "bad5ect0r" #do not include @
tags = ["vdp", "android"]
keywords = ["cordova", "android"]
description = "Decompiling an Android app to reveal hardcoded usernames and passwords."
showFullContent = false
+++

# Introduction

This was a relatively simple vulnerability I found for a company that deals with some potentially sensitive information. They offer paid services to their customers, but I was able to get free service by locating credentials in their Android application.

Obviously, I quickly disclosed this vulnerability responsibly and the company was very appreciative of my efforts. No bounty, but that’s okay since I mainly hack for fun rather than profit 😜. I will definitely consider purchasing their services just to keep testing for them.

# The Story

So as with any mobile app hacking, I started by downloading their APK onto my computer. You can easily do this with a service like APKPure. After that I ran apktool to unpack the APK:

```
[I] ✔︎ Test ls
drwxrwxr-x osboxes osboxes   4 KB Sun May 17 00:15:29 2020   .      
drwxrwxr-x osboxes osboxes   4 KB Sat May 16 23:37:31 2020   ..     
.rw-r--r-- osboxes osboxes 6.3 MB Sat May 16 23:37:49 2020   app.apk
[I] ✔︎ Test apktool d app.apk 
I: Using Apktool 2.4.0-dirty on app.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /home/osboxes/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
[I] ✔︎ Test ls
drwxrwxr-x osboxes osboxes   4 KB Sun May 17 00:15:41 2020   .      
drwxrwxr-x osboxes osboxes   4 KB Sat May 16 23:37:31 2020   ..     
drwxrwxr-x osboxes osboxes   4 KB Sun May 17 00:15:48 2020   app    
.rw-r--r-- osboxes osboxes 6.3 MB Sat May 16 23:37:49 2020   app.apk
[I] ✔︎ Test cd app
[I] ✔︎ app ls
drwxrwxr-x osboxes osboxes   4 KB Sun May 17 00:15:48 2020   .                  
drwxrwxr-x osboxes osboxes   4 KB Sun May 17 00:15:41 2020   ..                 
.rw-rw-r-- osboxes osboxes 2.6 KB Sun May 17 00:15:45 2020   AndroidManifest.xml
.rw-rw-r-- osboxes osboxes 467 B  Sun May 17 00:15:48 2020   apktool.yml        
drwxrwxr-x osboxes osboxes   4 KB Sun May 17 00:15:48 2020   assets             
drwxrwxr-x osboxes osboxes   4 KB Sun May 17 00:15:48 2020   original           
drwxrwxr-x osboxes osboxes   4 KB Sun May 17 00:15:45 2020   res                
drwxrwxr-x osboxes osboxes   4 KB Sun May 17 00:15:48 2020   smali              
```

The first thing I checked was the AndroidManifest.xml file. Within this file, there was mention of Apache Cordova (PhoneGap). At that time, I didn’t know what Cordova was other than hearing the name within app development circles:

```xml
<?xml version="1.0" encoding="utf-8" standalone="no"?><manifest xmlns:android="http://schemas.android.com/apk/res/android" android:hardwareAccelerated="true" package="REDACTED" platformBuildVersionCode="22" platformBuildVersionName="5.1.1-1819727">
    <supports-screens android:anyDensity="true" android:largeScreens="true" android:normalScreens="true" android:resizeable="true" android:smallScreens="true" android:xlargeScreens="true"/>
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
    <uses-permission android:name="android.permission.BLUETOOTH"/>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
    <uses-permission android:name="android.permission.RECORD_AUDIO"/>
    <uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS"/>
    <uses-permission android:name="android.permission.READ_PHONE_STATE"/>
    <application android:hardwareAccelerated="true" android:icon="@drawable/icon" android:label="@string/app_name" android:supportsRtl="true">
        <activity android:allowTaskReparenting="true" android:configChanges="keyboard|keyboardHidden|locale|orientation|screenSize" android:exported="true" android:label="@string/activity_name" android:launchMode="singleTop" android:name="REDACTED.MainActivity" android:theme="@android:style/Theme.Black.NoTitleBar" android:windowSoftInputMode="adjustResize"/>
        <activity android:excludeFromRecents="true" android:name="org.chromium.BackgroundLauncherActivity" android:taskAffinity=".launcher" android:theme="@android:style/Theme.NoDisplay">
            <intent-filter android:label="@string/launcher_name">
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <activity android:alwaysRetainTaskState="true" android:configChanges="keyboard|keyboardHidden|locale|orientation|screenSize" android:excludeFromRecents="true" android:exported="false" android:launchMode="singleTop" android:name="org.chromium.BackgroundActivity" android:taskAffinity=".cordovabackground" android:theme="@android:style/Theme.NoDisplay"/>
        <receiver android:name="org.chromium.ChromeAlarmsReceiver"/>
        <receiver android:name="org.chromium.ChromeNotificationsReceiver"/>
    </application>
</manifest>
```

With some research, I found a blog post detailing how source code could be extracted from IPAs and APKs built using Cordova/PhoneGap. TLDR: You can just go to `app.apk/assets/www/js/` to view source files.

When viewing `app.apk/assets/www/js/ViewModels/IndexViewModel.js` on this application, I was shocked to see commented out bits of code!

![app.apk/assets/www/js/ViewModels/IndexViewModel.js](/android-app-hacking-hardcoded-credentials/comments.png)

Among some of these comments, there was code that was assigning a username and password. Presumably this was done during testing to automatically authenticate the developer rather than having them manually type out the password each time:

![Plaintext credentials exposed in comments!](/android-app-hacking-hardcoded-credentials/creds.png)

I tried logging into the app using these credentials but the first few failed so I started losing hope. That quickly changed when I tried the last one which allowed me to successfully authenticate!

![*ACCESS GRANTED*](/android-app-hacking-hardcoded-credentials/loggedin.png)

# Takeaway

PhoneGap apps are fun to test since you get the source code!

# Disclosure Timeline

| Date | Details |
|------|---------|
| 21/03/2020 | Issue was reported to the company. |
| 25/03/2020 | Follow up. |
| 27/03/2020 | Acknowledged by the company. |
| 03/04/2020 | Issues were fixed. |
| 15/05/2020 | Partial disclosure was authorized. |