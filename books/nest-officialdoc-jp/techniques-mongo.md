---
title: techniques-mongo
---

# Mongo

Nestは[MongoDB](https://www.mongodb.com/)と２つの方法で連携できる。前章で紹介した[TypeORM](https://github.com/typeorm/typeorm)の組み込みモジュール（MongoDB用のコネクタがある）を使うか、MongoDBのオブジェクトモデリングツールとして最も人気のある[Mongoose](https://mongoosejs.com/)を使うか。本章では後者について、専用の`@nestjs/mongoose`パッケージを使って説明する。

まず必要なdependenciesをインストールしよう。

```
$ npm install --save @nestjs/mongoose mongoose
```

インストールが終わったら、ルートの`AppModule`に`MongooseModule`をインストールする。

```ts :app.module.ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [MongooseModule.forRoot('mongodb://localhost/nest')],
})
export class AppModule {}
```

`fooRoot()`メソッドは、[ここ](https://mongoosejs.com/docs/connections.html)で説明しているMongooseパッケージの`mongoose.connect()`と同じ設定オブジェクトを受け取る。

## モデルインジェクション

Mongooseにおいては、全てが[スキーマ](https://mongoosejs.com/docs/guide.html)から派生する。各スキーマはMongoDBのコレクションに対応しており、そのコレクション内のドキュメントの形状を定義する。スキーマはモデルを定義するために使われる。モデルは、MongoDBデータベースからドキュメントを生成したり読み込んだりする役割を果たす。

スキーマはNestJSのデコレータを使って作る事もできるし、Mongoose自体に手動で作らせる事もできる。デコレータを使うと定型文が大幅に減り、コード全体が読みやすくなる。

`CatSchema`を定義してみよう。

```ts :schemas/cat.schema.ts 
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

export type CatDocument = Cat & Document;

@Schema()
export class Cat {
  @Prop()
  name: string;

  @Prop()
  age: number;

  @Prop()
  breed: string;
}

export const CatSchema = SchemaFactory.createForClass(Cat);
```

>HINT  
>`nestjs/mongoose`の`DeifinitiosFactory`クラスを使えば生のスキーマ定義を生成できる。そして用意したメタデータに基づいて生成したスキーマ定義を手動で修正できる。これはデコレータで全てを表現するのが難しいようなエッジケースに有効だ。

`@Schema()`デコレータは、クラスをスキーマ定義として印付ける。これは、`Cat`クラスをMongoDBの同名のコレクションに対応させる。ただし最後に"s"を追加する為、最終的なmongoのコレクション名は`cats`となる。このデコレータはスキーマオプションオブジェクトという省略可能な引数をひとつだけ受け取る。これは通常の`mongoose.Schema`クラスのコンストラクタ（例：`new mongoose.Schema(_,options)`）の第二引数として渡すオブジェクトと考えてほしい。利用可能なスキーマオプションについては[こちら](https://mongoosejs.com/docs/guide.html#options)。

`@Prop()`デコレータは、ドキュメントのプロパティを定義する。例えば上記のスキーマ定義では、`name`、`age`、`breed`の３つのプロパティを定義している。これらのプロパティの[スキーマタイプ](https://mongoosejs.com/docs/schematypes.html)は、TypeScriptのメタデータ（及びリフレクション）機能によって自動推論される。しかし暗黙のうちに型が反映されない複雑な状況（配列やネストしたオブジェクト構造等）では、以下のように型を明示する必要がある。

```ts
@Prop([String])
tags: string[];
```

また、`@Prop()`デコレータはオプションオブジェクトの引数を受け取る（利用可能なオプションは[こちら](https://mongoosejs.com/docs/schematypes.html#schematype-options)）。これにより、プロパティが必須か否か示したり、デフォルト値を指定したり、不変な値である事を示したりできる。例：

```ts
@Prop({ required: true })
name: string;
```

また、他のモデルとの関係を指定して後で入力する場合にも、`@Prop()`デコレータを使用する事ができる。例えば、`Cat`が`Owner`を持ち、それが`Owners`という別のコレクションに格納されている場合、プロパティは`type`と`ref`を持つ必要がある。例：

```ts
import * as mongoose from 'mongoose';
import { Owner } from '../owners/schemas/owner.schema';

// クラス定義の中
@Prop({ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' })
owner: Owner;
```

複数のownersがある場合、プロパティ設定はこうなる。

```ts
@Prop({ type: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' }] })
owner: Owner[];
```

そして最後に、**生**のスキーマ定義をデコレータに渡すこともできる。これは例えば、プロパティがネストされたオブジェクト（クラスとして定義されていないもの）を表している場合に便利だ。`@nestjs/mongoose`パッケージの`raw()`関数を以下のように使う。

```ts
@Prop(raw({
  firstName: { type: String },
  lastName: { type: String }
}))
details: Record<string, any>;
```

**デコレータを使いたくない**場合、手動でスキーマを定義する事もできる。例：

```ts
export const CatSchema = new mongoose.Schema({
  name: String,
  age: Number,
  breed: String,
});
```

`cat.schema`は`cats`ディレクトリ内のフォルダに格納される。`CatModule`も同じ場所で定義される。スキーマファイルはどこにでも保存できるが、関連するドメインオブジェクトの近く、適切なモジュールディレクトリに保存する事を勧める。

`CatsModule`を見てみよう。

```ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';
import { Cat, CatSchema } from './schemas/cat.schema';

@Module({
  imports: [MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }])],
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

`MongooseModule`には`forFeature()`メソッドが用意されていて、どのモデルを現在のスコープに登録するのか等モジュールの設定を行う。別のモジュールでもモデルを使いたい場合は、`CatsModule`の`exports`セクションに`MongooseModule`を追加して、別のモジュールで`CatsModule`を`import`する。

スキーマを登録したら、`@InjectModel()`デコレータを使って`CatsService`に`Cat`モデルをインジェクションする。

```ts :cats.service.ts 
import { Model } from 'mongoose';
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Cat, CatDocument } from './schemas/cat.schema';
import { CreateCatDto } from './dto/create-cat.dto';

@Injectable()
export class CatsService {
  constructor(@InjectModel(Cat.name) private catModel: Model<CatDocument>) {}

  async create(createCatDto: CreateCatDto): Promise<Cat> {
    const createdCat = new this.catModel(createCatDto);
    return createdCat.save();
  }

  async findAll(): Promise<Cat[]> {
    return this.catModel.find().exec();
  }
}
```

## 接続

状況によってはネイティブの[Mongoose Connection](https://mongoosejs.com/docs/api.html#Connection)オブジェクトにアクセスする必要がある。たとえば、接続オブジェクトに対してネイティブAPIコールをしたい場合等。次のように`@InjectConnection()`デコレータを使うとMongoose Connectionをインジェクションする事ができる。

```ts
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection() private connection: Connection) {}
}
```

## マルチプルデータベース

プロジェクトによっては、複数のデータベース接続が必要な場合がある。これもこのモジュールで実現可能。まず接続を作成しよう。この場合名前をつける必要がある。

```ts :app.module.ts 
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionName: 'cats',
    }),
    MongooseModule.forRoot('mongodb://localhost/users', {
      connectionName: 'users',
    }),
  ],
})
export class AppModule {}
```

>NOTICE  
>名無しの接続や複数の同名の接続があると上書きされる。

このセットアップでは、どの接続を使うか`MongooseModule.foreFeature()`関数で指定する必要がある。

```ts
@Module({
  imports: [
    MongooseModule.forFeature([{ name: Cat.name, schema: CatSchema }], 'cats'),
  ],
})
export class AppModule {}
```

また、指定した接続に対して`Connection`をインジェクションする事もできる。

```ts
import { Injectable } from '@nestjs/common';
import { InjectConnection } from '@nestjs/mongoose';
import { Connection } from 'mongoose';

