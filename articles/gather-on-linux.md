---
title: "nspawn と mstack と Wine を使って Linux で Gather の Windows アプリを動かす"
emoji: "🍷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nspawn", "mstack", "wine"]
published: true
---
## 背景

今さらながら職場で [gather](https://www.gather.town/ja/) の試用を始めてます。
ブラウザ版以外に、 Windows / Mac 向けのアプリが提供されています。インストール版を使いたいが、私が仕事で使っている PC は Linux なので困りました。
Linux 版が無いなら、 Wine でいいじゃないの。
ということで、試してみました。

## 使った技術

環境を汚したくないので、コンテナで動かすことは必須と考え、下記のような構成としました。

- 1. nspawn (systemd)
- 2. mstack (systemd)
- 3. Wine

### 1. nspawn (systemd)

systemd に付属している、名前空間に閉じ込めたコンテナを動作させるためのもの。
そこそこ古くからあり、私個人は Steam を動かすために使ってました。これも環境を汚したくなかったためです。
詳細は [ArchWiki](https://wiki.archlinux.jp/index.php/Systemd-nspawn) に譲るとして
私のざっくりとした認識を以下に記述します。

Docker や Podman との違いは、環境を隔離する事のみにフォーカスしている。という点です。
Docker 等は、ポータビリティにフォーカスしているため、ストレージは基本イミュータブルです。
nspawn のストレージはミュータブルで、どちらかというと、仮想マシンの使い勝手に近いです。
lxc なんかが親戚ですね。
nspawn は、 systemd に付属していて、最近の Linux では割とセットアップ不要で簡単に動作させることができるため、
昔から愛用しています。

### 2. mstack (systemd)

つい先日リリースされた systemd 260 で同梱された機能です。
Docker などで使われている overlayfs のレイヤーを簡単に記述できマウントできます。
最近リリースされたばかりなので、日本語の記事が現時点ではあまりないです。

例えば、下記のようなディレクトリ構造を

```bash
$ fd .
test.mstack/
test.mstack/layer@10/
test.mstack/layer@10/a.txt
test.mstack/layer@20/
test.mstack/layer@20/b.txt
test.mstack/rw/
```

`sudo mount -t mstack test.mstack/ dest` とすると、

```bash
$ ls dest
a.txt  b.txt
```

このようになります。
`<name>.mstack/` というディレクトリ以下に決められた形式のディレクトリを作成することで、それぞれが `overlayfs` のレイヤーとなります。

そのマウントポイントに書き込むと...

```bash
$ sudo touch dest/c.txt
$ fd . test.mstack
test.mstack/layer@10/
test.mstack/layer@10/a.txt
test.mstack/layer@20/
test.mstack/layer@20/b.txt
test.mstack/rw/
test.mstack/rw/data/
test.mstack/rw/data/c.txt
test.mstack/rw/work/
test.mstack/rw/work/index/
test.mstack/rw/work/work/
```

のように `test.mstack/rw` 以下に差分が書き込まれます。(`data/` や `work/` は `overlayfs` 由来)
`test.mstack/rw/data/` を `test.mstack/layer@30` として移動させると、"コミット" したように扱うことができます。
ディレクトリベースでレイヤーを定義できるので、柔軟にかつ簡単に `overlayfs` を扱うことができて、とても嬉しいものとなってます。

nspawn との連携も充実していて、 `systemd-nspawn --mstack=<name>.mstack` でコンテナのルートディレクトリとしてマウントすることができます。

### 3. Wine

いわずと知れた、 Linux で Windows アプリケーションを動作させるためのソフトウェアです。
Windows の MinGW とは逆の事をやっているイメージでしょうか。
Windows の Win32 API 等を Linux 上でエミュレーションして、 Windows ソフトウェアを動作させることができます。

Arch Linux に Wine を導入すると、結構環境が汚れてしまうので、私は昔から nspawn 上で動作させています。

(余談ですが、Proton が出る前までは、私が Steam でゲームを買う目的は Wine で動作させることだった時期がありました。)

## どのように動かしたか

### 準備

mstack で、下記のようにレイヤーを定義しました。

- `layer@10` ... `pacstrap base` のみしたもの
- `layer@20` ... `multilib` を有効化し、 `wine` や必要となるソフトウェアをインストール
    - wine
    - vulkan-icd-loader
    - libglvnd
    - vulkan-intel
    - winetricks
    - wine-mono
    - pipewire
    - pipewire-pulse
    - lib32-pipewire
    - lib32-libpulse
    - noto-fonts-cjk
    - noto-fonts-emoji
    - noto-fonts-extra
- `layer@30` ... `id -u` と一致させたユーザーの作成や、 `winetricks dxvk` の実行、 Gather の Windows 版をダウンロードして展開

なお、 Gather のインストラーは、PowerShell を使って既にアプリケーションが起動している場合の処理が入っています。
が、 Wine の powershell.exe は常に 0 (成功) を返すだけのスタブになっており、 1 (失敗) を返さないと、起動中と判定されてしまうため、
インストーラーを実行することができませんでした。

ちゃっぴ に相談したところ、7zip で固められているだけだよ。と教わったので、7zip で展開、配置をしました。

### 起動

下記のように起動しました。

`sudo systemd-nspawn --mstack="$mstack" --user=user --bind "$data:/home/user/.wine/" -EDISPLAY=$DISPLAY -EXDG_RUNTIME_DIR=/run/user/$(id -u) --bind /run/user/$(id -u) --bind /dev/dri/ --property='DeviceAllow=char-drm rwm' --bind /tmp/.X11-unix --bind $XAUTHORITY -EXAUTHORITY=$XAUTHORITY -EWINEDLLOVERRIDES="libglesv2.dll=d" wine /home/user/app/GatherV2.exe --force-device-scale-factor=1.5`

systemd-nspawn に指定したオプションと理由は下記のとおりです

- `--mstack="$mstack"` ... 先に作成した `.mstack/` をルートとして指定
- `--user=user` ... 作業用のユーザー
- `--bind "$data:/home/user/.wine/"` ... `.mstack/rw/` を配置しない ro イメージとして、 Wine の書き込み先はバインドマウントした。(Docker の Volume 相当)
- `-EDISPLAY=$DISPLAY` ... Wayland だとうまく動かせなかったため、 XWayland 経由とした
- `--bind /tmp/.X11-unix` ... X 用
- `--bind $XAUTHORITY`` ... X 用
- `-EXAUTHORITY=$XAUTHORITY`` ... X 用
- `-EXDG_RUNTIME_DIR=/run/user/$(id -u)` ... pipewire 等用のホストの XDG_RUNTIME_DIR をコンテナに公開
- `--bind /run/user/$(id -u)`
- `--bind /dev/dri/` ... GPU をコンテナに公開
- `--property='DeviceAllow=char-drm rwm'` ... /dev/dri を公開しても、これをしないと権限が足りなかったため
- `-EWINEDLLOVERRIDES="libglesv2.dll=d"` ... dxvk を導入して、 libglesv2 の優先度を変えることで画面表示ができた

## 感想

Electron 実装っぽいので、ブラウザ版と変わらなかった。
