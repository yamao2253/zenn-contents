---
title: "「ラズパイPicoで1500行 ゼロから作るOS」を2026年に動かす"
emoji: "🥧"
type: "tech"
topics: ["raspberrypi", "pico", "os", "gdb", "openocd"]
published: true
---

# 「ラズパイPicoで1500行 ゼロから作るOS」を2026年に動かす

2026/10/6

## Eclipseを使わず Make + OpenOCD + GDB で環境構築

## はじめに

> 本稿は『Interface 2023年7月号』の特集「ラズパイPicoで1500行 ゼロから作るOS」を、現在のWindows環境で試した際の個人的な検証記録です。書籍の内容を転載するものではなく、書籍掲載のソースコードをEclipseを使わずGNU Make、Arm GCC、OpenOCD、GDBでビルド・デバッグするために筆者が行った手順をまとめています。

### なぜこの記事を書いたか

『Interface 2023年7月号』では、Raspberry Pi Picoをターゲットとして、
OSをゼロから作成していく過程が解説されています。

この特集を読み進めようとしましたが、書籍で採用されているEclipseを使った
開発環境をうまく構築できず、私はそこで一度挫折しました。

OSをゼロから構築していくというテーマには大変興味が湧いたものの、
環境構築ができないため、書籍の内容を自分で動かしながら学ぶことができない状態でした。

そこで、普段使っているWindows環境で、Eclipseを使わず、
コマンドラインからビルド・デバッグできる環境を構築して、
書籍の内容を実際に動かすことを目指しました。

この記事は、その際に行った作業を記録したものです。

Eclipseを使った環境よりも、CLIベースの環境の方が構成がシンプルで、
環境構築のハードルも低いと感じました。

また、AIの支援を受けながら環境を構築したり、問題を切り分けたりする場合にも、
実行したコマンドとその結果が明確になるCLIベースの環境は相性がよいと感じています。
私自身も、実際にこの環境を構築するにあたって、かなりAIの支援を受けました。

私と同じように、書籍の内容には興味があるものの、
開発環境の構築で止まってしまった方の参考になればと思います。

### 書籍との違い

- 書籍掲載のソースコードやリンカスクリプトは、できるだけそのまま使用します。
- 書籍でEclipseに設定するコンパイル・リンク条件はMakefileに記述し、PowerShellから `make` してビルドします。
- デバッグ環境についても、現在入手できるツールを使用します。

つまり、Try Kernelそのものを変更するのではなく、**その周囲の開発環境を現在の環境に置き換える**ことを基本方針とします。

### この記事でできるようになること

この記事では、Windows上にEclipseを使わないTry Kernelの開発環境を構築します。

具体的には、次のことができるようになります。

- GNU MakeとArm GCCを使ってTry Kernelをビルドする
- 生成したELFファイルのメモリ配置を確認する
- Pico WをDebug Probeとして使用し、SWDでターゲットのRP2040へ接続する
- OpenOCDとGDBを使ってTry KernelをPico Wへ書き込む

最終的には、次のような開発環境を構築します。

![開発環境の構成](/images/pico-trykernel-windows-env/blockdiagram.png)

---

## 1. 開発環境の構成と準備

### 1.1 構築する開発環境

書籍のAppendixの環境（Eclipse中心）を、Windows上のコマンドライン環境に置き換えて構築します。

最大のポイントは、**Eclipseを使わずにPowerShellから `make` でビルド**を行い、
OpenOCDとGDBを使って実機のRaspberry Pi Pico（RP2040）に書き込んで動作確認することです。

- **Windows上で完結**
  - Eclipseは使用せず、PowerShellからGNU Makeでビルドします。
- **書籍と同じソースコード**
  - 著者のGitHubリポジトリ `interface_trykernel` を使用します。
- **同等のビルド設定**
  - 書籍のEclipse設定と同等のコンパイル・リンクオプションをMakefileに記述します。
- **Arm GCCツールチェーン**
  - xPackのGNU Arm Embedded GCCを使用します（`arm-none-eabi-gcc` / `arm-none-eabi-gdb` など）。
- **OpenOCD + CMSIS-DAP**
  - xPackのOpenOCDと、Pico W（Debug Probe）のCMSIS-DAP機能を使用します。
- **実機デバッグ／書き込み**
  - GDBからOpenOCDに接続し、RP2040搭載のPico W（ターゲット）にプログラムを書き込み、動作を確認します。

