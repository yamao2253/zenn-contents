---
title: "Raspberry Pi Picoで自作OSの動きを見る ― Try KernelにUARTトレースを追加する"
emoji: "🔍"
type: "tech"
topics: ["raspberrypi", "pico", "rtos", "c", "embedded"]
published: false
---

## はじめに

## はじめに

CQ出版社『Interface』2023年7月号の特集「ラズパイPicoで1500行 ゼロから作るOS」を参考に、Raspberry Pi PicoでリアルタイムOS「Try Kernel」を動かしています。

これまでにタスク管理やタスク間同期のサンプルを動作させ、セマフォやイベントフラグを利用するところまで進みました。

開発環境の構築手順については、[こちらの記事](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)で紹介しています。

ただ、プログラムが期待どおりに動いていても、OSの内部で何が起きているのかは、なかなか見えてきません。

例えば、セマフォを獲得する処理です。

```c
tk_wai_sem(semid, 1, TMO_FEVR);
```

この関数を呼び出したとき、セマフォをすぐに獲得できたのか、それとも他のタスクが解放するまで待たされたのか。

ソースコードを追えば仕組みは理解できますが、実際の動作を観察できれば、理解はさらに深まるはずです。

そこで今回は、Try Kernelに**時刻付きのUARTトレース機能**を追加してみます。

## 今回の目標

実装するのは、次のようなログをUARTへ出力する機能です。

```text
[4180] LED1 SEM WAIT
[4180] LED1 SEM ACQUIRED
```

先頭の数値は、OSのシステムタイマを利用した起動後の経過時間です。単位はミリ秒です。

さらに、セマフォの獲得待ちが発生すると、次のようになります。

```text
[18810] LED1 SEM WAIT
[19360] LED1 SEM ACQUIRED
```

この場合、`tk_wai_sem()` の呼び出しから復帰までに550ms経過しています。

これなら、タスクがいつ待ち状態に入り、いつ処理を再開したのかを観察できます。

今回は、次の機能を追加します。

- システムタイマによる経過時間の取得
- 時刻付きUARTトレース
- 数値を出力するための10進数・16進数表示
- マクロによるトレースの有効・無効切り替え
- Gitのコミット情報を起動メッセージに表示

なお、これは書籍の掲載コードをそのまま再現する作業ではなく、**書籍をベースにした独自拡張**です。

## 開発環境

| 項目 | 内容 |
|---|---|
| 開発PC | Windows 11 |
| ターゲット | Raspberry Pi Pico W |
| 使用基板 | Robo Pico Board |
| 言語 | C |
| コンパイラ | arm-none-eabi-gcc |
| ビルド | GNU Make |
| デバッグ | OpenOCD / GDB |
| ソース管理 | Git / GitHub |

対象はTry Kernelの `part_4/sect_3`、セマフォを使用するサンプルです。

開発用リポジトリは、書籍のサンプルをベースに作成した `trykernel-lab` を使用しています。

https://github.com/yamao2253/trykernel-lab

## 1. システムタイマから経過時間を取得する

まず、ログに付ける時刻を用意します。

Try Kernelには、タスクの待ち時間などを管理するためのシステムタイマがあります。

今回の実装では、システムタイマの割込み処理が実行されるたびに、経過時間を加算することにしました。

対象ファイルは次のとおりです。

```text
part_4/sect_3/kernel/systimer.c
```

追加した処理の中心部分を示します。

```c
static volatile UW system_time_ms = 0;

UW tm_get_msec(void)
{
    return system_time_ms;
}
```

そして、システムタイマの割込みハンドラ内で経過時間を更新します。

```c
void systimer_handler(void)
{
    /* 既存の変数宣言 */

    system_time_ms += TIMER_PERIOD;

    /* 既存の待ち時間管理・スケジューリング処理 */
}
```

`TIMER_PERIOD` は10msです。

したがって、割込みが1回発生するごとに、`system_time_ms` が10増加します。

`tm_get_msec()` を呼び出せば、その時点の経過時間を取得できます。

### なぜvolatileを付けるのか

```c
static volatile UW system_time_ms;
```

この変数は、システムタイマの割込みハンドラから更新され、通常のタスクから参照されます。

`volatile` を付けることで、コンパイラが値をレジスタに保持したままにするような最適化を抑制し、参照時にメモリから読み出すよう指示します。

ただし、`volatile` 自体が排他制御やアトミック性を保証するわけではありません。

今回はRP2040上のTry Kernelで使用する32bitカウンタとして実装しています。

また、タイマの分解能は10msです。表示単位はミリ秒ですが、1ms単位の精度で測定できるわけではありません。

## 2. UARTトレース関数を追加する

続いて、ログを整形してUARTへ出力する関数を追加します。

