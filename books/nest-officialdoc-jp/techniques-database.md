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