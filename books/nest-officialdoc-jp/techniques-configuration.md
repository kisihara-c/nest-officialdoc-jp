---
title: techniques-configuration
---

# 設定

アプリケーションはしばしば異なる**環境**で動作する。環境に応じて異なる構成設定を使用する必要がある。例えば通常ローカル環境ではローカルのDBインスタンスでのみ有効な特定のデータベース認証情報に依存している。本番環境では、別のDB資格情報のセットを使う。（原文、ここでamlという文字列が入ってるけど訳出できず…）ベストプラクティスは、[環境変数を保存](https://12factor.net/config)する事だ。外部で定義された環境変数は、`process.env`グローバル変数を通じてNode.js内部で確認できる。環境変数をそれぞれの環境で個別に設定する事で、複数の環境の問題を解決可能となる。特に、これらの値を簡単にモックしたり変更したりする必要がある開発環境やテスト環境では、あっという間に扱いにくくなってしまう。

Node.jsアプリケーションでは、各環境を表すために、各キーが特定の値を表すキーと値のペアを保持する`.env`ファイルを使用するのが一般的だ。異なる環境でアプリケーションを実行するには、適切に`.env`ファイルを入れ替えるだけで良い。

Nestでこのテクニックを使用するには、`ConfigModule`を作成するのが良い。このモジュールでは適切な`.env`ファイルをロードする`ConfigService`を表出する。Nestでは便利なように`@nestjs/config`パッケージが用意されている。このパッケージについて説明する。

## インストール

まず依存関係をインストールしよう。

```
$ npm i --save @nestjs/config
```

>HINT
>`@nestjs/config`パッケージは[dotenv](https://github.com/motdotla/dotenv)を使う。

## スタートアップ

インストールプロセスが完了したら、`ConfigModule`をインポートする事ができる。一般的には、ルートの`AppModule`にインポートし、静的メソッド`.forRoot()`を使ってその動作を制御する。ここでは環境変数のkey/valueペアが解析・解決される。後で、他の機能モジュールから`ConfigModule`の`ConfigService`クラスにアクセスする為のいくつかのオプションを見よう。

```ts :app.module.ts 
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot()],
})
export class AppModule {}
```

上記のコードは、デフォルトの場所（プロジェクトのルートディレクトリ）から`.env`ファイルをロード・解析し、`.env`ファイルのkey/valueペアを`process.env`に割り当てられた環境変数とマージし、その結果を`ConfigService`を通じてアクセスできるprivateな構造体に格納する。`forRoot()`は`ConfigService`プロバイダを登録する。`ConfigService`プロバイダはこれらの解析/マージされた設定変数を読み取るための`get()`メソッドを提供する。`@nestjs/config`は[dotenv](https://github.com/motdotla/dotenv)に依存しているため、環境変数名の競合を解決する際はdotenvのルールに則る。あるキーがランタイム環境の環境変数（例：OSシェルのエクスポート　`export DATABASE_USER=test`）と`.env`ファイルの両方に存在する場合、ランタイム環境の環境変数が優先される。

`.env`ファイルのサンプルは以下。

```
DATABASE_USER=test
DATABASE_PASSWORD=test
```

## カスタムenvファイルのパス

デフォルトでは、アプリケーションのルートディレクトリが第一候補となる。別のパスを指定するには、`forRoot()`に渡すオプションオブジェクトの`envFilePath`プロパティを以下のように設定の事。

```ts
ConfigModule.forRoot({
  envFilePath: '.development.env',
});
```

また、以下のように`.env`ファイルのパスを複数指定する事もできる。

```ts
ConfigModule.forRoot({
  envFilePath: ['.env.development.local', '.env.development'],
});
```

１つの変数が複数のファイルに存在する場合、最初のファイルが優先される。

## env変数の読み込みの無効

`.env`ファイルをロードせず、（`export DATABASE_USER=test`のようなOSシェルのエクスポートの様に）実行環境から環境変数にアクセスするだけにしたい場合、以下のようにオプションオブジェクトの`ignoreEnvFile`プロパティを`true`に設定する。

```ts
ConfigModule.forRoot({
  ignoreEnvFile: true,
});
```

## モジュールをグローバルに使う

`ConfigModule`を他のモジュールで使用したい場合は、（他のNestモジュールと同様）インポートする必要がある。あるいは以下のようにオプションオブジェクトの`isGlobal`プロパティを`true`に設定してグローバルモジュールとして宣言する事もできる。そうすれば、ルートモジュール（例：`AppModule`）にロードされた後、他のモジュールで`ConfigModule`をインポートする必要はない。

```ts
ConfigModule.forRoot({
  isGlobal: true,
});
```

## カスタムの設定ファイル

より複雑なプロジェクトでは、カスタム設定ファイルを利用して、ネストした設定オブジェクトを返す事ができる。これにより、関連する設定を機能別にグループ化したり（データベース関連の設定など）、関連する設定を個別のファイルに格納して独立して管理する事が可能になる。

カスタム構成ファイルは、構成オブジェクトを返すファクトリー関数をエクスポートする。設定オブジェクトには、自由にネストされたプレーンなJavaScriptオブジェクトを使用できる。`process.env`オブジェクトには、完全に解決された環境変数のkey/valueペアが含まれる。（`.env`ファイルと、外部で定義された変数は、上述のように解決・マージされる）返された設定オブジェクトは貴方がコントロールするので、値を適切な型にキャストしたり、デフォルト値を設定する等、必要なロジックを追加する事ができる。例：

```ts :config/configuration.ts 
export default () => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  database: {
    host: process.env.DATABASE_HOST,
    port: parseInt(process.env.DATABASE_PORT, 10) || 5432
  }
});
```

`ConfigModule.forRoot() `メソッドに渡したオプションオブジェクトの`load`プロパティを使って、このファイルを読み込む。

```ts
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [configuration],
    }),
  ],
})
export class AppModule {}
```

>Notice  
>`load`プロパティに割り当てられた値は配列だ。複数の設定ファイルを読むことができる。（例：`load:[databaseConfig,authConfig]`）

カスタム設定ファイルでは、YAMLファイルなどのカスタムファイルを管理する事もできる。ここではYAMLファイルの設定の例を紹介する。

```yaml
http:
  host: 'localhost'
  port: 8080

db:
  postgres:
    url: 'localhost'
    port: 5432
    database: 'yaml-db'

  sqlite:
    database: 'sqlite.db'
```

YAMLファイルは`js-yaml`パッケージで使える。

```
$ npm i js-yaml
$ npm i -D @types/js-yaml
```

パッケージのインストールが完了したら、`yaml#load`関数を使って上記のYAMLファイルを読み込む。

```ts :config/configuration.ts 
import { readFileSync } from 'fs';
import * as yaml from 'js-yaml';
import { join } from 'path';

const YAML_CONFIG_FILENAME = 'config.yml';

export default () => {
  return yaml.load(
    fs.readFileSync(join(__dirname, YAML_CONFIG_FILENAME), 'utf8'),
  ) as Record<string, any>;
};
```

>NOTE  
>NestCLIはビルドプロセス中に"assets"（非TSファイル）を自動的に`dist`フォルダに移動しない。YAMLファイルが自動的にコピーされるようにしたい場合は、`nest-cli.json`ファイルの`compilerOptions#assets`オブジェクトで指定する必要がある。例として、`config`フォルダが`src`フォルダと同じ階層にある場合、`compilerOptions#assets`に`"assets": [{"include": "../config/*.yaml", "outDir": "./dist/config"}]`という値を追加する。詳細は[こちら](https://docs.nestjs.com/cli/monorepo#assets)

## `ConfigService`を使う

`ConfigService`から設定値にアクセスするにはまず`ConfigService`をインジェクションする必要がある。他のプロバイダと同様に、`ConfigService`を含むモジュールである`ConfigModule`を、使用するモジュールにインポートする必要がある（`ConfigModule.forRoot()`メソッドに渡されるオプションオブジェクトの`isGlobal`プロパティを`true`に設定していない場合）。以下のように`feature`モジュールにインポートしてみよう。

```ts :feature.module.ts 
@Module({
  imports: [ConfigModule],
  // ...
})
```

次に、標準的なコンストラクタ・インジェクションを使用して、これをインジェクションする。

```ts
constructor(private configService: ConfigService) {}
```

>HINT
>`ConfigService`は`@nestjs/config`パッケージからインポートされている。

これをクラスの中で使ってみよう。

```ts
// 環境変数をget
const dbUser = this.configService.get<string>('DATABASE_USER');

// カスタム設定変数をget
const dbHost = this.configService.get<string>('database.host');
```

上記のように、`configService.get()`メソッドを使って変数名を渡せばシンプルな環境変数を取得できる。型を渡すことで、TypeScriptの型ヒントをつける事もできる（例：`get<string>(...)`）。get()メソッドは、上記の２番目の例のように、（カスタム設定ファイルを介して作成された）ネストしたカスタム設定オブジェクトを探索する事もできる。

また、型ヒントとしてインターフェイスを使用し、ネストしたカスタム設定オブジェクト全体を取得する事もできる。

```ts
interface DatabaseConfig {
  host: string;
  port: number;
}

const dbConfig = this.configService.get<DatabaseConfig>('database');

// こうすれば`dbConfig.port`と`dbConfig.host`を使える
const port = dbConfig.port;
```

`get()`メソッドは省略可能な第２引数を取り、キーが存在しない場合に返されるデフォルト値を定義する。

```ts
// "database.host"が存在しない時"localhost"を使用
const dbHost = this.configService.get<string>('database.host', 'localhost');
```

`ConfigService`には、存在しないコンフィグプロパティへのアクセスを防止するために、オプションのジェネリック（型引数）が用意されている。以下のように使ってほしい。

```ts
interface EnvironmentVariables {
  PORT: number;
  TIMEOUT: string;
}

// コードのどこか
constructor(private configService: ConfigService<EnvironmentVariables>) {
  // 有効
  const port = this.configService.get<number>('PORT');

  // URLは EnvironmentVariablesインターフェイスのプロパティではない為、無効
  const url = this.configService.get<string>('URL');
}
```

>NOTICE  
>上記の`database.host`の例のように、コンフィグにネストされたプロパティがある場合、インターフェイスにはマッチする`'database.host': string;`が必要。なかったらTypeScriptエラーが発生。

## 設定の名前空間

`ConfigModule`では、上記のカスタム設定ファイルセクションで示したように、複数のカスタム設定ファイルを定義してロードできる。同じ場所で示したように、ネストした設定オブジェクトを使って、複雑な設定オブジェクトの階層管理を行える。また、以下のように`registerAs()`関数で「名前空間」化した構成オブジェクトを返す事もできる。

```ts :config/database.config.ts 
export default registerAs('database', () => ({
  host: process.env.DATABASE_HOST,
  port: process.env.DATABASE_PORT || 5432
}));
```

カスタム設定ファイルを使う場合と同様、`registerAs()`ファクトリ関数の内部では、`process.env`オブジェクトに、完全に解決された環境変数のkey/valueペアが格納される（上記のように、`.env`ファイルと外部で定義された変数が解決・マージされる）。

>HINT  
>`registerAs`関数は`@nestjs/config`パッケージからエクスポートされる。

`forRoot()`メソッドの`options`オブジェクト内`load`プロパティを使用し、カスタム設定ファイルを読み込むのと同じ方法で名前空間の設定を読みこんでみよう。

```ts :
import databaseConfig from './config/database.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [databaseConfig],
    }),
  ],
})
export class AppModule {}
```

さて、データベースの名前空間から`host`変数を取得するには、ドット記法を使う。プロパティ名のプレフィックスとして`database`を使用し、（`registerAs()`関数の第一引数として渡される）名前空間の名前に対応させる。

```ts
const dbHost = this.configService.get<string>('database.host');
```

データベースの名前空間を直接インジェクションするという方法もある。こうすると強力な型付けの恩恵を受ける事ができる。

```ts
constructor(
  @Inject(databaseConfig.KEY)
  private dbConfig: ConfigType<typeof databaseConfig>,
) {}
```

>HINT  
>`ConfigType`は`@nestjs/config`パッケージからエクスポートされる。

## 環境変数のキャッシュ
`process.env`へのアクセスには時間がかかる。`ConfigModule.forRoot()`に渡されるオプションオブジェクトに`cache`プロパティを設定する事で、`process.env`に格納されている変数に関して`ConfigService#get`メソッドのパフォーマンスを向上させる事ができる。

```ts
ConfigModule.forRoot({
  cache: true,
});
```

## 部分的レジストレーション

ここまでは`forRoot()`メソッドを使ってルートモジュール（`AppModule`など）の設定ファイルを処理してきた。しかし、プロジェクトの構造がもっと複雑で、機能別の設定ファイルが複数の異なるディレクトリに置かれている場合もあるかもしれない。`@nestjs/config`パッケージでは、ルートモジュールでこれらを全て読み込むのではなく、各機能モジュールに関する設定ファイルのみを参照する**部分的レジストレーション**機能を提供している。部分的レジストレーションを行うには、以下のように、機能モジュール内で`forFeture()`静的メソッドを使用してほしい。

```ts
import databaseConfig from './config/database.config';

@Module({
  imports: [ConfigModule.forFeature(databaseConfig)],
})
export class DatabaseModule {}
```

>WARNING  
>状況次第で、部分的レジストレーションによって読み込まれたプロパティに、コンストラクタではなく`onModuleInit()`フックを使ってアクセスする必要がある。これは、`forFeature()`メソッドがモジュールの初期化中に実行され、モジュール初期化の順序が不定である為だ。他のモジュールから読み込まれた値にアクセスすると、コンストラクタでは、その設定が依存するモジュールがまだイニシャライズされていない可能性がある。`onModuleInit()`メソッドは依存する全てのモジュールが初期化された後にのみ実行される。安全だ。

## スキーマバリデーション

アプリケーションの起動時に、必要な環境変数が提供されていない場合や、特定の検証ルールを満たしていない場合には、例外を発生させるのが標準的なやり方だ。`@nestjs/config`パッケージでは、２通りの方法で行える。

- [Joi](https://github.com/sideway/joi)組み込みバリデータ　Joiではオブジェクトスキーマを定義して、それに対してJavaScriptオブジェクトを検証する。
- 環境変数を入力して受け取るカスタム`validate()`関数

Joiを使うにはパッケージをインストールしなければならない。

```
$ npm install --save joi
```

>Notice  
>最新版のJoiは、Node v12以降のバージョンを必要とする。古いバージョンを持っている場合は、v16.1.8をインストールしてほしい。これはビルド時にエラーが発生するv17.0.2がリリースされた後だからだ。詳細については、Joi v17.0.0の[リリースノート](https://github.com/sideway/joi/issues/2262)を参照の事。

Joiの検証スキーマを定義し、以下のように`forRoot()`メソッドのオプションオブジェクトの`validationSchema`プロパティを介して渡すことができる。

```ts :app.module.ts 
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().default(3000),
      }),
    }),
  ],
})
export class AppModule {}
```

デフォルトでは、全てのスキーマのキーは省略可能とみなされる。ここでは、`NODE_ENV`と`PORT`にデフォルト値を設定している。この値は、環境（`.env`ファイルかプロセス環境）内でこれらの変数が提供されていない場合使われる。また、`required()`バリデーション・メソッドを使用して、環境内（同上）で値が定義されている必要を明示する事もできる。この場合、環境内で該当の変数が提供されていないと、バリデーションの際に例外が発生する。検証スキーマの構築方法の詳細については、[Joiのバリデーションメソッド](https://joi.dev/api/?v=17.3.0#example)を参照の事。

デフォルトでは、未知の環境変数（スキーマにキーが存在しない環境変数）が許可されて、バリデーションエクセプションは発生しない。デフォルトでは全てのバリデーションエラーが報告される。これらの動作を変更するには、`forRoot()`オプションオブジェクトの`validationOptions`キーにオプションオブジェクトを渡す。このオプションオブジェクトは、[Joiのバリデーションオプション](https://joi.dev/api/?v=17.3.0#anyvalidatevalue-options)で提供される標準的なバリデーションオプションのプロパティを含む事ができる。例えば上記の２つの設定を切り替えるには、以下のオプションを通す。

```ts :app.module.ts 
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().default(3000),
      }),
      validationOptions: {
        allowUnknown: false,
        abortEarly: true,
      },
    }),
  ],
})
export class AppModule {}
```

`@nestjs/config`パッケージは以下のデフォルト設定を使っている。

- allowUnknown　環境変数の未知のキーを許可するかどうかを制御。デフォルトは`true`
- abortEarly　`true`時最初のエラーでバリデーションを停止、`false`時全てのエラーを返す。デフォルトは`false`

注意としては、`validationOptions`オブジェクトを渡す事にすると、明示的に渡さなかった設定は、（`@nestjs/config`のデフォルトではなく）Joiの標準的なデフォルトになる事。例えば、カスタム`validationOptions`オブジェクトで`allowUnknowns`を指定しないままにしておくと、Joiのデフォルト値である`false`になる。したがって、これらの設定の両方をカスタムオブジェクトで指定するのが最も安全だろう。

## カスタムバリデーション関数

別の手段として、同期型のバリデーション関数を指定する事もできる。この関数は、（`env`ファイルとプロセスからの）環境変数を含むオブジェクトを受け取り、バリデーション済みの環境変数を含むオブジェクトを返す為、必要に応じて環境変数をコンバート/ミューテートする事ができる。この関数がエラーを出した場合、アプリケーションの起動が防がれる。

この例では`class-transformer`と`class-validator`のパッケージを使って勧める。まず定義する必要がある。

- バリデーション制約を持つクラス
- `plainToClass`関数と`validateSync`関数を使った`validate`関数


```ts :env.validation.ts 
import { plainToClass } from 'class-transformer';
import { IsEnum, IsNumber, validateSync } from 'class-validator';

enum Environment {
  Development = "development",
  Production = "production",
  Test = "test",
  Provision = "provision",
}

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment;

  @IsNumber()
  PORT: number;
}

export function validate(config: Record<string, unknown>) {
  const validatedConfig = plainToClass(
    EnvironmentVariables,
    config,
    { enableImplicitConversion: true },
  );
  const errors = validateSync(validatedConfig, { skipMissingProperties: false });

  if (errors.length > 0) {
    throw new Error(errors.toString());
  }
  return validatedConfig;
}
```