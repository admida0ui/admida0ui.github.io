---
title: Install Flutter on WSL2 and run apps directly in WSA or in a physical device using ADB.
date: 2021-12-1 21:00:00
tags: [flutter, wsl2, windows, linux, android]
categories: Work
thumbnail: "https://i.gifer.com/9wb3.gif"
---

I have successfully been able to work on Flutter on WSL2 and the new WSA in Windows 11 without the need of Android Studio.
This is very useful if you are a fan of Microsoft ecosystem or you don’t see the need of Android Studio and its emulators as we now have WSA.

Originally published on [Medium](https://medium.com/@adhbbam/install-flutter-on-wsl2-and-run-apps-directly-in-wsa-or-in-a-physical-device-using-adb-3602f2053f8e).

# Installing WSA — Windows Subsystem For Android

You will need to have “Virtual Machine Platform” enabled in “Optional Features” in Control Panel.

![](images/2024-05-23-16-32-39.png)

The best way to do this for now, is to head to rg-adguard.net, select “Productid” in the first drop-down menu. Put “9p3395vx91nr” in the search field. And select “slow” in the next drop-down.

Scroll down to the bottom of the page, and download the largest file in size, it should be mostly the last one (approximately 1.2GB).

![](images/2024-05-23-16-33-56.png)

Once the download is complete, right-click the start menu button and open Windows Terminal (Admin), switch to the path where you downloaded the file, usually cd %USERPROFILE%/downloads. and execute the following command:
Add-AppxPackage ./MicrosoftCorporationII.WindowsSubsystemForAndroid_1.8.32828.0_neutral_~_8wekyb3d8bbwe.msixbundle

![](images/2024-05-23-16-34-15.png)

Note that the version might change there, so copy the name of the file and paste it next to Add-AppxPackage.

Wait for the installation to complete, Windows 11 will also install the Amazon app store for you automatically.

We’re not done yet with WSA, you still need to enable developer mode so we can interact with it using adb.

![](2024-05-23-16-34-35.png)

Remember to enable Developer Mode

Switch Subsystem resources to “Continuous” when you are trying to run your app later, as it takes time, I noticed that with the “As needed” option, Flutter won’t be able to wake up your device.

Of course to save system resources, you can turn it back to “As needed” once you’re done with Flutter.

Here ends the part of WSA Installation, we will move to Flutter installation on WSL2 which is the same process as installing Flutter in a Linux distro.

# Installing Flutter On WSL2

I will assume that you already have WSL2 installed, if not, it is an easy process, just use the command in Windows Terminal (Admin): “wsl — install -d <Distro>” Distro can be Ubuntu, Debian, etc… You can easily just use the Microsoft Store instead and look up the distro of your choice.

For this tutorial, I will be using Ubuntu but it is a similar process for Debian and other Distros.

You will need to enable “Virtual Machine Platform” and “Windows Subsystem for Linux” using “Optional Features” in Control Panel and restart Windows.

## Install OpenJDK8

I am using JAVA 8 since the recent version of JAVA have some problems with Flutter, JAVA 8 should be fine.

In WSL2 terminal type:

```bash
sudo apt install openjdk-8-jre
sudo apt install openjdk-8-jdk
```

## Install Git

```bash
sudo apt install git
```

## Setting up Flutter

It is better to keep everything in a single directory, I will be using `$HOME/Android`.

```bash
cd ~
mkdir Android
cd Android
mkdir cmdline-tools
```

## Install Flutter SDK (Latest Version)

Download here (Install Flutter manually section): [Linux install | Flutter](https://docs.flutter.dev/get-started/install/linux).

Go to the downloaded location of the compressed archive and extract it

```bash
tar xvf flutter_linux_2.5.3-stable.tar.xz
```

Versions may differ!

Now, move “flutter” folder to “~/Android/” directory

```bash
sudo mv flutter/ ~/Android/
```

You can delete the archive now.

```bash
rm flutter_linux_2.5.3-stable.tar.xz
```

## Install Android Command Line Tools

Download the Android command line tools for Linux from here: [https://developer.android.com/studio#command-tools](https://developer.android.com/studio#command-tools)

Go to the download location of the compressed archive and unzip it.

```bash
unzip commandlinetools-linux-7583922_latest.zip
```

Versions may differ!

Move the “tools” folder to “~/Android/cmdline-tools/”

```bash
sudo mv tools/ ~/Android/cmdline-tools/
```

You can delete the zip archive now.

```bash
rm commandlinetools-linux-7583922_latest.zip
```

## Setting Up Environment Variables

Now, go back to home directory and open up “.bashrc” file with an editor of your choice.

```bash
cd ~
vi .bashrc
```

Add the following lines at the end of the file:

```bash
export ANDROID=$HOME/Android
export PATH=$ANDROID/cmdline-tools/tools:$PATH
export PATH=$ANDROID/cmdline-tools/tools/bin:$PATH
export PATH=$ANDROID/platform-tools:$PATH
# Android SDK
export ANDROID_SDK=$HOME/ANDROID
export PATH=$ANDROID_SDK:$PATH
# Flutter
export FLUTTER=$ANDROID/flutter
export PATH=$FLUTTER/bin:$PATH
```

Save the file. And restart your terminal or use this command

```bash
source .bashrc
```

## Download Android SDK

For Flutter to run, we still need to install system-image, platform-tools, build-tools, platforms, android.

I will be using android-29 for Android 10 SDK, if you want Android 11 SDK just change to android-30.

[You can use the version that is compatible with your device](https://developer.android.com/studio/releases/platforms). or you can check it from your terminal

```bash
sdkmanager --list
```

for version 29, type the following:

```bash
sdkmanager "system-images;android-29;default;x86_64"
sdkmanager "platforms;android-29"
sdkmanager "platform-tools"
sdkmanager "build-tools;29.0.3"
```

Now, accept the licenses

```bash
sdkmanager --licenses
```

## Configure SDK Path For Flutter

```bash
flutter config --android-sdk ~/Android
```

## Checking Flutter Doctor

We’re done with Flutter installation, let’s go check Flutter Doctor

```bash
flutter doctor -v
```

This command should give you all green tick, ignore Android Studio and Chrome. you can still install chrome for WSL2 since it support GUI now. Or just use your Windows browser. I believe it is possible…

![](images/2024-05-23-16-41-11.png)

If Flutter doctor asks for accepting licenses, use this command

```bash
flutter doctor --android-licenses
```

# Installing ADB on Windows

This step is optional, unless you want to test your app not only on WSA but also on a physical device.

You just need to have the very same version of your Linux ADB inside Windows Powershell.

![](images/2024-05-23-16-42-15.png)

![](images/2024-05-23-16-42-40.png)

It is exactly the same version.

ADB will be installed automatically within the platform-tools we installed in the previous step, so check “adb — version” in WSL2. Then install the same one on Powershell. This website will help you: [Download Platform Tools for Android SDK Manager](https://androidsdkmanager.azurewebsites.net/Platformtools).

We will be using ADB over WiFi, since WSL2 still doesn’t have access to host USB devices.

So connect your device using USB. make sure it is connected to the same WiFi as your computer, and type the following in Powershell:

```bash
adb kill-server
adb devices
adb tcpip 5555
```

It should show your physical device in the list, now you jump to WSL2 and connect to it using the IP Address of the device and the port you picked, for me 5555

```bash
adb connect <IP_ADDRESS>:<PORT>
```

![](images/2024-05-23-16-43-46.png)

And Flutter Doctor is recognizing it:

![](images/2024-05-23-16-44-03.png)

As well as Visual Studio Code.

![](images/2024-05-23-16-44-24.png)

For Windows 10 users, WSA is not yet available for you, the only way to run your app is by using a physical device or maybe using an emaulator since GUI is now supported in WSL2 and should be in Windows 10 21H2, to install a virtual emulator use the following command in WSL2:

Check the available virtual devices

```bash
avdmanager list
```

Copy the desired device ID

Give the emulator a name and paste the ID

```bash
avdmanager -s create avd -n [name] -k "system-images;android-29;google_apis;x86_64" -d [id_device]
```

Once done, it should show up in the Flutter Doctor output.

If you encounter issues with ADB during connection with a physical device, try to ping the IP address, if it wasn’t successful, go to the Firewall settings and create a firewall entry, create a new “Inbound” Rule. Select “Port” then Specific TCP port “5555” then “Allow the connection” then check all of Domain, Private and Public and add a name. After the firewall entry is added, open up its properties, go to Scope -> Remote IP Addresses -> Add “192.168.1.0/24” or your specific LAN IP range.

# Running A Flutter App On WSA

This is the moment of truth, let’s check if it is possible to run a Flutter app built on WSL, on WSA.

Let’s use the Hello World template

```bash
cd ~
flutter create hello_world
cd hello_world
```

Copy the WSA IP Address from the WSA App and make sure to turn on Developer Mode, and put it into Continuous mode during all the test.

![](images/2024-05-23-16-45-40.png)

Now, connect the device using ADB

```bash
adb connect 172.25.97.191:5555
```

The IP address changes everytime you start the WSA app, so make sure to keep that in mind.

![](images/2024-05-23-16-46-07.png)

Flutter Doctor and VS Code.

![](images/2024-05-23-16-46-28.png)

![](images/2024-05-23-16-46-34.png)

Now, let’s run the app

```bash
flutter run
```

Give it some time… And Voila!

![](images/2024-05-23-16-47-05.png)

WSA serves as both an Android emulator for your Apps like Android Studio as well as a Virtual Machine for your favorite apps and games.

Thanks for following this tutorial, don’t forget to share it with people who might need it.
