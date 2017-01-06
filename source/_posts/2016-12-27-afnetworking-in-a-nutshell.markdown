---
layout: post
title: AFNetworking 3.0 源码阅读笔记
date: 2016-12-27 09:36:53 +0800
comments: true
categories: iOS AFNetworking
---
上周阅读了 AFNetworking 3.0 的源代码，对阅读中遇到的问题进行了些思考，整理成此文。

<!--more-->

## AFURLSessionManager (核心类)

#### 1. AFHTTPSessionManager 和 AFURLSessionManager 的关系？
前者是后者的子类，封装了和 HTTP Method 相关的接口。核心代码均在后者的实现中，包括 Session 的创建、Session 回调方法的实现、Task 和 Task 委托对象 (AFURLSessionManagerTaskDelegate) 的映射关系等。

值得关注的是 request serializer 是定义在 AFHTTPSessionManager 中的，而 response serializer 是定义在 AFURLSessionManager 中的。

#### 2. 创建 Session 时指定的 delegate queue 为何设置  maxConcurrentOperationCount 为 1？
delegate queue 是 session 的回调方法执行时所在的 queue，并不影响 HTTP 请求的并发执行 (由 Session Configuration 的 HTTPMaximumConnectionsPerHost 控制)。查看 Session 的几个 Delegate 方法，我们可以看到，会涉及到对 task 的 delegate 处理（get / remove 等），而 task 和 task 的 delegate 是由 mutable 的字典管理的，处理时需要用到 lock 以支持多线程安全。所以，这个 delegate queue 本质上还是得一个任务一个任务的处理。

