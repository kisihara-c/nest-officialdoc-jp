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

[パイプ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-pipes)の章では、シンプルなパイプを作った。コントローラやメソッド、あるいはグローバルアプリにバインドして、プロセスがどのように機能するかを説明した。本章のトピックを理解する為に、ぜひ復習してほしい。ここでは`ValidationPipe`の実際の使用例に焦点を当て、高度なカスタマイズ機能の使用方法を紹介する。

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

`ValidationPipe`は、メソッドハンドラが受け取るべきでないプロパティをフィルタリングする事もできる。この場合、受け入れ可能なプロパティをホワイトリストに登録すると、ホワイトリストに含まれないプロパティはオブジェクトから自動的に除去される。例えばハンドラが`email`と`password`プロパティを期待しているにも書くぁラズ、リクエストに`age`プロパティが含まれていた場合、このプロパティは結果の`DTO`から自動的に削除される。このような動作を有効にするには、`whitelist`を`true`に設定する。

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