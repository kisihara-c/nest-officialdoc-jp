---
title: techniques-validation
---

# バリデーション

Webアプリケーションに送られてくるあらゆるデータを検証する事はベストプラクティスと言えよう。Nestでは、受信したリクエストを自動的に検証する為、あっという間に使えるいくつかのパイプが用意されている。

- `ValidationPipe`
- `ParseIntPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`

`ValidationPipe`は、強力な`class-validator`パッケージとその宣言型検証デコレータを利用している。受信する全てのクライアントのペイロードに対して検証ルールを実施する為の便利なアプローチを提供する。その中でも特定のルールは、各モジュールのローカルクラス・ローカルDTOの中で宣言される際、アノテーションを使う事で発効される。

## Overview

[パイプ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-pipes)の章では、シンプルなパイプを作った。コントローラやメソッド、あるいはグローバルアプリにバインドして、プロセスがどのように機能するかを説明した。本章のトピックを理解する為に、ぜひ事前に復習してほしい。ここからは`ValidationPipe`の実際の使用例に焦点を当て、高度なカスタマイズ機能の使用方法を紹介する。

## 組み込みの`ValidationPipe`を使う

>HINT  
>`ValidationPipe`は`@nestjs/common`パッケージからエクスポートされる。

このパイプは`class-validator`と`class-transformer`ライブラリを使用している為、多くのオプションが用意されている。設定オブジェクトをパイプに渡す事で設定できる。以下は組み込みのオプションだ。

```ts
export interface ValidationPipeOptions extends ValidatorOptions {
  transform?: boolean;
  disableErrorMessages?: boolean;
  exceptionFactory?: (errors: ValidationError[]) => any;
}
```

これらに加えて、全てのclass-validatorのオプション（`Validator`インターフェイスから継承されたもの）が使用できる。

|Option|型|説明|
| ---- | ---- | ---- |
|`skipMissingProperties`|`boolean`|`true`時、バリデータは検証対象のオブジェクトに中にないすべてのプロパティの検証をスキップする。|
|`whitelist`|`boolean`|`true`時バリデータは検証済みの（返ってきた）オブジェクトから検証デコレータを使用していないプロパティを取り除く。|
|`forbidNonWhitelisted`|`boolean`|`true`時、バリデータは、ホワイトリストに載ってないプロパティを、取り除かずに例外とする。|
|`forbidUnknownValues`|`boolean`|`true`時、未知のオブジェクトを検証しようとした際速やかに失敗する。|
|`disableErrorMessages`|`boolean`|`true`時、検証エラーはクライアントに返されない。|
|`errorHttpStatusCode`|`number`|この設定により、エラーが発生した場合に使用する例外の型を指定できる。デフォルトでは`BadRequestException`を投げる。|
|`exceptionFactory`|`Function`|`Function`バリデーションエラーの配列を受け取り、投げられるであろう例外オブジェクトを返す。|
|`groups`|`string[]`|Groups to be used during validation of the object.（ニュアンスを訳せず…）|
|`dismissDefaultMessages`|`boolean`|`true`時、検証はデフォルトメッセージを用いない。エラーメッセージが明示的に設定されていない場合、常に未定義となる。|
|`validationError.target`|`boolean`|`ValidationError`の中で対象が公開されるかどうかを示す。|
|`validationError.value`|`boolean`|検証された値が`ValidationError`の中で表出されるべきかを示す。|

>NOTICE  
>`class-validator`パッケージの詳細については、そのリポジトリを参照してください。

## オートバリデーション

`ValidationPipe`をアプリケーションレベルでバインドする事で、全てのエンドポイントが不正なデータの受信から保護されるようにしてみる。

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

そしてパイプをテストする為、ベーシックなエンドポイントを作ってみよう。

```ts
@Post()
create(@Body() createUserDto: CreateUserDto) {
  return 'This action adds a new user';
}
```

>HINT  
>TypeScriptではジェネリックやインターフェイスに関するメタデータが保存されていないため、これらをDTOで使用すると`ValidationPipe`が受信データを正しく検証できない場合がある。よって、DTOにコンクリートクラスを使用する事を検討してほしい。

それでは`CreateUserDto`にいくつかの検証ルールを追加してみよう。class-validatorパッケージで提供されているデコレータを使用する。この方法をとると、`CreateuserDto`を使用する全てのrouteは、自動的にこれらの検証ルールを実施する。

```ts
import { IsEmail, IsNotEmpty } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsNotEmpty()
  password: string;
}
```

リクエストのbodyに無効な`email`プロパティが含まれるリクエストがエンドポイントに届いた場合、アプリケーションは自動的に`400 Bad Request`コードと次のようなレスポンスbodyを返す。

```ts
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": ["email must be an email"]
}
```

`ValidationPipe`は、リクエストbodyの検証に加えて、他のリクエストオブジェクトのプロパティにも使用できる。例えば、エンドポイントのパスで`:id`を受けたいとする。このリクエストパラメータで数字しか受け付けないようにするには、以下のコードを使う。

