---
title: overview-providers
---

# プロバイダ
プロパイダはNestの基本的な概念である。Nestの基本的なクラスの多く――サービス、リポジトリ、ファクトリ、ヘルパー他――はプロバイダとして扱われる、とも言える。プロバイダについて主要なスタンスは、依存関係をインジェクションする事である。オブジェクトはお互い同士で様々な依存関係を作る事ができ、オブジェクトのインスタンスを「配線」する機能を、主にNestのランタイムシステムに委任する事ができる。

[画像](https://docs.nestjs.com/assets/Components_1.png)

前章ではシンプルな`CatsController`を作った。コントローラには、HTTPリクエストを処理し、更に複雑なタスクをプロバイダに委譲する仕事がある。プロバイダは、`@Injectable()`デコレータを前述したプレーンなJavaScriptのクラスだ。

>HINT  
>よりオブジェクト指向的な依存関係の設計・整理の為、SOLID原則に従うことを強く勧める。

# サービス
シンプルな`CatService`を作ろう。このサービスはデータの保存と取得を担当する。`CatsController`で使用するように設計されているので、プロパイダとして定義する。つまり前述の通り`@Injectable()`の装飾を施す。

```ts: cats.service.ts

import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

  findAll(): Cat[] {
    return this.cats;
  }
}

```

>HINT  
>CLIでは、`$ nest g service cats`コマンドでサービスを作れる。

`CatsService`は、プロパティ１つとメソッド２つを持つ基本的なクラスだ。唯一の特筆点は`@Injectable()`デコレータを使用している事だけ。デコレータがメタデータをアタッチし、Nestはこのクラスがプロバイダである事を知る。さて、このサンプルでは`Cat`インターフェイスを使用している。中身はこんな感じだろう。

```ts: interfaces/cat.interface.ts 
export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```

猫たちを受け入れる為のサービスクラスができたので、`CatController`の中で使ってみよう。

```ts: cats.controller.ts 

import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}


```

`CatsService`はクラスのコンストラクタで注入される。private記法に注目の事。この構文で、`CatsService`メンバの宣言と初期化を同時にすぐ行う事ができる。

# 依存関係の注入
NestはDependency injectionと呼ばれる強力なデザインパターンを中心に構築されている。この概念についてはAngularの[ドキュメント](https://angular.io/guide/dependency-injection)を参照の事。

Nestでは、TypeScriptの機能により依存関係は型だけで解決される為、管理が非常に簡単だ。以下の例では、Nestが`CatsService`のインスタンスを作成して返す事で`catsService`を解決する（通常のシングルトンでは、既に他の場所でリクエストされている場合は既存のインスタンスを返す）。依存関係は解決され、変数がコントローラのコンストラクタに渡される（あるいは、指定したプロパティに代入される）。

```ts
constructor(private catsService: CatsService) {}
```

# スコープ
プロパイダは通常、アプリケーションのライフサイクルと同期したライフタイム（「スコープ」）を持っている。アプリケーションが立ち上げられた時、全ての依存関係は解決されなければならず、したがってすべてのプロパイダがインスタンス化されなければならない。同様に、アプリケーションが終了した際、各プロパイダは破棄される。しかしながらプロパイダのライフタイムをリクエストスコープにする方法もある。この手法についてはinjection-scopesの項を参照の事。

# カスタムプロパイダ
Nestには、プロパイダ間の依存関係を解決する制御反転（IoC）コンテナが組み込まれている。この機能は上記のインジェクション機能の根底にあるものだが、実際にはここまでの説明よりはるかに強力に機能する。`@Injectable()`デコレータはプロパイダを定義する唯一の方法ではなく、いわば氷山の一角に過ぎない。実のところNestの仕様者はプレーンな変数、クラス、非同期or同期のファクトリを使用できる。多くの実例はdependency-injectionの項を参照の事。

# オプショナルプロバイダ
時には解決する必要のない依存関係があるかもしれない。例えば、あるクラスが設定オブジェクトに依存しつつもデフォルト値を予め持っているような場合。その場合は依存関係は省略可能（オプショナル）となり、設定プロバイダがなくてもエラーは起きない。  
オプショナルプロバイダを作るには、コンストラクタのシグネイチャで`@Optional`デコレータを使用する。

```ts
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  constructor(@Optional() @Inject('HTTP_OPTIONS') private httpClient: T) {}
}
```

上記の例では`HTTP_OPTION`カスタムトークンを入れ込むため、カスタムプロパイダを使用している。先の例では、コンストラクタでクラスを介して依存関係を示す、コンストラクタベースのインジェクションを示してきた。カスタムプロパイダと関連するトークンについてはcustom-providersの項を参照の事。

# プロパティベースインジェクション
これまで使用してきた手法では、プロパイダはコンストラクタを介してインジェクションされる。非常に特殊なケースでは、プロパティベースのインジェクションが有用な場合もある。たとえば、もしトップレベルのクラスが１つまたは複数のプロバイダに依存している場合、コンストラクタからサブクラスの`super()`を呼び出してすべて渡していくのは非常に面倒な作業になる。それを避ける為には、プロパティレベルで`@Inject`デコレータを使用する。

```ts

import { Injectable, Inject } from '@nestjs/common';

@Injectable()
export class HttpService<T> {
  @Inject('HTTP_OPTIONS')
  private readonly httpClient: T;
}

```

>Warning  
>クラスがプロパイダを横断していない場合は、基本的にコンストラクタベースのインジェクションが望ましい。

# プロバイダの登録

プロパイダ（ここでは`CatsService`）を定義し、そのサービスのコンシューマ（ここでは`CatsController`）を持っているので、インジェクションを行えるようにサービスをNestに登録する。モジュールファイル（`app.module.ts`）を編集し、`@Module()`デコレータの`providers`配列にサービスを追加する。

```ts: app.module.ts  JS

import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}

```

これでNestが`CatsController`クラスの依存関係を解決できるようになった。  
ディレクトリは以下のようになる。

- src
    - cats
        - dto
            - create-cat.dto.ts
        - interfaces
            - cat.interface.ts
        - cats.service.ts
        - cats.controller.ts
 - app.module.ts
 - main.ts

# 手動インスタンス化
ここまではNestが持つ依存関係解決法のほぼ全てについて、自動的に処理する方法で説明してきた。しかし状況によっては、組み込みのインジェクションシステムの外で、手動でプロパイダを取得したり、インスタンス化したりする必要があるかもしれない。以下では、関連する２つのトピックについて説明する。  
既存のインスタンスを取得したり、プロパイダを動的にインスタンス化したりするには、モジュール参照を使用する。fundamentals-module-refの項を参照の事。  
コントローラのないスタンドアロンアプリケーションの場合や、ブートストラッピング中に設定サービスを利用する場合等、bootstrap()関数内でプロパイダを取得するには、standalone-applicationの項を参照の事。