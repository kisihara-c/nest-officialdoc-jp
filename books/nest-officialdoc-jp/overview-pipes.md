---
title: overview-pipes
---

# パイプ
パイプは`@Injectable()`デコレータで装飾されたクラスだ。`PipeTransform`インターフェイスの実装が必要となる。

![画像](https://docs.nestjs.com/assets/Pipe_1.png)

パイプには２つのユースケースがある。
- 変換：入力データを希望の形に変換する（例：文字列→整数）
- 検証：入力データを評価し、データが正しければそのまま実行を続け、データが間違っていれば例外をthrowする

いずれもパイプはコントローラのルートハンドラによって処理される`arguments`を操作する。Nestはメソッドが呼び出される直前にパイプを挿入し、パイプはメソッドの引数を受け取って処理する。変換・検証はその時点で行われ、最後に変換された（かもしれない）引数でルートハンドラが呼び出される。  
Nestには組み込みのパイプが多数用意されており、すぐに使うことができる。独自のカスタムパイプを構築する事もできる。この章では組み込みパイプを紹介し、ルートハンドラにバインドする方法を示す。次に、いくつかカスタムパイプを見てみて、一からパイプを作る方法を示す。

>Hint  
>パイプは例外範囲内で動作する。つまり、パイプが例外を投げた場合、例外レイヤ（グローバル例外フィルタと現在のコンテキストに適用されているすべての例外フィルタ）で処理される。その事から、パイプで例外がthrowされた時、それ以降のコントローラは実行されない。システムの境界で外部の情報元から入ってくるデータを検証する為の、ベストプラクティスとなるテクニックだといえる。

## 組み込みパイプ

以下6つのパイプが即座に使用可能だ。

- `ValidationPipe`
- `ParseIntPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`
- `DefaultValuePipe`

上記は`@nestjs/common`パッケージからエクスポートされる。  
`ParseIntPipe`の使い方を簡単に確認する。これは**変換**パイプのユースケースで、パイプはメソッドハンドラの変数がJavaScriptの整数に変換されることを保証する。変換に失敗した場合例外を投げる。この章の後半では、`ParseIntPipe`のシンプルなカスタム例を紹介する。下記のテクニック例は、この章では`Parse*Pipe`とも呼ぶ他の組み込み変換パイプ――`ParseBoolPipe`、`ParseArrayPipe`、`ParseUUIDPipe`にも適用される。

## パイプのバインディング
パイプを使用するには、パイプクラスのインスタンスを適切なコンテキストにバインドする必要がある。`ParseIntPipe`の例では、パイプを特定のルートハンドラメソッドに関連付けて、そのメソッドが呼ばれる前に実行されるようにしたい。次のような構文を使用する。メソッド変数のレベルでパイプをバインドするものだ。

```ts
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

この構文で次の２つのいずれかが保証される。（我々が`this.catsService.findOne()`を呼ぶ時期待したように）`findOne()`メソッドで受け取る数値が数値である。あるいは、ルートハンドラが呼ばれる前に例外がthrowされる。  
例えば、こんな感じでルートが呼ばれたとする。

```
GET localhost:3000/abc
```

Nestはこんな例外を投げる。

```
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```

結果、`findOne()`の本体の実行は阻害される。  
上の例ではインスタンスではなくクラス（`ParseIntPipe`）を渡している。パイプやガードもそうだし、代わりにin-place instanceも渡すことができる。お決まりのin-place instanceを渡すのは、以下のように補足を渡して組み込みパイプの動作をカスタマイズしたい場合に便利だ。

```ts
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

変換機能を持つ他のパイプ（全Parse*パイプ）をバインドしても、同様に動作する。これらのパイプは全て、ルート変数、クエリ文字列変数、リクエストの本文の情報の値を検証するコンテキストで動作する。  
例えばクエリ文字列変数の場合。

```ts
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```
`ParseUUIDPipe`を使って文字列パラメータを解析し、`UUID`かどうか検証する例を示す。

```ts
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
```

>HINT
>When using `ParseUUIDPipe()` you are parsing UUID in version 3, 4 or 5, if you only require a specific version of UUID you can pass a version in the pipe options. （※訳出できず…！　申し訳有りません）

上記では様々なParse*ファミリーの組み込みパイプのバインド例を見てきた。検証パイプのバインドは少し異なる。その事は続く章で議論していく。

>HINT  
>検証パイプの豊富な例は[techniques-validationの章](./techniques-validation)を参照の事。

## カスタムパイプ

前述の通り独自のカスタムパイプを構築する事ができる。Nestは堅牢な`ParseIntPipe`と`ValidationPipe`を組み込みで提供しているが、カスタムパイプの構築方法を見る為にそれぞれのシンプルなカスタム版をゼロから構築してみよう。  
まずはシンプルな`ValidationPipe`から始める。最初は単純に入力値を受け取らせ、すぐに同じ値を返させよう。ID関数のように動作する。

```ts :validation.pipe.ts 
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
```

>Hint
>`PipeTransform<T, R>`は全てのパイプで実装されなければならない汎用インターフェイスだ。入力値の型を示すTと、`transform()`メソッドの戻り値の型を示すRを使用する。

全てのパイプは`PipeTransform`インターフェイスの規約を満たす為に`transform()`メソッドを実装する必要がある。このメソッドは２つパラメータを持つ。

- `value`
- `metadata`

`value`は現在処理中のメソッド引数（ルートハンドリングメソッドに受信される前のもの）。`metadata`は現在処理中のメソッド引数のメタデータだ。以下のプロパティを持つ。

```ts
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```

これらのプロパティは、現在処理されている引数を説明している。

|||
| ---- | ---- |
|type|引数が本文（`@Body()`）、クエリ（`@Query()`）、param（`@Param()`）あるいはカスタムパラメータのいずれであるかを示す|
|metatype|引数のメタタイプを提供する。例えば、文字列等。注意として、ルートハンドラのメソッドシグネイチャで型宣言を省略するか、バニラJavaScriptを使用している場合値はundefinedとなる。|
|data|デコレータに渡される文字列（例えば`@Body(``string``)`）のようなもの。デコレータのカッコを空にした場合はundefinedとなる。|

>Warning  
>TypeScriptのインターフェイスはトランスパイル時に消える。よって、メソッド変数の型がクラスではなくインターフェイスとして宣言されている場合、メタタイプの値はObjectとなる。

## スキーマに基づいた検証
検証パイプをもう少し便利にしよう。`CatsController`の`create()`メソッドに注目する。ここで、来たポストの本体であるオブジェクトが有効か否か、サービスメソッドが起動する前に確かめられる事を確認したい。

```ts
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

`createCatDto`の本体により深く注目していく。型は`CreateCatDto`となる。

```ts :create-cat.dto.ts 
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

createメソッドに入ってくるリクエストに有効な本文が含まれている事を確認したい。その為に、`createCatDto`オブジェクトの３つのメンバを確認する必要がある。ルートハンドラメソッドの中で行う事はできるが、単一責任の原則（SRP）を破る為理想的ではない。  
検証用クラスを作成してタスクを委譲する方法もある。この場合、各メソッドの最初にこのバリデータを呼び出さなければならないデメリットがある。  
バリデーション用ミドルウェアを作る手はどうか。動く可能性はないとはいえないが、残念ながらアプリケーション全体の全てのコンテキストで使える汎用的なミドルウェアは作れない。ミドルウェアは、呼び出されるハンドラやそのパラメータを含む実行コンテキストを認識していないからだ。  
こういう時こそまさにパイプが求められるユースケースだ。検証用パイプを改良していこう。  

## オブジェクトスキーマの検証
オブジェクトの検証をクリーンでDRYな方法で行うには、いくつかのアプローチがある。一般的なアプローチの一つは、**スキーマベース**の検証を行う事だ。試してみよう。  
[Joi](https://github.com/sideway/joi)ライブラリを使用すると、読みやすいAPIを用いてまっすぐスキーマを作成する事ができる。Joiベースのスキーマを使った検証パイプを作ってみよう。  
パッケージのインストールから始める。

```
$ npm install --save joi
$ npm install --save-dev @types/joi
```

以下のコードサンプルでは、`constructor`の引数としてスキーマを受け取るシンプルなクラスを作成する。次に`schema.validate()`メソッドを適用し、指定したスキーマに対して入力された引数を検証する。  
先述通り、検証パイプは値をそのまま返すか例外を投げる。  
次のセクションでは、`@UsePipes()`デコレータを使用して指定したコントローラメソッドに適切なスキーマを提供する方法を見ていく。そうすれば、検証パイプをコンテキスト間で再利用できるようになる。

```ts
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { ObjectSchema } from 'joi';

@Injectable()
export class JoiValidationPipe implements PipeTransform {
  constructor(private schema: ObjectSchema) {}

  transform(value: any, metadata: ArgumentMetadata) {
    const { error } = this.schema.validate(value);
    if (error) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }
}
```

## 検証パイプのバインディング

上記で変換パイプ（`ParseIntPipe`や`Parse*`パイプ）をバインドする方法を確認した。  
検証パイプのバインドの方法も非常に簡単だ。  
今回はメソッド呼び出しの段階でパイプをバインドしたい。現在の例では、`JoiValidationPipe`を使う為に以下を行う必要がある。

1. `JoiValidationPipe`のインスタンスを作成する
2. パイプのクラスコンストラクタにコンテキスト固有のJoiスキーマを渡す
3. パイプをメソッドにバインドする

以下に示す通り`@UsePipe()`を使って進める。

```ts
@Post()
@UsePipes(new JoiValidationPipe(createCatSchema))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

>HINT  
>`@UsePipes()`デコレータは`@nestjs/common`からインポートされている。

## クラスバリデータ

>Warning  
>このセクションのテクニックはTypeScript必須となる。バニラJavaScriptでは行えない。

バリデーションテクニックの代替となる実装を見てみたい。  
Nestは[class-validator](https://github.com/typestack/class-validator)ライブラリと相性がいい。この強力なライブラリを使えば、デコレータベースの検証を行える。この手法はとても強力で、特にNestのパイプ機能と組み合わせると、処理中のプロパティのメタタイプにアクセスできるので非常に便利だ。まずパッケージをインストールしよう。

```
$ npm i --save class-validator class-transformer
```

インストールすると、`CreateCatDto`クラスにいくつかのデコレータを追加できる。この手法には大きな利点がある。`CreateCatDto`クラスはポストのbodyオブジェクトに対し、（複数のバリデーションクラスを作る必要はなく、）真実にして単一のソースであり続けることだ。

```ts :create-cat.dto.ts 
import { IsString, IsInt } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```

>HINT  
>クラスバリデータのデコレータをより知るには、[class-validatorのドキュメント](https://github.com/typestack/class-validator#usage)を参照のこと。

これらのアノテーションを使う`validationPipe`クラスができた。

```ts :validation.pipe.ts 
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToClass } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToClass(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

>NOTICE  
>上記では[class-transformer](https://github.com/typestack/class-transformer)ライブラリを使っている。class-validatorと同じ作者によるライブラリで、結果としてとても相性がいい。

コードを見ていこう。まず、`transform()`メソッドが`async`と設定されている事に注意。これは、Nestが同期と**非同期**両方のパイプをサポートしているからできる事だ。今回`async`と設定したのは、クラスバリデータによる検証のうちいくつかが（Promiseによって）[非同期にできる](https://github.com/typestack/class-validator#custom-validation-classes)為。  
次に、メタタイプフィールドをメタタイプ変数として抽出する（`ArgumentMetadata`からこのメンバだけを抽出する）為に分割代入を使用している事に注意。これは「全ての`ArgumentMetadata`を取得してからメタタイプ変数を割り当てる為の追加の文を持つ」動作を表す構文にすぎない。  
また次に、ヘルパー関数`toValidate()`に注目しよう。これは現在処理中の引数がネイティブのJavaScript型である場合に検証ステップをバイパスする役割を果たす（そういった引数には検証用のデコレータをつける事ができない為、検証ステップを実行する理由がない）。  
更に次に、クラス変換関数`plainToClass()`を使用して、プレーンなJavaScriptの引数オブジェクトを型付きオブジェクトに変換し、検証を適用できるようにする。理由は、ネットワークのリクエストがデシリアライズされた時、受信したpostのbodyオブジェクトは**あらゆる型情報を持たない**からだ（基礎となるプラットフォーム、例えばExpressが動く方法となる）。クラスバリデータは先程DTO用に定義したバリデーションデコレータを使用する必要がある為、この変換を実行して受信したbodyを適切にデコレーションされたオブジェクトとして扱う必要がある。  
最後に、先述の通り、これは検証パイプである為、値を変更せずに返すか例外を投げる。  
最終ステップは検証パイプをバインドすることだ。パイプは、パラメータ・メソッド・コントローラ・グローバルの単位でスコープ化できる。さっき、Joiベースの検証パイプではメソッドレベルでパイプをバインドする例を見た。以下の例では、パイプのインスタンスをルートハンドラの`@Body()`デコレータにバインドして、投稿のbody部分を検証する為にパイプが呼び出されるようにする。

```ts :cats.controller.ts 
@Post()
async create(
  @Body(new ValidationPipe()) createCatDto: CreateCatDto,
) {
  this.catsService.create(createCatDto);
}
```

パラメータスコープ付きのパイプは、検証ロジックがある特定のパラメータにのみ関係する場合に便利。

## グローバルスコープ化パイプ
`ValidationPipe`は可能な限り汎用的に作成されている。アプリケーション全体の全てのルートハンドラに適用されるよう、グローバルスコープ付きパイプとして設定する事で、実用性を最大限にできる。

```ts :main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

>Notice  
>ハイブリッドアプリ（hybrid apps）の場合、`useGlobalPipes()`メソッドはゲートウェイとマイクロサービス用のパイプを作らない。「標準的な」（ハイブリッドでない）マイクロサービスアプリ（microservice apps）の場合、`useGlobalPipes()`はグローバルにパイプをマウントする。

グローバルパイプは、全てのコントローラと全てのルートハンドラに対して、アプリケーション全体で使用される。  

依存関係という点では、（上記の例のように`useGlobalPipes()`を使用して）任意のモジュールの外部から登録されたグローバルパイプは、バインディングが任意のモジュールのコンテキストの外で行われている為、依存関係をインジェクションできない事に注意。この問題を解決する為に、以下のような構成で、任意のモジュールから直接グローバルパイプを設定する。

```ts :app.module.ts
import { Module } from '@nestjs/common';
import { APP_PIPE } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_PIPE,
      useClass: ValidationPipe,
    },
  ],
})
export class AppModule {}
```

>HINT  
>このアプローチを使用してパイプの依存性インジェクションを行う場合、この構造が採用されているモジュールに関係なく、パイプは実際にはグローバルである事に注意。どこでやるべきか？　パイプ（上の例では`ValidationPipe`）が定義されているモジュールを選ぼう。また、カスタムプロバイダの登録を扱う方法は`useClass`だけではない。詳細はcustom-providersの項にて。

## 変換パイプの使用例
カスタムパイプの使用例は検証だけではない。この章の最初に、パイプは入力データを希望の形式に**変換**する事もできると述べた。これは、`transform`関数から返される値が引数の前の値を完全に上書きするからだ。

変換パイプはどんな時に便利だろう？　クライアントから渡されたデータがルートハンドラメソッドによって適切に処理される前に、データの変更（例えば文字列を整数に変換する等）が必要な場合を考えてみよう。さらに言えば、いくつかの必須データフィールドが欠落していて、デフォルト値を適用したい場合もある。**変換パイプ**は、クライアントリクエストとリクエストハンドラに間に処理関数を介在させる事で、これらの機能を実現する事ができる。  
文字列を整数値にパースする為のシンプルな`ParseIntPipe`を提示する。（※前述の通り、Nestにはより洗練された`ParseIntPipe`が組み込まれている。カスタム変換パイプの単純な例として組み込む）

```ts :parse-int.pipe.ts 
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed');
    }
    return val;
  }
}
```
このパイプを、選ばれたparamにバインドする。

```ts
@Get(':id')
async findOne(@Param('id', new ParseIntPipe()) id) {
  return this.catsService.findOne(id);
}
```

もう一つの有用な変換の例は、リクエストで提供されたidを使ってデータベースから**既存のユーザー**のエンティティを選択する事だ。

```ts
@Get(':id')
findOne(@Param('id', UserByIdPipe) userEntity: UserEntity) {
  return userEntity;
}
```

このパイプの実装は読者に委ねるが、他のすべての変換パイプと同様、入力値（id）を受け取り、出力値（UserEntityオブジェクト）を返す事に注意。この事で、お決まりのコードをハンドラから共通のパイプに抽象化して、コードをより宣言的でDRYなものにできる。

## デフォルトの提供
`Parse*`パイプはパラメータの値が定義されていることを期待する。Null、undifinedを受け取った時に例外を投げる。エンドポイントが、失われた`querystring`パラメータの値を処理できるようにするには、`Parse*`パイプがこれらの値を処理する前にデフォルト値を注入しなければならない。`DefaultValuePipe`がその役割を果たす。以下に示すように、関連する`@Parse*`パイプの前に`@Query()`デコレータで`DefaultValuePipe`をインスタンス化するだけだ。

```ts
@Get()
async findAll(
  @Query('activeOnly', new DefaultValuePipe(false), ParseBoolPipe) activeOnly: boolean,
  @Query('page', new DefaultValuePipe(0), ParseIntPipe) page: number,
) {
  return this.catsService.findAll({ activeOnly, page });
}
```

## 組み込みの検証パイプ
注意点として、`ValidationPipe`はNestによって提供されており、一般的な検証パイプを独自に構築する必要はない。組み込みの`ValidationPipe`はこの章で構築したサンプルよりも沢山のオプションを提供する。詳細とたくさんの例については[validationの項](./techniques-validation)に記載。