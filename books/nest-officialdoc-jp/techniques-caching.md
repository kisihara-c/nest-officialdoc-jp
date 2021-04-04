---
title: techniques-caching
---

# キャッシュ

キャッシュは貴方のアプリのパフォーマンスを向上させるグレートでシンプルな技術だ。それは一時的なデータストアとして機能し、ハイパフォーマンスなデータアクセスを実現する。

## インストール

まず必要なパッケージをインストールしよう。

```
$ npm install cache-manager
$ npm install -D @types/cache-manager
```

## インメモリキャッシュ

Nestは様々なキャッシュストレージプロバイダのための統一されたAPIを提供している。ビルトインされているのは、インメモリデータストアだ。しかし、Redisのようなより包括的なソリューションに簡単に切り替えることができる。

キャッシュを有効にするには、`CacheModule`をインポートして、その`register()`メソッドを呼び出す。

```ts
import { CacheModule, Module } from '@nestjs/common';
import { AppController } from './app.controller';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
})
export class AppModule {}
```

## キャッシュストアと対話

キャッシュマネージャのインスタンスを操作するには、以下のように`CACHE_MANAGER`トークンを使用してクラスにインジェクションする。

```ts
constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}
```

>HINT  
>`Cache`クラスは、`@nestjs/common`パッケージの`CACHE_MANAGER`トークンを使用して、`cache-manager`からインポートされる。

（`chache-manager`パッケージの）`Cache`インスタンスの`get`メソッドは、キャッシュから項目を取得するために使用される。項目がキャッシュに存在しない場合は、例外がスローされる。

```ts
const value = this.cacheManager.get('key');
```

アイテムをキャッシュに追加するには、`set`メソッドを使用する。

```ts
await this.cacheManager.set('key', 'value');
```

キャッシュのデフォルトの有効期限は5秒だ。

以下のように、この特定のキーのTTL（有効期限）を手動で指定することができる。

```ts
await this.cacheManager.set('key', 'value', { ttl: 1000 });
```

キャッシュの有効期限を無効にするには、設定プロパティ`ttl`を`null`にする。

```ts
await this.cacheManager.set('key', 'value', { ttl: null });
```

キャッシュからアイテムを削除するには、`del`メソッドを使う。

```ts
await this.cacheManager.del('key');
```

キャッシュ全体をクリアするには、`reset`メソッドを使う。

```ts
await this.cacheManager.reset();
```

## オートキャッシングレスポンス

>WARNING  
>GraphQLアプリケーションでは、インターセプタはフィールドリゾルバごとに個別に実行される。そのため、インターフェプタを使用してレスポンスをキャッシュする`CacheModule`は正しく動作しない。

オートキャッシングレスポンスを有効にするには、データをキャッシュしたい場所に`CacheInterceptor`を結びつけるだけだ。

```ts
@Controller()
@UseInterceptors(CacheInterceptor)
export class AppController {
  @Get()
  findAll(): string[] {
    return [];
  }
}
```

