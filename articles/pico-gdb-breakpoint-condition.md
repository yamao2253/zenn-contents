---
title: "GDBでRaspberry Pi Picoのブレークポイントに条件を設定する"
emoji: "🎯"
type: "tech"
topics: ["gdb", "raspberrypi", "pico", "debug", "embedded"]
published: false
---

# はじめに

CQ出版社『Interface』2023年7月号の特集「ラズパイPicoで1500行 ゼロから作るOS」を読み進めながら、Raspberry Pi PicoでTry Kernelを動かしています。

前回の「[GDBでRaspberry Pi Picoのブレークポイントを設定・管理する](https://zenn.dev/helowota/articles/pico-gdb-breakpoint-management)」では、調べたい場所にブレークポイントを設定する方法と、設定したブレークポイントを管理する方法を確認しました。

しかし、同じ関数が何度も呼び出される場合、停止する場所を指定するだけでは、調べたい処理を十分に絞り込めないことがあります。

例えば、関数の引数が特定の値になったときだけ停止したい場合や、何度か呼び出された後に停止したい場合です。

GDBでは、ブレークポイントに条件や通過回数を指定することで、こうした要求に対応できます。

今回は、Try Kernelのタスク生成処理を題材に、条件式や通過回数を利用して、調べたい関数呼び出しで停止する方法を実機で確認します。

# 使用環境

| 項目 | 内容 |
|---|---|
| OS | Windows 11 |
| シェル | PowerShell 7 |
| ターゲット | Raspberry Pi Pico W（RP2040） |
| デバッグ接続 | CMSIS-DAP / SWD |
| OpenOCD | xPack OpenOCD 0.12.0-7.1 |
| GDB | MSYS2 GDB Multiarch |
| デバッグ対象 | Try Kernel（`part_4/sect_3`） |

開発環境とOpenOCD/GDBの基本的な操作については、以下の記事を参照してください。

- [「ラズパイPicoで1500行 ゼロから作るOS」を2026年に動かす](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)
- [OpenOCDとGDBでRaspberry Pi Picoをデバッグする](https://zenn.dev/helowota/articles/pico-openocd-gdb-debug)
- [GDBでRaspberry Pi Picoの変数をウォッチする](https://zenn.dev/helowota/articles/pico-gdb-variable-watch)

前回と同じプログラムを使用します。今回もTry Kernelのソースコードを変更せず、GDBから動作を調べます。

# 同じ関数が何度も呼ばれる場合にどうするか

例えば、ある関数がプログラムの中で何度も呼び出されているとします。

その関数に通常のブレークポイントを設定すると、呼び出されるたびに停止します。

調べたい呼び出しに到達するまで、引数や変数の値を確認しながら`continue`を繰り返すこともできますが、呼び出し回数が多くなると効率的ではありません。

GDBでは、次のような指定ができます。

- 引数や変数の値が特定の条件を満たした場合だけ停止する。
- ブレークポイントへの到達回数を指定し、その回数に達したときに停止する。

停止する場所だけでなく、どのような条件で停止するかを指定することで、調査の対象を絞り込めます。

今回は、こうした操作を実現するための3つのコマンドを取り上げます。

| コマンド | 用途 |
|---|---|
| `break ... if` | 条件付きブレークポイントを設定する |
| `condition` | 設定済みのブレークポイントの条件を変更する |
| `ignore` | 指定回数だけブレークポイントでの停止をスキップする |

これらのコマンドを使うと、どのように調べたい関数呼び出しを絞り込めるのか、Try Kernelのタスク生成処理を題材に確認していきます。

## 今回の題材：何度も呼ばれるtk_cre_tsk

著者公開ソースの`kernel/inittsk.c`と`application/usermain.c`には、`tk_cre_tsk()`を使ったタスク生成処理があります。

初期タスク、ボタン検出タスク、LED制御タスク2個を生成するため、ソースコード上には合計4箇所の呼び出しが記述されています。

Try Kernelでは、タスクを生成するときに優先度やスタックサイズなどを指定します。

今回生成する4つのタスクについて、それぞれの設定値を整理します。

| 呼び出し順（ソースコード上） | 呼び出し元 | タスク | `itskpri`（優先度） | `stksz`（バイト） |
|---:|---|---|---:|---:|
| 1 | `kernel/inittsk.c` | 初期タスク | 1 | 256 |
| 2 | `application/usermain.c` | ボタン検出タスク | 10 | 1024 |
| 3 | `application/usermain.c` | LED制御タスク1 | 10 | 1024 |
| 4 | `application/usermain.c` | LED制御タスク2 | 10 | 1024 |

生成するタスクは異なりますが、いずれも同じ`tk_cre_tsk()`を呼び出しています。

初期タスクとそれ以外のタスクでは、生成時に指定される優先度が異なります。一方、ボタン検出タスクと2つのLED制御タスクでは、優先度もスタックサイズも同じです。

この違いを利用して、条件式と通過回数によるブレークポイントの使い分けを確認します。

:::message
`task_mange.c`は著者公開ソースのファイル名です。本記事でも元の表記を使用します。
:::

---

# デバッグの準備

まず、PowerShell 7でOpenOCDを起動します。

```powershell
openocd -f interface/cmsis-dap.cfg -c "adapter speed 1000" -f target/rp2040.cfg
```

別のPowerShell 7で`part_4/sect_3`ディレクトリに移動し、GDBを起動します。

```powershell
gdb-multiarch -q build/trykernel.elf
```

GDBからOpenOCDに接続してプログラムをロードします。

```gdb
target remote localhost:3333
monitor reset halt
load
```

以降はGDBのプロンプトでコマンドを入力します。

各項で設定したブレークポイントは、必要に応じて整理しながら進めます。

なお、ブレークポイント番号はGDBが設定時に割り当てるため、操作履歴によって異なります。本記事の実行結果と番号が異なる場合は、`info breakpoints`で確認した番号を使用してください。

---

# 実際にGDBで停止条件を指定する

## 1. 条件を満たす場合だけ停止する - break ... if / condition

関数が繰り返し呼ばれる場合、その引数や変数の値を条件として、特定の呼び出しだけを調べたいことがあります。

GDBでは、ブレークポイントに条件式を設定できます。

このようなブレークポイントを条件付きブレークポイント（conditional breakpoint）と呼びます。

今回は、タスク生成時に指定される優先度を条件にして、調べたい関数呼び出しを絞り込みます。

### 確認すること

次の3つを確認します。

- ブレークポイントに条件式を設定できること。
- 設定済みのブレークポイントの条件を変更できること。
- 条件式を変更することで、同じ関数でも異なる呼び出しで停止できること。

### 操作手順

#### 優先度10のタスク生成で停止する

今回生成する4つのタスクのうち、最初のタスクの優先度は`1`で、2つ目以降のタスクの優先度はすべて`10`です。

そこで、優先度`10`を停止条件に指定すれば、最初の初期タスクの生成処理では停止せず、2つ目のボタン検出タスクの生成処理で停止するはずです。

次のように条件付きブレークポイントを設定します。

```gdb
break tk_cre_tsk if pk_ctsk->itskpri == 10
```

実行結果：

```text
Breakpoint 1 at 0x10000ad0: file kernel/task_mange.c, line 18.
Note: automatically using hardware breakpoints for read-only addresses.
```

`break`コマンドの`if`以降に条件式を指定すると、ブレークポイントに到達したときに、その条件式が真の場合だけ停止します。

今回指定した条件式は、次のとおりです。

```c
pk_ctsk->itskpri == 10
```

`pk_ctsk`は、`tk_cre_tsk()`の引数として渡されるタスク生成情報へのポインタです。

その構造体のメンバである`itskpri`を参照し、生成するタスクの優先度が`10`であるかを判定します。

続いてリセットして実行します。

```gdb
monitor reset halt
continue
```

実行結果：

```text
[rp2040.core0] halted due to debug-request, current mode: Thread
xPSR: 0xf1000000 pc: 0x000000ea msp: 0x20041f00
[rp2040.core1] halted due to debug-request, current mode: Thread
xPSR: 0xf1000000 pc: 0x000000ea msp: 0x20041f00
Continuing.

Thread 1 "rp2040.core0" hit Breakpoint 1, tk_cre_tsk (pk_ctsk=0x20000028 <ctsk_btn>)
    at kernel/task_mange.c:18
18          if((pk_ctsk->tskatr & ~TA_RNG3) != (TA_HLNG|TA_USERBUF)) return E_RSATR;
```

引数`pk_ctsk`が`ctsk_btn`を指していることから、ボタン検出タスクの生成処理で停止したことが分かります。

条件に使用した優先度の値も確認します。

```gdb
print pk_ctsk->itskpri
```

実行結果：

```text
$1 = 10
```

指定した条件と一致しています。

さらに、`backtrace`コマンドで現在の関数の呼び出し元を確認します。

```gdb
backtrace
```

実行結果：

```text
#0  tk_cre_tsk (pk_ctsk=0x20000028 <ctsk_btn>) at kernel/task_mange.c:18
#1  0x10001904 in usermain () at application/usermain.c:139
#2  0x1000091a in initsk (stacd=0, exinf=0x0) at kernel/inittsk.c:25
#3  0x00000000 in ?? ()
```

`#0`は現在停止している関数、`#1`はその呼び出し元を示しています。

`tk_cre_tsk()`が`application/usermain.c`の139行目から呼び出されていることが分かります。

最初に呼び出される初期タスクの生成処理では、優先度が`1`であるため、指定した条件に一致しません。

そのため、初期タスクの生成処理ではGDBのプロンプトに制御が戻ることなく、条件に一致するボタン検出タスクの生成処理まで実行されました。

条件に一致する場合は停止し、一致しない場合は通過することを確認できました。

## 2. 指定回数だけ通過してから停止する - ignore

前項では、引数の値を条件にして停止する呼び出しを絞り込みました。

一方、同じ関数が繰り返し呼ばれる場合に、何回目の呼び出しで停止したいかが分かっていることもあります。

GDBの`ignore`コマンドを使うと、指定した回数だけブレークポイントでの停止をスキップできます。

今回の題材では、`tk_cre_tsk()`は次の順序で呼び出されます。

| 呼び出し順 | 生成するタスク |
|---:|---|
| 1 | 初期タスク |
| 2 | ボタン検出タスク |
| 3 | LED制御タスク1 |
| 4 | LED制御タスク2 |

この呼び出し順を利用して、今回はLED制御タスク2の生成処理で停止する方法を確認します。

### 確認すること

次の2つを確認します。

- ブレークポイントでの停止をスキップする回数を指定できること。
- 最初の3回を通過し、4回目の`tk_cre_tsk()`の呼び出しで停止できること。

### 操作手順

まず、前項で設定した条件付きブレークポイントを削除し、条件を指定せずにブレークポイントを設定します。

```gdb
clear tk_cre_tsk
break tk_cre_tsk
info breakpoints
```

実行結果：

```text
Deleted breakpoint 1
Breakpoint 2 at 0x10000ad0: file kernel/task_mange.c, line 18.
Num     Type           Disp Enb Address    What
2       breakpoint     keep y   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
```

ブレークポイント番号`2`が設定されました。

LED制御タスク2は4回目の`tk_cre_tsk()`の呼び出しで生成されます。

そこで、最初の3回の停止をスキップするように設定すれば、4回目のLED制御タスク2の生成処理で停止するはずです。

次のように`ignore`コマンドで設定します。

```gdb
ignore 2 3
```

実行結果：

```text
Will ignore next 3 crossings of breakpoint 2.
```

`ignore`の第1引数はブレークポイント番号、第2引数は停止せずに通過する回数です。

| 引数 | 指定値 | 意味 |
|---|---:|---|
| 第1引数 | 2 | 対象のブレークポイント番号 |
| 第2引数 | 3 | 停止をスキップする回数 |

今回の指定では、ブレークポイントへの最初の3回の到達では停止せず、4回目の到達で停止することになります。

`info breakpoints`コマンドで設定内容を確認します。

```gdb
info breakpoints
```

実行結果：

```text
Num     Type           Disp Enb Address    What
2       breakpoint     keep y   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
        ignore next 3 hits
```

`ignore next 3 hits`と表示されていることから、次の3回の停止をスキップする設定になっていることが分かります。

続いてリセットして実行します。

```gdb
monitor reset halt
continue
```

実行結果：

```text
[rp2040.core0] halted due to debug-request, current mode: Thread
xPSR: 0xf1000000 pc: 0x000000ea msp: 0x20041f00
[rp2040.core1] halted due to debug-request, current mode: Thread
xPSR: 0xf1000000 pc: 0x000000ea msp: 0x20041f00
Continuing.

Thread 1 "rp2040.core0" hit Breakpoint 2, tk_cre_tsk (pk_ctsk=0x20000050 <ctsk_led2>)
    at kernel/task_mange.c:18
18          if((pk_ctsk->tskatr & ~TA_RNG3) != (TA_HLNG|TA_USERBUF)) return E_RSATR;
```

引数`pk_ctsk`が`ctsk_led2`を指していることから、LED制御タスク2の生成処理で停止したことが分かります。

前項と同様に、生成するタスクの優先度と関数の呼び出し元を確認します。

```gdb
print pk_ctsk->itskpri
```

実行結果：

```text
$3 = 10
```

```gdb
backtrace
```

実行結果：

```text
#0  tk_cre_tsk (pk_ctsk=0x20000050 <ctsk_led2>) at kernel/task_mange.c:18
#1  0x10001938 in usermain () at application/usermain.c:147
#2  0x1000091a in initsk (stacd=0, exinf=0x0) at kernel/inittsk.c:25
#3  0x00000000 in ?? ()
```

優先度はボタン検出タスクと同じ`10`ですが、`tk_cre_tsk()`の呼び出し元は`application/usermain.c`の147行目です。

前項で停止したボタン検出タスクの生成処理とは、異なる呼び出しで停止していることが分かります。

最後に、再度`info breakpoints`コマンドを実行し、ブレークポイントへの到達回数と残りのスキップ回数を確認します。

```gdb
info breakpoints
```

実行結果：

```text
Num     Type           Disp Enb Address    What
2       breakpoint     keep y   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
        breakpoint already hit 4 times
```

`breakpoint already hit 4 times`と表示されていることから、ブレークポイントへの到達回数が4回であることが分かります。

また、設定直後に表示されていた`ignore next 3 hits`がなくなっていることから、指定した3回のスキップ回数が消費されたことも分かります。

停止時の引数や呼び出し元に加え、GDBが管理している到達回数からも、4回目の呼び出しで停止したことを確認できました。

### 結果と考察

今回の実行結果を、ソースコード上の呼び出し順と照らし合わせると、次のようになります。

| 呼び出し順 | 生成するタスク | 動作 |
|---:|---|---|
| 1 | 初期タスク | 通過 |
| 2 | ボタン検出タスク | 通過 |
| 3 | LED制御タスク1 | 通過 |
| 4 | LED制御タスク2 | 停止 |

最初の3回を通過し、4回目の呼び出しで停止できました。

前項では、優先度を条件にして調べたい関数呼び出しを絞り込みました。

今回は、引数や変数の値を条件式で判定することなく、ブレークポイントへの到達回数を利用して目的の呼び出しまで実行できました。

:::message
`ignore 5 3`は、3回目に停止するという指定ではありません。

次の3回の停止をスキップする指定なので、条件のないブレークポイントでは4回目に停止します。

また、通過するたびに残りのスキップ回数は減少します。同じ操作を繰り返す場合は、必要に応じて`ignore`を設定し直します。
:::

**呼び出し順が分かっている場合は、通過回数を指定することで、調べたい呼び出しまで実行できます。**

---

# 条件式と通過回数を使い分ける

今回使用したコマンドを、用途ごとに整理します。

| 用途 | コマンド |
|---|---|
| 条件付きブレークポイントを設定したい | `break ... if` |
| 設定済みのブレークポイントの条件を変更したい | `condition` |
| 指定回数だけ停止をスキップしたい | `ignore` |
| 設定されている条件などを確認したい | `info breakpoints` |

条件式と通過回数では、調べたい呼び出しを特定するために使用する情報が異なります。

| 方法 | 判断に使用する情報 | 適した場面 |
|---|---|---|
| 条件式 | 引数や変数の値 | 特定の値や状態になったときに停止したい |
| 通過回数 | ブレークポイントへの到達回数 | 呼び出し順や到達回数が分かっている |

例えば、ある変数が特定の値になるタイミングを調べたい場合には、条件付きブレークポイントが適しています。

一方、決まった順序で繰り返し実行される処理のうち、特定の回だけを調べたい場合には、`ignore`が便利です。

ただし、処理の順序や回数が変わる可能性がある場合、通過回数による指定では意図した呼び出しで停止できないことがあります。

調べたい処理の特徴に応じて、停止条件を使い分けることが重要です。

---

# 今回分かったこと

今回は、Try Kernelのタスク生成処理を題材に、ブレークポイントの停止条件を指定する方法を調べました。

実機で確認できた内容をまとめます。

| 確認したこと | GDBコマンド | 実行結果 |
|---|---|---|
| 条件を満たす場合だけ停止する | `break ... if` | 優先度1の初期タスク生成で停止 |
| 設定済みの停止条件を変更する | `condition` | 優先度10のボタン検出タスク生成で停止 |
| 指定回数だけ通過してから停止する | `ignore` | 4回目のLED制御タスク2の生成で停止 |

前回は、調べたい場所にブレークポイントを設定し、その設定を管理する方法を確認しました。

今回は、停止する場所に加えて、条件式や通過回数を指定することで、同じ関数の中から調べたい呼び出しを絞り込めることを確認しました。

**どこで停止するかだけでなく、どのような条件で停止するか。**

ブレークポイントに条件を設定することで、目的の処理に到達するまで手動で操作を繰り返す必要がなくなり、効率よくデバッグを進められます。

ところが、関数が呼び出されるたびに引数や変数の値を確認したい場合には、停止する呼び出しを絞り込むのではなく、情報を記録すること自体が目的となります。

次回は、ブレークポイントにコマンドを登録し、必要な情報を表示して自動的に実行を再開する方法を調べます。

<!-- TODO:
- 新しいGDBセッションで本記事の操作手順を最初から再確認する。
- ブレークポイント番号と実行ログを再検証結果に差し替える。
- 条件変更前後のブレークポイント設定を確認する。
- 再検証後にZennプレビューを確認し、published: trueへ変更する。
-->
