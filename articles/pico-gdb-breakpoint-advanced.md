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

<!-- TODO: 関数名指定ブレークポイントの番号とアドレスを実機で取得する -->

設定済みのブレークポイントを一覧表示します。

```gdb
info breakpoints
```

実行結果：

<!-- TODO: 設定一覧を実機で取得する -->

`info breakpoints`では、ブレークポイント番号や有効・無効、設定位置を確認できます。

続いてリセットして実行します。

```gdb
monitor reset halt
continue
```

実行結果：

<!-- TODO: tk_cre_tskの入口で停止することを実機で確認する -->

`main()`から処理を追わなくても、**目的の関数が呼ばれた場所で停止できる**ことを確認します。

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

<!-- TODO: 行番号指定ブレークポイントの設定結果を取得する -->

```gdb
monitor reset halt
continue
```

実行結果：

<!-- TODO: inittsk.c:32への停止結果を取得する -->

`ファイル名:行番号`を使えば、関数の途中にも停止位置を設定できます。

同名のファイルが複数ある場合は、パスを含めて指定する方法もあります。

:::message
`inittsk.c:32`は第1回の実測結果と一致する行です。ただし、今回も実機で同じ位置に停止することを確認してから確定事項として記載します。
:::

## 3. 条件を満たす場合だけ停止する - break ... if

関数が繰り返し呼ばれる場合、毎回止める必要はありません。

今回は、優先度`1`が指定されたタスク生成だけを調べます。

```gdb
clear inittsk.c:32
break tk_cre_tsk if pk_ctsk->itskpri == 1
```

実行結果：

<!-- TODO: 条件付きブレークポイントの設定結果を取得する -->

```gdb
monitor reset halt
continue
```

実行結果：

<!-- TODO: 初期タスク生成時に停止することを実機で確認する -->

停止したら、関数の引数を確認します。

```gdb
print pk_ctsk->itskpri
```

実行結果：

<!-- TODO: itskpri=1の実測結果を取得する -->

`if`以降に指定した式が真の場合に停止します。

今回のソースでは初期タスクだけが優先度`1`ですが、**優先度で絞り込んでいるのであって、初期タスクIDを判定しているわけではありません。**

## 4. 何回目かを指定して停止する - ignore

特定の関数を複数回呼ぶ場合、何回目に呼ばれたかで停止位置を絞る方法があります。

`ignore`は、指定したブレークポイントについて、**次の指定回数だけ停止せずに通過する**コマンドです。

今回は、最初の3回を通過し、4回目の`tk_cre_tsk()`で止める実験を考えます。

まず、前の実験の条件付きブレークポイントを解除します。

```gdb
clear tk_cre_tsk
break tk_cre_tsk
info breakpoints
```

実行結果：

<!-- TODO: 新しいブレークポイント番号を実機で確認する -->

続いて、その番号に対して`ignore`を設定します。

例えば、`info breakpoints`で番号が`4`と表示された場合は、次のように指定します。

```gdb
ignore 4 3
```

ブレークポイント番号は設定順で変わります。**上記の`4`は例なので、実機で表示された番号に置き換えてください。**

```gdb
monitor reset halt
continue
```

実行結果：

<!-- TODO: 呼び出し4回目で停止するか、途中で実行状態が変化するかを実機で確認する -->

ソースの処理が予定どおり進めば、初期タスク・ボタン検出タスク・LED制御タスク1の3回を素通りし、LED制御タスク2の生成で停止すると予想できます。

ただし、ここまで実行されることは**まだ確認していません**。タスク切替を含むため、実際の停止状態を見て判断します。

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