@Injectable()
export class CatsService {
  constructor(@InjectConnection('cats') private connection: Connection) {}
}
```

指定した`Connection`をカスタムプロバイダ（例：ファクトリプロバイダ）にインジェクションするには、引数に`Connection`の名前を入れこんで`getConnectionToken()`関数を使用する。

```ts
{
  provide: CatsService,
  useFactory: (catsConnection: Connection) => {
    return new CatsService(catsConnection);
  },
  inject: [getConnectionToken('cats')],
}
```

## フック（ミドルウェア）

ミドルウェア（プリ・ポストフックとも呼ばれる）は、非同期関数の実行中に制御を渡す関数。ミドルウェアはスキーマレベルで指定され、プラグインを書く際に有用だ（[ソース](https://mongoosejs.com/docs/middleware.html)）。モデルをコンパイルした後で`pre()`や`post()`を呼んでもMongooseでは上手くいかない。モデルの登録前にフックを登録するには、`MongooseModule`の`foreFeartureAsync()`メソッドをファクトリプロバイダ（つまり`useFactory`）と一緒に使う。こうすれば、スキーマオブジェクトにアクセスしてから`pre()`、`post()`メソッドでフックを登録できる。例：

```ts
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatsSchema;
          schema.pre('save', function() { console.log('Hello from pre save') });
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

