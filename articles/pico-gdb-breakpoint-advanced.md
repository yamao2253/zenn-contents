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

## 1. 関数名を指定して停止する - break / info breakpoints

第2回では`main()`から`step`で`tk_cre_tsk()`の内部に入りました。

調べたい関数が決まっているなら、最初から関数名を指定して停止できます。

```gdb
break tk_cre_tsk
```

実行結果：

```text
Breakpoint 1 at 0x10000ad0: file kernel/task_mange.c, line 18.
Note: automatically using hardware breakpoints for read-only addresses.
```

`kernel/task_mange.c`の18行目に、ブレークポイント番号`1`が設定されました。

また、読み出し専用アドレスに対して、GDBが自動的にハードウェアブレークポイントを使用することも表示されています。

設定済みのブレークポイントを一覧表示します。

```gdb
info breakpoints
```

実行結果：

```text
Num     Type           Disp Enb Address    What
1       breakpoint     keep y   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
```

`info breakpoints`では、設定済みのブレークポイントの状態を確認できます。

| 項目 | 表示内容 | 意味 |
|---|---|---|
| Num | `1` | ブレークポイント番号 |
| Type | `breakpoint` | ブレークポイントの種類 |
| Disp | `keep` | 停止後も設定を保持する |
| Enb | `y` | 有効になっている |
| Address | `0x10000ad0` | 設定されたアドレス |
| What | `in tk_cre_tsk at kernel/task_mange.c:18` | 関数名とソースコード上の位置 |

ブレークポイント番号は、後で設定を変更したり、削除したりするときに使用します。

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

Thread 1 "rp2040.core0" hit Breakpoint 1, tk_cre_tsk (pk_ctsk=0x20000000 <ctsk_ini>)
    at kernel/task_mange.c:18
18          if((pk_ctsk->tskatr & ~TA_RNG3) != (TA_HLNG|TA_USERBUF)) return E_RSATR;
```

第2回で`step`を使って関数内部に入ったときと同じ、`kernel/task_mange.c`の18行目で停止しました。

引数`pk_ctsk`が`ctsk_ini`を指していることから、初期タスクの生成処理で停止したことも分かります。

`main()`からステップ実行で処理を追わなくても、**関数名を指定するだけで、調べたい関数の内部に直接停止できる**ことを確認できました。

## 2. 関数の途中で停止する - break ファイル名:行番号

関数の入口ではなく、特定の処理の直前で止めたい場合もあります。

書籍P.74の`kernel/inittsk.c`では、32行目に初期タスクの実行を開始する処理があります。

```c
tskid_ini = tk_cre_tsk(&ctsk_ini);  // 初期タスク生成
tk_sta_tsk(tskid_ini, 0);           // 初期タスク実行
```

関数名で設定したブレークポイントを解除してから、ソースの位置を指定します。

```gdb
clear tk_cre_tsk
break inittsk.c:32
```

実行結果：

```text
Deleted breakpoint 1
Breakpoint 2 at 0x1000093e: file kernel/inittsk.c, line 32.
```

`clear`は、指定した場所に設定されているブレークポイントを削除するコマンドです。

今回は`tk_cre_tsk()`に設定していたブレークポイント番号`1`を削除し、`inittsk.c`の32行目にブレークポイント番号`2`を設定しました。

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

Thread 1 "rp2040.core0" hit Breakpoint 2, main () at kernel/inittsk.c:32
32          tk_sta_tsk(tskid_ini, 0);                       // 初期タスク実行
```

初期タスクの生成処理を終え、`tk_sta_tsk()`を呼び出す直前で停止しました。

第2回ではステップ実行でこの位置まで進めましたが、今回はブレークポイントを設定することで直接停止できました。

このように、**ファイル名と行番号を指定すれば、関数の途中にある特定の処理でも停止できます。**

同名のファイルが複数ある場合は、パスを含めて指定する方法もあります。

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

まず、前の実験の条件付きブレークポイントを解除し、条件を指定せずにブレークポイントを設定します。

---

特定の関数が繰り返し呼ばれる場合、条件式ではなく、ブレークポイントへの到達回数で停止位置を絞り込む方法もあります。

今回は、「最初の3回を通過し、4回目の`tk_cre_tsk()`の呼び出しで停止する」という実験を行います。

まず、前の実験の条件付きブレークポイントを解除し、条件を指定せずにブレークポイントを設定します。

---

特定の関数が繰り返し呼ばれる場合、条件式ではなく、ブレークポイントへの到達回数で停止位置を絞り込む方法もあります。

