---
title: techniques-database
---

# データベース

Nestはデータベース不可知であり、任意のSQLやNoSQLと簡単に統合できる。好みに応じて、いくらかの選択肢がある。最も一般的なレベルでNestをデータベースに接続するなら、[Express](https://expressjs.com/en/guide/database-integration.html)やFastifyと同様、データベース用の適切なNode.jsドライバを読み込むだけだ。

また、より抽象度の高い操作を行う為に、[Sequelize](https://sequelize.org/)（下記でも統合方法を説明している）、[Knex.js](https://knexjs.org/)（[チュートリアル](https://dev.to/nestjs/build-a-nestjs-module-for-knex-js-or-other-resource-based-libraries-in-5-minutes-12an)）、[TypeORM](https://github.com/typeorm/typeorm)、[Prisma](https://github.com/prisma/prisma)（recipeチャプターで詳細説明）等の汎用的Node.jsデータベース統合ライブラリ・ORMをダイレクトに使用する事もできる。

便利が良いように、NestはTypeORMとSequelize、Mongooseとの緊密なインテグレーションを提供しており、それぞれ`@nestjs/typeorm`、`@nestjs/sequelize`、`@nestjs/mongoose`パッケージですぐに利用できる。これらの統合により、モデル/リポジトリのインジェクション、テスト可能性、非同期設定等NestJS特有の機能が追加され、選択したデータベースへのアクセスをより簡単に行える。

## TypeORMのインテグレーション

SQLやNoSQLデータベースとの統合のために、Nestは`@nestjs/typeorm`パッケージを提供している。NestがTypeORMを使用しているのは、TypeScriptで利用できる最も成熟したORMだからだ。TypeScriptで書かれているがゆえに、Nestフレームワークと上手く統合できる。

まず必要な依存関係をインストールしよう。この章では一般的な[MySQL](https://www.mysql.com/)リレーショナルDBMSでデモを行うが、TypeORMはPostgreSQL、Oracle、Microsoft SQL Server、SQLite、さらにはMongoDBのようなNoSQLデータベースなど、多くのリレーショナル・データベースをサポートしている。この章の手順はTypeORMでサポートされているどのデータベースでも同じだ。必要なのは、選択したデータベースに関連する、クライアントAPIをインストールする事だけ。

```
$ npm install --save @nestjs/typeorm typeorm mysql2
```

インストールが完了したら、ルートのAppModuleにTypeOrmModuleをインポートする。

```ts :app.module.ts 
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [],
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

> Warning
> 静的なglob path（例：`dist/**/*.entity{ .ts,.js}`）は[Webpack](https://webpack.js.org/)では正しく動作しない。

> HINT
> `ormconfig.json`ファイルは`typeorm`ライブラリによって読み込まれることに注意してほしい。したがって、上記で説明した追加のプロパティ（`forRoot()`メソッドによって内部的にサポートされているもの。例えば、`autoLoadEntities`や`retryDelay`）は適用されない。幸い、TypeORMは接続オプションを`ORMconfig`ファイルや環境変数から読み込む`getConnectionOptions`関数を提供している。この関数を使う事でも、以下のように設定ファイルからNest特有のオプションを設定できる。
> ```ts
> TypeOrmModule.forRootAsync({
> useFactory: async () =>
>   Object.assign(await getConnectionOptions(), {
>     autoLoadEntities: true,
>   }),
>})
>```

これが完了すると、TypeORMの`Connection`と`EntytyManager`オブジェクトはプロジェクト全体にインジェクション出来るようになる（モジュールをインポートする必要はない）。

```ts :app.module.ts
import { Connection } from 'typeorm';

@Module({
  imports: [TypeOrmModule.forRoot(), UsersModule],
})
export class AppModule {
  constructor(private connection: Connection) {}
}
```

## Repositoryパターン

TypeORMはRepositoryパターンをサポートしているので、各エンティティは独自のリポジトリを持っている。データベース接続を通して取得できる。

例を続ける為には最低１つのエンティティが必要だ。`User`エンティティを定義しよう。

```ts :user.entity.ts 
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ default: true })
  isActive: boolean;
}
```

> HINT
> エンティティについては、[TypeORM](https://typeorm.io/#/entities)のドキュメントを参照のこと。

`User`エンティティファイルは、`users`ディレクトリにある。このディレクトリには`UsersModule`に関連するすべてのファイルが含まれている。モデルファイルをどこに保存するかは自由に決められるが、モデルファイルはその**ドメイン**の近く、対応するモジュールディレクトリに作成する事を勧める。

`User`エンティティの使用を開始するには、モジュールの`forRoot()`メソッドオプションの`entities`配列に挿入してTypeORMに読み込ませる必要がある（静的なglob　pathを使っている場合を除く）。

```ts :app.module.ts 
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './users/user.entity';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [User],
      synchronize: true,
    }),
  ],
})
export class AppModule {}
```

次に、`UsersModule`を見てみよう。

```ts: users.module.ts 
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