他の[ファクトリープロバイダ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-customproviders#%E3%83%95%E3%82%A1%E3%82%AF%E3%83%88%E3%83%AA%E3%83%BC%E3%83%97%E3%83%AD%E3%83%90%E3%82%A4%E3%83%80%E3%80%81usefactory)と同様、このファクトリー関数は非同期で使えるし、依存関係をインジェクションできる。

```ts
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        imports: [ConfigModule],
        useFactory: (configService: ConfigService) => {
          const schema = CatsSchema;
          schema.pre('save', function() {
            console.log(
              `${configService.get('APP_NAME')}: Hello from pre save`,
            ),
          });
          return schema;
        },
        inject: [ConfigService],
      },
    ]),
  ],
})
export class AppModule {}
```

## プラグイン

スキーマに対して[プラグイン](https://mongoosejs.com/docs/plugins.html)を登録するには、`forFeatureAsync()`メソッドを使う。

```ts
@Module({
  imports: [
    MongooseModule.forFeatureAsync([
      {
        name: Cat.name,
        useFactory: () => {
          const schema = CatsSchema;
          schema.plugin(require('mongoose-autopopulate'));
          return schema;
        },
      },
    ]),
  ],
})
export class AppModule {}
```

全てのスキーマに対して一度でプラグインを登録する場合は`Connection`オブジェクトの`.plugin()`メソッドを呼び出す。モデルが作成される前に接続にアクセスする必要があるので、`connectionFactory`を使用する。

```ts :app.module.ts
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/test', {
      connectionFactory: (connection) => {
        connection.plugin(require('mongoose-autopopulate'));
        return connection;
      }
    }),
  ],
})
export class AppModule {}
```

## ディスクリミネータ

[ディスクリミネータ](https://mongoosejs.com/docs/discriminators.html)とはスキーマを継承する仕組みのことだ。使用すれば、同じMongoDBのコレクションを基礎として、その上に重複するスキーマを持つ複数のモデルを持つことが出来る（have multiple models with overlapping schemas on top of the same underlying MongoDB collection）。

さまざまな種類のイベントを一つのコレクションで管理したいとしよう。全てのイベントにはタイムスタンプがある。

```ts :event.schema.ts 
@Schema({ discriminatorKey: 'kind' })
export class Event {
  @Prop({
    type: String,
    required: true,
    enum: [ClickedLinkEvent.name, SignUpEvent.name],
  })
  kind: string;

  @Prop({ type: Date, required: true })
  time: Date;
}

export const EventSchema = SchemaFactory.createForClass(Event);
```

>HINT  
>mongooseがディスクリミネータモデルを区別する方法は「ディスクリミネータキー」、デフォルトの値は`_t`だ。Mongooseはスキーマに_tという名前の文字列pathを追加して、このドキュメントがどのディスクリミネータのインスタンスであるかを追跡する為に使う。`discriminatorKey`オプションで判別用のpathを定義する事もできる。

`SignedUpEvent`、`ClickedLinkEvent`インスタンスは、一般的なイベントと同じコレクションに格納される。

では、`ClickedLinkEvent`クラスを以下のように定義してみよう。

```ts :click-link-event.schema.ts 
@Schema()
export class ClickedLinkEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  url: string;
}

export const ClickedLinkEventSchema = SchemaFactory.createForClass(ClickedLinkEvent);
```

`SignUpEvent`クラスも。

```ts :sign-up-event.schema.ts 
@Schema()
export class SignUpEvent {
  kind: string;
  time: Date;

  @Prop({ type: String, required: true })
  user: string;
}

export const SignUpEventSchema = SchemaFactory.createForClass(SignUpEvent);
```

以上を前提に、`discriminators`オプションを使って使いたいスキーマ用のディスクリミネータを登録する。これは`MongooseModule.forFeature`でも`MongooseModule.forFeatureAsync`でも動く。

```ts :event.module.ts 
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forFeature([
      {
        name: Event.name,
        schema: EventSchema,
        discriminators: [
          { name: ClickedLinkEvent.name, schema: ClickedLinkEventSchema },
          { name: SignUpEvent.name, schema: SignUpEventSchema },
        ],
      },
    ]),
  ]
})
export class EventsModule {}
```

## テスト
アプリケーションのユニットテストを行う場合、通常はデータベースへの接続を避け、テストスイートの設定を単純にし、実行速度を早くしたいと考える。けれど我々の作ったクラスがconnectionインスタンスから引き出されたモデルに依存しているかもしれない。これらのクラスをどうテストすればよいだろう。解決策はモックモデルを作ることだ。

`@nestjs/mongoose`パッケージは、トークン名に基づいて準備されたインジェクショントークンを返す`getModelToken()`関数を公開している。`useClass`、`useValue`、`useFactory`などの標準的なカスタムプロバイダのテクニックを使って、簡単にモックの実装を提供できる。例：

```ts
@Module({
  providers: [
    CatsService,
    {
      provide: getModelToken(Cat.name),
      useValue: catModel,
    },
  ],
})
export class CatsModule {}
```

この例では、いつ誰が`@InjectModel()`デコレータを使って`Model<cat>`をインジェクションしても、ハードコードされた`catModel`（オブジェクトのインスタンス）が提供される。

## 非同期設定

モジュールのオプションを静的にではなく非同期的に渡す必要がある場合、`forRootAsync()`メソッドを使用する。ほとんどのダイナミックモジュールと同様に、Nestは非同期設定を扱うためのテクニックをいくつか提供している。

ひとつはファクトリ関数を使う方法だ。

```ts
MongooseModule.forRootAsync({
  useFactory: () => ({
    uri: 'mongodb://localhost/nest',
  }),
});
```

他の[ファクトリープロバイダ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-customproviders#%E3%83%95%E3%82%A1%E3%82%AF%E3%83%88%E3%83%AA%E3%83%BC%E3%83%97%E3%83%AD%E3%83%90%E3%82%A4%E3%83%80%E3%80%81usefactory)のように、このファクトリー関数も非同期にできる。

```ts
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    uri: configService.get<string>('MONGODB_URI'),
  }),
  inject: [ConfigService],
});
```

別の方法として、クラスを使って`MongooseModule`を設定する事もできる。

```ts
MongooseModule.forRootAsync({
  useClass: MongooseConfigService,
});
```

上記のコードでは`MongooseModule`の中で`MongooseConfigService`をインスタンス化し、それを使って必要なオプションオブジェクトを作る。この例では、`MongooseConfigService`は`MongooseOptionsFactory`インターフェイスを実装する必要がある事に注意（下記参照）。`MongooseModule`は、提供されたクラスのインスタンス化されたオブジェクトに対して`createMongooseOptions()`メソッドを呼び出す。

```ts
@Injectable()
class MongooseConfigService implements MongooseOptionsFactory {
  createMongooseOptions(): MongooseModuleOptions {
    return {
      uri: 'mongodb://localhost/nest',
    };
  }
}
```

`MongooseModule`の中にprivateなコピーを作るのではなく、既存のオプションプロバイダを再利用したい場合は`useExisting`構文を使う。

```ts
MongooseModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

## サンプル
実際に動くサンプルは[こちら](https://github.com/nestjs/nest/tree/master/sample/06-mongoose)