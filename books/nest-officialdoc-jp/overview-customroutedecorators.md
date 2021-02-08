---
title: overview-customroutedecorators
---

# カスタムルートデコレータ
Nestの構築はデコレータと呼ばれる言語機能が中心だ。デコレータは多くの有名プログラミング言語では馴染みのある概念だが、JavaScriptの世界ではまだ比較的新しい概念となる。デコレータの仕組みをよりよく理解する為、[この記事](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841)を読もう。簡単な定義なら以下の通り。  
ES2016のデコレータは、関数を返し、引数としてターゲット、名前、プロパティ記述子を取れる式である。各デコレータの前に@マークをつけて設定し、適用したい対象の最上部に配置する。デコレータはクラス、メソッド、プロパティのいずれかに定義可能。

## Paramデコレータ
NestはHTTPルートハンドラと一緒に使用できる便利なParamデコレータを提供している。以下に、提供されているデコレータのリストと、各々が表すプレーンなExpress（かFastify）オブジェクトを示す。

|||
| ---- | ---- |
|@Request(), @Req()|req|
|@Response(), @Res()|res|
|@Next()|next|
|@Session()|req.session|
|@Param(param?: string)|req.params / req.params[param]|
|@Body(param?: string)|req.body / req.body[param]|
|@Query(param?: string)|req.query / req.query[param]|
|@Headers(param?: string)|req.headers / req.headers[param]|
|@Ip()|req.ip|
|@HostParam()|req.hosts|

加えて、独自のカスタムデコレータを作成する事ができる。とても便利だ。なぜか？  
node.jsの世界では、requestオブジェクトにプロパティをアタッチするのが一般的だ。そうなると貴方は、以下のようなコードを使用して、各ルートハンドラの中、手動でそれらを抽出する事になる。

```ts
const user = req.user;
```

コードをもっと読みやすく明晰にする為、`@User()`デコレータを作成して、全てのコントローラで再利用する事ができる。

```ts :user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  },
);
```

これで必要な場所で簡単に使える。

```ts
@Get()
async findOne(@User() user: UserEntity) {
  console.log(user);
}
```

## データを渡す
デコレータの動作がいくつかの条件に依存する場合、dataパラメータを使用してデコレータのファクトリー関数に引数を渡すことができる。その為の一つのユースケースとして、requestオブジェクトからキーによってプロパティを抽出するカスタムデコレータを紹介する。例えば、認証レイヤがリクエストを検証し、ユーザエンティティをrequestオブジェクトにアタッチするとしてみよう。認証済みリクエストのユーザエンティティは次のようになる。

```ts
{
  "id": 101,
  "firstName": "Alan",
  "lastName": "Turing",
  "email": "alan@email.com",
  "roles": ["admin"]
}
```

プロパティ名をキーとして受け取り、そのプロパティ名が存在する場合は関連する値を返すデコレータを定義してみよう。存在しない場合、ユーザーオブジェクトが作成されていない場合はundefinedとする。

```ts :user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const User = createParamDecorator(
  (data: string, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);
```

コントローラの`@User()`デコレータを通じて特定のプロパティにアクセスする方法は以下の通り。

```ts
@Get()
async findOne(@User('firstName') firstName: string) {
  console.log(`Hello ${firstName}`);
}
```

同じデコレータを使って、別のキーで別のプロパティにアクセス可能だ。`user`オブジェクトがdeepだったり複雑だったりする場合は、この記法によってリクエストハンドラの実装をより簡単に、より読みやすくできる。

>Hint  
>TypeScriptユーザへ。`createParamDecorator<T>()`はジェネリックだ。つまり、型の安全性を明示的に強制できる。例えば`createParamDecorator<string>((data, ctx) => ...)`等。あるいは、ファクトリー関数でパラメータの型を指定する事もできる。例えば`createParamDecorator((data: string, ctx) => ...)`等。ともに省略した場合、`data`の型は`any`となる。

## パイプで動かす
Nestはカスタムのparamデコレータを組み込みのデコレータ（`@Body()`、`@Param()`、`@Query()`等）と同じ方法で扱う。つまり、パイプはカスタムアノテーションされたパラメータ（ここでの例では`user`引数）に対しても実行される。さらに、カスタムデコレータに直接パイプを適用する事もできる。

```ts
@Get()
async findOne(
  @User(new ValidationPipe({ validateCustomDecorators: true }))
  user: UserEntity,
) {
  console.log(user);
}
```

>HINT  
>`validateCustomDecorators`はtrueにしなければならない。`ValidationPipe`は、デフォルトではカスタムデコレータでアノテーションされた引数を検証しない。

## デコレータの構成
Nestには複数のデコレータをまとめる為のヘルパーメソッドが用意されている。例えば、認証に関連する全てのデコレータを一つのデコレータにまとめたいとする。これは以下のコードで行える。

```ts :auth.decorator.ts 
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: Role[]) {
  return applyDecorators(
    SetMetadata('roles', roles),
    UseGuards(AuthGuard, RolesGuard),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' }),
  );
}
```

そして、この`@Auth()`カスタムデコレータを次のように使用する。

```ts
@Get('users')
@Auth('admin')
findAllUsers() {}
```

こうすると、単一の宣言で四つのデコレータを全て適用できる。

>Warning  
>`applyDecorators`機能を使っても、`@nestjs/swagger`パッケージの`@ApiHideProperty()`デコレータはまとめる事ができず、適切に動かない。