### 1.2 用意するもの

#### ハードウェア

- Raspberry Pi Picoシリーズ
  - Try Kernelを動かすターゲットとして使用します
  - 私はPico Wを使用しました
- Debug Probe
  - 市販のRaspberry Pi Debug Probeを使用できます
  - 別のPicoシリーズをDebug Probeとして使用することもできます
- ブレッドボード
- ジャンパワイヤ
- USBケーブル

#### ソフトウェア

##### あらかじめ用意するもの

- Windows 11
- PowerShell
- Git
- 任意のテキストエディタ

##### 環境構築の中で準備するもの

以下のソフトウェアやファームウェアは、本記事の環境構築手順の中で
必要に応じてダウンロード・インストールします。

- Node.js（xpmのインストールに使用）
- xpm
- xPack GNU Arm Embedded GCC
- xPack Windows Build Tools（GNU Make）
- xPack OpenOCD
- Raspberry Pi Debug Probe firmware

### 1.3 書籍のソースコードを取得する

書籍で使用するソースコードは、著者のGitHubリポジトリで公開されています。

本記事では、このリポジトリをcloneして使用します。

#### ソースコードをcloneする

PowerShellを開き、作業用のフォルダへ移動して、次のコマンドを実行します。

```powershell
git clone https://github.com/ytoyoyama/interface_trykernel.git
cd interface_trykernel
```

cloneすると、書籍の各章・節に対応したソースコードが取得できます。

#### 最初に使用するソースコード

まずは、次のフォルダを使用します。

```text
part_2/sect_3
```

本記事でも、最初のビルド対象としてこのフォルダを使用します。

```powershell
cd part_2\sect_3
```

#### ソースコードの構成

`part_2/sect_3` は次のような構成になっています。

```text
part_2/sect_3/
├─ application/
│  └─ main.c
├─ boot/
│  ├─ boot2.c
│  ├─ reset_hdr.c
│  └─ vector_tbl.c
├─ include/
│  ├─ knldef.h
│  ├─ sysdef.h
│  ├─ syslib.h
│  └─ typedef.h
└─ linker/
   └─ pico_memmap.ld
```

ここにはCのソースコードだけでなく、Raspberry Pi Picoの起動処理や
メモリ配置を定義するリンカスクリプトも含まれています。

なお、リポジトリにはMakefileは含まれていません。

書籍ではEclipseのプロジェクト設定としてコンパイルやリンクの条件を設定しますが、
本記事では、それらの設定をMakefileとして記述します。

Makefileについては、次の章で作成します。

---

## 2. Try Kernelをビルドする

### 2.1 GNU Arm Embedded GCCを準備する

Try Kernelをビルドするため、Arm Cortex-M向けのGNUツールチェーンを準備します。

本記事では、xPack GNU Arm Embedded GCCを使用します。

このツールチェーンには、次のようなコマンドが含まれています。

- `arm-none-eabi-gcc`：Cソースコードのコンパイルとリンク
- `arm-none-eabi-gdb`：実機デバッグ
- `arm-none-eabi-objcopy`：ELFファイルの形式変換など
- `arm-none-eabi-size`：ELFファイルのセクションサイズや配置の確認

#### xpmを準備する

xPack GNU Arm Embedded GCCのインストールには、
xPack Project Manager（xpm）を使用します。

まず、xpmがインストールされているか確認します。

```powershell
xpm --version
```

バージョンが表示されれば、この節のインストール作業は不要です。

##### Node.jsをインストールする

xpmはNode.js上で動作するため、Node.jsがインストールされていない場合は
先にNode.jsをインストールします。

次のコマンドで確認できます。

```powershell
node --version
npm --version
```

インストールされていない場合は、Node.jsの公式サイトからWindows用の
LTS版をダウンロードしてインストールします。

Node.jsをインストールすると、npmも一緒にインストールされます。

本記事ではNode.jsそのものを開発に使用するわけではなく、
xpmをインストールするためにnpmを使用します。

##### xpmをインストールする

Node.jsのインストール後、PowerShellを開き直して次のコマンドを実行します。

```powershell
npm install --location=global xpm@latest
```

インストールできたことを確認します。

```powershell
xpm --version
```

私が環境を構築した際には、次のバージョンを使用しました。

```text
Node.js : 22.16.0
npm     : 11.4.2
xpm     : 0.20.8
```

#### GNU Arm Embedded GCCをインストールする

