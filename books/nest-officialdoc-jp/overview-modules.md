---
title: "overview-modules"
---

# モジュール

モジュールは`@Module()`デコレータで装飾されたクラス。アプリケーションの構造を整理する為にNestが使うメタデータを提供する。  

![画像](https://docs.nestjs.com/assets/Modules_1.png)

各アプリケーションは、少なくとも１つのモジュール（ルートモジュール）を持っている。ルートモジュールはNestがアプリケーションのツリー構造（つまり、モジュールやプロバイダの関係・依存関係を解決する為に使われる内部データ構造）を構築する為の出発点です。ほとんどのアプリケーションでは、アーキテクチャの中で複数のモジュールを使用する事になります。非常に小さなアプリケーションではツリーがルートモジュールのみという事もありえますが、それはあくまで例外です。コンポーネントを整理する為、モジュールの活用を強くおすすめします。  
`@Module()`デコレータは、モジュールを管理するプロパティを持つ単一のオブジェクトを受け付けます。
|||
| ---- | ---- |
|providers|Nestのインジェクタによってインスタンス化され、少なくともこのモジュールで共有される可能性のあるプロパイダ一覧|
|controllers|インスタンス化された、このモジュールで宣言されるコントローラ一覧|
|imports|このモジュールで必要なプロパイダをエクスポートするモジュールの一覧|
|exports|このモジュールで提供するプロパイダをインポートする別のモジュール一覧|

モジュールは標準でプロパイダをカプセル化する。つまり、そのモジュールの一部でもimportsでもexportsでもないプロパイダを注入する事はできない。したがって、モジュールからエクスポートされたプロパイダは、モジュールのパブリックインターフェイス、もしくはAPIと考える事ができる。

## 機能モジュール

`CatsController`と`CatsService`は同じアプリケーションドメインに属している。この２つは密接に関連しており、「機能モジュール」に移動させる事は理に適っている。機能モジュールは特定の機能に関連するコードを整理し、組織化して、明確な境界線を作る。この事で、特にアプリケーションやチームの規模が大きくなってきた時に、複雑さを管理しながらSOLID原則に基づいて開発できる。  
`CatsModule`を作りデモンストレーションする。

```ts :cats/cats.module.ts 
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

>HINT  
>CLIツールでモジュールを作成するには、`$ nest g module cats`を使用。

上記では`CatsModule`を定義し、関連する全てを`cats`ディレクトリに移動した。最後にこのモジュールをルートモジュールにインポートする。ルートモジュールは、`app.module.ts`ファイルで`AppModule`として定義されている。

```ts :app.module.ts 
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```

現在のディレクトリ構造は以下の通り。

- src
    - cats
        - dto
            - create-cat.dto.ts
        - interfaces
            - cat.interface.ts
        - cats.service.ts
        - cats.controller.ts
        - cats.module.ts
 - app.module.ts
 - main.ts



 ## 共有モジュール
 Nestにおいてはモジュールは標準でシングルトンである為、複数のモジュール間で任意のプロバイダのインスタンスを共有できる。

![画像](https://docs.nestjs.com/assets/Shared_Module_1.png)

 すべてのモジュールは自動的に共有モジュールとなる。一度作成されたモジュールは、全てのモジュールで再利用できる。例えば、`CatsService`のインスタンスを他の複数のモジュール間で共有したい場合をイメージする。まず、`CatsService`プロバイダをモジュールの`exports`配列に追加し、エクスポートする。

 ```ts :cats.module.ts
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
 ```

 `CatsModule`をインポートしたモジュールは`CatsService`にアクセスでき、`CatService`をインポートした他のすべてのモジュールと同じインスタンスを共有します。

 ## モジュールの再エクスポート
上記の通り、モジュールは内部プロパイダをエクスポートできる。加えて言えば、モジュールはインポートしたモジュールを再エクスポートできる。以下の例では、`CommonModule`は`CoreModule`にインポートされ、かつそこからエクスポートされている。つまり、これをインポートすれば`CommonModule`を他のモジュールで使用できる。

```ts
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

## 依存関係のインジェクション
モジュールクラスは、（例えば構成のために）プロバイダをインジェクションする事もできる。

```ts :cats.module.ts 
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
```

しかし、モジュールクラス自体は[Circular dependency（該当項を参照の事）](./fundamentals-circulardependency)の為、インジェクションできない。

## グローバルモジュール
全ての場所で同じモジュールのセットをインポートする必要があるとなると、面倒だ。Angularのproviderはグローバルスコープに載り、一度定義すればどこでも利用できるものの、Nestではプロパイダがモジュールのスコープ内にカプセル化され、インポートしない限り他の場所では使えない。  
どこでも利用可能でお手軽なプロバイダを提供したい場合（例：ヘルパー、データベース接続）、`@Global()`デコレータを使用してモジュールをグローバルに設定する。

```ts
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

`Global()`デコレータはモジュールをグローバルスコープ化する。グローバルモジュールは、一般的にはルートもしくはコアモジュールによって一度だけ登録されるべき。上記の例では`CatsService`プロバイダはシングルトンとなり、サービスをインジェクションしたいモジュールが、`CatsModule`を`Import`に付け加える必要はない。

>Hint
>全モジュールをグローバル化する事は良いデザインではない。グローバルモジュールは必要な決まり文句の量を減らす為に利用できるものだ。一般的にはモジュールのAPIを提供する為にはインポート機能を利用したほうが良い。

## 動的モジュール
Nestのモジュールシステムには動的モジュールという強力な機能がある。プロバイダの登録や設定を動的に行えるカスタマイズ可能なモジュールを、簡単に作成する事ができる。詳細は該当の別項目を参照の事。この章では導入の為、簡単な概要を示す。  
以下は動的モジュール`DatabaseModule`の定義の例。

```ts
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

>Hint  
>`forRoot()`メソッドは同期/非同期（`Promise`経由）で動的モジュールを返すメソッド。

このモジュールは標準で接続プロバイダを定義している（`@Module()`デコレータのメタデータによる）が、さらに`footRoot()`メソッドに渡されたエンティティや追加オブジェクトに応じて複数のプロバイダ（例：リポジトリ等）を公開する。注意としては、動的モジュールが返すプロパティは、`@Module()`デコレータで定義されているベースモジュールの（上書きではなく）拡張である事。この流れで、静的に宣言された`Connection`プロバイダと動的に生成されたリポジトリプロバイダの両方がモジュールからエクスポートされる。  
グローバルスコープに動的モジュールを登録したい場合は、`global`プロパティを`true`とする。

```ts
{
  global: true,
  module: DatabaseModule,
  providers: providers,
  exports: providers,
}
```

>WARNING  
>前述の通り、全てをグローバルスコープにするのは良いデザインではない。

`DatabaseModule`は以下のようにインポートして設定する。

```ts
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
})
export class AppModule {}
```

動的モジュールを再エクスポートしたい場合は、exports配列の`forRoot()`メソッドの呼び出しを省略可能。

```ts
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
export class AppModule {}
```
なお、この先の[Dynamic module項](./fundamentals-dynamicmodules)にて更に詳細と実例を説明している。
