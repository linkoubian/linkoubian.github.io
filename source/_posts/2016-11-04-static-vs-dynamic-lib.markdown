---
layout: post
title: iOS 静态库及动态库开发
date: 2016-11-04 18:48:51 +0800
comments: true
categories: iOS
---
本文总结在好房移动架构团队做 Framework 开发中的一些经验。

<!--more-->

之前负责好房 APP 开发时，需要支持 iOS 7+，所以五月份设计统计 SDK 时只好采用静态库的方式。随着 iOS 10 的推出，iOS 7 的支持默认被移除，结合 APP 的用户设备分布，目前 APP 已改为支持 iOS 8+，所以上个月设计的 React Native 增量 Patch 更新 SDK 可采用动态库的方案。

至于 iOS 中静态库与动态库的差别，网上有很多文章介绍，本文不再赘述，而将重点放在这两种库的具体开发实现过程。

## 静态库
Google 的工程师已经写了一篇非常赞的[文章](https://github.com/jverkoey/iOS-Framework)，好房统计 SDK 就是按照此方案一步步配置的。经验证，效果非常好。其提供的脚本也很精致，无冗余。

当然 Raywenderlich 的网站也有一篇[文章](https://www.raywenderlich.com/65964/create-a-framework-for-ios)，方案类似，相比于 Google 程序员写的指南，多个实例。初次接触静态库开发的开发者可以读一读。

故此处也不再重复介绍。

## 动态库
Xcode自带的 framework 模板，创建的动态库（包含资源）可以在iOS 7上跑（真机测试过），但官方要求iOS 8+，可能提交 app store 验证不过。

更大的问题就是提交 app store 时会提示包含 x86_64, i386 ... 截图[在此](http://ikennd.ac/pictures/iTC-Unsupported-Archs.png)。该问题在 Xcode 6.3.2 之前及 7.1 上都有开发者遇到，PSPDFKit 这个库的开发者是在分发动态库时在 framework 里嵌入一个 shell 脚本，供使用方在 Xcode 里运行。我最终没有采用该方式。

关于动态库的更多讨论，有一篇[文章](http://stackoverflow.com/questions/30963294/creating-ios-osx-frameworks-is-it-necessary-to-codesign-them-before-distributin)值得一看。

有上述背景知识，我们的动态库的具体做法：   
### 针对 Dynamic Library 工程中 Aggregate 构建目标的脚本
仿照 jverkoey 文章中的 Aggregate 脚本，很 Easy 的。

```bash
set -e
set +u

# Avoid recursively calling this script.
if [[ $SF_MASTER_SCRIPT_RUNNING ]]
then
exit 0
fi
set -u
export SF_MASTER_SCRIPT_RUNNING=1

SF_TARGET_NAME=${PROJECT_NAME}
SF_EXECUTABLE_PATH=${PROJECT_NAME}
SF_WRAPPER_NAME="${SF_TARGET_NAME}.framework"

# The following conditionals come from
# https://github.com/kstenerud/iOS-Universal-Framework

if [[ "$SDK_NAME" =~ ([A-Za-z]+) ]]
then
SF_SDK_PLATFORM=${BASH_REMATCH[1]}
else
echo "Could not find platform name from SDK_NAME: $SDK_NAME"
exit 1
fi

if [[ "$SDK_NAME" =~ ([0-9]+.*$) ]]
then
SF_SDK_VERSION=${BASH_REMATCH[1]}
else
echo "Could not find sdk version from SDK_NAME: $SDK_NAME"
exit 1
fi

if [[ "$SF_SDK_PLATFORM" = "iphoneos" ]]
then
SF_OTHER_PLATFORM=iphonesimulator
else
SF_OTHER_PLATFORM=iphoneos
fi

if [[ "$BUILT_PRODUCTS_DIR" =~ (.*)$SF_SDK_PLATFORM$ ]]
then
SF_OTHER_BUILT_PRODUCTS_DIR="${BASH_REMATCH[1]}${SF_OTHER_PLATFORM}"
else
echo "Could not find platform name from build products directory: $BUILT_PRODUCTS_DIR"
exit 1
fi

rm -r "${BUILT_PRODUCTS_DIR}/${SF_WRAPPER_NAME}/_CodeSignature"

# Build the other platform.
xcrun xcodebuild -project "${PROJECT_FILE_PATH}" -target "${TARGET_NAME}" -configuration "${CONFIGURATION}" -sdk ${SF_OTHER_PLATFORM}${SF_SDK_VERSION} BUILD_DIR="${BUILD_DIR}" OBJROOT="${OBJROOT}" BUILD_ROOT="${BUILD_ROOT}" SYMROOT="${SYMROOT}" $ACTION

# Smash the two static libraries into one fat binary and store it in the .framework
xcrun lipo -create "${BUILT_PRODUCTS_DIR}/${SF_WRAPPER_NAME}/${SF_EXECUTABLE_PATH}" "${SF_OTHER_BUILT_PRODUCTS_DIR}/${SF_WRAPPER_NAME}/${SF_EXECUTABLE_PATH}" -output "${BUILT_PRODUCTS_DIR}/${SF_WRAPPER_NAME}/${SF_TARGET_NAME}"

# Copy the binary to the other architecture folder to have a complete framework in both.
cp -a "${BUILT_PRODUCTS_DIR}/${SF_WRAPPER_NAME}/${SF_TARGET_NAME}" "${SF_OTHER_BUILT_PRODUCTS_DIR}/${SF_WRAPPER_NAME}/${SF_TARGET_NAME}"
```

### 业务方工程在 Embed Framework 之后增加[脚本](https://github.com/realm/realm-cocoa/blob/f07d1af226b67c0aefb150d12da3fd34c5d64087/scripts/strip-frameworks.sh)
该脚本从动态库里移除不必要的处理器架构。因为改动了 ipa 中动态库的可执行文件，所以该 strip 脚本还需要重新对动态库中可执行文件签名。正因为会重新签名，Embed Framework 处不必勾选 Code Sign on Copy。

## 第三方依赖
我的做法是尽量不引入第三方代码到我的 Framework 中。但如果有些算法类的库，比如 ZipArchive 等，很多时候还是需要在 Framework 中用的。直接把第三方代码拉进来，可能会和业务方引入的代码冲突。怎么办？

Kamil Burczyk 在他的[文章](http://blog.sigmapoint.pl/avoiding-dependency-collisions-in-ios-static-library-managed-by-cocoapods/)中给出了一种通过脚本改类名的方案。

我们的方案是**在 Framework 的工程里，只引入第三方的头文件**。要求业务方使用时确保引入第三方库。

