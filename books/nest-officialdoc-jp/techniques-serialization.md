---
title: techniques-serialization
---

# シリアライゼーション

シリアライゼーションとは、ネットワークレスポンスでオブジェクトが返される前に行われるプロセスだ。これは、クライアントに返されるデータを変換し、サニタイズするためのルールを提供するのに適した場所だ。例えばパスワードのような機密データは、常にレスポンスから除外する必要がある。また、特定のプロパティについては、エンティティのプロパティのサブセットのみを送信するなど、追加の変換が必要な場合がある。手作業では退屈でエラーが発生しやすいし、すべてのケースがカバーされているかどうかわからない。

## 概要

Nestでは、シリアライゼーションを簡単な方法で実行できる組み込み機能を提供している。インターセプター`ClassSerializerInterceptor`は、強力な[class-transformer](https://github.com/typestack/class-transformer)パッケージを使用して、オブジェクトを変換する為の宣言的かつ拡張可能な方法を提供する。このインターセプターが行う基本的な操作は、メソッドハンドラから返された値を受け取り、class-transformerの`classToPlain()`関数を適用する事です。この際、以下のようにclass-transformerのデコレータで表現されたルールを、エンティティ/DTOクラスに適用する事ができる。

## プロパティを除外する

ユーザーエンティティからパスワードのプロパティを自動的に除外したいとしよう。以下のようにエンティティをアノテーションする。

```ts
import { Exclude } from 'class-transformer';

export class UserEntity {
  id: number;
  firstName: string;
  lastName: string;

  @Exclude()
  password: string;

  constructor(partial: Partial<UserEntity>) {
    Object.assign(this, partial);
  }
}
```

そうしたら、このクラスのインスタンスを返すメソッドハンドラを持つコントローラを考えてみよう。

```ts
@UseInterceptors(ClassSerializerInterceptor)
@Get()
findOne(): UserEntity {
  return new UserEntity({
    id: 1,
    firstName: 'Kamil',
    lastName: 'Mysliwiec',
    password: 'password',
  });
}
```

>WARNING  
>クラスのインスタンスを返す必要がある事に注意。`{ user: new UserEntity() }`のようなプレーンなJavaScriptオブジェクトを返すと、オブジェクトは正しくシリアライズされない。

>HINT  
>`ClassSerializerInterceptor`は`@nestjs/common`パッケージからインポートされている。

このエンドポイントがリクエストされると、クライアントは次のようなレスポンスを受け取る。

```ts
{
  "id": 1,
  "firstName": "Kamil",
  "lastName": "Mysliwiec"
}
```

インターセプターはアプリケーション全体に適用できることに注意（[ここ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-interceptors#%E3%82%A4%E3%83%B3%E3%82%BF%E3%83%BC%E3%82%BB%E3%83%97%E3%82%BF%E3%83%BC%E3%81%AE%E3%83%90%E3%82%A4%E3%83%B3%E3%83%87%E3%82%A3%E3%83%B3%E3%82%B0)で説明した通り）。インターセプターとエンティティクラスの宣言を組み合わせる事で、`UserEntity`を返すメソッドは必ず`password`プロパティを削除するようになる。このbussiness ruleを本気で実施する事ができる。

## プロパティの公開

以下のように、`@Expose()`デコレータを使用して、プロパティの別名を提供したり、プロパティの値を計算する関数を実行したりする事ができる（ゲッター関数に似ている）。

```ts
@Expose()
get fullName(): string {
  return `${this.firstName} ${this.lastName}`;
}
```

## 変換

`Transform()`デコレータを使用して、追加でデータ変換を行う事ができる。たとえば、次の構文は、オブジェクト全体を返すのではなく、`RoleEntity`の`name`プロパティを返す。

```ts
@Transform(role => role.name)
role: RoleEntity;
```

## オプションを渡す

変換関数のデフォルトの動作を変更したい場合がある。デフォルトの設定を上書きするには、`@SerializeOptions()`デコレータを使って、オプションオブジェクトを渡す。

```ts
@SerializeOptions({
  excludePrefixes: ['_'],
})
@Get()
findOne(): UserEntity {
  return new UserEntity();
}
```

>HINT  
>`SerializeOptions()`デコレータは`@nestjs/common`からインポートされている。

`SerializeOptions()`で渡されたオプションは、基礎となる`classToPlain()`関数の第二引数として渡される。この例では、_で始まる全てのプロパティを自動的に除外している。

## サンプル

動くサンプルは[こちら](https://github.com/nestjs/nest/tree/master/sample/21-serializer)

## WebSocketsとマイクロサービス

この章ではHTTPスタイルアプリケーションを例に出したが、`ClassSerializerInterceptor`はWebSocketsとマイクロサービスでも動く。