対象ファイルは次の2つです。

```text
part_4/sect_3/include/syslib.h
part_4/sect_3/kernel/syslib.c
```

### トレースのインタフェース

ヘッダファイルに、次の定義を追加します。

```c
UW tm_get_msec(void);

#define TM_FMT_NONE 0
#define TM_FMT_DEC  1
#define TM_FMT_HEX  2

#ifndef TM_TRACE_ENABLE
#define TM_TRACE_ENABLE 1
#endif

void tm_trace(const char *msg, UW value, UINT fmt);

#if TM_TRACE_ENABLE

#define TM_TRACE(msg) \
    tm_trace((msg), 0, TM_FMT_NONE)

#define TM_TRACE_DEC(msg, val) \
    tm_trace((msg), (UW)(val), TM_FMT_DEC)

#define TM_TRACE_HEX(msg, val) \
    tm_trace((msg), (UW)(val), TM_FMT_HEX)

#else

#define TM_TRACE(msg) ((void)0)
#define TM_TRACE_DEC(msg, val) ((void)0)
#define TM_TRACE_HEX(msg, val) ((void)0)

#endif
```

呼び出し側では、次のように記述できます。

```c
TM_TRACE("LED1 SEM WAIT");
TM_TRACE_DEC("COUNT", count);
TM_TRACE_HEX("FLAGS", flags);
```

`TM_TRACE_ENABLE` を0にすれば、トレースの呼び出しを無効化できます。

### printfを使わない理由

今回の実装では、`printf()` を使用していません。

Try Kernelには、UARTへ文字列を出力する `tm_putstring()` が用意されています。

これを利用すれば、標準入出力ライブラリに依存せずにトレース機能を実装できます。

また、10進数・16進数への変換処理も小さな関数として実装しました。

トレースの基本処理は、次のようになっています。

```c
void tm_trace(const char *msg, UW value, UINT fmt)
{
#if TM_TRACE_ENABLE
    char buf[128];
    char *p = buf;
    UW msec = tm_get_msec();

    *p++ = '[';
    p = trace_dec(p, msec);
    *p++ = ']';
    *p++ = ' ';

    while (*msg && p < buf + sizeof(buf) - 16) {
        *p++ = *msg++;
    }

    if (fmt == TM_FMT_DEC) {
        *p++ = '=';
        p = trace_dec(p, value);
    }
    else if (fmt == TM_FMT_HEX) {
        *p++ = '=';
        *p++ = '0';
        *p++ = 'x';
        p = trace_hex(p, value);
    }

    *p++ = '\n';
    *p = '\0';

    tm_putstring(buf);
#else
    (void)msg;
    (void)value;
    (void)fmt;
#endif
}
```

`trace_dec()` と `trace_hex()` は、数値を文字列に変換するための補助関数です。

今回は数値の表示幅を固定せず、経過時間に応じて必要な桁数だけ出力します。

## 3. セマフォの待ち時間を観察する

トレース機能ができたので、実際のタスクに組み込みます。

対象は次のファイルです。

```text
part_4/sect_3/application/usermain.c
```

LED1を制御するタスクで、セマフォ獲得処理の前後にトレースを追加しました。

```c
TM_TRACE("LED1 SEM WAIT");

tk_wai_sem(semid, 1, TMO_FEVR);

TM_TRACE("LED1 SEM ACQUIRED");
```

変更はこれだけです。

`tk_wai_sem()` の直前と直後にログを追加することで、セマフォを待つ時間が観察できます。

### 実機で確認

Pico Wに書き込み、UARTログを確認しました。

```text
[4180] LED1 SEM WAIT
[4180] LED1 SEM ACQUIRED
```

この場合、2つのログの時刻が同じです。

少なくとも10ms単位のシステムタイマでは、待ち時間を確認できません。

一方、次のログも取得できました。

```text
[18810] LED1 SEM WAIT
[19360] LED1 SEM ACQUIRED
```

差を計算すると、

```text
19360 - 18810 = 550ms
```

となります。

`tk_wai_sem()` がすぐに復帰せず、待ち状態を経て処理を再開したことが分かります。

### OSの動作が見えてきた

セマフォは、単なるフラグの確認ではありません。

獲得できない場合、タスクは待ち状態となり、スケジューラは別の実行可能なタスクを選択します。

その後、セマフォが解放されると、待っていたタスクが再び実行可能になります。

今回のログでは、その結果として生じた待ち時間を確認できました。

ソースコード上で理解していたタスク間同期の仕組みを、実機の動作として観察できたわけです。

ただし、今回のログだけでは、セマフォが解放された正確な時刻や、実行可能状態になってから実際にCPUを獲得するまでの時間までは分かりません。

