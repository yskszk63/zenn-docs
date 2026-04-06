---
title: "FreeRDP で Entra ID (AAD) のユーザで接続する"
emoji: "😸"
type: "tech"
topics: ["freerdp"]
published: false
---

この記事では、

- FreeRDP の AAD 認証を外部化し
- D-Bus 経由でトークン取得する Broker を実装する

という構成を紹介します

## 背景

RDP で Windows に接続する際に、接続するユーザが Entra ID (旧 Azure Active Directory) 管理のユーザの場合、 OAuth2 認証が必要となります。
Linux でのRDP 接続によく利用される FreeRDP では、まだこのあたりのサポートがまだ薄いため、接続には苦労します。
→ v3 で徐々に実装・改善されている状況のようです

```bash
$ xfreerdp /v:w11 /sec:aad
[10:57:54:261] [34580:00008718] [INFO][com.freerdp.client.x11] - [xf_pre_connect]: No user name set. - Using login name: yskszk63
[10:57:54:262] [34580:00008718] [WARN][com.freerdp.client.x11] - [load_map_from_xkbfile]:     : keycode: 0x08 -> no RDP scancode foun
d
[10:57:54:262] [34580:00008718] [WARN][com.freerdp.client.x11] - [load_map_from_xkbfile]: ZEHA: keycode: 0x5d -> no RDP scancode foun
d
[10:57:54:320] [34580:00008718] [INFO][com.freerdp.codec] - [libavcodec_init]: Using VAAPI for accelerated H264 decoding
[10:57:54:342] [34580:00008718] [INFO][com.freerdp.codec] - [libavcodec_init]: Using VAAPI for accelerated H264 decoding
[10:57:54:343] [34580:00008718] [WARN][com.freerdp.core.rdp] - [log_build_warn][0x56455fb5f800]: ************************************
*************
[10:57:54:343] [34580:00008718] [WARN][com.freerdp.core.rdp] - [log_build_warn][0x56455fb5f800]: This build is using [experimental] b
uild options:
[10:57:54:343] [34580:00008718] [WARN][com.freerdp.core.rdp] - [log_build_warn][0x56455fb5f800]: * 'WITH_VAAPI=ON'
[10:57:54:343] [34580:00008718] [WARN][com.freerdp.core.rdp] - [log_build_warn][0x56455fb5f800]: *
[10:57:54:343] [34580:00008718] [WARN][com.freerdp.core.rdp] - [log_build_warn][0x56455fb5f800]: [experimental] build options might c
rash the application
[10:57:54:343] [34580:00008718] [WARN][com.freerdp.core.rdp] - [log_build_warn][0x56455fb5f800]: ************************************
*************
[10:57:54:409] [34580:00008718] [WARN][com.freerdp.crypto] - [verify_cb]: Certificate verification failure 'unable to get local issue
r certificate (20)' at stack position 0
[10:57:54:409] [34580:00008718] [WARN][com.freerdp.crypto] - [verify_cb]: DC = 00000000-1111-2222-3333-444444444444, CN = 55555555-6666-7777-8888-999999999999
Browse to: https://login.microsoftonline.com/common/oauth2/v2.0/authorize?client_id=a85cf173-4192-42f8-81fa-777a763e6e2c&response_typ
e=code&scope=ms-device-service%3A%2F%2Ftermsrv.wvd.microsoft.com%2Fname%2Fw11%2Fuser_impersonation&redirect_uri=https%3A%2F%2Flogin.m
icrosoftonline.com%2Fcommon%2Foauth2%2Fnativeclient
Paste redirect URL here: 
```

→ Entra ID 管理のユーザの場合は、 OAuth2 の認可コードフローを途中まで進めてリダイレクトした URL を貼りつける必要がある

これは、さすがに実用的では無いため、この面倒な状況を解決しておきたい。と思い、まずは libfreerdp を利用したクライアントを実装しました。
が、やはり実用的にするにはかなりボリュームになってしまうため、断念しました。
→ クリップボードの同期や、動的リサイズ、 動的チャネル (drdynvc)、単純な画面描画 (HiDPI 対応やできれば GPU 使いたい)、 Wayland でのキーボードのよこどり (keyboard_shortcuts_inhibitor) 、ほか...

