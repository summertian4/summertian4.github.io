title: iOS——iOS8 PhotoKit 库 PHAsset之坑
date: 2016-03-05 14:06:28
tags:
  - iOS
categories:
  - iOS
---

# 问题重现


最近在做项目的时候，我们选择了一个可以下载 iClould 图片的三方 Picker——CTAssetsPickerController，这个 Picker 会根据不同系统版本，返回不同类型的图片资源。

iOS8 之前，访问系统照片视频使用的是 AssetsLibrary 框架，iOS8之后有了新的系统框架 PhotoKit，这一框架对 iCloud 选图有很好的支持。

AssetsLibrary 的图片资源类型是 ALAsset，而 PhotoKit 的图片资源类型是 PHAsset。

如果通过 PHAsset 获取图片资源，可以调用以下方法：


```objc
[[PHImageManager defaultManager] requestImageForAsset:asset targetSize:CGSizeMake(pixelWidth, pixelHeight) contentMode:PHImageContentModeAspectFit options:nil resultHandler:^(UIImage * _Nullable result, NSDictionary * _Nullable info) {
	// 回调部分
            
}];  
```

<!--more-->

但是这个方法有一个潜在的坑——`回调部分会被调两次`。

这里是该方法的官方注释：

```objc
// If the asset's aspect ratio does not match that of the given targetSize, contentMode determines how the image will be resized.
//      PHImageContentModeAspectFit: Fit the asked size by maintaining the aspect ratio, the delivered image may not necessarily be the asked targetSize (see PHImageRequestOptionsDeliveryMode and PHImageRequestOptionsResizeMode)
//      PHImageContentModeAspectFill: Fill the asked size, some portion of the content may be clipped, the delivered image may not necessarily be the asked targetSize (see PHImageRequestOptionsDeliveryMode && PHImageRequestOptionsResizeMode)
//      PHImageContentModeDefault: Use PHImageContentModeDefault when size is PHImageManagerMaximumSize (though no scaling/cropping will be done on the result)
// If -[PHImageRequestOptions isSynchronous] returns NO (or options is nil), resultHandler may be called 1 or more times.
//     Typically in this case, resultHandler will be called asynchronously on the main thread with the requested results.
//     However, if deliveryMode = PHImageRequestOptionsDeliveryModeOpportunistic, resultHandler may be called synchronously on the calling thread if any image data is immediately available. If the image data returned in this first pass is of insufficient quality, resultHandler will be called again, asychronously on the main thread at a later time with the "correct" results.
//     If the request is cancelled, resultHandler may not be called at all.
// If -[PHImageRequestOptions isSynchronous] returns YES, resultHandler will be called exactly once, synchronously and on the calling thread. Synchronous requests cannot be cancelled. 
// resultHandler for asynchronous requests, always called on main thread

- (PHImageRequestID)requestImageForAsset:(PHAsset *)asset targetSize:(CGSize)targetSize contentMode:(PHImageContentMode)contentMode options:(nullable PHImageRequestOptions *)options resultHandler:(void (^)(UIImage *__nullable result, NSDictionary *__nullable info))resultHandler;
```

大意是：如果 asset 对应的资源不符合给定的尺寸，将由 contentMode 决定返回的图片如何压缩。
后面说到了回调可能被调用的次数，如果没有仔细看，你根本就搞不明白是怎么一回事。

stackoverflow上有人一语中的([原链接](http://stackoverflow.com/questions/26663258/uiimage-size-returned-from-requestimageforasset-is-not-even-close-to-the-targ))：

```objc
This is because requestImageForAsset will be called twice.

The first time, it will return a very same size image, such as (60 * 45) which I think is the thumbnail of that image.

The second time, you will get the full size image.

I use

if ([[info valueForKey:@"PHImageResultIsDegradedKey"]integerValue]==0){
    // Do something with the FULL SIZED image
} else {
    // Do something with the regraded image
}

to distinguish the two different images.
```

意思是因为该回调会被调用两次，第一次返回你指定尺寸的图片，第二次将会返回原尺寸图片
如果你想区分，可以这样使用：

```objc
if ([[info valueForKey:@"PHImageResultIsDegradedKey"]integerValue]==0){
    // Do something with the FULL SIZED image
} else {
    // Do something with the regraded image
}
```

## 问题为什么是问题

如果你不去注意这个地方，就可能和我一样去踩坑。
为什么这样说？在项目中，我们选中了一批图片，当我需要上传图片到我们的服务器时，需要将 PHAsset 换成 UIImage。
当调用`- (PHImageRequestID)requestImageForAsset:(PHAsset *)asset targetSize:(CGSize)targetSize contentMode:(PHImageContentMode)contentMode options:(nullable PHImageRequestOptions *)options resultHandler:(void (^)(UIImage *__nullable result, NSDictionary *__nullable info))resultHandler;`并在回调中写入上传图片至服务器的代码。这时候，如果回调被调用了两次，就会导致同一张图片被上传两次。如果做了同图判断，就会不断的报出一张图片上传不成功，或者少传一张的情况。

## 其他
关于 AssetsLibrary 框架的坑，可以看[这篇文章](http://kayosite.com/ios-development-and-detail-of-photo-framework.html/comment-page-1)

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


