---
title: techniques-configuration
---

# 設定

アプリケーションはしばしば異なる**環境**で動作する。環境に応じて異なる構成設定を使用する必要がある。例えば通常ローカル環境ではローカルのDBインスタンスでのみ有効な特定のデータベース認証情報に依存している。本番環境では、別のDB資格情報のセットを使う。（原文、ここでamlという文字列が入ってるけど訳出できず…）ベストプラクティスは、[環境変数を保存](https://12factor.net/config)する事だ。外部で定義された環境変数は、`process.env`グローバル変数を通じてNode.js内部で確認できる。環境変数をそれぞれの環境で個別に設定する事で、複数の環境の問題を解決可能となる。特に、これらの値を簡単にモックしたり変更したりする必要がある開発環境やテスト環境では、あっという間に扱いにくくなってしまう。

Node.jsアプリケーションでは、各環境を表すために、各キーが特定の値を表すキーと値のペアを保持する`.env`ファイルを使用するのが一般的だ。異なる環境でアプリケーションを実行するには、適切に`.env`ファイルを入れ替えるだけで良い。

Nestでこのテクニックを使用するには、`ConfigModule`を作成するのが良い。このモジュールでは適切な`.env`ファイルをロードする`ConfigService`を表出する。Nestでは便利なように`@nestjs/config`パッケージが用意されている。このパッケージについて説明する。

## インストール

まず依存関係をインストールしよう。

```
$ npm i --save @nestjs/config
```

>HINT
>`@nestjs/config`パッケージは[dotenv](https://github.com/motdotla/dotenv)を使う。

## スタートアップ

インストールプロセスが完了したら、`ConfigModule`をインポートする事ができる。一般的には、ルートの`AppModule`にインポートし、静的メソッド`.forRoot()`を使ってその動作を制御する。ここでは環境変数のkey/valueペアが解析・解決される。後で、他の機能モジュールから`ConfigModule`の`ConfigService`クラスにアクセスする為のいくつかのオプションを見よう。

```ts :app.module.ts 
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot()],
})
export class AppModule {}
```

上記のコードは、デフォルトの場所（プロジェクトのルートディレクトリ）から`.env`ファイルをロード・解析し、`.env`ファイルのkey/valueペアを`process.env`に割り当てられた環境変数とマージし、その結果を`ConfigService`を通じてアクセスできるprivateな構造体に格納する。`forRoot()`は`ConfigService`プロバイダを登録する。`ConfigService`プロバイダはこれらの解析/マージされた設定変数を読み取るための`get()`メソッドを提供する。`@nestjs/config`は[dotenv](https://github.com/motdotla/dotenv)に依存しているため、環境変数名の競合を解決する際はdotenvのルールに則る。あるキーがランタイム環境の環境変数（例：OSシェルのエクスポート　`export DATABASE_USER=test`）と`.env`ファイルの両方に存在する場合、ランタイム環境の環境変数が優先される。

`.env`ファイルのサンプルは以下。

```
DATABASE_USER=test
DATABASE_PASSWORD=test
```

## カスタムenvファイルのパス

デフォルトでは、アプリケーションのルートディレクトリが第一候補となる。別のパスを指定するには、`forRoot()`に渡すオプションオブジェクトの`envFilePath`プロパティを以下のように設定の事。

```ts
ConfigModule.forRoot({
  envFilePath: '.development.env',
});
```

また、以下のように`.env`ファイルのパスを複数指定する事もできる。

```ts
ConfigModule.forRoot({
  envFilePath: ['.env.development.local', '.env.development'],
});
```

１つの変数が複数のファイルに存在する場合、最初のファイルが優先される。

## env変数の読み込みの無効

`.env`ファイルをロードせず、（`export DATABASE_USER=test`のようなOSシェルのエクスポートの様に）実行環境から環境変数にアクセスするだけにしたい場合、以下のようにオプションオブジェクトの`ignoreEnvFile`プロパティを`true`に設定する。

```ts
ConfigModule.forRoot({
  ignoreEnvFile: true,
});
```

## モジュールをグローバルに使う

`ConfigModule`を他のモジュールで使用したい場合は、（他のNestモジュールと同様）インポートする必要がある。あるいは以下のようにオプションオブジェクトの`isGlobal`プロパティを`true`に設定してグローバルモジュールとして宣言する事もできる。そうすれば、ルートモジュール（例：`AppModule`）にロードされた後、他のモジュールで`ConfigModule`をインポートする必要はない。

```ts
ConfigModule.forRoot({
  isGlobal: true,
});
```

## カスタムの設定ファイル

より複雑なプロジェクトでは、カスタム設定ファイルを利用して、ネストした設定オブジェクトを返す事ができる。これにより、関連する設定を機能別にグループ化したり（データベース関連の設定など）、関連する設定を個別のファイルに格納して独立して管理する事が可能になる。

カスタム構成ファイルは、構成オブジェクトを返すファクトリー関数をエクスポートする。設定オブジェクトには、自由にネストされたプレーンなJavaScriptオブジェクトを使用できる。`process.env`オブジェクトには、完全に解決された環境変数のkey/valueペアが含まれる。（`.env`ファイルと、外部で定義された変数は、上述のように解決・マージされる）返された設定オブジェクトは貴方がコントロールするので、値を適切な型にキャストしたり、デフォルト値を設定する等、必要なロジックを追加する事ができる。例：

```ts :config/configuration.ts 
export default () => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  database: {
    host: process.env.DATABASE_HOST,
    port: parseInt(process.env.DATABASE_PORT, 10) || 5432
  }
});
```

`ConfigModule.forRoot() `メソッドに渡したオプションオブジェクトの`load`プロパティを使って、このファイルを読み込む。

```ts
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [configuration],
    }),
  ],
})
export class AppModule {}
```

>Notice  
>`load`プロパティに割り当てられた値は配列だ。複数の設定ファイルを読むことができる。（例：`load:[databseConfig,authConfig]`）

カスタム設定ファイルでは、YAMLファイルなどのカスタムファイルを管理する事もできる。ここではYAMLファイルでの設定の例を紹介する。