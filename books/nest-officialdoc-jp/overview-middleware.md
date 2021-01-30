---
title: overview-middleware
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