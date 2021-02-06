---
title: overview-interceptors
---

# インターセプター
インターセプターは`@Injectable()`デコレータでアノテーションされたクラスだ。`NestInterceptor`インターフェースを実装する必要がある。  
[画像](https://docs.nestjs.com/assets/Interceptors_1.png)  
インターセプターは[アスペクト指向プログラミング（AOP）](https://en.wikipedia.org/wiki/Aspect-oriented_programming)の技術に触発された便利な機能を持ち、以下の事が可能になっている。

- メソッドの前後に追加のロジックをバインド
- 関数の結果を変換
- 関数からthrowされた例外を変換
- 基本関数の動作を拡張
- 条件次第では関数を完全に乗っ取る（例：cathing purpose（すみません訳出できず。例外のcatchですかね…））

## 基本
各インターセプターは`intercept()`メソッドを実装している。最初の引数は`ExecutionContext`インスタンスだ（ガードの場合と全く同じオブジェクト）。`ExecutionContext`は`ArugmentsHost`を継承している。以前に例外フィルタの項で`Arg～`を見た。元のハンドラに渡された引数のラッパーであり、アプリケーションのタイプに応じて異なる引数の配列を含んでいることを確認した。このトピックについてはexeption-filtersの項を参照の事。

## 実行コンテキスト
`ArgumentsHost`を拡張する事で、`ExectutionContext`は現在の実行プロセスに関するいくつかの新しいヘルパーメソッドも手に入れた。より幅広い範囲のコントローラ・メソッド・及び実行コンテキストで動く、より汎用的なガードを構築する際に役に立つ。詳細はexecution-contextの項を参照の事。

## 呼び出しハンドラ
２番目の引数は`CallHandler`だ。`CallHandller`インターフェイスは`handle()`メソッドを実装しており、インターセプター内のいくつかの場所でルートハンドラメソッドを呼び出す為に使用できる。`intercept()`メソッドの実装で`handle()`を呼び出さなければ、ルートハンドラメソッドは全く実装されない。  
このアプローチは、`intercept()`メソッドがリクエスト/レスポンスの流れを効果的に**補足する(wrap)**事を意味する。結果として、最終的なルートハンドラの**実行前後の両方**に独自のロジックを追加できる。つまり`handle()`の前を呼び出す**前**に実行するコードを`intercept()`メソッドに書く事ができるわけだが、それがその後の処理にどう影響するのだろう？　`handle()`メソッドはObservableを返すので、強力な[RxJS](https://github.com/ReactiveX/rxjs)演算子を使ってレスポンスをさらに操作することができる。アスペクト指向プログラミングの用語を使うと、ルートハンドラの呼び出し（つまり`handle()`の呼び出し）はポイントカット（[Pointcut](https://en.wikipedia.org/wiki/Pointcut)）と呼ばれ、追加ロジックが挿入される場所である事を示す。  
例えばPOST/catsリクエストが入ってきたとする。このリクエストは、`CatsController`内で定義されている`create()`ハンドラに送られてくる。途中で`handle()`メソッドを呼ばないインターセプターが呼ばれた場合、`create()`メソッドは実行されない。`handle()`が呼ばれ（て`Observable`が返され）ると`create()`ハンドラが起動する。レスポンスストリームが`Observable`を介して受信されると、ストリームに対して追加操作を行う事ができ、最終結果が呼び出し元に返される。

## アスペクトインターセプション
最初のユースケースはインターセプターを利用してユーザのインタラクションをログに記録する事だ。例としてはユーザーコールの保存、イベントの非同期ディスパッチ、タイムスタンプの計算等。以下にシンプルな`LogginInterceptor`を示す。  
```ts :logging.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
```
>HINT  
>`NestInterceptor<T,R>`は汎用インターフェイスで、Tがレスポンスストリームをサポートする`Observable<T>`の型、Rが`Observable<R>`でラップされた値の型を示す。

>Notice  
>コントローラ、プロパイダ、ガードなどのインターセプタは、そのコンストラクタを通して依存課員系のインジェクションを行える。

handle()はRxJSの`Observable`を返すので、ストリームを操作するために使用できる演算子の選択肢が豊富だ。上記の例では、`tap()`演算子を使用している。この演算子では、`observable`ストリームの優雅なor例外を吐く終了時に匿名ロギング関数を呼び出すが、それ以外の場合はレスポンスサイクルには干渉しない。

## インターセプターのバインディング
インターセプターを設定するために、`@nestjs/common`パッケージからインポートされた`@UseInterceptors()`デコレータを使用します。パイプやガードと同様に、インターセプタもコントローラ・メソッド・グローバルの範囲でスコープ化できます。

```ts :cats.controller.ts
@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```

>HINT  
>`@UseInterceptors()`デコレータは`@nestjs/common`パッケージからインポートされている。

上記のコードを使って、`CatsController`で定義された各ルートハンドラは`LoggingInterceptor`を使用する。誰かがGET/catsエンドポイントを呼び出すと、標準出力では以下のように出力される。

```
Before...
After... 1ms
```

（インスタンスの代わりに）`LoggingInterceptor`型を渡し、フレームワークにインスタンス化の責任を任せ、依存性のインジェクションを可能にしている事に注意。パイプ・ガード・例外フィルタと同様に、in-placeなインスタンスを渡すこともできる。

```ts :cats.controller.ts 
@UseInterceptors(new LoggingInterceptor())
export class CatsController {}
```

前述したように、上記のコードでは、このコントローラに宣言された全てのハンドラにインターセプターをアタッチしている。インターセプターのスコープを単一のメソッドに制限したい場合は、単に**メソッドレベル**でデコレータを適用する。  
グローバルなインターセプターを作るには、Nestアプリケーションインスタンスの`useGloballInterceptors()`メソッドを使用する。

```ts
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

グローバルインターセプターは全てのコントローラと全てのルートハンドラに対してアプリケーション全体を通して使用される。依存関係の点では、すべてのモジュールの外部から登録されたグローバルインターセプターは（上記の例のように`useGlobalInterceptors()`を使用して）依存関係をインジェクションできない。なぜなら全てのモジュールのコンテキストの外で実行されるからだ。以下のコードでこの問題を解決して、全てのモジュールから直接インターセプターをセットアップできる。

```ts app.module.ts
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```

>Hint  
>このアプローチを採用してガードの依存性インジェクションを実行する場合、この構造が採用されているモジュールに関係なく、インターセプターは実際にはグローバルであることに注意。どこでやるべきか？　インターセプター（上の例では`LoggingInterceptor`）が定義されているモジュールを選ぼう。また、カスタムプロバイダの登録を扱う方法は`useClass`だけではない。詳細はcustom-providersの項にて。

## レスポンスのマッピング
知っての通り、`handle()`は`Observable`を返す。ストリームにはルートハンドラから返された値が含まれているので、RxJSの`map()`演算子を使って簡単に変更できる。

>Waring  
>レスポンスマッピング機能は、ライブラリ固有のレスポンス操作方針上だと動作しません（`@Res()`オブジェクトを直接使用する事は禁止されている）

`TransformInterceptor`を作成してみよう。プロセスをデモンストレーションする為の、簡単な方法で各レスポンスを修正するものだ。RxJSの`map()`演算子を使用して、reponseオブジェクトを新しく作成されたオブジェクトのデータプロパティに代入し、新しいオブジェクトをクライアントに返す。

```ts :transform.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

>HINT  
>Nestのインターセプターは、同期と非同期型の両方の`intercept()`メソッドで動作する。必要に応じてメソッドを非同期に切り替える事ができる。

上記のコードで誰かがGET/catsエンドポイントを呼び出すと、レスポンスは以下のようになる（ルートハンドラは空の配列`[]`を返すと仮定する）。

```ts
{
  "data": []
}
```

インターセプターは、アプリケーション全体で発生する要件に対し、再利用可能なソリューションを作成する中に大きな価値がある。例えば`null`が発生するたびに空文字列` `` `を作成したい場面を想像してほしい。１行のコードを使い、インターセプターをグローバルにバインドして、登録された各ハンドラによって自動的に使用されるようにすれば、可能だ。

```ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
```

## 例外マッピング
別の面白い使用例を挙げる。RxJSの`catchError()`演算子を利用して、投げられた例外をオーバーライドする事だ。

```ts :errors.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  BadGatewayException,
  CallHandler,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(new BadGatewayException())),
      );
  }
}
```

## ストリームオーバーライド
ハンドラの呼び出しを完全に防いで代わりに別の値を返したい事やその理由がたまにある。明白な例は、キャッシュを実装してレスポンスタイムを改善したいケースだ。キャッシュからレスポンスを返す、シンプルな**cache interceptor**を見てみよう。現実的な例ではTTL、キャッシュの無効化、キャッシュサイズその他の要因を考慮したいところだが、それは議論のスコープを超えてしまう。ここでは主要概念を示す基本的な例を提供する。

```ts :cache.interceptor.ts 
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```

`CacheInterceptor`では`isCached`変数とレスポンス`[]`がある。キーポイントは、ここではRxJSの`of()`オペレータで作成された新しいストリームを返しており、ルートハンドラは全く呼び出されないということだ。誰かが`CacheInterceptor`を使用するエンドポイントを呼び出すと、レスポンス（ハードコードされた空の配列）がすぐに返される。汎用的なソリューションを作成するには、`Reflector`を活用してのカスタムデコレータを作成する。`Reflector`についてはguards項でよく説明されている。

## 他の演算子
RxJSの演算子でストリームを操作できる事は、多くの機能に繋がる。別の一般的なユースケースを考えてみよう。ルートリクエストのタイムアウトを処理したいとする。エンドポイントが一定時間経過しても何も返さない場合、エラーのレスポンスで終了させたい。次のコードで実現できる。

```ts :timeout.interceptor.ts 
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(new RequestTimeoutException());
        }
        return throwError(err);
      }),
    );
  };
};
```

5秒経つとリクエスト処理がキャンセルされる。`RequestTimeoutExeption`を投げる前にカスタムロジックを追加する事もできる。（例：リソースの解放等）