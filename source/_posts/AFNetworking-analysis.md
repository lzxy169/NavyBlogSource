---
title: AFNetworking 解析
date: 2018-06-02 13:02:41
tags:
categories: Objective-C
---


### NSURLSession
NSURLSession 使用 NSURLRequest 来包含请求信息，来生成各种类型的 Task，使用 NSURLSessionConfiguration 来生成配置信息
调用 - (void)resume 来发送请求， 生成 NSURLResponse 来返回请求的信息。回调的方式有delegate，和block两种形式。

```
@interface NSURLSessionDataTask : NSURLSessionTask
@interface NSURLSessionUploadTask : NSURLSessionDataTask
@interface NSURLSessionDownloadTask : NSURLSessionTask
@interface NSURLSessionStreamTask : NSURLSessionTask

@protocol NSURLSessionDelegate <NSObject>
@protocol NSURLSessionTaskDelegate <NSURLSessionDelegate>
@protocol NSURLSessionDataDelegate <NSURLSessionTaskDelegate>
@protocol NSURLSessionDownloadDelegate <NSURLSessionTaskDelegate>
@protocol NSURLSessionStreamDelegate <NSURLSessionTaskDelegate>
```

### AFNetworking
AFNetworking是封装的NSURLSession的网络请求。

**AFNetworking核心由五个模块组成：**
SessionManager ：主要对象 NSURLSession 对象进行了进一步的封装
Request/Response Serialization，提供了与请求数据和解析数据相关的操作接口
SecurityPolicy，提供了与安全性相关的操作接口，主要是证书验证
NetworkReachabilityManager，提供了与网络状态监听相关的操作接口
以及的UIKit的一些类扩展，提供了大量网络请求过程中与UI界面显示相关的操作接口，通常用于网络请求过程中提示，使用户交互更加友好

**SessionManager：**
```
@interface AFURLSessionManager : NSObject <NSURLSessionDelegate, NSURLSessionTaskDelegate, NSURLSessionDataDelegate, NSURLSessionDownloadDelegate, NSSecureCoding, NSCopying>

@interface AFHTTPSessionManager : AFURLSessionManager <NSSecureCoding, NSCopying>
```

**HTTPMethod：**
```
@"GET"，@"HEAD"，@"POST"，@"PUT"，@"PATCH"，@"DELETE"
```

**Request：**
```
@protocol AFURLRequestSerialization <NSObject, NSSecureCoding, NSCopying>
@interface AFHTTPRequestSerializer : NSObject <AFURLRequestSerialization>
@interface AFJSONRequestSerializer : AFHTTPRequestSerializer
@interface AFPropertyListRequestSerializer : AFHTTPRequestSerializer
```

**Response：**
@protocol AFURLResponseSerialization <NSObject, NSSecureCoding, NSCopying>
@interface AFHTTPResponseSerializer : NSObject <AFURLResponseSerialization>
@interface AFJSONResponseSerializer : AFHTTPResponseSerializer
@interface AFXMLParserResponseSerializer : AFHTTPResponseSerializer
@interface AFPropertyListResponseSerializer : AFHTTPResponseSerializer
@interface AFImageResponseSerializer : AFHTTPResponseSerializer
@interface AFCompoundResponseSerializer : AFHTTPResponseSerializer

### YTKNetwork
YTKNetwork 由 Request， RequestAgent，NetworkConfig组成，对 AFNetworking 的封装

```
YTKRequestMethod：
YTKRequestMethodGET,
YTKRequestMethodPOST,
YTKRequestMethodHEAD,
YTKRequestMethodPUT,
YTKRequestMethodDELETE,
YTKRequestMethodPATCH,

Request：
@protocol YTKRequestDelegate <NSObject>
@interface YTKBaseRequest : NSObject

@protocol YTKBatchRequestDelegate <NSObject>
@interface YTKBatchRequest : NSObject

@protocol YTKChainRequestDelegate <NSObject>
@interface YTKChainRequest : NSObject
```

