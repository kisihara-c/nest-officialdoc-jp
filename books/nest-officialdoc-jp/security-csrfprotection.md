---
title: security-csrfprotection
---

# CSRF対策

クロスサイトリクエストフォージェリ（CSRF、XSRF）とは、Webアプリケーションが信頼するユーザから不正なコマンドが送信される、Webサイト攻撃の一種だ。対抗する為、[csurf](https://github.com/expressjs/csurf)パッケージを使用できる。

## Expressで使う

必要なパッケージのインストールから始めよう。

```
$ npm i --save csurf
```

>WARNING  
>[csurfミドルウェアのページ](https://github.com/expressjs/csurf#csurf)で説明されているように、csurfモジュールはセッションミドルウェアかcookieパーサーを最初に初期化する必要がある。詳細はその説明を参照の事。

インストールが完了したら、csurfミドルウェアをグローバルミドルウェアとして適用する。

```ts
import * as csurf from 'csurf';
// 初期化ファイルのどこか
app.use(csurf());
```

## Fastifyで使う

必要なパッケージのインストールから始めよう。

```
$ npm i --save fastify-csrf
```

インストールが完了したら、以下のように`fastify-csrf`プラグインを登録する。

```ts
import fastifyCsrf from 'fastify-csrf';
// 初期化ファイルのどこか
app.register(fastifyCsrf);
```