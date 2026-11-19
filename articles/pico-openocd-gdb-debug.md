---
title: "OpenOCDとGDBでRaspberry Pi Picoをデバッグする"
emoji: "🔧"
type: "tech"
topics: ["raspberrypi", "pico", "openocd", "gdb"]
published: true
---

# はじめに

『Interface 2023年7月号』の特集「ラズパイPicoで1500行 ゼロから作るOS」を読み進めながら、OpenOCDとGDBを使った実機デバッグを行っています。

OpenOCDとGDBは組込み開発で便利なツールですが、しばらく使わないと細かいコマンドや操作手順を忘れてしまい、使うたびに調べ直すことがあります。

そこで、両者の役割を整理するとともに、基本的なデバッグ操作を備忘録としてまとめました。

本記事では、Windows 11上のPowerShellを使用し、Raspberry Pi Pico W（RP2040）をコマンドラインからデバッグする方法を紹介します。

開発環境の構築方法については、以下の記事を参照してください。

[「ラズパイPicoで1500行 ゼロから作るOS」を2026年に動かす](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)

本記事では環境構築の説明を省略し、OpenOCDとGDBの役割や、実際のデバッグ操作に焦点を当てます。

# OpenOCDとGDBの役割

OpenOCDとGDBは、どちらもデバッグに使用するツールですが、役割が異なります。

- **GDB（GNU Debugger）**：ブレークポイントの設定、ステップ実行、変数やレジスタの参照など、デバッグ操作を行う。
- **OpenOCD（Open On-Chip Debugger）**：GDBからの指示を受け、デバッグプローブを介してマイコンを制御する。

今回のデバッグ環境は、次の構成です。

```text
Windows 11
┌─────────────────────────────────────────┐
│                                         │
│  GDB                                    │
│   │                                     │
│   │ GDB Remote Serial Protocol          │
│   │ (TCP/IP localhost:3333)             │
│   ▼                                     │
│  OpenOCD                                │
│   │                                     │
└───┼─────────────────────────────────────┘
    │ USB（CMSIS-DAP）
    ▼
 Debug Probe（Pico W）
    │
    │ SWD
    ▼
 Raspberry Pi Pico W（RP2040）
```

GDBとOpenOCDは、Windows上でそれぞれ独立したプロセスとして動作します。

GDBはTCP/IPでOpenOCDのGDBサーバに接続します。OpenOCDはUSB経由でDebug Probeと通信し、SWD（Serial Wire Debug）を使ってRP2040にアクセスします。

つまり、**GDBはデバッグ操作を行うためのインタフェース、OpenOCDはマイコンとの通信を仲介する役割**です。

VSCodeなどのGUIからデバッグする場合も基本的な仕組みは同じですが、本記事ではそれぞれの役割を理解するため、OpenOCDとGDBを個別に起動します。

---

# OpenOCDの起動

PowerShellを開き、次のコマンドを実行します。

```powershell
openocd -f interface/cmsis-dap.cfg -c "adapter speed 1000" -f target/rp2040.cfg
```

各オプションの意味は次のとおりです。

| 指定 | 内容 |
|---|---|
| `-f interface/cmsis-dap.cfg` | デバッグプローブのインタフェースをCMSIS-DAPに指定 |
| `-c "adapter speed 1000"` | SWDの通信速度を1000 kHz（1 MHz）に指定 |
| `-f target/rp2040.cfg` | デバッグ対象をRP2040に指定 |

## 設定ファイル（cfg）の役割

OpenOCDには、さまざまなデバッグプローブやマイコンに対応する設定ファイル（`.cfg`）が標準で用意されています。

今回 `-f` オプションで指定している2つの設定ファイルも、OpenOCDに付属するもので、自分で作成する必要はありません。

- `interface/cmsis-dap.cfg`：CMSIS-DAP対応のデバッグプローブを使用するための設定ファイル
- `target/rp2040.cfg`：RP2040をデバッグするための設定ファイル

OpenOCDでは、デバッグプローブ側の設定と、デバッグ対象のマイコン側の設定を分けて管理しています。

そのため、デバッグプローブやマイコンを変更する場合も、対応する設定ファイルを選択することで構成を切り替えられます。

### 設定ファイルのカスタマイズ

ここまでは、OpenOCDに付属する標準の設定ファイルを使用しました。

標準の設定ファイルを変更しなくても、`-c` オプションでOpenOCDのコマンドを指定し、起動時に実行できます。今回の `-c "adapter speed 1000"` は、SWDの通信速度を設定するコマンドを実行している例です。

