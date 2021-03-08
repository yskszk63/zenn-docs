---
title: "RustのWindows開発環境 on Wine on Nspawn (準備編)"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust", "nspawn", "wine"]
published: true
---
Rustの開発環境をWineが動いているsystemd-nspawnコンテナ上に用意する手順を書きます。
※Wineを使うため、100%の動作は保証できません。念の為。

久々にsystemd-nspawnを使ったら、分からなくなっていたので頭の整理のためにまとめます。

# 対象読者

- RustのWindows向けのコードを書きたい人
- にもかかわらず、Windows PCを所持していない人
- なので、ひとまず手元のLinux PCでしのぎたい人

# 執筆時の環境

```
$ uname -a
Linux a285 5.11.2-arch1-1 #1 SMP PREEMPT Fri, 26 Feb 2021 18:26:41 +0000 x86_64 GNU/Linux
$ systemctl --version
systemd 247 (247.3-1-arch)
+PAM +AUDIT -SELINUX -IMA -APPARMOR +SMACK -SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +ZSTD +SECCOMP +BLKID +ELFUTILS +KMOD +IDN2 -IDN +PCRE2 default-hierarchy=hybrid
$ mount |grep -e' / '|cut -d' ' -f-5
/dev/sda2 on / type btrfs
```

# 手順

## コンテナ準備

btrfsのsubvolume用意

```
$ sudo btrfs subvolume create /var/lib/machines/winedev
Create subvolume '/var/lib/machines/winedev'
$ machinectl list-images
NAME    TYPE      RO USAGE CREATED                     MODIFIED
winedev subvolume no   n/a Mon 2021-03-08 19:40:41 JST n/a

1 images listed.
```

パッケージインストール

```
$ sudo pacstrap -c /var/lib/machines/winedev base base-devel sudo neovim bash-completion
==> Creating install root at /var/lib/machines/winedev
==> Installing packages to /var/lib/machines/winedev
...
(12/12) Updating the info directory file...
```

ユーザ追加

```
$ machinectl start winedev
machinectl shell winedev
Connected to machine winedev. Press ^] three times within 1s to exit session.
[root@winedev ~]# useradd -m -Gwheel dev
[root@winedev ~]# id dev
uid=1000(dev) gid=1000(dev) groups=1000(dev),998(wheel)
[root@winedev ~]# EDITOR=nvim visudo -f /etc/sudoers.d/10-wheel
[root@winedev ~]# cat /etc/sudoers.d/10-wheel
%wheel ALL=(ALL) NOPASSWD: ALL
[root@winedev ~]# exit
logout
Connection to machine winedev terminated.
$ machinectl shell dev@winedev
Connected to machine winedev. Press ^] three times within 1s to exit session.
[dev@winedev ~]$ sudo id
uid=0(root) gid=0(root) groups=0(root)
```

ネットワーク有効化

```
[dev@winedev ~]$ sudo systemctl enable --now systemd-networkd
Created symlink /etc/systemd/system/dbus-org.freedesktop.network1.service -> /usr/lib/systemd/system/systemd-networkd.service.
Created symlink /etc/systemd/system/multi-user.target.wants/systemd-networkd.service -> /usr/lib/systemd/system/systemd-networkd.service.
Created symlink /etc/systemd/system/sockets.target.wants/systemd-networkd.socket -> /usr/lib/systemd/system/systemd-networkd.socket.
Created symlink /etc/systemd/system/network-online.target.wants/systemd-networkd-wait-online.service -> /usr/lib/systemd/system/systemd-networkd-wait-online.service.
[dev@winedev ~]$ sudo systemctl enable --now systemd-resolved
Created symlink /etc/systemd/system/dbus-org.freedesktop.resolve1.service -> /usr/lib/systemd/system/systemd-resolved.service.
Created symlink /etc/systemd/system/multi-user.target.wants/systemd-resolved.service -> /usr/lib/systemd/system/systemd-resolved.service.
[dev@winedev ~]$ sudo ln -svf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
'/etc/resolv.conf' -> '/run/systemd/resolve/stub-resolv.conf'
[dev@winedev ~]$ sudo pacman -Syu
:: Synchronizing package databases...
 core is up to date
 extra is up to date
 community is up to date
:: Starting full system upgrade...
 there is nothing to do
```

## 開発環境準備

Rust等をインストール

```
[dev@winedev ~]$ sudo pacman -S rustup
...
[dev@winedev ~]$ mkdir -p ~/.local/share/bash-completion/completions
[dev@winedev ~]$ rustup completions bash cargo > ~/.local/share/bash-completion/completions/cargo
[dev@winedev ~]$ rustup completions bash rustup > ~/.local/share/bash-completion/completions/rustup
[dev@winedev ~]$ rustup toolchain install stable
...
[dev@winedev ~]$ rustup target add x86_64-pc-windows-gnu
...
```

## Wineインストール

Multilibリポジトリ有効化
```
[dev@winedev ~]$ EDITOR=nvim sudoedit /etc/pacman.conf
[dev@winedev ~]$ cat /etc/pacman.conf |grep -e'\[multilib\]'
[multilib]
[dev@winedev ~]$ sudo pacman -Syyu
...
[dev@winedev ~]$ sudo pacman -S wine gnutls lib32-gnutls mpg123 lib32-mpg123
...
```

## MinGWのGCCをインストール

```
[dev@winedev ~]$ sudo pacman -S mingw-w64-gcc
...
```

## 動作確認

```
[dev@winedev ~]$ cargo init b
     Created binary (application) package
[dev@winedev ~]$ cd b
[dev@winedev b]$ mkdir .cargo
[dev@winedev b]$ nvim .cargo/config.toml
[dev@winedev b]$ cat .cargo/config.toml
[target.x86_64-pc-windows-gnu]
linker = "/usr/bin/x86_64-w64-mingw32-gcc"
ar = "/usr/x86_64-w64-mingw32/bin/ar"
runner = "wine"
[dev@winedev b]$ cargo build --target x86_64-pc-windows-gnu
...
    Finished dev [unoptimized + debuginfo] target(s) in 0.01s
[dev@winedev b]$ file target/x86_64-pc-windows-gnu/debug/b.exe
target/x86_64-pc-windows-gnu/debug/b.exe: PE32+ executable (console) x86-64, for MS Windows
[dev@winedev b]$ wine target/x86_64-pc-windows-gnu/debug/b.exe
0048:err:explorer:initialize_display_settings Failed to query current display settings for L"\\\\.\\DISPLAY1".
Hello, world!
```

# さいごに

Visual StudioのBuild Toolsを使うと、msvcもコンパイル可能だと思いますが、[ライセンス条項](https://visualstudio.microsoft.com/ja/license-terms/mlt031519/)によると、Visual Studio (無償版のCommunityも含む) を共に利用する場合に限り利用できる旨記載があります。
もし、使用権があるのであれば、[mstorsjo/msvc-wine](https://github.com/mstorsjo/msvc-wine) の

- `vsdownload.py`
- `install.sh`

を使いリンカを設定すればLinuxでもmsvc版をコンパイルできると思うが、私は所持していないため確認できず。。
Windows向けのコードを書くのであれば、Windows動く環境ほしいな。。

実際に使ってみて何かあれば書きます。
