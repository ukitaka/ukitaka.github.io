---
layout: post
title: Stimulatorというライブラリを作った
---



## Viewの再利用について

Viewには具体的な実装をなるべく持たせたくないので、UIイベントは可能な限りViewControllerまで戻して処理をするようにしたいところです。

そのためのアプローチとして

+ delegateパターンを利用する
+ nibのownerとして@IBAction, @IBOutletをつなぐオブジェクトをインジェクションする
+ Blocks, Closuresを渡せるようにする
+ NSNotificationCenterで通知を投げる

等が考えられますが、上三つはそのViewオブジェクトを直接参照できないと設定できないので結構面倒です。例えば

```
MyViewController
	ContainerViewのViewController
    	ContainerView
    		TableView
    			TableViewCell
    				Button <- ここのイベントをMyViewControllerまで戻したい
```

とかってなるととても面倒です。
(ViewControllerに戻さなきゃいいじゃんとか、思うかもしれないですが、画面遷移やAlertを出すなどViewControllerじゃなきゃできないことが多いです。keyWindow.rootViewControllerを使うとかダメ、絶対。)

また、NotificationCenterは取り扱いが結構面倒で、対1の通知には向いてないです。

## そこでStimulator

[ukitaka/Stimulator](https://github.com/ukitaka/Stimulator)

+ レスポンダチェーンを使ったイベント処理ライブラリ
+ 深い階層にあるViewからViewControllerへの通知をしたいみたいなパターンに向いている
+ delegateやnibのownerとなるオブジェクトを深い階層まで持ち回らなくてもよい
+ UIResponderのサブクラスでしか利用できない
+ 対1の通知を実装するのに向いている(NSNotificationCenterはON/OFFの制御が面倒なので)
+ レスポンダチェーン上の子→親への通知のみ可能
+ 逆や兄弟へは通知できない


## 使い方

イベントの作成

```
struct ShowAlertEvent : Stimulator.Event {

    typealias Responder = ShowAlertResponder

    let title: String
    let message: String

    init(_ title: String, _ message: String) {
        self.title = title
        self.message = message
    }

    func stimulate(responder: Responder) {
        responder.showAlert(self)
    }
}

protocol ShowAlertResponder {

    func showAlert(event: ShowAlertEvent)

}
```

イベントの発火 

```
self.stimulate(ShowAlertEvent("title", "message"))
```

イベントの処理

```
extension MyViewController : ShowAlertResponder {

    func showAlert(event: ShowAlertEvent) {
        let alert = UIAlertController(title: event.title, message: event.message, preferredStyle: UIAlertControllerStyle.Alert)
        alert.addAction(UIAlertAction(title: "OK", style: UIAlertActionStyle.Cancel, handler: { _ in }))
        self.showViewController(alert, sender: nil)
    }

}
```

## 実装

実装はとてもシンプルで、これだけです。

```
public protocol Event {
    typealias Responder
    func stimulate(responder: Responder)
}

public extension UIResponder  {
    
    public func stimulate<E: Event>(event: E) -> E.Responder? {
        if let responder = stimulateResponder(event) {
            event.stimulate(responder)
            return responder
        }
        return nil
    }
    
    public func stimulateResponder<E: Event>(event: E) -> E.Responder? {
        var responder : UIResponder? = self
        while (responder != nil) {
            if let responder = responder as? E.Responder {
                return responder
            }
            responder = responder?.nextResponder()
        }
        return nil
    }
    
}
```

レスポンダチェーンを辿って行って、指定の型のレスポンダが見つかれば`Event.stimulate`で適当なメソッドを呼びだすという感じです。

