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

## 6. ブレークポイントの設定を管理する - disable / enable / delete / tbreak

デバッグ中に試すブレークポイントが増えたら、有効・無効や削除を使い分けます。

また、一度だけ停止したい場合には、停止後に自動的に削除されるブレークポイントも設定できます。

### 実験の目的

次の2つの動作を確認します。

- ブレークポイントの設定を残したまま、有効・無効を切り替える。
- 一度停止すると自動的に削除されるブレークポイントを設定する。

### 実験の手順

#### ブレークポイントの有効・無効を切り替える

第5項の実験で設定したブレークポイントを使用します。

まず、現在の設定を確認します。

```gdb
info breakpoints
```

実行結果：

```text
Num     Type           Disp Enb Address    What
7       breakpoint     keep y   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
        breakpoint already hit 4 times
        silent
        printf "tk_cre_tsk: itskpri=%d, stksz=%d\n", pk_ctsk->itskpri, pk_ctsk->stksz
        continue
```

ブレークポイント番号`7`が有効になっています。

また、ブレークポイントへの到達回数と、`commands`で設定したコマンド列も確認できます。

続いて、`disable`でブレークポイントを無効化します。

```gdb
disable 7
info breakpoints
```

実行結果：

```text
Num     Type           Disp Enb Address    What
7       breakpoint     keep n   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
        breakpoint already hit 4 times
        silent
        printf "tk_cre_tsk: itskpri=%d, stksz=%d\n", pk_ctsk->itskpri, pk_ctsk->stksz
        continue
```

`Enb`欄が`y`から`n`に変わりました。

ブレークポイントは無効になりましたが、設定したコマンド列はそのまま残っています。

次に、`enable`で再び有効化します。

```gdb
enable 7
info breakpoints
```

実行結果：

```text
Num     Type           Disp Enb Address    What
7       breakpoint     keep y   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
        breakpoint already hit 4 times
        silent
        printf "tk_cre_tsk: itskpri=%d, stksz=%d\n", pk_ctsk->itskpri, pk_ctsk->stksz
        continue
```

`Enb`欄が`y`に戻りました。

このように、`disable`と`enable`を使えば、ブレークポイントを設定し直すことなく、有効・無効を切り替えられます。

#### 一度だけ停止するブレークポイントを設定する

次に、一度停止すると自動的に削除されるブレークポイントを設定します。

これまで使用してきた`break`の代わりに、`tbreak`（temporary breakpoint）を使用します。

まず、前の実験のブレークポイントを`delete`で削除し、`tbreak`で新たに設定します。

```gdb
delete 7
tbreak tk_cre_tsk
info breakpoints
```

実行結果：

```text
Temporary breakpoint 8 at 0x10000ad0: file kernel/task_mange.c, line 18.
Num     Type           Disp Enb Address    What
8       breakpoint     del  y   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
```

第1項で確認した通常のブレークポイントと比較すると、`Disp`欄が異なります。

| 設定コマンド | Disp | 停止後の動作 |
|---|---|---|
| `break` | `keep` | 設定を保持する |
| `tbreak` | `del` | 設定を削除する |

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

Thread 1 "rp2040.core0" hit Temporary breakpoint 8, tk_cre_tsk (pk_ctsk=0x20000000 <ctsk_ini>)
    at kernel/task_mange.c:18
18          if((pk_ctsk->tskatr & ~TA_RNG3) != (TA_HLNG|TA_USERBUF)) return E_RSATR;
```

通常のブレークポイントと同様に、初期タスクの生成処理で停止しました。

ここで、ブレークポイントの設定が残っているか確認します。

```gdb
info breakpoints
```

実行結果：

```text
No breakpoints, watchpoints, tracepoints, or catchpoints.
```

ブレークポイントが自動的に削除されたことを確認できました。

### 実験結果と考察

ブレークポイントの設定を残したまま有効・無効を切り替える方法と、一度だけ停止する方法を確認できました。

これまでに使用したブレークポイントの管理に関するコマンドをまとめます。

| コマンド | 用途 |
|---|---|
| `info breakpoints` | 設定済みブレークポイントの一覧表示 |
| `clear 関数名` | 指定した場所のブレークポイントを削除 |
| `disable 番号` | 設定を残したまま一時的に無効化 |
| `enable 番号` | 再び有効化 |
| `delete 番号` | 番号を指定して削除 |
| `tbreak 関数名` | 一度停止すると自動削除されるブレークポイントを設定 |

一時的に別の処理を調べたい場合には、ブレークポイントを削除するよりも、`disable`で無効化しておく方が便利です。

特に、条件式やコマンド列を設定したブレークポイントでは、設定を残しておくことで再設定の手間を省けます。

また、調査の開始位置まで一度だけ実行したい場合には、`tbreak`が便利です。

調査の目的に応じて、ブレークポイントの設定だけでなく、その設定をいつまで保持するかも使い分けられます。

---

# 今回分かったこと

今回は、Try Kernelのタスク生成処理を題材に、ブレークポイントの場所・条件・通過回数・到達時の動作を指定する方法を調べました。

実機で確認できた内容をまとめます。

| 確認したこと | GDBコマンド | 実行結果 |
|---|---|---|
| 関数名を指定して停止 | `break tk_cre_tsk` | `task_mange.c`の18行目で停止 |
| ソース行を指定して停止 | `break inittsk.c:32` | 初期タスクの起動処理の直前で停止 |
| 引数の条件で絞り込む | `break ... if`、`condition` | 優先度1と10で異なるタスクの生成処理に停止 |
| 到達回数で絞り込む | `ignore` | 最初の3回を通過し、4回目のタスク生成処理で停止 |
| 値を記録して自動再開 | `commands`、`silent`、`printf`、`continue` | 4回分のタスク生成情報を取得 |
| 設定の有効・無効を切り替える | `disable`、`enable` | 設定を保持したまま有効・無効を切り替え |
| 設定を削除する | `clear`、`delete` | 場所または番号を指定して削除 |
| 一度だけ停止する | `tbreak` | 停止後にブレークポイントが自動的に削除された |

第2回では、ブレークポイントとステップ実行を使って関数内部の処理を追いました。

今回はさらに、ブレークポイントに条件や動作を設定することで、調べたい処理を絞り込んだり、必要な情報を自動的に取得したりできることを確認しました。

**必要な場所だけで停止し、調べたい情報を効率よく集める。**

ブレークポイントは、単にプログラムを停止させるだけでなく、調査の目的に合わせて実行を制御するための機能でもあります。

<!-- TODO: 実機検証後にpublished: trueへ変更する -->
