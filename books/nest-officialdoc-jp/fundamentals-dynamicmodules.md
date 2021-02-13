# 動的モジュール

[モジュールの章](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-modules)ではNestモジュールの基本をカバーした。動的モジュールも簡単に紹介した。この章では動的モジュールのテーマをより広く説明する。この章を終えるとモジュールとはなにか、どう、いつ使うかが十分に理解できるはずだ。

## イントロダクション
**Overview**部にあるほとんどのアプリケーションコード例は、通常のモジュールか静的モジュールを使用している。モジュールは協働する[プロバイダ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-providers)や[コントローラ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-controllers)などのコンポーネントのグループをアプリケーション全体の一単位として定義する。コンポーネントの実行コンテキストやスコープを提供する。例えばモジュール内で定義されたプロバイダはエクスポートしなくてもモジュールの他のメンバから見える。プロバイダをモジュール外で表示する必要がある場合、まずホストモジュールでエクスポートしてから、使用するモジュールにインポートする。

おなじみの例を見てみよう。

まず、`UsersService`を提供してエクスポートする`UsersModule`を定義する。`UsersModule`は`UsersService`のホストモジュールだ。

```ts
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

次に、`AuthModule`を定義して、`UsersModule`をインポートし、`UsersModule`内のエクスポートされたプロバイダを`AuthModule`内で利用できるようにする。

```ts
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';

@Module({
  imports: [UsersModule],
  providers: [AuthService],
  exports: [AuthService],
})
export class AuthModule {}
```

以下のコードによって、例えば`AuthModule`でホストされている`AuthService`の中で`UsersService`をインジェクションする事ができる。

```ts
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';

@Injectable()
export class AuthService {
  constructor(private usersService: UsersService) {}
  /*
    Implementation that makes use of this.usersService
  */
}
```

これを静的モジュールのバインディングと呼ぶ。Nestがモジュールを配線する為に必要なすべての情報は、すでにthe host and consuming modules（すみません、訳出できず）で宣言されている。このプロセスで何が起きているか紐解いてみよう。以下の順番で、Nestは`AuthModule`の中で`UsersService`を有効にしている。

1. `UsersModule`のインスタンスを作成し、`UsersModule`自身がconsumeする他のモジュールを遷移的にインポートし、依存関係を遷移的に解決（[customprovider](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-customproviders)を参照）
2. AuthModuleのインスタンスを作成し、`AuthModule`のコンポーネントで、（`AuthModule`で宣言されているかのように、）`usersModule`のエクスポートされたプロバイダを利用できるようにする。
3. `AuthService`に`UsersService`のインスタンスをインジェクションする。