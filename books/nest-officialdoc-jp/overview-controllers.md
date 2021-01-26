---
title: "overview-controllers"
---

# Controllers
コントローラはリクエストを処理してクライアントに応答を返す役割を持つ。  

[画像](https://docs.nestjs.com/assets/Controllers_1.png)

コントローラの目的はアプリケーションの特定のリクエストを受け取ること。どのコントローラがどのリクエストを受けるか、ルーティング機構が制御する。多くの場合、各コントローラは複数のルートを持ち、異なるルートは異なるアクションを実行できる。  

ベーシックなコントローラを作成するため、クラスとデコレータを使用する。デコレータはクラスを必要なメタデータに関連付け、ルーティングマップの作成――各リクエストを対応のコントローラに紐付ける動作を可能とする。

# ルーティング
以下の例では、ベーシックなコントローラを定義するために必要な`@Controller()`デコレータを使用する。オプションでcatsのパスプレフィックスを指定している。`@Controller()`デコレータでパスフレックスを使用することで、関連する複数のルートを簡単にグループ化でき、コードの繰り返しを最小限に抑えられる。たとえば顧客エンティティとのやりとりを管理する複数ルートを/customersと名付けてグループ化するケースを考える。その場合`@Controller()`デコレータでパスのプレフィックス`customers`を指定すれば、パスを繰り返し記述せずに済む。

```ts:cats.controller.ts
import { Controller, Get } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}
```

>HINT
>CLIを使ってコントローラを作りたい場合、`$nest g controller cats`で可能。

`findAll()`メソッドの前の`@Get()` HTTPリクエストメソッドデコレータは、HTTPリクエストを持つ特定のエンドポイントのハンドラを作成するコード。そのエンドポイントは、来たHTTPリクエストメソッド（この文章ではGET）と「ルートパス」に対応する。ハンドラに対するルートパスは、コントローラで宣言した（オプションの）接頭辞とリクエストデコレータで指定したパスを連結して決定される。全てのルート（`cats`）に対してプレフィックスを宣言しておらず、デコレータにパス情報を追加していないので、Nestは`Get/cats`のリクエストをこのハンドラにマッピングする。前述の通り、パスはオプションのコントローラのパスプレフィックスと、リクエストメソッドのデコレータで宣言されたパス文字列の両方が含まれる。例えばパスの接頭辞である`customers`とデコレータ`@Get('profile')`を組み合わせると、`GET /customers/profile`リクエストへのルートパッティングが作成される。  
上記の例では、このエンドポイントに`GET`リクエストが行われると、Nestがユーザ定義の`findAll()`メソッドにルーティングする。メソッド名は自由。明らかにルートをバインドするメソッドを宣言しなければならないが、選択されたメソッド名に意味はない。  
このメソッドは200のステータスコードと関連するレスポンスを返す。※このケースでは単なる文字列　どうしてそうなるか？　その説明として、Nestを使用する上で選べる、レスポンスを操作するための２つの選択肢を紹介する。

|||
| ---- | ---- |
| スタンダード（推奨）| この組み込みメソッドを使用すると、リクエストハンドラがJavaScriptのオブジェクトや配列を返した時、自動的にJSONにシリアライズされる。しかし、JavaScriptのプリミティブ型（文字列、数値、ブール値）を返す場合、Nestはシリアライズを試みずに値だけを送信する。結果、レスポンスの処理がシンプルになる。値を返すだけで、あとはNestが処理を行う。レスポンスのステータスコードは201を使用するPOSTを除き、デフォルトでは常に200。ハンドラレベルで`@HttpCode(...)`デコレータを追加することで、この動作を簡単に変更できる。 |
| ライブラリ固有 | ライブラリ固有（例：Express）のレスポンスオブジェクトを使用する事ができる。これはメソッドハンドラシグネイチャの`@Res()`デコレータを使用して注入することができる（例えば`findAll(@Res() response`　など)。このアプローチでは、各オブジェクトでネイティブなレスポンス処理メソッドを使用できる。例えばExpressを使用すると、`response.status(200),send()`とコーディングしてレスポンスを構築できる。 |

>WARNING  
>Nestはハンドラが`@rRes()`か`@Next()`のいずれかを使用している時、ライブラリ固有モードを選択した事を明示する。両方のモードが同時に使用された場合、スタンダードは単一のルートに対して自動的に無効化され、動作しなくなる。両方のアプローチを同時に使うには、`@Res({passthrough:true})`デコレータで`passthrough`オプションをtrueに設定しなければならない。
>※レスポンスオブジェクトを注入してcookie/headerのみ設定して、残りをフレームワークに任せる等

