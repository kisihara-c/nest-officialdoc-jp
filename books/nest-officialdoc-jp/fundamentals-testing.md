---
title: fundamentals-testing
---

# テスト

自動化テストは大切なソフトウェア開発の取り組みには不可欠だ。自動化によって、開発中に個々のテストやテストスイートを素早く簡単に繰り返せる。結果、リリースが品質とパフォーマンスの目標を確実に達成できるようになる。自動化はカバレッジを高め開発者へのフィードバックループを高速化するのに役立つ。個々の開発者の生産性を向上させると同時に、ソースコード管理のチェックイン、機能統合、バージョンリリースなど、開発ライフサイクルの重要な分岐点で、確実にテストを実行する事ができる。

このようなテストはユニットテスト、エンドツーエンド（e2e）テスト、統合テストなどさまざまなタイプがある。それらの利点は疑いようがないが、設定が面倒な場合もある。Nestは効果的なテストを含めた開発のベストプラクティスの推進に努めており、開発者やチームがテストを構築して自動化する為に以下の機能を搭載している。

（わからない単語が多く直訳が多くなってしまっています…）
- コンポーネント用のデフォルトのユニットテストとアプリケーション用のe2eテストを自動でスカフォールドとする
- デフォルトのツールを提供する（孤立したモジュール/アプリケーションローダーをビルドするテストランナーなど）
- [Jest](https://github.com/facebook/jest)や[Supertest](https://github.com/visionmedia/supertest)との統合を提供し、テストツールへの依存を避けながらも簡単に使える
- テスト環境でNestの依存性インジェクションシステムを利用できるようにして、コンポーネントを簡単にモックできる

前述の通り、Nestは特定のツールを強制しないので、好きな**テストフレームワーク**を使える。必要な要素（テストランナー等）を置き換えるだけで、Nestデフォルトのテスト機能の利点を享受できる。

## インストール
初めに必要なパッケージをインストールする。

```
$ npm i --save-dev @nestjs/testing
```

## ユニットテスト
次の例では２つのクラスをテストする。`CatsController`と`CatsService`だ。前述したように、Jestはデフォルトのテストフレームワークとして提供されている。テストランナーとして機能し、アサート関数やテストダブルユーティリティを提供する。以下の基本的なテストでは、これらのクラスを手動でインスタンス化し、コントローラとサービスがAPI contractを満たす事を確認する。

```ts :cats.controller.spec.ts 
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(() => {
    catsService = new CatsService();
    catsController = new CatsController(catsService);
  });

  describe('findAll', () => {
    it('should return an array of cats', async () => {
      const result = ['test'];
      jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

      expect(await catsController.findAll()).toBe(result);
    });
  });
});
```

>HINT  
>テストファイルはテストするクラスの近くに配置してほしい。テストファイルの拡張子は.specまたは.testにしてほしい。

上記のサンプルはちょっとしたものなので、実際にNestで特化したテストはしていない。実際依存性インジェクションも使っていない（`CatsService`のインスタンスを`catsController`に渡している事に注意）。この形式のテスト（テストされるクラスを手動でインスタンス化する）はフレームワークから独立している為、分離型テストと呼ばれる事が多い。ここでは、Nestの機能をより広範囲に利用するアプリケーションのテストに役立つ、より高度な機能を紹介する。

## テストユーティリティ