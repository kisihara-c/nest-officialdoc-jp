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