# c++産の実行ファイル分析その1
## 目的
- `c言語`産の実行ファイルとの`違い`を中心に眺めていくことで`c++`産実行ファイルや`c++`の言語仕様の理解を深める
---
## 環境
- cc: gcc 9.3.0
- cxx: g++ 9.3.0
- ld: collect2 version 9.3.0 GNU ld (GNU Binutils for Ubuntu) 2.34
---
## [ソースコード](hello.cpp)
- Hello!と出力するだけ

---
## 観察する実行ファイルを生成
```
$ g++ hello.cpp -o hello
```
---
## lddでリンクされている共有ライブラリを確認
```
$ ldd ./hello
	linux-vdso.so.1 (0x00007ffc71be2000)
	libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007ffa31c97000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ffa31aa5000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007ffa31956000)
	/lib64/ld-linux-x86-64.so.2 (0x00007ffa31ed8000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007ffa3193b000)
```
- libstdc++.so.6, libgcc_s.so.1が目新しい。
### libstdc++.so.6
- 何のファイル？
  - c++ standard library
  
```
$ realpath /lib/x86_64-linux-gnu/libstdc++.so.6
/usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.28
```
- 頻繁にアップデートされていそう

- 定義されているシンボル情報を眺める

```
$ $ nm -D --demangle --defined-only  /lib/x86_64-linux-gnu/libstdc++.so.6  | less -N
```
- std::basic_\*stream, std::basic_string\*辺りが定義されているので確かに標準ライブラリだ
- Weakシンボルが多く、関数など後に再定義しやすい構造になっているのかな
- libstdc++.so.6にはどんなライブラリがリンクされているかというと、、(おそらくほとんどリンクされていないんだろうなぁぁ...)
```
$ ldd   /lib/x86_64-linux-gnu/libstdc++.so.6 
	linux-vdso.so.1 (0x00007ffc74df5000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f28e9d78000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f28e9b86000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f28ea103000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f28e9b6b000)
```
- こんな感じ
- ちなみにlibcは
```
$ ldd /lib/x86_64-linux-gnu/libc.so.6 
	/lib64/ld-linux-x86-64.so.2 (0x00007f7d2a59b000)
	linux-vdso.so.1 (0x00007ffdc7dad000)
```
- libc++はlibc、数学ライブラリであるlibm、次に見るlibgcc_s.so.1で作られている(libgcc_soはおそらくc++の言語仕様に関する何かなんだろうなぁぁ...)

### libgcc_so.1
```
$ nm -D --demangle --defined-only  /lib/x86_64-linux-gnu/libgcc_s.so.1  | less -N
```
- \_Unwind_\*の関数が定義されている。c++の例外処理を定義している
---
## どうでもよいこと
- 共有ライブラリファイルであるが実行権限付いて、単体で実行できるものが共有ライブラリファイルが存在する。例えば、libc.soやld.soなど
```
$ find  /lib/x86_64-linux-gnu/ -name "*.so" -type f -perm -u+x  -or -name "*.so.*" -type f -perm -u+x | xargs ls -al
-rwxr-xr-x 2 root root  191472 Dec 16  2020 /lib/x86_64-linux-gnu/ld-2.31.so
-rwxr-xr-x 1 root root 2029224 Dec 16  2020 /lib/x86_64-linux-gnu/libc-2.31.so
-rwxr-xr-x 1 root root  506207 Dec  6  2018 /lib/x86_64-linux-gnu/libodbccr.so.2.0.0
-rwxr-xr-x 1 root root  608957 Dec  6  2018 /lib/x86_64-linux-gnu/libodbcinst.so.2.0.0
-rwxr-xr-x 1 root root 2453791 Dec  6  2018 /lib/x86_64-linux-gnu/libodbc.so.2.0.0
-rwxr-xr-x 1 root root  157224 Dec 16  2020 /lib/x86_64-linux-gnu/libpthread-2.31.so
```
- `ld-2.31.so`、`libc-2.31.so`、`libodbccr.so`、`libodbcinst.so`、`libodbc.so`、`libpthread-2.31.so`に実行権限が付いていた。通常であれば、共有ライブラリに実行権限は必要ない。それぞれ共有ライブラリファイルであると同時に実行可能ファイルであり、単体で実行すると理由が分かるに違いない！