xpmを使用して、xPack GNU Arm Embedded GCCをインストールします。

本記事では、動作確認を行ったバージョンを指定してインストールします。

```powershell
xpm install --global @xpack-dev-tools/arm-none-eabi-gcc@14.2.1-1.1.1
```

インストールしたxPackは、次のコマンドで確認できます。

```powershell
xpm list --global
```

私の環境では、次のようにインストールされています。

```text
- @xpack-dev-tools/arm-none-eabi-gcc
  - 14.2.1-1.1.1
```

#### PATHを設定する

xpmでglobal installしたGNU Arm Embedded GCCは、私の環境では次のフォルダに
インストールされました。

```text
C:\Users\<ユーザ名>\AppData\Roaming\xPacks\@xpack-dev-tools\arm-none-eabi-gcc\14.2.1-1.1.1\.content\bin
```

`<ユーザ名>` の部分は、使用しているWindowsのユーザ名に読み替えてください。

このフォルダをWindowsのユーザ環境変数 `Path` に追加します。

1. Windowsの「環境変数を編集」を開く
2. ユーザ環境変数の `Path` を選択して「編集」を開く
3. 「新規」を選択し、上記のフォルダを追加する
4. 設定後、PowerShellを開き直す

これでPowerShellから `arm-none-eabi-gcc` などを直接実行できるようになります。

#### インストールを確認する

PowerShellで次のコマンドを実行します。

```powershell
arm-none-eabi-gcc --version
arm-none-eabi-gdb --version
arm-none-eabi-objcopy --version
arm-none-eabi-size --version
```

私が環境を構築した際には、次のバージョンを使用しました。

```text
GNU Arm Embedded GCC : 14.2.1
GDB                  : 15.2.90
objcopy              : 2.43.1
```

`arm-none-eabi-objcopy` と `arm-none-eabi-size` はGNU Binutilsに含まれています。

バージョン番号が多少異なる環境でも動作する可能性はありますが、
本記事では上記のバージョンでビルドとデバッグができることを確認しています。

### 2.2 GNU Makeを準備する

Try KernelのビルドにはGNU Makeを使用します。

本記事では、xPack Windows Build Toolsに含まれるGNU Makeを使用します。

#### xPack Windows Build Toolsをインストールする

xpmを使用して、xPack Windows Build Toolsをインストールします。

本記事では、動作確認を行ったバージョンを指定してインストールします。

```powershell
xpm install --global @xpack-dev-tools/windows-build-tools@4.4.1-3.1
```

インストールしたxPackは、次のコマンドで確認できます。

```powershell
xpm list --global
```

私の環境では、次のようにインストールされています。

```text
- @xpack-dev-tools/windows-build-tools
  - 4.4.1-3.1
```

#### PATHを設定する

xpmでglobal installしたxPack Windows Build Toolsは、私の環境では
次のフォルダにインストールされました。

```text
C:\Users\<ユーザ名>\AppData\Roaming\xPacks\@xpack-dev-tools\windows-build-tools\4.4.1-3.1\.content\bin
```

`<ユーザ名>` の部分は、使用しているWindowsのユーザ名に読み替えてください。

このフォルダをWindowsのユーザ環境変数 `Path` に追加します。

設定方法は、GNU Arm Embedded GCCで行ったPATHの設定と同じです。

設定後、PowerShellを開き直します。

#### インストールを確認する

PowerShellで次のコマンドを実行します。

```powershell
make --version
```

私が環境を構築した際には、次のバージョンを使用しました。

```text
GNU Make 4.4.1
```

バージョンが表示されれば、GNU Makeの準備は完了です。

### 2.3 Makefileを作成する

ここまでで、CコンパイラとGNU MakeをPowerShellから実行できるようになりました。

次に、書籍ではEclipseのプロジェクトに設定しているコンパイル・リンク条件を、
GNU Makeから使用できるようにMakefileへ記述します。

本記事では書籍のソースコードやリンカスクリプトには手を加えず、
ビルド方法だけをEclipseからGNU Makeへ置き換えます。

#### 書籍の設定をMakefileに置き換える

書籍で設定している主なコンパイル・リンク条件と、
GNU GCCのオプションとの対応は次のようになります。

