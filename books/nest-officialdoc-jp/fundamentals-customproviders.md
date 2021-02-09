---
title: fundamentals-customproviders
---

# カスタムプロバイダ
先の章では、依存性インジェクション（DI）の様々な側面と、Nestでの使われ方について触れた。その一例として、インスタンス（多くの場合サービスプロバイダ）をクラスにインジェクションする為に使用される、コンストラクタベースの（※providersのdependency-injectionチャプターを参照）DIがある。DIがNestコアの根本部分に組み込まれている事を知って驚く事はないだろう。これまで主なパターンを一つだけ紹介してき。アプリケーションがより複雑になるにつれ、DIシステムの全機能を利用する必要が生ずるかもしれない。もっと詳しく調べてみよう。

## DIの基本
DIは[制御の逆転（inversion of control）](https://en.wikipedia.org/wiki/Inversion_of_control)の技法の一つだ。依存関係のインスタンス化を自分のコードで強制的に行うのではなく、IoCコンテナ（ここではNestJSランタイムシステム）に委譲する。Providersの章のサンプルで何が起きているのか確認しよう。

まずプロバイダを定義する。`@Injectable()`デコレータが`CatService`をプロバイダとして印付ける。

```ts :cats.service.ts
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  findAll(): Cat[] {
    return this.cats;
  }
}
```

そして、Nestにプロバイダをコントローラクラスへとインジェクションさせる。

```ts :cats.controller.ts 
import { Controller, Get } from '@nestjs/common';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

最後に、我々はNestのIoCコンテナにプロバイダを登録する。

```ts :app.module.ts 
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

この仕事のために、ベッドの中では正確には何が起きているのだろう？　そのプロセスには３つの重要なステップがある。

1. `cats.service.ts`では、`@Injectable()`デコレータはNestのIoCコンテナで管理できるクラスとして`CatsService`クラスを宣言している。
2. `cats.controller.ts`では、`CatsController`がコンストラクタのインジェクションを使って`CatsService`トークンへの依存関係を宣言している。

```ts
constructor(private catsService: CatsService)
```

3. `app.module.ts`では、トークンの`CatsService`を`cats.service.ts`ファイルの`CatsService`クラスに関連付けている。この関連付け（登録とも呼ばれる）が正しくはどのように行われるのか、下記standard providersのチャプターで参照していこう。

NestのIoCコンテナが`CatsController`のインスタンスを作成すると、最初に依存関係（※）を探す。`CatsService`の依存関係を見つけると、`CatsService`トークンの検索を行い、登録ステップ（先述の3.番）にしたがって`CatsService`クラスを返す。シングルトンスコープ（デフォルトの動作）を想定して、Nestは`CatsService`のインスタンスを作成して返すか、既にキャッシュされている場合は既存のインスタンスを返す。  
※この説明はポイントの説明の為少し簡略化している。重要な点としては、コードの依存関係を分析するプロセスが非常に洗練されており、アプリケーションのブートストラップ中に行われる事。１つの重要な特徴は、依存関係分析（もしくは「依存関係グラフの作成」）が繊維的であるという事だ。上記の例では、`CatsService`それ自体が依存性を持っていた時、それも解決される。依存関係グラフは、依存関係が正しい順序で解決される事を保証する。原則的に、「ボトムアップ」で。このメカニズムによって、開発者は複雑な依存関係グラフを管理する必要から解放される。