# リクエストオブジェクト
ハンドラは頻繁にクライアントのリクエストにアクセスする必要がある。Nestは基礎となるプラットフォーム（デフォルトではExpress）のリクエストオブジェクトへのアクセスを提供する。ハンドラのシグネイチャに`@Req()`デコレータを追加し、リクエストオブジェクトを注入する事で、リクエストオブジェクトにアクセスすることができる。

```ts:cats.controller.ts 

import { Controller, Get, Req } from '@nestjs/common';
import { Request } from 'express';

@Controller('cats')
export class CatsController {
  @Get()
  findAll(@Req() request: Request): string {
    return 'This action returns all cats';
  }
}

```

>HINT  
>上記の`request:request`パラメータのように`express`の型方式の利点を受けるには、`@types/express`パッケージをインストールする。

リクエストオブジェクトはHTTPリクエストを表し、リクエストクエリ文字列、パラメータ、HTTPヘッダ、bodyのプロパティを持っている。（[詳細](https://expressjs.com/en/api.html#req)）　多くの場合、これらのプロパティを手動で取得する必要はない。代わりに`@Bosy()`や`@Query()`のような専用の手軽なデコレータを使用することができる。以下に、提供されているデコレータと、対応するプラットフォーム固有オブジェクトの一覧を示す。

|||
| ---- | ---- |
|`@Request()`, `@Req()`|`req`|
|`@Response()`, `@Res()*`|`res`|
|`@Next()`|`next`|
|`@Session()`|`req.session`|
|`@Param(key?: string)`|`req.params` / `req.params[key]`|
|`@Body(key?: string)`|`req.body` / `req.body[key]`|
|`@Query(key?: string)`|`req.query` / `req.query[key]`|
|`@Headers(name?: string)`|`req.headers` / `req.headers[name]`|
|`@Ip()`|`req.ip`|
|`@HostParam()`|`req.hosts`|

*HTTPプラットフォーム（ExpressやFastify等）での型付けとの互換性のために、Nestは`@Res()`と`@Response()`のデコレータを提供しています。`Res'()`は単に`@Response()`のエイリアスです。いずれもネイティブプラットフォームのレスポンスオブジェクトのインターフェイスを直接見せています。使用する際には、基礎となるライブラリの型付け（`@types/express`等）もインポートして、最大限に活用する必要があります。注意：あなたがメソッドハンドラに`@res()`または`@Response()`のいずれかを突っ込んだ時、Nestをそのハンドラのためのライブラリ固有モードにする事となり、レスポンスの管理をしなければならなくなる。レスポンスオブジェクトを呼び出すことで、何らかのレスポンスを発行しなければならない。（`res.json(....)`、`res.send(....)`など）

>Hint  
>独自のカスタムデコレータを作成する方法については後述の該当章を参照の事。

# リソース

先だって、`cat`リソースを取得するためのエンドポイントを定義した（GETルート）。新しいレコードを作成するエンドポイントも用意したい。POSTハンドラを作成する。

```ts:cats.controller.ts 

import { Controller, Get, Post } from '@nestjs/common';

@Controller('cats')
export class CatsController {
  @Post()
  create(): string {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(): string {
    return 'This action returns all cats';
  }
}

```

シンプルだ。Nestha全ての標準的なHTTPメソッドにデコレータを提供している。※`@Get()`、 `@Post()`、 `@Put()`、 `@Delete()`、 `@Patch()`、 `@Options()`、 そして `@Head()`。  
加えて、`@All()`はこれら全てを処理するエンドポイントを定義している。

# ルートワイルドカード

パターンベースのルートもサポートされている。例えばアスタリスクはワイルドカードであり、任意の文字の組み合わせにマッチする。

```ts
@Get('ab*cd')
findAll() {
  return 'This route uses a wildcard';
}
```

`'ab*cd'`ルートパスは`abcd`、`ab_cd`、`abecd`等にマッチする。`?`、`+`、`+`、`()`の正規表現サブセット文字はルートパスで使用することができる。ハイフンとドットは文字列ベースのパスでは字の通り解釈される。

# ステータスコード
前述の通り、レスポンスのステータスコードは常に200で、POSTリクエストの場合のみ201となる。ハンドラレベルにおいて`@HttpCode(...)`デコレータを追加すれば、この動作を簡単に変更できる。

```ts
@Post()
@HttpCode(204)
create() {
  return 'This action adds a new cat';
}
```

>HINT  
>`HttpCode`は`@nestjs/common`パッケージからインポートできる。

多くの場合、ステータスコードは静的なものではない。様々な要因に依存する。その場合はライブラリ固有のレスポンス（`@Res()`の利用の注入）オブジェクトを利用できる。もしくは、エラーが起きた時例外を投げるためにも。

# ヘッダ
カスタムレスポンスヘッダを指定するには、`@Header()`デコレータを使用するか、ライブラリ固有のレスポンスオブジェクトを使用する（`res.header()`を直接呼び出す）。

```ts
@Post()
@Header('Cache-Control', 'none')
create() {
  return 'This action adds a new cat';
}
```

>HINT  
>`Header`は`@nestjs/common`パッケージからインポートできる。

# リダイレクト
レスポンスを特定のURLにリダイレクトするには、`@Redirect()`デコレータかライブラリ固有のレスポンスオブジェクトを使用する（`res.redirect()`を直接呼び出す）。
`@Redirect()`は必須のurl引数と省略可能な`statusCode`引数を受け取る。`statusCode`のデフォルト値は302（Found）。

```ts
@Get()
@Redirect('https://nestjs.com', 301)
```

HTTPステータスコードやリダイレクトURLを動的に決定したい場合、ルートハンドラメソッドからオブジェクトを返す。

```ts
{
  "url": string,
  "statusCode": number
}
```
戻り値は`@Redirect()`デコレータに渡された引数を上書きする。例としては下記のようになる。

```ts
@Get('docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if (version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5/' };
  }
}
```

# ルートパラメータ
静的なパスを持つルートは、リクエストの一部として動的なデータを受け入れる必要がある場合には動作しない。例えば、`GET /cats/1`で`id1`の猫を取得する場合等。パラメータを持つルートを定義するために、ルートのパスにパラメータトークンを設定すると、リクエストURLの動的な変数を受け付ける事ができる。以下の`@Get()`デコレータのサンプルの中で、パラメータトークンの使用法を例示する。この方法で宣言されたルートパラメータは`@Param()`デコレータを使用してアクセスすることができ、メソッドシグネイチャに追加する事ができる。

```ts
@Get(':id')
findOne(@Param() params): string {
  console.log(params.id);
  return `This action returns a #${params.id} cat`;
}
```

`@Param()`はメソッドのデコレータ（上記の例ではparams）をデコレーションするため使用される。ルートパラメータは、メソッドの内部で、デコレーションされたメソッドパラメータのプロパティとして利用できる。上記に見られるように、params.idを参照することでidパラメータにアクセス可能となる。また、デコレータに特定のパラメータトークンを渡して、メソッドの内部でルートパラメータの名前を直接参照することもできる。

>HINT  
>`Param`は`@nestjs/common`パッケージからインポートできる。

```ts
@Get(':id')
findOne(@Param('id') id: string): string {
  return `This action returns a #${id} cat`;
}
```

# サブドメインルーティング
`@Controller`デコレータはhostオプションを使用して、受信リクエストのHTTPホストと特定の値との一致を要求できる。

```ts
@Controller({ host: 'admin.example.com' })
export class AdminController {
  @Get()
  index(): string {
    return 'Admin page';
  }
}
```

>WARNING  
>Fastifyは入れ子ルータのサポートがないので、サブドメインルーティングを使用する場合は、デフォルトのExpressアダプタを使用する。

# スコープ
違うプログラミング言語文化圏から来た人々にとっては、Nestでほぼ全てのものが着信リクエストをまたいで共有されている事は想定外かも知れない。Nestはデータベースへの接続プール、グローバルな状態を持つシングルトンサービス等を持つ。Node.jsはrequest/response Multi-Threaded Stateless Model でない事を思い出してほしい。全てのリクエストが個別のスレッドによって処理されるモデルはとっていない。すなわち、シングルトンインスタンスは完全に安全である。  
しかし、リクエストベースのコントローラのライフタイムが必要な場合もある。例えば、GraphQLアプリケーションでの各リクエスト対応のキャッシングや、リクエストの追跡、マルチテナンシー等。スコープを制御する方法についてはInjection scopesの項目を参照の事。

# 非同期性
我々は最新のJavaScriptを愛している。よってデータの抽出はほとんどが非同期である事を知っている。Nestは非同期関数をサポートしている。

>Hint  
>async/awaitの機能については[ここ](https://kamilmysliwiec.com/typescript-2-1-introduction-async-await/)を参照の事。

全てのasync関数はPromiseを返す必要がある。Nestでは、Nestが自身で解決する遅延変数を返す事ができる。例は下記の通りとなる。

```ts :cats.controller.ts
@Get()
async findAll(): Promise<any[]> {
  return [];
}
```

上記のコードは有効だ。さらに、NestのルートハンドラはRxJSの[観測可能ストリーム](http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html)を返す機能を持ち、さらに強力なものとなっている。Nestは自動で以下のソースを読み込み、ストリームが完了次第最後に出力された値を受け取る。

```ts :cats.controller.ts