前面讨论了为何设置 maxConcurrentOperationCount 为 1，那这样是否影响 APP 的网络层处理效率？其实不会的。在 - (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error 的实现里我们可以看到，对 response 的处理 (使用 response serializer) 是放在一个 concurrent queue (见 url_session_manager_processing_queue 方法) 里的。

#### 3. 为何设计 completion group 和 completion queue 这两个属性？
纵观 AFN 的内部，我们只看到 AFURLSessionManagerTaskDelegate 的 - (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(nullable NSError *)error 方法中把 completion handler 的执行放在 completion group / queue 里，而且如果没有专门指定的话就用私有的默认 group 及 main queue。所以通常应用层的 success / failure block 在主线程调用。

当我们需要在多个 request 均处理完毕这个时间点，执行一些自定义的操作，那就可以给 session manager 指定专门的 completion group / queue 来实现。

#### 4. 和 AFURLSessionManagerTaskDelegate 中的 URL Session 相关的几个同名方法什么关系？
Session 的 delegate 是 URL Session Manager，所以 Session 相关的回调都是到 URL Session Manager 的相关方法里。Task Delegate 里的同名方法是由 Manager 里的回调再调用的，可以搜索 delegate URLSession: 查阅。

#### 5. 实现文件里还定义了辅助类 AFURLSessionManagerTaskDelegate 及 _AFURLSessionTaskSwizzling 各自的作用是什么？
前者保存了 task 的进度 (upload、download、completion) 相关的处理块。其进度信息是根据 session 委托方法传递过来的数据，计算后保存到 NSProgress 对象里的。

后者是为规避 NSURLSession Task 跟 KVO 相关的一个 bug，将 resume / suspend 方法改写了下。因为 class cluster 的原因，作者创建了一个 NSURLSessionDataTask 对象，使用 while 对其 super 一层层的查找。

#### 6. _AFURLSessionTaskSwizzling 中的 state 方法有何作用？为何直接 assert(NO, @"...")？
此处的 state 方法纯粹是为了使用 @selector(state) 以及 [self state] 时能顺利通过编译器的检查。af_resume 中调用的 [self state] 其实是 NSURLSessionDataTask 的 state 方法，_AFURLSessionTaskSwizzling 的 state 方法不会被调用，也不应该调用，所以作者使用了 assert(NO, @"...")。

#### 7. Session Manager 为何实现 Coding 协议？
我估计是为了支持 background session，同样值得关注的是 AFURLSessionManager 的初始化方法里调用 session 的 getTasksWithCompletionHandler 方法处理一遍 task delegate（乍一看也是不明所以，但如果是支持 background session，那就有那么点说的通了）。

#### 8. 创建 task 时为何使用  url_session_manager_create_task_safely 方法再包装一层？
iOS 8 修复了一个 bug，多线程下创建的 task 小概率拥有同样的 task identifier。但是 AFNetworking 使用 task identifier 和 task delegate 映射，需要 task identifier 唯一。

所以对于早前有这个 bug 的 iOS 版本，使用 dispatch_sync 来同步创建 task 以规避问题。

#### 9. Session Manager 是否有内存泄露的问题？
之所以提出这样的问题，是因为 Session Manager 拥有 Session，而 Session 对其 delegate 是强引用，而 Session Manager 就是这个 delegate，于是便 retain cycle 了。

查看苹果的文档得知『If you do not invalidate the session by calling the invalidateAndCancel or finishTasksAndInvalidate method, your app leaks memory until it exits.』

这也是为何 AFN 提供 - (void)invalidateSessionCancelingTasks: 接口。

## AFHTTPSessionManager

#### 10. 应用层的 success / failure block 是否需要预防 block 的 retain cycle 问题？
和 AFNetworking 2.0 里的 URL Connection 方案一致，不需要。以前是在 AFURLConnectionOperation 里通过  [strongSelf setCompletionBlock:nil] 释放对 completion block 的持有。

现在是在 Session Manager 里通过 [self removeDelegateForTask:task] 移除 task delegate 以释放对 completion handler 的持有。

#### 11. POST 相关接口的差异？
仅仅 POST 一个普通的字典到服务端使用 data task，若需要 POST 一个文件 (URL / Data) 到服务端则需要使用 multipart form request 及 upload task 实现。后者字典里的普通数据，也是放在请求的 Content-Disposition: form-data; name= XXX 部分，constructing body block 里若 append 文件 (比如图片) 的话是放在 Content-Disposition: form-data; name="XXX"; filename="XXX.png" 部分 (下一行会指定类型，如 Content-Type: image/png)。

#### 12. POST 多张图片的 request 如何构造其 body 的？
body 的构造依赖于 AFStreamingMultipartFormData 对象，待上传的单个图片会被封装到 AFHTTPBodyPart 里，然后 append 到 AFStreamingMultipartFormData → AFMultipartBodyStream → HTTPBodyParts 中。注意 AFMultipartBodyStream 继承自 NSInputStream，可直接设置到 request 中 (见 AFStreamingMultipartFormData → requestByFinalizingMultipartFormData)。

AFMultipartBodyStream 重写了- (NSInteger)read:(uint8_t *)buffer maxLength:(NSUInteger)length 方法，将图片按切片的方式读到 buffer 中。每一个 AFHTTPBodyPart 的构成大致包含 Boundary、Header、Body 等几个子部分，AFHTTPBodyPart 的 read 方法会按这些部分（如果存在）逐个读取出来。

## AFHTTPRequestSerializer

#### 13. AFHTTPRequestSerializer 和 AFURLRequestSerialization 之间是什么关系？
后者是协议，定义了一个将字典数据序列化到 request 中以得到新的 request 这么一个方法。AFHTTPRequestSerializer 实现了这个协议，使用  AFQueryStringFromParameters 方法将字典处理成 key1=value1&key2=value2 这样的形式。

因为字典里可能嵌套包含数组或字典，AFQueryStringPairsFromKeyAndValue 方法使用了递归的思想。虽然这段代码是可以把多层嵌套转成扁平的 KV，我们通常不会嵌套字典，否则服务端无法解析。

除了协议约束的方法之外，AFHTTPRequestSerializer 还实现了对 multipart request 的序列化方法，实现原理见 AFHTTPSessionManager 的第 3 点。

## AFJSONResponseSerializer

#### 14. AFJSONResponseSerializer 和 AFURLResponseSerialization 之间是什么关系？
后者是协议，定义了一个从 response 里解析出 response object 的方法。继承自 AFHTTPResponseSerializer 的 AFJSONResponseSerializer，使用 NSJSONSerialization 完成对 JSON 数据的解析。

#### 15. AFJSONResponseSerializer 有一个属性叫 removesKeysWithNullValues，有什么用？
在服务器返回的 json 数据若出现 "somevalue":null，会被解析成 NSNull 的对象。我们向这个 NSNull 对象发送消息的时候就会 crash。AFJSONResponseSerializer 提供的这个属性可以控制是否将此类数据过滤掉。

## AFSecurityPolicy

#### 16. 定义的三种 SSL Pinning Mode 分别有什么作用？
AFSSLPinningModeNone：默认值，完全信任服务器，相当于仅支持 HTTP 请求
AFSSLPinningModeCertificate：会比对服务端传过来的证书和 APP 内的证书
AFSSLPinningModePublicKey：仅比较服务端传过来的证书携带的公钥和本地证书里的公钥是否一致 

好房 APP 使用的是 AFNetworking 2.x，且将 validatesCertificateChain 置为了 NO，所以好房 APP 并不会严格的比对服务端传过来的证书和 APP 内的证书的内容。之所以这么设置，是因为服务端传来的是个证书链，本地证书只会和其中的一个证书 match，但 AFNetworking 2.x 的实现里要求和服务端证书链里所有的证书都匹配，APP 里不可能放入服务端可能用到的所有证书，所以只好选择不验证证书链。

AFNetworking 3.0 对 AFSSLPinningModeCertificate 的实现有所调整，只要有一个 match 就认为 OK，所以也不再提供 validatesCertificateChain 这个属性了。

#### 17. 如何选择适合的 Pinning Mode 呢？
AFSSLPinningModeCertificate 比较安全，他会比对服务端和客户端的证书是否匹配。随 APP 打包的证书是带有效期的。如果只是比较 Public Key，则不会受有效期的影响。个人觉得使用 Public Key 的方案基本足够了。

## AFNetworkReachabilityManager

#### 18. 略，主要看看对 IPV 6 的支持。