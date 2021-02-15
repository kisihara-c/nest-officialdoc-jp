---
title: fundamentals-injectionscopes
---

# インジェクションスコープ
Nestではほぼ全てのものがincoming requestを通して共有されている。別のプログラミング言語の経験者にとっては意外かもしれない。データベースへの接続プール、グローバルなシングルトンサービス等を使える。思い出してほしい、Node.jsは全てのリクエストが個別のスレッドによって処理されるrequest/responseマルチスレッドステートレスモデルに従っていない。したがって、シングルトンインスタンスを使っても完全に**安全**だ。

しかしrequestベースのライフタイムが望ましい場合もある。例えば、GraphQLアプリケーションでリクエストごとにキャッシングする時や、マルチテナンシー時等。インジェクションスコープは、求めるプロバイダのライフタイム的な振る舞いを得る為の機能だ。

## プロバイダスコープ
プロバイダは以下のスコープのいずれかを持つ事ができる。

|||
| ---- | ---- |
|`DEFAULT`|プロバイダの単一のインスタンスはアプリケーション全体で共有される。インスタンスの有効期限は、アプリケーションのライフサイクルに直接結び付けられる。アプリケーションが起動すると、全てのシングルトンプロバイダがインスタンス化される。デフォルトではシングルトンスコープが使用される。|
|`REQUEST`|プロバイダの新しいインスタンスは、各incoming requestの為だけに作成される。このインスタンスはrequestの処理完了後ガベージコレクションで処理される。|
|`TRANSIENT`|トランジェントプロバイダはconsumersの間で共有されない。トランジェントプロバイダをインジェクションする各consumerは、新しい専用のインスタンスを受け取る。|

>Hint  
>ほとんどのユースケースではシングルトンスコープの使用が**オススメ**。consumersやリクエストを通してプロバイダを共有するというのはつまり、インスタンスをキャッシュできる事、初期化がアプリケーションの起動時に一度だけ行われる事を意味する。

## 使用法
`scope`プロパティを`@Injectable()`デコレータのオプションオブジェクトに渡すことで、インジェクションスコープを指定しよう。

```ts
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {}
```

同様に、[カスタムプロバイダ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-customproviders)の場合はプロバイダを登録する為の手書きフォームにscopeプロパティを設定しよう。


```ts
{
  provide: 'CACHE_MANAGER',
  useClass: CacheManager,
  scope: Scope.TRANSIENT,
}
```

>HINT  
>`scope`enumは`@nestjs/common`からインポートする事。

>NOTICE  
>ゲートウェイはシングルトンとして動作しなければならない為、リクエストスコープ化されたプロバイダを使用するべきではない。各ゲートウェイは実際のソケットをカプセル化しており、複数回インスタンス化する事はできない。

シングルトンスコープはデフォルトで使用される。宣言する必要はない。プロバイダをシングルトンスコープとして宣言したい場合は`scope`プロパティで`Scope.DEFAULT`値を使う。

## コントローラのスコープ
コントローラもまたスコープを持つ。そのコントローラで宣言された全てのrequestメソッドハンドラに適用される。プロバイダのスコープと同様に、コントローラのスコープはそのライフタイムを宣言する。リクエストスコープ化されたコントローラの場合、受信したrequestごとに新インスタンスが作られ、requestの処理が完了次第ガベージコレクタで処理される。

`ControllerOptions`オブジェクトの`scope`プロパティでコントローラのスコープを宣言しよう。

```ts
@Controller({
  path: 'cats',
  scope: Scope.REQUEST,
})
export class CatsController {}
```

## スコープ階層
スコープはインジェクションチェーンを上部に浮かび上がらせる（bubbles up）。リクエストスコープ化されたプロバイダに依存するコントローラはそれ自体がリクエストスコープ化される。

`CatsController <- CatsService <- CatsRepository`  
こんな依存関係グラフを想像してほしい。もし`CatsService`がリクエストスコープ化されている場合（そうでなければデフォルトでシングルトンスコープ）、`CatsController`はインジェクションされたサービスに依存している為、そのままリクエストスコープ化される。`CatsRepository`は依存していないのでシングルトンスコープのままだ。

## リクエストプロバイダ
HTTPサーバーベースのアプリケーション（例：`@nestjs/platform-express`や`@nestjs/platform-fastify`）を使用中で、リクエストスコープ化されたプロバイダを使っている時、元のrequestオブジェクトへの参照にアクセスしたい場合があるかもしれない。これは`REQUEST`オブジェクトをインジェクションすれば可能だ。

```ts
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(REQUEST) private request: Request) {}
}
```

基本的なプラットフォーム/プロトコルの違いが理由で、MicroserviceやGlaphQLアプリケーションではインバウンドリクエストへのアクセス方法が若干異なる。GraphQLアプリケーションでは`REQUEST`の代わりに`CONTEXT`をインジェクションする。

```ts
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT } from '@nestjs/graphql';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private context) {}
}
```

そして、`context`変数（`GraphQLModule`内）を、プロパティとして`request`を持つように設定する。

## パフォーマンス
リクエストスコープ化されたプロバイダはアプリケーションのパフォーマンスに影響を与える。Nestは可能な限り多くのメタデータをキャッシュしようとする中、リクエストごとにクラスのインスタンスを作成する必要がある。したがって、平均応答時間や全体的なベンチマーク結果が遅くなる。プロバイダがリクエストスコープされなければならない場合を除き、デフォルトのシングルトンスコープの仕様を強く勧める。