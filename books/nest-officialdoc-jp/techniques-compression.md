---
title: techniques-compression
---

# 圧縮

圧縮機能を使うことで、レスポンス本文のサイズを大幅に縮小し、ウェブアプリケーションの速度を向上させる。

トラフィック量の多い実稼働中のウェブサイトでは、アプリケーションサーバーから圧縮をオフロードする事を強く勧める。通常はリバースプロキシ（例：Nginx）を使用する。この場合、圧縮ミドルウェアは使用すべきではない。

## Expressで使う

gzip圧縮を有効にするには、圧縮ミドルウェアパッケージを使用する。

まず必要なパッケージをインストールしよう。

```
$ npm i --save compression
```

インストールが完了したら、グローバルミドルウェアとして[圧縮ミドルウェア](https://github.com/expressjs/compression)を適用する。

```ts
import * as compression from 'compression';
// 初期化ファイルのある場所
app.use(compression());
```

## Fastifyと使う

`FastifyAdapter`を使うなら、[fastify-compress](https://github.com/fastify/fastify-compress)を使いたくなるだろう。

```
$ npm i --save fastify-compress
```

インストールが完了したら、fastify-compressのミドルウェアをグローバルミドルウェアとして適用する。

```ts
import compression from 'fastify-compress';
// 初期化ファイルのある場所
app.register(compression);
```

デフォルトでは、ブラウザがBrotli圧縮をサポートしている場合、fastify-compressはBrotli圧縮を使用する（Nodeのvが11.7.0以上の場合）。Brotliは圧縮率は良いが、非常に重い。fastify-compressにはdeflateとgzipのみを使用してレスポンスを圧縮させるのがオススメだ。

エンコーディングを指定するには、`app.register`に2番目の引数を与える。

```ts
app.register(compression, { encodings: ['gzip', 'deflate'] });
```

この例では、`fastify-compress`はgzipとdeflateのエンコーディングのみを使用する。そしてクライアントが両方をサポートしている場合はgzipを優先する。