---
title: techniques-httpmodule
---

# HTTPモジュール

[Axios](https://github.com/axios/axios)は豊富な機能を持ち、広く使われているHTTPクライアントパッケージだ。NestはAxiosをラップして、組み込みの`HttpModule`として使用可能な状態にしている。`HttpModule`は、HTTPリクエストを実行するAxiosベースのメソッドを公開する`HttpService`クラスをエクスポートする。また、このライブラリは返ってきたHTTPレスポンスを`Observables`に変換する。

`HttpService`を使用するには、まず`HttpModule`をインポートする。

```ts
@Module({
  imports: [HttpModule],
  providers: [CatsService],
})
export class CatsModule {}
```

次に、通常のコンストラクタインジェクションを使って`HttpService`をインジェクションする。

> HINT  
> `HttpModule`と`HttpService`は`@nestjs/common`パッケージからインポートされている。

```ts
@Module({
  imports: [HttpModule],
  providers: [CatsService],
})
export class CatsModule {}
```

すべての`HttpService`メソッドは、`Axiosresponse`を`Observable`オブジェクトでラップして返す。

## 設定

Axiosは`HttpService`の動作をカスタマイズするために、様々なオプションを設定することができる。詳細は[こちら](https://github.com/axios/axios#request-config)。基盤となるAxiosインスタンスを設定するには、`HttpModule`のインポート時に、省略可能なオプションオブジェクトを`register()`メソッドに渡す。このオプションオブジェクトは、背後のAxiosコンストラクタに直接渡される。

```ts
@Module({
  imports: [
    HttpModule.register({
      timeout: 5000,
      maxRedirects: 5,
    }),
  ],
  providers: [CatsService],
})
export class CatsModule {}
```

## 非同期設定

モジュールのオプションを非同期に渡す必要がある場合は、`registerAsync()`メソッドを使用する。ほとんどの動的モジュールと同様に、いくつかの手法で非同期設定を扱える。

一つはファクトリー関数を使う事。

```ts
HttpModule.registerAsync({
  useFactory: () => ({
    timeout: 5000,
    maxRedirects: 5,
  }),
});
```

他のファクトリープロバイダのように、ファクトリー関数は[async](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-customproviders#%E3%83%95%E3%82%A1%E3%82%AF%E3%83%88%E3%83%AA%E3%83%BC%E3%83%97%E3%83%AD%E3%83%90%E3%82%A4%E3%83%80%E3%80%81usefactory)にできるし、`inject`を使って依存性インジェクションを行える。

```ts
HttpModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    timeout: configService.getString('HTTP_TIMEOUT'),
    maxRedirects: configService.getString('HTTP_MAX_REDIRECTS'),
  }),
  inject: [ConfigService],
});
```

他の手法としては、ファクトリーではなくクラスを使用する事もできる。

```ts
HttpModule.registerAsync({
  useClass: HttpConfigService,
});
```

上記のコードでは、`HttpModule`内で`HttpConfigService`をインスタンス化する事でオプションオブジェクトを作成している。この例では、`HttpConfigService`は以下のように`HttpOptionsfactory`インターフェイスを実装する必要がある事に注意。`httpModule`は提供されたクラスがインスタンス化されたオブジェクトの`createhttpOptions()`メソッドを呼び出す。

```ts
@Injectable()
class HttpConfigService implements HttpModuleOptionsFactory {
  createHttpOptions(): HttpModuleOptions {
    return {
      timeout: 5000,
      maxRedirects: 5,
    };
  }
}
```

`HttpModule`内にプライベートな複製を作成するのではなく、既存のオプションプロバイダを再利用したい場合は、`useExisting`構文を使用する。

```ts
HttpModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```