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

## 5. 値を記録して自動的に実行を再開する - commands / silent / printf

ブレークポイントは、プログラムを停止させて手動で調べるためだけのものではありません。

GDBの`commands`を使うと、ブレークポイントに到達したときに実行するコマンドをあらかじめ設定できます。

### 実験の目的

`tk_cre_tsk()`が呼ばれるたびに、引数の優先度とスタックサイズをGDBの画面に出力し、自動的にプログラムの実行を再開させます。

ソースコードにデバッグ用の`printf()`を追加することなく、4回分のタスク生成情報を取得できることを確認します。

### 実験の手順

まず、前の実験のブレークポイントを解除し、改めて設定します。

```gdb
clear tk_cre_tsk
break tk_cre_tsk
```

実行結果：

```text
Deleted breakpoint 6
Breakpoint 7 at 0x10000ad0: file kernel/task_mange.c, line 18.
```

続いて、ブレークポイントに到達したときの動作を指定します。

```gdb
commands
silent
printf "tk_cre_tsk: itskpri=%d, stksz=%d\n", pk_ctsk->itskpri, pk_ctsk->stksz
continue
end
```

`commands`は、番号を省略すると直前に設定したブレークポイントを対象とします。

`commands`を入力すると、コマンド列の入力待ちになります。実行させたいコマンドを1行ずつ入力し、最後に`end`を入力します。

| コマンド | 役割 |
|---|---|
| `commands` | ブレークポイントに対するコマンド列の設定を開始する |
| `silent` | 通常のブレークポイント停止メッセージを抑制する |
| `printf` | 指定した値をGDBの画面に出力する |
| `continue` | 出力後にプログラムの実行を再開する |
| `end` | コマンド列の設定を終了する |

設定が完了したら、リセットして実行します。

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
tk_cre_tsk: itskpri=1, stksz=256
tk_cre_tsk: itskpri=10, stksz=1024
tk_cre_tsk: itskpri=10, stksz=1024
tk_cre_tsk: itskpri=10, stksz=1024
```

### 実験結果と考察

4回分のタスク生成情報が出力されました。

ソースコード上の呼び出し順と照らし合わせると、次のようになります。

| 呼び出し順 | 生成するタスク | 優先度 | スタックサイズ（バイト） |
|---|---|---:|---:|
| 1 | 初期タスク | 1 | 256 |
| 2 | ボタン検出タスク | 10 | 1024 |
| 3 | LED制御タスク1 | 10 | 1024 |
| 4 | LED制御タスク2 | 10 | 1024 |

第4項の`ignore`を使った実験結果とも一致しています。

この方法を使えば、**ソースコードを変更することなく、関数に渡された値をログとして取得できます。**

また、今回の`printf`はC言語の関数ではなく、GDBのコマンドです。出力先もターゲット側のUARTではなく、GDBの画面です。

`continue`が自動的に実行されるため、ログを取得した後もプログラムの実行は続きます。GDBの操作に戻るには、`Ctrl+C`で実行を中断します。

:::message
この方法では、ブレークポイントに到達するたびにプログラムが一時停止しています。

値の取得と出力が完了してから実行を再開するため、リアルタイム性が求められる処理を調べる場合は、デバッグ操作による実行タイミングへの影響に注意が必要です。
:::

