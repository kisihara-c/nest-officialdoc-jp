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

## 標準のプロバイダ
`@Module`デコレータをもっと詳しく見てみよう。`app.module`内で宣言する。

```ts
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
```

`providers`プロパティではプロパイダの配列を指定する。これまでのところ、これらプロパイダはクラス名のリスト越しに提供されている。実際には、`providers: [CatsService]`という記法は以下のもっと複雑な記法の短縮形だ。

```ts
providers: [
  {
    provide: CatsService,
    useClass: CatsService,
  },
];
```

さて、ここにある明確な構造が見えてきた。今なら登録処理を理解できるだろう。ここではトークン`CatsService`とクラス`CatsService`を明確に関連付けている。この短縮形は最も一般的なユースケースを単純化するための便宜上のものであり、トークンは同じ名前のクラスのインスタンスを探す為に使用される。

## カスタムプロバイダ

スタンダードプロバイダーが提供する機能以上が必要になったらどうなるだろう？　いくつか例をあげてみよう。

- Nestにクラスのインスタンスを作成させる（またはキャッシュされたインスタンスを返す）代わりに、カスタムインスタンスを作成したい
- 既存のクラスを二番目の依存関係で再利用したい
- テストのためにモック版でクラスをオーバーライドしたい

Nestではこういったケースを処理するためのカスタムプロバイダを定義する事ができる。方法はいくつかある。見てみよう。

## 変数のプロバイダ：`useValue`

`useValue`構文は、定数値をインジェクションしたり、外部ライブラリをNestのコンテナに入れたり、実際の実装をモックオブジェクトに置き換えたりする場合に便利だ。例えば、テストの為にモックの`CatsService`を強制的にNestで使用したい場合を考えてみよう。

```ts
import { CatsService } from './cats.service';

const mockCatsService = {
  /* mock implementation
  ...
  */
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}
```

この例では`CatsService`トークンは`mockCatsService`モックオブジェクトに解決される。`useValue`は値を必要とする。この場合は置き換え先の`CatsService`クラスと同じインターフェイスを持つリテラルオブジェクトだ。TypeScriptの[構造的型付け](https://www.typescriptlang.org/docs/handbook/type-compatibility.html)のおかげで、リテラルオブジェクトや`new`でインスタンス化されたクラスインスタンスなど、互換可能なインターフェイスを持つ全てのオブジェクトを使用する事ができる。

## 非クラスベースのプロバイダトークン

ここまでプロバイダトークン（プロバイダ配列にリストされているプロバイダの`provide`）としてクラス名を使用している。これはコンストラクタベースのインジェクション（providersの該当チャプター参照）で使用される標準パターンと一致しており、トークンはクラス名でもある（トークンの概念が明快でない場合は画面を遡ってDIの基本チャプターに戻ろう）。場合によっては、DIトークンとして文字列や記号を使用できる柔軟性が必要になる。例：

```ts
import { connection } from './connection';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useValue: connection,
    },
  ],
})
export class AppModule {}
```

この例では外部ファイルからインポートした既存の接続オブジェクトに文字列値のトークン（`CONNECTION`）を関連付けている。

> Notice
> トークン値として、文字列だけではなくJavaScriptのシンボルやTypeScriptの列挙型を使用する事もできる。