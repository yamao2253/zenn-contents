---
title: "Try Kernelのセマフォ — 共有資源の競合を防ぐ排他制御"
emoji: "⚙️"
type: "tech"
topics: ["raspberrypi", "pico", "rtos", "c"]
published: false
---

## はじめに

前回の記事「[Try Kernelのイベントフラグ — 複数の条件でタスクを同期する](https://zenn.dev/helowota/articles/pico-trykernel-event-flag)」では、ボタン操作をイベントフラグで通知し、2つのLED制御タスクを動かしました。

それぞれのタスクは異なる点滅パターンを持っています。しかし、両方のタスクが同じLEDを操作するため、点滅中にもう一方のボタンを押すと、LEDの点滅パターンが重なってしまいました。

タスク間で「処理を開始する条件」を伝えるだけでは、共有するハードウェアへのアクセスの競合は防げません。

今回は、Try Kernelの**セマフォ**を使って、複数のタスクが同じ資源を順番に使うための**排他制御**を学びます。

## 排他制御とは？

排他制御とは、複数のタスクが同じ資源を使用するとき、互いの処理が不適切に干渉しないように制御することです。

共有資源には、LEDのようなI/Oデバイスのほか、複数のタスクから読み書きするグローバル変数などがあります。

### グローバル変数でも競合する

書籍P.92の図1では、共有するグローバル変数への演算が競合する例が紹介されています。

例えば、共有変数 `count` の初期値が0で、タスクAとタスクBがそれぞれ1を加算する場合を考えます。

```c
count = count + 1;
```

C言語では1行ですが、一般的な処理は「メモリから読み出す → 加算する → メモリへ書き戻す」という複数の操作から構成されます。

| 順序 | タスクA | タスクB | 共有変数 `count` |
|---|---|---|---|
| 1 | 0を読み出す | — | 0 |
| 2 | — | 0を読み出す | 0 |
| 3 | — | 1を計算して書き戻す | 1 |
| 4 | 1を計算して書き戻す | — | 1 |

本来は2回加算して2になるはずですが、タスクBの更新がタスクAの書き戻しで上書きされ、結果が1になっています。

このような問題を防ぐには、一連の操作の途中に別のタスクが同じ資源を操作しないようにする必要があります。

共有資源を操作する、排他制御が必要な一連の処理を**クリティカルセクション**と呼びます。

## セマフォとは？

セマフォは、利用可能な**資源の数**を管理するカーネルオブジェクトです。

タスクは共有資源を使う前にセマフォから資源を獲得し、使用後に返却します。必要な数の資源が残っていない場合は、獲得できるまで待ち状態になります。

今回のLEDは1つだけなので、セマフォの初期資源数を1にします。

| 操作 | 資源数 | タスクの動作 |
|---|---:|---|
| 初期状態 | 1 | LEDは使用可能 |
| タスク1が獲得 | 0 | タスク1がLEDを使用 |
| タスク2が獲得を要求 | 0 | タスク2は待ち状態 |
| タスク1が返却 | 1相当 | 待っていたタスク2が獲得可能になる |
| タスク2が獲得 | 0 | タスク2がLEDを使用 |

※待ちタスクがある場合、返却された資源はそのタスクへの割り当てに使われます。上表の「1相当」は資源返却の概念を表しており、必ずしも内部カウンタが1のまま保持されることを意味しません。

### バイナリセマフォとカウンティングセマフォ

セマフォには、資源数を0または1で扱う**バイナリセマフォ**と、複数の資源数を管理できる**カウンティングセマフォ**があります。

書籍P.93では、バイナリセマフォをカウンティングセマフォの資源数1の場合として説明しています。Try Kernelが実装するのはカウンティングセマフォで、今回は初期値と最大値を1にして排他制御に使います。

## Try Kernelのセマフォ機能

書籍では第4部第3章（P.92〜96）で、セマフォの仕組みとAPIの実装を解説しています。ここではアプリケーションから使う基本APIに注目します。

| API | 機能 |
|---|---|
| `tk_cre_sem` | セマフォを生成する |
| `tk_wai_sem` | 指定した数の資源を獲得する。足りなければ待つ |
| `tk_sig_sem` | 指定した数の資源を返却する |

### セマフォを生成する：tk_cre_sem

```c
ID tk_cre_sem(const T_CSEM *pk_csem);
```

生成情報を設定した `T_CSEM` 構造体へのポインタを渡します。

```c
typedef struct t_csem {
    ATR sematr;   /* セマフォ属性 */
    INT isemcnt;  /* セマフォ初期値 */
    INT maxsem;   /* セマフォ最大値 */
} T_CSEM;
```

`isemcnt` は生成時の資源数、`maxsem` は保持できる最大資源数です。成功すると、以後の操作で使うセマフォIDを返します。

書籍には `TA_TFIFO`、`TA_FIRST` などの属性が紹介されています。ただし、**Try Kernelは簡易実装のため属性による動作の切り替えを実装していません**。規格にあるすべての待ち順序を利用できるわけではない点に注意します。

### 資源を獲得する：tk_wai_sem

```c
ER tk_wai_sem(ID semid, INT cnt, TMO tmout);
```

`semid` はセマフォID、`cnt` は要求する資源数、`tmout` は待ち時間です。

```c
tk_wai_sem(semid, 1, TMO_FEVR);
```

これは資源を1つ獲得する操作です。資源がなければ、`TMO_FEVR` の指定によって獲得できるまで待ち続けます。

### 資源を返却する：tk_sig_sem

```c
ER tk_sig_sem(ID semid, INT cnt);
```

```c
tk_sig_sem(semid, 1);
```

使用を終えた資源を1つ返却します。資源を待っているタスクがあれば、そのタスクの待ち条件が成立するかを確認します。

## セマフォによる排他制御の流れ

今回のサンプルでは、イベントフラグとセマフォを組み合わせます。

1. ボタン検出タスクが、押されたボタンに対応するイベントフラグをセットする。
2. LED制御タスクがイベントフラグ待ちから復帰する。
3. LED制御タスクがセマフォを獲得する。
4. LEDの点滅処理を実行する。
5. 点滅が終わったらセマフォを返却する。

2つのLED制御タスクが同時期に動作を始めても、LEDを操作できるのはセマフォを獲得した側だけです。

### イベントフラグとの役割の違い

| 項目 | イベントフラグ | セマフォ |
|---|---|---|
| 主な役割 | イベント発生の通知と条件待ち | 資源数の管理と獲得待ち |
| 今回の用途 | BTN1・BTN2の押下を知らせる | LEDの使用権を1タスクに限定する |
| タスクが待つ条件 | 指定したビットがセットされること | 必要な資源数を獲得できること |

イベントフラグによって処理を始めるきっかけを伝え、セマフォによって共有資源の競合を防ぎます。

## Try Kernelで実際に動かしてみる

書籍P.96のリスト8「セマフォ実験プログラムのタスク実行関数」に対応するサンプルを使います。

### 使用するコードと開発環境

ソースコードは、著者の[GitHubリポジトリ](https://github.com/ytoyoyama/interface_trykernel)をもとにしています。

- 使用するソースコード：`part_4/sect_3`
- サンプル・アプリケーション：`application/usermain.c`
- セマフォの実装：`kernel/semaphore.c`

| 項目 | 内容 |
|---|---|
| ターゲット | Raspberry Pi Pico W（RP2040）＋ Robo Pico Board |
| ホストOS | Windows 11 |
| ビルド | GNU Make、Arm GNU Toolchain |
| 書き込み・デバッグ | OpenOCD、GDB、CMSIS-DAP Debug Probe |
| UART確認 | Tera Term（115200 bps、8N1、フロー制御なし、受信改行LF） |
| 入力 | Robo Pico BoardのBTN1（GP20）、BTN2（GP21） |
| 出力 | Robo Pico BoardのLED（GP2） |

### 書籍からの変更点

書籍のGPIO割り当てとRobo Pico Boardの配線が異なるため、GPIO番号だけ変更しました。セマフォとイベントフラグの制御処理は変更していません。

| 用途 | 書籍掲載コード | Robo Pico Board |
|---|---|---|
| BTN1 | GP14 | GP20 |
| BTN2 | GP13 | GP21 |
| LED | GP25 | GP2 |

変更対象は `application/usermain.c` と `boot/reset_hdr.c` です。具体的な変更はAppendixにまとめます。

### サンプルコードの処理の流れ

**ボタン検出タスク**は100ms間隔でボタンを監視し、押下を検出すると対応するイベントフラグのビットをセットします。

**LED制御タスク1**はbit 0を待ち、セマフォを獲得すると500ms点灯・500ms消灯を3回繰り返します。

**LED制御タスク2**はbit 1を待ち、セマフォを獲得すると100ms点灯・100ms消灯を5回繰り返します。

どちらも点滅を終えるとセマフォを返却し、再びイベントフラグを待ちます。

### ビルドと実機への書き込み

Windows環境向けに `part_4/sect_3` にMakefileを用意しました。前回のイベントフラグ編のMakefileに `kernel/semaphore.c` を追加しています。

```makefile
TARGET := trykernel
CC := arm-none-eabi-gcc
SIZE := arm-none-eabi-size
OBJDUMP := arm-none-eabi-objdump

CFLAGS := -mcpu=cortex-m0plus -mthumb -O0 -ffreestanding -std=c99 -Iinclude -g
LDFLAGS := -mcpu=cortex-m0plus -mthumb -T linker/pico_memmap.ld -nostartfiles

SRCS := boot/boot2.c \
        boot/reset_hdr.c \
        boot/vector_tbl.c \
        kernel/syslib.c \
        kernel/context.c \
        kernel/inittsk.c \
        kernel/scheduler.c \
        kernel/systimer.c \
        kernel/task_mange.c \
        kernel/task_queue.c \
        kernel/task_sync.c \
        kernel/eventflag.c \
        kernel/semaphore.c \
        application/usermain.c

ASRCS := kernel/dispatch.S

OBJS := $(patsubst %.c,build/%.o,$(SRCS)) \
        $(patsubst %.S,build/%.o,$(ASRCS))

ELF := build/$(TARGET).elf

.PHONY: all clean size sections

all: $(ELF)

$(ELF): $(OBJS) linker/pico_memmap.ld
	$(CC) $(LDFLAGS) -o $@ $(OBJS)
	$(SIZE) $@

build/%.o: %.c
	@mkdir -p $(dir $@)
	$(CC) $(CFLAGS) -c $< -o $@

build/%.o: %.S
	@mkdir -p $(dir $@)
	$(CC) $(CFLAGS) -c $< -o $@

size: $(ELF)
	$(SIZE) $(ELF)

sections: $(ELF)
	$(OBJDUMP) -h $(ELF)

clean:
	rm -rf build
```

PowerShellでビルドします。

```powershell
make clean
make
make size
```

Robo Pico Board向けにGPIOを変更した後の実測結果は次のとおりです。

```text
arm-none-eabi-size build/trykernel.elf
   text    data     bss     dec     hex filename
   7514     100    5768   13382    3446 build/trykernel.elf
```

続いてPowerShellでOpenOCDを起動します。

```powershell
openocd -f interface/cmsis-dap.cfg -f target/rp2040.cfg
```

別のPowerShellからGDBを起動します。

```powershell
arm-none-eabi-gdb .\build\trykernel.elf
```

GDBから接続して書き込みます。

```text
target remote localhost:3333
monitor reset halt
load
monitor reset halt
```

Tera TermでUART受信を準備してから、GDBで実行します。

```text
continue
```

## 動作結果を確認する

### BTN1の点滅中にBTN2を押す

BTN1を押して低速点滅が始まった後、その点滅中にBTN2を押しました。

**実機では、BTN1による低速点滅が終了してから、BTN2による高速点滅が5回実行されることを確認しました。**

前回のイベントフラグ編では2つの点滅処理が重なりましたが、今回は先に始まった点滅が終わるまで次の点滅が開始されませんでした。

UART出力は次のとおりです。

```text
Strat Try Kernel
BTN-0 ON
BTN-1 ON
BTN-0 ON
BTN-1 ON
```

`Strat Try Kernel` はサンプルコードの出力をそのまま記録しています。`BTN-0 ON` と `BTN-1 ON` はボタンの押下検出を示します。

UARTログからはボタン検出順序が分かりますが、LEDの点滅順序は直接読み取れません。排他制御によって点滅が重ならなかったことは、**実機のLEDを観察して確認した結果**です。

## サンプルコードで使うセマフォ

### セマフォの生成

`usermain()` でセマフォを生成し、2つのLED制御タスクから同じセマフォIDを使用します。

今回のLEDは1つなので、セマフォの初期資源数と最大資源数をともに1にします。

```c
/* セマフォ生成情報の設定例 */
T_CSEM csem = {
    .sematr = TA_TFIFO | TA_FIRST,
    .isemcnt = 1,
    .maxsem = 1,
};

/* セマフォの生成 */
semid = tk_cre_sem(&csem);
```

※このコードは書籍の仕様に沿った説明例です。**実際の `application/usermain.c` における変数宣言・属性指定と一致するかは、公開前に確認します。**

### LED制御タスク1

セマフォの使い方が分かるよう、LED制御部分を抜き出します。GPIOはRobo Pico Board向けのGP2です。

```c
/* イベントフラグ待ち */
tk_wai_flg(flgid, (1<<0), TWF_ANDW | TWF_BITCLR, &flgptn, TMO_FEVR);

/* LEDの使用権を獲得 */
tk_wai_sem(semid, 1, TMO_FEVR);

for (INT i = 0; i < 3; i++) {
    out_w(GPIO_OUT_SET, (1<<2));
    tk_dly_tsk(500);
    out_w(GPIO_OUT_CLR, (1<<2));
    tk_dly_tsk(500);
}

/* LEDの使用権を返却 */
tk_sig_sem(semid, 1);
```

### LED制御タスク2

タスク2も同じセマフォを獲得します。

```c
/* イベントフラグ待ち */
tk_wai_flg(flgid, (1<<1), TWF_ANDW | TWF_BITCLR, &flgptn, TMO_FEVR);

/* LEDの使用権を獲得 */
tk_wai_sem(semid, 1, TMO_FEVR);

for (INT i = 0; i < 5; i++) {
    out_w(GPIO_OUT_SET, (1<<2));
    tk_dly_tsk(100);
    out_w(GPIO_OUT_CLR, (1<<2));
    tk_dly_tsk(100);
}

/* LEDの使用権を返却 */
tk_sig_sem(semid, 1);
```

セマフォの獲得と返却の間にあるLED点滅処理が、今回のクリティカルセクションに当たります。

ここで `tk_dly_tsk()` によりタスクが時間待ちになっても、**セマフォは自動的には返却されません**。したがって、点滅中に別のタスクへCPUが切り替わっても、そのタスクはLEDの使用権を獲得できません。

## セマフォ待ちを含むタイムチャート

<!-- TODO: 実際のソースコードと整合したタイムチャート画像を作成し、以下のタグを有効化する。 -->
<!--
<img src="/images/pico-trykernel-semaphore/pico-trykernel-semaphore-timeline.png"
     alt="イベントフラグ待ち、セマフォ待ち、LED点滅の時間関係"
     width="700">
-->

タイムチャートでは、イベントフラグ待ち、セマフォ待ち、`tk_dly_tsk()` による時間待ちを区別する予定です。

BTN1の点滅中にBTN2を押した場合、タスク2はイベントフラグ待ちから復帰しても、セマフォを獲得できないため待ち状態になります。タスク1がセマフォを返却すると、タスク2がLEDの点滅処理を開始できます。

## 実機で動かして分かったこと

前回のイベントフラグ編と今回のセマフォ編では、ボタン入力とLED点滅の基本動作は同じです。

違うのは、**LED点滅の前後でセマフォの獲得と返却を行う**点です。

| 項目 | イベントフラグ編 | セマフォ編 |
|---|---|---|
| ボタン操作の通知 | イベントフラグ | イベントフラグ |
| LED使用前の獲得処理 | なし | `tk_wai_sem()` |
| LED使用後の返却処理 | なし | `tk_sig_sem()` |
| 点滅中に別ボタンを押した場合 | 点滅が重なった | 先の点滅終了後に次の点滅が始まった |

この比較によって、タスクを同期させることと、共有資源を排他的に使わせることは別の役割であると確認できました。

## おわりに

今回はTry Kernelのセマフォを使い、複数のタスクが共有するLEDを順番に操作する排他制御を確認しました。

前回のイベントフラグではタスク間の通知を実現しましたが、それだけでは共有資源へのアクセスの競合を防げませんでした。セマフォを追加することで、先に始まった点滅が終了するまで、次のタスクがLEDを操作しないようにできました。

書籍では、この第4部でTry Kernelの基本的なRTOS機能が一通り完成し、続く第5部でアプリケーションの作成に進みます。

## 参考資料

- CQ出版社『Interface』2023年7月号「ラズパイPicoで1500行 ゼロから作るOS」
  - 第4部 第2章「複数のタスク間で複雑な同期もできる…イベント・フラグ」
  - 第4部 第3章「複数のタスク間での資源競合を防ぐ…セマフォによる排他制御」
- [Try Kernel 著者公開ソースコード](https://github.com/ytoyoyama/interface_trykernel)
- [Try Kernelのタスク管理 — なぜOSが必要なのか？](https://zenn.dev/helowota/articles/pico-trykernel-task-management)
- [Try Kernelのタスク付属同期 — タスクを待たせる・起こす](https://zenn.dev/helowota/articles/pico-trykernel-task-sync)
- [Try Kernelのイベントフラグ — 複数の条件でタスクを同期する](https://zenn.dev/helowota/articles/pico-trykernel-event-flag)

---

## Appendix：Robo Pico Board向けGPIO設定

書籍のGPIO設定をRobo Pico Boardに合わせて変更しました。

| 用途 | 書籍掲載コード | Robo Pico Board |
|---|---|---|
| BTN1 | GP14 | GP20 |
| BTN2 | GP13 | GP21 |
| LED | GP25 | GP2 |

`application/usermain.c` のボタン入力の初期化、入力マスク、押下判定をGP20・GP21に変更しました。2つのLED制御タスクのLED出力をGP2に変更しました。

`boot/reset_hdr.c` のLED出力初期化もGP2に変更しました。

```c
out_w(GPIO_OE_CLR, (1<<2));
out_w(GPIO_OUT_CLR, (1<<2));
out_w(GPIO_CTRL(2), 5);
out_w(GPIO_OE_SET, (1<<2));
```

なお、今回の作業開始時点のローカルソースは、以前のブレッドボード実験のためLEDがGP15に変更されていました。記事の差分説明では、書籍掲載コードのGP25を基準としています。

イベントフラグのビット番号とセマフォの制御処理は変更していません。
