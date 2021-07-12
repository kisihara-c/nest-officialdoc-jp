---
title: security-helmet
---

# Helmet

[Helmet](https://github.com/helmetjs/helmet)は、HTTPヘッダーを適切に設定する事で、よく知られたウェブの脆弱性からアプリケーションを保護してくれるものだ。一般的に言うと、Helmetはセキュリティ関連のHTTPヘッダを設定する14個の小さなミドルウェア関数が構成される（[詳細](https://github.com/helmetjs/helmet#how-it-works)）。

>HINT  
>helmetをグローバルに適用or登録するには、`app.use()`や`app.use`を呼ぶかもしれないセットアップ関数を呼び出す前に行わなければならない事に注意。これは基盤となるプラットフォーム（ExpressやFastify等）の仕組によるもので、ミドルウェアやルートが定義される順番は重要となる。ルートを定義した後にhelmetやcors等のミドルウェアを使用した場合、そのミドルウェアはそのルートには適用されず、ルートの後に定義されたミドルウェアのみに適用される（then that middleware will not apply to that route, it will only apply to middleware defined after the route. ）。

## Expressで使う（デフォルト）

まず必要なパッケージをインストールしよう。

```
$ npm i --save helmet
```

インストールが完了したら、グローバルミドルウェアとして適用する。

```ts
import * as helmet from 'helmet';
// 初期化ファイルのどこか
app.use(helmet());
```

>HINT  
>Helmetをインポートしようとして`This expression is not callable`エラーが出た場合、`tsconfig.json`内で`allowSyntheticDefaultImports`と`esModuleInterop`が`true`になっている可能性がある。その場合は代わりのインポート文として`import helmet from 'helmet'`としてほしい。

## Fastifyを使う。

`FastifyAdapter`を使う場合、[fastify-helmet](https://github.com/fastify/fastify-helmet)をインストールしよう。

```
$ npm i --save fastify-helmet
```

[fastify-helmet](https://github.com/fastify/fastify-helmet)はミドルウェアとしてではなくFastifyのプラグインとして、つまり`app.register()`を使って使用する。

```ts
import { fastifyHelmet } from 'fastify-helmet';
// 初期化ファイルのどこか
await app.register(fastifyHelmet);
```

>WARNING  
>`apollo-server-fastify`と`fastify-helmet`を使用している場合、GraphQL playgroundの[CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)に問題が発生する可能性がある。
>
>```ts
>await app.register(fastifyHelmet, {
>  contentSecurityPolicy: {
>    directives: {
>      defaultSrc: [`'self'`],
>      styleSrc: [`'self'`, `'unsafe-inline'`, 'cdn.jsdelivr.net', 'fonts.googleapis.com'],
>      fontSrc: [`'self'`, 'fonts.gstatic.com'],
>      imgSrc: [`'self'`, 'data:', 'cdn.jsdelivr.net'],
>      scriptSrc: [`'self'`, `https: 'unsafe-inline'`, `cdn.jsdelivr.net`],
>    },
>  },
>});
>
>// CSPを全く使用しない場合は、これを使える。
>await app.register(fastifyHelmet, {
>  contentSecurityPolicy: false,
>});
>```