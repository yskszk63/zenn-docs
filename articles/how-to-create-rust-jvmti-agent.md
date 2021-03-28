---
title: "RustでJVMTIエージェントを作る"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust", "java"]
published: false
---

需要は少ないと思いますが、RustでJavaのJVMTIエージェントを作る方法を書きます。

# JVMTIとは

JVMTIとは[JVM Tool Interface](https://docs.oracle.com/en/java/javase/16/docs/specs/jvmti.html)の省略です。
起動時に指定したり、起動後に動的にJVMTIエージェントをアタッチさせることで
Javaの動作を変更したり監視したりさせることができます。

# 作り方

## 前提

```bash
$ cat /etc/os-release
NAME="Arch Linux"
...
$ cargo --version
cargo 1.51.0 (43b129a20 2021-03-16)
$ cargo install --list|grep cargo-edit
cargo-edit v0.7.0:
$ pacman -Q jdk-openjdk jre-openjdk
jdk-openjdk 15.0.2.u7-1
jre-openjdk 15.0.2.u7-1
```

## クレートの準備

まずはクレートを作成します。

```bash
$ cargo new --lib jvmti-study
     Created library `jvmti-study` package
$ cd jvmti-study
```

動的ライブラリ用にします。

```toml
Cargo.toml
----------
...
[lib]
crate-type = ["cdylib"]
...
```

## bindgen

bindgenを使って、sys-モジュールを作成します。
まずは、bindgenを依存クレートに追加

```bash
$ cargo add --build bindgen
    Updating 'https://github.com/rust-lang/crates.io-index' index
      Adding bindgen v0.57.0 to build-dependencies
```

`build.rs`スクリプトを作成

```rust:build.rs
use std::env;
use std::path;

fn main() {
    const LIB: &str = "/usr/lib/jvm/default-runtime/lib/server";
    const INCLUDE: &str = "/usr/lib/jvm/default/include";
    const INCLUDE_LINUX: &str = "/usr/lib/jvm/default/include/linux";

    println!("cargo:rustc-link-lib=jvm");
    println!("cargo:rustc-link-search=native={}", LIB);

    let bindings = bindgen::builder()
        .header_contents("bindings.h", "#include <jvmti.h>")
        .clang_arg(format!("-I{}", INCLUDE))
        .clang_arg(format!("-I{}", INCLUDE_LINUX))
        .derive_debug(true)
        .derive_default(true)
        .parse_callbacks(Box::new(bindgen::CargoCallbacks))
        .generate()
        .expect("failed to generate bindgen.");

    let out_path = path::PathBuf::from(env::var("OUT_DIR").unwrap());
    bindings
        .write_to_file(out_path.join("bindings.rs"))
        .expect("failed to write bindings.rs.");
}
```

ソースコードに生成したコードを埋め込む

```rust:src/lib.rs
#[allow(non_upper_case_globals)]
#[allow(non_camel_case_types)]
#[allow(non_snake_case)]
#[allow(unused)]
mod sys {
    include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
}
```

## 最低限のエージェントを実装

JVMからのエントリポイントは下記です。
いずれも実装は必要なもののみで大丈夫です。
- `Agent_OnLoad`
  + 起動時に動作するエージェントのエントリポイントです
  + JVMは初期化が終わっていないので、この時点ではJNI関数にアクセスできません。
- `Agent_OnAttach`
  + JVM動作中に動的にアタッチするエージェントのエントリポイントです
- `Agent_OnUnload`
  + JVMTIエージェント終了時のエントリポイントです

```rust:src/lib.rs
...
#[no_mangle]
pub extern "C" fn Agent_OnLoad(
    _vm: *mut sys::JavaVM,
    _options: *const std::os::raw::c_char,
    _reserved: *const std::ffi::c_void,
) -> sys::jint {
    println!("Hello, JVMTI!");
    0
}

#[no_mangle]
pub extern "C" fn Agent_OnAttach(
    _vm: *mut sys::JavaVM,
    _options: *const std::os::raw::c_char,
    _reserved: *const std::ffi::c_void,
) -> sys::jint {
    println!("JVMTI Attached.");
    0
}

#[no_mangle]
pub extern "C" fn Agent_OnUnload(_vm: *mut sys::JavaVM) {
    println!("Good bye.");
}
```

コンパイル

```shell
$ cargo build
...
$ ls target/debug/libjvmti_study.so
target/debug/libjvmti_study.so
```

## 動作確認

動作確認用のJavaプログラム

```java:Test.java
public class Test {
    public static void main(String...args) throws Exception {
        System.out.println("Hello, world!");
        Thread.sleep(Long.MAX_VALUE);
    }
}
```

```
$ javac Test.java
```

まずは、動作確認用のJavaプログラムの動作確認

```bash
$ java Test
Hello, world!
^C
```

Java起動時にエージェントを開始

```bash
$ java -agentpath:target/debug/libjvmti_study.so Test
Hello, JVMTI!
Hello, world!
^CGood bye.
```

動的にアタッチ

```bash
$ java Test &
[1] 84459
$ jcmd 84459 JVMTI.agent_load target/debug/libjvmti_study.so
84459:
JVMTI Attached.
return code: 0
$ kill 84459
Good bye.
```

TODO イベント、環境の管理

# おわり
