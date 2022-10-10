---
title: "overview-firststeps"
---

# First steps
このチャプターでは、Nestの最も重要な基礎を学ぶ。入門レベルの機能をカバーしたシンプルなCRUDアプリケーションの作成により、Nestアプリケーションの本質的な構成に慣れ親しむ。

## 言語
当フレームワークはTypeScriptへの恋に落ちている。しかしそれにもましてNode.jsも愛している。よってNestはTypeScriptとJavaScriptを両立させている。Nestは言語の最新機能に強みを持つ。そのため、バニラJavaScriptで動かすにはBabel compilerを用いる必要がある。  
サンプルコードはTypeScript製だが、公式のドキュメントでは右上のボタンを押せばどのサンプルもバニラJavaScriptのコードに変換可能。

## 必要条件
Node.js（10.13.0以上）のインストールが必要。

## 導入
セットアップは簡単で、「NestCLI」を使う。npmがあれば、ターミナルから以下のコマンドでインストールする。

```
$ npm i -g @nestjs/cli
$ nest new project-name
```

`project`ディレクトリ、nodeモジュールといくつかのボイラープレートファイル、基本ファイル数点の入った`src/`ディレクトリが作成される。

```
src
 app.controller.ts
 app.controller.spec.ts
 app.module.ts
 app.service.ts
 main.ts
```

コアファイルの概要は以下の通り。

|||
| ---- | ---- |
| app.controller.ts | 簡便なシングルルート用コントローラ |
| app.controller.spec.ts | ユニットテスト用コントローラ |
| app.module.ts | アプリケーションのルートモジュール |
| app.service.ts | シングルメソッドの簡便なサービス |
| main.ts | NestFactory機能を使いNest アプリケーションインスタンスを作る為のファイル |

main.tsにはアプリケーションを起動するasync関数が含まれている。

```ts:main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

Nestアプリケーションインスタンスを作る為、コアの`NestFactory`クラスを使う。その際`NestFactory`クラスではいくつかの静的関数が利用可能。`create()`メソッドはアプリケーションオブジェクトを返し、`INestApllication`インターフェイスを持つ。此のオブジェクトは次回以降説明する関数のセットを提供する。上記のmain.tsのコードでは、HTTPリクエストを待つ簡単なHTTPリスナーを作っている。  
NestCLIで組み立てられたプロジェクトは、自身のプロジェクトのそれぞれのモジュールを維持しやすいような初期構造になっている。

## プラットフォーム
Nestはplatform-agnostic（プラットフォーム不可知）なフレームワークを目指している。プラットフォームの独立性は使い回しやすいロジックに繋がり、開発者は複数のアプリケーションを横断してその利点を享受できる。技術的には、Nestは一度作られたどんなNodeのHTTPフレームワークとも動く。特にexpress・fastifyは基本セットとして準備済みで、必要に応じて使い分ける事ができる。

|||
| ---- | ---- |
| `platform-express` | Expressはよく知られたNodeのマイクロフレームワーク。 多くの実戦をくぐり抜け、コミュニティと潤沢なリソースによって、すぐにでも使える沢山のライブラリがある。Nestのデフォルト設定では`@nestjs/platform-express `が使用される。多くのユーザはExpressを使用するし、その為の手間は不要である。|
| `platform-fastify` | Fastifyはハイパフォーマンスでオーバーヘッドの少ない最高効率と最高速度を提供するフレームワーク。使い方は[こちら](https://docs.nestjs.com/techniques/performance) |
| ---- | ---- |

どのプラットフォームと使われても、Nestはそれ自体がアプリケーションインターフェイスを持つ。上記の例だと、`NestExpressApplication`と`NestFastifyApplication`となる。  
以下の例のように、`NestFactory.create()`メソッドに型を渡すと、`app`オブジェクトは特定のプラットフォーム専用のメソッドを手に入れる。ただし、実際に基礎となるプラットフォームAPIにアクセスしたい場合を除き、型を指定する必要はない。

`const app = await NestFactory.create<NestExpressApplication>(AppModule);`

## アプリケーションを起動する
インストールが完了すれば、以下のコマンドでHTTPリクエストを待つアプリケーションを起動できる。

`$ npm run start`

このコマンドを使用すれば、`src/main.ts`で定義されたポートで待つHTTPサーバーアプリケーションが起動する。ブラウザが開き`http://localhost:3000/`を表示する。`Hello world`が見えるはずだ。
