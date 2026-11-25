---
title: "GDBでRaspberry Pi Picoのブレークポイントを設定・管理する"
emoji: "🎯"
type: "tech"
topics: ["gdb", "raspberrypi", "pico", "debug", "embedded"]
published: false
---

# はじめに

CQ出版社『Interface』2023年7月号の特集「ラズパイPicoで1500行 ゼロから作るOS」を読み進めながら、Raspberry Pi PicoでTry Kernelを動かしています。

前回の「[GDBでRaspberry Pi Picoのプログラムを追う — ブレークポイントとステップ実行](https://zenn.dev/helowota/articles/pico-gdb-breakpoint-step)」では、`break`でプログラムを停止し、`step`、`next`、`finish`を使って処理の流れを追いました。

今回は、ブレークポイントそのものに焦点を当てます。

実際のデバッグでは、調べたい関数や処理が決まっていれば、その場所にブレークポイントを設定することで、目的の位置まで直接実行できます。

また、調査の進み具合に応じて、設定したブレークポイントを一時的に無効にしたり、削除したりすることもあります。

Try Kernelのタスク生成処理を題材に、ブレークポイントの設定方法と管理方法を実機で確認します。

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

これまでの記事でも使用してきた`break`は、GDBでデバッグを行う上で基本となるコマンドです。

実際のデバッグでは、調査の目的に応じて、次のような操作が必要になります。

- 調べたい関数で停止する。
- 関数の途中にある特定の処理で停止する。
- 設定したブレークポイントを一時的に無効にする。
- 不要になったブレークポイントを削除する。
- 一度だけ停止するブレークポイントを設定する。

GDBには、こうした操作を行うためのコマンドが用意されています。

## 今回の題材：Try Kernelの初期タスク生成

第2回と同じく、`kernel/inittsk.c`の初期タスク生成処理を題材とします。

初期タスクを生成する`tk_cre_tsk()`と、その実行を開始する`tk_sta_tsk()`の呼び出し箇所を利用して、ブレークポイントの設定方法と管理方法を確認します。

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

前の項で設定したブレークポイントは、必要に応じて整理しながら進めます。

なお、ブレークポイント番号はGDBが設定時に割り当てるため、操作履歴によって異なります。本記事の実行結果と番号が異なる場合は、`info breakpoints`で確認した番号を使用してください。

---

# 実際にGDBでブレークポイントを設定・管理する

## 1. 関数名を指定して停止する - break / info breakpoints

第2回では`main()`から`step`で`tk_cre_tsk()`の内部に入りました。

調べたい関数が決まっているなら、最初から関数名を指定してブレークポイントを設定できます。

### 確認すること

次の2つを確認します。

- 関数名を指定して、調べたい関数で停止できること。
- 設定したブレークポイントの位置や状態を確認できること。

### 操作手順

`tk_cre_tsk()`にブレークポイントを設定します。

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

続いて、設定済みのブレークポイントを一覧表示します。

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

### 結果と考察

第2回で`step`を使って関数内部に入ったときと同じ、`kernel/task_mange.c`の18行目で停止しました。

引数`pk_ctsk`が`ctsk_ini`を指していることから、初期タスクの生成処理で停止したことも分かります。

`main()`からステップ実行で処理を追わなくても、**関数名を指定するだけで、調べたい関数の内部に直接停止できる**ことを確認できました。

また、`info breakpoints`を使用することで、ブレークポイントがどこに設定され、どのような状態になっているかを確認できます。

## 2. 関数の途中で停止する - break ファイル名:行番号 / clear

関数の入口ではなく、関数の途中にある特定の処理の直前で止めたい場合もあります。

GDBでは、関数名だけでなく、ソースコードのファイル名と行番号を指定してブレークポイントを設定できます。

### 確認すること

次の2つを確認します。

- `clear`を使って、指定した場所のブレークポイントを削除できること。
- ファイル名と行番号を指定して、関数の途中にある特定の処理で停止できること。

### 操作手順

書籍P.74の`kernel/inittsk.c`では、32行目に初期タスクの実行を開始する処理があります。

```c
tskid_ini = tk_cre_tsk(&ctsk_ini);  // 初期タスク生成
tk_sta_tsk(tskid_ini, 0);           // 初期タスク実行
```

今回は、初期タスクの生成を終え、その実行を開始する直前で停止させます。

第1項で設定したブレークポイントを削除してから、新たにファイル名と行番号を指定します。

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

### 結果と考察

初期タスクの生成処理を終え、`tk_sta_tsk()`を呼び出す直前で停止しました。

第2回ではステップ実行でこの位置まで進めましたが、今回はブレークポイントを設定することで直接停止できました。

第1項と今回のブレークポイントの設定方法を比較します。

| 設定方法 | コマンド | 停止位置 |
|---|---|---|
| 関数名を指定 | `break tk_cre_tsk` | `tk_cre_tsk()`の先頭付近 |
| ファイル名と行番号を指定 | `break inittsk.c:32` | `main()`内の初期タスク実行処理の直前 |

関数全体を調べたい場合は関数名を、関数内の特定の処理を調べたい場合はファイル名と行番号を指定すると便利です。

同名のファイルが複数ある場合は、パスを含めて指定する方法もあります。

また、`clear`を使用することで、ブレークポイント番号ではなく、設定した場所を指定して削除できることも確認できました。

## 3. ブレークポイントの有効・無効を切り替える - disable / enable

複数のブレークポイントを設定してデバッグしていると、一時的に特定のブレークポイントでは停止させたくない場合があります。

そのたびにブレークポイントを削除して設定し直すのではなく、`disable`と`enable`を使用することで、設定を残したまま有効・無効を切り替えられます。

### 確認すること

次の3つを確認します。

- ブレークポイントの設定を残したまま、有効・無効を切り替えられること。
- 無効にしたブレークポイントでは停止しないこと。
- 再び有効にすると、そのブレークポイントで停止すること。

### 操作手順

#### ブレークポイントを追加する

第2項で設定した`inittsk.c`の32行目のブレークポイントを残したまま、`tk_cre_tsk()`にもブレークポイントを設定します。

```gdb
break tk_cre_tsk
info breakpoints
```

実行結果：

```text
Breakpoint 3 at 0x10000ad0: file kernel/task_mange.c, line 18.
Num     Type           Disp Enb Address    What
2       breakpoint     keep y   0x1000093e in main at kernel/inittsk.c:32
        breakpoint already hit 1 time
3       breakpoint     keep y   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
```

現在、2つのブレークポイントが設定されています。

| ブレークポイント番号 | 設定位置 | 停止する処理 |
|---|---|---|
| `2` | `inittsk.c:32` | 初期タスクの実行開始直前 |
| `3` | `task_mange.c:18` | タスク生成関数の先頭付近 |

プログラムをリセットして実行した場合、初期タスクの生成処理が先に行われるため、ブレークポイント番号`3`、`2`の順に到達します。

今回は、この2つのブレークポイントを利用して、有効・無効の切り替えによる停止位置の違いを確認します。

#### ブレークポイントを無効にする

`disable`にブレークポイント番号を指定して、ブレークポイントを無効にします。

```gdb
disable 3
info breakpoints
```

実行結果：

```text
Num     Type           Disp Enb Address    What
2       breakpoint     keep y   0x1000093e in main at kernel/inittsk.c:32
        breakpoint already hit 1 time
3       breakpoint     keep n   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
```

ブレークポイント番号`3`の`Enb`欄が`y`から`n`に変わりました。

ブレークポイントの設定自体は残っています。

この状態でリセットして実行します。

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

`tk_cre_tsk()`では停止せず、初期タスクの実行を開始する直前で停止しました。

#### ブレークポイントを再び有効にする

次に、`enable`にブレークポイント番号を指定して、無効にしたブレークポイントを再び有効にします。

```gdb
enable 3
info breakpoints
```

実行結果：

```text
Num     Type           Disp Enb Address    What
2       breakpoint     keep y   0x1000093e in main at kernel/inittsk.c:32
        breakpoint already hit 2 times
3       breakpoint     keep y   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
```

ブレークポイント番号`3`の`Enb`欄が`y`に戻りました。

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

Thread 1 "rp2040.core0" hit Breakpoint 3, tk_cre_tsk (pk_ctsk=0x20000000 <ctsk_ini>)
    at kernel/task_mange.c:18
18          if((pk_ctsk->tskatr & ~TA_RNG3) != (TA_HLNG|TA_USERBUF)) return E_RSATR;
```

今度は、初期タスクの生成処理で停止しました。

### 結果と考察

ブレークポイント番号`3`の有効・無効を切り替えたときの確認結果をまとめます。

| 操作 | `Enb`欄 | `tk_cre_tsk()`での動作 | 実際の停止位置 |
|---|---|---|---|
| `disable 3` | `n` | 停止せずに通過 | `inittsk.c:32` |
| `enable 3` | `y` | 停止 | `task_mange.c:18` |

`disable`と`enable`を使用することで、ブレークポイントを設定し直すことなく、実際の停止位置を切り替えられることを確認できました。

また、`info breakpoints`の`breakpoint already hit`という表示から、ブレークポイントへの到達回数も確認できます。今回の確認では、ターゲットをリセットしても、この回数は保持されていました。

デバッグ中に一時的に別の処理を調べたい場合には、ブレークポイントを削除するのではなく、無効にしておくと便利です。

調査対象を元に戻したいときには、`enable`で再び有効にするだけで済みます。

## 4. 一度だけ停止するブレークポイントを設定する - delete / tbreak

デバッグでは、調査を開始したい位置まで一度だけ実行し、その後は同じ場所で停止する必要がない場合もあります。

通常のブレークポイントでは、不要になった時点で削除する操作が必要です。

GDBには、一度停止すると自動的に削除される一時ブレークポイント（temporary breakpoint）を設定する`tbreak`というコマンドがあります。

### 確認すること

次の3つを確認します。

- `delete`を使って、番号を指定したブレークポイントを削除できること。
- `tbreak`を使って、通常のブレークポイントと同様に指定した場所で停止できること。
- 一度停止すると、ブレークポイントが自動的に削除されること。

### 操作手順

#### ブレークポイントを削除する

まず、第3項の確認で設定したブレークポイントを確認します。

```gdb
info breakpoints
```

実行結果：

```text
Num     Type           Disp Enb Address    What
2       breakpoint     keep y   0x1000093e in main at kernel/inittsk.c:32
        breakpoint already hit 2 times
3       breakpoint     keep y   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
        breakpoint already hit 1 time
```

`delete`にブレークポイント番号を指定して、2つのブレークポイントを削除します。

```gdb
delete 2 3
info breakpoints
```

実行結果：

```text
No breakpoints, watchpoints, tracepoints, or catchpoints.
```

2つのブレークポイントが削除されました。

`delete`には、複数のブレークポイント番号を指定することもできます。

第2項で使用した`clear`と、今回使用した`delete`を比較します。

| コマンド | 削除対象の指定方法 | 使用例 |
|---|---|---|
| `clear` | 設定した場所を指定 | `clear tk_cre_tsk` |
| `delete` | ブレークポイント番号を指定 | `delete 2 3` |

設定した場所が分かっている場合は`clear`を、`info breakpoints`で設定を確認しながら削除する場合は`delete`を使用すると便利です。

#### 一時ブレークポイントを設定する

これまで使用してきた`break`の代わりに、`tbreak`を使って`tk_cre_tsk()`にブレークポイントを設定します。

```gdb
tbreak tk_cre_tsk
info breakpoints
```

実行結果：

```text
Temporary breakpoint 4 at 0x10000ad0: file kernel/task_mange.c, line 18.
Num     Type           Disp Enb Address    What
4       breakpoint     del  y   0x10000ad0 in tk_cre_tsk at kernel/task_mange.c:18
```

一時ブレークポイント番号`4`が設定されました。

第1項で確認した通常のブレークポイントと比較すると、`Disp`欄が異なります。

| 設定コマンド | `Disp`欄 | ブレークポイントで停止した後の動作 |
|---|---|---|
| `break` | `keep` | 設定を保持する |
| `tbreak` | `del` | 設定を自動的に削除する |

設定されたアドレスは、第1項で`break tk_cre_tsk`を実行したときと同じ`0x10000ad0`です。

停止する位置は同じですが、停止後のブレークポイントの扱いが異なります。

#### 停止後のブレークポイントの状態を確認する

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

Thread 1 "rp2040.core0" hit Temporary breakpoint 4, tk_cre_tsk (pk_ctsk=0x20000000 <ctsk_ini>)
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

ブレークポイントを削除するコマンドを実行していないにもかかわらず、設定がなくなっています。

### 結果と考察

`tbreak`を使用することで、指定した場所で一度停止し、その後はブレークポイントが自動的に削除されることを確認できました。

これまでに使用したブレークポイントの設定管理に関する操作を、用途ごとに整理します。

| 用途 | コマンド |
|---|---|
| 設定済みのブレークポイントを確認したい | `info breakpoints` |
| 一時的に停止させたくない | `disable` |
| 無効にした設定を再び使用したい | `enable` |
| 不要になった設定を削除したい | `clear` / `delete` |
| 最初から一度だけ停止させたい | `tbreak` |

例えば、特定の関数まで実行し、そこからステップ実行を始めたい場合には、`tbreak`が便利です。

通常のブレークポイントを設定して後から削除する必要がなく、調査を開始したい位置までの実行に利用できます。

ブレークポイントは、停止する場所だけでなく、設定をいつまで保持するかについても、調査の目的に応じて使い分けることができます。

---

# 今回分かったこと

今回は、Try Kernelのタスク生成処理を題材に、ブレークポイントの設定方法と管理方法を調べました。

実機で確認できた内容をまとめます。

| 確認したこと | GDBコマンド | 実行結果 |
|---|---|---|
| 関数名を指定して停止 | `break tk_cre_tsk` | `task_mange.c`の18行目で停止 |
| ソースコードの行番号を指定して停止 | `break inittsk.c:32` | 初期タスクの実行開始直前で停止 |
| 設定内容を確認する | `info breakpoints` | 設定位置・有効状態・到達回数などを確認 |
| 設定の有効・無効を切り替える | `disable`、`enable` | 設定を保持したまま停止位置を切り替え |
| 設定を削除する | `clear`、`delete` | 場所または番号を指定して削除 |
| 一度だけ停止する | `tbreak` | 停止後にブレークポイントが自動的に削除された |

第2回では、ブレークポイントとステップ実行を使って関数内部の処理を追いました。

今回は、調べたい処理まで直接実行する方法に加え、調査の進み具合に応じてブレークポイントの設定を管理する方法を確認しました。

**調べたい場所で停止し、不要になった停止を減らす。**

ブレークポイントの設定と管理を使い分けることで、効率よくデバッグを進められます。

ところが、今回の題材である`tk_cre_tsk()`のように、同じ関数が何度も呼び出される場合、停止位置を指定するだけでは調べたい処理を十分に絞り込めないことがあります。

次回は、引数の値やブレークポイントの通過回数を利用して、特定の条件を満たした場合だけ停止する方法を調べます。