さらに、使用するデバッグプローブやマイコンの設定、通信速度などを1つにまとめたい場合は、**独自の `.cfg` ファイルを作成する方法**もあります。

例えば、`my-pico-debug.cfg` というファイルを次の内容で作成します。

```tcl
source [find interface/cmsis-dap.cfg]
adapter speed 1000
source [find target/rp2040.cfg]
```

`source [find ...]` は、OpenOCDに付属する設定ファイルを検索して読み込むための記述です。

このファイルをカレントディレクトリに保存すれば、PowerShellから次のように起動できます。

```powershell
openocd -f .\my-pico-debug.cfg
```

この方法なら、標準の設定ファイルを直接書き換えずに、プロジェクト専用の設定を管理できます。

なお、今回のTry Kernelでは独自の `.cfg` は作成せず、標準の設定ファイルと `-c` オプションを組み合わせて使用しています。

## 設定ファイルの保存先を確認する

今回使用しているxPack OpenOCD 0.12.0-7.1では、実行ファイルと設定ファイルは次の場所に配置されています。

```text
.content/
├── bin/
│   └── openocd.exe
└── openocd/
    └── scripts/
        ├── interface/
        │   └── cmsis-dap.cfg
        └── target/
            └── rp2040.cfg
```

実行ファイルの保存先は、PowerShellから次のコマンドで確認できます。

```powershell
Get-Command openocd | Select-Object Source
```

私の環境では、次の場所にインストールされていました（ユーザ名は置き換えています）。

```text
C:\Users\<ユーザ名>\AppData\Roaming\xPacks\@xpack-dev-tools\openocd\0.12.0-7.1\.content\bin\openocd.exe
```

また、OpenOCDが設定ファイルをどこから検索しているかは、次のコマンドで確認できます。

```powershell
openocd -c "echo [find interface/cmsis-dap.cfg]; echo [find target/rp2040.cfg]; shutdown"
```

実行結果は次のとおりです（起動時のバージョン表示などは省略し、ユーザ名は置き換えています）。

```text
C:/Users/<ユーザ名>/AppData/Roaming/xPacks/@xpack-dev-tools/openocd/0.12.0-7.1/.content/bin/../openocd/scripts/interface/cmsis-dap.cfg
C:/Users/<ユーザ名>/AppData/Roaming/xPacks/@xpack-dev-tools/openocd/0.12.0-7.1/.content/bin/../openocd/scripts/target/rp2040.cfg
shutdown command invoked
```

`bin/..` は、`bin` の1つ上のディレクトリを意味します。したがって、2つの設定ファイルは、実際には次のディレクトリ以下に配置されています。

```text
C:/Users/<ユーザ名>/AppData/Roaming/xPacks/@xpack-dev-tools/openocd/0.12.0-7.1/.content/openocd/scripts/
```

ここで、OpenOCDをどのディレクトリからでも実行できる理由も分かります。

- `openocd.exe` は、Windowsの環境変数 `PATH` から検索される。
- `-f` で指定した `.cfg` は、OpenOCD自身のスクリプト検索パスから検索される。

つまり、**実行ファイルと設定ファイルは、それぞれ別の仕組みで検索されます。**

そのため、カレントディレクトリがTry Kernelのソースコードの場所であっても、OpenOCDを起動できます。

## 起動後の動作

正常に起動すると、OpenOCDはCMSIS-DAP対応のDebug Probeを認識し、SWD経由でRP2040に接続します。

RP2040にはCortex-M0+が2コア搭載されているため、OpenOCDのログには `rp2040.core0` と `rp2040.core1` が表示されます。

また、OpenOCDはGDBからの接続を受け付けるためのGDBサーバを起動します。標準ではTCPポート3333を使用します。

```text
[rp2040.core0] Cortex-M0+ r0p1 processor detected
[rp2040.core1] Cortex-M0+ r0p1 processor detected
[rp2040.core0] starting gdb server on 3333
```

GDBからは、次のコマンドでOpenOCDに接続します。

```gdb
target remote localhost:3333
```

`localhost` は自分自身のPCを表し、`3333` はOpenOCDが待ち受けているTCPポート番号です。

今回はOpenOCDとGDBを同じWindows PC上で実行するため、`localhost` を指定しています。

なお、ポート番号はOpenOCDの設定で変更できますが、本記事では標準の3333を使用します。

