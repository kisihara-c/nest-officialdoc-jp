---
title: techniques-queues
---

## キュー

キューはアプリケーションのスケーリングやパフォーマンスに関する一般的な問題に対処する為役立つ、強力なデザインパターンだ。キューを使って解決可能な問題の例をいくつか挙げてみよう。

- 処理のピークをスムーズにする。例えば、ユーザがリソースを必要とするタスクを任意の時間に開始できる場合、これらのタスクを同時的に実行するのではなく、キューに追加する事ができる。そして、制御された方法で、ワーカープロセスにキューからタスクを引き出させる事ができる。アプリケーションのスケールアップに合わせて、新しいキューのコンシューマーを追加し、バックエンドのタスク処理を拡張する事も簡単だ。
- Node.jsのイベントループをブロックしてしまうような一枚岩のタスクを分割する。例えば、ユーザのリクエストが（オーディオトランスコーディングのような）CPU負荷の高い作業を必要とする場合、このタスクを他のプロセスに委ねることができ、ユーザに接するプロセスを解放して応答性を維持することができる。
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

各キューは、１つまたは複数のプロデューサー、コンシューマー、およびリスナーを持つことができる。コンシューマーは特定の順序でキューからジョブを取得する。FIFO（デフォルト）、LIFO、または各優先順位に従う。実行順の詳細については後のコンシューマーの項にて。

## 名前付き設定

キューが複数の異なるRedisインスタンスに接続している場合、名前付き設定と呼ばれる機能を使用できる。指定したキーの下に複数の構成を登録し、キューのオプションで参照することができる。

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
>名前付きジョブを使用する場合、キューに追加された一意の名前ごとにプロセッサを作成しなければならない。そうしないと、キューは与えられたジョブのプロセッサがないと不平を言う。名前付きジョブの消化についての詳細は以下のコンシューマーの項にて。

## ジョブのオプション

ジョブには、関連した追加のオプションを設定可能。`Queue.add()`メソッドのjob引数の後に、`options`オブジェクトを渡す。ジョブオプションのプロパティは

- `priority`:`number` - 省略可能な優先度の値。1（優先度最高）からMAX_INT（優先度最低）までの範囲で指定する。優先度を使う事はパフォーマンスに若干の影響を与える為注意。
- `delay`:`number` - このジョブが処理されるまでの待ち時間（ミリ秒）。正確なdelayの為には、サーバーとクライアント両方の時計の同期が必要。
- `attempts`:`number` - ジョブが完了するまでの総試行回数。
- `repeat`:`RepeatOpts` - cronの仕様に従ってジョブを繰り返し実行する。[RepeatOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)を参照の事。
- `backoff`:`number | BackoffOpts` - ジョブが失敗した場合の自動再試行のバックオフ設定です。[BackoffOpts](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueadd)を参照の事。
- `lifo`:`boolean` - `true`の場合、ジョブをlifoで処理する――キューの左端ではなく右端に追加する（デフォルトは`false`）。
- `timeout`: `number` - ジョブがタイムアウト・エラーを起こすまでのミリ秒数。
- `jobId`: `number | string` - ジョブIDを上書きする。デフォルトではジョブIDは一意の整数だが、これを使えば上書き可能。使用時ジョブIDの一意性は自分で確認する事。既に存在しているIDでジョブを追加しようとしても、追加されない。
- `removeOnComplete`: `boolean | number` - `true`時、ジョブが正常に完了したときにジョブを削除する。数字は保持するジョブの量を指定する。デフォルトでは、完了したセットの中でジョブは保持される。
- `removeOnFail`: `boolean | number` - `true`時、すべての試行に失敗した後ジョブを削除する。数値は保持するジョブの量を指定する。デフォルトでは、完了したセットの中でジョブは保持される。
- `stackTraceLimit`: `number` - スタックトレースに記録されるスタックトレース行数を制限する。

、ジョブのカスタマイズ例をいくつか紹介する。

ジョブの開始を遅らせるため、`delay`を使ってみよう。

```ts
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { delay: 3000 }, // 3秒のdelay
);
```

ジョブをlifoで処理するには`lifo`を`true`にしよう。

```ts
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { lifo: true },
);
```

ジョブに優先順位をつける為、`priority`を使ってみよう。

