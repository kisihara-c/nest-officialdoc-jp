---
title: techniques-performance
---

# パフォーマンス（Fastify）

Nestはデフォルトでは[Express](https://expressjs.com/)を使用している他に、先述通り[Fastify](https://github.com/fastify/fastify)等他のライブラリとの互換性も提供している。フレームワークアダプタを実装することで、フレームワーク依存からの脱却を目指している。このアダプタの主な機能は、ミドルウェアとハンドラを適切なライブラリ固有の実装にプロキシする事だ。

>HINT  
>フレームワークアダプタを実装するためには、ターゲットとなるライブラリが、Expressに見られるようなrequest/responceのパイプライン処理を提供しなければならない。

FastifyはExpressと同様の方法で設計上の問題を解決する為、代替フレームワークとして適している。しかし、FastifyはExpressよりもはるかに**高速**で、ベンチマークの結果は約2倍となっている。なぜNestはデフォルトでExpressを使っているのか、と思うのは当然だが、それについてはExpressが有名で、互換性のあるミドルウェアをたくさん持っていて、それらを簡単に使えるからだ。

しかしNestはフレームワークに依存しないから、二者を簡単に移行できる。高速なパフォーマンスを重視する場合には、Fastifyがより良い選択となる。その為には、後述する通り、組み込みの`FastifyAdapter`を使うだけだ。

## インストール

最初にパッケージをインストールする。

```
$ npm i --save @nestjs/platform-fastify
```

>WARNING  
>バージョン7.5.0以上の`@nestjs/platform-fastify`と、`apollo-server-fastify`を使っている時、バージョン`^3.0.0`（訳注：キャレット記号の意味がわからず。チルダだったら「前後」らしいが…？）のfastifyとの互換性がないため、GraphQL playgroundが動作しない可能性がある。

## アダプタ

Fastifyプラットフォームがインストールされたら、`FastifyAdapter`を使う。

```ts :main.ts 
import { NestFactory } from '@nestjs/core';
import {
  FastifyAdapter,
  NestFastifyApplication,
} from '@nestjs/platform-fastify';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter()
  );
  await app.listen(3000);
}
bootstrap();
```

デフォルトでは、Fastifyは`localhost 127.0.0.1`インターフェイスのみリッスンする（[詳細](https://www.fastify.io/docs/latest/Getting-Started/#your-first-server)）。他のホストでの接続を受け入れたい場合は、`listen()`コールで`'0.0.0.0'`を指定する必要がある。

```ts
async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter()
  );
  await app.listen(3000, '0.0.0.0');
}
```

## プラットフォーム固有パッケージ

`FastifyAdapter`を使う時NestはFastifyを使う事、そして結果Expressに依存したレシピが動作しなくなる可能性がある事を意味する。代わりに、Fastifyの同等パッケージを使う必要がある。

## リダイレクトレスポンス

FastifyはリダイレクトのresponceをExpressとは少し違う形で処理する。Fastifyで適切なリダイレクトを行うには、以下のようにステータスコードとURLの両方を返す。

```ts
@Get()
index(@Res() res) {
  res.status(302).redirect('/login');
}
```

## Fastifyのオプション

FastifyAdapterコンストラクタを通して、Fastifyのコンストラクタに引数を渡す事ができる。

```ts
new FastifyAdapter({ logger: true });
```

## サンプル
実際に動くサンプルは[こちら](https://github.com/nestjs/nest/tree/master/sample/10-fastify)