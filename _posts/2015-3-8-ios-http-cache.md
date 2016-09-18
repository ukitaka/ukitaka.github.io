---
layout: post
title: iOSのHTTP通信のキャッシュの話(NSURLRequest, AFNetworking, SDWebImage)
---

[前回の記事](http://blog.waft.me/web-api-2/)でHTTPキャッシュ周りの整理をしたので、これらとiOSのHTTP通信周りの関わりを見ていきます。


## NSURLRequest


Appleのドキュメントの[Understanding Cache Access](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/URLLoadingSystem/Concepts/CachePolicies.html)を読むとこの辺りの挙動が詳しく書いてあります。


まず、`NSURLRequest`は`cachePolicy`というプロパティを持っていてここでキャッシュの制御ができます

| NSURLRequestCachePolicy | 説明 |
| ------------------------|-----|
| NSURLRequestUseProtocolCachePolicy | デフォ値。プロトコル(http://ならHTTP)のキャッシュ方針にしたがう。 |
| NSURLRequestReloadIgnoringCacheData | 常にキャッシュを無視してoriginating sourceにデータを読みに行く |
| NSURLRequestReturnCacheDataElseLoad | キャッシュがstaleかどうかに関わらずキャッシュを読みに行く。 キャッシュが存在しない場合のみoriginating sourceにデータを読みに行く |
| NSURLRequestReturnCacheDataDontLoad | 常にキャッシュを読みに行く。キャッシュがない場合はnilを返す。 |


HTTP通信で`NSURLRequestUseProtocolCachePolicy`が設定されている場合はHTTPのキャッシュ方針にしたがうわけですが、実際の挙動も[Understanding Cache Access](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/URLLoadingSystem/Concepts/CachePolicies.html)で説明されています。


#### HTTPでの挙動

+  `NSCachedURLResponse`が存在しない場合は、originating sourceにデータを読みに行く
+  `NSCachedURLResponse`が存在する場合は、再検証(=条件つきリクエストを投げる必要があるか)どうかを決めるために`NSCachedURLResponse`を確認する
	+ 具体的にどうやって再検証する/しないを決めてるのかは説明されていないが、おそらく`Cache-Control`ヘッダの`must-revalidate`とかをみていると予想

+ 再検証の必要があればHEADメソッドで条件つきリクエストを投げる
	+ 変更されていれば originating sourceからデータをフェッチする
	+ されていなければ(=304なら)キャッシュを読みに行く

+ 再検証の必要があるかどうか示されていない場合は`max-age`とか`Expires`の値をチェックして、
	+ 期限内ならキャッシュを読みに行く 
	+ staleだったらHEADメソッドで条件つきリクエストを投げる
		+ 変更されていれば originating sourceからデータをフェッチする
		+ されていなければ(=304なら)キャッシュを読みに行く

+ 上記以外ならキャッシュを読みに行く

条件つきGETでなくて、HEADメソッドで条件つきリクエストを投げたあとにGETしに行っているように受け取れます。なんでだろう。。


## AFNetworking

基本的には`NSOperation`のサブクラスである`AFHTTPRequestOperation`, `AFURLConnectionOperation`内で`NSURLRequest`を実行しているだけなので基本は`NSURLRequest`と同じと思っています。

ただ`AFHTTPRequestOperationManager`にはキャッシュポリシーを指定するAPIがなく、デフォ値の`NSURLRequestUseProtocolCachePolicy`が使われているのでHTTPのキャッシュ方針に従っています。

## SDWebImage

SDWebImageは独自のキャッシュ機構を持っているいるので、それと`NSURLRequest`のキャッシュの組み合わせになります。

`SDWebImageManager`の`downloadImageWithURL:options:progress:completed:`で、まずディスクキャッシュを確認しています。

```
operation.cacheOperation = [self.imageCache queryDiskCacheForKey:key done:^(UIImage *image, SDImageCacheType cacheType) {

...

}
```

### 1.ディスクキャッシュがない場合

上記のdoneブロック内で`SDWebImageDownloader`を使ってダウンロードが行われます。
この時とくにoptionは指定されずにダウンロードが開始されるので`NSURLRequestReloadIgnoringLocalCacheData`が利用されるようです。

```
NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url cachePolicy:(options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData) timeoutInterval:timeoutInterval];
```



### 2.`SDWebImageRefreshCached`が指定された場合

`SDWebImageRefreshCached`というオプションが指定されてる場合はディスクキャッシュがあってもHTTPのレスポンスを優先し、ディスクキャッシュを上書きします。

```
 /**
  * Even if the image is cached, respect the HTTP response cache control, and refresh the image from remote location if needed.
 * The disk caching will be handled by NSURLCache instead of SDWebImage leading to slight performance degradation.
 * This option helps deal with images changing behind the same request URL, e.g. Facebook graph api profile pics.
 * If a cached image is refreshed, the completion block is called once with the cached image and again with the final image.
 *
 * Use this flag only if you can't make your URLs static with embeded cache busting parameter.
 */
 SDWebImageRefreshCached = 1 << 4,
```

先ほどのdoneブロック内で`SDWebImageDownloader`クラスに渡すオプションに`SDWebImageDownloaderUseNSURLCache`を追加しています。

```
if (options & SDWebImageRefreshCached) downloaderOptions |= SDWebImageDownloaderUseNSURLCache;
```

このオプションが指定されているとキャッシュポリシーに`NSURLRequestUseProtocolCachePolicy`が指定されるのでHTTPのキャッシュを使うことになります。

```
NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url cachePolicy:(options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData) timeoutInterval:timeoutInterval];
```


### 3.同じリソースだけどURLが変わっている場合

`http://hogehoge.com/img/fuga.jpg?v=12345`のようにキャッシュ制御のためのパラメータが付いている場合がありますが、
SDWebImageは基本的には`absoluteString`(パラメータも含まれます)をキーとしてキャッシュを保存しています。

```
- (NSString *)cacheKeyForURL:(NSURL *)url {
    if (self.cacheKeyFilter) {
        return self.cacheKeyFilter(url);
    }
    else {
        return [url absoluteString];
    }
}
```

パラメータがきちんとバージョン管理のために付けられており、リソースが更新された時のみ変わるものであれば問題ないですが、
開発環境などでキャッシュさせないために現在時刻などをパラメータとしてつけていると

1. 一致するキーのディスクキャッシュがないので`NSURLRequestReloadIgnoringLocalCacheData`の状態でロードする
2. 違うキーでディスクにキャッシュされる

のでちょっといろいろ重いです。(キャッシュさせないようにしているので想定通りの動きにはなってます)


## まとめ

内部のキャッシュの挙動を知っとくと、API設計のときにも安心感があって開発できますね！！！！
