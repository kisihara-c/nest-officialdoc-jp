# ガード

ガードは`@Injectable()`デコレータで装飾されたクラスです。ガードは`CanActivate`インターフェイスを実装する必要がある。  
ガードが持つ**責任は一つ**。ランタイムに存在する特定の条件（パーミッション、ロール、ACL等）に依存して、与えられたリクエストがルートハンドラによって処理されるかどうかを決定する事。これはしばしば**認可**と呼ばれる。**認可**（と、通常一緒に動く親戚的な存在の「認証」）は、伝統的なExpressアプリケーションでは通常ミドルウェアで処理されてきた。トークンの検証やリクエストオブジェクトへのプロパティのアタッチのような事柄は、特定のルートコンテキスト（とそのメタデータ）にはあまり関係がないので、ミドルウェアは検証には最適な選択といえる。  
しかし、ミドルウェアはその性質上少し頭が弱い。`next()`関数を呼び出した後どのハンドラが呼ばれるかを把握していない。**一方**ガードは`ExecutionContext`インスタンスにアクセスできる為、次に何が実行されるかを確かめられる。ガードは、例外フィルタやパイプ、インターセプタと同様に、リクエスト/レスポンスのサイクルの適切なタイミングで処理ロジックを挿入できるように設計されており、宣言的に処理を行う事ができる。よって、コードをDRYで宣言的なものに保つ事ができる。

>HINT  
>ガードは各ミドルウェアの後に実行されるが、インターセプターやパイプよりは前に実行される。

## 認可ガード
前述の通り、認証はガードにとって最適な使用例だ。これから構築する`AuthGuard`は認証されたユーザを想定している（したがって、リクエストヘッダにはトークンが添付されている。）。トークンを抽出して検証し、抽出された情報を使ってリクエストを処理できるかどうかを判断する。

```ts :auth.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

>Hint  
>アプリケーションに認証メカニズムを実装する方法の生きた例を探している場合・もっと洗練された認証機能のサンプルが欲しい場合は、authorizationの項を参照の事。

`validateRequest()`関数内のロジックは必要に応じてシンプルなものにも洗練されたものにもできる。この例の主題はリクエスト/レスポンスサイクルにガードがどのように適合するかを示す事。  
全てのガードは`canActivate()`関数を実装しなければならない。この関数は、現在のリクエストが許可されているかどうかを示すブール値を返すだろう。この関数は、同期的にも非同期的（`Promise`、`Observable`を使って）にもレスポンスを返す事ができる。Nestは次のアクションを制御する為に、戻り値を使用する。
- true時リクエストは処理され、`false`時Nestはリクエストを拒否する。
- falseを返した場合、Nestはリクエストを拒否する。

## ExecutionContext
`canActivate()`関数は、単一の引数である`ExecutionContext`インスタンスを受け取る。`ExecutionContext`は`ArgumentsHost`を継承する。我々は以前に例外フィルタの項でも`ArgumentsHost`を見た。上記のサンプルでは、以前に使用した`Arg～`で定義されたものととまさに同じヘルパーメソッドを使っている。このトピックの詳細については、例外フィルタの項の該当セクションを参照の事。  
`ArgumentsHost`を拡張する事で、`ExectutionContext`は現在の実行プロセスに関するいくつかの新しいヘルパーメソッドも手に入れた。より幅広い範囲のコントローラ・メソッド・及び実行コンテキストで動く、より汎用的なガードを構築する際に役に立つ。詳細はexecution-contextの項を参照の事。

# ロールベースの認証
特定のロールを持つユーザのみにアクセスを許可する、より機能的なガードを構築してみよう。基本的なテンプレートから構築していく。まずは全てのリクエストを許可するコードを書く。

```ts :roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
```

## バインドするガード
パイプや例外フィルタと同様に、ガードもコントローラ・メソッド・グローバルのいずれかにスコープ化できる。以下、`@UseGuards()`デコレータを使用してコントローラスコープ付きガードを設定する。このデコレータは、単一の引数、またはカンマで区切られた引数のリストを受け取ることができる。結果、適切なガードのセットを一つの宣言で簡単に設定する事ができる。

```ts
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

>Hint
>`@UseGuards()`デコレータは`@nestjs/common`パッケージからインポートされている。

上記では（インスタンスの代わりに）RoleGuard型を渡しているが、フレームワークにインスタンス化の責任を任せ、依存性のインジェクションを可能にしている。パイプや例外フィルタと同様に、in-placeなインスタンスを渡すこともできる。

```ts
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

上記ではこのコントローラで宣言された全てのハンドラにガードをアタッチしています。ガードを単一のメソッドにのみ適用したい場合は、メソッドレベルで@UseGurds()デコレータを適用します。  
グローバルなガードを設定するには、Nestアプリケーションインスタンスの`useGlobalGuards()`メソッドを使用します。

```ts
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

