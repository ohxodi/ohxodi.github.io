---
layout: post
title:  "Verify Android application signature"
date:   2024-07-29 22:30:00 +0200
categories: Cryptography
tags: Android APK Signature
---

![Signature](/assets/signature.jpg "Signature")

## Introduction

The "official" way to get applications on your Android device is to use Google Play. You can easily find a large amount of apps and manage those you want to install or remove directly from the plateform.

There are however many reasons why you would prefer to avoid using Google Play and directly install apps on your device without using this plateform. An Android application is after all a package (*Android Package Kit* (APK) format) that can be installed directly on the device and Google Play is only a package manager (others exist on Android (e.g. [F-Droid](https://f-droid.org/){:target="_blank"})). But there are some risks to do so and this is why there are safeguards implemented by default on the operating system. Beyond those, there are also checks that can be directly done on the APK file you want to install in order to be sure that you are installing the one produced by the developer you trust. How to proceed to these verifications ? This what we are going to discuss in this article.

## Default safeguards

### Install permission

By default, Android doesn't let you install a package that you have downloaded on the device or transfered from another device. As soon as you want to launch the app installation by selecting the APK file from the file manager, Android will complain that the file manager application is not allowed to install packages.

To bypass this safeguard, go to `Settings > Apps`, select the file manager app and go to `Install unknown apps` to allow package installation. No need to keep this permission permanently, so you can remove it just after you have finished the package installation.

The same situation could occur when launching the APK file directly from your web browser. In this case, the latter should be allowed to install packages.

### Signing check

If you want to manually update an application that has already been installed on the device with Google Play for exemple, Android will probably complain, when launching the APK file, that the application is conflicting with an onther one on the device. In such a case, this is due to the signing check done by Android : both packages are signed with different keys and the operating system prevents the installation of a potentially malicious update that has not been built by an entity or a developer you trust. 

However, there are many situations where there is no security problem. Indeed, it is simply possible that the package installed with a package manager like Google Play or F-Droid is signed with a different key than the one directly downloaded for a manual installation. As described in [Android documentation](https://developer.android.com/studio/publish/app-signing#app-signing-google-play){:target="_blank"}, apps created after August 2021 must be signed with a key provided and managed by Google (app signing key). As a result, if a developer also want to provide its app outside Google Play pipeline, there is a significant chance that the APK will be signed with a different key.

In this situation, if you are sure that the APK file is signed by a trusted entity, the only way to bypass this safguard is to remove the previously installed package before installing manually the new one.

But how to manually verify a package signature ? This is in particular what we are going to discuss in the next section.

## Signing key comparison

The second safeguard described above only applies when a similar package is already installed. But when you want to manually install a new app on your device, the signature is a way to verify that the package has been built by a developer you trust. It reduces the risk that it contains milicious code which could be harmful for your device. Unfortunately, this verification needs some steps to be done.

Let say that you want to manually install Signal App on your smartphone without using Google Play. Even if it is risky and dedicated to "advanced user", Signal let you [download the APK file directly](https://signal.org/android/apk/){:target="_blank"} from their website and incite users to verify the SHA-256 fingerprint of the signing certificate. In other words, the safest way to manually install Signal is to compare the fingerprint of the signing certificate which belongs to the file you've downloaded with the one provided on Signal website. Doing so, if the verification succeeds, you can be 100% sure that the APK file has been built by Signal developers and can be safely installed on your device. (Except if the signing key has been stolen... Risk management is always a question of decision : at some point, you will have to trust enough the entity (organisation or developer) somehow.)

### Get `apksigner` tool

This verification needs a tool called `apksigner` which is provided by the Android SDK build tools. You can download [Android Studio command line tools](https://developer.android.com/studio#command-line-tools-only){:target="_blank"} to get it. As the tools are written with Java, a prerequisite is the installation of a Java JDK. For example, it can be easily installed with the command below :

```bash
sudo dnf install java-latest-openjdk # Fedora
sudo apt-get update && sudo apt-get install default-jdk # Debian
```

If you have several Java versions installed on your system and if you need to switch to the latest, you can do so with the following command :

```bash
sudo alternatives --config java # Fedora
sudo update-alternatives --config javac # Debian
```

Once the command line tools have been downloaded, you can verify the SHA-256 checksum of the zip file with the one provided on Android website before extracting the compressed files :

```bash
sha256sum commandlinetools-*.zip
# If everything is OK :
unzip commandlinetools-*.zip
cd cmdline-tools
./bin/sdkmanager --sdk_root=./ "build-tools;34.0.0"
```

If the last command above returns with no error, you can then use `apksigner` tool.

### APK signature verification

From `cmdline-tools` folder, you can run `apksigner` with the following command where `<PATH_TO_APK>` is the path to the APK file :

```bash
./build-tools/34.0.0/apksigner verify --print-certs <PATH_TO_APK> 
```

With Signal example, you should obtain the following result which is consistent with the information provided by Signal's website.

```bash
./build-tools/34.0.0/apksigner verify --print-certs ../Signal*.apk

Signer (minSdkVersion=33, maxSdkVersion=2147483647) certificate DN: CN="Signal Messenger, LLC", SERIALNUMBER=6703101, OID.2.5.4.15=Private Organization, O="Signal Messenger, LLC", OID.1.3.6.1.4.1.311.60.2.1.2=Delaware, OID.1.3.6.1.4.1.311.60.2.1.3=US, L=Mountain View, ST=California, C=US
Signer (minSdkVersion=33, maxSdkVersion=2147483647) certificate SHA-256 digest: 4be4f6cd5be844083e900279dc822af65a547fecc26aba7ff1f5203a45518cd8
Signer (minSdkVersion=33, maxSdkVersion=2147483647) certificate SHA-1 digest: 5c6740091301285db5409fdcd1b90f1ac3ba2dcf
Signer (minSdkVersion=33, maxSdkVersion=2147483647) certificate MD5 digest: 34ac0b4e5d2c5c08b704fc05874f9a10
Signer (minSdkVersion=24, maxSdkVersion=32) certificate DN: CN=Whisper Systems, OU=Research and Development, O=Whisper Systems, L=Pittsburgh, ST=PA, C=US
Signer (minSdkVersion=24, maxSdkVersion=32) certificate SHA-256 digest: 29f34e5f27f211b424bc5bf9d67162c0eafba2da35af35c16416fc446276ba26
Signer (minSdkVersion=24, maxSdkVersion=32) certificate SHA-1 digest: 45989dc9ad8728c2aa9a82fa55503e34a8879374
Signer (minSdkVersion=24, maxSdkVersion=32) certificate MD5 digest: d90db364e32fa3a7bda4c290fb65e310
```

### Source of truth

Unfortunately, many developers don't advertise the fingerprint of their signing certificate as explicitly as Signal. It is sometime possible to find it somewhere, like GitHub/GitLab issues or discussions on social networks. 

If you cannot retrieve easily the information on Internet, an option is to get it from the package obtained via the "official" way (i.e. Google Play) **if this one is signed with a developer key and not with one managed by Google as described above** (only possible for apps created before August 2021).

In such a case, you can install first the package with Google Play, transfer the APK file on your computer and get the fingerprint with `apksigner`.

However, how to transfer the APK file from your device to your computer ? This can be done with *Android Debug Bridge* (ADB) that can be installed with the following command :

```bash
sudo dnf install android-tools # Fedora
sudo apt-get update && sudo apt-get install adb # Debian
```

Before using ADB, you have to enable developer options and USB debugging on your device as described [here](https://developer.android.com/studio/debug/dev-options#enable){:target="_blank"}.

You can then check that everything is working plugging in your Android device to your computer. A prompt on the device should ask you to trust the computer. Then the following command on the computer should show the connected device :

```bash
adb devices -l
```

The list of installed packages can be obtained with the following command :

```bash
adb shell pm list packages
```

If you have multiple users on your Android device, you can get the list of installed package for a specific user with this command where `<USER_ID>` is the ID of this user obtained with `adb shell pm list users` :

```bash
adb shell pm list packages --user <USER_ID>
```

The package names are displayed with their reversed Internet domain name which is a Java convention for package naming. Find the one you are looking for and get the path of the APK file on the device with this command where `<APK_NAME>` is the reversed Internet domain name of your package :

```bash
adb shell pm path <APK_NAME>
```

Finally, you can transfer your file on your computer with the following command where `<REMOTE_APK_PATH>` is the path previously obtained and `<LOCAL_APK_PATH>` is the path on your computer where you want to store the APK file :

```bash
adb pull <REMOTE_APK_PATH> <LOCAL_APK_PATH>
```

Once the APK file is on your computer, you can use `apksigner` to get the fingerprint and compare it with the one from the APK file manually downloaded. Once a package is manually installed, the second safeguard still applies and protects your device from installing a malicious package while updating the app.

## Conclusion

Signing packages is one of the most powerfull method to ensure that files provided are built from an entity you trust. This is exactly what is done for years with Linux packages and Android has just implemented similar mechanisms. We have seen that these checks are processed in the background as package managers like Google Play are abstracting everything.

By-passing packages managers is not a reason to decrease security level and this article has shown methods to check that a package is the right one, provided by an entity you trust, before installing it manually on your Android device.

## Sources
- [Verifying APK Fingerprints](https://www.privacyguides.org/en/android/obtaining-apps/#verifying-apk-fingerprints){:target="_blank"}
- [Another way to verify APK Fingerprints](https://github.com/M66B/FairEmail?tab=readme-ov-file#downloads){:target="_blank"}
- [ADB Documentation](https://www.privacyguides.org/en/android/obtaining-apps/#verifying-apk-fingerprints){:target="_blank"}