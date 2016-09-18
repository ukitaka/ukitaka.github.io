---
layout: post
title: SDWebImageでダウンロードした画像のファイルパスを取得して画像を扱う
---

## 手順

まずはやり方を記述します。

### `SDWebImageManager`でダウンロードする

```objective_c
[[SDWebImageManager sharedManager] downloadImageWithURL:url
                                                options:SDWebImageRetryFailed
                                               progress:nil
                                              completed:^(UIImage *image, NSError *_error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
                                              	//完了処理. 以下を参照
                                              }];
```

### URLからキャッシュキーを取得して、`diskImageExistsWithKey:completion:`で確認する.

```objective_c
NSString *cacheKey = [[SDWebImageManager sharedManager] cacheKeyForURL:imageURL];
NSString *cachePath = [[SDImageCache sharedImageCache] defaultCachePathForKey:cacheKey];
[[SDImageCache sharedImageCache] diskImageExistsWithKey:cacheKey completion:^(BOOL isInCache) {
    if (isInCache) {
		// ここでようやく使えるようになる
    }
}];
```

## ポイント

`diskImageExistsWithKey:`でなくて`diskImageExistsWithKey:completion:`であることがポイントで、`completed`のタイミングではファイルの書き込みが完了しているとは限りません。ダウンロード完了後、ディスクへの書き込み処理は`SDImageCache`の`ioQueue`にdispatchされているだけです。

```objective_c
// ダウンロード後の書き込み処理
[self.imageCache storeImage:downloadedImage recalculateFromImage:NO imageData:data forKey:key toDisk:cacheOnDisk];
```


```objective_c
// storeImage:recalculateFromImage:imageData:forKey:toDisk:のおおまかな流れ
if (toDisk) {
    dispatch_async(self.ioQueue, ^{
	    ...
		[_fileManager createFileAtPath:[self defaultCachePathForKey:key] contents:data attributes:nil];
	})
}
```


`diskImageExistsWithKey:completion:`は存在確認処理を`ioQueue`にdispatchします。



```objective_c
- (BOOL)diskImageExistsWithKey:(NSString *)key {
    BOOL exists = NO;

    // this is an exception to access the filemanager on another queue than ioQueue, but we are using the shared instance
    // from apple docs on NSFileManager: The methods of the shared NSFileManager object can be called from multiple threads safely.
    exists = [[NSFileManager defaultManager] fileExistsAtPath:[self defaultCachePathForKey:key]];

    return exists;
}

- (void)diskImageExistsWithKey:(NSString *)key completion:(SDWebImageCheckCacheCompletionBlock)completionBlock {
    dispatch_async(_ioQueue, ^{
        BOOL exists = [_fileManager fileExistsAtPath:[self defaultCachePathForKey:key]];
        if (completionBlock) {
            dispatch_async(dispatch_get_main_queue(), ^{
                completionBlock(exists);
            });
        }
    });
}
```

`ioQueue`はシリアルキューなので存在確認処理は必ず書き込み完了後になります。

```objective_c
_ioQueue = dispatch_queue_create("com.hackemist.SDWebImageCache", DISPATCH_QUEUE_SERIAL);
```
