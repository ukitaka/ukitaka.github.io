---
layout: post
title: DockerでMySQLクライアントを動かす
---

サーバーとしてではなくて`mysql` コマンドを使いたいとき。

```shell
docker run -it --rm mysql mysql -hsome.mysql.host -usome-mysql-user -p
```

[[参考] Docker Hub - MySQL](https://hub.docker.com/_/mysql/)
