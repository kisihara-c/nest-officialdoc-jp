---
title: fundamentals-modulereference
---

# モジュール参照
Nestは`ModuleRef`クラスを用意している。内部のプロバイダのリストをナビゲートし、そのインジェクショントークンを検索キーとして任意のプロバイダへの参照を取得する為のものだ。`ModuleRef`クラスはまた、静的プロバイダ及びスコープ化プロバイダを動的にインスタンス化する方法も提供する。`ModuleRef`は通常の方法でクラスにインジェクションする事ができる。

```ts :cats.service.ts
@Injectable()
export class CatsService {
  constructor(private moduleRef: ModuleRef) {}
}
```

>HINT  
>`ModuleRef`クラスは`@nestjs/core`パッケージからインポートする事。


## インスタンスの取得
`ModuleRef`インスタンス（以下、**モジュールリファレンス**と呼称）には`get()`メソッドがある。**現在の**モジュールに存在する（かつ、インスタンス化されている）もの（プロバイダ、コントローラ、インジェクション可能なオブジェクト（ガード、インターセプター等））を、そのインジェクショントークン/クラス名を使って取得できる。

```ts :cats.service.ts
@Injectable()
export class CatsService implements OnModuleInit {
  private service: Service;
  constructor(private moduleRef: ModuleRef) {}

  onModuleInit() {
    this.service = this.moduleRef.get(Service);
  }
}
```

>WARNING  
>スコープ化されたプロバイダ（遷移的、もしくはリクエストスコープ）はget()メソッドで取得不能。代わりに次節のテクニックを使用の事。スコープの制御の方法については[こちら](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-injectionscopes)

グローバルコンテキストからプロバイダを取得するには（たとえばプロバイダが別のモジュールにインジェクションされている場合等）、get()の第二引数に`{ strict: false }`オプションを渡してほしい。

```ts
this.moduleRef.get(Service, { strict: false });
```

## スコープ化されたプロバイダの解決

スコープ化されたプロバイダ（遷移的もしくはリクエストスコープ）を動的に解決（resolve）するには`resolve()`メソッドを使い、プロバイダのインジェクショントークンを引数として渡す。

```ts : cats.service.ts
@Injectable()
export class CatsService implements OnModuleInit {
  private transientService: TransientService;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.transientService = await this.moduleRef.resolve(TransientService);
  }
}
```

`resolve()`メソッドは、自分自身のDIコンテナのサブツリーから、プロバイダの一意のインスタンスを返す。各サブツリーは一意のコンテキスト識別子を持っている。したがって、このメソッドを複数回呼び出してインスタンスリファレンスを比較すると、等しくない。

```ts :cats.service.ts
@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService),
      this.moduleRef.resolve(TransientService),
    ]);
    console.log(transientServices[0] === transientServices[1]); // false
  }
}
```

複数の`resolve()`コールを超えて単一のインスタンスを生成し、生成されたDIコンテナのサブツリーを共有するようにする為には、`resolve()`メソッドにコンテキスト識別子を渡す事ができる。コンテキスト識別子を生成するには、`ContextIdFactory`クラスを使用する。このクラスは適切な一意の識別子を返す`create()`メソッドを提供する。

```ts :cats.service.ts
@Injectable()
export class CatsService implements OnModuleInit {
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    const contextId = ContextIdFactory.create();
    const transientServices = await Promise.all([
      this.moduleRef.resolve(TransientService, contextId),
      this.moduleRef.resolve(TransientService, contextId),
    ]);
    console.log(transientServices[0] === transientServices[1]); // true
  }
}
```

>HINT
>`ContextIdFactory`クラスは`@nestjs/core`パッケージからインポートの事。

## `REQUEST`プロバイダの登録

手動で生成されたコンテキスト識別子（`ContextIdFactory.create()`を使用）は、Nestの依存性インジェクションシステムによってインスタンス化・管理されていない為、`REQUEST`が`undefined`のままであるDIサブツリーを表す。

手動で作成したDIサブツリーにカスタム`REQUEST`オブジェクトを登録するには、以下のように`ModuleRef#registerRequestByContextId()`メソッドを使う。

```ts
const contextId = ContextIdFactory.create();
this.moduleRef.registerRequestByContextId(/* YOUR_REQUEST_OBJECT */, contextId);
```

## 現在のサブツリーの取得

**リクエストのコンテキスト**内でリクエストスコープ化されたプロバイダのインスタンスを解決（resolve）したい場合がある。例えば、`CatsService`がリクエストスコープ化されており、リクエストスコープ化されたプロバイダとして記録されている`CatsRepository`のインスタンスを解決したいとする。同じDIコンテナサブツリーを共有する為には、新しいコンテキストの識別ではなく、現在のコンテキスト識別子の取得が必要（例えば、上記のように`ContextIdFactory.create()`メソッドを使用する）。現在のコンテキスト識別子を取得するには、`@Inject()`デコレータを使用してリクエストオブジェクトをインジェクションする事から始める。

```ts :cats.service.ts 
@Injectable()
export class CatsService {
  constructor(
    @Inject(REQUEST) private request: Record<string, unknown>,
  ) {}
}
```

>HINT  
>requestプロバイダについて更に詳しくは[こちら](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-injectionscopes#%E3%83%AA%E3%82%AF%E3%82%A8%E3%82%B9%E3%83%88%E3%83%97%E3%83%AD%E3%83%90%E3%82%A4%E3%83%80)

では、`ContextIdFactory`クラスの`getByRequest()`メソッドを使用する。リクエストオブジェクトを元にコンテキストIDを作成し、それを`resolve()`の呼び出しに渡す。

```ts
const contextId = ContextIdFactory.getByRequest(this.request);
const catsRepository = await this.moduleRef.resolve(CatsRepository, contextId);
```

## カスタムクラスのインスタンスを動的に作成する

以前はプロバイダとして登録されていなかったクラスを動的にインスタンス化するには、モジュール参照の`create()`メソッドを使用。

```ts cats.service.ts
@Injectable()
export class CatsService implements OnModuleInit {
  private catsFactory: CatsFactory;
  constructor(private moduleRef: ModuleRef) {}

  async onModuleInit() {
    this.catsFactory = await this.moduleRef.create(CatsFactory);
  }
}
```
このテクニックによって、フレームワークコンテナの外で、異なるクラスを条件付きでインスタンス化できる。