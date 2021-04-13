---
title: techniques-queues
---

## キュー

キューはアプリケーションのスケーリングやパフォーマンスに関する一般的な問題に対処する為、役立つ強力なデザインパターンだ。キューを使って解決可能な問題の例をいくつか挙げてみよう。

- 処理のピークをスムーズにする。例えば、ユーザがリソースを必要とするタスクを任意の時間に開始できる場合、これらのタスクを同時的に実行するのではなく、キューに追加する事ができる。そして、制御された方法で、ワーカープロセスにキューからタスクを引き出させる事ができる。アプリケーションのスケールアップに合わせて、新しいキューの解消者を追加し、バックエンドのタスク処理を拡張する事も簡単だ。
- Node.jsのイベントループをブロックしてしまうような一枚岩のタスクを分割する。例えば、ユーザのリクエストがオーディオnのトランスコーディングのようなCPU負荷の高い作業を必要とする場合、このタスクを他のプロセスに委ねることができ、ユーザに接するプロセスを解放して応答性を維持することができる。
- 様々なサービス間で信頼性の高い通信チャネルを提供する。例えば、あるプロセスやサービスでタスク（ジョブ）をキューに入れ、別のプロセスやサービスでそれを消費する事ができる。ジョブのライフサイクルに置いて、完了、エラー、その他の状態変化が合った場合には、どのプロセスやサービスからでも（ステータスイベントをlistenする事で）通知を受ける事ができる。キューのプロデューサーやコンシューマーに障害が発生しても、状態は保持されて、nodeの再起動時にタスク処理を自動的に再開できる。

