---
title: "何も考えずにWASMインスタンスを混ぜると危ないかも"
emoji: "🥶"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wasm"]
published: true
---

WASMの複数インスタンス間で[WebAssembly.Memory](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory)を共有すればダイナミックリンクみたいなことが実現できるかも。と思い調査したときのメモです。
結論としては、WASMインスタンス間での`WebAssembly.Memory`の共有は、私の力量では危ないということが分かりました。(2021年9月現在)

下記バージョンのClangを利用して確認しています。

```
$ clang -v
clang version 12.0.1
...
```

# WASMの状態管理

はじめに、WASMインスタンスは下記状態があります。

- [WebAssembly.Memory](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory) ... いわゆるメモリで、Clangが生成したコードではヒープやスタックの一部、定数が格納されているようでした。
- [WebAssembly.Global](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Global) ... WASM内の環境、外側の環境からアクセスできるグローバル変数を表します。
- [WebAssembly.Table](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Table) ... (現状は)WASM内からアクセスする関数ポインタを管理します。

これらの状態がWASMインスタンス間で適切に隔離・共有できていれば、`WebAssembly.Memory`を共有した複数WASMインスタンスの動作が可能だと考えています。

# 試したこと

## WASMの定数

定数の格納等に利用されるdataセクションはインスタンス化時にメモリへ書き込まれるようです。
https://developer.mozilla.org/ja/docs/WebAssembly/Understanding_the_text_format#webassembly_memory
> データセクションではインスタンス化時にオフセットを指定してバイト列の文字列を書きこむことができます。

確かにそのようです。

```wasm
;; a.wasm
(module
  (import "env" "memory" (memory 1))
  (data (i32.const 0) "HELLO")
)
```

```JavaScript
// run.ts
const memory = new WebAssembly.Memory({initial: 32});
const bc = await Deno.readFile(new URL('./a.wasm', import.meta.url));
console.log(new Uint8Array(memory.buffer, 0, 8));
await WebAssembly.instantiate(bc, { env: { memory } });
console.log(new Uint8Array(memory.buffer, 0, 8));
```

Output
```bash
$ deno run --allow-read ./run.ts
Uint8Array(8) [
  0, 0, 0, 0,
  0, 0, 0, 0
]
Uint8Array(8) [
  72, 69, 76, 76,
  79,  0,  0,  0
]
```

それでは、メモリを共有する2つのWASMをインスタンス化したする場合、後からインスタンス化したWASMモジュールの内容で定数領域は上書きされてしまうのでしょうか。確認してみます。

## 定数を上書きしてみる実験

メモリを共有した２つのWASMモジュールをインスタンス化し、あとからインスタンス化したWASMの内容で定数を上書きしてみます。
まずは１つ目のコードです。

```c
// main.c
extern void print(void *ptr, int len) __attribute__((import_module("env"), import_name("print")));

void _start() __attribute__((export_name("_start"))) {
    char buf[6] = "hello";
    print(&buf, 6);
}
```

> char buf[6] = "hello";

が上書き対象の定数です。

次に後からインスタンス化される定数だけ配置したコードです。最適化で定数が無くならないように呼び出されない関数内に定義します。

```c
// extra.c
void empty() __attribute__((export_name("empty"))) {
    const char *buf = "HELLO";
}
```

> const char \*buf = "HELLO";

の定数で書き換えられるはずです。

WASMを下記コードで実行します。

```typescript
// run.ts
const memory = new WebAssembly.Memory({initial: 128});

const mainbc = await Deno.readFile(new URL("./main.opt.wasm", import.meta.url));
const maininstance = await WebAssembly.instantiate(mainbc, {
    env: {
        memory,
        print: (ptr: number, size: number) => console.log(new TextDecoder().decode(new Uint8Array(memory.buffer, ptr, size))),
    }
});

const extrabc = await Deno.readFile(new URL("./extra.opt.wasm", import.meta.url));
await WebAssembly.instantiate(extrabc, { env: { memory, } });

(maininstance.instance.exports._start as () => void)();
```

```
$ deno run --allow-read ./run.ts
HELLO
```

