---
title: security-ratelimiting
---

# Rate Limiting

ブルートフォース攻撃からアプリケーションを守る為の一般的な手法としてレート制限が挙げられる。まず`@nestjs/throttler`パッケージをインストールする必要がある。

```
$ npm i --save @nestjs/throttler
```

インストールが完了すると、`ThrottlerModule`は`forRoot`や`forRootAsync`メソッドを持つ他のNestパッケージと同様に設定できる。

```ts
@Module({
  imports: [
    ThrottlerModule.forRoot({
      ttl: 60,
      limit: 10,
    }),
  ],
})
export class AppModule {}
```

上記のコードは、ガードされるアプリケーションのルートに対して`ttl`(time to live)や`limit`(ttl内のリクエストの最大数)のグローバルオプションを設定するものだ。

モジュールがインポートされたら、`ThlottlerGuard`をどのようにバインドするかを設定できる。[ガード](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-guards)のセクションで述べたあらゆる種類のバインドが可能だ。例えばガードをグローバルにバインドしたい場合は次のようなプロバイダを任意のモジュールに追加してほしい。

```ts
{
  provide: APP_GUARD,
  useClass: ThrottlerGuard
}
```

## カスタマイズ

ガードをコントローラやグローバルにバインドしたいが、１つまたは複数のエンドポイントのレート制限を無効化したい場合があるかもしれない。そういった場合には`@SkipThrottle()`デコレータを使用してクラス全体または単一のルートに対してThrottlerを無効化できる。`@SkipThrottle()`にはブーリアン値を指定する事もできる。コントローラの大部分は除外したいものの、全てのルートを除外したいわけではない時に便利だ。

また`@Throttle()`デコレータも使える。これはグローバルモジュールの`limit`、`ttl`の設定を上書きする為に使うもので、セキュリティオプションを厳しくしたり緩くしたりできる。このデコレータはクラスや関数にも使える。デコレータの引数が`limit`、`ttl`の順である事には注意。

## Websockets

このモジュールはWebsocketsを扱えるが、いくつかのクラスの拡張が必要となる。`ThrottlerGuard`を拡張して`handleRequest`メソッドをオーバーライドできる。

```ts
@Injectable()
export class WsThrottlerGuard extends ThrottlerGuard {
  async handleRequest(context: ExecutionContext, limit: number, ttl: number): Promise<boolean> {
    const client = context.switchToWs().getClient();
    const ip = client.conn.remoteAddress; 
    const key = this.generateKey(context, ip);
    const ttls = await this.storageService.getRecord(key);

    if (ttls.length >= limit) {
      throw new ThrottlerException();
    }

    await this.storageService.addRecord(key, ttl);
    return true;
  }
}
```

>HINT  
>もし`@nestjs/platform-ws` パッケージを使っている場合は`client._socket.remoteAddress`を代わりに使える。

## GraphQL

`ThrottlerGuard`はGraphQLリクエストにも使える。ガードの拡張もできるが、今回は`getRequestResponse`メソッドをオーバーライドしよう。

```ts
@Injectable()
export class GqlThrottlerGuard extends ThrottlerGuard {
  getRequestResponse(context: ExecutionContext) {
    const gqlCtx = GqlExecutionContext.create(context);
    const ctx = gql.getContext();
    return { req, ctx.req, res: ctx.res }
  }
}
```

## 設定

`ThrottlerModule`では以下のオプションが有効だ。

|||
| ---- | ---- |
|`ttl`|各リクエストがストレージされる時間の秒数|
|`limit`|ttl制限内の最大リクエスト数|
|`ignoreUserAgents`|リクエストをThrottleする際に無視するユーザエージェントの正規表現の配列|
|`storage`|リクエストを追跡する為のストレージの設定|

## 非同期設定

レート制限の設定を、同期ではなく非同期で取得したい場合がある。`forRootAsync()`メソッドを使用すると、依存性インジェクションや``async`メソッドを使用できる。

ファクトリー関数を使うのが一つのアプローチだ。

```ts
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        ttl: config.get('THROTTLE_TTL'),
        limit: config.get('THROTTLE_LIMIT'),
      }),
    }),
  ],
})
export class AppModule {}
```

`useClass`シンタックスも使える。

```ts
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      useClass: ThrottlerConfigService,
    }),
  ],
})
export class AppModule {}
```

`ThrottlerConfigService`が`ThrottlerOptionsFactory`インターフェイスを実装していれば可能だ。

## ストレージ

組み込みのストレージは、グローバルオプションで設定されたttlを過ぎるまでのリクエストを追跡するインメモリーキャッシュだ。`ThrottlerStorage`インターフェイスを実装したクラスであれば、`ThrottlerModule`の`storage`オプションに、独自の`storage`オプションをドロップインできる。

>Note  
>`ThrottlerStorage`は`@nestjs/throttler`からインポートできる。