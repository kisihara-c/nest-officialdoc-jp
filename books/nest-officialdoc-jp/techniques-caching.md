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
>`Cache`区rサウは、`@nestjs/common`パッケージの`CACHE_MANAGER`トークンを使用して、`cache-manager`からインポートされる。

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