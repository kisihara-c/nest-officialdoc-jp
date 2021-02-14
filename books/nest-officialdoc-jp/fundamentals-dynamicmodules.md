---
title: fundamentals-dynamicmodules
---

# 動的モジュール

[モジュールの章](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-modules)ではNestモジュールの基本をカバーした。動的モジュールも簡単に紹介した。この章では動的モジュールのテーマをより広く説明する。この章を終えるとモジュールとはなにか、どう、いつ使うかが十分に理解できるはずだ。

## イントロダクション
**Overview**部で紹介したほとんどのアプリケーションコード例は、通常のモジュールか静的モジュールを使用している。モジュールは協働する[プロバイダ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-providers)や[コントローラ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-controllers)などのコンポーネントのグループをアプリケーション全体の一単位として定義する。コンポーネントの実行コンテキストやスコープを提供する。例えばモジュール内で定義されたプロバイダはエクスポートしなくてもモジュールの他のメンバから見える。プロバイダをモジュール外で表示する必要がある場合、まずホストモジュールでエクスポートしてから、使用するモジュールにインポートする。

おなじみの例を見てみよう。

まず、`UsersService`を提供してエクスポートする`UsersModule`を定義する。`UsersModule`は`UsersService`のホストモジュールだ。

```ts
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

次に、`AuthModule`を定義して、`UsersModule`をインポートし、`UsersModule`内のエクスポートされたプロバイダを`AuthModule`内で利用できるようにする。

```ts
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

以下のコードによって、例えば`AuthModule`でホストされている`AuthService`の中で`UsersService`をインジェクションする事ができる。

```ts
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}
  /*
    Implementation that makes use of this.usersService
  */
}
```

これを静的モジュールのバインディングと呼ぶ。Nestがモジュールを配線する為に必要なすべての情報は、すでにthe host and consuming modules（すみません、訳出できず）で宣言されている。このプロセスで何が起きているか紐解いてみよう。以下の順番で、Nestは`AuthModule`の中で`UsersService`を有効にしている。

1. `UsersModule`のインスタンスを作成し、`UsersModule`自身がconsumeする他のモジュールを遷移的にインポートし、依存関係を遷移的に解決（[customprovider](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-customproviders)を参照）
2. AuthModuleのインスタンスを作成し、`AuthModule`のコンポーネントで、（`AuthModule`で宣言されているかのように、）`usersModule`のエクスポートされたプロバイダを利用できるようにする。
3. `AuthService`に`UsersService`のインスタンスをインジェクションする。

## 動的モジュールのユースケース
静的モジュールのバインディングを使っても、ホストモジュールからのプロバイダの設定方法にconsumingモジュールが**影響**を与える機会はない。重要なことだ。なぜか？　異なるユースケースで異なる動作をする必要がある、とある汎用モジュールがあるケースを想定しよう。これは多くのシステムにおける「プラグイン」の概念に類似している。汎用的な機能を消費者が使う前に、多少の設定を必要とする。

Nestでは、**設定モジュール**が良い例だ。多くのアプリケーションでは設定モジュールを使用して設定の詳細を外部化すれば便利になる。開発者のための開発データベース、ステージング/テスト環境のためのステージングデータベースなど、それぞれのデプロイごとにアプリケーションの設定を**動的**に変更できるようになる。設定パラメータの管理を設定モジュールに委任すれば、アプリケーションのソースコードを設定パラメータから独立したままにできる。

課題は設定モジュール自体が（「プラグイン」に似て）汎用的ゆえに、consumingモジュールによってカスタマイズする必要がある事だ。そこで動的モジュールの出番だ。動的モジュールの機能を使用すると、設定モジュールを動的にできる。consumingモジュールがAPIを使用する事で、インポート時に設定モジュールがどうカスタマイズされるか制御できるようになる。

言い換えれば、動的モジュールは、これまで見てきた静的なバインディングとは対称的なものだ。あるモジュールを別のモジュールにインポートしつつ、インポート時のプロパティや動作をカスタマイズするAPIを提供する。

## 設定モジュールの例
このチャプターではTECHNIQUES-configurationのチャプターからサンプルコードを取り上げている。完成品は[こちら](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)。

`ConfigModule`に、カスタマイズ用の`options`オブジェクトを受け入れさせたい。以下の機能をつけたい。基本のサンプルでは`.env`ファイルをプロジェクトのルートフォルダにハードコーディングしている。これを設定可能にして、`.env`ファイルを好きなフォルダで管理できるようにしたい。例えば、様々な`.env`ファイルを、プロジェクトルートの下の`config`フォルダ（つまり`src`フォルダの兄弟となる）で保存したいと想定してほしい。違うプロジェクトで`ConfigModule`を使用する際に、違うフォルダを選択できるようにしたい。

