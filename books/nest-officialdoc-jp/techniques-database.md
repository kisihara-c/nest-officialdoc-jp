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

このオプションを指定すると、`forFeature()`メソッドで登録された全てのエンティティが、設定オブジェクトの`entities`配列に自動で追加される。

>WARNING
>`autoLoadEntities`の設定によっては、`forFeature()`メソッドで登録されておらずエンティティから（リレーションシップを通して）参照されているだけのエンティティには適用されない。

## エンティティの定義を分ける

デコレータを使用すれば、モデル内でエンティティとその行を定義できる。だが、「[エンティティスキーマ](https://typeorm.io/#/separating-entity-definition)」を利用して、別のファイル内でエンティティとその行を定義したい人もいるだろう。

```ts
import { EntitySchema } from 'typeorm';
import { User } from './user.entity';

export const UserSchema = new EntitySchema<User>({
  name: 'User',
  target: User,
  columns: {
    id: {
      type: Number,
      primary: true,
      generated: true,
    },
    firstName: {
      type: String,
    },
    lastName: {
      type: String,
    },
    isActive: {
      type: Boolean,
      default: true,
    },
  },
  relations: {
    photos: {
      type: 'one-to-many',
      target: 'Photo', // the name of the PhotoSchema
    },
  },
});
```

>WARNING  
>`target`オプションを指定した場合、`name`オプションの値はターゲットクラスの名前と同じでなければならない。`target`を指定しない場合は任意の名前を使用可能。

Nestでは、Entityを置ける場所ならどこでも（wherever an Entity is expected）EntitySchemeインスタンスを使用可能。

例：

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UserSchema } from './user.schema';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  imports: [TypeOrmModule.forFeature([UserSchema])],
  providers: [UsersService],
  controllers: [UsersController],
})
export class UsersModule {}
```

## トランザクション

データベーストランザクションとは、データベース管理システム内でデータベースに対して実行される作業の単位を表す（[詳細](https://en.wikipedia.org/wiki/Database_transaction)）。そして、他のトランザクションとは独立し首尾一貫している信頼性の高い形で実行される。

[TypeORMのトランザクション](https://typeorm.io/#/transactions)を扱うためには、多くの異なる戦略がある。オススメは`QueryRunner`クラスだ。トランザクションについての完全なコントロールが行える。

まず通常の方法で`Connection`オブジェクトをクラスに注入する必要がある。

```ts
@Injectable()
export class UsersService {
  constructor(private connection: Connection) {}
}
```

>HINT  
>`Connection`クラスは`typeorm`パッケージからインポートする。

ではこのオブジェクトを使ってトランザクションを作成しよう。

```ts

async createMany(users: User[]) {
  const queryRunner = this.connection.createQueryRunner();

  await queryRunner.connect();
  await queryRunner.startTransaction();
  try {
    await queryRunner.manager.save(users[0]);
    await queryRunner.manager.save(users[1]);

    await queryRunner.commitTransaction();
  } catch (err) {
    // エラーが発生したので変更をロールバック
    await queryRunner.rollbackTransaction();
  } finally {
    // 手動でインスタンス化されたをreleaseする必要がある
    await queryRunner.release();
  }
}
```

>HINT  
>`connection`は`QueryRunner`の為だけに作られる事に注意。しかし、このクラスのテストのためには`Connection`オブジェクト全体をモックする必要がある（複数のメソッドを表出している）。そこで、ヘルパーファクトリクラス（例：`QueryRunnerFactory`）を使い、トランザクションを可能にするために必要なメソッドの特定のセットを持つ、インターフェイスを定義する事を勧める。このテクニックを使うと、メソッドのモックがすごく簡単になる。

もしくは`Connection`オブジェクトのトランザクションメソッドを使用してコールバックスタイルのアプローチを使用する事もできる（[詳細](https://typeorm.io/#/transactions/creating-and-using-transactions)）。

```ts
async createMany(users: User[]) {
  await this.connection.transaction(async manager => {
    await manager.save(users[0]);
    await manager.save(users[1]);
  });
}
```

デコレータを使ってトランザクションを制御する（`@Transaction()`や`TransactionManager()`）のは勧められない。

## サブスクライバ
TypeORMの[サブスクライバ](https://typeorm.io/#/listeners-and-subscribers/what-is-a-subscriber)を使うと特定のエンティティイベントをlistenできる。

```ts
import {
  Connection,
  EntitySubscriberInterface,
  EventSubscriber,
  InsertEvent,
} from 'typeorm';
import { User } from './user.entity';

