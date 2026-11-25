---
title: "GDBでRaspberry Pi Picoのブレークポイントを使いこなす"
emoji: "🎯"
type: "tech"
topics: ["gdb", "raspberrypi", "pico", "debug", "embedded"]
published: false
---

# はじめに

CQ出版社『Interface』2023年7月号の特集「ラズパイPicoで1500行 ゼロから作るOS」を読み進めながら、Raspberry Pi PicoでTry Kernelを動かしています。

前回の「[GDBでRaspberry Pi Picoのブレークポイントとステップ実行を学ぶ](https://zenn.dev/helowota/articles/pico-gdb-breakpoint-step)」では、`step`、`next`、`finish`を使って、初期タスクを生成する`tk_cre_tsk()`の内部を追いました。

しかし、調べたい関数が何度も呼ばれる場合、毎回停止して操作を繰り返すのは面倒です。

今回は、**どの場所で、どんな条件で、何回目に停止するか**を指定する方法と、停止せずに情報を記録する方法を調べます。

単なるコマンドの紹介ではなく、Try Kernelのタスク生成処理を題材に、目的に応じてブレークポイントを使い分けていきます。

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

第2回と同じプログラムを使用します。今回もTry Kernelのソースコードを変更せず、GDBから動作を調べます。

# ブレークポイントをもっと活用したい

`break main`は、`main()`の入口で停止するための指定です。

ところが実際のデバッグでは、特定の関数だけでなく、**関数の途中で止めたい、引数によって止めたい、何度か通過してから止めたい**といった状況があります。

GDBでは、ブレークポイントを設定する場所や条件を変えることで、調査したい範囲を絞り込めます。

## 今回の題材：何度も呼ばれるtk_cre_tsk

著者公開ソースの`kernel/inittsk.c`と`application/usermain.c`には、`tk_cre_tsk()`を使ったタスク生成処理があります。

ソース上では、初期タスク・ボタン検出タスク・LED制御タスク2個を生成するため、**合計4回の呼び出し**が記述されています。

| 呼び出し元 | タスク | `itskpri` | `stksz`（バイト） |
|---|---|---:|---:|
| `kernel/inittsk.c` | 初期タスク | 1 | 256 |
| `application/usermain.c` | ボタン検出タスク | 10 | 1024 |
| `application/usermain.c` | LED制御タスク1 | 10 | 1024 |
| `application/usermain.c` | LED制御タスク2 | 10 | 1024 |

この4回は**ソースコード上の呼び出し箇所から分かること**であり、GDBで実際に4回通過できるかは、実機で確認します。

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

以降はGDBのプロンプトでコマンドを入力します。各実験は、前の実験で設定したブレークポイントを整理してから行います。

---

# 実際にGDBでブレークポイントを使う

## 3. 条件を満たす場合だけ停止する - break ... if / condition

関数が繰り返し呼ばれる場合、毎回止める必要はありません。

GDBでは、ブレークポイントに条件を設定し、特定の条件を満たした場合だけ停止させることができます。

今回は、タスク生成時に指定される優先度を条件にして、停止する位置を絞り込みます。

### 優先度1のタスク生成で停止する

まず、優先度`1`が指定されたタスク生成で停止するように設定します。

```gdb
clear inittsk.c:32
break tk_cre_tsk if pk_ctsk->itskpri == 1
```

実行結果：

```text
Deleted breakpoint 2
Breakpoint 3 at 0x10000ad0: file kernel/task_mange.c, line 18.
```

`break`コマンドの`if`以降に条件式を指定すると、その式が真の場合に停止します。

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

Thread 1 "rp2040.core0" hit Breakpoint 3, tk_cre_tsk (pk_ctsk=0x20000000 <ctsk_ini>)
    at kernel/task_mange.c:18
18          if((pk_ctsk->tskatr & ~TA_RNG3) != (TA_HLNG|TA_USERBUF)) return E_RSATR;
```

引数`pk_ctsk`が`ctsk_ini`を指していることから、初期タスクの生成処理で停止したことが分かります。

条件に使用した優先度の値も確認します。

```gdb
print pk_ctsk->itskpri
```

実行結果：

```text
$1 = 1
```

指定した条件と一致しています。

ただし、初期タスクは最初に生成されるため、条件を指定しないブレークポイントでも同じところで停止します。

これだけでは、条件に一致しない場合に停止せず通過することを確認できません。

### 条件を変更して優先度10のタスク生成で停止する

次に、優先度`10`が指定されたタスク生成で停止するように条件を変更します。

設定済みのブレークポイントの条件を変更するには、`condition`コマンドを使用します。

```gdb
condition 3 pk_ctsk->itskpri == 10
info breakpoints
```

実行結果：

```text
Num     Type           Disp Enb Address    What
3       breakpoint     keep y   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
        stop only if pk_ctsk->itskpri == 10
        breakpoint already hit 1 time
```

ブレークポイント番号とアドレスは変わらず、停止条件だけが変更されています。

再びリセットして実行します。

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

Thread 1 "rp2040.core0" hit Breakpoint 3, tk_cre_tsk (pk_ctsk=0x20000028 <ctsk_btn>)
    at kernel/task_mange.c:18
18          if((pk_ctsk->tskatr & ~TA_RNG3) != (TA_HLNG|TA_USERBUF)) return E_RSATR;
```

先ほどと同じ`task_mange.c`の18行目で停止しましたが、引数`pk_ctsk`が指しているのは`ctsk_ini`ではなく`ctsk_btn`です。

優先度を確認します。

```gdb
print pk_ctsk->itskpri
```

実行結果：

```text
$2 = 10
```

今回は、優先度`10`が指定されたボタン検出タスクの生成処理で停止しました。

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

`tk_cre_tsk()`が`application/usermain.c`の139行目から呼び出されていることが分かります。

初期タスクの生成処理では停止せず、その後のボタン検出タスクの生成処理まで実行されたことを確認できました。

### 条件による停止位置の違い

2つの実験結果を比較します。

| 項目 | 優先度1を指定 | 優先度10を指定 |
|---|---|---|
| 条件式 | `pk_ctsk->itskpri == 1` | `pk_ctsk->itskpri == 10` |
| ブレークポイント番号 | 3 | 3 |
| 停止アドレス | `0x10000ad0` | `0x10000ad0` |
| 引数が指す構造体 | `ctsk_ini` | `ctsk_btn` |
| 生成するタスク | 初期タスク | ボタン検出タスク |

同じアドレスにブレークポイントを設定していても、条件によって異なる関数呼び出しで停止することが確認できました。

このように、**条件付きブレークポイントを使えば、繰り返し呼ばれる関数の中から、調べたい条件に一致する呼び出しだけを選んで停止できます。**

なお、今回はタスクの優先度で絞り込んでいるのであって、タスクIDを判定しているわけではありません。

## 4. 指定回数だけ通過してから停止する - ignore

特定の関数が繰り返し呼ばれる場合、条件式ではなく、ブレークポイントへの到達回数で停止位置を絞り込む方法もあります。

`ignore`は、指定したブレークポイントについて、次の指定回数だけ停止せずに通過するコマンドです。

### 実験の目的

最初の3回を通過し、4回目の`tk_cre_tsk()`の呼び出しで停止することを確認します。

### 実験の手順

まず、前の実験の条件付きブレークポイントを解除し、条件を指定せずにブレークポイントを設定します。

```gdb
clear tk_cre_tsk
break tk_cre_tsk
info breakpoints
```

実行結果：

```text
Deleted breakpoint 3
Breakpoint 5 at 0x10000ad0: file kernel/task_mange.c, line 18.
Num     Type           Disp Enb Address    What
5       breakpoint     keep y   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
```

ブレークポイント番号`5`が設定されました。

このブレークポイントに対して、最初の3回を通過するように設定します。

```gdb
ignore 5 3
```

実行結果：

```text
Will ignore next 3 crossings of breakpoint 5.
```

`ignore`の第1引数はブレークポイント番号、第2引数は停止せずに通過する回数です。

ブレークポイント番号は設定順によって変わるため、`info breakpoints`で確認してから指定します。

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

Thread 1 "rp2040.core0" hit Breakpoint 5, tk_cre_tsk (pk_ctsk=0x20000050 <ctsk_led2>)
    at kernel/task_mange.c:18
18          if((pk_ctsk->tskatr & ~TA_RNG3) != (TA_HLNG|TA_USERBUF)) return E_RSATR;
```

引数`pk_ctsk`が`ctsk_led2`を指していることから、LED制御タスク2の生成処理で停止したことが分かります。

今回の実行結果を、ソースコード上の呼び出し順と照らし合わせると、次のようになります。

| 呼び出し順 | 生成するタスク | 動作 |
|---|---|---|
| 1 | 初期タスク | 通過 |
| 2 | ボタン検出タスク | 通過 |
| 3 | LED制御タスク1 | 通過 |
| 4 | LED制御タスク2 | 停止 |

最初の3回を通過し、4回目の呼び出しで停止できました。

注意したいのは、`ignore 5 3`が3回目に停止するという指定ではなく、3回分の停止をスキップする指定であることです。

**呼び出し順が分かっている場合は、条件式を指定しなくても、調べたい呼び出しまで実行できます。**

