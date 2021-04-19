---
title: techniques-logger
---

# ロガー

Nestはテキストベースのロガーが内蔵されており、アプリケーションの起動時やキャッチした例外の表示（システムロギング）などに使用される。この機能は`@nestjs/common`パッケージの`Logger`クラスで提供されている。ロギングシステムの動作は、以下のように完全に制御可能。

- ログを完全に無効にする
- ログの詳細レベルの指定（例：エラー、警告、デバッグ情報の表示等）
- デフォルトのロガーのタイムスタンプをオーバーライドする（例：ISO8601企画を日付フォーマットとして使用等）
- デフォルトのロガーを完全にオーバーライドする
- デフォルトのロガーを拡張してカスタマイズする
- 依存性インジェクションを利用して、アプリケーションの構成とテストを簡素化する

また、組み込みのロガーや独自のカスタムロガーを使ってアプリケーションレベルのイベントやメッセージを記録する事もできる。

より高度なロギング機能のためには、[Winston](https://github.com/winstonjs/winston)のようなNode.jsのロギングパッケージを利用して、完全にカスタム化されたプロダクショングレードのロギングシステムを実装する事ができる。

## 基本的なカスタマイズ

ロギングを無効にするには、`NestFactory.create()`メソッドの第二引数として渡される（省略可能な）Nestアプリケーションオプションオブジェクトの`logger`プロパティを`false`に設定する。

```ts
const app = await NestFactory.create(AppModule, {
  logger: false,
});
await app.listen(3000);
```

特定のログレベルを有効にするには、以下のように`logger`プロパティに対し、表示するログレベルを指定した文字列の配列を設定する。

```ts
const app = await NestFactory.create(AppModule, {
  logger: ['error', 'warn'],
});
await app.listen(3000);
```

配列は`'log'`、`'error'`、`'warn'`、`'debug'`、`'verbose'`の組み合わせの値になります。

>HINT  
>デフォルトのロガーのメッセージの色を無効にするには、環境変数`NO_COLOR`を設定する。

## カスタム実装

`logger`プロパティの値を`LoggerService`インターフェイスを満たすオブジェクトに設定する事で、Nestがシステムロギングに使用するカスタムロガーの実装を作る事ができる。例えば以下のように組み込みのグローバルJavaScript（`LoggerService`インターフェイスを実装している）を使用するようにNestに伝える事ができる。

```ts
const app = await NestFactory.create(AppModule, {
  logger: console,
});
await app.listen(3000);
```

独自のカスタムロガーの実装は簡単だ。以下に示すように`LoggerService`インターフェイスの各メソッドを実装するだけだ。

```ts
import { LoggerService } from '@nestjs/common';

export class MyLogger implements LoggerService {
  log(message: string) {
    /* あなたの実装 */
  }
  error(message: string, trace: string) {
    /* あなたの実装 */
  }
  warn(message: string) {
    /* あなたの実装 */
  }
  debug(message: string) {
    /* あなたの実装 */
  }
  verbose(message: string) {
    /* あなたの実装 */
  }
}
```

次に、Nestアプリケーションオプションオブジェクトの`logger`プロパティを介してMyLoggerのインスタンスを提供できる。

```ts
const app = await NestFactory.create(AppModule, {
  logger: new MyLogger(),
});
await app.listen(3000);
```

この記法はシンプルだが、`MyLogger`クラスの依存性注入を使っていない。これは特にテストの際に課題となり、`MyLogger`の再利用性を制限してしまう。以下の依存性注入の項を参照してほしい。

## 組み込みロガーの拡張

ゼロからロガーを書くのではなく、ビルトインの`Logger`クラスを拡張して、デフォルトの実装の一部の動作をオーバーライドする事で十分かもしれない。

```ts
import { Logger } from '@nestjs/common';

export class MyLogger extends Logger {
  error(message: string, trace: string) {
    // add your tailored logic here
    super.error(message, trace);
  }
}
```

拡張ロガーは後述の「アプリケーションロギング」の項で説明するように機能モジュールで使用可能。

Nestにシステムロギング用の拡張ロガーを使用させるには、（上記の「カスタム実装」のセクションで示したように）アプリケーションオプションオブジェクトの`logger`プロパティを介してそのインスタンスを渡すか、以下の「依存性インジェクション」の項で示す手法を使用する。その場合は上記のサンプルコードのように`super`を呼び出して、特定の`log`メソッドの呼び出しを親（組み込み）クラスに委譲し、Nestが期待する組み込み機能を利用できるようにしてほしい。

## 依存性インジェクション

より高度なロギング機能の実現の為、依存性インジェクションを活用すると良い。例えば`ConfigService`をロガーに注入してカスタマイズし、さらにカスタムロガーを他のコントローラやプロバイダにインジェクションする事ができる。カスタムロガーの依存性インジェクションを有効にするには、`LoggerService`を実装したクラスを作成し、そのクラスをモジュールのプロバイダとして登録する。例えばこう進められる――

1. 前の項のように組み込みの`Logger`を拡張するか、完全にオーバーライドする`MyLogger`クラスを定義してみよう。必ず`LoggerService`インターフェイスを実装してほしい。
2. 以下のように`LoggerModule`を作成し、そのモジュールから`MyLogger`を提供する。

```ts
import { Module } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Module({
  providers: [MyLogger],
  exports: [MyLogger],
})
export class LoggerModule {}
```