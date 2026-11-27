---
title: "Raspberry Pi Picoのリセット直後のCPU動作をGDBで理解する"
emoji: "🔍"
type: "tech"
topics: ["raspberrypi", "pico", "arm", "gdb", "embedded"]
published: false
---

# はじめに

CQ出版社『Interface』2023年7月号の特集「ラズパイPicoで1500行 ゼロから作るOS」を題材に、Try Kernelの学習を続けています。

これまでにTry Kernelの開発環境を構築し、OSの各機能やGDBによるデバッグ方法を学んできました。

今回からは、単にプログラムを動かすだけでなく、**なぜそのように動作するのか** を掘り下げていきます。

本シリーズでは、デバッグプローブとGDBを活用し、CPUのレジスタやメモリの内容を確認するとともに、逆アセンブルや命令単位のステップ実行によって実機の動作を観察することで、CPUやOSの仕組みへの理解を深めていきます。

書籍やソースコードを読むだけでは理解しにくい部分についても、CPUの仕様と実機で観察した結果を関連づけることで、理解を深めていきたいと思います。

本シリーズで使用するデバッグ環境の構築方法と、OpenOCDおよびGDBの基本的な操作方法については、以下の記事にまとめています。

- [「ラズパイPicoで1500行 ゼロから作るOS」を2026年に動かす](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)
- [OpenOCDとGDBでRaspberry Pi Picoをデバッグする](https://zenn.dev/helowota/articles/pico-openocd-gdb-debug)

## 今回取り上げる内容

最初に取り上げるのは、書籍の第2部第1章（P.47〜48）と第2章（P.49〜50）の起動処理です。特に、P.48の図1「Arm Cortex-M0+の例外ベクタ・テーブル」と、P.50の図2（a）の冒頭に着目します。

書籍では、Cortex-Mの場合、ROMの先頭に例外ベクタテーブルがあり、そこにリセット時のプログラムの実行開始アドレスが設定されていると説明されています。

この説明を読んで、いくつかの疑問が生じました。

- ROMの先頭とは、具体的にどのアドレスを指すのか？
- CPUはリセット直後に0番地から命令を実行するのではないのか？
- 例外ベクタテーブルに格納されているアドレスは、どのようにプログラムカウンタ（PC）に設定されるのか？
- スタックポインタ（SP）の初期値は、どこに格納され、いつ設定されるのか？
- 最初の命令を実行する前のCPUの動作を、GDBで確認することはできるのか？

今回は、これらの疑問を起点として、GDBでリセット直後のCPUの状態を観察し、ハードウェアによる初期化からソフトウェアによる命令実行へと移る部分を理解します。

# 使用環境

| 項目 | 内容 |
|---|---|
| OS | Windows 11 |
| シェル | PowerShell 7 |
| ターゲット | Raspberry Pi Pico W（RP2040） |
| CPU | Arm Cortex-M0+ |
| デバッグ接続 | CMSIS-DAP / SWD |
| OpenOCD | xPack OpenOCD 0.12.0-7.1 |
| GDB | MSYS2 GDB Multiarch |
| デバッグ対象のELF | `part_4/sect_3/build/trykernel.elf` |

## 事前準備

本記事では、Try Kernelの `part_4/sect_3` をビルド済みであり、`build/trykernel.elf` が生成されているものとします。また、デバッグプローブとターゲットの接続、OpenOCDおよびGDBのセットアップも完了しているものとします。

開発環境やデバッグ環境がまだ構築できていない場合は、「はじめに」で紹介した2つの記事を参照してください。今回使用するGDB Multiarchの環境構築については、以下の記事にまとめています。

- [Windows 11でGDBの日本語文字化けを解消 ― MSYS2 GDB Multiarchを使う](https://zenn.dev/helowota/articles/pico-gdb-multiarch-windows-env)

今回はTry Kernelのソースコードを変更せず、RP2040のBoot ROM上の例外ベクタテーブルとリセット直後のCPUの状態を観察します。

## OpenOCDとGDBの起動

PowerShellでOpenOCDを起動します。

```powershell
openocd -f interface/cmsis-dap.cfg -c "adapter speed 1000" -f target/rp2040.cfg
```

別のPowerShellで `part_4/sect_3` に移動し、GDBを起動します。

```powershell
gdb-multiarch -q build/trykernel.elf
```

GDBからOpenOCDに接続します。

```gdb
target remote localhost:3333
```

GDBからOpenOCDへの接続が完了したら、リセット直後のCPUの状態を確認していきます。

# 最初の命令を実行する前のCPUの動作

## 例外ベクタテーブル

例外ベクタテーブルは、例外が発生したときに実行するハンドラの開始アドレスなどを格納したデータです。

書籍P.48の図1では、Cortex-M0+の例外ベクタテーブルの構成が示されています。今回はリセットに関係する先頭の2項目に着目します。

| テーブル先頭からのオフセット | 内容 |
|---|---|
| `0x00` | スタックポインタの初期値 |
| `0x04` | リセットベクタ |

RP2040では、書籍P.49の図1に示されるように、アドレス `0x00000000` から16 KiBのBoot ROMが配置されています。リセット時に参照される例外ベクタテーブルも、このBoot ROMにあります。

したがって、リセット時に参照される先頭の2項目のアドレスは次のようになります。

```text
0x00000000  スタックポインタの初期値
0x00000004  リセットベクタ
```

ここには実行する命令ではなく、CPUの初期化に使用するデータが格納されています。

Cortex-M0+は0番地から順番に命令を実行するわけではありません。リセット時にCPUのハードウェアがこれらのデータを読み、命令を実行するための初期状態を設定します。

ここで、2つのレジスタに着目します。

- PC（Program Counter）：命令の実行位置を示すレジスタ
- SP（Stack Pointer）：スタックの現在位置を示すレジスタ

リセット時には、例外ベクタテーブルの内容に基づいて、MSP（Main Stack Pointer）、PC、およびArmアーキテクチャの命令実行状態を示すThumb状態が設定されます。

MSPとSPの関係は後ほど説明します。Thumb状態の詳細については、次回の記事で取り上げます。

## 確認すること

Boot ROM上の例外ベクタテーブルとリセット直後のCPUレジスタを比較し、CPUのスタックポインタと命令の実行開始アドレスがどのような値に設定されるのかを確認します。

## 操作手順

CPUをリセットして停止させます。

```gdb
monitor reset halt
```

実行結果：

```text
[rp2040.core0] halted due to debug-request, current mode: Thread
xPSR: 0xf1000000 pc: 0x000000ea msp: 0x20041f00
[rp2040.core1] halted due to debug-request, current mode: Thread
xPSR: 0xf1000000 pc: 0x000000ea msp: 0x20041f00
```

RP2040は2つのCortex-M0+コア（core0、core1）を搭載しています。

OpenOCDの出力から、両コアが停止したことを確認できます。Try Kernelではcore0のみを使用するため、以降はcore0の状態を観察します。

今回の環境では、リセット後にGDBのレジスタ表示が古い状態のままになる現象が確認されたため、`maintenance flush register-cache` コマンドでGDBのレジスタキャッシュを破棄します。

この現象の詳細については、後ほど説明します。

```gdb
maintenance flush register-cache
```

まず、例外ベクタテーブルが実際に0番地に配置されていることを確認します。

例外ベクタテーブルの先頭に格納されている **MSPの初期値とリセットベクタ** を確認するため、0番地から32ビットのデータを2個、16進数で表示します。

```gdb
x/2wx 0x00000000
```

実行結果：

```text
0x0:    0x20041f00      0x000000eb
```

続いてCPUレジスタを表示します。

```gdb
info registers sp msp psp pc xpsr
```

実行結果：

```text
sp             0x20041f00          0x20041f00
msp            0x20041f00          0x20041f00
psp            0xfffffffc          0xfffffffc
pc             0xea                0xea
xpsr           0xf1000000          -251658240
```

## 結果と考察

書籍P48、図1のメモリマップと対応付けて、ベクタテーブルの値とリセット時のCPUレジスタの値を比較します。

| 確認対象 | オフセットアドレス | ベクタテーブルの値 | レジスタの値 |
|---|---:|---:|---:|
| スタックポインタ初期値 | `0x0` | `0x20041f00` | MSP = `0x20041f00` |
| リセットベクタ | `0x4` | `0x000000eb` | PC = `0x000000ea` |

MSPは例外ベクタテーブルの先頭の値と一致しています。一方、PCはリセットベクタの値と1だけ異なっています。

Cortex-M0+の命令は2バイト境界に配置されるため、命令のアドレスは必ず偶数となります。そのため、リセットベクタの最下位ビット（bit 0）は、アドレスの一部ではなく、Thumb状態を指定するために使用されます。

Cortex-M0+はThumb命令セットにのみ対応しています。Thumb状態で実行することを示すため、リセットベクタのビット0は `1` になっています。

したがって、リセットベクタの値が `0x000000eb` であっても、PCには最下位ビットを0にした `0x000000ea` が設定されます。Thumb状態の詳細については、次回の記事で取り上げます。

以上の結果から、例外ベクタテーブルの内容に基づいてMSPとPCが設定されていることを確認できました。今回の実測値とCortex-M0+の仕様をもとに、リセット時の動作を次のように整理できます。

```text
リセット
    ↓
ハードウェアによる初期化
    MSP       ← 0x20041f00
    PC        ← 0x000000ea
    Thumb状態 ← 有効（bit 0 = 1）
    ↓
0x000000ea から命令実行開始
```

ここで重要なのは、ハードウェアによる初期化は、CPUが命令を実行して行う処理ではないということです。そのため、GDBの命令単位のステップ実行で初期化そのものを追うことはできません。

今回は、初期化に使用するメモリの値と、初期化後に停止したCPUのレジスタ値を比較することで、Cortex-M0+の仕様と整合する結果が得られることを確認しました。CPU内部の処理順序やバスアクセスを直接観測したわけではありません。

# SP、MSP、PSP

先ほどのGDBの結果には、SP、MSP、PSPという3つの名前が表示されていました。

Cortex-M0+には、独立した2つのスタックポインタがあります。

| 名称 | 意味 |
|---|---|
| MSP | Main Stack Pointer。リセット直後やHandlerモードで使用する |
| PSP | Process Stack Pointer。OSのタスクなどで使用できる |
| SP（R13） | 現在選択されているスタックポインタ |

SPは独立した3つ目のスタックポインタではありません。MSPとPSPのいずれかが選択され、SPとして扱われます。

```text
MSP ──┐
      ├─ 選択 ── SP (R13)
PSP ──┘
```

スイッチでMSPとPSPのどちらかを選択し、SPとして使用するイメージです。ただし、これは選択の仕組みを概念的に示したものであり、実際の内部回路を示した図ではありません。

Cortex-M0+にはThreadモードとHandlerモードがあり、通常のプログラムはThreadモードで、リセット以外の例外ハンドラはHandlerモードで実行されます。

| 動作モード | 使用するスタックポインタ |
|---|---|
| Threadモード | CONTROL.SPSEL（bit 1）が0ならMSP、1ならPSP |
| Handlerモード | MSP |

リセット直後はThreadモードで、MSPが選択されます。OSのタスク切り替えでの使い分けについては、今後の学習で取り上げます。

GDBでは、SP、MSP、PSPの値だけでなく、スタックポインタの選択を制御するCONTROLレジスタの値も確認できます。CONTROLレジスタのbit 1（SPSEL）に着目することで、どちらのスタックポインタが選択されているのかを確認できます。

## 確認すること

リセット直後のSP、MSP、PSP、およびCONTROL.SPSELの値をGDBで確認し、スタックポインタの選択状態がCortex-M0+の仕様と一致することを確認します。

## 操作手順

SP、MSP、PSPを表示します。

```gdb
info registers sp msp psp
```

実行結果（先ほどのレジスタ表示から抜粋）：

```text
sp             0x20041f00          0x20041f00
msp            0x20041f00          0x20041f00
psp            0xfffffffc          0xfffffffc
```

続いて、スタックポインタの選択状態を確認します。

Threadモードでは、CONTROLレジスタのbit 1（SPSEL）の値によって、使用するスタックポインタが決まります。

| SPSEL（bit 1） | 選択されるスタックポインタ |
|---|---|
| 0 | MSP |
| 1 | PSP |

GDBの `info registers` コマンドでCONTROLレジスタを表示し、そのbit 1の値を確認します。

```gdb
info registers control
```

実行結果：

```text
control        0x0                 0
```

CONTROLレジスタの値は `0x0` です。下位8ビットを2進数で表すと、次のようになります。

```text
ビット番号  7 6 5 4 3 2 1 0
           0 0 0 0 0 0 0 0
                       ↑
                     SPSEL
```

bit 1（SPSEL）は0であるため、ThreadモードではMSPが選択されていることが分かります。

GDBでは、SP（R13）の値として、CPUが現在選択しているスタックポインタの値が表示されます。そのため、今回のようにMSPが選択されている場合は、SPとMSPに同じ値が表示されます。

## 結果と考察

SPとMSPの値が一致し、PSPとは異なっています。これはリセット直後にMSPが選択されるというCortex-M0+の仕様と一致します。

```text
MSP (0x20041f00) ──┐
                   ├─ MSPを選択 ── SP (0x20041f00)
PSP (0xfffffffc) ──┘
```

なお、PSPの値は今回の観測結果です。Cortex-M0+の仕様ではPSPのリセット値は規定されておらず、必ず `0xfffffffc` になるという意味ではありません。

<!-- TODO: info registers control をリセット直後に実行し、CONTROL.SPSEL = 0 を直接確認して追記する。 -->

## MSP初期値はなぜ0x20041f00なのか（書籍外の補足）

書籍P.49の図1では、RP2040のSRAMは `0x20000000` から264 KiBと示されています。

| 項目 | 値 |
|---|---|
| SRAMの先頭 | `0x20000000` |
| SRAMの容量 | 264 KiB（`0x42000` バイト） |
| SRAMの終了境界 | `0x20042000` |

一方、今回のMSP初期値は `0x20041f00` であり、SRAMの終了境界より256バイト小さくなっています。

Raspberry Piが公開しているRP2040のBoot ROMソースを確認すると、`bootrom/bootrom.ld` では `_stacktop` が次のように定義されています。

```ld
/* Leave room above the stack for stage 2 load, so that stage 2
   can image SRAM from its beginning */
_stacktop = ORIGIN(SRAM) + LENGTH(SRAM) - 256;
```

コメントには、セカンドステージのブートローダ（boot2）がSRAMの先頭からプログラムイメージを配置できるように、スタックより上位のアドレスにboot2のロード領域を確保する旨が記述されています。

また、`bootrom/bootrom_rt0.S` では例外ベクタテーブルの先頭に `_stacktop` が設定されています。

実際に計算すると、次のようになります。

```text
0x20000000 + 0x42000 - 0x100 = 0x20041f00
```

例外ベクタテーブルの値およびGDBで確認したMSP初期値と一致します。

```text
高位アドレス
0x20042000 ──────────── SRAMの終了境界
           boot2用の領域（256バイト）
0x20041f00 ──────────── MSPの初期値
              ↓ スタックが伸びる方向
           SRAM
0x20000000 ──────────── SRAMの先頭
低位アドレス
```

ここで示しているのは公開ソースの定義と今回の実測値の対応です。boot2が実際に転送される過程については、次回以降に確認します。また、このMSP初期値はBoot ROM側の設定であり、Try Kernel側のスタックの設定とは別です。

参考：

- [pico-bootrom-rp2040 / bootrom.ld](https://github.com/raspberrypi/pico-bootrom-rp2040/blob/master/bootrom/bootrom.ld)
- [pico-bootrom-rp2040 / bootrom_rt0.S](https://github.com/raspberrypi/pico-bootrom-rp2040/blob/master/bootrom/bootrom_rt0.S)

# GDBのレジスタキャッシュに関する注意点


今回、`monitor reset halt` を実行したにもかかわらず、GDBでPCを表示するとTry Kernel側の `Reset_Handler` を指していることがありました。

| 確認方法 | PCの表示値 |
|---|---|
| OpenOCDのリセット・停止メッセージ | `0x000000ea` |
| GDBのレジスタ表示 | `0x100007b4 <Reset_Handler>` |

OpenOCDとGDBの表示が一致していません。

そこで、GDBのレジスタキャッシュを破棄して再度レジスタを表示しました。

```gdb
monitor reset halt
maintenance flush register-cache
info registers sp msp psp pc xpsr
```

先ほどの実機検証で示したとおり、PCの表示は `0x000000ea` となり、OpenOCDの表示と整合するようになりました。今回の現象はGDBのレジスタキャッシュによるものと考えられます。ただし、ツール内部の詳細な原因までは調査していません。

# まとめ

今回は書籍の第2部の起動処理を題材に、GDBでリセット直後のCPUの状態を確認しました。

- CPUは0番地の内容を命令として実行するのではなく、例外ベクタテーブルを参照して命令実行の準備をする。
- 例外ベクタテーブルの先頭にはMSPの初期値が格納されている。
- リセットベクタにはリセットハンドラの開始アドレスとThumb状態を示す情報が含まれている。
- SP、MSP、PSPの関係を整理し、リセット直後にMSPが選択されていることを実測値から確認できた。
- Boot ROM側のMSP初期値について、メモリマップと公開ソースの定義から根拠を確認できた。

特に理解が深まったのは、**CPUが最初の命令を実行する前に、ハードウェアによってMSPやPCが設定される** という点です。これまで漠然と理解していた起動処理を、実際のメモリやレジスタの値と結びつけることができました。

次回は、今回のレジスタ表示に登場したxPSRとThumb命令について掘り下げ、Boot ROMのリセットハンドラを逆アセンブルし、命令単位でステップ実行します。

# 参考資料

- CQ出版社『Interface』2023年7月号 特集「ラズパイPicoで1500行 ゼロから作るOS」P.47〜50
- [Arm Cortex-M0+ Devices Generic User Guide（ARM DUI 0662A）](https://www.keil.com/dd/docs/datashts/arm/cortex_m0p/r0p0/dui0662a_cortex_m0p_r0p0_dgug.pdf) — 2.1.3 Core registers、2.3.4 Vector table
- [Raspberry Pi公式 RP2040 Boot ROMソース](https://github.com/raspberrypi/pico-bootrom-rp2040)