@Get()
findAll(): Observable<any[]> {
  return of([]);
}

```

上記のアプローチのうち、必要に応じた選択肢を使用できる。

# ペイロードのリクエスト
先のPOSTルートハンドラの例では、クライアントのパラメータを受け付けていなかった。ここに`@Body()`デコレータを追加していく。  
しかし最初に、DTOスキーマ（Data Transfer Object）を決定する必要がある。DTOスキーマはTypeScriptのインターフェイスか単純クラスを使って決定可能である。ここではクラスの使用を推奨する。奇妙かもしれないが、理由は、ClassはJavaScriptES6標準機能の一部であり、コンパイルされたJavaScript上で実在のエンティティとして確保されるが、TypeScriptインターフェイスはトランスパイレーションの際に削除される為、Nestが参照不能となるからだ。実行時に変数のメタ型にアクセスできる場合、Pipesのような機能の可能性が広がる為、重要な要素となる。  
`CreateCatDto`クラスを作成しよう。

```ts :create-cat.dto.ts
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```
基本的なプロパティは三つのみ。その後、新規作成したDTOを`CatsCantroller`の中で使える。

```ts :cats.controller.ts
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  return 'This action adds a new cat';
}
```

# エラーの処理
エラーの処理（＝例外処理）についてはexeption-filtersの章を参考の事。

# フルリソースサンプル
以下に、利用可能なデコレータ数個を使って基本的なコントローラを作成した例を示す。このコントローラは、内部データにアクセスして操作する為のメソッドをいくつか明示している。

```ts :cats.controller.ts 

