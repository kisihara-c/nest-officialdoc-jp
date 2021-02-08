---
title: "overview-exceptionfilters"
---

# 例外フィルタ
Nestには未処理のすべての例外を処理するための**例外レイヤ**が組み込まれている。アプリケーションのコードで処理されない例外が発生した場合、このレイヤーが例外を補足し、ユーザーフレンドリなレスポンスを自動的に送信する。

[画像](https://docs.nestjs.com/assets/Filter_1.png)

素晴らしい事に、このアクションは組み込みのグローバル例外フィルタによって実行され、`HttpExepction`型の例外（およびそのサブクラス）を処理する。
例外が認識されない場合（つまり、例外が`HttpExeption`でも`HttpExeption`を継承するクラスでも無い場合）、組み込まれた例外フィルタは以下のような規定のJSONレスポンスを生成する。

```ts
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

# スタンダードな例外のthrow
Nestは`@nestjs/common`パッケージで公開されている`HttpException`クラスを提供している。典型的なHTTP RESt/GraphQL APIベースのアプリケーションでは、特定のエラー条件が発生した時に標準のHTTPレスポンスオブジェクトを送信するのがベストプラクティスとなる。  
例えば、`CatsController`には`findAll()`メソッド（`GET`ルートハンドラ）がある。このルートハンドラが何らかの理由で例外をthrowしたと考えてみよう。実際の流れを追う為、以下のようにハードコーディングする。

```ts :cats.controller.ts 
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

>HINT  
>ここでは`HTTPStatus`を使用する。`@nestjs/common`パッケージからインポートされたヘルパーのenumだ。

クライアントがこのエンドポイントを呼び出すと、レスポンスは以下の通りとなる。

```ts
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

`HttpException`コンストラクタは、レスポンスを決定する２つの必須の引数を取る。
- 引数`response`はJSONレスポンスの本文（body）を定義する。以下説明するように、`string`か`object`を指定できる。
- 引数`status`は[HTTPのステータスコード](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)を定義する。  

標準状態では、JSONレスポンスの本文には２つのプロパティが含まれる。
- `statusCode` : デフォルトでは`status`引数で提供されるHTTPステータスコード
- `message` : `status`に基づくHTTPエラーの短い説明

JSONレスポンス本文のメッセージ部分だけをオーバーライドするには、`response`引数に文字列を指定する事。JSONレスポンス本文の全体をオーバーライドする為には、`response`引数にオブジェクトを渡す事。NestはそのオブジェクトをシリアライズしてJSONレスポンス本文として返す。  
２番目のコンストラクタの引数`status`は、有効なHTTPステータスコードである必要がある。ベストプラクティスは、`@nestjs/common`からインポートされた`HTTPStatus`enumを使用する事。  
以下にレスポンス本文全体をオーバーライドする例を示す。

```ts :cats.controller.ts
@Get()
async findAll() {
  throw new HttpException({
    status: HttpStatus.FORBIDDEN,
    error: 'This is a custom message',
  }, HttpStatus.FORBIDDEN);
}
```
上記を使用すると、レスポンスは以下のようになる。
```ts
{
  "status": 403,
  "error": "This is a custom message"
}
```

## 例外のカスタマイズ
多くの場合例外を追加でコーディングする必要はなく、次のセクションで説明するように、Nest組み込みの例外を使える。どうしても例外をカスタマイズする必要がある場合は、`HttpException`を継承する、自分自身の**例外の階層**を作成する事を勧める。このアプローチなら、Nestが例外を認識し、エラー応答を自動的に処理する。このような例外を実装してみよう。

```ts :forbidden.exception.ts 
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

`ForbiddenException`は`HttpExeption`を継承していて、組み込みの例外ハンドラとシームレスに動作するので、したがって`findAll()`メソッドの中で使用する事ができる。

```ts :cats.controller.ts 
@Get()
async findAll() {
  throw new ForbiddenException();
}
```

## 組み込みHTTP exeptions
Nestはベースとなる`HttpExeption`を継承する標準の例外を複数提供する。これらは`@nestjs/common`パッケージから公開されており、最も一般的なHTTPExeptionの多くを記述している。

- BadRequestException
- UnauthorizedException
- NotFoundException
- ForbiddenException
- NotAcceptableException
- RequestTimeoutException
- ConflictException
- GoneException
- HttpVersionNotSupportedException
- PayloadTooLargeException
- UnsupportedMediaTypeException
- UnprocessableEntityException
- InternalServerErrorException
- NotImplementedException
- ImATeapotException
- MethodNotAllowedException
- BadGatewayException
- ServiceUnavailableException
- GatewayTimeoutException
- PreconditionFailedException

## 例外フィルタ
基本的な組み込みの例外は多くのケースを自動的に処理してくれるが、例外レイヤを**完全に**処理したい場合もあるかもしれない。例えばロギングを追加したり、動的な要因に基づいた異なるJSONスキーマを使用する等。**例外フィルタ**はまさにこの目的のために設計されている。制御中の正確なフローとクライアントに送られるレスポンスの内容を制御する事ができる。  
例外フィルタを作成してみよう。`HttpException`クラスのインスタンスとなる例外をキャッチし、カスタマイズされた例外ロジックを実装するものだ。その為には、基礎となるプラットフォームの`Request`オブジェクトと`Response`オブジェクトにアクセスする必要がある。`Request`オブジェクトにアクセスし、元の`url`を取得して、ログ情報に含める事ができる。`Response`オブジェクトを使用し、送信されるレスポンスを直接制御できる。

```ts :http-exception.filter.ts 
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
```

>HINT  
>すべての例外フィルタは一般的な`ExceptionFilter<T>`インターフェイスを実装する必要がある。その為には、`catch(exception: T, host: ArgumentsHost)`メソッドにそのシグネイチャを指定する必要がある。Tには例外の型を入れる。

`@Catch(HttpException)`デコレータは必要なメタデータを例外フィルタにバインドし、この特定のフィルタが`HttpException`型の例外を探しており他は不要である事をNestに伝達する。`Catch()`デコレータには単一のパラメータ・カンマ区切りのリストを指定する事ができる。一度に複数の型の例外に対するフィルタを設定する事ができる。

## ArgmentsHost
`catch()`メソッドのパラメータを見てみよう。`exception`変数は現在処理中の例外オブジェクトだ。ホストパラメータは`ArgumentsHost`となる。`ArgumentsHost`は強力で有用なオブジェクトだ。このコードサンプルでは`ArgumentsHost`といくつかのヘルパーメソッドを使用する。元のリクエストハンドラ（例外が発生したコントローラ内）に渡されている`Request`オブジェクトと`Response`オブジェクトへの参照を取得している。`ArgmentsHost`については後のexecution context（実行コンテキスト）の項で詳しく説明する。  

※この抽象度の理由は、`ArgumentsHost`が全てのコンテキストで機能する為だ。例えば今扱っているHTTPサーバーのコンテキストだけでなく、マイクロサービスやWebSocketsを含む。execution contextの項では、`ArgumentsHost`とそのヘルパー関数の力を使って、どのようにして**あらゆる**実行コンテキストの基礎たる適切な引数（appropriate underlying arguments）にアクセスできるかを見ていく。そうして、全てのコンテキストで動く汎用的な例外フィルタを書くことができるようになる。

## バインディングフィルタ
新しい`HttpExceptionFilter`を`CatsController`を`create()`メソッドに結びつけてみよう。

```ts :cats.controller.ts 
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

>Hint  
>`@UseFilters()`デコレータは`@nestjs/common`パッケージからインポートされている。

使用している`@UseFilters()`デコレータは、`@Catch()`デコレータと同様単一のフィルタのインスタンスを受け取ることもできるし、カンマで区切られた複数のフィルタインスタンスを受け取る事もできる。ここでは`HttpExceptionFilter`のインスタンスを生成している。あるいは、（インスタンスの代わりに）クラスを渡すこともできる。

```ts :cats.controller.ts
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

>Hint  
>フィルタの作成の上では、可能であればインスタンスではなくクラスを使用する事を勧める。Nestではモジュール全体で同じクラスのインスタンスを簡単に再利用できる為、**メモリ使用量**を削減できる。

上記の例では`HttpExceptionFilter`は単一の`create()`ルート・ハンドラにのみ適用され、メソッドでスコープ化されている。例外フィルタは、メソッド・コントローラ・グローバル、様々な単位でスコープ化できる。たとえばフィルタをコントローラ単位でスコープ化させるには以下の通り。

```ts :cats.controller.ts 
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```

このコンストラクタは`CatsContorller`内部で定義されているすべてのルートハンドラに対して`HttpExceptionFilter`を設定する。  
グローバルスコープのフィルタは以下の通り作成する。

```ts :main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```

>WARNING  
>`useGlobalFilters()`メソッドは、ゲートウェイやハイブリッドアプリケーション用のフィルタをセットアップしない。

グローバルスコープ化されたフィルタは、全てのコントローラと全てのルートハンドラに対してアプリケーション全体で使用される。依存関係のインジェクションは行えない。（上記の`useGlobalFilters()`を使った例のように）任意のモジュールの外部から登録されたグローバルフィルタについては、全てのモジュールの外部のコンテキストで実行される為。この問題は、次のような構成を使用し、**任意のモジュールから直接**グローバルスコープ化されたフィルタを登録する事で解決できる。

```ts :app.module.ts
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

>Hint  
>このアプローチを使用する際、この構造が採用されているモジュールに構わず、フィルタは実際にはグローバルである事に注意。この構造はどこで行われるべきか？　フィルタ（上記の例では`HttpExceptionFilter`）が定義されている場所を選ぼう。また、カスタムプロバイダの登録は`useClass`以外でもできる。詳細はcustomproviderの項を参照の事。

この方法なら、必要に応じていくつでも簡単にフィルタを追加する事ができる。単にそれぞれをプロバイダ配列へ追加するだけだ。

## Catch everything
種類に関わらず**全て**の未処理の例外をCatchする為には、`@Catch()`デコレータの変数リストを空にしておく。

```ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();

    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
    });
  }
}
```

上記の例ではフィルタは例外の種類（クラス）に関係なく、throwされたそれぞれの例外をCatchする。

## 継承
通常アプリケーションの要件を満たすために、完全にカスタマイズされた例外フィルタを作成する。しかし、組み込みの標準的な**グローバル例外フィルタ**をシンプルに拡張して、特定の理由に基づいて動作をオーバーライドしたい場合もあるだろう。  
例外処理をベースフィルタに委譲するには、`BaseExceptionFilter`を継承して、継承された`catch()`メソッドを呼ぶ必要がある。

```ts: all-exceptions.filter.ts 
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
```

>Warning  
>`BaseExceptionFilter`を拡張してできた、メソッドやコントローラのスコープ下にあるフィルタは、`new`によってインスタンス化されるべきではない。代わりに、フレームワークが自動的にインスタンスを生成する。

上記の実装はこのアプローチを示す為のただの骨組みだ。継承例外フィルタの実装には、カスタマイズされたビジネスロジック（様々な条件の処理等）が含まれる。  
グローバルフィルタは、ベースフィルタを拡張**可能**。２つの方法で行える。  
１つ目の方法は、カスタムグローバルフィルタをインスタンス化する時に`HttpServer`参照をインジェクションする事だ。

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));

  await app.listen(3000);
}
bootstrap();
```

２つめの方法はAPP_FILTERを使う方法だ。遡ってバインディングフィルタの項を参照の事。