動的モジュールではインポートされたモジュールにパラメータを渡すことができる。結果、モジュールの動作を変更できる。どう動くか見てみよう。これはconsumingモジュールの視点からどのように見えるかというゴールから始めて逆算してみるとわかりやすい。最初に、`ConfigModule`を静的にインポートする例（つまりインポートされたモジュールの動作に影響を与えられない場合）を手早く見てみよう。`@Module()`デコレータの`imports`配列に注意。

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

動的モジュールのインポートがどう見えるか考えよう。設定オブジェクトを渡した先だ。2例のインポート配列の違いを比較しよう。

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

動的に動く例を見てみよう。どの部分が動いているだろう？

1. `ConfigModule`は通常のクラスなので、`register()`という静的メソッドを持っているはずだ。クラスのインスタンスではなく`ConfigModule`クラスで呼び出している為、静的メソッドとわかる。注意：（私達がこれから作成する）このメソッドには任意の名前をつけられるが、慣例では`forRoot()`か`register()`と呼ぶ事になっている。
2. `register()`メソッドは自分たちで定義している為、好きな入力引数を受けつける事ができる。このケースでは適切なプロパティを持つ単純な`options`オブジェクトを受け取る。典型的なケースだ。
3. `register()`メソッドは`module`のようなものを返さなければならないと推論できる。戻り値がおなじみの`imports`リストに表示されており、これまでに見てきたようにモジュールのリストが含まれているからだ。

事実として、`register()`メソッドが返すのは`DynamicModule`だ。動的モジュールとは、言ってみれば実行時に作成されるモジュールにすぎない。静的モジュールが持つプロパティに加えて、`module`プロパティも持つだけだ。デコレータに渡されるモジュールのオプションに丁寧に注意しながら、静的モジュール宣言のサンプルを簡単に見てみよう。

```ts
@Module({
  imports: [DogsService],
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
```

動的モジュールは全く同じインターフェイスを持つオブジェクトに加えて、追加プロパティ１つ（`module`）を返す必要がある。`module`プロパティはモジュールの名前として機能し、以下の例のようにモジュールのクラス名と同じでなければならない。

>Hint  
>動的モジュールの場合、**`module`を除いて**モジュールオプションオブジェクトのプロパティは全て省略可能。

静的な`register()`メソッドについてはどうだろう？　このメソッドの仕事は`DynamicModule`インターフェイスを持つオブジェクトを返す事だとわかる。このメソッドを呼んだ時、静的に動かす場合にモジュールのクラス名をリストアップする方法と同様に、事実上インポートリストにモジュールを提供している事となる。言い換えれば、動的モジュールのAPIは単にモジュールを返すが、`@Module`デコレータでプロパティを固定せず、プログラム上で指定する。

この見取り図を完成させる為に、まだいくつか詳細説明がある。

1. 今や私達は`@Module()`デコレータの`imports`プロパティに、モジュールのクラス名（例：`imports:[UserModule]`）だけでなく動的モジュールを**返す**（例：`imports:[ConfigModule.register(...)]`）関数も渡せる。
2. 動的モジュールはそれ自身で他のモジュールをインポートできる。この例では行わないが、動的モジュールが他のモジュールのプロパイダに依存している場合、オプションの`imports`プロパティを使用してそれらをインポートする。`@Module()`デコレータを使用して静的モジュールのメタデータを宣言する方法と全く同じだ。

この理解を持ちつつ、動的な`ConfigModule`宣言がどのようなものでなければいけないか見てみよう。実際にやってみる。

```ts
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(): DynamicModule {
    return {
      module: ConfigModule,
      providers: [ConfigService],
      exports: [ConfigService],
    };
  }
}
```

それぞれのパーツの結びつきかたが明らかになったはずだ。`ConfigModule.register(...)`をコールすると、現状基本的には`@Module()`デコレータ経由でメタデータとして提供したものと同じプロパティを持つ`DynamicModule`オブジェクトが返される。

>Hint  
>`DynamicModule`は`@nestjs/common`からインポートする。

しかしながら私達のダイナミックモジュールはまだあまり面白くない。面白い面白くないを話す以前に設定箇所が少ない。次はそこを触ってみよう。

## モジュールの設定
`ConfigModule`の動作をカスタマイズする為の明白な解決策は、前述の推測通り静的な`register()`メソッドで`options`オブジェクトを渡す事だ。もう一度、consumingモジュールの`imports`プロパティを見てみよう。

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { ConfigModule } from './config/config.module';

