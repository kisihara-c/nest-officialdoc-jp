---
title: techniques-mongo
---

# Mongo

Nestは[MongoDB](https://www.mongodb.com/)と２つの方法で連携できる。前章で紹介した[TypeORM](https://github.com/typeorm/typeorm)の組み込みモジュールを使うか（MongoDB用のコネクタを持つ）、MongoDBのオブジェクトモデリングツールとして最も人気のある[Mongoose](https://mongoosejs.com/)を使うか。本章では後者について、専用の`@nestjs/mongoose`パッケージを使って説明する。

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

スキーマはNestJSのデコレータを使って作る事もできるし、Mongoose自体に手動で作らせる事もできる。デコレータを使うと定型文が大幅にヘリ、コード全体が読みやすくなる。

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
>`nestjs/mongoose`の`DeifinitionFactory`クラスを使えば生のスキーマ定義を生成できる。用意したメタデータに基づいて生成したスキーマ定義を手動で修正できる。これはデコレータで全てを表現するのが難しいようなエッジケースに有効だ。

`@Schema()`デコレータは、クラスをスキーマ定義として印付ける。これは、`Cat`クラスをMongoDBの同名のコレクションに対応させる。あだし最後に"s"を追加する為、最終的なmongoのコレクション名は`cats`となる。このデコレータはスキーマオプションオブジェクトという省略可能な引数をひとつだけ受け取る。これは通常の`mongoose.Schema`クラスのコンストラクタ（例：`new mongoose.Schema(_,options)`）の第二引数として渡すオブジェクトと考えてほしい。利用可能なスキーマオプションについては[こちら](https://mongoosejs.com/docs/guide.html#options)。

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

// inside the class definition
@Prop({ type: mongoose.Schema.Types.ObjectId, ref: 'Owner' })
owner: Owner;
```

複数のオーナーがある場合、プロパティ設定はこうなる。

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