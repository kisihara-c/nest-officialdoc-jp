---
title: security-authentication
---

# 認証

認証はほとんどのアプリで**不可欠**だ。認証を扱うアプローチ・戦略は幅広く、要件による。この章では様々な要件に対応できる認証のアプローチをいくつか紹介する。

[Passport](https://github.com/jaredhanson/passport)は最も人気のあるnode.jsの認証ライブラリで、多くの製品で使われている。このライブラリをNestアプリケーションに統合するには、`@nestjs/passport`モジュールを使うと簡単だ。抽象的には（At a high level）、Passportは次のようなステップを実行する。

- ユーザの「認証情報」（ユーザ名/パスワード、JSON Web Token([JWT](https://jwt.io/))、IDプロバイダからのIDトーク等）を確認してユーザを認証する。
- 認証された状態を管理する（JWTなどのポータブルトークンを発行したり、[Expressセッション](https://github.com/expressjs/session)を作成したりする）
- 認証されたユーザに関する情報を`Request`オブジェクトに添付し、ルートハンドラでさらに使用する。

Passportには、認証メカニズムを実装する様々な[ストラテジー](http://www.passportjs.org/)の豊富なエコシステムがある。コンセプトはシンプルだが、選択できるPassportストラテジーのセットは多様だ。Passportはこれらの多様なステップを標準的なパターンに抽象化し、`@nestjs/passport`モジュールはこのパターンをお馴染みのNestコードに纏めて標準化している。

本章ではこれらの強力で柔軟なモジュールを使用して、RESTful APIサーバーの完全なエンドツーエンドの認証ソリューションを実装する。ここで説明する概念を使って、任意のPassportストラテジを実装し、認証スキームをカスタマイズする事ができる。この章の手順に沿えば完全なサンプルを構築する事ができる。サンプルアプリのあるリポジトリは[こちら](https://github.com/nestjs/nest/tree/master/sample/19-auth-jwt)。

## 認証の要件

要件を具体化しよう。今回のユースケースでは、クライアントはまずユーザー名とパスワードで認証を行う。認証されると、サーバーはJWTを発行する。このJWTは認証を証明する為に、後続のリクエストの[認証ヘッダ内のBearerトークン](https://datatracker.ietf.org/doc/html/rfc6750)として送信する事ができる。また、有効なJWTを含むRequestのみにアクセス可能な保護されたルートを作成する。

まず、最初の要件であるユーザの認証から始める。次に、JWTを発行して、それを拡張する。最後に、リクエストに有効なJWTが含まれているかどうかをチェックする保護されたルートを作成する。

必要なパッケージをインストールしよう。Passportにはユーザー名とパスワードによる認証メカニズムを実装した[passport-local](https://github.com/jaredhanson/passport-local)というストラテジがあり、今回使えそうだ。

```
$ npm install --save @nestjs/passport passport passport-local
$ npm install --save-dev @types/passport-local
```

>NOTICE  
>どのようなPassportストラテジを選択しても、`@nestjs/passport`及び`passport`パッケージが必要となる。次に構築しようとしている特定の認証ストラテジを実装する固有のパッケージ（`passport-jwt`や`passport-local`等）インストールする必要がある。また、上記の`@types/passport-local`のように、任意のPassportストラテジの型定義をインストールする事で、TypeScriptコードを書く際の補助となる。

## Passportストラテジの実装

これで、認証機能を実装する準備が整った。まず使用するプロセスの概要を説明する。Passportはそれ自体が小さなフレームワークと考えると通りが良い。このフレームワークの優れた点は、認証プロセスをいくつかの基本的なステップに抽象化し、実装するストラテジに応じてカスタマイズできる事だ。フレームワークは拡張パラメータ（プレーンJSON）とコールバック関数の形でカスタムコードを提供する事で構成される。Passportは適切なタイミングでそれを呼び出す事ができる。`@nestjs/passport`モジュールは、このフレームワークをNestスタイルのパッケージで包み、Nestアプリケーションに簡単に統合できるようにしている。以下では`@nestjs/passport`を使用するが、まずは**生のPassport**の仕組みを考えてみよう。

生のPassportでは、以下の２つを提供する事でストラテジを設定する。

- ストラテジに固有のオプションのセット。例えば、JWTストラテジでトークンに署名する為のシークレットを提供できる。
- 「検証コールバック」：ユーザストア（ユーザアカウントを管理する場所）とどのようにやり取りするかをPassportに伝える場所だ。ここではユーザが存在するかどうか（または新しいユーザを作成するかどうか）、及びその資格情報が有効であるかどうかを検証する。Passportライブラリは、このコールバックが、検証が成功した場合は完全なユーザを返し、失敗した場合はnullを返すと仮定して動く。失敗とは、ユーザが見つからないか、passport-localの場合はパスワードが一致しないことと定義される。

`@nestjs/passport`では、PassportStrategyクラスを拡張してPassportストラテジを構成する。サブクラスの`super()`メソッドを呼び出しオプションオブジェクトを渡すことで、ストラテジのオプション（上記項目1）を渡せる。`validate()`メソッドをサブクラスに実装する事で、検証コールバック（上記項目2）を提供できる。

まずは`AuthMobile`を生成し、その中に`AuthService`を生成する事から始めよう。

```
$ nest g module auth
$ nest g service auth
```

これらの生成されたファイルの内容を以下のように置き換える。今回のサンプルアプリでは、`UsersService`はシンプルにする。メモリ上にハードコードされたユーザのリストと、ユーザ名でユーザを検索するfindメソッドを保持させる。実際にアプリではここでユーザモデルと永続化レイヤーを構築し、好みのライブラリを使う（TypeORM、Sequelize、Mongoose等）。

```ts :users/users.service.ts 
import { Injectable } from '@nestjs/common';

// ここはユーザーエンティティを表す実際のクラス/インターフェイスにしてほしい
export type User = any;

@Injectable()
export class UsersService {
  private readonly users = [
    {
      userId: 1,
      username: 'john',
      password: 'changeme',
    },
    {
      userId: 2,
      username: 'maria',
      password: 'guess',
    },
  ];

  async findOne(username: string): Promise<User | undefined> {
    return this.users.find(user => user.username === username);
  }
}
```

`UserModule`で必要な処理は、`@Module`デコレータの`exports`配列に`UsersService`を追加して、このモジュールの外から見えるようにする事だけだ（すぐに`AuthService`で使う）。

```ts :users/users.module.ts 
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

`AuthService`はユーザを取得してパスワードを検証する役割を持っている。この目的のために`validateUser()`メソッドを作成している。以下のコードでは、ES6の便利な`spread`演算子を使って、`user`オブジェクトから`password`プロパティを取り除いて返している。後でpassport localストラテジから`validateUser()`メソッドを呼び出してみよう。

```ts :auth/auth.service.ts 
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}

  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }
}
```

>WARNING  
>もちろん実際のアプリケーションではパスワードを平文で保存する事はない。[bcrypt](https://github.com/kelektiv/node.bcrypt.js#readme)などのライブラリを使い、ソルト化済の一方向性ハッシュアルゴリズムを使用する。この方法ではハッシュ化されたパスワードのみを保存し、保存されたパスワードとハッシュ化された入力パスワードを比較する為、ユーザのパスワードを平文で保存・公開する事はない。サンプルアプリでは、シンプル性の為にこの絶対的義務に違反している。***実際のアプリではしないでほしい！***

次に、`AuthModule`を更新して`UsersModule`をインポートする。

```ts :auth/auth.module.ts 
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
})
export class AuthModule {}
```

## Passport localを実装する

ではPassportの**ローカル認証ストラテジ**を実装しよう。`local.strategy.ts`ファイルを`auth`フォルダに入れ、以下のコードを追加する。

```ts :auth/local.strategy.ts 
import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { AuthService } from './auth.service';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super();
  }

  async validate(username: string, password: string): Promise<any> {
    const user = await this.authService.validateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

全てのPassportストラテジは前述のレシピに従っている。passport-localの仕様例では設定オプションがないため、コンストラクタでは単に`super()`を呼び出し、オプションオブジェクトを指定しない。

>HINT  
>パスポートストラテジの動作をカスタマイズする為に、`super()`への呼び出しでオプションオブジェクトを渡す事ができる。この例ではpassport-localストラテジはデフォルトでrequestの本文に`username`、`password`プロパティの存在を予想している。異なるプロパティ名を指定するには、オプションオブジェクトを渡す必要がある（例：`super({ usernameField: 'email' })`）。詳細は[Passportのドキュメント](http://www.passportjs.org/docs/configure/)にて。

`validate()`メソッドも実装済みだ。各ストラテジについて、Passportはストラテジ固有の適切なパラメータセットを使用して、バリデーション関数（`@nestjs/passport`の`validate()`メソッドで実装）を呼び出す。local-strategyの場合、Passportは`validate(username: string, password:string): any`のシグネチャを持つ`validate()`メソッドを待機している。


ほとんどの検証作業は`AuthService`で（`UsersService`の助けを借りて）行われるので、このメソッドは非常にシンプルだ。**どの**Passportストラテジの`validate()`メソッドも似たようなパターンで、資格の表現方法の詳細が異なるだけだ。ユーザが見つかり、認証情報が有効であればユーザが返され、Passportがタスク（例：`Request`オブジェクトの`user`プロパティの作成）を完了できるようになり、リクエスト処理のパイプラインが続くようになる。見つからない場合は例外を投げて例外レイヤーに処理させる。

一般的には、各ストラテジの`validate()`メソッドの唯一の大きな違いは、ユーザの存在・有効性の判断の方法だ。例えばJWTストラテジでは、要件に応じて、デコードされたトークンに含まれる`userId`が、ユーザデータベースのレコードや無効化されたトークンのリストと一致するかどうかを評価する。こういった事で、サブクラス化してストラテジ固有の検証を実装する仕組みは、一貫性があってエレガントで拡張性が高い。

さっき定義したPassportの機能を使用するために、`AuthModule`を設定する必要がある。`auth.module.ts`を以下のようにアップデートする。

```ts :auth/auth.module.ts 
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { LocalStrategy } from './local.strategy';

@Module({
  imports: [UsersModule, PassportModule],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
```

## 組み込みパスポートガード

[ガード](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-guards)の章では、リクエストがルートハンドラで処理されるかどうか、ガードの主な機能として判断できる事を説明した。勿論そのまま使えるが、`@nestjs/passport`モジュールを使うにあたり、最初は少し戸惑うような新機能を紹介する。認証の観点から、アプリに２つの状態があるとしよう。

- ユーザ/クライアントがログインして**いない**状態（認証なし）
- ユーザ/クライアントがログインして**いる**状態（認証された）

前者のケースでは２つの異なる機能を実行する必要がある。

- 認証されていないユーザがアクセスできるルートを制限する（すなわち、制限されたルートへのアクセスを拒否する）。その為にお馴染みのガードを使う。保護されたルートにガードを配置する。想像通り、このガードを使って有効なJWTの存在をチェックするので、JWTの発行に成功した後、このガードについて後で確認する。
- 認証されていないユーザがログインしようとしたときに**認証ステップ**を開始する。有効なユーザにJWTを**発行**するステップだ。少し考えると、認証の為にユーザ名/パスワードの認証情報を`POST`する必要がある事がわかるので、`POST/auth/login`ルートを設定する。ここで疑問が生じる。このルートでは、具体的にどのようにしてpassport-localストラテジを呼び出すんだろう？

答えはシンプル、別の少し異なるタイプのガードを使う事だ。`@nestjs/passport`モジュールは、これを行う組み込みのガードを提供している。このガードはPassportストラテジを起動し、上述のステップ（認証情報の取得、`verify`関数の実行、ユーザプロパティの作成等）を開始する。

なお後者のケース（ログイン済ユーザ）については、シンプルにログインユーザの為の保護されたルートにアクセスできるガードに頼ればいい。説明済の標準タイプのガードだ。（訳出困難につき原文：The second case enumerated above (logged in user) simply relies on the standard type of Guard we already discussed to enable access to protected routes for logged in users.）

## ログインルート

ストラテジができたので、今度はシンプルな`/auth/login`ルートを実装し、組み込みのガードを適用してpassport-localフローを準備してみよう。

`app.controller.ts`ファイルを開き、内容を以下のように置き換える。

```ts
import { Controller, Request, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller()
export class AppController {
  @UseGuards(AuthGuard('local'))
  @Post('auth/login')
  async login(@Request() req) {
    return req.user;
  }
}
```

`@UseGuards(AuthGuard('local'))`では、passport-localストラテジを拡張した際に`@nestjs/passport`が**自動で用意**してくれる`AuthGuard`を利用している。精査してみよう。Passport localストラテジには、デフォルトでは`'local'`という名前がついている。その名前を`@UseGurads()`デコレータで参照し、`passport-local`パッケージで提供されるコードと関連付ける。これは、アプリ内に複数のPassportストラテジがある場合に、どのストラテジを呼び出すかを明白にする為のものだ（それぞれのストラテジがそれぞれの`AuthGuard`を提供する可能性がある）。今の所ストラテジは１つしかないが、まもなく２つ目を追加する予定なので必要となる。

ルートをテストする為に、今回は`/auth/login`ルートが単にユーザを帰すようにする。これはPassportのもう一つの機能で、Passportは`validate()`メソッドから返された値に基づいて`user`オブジェクトを自動的に作成し、それを`req.user`として`Request`オブジェクトに割り当てる。後でこれを、代わりにJWTを作成して返すコードに置き換えてみる。

これらはAPIルートだから、例の[cURL](https://curl.haxx.se)を使ってテストしよう。`UsersSerivce`にハードコードされている`user`オブジェクトを使ってテストできる。

```
$ # POST to /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # result -> {"userId":1,"username":"john"}
```

問題なく動くが、`AuthGuard()`にストラテジ名を渡すとソースコードに魔法の文字列が混入してしまう。代わりに独自のクラスを作成しよう。

```ts :auth/local-auth.guard.ts 
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class LocalAuthGuard extends AuthGuard('local') {}
```

これで、`/auth/login`ルートハンドラを更新して、代わりに`LocalAuthGuard`を使える。

```ts
@UseGuards(LocalAuthGuard)
@Post('auth/login')
async login(@Request() req) {
  return req.user;
}
```

## JWTの機能

認証システムのJWT関係に進む準備ができた。要件を確認し、改善していこう。

- ユーザがユーザ名/パスワードで認証を行う事と、続けて保護済APIエンドポイントを呼び出す際使うJWTの返却を許可しよう。この要件は順調に実装できており、残り作業としてJWTを発行するコードを書く必要がある。
- bearerトークンとして有効なJWTの存在に基づいて保護されたAPIルートを作成しよう。

JWTの要件をサポートするために、さらにいくつかのパッケージをインストールする必要がある。

```
$ npm install --save @nestjs/jwt passport-jwt
$ npm install --save-dev @types/passport-jwt
```

`nestjs/jwt`パッケージ（[詳細](https://github.com/nestjs/jwt)）は、JWTの操作を支援するユーティリティパッケージだ。`passport-jwt`パッケージはJWTストラテジを実装するPassportパッケージで、`@types/passport-jwt`パッケージはTypeScriptの型定義を提供する。

`POST`の`/auth/login`リクエストがどのように処理されるかを詳しく見てみよう。我々はpassport-localストラテジで提供される組み込みの`AuthGuard`を使ってルートを装飾しているところだ。これは次を意味する。

- ルートハンドラはユーザが**認証された場合にのみ呼び出される**。
- `req`パラメータには`user`のプロパティが含まれる（passportがpassport-local認証フローで生成する）

これを念頭に置いて最終的に本物のJWTを生成し、このルートで返す事ができる。サービスをきれいにモジュール化する為、`authSerrvice`でJWTの生成を処理する。`auth`フォルダにある`auth.service.ts`ファイルを開き、`login()`メソッドを追加し、図のように`JwtService`をインポートする。

```ts　:auth/auth.service.ts 
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService
  ) {}

  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }

  async login(user: any) {
    const payload = { username: user.username, sub: user.userId };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}
```

ここでは、`@nestjs/jwt`ライブラリを使用している。このライブラリには`user`オブジェクトのプロパティのサブセットからJWTを生成する`sign()`関数が用意されており、使うと単一の`access_token`を持つ単純なオブジェクトを返す。注意：JWTの標準に合わせるため、`userId`の値を保持するプロパティ名として`sub`を選択している。JwtServiceプロバイダの`AuthService`へのインジェクションを忘れないこと。

次に、`AuthModule`を更新して新しい依存関係をインポートして、`JwtModule`を設定する。

まず、`auth`フォルダに`constants.ts`を作成し、以下のコードを追加する。

```ts :auth/constants.ts 
export const jwtConstants = {
  secret: 'secretKey',
};
```

これを使って、JWTの署名と検証のステップの間で鍵を共有する。

>WARNING  
>***この鍵を公開するべきではない***。ここではコードが何をしているかを明確にする為公開しているが、実運用システムではsecrets valut、環境変数、設定サービスなどの適切な手段を用いて***鍵を保護しなければならない***。

次に、`auth`フォルダにある`auth.module.ts`を開き、次のように更新する。

```ts :auth/auth.module.ts 
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy],
  exports: [AuthService, JwtModule],
})
export class AuthModule {}
```

`register()`に設定オブジェクトを渡して`JwtModule`を設定する。Nestの`JwtModule`の詳細は[こちら](https://github.com/nestjs/jwt/blob/master/README.md)、使用可能な設定オプションの詳細は[こちら](https://github.com/auth0/node-jsonwebtoken#usage)。

これで`/auth/login`ルートがJWTを返すように更新できた。

```ts :app.controller.ts 
import { Controller, Request, Post, UseGuards } from '@nestjs/common';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  async login(@Request() req) {
    return this.authService.login(req.user);
  }
}
```

進めよう。再びcURLを使ってルートをテストする。`UsersService`にハードコードされているユーザーオブジェクトを使ってテストできる。

```
$ # POST to /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # result -> {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
$ # Note: above JWT truncated
```

## Passport JWTの実装

これで最後の要件である、リクエストに有効なJWTを要求する機能（エンドポイントの保護）に取り組める。Passportはここでも役に立つ。PassportはJSON Web TokensでRESTFulなエンドポイントを保護するための[passport-jwt](https://github.com/mikenicholson/passport-jwt)ストラテジを提供する。まず`auth`フォルダ内に`jwt.strategy.ts`というファイルを作成し、以下のコードを追加する。

```ts :auth/jwt.strategy.ts
import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { jwtConstants } from './constants';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: jwtConstants.secret,
    });
  }

  async validate(payload: any) {
    return { userId: payload.sub, username: payload.username };
  }
}
```

`JwtStrategy`は、すべてのPassportストラテジについての規格に従っている。このストラテジは初期化が必要なので、`super()`にオプションオブジェクトを渡し初期化する。利用可能なオプションの詳細については[こちら](https://github.com/mikenicholson/passport-jwt#configure-strategy)。今回は以下を使う。

- `jwtFromRequest`：`Request`からJWTを抽出する方法を指定する。ここではAPIリクエストのAuthorizationヘッダにbearerトークンを供給する標準的な方法を使用する。他のオプションについては[こちら](https://github.com/mikenicholson/passport-jwt#extracting-the-jwt-from-the-request)
- `ignoreExpiration`：デフォルトの`false`を明示的に選択しておく。これはJWTが期限切れになっていないことを確認する責任をPassportモジュールに与えるものだ。つまり、我々のルートに期限切れのJWTが提供された場合、リクエストは拒否され、`401 Unauthorized`レスポンスが送信される事になる。Passportはこれを自動的に処理してくれて便利だ。
- `secretOrKey`：トークンに署名するためのシンメトリックなsecretを提供する、便利なオプションを使用している。本番アプリケーションではPEMエンコードされた公開鍵など、他の選択肢のほうが適切な場合がある（詳しくは[こちら](https://github.com/mikenicholson/passport-jwt#extracting-the-jwt-from-the-request)）。いずれにせよ、先述の通りこの**secretを公開しないでほしい**。

`validate()`メソッドについては少し議論に値する。jwt-strategyにおいて、PassportはまずJWTの署名を検証し、JSONをデコードする。次にデコードされたJSONを単一のパラメータとして渡して`validate()`メソッドを呼び出す。JWTの署名の仕組に基づき、有効なユーザに発行した署名済みの**有効なトークンを受け取っている事が保証される**。

最終的な結果として、`validate()`コールバックへの応答は簡単なものとなる。`userId`と`username`プロパティを含むオブジェクトを返すだけだ。また思い出してほしいのは、Passportは`validate()`メソッドの戻り値に基づいて`user`オブジェクトを構築して、`Request`オブジェクトのプロパティとして添付する事だ。

このアプローチでは他のビジネスロジックをプロセスにインジェクションする余地（いわゆる「フック」）がある事も言えるだろう。例えば`validate()`メソッドの中でデータベースの検索を行い、ユーザに関するより多くの情報を抽出する事で、より充実したユーザオブジェクトを`Request`で利用できる。また、このメソッドではトークンのさらなる検証も行える。たとえば失効したトークンのリストから`userId`を検索して、トークンを失効させられる。ここでサンプルコードに実装したモデルは高速な「ステートレスJWT」モデルで、各APIコールは有効なJWTの存在に基づいて即座に認証される。要求者に関する僅かな情報（そのuserIdおよびusername）は`Request`パイプラインで利用可能だ。

新しい`JwtStrategy`を`AuthModule`のプロバイダとして追加する。

```ts :auth/auth.module.ts 
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { LocalStrategy } from './local.strategy';
import { JwtStrategy } from './jwt.strategy';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '60s' },
    }),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

JWTへの署名時使ったものと同じsecretをインポートする事で、Passportによって実行される検証フェーズと、AuthServiceで実行される署名フェーズとが共通のsecretを使用する事を保証する。

最後に、組み込みの`AuthGuard`を拡張した`JwtAuthGuard`クラスを定義する。

```ts :auth/jwt-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

## 保護されたルートとJWTストラテジガードの実装

これで、保護されたルートとその関連ガードを実装できる。

`app.controller.ts`ファイルを開いて以下のように更新する。

```ts :app.controller.ts 
import { Controller, Get, Request, Post, UseGuards } from '@nestjs/common';
import { JwtAuthGuard } from './auth/jwt-auth.guard';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { AuthService } from './auth/auth.service';

@Controller()
export class AppController {
  constructor(private authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  async login(@Request() req) {
    return this.authService.login(req.user);
  }

  @UseGuards(JwtAuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
}
```

今回も、`@nestjs/passport`モジュールがpassport-jwtモジュールの設定時に自動で用意した`AuthGuard`を適用している。このガードはデフォルトの名前`jwt`で参照される。`GET/profile`ルートがヒットすると、`Guard`は自動的に`passport-jwt`カスタム構成ロジックを起動し、JWTを検証して、`user`プロパティを`Request`オブジェクトに割り当てる。

アプリの実行を確認し、cURLを使用してルートをテストしよう。

```
$ # GET /profile
$ curl http://localhost:3000/profile
$ # result -> {"statusCode":401,"error":"Unauthorized"}

$ # POST /auth/login
$ curl -X POST http://localhost:3000/auth/login -d '{"username": "john", "password": "changeme"}' -H "Content-Type: application/json"
$ # result -> {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm... }

$ # GET /profile using access_token returned from previous step as bearer code
$ curl http://localhost:3000/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2Vybm..."
$ # result -> {"userId":1,"username":"john"}
```

`AuthModule`では、JWTの有効期限を60秒と設定したことに注意。この文章ではトークンの有効期限やリフレッシュの詳細を扱っていないが、JWTとpassport-jwtストラテジの重要な特性を示すためにこう設定した。認証後60秒待ってから`GET/profile`リクエストを試みると、`401 Unauthorized`レスポンスが返ってくる。これはPassportがJWTの有効期限を自動的にチェックする為で、アプリケーションで有効期限をチェックする手間の削減となる。

これで、JWT認証の実装が完了した。JavaScriptクライアント（Angular/React/Vue等）やその他のJavaScriptアプリは、我々のAPIサーバーと安全に認証・通信できるようになった。

## サンプル
以上のコードの完全版は[こちら](https://github.com/nestjs/nest/tree/master/sample/19-auth-jwt)

## ガードの拡張

ほとんどの場合デフォルトの`AuthGuard`を使えば十分だが、デフォルトのエラー処理や認証ロジックを単純に拡張したい場合もあるだろう。組み込みクラスを拡張し、サブクラスでメソッドをオーバーライドする事ができる。

```ts
import {
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  canActivate(context: ExecutionContext) {
    // カスタム認証ロジックを追加しよう
    // 例えば、セッションを作る為にsuper.logIn(request) を呼ぶ等
    return super.canActivate(context);
  }

  handleRequest(err, user, info) {
    // "info"引数か"err"引数を元に例外を投げる事ができる
    if (err || !user) {
      throw err || new UnauthorizedException();
    }
    return user;
  }
}
```

デフォルトのエラー処理と認証ロジックの拡張に加えて、認証がストラテジのチェーンを経るようにできる。最初に成功、リダイレクト、またはエラーになったストラテジがチェーンを止める。認証に失敗した場合は、各ストラテジを実行し、全てのストラテジが失敗した場合に最終的に失敗となる。

```ts
export class JwtAuthGuard extends AuthGuard(['strategy_jwt_1', 'strategy_jwt_2', '...']) { ... }
```

## グローバルで認証を有効にする

大半のエンドポイントがデフォルトで保護されている場合は、認証ガードを[グローバルガード](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-guards#%E3%83%90%E3%82%A4%E3%83%B3%E3%83%89%E3%81%99%E3%82%8B%E3%82%AC%E3%83%BC%E3%83%89)として登録し、各コントローラで`@UserGuards()`デコレータを使用する代わりに、どのルートを公開するかを単純に指定する事ができる。

まず`JwtAuthGuard`をグローバルガードとして登録しよう。どんなモジュールでも以下の手順は可能だ。

```ts
providers: [
  {
    provide: APP_GUARD,
    useClass: JwtAuthGuard,
  },
],
```

これで、Nestは自動的に`JwtAuthGuard`を全てのエンドポイントにバインドする。

次に、ルートをパブリックとして宣言する為のメカニズムを提供する必要がある。そのためには、`SetMetadata`デコレータのファクトリー関数を使って、カスタムデコレータを作成する。

```ts
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

上のファイルでは、２つの定数をエクスポートした。ひとつはメタデータキー`IS_PUBLIC_KEY`で、もうひとつは新しいデコレータ`Public`だ（`SkipAuth`や`AllowAnon`等好きな名前に変えられる）。

これでカスタムの`@Public()`デコレータができたので、次のように任意のメソッドをデコレーションできる。

```ts
@Public()
@Get()
findAll() {
  return [];
}
```

最後に`"isPublic"`メタデータが見つかったときに`JwtAuthGuard`が`true`を返すようにする必要がある。そのためには`Reflector`クラスを使う（詳細は[こちら](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-guards#%E5%85%A8%E3%81%A6%E3%81%BE%E3%81%A8%E3%82%81%E3%81%A6)）。

```ts
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      return true;
    }
    return super.canActivate(context);
  }
}
```

## リクエストスコープストラテジ

PassportのAPIは、ライブラリのグローバルなインスタンスにストラテジを登録する事を基本としている。その為、ストラテジはリクエストに依存するオプションを持ったり、リクエストごとに動的にインスタンス化されるようには設計されていない（リクエストスコーププロバイダについては[こちら](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-injectionscopes)）。リクエストスコープストラテジを設定した場合、ストラテジは特定のルートに結びついていないので、Nestはストラテジをインスタンス化しない。リクエストごとにどのリクエストスコープストラテジを実行すべきか決定する物理的な方法はない。

しかし、ストラテジ内でリクエストスコーププロバイダを動的に解決する方法はある。[モジュール参照](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-modulereference)機能を使う。

まず`local.strategy.ts`ファイルを開き、通常の方法で`ModuleRef`をインジェクションする。

```ts
constructor(private moduleRef: ModuleRef) {
  super({
    passReqToCallback: true,
  });
}
```

>HINT  
>`ModuleRef`クラスは`@nestjs/core`パッケージからインポートされている。

上記のように`passreqToCallback`設定プロパティを必ず`true`にする事。

次のステップでは、新しいコンテキスト識別子を生成するのではなく、現在のコンテキスト識別子を取得する為にリクエストインスタンスを使用する（リクエストコンテキストの[詳細](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-modulereference#%E7%8F%BE%E5%9C%A8%E3%81%AE%E3%82%B5%E3%83%96%E3%83%84%E3%83%AA%E3%83%BC%E3%81%AE%E5%8F%96%E5%BE%97)）。

ここで、`LocalStrategy`クラスの`validate()`メソッド内で、`ContextIdFactroy`クラスの`getByRequest()`メソッドを使用して、リクエストオブジェクトに基づいてコンテキストIDを作成し、これを`resolove()`に渡す。

```ts
async validate(
  request: Request,
  username: string,
  password: string,
) {
  const contextId = ContextIdFactory.getByRequest(request);
  // "AuthService"はリクエストスコーププロバイダ
  const authService = await this.moduleRef.resolve(AuthService, contextId);
  ...
}
```

上記の例では、`resolove()`メソッドは`AuthService`プロバイダのリクエストスコープインスタンスを非同期的に返す（`AuthService`がリクエストスコープ化されている事を想定している）。

## パスポートのカスタマイズ

`register()`メソッドを使えば、標準的なPassportカスタマイズオプションを同じ様に渡す事ができる。利用可能なオプションは実装されているストラテジによって異なる。例：

```ts
PassportModule.register({ session: true });
```

また、ストラテジのコンストラクタでオプションオブジェクトを渡して、ストラテジを設定する事もできる。ローカルストラテジに対して、例えば以下のように渡せる。

```ts
constructor(private authService: AuthService) {
  super({
    usernameField: 'email',
    passwordField: 'password',
  });
}
```

プロパティ名については[Passport公式サイト](http://www.passportjs.org/docs/oauth/)を参照の事。

## 名前付きストラテジ

ストラテジを実装する際、`PassportStrategy`関数に第２引数を渡す事でストラテジの名前を指定できる。渡さなければ各ストラテジの名前はデフォルトのままとなる（例：jwt-strategy→'jwt'）。

```ts
export class JwtStrategy extends PassportStrategy(Strategy, 'myjwt')
```

そして`@UseGuards(AuthGuard('myjwt'))`のようなデコレータでこの名前を参照する。

## GraphQL

GraphQLで`AuthGuard`を使う為には、組み込みの`AuthGuard`クラスを拡張し、`getRequest()`メソッドをオーバーライドする。

```ts
@Injectable()
export class GqlAuthGuard extends AuthGuard('jwt') {
  getRequest(context: ExecutionContext) {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req;
  }
}
```

graphqlリゾルバで現在の認証済みユーザを取得するには、`@CurrentUser()`デコレータを定義する。

```ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';

export const CurrentUser = createParamDecorator(
  (data: unknown, context: ExecutionContext) => {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req.user;
  },
);
```

上記のデコレータをリゾルバで使う為には、必ずクエリやミューテーションのパラメータとして含めるようにする。

```ts
@Query(returns => User)
@UseGuards(GqlAuthGuard)
whoAmI(@CurrentUser() user: User) {
  return this.usersService.findById(user.id);
}
```