---
layout: post
title: 关于 iOS 私有 API 扫描
date: 2018-09-11 09:23:16 +0800
comments: true
categories: iOS
---

近期研究了关于私有 API 扫描这个主题。研读了业界现有的相关文章后发现，很多都是简单的摘录，也不对存在的谬误做任何点评。本人在阅读了网易游戏开源的 iOS private api checker 项目后，对如何构建私有 API 库、该项目又是如何识别 APP 中的私有 API、该方案存在哪些问题，一一做了阐述。

<!--more-->

## 审核案例

1. [自定义方法和私有 API 重名](http://www.cocoachina.com/bbs/read.php?tid=12514&page=1)

   APP 没有被拒绝，但是 Apple 提醒下次更新时修改相关 API 名称。

   然而多年前，广为使用的 Three20 里包含和私有 API 重名的[方法](https://stackoverflow.com/questions/1865778/iphone-app-rejected-because-of-three20-non-public-api-lineheight-and-previo)，导致很多使用该框架的 APP 审核不通过。

2. [使用了非公开方法](https://stackoverflow.com/questions/18756906/apple-reject-my-app-because-using-private-api-allowsanyhttpscertificateforhos)

   Apple 发现 APP 使用了非公开的方法 **```allowsAnyHTTPSCertificateForHost:```**，拒绝的同时还提供开发者自查的方法。

3. 未执行到的私有 API 调用

   Qzone 中曾自定义接口 **```_define:```** 但是并没有调用过，结果也被 Apple 发现并拒绝上架。```UITextView``` 导出的头文件中有该方法。

4. [Tim Cook 威胁下架 Uber 应用](https://www.nytimes.com/2017/04/23/technology/travis-kalanick-pushes-uber-and-himself-to-the-precipice.html)

   Uber 使用私有 API获取设备的序列号，苹果 CEO 严厉斥责该行为并威胁要下架。

## 调用方式

### 直接调用
```objc
[self.view recursiveDescription];
```
因为私有 API 没有暴露出来，编译会报错。可以添加匿名 ```Category``` 声明下私有方法。

```objc
@interface UIView()
-(id)recursiveDescription;
@end
```

### 字符拼接

```objc
NSArray *parts = @[@"_priva", @"teMethod"];
NSString *selectorString = [parts componentsJoinedByString:@""];
[self performSelector:NSSelectorFromString(selectorString) withObject:nil];
```

### 代码混淆

```objc
// statusBar
NSData *data = [NSData dataWithBytes:(unsigned char[]){0x73, 0x74, 0x61, 0x74, 0x75, 0x73, 0x42, 0x61, 0x72} length:9]; 
NSString *key = [[NSString alloc] initWithData:data encoding:NSASCIIStringEncoding];
```

## 检测方法

### 符号表

用 ```nm```, ```otool``` 等工具导出二进制包的函数符号表，以检查私有 API 的调用。缺点是无法检测字符串拼接方法的私有 API 调用。

### 动态扫描

动态扫描需要应用运行起来，每当调用方法时就判断是否是私有 API，但是效率会很低，而且不能保证代码完全覆盖。

### 静态分析

在对二进制文件反汇编结果的基础上，进行静态分析：

1. 找出动态调用 API 方法如 `performSelector:` ，以及调用对象的类
2. 检查参数，如果参数是拼接方法生成，推导求得拼接的结果

如何推导，请阅读加拿大 ```Laval University``` 发表的题为 ```Static Analysis of Binary Code to Isolate Malicious Behaviors``` 的[论文](https://pdfs.semanticscholar.org/70cd/4cd765313852369a2100301fde45dc09fbd5.pdf)。如果拼接字符串由服务端下发，依旧可以避开检查。

## 网易方案

### 构建私有 API 库

从 ```Github``` 下载 [iOS-private-api-checker](https://github.com/NetEaseGame/iOS-private-api-checker) 后，可使用 WEB 的方式上传一个 ```IPA``` 进行扫描。我们可以使用 virtualenv 创建一个虚拟环境，来安装所需的依赖库，以免影响系统级的 Python 环境。

```
# 创建虚拟环境
virtualenv venv

# 启用虚拟环境
. venv/bin/activate

# 安装依赖的库
pip install -r requirements.txt

# 启动监测服务
python run_web.py
```

工程的 ```app/templates/main/index_page.html``` 里介绍了检查的原理：

1. 通过 ```class-dump``` 导出 ```Frameworks``` 及 ```PrivateFrameworks``` 的头文件，分别设置为集合 PU 和 PR
2. 通过 Xcode 代码提示的 SQLite 数据库查询出所有的 documented API，设置为集合 DA
3. 那么 PU - DA 为公有 Framework 中的私有 API，设置为 A
4. PR 为私有 Framework 中的 API，都不能使用。则私有 API 集合 PRAPI = A + PR
5. 使用 ```class-dump``` 反编译 ipa 中的 APP 文件，然后和 PRAPI 集合取交集即可获得

但是，项目根目录下的 ```README.md``` 写道：

> 私有的api ＝ (class-dump Framework下的库生成的头文件中的api - (Framework下的头文件里的api = 有文档的api + 没有文档的api)) + PrivateFramework下的api

我第一眼看到这个公式，对其中每一个运算项的含义不是非常肯定，对括号里写上等于号也是有疑问的。另外，这个公式里还提到了 Framework 下的头文件里的 API，而在 index_page.html 中完全没有提到。所以，**建议先无视这个公式**，对 index_page.html 里的文字也不要纠结。

阅读 ```build_api_db.py``` 时，看到方法 ```rebuild_private_api``` 中的注释里写道：

> set_E private api  
> undocument_api = set_B - set_C  
> set_E = set_A - set_C - undocument_api = set_A - set_B  
> if include_private_framework: set_E = set_E + set_D

单从集合运算的角度看 set_E = set_A - set_C - undocument_api 和 set_A - set_B 能不能划等号？讲道理，应该是 **set_E = set_A - (set_B + set_C)** 吧。这里的 ```+``` 是套用原作者的简化写法，指集合的 ```∪``` 运算。所以，**建议无视这个注释**。

注释表述的虽有问题，但通读代码发现实际实现的逻辑是没有问题的。现根据 ```build_api_db.py``` 及相关的代码所对应的构建私有 API 库的原理做一简要阐述：

1. set_A，表示从系统 Frameworks 目录下所有的 ```.framework``` 文件 dump 出的头文件解析出的 API 集合。对应 ```ios_private.db``` 中的 ```framework_dump_apis``` 表记录。
2. set_B，表示从系统 Frameworks 目录下所有的 ```.framework``` 文件中的头文件解析出的 API 集合。对应 ```ios_private.db``` 中的 ```framework_header_apis``` 表记录。 
3. set_C，表示从 docSet 中索引文件解析出来的 API 集合。对应 ```ios_private.db``` 中的 ```document_apis``` 表记录。 
4. set_D，表示从系统 PrivateFrameworks 目录下所有的 ```.framework``` 文件 dump 出的头文件解析出的 API 集合。对应 ```ios_private.db``` 中的 ```private_framework_dump_apis``` 表记录。
5. set_E，表示私有 API，从 set_A 中识别出的私有 API 对应 ```framework_private_apis``` 中的记录，表 ```private_apis``` 中的是加上 set_D 的记录。
6. 如果 ```rebuild_sdk_private_api``` 函数的第二个参数是 ```False``` 则 set_D 不会被加入到 ```private_apis``` 表中。

#### 构建集合 A

在 ```api_utils.py``` 中已经封装好了使用 ```class-dump``` 导出 ```.framework``` 的头文件。所以不需要 ```DumpFrameworks.pl``` 这类的外部脚本，而且 ```DumpFrameworks.pl``` 生成的头文件目录结构和本项目不吻合。也不需要下载 [Nicolas Seiot](http://seriot.ch) 基于 [RuntimeBrowser](https://github.com/nst/RuntimeBrowser) 导出的[头文件](https://github.com/nst/iOS-Runtime-Headers)。

我们需要做的是，保证目标系统 (比如 8.1) 的模拟器在本机已经安装，并且知道 Frameworks 及 PrivateFrameworks 的路径。

```
/Library/Developer/CoreSimulator/Profiles/Runtimes/iOS 8.1.simruntime/Contents/Resources/RuntimeRoot/System/Library/Frameworks
```

需要注意的是，上述路径 iOS 和 8.1 之间存在一个空格。这个空格会引起执行 class-dump 的脚本出问题，具体如何修复后面会给出建议。

根据我的实验结果，将上述路径的 8.1 改成 9.3 或者 10.3 即为不同系统下的路径。iOS 11.4 的路径是：

```
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/Frameworks
```

我们不需要记住这些路径，需要的是掌握获取路径的方法，用 find 命令也是 OK 的。

#### 构建集合 B

Frameworks 路径已经在构建集合 A 的部分介绍过，```api_utils.py``` 中 ```framework_header_apis``` 方法就是用于构建 Frameworks 目录下所有的 ```.framework``` 文件中的头文件解析出的 API 集合。看出和集合 A 的区别了吧？一个是直接处理 ```.framework``` 中包含的头文件，一个是从 ```.framework``` 中的 Mach-O 文件导出对应的头文件。

构建集合 A/D 其实就比构建集合 B 多一步，即 dump 的过程。这也是为何在 dump 时，导出头文件的目录和系统 framework 文件内部结构一致，这样使得接下来的构建集合过程的代码可以通用。

#### 构建集合 C

生成 documented API 集合的主要障碍在于，本机缺乏 docSet。本文写于 2018 年 9 月初，我的工作机上只有 Xcode 9，而新版本的 Xcode 已经使用新的文档格式并直接集成在 Xcode 中。其实苹果官方提供了一个包含各版本文档链接等信息的 [XML](https://developer.apple.com/library/downloads/docset-index.dvtdownloadableindex)，将该 XML 下载到本地即可从中找到 iOS 8.1 等的文档下载链接。

```sh
# 各版本 iOS docSet 的元信息
https://developer.apple.com/library/downloads/docset-index.dvtdownloadableindex

# iOS 8.1 docSet
https://devimages-cdn.apple.com/docsets/20141020/031-07735-A.dmg

# iOS 9.3.5 docSet
https://devimages-cdn.apple.com/docsets/20160321/031-52212-A.dmg
```

安装下载下来的 dmg 后，在 Mac OS 根目录下便出现 docSet 文件了，你可以随意挪位置。docSet 内部的 Contents/Resources/docSet.dsidx 是我们获得集合 C 的数据源。

本人习惯使用 SQLPro for SQLite 工具查看 sqlite 数据库文件，将 docSet.dsidx 重命名为 docSet.sqlite 即可双击打开。其中 ```ZTOKENTYPE``` 表中的 **func**，**instm**，**clm**，**intfm**，**intfcm** 五种类型是我们要关注的：

1. func 表示全局 C 函数
2. instm 表示实例方法 instance method
3. clm 表示类方法 class method
4. intfm 表示协议方法，- 开头
5. intfcm 表示协议方法，+ 开头

> 凭感觉猜测 intf 是 interface 的缩写，interface 即 OOP 的接口而不是 Obj-C 定义类的那个 interface

至于最新版 iOS 的 documented API 怎么获得，本人没有研究。既然 Dash 的作者能生成 Apple API Reference 那理论上讲应该是可以生成 dsidx 文件的。记录有些许价值的 Dash Release Notes 作为日后研究的线索：

"Xcode 8 doesn’t come with docsets anymore and that means Dash won’t automatically support the iOS 10, macOS 10.12, watchOS 3 and tvOS 10 docs. I’m working on a version of Dash that supports the new docs and will release an update as soon as possible." -- [Jun 14th, 2016](https://blog.kapeli.com/dash-xcode-8-and-macos-sierra)

"Apple API Reference Support. Apple has new API docs. You can use them in Dash by installing the Apple API Reference docset." -- [Jul 2nd, 2016](https://blog.kapeli.com/dash-3-dot-3)

"**The Apple API Reference docset now reads the docs from within Xcode 8**. This reduces disk space usage while also allowing me to modify & improve the docs at display-time. Thanks a lot to the Xcode team at Apple for helping me understand the new documentation format!" -- [Oct 25th, 2016](https://blog.kapeli.com/dash-3-dot-4)

#### 构建集合 D

同构建集合 A，路径的 Frameworks 改成 PrivateFrameworks 即可：

```
/Library/Developer/CoreSimulator/Profiles/Runtimes/iOS 8.1.simruntime/Contents/Resources/RuntimeRoot/System/Library/PrivateFrameworks/
```

#### 构建集合 E

以 set_A 为处理对象：

1. 所有以 ```_``` 开头的方法，全部加到 set_E 中；
2. 其他 API，如果不在 set_B 也不在 set_C 中，则加到 set_E 中
3. 在不在 set_B / set_C 中的比较基准是 api_name，class_name，sdk 三个值
4. 步骤 3 是基于 db 查询来实现的

#### 代码缺陷

1. build_api_db.py 中  rebuild_sdk_private_api(sdk_version, False)，需改成 True

2. build_api_db.py 中  ```if include_private_framework``` 之后应该是把 private_framework_apis 插入到数据表中，而不是 ```framework_dump_private_apis``` 

3. api/api_utils.py 中 ```all_headers_path += iterate_dir(framework, "", os.path.join(framework_folder, header_path))``` 应该改成 ```all_headers_path += iterate_dir(framework, "", header_path)``` 

4. db/dsidx_dbs.py 中 sql = balabala 需要确认 dsidx 文件中五种 TOKEN 类型对应的 ID

   比如我从 Apple 下载下来的 8.1 docSet 对应相同 ```ZTOKENTYPE``` 的 ID 不是 (**3,9,12,13,16**) 而是 (**11,13,1,8,19**)。

   如果你是从百度网盘等地方直接下载别人的 ios_private.db，请打开这个 db 检查下 document_apis 表中的数据真的都是 API 么。

   另，原作者写 (3,9,12,13,16)  是因为当时 iOS 7.0 docSet 里确实是这几个 ID，这一点我通过往前翻 [commit](https://github.com/NetEaseGame/iOS-private-api-checker/commit/c2b14cc4788ad8e8a8bd2ef1ffac5877251204b2) 记录得到了确认。所以灵活一点的写法是根据 ```ZTYPENAME``` 筛选数据。

5. dump/class_dump_utils.py 中 ```ret = subprocess.call(cmd.split())``` 健壮性不够

   我在 Xcode 9 安装 iOS 8.1 模拟器后看到 Frameworks 路径是带空格的，经过 split 就会导致路径被拆分成两段。改成 ret = subprocess.call([class_dump_path, '-H', frame_path, '-o', out_path]) 应该就可以规避该问题。

### 扫描私有 API

#### 主要逻辑

阅读 ```iOS_private.py``` 梳理出识别 APP 中私有 API 的主要逻辑如下：

1. 基于 ```strings``` 工具从 Mach-O 文件导出字符串，按空格拆解得到集合1
2. 使用 ```otool -L``` 从 Mach-O 文件获得用到的 Frameworks 及 PrivateFrameworks 列表
3. 基于 ```class-dump``` 从 Mach-O 文件导出的头文件信息，解析出类名变量名集合2、方法集合3
4. 集合4 = 集合1 - 集合2（比较基准是 api_name）
5. 表 ```framework_private_apis``` 中按 api_name, class_name 分组得到类名方法名组合的集合5
6. 对集合5 和集合4 按 api_name 匹配，得到集中6
7. 集合6 和集合 3 按 api_name, class_name 匹配，得到最终的私有 API 集合

步骤2 的结果可以作为步骤5 的部分条件。白名单表 ```whitelist``` 里的数据，会从结果集中排除，对应到代码逻辑上也是在步骤5 被过滤掉。

#### 代码缺陷

因上述步骤6 中是按 api_name, class_name 的组合做匹配条件的，故原始代码的 SQL 语句中 group by 不仅要有 api_name 还应该加上 class_name 这个字端。

### 改进建议

直接使用网易方案大概率是发现不了私有 API 的。检测逻辑只考虑了 api_name, class_name 全匹配，局限性太大。

1. 在私有 API 数据库的建设上，TSRC 实验室的做法是进一步增加条件，比如一些纯小写字母的 API，大多是一些 C 函数，再过滤掉一批

2. 在扫描算法的设计上，如果步骤5 只 group by api_name，步骤6 只匹配 api_name，同时在源代码中存在 ```@selector(XXX)``` 这样的字符串，基本可以认定该 api_name 为私有 API

3. 对于静态拼接或者加解密的 API，可以通过动态 hook 的方式进行识别，但也存在一些局限性
4. 加入 ```prefs:``` 及 ```App-Prefs:``` 协议的扫描

## 验证特定 API

苹果审核提出使用了不该用的某某 API，那么我们势必要支持筛查该 API 用在何处，是我们的 APP 还是第三方 SDK 中。在代码工程根目录，执行：

```
find . -type f | grep ".a" | grep -v ".app" | xargs grep advertisingIdentifier
```

## 遗留主题

在研读网易游戏的开源方案时，对于 iOS 10+ 如何构建 documented API 数据集这个问题直接跳了过去，后续可进一步调研。

## 参考资料

\[ 1 ] Arming Lee. 腾讯 Qzone 工程师. [iOS私有API扫描工作总结](https://github.com/mrmign/iOS-private-api-scanner/blob/master/iOS-api-scan.md). 2014~2015  
\[ 2 ] 刘笑江. 腾讯 WeRead 工程师. [iOS 私有 API 调用检测机制探讨](http://davidlau.me/2017/08/23/ios-private-api-detection/). 2017.08.23  
\[ 3 ] 郑文明. [iOS状态栏操作](https://blog.csdn.net/wenmingzheng/article/details/50475671). 2016.01.07  
\[ 4 ] KFAaron. [导出系统库的头文件](https://www.jianshu.com/p/6484ac07c513). 2016.05.18  
[ 5 ] Friedrich Markgraf. [LegacyDocsets](https://gist.github.com/fzwo/01d62cebd21032683d87f51d094575d3). 2017.05.11  
\[ 6 ] 林桠泉. 腾讯安全应急响应中心. [浅谈 iOS 应用安全自动化审计](https://security.tencent.com/index.php/blog/msg/105). 2016.06.23  
\[ 7 ] Rumin Shah. [How do I check where my app is using IDFA](https://stackoverflow.com/a/31779425). 2015.08.03  
\[ 8 ] Zuik. [使用私有 API 跳转到设置界面](https://zuikyo.github.io/2016/10/10/私有API-iOS10%20openURL方法跳转到设置界面失效的解决方法/). 2016.10.10
