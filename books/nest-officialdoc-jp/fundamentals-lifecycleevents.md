---
title: fundamentals-lifecycleevents
---

# ライフサイクルイベント

Nestのアプリケーションは、すべてのアプリケーションの要素と同様に、Nestによって管理されるライフサイクルを持っている。Nestは主要なライフサイクルイベントを可視化する為の**ライフサイクルフック**を提供する。発生した際にアクション（`module`か、`injectable`か、`controller`に登録されたコードの実行）を行う機能を提供する。

## ライフサイクルシークエンス
次の図はアプリケーションがブートストラップされてからnodeプロセスが終了するまでの、主要なアプリケーションのライフサイクルイベントのシークエンスだ。全てのライフサイクルは、初期化、実行、終了の３つのフェーズに分けることができる。このライフサイクルを使用する事で、モジュールやサービスの適切な初期化を計画し、アクティブな接続を管理し、終了信号を受信した時にアプリケーションを優雅に終了できる。

[画像](https://docs.nestjs.com/assets/lifecycle-events.png)

## ライフサイクルイベント
ライフサイクルイベントはアプリケーションの起動時と終了時に発生する。Nestは以下のライフサイクルイベントのそれぞれで、`modules`、`injectables`、`conrollers`上で登録されたライフサイクルメソッドを呼び出す（後述するが、**終了時フック**を最初に有効にする必要がある）。上記の図の通り、Nestは接続の待機開始・終了の為に、基礎となる適切なメソッドを呼び出す。

続く下の表で、`onModuleDestroy`、`beforeApplicationShutdown`、`onApplicationShutdown`がトリガされるのは、明示的に`app.close()`を呼び出すか、プロセスが特別なシステムシグナル（SIGTERM等）を受け取って、アプリケーションの起動時に正しく`enableShutdownHooks`を呼び出した場合のみとなる（後述のApplication shutdownパートを参照のこと）。

|ライフサイクルメソッド|フックメソッドコールのトリガとなるライフルサイクルイベント|
| ---- | ---- |
|`onModuleInit()`|ホストモジュールの依存関係が解決された|
|`onApplicationBootstrap()`|全てのモジュールが初期化された後、接続待機前|
|`onModuleDestroy()`※|終了シグナル（`SIGTERM`等）を受け取った後|
|`beforeApplicationShutdown()`※|すべての`onModuleDestroy()`ハンドラが（Promiseがresolvedかrejectedとなって）完了した後に。完了するとすべての既存の接続が閉じられる。`app.close()`が呼び出される。|
|`onApplicationShutdown()`※|`onApplicationShutdown()`※の接続が終了した後に呼び出される。（app.close()がresolveする）

※のイベントについて、明示的に`app.close()`を呼んでいない場合は、`SIGTERM`のようなシステムシグナルで動作するように予め設定しなければならない。下記のApplication shutdownの項参照。

>WARNING  
>上記のライフサイクルフックは、リクエストスコープ化されたクラスに対してはトリガされない。リクエストスコープ化されたクラスはアプリケーションのライフサイクルに紐付いておらず、そのライフスパンは予想不可能のものだ。リクエストごとに排他的に作成され、レスポンスが送信された後に自動的にガベージコレクトされる。

## 使用法
それぞれのライフサイクルフックはインターフェイスによって表現される。インターフェイスはTypeScriptのコンパイル後には存在しなくなるため、技術として必須のものではないものの、強力なタイピング・エディタツールの恩恵を受けるためには使用するのがベストプラクティスといえる。ライフサイクルフックを登録するには、適切なインターフェイスを実装する。例えば、特定のクラス（コントローラ、プロバイダ、モジュール等）のモジュール初期化時に呼び出されるメソッドを登録するには、以下のように`onModuleInit()`メソッドを指定して`OnModuleInit`インターフェイスを実装する。

```ts
import { Injectable, OnModuleInit } from '@nestjs/common';

@Injectable()
export class UsersService implements OnModuleInit {
  onModuleInit() {
    console.log(`The module has been initialized.`);
  }
}
```

## 非同期初期化

`OnModuleInit`フックと`onApllicationBootstrap`共にアプリケーションの初期化プロセスを後ろ延ばしできる（Promiseを返すか、メソッドを`async`として記録しメソッド本体で非同期メソッドの完了を`await`する）。

```ts
async onModuleInit(): Promise<void> {
  await this.fetch();
}
```

## アプリケーションのシャットダウン
`onModuleDestroy()`・` beforeApplicationShutdown() `・`onApplicationShutdown()`フックは終了フェーズで呼び出される（`app.close()`への明示的な呼び出しに応答して、または組み込まれている場合は`SIGTERM`などのシステムシグナルの受信時に呼び出される）。この機能は[Kubernetes](https://kubernetes.io/)と一緒にコンテナのライフサイクルを管理する為、[Heroku](https://jp.heroku.com/home)によってdynosや似たサービスの為によく使われている。（前置詞が難しい…原文：This feature is often used with Kubernetes to manage containers' lifecycles, by Heroku for dynos or similar services.）

シャットダウンフックの待機はシステムリソースを消費する為デフォルトでは無効。使用するには、`enableShutdownHooks()`を呼び出してリスナーを有効にする必要がある。

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Starts listening for shutdown hooks
  app.enableShutdownHooks();

  await app.listen(3000);
}
bootstrap();
```

>WARNING  
>プラットフォーム固有の制限の為、NestJSはWindows上でのアプリケーションのシャットダウンフックのサポートを制限している。`SIGBREAK`やいくらかの`SIGHUP`と同様に、`SIGINT`も動作することが期待できる。詳細は[こちら](https://nodejs.org/api/process.html#process_signal_events)。しかしタスクマネージャでのプロセスの終了が無制限に可能な為、`SIGTERM`はWindows上では動作しない。言ってしまえば、アプリケーションがなんとかするのは無理だ（訳出自信なし…原文：there's no way for an application to detect or prevent it）。`SIGINT`・`SIGBREAK`などがWindows上でどう扱われているかはlibuvの関連ドキュメントを参考のこと。また、Node.jsドキュメントの[シグナルイベント処理](https://nodejs.org/api/process.html#process_signal_events)も見る事。

>Info  
>`enableShutdownHooks`はリスナーを起動する事でメモリを消費する。１つのNodeプロセスで複数のNestアプリを実行している場合（例：Jestで並列テスト中等）、Nodeはリスナープロセスの過剰について文句を言う事がある。この理由の為に`enableShutdownHooks`はデフォルトで無効となっている。1つのNodeプロセスで複数のインスタンスを実行している時、この状態に気をつけてほしい。

アプリケーションが終了信号を受信すると、アプリケーションは対応するシグナルを最初のパラメータとして、登録されている全ての`onModuleDestroy()`、` beforeApplicationShutdown()`、その後`onApplicationShutdown()`を（上記に説明したシークエンスの中）呼び出す。登録された関数が非同期呼び出し（promiseをreturnする）を待っている場合、プロミスが解決されるか拒否されるまで、Nestはシークエンスを実行しない。

```ts
@Injectable()
class UsersService implements OnApplicationShutdown {
  onApplicationShutdown(signal: string) {
    console.log(signal); // 例： "SIGINT"
  }
}
```