import { Controller, Get, Query, Post, Body, Put, Param, Delete } from '@nestjs/common';
import { CreateCatDto, UpdateCatDto, ListAllEntities } from './dto';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Body() createCatDto: CreateCatDto) {
    return 'This action adds a new cat';
  }

  @Get()
  findAll(@Query() query: ListAllEntities) {
    return `This action returns all cats (limit: ${query.limit} items)`;
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return `This action returns a #${id} cat`;
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateCatDto: UpdateCatDto) {
    return `This action updates a #${id} cat`;
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return `This action removes a #${id} cat`;
  }
}
```

>HINT  
>NestCLIはジェネレーター（セマティック）を提供している。すべてのボイラープレートコードを自動で生成する事で、作業を省略し、開発体験をよりシンプルにしてくれる。詳細は「crud-generator」の項にて。

# 起動と実行

上記のコントローラの完全な定義があっても、NestはまだCatsControllerの存在を知らず、勿論クラスのインスタンスも作成しない。  
コントローラは常にモジュールに属している為、コントローラの配列を`@Module()`デコレータに含めている。ルートの`AppModule`以外のモジュールを定義して`CatsController`を導入する。

```ts :app.module.ts

import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';

@Module({
  controllers: [CatsController],
})
export class AppModule {}

```

モジュールクラスに`@Module()`デコレータを使ってメタデータをアタッチすれば、どのコントローラをマウントするか簡単に設定できる。

# ライブラリー固有アプローチ
ここまでは、Nestの標準的なレスポンスの操作方法について説明した。レスポンスを操作する２つ目の方法は、ライブラリ固有のレスポンスオブジェクトを使用する事。特定のレスポンスオブジェクトを注入する為には、`@Res()`デコレータを使用する必要がある。違いを示すため、`CatsController`を以下のように書き換える。

```ts:

import { Controller, Get, Post, Res, HttpStatus } from '@nestjs/common';
import { Response } from 'express';

@Controller('cats')
export class CatsController {
  @Post()
  create(@Res() res: Response) {
    res.status(HttpStatus.CREATED).send();
  }

  @Get()
  findAll(@Res() res: Response) {
     res.status(HttpStatus.OK).json([]);
  }
}
```

このアプローチは正常に動作するし、実際にレスポンスオブジェクトを完全に制御して柔軟性を高める事ができる（ヘッダ操作やライブラリ固有の機能などなど）が、必要に注意を要する。あまり明確なアプローチではなく、いくつかの欠点が生じる為。主要な欠点は、コードがプラットフォームに依存するようになり（基礎となるライブラリがレスポンスオブジェクトに対して異なるAPIを持っている可能性がある為）テストの難易度が上がる事（レスポンスオブジェクトのモックが必要となる等）。  
また上記の例ではインターセプターや`@HttpCode()`/`@Header()`デコレータ等、Nest標準のレスポンス処理に依存するNest機能との互換性を失う。修正するには、以下のようにpassthroughオプションをtrueにする。

```ts
@Get()
findAll(@Res({ passthrough: true }) res: Response) {
  res.status(HttpStatus.OK);
  return [];
}
```

こうすればネイティブのレスポンスオブジェクトと対話可能となり（例えば条件に応じたcookieやheaderの設定）、他の事はプラットフォームのフレームワークに任せられるようになる。