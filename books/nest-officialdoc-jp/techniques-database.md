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