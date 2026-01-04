---
title: "PostgreSQL でクエリを実行せずにクエリのメタデータを取得するツールを書いた"
emoji: "✨"
type: "tech"
topics: ["postgresql", "nodejs", "deno"]
published: true
---
## はじめに

コードジェネレータを書く等で、SQL クエリのメタデータを取得したくなる場面が多々あると思います。
ここで言う、 SQL クエリのメタデータとは、バインディングされる変数の型や数、結果の列の名前や型などです。

Node.js の [node-postgres](https://node-postgres.com/) は実行をしないと結果の列のメタデータを取得することが出きません。
通常のクエリ実行（SELECT）では実際にサーバーが評価されるため副作用や実行コストが発生します。
そのため、 Node.js で PostgreSQL のクエリから入力や出力のデータモデルのコードを生成するには色々考慮が必要な状況でした。

しかし PostgreSQL の通信プロトコル（以下「PostgreSQL Wire Protocol」） の Describe を用いれば、実行せずに 解析段階の情報（メタデータ）だけ を得ることが出きます。
その情報を単に出力するだけのツール - [pgpd](https://github.com/yskszk63/pgpd) を書きました。
この記事はツールを書く上で必要になった知見の記録を目的にしています。

なお、筆者は PostgreSQL については詳しくありません。誤りなどありましたらご指摘いただけると幸いです。

### pgpd の紹介

まずは、書いたツール [pgpd](https://github.com/yskszk63/pgpd) について軽くご紹介します。

CLI / API で利用でき、 CLI では以下のようになります。

```bash
$ deno run -A jsr:@pgpd/pgpd/cli.ts 'postgres://postgres:password@localhost:5432/postgres?sslmode=disable' << EOF | jq
VALUES(\$1::text)
EOF
{
  "parameters": [
    {
      "type": {
        "oid": 25,
        "schema": "pg_catalog",
        "name": "text",
        "sqlType": "text"
      }
    }
  ],
  "rows": [
    {
      "name": "column1",
      "type": {
        "oid": 25,
        "schema": "pg_catalog",
        "name": "text",
        "sqlType": "text"
      },
      "format": "text"
    }
  ]
}
```

引数は接続先、クエリの書かれたファイル(省略の場合は標準入力)の順に指定し結果が標準出力に出力されます。

[npm - pgpd](https://www.npmjs.com/package/pgpd) 及び [jsr - @pgpd/pgpd](https://jsr.io/@pgpd/pgpd) で公開しており、 Node.js / Deno / Bun で動作します。
(なお、開発自体は Deno を利用)

---

## PostgreSQL のクライアントとサーバー間の通信で利用される メッセージベースのプロトコル

PostgreSQL のクライアントとサーバー間の通信で利用される メッセージベースのプロトコル (以降、 PostgreSQL Wire Protocol と呼びます) について説明をします。

### 概要

もし、プロトコルの詳細について知りたい場合は下記をご参照下さい。
https://www.postgresql.jp/docs/17/protocol.html

クライアントからサーバー、サーバーからクライアントのメッセージは同じ形式です。
先頭1バイトに、メッセージの種類を示すタグがあり、次の4バイトにメッセージの長さが格納されています。
それ以降は、メッセージの種類に応じて形式が決まっています。

```
| 1    | 2 | 3 | 4 | 5 | ... | N - 4 + 1 |
|------|---|---|---|---|-----|-----------|
| 種類 | 長さ N (自身含む)   | 種類による|
```

TLS (SSL) 開始要求や開始要求など、接続初期のハンドシェークの一部を除き、ほぼ同じような形式でメッセージがやりとりされます。

たとえば、バインディングの無い単純なクエリは、下記のような流れになります。
※ 判りやすさを優先して、メッセージを一部省略しているため不正確な記述になります。

1. C -> S ... `Query`
    - `Query(SELECT 1, 2)` のように 問い合わせ文字列を含む
2. S -> C ... `RowDescription`
    - 結果の列の名前や型などを返却
3. S -> C ... `DataRow`
    - `DataRow(1, 2)` のように行毎の列の値が含まれる
4. S -> C ... `ReadyForQuery`
    - 問い合わせの応答が完了し、クライアントからのコマンドを受け付けられる状態になった

### 拡張問い合わせ

先程の `Query` は簡易問い合わせで理解しやすい形式でした。
一方、プレースホルダを置きクエリ文とは別に値をバインドして再利用できるクエリの発行は拡張問い合わせを利用します。

よくある流れは下記のようになります。
※ 判りやすさを優先して、メッセージを一部省略しているため不正確な記述になります。また、ポータルやプリペアステートメントの名前などは触れません

1. C -> S `Parse`
    - `Parse(SELECT $1, $2)` のようにプレースホルダが置かれたSQL文の解析依頼
    - このメッセージ送信時点では実行されず、 後の `Sync` メッセージの送信により、実行される
2. C -> S `Describe`
    - `Parse` した SQL文のメタデータを要求
    - このメッセージは送信しなくても可
    - このメッセージ送信時点では実行されず、 後の `Sync` メッセージの送信により、実行される
3. C -> S `Bind`
    - `Bind(1, 2)` のようにプレースホルダにバインドする値を含む
    - このメッセージ送信時点では実行されず、 後の `Sync` メッセージの送信により、実行される
4. C -> S `Execute`
    - 解析したSQL文とバンドされた値でクエリの実行を要求
    - このメッセージ送信時点では実行されず、 後の `Sync` メッセージの送信により、実行される
5. C -> S `Sync`
    - このメッセージの送信により、 `Parse` の送信からここまで送信されたメッセージが実行される
6. C -> S `ParseComplete`
    - SQL文の解析が完了した旨
7. C -> S `ParameterDescription`
    - `Describe` が送信されたにより返却される、SQL文のプレースホルダの型情報を示したメタデータ
8. C -> S `RowDescription`
    - 結果の列の名前や型情報などのメタデータ
9. C -> S `DataRow`
    - 簡易問い合わせと同様
10. C -> S `ReadyForQuery`
    - 簡易問い合わせと同様

ここで注意が必要なのは、簡易問い合わせよりかなりステートフルなものとなっています。
`Parse` 発行から `ReadyForQuery` が来るまでは、他の用途での問い合わせを受け付けない設計になっており、意図しない状態での意図しないメッセージは受け付けられません。
状態管理をクライアント側でする必要があり、例えばクライアント都合でのエラーにより処理を途中で中断する場合も、最後の `ReadyForQuery` を受信してあげる必要があります。

### 認証

PostgreSQL はさまざまな認証方法をサポートしています。
その中でも、パスワードベースの認証や基本的な認証には下記があります。

- `trust` ... 認証しない
- `plain` ... パスワードを平文で送信するもの
- `md5` ... パスワードをハッシュ化して送信するもの。少し前まではこれがデフォルト。
- `scram-sha-256` ... SHA256 にソルトを絡めた認証方式

#### SCRAM-SHA-256

Salted Challenge Response Authentication Mechanism (SCRAM) の SHA-256 ハッシュ関数を利用する方式です。
英語版 Wikipedia によると、

> Although all clients and servers have to support the SHA-1 hashing algorithm, SCRAM is, unlike CRAM-MD5 or DIGEST-MD5, independent from the underlying hash function.[4] Any hash function defined by the IANA can be used instead.[5] As mentioned in the Motivation section, SCRAM uses the PBKDF2 mechanism, which increases the strength against brute-force attacks, when a data leak has happened on the server. Let H be the selected hash function, given by the name of the algorithm advertised by the server and chosen by the client. 'SCRAM-SHA-1' for instance, uses SHA-1 as hash function. 
> 
> DeepL訳 すべてのクライアントとサーバーがSHA-1ハッシュアルゴリズムをサポートする必要があるものの、SCRAMはCRAM-MD5やDIGEST-MD5とは異なり、基盤となるハッシュ関数に依存しない[4]。代わりにIANAによって定義された任意のハッシュ関数を使用できる。動機付けのセクションで述べたように、SCRAMはPBKDF2メカニズムを採用しており、サーバー上でデータ漏洩が発生した場合でも、ブルートフォース攻撃に対する耐性を高めます。Hを、サーバーが広告するアルゴリズム名に基づきクライアントが選択するハッシュ関数とします。例えば「SCRAM-SHA-1」はSHA-1をハッシュ関数として使用します。

詳しく知りたい方は、SCRAM 自体の仕様は RFC 5802 、 SCRAM-SHA-256 は RFC 7677 をご参照下さい。

PostgreSQL では以下のようなフローになります。(概要のため正確性はありません。詳細は RFC をご参照下さい)

1. S -> C ... `AuthenticationSASL`
    - SCRAM-SHA-256 認証が SASL 機構に含まれている。かもしれない。
2. C -> S ... `SASLInitialResponse`
    - SCRAM-SHA-256 認証で認証フローを続行する場合、
        - `SCRAM-SHA-256` である旨
        - `n,,n=user,r=rOprNGfwEbeRWgbNEkqO` のような形式をデータとする (RFC の例より引用)
            - `n` ... 認証するユーザー名
            - `r` ... クライアントの nonce
3. S -> C ... `AuthenticationSASLContinue`
    - 以下のようなデータを持っている (RFC の例より引用)
    - `r=rOprNGfwEbeRWgbNEkqO%hvYDpWUa2RaTCAfuxFIlj)hNlF$k0,s=W22ZaJ0SNY7soEsUEjb6gQ==,i=4096`
        - `r` ... サーバーの nonce。クライアントの noce で始まるため、検証が可能 (すべき)
        - `s` ... ソルト
        - `i` ... イテレーション
4. C -> S ... `SASLResponse`
    - 応答メッセージ生成手順の概略
        1. 検証したいパスワードおよびソルトとイテレーションで PBKDF2 (SHA256) で派生鍵 (derive key) を生成
        2. 先の派生鍵と `"Client Key"` でクライアントキーを生成。サーバーキーは同様に `"Server Key"` で生成できる。
        3. クライアントキーを SHA256 でストアドキーを生成
        4. 以下を `,` 区切りで連結したものと先のストアドキーで HMAC にかける
            - `SASLInitialResponse` のデータの `n,,` 以降
            - `AuthenticationSASLContinue` で受け取ったデータ
            - `c=biws,r=<サーバーの nonce>`
        5. クライアントキーと先程計算された HMAC の値を XOR することで、クライアントプルーフを生成
        6. `c=biws,r=<サーバーの nonce>,p=<クライアントプルーフ>` を送信
    - 送信するデータの例
        - `c=biws,r=rOprNGfwEbeRWgbNEkqO%hvYDpWUa2RaTCAfuxFIlj)hNlF$k0,p=dHzbZapWIk4jUhN+Ute9ytag9zjfMHgsqmmiz7AndVQ=` (RFC の例より引用)
5. S -> C ... `AuthenticationSASLFinal`
    - 以下のようなデータを持っている (RFC の例より引用)
    - `v=6rriTRBi23WpRR/wtup+mMhUZUn/dB5nLTJRsjl95G4=`
        - `v` ... クライアントプルーフ同様にサーバーキーから生成されたものと一致するか検証が可能 (すべき)

## 今後の展望

WASM / WASI で動作するように切り出して、JavaScript縛りを無くしたい。
