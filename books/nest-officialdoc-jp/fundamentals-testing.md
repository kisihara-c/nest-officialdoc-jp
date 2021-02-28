---
title: fundamentals-testing
---

# テスト

自動化テストはソフトウェアのシリアスな開発には不可欠だ。自動化によって、開発中に個々のテストやテストスイートを素早く簡単に繰り返せる。結果、リリースが品質とパフォーマンスの目標を確実に達成できるようになる。自動化はカバレッジを高め開発者へのフィードバックループを高速化するのに役立つ。個々の開発者の生産性を向上させると同時に、ソースコード管理のチェックイン、機能統合、バージョンリリースなど、開発ライフサイクルの重要な分岐点で、確実にテストを実行する事ができる。

このようなテストはユニットテスト、エンドツーエンド（e2e）テスト、統合テストなどさまざまなタイプがある。それらの利点は疑いようがないが、設定が面倒な場合もある。Nestは効果的なテストを含めた開発のベストプラクティスの推進に努めており、開発者やチームがテストを構築して自動化する為に以下の機能を搭載している。

（わからない単語が多く直訳が多くなってしまっています…）
- コンポーネント用のデフォルトのユニットテストとアプリケーション用のe2eテストを自動でスカフォールドできる
- デフォルトのツールを提供する（孤立したモジュール/アプリケーションローダーをビルドするテストランナーなど）
- [Jest](https://github.com/facebook/jest)や[Supertest](https://github.com/visionmedia/supertest)との統合を提供、テストツールへの依存を避けながらも簡単に使える
- テスト環境でNestの依存性インジェクションシステムを利用できるようにして、コンポーネントを簡単にモックできる

前述の通り、Nestは特定のツールを強制しないので、好きな**テストフレームワーク**を使える。必要な要素（テストランナー等）を置き換えるだけで、Nestデフォルトのテスト機能の利点を享受できる。

## インストール
初めに必要なパッケージをインストールする。

```
$ npm i --save-dev @nestjs/testing
```

## ユニットテスト
次の例では２つのクラスをテストする。`CatsController`と`CatsService`だ。前述したように、[Jest](https://github.com/facebook/jest)はデフォルトのテストフレームワークとして提供されている。テストランナーとして機能し、アサート関数やテストダブルユーティリティを提供する。以下の基本的なテストでは、これらのクラスを手動でインスタンス化し、コントローラとサービスがそのAPIコントラクトを満たす事を確認する。

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

上記のサンプルはちょっとしたものなので、実際にNestに特有のものについてのテストはしていない。実際依存性インジェクションも使っていない（`CatsService`のインスタンスを`catsController`に渡している事に注意）。この形式のテスト（テストされるクラスを手動でインスタンス化する）はフレームワークから独立している為、分離型テストと呼ばれる事が多い。ここでは、Nestの機能をより広範囲に利用したアプリケーションのテストに役立つ、より高度な機能を紹介する。

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

プロバイダの本番バージョンを使用する代わりに、テスト目的のために[カスタムプロバイダ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-customproviders)でオーバーライドする事ができる。たとえば生のデータベース（live database）に接続する代わりにデータベースサービスをモックする事ができる。オーバーライドについては次のセクションで説明するが、ユニットテストでも利用可能。

## E2Eテスト
個々のモジュールやクラスに焦点を当てる単体テストとは異なり、E2Eテストはクラスやモジュールの相互作用を、より集約的なレベルで、エンドユーザと本番システムとの相互関係に近いレベルでカバーする。アプリケーションが大きくなるにつれて、各APIエンドポイントのエンドツーエンドの振る舞いの手動のテストは難しくなる。自動化されたE2Eテストは、システムの全体的な動作が正しく、プロジェクトの要件を満たしている確認に役立つ。E2Eの実行の際は、先の**ユニットテスト**と同様の構成を使用する。さらに、Nestでは[Supertest](https://github.com/visionmedia/supertest)ライブラリでHTTPリクエストを簡単にシミュレートできる。

```ts :cats.e2e-spec.ts 
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
      .overrideProvider(CatsService)
      .useValue(catsService)
      .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({
        data: catsService.findAll(),
      });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

>HINT  
>HTTPアダプタとしてFastifyを使用している場合は、少し異なる設定が必要で、テスト機能は組み込みとなる。
>
>```ts
>let app: NestFastifyApplication;
>
>beforeAll(async () => {
>  app = moduleRef.>createNestApplication<NestFastifyApplic>ation>(
>    new FastifyAdapter(),
>  );
>
>  await app.init();
>  await app.getHttpAdapter().getInstance>().ready();
>})
>
>it(`/GET cats`, () => {
>  return app
>    .inject({
>      method: 'GET',
>      url: '/cats'
>    }).then(result => {
>      expect(result.statusCode).toEqual>(200)
>      expect(result.payload).toEqual(/* >expectedPayload */)
>    });
>})
>```

この例では説明済みの概念のいくつかをコードにしている。先に使用した`compile()`メソッドに加えて、Nestの完全な起動環境をインスタンス化する為に、`createNestApplication()`メソッドを使用している。実行中のアプリへの参照を`app`変数に保存し、HTTPリクエストをシミュレートするために使っている。

Supertestの`request()`関数を使用してHTTPテストをシミュレートする。HTTPリクエストを実行中のNestアプリに転送したいので、`request`関数にNestの基盤となっている（Expressによって順番に渡されているであろう）HTTPリスナーへの参照を渡す。（※「ので以降」原文：so we pass the request() function a reference to the HTTP listener that underlies Nest (which, in turn, may be provided by the Express platform)）したがって、コードは`request(app.getHttpServer())`となる。`request()`の呼び出しは、ラップされたHTTPサーバーに繋がり、その先ではNestアプリに接続される。結果として、実際のHTTPリクエストを模倣するメソッドを表出させる。（※原文：The call to request() hands us a wrapped HTTP Server, now connected to the Nest app, which exposes methods to simulate an actual HTTP request. ）例えば、`request(...).get('/cats')`を使うと、ネットワーク経由で送られてくる`/cats`を取得するような実際のHTTPリクエストと同じリクエストをNestアプリに作り出す。

この例では、テスト可能なハードコードを返す`CatsService`の代替（テストダブル）の実装も提供している。`overrideProvider()`を使用する。同様にNestは、ガード、インターセプター、フィルタ、パイプをオーバーライドするメソッド　`overrideGuard()`、`overrideInterceptor()`、`overrideFilter()`、`overridePipe()`を提供する。

それぞれのオーバーライドメソッドは、[カスタムプロバイダ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-customproviders)の為の記述を反映した、３つの異なるメソッドを持つオブジェクトを返す。

|||
| ---- | ---- |
|`useClass`|（オブジェクトをオーバーライドするインスタンスを提供する為にインスタンス化される）クラスを提供する（プロバイダ、ガード等）|
|`useValue`|オブジェクトをオーバーライドするインスタンスを提供する|
|`useFactory`|オブジェクトをオーバーライドするインスタンスを返す関数を提供する|

>HINT  
>e2eのテストファイルはテストディレクトリ内に保管してほしい。また、拡張子は.e2e-specにて。

## グローバルに登録されたエンハンサーのオーバーライド

グローバルに登録されたガード（もしくはパイプ、インターセプタ、フィルタ）を持っている場合、そのエンハンサーを上書きする為にいくつか手順がある。まず本来の登録作業は以下の通り。

```ts
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

これは`APP_*`トークンを介して"multi"なプロバイダとしてガードを登録している。ここで`JwtAuthGuard`を置き換えられるようにするには、このスロットにおいてすでに存在するプロバイダを使用する必要がある。

```ts
providers: [
  {
    provide: APP_GUARD,
    useExisting: JwtAuthGuard,
  },
  JwtAuthGuard,
],
```

>HINT  
>`useClass`を`useExisting`に変更。Nestがトークンの背後でインスタンスを作成する代わりに、登録されたプロバイダを参照するようにした。

こうすると、`JwtAuthGuard`は、`TestingModule`を作成する際にオーバーライドできる通常のプロバイダとして扱えるようになる。

```ts
const moduleRef = await Test.createTestingModule({
  imports: [AppModule],
})
  .overrideProvider(JwtAuthGuard)
  .useClass(MockAuthGuard)
  .compile();
```

これで、すべてのテストはすべてのリクエストに対して`MockAuthGuard`を使用するようになった。

## リクエストスコープ化されたインスタンスをテストする

[リクエストスコープ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-injectionscopes)化されたプロバイダは、入ってくる各リクエストに対して一意に作成される。インスタンスは、リクエストの処理完了後にガベージコレクションされる。これは問題だ、テスト用リクエストの為に特別に生成された依存性インジェクションのサブツリーにアクセスできない。

上記のセクションに基づくと、`resolve`メソッドを使えば動的にインスタンス化されたクラスを取得できることがわかる。また、[ここ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-modulereference#%E3%82%B9%E3%82%B3%E3%83%BC%E3%83%97%E5%8C%96%E3%81%95%E3%82%8C%E3%81%9F%E3%83%97%E3%83%AD%E3%83%90%E3%82%A4%E3%83%80%E3%81%AE%E8%A7%A3%E6%B1%BA)で説明したように、DIコンテナサブツリーのライフサイクルを制御する為に、一意のコンテキスト識別子を渡せる事もわかっている。この事をテストのコンテキストで活用するにはどうすれば良いだろう？

方針は、事前にコンテキスト識別子を生成し、Nestにこの一意のIDを使って全ての受信リクエストのサブツリーを作成させる事だ。そうすれば、テスト用リクエストの為に作成されたインスタンスを取得できる。

実現のため、`ContextIdFactory`で`jest.spieOn()`を使う。

```ts
const contextId = ContextIdFactory.create();
jest
  .spyOn(ContextIdFactory, 'getByRequest')
  .mockImplementation(() => contextId);
```

結果、続くリクエストに対して、生成された単一のDIコンテナサブツリーにアクセスする為に`contextId`を使えるようになった。

```ts
catsService = await moduleRef.resolve(CatsService, contextId);
```