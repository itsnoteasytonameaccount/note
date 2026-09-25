---
title: Android SDK 配置
date: 2022/11/15
tags:
- android
---

某些原因需要用一下avd，但是不想打开android studio，因此用命令行打开，结果发现报错，查看[官方文档](https://developer.android.google.cn/studio/command-line?hl=zh_cn)，发现本地很多环境配置过期了

<!-- more -->

# 变化配置

## `ANDROID_HOME`

原来的`ANDROID_SDK_ROOT`弃用了，改为`ANDROID_HOME`

## emulator的路径

由`$ANDROID_SDK_ROOT/tools`变为`ANDROID_HOME/emulator`

## avdmanager的路径

由`$ANDROID_SDK_ROOT/tools/bin`变为`$ANDROID_HOME/cmdline-tools`

# 其他重要配置

| 配置名                  | 作用                                                         |
| ----------------------- | :----------------------------------------------------------- |
| `ANDROID_EMULATOR_HOME` | 设置特定于用户的模拟器配置目录的路径。默认设置为 `$ANDROID_USER_HOME`较旧的工具（如 Android Studio 4.3 及更早版本）不会读取 `ANDROID_USER_HOME`，对于这些工具，默认值为 `$ANDROID_SDK_HOME/.android`。 |
|ANDROID_USER_HOME |为 Android SDK 中包含的工具设置用户偏好设置目录的路径。 默认为 $HOME/.android/。<br/><br/>某些较旧的工具（例如 Android Studio 4.3 及更低版本）不会读取 ANDROID_USER_HOME。 如需替换这些旧工具的用户偏好设置位置，请将 ANDROID_SDK_HOME 设置为要在其下创建 .android 目录的父目录。|
|`ANDROID_AVD_HOME` |设置包含所有 AVD 特定文件的目录的路径，这些文件大多包含非常大的磁盘映像。默认位置是 `$ANDROID_EMULATOR_HOME/avd/`。如果默认位置的磁盘空间不足，您可能需要指定新位置。|

# adb install 时报错



1. INSTALL_PARSE_FAILED_NO_CERTIFICATES: Failed collecting certificates for /data/app/vmdl691559446.tmp/base.apk: Failed to collect certificates from /data/app/vmdl691559446.tmp/base.apk: Attempt to get length of null array 

   **这个错代表未签名**

2. INSTALL_PARSE_FAILED_NO_CERTIFICATES: Failed collecting certificates for /data/app/vmdl1621401341.tmp/base.apk: Failed to collect certificates from /data/app/vmdl1621401341.tmp/base.apk: META-INF/MYRSACER.SF indicates /data/app/vmdl1621401341.tmp/base.apk is signed using APK Signature Scheme v2, but no such signature was found. Signature stripped?] 

   **这个错众说纷，有说重签名前需要删除apk中的已有签名文件的，也有说是bug的，本人尝试签名后不能进行字节对齐，需要先对齐字节再签名**

3. -124: Failed parse during installPackageLI: Targeting R+ (version 30 and above) requires the resources.arsc of installed APKs to be stored uncompressed and aligned on a 4-byte boundary

   **需要对齐字节**

   

# 重签名命令

```bash
# 生成证书并且密码是123456,有效期36000天
keytool -genkey -alias myrsacert -keyalg RSA -keystore myrsacert.jks -storepass 123456 -keypass 123456 -validity 36000
# v1签名
jarsigner -verbose -keystore myrescert.jks -signedjar output.apk RemotePrinting.apk myrsacert
# v1~v3签名
apksigner.bat sign --ks myrsacert.jks --ks-key-alias myrsacert --out app_signed.apk app.apk
# 字节对齐四字节
zipalign.exe -v 4 app.apk app_aligned.apk
# 验证apk签名
apksigner.bat verify --verbose --print-certs app_signed.apk
```