```ts
@Get(':id')
findOne(@Param() params: FindOneParams) {
  return 'This action returns a user';
}
```

`FindOneParams`は`DTO`と同様に、class-validatorを使って検証ルールを定義した単なるクラスだ。以下のようになる。

```ts
import { IsNumberString } from 'class-validator';

export class FindOneParams {
  @IsNumberString()
  id: number;
}
```

## 詳細なエラーを無効にする

エラーメッセージはリクエストのどこが間違っているかを説明するのに役立つ。しかし運用環境によっては、詳細なエラーを無効にしたい場合もある。この場合、`ValidationPipe`にオプション・オブジェクトを渡す。

```ts
app.useGlobalPipes(
  new ValidationPipe({
    disableErrorMessages: true,
  }),
);
```

これで詳細なエラーメッセージがレスポンスbodyに表示されなくなる。

## プロパティの除去

`ValidationPipe`は、メソッドハンドラが受け取るべきでないプロパティをフィルタリングする事もできる。この場合、受け入れ可能なプロパティをホワイトリストに登録すると、ホワイトリストに含まれないプロパティはオブジェクトから自動的に除去される。例えば、ハンドラが`email`と`password`プロパティを期待しているにも関わらずリクエストに`age`プロパティが含まれていた場合、このプロパティは結果の`DTO`から自動的に削除される。このような動作を有効にするには、`whitelist`を`true`に設定する。

```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,
  }),
);
```

`true`に設定すると、ホワイトリストに登録されていないプロパティ（バリデーションクラスにデコレータがないもの）が自動的に削除される。

また、ホワイトリストに登録されていないプロパティが存在する場合、リクエストの処理を停止して、ユーザにエラーレスポンスを返すこともできる。オプションの`forbidNonWhitelsted`プロパティを`true`、`whitelist`を`true`に設定する。

## ペイロードオブジェクトの変換

ネットワーク経由で送られてくるペイロードは、プレーンなJavaScriptオブジェクトだ。`ValidationPipe`は、ペイロードを`DTO`クラスに応じて片付けされたオブジェクトに自動的に変換する事もできる。自動変換を有効にするには、`transform`を`true`に設定する。これは、メソッドレベルで行える。

```ts
@Post()
@UsePipes(new ValidationPipe({ transform: true }))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

この動作をグローバルに有効化するなら、グローバルパイプに設定しよう。

```ts
app.useGlobalPipes(
  new ValidationPipe({
    transform: true,
  }),
);
```

自動変換オプションを有効にすると、`ValidationPipe`はプリミティブな型の変換も行う。次の例では、`findeOne()`メソッドは、抽出された`id`パスパラメータを表す１つの引数を取る。

```ts
@Get(':id')
findOne(@Param('id') id: number) {
  console.log(typeof id === 'number'); // true
  return 'This action returns a user';
}
```

デフォルトでは、パスパラメータやクエリパラメータは全て文字列としてネットワーク上に送信される。上記の例では`id`の型を（メソッドのシグネチャ内で）`number`として指定している。その為、`ValidationPipe`は文字列の識別子を自動的に数字に変換しようとする。

## 明示的な変換

ここまで`ValidationPipe`が暗黙のうちにクエリやパスのパラメータを想定される型に基づいて変換する方法を示した。しかしながらこの機能はこちらで自動変換を有効化する必要がある。

そうではなく（自動変換は無効化して）`ParseIntPipe`か`ParseBoolPipe`を用いて明示的に値をキャストできる。  
※前述のように、全てのパスパラメータとクエリパラメータはデフォルトで文字列としてネットワークの向こう側に送信される為、`ParseStringPipe`が必要とされる事はない

```ts
@Get(':id')
findOne(
  @Param('id', ParseIntPipe) id: number,
  @Query('sort', ParseBoolPipe) sort: boolean,
) {
  console.log(typeof id === 'number'); // true
  console.log(typeof sort === 'boolean'); // true
  return 'This action returns a user';
}
```

>HINT  
>`ParseIntPipe`と`ParseBoolPipe`は`@nestjs/common`パッケージからエクスポートされている。

## 型マッピング

**CRUD**(Create/Read/Update/Delete)のような機能を構築していく際に、ベースであるエンティティタイプのバリエーションを揃えると良い感じになる。Nestはこの作業をより便利にする為に、型変換を行ういくつかのユーティリティ関数を提供している。

>WARNING  
>アプリケーションが`@nestjs/swagger`パッケージを使用している場合、型マッピングについての詳細はNestJS公式　OPENAPI-Mapped Typesを参照してほしい。同様に、`@nestjs/graphql`パッケージの場合はGRAPHQL-mapped typesをお願いする。どちらのパッケージも型に大きく依存しているので、使用するには別のインポートが必要となる。その為`@nestjs/mapped-types`を（アプリの種類に応じて`@nestjs/swagger`や`@nestjs/graphql`と使い分けずに）使用した場合、ドキュメントになっていない多様な副作用が発生する。

型の検証（`DTO`）を構築する際には、同じtypeで**create**と**update**のバリエーションを構築するやり方をやりたくなる。例えば、createバリアントでは全てのフィールドを必須としたくて、updateバリアントでは全てのフィールドをオプションとしたいかもしれない。

Nestはこの作業を簡単にして定型文を最小限にする為に、`PartialType()`ユーティリティ関数を提供している。

`PartialType()`関数は、入力した型の全てのプロパティを省略可能に設定した型（クラス）を返す。例えば、以下のような**create**型があるとする。

```ts
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

