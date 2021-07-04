---
title: techniques-serversentevents
---

# サーバーセントイベント

Server-Sent Events（SSE）はサーバープッシュ技術の一つで、HTTP接続を介した、サーバーによるクライアント自動更新を可能とする。各通知は改行で終わるテキストのブロックとして送信される（[詳細](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)）。

## 使用

ルート（**コントローラクラス**に登録されたルート）でServer-sentイベントを起こすには、`@Sse`デコレータでメソッドハンドラをアノテートする。

```ts
@Sse('sse')
sse(): Observable<MessageEvent> {
  return interval(1000).pipe(map((_) => ({ data: { hello: 'world' } })));
}
```

>HINT  
>`@Sse()`デコレータは`@nestjs/common`から、`Observable`、`interval`、`and map`は`rxjs`パッケージからインポートされている。

>WARNING  
>サーバーセントイベントは`Observable`ストリームを返す必要がある。

上の例では、`sse`という名前のルートを定義して、リアルタイムの更新を伝播できるようにしている。これらのイベントは[EventSource API](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)を使ってリッスンする。

`sse`メソッドは、複数の`MessageEvent`を発行する`Observable`を返す（この例では1秒ごとに新しい`MessageEvent`を発行）。`MessageEvent`オブジェクトは、仕様に合わせて以下のインターフェイスを尊重（respect）する必要がある。

```ts
export interface MessageEvent {
  data: string | object;
  id?: string;
  type?: string;
  retry?: number;
}
```

これで、クライアントサイドにアプリケーションで`EventSource`クラスのインスタンスを作成し、コンストラクタの引数に`/sse`ルート（上記の`@Sse`デコレータに渡したエンドポイントと一致）を渡すことができる。

`EventSource`のインスタンスは、HTTPサーバーへの持続的な接続を開き、イベントを`text/event-stream`形式で送信する。この接続は`EventSource.close()`をコールして閉じるまで開いたままとなる。

接続が開かれると、サーバからの受信メッセージがイベントの形でコードに配信される。受信メッセージにイベント・フィールドがある場合トリガーされるイベントはイベントフィールドの値と同じものだ。イベントフィールドが存在しない場合は、一般的な`message`イベントが発生する（[情報元](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)）。

```ts
const eventSource = new EventSource('/sse');
eventSource.onmessage = ({ data }) => {
  console.log('New message', JSON.parse(data));
};
```

## サンプル
動くサンプルは[こちら](https://github.com/nestjs/nest/tree/master/sample/28-sse)