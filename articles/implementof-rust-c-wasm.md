---
title: "RustからCのコードを呼び出すWASMの作り方"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust", "c", "wasm", "wasi"]
published: true
---
RustからCのコードを呼び出すWASMの作り方のメモです。
Cで書かれた画像読み込みライブラリである[stb](https://github.com/nothings/stb)をRustから呼び出すWASMの作成に苦労したのでその方法をまとめました。

結論から申し上げると、下記で解決しました。
- `llvm-ar`を使う
- WASIを使う

## シンプルなWASM

下記Cコードを呼び出す、

```C
int add(int left, int right) {
    return left + right;
}
```

Rustのコードを書いたとします。

```rust
use std::os::raw::c_int;

extern "C" {
    fn add(left: c_int, right: c_int) -> c_int;
}

#[no_mangle]
pub extern fn plus1(n: c_int) -> c_int {
    unsafe { add(n, 1) }
}
```

cargoでCコードをビルドするため、[cc](https://docs.rs/cc/1.0.70/cc/)クレートを使った`build.rs`を書きます。

```rust
fn main() {
    cc::Build::new().file("src/lib.c").compile("lib");
}
```

`TARGET` = `HOST`のビルドは成功します。

```bash
$ cargo build
   Compiling cc v1.0.70
   Compiling simple v0.1.0 (/home/ysk/work/rust+c-wasm_example/simple)
    Finished dev [unoptimized + debuginfo] target(s) in 3.66s
```

そのままでは`TARGET` = `wasm32-unknonw-unknown`はエラーが出てしまいます。

```bash
$ cargo build --target wasm32-unknown-unknown
   Compiling simple v0.1.0 (/home/ysk/work/rust+c-wasm_example/simple)
error: linking with `rust-lld` failed: exit status: 1
  |
  = note: "rust-lld" "-flavor" "wasm" "--rsp-quoting=posix" "--export" "plus1" "--export=__heap_base" "--export=__data_end" "-z" "stack-size=1048576" "--stack-first" "--allow-undefined" "--fatal-warnings" "--no-demangle" "--no-entry" "--export-dynamic" ...
  = note: rust-lld: error: /home/ysk/work/rust+c-wasm_example/target/wasm32-unknown-unknown/debug/build/simple-a4fd5db105776d1e/out/liblib.a: archive has no index; run ranlib to add one


error: aborting due to previous error

error: could not compile `simple`

To learn more, run the command again with --verbose.
```

これは`llvm-ar`を使ってあげると解消です。

```bash
$ AR=llvm-ar cargo build --target wasm32-unknown-unknown
   Compiling cc v1.0.70
   Compiling simple v0.1.0 (/home/ysk/work/rust+c-wasm_example/simple)
    Finished dev [unoptimized + debuginfo] target(s) in 3.98s
$ ls ../target/wasm32-unknown-unknown/debug/simple.wasm
../target/wasm32-unknown-unknown/debug/simple.wasm
```

Denoで動作確認してみます。

```typescript
const bc = await Deno.readFile("../target/wasm32-unknown-unknown/debug/simple.wasm");
const instance = await WebAssembly.instantiate(bc, {});
const got = (instance.instance.exports.plus1 as Function)(3);
console.log(got);
```

```bash
$ deno run --allow-read test.ts
4
```

大丈夫そうですね。

## WASI

次にCライブラリを使ったCコードを

```C
#include<stdio.h>

void chello(void) {
    printf("Hello, World!\n");
}
```

Rustから呼び出します。

```rust
extern "C" {
    fn chello();
}

fn main() {
    unsafe { chello() }
}
```

`build.rs`は先程のものと同様です。

このままビルドすると、

```bash
$ AR=llvm-ar cargo build --target wasm32-unknown-unknown
   Compiling cc v1.0.70
   Compiling wasi v0.1.0 (/home/ysk/work/rust+c-wasm_example/wasi)
The following warnings were emitted during compilation:

warning: src/lib.c:1:9: fatal error: 'stdio.h' file not found
warning: #include<stdio.h>
warning:         ^~~~~~~~~
warning: 1 error generated.

error: failed to run custom build command for `wasi v0.1.0 (/home/ysk/work/rust+c-wasm_example/wasi)`

Caused by:
  process didn't exit successfully: `/home/ysk/work/rust+c-wasm_example/target/debug/build/wasi-37a19105184d97bf/build-script-build` (exit status: 1)
  --- stdout
  TARGET = Some("wasm32-unknown-unknown")
  OPT_LEVEL = Some("0")
  HOST = Some("x86_64-unknown-linux-gnu")
  CC_wasm32-unknown-unknown = None
  CC_wasm32_unknown_unknown = None
  TARGET_CC = None
  CC = None
  CFLAGS_wasm32-unknown-unknown = None
  CFLAGS_wasm32_unknown_unknown = None
  TARGET_CFLAGS = None
  CFLAGS = None
  CRATE_CC_NO_DEFAULTS = None
  DEBUG = Some("true")
  running: "clang" "-O0" "-ffunction-sections" "-fdata-sections" "-fPIC" "-g" "-fno-omit-frame-pointer" "--target=wasm32-unknown-unknown" "-Wall" "-Wextra" "-o" "/home/ysk/work/rust+c-wasm_example/target/wasm32-unknown-unknown/debug/build/wasi-1f09c036d21f248d/out/src/lib.o" "-c" "src/lib.c"
  cargo:warning=src/lib.c:1:9: fatal error: 'stdio.h' file not found
  cargo:warning=#include<stdio.h>
  cargo:warning=        ^~~~~~~~~
  cargo:warning=1 error generated.
  exit status: 1

  --- stderr


  error occurred: Command "clang" "-O0" "-ffunction-sections" "-fdata-sections" "-fPIC" "-g" "-fno-omit-frame-pointer" "--target=wasm32-unknown-unknown" "-Wall" "-Wextra" "-o" "/home/ysk/work/rust+c-wasm_example/target/wasm32-unknown-unknown/debug/build/wasi-1f09c036d21f248d/out/src/lib.o" "-c" "src/lib.c" with args "clang" did not execute successfully (status code exit status: 1).
```

`stdio.h`がないよ。って言われていますね。

syscall相当のものを提供するWASIを使っていきたいと思います。
まずは、Rust用の環境を揃えたいと思います。
wasiターゲットと、cargoのヘルパーコマンド、ランタイムであるwasmtimeを追加します。

```bash
$ rustup target add wasm32-wasi
...
$ cargo install cargo-wasi
...
$ paru -S wasmtime-bin
...
```

C用にWASI用のCライブラリが含まれた[wasi-sdk](https://github.com/WebAssembly/wasi-sdk)を追加します。

```bash
$ paru -S wasi-sdk-bin
...
$ ls -d /opt/wasi-sdk/wasi-sysroot
/opt/wasi-sdk/wasi-sysroot
```

Cコンパイルのためのclangのsysrootをwasi-sdkのものに向けて、
`cargo-wasi`を使って実行してみます。

```bash
$ AR=llvm-ar CFLAGS='--sysroot /opt/wasi-sdk/wasi-sysroot' cargo wasi run
   Compiling cc v1.0.70
   Compiling wasi v0.1.0 (/home/ysk/work/rust+c-wasm_example/wasi)
    Finished dev [unoptimized + debuginfo] target(s) in 4.10s
     Running `/home/ysk/.cargo/bin/cargo-wasi /home/ysk/work/rust+c-wasm_example/target/wasm32-wasi/debug/wasi.wasm`
     Running `/home/ysk/work/rust+c-wasm_example/target/wasm32-wasi/debug/wasi.wasm`
Hello, World!
```

成功です！

なお、[@wasmer/wasi](https://github.com/wasmerio/wasmer-js)等を使えば、Webブラウザ上でもハローワールドができます。
感動ですね。

この記事を書くにあたって書いたコードは下記においています。
https://github.com/yskszk63/rust-c-wasm_example
