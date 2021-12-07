---
title: "JavaでFFI"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Java", "FFI"]
published: false
---

これは[Javaのカレンダー | Advent Calendar 2021](https://qiita.com/advent-calendar/2021/java)のN日目の記事です。 FIXME

# 概要

JavaでJNIを使ったFFIをしようとすると、大変ですよね。
バインディング用のCコードを書いたり、Cを書いたり。

この状況を改善すべく、[Project Panama](https://openjdk.java.net/projects/panama/)が進行中しており、Java 14よりIncubatorとしてOpen JDKのリリースに取り込まれています。
- [Java 17 API](https://docs.oracle.com/en/java/javase/17/docs/api/jdk.incubator.foreign/jdk/incubator/foreign/package-summary.html)
- [Java 16 API](https://docs.oracle.com/en/java/javase/16/docs/api/jdk.incubator.foreign/jdk/incubator/foreign/package-summary.html)
- [Java 15 API](https://docs.oracle.com/en/java/javase/15/docs/api/jdk.incubator.foreign/jdk/incubator/foreign/package-summary.html)
- [Java 14 API](https://docs.oracle.com/en/java/javase/14/docs/api/jdk.incubator.foreign/jdk/incubator/foreign/package-summary.html)

今回はJava 17でProject Panamaを少し触ってみたいと思います。

# `puts(3)`してみる

先ずは簡単にHello, Worldをしてみます。

```java
import java.lang.invoke.MethodType;

import jdk.incubator.foreign.CLinker;
import jdk.incubator.foreign.MemoryAddress;
import jdk.incubator.foreign.FunctionDescriptor;
import jdk.incubator.foreign.ResourceScope;

public class Main {
    public static void main(String...args) throws Throwable {
        var puts = CLinker.getInstance().downcallHandle(
            CLinker.systemLookup().lookup("puts").orElseThrow(),
            MethodType.methodType(void.class, MemoryAddress.class),
            FunctionDescriptor.ofVoid(CLinker.C_POINTER));
        try (var scope = ResourceScope.newConfinedScope()) {
            var cstr = CLinker.toCString("Hello, World!", scope);
            puts.invokeExact(cstr.address());
        }
    }
}
```

Pythonの[ctypes](https://docs.python.org/3/library/ctypes.html)に雰囲気が似ていると感じました。

```bash
$ java --add-modules jdk.incubator.foreign --enable-native-access=ALL-UNNAMED ./Main.java
WARNING: Using incubator modules: jdk.incubator.foreign
warning: using incubating module(s): jdk.incubator.foreign
1 warning
Hello, World!
$
```

実行できましたね！

# クラスローダといっしょ

簡単にFFIが出来ました。そうなると、個人的にはクラスローダが絡むと出てくる`java.lang.UnsatisfiedLinkError`が気になってきます。
ある動的ライブラリをロードできるのは、一つのクラスローダ縛りのあれです。

理由は無いですが、Native側はRustで書きました。ほぼ同じ内容のライブラリをふたつ用意して読み込んでみます。

```
.
├── ...
├── Cargo.toml
├── Main.java
├── v1
│  ├── Cargo.toml
│  └── src
│     └── lib.rs
└── v2
   ├── Cargo.toml
   └── src
      └── lib.rs
```

```rust
// v1/src/lib.rs & v2/src/lib.rs
#[no_mangle]
pub extern "C" fn hello() {
    println!("Hello, world!{}", env!("CARGO_PKG_NAME"))
}
```

```java
// Main.java
...
public class Main implements Consumer<String> {
    ...
    public static void main(String...args) throws Exception {
        new Main().accept("target/debug/libv1.so");
        new Main().accept("target/debug/libv2.so");

        // create new classloader. (same class path)
        var cp = System.getProperty("java.class.path");
        var paths = Stream.of(cp.split(Pattern.quote(File.pathSeparator))).map(p -> must(() -> Path.of(p).toUri().toURL())).toArray(URL[]::new);
        var loader = new URLClassLoader(paths, null);
        @SuppressWarnings("unchecked")
        var instance = (Consumer<String>) loader.loadClass("Main").getDeclaredConstructor().newInstance();
        instance.accept("target/debug/libv2.so");
        instance.accept("target/debug/libv2.so");
    }

    @Override
    public void accept(String lib) {
        System.load(Path.of(lib).toAbsolutePath().toString());

        var hello = CLinker.getInstance().downcallHandle(
            SymbolLookup.loaderLookup().lookup("hello").orElseThrow(),
            MethodType.methodType(void.class),
            FunctionDescriptor.ofVoid());
        try {
            hello.invokeExact();
        } catch (RuntimeException e) {
            throw e;
        } catch (Throwable t) {
            throw new RuntimeException(t);
        }
    }
}
```

書いていて気づきましたが、シンボルのルックアップはSystem.load(String)やSystem.LoadLibrary(String)でロードされたものに対して行なわれるようです。
<https://docs.oracle.com/en/java/javase/17/docs/api/jdk.incubator.foreign/jdk/incubator/foreign/SymbolLookup.html>

> A symbol lookup. Exposes a lookup operation for searching symbol addresses by name, see lookup(String). A symbol lookup can be used to lookup a symbol in a loaded library. Clients can obtain a loader lookup, which can be used to search symbols in libraries loaded by the current classloader (e.g. using System.load(String), or System.loadLibrary(String)). Alternatively, clients can obtain a platform-dependent lookup, to search symbols in the standard C library.

System.loadを使っているので、`java.lang.UnsatisfiedLinkError`が発生しそうですね。ひとまず実行してみます。

```bash
$ java --add-modules jdk.incubator.foreign --enable-native-access=ALL-UNNAMED Main
WARNING: Using incubator modules: jdk.incubator.foreign
Hello, world!v1
Hello, world!v1
Exception in thread "main" java.lang.UnsatisfiedLinkError: Native Library <...>/target/debug/libv2.so already loaded in another classloader
        at java.base/jdk.internal.loader.NativeLibraries.loadLibrary(NativeLibraries.java:197)
        at java.base/jdk.internal.loader.NativeLibraries.loadLibrary(NativeLibraries.java:170)
        at java.base/java.lang.ClassLoader.loadLibrary(ClassLoader.java:2389)
        at java.base/java.lang.Runtime.load0(Runtime.java:755)
        at java.base/java.lang.System.load(System.java:1953)
        at Main.accept(Main.java:48)
        at Main.accept(Main.java:21)
        at Main.main(Main.java:42)
```

ダメでしたね。シンボルも被っているので、v2も呼びだされていません。

# libdlを使う

簡単にFFIが出きるので、libdlを使ってJavaの縛りを回避してしまえば良いのでは？
ということで、やってみます。

```java
// Main2.java
...
public class Main2 implements Consumer<String> {
    ...
    public static void main(String...args) throws Exception {
        new Main2().accept("target/debug/libv1.so");
        // same class loader.
        new Main2().accept("target/debug/libv2.so");

        // create new classloader. (same class path)
        var cp = System.getProperty("java.class.path");
        var paths = Stream.of(cp.split(Pattern.quote(File.pathSeparator))).map(p -> must(() -> Path.of(p).toUri().toURL())).toArray(URL[]::new);
        var loader = new URLClassLoader(paths, null);
        @SuppressWarnings("unchecked")
        var instance = (Consumer<String>) loader.loadClass("Main2").getDeclaredConstructor().newInstance();
        instance.accept("target/debug/libv1.so");
        instance.accept("target/debug/libv2.so");
    }

    @Override
    public void accept(String lib) {
        try {
            var handle = dlopen(lib, 0x00001 /* RTLD_LAZY */);
            var helloAddr = dlsym(handle, "hello");
            var hello = CLinker.getInstance().downcallHandle(
                helloAddr,
                MethodType.methodType(void.class),
                FunctionDescriptor.ofVoid());
            hello.invokeExact();
        } catch (RuntimeException e) {
            throw e;
        } catch (Throwable t) {
            throw new RuntimeException(t);
        } finally {
            //dlclose(handle);
        }
    }

    static MemoryAddress dlopen(String filename, int flag) throws Throwable {
        var dlopen = CLinker.getInstance().downcallHandle(
                CLinker.systemLookup().lookup("dlopen").orElseThrow(),
                MethodType.methodType(MemoryAddress.class, MemoryAddress.class, int.class),
                FunctionDescriptor.of(CLinker.C_POINTER, CLinker.C_POINTER, CLinker.C_INT));
        try (var scope = ResourceScope.newConfinedScope()) {
            var cname = CLinker.toCString(filename, scope);
            var result = (MemoryAddress) dlopen.invokeExact(cname.address(), flag);
            if (result.equals(MemoryAddress.NULL)) {
                String msg = dlerror();
                throw new RuntimeException(msg);
            }
            return result;
        }
    }

    static String dlerror() {
        try {
            var dlerror = CLinker.getInstance().downcallHandle(
                    CLinker.systemLookup().lookup("dlerror").orElseThrow(),
                    MethodType.methodType(MemoryAddress.class),
                    FunctionDescriptor.of(CLinker.C_POINTER));
            var result = (MemoryAddress) dlerror.invokeExact();
            if (result.equals(MemoryAddress.NULL)) {
                return "";
            }
            return CLinker.toJavaString(result);
        } catch (Throwable t) {
            return "failed to get error message. because: " + t;
        }
    }

    static MemoryAddress dlsym(MemoryAddress handle, String symbol) throws Throwable {
        var dlsym = CLinker.getInstance().downcallHandle(
                CLinker.systemLookup().lookup("dlsym").orElseThrow(),
                MethodType.methodType(MemoryAddress.class, MemoryAddress.class, MemoryAddress.class),
                FunctionDescriptor.of(CLinker.C_POINTER, CLinker.C_POINTER, CLinker.C_POINTER));
        try (var scope = ResourceScope.newConfinedScope()) {
            var cname = CLinker.toCString(symbol, scope);
            var result = (MemoryAddress) dlsym.invokeExact(handle, cname.address());
            if (result.equals(MemoryAddress.NULL)) {
                String msg = dlerror();
                throw new RuntimeException(msg);
            }
            return result;
        }
    }

    static void dlclose(MemoryAddress handle) throws Throwable {
        var dlclose = CLinker.getInstance().downcallHandle(
                CLinker.systemLookup().lookup("dlclose").orElseThrow(),
                MethodType.methodType(int.class, MemoryAddress.class),
                FunctionDescriptor.of(CLinker.C_INT, CLinker.C_POINTER));
        var result = (int) dlclose.invokeExact(handle); // Probably no NPE will occur. (Object -> Integer -> int)
        if (result != 0) {
            String msg = dlerror();
            throw new RuntimeException(msg);
        }
    }
}
```

実行します。

```bash
$ java --add-modules jdk.incubator.foreign --enable-native-access=ALL-UNNAMED Main2
WARNING: Using incubator modules: jdk.incubator.foreign
Hello, world!v1
Hello, world!v2
Hello, world!v1
Hello, world!v2
```

できました！

# さいごに

暫くJavaから離れていたので、Project Panamaに限らず最近のJavaの進化には驚かされるばかりです。

JavaでFFIが簡単に扱える未来が見えてとても楽しみです。
早く運河が開通してショートカットしたいですね！
