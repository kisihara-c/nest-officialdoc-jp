---
title: fundamentals-circulardependency
---

# 循環依存関係

循環依存関係は２つのクラスが互いに依存している場合に発生する。クラスAがクラスB、クラスBがクラスAを必要としている場合だ。Nestではモジュール間やプロバイダ間で循環依存関係が発生する事がある。

この現象は可能な限り回避すべきとはいえ、常に回避できるわけではない。こういった場合、Nestでは２つの方法でプロバイダ間の循環依存関係を解決できる。「前方参照」と、`ModuleRef`クラスを使用してDIコンテナからプロバイダのインスタンスを取得する方法の２つだ。

またモジュール間の循環依存関係の解決法も説明する。

>WARNING  
>index.ts（バレル）を使用してインポートをグループ化する時、循環的な依存関係が発生する可能性がある。モジュール/プロバイダクラスに関してはバレルファイルを排除するべきだ。例えばバレルファイルと同じディレクトリ内のファイルをインポートする時、バレルファイルを使用すべきではない。もっと具体的に例えると、`cats/cats.controller`は`cats/cats.service`ファイルをインポートするために`cats`をインポートするべきではない。詳細についてはこの[github issue](https://github.com/nestjs/nest/issues/1181#issuecomment-430197191)も参照の事。

## 前方参照
前方参照を使用すると、NestはforwardRef()ユーティリティ関数を使って、まだ定義されていないクラスを参照できる。例えば`CatsService`と`CommonService`が互いに依存している場合、関係の両側で`@inject()`と`forwardRef()`ユーティリティを使って循環依存関係を解決できる。さもなければ重要なメタデータが全て利用不可能になる為、Nestはそれらをインスタンス化しない。  
例はこちら：

```ts :cats.service.ts@Injectable()
export class CatsService {
  constructor(
    @Inject(forwardRef(() => CommonService))
    private commonService: CommonService,
  ) {}
}
```

>HINT  
>`forwardRef()`関数は`@nestjs/common`パッケージからインポートされている。

関係の片側をカバーできた。`CommonService`でも同様にしてみよう。

```ts :common.service.ts 
@Injectable()
export class CommonService {
  constructor(
    @Inject(forwardRef(() => CatsService))
    private catsService: CatsService,
  ) {}
}
```

> WARNING  
> インスタンス化の順番は不確定だ。どのコンストラクタが最初に呼び出されるかについて、コードが依存しないようにしてほしい。

## ModuleRefクラスオルタ

`forwardRef()`の代わりに、コードをリファクタリングし、`ModuleRef`クラスを使用して、循環関係のもう片側にあるプロバイダを取得する事ができる。`ModuleRef`ユーティリティクラスの詳細はfundamentals-module-refの章にて。

## モジュール前方参照

モジュール間の循環的な依存関係を解決するには、モジュールの関連付けの両側で同じ`forwardRef()`ユーティリティ関数を使用する。  
例：
```ts :common.module.ts
@Module({
  imports: [forwardRef(() => CatsModule)],
})
export class CommonModule {}
```