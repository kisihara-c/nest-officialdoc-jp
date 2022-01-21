---
title: fundamentals-customproviders
---

# カスタムプロバイダ
先の章では、依存性インジェクション（DI）の様々な側面と、Nestでの使われ方について触れた。その一例として、インスタンス（多くの場合サービスプロバイダ）をクラスにインジェクションする為に使用される、コンストラクタベースの（※[providersのdependency-injectionの章](./overview-providers##依存性の注入)を参照）DIがある。DIがNestコアの根本部分に組み込まれている事を知って驚く事はないだろう。これまで主なパターンを一つだけ紹介してき。アプリケーションがより複雑になるにつれ、DIシステムの全機能を利用する必要が生ずるかもしれない。もっと詳しく調べてみよう。

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
※この説明はポイントの説明の為少し簡略化している。重要な点としては、コードの依存関係を分析するプロセスが非常に洗練されており、アプリケーションのブートストラップ中に行われる事。１つの重要な特徴は、依存関係分析（もしくは「依存関係グラフの作成」）が**推移的（transitive）**である事だ。上記の例では、`CatsService`それ自体が依存性を持っていた時、それも解決される。依存関係グラフは、依存関係が正しい順序で解決される事を保証する。原則的に、「ボトムアップ」で。このメカニズムによって、開発者は複雑な依存関係グラフを管理する必要から解放される。

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

標準のプロバイダの提供機能以上の機能が必要になったらどうなるだろう？　いくつか例をあげてみよう。

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

ここまでで、標準の、コンストラクタベースのインジェクションパターン（providers章のdependeny-injectionチャプターを参照）を使ったプロバイダのインジェクションを見てきた。このパターンでは、依存関係をクラス名で**宣言**する必要がある。`'CONECTION'`カスタムプロバイダでは、文字列値のトークンを使用する。こういった状況でのインジェクションの方法を見てみよう。まずその為には`@Inject()`デコレータを使用する。このデコレータは、単一の引数――トークンを受け取る。

```ts
@Injectable()
export class CatsRepository {
  constructor(@Inject('CONNECTION') connection: Connection) {}
}
```

> Hint  
> `@inject()`デコレータは`@nestjs/common`パッケージからインポートされている。

上記の例では、説明のために文字列`'CONNECTION'`を直接使用しているが、よりクリーンなコードの為には、トークンを別ファイルに`constants.ts`等として分ける事を推奨する。シンボル型やenum型を扱うのと同様、独自のファイルに記述し、必要に応じてインポートして扱ってほしい。

## クラスプロバイダ、`useclass`
`useClass`構文を使うと、トークンが解決すべきクラスを動的に決定できる。例えば、抽象（かデフォルト）の`ConfigService`クラスがあるとする。現在の環境に応じて、Nestでサービス設定の異なる実装を提供したい。以下のコードでそのストラテジを実装・実現している。

```ts
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};

@Module({
  providers: [configServiceProvider],
})
export class AppModule {}
```

いくつか詳細を見てみよう。最初に`configServiceProvider`をリテラルオブジェクトで定義し、それをモジュールデコレータの`providers`プロパティに渡している事に気づくだろう。これはちょっとしたコード捌きだが、この章でこれまで利用してきた例と機能的には同等のものだ。

また、ここでは`ConfigService`のクラス名をトークンとして使っている。`ConfigService`に依存するあらゆるクラスに対し、Nestは提供されたクラス（`DevelopmentConfigService`か`ProductionConfigService`）のインスタンスをインジェクションし、他の場所で宣言されたデフォルトの実装をオーバーライドする（例えば、`@Injectable()`デコレータで宣言された`ConfigService`等）。

## ファクトリープロバイダ、`useFactory`
`useFactory`構文を使用すると、プロバイダを**動的**に作成できる。実際のプロバイダはファクトリー関数から返される値によって供給される。ファクトリー関数は必要に応じて単純なものでも複雑なものでも構わない。単純なファクトリー関数は他のプロバイダに依存しないかもしれない。複雑なファクトリー関数は、その関数自体で、結果を導き出す為に必要なプロバイダをインジェクションできる。後者のケースの為に、ファクトリープロバイダ構文は一対の関連するメカニズムを持っている。

1. ファクトリー関数は、省略可能な引数を受け入れられる。
2. 省略可能なinjectプロパティはプロバイダの配列を受け取る。対象となるプロバイダは、インスタンス化プロセス中にNestが解決し、ファクトリ関数への引数として渡すもの。２つのリストは相関関係がある。Nestは`inject`リストからのインスタンスを、同じ順序でファクトリー関数に引数として渡す。

サンプルでデモンストレーションする。

```ts
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
})
export class AppModule {}
```

## エイリアスプロバイダ、`useExisting`
`useExisting`構文を使うと、既存のプロバイダのエイリアスを作る事ができる。エイリアスを作れば、同じプロバイダにアクセスする二つの方法が生まれる。以下の例において、（文字列ベースの）`'AliasedLoggerSerive'`トークンは（クラスベースの）`LoggerService`トークンのエイリアスである。``AliasedLoggerService``、`LoggerService`のそれぞれに一つずつ異なる依存関係があるとしよう。両方の依存関係が`SINGLETON`スコープで指定されている場合、両方とも同じインスタンスに解決される。

```ts
@Injectable()
class LoggerService {
  /* implementation details */
}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}
```

## 非サービスベースのプロバイダ
プロバイダはサービスを提供する事が多い。その利用法に限りはない。プロバイダはどんな値も提供できる。例えば以下のように、プロバイダは現在の環境に基づいた設定オブジェクトの配列を提供できる。

```ts
const configFactory = {
  provide: 'CONFIG',
  useFactory: () => {
    return process.env.NODE_ENV === 'development' ? devConfig : prodConfig;
  },
};

@Module({
  providers: [configFactory],
})
export class AppModule {}
```

## カスタムプロバイダのエクスポート
あらゆるプロバイダと同様、カスタムプロバイダは宣言したモジュールにスコープ化される。他のモジュールから見えるようにするには、エクスポートする必要がある。カスタムプロバイダをエクスポートする為には、そのトークンか、プロバイダオブジェクト自体のいずれも使用できる。

```ts
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
```

プロバイダオブジェクト自体の場合は以下もう一つの例にて。

```ts
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```