そのためには、セマフォを解放する側やスケジューラ側にもトレースが必要です。

## 4. Gitのバージョンを起動時に表示する

実機で何度もファームウェアを書き換えていると、現在どのソースコードが動いているのか分からなくなることがあります。

そこで、Gitのコミット情報を起動メッセージに表示することにしました。

使用するコマンドは次のとおりです。

```bash
git describe --tags --always --dirty
```

このコマンドをビルド時に実行し、`build/version.h` を自動生成します。

生成されるヘッダの例です。

```c
#define FW_VERSION "b558e02-dirty"
```

`-dirty` は、コミット済みの状態から追跡対象ファイルに変更があることを示します。

### Makefileで自動生成する

Makefileには、次のような処理を追加しました。

```make
VERSION_H := build/version.h

.PHONY: FORCE
FORCE:

$(VERSION_H): FORCE
	@mkdir -p build
	@version=$$(git describe --tags --always --dirty); \
	printf '#define FW_VERSION "%s"\n' "$$version" > build/version.h.tmp; \
	if ! cmp -s build/version.h.tmp $(VERSION_H); then \
		mv build/version.h.tmp $(VERSION_H); \
	else \
		rm build/version.h.tmp; \
	fi
```

一時ファイルを生成し、内容が変化した場合だけ `version.h` を更新します。

これにより、Gitのバージョンが変わらない場合は、不要な再コンパイルを避けられます。

また、`inittsk.o` が `version.h` に依存するように設定しました。

### 起動メッセージの変更

`kernel/inittsk.c` で生成されたヘッダを読み込みます。

```c
#include "../build/version.h"
```

起動メッセージには、文字列リテラルの連結を利用しました。

```c
tm_putstring(
    "Strat Try Kernel - trykernel-lab (" FW_VERSION ")\n"
);
```

実機では、次の表示を確認できました。

```text
Strat Try Kernel - trykernel-lab (b558e02-dirty)
```

これで、UARTログを見ただけでファームウェアのGitバージョンを確認できます。

なお、`Strat` という綴りは、元のサンプルの表示をそのまま残しています。

## 5. 今回の実装で注意したいこと

今回のトレースは便利ですが、いくつか制約があります。

### UART出力がタスクの実行時間に影響する

今回使用している `tm_putstring()` は、UARTの送信完了を待ちながら文字を出力するポーリング方式です。

そのため、トレースを追加すると、タスクの処理時間が増えます。

つまり、**観察するための処理が、観察対象の動作に影響する**という問題があります。

トレースを増やすほど、この影響は大きくなります。

今回の550msという値も、トレースを含む実行環境で観察した時間であり、トレースが存在しない場合の待ち時間と完全に同じとは限りません。

### タイマの分解能は10ms

ログにはミリ秒単位の数値を表示していますが、実際には10msごとに更新しています。

したがって、数ms程度の処理時間を正確に測定する用途には向いていません。

### 複数タスクからの同時出力

今後、複数のタスクからトレースを出力する場合、UART出力の競合やログの混在にも注意が必要です。

現在の実装は、汎用的なトレース基盤として完成しているわけではありません。

## 6. Gitで管理する

今回の変更は、独立した開発ブランチで作業しました。

```text
feature/uart-trace
```

コミットメッセージは次のとおりです。

```text
feat: add UART tracing and automatic Git version display
```

実機で動作を確認した後、`main` にマージしました。

```text
f513b88 Merge branch 'feature/uart-trace'
743dfdf feat: add UART tracing and automatic Git version display
```

書籍のコードを再現するだけでなく、自分なりに機能を追加していくため、今後も機能単位でブランチを分けて開発する予定です。

## まとめ

今回は、Try Kernelに時刻付きUARTトレースを追加しました。

これにより、セマフォの獲得待ちに550msかかったことを実機で観察できました。

OSの内部処理は、ソースコードを読めば理解できます。しかし、実際にタスクがどのように動いているかを確認するには、動作を観察する手段が必要です。

今回のトレースは、そのための最初の一歩です。

次は、セマフォを解放する側のタスクにもトレースを追加し、待ち状態から復帰するまでの流れを詳しく観察したいと思います。

さらにイベントフラグにも対象を広げ、タスク間同期の動作を時系列で追えるようにする予定です。

将来的には、UARTへその場で出力するのではなく、メモリ上にイベントを記録して後から取り出す方式も検討したいと考えています。

OSを作るだけでなく、**自分で作ったOSの動作を、自分で観察できるようにする。**

そのための基盤ができました。

## 参考資料

- Interface 2023年7月号『ラズパイPicoで1500行 ゼロから作るOS』
- [Try Kernel開発リポジトリ](https://github.com/yamao2253/trykernel-lab)