```ts
const job = await this.audioQueue.add(
  {
    foo: 'bar',
  },
  { priority: 2 },
);
```

## コンシューマー

コンシューマーは、キューに追加されたジョブをこなすか、キュー上のイベントをlistenするか、あるいはその両方を行うメソッドを定義するクラスの事だ。以下のように`@Processor()`デコレータを使用してコンシューマークラスを宣言する。

```ts
import { Processor } from '@nestjs/bull';

@Processor('audio')
export class AudioConsumer {}
```

>HINT  
>コンシューマーは、`@nestjs/bull`パッケージで拾い上げられるように、プロバイダとして登録されている必要がある。

デコレータの文字列引数（例：`audio`）には、クラスのメソッドに関連付けられるキューの名前が入る。

コンシューマークラス内では、ハンドラ・メソッドを`@Process()`デコレータで装飾して、ジョブハンドラを宣言する。

```ts
import { Processor, Process } from '@nestjs/bull';
import { Job } from 'bull';

@Processor('audio')
export class AudioConsumer {
  @Process()
  async transcode(job: Job<unknown>) {
    let progress = 0;
    for (i = 0; i < 100; i++) {
      await doSomething(job.data);
      progress += 10;
      job.progress(progress);
    }
    return {};
  }
}
```

装飾されたメソッド（例：`transcode()`）は、ワーカーがアイドル状態で、キューに処理すべきジョブがあるときにはいつでも呼び出される。このハンドラメソッドは、唯一の引数としてジョブオブジェクトを受け取る。返す値はジョブオブジェクトに格納され、後で、完了イベントのリスナーなどでアクセスできる。

