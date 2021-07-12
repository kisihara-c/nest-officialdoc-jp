---
title: security-encryptionandhashing
---

# 暗号化とハッシュ化

暗号化とは情報を変換するプロセスだ。平文と呼ばれる情報元を暗号文と呼ばれる別形式に変換する。理想的には認証済の当事者のみが暗号文を解読して平文に戻し元の情報にアクセスできる。暗号化は干渉を防ぐ事はできないが、傍受者になりうる相手に対して理解可能なコンテンツの提供を拒む。暗号化は双方向の機能であり、適切な鍵があれば復号できる。

ハッシュ化とは与えられた鍵を別の値に変換するプロセスだ。ハッシュ関数は数学的アルゴリズムに基づいて新しい値を生成する為に使われる。ハッシュ化が行われると、出力値から入力値へ戻す事は不可能になる。

## 暗号化

Node.jsには文字列・数値・バッファ・ストリームなどを暗号化・復号する為の[cryptoモジュール](https://nodejs.org/api/crypto.html)が組み込まれている。Nestは不要な抽象化を避けるようにこのモジュールの上に追加のパッケージを提供していない。

例としてAES（Advanced Encryption System）の`'aes-256-ctr'`アルゴリズムのCTR暗号化モードを使ってみよう。

```ts
import { createCipheriv, randomBytes, scrypt } from 'crypto';
import { promisify } from 'util';

const iv = randomBytes(16);
const password = 'Password used to generate key';

// キーの長さはアルゴリズム次第。
// ここのaes256では32バイト。
const key = (await promisify(scrypt)(password, 'salt', 32)) as Buffer;
const cipher = createCipheriv('aes-256-ctr', key, iv);

const textToEncrypt = 'Nest';
const encryptedText = Buffer.concat([
  cipher.update(textToEncrypt),
  cipher.final(),
]);
```

では復号しよう。

```ts
import { createDecipheriv } from 'crypto';

const decipher = createDecipheriv('aes-256-ctr', key, iv);
const decryptedText = Buffer.concat([
  decipher.update(encryptedText),
  decipher.final(),
]);
```

## ハッシュ化

ハッシュ化には、[bcrypt](https://www.npmjs.com/package/bcrypt)か[argon2](https://www.npmjs.com/package/argon2)の使用を勧める。Nestは不要な抽象化を避けるように（学習曲線を短くできるように）このモジュールの上に追加のパッケージを提供していない。

例として、bcryptでランダムなパスワードをハッシュ化しよう。

まず必要なパッケージをインストールする。

```
$ npm i bcrypt
$ npm i -D @types/bcrypt
```

インストールが完了したら、次のようにハッシュ関数を使える。

```ts
import * as bcrypt from 'bcrypt';

const saltOrRounds = 10;
const password = 'random_password';
const hash = await bcrypt.hash(password, saltOrRounds);
```

ソルトを生成するには`genSalt`関数を使う。

```ts
const salt = await bcrypt.genSalt();
```

パスワードの比較・チェックには`compare`関数を使う。

```ts
const isMatch = await bcrypt.compare(password, hash);
```

使用可能な関数の詳細は[こちら](https://www.npmjs.com/package/bcrypt)