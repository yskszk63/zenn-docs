---
title: "git archive で作成した zip ファイルを AWS CodePipeline のビルドのソースとして使う場合はDOSタイムスタン"
emoji: "⏳"
type: "tech"
topics: ["aws", "codepipeline", "Makefile", "zip"]
published: false
---
# git archive で作成した zip ファイルを AWS CodePipeline のビルドのソースとして使う場合はDOSタイムスタンプをUTCで作成すべき

## 要約

- zip ファイルには DOS タイムスタンプと拡張タイムスタンプがある
- DOS タイムスタンプはタイムゾーン無しのタイムスタンプ
- 拡張タイムスタンプはUTCで格納されたタイムスタンプ
- AWS CodePipeline は zip ファイル展開時にDOSタイムスタンプでmtimeを設定するようだ
- AWS CodePipeline のソースとして利用する zip ファイルは `TZ=UTC` として作成すべき

## 何が起きた

以下のような構成のプロジェクトで `make` が意図した通りに動かなかった。

- AWS CodePipeline で ビルドをしている
- ビルドのソースは S3 として .zip ファイルを使っている
- .zip は `git archive` で生成している
- 具体的なビルドタスクは `Makefile` に記述している

→ コードリポジトリから AWS 環境に直接アクセスできない、特殊な環境です

この構成で `make` を実行すると、

> make: Warning: File 'Makefile' has modification time 30040 s in the future

のような警告が発せられた。`make` は mtime に依存して処理を行なうため、タスクの記述内容によっては意図した通りに動かなかった。
時計の進み方から、どこかでタイムゾーンの解釈の違いによる箇所があると思われる。調査を進めたところ、 CodePipeline の Zip 展開で DOS タイムスタンプから mtime を設定しているようにうかがえた。

## 検証

ソースとして利用していた .zip ファイルのタイムスタンプを `zipinfo` で調べたところ、

```bash
$ zipinfo -v src.zip
...
  file last modified on (DOS date/time):          2026 Oct 24 21:28:58
  file last modified on (UT extra field modtime): 2026 Oct 24 21:28:57 local
  file last modified on (UT extra field modtime): 2026 Oct 24 12:28:57 UTC
...
```

のように、 `make` で警告されていた時刻と一致する時刻が `DOS date/time` に記録されていることが分かった。
.zip ファイルの DOS タイムスタンプの精度は2秒単位のため、このケースでは拡張タイムスタンプとも見わけることができた。

CodePipeline の Zip 展開に DOS タイムスタンプが使われた結果、ビルド時点で `make` が警告をしていると推測し、実験を行なった。

まず、実験のビルド用に架空のビルドを定義した `Makefile` を用意した。

```Makefile
.PHONY: release

release: build-settings.txt
	touch '$(shell cat $<)'

build-settings.txt: VERSION.txt
	echo 'app-$(shell cat $<).jar' > $@
```

- `VERSION.txt` をソースとしてビルド設定(`build-settings.txt`)を出力する。
- `build-settings.txt` にはビルドされるファイル名が記述されている。また、このファイルはバージョン管理されている。

:::message
`build-settings.txt` をバージョン管理の対象とするな。という話もあるかもしれませんが、そういう状況だった。と捉えて下さい。
:::

このビルドを AWS CodePipeline で実行する。

```typescript:cdk-app.ts
const buildStage = pipeline.addStage({
  stageName: "Build",
});
buildStage.addAction(new cpa.CommandsAction({
  actionName: "build",
  input: artifact,
  commands: [
    "date", // DEBUG
    "stat *", // DEBUG
    "echo v1 > VERSION.txt",
    "make",
    "ls app-*", // DEBUG
  ],
}));
```

期待値は、このビルド実行により `app-v1.jar` が生成されること。

`TZ` 環境変数に対するDOS タイムスタンプの扱いは、 `git archive` も `zip` (info-zip) も同じように見えたため、この実験では `zip` を利用している。

## 結果

### `TZ=Asia/Tokyo` (JST) で Zip をアーカイブ

`TZ=Asia/Tokyo` (JST) で Zip をアーカイブしてビルドを実行したところ以下のような結果となった。

