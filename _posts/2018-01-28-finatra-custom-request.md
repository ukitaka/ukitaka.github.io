---
layout: post
title:   finatraでCustom requestを作るときはRequestProxyを使う
---

基本的にはこのドキュメントにあるように、適当なcase classを作ると自動でマッピングしてくれる。

[HTTP Requests — Finatra 18.1.0 documentation](https://twitter.github.io/finatra/user-guide/http/requests.html#custom-request-case-class)


```scala
case class HiRequest(id: Long, name: String)

...

post("/hi") { hiRequest: HiRequest =>
  "Hello " + hiRequest.name + " with id " + hiRequest.id
}
```


ただしこれだと元の`com.twitter.finagle.http.Request`を実装していないことになってしまって、もともとあったヘッダーやHTTPメソッドなどの情報が取れなくなってしまう。
`Request`を実装しようにも必要なフィールドが多くて難しそう。

```scala
case class HogeRequest extends Request {
  override def ctx: Schema.Record = ???

  override def multipart: Option[Multipart] = ???

  override def method: Method = ???

  override def method_=(method: Method): Unit = ???

  override def uri: String = ???

  override def uri_=(uri: String): Unit = ???

  override def remoteSocketAddress: InetSocketAddress = ???

  override def reader: Reader = ???

  override def writer: Writer with Closable = ???

  override def headerMap: HeaderMap = ???
}
```

## com.twitter.finagle.http.RequestProxyを使う

よくよくコメントを見ると、`RequestProxy`を使えと書いてあった。

```scala
/**
 * Rich HttpRequest.
 *
 * Use RequestProxy to create an even richer subclass.
 */
abstract class Request private extends Message {
```

これを使うと`request`だけを実装すればOK。

```scala
/**
 * Underlying `Request`
 */
def request: Request
```


```scala
case class MyRequest(hoge: String, request: Request) extends RequestProxy
```