| 書籍の設定 | GCCのオプション |
|---|---|
| Target Processor：cortex-m0plus | `-mcpu=cortex-m0plus` |
| Optimization：None (-O0) | `-O0` |
| Assume freestanding environment | `-ffreestanding` |
| Language standard：ISO C99 | `-std=c99` |
| Include paths：include | `-Iinclude` |
| Linker script：linker\pico_memmap.ld | `-T linker/pico_memmap.ld` |
| Do not use standard start files | `-nostartfiles` |

本記事では、これらに加えて次の2つのオプションを使用します。

- `-mthumb`：Cortex-M0+向けにThumb命令セットを使用することを明示します
- `-g`：後でGDBを使ってデバッグするため、ELFファイルにデバッグ情報を付加します

この2つは書籍のEclipse設定から置き換えたものではなく、
本記事のCLI環境で追加した設定です。

#### Makefileを作成する

`part_2/sect_3` フォルダに、次の内容で `Makefile` を作成します。

```makefile
TARGET = trykernel

CC      = arm-none-eabi-gcc
OBJCOPY = arm-none-eabi-objcopy

CFLAGS  = -mcpu=cortex-m0plus
CFLAGS += -mthumb
CFLAGS += -O0
CFLAGS += -ffreestanding
CFLAGS += -std=c99
CFLAGS += -Iinclude
CFLAGS += -g

LDFLAGS  = -mcpu=cortex-m0plus
LDFLAGS += -mthumb
LDFLAGS += -T linker/pico_memmap.ld
LDFLAGS += -nostartfiles

SRCS = \
	application/main.c \
	boot/boot2.c \
	boot/reset_hdr.c \
	boot/vector_tbl.c

OBJS = $(SRCS:.c=.o)

all: $(TARGET).elf

$(TARGET).elf: $(OBJS)
	$(CC) $(LDFLAGS) -o $@ $(OBJS)

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET).elf

.PHONY: all clean
```

このMakefileでは、4つのCソースコードをそれぞれコンパイルしてオブジェクトファイルを作成し、
最後にリンカスクリプト `linker/pico_memmap.ld` を使用して
`trykernel.elf` を生成します。

なお、`OBJCOPY` はこの時点のMakefileでは使用していませんが、
書籍の開発環境との対応を分かりやすくするため定義しています。

### 2.4 Try Kernelをビルドする

Makefileを作成したら、Try Kernelをビルドします。

PowerShellで `part_2/sect_3` フォルダに移動し、次のコマンドを実行します。

```powershell
make
```

正常にビルドできると、次のようにコンパイルとリンクが実行されます。

```text
arm-none-eabi-gcc -mcpu=cortex-m0plus -mthumb -O0 -ffreestanding -std=c99 -Iinclude -g -c application/main.c -o application/main.o
arm-none-eabi-gcc -mcpu=cortex-m0plus -mthumb -O0 -ffreestanding -std=c99 -Iinclude -g -c boot/boot2.c -o boot/boot2.o
arm-none-eabi-gcc -mcpu=cortex-m0plus -mthumb -O0 -ffreestanding -std=c99 -Iinclude -g -c boot/reset_hdr.c -o boot/reset_hdr.o
arm-none-eabi-gcc -mcpu=cortex-m0plus -mthumb -O0 -ffreestanding -std=c99 -Iinclude -g -c boot/vector_tbl.c -o boot/vector_tbl.o
arm-none-eabi-gcc -mcpu=cortex-m0plus -mthumb -T linker/pico_memmap.ld -nostartfiles -o trykernel.elf application/main.o boot/boot2.o boot/reset_hdr.o boot/vector_tbl.o
```

エラーが発生せず、`trykernel.elf` が生成されればビルドは成功です。

生成物を削除して最初からビルドし直す場合は、次のように実行します。

```powershell
make clean
make
```

### 2.5 ELFのメモリ配置を確認する

ビルドで生成された `trykernel.elf` のメモリ配置を確認します。

PowerShellで次のコマンドを実行します。

```powershell
arm-none-eabi-size -A trykernel.elf
```

私の環境では、次のような結果になりました（主要部分を抜粋）。

```text
section            size        addr
boot2               256   268435456
.text              2788   268435712
.ARM.exidx            8   268438500
.data                 0   536870912
.bss                  0   536870912
```

`size` はバイト単位、`addr` は10進数で表示されています。

16進数に読み替えると、主なメモリ配置は次のようになります。

| セクション | 開始アドレス | サイズ |
|---|---:|---:|
| `boot2` | `0x10000000` | 256バイト |
| `.text` | `0x10000100` | 2788バイト |
| `.data` | `0x20000000` | 0バイト |
| `.bss` | `0x20000000` | 0バイト |

