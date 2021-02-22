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

`@nestjs/testing`パッケージは、より堅牢なテストプロセスを可能にするユーティリティセットを提供する。組み込みの`Test`クラスを使って先の例を書き換えてみよう。

```ts :cats.controller.spec.ts 
import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
  let catsController: CatsController;
  let catsService: CatsService;

  beforeEach(async () => {
    const moduleRef = await Test.createTestingModule({
        controllers: [CatsController],
        providers: [CatsService],
      }).compile();

    catsService = moduleRef.get<CatsService>(CatsService);
    catsController = moduleRef.get<CatsController>(CatsController);
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

`Test`クラスは、アプリケーション実行コンテキスト（基本的にNest全てをモックする）を提供するのに便利だが、モックやオーバーライドを含んだクラスインスタンスの管理を容易にするフックを提供する。`Test`クラスには`createTestingModule()`メソッドがあり、モジュールのメタデータオブジェクトを引数に取る（`@Module()`デコレータに渡すのと同じオブジェクト）。このメソッドは`TestingModule`のインスタンスを返す。そのインスタンスは順番にいくつかのメソッドを提供する。ユニットテストの場合重要なのは、`compile()`メソッドだ。このメソッドは依存関係を持つモジュールをブートストラップし（`NestFactory.create()`を使用して従来の`main.ts`ファイルからアプリケーションをブートストラップする方法に類似）、テストの準備ができたモジュールを返す。

>HINT
>`compile()`メソッドは**非同期**な為awaitする必要がある。モジュールが一度コンパイルされると、`get()`メソッドを使用して、モジュールが宣言している静的インスタンス（コントローラやプロバイダ）を取得する事ができる。

`TestingModule`は[モジュール参照クラス](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-modulereference)を継承している為、スコープされたプロバイダ（遷移的orリクエストスコープ）を動的に解決する機能を持っている。`resolve()`メソッドを使う（`get()`メソッドは静的インスタンスのみを取得できる）。

```ts
const moduleRef = await Test.createTestingModule({
  controllers: [CatsController],
  providers: [CatsService],
}).compile();

catsService = await moduleRef.resolve(CatsService);
```

>WARNING
>`resolve()`メソッドは、それ自身のDIコンテナのサブツリーから、プロバイダの一意のインスタンスを返す。各サブツリーは、一意のコンテキスト識別子を持っている。したがって、このメソッドを複数回呼び出してインスタンス参照を比較した時、イコールにはならない。

>HINT
>モジュールリファレンス機能の詳細については[こちら](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-modulereference)

プロバイダの本番バージョンを使用する代わりに、テスト目的のためにカスタムプロバイダでオーバーライドする事ができる。たとえば生のデータベース（live database）に接続する代わりにデータベースサービスをモックする事ができる。オーバーライドについては次のセクションで説明するが、ユニットテストでも利用可能。