このモジュールは`forFeature()`メソッドを用いて、現在のスコープに登録されているリポジトリを定義する。結果、`@InjectRepository()`デコレータを使用して、`UsersRepository`を`UsersService`にインジェクションする事ができる。

```ts :users.service.ts 
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  findAll(): Promise<User[]> {
    return this.usersRepository.find();
  }

  findOne(id: string): Promise<User> {
    return this.usersRepository.findOne(id);
  }

  async remove(id: string): Promise<void> {
    await this.usersRepository.delete(id);
  }
}
```

> NOTICE
> ルートの`AppModule`に`UserModule`をインポートする事を忘れない事。

`TypeOrmModule.forFeature`をインポートしたモジュール以外のリポジトリを使用したい場合は、生成されたプロバイダを再インポートする必要がある。これは、以下のようにモジュール全体をエクスポートする事で行える。

```ts :users.module.ts 
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  exports: [TypeOrmModule]
})
export class UsersModule {}
```

さて、`UserHttpModule`で`UsersModule`をインポートすると、後者に属するプロバイダで`@InjectRepository(User)`を使うことができる。

```ts :users-http.module.ts 

import { Module } from '@nestjs/common';
import { UsersModule } from './users.module';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';

@Module({
  imports: [UsersModule],
  providers: [UsersService],
  controllers: [UsersController]
})
export class UserHttpModule {}
```

## リレーション
リレーションとは、２つ以上のテーブル間の関係性の事だ。リレーションは各テーブルの共通フィールドに基づいており、多くの場合主キーと外部キーが関係している。

関係には３つのタイプがある。


|||
| ---- | ---- |
|一対一|主テーブルのすべての行が、外部テーブルの関連行を一つだけ持っている。このタイプのリレーションを定義するには`@OneToOne()`デコレータを使用する。|
|一対多/多対一|主テーブル内のすべての行が外部テーブル内に、１つ以上の関連行を持つ。`@OneToMany()`、`@ManyToOne()` を使う。|
|多対多|主テーブルの全ての行が外部テーブルの中に多くの関連行を持ち、外部テーブルの全ての行が主テーブルの中に多くの関連行を持つ。`@ManyToMany`デコレータを使用する。|

エンティティ内の関係を定義するには、対応するデコレータを使用する。例えば各ユーザが複数の写真を持つ場合、`@OneToMany()`デコレータを使用する。

```ts :user.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, OneToMany } from 'typeorm';
import { Photo } from '../photos/photo.entity';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  firstName: string;

  @Column()
  lastName: string;

  @Column({ default: true })
  isActive: boolean;

  @OneToMany(type => Photo, photo => photo.user)
  photos: Photo[];
}
```

>HINT  
>TypeORMのリレーションについての詳細は、TypeORMの[ドキュメント](https://typeorm.io/#/relations)を参照の事。

## オートロードエンティティ

接続オプションの`entities`配列にエンティティを手動で追加するのは面倒だ。さらに、ルートモジュールからエンティティを参照すると、アプリケーションのドメイン境界が破られて、アプリケーションの他部分に実装の詳細が漏れる原因となる。この問題を解決するために、静的なglob pathを使用できる。（例：`dist/**/*.entity{ .ts,.js}`）

しかしながら、glob pathはwebpackではサポートされていない。アプリケーションをmonorepoの仲で構築している場合は使えない。代わりに別の解決策が用意されている。エンティティをおーとろーどするには、以下に示すように、設定オブジェクト（`forRoot()`メソッドに渡される）の`autLoadEntities`プロパティを`true`に設定する。

```ts :app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      ...
      autoLoadEntities: true,
    }),
  ],
})
export class AppModule {}
```