デフォルトではこれらのフィールドは全て必須だ。同じフィールドを持ち、全てがオプションである型を作成するには、引数としてクラス参照（`CreateCatDto`）を渡して`PartialType()`を使用する。

```ts
export class UpdateCatDto extends PartialType(CreateCatDto) {}
```

>HINT  
>`PartialType()`関数は`@nestjs/mapped-types`パッケージからインポートされている。`PickType()`暗数は、入力した型からプロパティのセットを選んで、新しい型（クラス）を構築する。例えば次のような型から始めるとしよう。

```ts
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

`PickType()`ユーティリティ関数を使って、このクラスからプロパティのセットを選ぶことができる。

```ts
export class UpdateCatAgeDto extends PickType(CreateCatDto, ['age'] as const) {}
```

>HINT  
>`PickType()`関数は`@nestjs/mapped-types`パッケージからインポートされている。`OmitType()`関数は、入力型からすべてのプロパティを抽出し、特定のキーセットを削除して型を構築する。例えば入力としてこんな型を考えてみよう。

```ts
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```

以下に示すように、`name`**以外**の全てのプロパティを持つ派生型を生成する事ができる。このコードにおいて、`OmitType`の第２引数はプロパティ名の配列だ。

```ts
export class UpdateCatDto extends OmitType(CreateCatDto, ['name'] as const) {}
```

>HINT  
>`OmitType()`関数は、`@nestjs/mapped-types`パッケージからインポートされている。`IntersectionType()`関数は２つの型を１つの新しい型（クラス）に結合する。例えば入力としてこんな２つの型を考えてみよう。

```ts
export class CreateCatDto {
  name: string;
  breed: string;
}

export class AdditionalCatInfo {
  color: string;
}
```

両タイプの全てのプロパティを組み合わせた新しい型を生成する事ができる。


```ts
export class UpdateCatDto extends IntersectionType(
  CreateCatDto,
  AdditionalCatInfo,
) {}
```

>HINT  
>`IntersectionType()`関数は、`@nestjs/mapped-types`パッケージからインポートされている。

型マッピングのユーティリティ関数は合成可能だ。たとえば次のようにすると、`CreateCatDto`型のプロパティのうち、`name`を除く全てのプロパティを持つ型（クラス）が生成され、それらのプロパティは省略可能に設定される。

```ts
export class UpdateCatDto extends PartialType(
  OmitType(CreateCatDto, ['name'] as const),
) {}
```

## 配列の解析と検証

TypeScriptはジェネリックやインターフェイスに関するメタデータを保存していないため、DTOでそれらを使用すると、`ValidationPipe`が受信データを適切に検証できない場合がある。例えば以下のコードでは`createuserDtos`が正しく検証されない。

```ts
@Post()
createBulk(@Body() createUserDtos: CreateUserDto[]) {
  return 'This action adds new users';
}
```

検証するには、配列をラップするプロパティを持つ専用クラスを作成するか、`ParseArrayPipe`を使用する。

```ts
@Post()
createBulk(
  @Body(new ParseArrayPipe({ items: CreateUserDto }))
  createUserDtos: CreateUserDto[],
) {
  return 'This action adds new users';
}
```

さらに、`ParseArrayPipe`は、クエリパラメータを解析するときにも便利だ。クエリパラメータとして渡された識別子に基づいてユーザーを返す`findByIds()`メソッドを考えてみよう。

```ts
@Get()
findByIds(
  @Query('id', new ParseArrayPipe({ items: Number, separator: ',' }))
  ids: number[],
) {
  return 'This action returns users by ids';
}
```

この構文は、以下のようなHTTP `GET`リクエストから入力されるクエリパラメータを検証する。

```
GET /?ids=1,2,3
```

## Websocketsとマイクロサービス

この章ではHTTPスタイルのアプリケーション（ExpressやFastifyなど）を使った例を紹介しているが、`ValidationPipe`は使用するトランスポートメソッドに関わらずWebSocketやマイクロサービスでも同様に動作する。

## 詳しく学ぶ
カスタムバリデータ、エラーメッセージ、そして`class-validator`パッケージで使えるデコレータについての詳細は[こちら](https://github.com/typestack/class-validator)