@EventSubscriber()
export class UserSubscriber implements EntitySubscriberInterface<User> {
  constructor(connection: Connection) {
    connection.subscribers.push(this);
  }

  listenTo() {
    return User;
  }

  beforeInsert(event: InsertEvent<User>) {
    console.log(`BEFORE USER INSERTED: `, event.entity);
  }
}
```

> WARNING
> イベントサブスクライバはリクエストスコープ化できない。

`providers`配列に`UserSubscriber`クラスを追加しよう。

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UserSubscriber } from './user.subscriber';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService, UserSubscriber],
  controllers: [UsersController],
})
export class UsersModule {}
```

>HINT
>エンティティサブスクライバの詳細は[こちら](https://typeorm.io/#/listeners-and-subscribers/what-is-a-subscriber)

## マイグレーション
[マイグレーション](https://typeorm.io/#/migrations)はデータベース内の既存のデータを保持しつつ、アプリケーションのデータモデルと動悸させる為、データベーススキーマを段階的に更新する方法を提供する。マイグレーションの生成・実行・復帰の為に、TypeORMは専用の[CLI](https://typeorm.io/#/migrations/creating-a-new-migration)を提供する。

マイグレーションクラスはNestアプリケーションのソースコードから分離されている。そのライフサイクルはTypeORM CLIによって管理される。したがって、依存性のインジェクションやその他Nest特有の昨日をマイグレーションで利用する事はできない。マイグレーションの詳細は、[TypeORMのドキュメント](https://typeorm.io/#/migrations/creating-a-new-migration)にて。

## マルチプルデータベース

プロジェクトによっては複数のデータベース接続が必要となる事もある。このモジュールを使えば実現できる。複数の接続を使用するには、まず接続を作成する。この場合、接続の命名が**必須**となる。

独自のデータベースに保存されている`Album`エンティティがあるとする。

```ts
const defaultOptions = {
  type: 'postgres',
  port: 5432,
  username: 'user',
  password: 'password',
  database: 'db',
  synchronize: true,
};

@Module({
  imports: [
    TypeOrmModule.forRoot({
      ...defaultOptions,
      host: 'user_db_host',
      entities: [User],
    }),
    TypeOrmModule.forRoot({
      ...defaultOptions,
      name: 'albumsConnection',
      host: 'album_db_host',
      entities: [Album],
    }),
  ],
})
export class AppModule {}
```

>NOTICE  
>接続に名前を設定しない場合、`default`の接続に設定される。名前を設定しないまま、もしくは同じ名前で複数の接続を持つべきではない。

この時点で、`User`と`Albume`のエンティティが独自の接続で登録されている。`TypeOrmModule.forFeature()`メソッドと`@InjectRepository()`デコレータに対してどの接続を使用するか、指定しなければならない。接続名を渡さない場合は`default`に設定される。

```ts
@Module({
  imports: [
    TypeOrmModule.forFeature([User]),
    TypeOrmModule.forFeature([Album], 'albumsConnection'),
  ],
})
export class AppModule {}
```

また、与えられた接続の`Connection`や`EntityManager`をインジェクションする事もできる。

```ts
@Injectable()
export class AlbumsService {
  constructor(
    @InjectConnection('albumsConnection')
    private connection: Connection,
    @InjectEntityManager('albumsConnection')
    private entityManager: EntityManager,
  ) {}
}
```

プロバイダに任意のConnectionをインジェクションする事も可能だ。

```ts
@Module({
  providers: [
    {
      provide: AlbumsService,
      useFactory: (albumsConnection: Connection) => {
        return new AlbumsService(albumsConnection);
      },
      inject: [getConnectionToken('albumsConnection')],
    },
  ],
})
export class AlbumsModule {}
```

## テスト
アプリケーションの単体テストを行う場合、通常はデータベースへの接続を避け、テストスイートを独立させて実行プロセスを可能な限り保ちたくなる。しかし我々のクラスが接続インスタンスから引き出されるリポジトリに依存している場合もある。どうすればいいだろう？　解決策はモックリポジトリを作成する事だ。[カスタムプロバイダ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-customproviders)を設定しよう。登録された各リポジトリは自動的に[<EntytyName>Repository]として表出する。

`@nestjs/typeorm`パッケージは与えられたエンティティに基づいて用意されたトークンを返す`getRepositoryToken()`関数を持っている。

```ts
@Module({
  providers: [
    UsersService,
    {
      provide: getRepositoryToken(User),
      useValue: mockRepository,
    },
  ],
})
export class UsersModule {}
```

これで、`mockRepository`が`UsersRepository`の代替として使用されるようになる。任意のクラスが`@InjectRepository()`デコレータを使用して`UsersRepository`を要求すると、Nestは登録された`mockRepoitory`オブジェクトを使用する。

## カスタムリポジトリ
TypeORMは**カスタムリポジトリ**という機能を提供している。基礎となるリポジトリクラスを拡張したり、いくつかの特別なメソッドを使って機能強化できる。詳細は[こちら](https://typeorm.io/#/custom-repository)。

カスタムリポジトリを作るためには、`@EntityRepository()`デコレータを使い、`Repository`クラスを拡張する。

```ts 
@EntityRepository(Author)
export class AuthorRepository extends Repository<Author> {}
```

>HINT
>`@EntityRepository()`、`Repository`はそれぞれ`typeorm`パッケージからインポートしている。

クラスを作成したら、次はインスタンス化の責任をNestに委譲する。そのためには、`TypeOrm.forFeature()`メソッドに`AuthorRepository`クラスを渡す必要がある。

```ts
@Module({
  imports: [TypeOrmModule.forFeature([AuthorRepository])],
  controller: [AuthorController],
  providers: [AuthorService],
})
export class AuthorModule {}
```

後は、以下のような構成でリポジトリをインジェクションするだけだ。

```ts
@Injectable()
export class AuthorService {
  constructor(private authorRepository: AuthorRepository) {}
}
```

## Asyncの設定
リポジトリモジュールのオプションを静的に渡すのではなく、非同期に渡したい場合があるかもしれない。この場合は`forRootAsync()`メソッドを使用する。asyncの設定に対して複数の方法を提供するメソッドだ。

まずファクトリー関数を使う方法がある。

```ts
TypeOrmModule.forRootAsync({
  useFactory: () => ({
    type: 'mysql',
    host: 'localhost',
    port: 3306,
    username: 'root',
    password: 'root',
    database: 'test',
    entities: [__dirname + '/**/*.entity{.ts,.js}'],
    synchronize: true,
  }),
});
```

ファクトリーは他の非同期プロバイダと同じように動作する（例：非同期にでき、依存性のインジェクションが可能）。

```ts
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: (configService: ConfigService) => ({
    type: 'mysql',
    host: configService.get('HOST'),
    port: +configService.get<number>('PORT'),
    username: configService.get('USERNAME'),
    password: configService.get('PASSWORD'),
    database: configService.get('DATABASE'),
    entities: [__dirname + '/**/*.entity{.ts,.js}'],
    synchronize: true,
  }),
  inject: [ConfigService],
});
```

あるいは、`useClass`構文を使う事もできる。

```ts
TypeOrmModule.forRootAsync({
  useClass: TypeOrmConfigService,
});
```

上記のコードでは、`TypeOrmModule`内に`TypeOrmConfigService`をインスタンス化し、それを使用して`createTypeOrmOptions()`を呼び出してオプションオブジェクトを提供している。これは、以下に示すように、`TypeOrmConfigService`が`TypeOrmOptionsFactory`インターフェイスを実装しなければならない事を意味する。

```ts
@Injectable()
class TypeOrmConfigService implements TypeOrmOptionsFactory {
  createTypeOrmOptions(): TypeOrmModuleOptions {
    return {
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [__dirname + '/**/*.entity{.ts,.js}'],
      synchronize: true,
    };
  }
}
```

`TypeOrmModule`内での`TypeOrmConfigService`の生成を止めて別のモジュールからインポートしたプロバイダを使用するには、`useExisting`構文を使用する。

```ts
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

このコードは`useClass`と同様に動作する。重要なのは、`TypeOrmModule`がインポートされたモジュールを検索し、既存の`ConfigService`を再利用する事だ。

>HINT
>`name`プロパティが`useFactory`、`useClass`、`useValue`プロパティと同じレベルで定義されている事を確認の事。これにより、Nestは適切なインジェクショントークンの下で適切に接続を登録できる。

## 例
動くサンプルは[こちら](https://github.com/nestjs/nest/tree/master/sample/05-sql-typeorm)

## Sequelizeのインテグレーション