ジョブオブジェクトには、状態を操作する為の複数のメソッドがある。たとえば上記のコードでは`progress()`メソッドをツ使ってジョブの進行状況を更新している。ジョブオブジェクトのAPIリファレンスについては[こちら](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#job)

以下に示すように、`@Process()`デコレータに名前を渡すことで、ジョブハンドラメソッドがある種類のジョブ（特定の名前を持つジョブ）のみを処理するよう指定できる。コンシューマー・クラスでは、各ジョブの種類（名前）に対応した複数の`@Process()`ハンドラを持つことができる。名前付きジョブを使用する場合は、必ずそれぞれの名前に対応するハンドラを持つようにする事。

```ts
@Process('transcode')
async transcode(job: Job<unknown>) { ... }
```

## リクエストスコープ化されたコンシューマー

コンシューマーがリクエストスコープとしてフラグ付されると（インジェクションスコープの[詳細](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-injectionscopes#%E3%83%97%E3%83%AD%E3%83%90%E3%82%A4%E3%83%80%E3%82%B9%E3%82%B3%E3%83%BC%E3%83%97)）、各ジョブ専用に新しいインスタンスが作成される。このインスタンスはジョブ完了後ガベージコレクションの対象となる。

```ts
@Processor({
  name: 'audio',
  scope: Scope.REQUEST,
})
```

リクエストに対応したコンシューマクラスは動的にインスタンス化され、単一のジョブにスコープ化されているので、標準的な方法でコンストラクタから`JOB_REF`をインジェクションできる。

```ts
constructor(@Inject(JOB_REF) jobRef: Job) {
  console.log(jobRef);
}
```

>HINT  
>`JOB_REF`トークンは`@nestjs/bull`パッケージからインポートできる。

## イベントリスナー

Bullは、キューやジョブの状態が変化した際、使いやすい形でイベントのセットを生成する。Nestでは標準的なイベントのコアセットをサブスクライブできるデコレータのセットを提供している。これらは`@nestjs/bull`パッケージからエクスポートされる。

イベントリスナーは、コンシューマークラス内（すなわち`@Procesor()`デコレータで装飾されたクラス内）で宣言する必要がある。イベントをリッスンするには、以下の表にあるデコレータのいずれかを使用して、イベントのハンドラを宣言する。例えば、オーディオキューでジョブがアクティブな状態になった時に発せられるイベントをリッスンするには、次のように記述する。

```ts
import { Processor, Process } from '@nestjs/bull';
import { Job } from 'bull';

@Processor('audio')
export class AudioConsumer {

  @OnQueueActive()
  onActive(job: Job) {
    console.log(
      `Processing job ${job.id} of type ${job.name} with data ${job.data}...`,
    );
  }
  ...
```

Bullは分散型（マルチノード）環境で動作する為、イベントローカリティの概念を定義している。この概念は、イベントが単一のプロセス内で完全にトリガされるケースと、異なるプロセスで共有されるキュー上でトリガされるケースを認識している。**ローカル**イベントとは、ローカルプロセス内のキューでアクションや状態変化がトリガされたときに生成されるイベントだ。言い換えれば、イベントプロデューサ及びコンシューマが単一のプロセスに対しローカルな場合、キュー上で発生する全てのイベントはローカルとなる。

キューを複数のプロセスで共有する場合、**グローバル**なイベントに触れる可能性があるかもしれない。あるプロセスのリスナーが、他のプロセスで発生したイベント通知を受け取るためには、グローバルイベントに登録する必要がある。

イベントハンドラは対応するイベントが発効されるたびに呼び出される。ハンドラは以下の表に示すシグネチャで呼び出され、イベントに関連する情報にアクセスできる。ローカル・グローバルイベントリスナーのシグネチャのそれぞれを以下に示す。

|ローカルイベントリスナー|グローバルイベントリスナー|ハンドラメソッドのシグネチャ、発動時|
| ---- | ---- | ---- |
|`@OnQueueError()`|`@OnGlobalQueueError()`|`handler(error: Error)` - エラー発生。`error`はトリガとなったエラーを持っている。|
|`@OnQueueWaiting()`|`@OnGlobalQueueWaiting()`|	`handler(jobId: number | string)` - ワーカーがアイドリング状態になり次第すぐ処理される為ジョブが待機している。`jobID`はこの状態になったジョブのID。|
|`@OnQueueActive()`|`@OnGlobalQueueActive()`|`handler(job: Job)` - ジョブ`job`が開始された。|
|`@OnQueueStalled()`|`@OnGlobalQueueStalled()`|`handler(job: Job)` - ジョブ`job`が停止したと記された。これはクラッシュするジョブワーカーのデバッグや、イベントループの一時停止のため使える。|
|`@OnQueueProgress()`|`@OnGlobalQueueProgress()`|`handler(job: Job, progress: number)` - Job`job`の進捗状況が変数`progress`の進捗状態へと更新された。.|
|`@OnQueueCompleted()`|`@OnGlobalQueueCompleted()`|`handler(job: Job, result: any)` - ジョブ`job`の実行が成功し、結果`result`が得られた。|
|`@OnQueueFailed()`|`@OnGlobalQueueFailed()`|`handler(job: Job, err: Error)` ジョブ`job`は理由`err`で失敗した。|
|`@OnQueuePaused()`|`@OnGlobalQueuePaused()`|`handler()`が一時停止した。|
|`@OnQueueResumed()`|`@OnGlobalQueueResumed()`|`handler(job: Job)` handler()キューが一時停止した。|
|`@OnQueueCleaned()`|`@OnGlobalQueueCleaned()`|	`handler(jobs: Job[], type: string)` 古いジョブがキューから削除された。`jobs`は削除されたジョブの配列で、`type`は削除されたジョブの型だ。|
|`@OnQueueDrained()`|`@OnGlobalQueueDrained()`|`handler()`キューが待機中のジョブを全て処理したときに発効されるdelayedジョブがまだ処理されていない場合含む）。 |
|`@OnQueueRemoved()`|`@OnGlobalQueueRemoved()`|`handler(job: Job)` - ジョブ`job`の削除に成功した。|

グローバルイベントをlistenする場合、メソッドシグネチャはローカルイベントと若干異なる場合がある。具体的には、ローカルでジョブオブジェクトを受け取るメソッドシグネチャは、グローバルでは代わりに`jobID`（`number`）を受け取る。こういった場合、実際の`job`オブジェクトへの参照を取得するには、`Queue#getJob`メソッドを使う。このメソッドコールは`await`である必要がある為、ハンドラを`async`で宣言する必要がある。例：

```ts
@OnGlobalQueueCompleted()
async onGlobalCompleted(jobId: number, result: any) {
  const job = await this.immediateQueue.getJob(jobId);
  console.log('(Global) on completed: job ', job.id, ' -> result: ', result);
}
```

>HINT  
>`Queue`オブジェクトにアクセスする（`getJob()`コールを行う）ためには、もちろんインジェクションを行う必要がある。また、`Queue`は、インジェクションを行うモジュールに登録されていなければならない。

特定のイベント・リスナー・デコレータに加えて、一般的な`@OnQueueEvent()`デコレータを`BullQueueEvents`か`BullQueueGlobalEvents`enumと組み合わせて使用する事もできる。イベントについての詳細は[こちら](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#events)

## キューの管理

キューにはAPIがある。一時停止や再開等、様々な状態にあるジョブの数の取得などの管理機能を実行できる。キューのAPIについて詳細は[こちら](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue)。以下のpause/resumeのサンプルのように、これらのメソッドのいずれかを`Queue`オブジェクト上で直接呼び出せる。

`pause`メソッドコールでキューを一時停止してみよう。一時停止したキューは、再開されるまで新しいジョブは処理されない。現在処理されているジョブはfinalizeされるまで継続される。

```ts
await audioQueue.pause();
```

`resume()`メソッドを使って一時停止したキューを再開してみよう。

```ts
await audioQueue.resume();
```

## プロセスの分離

ジョブハンドラは、別の（フォークされた）プロセスで実行する事も[できる](https://github.com/OptimalBits/bull#separate-processes)。いくつかの利点がある。

- プロセスはサンドボックス化されているので、クラッシュしてもワーカーには影響しない。
- キューに影響を与えずにブロックしたコードを実行できる（ジョブは仕切られない）。
- マルチコアCPUの利用率が格段に上がる。
- redisへの接続が減る。

```ts :app.module.ts 
import { Module } from '@nestjs/common';
import { BullModule } from '@nestjs/bull';
import { join } from 'path';

@Module({
  imports: [
    BullModule.registerQueue({
      name: 'audio',
      processors: [join(__dirname, 'processor.js')],
    }),
  ],
})
export class AppModule {}

```

この関数はフォークされたプロセスで実行されるため、依存性インジェクション（とIoCコンテナ）は利用できない事に注意。つまり、`processor`関数は、必要な外部依存関係の全てのインスタンスを含む（か作る）必要がある。

```ts :processor.ts 
import { Job, DoneCallback } from 'bull';

export default function (job: Job, cb: DoneCallback) {
  console.log(`[${process.pid}] ${JSON.stringify(job.data)}`);
  cb(null, 'It works');
}
```

## 非同期設定

bullオプションを非同期に渡したい場合もあるだろう。この場合は`forRootAsync()`メソッドを使用して処理する。同様に、キューのオプションを非同期的に渡したい場合は`registerQueueAsync()`メソッドを使用する。

ファクトリー関数を使うという方法もある。

```ts
BullModule.forRootAsync({
  useFactory: () => ({
    redis: {
      host: 'localhost',
      port: 6379,
    },
  }),
});
```

ファクトリーは他の[非同期プロバイダ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-asyncproviders)と同じ様に動作する。例えば、`async`で非同期にする事ができ、`inject`で依存関係をインジェクションできる。

```ts
BullModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    redis: {
      host: configService.get('QUEUE_HOST'),
      port: +configService.get('QUEUE_PORT'),
    },
  }),
  inject: [ConfigService],
});
```

他の手段として、`useClass`構文も使える。

```ts
BullModule.forRootAsync({
  useClass: BullConfigService,
});
```

上記の構造では、`BullModule`内で`BullConfigService`をインスタンス化し、それを使って`createSharedConfiguration()`を呼び出してオプションオブジェクトを提供する。この場合、`BullConfigService`は以下のように`SharedBullConfigurationFactory`インターフェイスを実装する必要がある事に注意。

```ts
@Injectable()
class BullConfigService implements SharedBullConfigurationFactory {
  createSharedConfiguration(): SharedBullConfigurationFactory {
    return {
      redis: {
        host: 'localhost',
        port: 6379,
      },
    };
  }
}
```

`BullModule`の中で`BullConfigService`を作らず、別のモジュールからインポートしたプロバイダを使うためには、`useExisting`構文を使う。

```ts
BullModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

この構造は`useClass`と同じ様に動作するが、1つ重要な違いがある。`BullModule`は新しい`ConfigService`をインスタンス化する代わりに、インポートされたモジュールを調べて既存の`ConfigService`を再利用する。

## サンプル

実際に動くサンプルは[こちら](https://github.com/nestjs/nest/tree/master/sample/26-queues)