ここでは、次の2点を確認します。

- `boot2` が `0x10000000` から256バイト配置されていること
- `.text` が、その直後の `0x10000100` から配置されていること

これは、Makefileで指定したリンカスクリプト `linker/pico_memmap.ld` に従った配置です。

このメモリ配置は、今後Try Kernelの起動処理を理解する際にも重要になります。

ここまで確認できれば、ビルド環境の準備は完了です。

---

## 3. Debug ProbeとOpenOCDを準備する

### 3.1 OpenOCDをインストールする

Raspberry Pi Picoへのプログラムの書き込みやデバッグには、OpenOCDを使用します。

OpenOCDは、GDBとDebug Probeの間に入り、
ターゲットとなるRP2040への接続を行うソフトウェアです。

本記事では、xPack OpenOCDを使用します。

#### xPack OpenOCDをインストールする

第2章で準備したxpmを使用して、xPack OpenOCDをインストールします。

本記事では、動作確認を行ったバージョンを指定してインストールします。

```powershell
xpm install --global @xpack-dev-tools/openocd@0.12.0-7.1
```

インストールしたxPackは、次のコマンドで確認できます。

```powershell
xpm list --global
```

私の環境では、次のようにインストールされています（該当部分を抜粋）。

```text
- @xpack-dev-tools/openocd
  - 0.12.0-7.1
```

#### PATHを設定する

xpmでglobal installしたxPack OpenOCDは、私の環境では次のフォルダに
インストールされました。

```text
C:\Users\<ユーザ名>\AppData\Roaming\xPacks\@xpack-dev-tools\openocd\0.12.0-7.1\.content\bin
```

`<ユーザ名>` の部分は、使用しているWindowsのユーザ名に読み替えてください。

このフォルダをWindowsのユーザ環境変数 `Path` に追加します。

設定方法は、GNU Arm Embedded GCCで行ったPATHの設定と同じです。

設定後、PowerShellを開き直します。

#### インストールを確認する

PowerShellで次のコマンドを実行します。

```powershell
openocd --version
```

私の環境では、次のバージョンが表示されました。

```text
xPack Open On-Chip Debugger 0.12.0+dev-02228-ge5888bda3-dirty (2025-10-04-22:44)
```

バージョンが表示されれば、OpenOCDの準備は完了です。

### 3.2 Pico WをDebug Probeにする

本記事では、Raspberry Pi Pico WをDebug Probeとして使用します。

Debug Probeは、PC上のOpenOCDとターゲットとなるRP2040の間に入り、
SWDによるデバッグ通信を行うためのものです。

市販のRaspberry Pi Debug Probeを使用することもできますが、
今回は手持ちのPico Wを使用しました。

#### Debug Probeのファームウェアを取得する

Raspberry Pi公式のDebug Probe用ファームウェアをダウンロードします。

ダウンロード先：

https://github.com/raspberrypi/debugprobe/releases

今回使用するファイルは次のものです。

```text
debugprobe_on_pico.uf2
```

これは、PicoシリーズをDebug Probeとして動作させるためのファームウェアです。

#### Pico Wにファームウェアを書き込む

Debug Probeとして使用するPico Wを、BOOTSELモードでPCに接続します。

1. Pico WのUSBケーブルを抜いておきます。
2. Pico WのBOOTSELボタンを押したまま、USBケーブルをPCに接続します。
3. Windowsのエクスプローラに `RPI-RP2` というドライブが表示されることを確認します。
4. ダウンロードした `debugprobe_on_pico.uf2` を `RPI-RP2` にコピーします。

コピーが完了すると、`RPI-RP2` ドライブは自動的に消えます。

これは、Pico Wが再起動してDebug Probe用ファームウェアを実行するためで、
正常な動作です。

#### Windowsで認識を確認する

ファームウェアの書き込み後、Windowsのデバイスマネージャを開きます。

次のデバイスが表示されていることを確認します。

```text
CMSIS-DAP v2 Interface
```

私の環境では、PowerShellからも次のコマンドで確認できました。

```powershell
Get-PnpDevice | Where-Object FriendlyName -Like "*CMSIS-DAP*"
```

実行結果：

```text
Status  Class      FriendlyName
------  -----      ------------
OK      USBDevice  CMSIS-DAP v2 Interface
```