OpenOCDは起動したままにして、別のPowerShellからGDBを実行します。

---

# GDBの起動と接続

GDBを起動する際には、デバッグ対象のELFファイルを指定します。

ELFファイルには、プログラムの実行コードに加えて、関数名や変数名などのシンボル情報や、ビルド時の設定によってはソースコードとの対応関係など、デバッグに必要な情報が含まれています。

本記事では、MSYS2のGDB Multiarchを使用します。導入方法については、[Try Kernelの開発環境構築記事](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)を参照してください。

また、Windows環境でのGDBの選定と日本語表示への対応については、[GDB Multiarchの導入記事](https://zenn.dev/helowota/articles/pico-gdb-multiarch-windows-env)で紹介しています。

別のPowerShellを開き、Try Kernelの `part_5` ディレクトリから次のコマンドでGDBを起動します。

```powershell
gdb-multiarch build/trykernel.elf
```

GDBが起動すると、ELFファイルのシンボル情報が読み込まれます。

```text
GNU gdb (GDB) 18.1
（途中省略）
Reading symbols from build/trykernel.elf...
(gdb)
```

GDBが起動したら、`target remote` コマンドを使用してOpenOCDのGDBサーバに接続します。

```text
(gdb) target remote localhost:3333
Remote debugging using localhost:3333
0x10000746 in init_section () at boot/reset_hdr.c:172
172             *top++ = 0;
```

`localhost:3333` は、同じPC上で動作しているOpenOCDのGDBサーバを指定しています。

接続後、RP2040をリセットして停止します。

```text
(gdb) monitor reset halt
[rp2040.core0] halted due to debug-request, current mode: Thread
xPSR: 0xf1000000 pc: 0x000000ea msp: 0x20041f00
[rp2040.core1] halted due to debug-request, current mode: Thread
xPSR: 0xf1000000 pc: 0x000000ea msp: 0x20041f00
```

`monitor` はGDBからOpenOCDのコマンドを実行するためのコマンドです。`reset halt` は、マイコンをリセットして実行を停止させます。

RP2040にはCortex-M0+が2コア搭載されており、両方のコアが停止したことを確認できます。

# 基本的なデバッグ操作

OpenOCDへの接続後は、GDBからプログラムの実行を制御したり、レジスタやメモリの内容を確認したりできます。

GUIデバッガでおなじみの操作を中心に、対応するGDBコマンドを一覧にまとめます。

| 操作 | GDBコマンド | 内容 |
|---|---|---|
| ブレークポイント設定 | `break main` | `main` 関数にブレークポイントを設定 |
| 実行再開 | `continue` | 停止しているプログラムの実行を再開 |
| 一時停止 | `Ctrl+C` | 実行中のプログラムを中断 |
| ステップオーバ | `next` | 次のソース行へ進む（関数内部には入らない） |
| ステップイン | `step` | 次のソース行へ進む（関数内部にも入る） |
| 変数の確認 | `print 変数名` | 変数の値を表示 |
| レジスタの確認 | `info registers` | CPUレジスタを表示 |
| メモリの確認 | `x/16wx アドレス` | 指定アドレスから32ビット値を16個表示 |
| リセット・停止 | `monitor reset halt` | OpenOCD経由でマイコンをリセットして停止 |
| GDBの終了 | `quit` | GDBを終了 |

以下では、実際にTry Kernelをデバッグした結果を紹介します。

## ブレークポイントを設定する

`main` 関数で停止させるため、`break main` でブレークポイントを設定します。

```text
(gdb) break main
Breakpoint 1 at 0x1000093c: file kernel/inittsk.c, line 31.
Note: automatically using hardware breakpoints for read-only addresses.
```

続いて、`continue` でプログラムの実行を再開します。

```text
(gdb) continue
Continuing.

Thread 1 "rp2040.core0" hit Breakpoint 1, main () at kernel/inittsk.c:31
31          tskid_ini = tk_cre_tsk(&ctsk_ini);              // 初期タスク生成
```

`kernel/inittsk.c` の31行目で停止しました。

また、日本語コメントの「初期タスク生成」も正常に表示されています。

### デバッグ情報について

GDBでソースコードを表示しながらデバッグするには、コンパイル時に `-g` オプションを指定して、ELFファイルにデバッグ情報を含めておく必要があります。

```makefile
CFLAGS += -g
```

デバッグ情報には、ソースコードの行番号や変数などに関する情報が含まれます。これにより、GDBでソースコードを追いながらステップ実行したり、変数の値を確認したりできます。

Try Kernelのビルド環境とMakefileの設定については、[「ラズパイPicoで1500行 ゼロから作るOS」を2026年に動かす](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)で紹介しています。

## ステップ実行する

停止した状態から、次のソース行へ進めるには `next` を使用します。これはGUIデバッガのステップオーバに相当します。

```gdb
next
```

関数の内部に入って処理を追いかけたい場合は、`step` を使用します。こちらはステップインに相当します。

```text
(gdb) step
tk_sta_tsk (tskid=0, stacd=125000000) at kernel/task_mange.c:49
49      {
```

今回の実行では、`tk_sta_tsk()` 関数の内部に入り、`kernel/task_mange.c` の49行目で停止しました。

## レジスタやメモリを確認する

CPUレジスタを表示するには、`info registers` コマンドを使用します。

```text
(gdb) info registers
r0             0x1                 1
r1             0x0                 0
r2             0x1                 1
r3             0x0                 0
r4             0x0                 0
r5             0x20041f01          537140993
r6             0x18000000          402653184
r7             0x2003ffc8          537133000
r8             0xffffffff          -1
r9             0xffffffff          -1
r10            0xffffffff          -1
r11            0xffffffff          -1
r12            0x3000              12288
sp             0x2003ffc8          0x2003ffc8
lr             0x1000094b          268437835
pc             0x10000c10          0x10000c10 <tk_sta_tsk>
xpsr           0x41000000          1090519040
msp            0x2003ffc8          0x2003ffc8
psp            0xfffffffc          0xfffffffc
primask        0x0                 0
basepri        0x0                 0
faultmask      0x0                 0
control        0x0                 0
```

例えば、`pc` はプログラムカウンタ、`sp` はスタックポインタ、`lr` はリンクレジスタです。

今回の `pc` には `tk_sta_tsk` と表示されており、現在の命令位置が関数名で示されています。

メモリの内容を確認するには、`x` コマンドを使用します。

```text
(gdb) x/16wx 0x20000000
0x20000000 <ctsk_ini>:  0x00000021      0x10000901      0x00000001      0x00000100
0x20000010 <ctsk_ini+16>:       0x20000064      0x00000008      0x00000000      0x00000000
0x20000020 <dev_tbl+12>:        0x00000001      0x00000001      0x00000321      0x100016a9
0x20000030 <dev_tbl+28>:        0x0000000a      0x00000400      0x20000ae0      0x00000321
```

この例では、SRAMの先頭アドレスから32ビット単位で16個の値を16進数表示しています。

`x/16wx` の意味は次のとおりです。

- `16`：表示する個数
- `w`：1個あたり4バイト（32ビット）
- `x`：16進数で表示

また、`ctsk_ini` や `dev_tbl` のように、対応するシンボル名も表示されています。


# OpenOCDとGDBの終了

GDBを終了するには、GDBのプロンプトから `quit` コマンドを実行します。

```text
(gdb) quit
A debugging session is active.

        Inferior 1 [Remote target] will be detached.

Quit anyway? (y or n) y
Detaching from program: C:/home/<ユーザ名>/repos/src/pico/interface_trykernel/part_5/build/trykernel.elf, Remote target
Ending remote debugging.
[Inferior 1 (Remote target) detached]
```

実行中のデバッグセッションがあるため、終了してよいか確認されます。ここでは `y` を入力して終了します。

OpenOCDはGDBとは別プロセスとして動作しているため、GDBを終了しても通常は起動したままです。

OpenOCDを終了するには、OpenOCDを実行しているPowerShellで `Ctrl+C` を入力します。

# おわりに

OpenOCDとGDBは、それぞれ独立したツールですが、組み合わせることでマイコンの実機デバッグを行えます。

OpenOCDがデバッグプローブを介したハードウェア制御を担当し、GDBがブレークポイントやステップ実行などのデバッグ操作を担当します。

今回はコマンドラインから操作しましたが、VSCodeやVim9のGUIを利用する場合も、基本的な仕組みは変わりません。

GUIを使用したデバッグについては、以下の記事で紹介しています。

- [VSCodeでCortex-Debugを使ってRaspberry Pi Picoをデバッグする](https://zenn.dev/helowota/articles/pico-vscode-cortex-debug)
- [Vim9のTermdebugでRaspberry Pi Picoをデバッグする](https://zenn.dev/helowota/articles/pico-vim9-termdebug)
