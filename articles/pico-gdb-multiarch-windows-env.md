---
title: "Windows 11でGDB環境を改善する ― 日本語表示・Python への対応"
emoji: "🔧"
type: "tech"
topics: ["gdb", "msys2", "raspberrypi", "pico", "windows"]
published: false
---

# はじめに

CQ出版社『Interface』2023年7月号の特集「ラズパイPicoで1500行 ゼロから作るOS」を読み進めながら、Raspberry Pi PicoでTry Kernelを動かしています。

これまでに、Windows 11上で、GNU Make、Arm GCC、OpenOCD、GDBによる開発環境を構築しました。

環境構築と基本的なデバッグ操作については、以下の記事にまとめています。

- [「ラズパイPicoで1500行 ゼロから作るOS」を2026年に動かす](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)
- [OpenOCDとGDBでRaspberry Pi Picoをデバッグする](https://zenn.dev/helowota/articles/pico-openocd-gdb-debug)

GDBによるデバッグは問題なく行えていましたが、環境を使い込むうちに、いくつか気になる点が出てきました。

- ソースコードに含まれる日本語コメントを正しく表示したい
- GDBからPythonを使いたい

これらを満たすGDBを探してみたところ、意外と簡単には解決しませんでした。

最終的には、**MSYS2のGDB Multiarchを採用することで、既存のビルド環境を変更せずに問題を解決できました。**

本記事では、GDBを比較した過程と、最終的に採用した環境の構築手順をまとめます。


# 使用環境

| 項目 | 内容 |
|---|---|
| OS | Windows 11 |
| シェル | PowerShell 7 |
| ターミナル | WezTerm |
| ターゲット | Raspberry Pi Pico W（RP2040） |
| コンパイラ | xPack Arm GNU Toolchain 14.2.1 |
| OpenOCD | xPack OpenOCD 0.12.0-7.1 |
| デバッグ接続 | CMSIS-DAP / SWD |
| デバッグ対象 | Try Kernel |

GDB以外の環境は、これまで構築したものをそのまま使用します。

# GDBの選定

## xPack Arm GNU ToolchainのGDB

最初に使用していたのは、xPack Arm GNU Toolchainに含まれるGDBです。

このツールチェーンには、次の2種類のGDBが含まれていました。

- `arm-none-eabi-gdb.exe`
- `arm-none-eabi-gdb-py3.exe`

前者は通常のGDB、後者はPython対応版です。

Pythonを使うため、`arm-none-eabi-gdb-py3.exe`を試しました。

Pythonは使用できましたが、日本語コメントの表示で問題が発生しました。

GDBで文字コードを確認すると、次の状態でした。

```text
(gdb) show host-charset
The host character set is "auto; currently CP1252".
```

UTF-8を指定しようとしても、このGDBではUTF-8を選択できませんでした。

日本語コメントを含むソースを表示しようとすると、次の警告が発生します。

```text
warning: could not convert 'main' from the host encoding (CP1252) to UTF-32
```

PowerShell側の出力文字コードをUTF-8に変更しても、この問題は解消しませんでした。

つまり、Pythonには対応しているものの、今回の環境では日本語ソースを快適に表示できません。

## MinGW-w64のGDB

次に試したのは、MinGW-w64環境のGDB 16.2です。

このGDBはUTF-8に対応しており、日本語コメントも表示できました。

しかし、別の問題が発生しました。

**GDBのコマンド入力でBackspaceが正常に動作しません。**

WezTermだけの問題かと思い、Windows Terminalや従来のコンソールでも試しましたが、同様でした。

GDBの設定を変更するとBackspaceは使えるようになりました。

```gdb
set editing off
```

ただし、この設定ではGDBのコマンドライン編集機能が無効になります。

Backspaceのために編集機能を犠牲にするのは避けたいところです。

また、今回使用したMinGW-w64のGDBはWindows向けの構成であり、Armターゲットのデバッグには適していませんでした。

そこで、別のGDBを探すことにしました。

## MSYS2のGDB Multiarch

最終的に採用したのが、MSYS2で提供されているGDB Multiarchです。

GDB Multiarchは、複数のCPUアーキテクチャを扱えるGDBです。

今回使用したパッケージは、次のものです。

```text
mingw-w64-ucrt-x86_64-gdb-multiarch
```

インストールされたGDBは18.1でした。

実際に試したところ、次の条件を満たしていました。

| GDB | Python | UTF-8日本語 | Backspace | Armデバッグ |
|---|---|---|---|---|
| xPack GDB 15.2.90 | × | 未確認 | ○ | ○ |
| xPack GDB py3 15.2.90 | ○ | × | ○ | ○ |
| MinGW-w64 GDB 16.2 | ○ | ○ | × | × |
| MSYS2 GDB Multiarch 18.1 | ○ | ○ | ○ | ○ |

※表は今回使用した環境での確認結果です。各GDBの一般的な対応状況を示すものではありません。

# MSYS2 GDB Multiarchの導入

## MSYS2のインストール

MSYS2をインストールします。

https://www.msys2.org/

今回は、既存のMinGW-w64環境とは別に、次のディレクトリへインストールしました。

```text
C:\msys64
```

既存のArm GNU ToolchainやOpenOCDは変更しません。

## GDB Multiarchのインストール

MSYS2 UCRT64のターミナルを起動し、次のコマンドを実行します。

```bash
pacman -S mingw-w64-ucrt-x86_64-gdb-multiarch
```

GDB本体と必要な依存パッケージがインストールされます。

今回の環境では、PythonやReadlineなどを含む31個のパッケージがインストールされました。

GDBの実行ファイルは次の場所に配置されます。

```text
C:\msys64\ucrt64\bin\gdb-multiarch.exe
```

WindowsのPATHには追加せず、PowerShellから実行ファイルを直接指定する構成にしました。

## PowerShellから起動する

PowerShell 7で、GDBを起動します。

```powershell
& "C:\msys64\ucrt64\bin\gdb-multiarch.exe" --version
```

今回の環境ではGDB 18.1が起動しました。

続いて、Try KernelのELFファイルを指定します。

```powershell
& "C:\msys64\ucrt64\bin\gdb-multiarch.exe" -q build/trykernel.elf
```

GDBからアーキテクチャを確認します。

```gdb
show architecture
```

結果は次のとおりでした。

```text
The target architecture is set to "auto" (currently "armv6s-m").
```

RP2040のCortex-M0+に対応するアーキテクチャが認識されています。

また、GDB内でPythonを実行できることも確認しました。

```gdb
python import sys; print(sys.version)
```

今回の環境ではPython 3.14.10が使用されました。

# 日本語コメントの文字化けを解消する

GDB MultiarchではUTF-8が使用できますが、最初から日本語が正しく表示されたわけではありません。

原因は、PowerShellのコンソール文字コード設定でした。

## PowerShellの文字コードを確認

PowerShell 7で次のコマンドを実行します。

```powershell
[Console]::OutputEncoding.CodePage
```

最初の環境では、次の結果でした。

```text
932
```

コードページ932は、日本語Windowsで使用されるCP932です。

一方、今回のソースコードはUTF-8で保存しています。

GDBの文字コードが自動設定になっている場合、PowerShell側の文字コードの影響を受けます。

そのため、GDBでソースコードを表示すると、日本語コメントが文字化けしました。

## UTF-8に変更する

PowerShellで次のコマンドを実行します。

```powershell
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
```

再度確認します。

```powershell
[Console]::OutputEncoding.CodePage
```

結果は次のとおりです。

```text
65001
```

65001はUTF-8を示します。

この状態でGDBを起動すると、次のようになりました。

```text
(gdb) show host-charset
The host character set is "auto; currently UTF-8".
```

ソースコードも正常に表示できました。

```text
(gdb) list main
26          usermain();
27          tk_ext_tsk();
28      }
29
30      int main(void)
31      {
32          tskid_ini = tk_cre_tsk(&ctsk_ini);              // 初期タスク生成
33          tk_sta_tsk(tskid_ini, 0);                       // 初期タスク実行
34
35           while(1);      // ここは実行されない
```

これで、日本語コメントの問題は解決しました。

# Windows OS ABIの警告を解消する

GDB MultiarchでELFファイルを読み込むと、次の警告が表示されました。

```text
warning: A handler for the OS ABI "Windows" is not built into this configuration
of GDB. Attempting to continue with the default armv6s-m settings.
```

実際のデバッグはできていましたが、毎回表示される警告なので原因を調べました。

## OS ABIの設定を確認

GDBで次のコマンドを実行します。

```gdb
show osabi
```

結果は次のとおりでした。

```text
The current OS ABI is "auto" (currently "Windows").
The default OS ABI is "Windows".
```

GDBのOS ABIがWindowsになっています。

しかし、今回のターゲットはRP2040上で動作するベアメタルのTry Kernelです。

Windows向けのOS ABIを使用する必要はありません。

## OS ABIを明示的に指定

GDBで次のコマンドを実行します。

```gdb
set osabi none
```

設定を確認します。

```text
(gdb) show osabi
The current OS ABI is "none".
The default OS ABI is "Windows".
```

これでOS ABIを`none`に変更できました。

ただし、GDBの起動後に変更しても、ELF読み込み時に発生した警告は消せません。

そこで、GDBの起動オプション`-iex`を使用します。

PowerShellから次のコマンドで起動します。

```powershell
& "C:\msys64\ucrt64\bin\gdb-multiarch.exe" -q -iex "set osabi none" build/trykernel.elf
```

`-iex`は、ELFファイルの読み込み前にGDBコマンドを実行するオプションです。

この方法で起動したところ、警告は表示されなくなりました。

```text
Reading symbols from build/trykernel.elf...
(gdb) show osabi
The current OS ABI is "none".
The default OS ABI is "Windows".
```

日本語コメントも引き続き正常に表示されました。

なお、これは今回のRP2040ベアメタル環境向けの設定です。異なるOSをデバッグする場合には、適切なOS ABIを選択する必要があります。

# PowerShellのプロファイルに設定を追加

毎回文字コードを変更したり、GDBの長いパスを指定したりするのは面倒です。

そこで、PowerShell 7のプロファイルに設定を追加しました。

## プロファイルを開く

PowerShell 7で次のコマンドを実行します。

```powershell
nvim $PROFILE
```

プロファイルの保存場所は環境によって異なります。

今回の環境では、OneDrive配下のPowerShell用ディレクトリに配置されていました。

既存の設定はそのまま残し、次の内容を追加します。

```powershell
# コンソールの文字コードをUTF-8に統一
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)

# MSYS2 GDB Multiarch
function gdb-multiarch {
    & "C:\msys64\ucrt64\bin\gdb-multiarch.exe" -iex "set osabi none" @args
}
```

PowerShellの関数を使用することで、WindowsのPATHを変更せずにGDB Multiarchを起動できます。

また、`@args`によって、関数に指定した引数がGDBへ渡されます。

## 設定を反映する

PowerShellで次のコマンドを実行します。

```powershell
. $PROFILE
```

文字コードを確認します。

```powershell
[Console]::OutputEncoding.CodePage
```

結果は`65001`です。

関数が登録されていることも確認します。

```powershell
Get-Command gdb-multiarch
```

次のように表示されました。

```text
CommandType     Name
-----------     ----
Function        gdb-multiarch
```

これで、通常のコマンドとしてGDBを起動できます。

```powershell
gdb-multiarch -q build/trykernel.elf
```

OS ABIの設定は関数内で行われるため、起動時の警告も表示されません。

※今回のPowerShellプロファイルには、起動時に作業ディレクトリを変更する設定も含まれています。プロファイルを再読み込みした場合は、ELFファイルのあるディレクトリに移動してからGDBを起動します。

# Pico実機での動作確認

最後に、OpenOCDを使用してPico実機に接続しました。

## OpenOCDの起動

PowerShell 7でOpenOCDを起動します。

```powershell
openocd -f interface/cmsis-dap.cfg -c "adapter speed 1000" -f target/rp2040.cfg
```

今回は、Pico WをDebug Probeとして使用し、別のPico WへSWD接続しています。

## GDBから接続

別のPowerShellを開き、Try Kernelのビルドディレクトリがある場所でGDBを起動します。

```powershell
gdb-multiarch -q build/trykernel.elf
```

GDBからOpenOCDへ接続します。

```gdb
target remote localhost:3333
```

ターゲットを停止し、プログラムを書き込みます。

```gdb
monitor reset halt
load
```

`main`関数にブレークポイントを設定します。

```gdb
break main
continue
```

ソースコードを確認し、ステップ実行します。

```gdb
list main
next
```

今回の環境では、OpenOCDとの接続、プログラムの書き込み、ブレークポイント、ステップ実行が正常に動作しました。

また、GDBのコマンド入力ではBackspaceも正常に使用できています。

# まとめ

今回、Windows 11で使用するGDBを見直しました。

最初はPython対応版のGDBを使えばよいと考えていましたが、日本語表示やキー操作など、実際に使ってみないと分からない問題がありました。

最終的に、MSYS2のGDB Multiarchを採用することで、次の環境を構築できました。

- Pythonを使用できる
- UTF-8の日本語コメントを正しく表示できる
- Backspaceを含むコマンドライン編集が正常に動作する
- RP2040をOpenOCD経由でデバッグできる
- 起動時のOS ABI警告を解消できる
- WindowsのPATHを変更せずに使用できる

既存のArm GNU ToolchainとOpenOCDはそのまま使用し、GDBだけを置き換えています。

GDBは普段何気なく使っているツールですが、配布元やビルド構成によって、対応機能や動作が異なることを改めて実感しました。

今回の環境を使って、引き続きTry Kernelの動作を追いかけていきます。