`CMSIS-DAP v2 Interface` が正常に認識されていれば、
Debug Probe側の準備は完了です。

### 3.3 ターゲットPico WとSWD接続する

Debug Probe用Pico Wの準備ができたら、
Try Kernelを動かすターゲット用Pico Wと接続します。

今回は、2台のPico Wをブレッドボードに取り付け、
ジャンパワイヤで接続しました。

#### SWDの配線

2台のPico Wを、次のように接続します。

| Debug Probe用Pico W | ターゲット用Pico W | 用途 |
|---|---|---|
| GP2（4番ピン） | SWCLK | SWDクロック |
| GP3（5番ピン） | SWDIO | SWDデータ |
| GND（3番ピン） | GND | グランド |
| VBUS（40番ピン） | VSYS（39番ピン） | 電源供給 |

接続の概要は次のとおりです。

```text
Debug Probe Pico W          Target Pico W
       GP2  --------------> SWCLK
       GP3  --------------> SWDIO
       GND  --------------> GND
      VBUS  --------------> VSYS
```

#### ターゲットへの電源供給

今回の構成では、Debug Probe用Pico WをUSBケーブルでPCに接続し、
そのVBUSからターゲット用Pico WのVSYSへ電源を供給しています。

そのため、ターゲット用Pico WにはUSBケーブルを接続していません。

**この配線のまま、ターゲット側にも別のUSB電源を接続しないでください。**
電源同士が接続される可能性があります。

なお、市販のDebug Probeを使用する場合などは、
ターゲットへの電源供給方法が異なります。
使用する機器に合わせて配線してください。

#### 実際の接続例

私は、次のようにブレッドボードを使用して接続しました。

（ここに実際の接続写真を掲載）

配線が完了したら、Debug Probe用Pico WをUSBケーブルでPCに接続します。

### 3.4 OpenOCDからRP2040を認識する

Debug Probeとターゲット用Pico Wの接続が完了したら、
OpenOCDを起動してRP2040を認識できることを確認します。

#### OpenOCDを起動する

PowerShellを開き、次のコマンドを実行します。

```powershell
openocd -f interface/cmsis-dap.cfg -f target/rp2040.cfg
```

ここでは、次の2つの設定ファイルを指定しています。

- `interface/cmsis-dap.cfg`：CMSIS-DAP対応のDebug Probeを使用する設定
- `target/rp2040.cfg`：ターゲットをRP2040とする設定

#### RP2040の認識を確認する

正常に接続できると、次のようなログが表示されます（主要部分を抜粋）。

```text
Info : Using CMSIS-DAPv2 interface with VID:PID=0x2e8a:0x000c
Info : CMSIS-DAP: SWD supported
Info : CMSIS-DAP: Interface ready
Info : SWD DPIDR 0x0bc12477
Info : [rp2040.core0] Cortex-M0+ r0p1 processor detected
Info : [rp2040.core0] Examination succeed
Info : [rp2040.core1] Cortex-M0+ r0p1 processor detected
Info : [rp2040.core1] Examination succeed
Info : [rp2040.core0] starting gdb server on 3333
Info : Listening on port 3333 for gdb connections
```

ここでは、次の3点を確認します。

1. `CMSIS-DAP: Interface ready` が表示されていること
2. `rp2040.core0` と `rp2040.core1` の両方が認識されていること
3. `Listening on port 3333 for gdb connections` が表示されていること

RP2040はデュアルコアのマイコンなので、`core0` と `core1` の2つが表示されます。

最後のメッセージは、GDBからの接続を待ち受けていることを示しています。

#### OpenOCDを起動したままにする

OpenOCDは、起動後も終了せずに動作し続けます。
これは正常な状態です。

**このPowerShellは閉じずに、そのままにしておいてください。**

---

## 4. GDBからTry Kernelを書き込む

### 4.1 GDBを起動してOpenOCDへ接続する

OpenOCDを起動したまま、別のPowerShellを開きます。

`trykernel.elf` がある `part_2/sect_3` フォルダへ移動し、
GDBを起動します。

```powershell
arm-none-eabi-gdb trykernel.elf
```

正常に起動すると、`trykernel.elf` のデバッグ情報が読み込まれ、
GDBのプロンプトが表示されます。

```text
Reading symbols from trykernel.elf...
(gdb)
```

GDBから、OpenOCDが待ち受けているポート3333へ接続します。

```text
(gdb) target remote localhost:3333
```

接続できたら、ターゲットをリセットして停止させます。

