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

今回は、その**戻り値が決まるまでの過程**を調べます。

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

第1回で確認した実行結果：

```text
Thread 1 "rp2040.core0" hit Breakpoint 1, main () at kernel/inittsk.c:31
31          tskid_ini = tk_cre_tsk(&ctsk_ini);              // 初期タスク生成
```

`break main`は、`main()`にブレークポイントを設定するコマンドです。

`continue`は、現在の停止位置からプログラムを再開するコマンドです。今回は、ブレークポイントに到達したところで停止します。

**ブレークポイントは「どこで止めるか」、`continue`は「そこまで実行するか」を指定する**と考えると分かりやすいです。

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

`next`は、関数呼び出しがあっても通常はその内部に入らず、次のソース行まで実行します。

今回は`tk_cre_tsk()`の内部で`next`を実行し、条件判定を1つ進めることができました。

ここでの`next`は、**`tk_cre_tsk()`の内部にとどまったまま、次のソース行へ進む**ために使います。

関数内に別の関数呼び出しがあっても、通常はその呼び出し先に入らず、元の関数で次の行まで進みます。

### 何を確認したいのか？

`tk_cre_tsk()`は、タスク属性、優先度、スタックサイズなどの引数をチェックします。

この部分で停止すれば、渡した値と条件式を照らし合わせて、どの分岐を通るのか調べられます。

すべての行を追う必要はありません。**気になる分岐では細かく進め、関係の薄い処理はまとめて進める**のが効率的です。

## 4. 関数の処理を終えて呼び出し元に戻る - finish

`tk_cre_tsk()`の入口から内部を追ったところで、戻り値も確認したいと思います。

GDBには、現在の関数が戻るところまで実行する`finish`コマンドがあります。

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

`finish`を実行すると、`tk_cre_tsk()`の残りの処理が実行され、呼び出し元の`main()`に戻りました。

`Value returned is $3 = 1`から、タスク生成APIが`1`を返したことが分かります。

ただし、停止位置はまだ`main()`の31行目です。関数から戻ったことと、戻り値が変数に代入されたことは区別して確認する必要があります。

`finish`は、**現在の関数の実行を続け、呼び出し元に戻ったところで停止する**コマンドです。

GDBは通常、関数の戻り値も表示します。これにより、`tk_cre_tsk()`からどの値が返されたのかを確認できます。

呼び出し元へ戻ったら、前回と同じ変数を表示してみます。

```gdb
print tskid_ini
```

`finish`の直後に、呼び出し元の変数`tskid_ini`を確認します。

```gdb
print tskid_ini
```

実行結果：

```text
$4 = 0
```

`tk_cre_tsk()`の戻り値は`1`でしたが、`tskid_ini`はまだ`0`です。

これは、**関数から戻った時点では、戻り値を変数に代入する処理が完了していない**ためです。

C言語では1行で書かれている処理でも、関数呼び出し、戻り値の取得、変数への代入は別々に実行されます。

第1回では、タスク生成後の`tskid_ini`が`1`になりました。

今回は、関数内の処理を追いながら、その値が戻り値として返されるまでの流れを確認します。

:::message
`finish`は、関数が通常どおり戻ることを前提に使います。タスク切替などにより呼び出し元へ戻らない処理では、期待した場所で停止しない場合があります。
:::

## 5. 同じ場所からnextを実行し、stepと比較する

§2で`step`を使って関数内部に入り、§4では`finish`で呼び出し元へ戻りました。

ここでは実行位置を戻して`next`を試し、§2の`step`の結果と比較します。

RP2040をリセットし、再び`main()`のブレークポイントまで進めます。

```gdb
monitor reset halt
continue
```

実行結果：

<!-- TODO: mainの31行目で停止することを再確認する -->

§1で設定した`break main`は同じGDBセッション内で残っているので、もう一度設定する必要はありません。


そこで、`next`で処理を1行進め、もう一度`tskid_ini`を確認します。

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

`finish`では`tk_cre_tsk()`の戻り値が`1`と表示されましたが、その直後の`tskid_ini`は`0`でした。さらに`next`を実行すると、戻り値が代入され、`1`になりました。

**関数から戻ることと、戻り値を変数へ代入することは、別々の処理である**と確認できました。

C言語では1行に見える処理でも、GDBではその途中で停止することがあります。

第1回では、この操作で`tk_cre_tsk()`の呼び出しを終え、次の`tk_sta_tsk()`の行に進みました。

一方、§2では同じ位置から`step`を実行し、`tk_cre_tsk()`の内部を調べました。

この次の`tk_sta_tsk()`はタスク切替に関わるため、本記事では実行を進めず、ここで比較を終えます。

| コマンド | 関数呼び出しの行で実行した場合 |
|---|---|
| `step` | 呼び出し先の関数内部に入る |
| `next` | 呼び出し先の内部には入らず、呼び出し元の次の行まで進む |
| `finish` | 現在の関数が戻るところまで実行する |

**関数の内部を調べたいなら`step`、呼び出した結果だけを見たいなら`next`**という使い分けができます。

---

# 今回分かったこと

今回は、Try Kernelの初期タスク生成処理を題材に、ブレークポイントとステップ実行を使って関数内部を調べる手順を整理しました。

実機で確認する項目をまとめます。

| 確認すること | GDBコマンド | 確認する内容 |
|---|---|---|
| 目的の関数で停止 | `break`、`continue` | `main()`のタスク生成直前で停止できるか |
| 呼び出した関数の内部に入る | `step` | `tk_cre_tsk()`へ進めるか |
| 引数と処理を確認 | `info args`、`print`、`next` | 引数の値や分岐を確認できるか |
| 関数の処理を終える | `finish` | 戻り値と呼び出し元を確認できるか |
| `step`と`next`を比較 | `step`、`next` | 関数内部に入るかどうかの違いを確認できるか |

<!-- TODO: 実機検証後、確認結果に基づいて表の時制と結果を書き換える -->

第1回では、変数の値が変化したことを確認しました。

今回はさらに、**その値が決まるまでの処理に入り、必要な範囲だけ実行する**方法を学びます。

`step`、`next`、`finish`を使い分ければ、関数の内部を細かく調べることも、呼び出し元へ戻ることもできます。

<!-- TODO: 実機で各コマンドを確認し、停止位置・出力を反映してからpublished: trueにする -->