`hello`と出力されずに`HELLO`で上書きされてしまいましたね。
複数のWASMインスタンスで`WebAssembly.Memory`を共有する場合、dataセクションのオフセットを適切に調整してあげないと、
不意に定数が書き換わってしまうと思われます。

## スタックの管理

ClangでコンパイルしたWASMでは、スタック上の配列等は`WebAssembly.Memory`のスタック領域としている場所で管理しているようです。また、スタックがどこまで積み上がったかは、`WebAssembly.Global`変数で管理しているようです。

スタックが積み上がっていく様子を確認してみましょう。

まずは、スタックを積み上げるWASMの本体です。
スタックにメモリを確保した後は、環境のコールバックを呼び出します。

```c
// stack.c
extern void callback(void* ptr) __attribute__((import_module("env"), import_name("callback")));

void func() __attribute__((export_name("func"))) {
    char buf[8192];
    callback(&buf);
}
```

コンパイルされたWASMでは`WebAssembly.Global`はエクスポートされていないので、WATに変換してから無理やりエクスポートするように書き換えます。

```bash
$ sed -e'$ i (export "global" (global 0))' stack.opt.wat > stack.mod.wat
$ wat2wasm stack.mod.wat > stack.mod.wasm
```

上記WASMを呼び出すJavaScriptです。
コールバック後にWASM関数を呼び出しスタックをどんどん積んでいきます。
その際に、スタックのポインタと`WebAssembly.Global`の値を確認します。

```typescript
const bc = await Deno.readFile(new URL('./stack.mod.wasm', import.meta.url));
const instance = await WebAssembly.instantiate(bc, {
    env: {
        callback: (ptr: number) => {
            const global = instance.instance.exports.global as WebAssembly.Global;
            console.log(ptr, global.value);
            if (ptr !== global.value) {
                throw new Error();
            }
            if (ptr > 0) {
                (instance.instance.exports.func as () => void)();
            }
        }
    }
});
(instance.instance.exports.func as () => void)();
```

実行してみると、、

```bash
$ deno run --allow-read /home/ysk/work/is-mix-wasm-instance-danger/03_stack/run.ts
58368 58368
50176 50176
41984 41984
33792 33792
25600 25600
17408 17408
9216 9216
1024 1024
-7168 -7168
```

ポインタと`WebAssembly.Global`が一致しているので、`WebAssembly.Global`でスタックのが管理されていることがわかりますね。
(最後、スタックオーバーフローしている。。)

`WebAssembly.Memory`を共有しただけではWASMインスタンスそれぞれで、この`WebAssembly.Global`が共有できていないので、
スタック上に確保したバッファ等が変な動きをしてしまいそうです。
この`WebAssembly.Global`をエクスポートする方法が[wasm-ld](https://lld.llvm.org/WebAssembly.html)にはなさそうなので、
状態の共有はなかなかむずかしそうです。

## ぐちゃぐちゃ

最後にわかったことであそんでみます。

```c
// lib.c
void func() __attribute__((export_name("func"))) {
    char buf[6] = "abcde";
}
```

```c
// main.c
extern void print(void *ptr, int len) __attribute__((import_module("env"), import_name("print")));
extern void func() __attribute__((import_module("env"), import_name("func")));

void _start() __attribute__((export_name("_start"))) {
    char _[16] = "💩!!";
    char buf[6] = "hello";
    func();
    print(&buf, 6);
}
```

```typescript
const memory = new WebAssembly.Memory({initial: 128});

const libbc = await Deno.readFile(new URL("./lib.opt.wasm", import.meta.url));
const libinstance = await WebAssembly.instantiate(libbc, {
    env: { memory, }
});

const mainbc = await Deno.readFile(new URL("./main.opt.wasm", import.meta.url));
const maininstance = await WebAssembly.instantiate(mainbc, {
    env: {
        memory,
        func: libinstance.instance.exports.func,
        print: (ptr: number, size: number) => {
            const buf = new Uint8Array(memory.buffer, ptr, size);
            console.log(new TextDecoder().decode(buf));
        }
    }
});

(maininstance.instance.exports._start as () => void)();
```

```bash
$ deno run --allow-read ./run.ts
💩!!
```

# 使用したコード

今回実験に使ったコードはここに置いています。
https://github.com/yskszk63/is-mix-wasm-instance-danger

WASM楽しいです。
