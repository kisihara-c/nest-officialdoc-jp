---
title: fundamentals-executioncontext
---

# 実行コンテキスト

Nestは複数のアプリケーションコンテキスト（例：NestHTTPサーバーベースの、マイクロサービスやWebSocketsのアプリケーションコンテキスト）にまたがって動くアプリケーションを、簡単に作成する為に役立つ、いくつかのユーティリティクラスを提供している。多くのコントローラ、メソッド、および実行テキストを超えて動作する汎用的な[ガード](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-guards)、[フィルタ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-excepitonfilters)、[インターセプター](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-interceptors)の構築に役に立つはずだ。

この章では`ArgumentsHost`クラスと`ExecutionContext`クラスを取り上げる。

## ArgumentsHostクラス

`ArgumentHost`クラスはハンドラに渡された引数を取得するためのメソッドを提供する。結果、引数を取得するための適切なコンテキスト（例えばHTTP、RPC（マイクロサービス）、WebSockets）を選択できる。フレームワークは貴方のアクセスしようとしていた場所で`ArgumentsHost`のインスタンスを提供する。典型的には`host`パラメータとして引用されるものだ。例えば[例外フィルタ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-excepitonfilters)の`catch()`メソッドは`ArgumentsHost`インスタンスと一緒に（前置詞with、「～を使って」かも…）呼び出される。

`ArgumentsHost`はシンプルにハンドラの引数を抽象化したものとして機能する。例えばHTTPサーバーアプリケーション（`@nestjs/platform-express`が使用されている時）に対しては、`host`オブジェクトはExpressの`[request, response, next]`配列をカプセル化する。`request`はリクエストオブジェクトとなり、`response`はレスポンスオブジェクトとなり、`next`はアプリケーションのrequest-responseサイクルをコントロールする関数となる。一方、GraphQLアプリケーションの場合、ホストオブジェクトには`[root, args, context, info]`配列が含まれる。

## 現在のアプリケーションのコンテキスト

複数のアプリケーションコンテキストで動作する汎用の[ガード](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-guards)、[フィルタ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-excepitonfilters)、[インターセプター](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-interceptors)を構築する際、メソッドが現在動作しているアプリケーションの型を決定する方法が必要だ。これは`ArgumentsHost`の`getType()`メソッドで行う。

```ts
if (host.getType() === 'http') {
  // 通常のHTTPリクエスト（REST）のコンテキストでのみ行いたい動作をやる
} else if (host.getType() === 'rpc') {
  // マイクロサービスのリクエストのコンテキストでのみ行いたい動作をやる
} else if (host.getType<GqlContextType>() === 'graphql') {
  // GlaphQLリクエストのコンテキストでのみ行いたい動作をやる
}
```

>HINT  
>`GqlContest`は`@nestjs/graphql`パッケージからインポートしている。

アプリケーションの型が利用できるようになったので、以下のようにもっと汎用的なコンポーネントを書ける。

## ホストハンドラの引数
ハンドラに渡される引数の配列を取得する為には、ホストオブジェクトの`getArgs()`メソッドを使うアプローチが一つある。

```ts
const [req, res, next] = host.getArgs();
```

`getArgByIndex()`メソッドを使用して、インデックスを使って特定の引数をpluckできる。

```ts
const request = host.getArgByIndex(0);
const response = host.getArgByIndex(1);
```

以上の例の上ではリクエストとレスポンスのオブジェクトをインデックスから取得しているが、これはアプリケーションを特定の実行コンテキストと結びつける為、一般的にはオススメできない。代わりに`host`オブジェクトのユーティリティメソッドを利用し、状況に適したアプリケーションコンテキストに切り替える事で、コードをより堅牢かつ再利用しやすいものにできる。コンテキストを切り替えるユーティリティメソッドは以下の通り。

```ts
/**
 * コンテキストをRPCに切り替える。
 */
switchToRpc(): RpcArgumentsHost;
/**
 * コンテキストをHTTPに切り替える。
 */
switchToHttp(): HttpArgumentsHost;
/**
 * コンテキストをWebSocketsに切り替える。
 */
switchToWs(): WsArgumentsHost;
```

先程の例を`switchToHttp()`メソッドを使って書き換えてみる。`host.switchToHttp()`ヘルパーを呼び出すと、HTTPアプリケーションのコンテキストに適した`HttpArgumentsHost`オブジェクトを返す。このオブジェクトには目的のオブジェクトを抽出する為に使用可能な便利なメソッドが２つある。ネイティブなExpress型のオブジェクトを返す場合は、Expressの型アサーションも使おう。

```ts
const ctx = host.switchToHttp();
const request = ctx.getRequest<Request>();
const response = ctx.getResponse<Response>();
```

同様に、`WsArgumentsHost`と`RpcArgumentsHost`には、マイクロサービスと`WebSockets`のコンテキストで適切なオブジェクトを返すメソッドがある。

```ts
export interface WsArgumentsHost {
  /**
   * データオブジェクトを返す。
   */
  getData<T>(): T;
  /**
   * クライアントオブジェクトを返す。
   */
  getClient<T>(): T;
}
```

```ts
export interface RpcArgumentsHost {
  /**
   * データオブジェクトを返す。
   */
  getData<T>(): T;

  /**
   * クライアントオブジェクトを返す。
   */
  getContext<T>(): T;
}
```
## ExecutionContextクラス
`ExecutionContext`は`ArgumentsHost`を拡張し、現在の実行プロセスに関するさらなる詳細を提供する。`ArgmentsHost`と同様に、Nestは[ガード](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-guards)の`canActivate()`メソッドや[インターセプター](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-interceptors)の`intercept()`メソッドなど、必要に応じて`ExecutionContext`のインスタンスを提供する。提供されるメソッドは以下等。

