---
title: fundamentals-asyncproviders
---

# 非同期プロバイダ
時には一つかそれ以上の非同期タスクが終わるまでアプリケーションの開始を遅らせる必要が出る。例えば、データベースとの接続が確立されるまではリクエストの受付を開始したくないかもしれない。これは非同期プロバイダを使う事で実現できる。

その為の構文として、`useFactory`で`async/await`を使おう。ファクトリーは`Promise`を返す。そしてファクトリー関数が非同期タスクを`await`する事ができる。Nestはそういったプロバイダに依存する（インジェクションしている）あらゆるクラスをインスタンス化する前に、プロミスの解決を待ち構える事ができる。

```ts
{
  provide: 'ASYNC_CONNECTION',
  useFactory: async () => {
    const connection = await createConnection(options);
    return connection;
  },
}
```

> HINT
> カスタムプロバイダの構文の詳細はcustom-providersの項目にて。

## インジェクション
非同期プロバイダは、他のプロバイダと同様、持つトークンによって他のコンポーネントに注入される。上の例では`pInject'ASYNC_CONNECTION'`という記述を使用している。

## 例
recipes　SQL(TypeORM)の章で、より充実した非同期プロバイダの例を用意している。