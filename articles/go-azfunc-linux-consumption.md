---
title: "鉱夫さんをやつけたいのでGoでAzure Functions Linux従量課金プランをためす"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "azure", "azurefunctions"]
published: true
---

一月ほど前にGitHub Actionsで仮想通貨をホリホリするPull Requestを仕掛けられました。

https://twitter.com/yskszk63/status/1380778347437821960

Pull Requestが来ると、Pull RequestでトリガーされるGitHub Actionsのジョブが20並列くらいで延々と計算を実行されてしまいます。
気づいたらすぐにキャンセルしますがGitHubのFreeプランは上限が2,000分/月なので、そこそこ悔しい思いをしてしまします。

嫌なので、そういうGitHub ActionsのジョブをキャンセルするGitHub Appを作りました。

https://github.com/yskszk63/cancel-workflow-run

:::message
既にGitHubにより色々と対策されている様なので、このGitHub App自体は役に立たないかもしれません。
:::

# 作戦

GitHub Actionsのジョブ起動をフックし、当該ジョブがPull Requestにより新規作成された定義である場合にキャンセルできれば私のケースの場合、十分です。

最初はバックエンドをPython + HerokuのFreeプランで作成していました。
ただ、GitHubのWebHookは10秒以内にレスポンスを返さないとTimed Outとして記録されてしまいます。
HerkuのFreeプランはすぐに寝てしまい、WebHookがある時はだいたい寝ているので、Timed Outで履歴が残ってしまい気持ち悪いです。

なので、お金をかけずにすぐに起きるバックエンドを作りたいと思い Go + Azure Functionsを試しました。

# Azure Functions Linux従量課金プランとは

AzureのFaaSです。
Windows + C#のイメージがあるかもしれないですが、Linux上で様々な言語で動かすこともできます。
私もWindowsは所持しておらず、C#も書けないのですが、動くものを作れました。