今回は、**最初の3回を通過し、4回目の`tk_cre_tsk()`の呼び出しで停止する**ことを目指します。

まず、前の実験の条件付きブレークポイントを解除し、条件を指定せずにブレークポイントを設定します。

---

特定の関数が繰り返し呼ばれる場合、条件式ではなく、ブレークポイントへの到達回数で停止位置を絞り込む方法もあります。

今回の実験の目的は、**最初の3回を通過し、4回目の`tk_cre_tsk()`の呼び出しで停止すること**です。

まず、前の実験の条件付きブレークポイントを解除し、条件を指定せずにブレークポイントを設定します。

---

特定の関数が繰り返し呼ばれる場合、条件式ではなく、ブレークポイントへの到達回数で停止位置を絞り込む方法もあります。

今回は、次の動作を実現します。

**最初の3回を通過し、4回目の`tk_cre_tsk()`の呼び出しで停止する。**

まず、前の実験の条件付きブレークポイントを解除し、条件を指定せずにブレークポイントを設定します。

---

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

## 5. 停止せずに値を記録する - commands / silent / printf

ブレークポイントは、停止するためだけのものではありません。

GDBの`commands`で、ブレークポイントに到達したときに実行するコマンドを設定できます。

今回は、`tk_cre_tsk()`が呼ばれるたびに、引数の優先度とスタックサイズをGDBの画面に表示させます。

まず、前の実験のブレークポイントを解除し、改めて設定します。

```gdb
clear tk_cre_tsk
break tk_cre_tsk
```

続いて、ブレークポイントに対する動作を指定します。

```gdb
commands
silent
printf "tk_cre_tsk: itskpri=%d, stksz=%d\n", pk_ctsk->itskpri, pk_ctsk->stksz
continue
end
```

`commands`は直前に設定したブレークポイントにコマンドを結び付けます。

- `silent`：通常のブレークポイント停止メッセージを抑制する
- `printf`：指定した値をGDBの画面に出力する
- `continue`：出力後に実行を再開する
- `end`：コマンド列の指定を終了する

実行を再開します。

```gdb
monitor reset halt
continue
```

実行結果：

<!-- TODO: 生成情報のログを実機で取得する。4回分が取得できるかも確認する -->

この方法なら、ソースコードにデバッグ用の`printf`を追加することなく、**関数に渡された値を記録できる**可能性があります。

ただし、`continue`が自動実行されるため、処理が繰り返し続く場合はGDBの入力待ちに戻らないことがあります。必要に応じて`Ctrl+C`で中断して確認します。

また、この出力はターゲット側のUARTではなく、**GDBの画面に表示されるログ**です。

## 6. ブレークポイントの設定を管理する

デバッグ中に試すブレークポイントが増えたら、有効・無効や削除を使い分けます。

| コマンド | 用途 |
|---|---|
| `info breakpoints` | 設定済みブレークポイントの一覧表示 |
| `clear 関数名` | 指定した場所のブレークポイントを解除 |
| `disable 番号` | 設定を残したまま一時的に無効化 |
| `enable 番号` | 再び有効化 |
| `delete 番号` | 番号を指定して削除 |
| `tbreak 関数名` | 一度停止すると自動削除されるブレークポイントを設定 |

ブレークポイント番号はセッションごとに異なるため、`info breakpoints`で確認してから指定します。

`disable`や`enable`は設定を残したまま切り替えますが、`clear`や`delete`は設定を取り除きます。

---

# 今回分かったこと

今回は、ブレークポイントの場所・条件・通過回数・到達時の動作を指定する方法を整理しました。

実機で確認する内容をまとめます。

| 確認すること | GDBコマンド | 確認する内容 |
|---|---|---|
| 関数の入口で停止 | `break tk_cre_tsk` | 関数の先頭で停止するか |
| ソース行を指定 | `break inittsk.c:32` | 行番号で指定できるか |
| 引数の条件で絞る | `break ... if ...` | 優先度`1`で停止するか |
| 呼び出し回数で絞る | `ignore` | 3回通過後に停止するか |
| ログを記録 | `commands`、`printf` | 引数の値を出力できるか |
| 設定を管理 | `info breakpoints`など | 状態を確認・切替できるか |

<!-- TODO: 実機検証後、確認項目と実測結果を対応づけて表を更新する -->

**必要な場所だけで停止し、調べたい情報を効率よく集める**ことが今回の狙いです。

<!-- TODO: 実機検証後にpublished: trueへ変更する -->