@Module({
  imports: [ConfigModule.register({ folder: './config' })],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

動的モジュールへの`options`オブジェクトの渡し方をうまくハンドリングしている。ではその`options`オブジェクトを`ConfigModule`でどう使うのか考えてみよう。`ConfigModule`は基本的に、他のプロバイダが使用するインジェクション可能なサービス（`ConfigService`）を、提供したりエクスポートする為のホストだ。実のところ動作をカスタマイズする為に`options`オブジェクトを必要としているのは`ConfigService`だ。ここでは`register()`メソッドから`ConfigService`に`options`を入れ込む方法については、なにかに決めていると仮定し省略する。その前提の上で、`optins`オブジェクトのプロパティに基づいてサービスの動作をカスタマイズする為に、サービスにいくつかの変更を加える事ができる。（**注意**：当面の間は実際にどのようにして入れ込むか決めていない。一度`options`をハードコーディングして、後ですぐ修正する）

```ts
import { Injectable } from '@nestjs/common';
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor() {
    const options = { folder: './config' };

    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

今、`ConfigService`は`options`で指定したフォルダ内の`.env`ファイルの見つけ方を知っている。

残ったタスクは`register()`の処理の`options`オブジェクトを`ConfigService`へなんとかしてインジェクションする事だ。もちろん依存性のインジェクションを行う。ここはキーポイントだからしっかり理解してほしい。`ConfigModule`は`ConfigService`を提供している。同様に、`ConfigService`は実行時にのみ提供される`options`オブジェクトに依存する。ゆえに、実行時ではまず`options`オブジェクトをNestのIoCコンテナにバインドしてから、Nestを通して`ConfigService`にインジェクションする必要がある。customprovidersの章で説明したことを思い出してほしい、プロバイダはサービスだけでなく[あらゆる変数を含む事ができる](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-customproviders#%E9%9D%9E%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E3%83%99%E3%83%BC%E3%82%B9%E3%81%AE%E3%83%97%E3%83%AD%E3%83%90%E3%82%A4%E3%83%80)。シンプルな`options`オブジェクトを処理する為に依存性のインジェクションを行っても問題はない。

まず`options`オブジェクトをIoCコンテナにバインドしてみよう。これは静的な`register()`メソッドで行う。モジュールを動的に構築している事、そしてモジュールのプロパティのうちの一つにプロバイダのリストがある事を覚えておいてほしい。そこで必要になるのは`options`オブジェクトをプロパイダとして定義する事だ。`ConfigService`に対してインジェクション可能となる為、次のステップで必要だ。以下のコードではプロバイダの配列に注目してほしい。

```ts
import { DynamicModule, Module } from '@nestjs/common';
import { ConfigService } from './config.service';

@Module({})
export class ConfigModule {
  static register(options): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options,
        },
        ConfigService,
      ],
      exports: [ConfigService],
    };
  }
}
```

こうなれば、`CONFIG_OPTIONS`プロバイダを`ConfigService`にインジェクションする事で、プロセスを完了できる。非クラスのトークンを使用してプロバイダを定義する時は、[ここ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-customproviders#%E9%9D%9E%E3%82%AF%E3%83%A9%E3%82%B9%E3%83%99%E3%83%BC%E3%82%B9%E3%81%AE%E3%83%97%E3%83%AD%E3%83%90%E3%82%A4%E3%83%80%E3%83%88%E3%83%BC%E3%82%AF%E3%83%B3)で説明した通り`@inject()`デコレータを使用する必要がある事を覚えておいてほしい。

```ts
import * as dotenv from 'dotenv';
import * as fs from 'fs';
import { Injectable, Inject } from '@nestjs/common';
import { EnvConfig } from './interfaces';

@Injectable()
export class ConfigService {
  private readonly envConfig: EnvConfig;

  constructor(@Inject('CONFIG_OPTIONS') private options) {
    const filePath = `${process.env.NODE_ENV || 'development'}.env`;
    const envFile = path.resolve(__dirname, '../../', options.folder, filePath);
    this.envConfig = dotenv.parse(fs.readFileSync(envFile));
  }

  get(key: string): string {
    return this.envConfig[key];
  }
}
```

最後に一つだけ気をつけてほしい。シンプルさの為に文字列ベースのインジェクショントークン（`'Config_OPTIONS'`）を使用したが、ベストプラクティスは別ファイルで定数（またはシンボル）として定義してそのファイルをインポートする事だ。  
例：

```ts
export const CONFIG_OPTIONS = 'CONFIG_OPTIONS';
```

## サンプル
このチャプターの完成サンプルは[こちら](https://github.com/nestjs/nest/tree/master/sample/25-dynamic-modules)