```text
(gdb) monitor reset halt
```

### 4.2 Try Kernelを書き込む

`trykernel.elf` をターゲットのフラッシュメモリへ書き込みます。

```text
(gdb) load
```

私の環境では、次のように表示されました。

```text
Loading section boot2, size 0x100 lma 0x10000000
Loading section .text, size 0xae4 lma 0x10000100
Loading section .ARM.exidx, size 0x8 lma 0x10000be4
Start address 0x10000854, load size 3052
Transfer rate: 623 bytes/sec, 1017 bytes/write.
```

エラーが発生せず、各セクションの書き込み結果が表示されれば書き込みは完了です。

ここで表示される `boot2` の `0x10000000` や `.text` の `0x10000100` は、
第2章で確認したELFのメモリ配置とも対応しています。

### 4.3 動作確認

ここまでの手順で、Try Kernelをビルドし、
Debug Probeを経由してターゲットのPico Wへ書き込むところまで確認できました。

今回の環境で確認できた内容は次のとおりです。

- OpenOCDからRP2040のCortex-M0+ `core0` / `core1` を認識できる
- GDBからOpenOCDへ接続できる
- GDBからターゲットのreset / haltを実行できる
- `trykernel.elf` の `boot2` / `.text` などをフラッシュメモリへ書き込める

OpenOCDでは、次のようにRP2040の2つのコアが認識されました。

```text
Info : [rp2040.core0] Cortex-M0+ r0p1 processor detected
Info : [rp2040.core0] Examination succeed
Info : [rp2040.core1] Cortex-M0+ r0p1 processor detected
Info : [rp2040.core1] Examination succeed
```

また、GDBから `load` を実行することで、
ELFの各セクションを書き込むことができました。

```text
Loading section boot2, size 0x100 lma 0x10000000
Loading section .text, size 0xae4 lma 0x10000100
Loading section .ARM.exidx, size 0x8 lma 0x10000be4
Start address 0x10000854, load size 3052
```

これで、Eclipseを使用せず、
コマンドラインからTry Kernelをビルド・書き込み・デバッグするための
基本的な環境が構築できました。

---

## 書籍Appendixとの違い

今回構築した環境は、Try Kernelのソースコードやリンカスクリプトは
できるだけそのまま使用し、開発環境を現在利用できるツールへ置き換えています。

主な違いは次のとおりです。

| 項目 | 書籍 | 本記事 |
|---|---|---|
| 開発環境 | Eclipse | PowerShell + GNU Make |
| ビルド | Eclipseのプロジェクト設定 | Makefile |
| コンパイラ | GNU Arm Embedded GCC | xPack GNU Arm Embedded GCC |
| デバッグ | OpenOCD + Picoprobe | OpenOCD + CMSIS-DAP |
| Debug Probe | Picoprobe | 現行のDebug Probe firmwareを書き込んだPico W |
| ターゲット | Raspberry Pi Pico | Raspberry Pi Pico W |

- 本記事ではVS CodeなどのIDE・エディタとの連携は行っていません。
- まずはPowerShellからビルド、書き込み、デバッグできる最小限の環境を構築し、VS Codeとの連携などは必要になった段階で追加していく方針としました。
- この構成にすることで、書籍のソースコードそのものにはできるだけ手を加えず、現在のWindows環境でもTry Kernelの学習を進められるようにしています。

---

## おわりに

今回の環境構築で、Eclipseを使わずにTry Kernelをビルドし、
Pico Wへ書き込み、GDBからデバッグするための環境を用意することができました。

しかし、私が本来やりたかったのは環境を構築することではありません。

『ラズパイPicoで1500行 ゼロから作るOS』を実際に動かしながら、
OSがどのように作られているのかを理解することです。

ここまでで、ようやくそのスタート地点に立つことができました。

今後はこの環境を使って書籍を読み進めながら、
タスク管理、スケジューリング、割り込み、システムコールなどが
どのように実装されていくのかを、実際のソースコードと動作を見ながら
理解していきたいと思います。

また、今回少しだけ確認したELFのメモリ配置やリンカスクリプトについても、
自作OSを理解するうえで面白いテーマだと感じました。
このあたりは、環境構築とは別に掘り下げてみたいと思っています。

この記事が、私と同じように書籍の内容には興味を持ったものの、
開発環境の構築で止まってしまった方にとって、
Try Kernelを始めるきっかけになれば幸いです。

