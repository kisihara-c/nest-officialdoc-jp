---
title: techniques-session
---

# セッション

**HTTPセッション**を使うと、複数のリクエスト間でユーザーに関する情報を保存できる。これはMVCアプリケーションでは特に便利だ。

## expressで使う（デフォルト）

まず必要なパッケージをインストールしよう。TypeScriptユーザーの場合は型も導入しよう。

```
$ npm i express-session
$ npm i -D @types/express-session
```

インストールが完了したら、ミドルウェア`express-session`をグローバルミドルウェアとして適用する。（`main.ts`の中など）

```ts
import * as session from 'express-session';
// 初期化ファイルの中のどこか
app.use(
  session({
    secret: 'my-secret',
    resave: false,
    saveUninitialized: false,
  }),
);
```

>NOTICE  
>デフォルトのサーバーサイドセッションストレージは本番環境向けには設計されていない。ほとんどの環境でメモリリークを起こし、単一のプロセスを超えて拡張する事はできない。デバッグ・開発用だ。詳しくは[公式リポジトリ](https://github.com/expressjs/session)にて。

`secret`はセッションIDクッキーの署名に使用される。単一の秘密の文字列か、複数の秘密の配列を指定するものだ。秘密の配列を指定した場合、最初の要素だけがセッションIDクッキーの署名に使用され、リクエストの署名を検証するときにすべての要素が使われる。secretは人間が簡単に解析できないランダム文字の集合である事が望ましい。

`resave`オプションを有効にすると、リクエスト中にセッションが変更されていなくても、セッションストアにセッションが保存される。デフォルト値は`true`だが、将来的にデフォルト値が変更される予定なので、デフォルト値の仕様は推奨されない。

同様に、`saveUnitialized`オプションを有効にすると、「初期化されていない」セッションがストアに保存される。初期化されていないセッションとは、新しく作成されたが変更されていないセッションのことだ。`false`を選択すると、ログインセッションの実装、サーバーのストレージ使用量の削減、またはCookieの設定に許可を求める法律への準拠に役立つ。また、`false`を選択すると、クライアントがセッションを持たずに複数のリクエストを並行して行うような競合状態を[回避できる](https://github.com/expressjs/session#saveuninitialized)。

`session`ミドルウェアには、他にもいくつかの引数を渡す事ができる。[API documentation](https://github.com/expressjs/session#options)を参照の事。

>HINT  
>`secure:true`設定を推奨する。ただし、これにはhttps対応のウェブサイトが必要となる。secureが設定されていても、HTTPでサイトにアクセスするとクッキーは設定されない。node.jsをプロキシの背後において`secure:true`を使っている場合は、expressで`trust proxy`を設定する必要がある。

これで、ルートハンドラ内からセッション値の設定や読み出しができるようになった。

```ts
@Get()
findAll(@Req() request: Request) {
  request.session.visits = request.session.visits ? request.session.visits + 1 : 1;
}
```

>HINT  
>`@session()`デコレータは`@nestjs/common`パッケージからインポートする。

## Fastlyで使う

まず必要なパッケージをインストールしよう。

```
$ npm i fastify-secure-session
```

インストールが完了したら、`fastify-secure-session`プラグインを登録する。

```ts
import secureSession from 'fastify-secure-session';

// 初期化ファイルのどこか
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter(),
);
app.register(secureSession, {
  secret: 'averylogphrasebiggerthanthirtytwochars',
  salt: 'mq9hDxBVDbspDR6n',
});
```

>HINT  
>鍵を事前生成したり（[instructions](https://github.com/fastify/fastify-secure-session)）、
[キーローテーション](https://github.com/fastify/fastify-secure-session#using-keys-with-key-rotation)を使うこともできる。

追加オプションは[公式リポジトリ](https://github.com/fastify/fastify-secure-session)を参照の事。

これで、次のようにルートハンドラ内でセッション値を設定したり読み出したりできるようになった。

```ts
@Get()
findAll(@Req() request: FastifyRequest) {
  const visits = request.session.get('visits');
  request.session.set('visits', visits ? visits + 1 : 1);
}
```


あるいは、`@session()`デコレータを使って、以下のようにリクエストからセッションオブジェクトを抽出することもできる。

```ts
@Get()
findAll(@Session() session: secureSession.Session) {
  const visits = session.get('visits');
  session.set('visits', visits ? visits + 1 : 1);
}
```

>HINT  
>`@Session()`デコレータは`@nestjs/common`から、`secureSession.Session`は`fastify-secure-session`パッケージからインポートされている。※`import * as secureSession from 'fastify-secure-session'`