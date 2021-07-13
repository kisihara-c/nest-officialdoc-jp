---
title: security-authorization
---

# 認可

認可とは、ユーザが何をできるかを決定するプロセスを指す。例えば管理者ユーザは投稿を作成・編集・削除する事ができる。管理者ではないユーザは投稿を読む事しかできない。

認可とは認証と直交・独立したものだ。しかし認可には認証メカニズムが必要となる。

認可を扱うには様々なアプローチや戦略があり、どのようなアプローチを取るかはそのプロジェクトの要件による。この章では様々な要求に対応できる認可のアプローチをいくつか紹介する。

## RBACの基本的な実装方法

ロールベースアクセスコントロール（RBAC）はロールと権限に基づいて定義されたポリシーニュートラルなアクセスコントロール・メカニズムだ。この項ではガードを使った非常に基本的なRBACメカニズムの実装方法を説明する。

まずシステム内のロールを表すenum `Role`を作成する。

```ts :role.enum.ts 
export enum Role {
  User = 'user',
  Admin = 'admin',
}
```

>HINT  
>より洗練されたシステムでは、ロールをデータベースに格納したり、外部の認証プロバイダからロールを取得したりする。

続けて`@Roles()`デコレータを作成する。このデコレータでは特定のリソースにアクセスする為に必要なロールを指定できる。

```ts :roles.decorator.ts 
import { SetMetadata } from '@nestjs/common';
import { Role } from '../enums/role.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
```

カスタムの`@Roles()`デコレータができたので、これを使って任意のルートハンドラを装飾できる。

```ts :roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
import { Role } from '../enums/role.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
```

最後に、`RolesGuard`クラスを作成し、現在のユーザに割り当てられたロールと、現在のルートで必要とされる実際かつ処理中のロールを比較する。ルートのロール（カスタムメタデータ）（訳注：role(s)という表記有り。単数の場合も複数の場合もあるという強調と思うが）にアクセスする為に、`Reflector`ヘルパークラスを使用する。`Reflector`ヘルパークラスはフレームワークによってすぐに提供される（`@nestsjs/core`パッケージから公開されている）。

```ts :roles.guard.ts 
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) {
      return true;
    }
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
```

