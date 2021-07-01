---
title: techniques-modelviewcontroller
---

# MVC

Nestはデフォルトで[Express](https://github.com/expressjs/express)ライブラリを使っている。結果、ExpressでMVCパターンを使用するためのあらゆるテクニックを適用できる。

まずは[CLI](https://github.com/nestjs/nest-cli)ツールを使ってシンプルなNestアプリケーションを作ってみよう。

```
$ npm i -g @nestjs/cli
$ nest new project
```

MVCアプリケーションを作るためには、HTMLビューをレンダリングする[テンプレートエンジン](https://expressjs.com/en/guide/using-template-engines.html)も必要だ。

```
$ npm install --save hbs
```

ここでは、[hbs](https://github.com/pillarjs/hbs#readme)（Handlebars）エンジンを使う。必要に応じて何を使っても問題ない。インストールが完了したら、次のコードでexpressインスタンスの設定を行う。

```ts :main.ts
import { NestFactory } from '@nestjs/core';
import { NestExpressApplication } from '@nestjs/platform-express';
import { join } from 'path';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(
    AppModule,
  );

  app.useStaticAssets(join(__dirname, '..', 'public'));
  app.setBaseViewsDir(join(__dirname, '..', 'views'));
  app.setViewEngine('hbs');

  await app.listen(3000);
}
bootstrap();
```

`public`ディレクトリに静的アセット、`views`にはテンプレートを格納し、HTML出力のレンダリングには`hbs`を使うよう明記した。

## テンプレートのレンダリング

それでは、`views`ディレクトリを作成し、その中に`index.hbs`テンプレートを作成してみよう。テンプレートでは、コントローラから渡された`message`を表示する。

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>App</title>
  </head>
  <body>
    {{ message }}
  </body>
</html>
```

次に、ファイル`app.controller`を開き、`root()`メソッドを以下のコードで置き換える。

```ts :app.controller.ts
import { Get, Controller, Render } from '@nestjs/common';

@Controller()
export class AppController {
  @Get()
  @Render('index')
  root() {
    return { message: 'Hello world!' };
  }
}
```

このコードでは、`@Render()`デコレータでテンプレートを指定している。ルートハンドラメソッドの戻り値がテンプレートに渡され、レンダリングされる。注目してほしい、戻り値はテンプレートで作成した`message`プレースホルダに当てはまる、`message`プロパティを持つオブジェクトだ。

## 動的テンプレートレンダリング

アプリケーションロジックがレンダリングするテンプレートを動的に決定しなければならない場合は、`@Res()`デコレータを使い、ビュー名を`@Render()`デコレータではなくルートハンドラで指定する必要がある。

>HINT  
>Nestは`@Res()`デコレータを検出すると、ライブラリ固有の`response`オブジェクトをインジェクションする。このオブジェクトを使って、テンプレートを動的にレンダリングする事ができる。`response`オブジェクトAPIの詳細は[こちら](https://expressjs.com/en/api.html)

```ts :app.controller.ts 
import { Get, Controller, Res, Render } from '@nestjs/common';
import { Response } from 'express';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private appService: AppService) {}

  @Get()
  root(@Res() res: Response) {
    return res.render(
      this.appService.getViewName(),
      { message: 'Hello world!' },
    );
  }
}
```

## サンプル

実際に動くサンプルは[こちら](https://github.com/nestjs/nest/tree/master/sample/15-mvc)

## fastify

前述の通り、Nestでは互換性のある任意のHTTPプロバイダを使用できる。Fastifyを使ってMVCアプリケーションを作成するには、以下のパッケージをインストールする必要がある。

```
$ npm i --save fastify-static point-of-view handlebars
```

次のステップはExpressとほぼ同じだが、プラットフォーム固有の細かい違いがある。インストールプロセスが完了したら、main.tsを開いて内容を更新する。

```ts :main.ts
import { NestFactory } from '@nestjs/core';
import { NestFastifyApplication, FastifyAdapter } from '@nestjs/platform-fastify';
import { AppModule } from './app.module';
import { join } from 'path';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter(),
  );
  app.useStaticAssets({
    root: join(__dirname, '..', 'public'),
    prefix: '/public/',
  });
  app.setViewEngine({
    engine: {
      handlebars: require('handlebars'),
    },
    templates: join(__dirname, '..', 'views'),
  });
  await app.listen(3000);
}
bootstrap();
```

FastifyのAPIは若干異なるが、これらのメソッド呼び出しの最終結果は同じとなる。1つの違いは、`@Render()`デコレータに渡されるテンプレート名が、ファイル拡張子を含まなければならない事だ。

```ts :app.controller.ts
import { Get, Controller, Render } from '@nestjs/common';

@Controller()
export class AppController {
  @Get()
  @Render('index.hbs')
  root() {
    return { message: 'Hello world!' };
  }
}
```

`http://localhost:3000`にアクセスしてみよう。`Hello world!`のメッセージが表示されるはずだ。

## サンプル

実際に動くサンプルは[こちら](https://github.com/nestjs/nest/tree/master/sample/17-mvc-fastify)