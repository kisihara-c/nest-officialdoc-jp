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

このコードによってカスタムロガーを他のモジュールで使用できる。`MyLogger`クラスはモジュールの一部である為、依存性インジェクションを使用できる（例えば、`ConfigService`をインジェクションする為等）。このカスタムロガーを`Nest`がシステムロギング（ブートストラップやエラー処理など）に使用する為には、もうひとつ必要なテクニックがある。

アプリケーションのインスタンス化（`NestFactory.create()`）は、あらゆるモジュールのコンテキストの外で行われる為、初期化時の通常の依存性インジェクションフェーズには参加しない。そのため、少なくとも1つのアプリケーションモジュールに`LoggerModule`をインポートさせて、Nestが`MyLogger`クラスのシングルトンインスタンスをインスタンス化するようにしなければならない。そして次のコードによって、Nestに単一の`MyLogger`シングルトンを使用させられる。

```ts
const app = await NestFactory.create(AppModule, {
  logger: false,
});
app.useLogger(app.get(MyLogger));
await app.listen(3000);
```

ここでは、`NestApplication`インスタンスの`get()`メソッドを使用して、`MyLogger`オブジェクトのシングルトンを取得している。この手法は基本的に、Nestで使用するロガーのインスタンスをインジェクションする方法だ。`app.get()`呼び出しは`MyLogger`のシングルトンを取得する。そして上記のように、別のモジュールで最初にインジェクションされるインスタンスに依存している。

この方法の唯一の欠点は、最初の初期化メッセージがどこにも表示されない事で、ごくたまに重要な初期化エラーを見逃す。別の方法として、デフォルトのロガーを使って最初の初期化メッセージを出力し、その後カスタムロガーに切り替える事もできる。

```ts
const app = await NestFactory.create(AppModule);
app.useLogger(app.get(MyLogger));
await app.listen(3000);
```

また、この`MyLogger`プロバイダを機能クラスにインジェクションする事で、Nestのシステムロギングとアプリケーションロギングの両方で一貫したロギング動作を実現できる。詳細については以下のそれぞれの項を参照の事。

## アプリケーションのロギングにロガーを使用する

上記のテクニックを組み合わせて、Nestシステムのロギングと独自のアプリケーションのイベント/メッセージのロギングの両方で、一貫した動作とフォーマットを提供する事ができる。

グッドプラクティスは、`@nestjs/common`の`Logger`クラスを各サービスでインスタンス化する事だ。`Logger`のコンストラクタの`context`引数には、以下のようにサービス名を指定する。

```ts
import { Logger, Injectable } from '@nestjs/common';

@Injectable()
class MyService {
  private readonly logger = new Logger(MyService.name);
  
  doSomething() {
    this.logger.log('Doing something...');
  }
}
```

デフォルトのロガーの実装では、以下の例の`NestFactory`のように、角括弧の中に`context`が出力される。

```ts
[Nest] 19096   - 12/08/2019, 7:12:59 AM   [NestFactory] Starting Nest application...
```

`app.useLogger()`でカスタムロガーを指定すると、実際にNestが内部で使用する。つまり`app.userLogger()`を呼び出す事でデフォルトのロガーを簡単にカスタムロガーに切り替えられる事で、コードが実装に対して独立しているといえる。

前節の手順で`app.useLogger(app.get(MyLogger))`を呼び出した場合、`MyService`から`this.logger.log()`を呼び出すと、`MyLogger`インスタンスから`log`メソッドを呼び出す事になる。

これはほとんどの場合で有用だが、もっとカスタマイズが必要な場合（カスタムメソッドの追加や呼び出し等）は次のセクションを参照の事。

## カスタムロガーをインジェクションする

手始めに、以下のようなコードで組み込みのロガーを拡張する。ここでは`Logger`クラスの設定メタデータとして`scope`オプションを指定し、[一時的な](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-injectionscopes)スコープを指定する事で、各機能モジュールで`MyLogger`の一意のインスタンスを持つようにしている。この例では個々の`Logger`メソッド（`log()`、`warn()`など）を拡張していないが、必要に応じて拡張可能。

```ts
import { Injectable, Scope, Logger } from '@nestjs/common';

@Injectable({ scope: Scope.TRANSIENT })
export class MyLogger extends Logger {
  customLog() {
    this.log('Please feed the cat!');
  }
}
```

次に、以下のコードで`LoggerModule`を作成しよう。

```ts
import { Module } from '@nestjs/common';
import { MyLogger } from './my-logger.service';

@Module({
  providers: [MyLogger],
  exports: [MyLogger],
})
export class LoggerModule {}
```