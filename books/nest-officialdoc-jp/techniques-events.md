---
title: techniques-events
---

# イベント

[event-emitter](https://www.npmjs.com/package/@nestjs/event-emitter)パッケージ（`@nestjs/event-emitter`）はシンプルなオブザーバの実装を提供している。アプリケーションで発生する様々なイベントをサブスクライブしてリッスンする事ができる。イベントはアプリケーションの様々な側面を切り離すのに最適な方法と言える。一つのイベントが、お互いに依存しない複数のリスナーを持てるからだ。

`EventEmitterModule`は内部で[eventemitter2](https://github.com/EventEmitter2/EventEmitter2)パッケージを使用している。

## 導入

まず必要なパッケージをインストールしよう。

```
$ npm i --save @nestjs/event-emitter
```

インストールが終わったら、ルートの`AppModule`に`EventEmitterModule`をインポートし、以下のように`forRoot()`静的メソッドを実行する。

```ts :app.module.ts 
import { Module } from '@nestjs/common';
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [
    EventEmitterModule.forRoot()
  ],
})
export class AppModule {}
```

`.fooRoot()`の呼び出しはイベントエミッタを初期化し、アプリ内に存在する宣言型のイベントリスナーを登録する。登録は`onApplicationBootstrap`ライフサイクルフックが発生した時に行われ、すべてのモジュールがロードされ、スケジュールされたジョブが宣言されている事を確認する。

基礎となる`EventEmitter`インスタンスを構成するために、以下のように構成オブジェクトを`.fooRoot()`メソッドに渡す。

```ts
EventEmitterModule.forRoot({
  // `true`にするとワイルドカードを使う
  wildcard: false,
  // 名前空間分割のために使われるデリミタ
  delimiter: '.',
  // `true`にするとnewListenerイベントを発行する
  newListener: false,
  // `true`にするとremoveListenerイベントを発行する
  removeListener: false,
  // 1つのイベントに割り当てられるリスナーの最大数
  maxListeners: 10,
  // 最大数以上のリスナーが割り当てられた時、メモリリークメッセージにイベント名を表示する
  verboseMemoryLeak: false,
  // リスナーがいないエラーイベントが発生した際uncaughtExceptionをスローする動作を無効にする
  ignoreErrors: false,
});
```

## イベントのディスパッチ

イベントをディスパッチ（発火）するには、まず標準的なコンストラクタインジェクションを使用して`EventEmitter2`を注入する。

```ts
constructor(private eventEmitter: EventEmitter2) {}
```

>HINT  
>`EventEmitter2`は`@nestjs/event-emitter`パッケージからインポートする。

そしてクラス内で使用する。

```ts
this.eventEmitter.emit(
  'order.created',
  new OrderCreatedEvent({
    orderId: 1,
    payload: {},
  }),
);
```

## イベントのリッスン

イベントリスナーを宣言するには、次のように実行されるコードを含むメソッド定義の前に`@OnEvent()`デコレータをつけてメソッドを装飾する。

```ts
@OnEvent('order.created')
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  // "OrderCreatedEvent"イベントを操作・処理する。
}
```

>WARNING  
>イベントサブスクライバはリクエストスコープ化できない。

最初の引数は、単純なイベントエミッタの場合`string`または`symbol`、ワイルドカードエミッタの場合は`string`か`symbol`か`Array<string|symbol>`となる。2番目の引数（省略可能）は、リスナーのオプションオブジェクトだ（[説明](https://github.com/EventEmitter2/EventEmitter2#emitteronevent-listener-options-objectboolean)）。

```ts
@OnEvent('order.created', { async: true })
handleOrderCreatedEvent(payload: OrderCreatedEvent) {
  //"OrderCreatedEvent"イベントを操作・処理する。
}
```

名前空間/ワイルドカードを使用するには、`EventEmitterModule#forRoot()`メソッドに`wildcard`オプションを渡す。名前空間/ワイルドカードを有効にすると、イベントはデリミタで区切られた文字列（`foo.bar`）か配列（`['foo', 'bar']`）となる。区切り文字は設定プロパティ（`delimiter`）として設定する事も可能。名前空間の機能を有効にすると、ワイルドカードを使ってイベントをサブスクライブする事ができる。

```ts
@OnEvent('order.*')
handleOrderEvents(payload: OrderCreatedEvent | OrderRemovedEvent | OrderUpdatedEvent) {
  // イベントを操作・実行する。
}
```

こういったワイルドカードは１つのブロックにしか適用されないことに注意。引数`order.*`は、`order.created`や`order.shipped`には適用されるが、`order.delayed.out_of_stock`には適用されない。そういったイベントをリッスンするためには`multilevel wildcard`パターン（例：`**`）を使用してほしい（[説明](https://github.com/EventEmitter2/EventEmitter2#multi-level-wildcards)）。

例えばこのパターンを使用すると、全てのイベントを拾うイベントリスナーを作成できる。

```ts
@OnEvent('**')
handleEverything(payload: any) {
  // イベントを操作・実行する
}
```

>HINT  
>`EventEmitter2`クラスには`waitFor`や`onAny`など、イベントを処理するための便利なメソッドがいくつか用意されている。[詳細](https://github.com/EventEmitter2/EventEmitter2)

## サンプル
実際に動くサンプルは[こちら](https://github.com/nestjs/nest/tree/master/sample/30-event-emitter)