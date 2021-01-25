---
title: "introduction"
---

# Introduction
Nest (NestJS) は効率的でスケーラブルなNode.jsサーバーサイドアプリケーションを構築する為のフレームワーク。最新のJavascriptで構築され、Typescriptを完全にサポートしており、にも関わらずピュアJavascriptでコーティング可能。オブジェクト指向プログラミング、関数プログラミング、リアクティブプログラミングの組み合わせで構成される。  
中身を見れば、Nestは（デフォルトでは）Expressのような堅牢なHTTPサーバーフレームワークを利用している。オプションでFastifyを使用する事もできる。  
Nestはこれらの一般的なNode.jsフレームワークよりも抽象化されているが、APIは開発者から可視化されている。したがって開発者は無数のサードパーティモジュールを自由に使える。

# 哲学
近年、Node.jsのお陰でJavaScriptはフロントエンド・バックエンドの「共通言語」となった。Angular、React、Vueのような素晴らしいプロジェクトが生まれ、開発者の生産性は向上し、速くテストしやすく拡張性のあるフロントエンドアプリケーションに繋がった。しかし、沢山のツールがNode・サーバーサイドJavaScriptの為に存在する一方で、肝心なアーキテクチャの問題を効果的に解決する物は見当たらない。  
Nestは、開発者やチームが高度にテスト可能で、スケーラブルで、疎結合で、簡単にメンテナンスできるワンタッチのアプリケーションを提供する。アーキテクチャの面で、Angularに大きく影響を受けている。

# インストール
Nestを使用する為には、NestCLIを使いプロジェクトを構築するか、スタータープロジェクトをクローンする必要がある。（結果は変わらない）  
NestCLIを使う場合、以下のコマンドを入力する。新しいプロジェクトのディレクトリができ、初期のコアNestファイルとサポート・モジュールが作成され、プロジェクトのテンプレートが作成される。NestCLIによる手法は初心者におすすめで、このガイドで言うとそのまま「First Steps」の項目に繋がる。
```
$ npm i -g @nestjs/cli
$ nest new project-name
```
他の方法として、Gitを使ってTypescript製のスタータープロジェクトをインストールする事ができる。
```
$ git clone https://github.com/nestjs/typescript-starter.git project
$ cd project
$ npm install
$ npm run start
```
ブラウザを開き`http://localhost:3000/`に移動する。  
スタータープロジェクトのJavaScript版が必要な場合、`javascript-starter.git`を使用する。  
npmやyarnを使ってNestをインストールする事もできる。その場合は勿論、プロジェクトの雛形を自分自身で作る事になる。
```
$ npm i --save @nestjs/core @nestjs/common rxjs reflect-metadata
```