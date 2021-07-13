---
title: security-cors
---

# cors
CORS（Cross-Origin Resource Sharing）とは、別のドメインからリソースを要求できるようにするメカニズムだ。NestではExpressの[cors](https://github.com/expressjs/cors)パッケージを使用している。このパッケージには様々なオプションが用意されており、必要に応じてカスタマイズできる。

## 始める
CORSを有効にするには、Nestアプリケーションオブジェクトの`enableCors()`メソッドを呼び出す。

```ts
const app = await NestFactory.create(AppModule);
app.enableCors();
await app.listen(3000);
```

`enableCors()`メソッドは、オプションで設定オブジェクトの引数を取る。このオブジェクトの利用可能なプロパティについては、[CORS](https://github.com/expressjs/cors#configuration-options)の公式ドキュメントに記載されている。また、[コールバック関数](https://github.com/expressjs/cors#configuring-cors-asynchronously)を渡す事で、リクエストに応じて（動的に）非同期に設定オブジェクトを定義することもできる。

`create()`メソッドのオプションオブジェクトでもCORSを有効にできる。デフォルトの設定でCORSを有効にするには、`cors`プロパティを`true`に設定する。また、`cors`プロパティの値として、[CORS設定オブジェクト](https://github.com/expressjs/cors#configuration-options)や[コールバック関数](https://github.com/expressjs/cors#configuring-cors-asynchronously)を渡して動作をカスタマイズする事もできる。

```ts
const app = await NestFactory.create(AppModule, { cors: true });
await app.listen(3000);
```

前述の方法はRESTエンドポイントにのみ適用される。

GraphQLでCORSを有効にするには、`cors`プロパティを`true`に設定するか、GraphQLモジュールのインポート時に`cors`プロパティの値として[CORS設定オブジェクト](https://github.com/expressjs/cors#configuration-options)または[コールバック関数](https://github.com/expressjs/cors#configuring-cors-asynchronously)を渡してほしい。

>WARNING 
>`CorsOptionsDelegate`ソリューションは`apollo-server-fastify`パッケージではまだ動かない。

```ts
GraphQLModule.forRoot({
  cors: {
    origin: 'http://localhost:3000',
    credentials: true,
  },
}),
```