>WARNING  
>`GET`エンドポイントのみがキャッシュされる。また、ネイティブレスポンスオブジェクト（`@Res()`）をインジェクションするHTTPサーバルートではキャッシュインターセプターが使えない。詳細は[こちら](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-interceptors#%E3%83%AC%E3%82%B9%E3%83%9D%E3%83%B3%E3%82%B9%E3%81%AE%E3%83%9E%E3%83%83%E3%83%94%E3%83%B3%E3%82%B0)

必要なボイラープレートの量を減らす為、`CacheInterceptor`をすべてのエンドポイントにグローバルにバインドできる。

```ts
import { CacheModule, Module, CacheInterceptor } from '@nestjs/common';
import { AppController } from './app.controller';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  imports: [CacheModule.register()],
  controllers: [AppController],
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: CacheInterceptor,
    },
  ],
})
export class AppModule {}
```

## キャッシュのカスタマイズ

すべてキャッシュされたデータは、独自の有効期限（TTL）を持つ。デフォルトの設定をカスタマイズするには、`register()`メソッドにオプションオブジェクトを通す。

```ts
CacheModule.register({
  ttl: 5, // 秒
  max: 10, // キャッシュされるアイテムの最大数
});
```

## グローバルキャッシュのオーバーライド

グローバルキャッシュが有効な場合、キャッシュエントリは、ルートパスに基づいて自動生成された`CacheKey`の下に保存される。特定のキャッシュ設定（`@ChacheKey()`および`@ChacheTTL()`）をメソッドごとにオーバーライドする事で、個々のコントローラメソッドのキャッシュ方針をカスタマイズすることができる。これは異なるキャッシュストアを使用している場合に有効となる。

```ts
@Controller()
export class AppController {
  @CacheKey('custom_key')
  @CacheTTL(20)
  findAll(): string[] {
    return [];
  }
}
```

>HINT  
>`@CacheKey`と`@CacheTTL`デコレータは`@nestjs/common`パッケージからインポートされている。

`@CacheKey()`デコレータは対応する`@CacheTTL()`デコレータと一緒に使ってもいいし、使わなくても良い。それぞれを単独で使う事もできる。デコレータでオーバーライドされていない場合は、グローバルに登録されているデフォルト値が使用される（前段「キャッシュのカスタマイズ」を参照の事）。

## WebSocketsとマイクロサービス

キャッシュインターセプターは、使用されているトランスポート方法に関わらず、WebSocketサブスクライバやマイクロサービスのパターンにも適用できる。

```ts
@CacheKey('events')
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
```

しかしながら、キャッシュされたデータを保存・取得するためのキーを指定するには、追加で`@CacheKey()`デコレータが必要だ。また、全てを全てキャッシュすべきでない事も注意。単なるデータのクエリではなく、何らかのビジネスオペレーションを行うアクションは、キャッシュされてはならないだろう。

さらに、`@CacheTTL()`デコレータでキャッシュの有効期限（TTL）を指定する事もできる。これはグローバルなデフォルトのTTLの値を上書きする。

```ts
@CacheTTL(10)
@UseInterceptors(CacheInterceptor)
@SubscribeMessage('events')
handleEvent(client: Client, data: string[]): Observable<string[]> {
  return [];
}
```

>HINT  
>`CacheTTL()`デコレータは、対応する`@CacheKey()`デコレータと一緒に、もしくは単独で使用することができる。

## トラッキングの調整

デフォルトでは、Nestは（HTTPアプリでは）リクエストのURL、または（websocketsとマイクロサービスのアプリで`@CacheKey()`デコレータで設定した場合）キャッシュキーを使用して、キャッシュレコードとエンドポイントを関連付ける。とはいえ、HTTPヘッダー（プロファイルエンドポイントを適切に識別する為の`Authorization`など）を使用するなど、異なる要素に基づいてトラッキングを設定したい場合もあるだろう。

これを実現するには、`CacheInterceptor`のサブクラスを作成し、`trackBy()`メソッドをオーバーライドする。

```ts
@Injectable()
class HttpCacheInterceptor extends CacheInterceptor {
  trackBy(context: ExecutionContext): string | undefined {
    return 'key';
  }
}
```

## 別のストア

このサービスは内部で[cache-manager](https://github.com/BryanDonovan/node-cache-manager)を使用している。cache-managerパッケージは、例えば[Redis](https://github.com/dabroek/node-cache-manager-redis-store)のような便利なストアを幅広くサポートしている。サポートされているストアの完全なリストは[こちら](https://github.com/BryanDonovan/node-cache-manager#store-engines)。Redisをセットアップするには、パッケージと対応するオプションを`register()`メソッドに渡すだけだ。

```ts
import * as redisStore from 'cache-manager-redis-store';
import { CacheModule, Module } from '@nestjs/common';
import { AppController } from './app.controller';

@Module({
  imports: [
    CacheModule.register({
      store: redisStore,
      host: 'localhost',
      port: 6379,
    }),
  ],
  controllers: [AppController],
})
export class AppModule {}
```

## 非同期設定

モジュールのオプションをコンパイル時に静的に渡すのではなく、非同期で渡したい場合がある。この場合`registerAsync()`メソッドを使う。このメソッドには、非同期瀬底を扱う方法がいくつか用意されている。

１つの方法は、ファクトリ関数を使用する事だ。

```ts
CacheModule.registerAsync({
  useFactory: () => ({
    ttl: 5,
  }),
});
```

このファクトリは、他の非同期モジュールファクトリと同様に動作する。`async`化できて、`inject`で依存性をインジェクションできる。

```ts
CacheModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    ttl: configService.get('CACHE_TTL'),
  }),
  inject: [ConfigService],
});
```

代わりに、`useClass`メソッドを使用する事もできる。

```ts
CacheModule.registerAsync({
  useClass: CacheConfigService,
});
```

上記のコードでは、`CacheModule`内に`CacheConfigService`をインスタンス化し、これを使用してオプションオブジェクトを取得する。`CacheConfigService`は、構成オプションを提供するために、`CacheOptionsFactory`インターフェイスを実装する必要がある。

```ts
@Injectable()
class CacheConfigService implements CacheOptionsFactory {
  createCacheOptions(): CacheModuleOptions {
    return {
      ttl: 5,
    };
  }
}
```

別のモジュールからインポートされた既存の構成プロバイダを使用したい場合は、`useExisting`構文を使用する。

```ts
CacheModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

`CacheModule`は、自分自身のインスタンスを作成するのではなく、インポートされたモジュールを検索し、作成済みの`ConfigService`を再利用する。

## サンプル
動作するサンプルは[こちら](https://github.com/nestjs/nest/tree/master/sample/20-cache)