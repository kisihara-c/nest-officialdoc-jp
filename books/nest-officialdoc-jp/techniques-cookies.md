---
title: techniques-cookies
---

# cookie

**HTTPクッキー**はユーザーのブラウザが保存する小さなデータだ。ウェブサイトがステートフルな情報を記憶する為の信頼できるメカニズムとして設計されている。ユーザーが再度ウェブサイトにアクセスすると、クッキーは自動的にリクエストともに送信される。

まず必要なパッケージ（とTypeScriptユーザーならその型）をインストールしよう。

```
$ npm i cookie-parser
$ npm i -D @types/cookie-parser
```

インストールが完了したら、`cookie-parser`ミドルウェアをグローバルミドルウェアとして適用する（例えば、main.ts内で）。

```ts
import * as cookieParser from 'cookie-parser';
// 初期化ファイルがあるどこか
app.use(cookieParser());
```

いくつかのオプションを`cookieParser`ミドルウェアに渡せる。

- `secret` クッキーの書名に使用する文字列または配列。これは省略可能で、指定されない場合は署名されたクッキーを解析しない。文字列が指定された場合、これはシークレットとして扱われる。配列した場合は、各シークレットを順番に使ってクッキーの書名を解除しようとする。
- `options` 2番めのオプションとしてクッキーに渡されるオブジェクト。詳しくは[cookie](https://www.npmjs.com/package/cookie)を参照のこと。

ミドルウェアはリクエストの`Cookie`ヘッダを解析し、cookieデータを`req.cookies`プロパティとして、またシークレットが提供されている場合は`req.signedCookies`プロパティとして公開する。これらのプロパティは、クッキー名とクッキーの値のname valueペアだ。

secretが提供されると、このモジュールは署名されたクッキーの値の書名解除と検証を行い、そのname valueペアを`req.cookies`から`req.signedCookies`に移動させる。署名付きクッキーとは、値の前に`s:`がついているクッキーの事だ。書名の検証に失敗した署名付きクッキーは、改ざんされた値の代わりに`false`の値を持つ。

結果、以下のようにルートハンドラ内からクッキーを読み取れるようになった。

```ts
@Get()
findAll(@Req() request: Request) {
  console.log(request.cookies); // か"request.cookies['cookieKey']"
  // もしくはconsole.log(request.signedCookies);
}
```

>HINT  
>`@Req()`デコレータは`@nestjs/common`から、`Request`は`express`パッケージからインポートされている。

発信するレスポンスにクッキーを添付するには、`Response#cookie()`メソッドを使用する。

```ts
@Get()
findAll(@Res({ passthrough: true }) response: Response) {
  response.cookie('key', 'value')
}
```

>WARNING  
>レスポンス処理のロジックをフレームワークに任せたい場合は、上記のように`passthrough`オプションを`true`に忘れずに設定する事。詳しくは[こちら](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-controllers#%E3%83%A9%E3%82%A4%E3%83%96%E3%83%A9%E3%83%AA%E3%83%BC%E5%9B%BA%E6%9C%89%E3%82%A2%E3%83%97%E3%83%AD%E3%83%BC%E3%83%81)

>HINT  
>`@Res`デコレータは`@nestjs/common`から、`Response`は`express`パッケージからインポートされている。

## Fastifyとの併用

まず、必要なパッケージをインストールする。

```
$ npm i fastify-cookie
```

インストールが完了したら、`fastify-cookie`プラグインを登録する。

```ts
import fastifyCookie from 'fastify-cookie';

// 初期化ファイルがあるどこか
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
app.register(fastifyCookie, {
  secret: 'my-secret', // クッキーの署名の為のもの
});
```

こうすれば、ルートハンドラ内からクッキーを読み取れる。

```ts
@Get()
findAll(@Req() request: FastifyRequest) {
  console.log(request.cookies); // か "request.cookies['cookieKey']"
}
```

>HINT  
>`@Req()`デコレータは`@nestjs/common`、`FastifyRequest`は`fastify`パッケージからインポートされている。

発信するレスポンスにクッキーを添付するには、`FastifyReply#setCookie()`メソッドを使う。

```ts
@Get()
findAll(@Res({ passthrough: true }) response: FastifyReply) {
  response.setCookie('key', 'value')
}
```

`FastifyReply#setCookie()`の詳細は[こちら](https://github.com/fastify/fastify-cookie#sending)

>WARNING  
>レスポンス処理のロジックをフレームワークに任せたい場合は、上記のように`passthrough`オプションを`true`に忘れずに設定する事。詳しくは[こちら](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-controllers#%E3%83%A9%E3%82%A4%E3%83%96%E3%83%A9%E3%83%AA%E3%83%BC%E5%9B%BA%E6%9C%89%E3%82%A2%E3%83%97%E3%83%AD%E3%83%BC%E3%83%81)

>HINT  
>`@Res()`デコレータは`@nestjs/common`から、`FastifyReply`は`fastify`パッケージからインポートされている。

## カスタムデコレータの作成（クロスプラットフォーム）

受信したクッキーにアクセスする為の便利な宣言的方法を提供する為に、[カスタムデコレータ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-customroutedecorators)を作成する。

```ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const Cookies = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return data ? request.cookies?.[data] : request.cookies;
  },
);
```

`@Cookies()`デコレータは`res.cookies`オブジェクトからすべてのcookie、または名前付きcookieを抽出し、その値をデコレーションされた値に入力する。

これで、次のようにルートハンドラのシグネチャでデコレータを使用できるようになった。

```ts
@Get()
findAll(@Cookies('name') name: string) {}
```