- `ld-2.31.so`は共有ライブラリ関連のことを調べるために使えるようだ。
- また、shell scriptなどのプログラムやコマンドとして単体で使用されることもあるので、実行権限が付いていることは納得である。
```
$ /lib/x86_64-linux-gnu/ld-2.31.so 
Usage: ld.so [OPTION]... EXECUTABLE-FILE [ARGS-FOR-PROGRAM...]
You have invoked `ld.so', the helper program for shared library executables.
This program usually lives in the file `/lib/ld.so', and special directives
in executable files using ELF shared libraries tell the system's program
loader to load the helper program from this file.  This helper program loads
the shared libraries needed by the program executable, prepares the program
to run, and runs it.  You may invoke this helper program directly from the
command line to load and run an ELF executable file; this is like executing
that file itself, but always uses this helper program from the file you
specified, instead of the helper program file specified in the executable
file you run.  This is mostly of use for maintainers to test new versions
of this helper program; chances are you did not intend to run this program.

  --list                list all dependencies and how they are resolved
  --verify              verify that given object really is a dynamically linked
			object we can handle
  --inhibit-cache       Do not use /etc/ld.so.cache
  --library-path PATH   use given PATH instead of content of the environment
			variable LD_LIBRARY_PATH
  --inhibit-rpath LIST  ignore RUNPATH and RPATH information in object names
			in LIST
  --audit LIST          use objects named in LIST as auditors
  --preload LIST        preload objects named in LIST
  
  ```
#### lddコマンド(shell script)内でld-2.31.soが使われていたりする。


```
$ ll /lib64/ld-linux-x86-64.so.2 
lrwxrwxrwx 1 root root 32 Dec 16  2020 /lib64/ld-linux-x86-64.so.2 -> /lib/x86_64-linux-gnu/ld-2.31.so*

$ file /usr/bin/ldd
/usr/bin/ldd: Bourne-Again shell script, ASCII text executable

$ grep '/lib64/ld-linux-x86-64.so.2' /usr/bin/ldd
RTLDLIST="/lib/ld-linux.so.2 /lib64/ld-linux-x86-64.so.2 /libx32/ld-linux-x32.so.2"
```

- `libc.so`、`libpthread.so`は自分自身のバージョンを確認するために使うことができる。そんな機能めったに使わないが、まぁ実行権限が付いていることは納得できる。
  - 使っているライブラリのバージョンが脆弱性のあるバージョンか確認する時とかに使いそう。
  - 共有ライブラリはファイル名見たらバージョンの予想はできますが。。
```
$ /lib/x86_64-linux-gnu/libc.so.6 
GNU C Library (Ubuntu GLIBC 2.31-0ubuntu9.2) stable release version 2.31.
Copyright (C) 2020 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.
Compiled by GNU CC version 9.3.0.
libc ABIs: UNIQUE IFUNC ABSOLUTE
For bug reporting instructions, please see:
<https://bugs.launchpad.net/ubuntu/+source/glibc/+bugs>.

$ /lib/x86_64-linux-gnu/libpthread.so
Native POSIX Threads Library
Copyright (C) 2020 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.
Forced unwind support included.

```

- 残りの `libodbccr.so`、`libodbcinst.so`、`libodbc.so`はどんな風に使えるのだろうか？
```
$ /lib/x86_64-linux-gnu/libodbccr.so
Segmentation fault (core dumped)

$ /lib/x86_64-linux-gnu/libodbccr.so -h
Segmentation fault (core dumped)

$ /lib/x86_64-linux-gnu/libodbccr.so --help
Segmentation fault (core dumped)
```
#### セグフォ、セグフォ、セグフォ、、、！！
#### 実行できないのでなぜ実行権限が付いているのだろう。 セグフォしかしないなら実行権限付けないで欲しいな。
#### 無駄に実行権限付けてると悪用できるバグが合った時に悪い人に悪用されかねない。
#### odbcで調べると、これら共有ライブラリはデータベースに関連するものであると知った。

---

## g++について
- cppのファイルのコンパイルだけならgccで行える。cでは存在しないクラスや例外処理を含んだソースコードであってもだ。
```
$ gcc -c hello.cpp
```
- 無事hello.oが生成できる
```
$ nm --demangle ./hello.o 
                 U __cxa_atexit
                 U __dso_handle
                 U _GLOBAL_OFFSET_TABLE_
0000000000000084 t _GLOBAL__sub_I_main
0000000000000000 T main
0000000000000037 t __static_initialization_and_destruction_0(int, int)
                 U std::ostream::operator<<(std::ostream& (*)(std::ostream&))
                 U std::ios_base::Init::Init()
                 U std::ios_base::Init::~Init()
                 U std::cout
                 U std::basic_ostream<char, std::char_traits<char> >& std::endl<char, std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&)
0000000000000000 r std::piecewise_construct
0000000000000000 b std::__ioinit
                 U std::basic_ostream<char, std::char_traits<char> >& std::operator<< <std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*)

```
- nmコマンドで生成されたオブジェクトファイルを見ると、未定義(U)となっている部分があるがこれらは先程見たlibstdc++やlibgcc_sで定義されているものである
- g++コマンドのやっていることは、hello.oのようなオブジェクトファイルを生成し、libstdc++やlibgcc_sをリンクするということなのだろう