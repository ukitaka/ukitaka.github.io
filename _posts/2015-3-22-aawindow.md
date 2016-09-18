---
layout: post
title: ライブラリメモ 「AAWindow」
---

githubのswift, objcのトレンドに入っているライブラリを読んで気になるところをメモしていこうと思います。


## AAWindow

[https://github.com/aaronabentheuer/AAWindow.git](https://github.com/aaronabentheuer/AAWindow.git)

![画像](https://raw.githubusercontent.com/aaronabentheuer/AAWindow/master/screencast.gif)


## 機能

UIWindowのサブクラスとして作られていて、主に以下の2つの機能があるようです

1. Window自体にcornerRadiusを設定する
2. コントロールセンターを開いたことを検知できる

## Window自体にcornerRadiusを設定する

READMEにも書いてありますが、Instagramが出している[Hyperlapse](https://itunes.apple.com/jp/app/hyperlapse-from-instagram/id740146917?mt=8)など、最近画面全体に角丸が設定してあるアプリは確かにちょくちょく見かけます。

角丸にする事自体は特に難しくなく、こんな感じで

```swift
var window: UIWindow? = {
    let window = UIWindow(frame: UIScreen.mainScreen().bounds)
    window.layer.cornerRadius = 8
    window.clipsToBounds = true
    return window;
}()
```

UIWindowのlayerに`cornerRadius`を設定してあげればできるのですが、
マルチタスク切り替えの画面では角丸に設定していると、角の背景色が目立ってしまって少し残念な見た目になります。

これを防ぐために、AAWindowではバックグラウンドに移行する際に自動で`cornerRadius`を0に設定、フォアグラウンドに戻ってきたら再度`cornerRadius`を設定という動作を、いい感じのアニメーション付きやってくれるようです。


実装も細かい制御をのぞけばシンプルで`UIApplicationWillResignActiveNotification`,`UIApplicationDidBecomeActiveNotification`を拾って再設定しているだけです。

```swift
@objc private func applicationWillResignActive (notification : NSNotification) {
      ...
     self.layer.cornerRadius = inactiveCornerRadius
     self.layer.addAnimation(animateCornerRadius(activeCornerRadius, toValue: inactiveCornerRadius, withDuration: cornerRadiusAnimationDuration, forKey: "cornerRadius"), forKey: "cornerRadius")
}
```

```swift
@objc private func applicationDidBecomeActive (notification : NSNotification) {
	...
     self.layer.cornerRadius = activeCornerRadius
     self.layer.addAnimation(animateCornerRadius(inactiveCornerRadius, toValue: activeCornerRadius, withDuration: cornerRadiusAnimationDuration, forKey: "cornerRadius"), forKey: "cornerRadius")
}
```

## コントロールセンターが開いた事を検知できる

コントロールセンターを開いた場合はバックグラウンドに移行する時と同じく`UIApplicationWillResignActiveNotification`が飛んでくるので、これだけだとコントロールセンターを開いたのかバックグラウンドに移行したのかわからないという問題がありました。(あまりここで実装を分けることもないのかもしれないですが...)


AAWindowではこれを区別できるようになっていて、コントロールセンター用のNotificationが用意されています。

```swift
//This notification will fire when the user opens Control Center.
private var applicationWillResignActiveWithControlCenterNotification = NSNotification(name: "applicationWillResignActiveWithControlCenter", object: nil)

//This notification will fire when the application becomes inactive for whatever reason, except when the user launches Control Center.
private var applicationWillResignActiveWithoutControlCenterNotification = NSNotification(name: "applicationWillResignActiveWithoutControlCenter", object: nil)
```

必要に応じて以下の名前をobserveしておくといいかと思います。

+ `applicationWillResignActiveWithControlCenter`
+ `applicationWillResignActiveWithoutControlCenter`



実際の実装はかなりゴリゴリで、まずタッチ座標を取得して画面の下部10%にあるかどうかを判定して、そうであればフラグをたてておきます。


```swift
if (touch.phase == UITouchPhase.Began && touch.locationInView(self).y - self.frame.height * 0.9 >= 0) {
    //willOpenControlCenter is true for a short period of time when the user touches in the bottom area of the screen. If in this period of time "applicationWillResignActive" is called it's highly likely (basically certain) that the user has launched Control Center.
    willOpenControlCenter = true

    ...
```


その後一定の秒数以内に実際に`UIApplicationWillResignActiveNotification`が飛んでくればコントロールセンターが表示されたと見なす、という実装です。


この**一定の秒数**を決める部分の実装が少し参考になったのですが、ステータスバーが隠れている状態(=フルスクリーン)だと、コントロールセンターは1度目の操作で少しだけ出てきて、2回目の操作で完全に表示されるという挙動をします。

```swift
//If the Statusbar is hidden (which means the app is in full-screen mode) the timerInterval has to be longer since it will take the user a maximum amount of ~3 seconds to open Control Center since he has to use the little handle coming up from the bottom.
var timerInterval : Double = {
	if (UIApplication.sharedApplication().statusBarHidden) {
		return 2.75
	} else {
		return 0.5
	}
}()
```

言われてみれば、ですがステータスバーの状態で動作が決まってたんですね。知らなかった。
なので上記のようにステータスバーがでていれば2.75秒、そうでなければ0.5秒間の間に`UIApplicationWillResignActiveNotification`が飛んでこなければフラグをfalseに戻す、というのをタイマーを使って実装しています。

ちなみに上記角丸の設定もコントロールセンターかどうかで動作を変えています。本当にバックグラウンドに移行した場合のみ角丸をはずすようになっているようです。

## 感想

+ 角丸設定ライブラリを作ろうとして、必要なのでコントロールセンター判定も作った、という流れですかね

+ UIWindow上のすべてのタッチイベントを拾っているので、それなりに重くなりそうな気はします。(きちんと計測はしてないですが..)

+ 音楽や動画などのアプリでは、「コントロールセンターを開くときには再生を止めたくない」のような要望はあると思うのでコントロールセンター表示のイベントを拾えるのはなかなかうれしい
