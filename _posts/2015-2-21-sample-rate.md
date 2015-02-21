---
layout: post
title: AVCaptureSessionのマイク入力のサンプリングレートを設定する
---


サンプリングレートはAVAudioSession経由で設定可能ですが(デフォルトだと44.1kHzでサンプリングするみたいです。) 、
AVCaptureSessionに設定する場合は以下のようにするようです。

[Audio Session プログラミングガイド](https://developer.apple.com/jp/devcenter/ios/library/documentation/AudioSessionProgrammingGuide.pdf)　によると

> AV Foundationの取り込み処理API（AVCaptureDevice、AVCaptureSession）で、カメラやマイクの
入力から、同期音声/画像を取り込むことができます。iOS 7以降、マイク入力を表すAVCaptureDevice
オブジェクトは、アプリケーションのAVAudioSessionを共有できるようになりました。通常は
AVCaptureSessionが、AVCaptureSessionがマイクを使っているとき録音に適したように
AVAudioSessionを設定するようになっています。一方、
automaticallyConfiguresApplicationAudioSessionプロパティをNOにすると、AVCaptureDevice
が、現在のAVAudioSession設定をそのまま使うようになります。

基本的には自動でうまいこと設定してくれますが、アプリケーションのAVAudioSessionの設定を反映したいなら`automaticallyConfiguresApplicationAudioSession`をNOにして手動で設定してね、ということみたいです。

### AVAudioSessionの設定
+ sampleRate
+ Category
+ Mode
+ I/O Buffer

を設定したらOKでした。

エラー処理・値の反映確認は省略します。

```
NSError *error;

//8kHzに設定
[[AVAudioSession sharedInstance] setPreferredSampleRate:8000 error:&error];

[[AVAudioSession sharedInstance] setCategory:AVAudioSessionCategoryPlayAndRecord error:&error];

[[AVAudioSession sharedInstance] setMode:AVAudioSessionModeVideoRecording error:&error];

[[AVAudioSession sharedInstance] setPreferredIOBufferDuration:0.5 error:&error];
            
[[AVAudioSession sharedInstance] setActive:YES error:&error];
```
    
### AVCaptureSessionの設定

```
self.session = [[AVCaptureSession alloc] init];
[self.session beginConfiguration];
self.session.automaticallyConfiguresApplicationAudioSession = NO;
   ...
[self.session commitConfiguration]; 
```