```ts
export interface ExecutionContext extends ArgumentsHost {
  /**
   * 現在のハンドラが属するコントローラクラスの型を返す
   */
  getClass<T>(): Type<T>;
  /**
   * リクエストパイプラインで次に呼び出される
   * ハンドラ（メソッド）への参照を返す
   */
  getHandler(): Function;
}
```

`getHandler()`メソッドは、呼び出されようとしているハンドラへの参照を返す。`getClass()`メソッドは、このハンドラが属する`Controller`クラスの型を返す。例えばHTTPのコンテキストの中、現在処理中のリクエストがPOSTリクエストで、`CatsController`の`create()`メソッドにバインドされている場合、`getHandler()`は`create()`メソッドへの参照を返し、`getClass()`は`CatsController`の（インスタンスではなく）**型**を返す。

```ts
const methodKey = ctx.getHandler().name; // "create"
const className = ctx.getClass().name; // "CatsController"
```

currentなクラスとハンドラメソッドの両方へアクセスできる機能は、強い柔軟性を提供する。最も大事なのは、ガードやインターセプタ内から`@SetMetadata()`デコレータを通してメタデータの集合にアクセスする方法がある事だ。以下ではこのユースケースを取り上げる。

## リフレクションとメタデータ
Nestは`@SetMetadata()`デコレータを使って**カスタムメタデータ**をルートハンドラにアタッチする機能を提供している。そうなればクラス内からこのメタデータにアクセスし、certain decisionsをmakeする事ができる（訳出できず）。

```ts :cats.controller.ts
@Post()
@SetMetadata('roles', ['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

>HINT  
>`@SetMetadata()`デコレータは`@nestjs/common`パッケージからインポートしている。

上記のコードでは`roles`メタデータ（`roles`はメタデータのキー、`['admin']`は関連する変数）を`create()`メソッドにアタッチしている。これは動くが、ルートで直接`@SetMetadata()`を使うのはグッドプラクティスとはいえない。代わりに以下のように独自のデコレータを作成してほしい。

```ts :roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

このアプローチのほうがずっとすっきりしていて読みやすく、強く型付けされている。カスタムの`@Roles()`デコレータができたので、`create()`メソッドを改めて装飾できる。

```ts :cats.controller.ts
@Post()
@Roles('admin')
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

ルートのロール（role、roles。カスタムメタデータ）にアクセスするには、フレームワークから提供され`@nestjs/core`パッケージで公開されている`Reflector`ヘルパークラスを使用する。`Reflector`は通常の方法でクラスにインジェクションできる。

```ts :roles.guard.ts
@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
```

>HINT  
>`Refletor`クラスは`@nestjs/core`パッケージからインポートしている。

ハンドラのメタデータを読むためには、`get()`メソッドを使用する。

```ts
const roles = this.reflector.get<string[]>('roles', context.getHandler());
```

`Reflector#get`メソッドを使うと、２つの引数（メタデータ**キー**とメタデータを取得する**コンテキスト**（デコレータの対象））を取得する事でメタデータに簡単にアクセスできる。この例では、指定された**キー**は`roles`だ（上記の`roles.decorator.ts`とその中の`SetMetadata()`の呼び出しを参照の事）。コンテキストが`contet.getHandler()`の呼び出しによって提供され、現在処理中のルートハンドラのメタデータが抽出される。思い出してほしい、`getHandler()`はルートハンドラ関数への**参照**を提供している。

もしくは、コントローラレベルでメタデータを適用し、コントローラクラス内の全てのルートに適用することで、コントローラを整理する事もできる。

```ts :cats.controller.ts 
@Roles('admin')
@Controller('cats')
export class CatsController {}
}
```

この場合、コントローラのメタデータを抽出するために、`context.getHandler`の代わりに2つめの引数として`context.getClass`を渡す。これは、メタデータ抽出の為のコンテキストとしてコントローラクラスを提供する為。

```ts :roles.guard.ts
const roles = this.reflector.get<string[]>('roles', context.getClass());
```

複数のレベルでメタデータを提供できる事を考えると、複数のコンテキストからメタデータを抽出してマージする必要があるかもしれない。`Reflector`クラスにはこれを支援する為の２つのユーティリティメソッドが用意されている。これらのメソッドはコントローラとメソッドの**両方**のメタデータを一度に抽出し、別々の方法で結合する。

以下のシナリオでは、両方のレベルで`roles`メタデータを用意している。

```ts :cats.controller.ts 
@Roles('user')
@Controller('cats')
export class CatsController {
  @Post()
  @Roles('admin')
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }
}
```

デフォルトのロールとして`'user'`を指定し、特定のメソッドに対して選択的にオーバーライドしたい場合は、`getAllAndOverride()`を使う事がある。

```ts
const roles = this.reflector.getAllAndOverride<string[]>('roles', [
  context.getHandler(),
  context.getClass(),
]);
```

このコードを持つガードが、上記のメタデータを持つ`create()`メソッドのコンテキストで実行されると、`['admin']`を含むロールになる。

両方のメタデータを取得してマージするには（このメソッドは配列とオブジェクトの両方をマージできる）`getAllAndMerge()`メソッドを使用する。

```ts
const roles = this.reflector.getAllAndMerge<string[]>('roles', [
  context.getHandler(),
  context.getClass(),
]);
```

これは`['user', 'admin']`を含むロールになる。

これらのマージメソッドの両方において、第一引数にメタデータキーを渡し、第二引数にメタデータターゲットのコンテキストの配列（すなわち`getHandler()`メソッド及び/あるいは`getClass()`メソッド）を渡す事になる。