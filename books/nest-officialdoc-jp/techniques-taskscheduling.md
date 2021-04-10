---
title: techniques-taskscheduling.md
---

# タスクスケジューリング

タスクスケジューリングとは、任意のコード（メソッドや関数）を、定期実行したり、定期間隔で実行したり、指定した時間で１日実行するスケジューリング機能だ。Linuxでは、OSレベルでは[cron](https://en.wikipedia.org/wiki/Cron)などのパッケージが使われる。Node.jsアプリでもエミュレートできるパッケージがいくつかある。Nestは`@nestjs/schedule`パッケージを提供しており、これは人気の[node-cron](https://github.com/kelektiv/node-cron)(Node.js)と統合されている。このパッケージについて説明していこう。

## インストール

まず必要な依存関係をインストールしよう。

```
$ npm install --save @nestjs/schedule
$ npm install --save-dev @types/cron
```

ジョブスケジューリングを有効するには、`SchedulingModule`をルートの`AppModule`ににインポートして、以下のように`forRoot()`静的メソッドを実行する。

```ts
import { Module } from '@nestjs/common';
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot()
  ],
})
export class AppModule {}
```

`.forRoot()`の呼び出しは、スケジューラを初期化し、アプリ内に存在する宣言的なcronジョブ、タイムアウト、インターバルを登録する。登録は`onApplicationBootstrap`ライフサイクルフックが発生したときに行われ、すべてのモジュールのロード、全てのジョブのスケジュールの宣言を確実に行う。

## 宣言型cronジョブ

cronジョブは、任意の関数（メソッド呼び出し）を自動的に実行するようにスケジュールする。以下のように実行可能。

- 指定した日時に一度だけ実行。
- 指定した間隔で定期的に実行。（例：一時間に一介、一週間に一回、五分に一回）

以下のように、実行されるコードを含むメソッド定義の前に`@Cron()`デコレータをつけてcronジョブを宣言する。

```ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron('45 * * * * *')
  handleCron() {
    this.logger.debug('Called when the current second is 45');
  }
}
```

この例では、現在の秒数が45になるたびに`handleCron()`メソッドが呼び出される。言い換えれば、このメソッドは1分間に1回実行される。

`@Cron()`デコレータは標準的なcronパターンを全てサポートしている。

- Asterisk(*)
- Ranges(例：1-3,5)
- Steps(例：*/2)

上の例では、45****をデコレータに渡した。次のキーは、cronのパターン文字列の各位置がどのように解釈されるかを示す。

```ts
* * * * * *
| | | | | |
| | | | | day of week
| | | | month
| | | day of month
| | hour
| minute
second (省略可能)
```

|||
| ---- | ---- |
|`* * * * * *`|秒ごと|
|`45 * * * * *`|毎分45秒|
|`0 10 * * * *`|毎時間、10分0秒|
|`0 */30 9-17 * * *`|9時と5時の間のすべての30分|
|`0 30 11 * * 1-5`|月曜から金曜日の11時30分|

`@nestjs/schedule`パッケージでは、よく使われる`cron`パターンを集めた便利なenumを提供している。以下のように使用できる。

```ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  @Cron(CronExpression.EVERY_45_SECONDS)
  handleCron() {
    this.logger.debug('45秒ごとに呼び出される');
  }
}
```

この例では、`handleCron()`メソッドが45秒ごとに呼び出される。

別の方法として、JavaScriptの`Date`オブジェクトを`@Cron()`デコレータに指定する事もできる。こうすると、ジョブは指定された日付に一度だけ実行される。

>HINT  
>JavaScriptの日付計算を使って、現在の日付を基準にしてジョブをスケジュールしてみよう。たとえば`@Cron(new Date(Date.now() + 10 * 1000))`と入力すると、アプリが起動してから10秒後にジョブがスケジュールされる。

また、`@Cron()`デコレータの2番めのパラメータとして、追加のオプションを与える事ができる。

|||
| ---- | ---- |
|`name`|宣言されたcronジョブにアクセスして制御する為に便利|
|`timeZone`|実行時のタイムゾーンを指定する。すると、指定したタイムゾーンを基準として、実際の時間を修正する（modify the actual time）。タイムゾーンが無効な場合はエラーが発生する。利用可能な全てのタイムゾーンは、[Moment Timezone](https://momentjs.com/timezone/)にて確認可能。|
|`utcOffset`|`timeZone`パラメータを使う代わりに、タイムゾーンのオフセットを指定することができる。|

```ts
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class NotificationService {
  @Cron('* * 0 * * *', {
    name: 'notifications',
    timeZone: 'Europe/Paris',
  })
  triggerNotifications() {}
}
```

宣言済みのcronジョブにアクセスして制御する事も、Dynamic APIを使用してcronジョブを動的に作成する事もできる（その際cronパターンは実行時に定義される（where its cron pattern is defined at runtime…ちゃんと読めてるか怪しいです））。APIを使用して宣言型のcronジョブにアクセスするには、省略可能なオプションオブジェクト　`name`プロパティをデコレータの第２引数として渡して、ジョブに名前を関連付ける必要がある。

## 宣言型インターバル

指定の間隔で定期的にメソッドを実行する事を宣言するには、メソッド定義の前に`@Interval()`デコレータをつける。インターバルの値をミリ秒単位の数値として、以下のようにデコレータに渡す。

```ts
@Interval(10000)
handleInterval() {
  this.logger.debug('Called every 10 seconds');
}
```

>HINT  
>内部ではJavaScriptの`setInterval()`関数を利用した仕組みとなっている。cronジョブを使って定期的なジョブをスケジューリングする事もできる。

宣言したクラスの外からDynamic APIを使って宣言した間隔を制御したい場合は、次のようにしてインターバルに名前をつける。

```ts
@Interval('notifications', 2500)
handleInterval() {}
```

Dynamic APIは実行時にインターバルのプロパティが定義されるダイナミック・インターバルの**作成**や、インターバルの**リスト化**、**削除**も行える。

## 宣言型タイムアウト

指定したタイムアウト時に（一度だけ）メソッドを実行する事を宣言する場合は、メソッドの定義の前に`@Timeout()`デコレータをつける。以下のように、アプリケーション起動時からの相対的なタイムオフセット（ミリ秒単位）をデコレータに渡す。

```ts
@Timeout(5000)
handleTimeout() {
  this.logger.debug('5秒後に呼ばれる');
}
```

>HINT  
>Javascriptの`setTimeout()`関数を仕組みとしている。

宣言したクラスの外からDynamic APIを使って制御したい場合は、以下のようにタイムアウトに名前をつける。

```ts
@Timeout('notifications', 2500)
handleTimeout() {}
```

Dynamic APIは実行時にタイムアウトのプロパティが定義されるダイナミック・タイムアウトの**作成**や、タイムアウトの**リスト化**、**削除**も行える。

## 動的スケジュールモジュールのAPI

`nestjs/schedule`モジュールは、宣言的なcronジョブ、タイムアウト、インターバルを管理できるDynamic APIを提供する。このAPIでは実行時にプロパティが定義される、**動的な**cronジョブ、タイムアウト、およびインターバルの作成と管理も可能である。

## 動的なcronジョブ

`SchedulerRegistry`APIを利用して、コードのどこからでも`CronJob`インスタンスへ、名前を使って参照できるようになる。まず標準的なコンストラクタインジェクションを使って`SchedulerReggistery`をインジェクションしよう。

```ts
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

>HINT  
>`SchedulerRegistry`は`@nestjs/schedule`パッケージからインポートする。

これを次のようにクラスで使用してみよう。以下のような宣言でcronジョブが作成されたとする。

```ts
@Cron('* * 8 * * *', {
  name: 'notifications',
})
triggerNotifications() {}
```

次のコードを使ってこのジョブにアクセスしよう。

```ts
const job = this.schedulerRegistry.getCronJob('notifications');

job.stop();
console.log(job.lastDate());
```

`getCronJob()`メソッドは指定されたcronジョブを返す。返された`CronJob`オブジェクトは以下のメソッドを持つ。

- `stop()` 実行が予定されているジョブを停止する。
- `start()` 停止されたジョブを再起動する。
- `setTime(time:CronTime)`ジョブを停止し、新しい時間を設定した後、ジョブを開始する。
- `lastDate()`ジョブが最期に実行された日付の文字列表現を返す。
- `nextDates(count:number)` 次のジョブ実行日を返す`moment`オブジェクトの配列を（サイズは`count`で）返す。

>HINT  
>`toDate()`を使って、`moment`オブジェクトを人間が読める形にしよう。

以下のように`SchedulerRegistry.addCronJob()`メソッドを使用して、新しいcronジョブを動的に作成する。

```ts
addCronJob(name: string, seconds: string) {
  const job = new CronJob(`${seconds} * * * * *`, () => {
    this.logger.warn(`毎 (${seconds}) 秒 でjob ${name} が動く！`);
  });

  this.schedulerRegistry.addCronJob(name, job);
  job.start();

  this.logger.warn(
    `job ${name} が毎分 ${seconds} 秒で動く！`,
  );
}
```

このコードでは、cronパッケージの`CronJob`オブジェクトを使用してcronジョブを作成している。`CronJob`のコンストラクタは、最初の引数としてcronパターン（`@Cron()`デコレータのようなもの）をとり、２番めの引数としてcronタイマーが起動したときに実行されるコールバックを取る。`SchedulerRegistry.addCronJob()`メソッドは`CronJob`の名前と`CronJob`オブジェクト自体を引数として取る。

>WARNING  
>アクセスする前に`SchedulerRegistry`をインジェクションする事を忘れないでほしい。cronパッケージから`CronJob`をインポートしてほしい。

以下のように`SchedulerRegistry.deleteCronJob()`メソッドを使って名前付きのcronジョブを**削除**してみよう。

```ts
deleteCron(name: string) {
  this.schedulerRegistry.deleteCronJob(name);
  this.logger.warn(`job ${name} deleted!`);
}
```

以下のように`SchedulerRegistry.getCronJobs()`メソッドを使って全てのcronジョブを**リスト化**してみよう。

```ts
getCrons() {
  const jobs = this.schedulerRegistry.getCronJobs();
  jobs.forEach((value, key, map) => {
    let next;
    try {
      next = value.nextDates().toDate();
    } catch (e) {
      next = 'error: next fire date is in the past!';
    }
    this.logger.log(`job: ${key} -> next: ${next}`);
  });
}
```

`getCronJobs()`は`map`を返す。このコードではマップをイテレートして、各`CronJob`の`nextDates()`にアクセスしようとしている。CronJob APIでは、ジョブが既に起動していて、将来の起動日がない場合は、例外を投げる。

## 動的なインターバル

`SchedulerRegistry.getInterval()`メソッドを使ってインターバルへの参照を取得する。先述のように、標準的なコンストラクタ・インジェクションを使用して`SchedulerRegistry`をインジェクションする。

```ts
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

そして以下のように使う。

```ts
const interval = this.schedulerRegistry.getInterval('notifications');
clearInterval(interval);
```

次のように`SchedulerRegistry.addInterval()`メソッドを使用して、新しいインターバルを動的に**作成**してみよう。

```ts
addInterval(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`Interval ${name} executing at time (${milliseconds})!`);
  };

  const interval = setInterval(callback, milliseconds);
  this.schedulerRegistry.addInterval(name, interval);
}
```

このコードでは、標準的なJavaScriptのインターバルを作成し、それを`ScheduleRegistry.addInterval()`メソッドに渡している。このメソッドはインターバルの名前とインターバル自体、２つの引数を取る。

`SchedulerRegistry.deleteInterval()`メソッドを使うとインターバルを**削除**できる。

```ts
deleteInterval(name: string) {
  this.schedulerRegistry.deleteInterval(name);
  this.logger.warn(`Interval ${name} deleted!`);
}
```

`SchedulerRegistry.getIntervals()`メソッドを使うと全てのインターバルを**リスト化**できる。

```ts
getIntervals() {
  const intervals = this.schedulerRegistry.getIntervals();
  intervals.forEach(key => this.logger.log(`Interval: ${key}`));
}
```

## 動的なタイムアウト

タイムアウトへの参照は、`SchedulerRegistry.getTimeout()`メソッドで得られる。先述のように、標準的なコンストラクタ・インジェクションを使用して`SchedulerRegistry`をインジェクションする。

```ts
constructor(private schedulerRegistry: SchedulerRegistry) {}
```

以下のように使う。

```ts
const timeout = this.schedulerRegistry.getTimeout('notifications');
clearTimeout(timeout);
```

次のように`SchedulerRegistry.addTimeout()`メソッドを使用して、新しいタイムアウトを動的に**作成**してみよう。

```ts
addTimeout(name: string, milliseconds: number) {
  const callback = () => {
    this.logger.warn(`Timeout ${name} executing after (${milliseconds})!`);
  };

  const timeout = setTimeout(callback, milliseconds);
  this.schedulerRegistry.addTimeout(name, timeout);
}
```

このコードでは、標準的なJavaScriptのタイムアウトを作成し、それをScheduleRegistry.addTimeout()メソッドに渡している。このメソッドはタイムアウトの名前とタイムアウト自体、２つの引数を取る。

`SchedulerRegistry.deleteTimeout()`メソッドで名前付きのタイムアウトを**削除**してみよう。

```ts
deleteTimeout(name: string) {
  deleteTimeout(name: string) { this.schedulerRegistry.deleteTimeout(name);
  this.logger.warn(`Timeout ${name} deleted!`);
}
```

`SchedulerRegistry.getTimeouts()`メソッドで全てのタイムアウトを**リスト化**してみよう。

```ts
getTimeouts() {
  const timeouts = this.schedulerRegistry.getTimeouts();
  timeouts.forEach(key => this.logger.log(`Timeout: ${key}`));
}
```

## サンプル
動く例は[こちら](https://github.com/nestjs/nest/tree/master/sample/27-scheduling)