OAuth2 なので、同じスコープで認証できる OAuth2 のアプリケーションを作成すれば良いのでは。と考えましたが、スコープに接続先デバイス名が埋め込まれるため、デバイス毎に異なるスコープ定義の管理も別途必要となり現実的ではありません。
そのため `common` テナントに定義されている OAuth2 アプリケーションを利用するのが良いのですが、 redirect_uri がネイティブクライアント用のため、 WEB でさくっと作成することもできません。

八方ふさがりです。

そうした時にFreeRDP のコードを眺めていると、 `WITH_SSO_MIB` というビルドオプション (デフォルト: OFF) を見つけました。

## WITH_SSO_MIB

これは、 [microsoft-identity-broker](https://learn.microsoft.com/ja-jp/entra/identity/devices/sso-linux) から SSO 用にトークンを取り出すライブラリ [sso-mib](https://github.com/siemens/sso-mib)  と連携して、 Windows で実現されている SSO と同等の仕組みを FreeRDP で実現するための仕組みのようです。
microsoft-identity-broker は D-Bus 経由でクライアントプログラムと IPC をして、トークンを授受するもので、 sso-mib は、その D-Bus のインターフェースを叩くライブラリです。
つまり、このビルドオプションは FreeRDP が microsoft-identity-broker からトークンを取得することで、 Entra ID が必要なケースでの RDP を容易にする機能を有効にするためのものです。

ただ、問題が、 この microsoft-identity-broker に SSO 用の情報を格納するプログラムは別途必要で、たとえば Intune 用の Linux クライアントプログラム群が必要となります。
RDP したいだけなのに、そんなおおがかりな仕組みは入れたくない。

## rdpmib

microsoft-identity-broker は D-Bus を利用しているため、同じインタフェースのサービスプログラムを実装することにしました。
それがこちら。

https://github.com/yskszk63/rdpmib

microsoft-identity-broker の D-Bus I/F の `getAccounts` と `acquireTokenSilently` の同じシグニチャのメソッドを実装しました。
実際のトークンの取得は、 `acquireTokenSilently` でおこなわれます。このメソッド呼び出し中 に WebView でログインページ (KMSI用) でのログインと OAuth2 の認可コード取得用のページを表示、よこどりすることで、RDP 接続用のトークンを取得します。
FreeRDP はそのトークンを利用することで、無事、 Entra ID ユーザーの Windows PC にログインできるのでうまあじ。

## 使い方

まだ、課題は山積なのですが、今時点 (2026/4/6) で使う場合の手順です。
なお、筆者の PC は Arch Linux なので、それを前提とします。

### 1. ビルド済パッケージを取得

個人的な PKGBUILD を管理しているリポジトリ
https://github.com/yskszk63/pkgbuilds
のリリース
https://github.com/yskszk63/pkgbuilds/releases/latest
から、下記ファイルを取得します。

- freerdp-sso-mib-*-x86_64.pkg.tar.zst
- libjwt-*-x86_64.pkg.tar.zst 
- rdpmib-*-x86_64.pkg.tar.zst 
- sso-mib-*-x86_64.pkg.tar.zst 

→ 当該ページには `-debug-` も含めていますが、ここでは不要です。

### 2. インストール

`pacman -U *.pkg.tar.zst`

→ 雑な例ですが、よしなに

### 3. rdpmib

`rdpmib` を実行します。
トークン取得時に WebView を使うためデスクトップ環境で立ち上げる必要があります。

### 4. xfreerdp

`xfreerdp /v:<host> /sec:aad` で xfreerdp を立ち上げます。 D-Bus 経由でトークンの取得ができれば、無事 RDP することができます。
なお、 `<host>` は Entra ID に登録されているデバイスの名前と一致する必要があります。
→ 単純名で名前解決できる必要がある

## rdpmib の今後の展望

- XDG の AutoStart で起動
- zbus に移行したい
- シングルバイナリ化

また、 Remmina に `/sec:aad` 相当の指定をする口が今のところ無いため、 Remmina では利用できません。それを解決したい。