Nestは人気があってしっかりサポートされている、高性能なNode.jsベースのキューシステム実装　[Bull](https://github.com/OptimalBits/bull)の抽象化/ラッパーとして`@nestjs/bull`パッケージを提供している。このパッケージを使えば、Nestフレンドリーな方法でBull Queuesをアプリケーションに簡単に統合できる。

Bullはジョブデータの永続化に[Redis](https://redis.io/)を使用している為、システムにRedisがインストールされている必要がある。Redisに支えられている事で、Queueのアーキテクチャは完全に分散されてプラットフォームに依存しない。例えば、一部のキューのプロデューサーとコンシューマー、リスナーを一つ（または複数）のノード上のNestで待機させ、他のそれぞれを他のネットワークノード上の他のNode.jsプラットフォームで動作させられる。

本章では`@nestjs/bull`パッケージを説明する。背景や具体的な実装の詳細については[Bullのドキュメント](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md)の参照を勧める。

## インストール

パッケージを使い始める為、まず依存関係をインストールしよう。

```
$ npm install --save @nestjs/bull bull
$ npm install --save-dev @types/bull
```

インストールが終わったら、ルートの`AppModule`に`BullModule`をインポートしよう。

```ts :app.module.ts 
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';

@Module({
  imports: [
    BullModule.forRoot({
      redis: {
        host: 'localhost',
        port: 6379,
      },
    }),
  ],
})
export class AppModule {}
```

`forRoot()`メソッドは、（通常）アプリケーションに登録されているすべてのキューで使用される`bull`のパッケージ設定オブジェクトを登録するために使用される。設定オブジェクトは以下のプロパティで構成される。

- `limiter: RateLimiter` - キューのジョブが処理される速度を制御する為の省略可能な設定。詳細は[RateLimiter](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)にて。
- `redis: RedisOpts` - Redisへの接続を構成する為の省略可能な設定。詳細は[RedisOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)にて。
- `prefix: string` - すべてのキュー・キーのプレフィックス。省略可能。
- `defaultJobOptions: JobOpts` - 新しいジョブのデフォルト設定を制御する為の省略可能な設定。詳細は[JobOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)にて。
- `settings: AdvancedSettings` - 高度なキューの構成設定。省略可能で、通常は変更を推奨されない。詳細は[AdvancedSettings](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)にて。

全てのオプションは、キューの動作を詳細に設定する為の省略可能なものだ。これらはBullの`Queue`コンストラクタにダイレクトに渡される。詳細は[こちら](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)

キューを登録するには、動的モジュール`BullModule#registerQueue()`をインポートして、以下のようにする。

```ts
BullModule.registerQueue({
  name: 'audio',
});
```

>HINT  
>`registerQueue()`メソッドにカンマで区切られた複数の設定オブジェクトを渡すことで、複数のキューを作成してみよう。

`registerQueue()`メソッドは、キューのインスタンス化や登録に使う。キューは同じ認証情報・同じ基礎のRedisデータベースに接続する、モジュールやプロセス間で共有される。各キューは自身のnameプロパティによって一意に特定される。キューの名前は（コントローラやプロパイダにキューをインジェクションするための）トークンとしても、コンシューマクラスやリスナーとキューを関連付けるデコレータの引数としても使われる。

また、以下のように、特定のキューに対して事前に設定されたオプションの一部をオーバーライドする事もできる。

```ts
BullModule.registerQueue({
  name: 'audio',
  redis: {
    port: 6380,
  },
});
```

ジョブはRedisの中で永続化されている為、特定の名前のキューがインスタンス化されるたびに（例えばアプリが起動/再起動された時）以前の未完了セッションから、存在するであろう古いジョブの処理を試みる。

カクキューは、１つまたは複数のプロデューサー、コンシューマー、およびリスナーを持つことができる。コンシューマーは特定の順序でキューからジョブを取得する。FIFO（デフォルト）、LIFO、または各優先順位に従う。実行順の詳細については後のコンシューマーの項にて。

## 名前付き設定

キューが複数の異なるRedisインスタンスに接続している場合、名前付き設定と呼ばれる機能を使用できる。指定したキーの下に複数の構成を登録し、キューのプションで参照することができる。

例えば、アプリケーションに登録されているいくつかのキューが使用するRedisインスタンスが（デフォルトのものとは別に）追加されていると仮定すると、以下のようにその構成を登録できる。

```ts
BullModule.forRoot('alternative-config', {
  redis: {
    port: 6381,
  },
});
```

※`alternative-config`は任意の文字列部分

これで、`registerQueue()`のオプションオブジェクトで、この設定を指定する事ができるようになった。

```ts
BullModule.registerQueue({
  configKey: 'alternative-queue'
  name: 'video',
});
```

## プロデューサー

ジョブプロデューサーは、通常アプリケーションサービス（Nestプロバイダ）で、キューにジョブを追加する。その為にはまず以下のようにキューをサービスにインジェクションする。

```ts
import { Injectable } from '@nestjs/common';
import { Queue } from 'bull';
import { InjectQueue } from '@nestjs/bull';

@Injectable()
export class AudioService {
  constructor(@InjectQueue('audio') private audioQueue: Queue) {}
}
```

>HINT  
>`@InjectQueue()`デコレータは、`registerQueue()`メソッドコールで提供された名前でキューを識別する（例：`audio`）。

ここで、キューのadd()メソッドを呼び出し、ユーザー定義のジョブオブジェクトを渡してジョブを追加する。ジョブはシリアライズ可能なJavaScriptオブジェクトとして表現される（そのままRedisに保存される形）。渡されたジョブの形状は任意で、ジョブオブジェクトのセマンティクスを表現できる。

```ts
const job = await this.audioQueue.add({
  foo: 'bar',
});
```

## 名前付きジョブ

ジョブは一意の名前を持てる。結果、指定された名前のジョブのみを処理する特殊なコンシューマーを作成できる。

```ts
const job = await this.audioQueue.add('transcode', {
  foo: 'bar',
});
```

>WARNING  
>名前付きジョブを使用する場合、キューに追加された一意の名前ごとにプロセッサを作成しなければならない。そうしないと、キューは与えられたジョブのプロセッサがないと不平を言う。名前付きジョブのコンシューミングについての詳細は[こちら](https://docs.nestjs.com/techniques/queues#consumers)

## ジョブのオプション

ジョブには、関連した追加のオプションを設定可能。`Queue.add()`メソッドのjob引数の後に、`options`オブジェクトを渡す。ジョブオプションのプロパティは

- `priority`:`number` - 省略可能な優先度の値。1（優先度最高）からMAX_INT（優先度最低）までの範囲で指定する。優先度を使う事はパフォーマンスに若干の影響を与える為注意。
- `delay`:`number` - このジョブが処理されるまでの待ち時間（ミリ秒）。正確なdelayの為には、サーバーとクライアント両方の時計の同期が必要。
- `attempts`:`number` - ジョブが完了するまでの総試行回数。
- `repeat`:`RepeatOpts` - cronの仕様に従ってジョブを繰り返し実行する。[RepeatOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)を参照の事。
- `backoff`:`number | BackoffOpts` - ジョブが失敗した場合の自動再試行のバックオフ設定です。[BackoffOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)を参照の事。
- `lifo`:`boolean` - `true`の場合、ジョブをキューの左端ではなく右端に追加する（デフォルトは`false`）。