当初はAzure Functionsの従量課金はWindowsのみでしたが、Linux従量課金プランも[GA](https://azure.microsoft.com/ja-jp/updates/azure-function-consumption-plan-for-linux-is-now-available/)しています。
詳細な仕組みは[こちら](https://qiita.com/TsuyoshiUshio@github/items/4b590287d346dd7cfd88)の記事がとても参考になりました。

Azure Functionsは[100 万回の要求と 400,000 GB 秒のリソース使用量](https://azure.microsoft.com/ja-jp/free/free-account-faq/)はいつでも無料とのことです。

:::message
Azure FunctionsのLinux従量課金プランはStorageAccountの紐付けが必要なため、その費用が少し発生するかもしれません。
私は[12ヶ月無料](https://azure.microsoft.com/ja-jp/free/)のサブスクリプションを使っているため、少し様子を見たいと思います。
:::

カスタムハンドラという仕組みが少し前に[GA](https://azure.microsoft.com/ja-jp/updates/azure-functions-custom-handlers-are-now-generally-available/)しており、
C# / JavaScript / Java / Python などではなくとも、言語に縛られない関数の実装が可能になりました。

# カスタムハンドラーとは

まずは、Azure Functionsのおさらいです。
Azure Functionsとは、トリガーをもとに関数をキックするしくみです。
![](https://storage.googleapis.com/zenn-user-upload/0vqf1emya8yoa3srg7f06a3oe65x)
トリガーの他に付随するインプット、アウトプットを関数へバインドさせることができます。

- トリガー
    - HTTPトリガー ... HTTPリクエストをもとに関数を起動。HTTPリクエストのURL、メソッド、ヘッダや本体が関数の引数としてバインドされる
    - タイマートリガー ... cron式風の時刻指定をもとに定期的に関数を起動。
    - EventGridトリガー ... EventGrid(Pub/Subのサービス)からのイベント配信をもとに関数を起動。
    - など
- インプット
    - Blob ... AzureStorage中の指定されたBlobを関数の引数としてバインド。
    - CosmosDB ... CosmosDBの指定されたレコードを関数の引数としてバインド。
    - など
- アウトプット
    - HTTP ... HTTPトリガーのリクエストに対するレスポンスを関数の戻り値としてバインド。
    - EventGrid ... 関数の戻り値をEventGridへのインベント発行としてバインド。
    - Blob ... 関数の戻り値をAzureStorageの指定されたBlobへ格納。
    - CosmosDB ... 関数の戻り値をCosmosDBの指定されたレコードへ格納。
    - など

詳細は[この辺](https://docs.microsoft.com/ja-jp/azure/azure-functions/functions-bindings-http-webhook-trigger?tabs=javascript)を見ていただければ雰囲気をつかめるかと思います。

これらのバインディングを捌いてくれるのがサービスの一部である[Functions Host](https://github.com/Azure/azure-functions-host)です。
Functions Hostからの関数呼び出しは、対応している言語毎に特化した仕組みが用意されていました。
カスタムハンドラーは汎用的に使えるよう、HTTPリクエスト/レスポンスを使うことで、様々な言語でも関数を実装できるようになっています。
![](https://storage.googleapis.com/zenn-user-upload/s1h6acx5jvowf9w23zr9zycdjede)
更に、扱うトリガーがHTTPのみの場合は、HTTPリクエスト/レスポンスをバイパスさせることができ、普通のAPIサーバを実装する気分で関数を実装することができます。

:::message
[こちら](https://docs.microsoft.com/ja-jp/azure/azure-functions/create-first-function-vs-code-other?tabs=go%2Clinux)にカスタムハンドラーのチュートリアルがあります。例としてGoとRustが記載されていますが、最近のMicrosoftはすごいですね。
:::

# ローカルでの開発

[Azure Functions Core Tools](https://github.com/Azure/azure-functions-core-tools)でローカルで動作させることができます。サービスをバインドさせていると、少し工夫が必要な場面がありますが、いちいちデプロイせずとも動作確認ができるので楽ですね。

# 実装する上で注意したこと、はまったこと

## CGO

単純に`go build`を実行すると、動的ライブラリがリンクしてしまいます。

```bash
$ go build -o app
$ ldd app
	linux-vdso.so.1 (0x00007ffc733ca000)
	libpthread.so.0 => /usr/lib/libpthread.so.0 (0x00007f7e2ae79000)
	libc.so.6 => /usr/lib/libc.so.6 (0x00007f7e2acac000)
	/lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f7e2aeb6000)
$ file app
app: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, Go BuildID=F3hqAn1f_eQ3ixd2paU0/L-Z6juymmivPuIHPXFPg/lYGTqgl_U_LogSimNKOf/n5eybwLJ6tJ27j10m8PQ, not stripped
```

私の環境はArch Linuxなのでglibcが少し新し目です。そのため、Azure Functionsへデプロイして起動させると、コケてしまいます。
そのため、下記のようにシングルバイナリにコンパイルされるように指定しています。

```bash
$ CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -o app
$ ldd app
	not a dynamic executable
$ file app
app: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=e0bWgJUij4gee1Pz8_Px/OlgtQa0T9BqCawWYIm9z/fl_6voTAytl1LRrRcqxF/5CcVo8_9lNEeDTq1Q6E6, not stripped
```

Goはシングルバイナリでコンパイルされるものだと思っていました。（私はGoを書き始めて日が浅いです。）
なぜ動的ライブラリがリンクしてしまうのか、詳しい人教えていただけると幸いです。。

## 飛び立つのを見届ける

Azure Functionsへデプロイ後は、HTTPのリッスンをするまでにコケてしまうとログがどこかへ行ってしまいます。
そうなると、何が起きたのか追跡が辛いです。
例えば、プロセス起動時に環境変数が不足しているなどで、Fail Fastするような実装をすると、何が起きたのか分からなくなり暫くしてから気づくみたいになってしまいます。
HTTPのリッスンをするまでなるべく止めない実装にしたほうが良さそうです。

## ログ

ログ出力はふた通りの方法があります。
- ①標準出力へ出力
- ②関数の戻り値としてHTTPレスポンスへ埋め込む

②の方法は関数の呼び出しにひも付き追跡がとても容易です。ただ、HTTPのレスポンスが返せない状況になると、ログがすべてどこかへ消えてしまします。
また、HTTPリクエスト/レスポンスをバイパスする方式の場合、この方法でログ出力することができません。

私は[こちら](https://dev.to/mattiasfjellstrom/azure-functions-custom-handlers-in-go-logging-31bp)の記事に倣い、構造化されたログを標準出力へ垂れ流す方式にしました。
ただし、ログのクエリが複雑になってしまします。この辺は今後に期待したいです。

# デプロイ

Linux従量課金プランの場合、Kudoが無いようです。
そのため他のプランに比べデプロイに色々制約があります。

アプリケーション設定(環境変数)の`WEBSITE_RUN_FROM_PACKAGE`にデプロイしたいディレクトリ構造一式を格納したzipファイルのURLを指定し、
`/admin/host/synctriggers?<_masterキー>`を叩くことでAzure Functionsへ変更を反映させることができます。

私は`WEBSITE_RUN_FROM_PACKAGE`へGitHubのリリースを指定し、GitHub Actionsで`/admin/host/synctriggers?<_masterキー>`を叩くことでPush毎にテスト環境へデプロイされるようにしました。

:::message
当初、[Azure/functions-action](https://github.com/Azure/functions-action)を試していましたが、何故かpackageがsquashfsとして固められてしまい上手く動きませんでした。
[このIssue](https://github.com/Azure/functions-action/issues/39)を見ると、どうやらLinux従量課金プランで少し動きが怪しいようです。解消を期待したいです。
:::
