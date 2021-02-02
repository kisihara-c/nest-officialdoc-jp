---
title: overview-pipes
---

# パイプ
パイプは`@Injectable()`デコレータで装飾されたクラスだ。`PipeTransform`インターフェイスの実装が必要となる。

[画像](https://docs.nestjs.com/assets/Pipe_1.png)

パイプには２つのユースケースがある。
- 変換：入力データを希望の形に変換する（例：文字列→整数）
- 検証：入力データを評価し、データが正しければそのまま実行を続け、データが間違っていれば例外をthrowする

いずれもパイプはコントローラのルートハンドラによって処理される`arguments`を操作する。Nestはメソッドが呼び出される直前にパイプを挿入し、パイプはメソッドの引数を受け取って処理する。変換・検証はその時点で行われ、最後に変換された（かもしれない）引数でルートハンドラが呼び出される。  
Nestには組み込みのパイプが多数用意されており、すぐに使うことができる。独自のカスタムパイプを構築する事もできる。この章では組み込みパイプを紹介し、ルートハンドラにバインドする方法を示す。次に、いくつかカスタムパイプを見てみて、一からパイプを作る方法を示す。

>Hint  
>パイプは例外範囲内で動作する。つまり、パイプが例外をthrowした場合、例外レイヤ（グローバル例外フィルタと現在のコンテキストに適用されているすべての例外フィルタ）で処理される。その事から、パイプで例外がthrowされた時、其れ以降のコントローラは実行されない。システムの境界で外部の情報元から入ってくるデータを検証する為の、ベストプラクティス・テクニックだといえる。

## 組み込みパイプ

以下6つのパイプが即座に使用可能だ。

- `ValidationPipe`
- `ParseIntPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`
- `DefaultValuePipe`

上記は`@nestjs/common`パッケージからエクスポートされる。  
`ParseIntPipe`の使い方を簡単に見る。これは変換機能のユースケースで、パイプはメソッドハンドラの変数がJavaScriptの整数に変換されることを保証します。変換に失敗した場合例外を投げる。この章の後半では、`ParseIntPipe`のシンプルなカスタム例を紹介する。下記のテクニック例は、この章では`Parse*Pipe`とも呼ぶ他の組み込み変換パイプ――`ParseBoolPipe`、`ParseArrayPipe`、`ParseUUIDPipe`にも適用される。

## パイプのバインディング
パイプを使用するには、パイプクラスのインスタンスを適切なコンテキストにバインドする必要がある。`ParseIntPipe`の例では、パイプを特定のルートハンドラメソッドに関連付けて、そのメソッドが呼ばれる前に実行されるようにしたい。次のような構文を使用する。メソッド変数のレベルでパイプをバインドするものだ。

```ts
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

この構文で次の２つのいずれかが保証される。（我々が`this.catsService.findOne()`を呼ぶ時期待したように）`findOne()`メソッドで受け取る数値が数値である。あるいは、ルートハンドラが呼ばれる前に例外がthrowされる。  
例えば、こんな感じでルートが呼ばれたとする。

```
GET localhost:3000/abc
```

Nestはこんな例外を投げる。

```
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```

結果、`findOne()`の本体の実行は阻害される。  
上の例ではインスタンスではなくクラス（`ParseIntPipe`）を渡している。パイプやガードもそうだし、代わりにin-place instanceも渡すことができる。お決まりのin-place instanceを渡すのは、以下のように補足を渡して組み込みパイプの動作をカスタマイズしたい場合に便利だ。

```ts
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

変換機能を持つ他のパイプ（全Parse*パイプ）をバインドしても、同様に動作する。これらのパイプは全て、ルート変数、クエリ文字列変数、リクエストの本文の情報の値を検証するコンテキストで動作する。  
例えばクエリ文字列変数の場合。

```ts
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```
`ParseUUIDPipe`を使って文字列パラメータを解析し、`UUID`かどうか検証する例を示す。

```ts
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
```

>HINT
>`ParseUUIDPipe()`を使ってUUIDのver3,4,5を通している時、もし特定のバージョンのUUIDのみが必要な場合はパイプのオプション設定でバージョンを渡す事ができる。（※訳出怪しい…！　申し訳有りません）

上記では様々なParse*ファミリーの組み込みパイプのバインド例を見てきた。検証パイプのバインドは少し異なる。その事は続く章で議論していく。

>HINT  
>検証パイプの豊富な例はtechniques-validationの章を参照の事。

## カスタムパイプ

前述の通り独自のカスタムパイプを構築する事ができる。Nestは堅牢な`ParseIntPipe`と`ValidationPipe`を組み込みで提供しているが、カスタムパイプの構築方法を見る為にそれぞれのシンプルなカスタム版をゼロから構築してみよう。  
まずはシンプルな`ValidationPipe`から始める。最初は単純に入力値を受け取らせ、すぐに同じ値を返させよう。ID関数のように動作する。

```ts :validation.pipe.ts 
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
```

>Hint
>`PipeTransform<T, R>`は全てのパイプで実装されなければならない汎用インターフェイスだ。入力値の型を示すTと、`transform()`メソッドの戻り値の型を示すRを使用する。

全てのパイプは`PipeTransform`インターフェイスの規約を満たす為に`transform()`メソッドを実装する必要がある。このメソッドは２つパラメータを持つ。

- `value`
- `metadata`

`value`は現在処理中のメソッド引数（ルートハンドリングメソッドに受信される前のもの）。`metadata`は現在処理中のメソッド引数のメタデータだ。以下のプロパティを持つ。

```ts
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```