:::details CodePipeline のログ
```
...
[Container] 2026/10/24 13:08:08.884408 Entering phase BUILD
[Container] 2026/10/24 13:08:08.929758 Running command date
Sat Oct 24 01:08:08 PM UTC 2026

[Container] 2026/10/24 13:08:08.936441 Running command stat *
  File: build-settings.txt
  Size: 20          Blocks: 8          IO Block: 4096   regular file
Device: 10302h/66306d   Inode: 265168032   Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-10-24 21:28:58.000000000 +0000
Modify: 2026-10-24 21:28:58.000000000 +0000
Change: 2026-10-24 13:08:08.157463490 +0000
 Birth: 2026-10-24 13:08:08.157463490 +0000
  File: Makefile
  Size: 140         Blocks: 8          IO Block: 4096   regular file
Device: 10302h/66306d   Inode: 265161823   Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-10-24 21:28:50.000000000 +0000
Modify: 2026-10-24 21:28:50.000000000 +0000
Change: 2026-10-24 13:08:08.153463483 +0000
 Birth: 2026-10-24 13:08:08.153463483 +0000
  File: VERSION.txt
  Size: 12          Blocks: 8          IO Block: 4096   regular file
Device: 10302h/66306d   Inode: 265161822   Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-10-24 21:28:02.000000000 +0000
Modify: 2026-10-24 21:28:02.000000000 +0000
Change: 2026-10-24 13:08:08.153463483 +0000
 Birth: 2026-10-24 13:08:08.153463483 +0000

[Container] 2026/10/24 13:08:08.984452 Running command echo v1 > VERSION.txt

[Container] 2026/10/24 13:08:08.991316 Running command make
make: Warning: File 'Makefile' has modification time 30040 s in the future
touch 'app-pre-release.jar'
make: warning:  Clock skew detected.  Your build may be incomplete.

[Container] 2026/10/24 13:08:09.722678 Running command ls app-*
app-pre-release.jar

[Container] 2026/10/24 13:08:09.766192 Phase complete: BUILD State: SUCCEEDED
...

```

:::

> [Container] 2026/10/24 13:08:08.991316 Running command make
> make: Warning: File 'Makefile' has modification time 30040 s in the future
> touch 'app-pre-release.jar'
> make: warning:  Clock skew detected.  Your build may be incomplete.

と警告があり、生成されたファイルは `app-pre-release.jar` で期待通りではなかった。

### `TZ=UTC` で Zip をアーカイブ

`TZ=UTC` で Zip をアーカイブしてビルドを実行したところ以下のような結果となった。

:::details CodePipeline のログ
```
...
[Container] 2026/10/24 13:12:01.432375 Entering phase BUILD
[Container] 2026/10/24 13:12:01.487686 Running command date
Sat Oct 24 01:12:01 PM UTC 2026

[Container] 2026/10/24 13:12:01.494497 Running command stat *
  File: build-settings.txt
  Size: 20          Blocks: 8          IO Block: 4096   regular file
Device: 10302h/66306d   Inode: 265161822   Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-10-24 12:28:58.000000000 +0000
Modify: 2026-10-24 12:28:58.000000000 +0000
Change: 2026-10-24 13:12:00.657709319 +0000
 Birth: 2026-10-24 13:12:00.657709319 +0000
  File: Makefile
  Size: 140         Blocks: 8          IO Block: 4096   regular file
Device: 10302h/66306d   Inode: 265161821   Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-10-24 12:28:50.000000000 +0000
Modify: 2026-10-24 12:28:50.000000000 +0000
Change: 2026-10-24 13:12:00.657709319 +0000
 Birth: 2026-10-24 13:12:00.657709319 +0000
  File: VERSION.txt
  Size: 12          Blocks: 8          IO Block: 4096   regular file
Device: 10302h/66306d   Inode: 265161820   Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-10-24 12:28:02.000000000 +0000
Modify: 2026-10-24 12:28:02.000000000 +0000
Change: 2026-10-24 13:12:00.657709319 +0000
 Birth: 2026-10-24 13:12:00.657709319 +0000

[Container] 2026/10/24 13:12:01.537186 Running command echo v1 > VERSION.txt

[Container] 2026/10/24 13:12:01.546770 Running command make
echo 'app-v1.jar' > build-settings.txt
touch 'app-v1.jar'

[Container] 2026/10/24 13:12:02.445084 Running command ls app-*
app-v1.jar

[Container] 2026/10/24 13:12:02.493924 Phase complete: BUILD State: SUCCEEDED
...
```
:::

> [Container] 2026/10/24 13:12:01.546770 Running command make
> echo 'app-v1.jar' > build-settings.txt
> touch 'app-v1.jar'

`app-v1.jar` が生成され、期待通りの結果となった。

## まとめ

- zip ファイル の DOS タイムスタンプにはタイムゾーン情報がない
- ビルド環境に zip ファイル のソースが展開された際、DOS タイムスタンプに由来すると考えられる mtime が設定されていた
- make の誤動作を避けるため、CodePipeline のソースとして利用する zip ファイル は TZ=UTC で生成するのがよさそう

なお、今回検証で利用したコードは下記に置いています。

https://github.com/yskszk63/codepipeline-zip-dostime
