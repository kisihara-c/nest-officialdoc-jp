---
title: techniques-fileupload
---

# ファイルのアップロード

Nestでは、ファイルアップロードを処理する為に、Express用のミドルウェアパッケージ　[multer](https://github.com/expressjs/multer)をベースにしたモジュールを内蔵している。Multerは`multipart/form-data`形式でポストされたデータを処理する。これはHTTP `POST`リクエストを介したアップロードに使用されるものだ。このモジュールはフルにカスタマイズ可能で、アプリケーションの要件に合わせて動作を調整する事ができる。

>WARNING  
>Multerはサポートされているマルチパート形式（`multipart/form-data`）以外のデータを処理できない。また、このパッケージは`FastifyAdapter`とは互換性がない事にも注意。

方の安全性を高める為に、Multer typingsパッケージを導入しよう。

```
$ npm i -D @types/multer
```

## 基本的な例

単一のファイルをアップロードするには、インターセプタ`FileInterceptor()`をルートハンドラに結びつけ、`@UploadedFile()`デコレータを使用してリクエストからファイルを抽出する。シンプルだ。

```ts
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
uploadFile(@UploadedFile() file: Express.Multer.File) {
  console.log(file);
}
```

>HINT  
>`FileInterceptor()`デコレータは`@nestjs/platform-express`パッケージからエクスポートされる。`@UploadedFile() `デコレータは`@nestjs/common`からエクスポートされる。

`FileInterceptor()`デコレータは２つの引数を取る。

- `fieldName`：ファイルを格納するHTMLフォームのフィールド名を指定する文字列
- `options`：省略可能。`MulterOptions`タイプのオブジェクト。これはmulterのコンストラクタで使用されるオブジェクトと同じもの（[詳細](https://github.com/expressjs/multer#multeropts)）。

>WARNING  
>`FileInterceptor()`はGoogle Firebaseなどのサードパーティのクラウドプロバイダと互換性がない場合がある。

## ファイルの配列

ファイルの配列（単一のフィールド名で識別）をアップロードするには、`FilesInterceptor()`デコレータを使用する（デコレータ名に複数形の**Files**が含まれている事に注目）。

- `fieldName`：上記通り
- `maxCount`：省略可能な数値、受け入れ可能なファイル最大数を定義する
- `options`：省略可能。`MulterOptions`タイプのオブジェクト。上記通り。

`FilesInterceptor()`を使用する場合は、`@UploadedFiles()`デコレータを使用して`request`からファイルを抽出する。

```ts
@Post('upload')
@UseInterceptors(FilesInterceptor('files'))
uploadFile(@UploadedFiles() files: Array<Express.Multer.File>) {
  console.log(files);
}
```

>HINT  
>`FilesInterceptor()`デコレータは`@nestjs/platform-express`パッケージからエクスポートする。`@UploadedFiles() `は`@nestjs/common`パッケージからエクスポートする。

## 複数のファイル

複数のフィールド（全てのフィールド名のキーが異なる）をアップロードするには、`FileFieldsInterceptor()`デコレータを使用する。このデコレータは２つの引数を取る。

- `uploadFields`：オブジェクトの配列。各オブジェクトはフィールド名を指定する文字列値を持つ必須の`name`プロパティと、オプションの`maxCount`プロパティを指定する。いずれも上述通りの性質。
- `options`：省略可能。`MulterOptions`タイプのオブジェクト。上記通り。

`FileFieldsInterceptor() `を使用する場合は、`@UploadedFiles()`デコレータを使用してリクエストからファイルを抽出する。

```ts
@Post('upload')
@UseInterceptors(FileFieldsInterceptor([
  { name: 'avatar', maxCount: 1 },
  { name: 'background', maxCount: 1 },
]))
uploadFile(@UploadedFiles() files) {
  console.log(files);
}
```

## あらゆるファイル

任意のフィールド名キーを持つすべてのフィールドをアップロードするには、`AnyFilesInterceptor()`デコレータを使用する。このデコレータは前述のように省略可能なオプションオブジェクトを受け入れられる。

`AnyFIlesInterceptor()`を使用する場合は、`@UploadedFIles()`デコレータを使用して、リクエストからファイルを抽出する。

```ts
@Post('upload')
@UseInterceptors(AnyFilesInterceptor())
uploadFile(@UploadedFiles() files: Array<Express.Multer.File>) {
  console.log(files);
}
```

## デフォルトのオプション

上記のようにファイルインターセプタではmulterオプションが使える。もしデフォルト値を設定したいなら、`MulterModule`をインポートする際に、静的メソッド`register()`を呼び出しサポートされている引数を渡す。[ここ](https://github.com/expressjs/multer#multeropts)で挙がっているすべてのオプションを使用可能。

```ts
MulterModule.register({
  dest: './upload',
});
```

>HINT  
>`MulterModule`クラスは`@nestjs/platform-express`パッケージからエクスポートする。

## 非同期の設定

`MulterModule`のオプションを静的に設定するのではなく、非同期に設定する必要がある場合は、`registerAsync()`メソッドを使う。ほとんどの動的モジュールと同様に、Nestは非同期の設定を扱う為のいくつかのテクニックを提供している。

一つはファクトリー関数を使う事だ。

```ts
MulterModule.registerAsync({
  useFactory: () => ({
    dest: './upload',
  }),
});
```

他の[ファクトリープロバイダ](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/fundamentals-customproviders#%E3%83%95%E3%82%A1%E3%82%AF%E3%83%88%E3%83%AA%E3%83%BC%E3%83%97%E3%83%AD%E3%83%90%E3%82%A4%E3%83%80%E3%80%81usefactory)と同様、ファクトリー関数も`async`であり、`inject`で依存関係をインジェクションできる。

```ts
MulterModule.registerAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    dest: configService.getString('MULTER_DEST'),
  }),
  inject: [ConfigService],
});
```

他の選択肢として、以下のようにクラスを使って`MulterModule`を設定する事もできる。

```ts
MulterModule.registerAsync({
  useClass: MulterConfigService,
});
```

上記のコードでは`MulterModule`内で`MulterConfigService`をインスタンス化し、それを使用して必要なオプションオブジェクトを作成している。この例では、`MulterConfigService`は以下に示すように`MulterOptionsFactory`インターフェイスを実装しなければならないことに注意。`MulterModule`は、提供されたクラスのインスタンス化されたオブジェクトの`createMulterOptions()`メソッドを呼び出す。

```ts
@Injectable()
class MulterConfigService implements MulterOptionsFactory {
  createMulterOptions(): MulterModuleOptions {
    return {
      dest: './upload',
    };
  }
}
```

`MulterModule`内にprivateなコピーを作成するのではなく、既存のオプションプロバイダを再利用したい場合は、`useExisting`構文を使用する。

```ts
MulterModule.registerAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
});
```

## サンプル

動くサンプルは[こちら](https://github.com/nestjs/nest/tree/master/sample/29-file-upload)