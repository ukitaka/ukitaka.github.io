---
layout: post
title: CircleCIはシミュレータ起動が失敗してテストこける問題に対応してた
---


CircleCI, iOSのビルドがβ版ですが利用できるようになって試している方も多そうな気がします。
自分も正月くらいに試してみましたが、たまーにシミュレータ起動でこけてfailするときがありました。
Jenkinsの場合ですが、[この記事](http://qiita.com/mzp/items/cca2bef33ecb81efd0f5)によると、

> iPhoneシミュレータの起動にGUIセッションがいるので、特定ユーザでログインしっぱなしにしないといけない。

とのことなのでおそらくCircleCIでも同じような問題が起こっていたのかなと予想しています。
(ちなみに[解決策](https://github.com/facebook/xctool/issues/404#issuecomment-67274317)はこうらしい)


そして今日たまたまCircleCIのCHANGELOGを眺めてたら、なにやら改善されていました。


> Split iOS compilation and testing to improve iOS simulator stability

> We have changed the default steps that we take to build an iOS project. We have split the compilation phase and the testing phase in two. This improves the stability when running tests in the iOS simulator.

> Update Feb 17


> Automatically retry iOS tests when the simulator fails to launch

> Circle will now automatically re-run iOS tests if the simulator fails to launch. This allows us to work around known issues with the iOS simulator that have been causing builds to fail.

> Update by Gordon Syme on Feb 18


まだfailすることはあるけど、自動でRetryしてくれるになったんですね。ありがたや。
