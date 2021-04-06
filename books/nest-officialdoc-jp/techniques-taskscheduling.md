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
|`utcOffset`|timeZoneパラメータを使う代わりに、タイムゾーンのオフセットを指定することができる。|