>お知らせ
>ハイブリッドアプリ（hybrid apps）の場合、`useGlobalGuards()`メソッドは、デフォルトではゲートウェイとマイクロサービスのガードをセットアップしない。（この動作を変更する方法はhybrid-applicationの項を参考の事。）「（ハイブリッドではない）標準」のマイクロサービスアプリ（microservice app）の場合、`useGlobalGuards()`はグローバルにガードをマウントする。

グローバルガードはすべてのコントローラと全てのルートハンドラに対してアプリケーション全体で使用される。依存関係の面では、任意のモジュールの外部から登録されたグローバルガードは（上記の例のように`useGlobalGuards()`を使用して）依存関係を注入することができない。全てのモジュールの外で実行されるからだ。この問題を解決する為に、以下の構成で全てのモジュールから直接ガードを設定できる。

```ts :app.module.ts 
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```

>HINT  
>このアプローチを採用してガードの依存性インジェクションを実行する場合、この構造が採用されているモジュールに関係なく、ガードは実際にはグローバルであることに注意。どこでやるべきか？　ガード（上の例では`RolesGuard`）が定義されているモジュールを選ぼう。また、カスタムプロバイダの登録を扱う方法は`useClass`だけではない。詳細はcustom-providersの項にて。

## ハンドラごとのロールの設定
この`RolesGuard`は動いているがまだあまりスマートではない。最も重要なガードの機能である実行コンテキストをまだ活用できていない。`RolesGuard`はロールについて、あるいは各ハンドラにどのロールが許可されているかをまだ把握していない。例えば、`CatsController`はルートごとに異なるパーミッションのスキームを持つ事ができる。管理者のみが利用できるものも、誰でも利用できるものも用意できる。柔軟で再利用可能な方法でロールとルートを一致させるにはどうすればいいだろうか？  
ここでカスタムメタデータの出番となる（詳細はexecution-context項の該当項目にて）。Nestは`@SetMetadata()`デコレータを使ってカスタムメタデータをルートハンドラにアタッチする機能を提供している。このメタデータは、スマートなガードが意思決定をするために必要な、欠けていたロールデータを提供する。`@SetMetadata()`を使った方法を見てみよう。

```ts :cats.controller.ts
@Post()
@SetMetadata('roles', ['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

>HINT
>`@SetMetadata()`デコレータは`@nestjs/common`パッケージからインポートされている。

上記のコードでは、`roles`のメタデータ（`roles`はキーで、['admin']は特定の変数）を`create()`メソッドにアタッチしている。このコードは動作するが、ルートで直接`@SetMetadata()`を使うのは良い習慣とはいえない。代わりに、以下のように独自のデコレータを作成してほしい。

```ts :roles.decorator.ts 
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

このアプローチのほうがずっとすっきりしていて読みやすい。強固な型付けもできている。カスタムの`@Roles()`デコレータができたので、`create()`メソッドに使ってみよう。

```ts :cats.controller.ts 
@Post()
@Roles('admin')
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

## 全てまとめて
これを`RolesGuard`と結びつけてみよう。現在の所、`RolesGuard`はシンプルにすべての場合に`true`を返し、すべてのリクエストに許可をする。現在のユーザに割り当てられたロールと、処理されている現在のルートで必要な実際のロールを比較して、戻り値を条件付きにしたい。ルートのロール（カスタムメタデータ）にアクセスするために、`Reflector`ヘルパークラスを使用する。

```ts :roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```

>Hint  
>node.jsの世界では、許可されたユーザーを`request`オブジェクトにアタッチするのが一般的だ。したがって、上記のサンプルコードでは、ユーザーインスタンスと許可されたロールが`request.user`に含まれていると仮定している。アプリでは、おそらくカスタム認証ガード（かミドルウェア）でこの関連付けを行う事になるだろう。このトピックの詳細は`Authentication`の章を参照の事。

>WARNING  
>`matchRoles()`関数内のロジックは、必要に応じてシンプルにも洗練されたものにもできる。この例の主なポイントは、リクエスト/レスポンスサイクルにガードがどのように適合するかを示す事だ。

コンテキストに依存した方法で`Reflector`を利用する方法の詳細については、Execution context chapter項のReflection and metadataセクションを参考の事。  
十分な権限を持たないユーザがエンドポイントを要求すると、Nestは自動で以下の応答を返す。

```ts
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

この裏では、ガードがfalseを返すとフレームワークが`ForbiddenExeption`をthrowする事に注意。別のエラーレスポンスを返したい場合は、独自の例外を投げる必要がある。例えば以下等。

```ts
throw new UnauthorizedException();
```

ガードによって投げられた例外は例外レイヤー（グローバル例外フィルタと現在のコンテキストに適用されているすべての例外フィルタ）によって処理される。

>HINT  
>認証を実装する方法の詳細はAuthorization項を参照の事。