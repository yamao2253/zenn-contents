---
title: "GDBでRaspberry Pi Picoのブレークポイントとステップ実行を学ぶ"
emoji: "🛑"
type: "tech"
topics: ["gdb", "raspberrypi", "pico", "debug", "embedded"]
published: false
---

# はじめに

CQ出版社『Interface』2023年7月号の特集「ラズパイPicoで1500行 ゼロから作るOS」を読み進めながら、Raspberry Pi PicoでTry Kernelを動かしています。

前回の「[GDBでRaspberry Pi Picoの変数をウォッチする](https://zenn.dev/helowota/articles/pico-gdb-variable-watch)」では、初期タスクの生成処理を題材に、`print`、`display`、`watch`などを使って変数の値を調べました。

そのときは`next`コマンドで`tk_cre_tsk()`を実行し、戻り値が`tskid_ini`に格納されることを確認しました。

では、**`tk_cre_tsk()`の内部で、どのような処理が行われているのでしょうか。**

今回はブレークポイントとステップ実行を使い、プログラムの実行位置を制御しながら、タスク生成APIの処理を追いかけます。

単にコマンドの使い方を覚えるのではなく、**どこで止めたいのか、どこまで実行したいのか**を考えながら進めます。

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

開発環境の構築とOpenOCD/GDBの基本的な接続方法については、以下の記事で紹介しています。

- [「ラズパイPicoで1500行 ゼロから作るOS」を2026年に動かす](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)
- [OpenOCDとGDBでRaspberry Pi Picoをデバッグする](https://zenn.dev/helowota/articles/pico-openocd-gdb-debug)

前回と同じプログラムを使用します。今回もTry Kernelのソースコードは変更せず、GDBから動作を調べます。

# どこで止めて、どこまで実行するのか

デバッグで変数の値を確認するには、まず適切な場所でプログラムを停止させる必要があります。

例えば、関数の戻り値だけを調べるなら、呼び出しの前後で停止できれば十分です。

一方、関数内部で問題が発生している場合は、**関数に入って、どの条件を通り、どの値を使って処理を進めたのか**を確認したくなります。

GDBでは、ブレークポイントを使って停止位置を指定し、ステップ実行で少しずつプログラムを進められます。

今回は、初期タスクを生成する`tk_cre_tsk()`を題材に、この2つの機能を使い分けます。

## 初期タスク生成の流れを追う

書籍P.74の`kernel/inittsk.c`には、次の処理があります。

```c
tskid_ini = tk_cre_tsk(&ctsk_ini);  // 初期タスク生成
tk_sta_tsk(tskid_ini, 0);           // 初期タスク実行
```

前回は、`tk_cre_tsk()`の戻り値が`1`となり、初期タスクの生成に成功したことを確認しました。

今回は、**関数の入口と引数チェック、戻り値が呼び出し元に渡される様子**を調べます。

書籍P.72〜73のタスク管理APIの説明と、著者公開ソースの`kernel/task_mange.c`を見ると、`tk_cre_tsk()`は次のような処理を行います。

1. タスク生成情報の引数をチェックする
2. 未使用のタスク管理ブロック（TCB）を探す
3. 見つかったTCBを初期化する
4. タスクIDまたはエラーコードを返す

実際の実装は、[著者公開の`kernel/task_mange.c`](https://github.com/ytoyoyama/interface_trykernel/blob/main/part_4/sect_3/kernel/task_mange.c)でも確認できます。

この流れを頭に置いて、GDBで処理を追っていきます。

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

GDBからOpenOCDに接続し、プログラムをロードします。

```gdb
target remote localhost:3333
monitor reset halt
load
```

これで準備ができました。

以降のGDBコマンドは、GDBのプロンプトから入力します。

---

# 実際にGDBでプログラムを追いかける

## 1. ブレークポイントでmain関数に停止する - break / continue

最初に、前回と同じ`main()`で停止させます。

```gdb
break main
continue
```

実行結果：

```text
Thread 1 "rp2040.core0" hit Breakpoint 1, main () at kernel/inittsk.c:31
31          tskid_ini = tk_cre_tsk(&ctsk_ini);              // 初期タスク生成
```

`break main`は、`main()`にブレークポイントを設定するコマンドです。

`continue`は、現在の停止位置からプログラムを再開するコマンドです。今回は、ブレークポイントに到達したところで停止します。

**`break`で停止位置を設定し、`continue`で実行を再開する**、という役割分担です。

今回は`tk_cre_tsk()`を呼び出す直前で停止できるため、関数の内部を調べる出発点になります。

## 2. 呼び出した関数の中に入る - step

前回は、ここで`next`を実行しました。

しかし、`next`では`tk_cre_tsk()`の内部には入らず、呼び出し元の次の行まで進みます。

今回は`step`を実行してみます。

```gdb
step
```

実行結果：

```text
tk_cre_tsk (pk_ctsk=0x20000000 <ctsk_ini>) at kernel/task_mange.c:18
18          if((pk_ctsk->tskatr & ~TA_RNG3) != (TA_HLNG|TA_USERBUF)) return E_RSATR;
```

`step`は、ソースコードの次の行へ進むコマンドです。関数呼び出しがあれば、通常は**呼び出し先の関数内部に入って停止**します。

今回は`main()`の31行目から`tk_cre_tsk()`の内部に入り、`kernel/task_mange.c`の18行目で停止しました。

また、引数`pk_ctsk`が`0x20000000`にある`ctsk_ini`を指していることも分かります。

停止した18行目は、タスク属性をチェックする条件式です。ここから関数内部の処理を追いかけていきます。

:::message
実際の停止行は、ビルド時のデバッグ情報やコンパイラの生成コードによって異なる場合があります。
:::

## 3. 引数を確認しながら進める - info args / next

`tk_cre_tsk()`の引数は、タスク生成情報を指すポインタ`pk_ctsk`です。

まず、現在の関数に渡された引数を確認します。

```gdb
info args
```

実行結果：

```text
pk_ctsk = 0x20000000 <ctsk_ini>
```

続いて、引数が指す構造体のメンバを調べます。

```gdb
print pk_ctsk->itskpri
print pk_ctsk->stksz
```

実行結果：

```text
$1 = 1
$2 = 256
```

著者公開ソースの`ctsk_ini`では、初期タスクの優先度`itskpri`は`1`、スタックサイズ`stksz`は`256`バイトに設定されています。

実機でも同じ値が取得でき、`main()`から`tk_cre_tsk()`へ設定どおりの引数が渡されていることを確認できました。

次に、`next`で処理を進めます。

```gdb
next
```

実行結果：

```text
19          if(pk_ctsk->itskpri <= 0 || pk_ctsk->itskpri > CNF_MAX_TSKPRI) return E_PAR;
```

18行目のタスク属性チェックを通過し、19行目の優先度チェックで停止しました。

`next`は、関数呼び出しがあっても通常はその内部に入らず、次のソース行まで実行するコマンドです。今回は`tk_cre_tsk()`の内部で実行したため、同じ関数内の19行目に進みました。

このように、引数の値と条件式を照らし合わせることで、どのチェックを通過したかを確認できます。

## 4. 関数の処理を終えて呼び出し元に戻る - finish

`tk_cre_tsk()`の引数と最初の条件判定を確認できました。残りの処理を実行し、呼び出し元へ戻るには`finish`を使います。

```gdb
finish
```

実行結果：

```text
Run till exit from #0  tk_cre_tsk (pk_ctsk=0x20000000 <ctsk_ini>) at kernel/task_mange.c:19
0x10000938 in main () at kernel/inittsk.c:31
31          tskid_ini = tk_cre_tsk(&ctsk_ini);              // 初期タスク生成
Value returned is $3 = 1
```

`finish`は、現在の関数が戻るまで実行し、呼び出し元で停止するコマンドです。今回は`main()`の31行目に戻り、`tk_cre_tsk()`の戻り値が`1`であることを確認できました。

### 戻り値と変数への代入は同じタイミングか？

`finish`の直後に、呼び出し元の変数`tskid_ini`を確認します。

```gdb
print tskid_ini
```

実行結果：

```text
$4 = 0
```

関数の戻り値は`1`でしたが、`tskid_ini`はまだ`0`です。今回の停止位置では、戻り値を変数に代入する処理が完了していません。

そこで、`next`で処理を進め、もう一度確認します。

```gdb
next
print tskid_ini
```

実行結果：

```text
32          tk_sta_tsk(tskid_ini, 0);                       // 初期タスク実行
$5 = 1
```

今度は`tskid_ini`が`1`になりました。

**`finish`直後は`tskid_ini = 0`、`next`後は`tskid_ini = 1`**という違いから、関数が戻り値を返すことと、その値が呼び出し元の変数へ代入されることは、別々の処理であると分かります。

C言語では1行に見える処理でも、GDBではその途中で停止することがあります。

:::message
`finish`は、関数が通常どおり戻ることを前提に使います。タスク切替などにより呼び出し元へ戻らない処理では、期待した場所で停止しない場合があります。
:::

## 5. 同じ呼び出し位置でstepとnextを比較する

ここまでの結果を、第1回で確認した`next`の動作と比較します。どちらも`main()`の31行目にある`tk_cre_tsk(&ctsk_ini)`を対象にした実験です。

| コマンド | 実行結果 |
|---|---|
| `step` | `tk_cre_tsk()`の内部に入り、`task_mange.c`の18行目で停止 |
| `next` | 関数内部には入らず、`main()`の32行目で停止 |

`step`は呼び出した関数の内部を調べたいときに、`next`は関数呼び出しをまとめて実行したいときに使います。

一方、関数内部に入った後、残りの処理を実行して呼び出し元へ戻りたいときには、§4で確認した`finish`が使えます。

なお、32行目の`tk_sta_tsk()`はタスク切替に関わる処理です。本記事ではここから先へは進まず、ステップ実行の比較を終えます。

---

# 今回分かったこと

今回は、Try Kernelの初期タスク生成処理を題材に、ブレークポイントとステップ実行で関数内部を調べました。

実機で確認できた内容をまとめます。

| 確認したこと | GDBコマンド | 実行結果 |
|---|---|---|
| `main()`で停止 | `break main`、`continue` | `inittsk.c`の31行目で停止 |
| 関数内部に入る | `step` | `tk_cre_tsk()`の18行目で停止 |
| 引数を調べる | `info args`、`print` | `itskpri = 1`、`stksz = 256`を確認 |
| 条件判定を進める | `next` | 18行目から19行目へ進む |
| 呼び出し元へ戻る | `finish` | `main()`に戻り、戻り値`1`を確認 |
| 代入のタイミングを調べる | `print`、`next` | `finish`直後は`tskid_ini = 0`、`next`後は`1` |

第1回では、変数の値が変化したことを確認しました。今回はさらに、**関数内部に入り、必要な範囲だけ実行して、戻り値が変数に代入されるまで**を確認できました。

`step`、`next`、`finish`は、どこまで処理を追いたいかに応じて使い分けられます。
