---
title: "overview-middleware"
---


# ミドルウェア
ミドルウェアとは、ルートハンドラの**前**に呼び出される関数です。アプリケーションのリクエストとレスポンスの中で、リクエスト・オブジェクト、レスポンス・オブジェクト、そして`next()`ミドルウェアへのアクセス権限を持っています。**次**のミドルウェアが通常変数名`next`で表されます。  
Nestのミドルウェアはデフォルトではexpressのミドルウェアと等しいです。以下の説明はexpress公式のもので、ミドルウェアの機能を説明しています。

>ミドルウェアは以下の仕事を遂行する。
>- あらゆるコードの実行
>- リクエスト、レスポンスオブジェクトの変更
>- リクエスト-レスポンスサイクルの終了
>- スタック内の次のミドルウェアを呼び出す。
>- 現在のミドルウェアがリクエスト-レスポンスサイクルを終了しない場合、次のミドルウェアに制御を渡す為にnest()を呼ぶ必要がある。呼ばなければ、リクエストはハングアップする。

カスタムミドルウェアは関数か`@injectable()`デコレータ持ちのクラスによって実装する。クラスは`NestMiddleware`インターフェイスを実装する必要があるものの、関数に特別な要件はない。まずはクラスメソッドを使いシンプルなミドルウェアを作成しよう。

```ts :logger.middleware.ts 
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('Request...');
    next();
  }
}
```

# インジェクション
Nestのミドルウェアは、依存性インジェクションを完全にサポートしている。プロバイダ・コントローラと同様に、同モジュール内で依存関係のインジェクションを行える。その際はコンストラクタを使う。

# ミドルウェアを適用する
`@Module()`デコレータにはミドルウェアを配置する場所がない。代わりに、モジュールクラスの`configure()`メソッドを使って設定する。ミドルウェアを含むモジュールは、`NestModule`インターフェイスを実装する必要がある。ここでは、`AppModule`のレベルで`LoggerMiddleWare`を設定しよう。

```ts :app.module.ts 
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
```
>HINT  
>`apply()`メソッドは単一のミドルウェア、もしくはマルチプルミドルウェア（multiple middlewaresの項を参照）を指定するための複数の引数を取る事ができる。

## ルートの除外
At times we want to exclude certain routes from having the middleware applied. （訳出できず、一旦置きます…）　`exclude()`メソッドを使用すると特定のルートを簡単に除外する事ができる。以下のように単一の文字列、複数の文字列、もしくはRouteInfoオブジェクトを渡して実行する。

```ts
consumer
  .apply(LoggerMiddleware)
  .exclude(
    { path: 'cats', method: RequestMethod.GET },
    { path: 'cats', method: RequestMethod.POST },
    'cats/(.*)',
  )
  .forRoutes(CatsController);
```
>Hint
>`exclude()`メソッドは[`path-to-regexp`](https://github.com/pillarjs/path-to-regexp#parameters)パッケージを使用したワイルドカードパラメータをサポートしている。

上記の例では、`LoggerMiddleWare`は`exclude()`メソッドに渡された３つのパスを除いた状態で`CatsController()`のルートに従う。

## 関数ミドルウェア
これまで説明してきた`LoggerMiddleware`クラスは非常にシンプルで、メンバも追加メソッドも依存関係もない。クラスではなくシンプルな関数で実装してみようか？　実は可能だ。このタイプのミドルウェアは関数ミドルウェアと呼ばれる。違いを説明するため、LoggerMiddleWareを関数ミドルウェアにしてみよう。

```ts :logger.middleware.ts 
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`Request...`);
  next();
};
```

そして、`AppModule`で使う。

```ts: app.module.ts 
consumer
  .apply(logger)
  .forRoutes(CatsController);
```

>Hint  
>作成検討中のミドルウェアが依存関係を必要としない場合、よりシンプルな関数ミドルウェアの使用の検討余地がある。

## マルチプルミドルウェア
前述のように、順次実行される複数のミドルウェアをバインドするには、`apply()`メソッドの中でカンマ区切りのリストを指定する。

```
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
```

## グローバルミドルウェア
登録されている全てのルートに一気にミドルウェアをバインドしたい場合は、`INestApplication`インスタンスから供給される`use()`メソッドを使用する。

```ts
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(3000);
```