>HINT  
>コンテキストに応じて`Reflector`を使用する方法の詳細については、「[Reflectionとメタデータ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-executioncontext#%E3%83%AA%E3%83%95%E3%83%AC%E3%82%AF%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%A8%E3%83%A1%E3%82%BF%E3%83%87%E3%83%BC%E3%82%BF)」の項目を参照の事。

>NOTICE  
>この例は"`basic`"と名付けた。ルートハンドラレベルでのロールの存在をチェックするだけだからだ。実際のアプリケーションでは複数の操作を行うエンドポイント/ハンドラがあり、それぞれの操作に特定の権限セットが必要になる場合がある。この場合ビジネスロジックのどこかでロールをチェックするメカニズムを提供しなければならない。パーミッションを特定のアクションに関連づける中心点（centralized place）がないのでメンテナンスがやや困難になる。

このサンプルでは`request.user`にユーザ・インスタンスと許可されたロール（`roles`プロパティの中）が含まれていると仮定している。貴方のアプリでは、カスタム**認証ガード**でこの関連付けを行う事になるだろう。詳細は[認証の章](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/security-authentication)にて。

この例が確実に動作する為には、`User`クラスが以下のようになっている必要がある。

```ts
class User {
  // ...他のプロパティ
  roles: Role[];
}
```

最後に、`RolesGuard`の登録を行おう。コントローラレベルかグローバルに登録する。

```ts
providers: [
  {
    provide: APP_GUARD,
    useClass: RolesGuard,
  },
],
```

十分な権限を持たないユーザがエンドポイントをリクエストすると、Nestは自動的に次のようなレスポンスを返す。

```ts
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

>HINT  
>別のエラーレスポンスを返したい場合は、booleanを返すかわりに独自の特定の例外を投げる必要がある。

## クレーム・ベースの認可

IDが作成されると、信頼できる当事者が発行した１つ以上のクレームが割り当てられる事がある。クレームはname-valueペアによって対象者が何をできるかを表すものだ。何であるかを表すものではない。

クレームベースの認証をNestで実装するには、上記のRBACの件と同じ手順を踏むことができるが、１つ大きな違いがある。特定のロールをチェックするのではなく、全てのユーザにはパーミッションのセットが割り当てられるという事だ。同様に各リソースやエンドポイントでどういったパーミッションが必要かを定義する。例えば、専用の`@Require()`デコレータを使用する。

```ts :cats.controller.ts
@Post()
@RequirePermissions(Permission.CREATE_CAT)
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

>HINT  
>上記の例において、`Permission`（RBACセクションで示した`Role`と同様のもの）は、システムで利用可能な全てのパーミッションを含むTypeScriptのenum型だ。

## CASLの導入

[CASL](https://casl.js.org/v5/en/)は、クライアントがアクセスを許可されるリソースを制限するisomorphicな認可ライブラリだ。段階的に導入できるように設計されており、シンプルなクレームベースの認可から、フル機能のサブジェクト・属性ベースの認可まで（between a simple claim based and fully featured subject and attribute based authorization）、簡単に拡張する事ができる。

まず、`@casl/ability`パッケージをインストールする。

```
$ npm i @casl/ability
```

>HINT  
>この例ではCASLを選択したが、好みやプロジェクトの必要性に応じて、`accesscontrol`や`acl`等他のライブラリを使うこともできる。

インストールが完了したら、CASLの仕組みを説明する為に、`User`と`Article`、２つのエンティティクラスを定義する。

```ts
class User {
  id: number;
  isAdmin: boolean;
}
```

`User`クラスはユニークなユーザ識別子である`id`と、ユーザが管理者権限を持っているかどうかを示す`isAdmin`、２つのプロパティで構成されている。

```ts
class Article {
  id: number;
  isPublished: boolean;
  authorId: string;
}
```

`Article`クラスには、`id`、`isPublished`、`authorId`という３つのプロパティがある。`id`は記事のユニークな識別子、`isPublished`は記事が公開済か、`authorId`は記事を書いたユーザのIDを表す。

ここでこのサンプルの要件を確認・精査しよう。

- 管理者は全てのエンティティを管理（CRUD）できる
- ユーザはすべてのものに対して読み取り専用のアクセス権を持つ
- ユーザは自分の記事を更新できる（`arcticle.authorId`===`userId`）
- すでに公開されている記事は削除できない（`article.isPublished` === `true`）

以上を念頭に置いて、まずユーザがエンティティに対して実行できるすべての可能なアクションを表す`Action` enumを作成する。

```ts
export enum Action {
  Manage = 'manage',
  Create = 'create',
  Read = 'read',
  Update = 'update',
  Delete = 'delete',
}
```

>NOTICE  
>`manage`はCASLの用語で、"あらゆる"操作を意味する。

CASLライブラリをカプセル化する為に、`CaslModule`と`CaslAbilityFactory`を生成してみよう。

```
$ nest g module casl
$ nest g class casl/casl-ability.factory
```

この状態で、`CaslAbilityFactory`の`createForUser()`メソッドを定義する。このメソッドは与えられたユーザの`Ability`オブジェクトを作成する。

```ts

type Subjects = InferSubjects<typeof Article | typeof User> | 'all';

export type AppAbility = Ability<[Action, Subjects]>;

@Injectable()
export class CaslAbilityFactory {
  createForUser(user: User) {
    const { can, cannot, build } = new AbilityBuilder<
      Ability<[Action, Subjects]>
    >(Ability as AbilityClass<AppAbility>);

    if (user.isAdmin) {
      can(Action.Manage, 'all'); // 全てに対する読み書きのアクセス
    } else {
      can(Action.Read, 'all'); // 全てに対する読み取り専用のアクセス
    }

    can(Action.Update, Article, { authorId: user.id });
    cannot(Action.Delete, Article, { isPublished: true });

    return build({
      // 詳細はこちら https://casl.js.org/v5/en/guide/subject-type-detection#use-classes-as-subject-types
      detectSubjectType: item => item.constructor as ExtractSubjectType<Subjects>
    });
  }
}
```

>NOTICE  
>`all`はCASLの用語で、"あらゆる対象"を意味する。

>HINT  
>`Ability`、`AbilityBuilder`、`AbilityClass`、`ExtractSubjectType`は`@casl/ability`パッケージからエクスポートされている。

>HINT  
>`detectSubjectType`オプションは、CASLにオブジェクトから対象の種類を取得する方法を理解させる為のものだ。詳しくは[CASLのドキュメント](https://casl.js.org/v5/en/guide/subject-type-detection#use-classes-as-subject-types)にて。

上の例では、`AbilityBuilder`クラスを使って`Ability`インスタンスを作成する。推測の通り`can`と`cannot`は同じ引数を取るが、`can`は指定された対象に対してアクションを許可し、`cannot`は禁止する。どちらも最大４つの引数を取る。これらの関数の詳細は[CASLのドキュメント](https://casl.js.org/v5/en/guide/subject-type-detection#use-classes-as-subject-types)にて。

最後にモジュール`CatsModule`定義の`providers`及び`exports`配列に`CaslAbilityFactory`を追加する事を忘れないようにする。

```ts
import { Module } from '@nestjs/common';
import { CaslAbilityFactory } from './casl-ability.factory';

@Module({
  providers: [CaslAbilityFactory],
  exports: [CaslAbilityFactory],
})
export class CaslModule {}
```

これにより、`CaslModule`がホストのコンテキストでインポートされていれば、標準的なコンストラクタインジェクションを使用して、`CaslAbilityFactory`を任意のクラスにインジェクションできる。

```ts
constructor(private caslAbilityFactory: CaslAbilityFactory) {}
```

そしてクラスの中でこうして使う。

```ts
const ability = this.caslAbilityFactory.createForUser(user);
if (ability.can(Action.Read, 'all')) {
  // "user"は全てに対して読み取りを行える
}
```

>HINT  
>`Ability`クラスの詳細については、[CASLの公式ドキュメント](https://casl.js.org/v5/en/guide/intro)参照の事。

例えばadminではないユーザがいるとする。この場合ユーザは記事を読む事はできるが、記事の新規作成、削除は禁止されているはずだ。

```ts
const user = new User();
user.isAdmin = false;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Read, Article); // true
ability.can(Action.Delete, Article); // false
ability.can(Action.Create, Article); // false
```

>HINT  
>`Ability`クラスと`AbilityBuilder`クラスには`can`メソッドと`cannot`メソッドがありますが、目的が異なり、受け付ける引数も若干異なる。

また、要件で指定したように、ユーザが記事を更新できるようにする必要がある。

```ts
const user = new User();
user.id = 1;

const article = new Article();
article.authorId = user.id;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Update, article); // true

article.authorId = 2;
ability.can(Action.Update, article); // false
```

ご覧のように、`Ability`インスタンスでは読みやすい方法でパーミッションを確認できる。同様に`AbilityBuilder`でも同様の方法でパーミッションを定義したり、様々な条件を指定できる。その他の例については公式ドキュメントを参照の事。

## 発展：`PoliciesGuard`の実装

このセクションでは、より洗練されたガードを構築する方法を説明する。そのガードとは、ユーザがメソッドレベルで設定された特定の認可ポリシーを満たしているかをチェックできるものだ（クラスレベルでの設定も可能）。この例では説明のためにCASLパッケージを使うが、使わなくても良い。前のセクションで作成した`CaslAbilityFactory`プロバイダを使用する。

最初に要件を具体的に説明しよう。目標はルートハンドラごとにポリシーチェックを指定できる仕組を提供する事だ。ここではオブジェクトと関数の両方をサポートする（シンプルなチェックと、関数的なコードを好む人向け）。

ポリシーハンドラのインターフェイスを定義する事から始めよう。

```ts
import { AppAbility } from '../casl/casl-ability.factory';

interface IPolicyHandler {
  handle(ability: AppAbility): boolean;
}

type PolicyHandlerCallback = (ability: AppAbility) => boolean;

export type PolicyHandler = IPolicyHandler | PolicyHandlerCallback;
```

前述の通り、ポリシー・ハンドラを定義する方法として、オブジェクト（`IPolicyHandler`インターフェイスを実装したクラスのインスタンス）と関数（`PolicyHandlerCallback`型を満たすもの）の2種類を用意した。

これが揃っていれば`@CheckPolicies()`デコレータを作成できる。このデコレータでは特定のリソースにアクセスする為に満たさなければならないポリシーを指定できる。

```ts
export const CHECK_POLICIES_KEY = 'check_policy';
export const CheckPolicies = (...handlers: PolicyHandler[]) =>
  SetMetadata(CHECK_POLICIES_KEY, handlers);
```

それではルートハンドラにバインドされたすべてのポリシーハンドラを抽出・実行する`PoliciesGuard`を作成しよう。

```ts
@Injectable()
export class PoliciesGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private caslAbilityFactory: CaslAbilityFactory,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const policyHandlers =
      this.reflector.get<PolicyHandler[]>(
        CHECK_POLICIES_KEY,
        context.getHandler(),
      ) || [];

    const { user } = context.switchToHttp().getRequest();
    const ability = this.caslAbilityFactory.createForUser(user);

    return policyHandlers.every((handler) =>
      this.execPolicyHandler(handler, ability),
    );
  }

  private execPolicyHandler(handler: PolicyHandler, ability: AppAbility) {
    if (typeof handler === 'function') {
      return handler(ability);
    }
    return handler.handle(ability);
  }
}
```

>HINT  
>このサンプルでは、`request.user`にユーザのインスタンスが含まれることを仮定している。アプリケーションでは、カスタム認証ガードでこの関連付けを行う事になるだろう。詳細は[認証のページ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/security-authentication)にて。

このサンプルを噛み砕いてみよう。`policyhandlers`は`@CheckPolicies()`デコレータによってメソッドに割り当てられたハンドラの配列だ。次に、`CaslAbility#create`メソッドを使用して、`Ability`オブジェクトを構築し、ユーザが特定のアクションを実行するのに十分な権限を持っているかを検証する。このオブジェクトは、関数あるいはクラスのインスタンス（`IPolicyHandler`を実装）であるポリシーハンドラに渡すもので、booleanを返す`handle()`メソッドを提供している。最後に、`Array#every`メソッドを使って、全てのハンドラが真の値を返すようにする。

最後にこのガードをテストする為に、任意のルートハンドラにバインドし、以下のようにインラインのポリシーハンドラを登録する（関数によるアプローチ（functional approach））。

```ts
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies((ability: AppAbility) => ability.can(Action.Read, Article))
findAll() {
  return this.articlesService.findAll();
}
```

別の方法として、`IPolicyHandler`を実装したクラスを定義する事もできる。

```ts
export class ReadArticlePolicyHandler implements IPolicyHandler {
  handle(ability: AppAbility) {
    return ability.can(Action.Read, Article);
  }
}
```

そしてこう使う。

```ts
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies(new ReadArticlePolicyHandler())
findAll() {
  return this.articlesService.findAll();
}
```

>NOTE  
>`new`キーワードでそのバでポリシーハンドラをインスタンス化する必要がある為、`CreateArticlePolicyHandler`クラスは依存性インジェクションを使用できない。この事は`ModuleRef#get`メソッドで対処できる（詳細は[こちら](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-modulereference)）。基本的には`@CheckPolicies()`デコレータを通して関数やインスタンスを登録する代わりに`Type<IPolicyHandler>`を渡せるようにしなければならない。ガードの中では、型参照`moduleRef.get(YOUR_HANDLE_TYPE)`をツアkってインスタンスを取得したり、`ModuleRef#create`メソッドを使って動的にインスタンスを生成できる。