---
title: Windows 11でGDBの日本語文字化けを解消する ― MSYS2 GDB MultiarchでRaspberry Pi Picoをデバッグ
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

これらの条件を満たすGDBを探すのは、思ったほど簡単ではありませんでした。

いくつか試した結果、**MSYS2のGDB Multiarchを採用することで、既存のビルド環境を変更せずに問題を解決できました。**

本記事では、GDBの比較・選定から、導入手順までを紹介します。


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

以前は、[環境構築の記事](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)で導入したxPack Arm GNU Toolchainに含まれる`arm-none-eabi-gdb.exe`を使用していました。

このGDBでは、`list <関数名>`でソースコードを表示すると、UTF-8で記述された日本語コメントが文字化けする問題がありました。

```text
(gdb) list main
warning: could not convert 'main' from the host encoding (CP1252) to UTF-32.
This normally should not happen, please file a bug report.
26          usermain();
27          tk_ext_tsk();
28      }
29
30      int main(void)
31      {
32          tskid_ini = tk_cre_tsk(&ctsk_ini);              // 蛻晄悄繧ｿ繧ｹ繧ｯ逕滓・
34          tk_sta_tsk(tskid_ini, 0);                       // 蛻晄悄繧ｿ繧ｹ繧ｯ螳溯｡・
```

同じツールチェーンにはPython対応版の`arm-none-eabi-gdb-py3.exe`も含まれています。GDBからPythonを利用したかったため、こちらも試してみました。

ところが、`list main`を実行すると、同じ警告が発生するだけでなく、ソースコードが表示されずにGDBが終了してしまいました。

```text
(gdb) list main
warning: could not convert 'main' from the host encoding (CP1252) to UTF-32.
This normally should not happen, please file a bug report.
```

文字コードを確認するとCP1252になっており、`arm-none-eabi-gdb.exe`と`arm-none-eabi-gdb-py3.exe`のどちらもUTF-8を指定できませんでした。

```text
(gdb) show host-charset
The host character set is "auto; currently CP1252".
(gdb) set host-charset UTF-8
Undefined item: "UTF-8".
(gdb) set host-charset
CP1252  auto
```

PowerShellのコンソール出力文字コードの設定値（`OutputEncoding`）も、次のコマンドで一時的にUTF-8へ変更してみましたが、GDBの`host-charset`はCP1252のままで、問題は解消しませんでした。

```powershell
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
```

日本語コメントを正しく表示でき、Pythonも利用できる他のGDBを探すことにしました。

## GDBの比較

Windows向けに公開されているGDBのうち、ソースコードをビルドせずに導入できる次の3種類を試しました。

- xPack Arm GNU ToolchainのGDB
- MinGW-w64のGDB
- MSYS2のGDB Multiarch

比較結果は次のとおりです。

| GDB | バージョン | 日本語表示 | Python | Arm対応 |
|---  |---         |:----------:|:------:|:-------:|
| xPack GDB（無印） | 15.2.90 | × | × | ○ |
| xPack GDB（py3） | 15.2.90 | × | ○ | ○ |
| MinGW-w64 GDB | 16.2 | ○ | ○ | × |
| MSYS2 GDB Multiarch | 18.1 | ○ | ○ | ○ |

※今回のWindows 11環境で確認した結果です。

MinGW-w64のGDBでは日本語表示とPython対応を確認できましたが、今回使用したものはArmターゲットに対応していませんでした。また、コマンド入力でBackspaceが正常に動作しない問題もありました。

一方、MSYS2のGDB Multiarchでは必要な条件をすべて満たしていたため、こちらを採用しました。

## MSYS2のGDB Multiarch

GDB Multiarchは、複数のCPUアーキテクチャを扱えるGDBです。

今回使用したMSYS2のパッケージは、次のものです。

```text
mingw-w64-ucrt-x86_64-gdb-multiarch
```

インストールされたGDBのバージョンは18.1でした。実行結果は次のとおりです。
```powershell
PS> C:\msys64\ucrt64\bin\gdb-multiarch.exe --version
GNU gdb (GDB) 18.1
Copyright (C) 2026 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```

---

# MSYS2 GDB Multiarchの導入

## MSYS2のインストール

MSYS2の[公式サイト](https://www.msys2.org/)からWindows用インストーラ（`.exe`）をダウンロードし、案内に従ってインストールします。

インストーラにはx64版とARM64版があります。今回はx64版を使用し、公式サイトの手順に従って、デフォルト設定のままインストールしました。

今回は、インストール先を含め、すべてデフォルトの設定を使用しました。

インストール先は次のとおりです。 (公式推奨)

```text
C:\msys64
```

[以前の記事で構築したArm GNU ToolchainとOpenOCDの環境](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)は、そのまま使用します。

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

GDBはPowerShellの関数から起動することにしたので、WindowsのPATHには追加しません。関数の設定方法は後述します。

## GDBの動作確認

インストールしたGDB Multiarchで、Armアーキテクチャの認識とPythonの動作を確認します。

まず、PowerShell 7からTry KernelのELFファイルを指定してGDBを起動します。

```powershell
C:\msys64\ucrt64\bin\gdb-multiarch.exe build/trykernel.elf
```

起動時には、次のメッセージが表示されました。

```text
GNU gdb (GDB) 18.1
Copyright (C) 2026 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-w64-mingw32".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.

┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Find the GDB manual online at:                                               ┃
┃ http://www.gnu.org/software/gdb/documentation/.                              ┃
┃ For help, type "help".                                                       ┃
┃ Type "apropos word" to search for commands related to "word".                ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛


⚠️ warning: A handler for the OS ABI "Windows" is not built into this configuration
of GDB.  Attempting to continue with the default armv6s-m settings.

Reading symbols from build/trykernel.elf...
(gdb)
```

OS ABIに関する警告が表示されますが、GDBは起動し、ELFファイルのシンボルも読み込まれています。この警告への対処方法は後述します。

続いて、`show architecture`コマンドで、GDBが認識しているターゲットのアーキテクチャを確認します。

```gdb
show architecture
```

実行結果：

```text
The target architecture is set to "auto" (currently "armv6s-m").
```

アーキテクチャの設定は`auto`となっており、ELFファイルからRP2040のCortex-M0+に対応する`armv6s-m`が自動認識されています。

続いて、GDB内でPythonを実行できることを確認します。

```gdb
python import sys; print(sys.version)
```

実行結果：

```text
3.14.10 (main, Nov 14 2026, 15:01:54)  [MINGW GCC UCRT 16.2.0 64 bit (AMD64)]
```

今回の環境ではPython 3.14.10が使用されました。

このPythonは、GDB Multiarchの依存パッケージとしてMSYS2にインストールされたもので、Windowsにインストール済みのPythonとは独立しています。そのため、GDB用にPythonを別途インストールする必要はありません。

なお、PythonのバージョンはMSYS2のパッケージ更新状況によって異なります。

---

# 日本語コメントの文字化けを解消する

GDB Multiarchで日本語コメントを正しく表示するため、PowerShellのコンソール出力文字コードをUTF-8に設定します。

## PowerShellの文字コードを確認

PowerShell 7で、次のコマンドを実行し、コンソールの出力文字コードをコードページ番号で確認します。

```powershell
[Console]::OutputEncoding.CodePage
```

実行結果：

```text
932
```

私の環境では`932`でした。この番号は、日本語Windowsで使用されるShift_JIS系の文字コード（CP932）を表します。

一方、今回のソースコードはUTF-8で保存しています。

GDBの文字コードが自動設定になっている場合、PowerShell側の文字コードの影響を受けます。

この状態でGDBからソースコードを表示すると、日本語コメントが文字化けしました。

## UTF-8に変更する

PowerShell 7で、次のコマンドを実行し、コンソールの出力文字コードをUTF-8に変更します。

```powershell
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
```

続いて、文字コードが変更されたことを確認します。

```powershell
[Console]::OutputEncoding.CodePage
```

実行結果：

```text
65001
```

`65001`はUTF-8を表します。

この状態でGDBを起動し、`host-charset`を確認します。

```gdb
show host-charset
```

実行結果：

```text
The host character set is "auto; currently UTF-8".
```

GDBでもUTF-8が認識されています。

続いて、`list main`コマンドでソースコードを表示します。

```gdb
list main
```

実行結果：

```text
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

日本語コメントも正常に表示され、文字化けの問題が解決しました。

---

# Windows OS ABIの警告を解消する

GDB MultiarchでELFファイルを読み込むと、次の警告が表示されました。

```text
warning: A handler for the OS ABI "Windows" is not built into this configuration
of GDB. Attempting to continue with the default armv6s-m settings.
```

OS ABIは、プログラムが動作するOSに応じた、バイナリレベルの規約です。

今回、GDBはデバッグ対象のOS ABIを`Windows`として扱おうとしました。しかし、デバッグ対象はWindows上のプログラムではなく、Raspberry Pi Picoで動作するTry Kernelです。

まず、`show osabi`コマンドで現在の設定を確認します。

```gdb
show osabi
```

実行結果：

```text
The current OS ABI is "auto" (currently "Windows").
The default OS ABI is "Windows".
```

OS ABIの設定は`auto`で、現在のOS ABIとデフォルトのOS ABIはいずれも`Windows`になっています。

Try Kernelはベアメタルで動作するため、特定のOS ABIを指定しない`none`に変更します。

```gdb
set osabi none
```

続いて、設定が変更されたことを確認します。

```gdb
show osabi
```

実行結果：

```text
The current OS ABI is "none".
The default OS ABI is "Windows".
```

これで、ターゲットのOS ABIを`none`に設定できました。

ただし、GDBの起動後に変更しても、ELF読み込み時に発生した警告は消せません。

そこで、GDBの起動オプション`-iex`を使用します。

PowerShellから次のコマンドで起動します。

```powershell
C:\msys64\ucrt64\bin\gdb-multiarch.exe -q -iex "set osabi none" build/trykernel.elf
```

`-iex`は、ELFファイルの読み込み前にGDBコマンドを実行するオプションです。

この方法で起動したところ、警告は表示されなくなりました。

続いて、OS ABIの設定を確認します。

```gdb
show osabi
```

実行結果：

```text
The current OS ABI is "none".
The default OS ABI is "Windows".
```

現在のOS ABIが`none`に設定されていることを確認できました。

なお、これは今回のRP2040ベアメタル環境向けの設定です。異なるOSをデバッグする場合には、適切なOS ABIを選択する必要があります。

---

# PowerShellのプロファイルに設定を追加

毎回文字コードを変更したり、GDBの長いパスや起動オプションを指定したりするのは面倒です。

そこで、PowerShell 7のプロファイルに、**コンソールの文字コード設定とGDB起動用の関数**を追加します。

## プロファイルを開く

PowerShell 7で、次のコマンドを実行してプロファイルの保存先を確認します。

```powershell
$PROFILE
```

表示されたパスのファイルを、お使いのテキストエディタで開いてください。

プロファイルの保存場所は環境によって異なります。

既存の設定は変更せず、プロファイルの末尾に次の内容を追加します。

```powershell
# コンソールの出力文字コードをUTF-8に設定
[Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)

# MSYS2 GDB Multiarch
function gdb-multiarch {
    & "C:\msys64\ucrt64\bin\gdb-multiarch.exe" -iex "set osabi none" @args
}
```

`gdb-multiarch`関数は、GDB Multiarchをフルパスで呼び出すためのものです。

関数内では、次の指定を行っています。

- `&`：指定した実行ファイルを呼び出すPowerShellの演算子
- `-iex "set osabi none"`：ELFファイルを読み込む前にOS ABIを`none`に設定
- `@args`：関数に渡された引数をGDBへそのまま渡す

これにより、WindowsのPATHを変更せずに、`gdb-multiarch`コマンドでGDBを起動できるようになります。


## 設定を反映する

PowerShellで次のコマンドを実行し、プロファイルの設定を反映します。

```powershell
. $PROFILE
```

文字コードを確認します。

```powershell
[Console]::OutputEncoding.CodePage
```

実行結果：

```text
65001
```

`65001`はUTF-8を示します。

関数が登録されていることも確認します。

```powershell
Get-Command gdb-multiarch
```

実行結果：

```text
CommandType     Name
-----------     ----
Function        gdb-multiarch
```

これで、OS ABIの警告を表示させずに、`gdb-multiarch`コマンドでGDBを起動できます。

```powershell
gdb-multiarch -q build/trykernel.elf
```

※今回のPowerShellプロファイルには、起動時に作業ディレクトリを変更する設定も含まれています。プロファイルを再読み込みした場合は、ELFファイルのあるディレクトリに移動してからGDBを起動します。

---

# Pico実機での動作確認

最後に、OpenOCDを使用してPico実機での動作を確認します。

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

ターゲットを停止します。

```gdb
monitor reset halt
```

プログラムを書き込みます。

```gdb
load
```

`main`関数にブレークポイントを設定します。

```gdb
break main
```

実行を再開します。

```gdb
continue
```

ソースコードを確認します。

```gdb
list main
```

ステップ実行します。

```gdb
next
```

今回の環境では、OpenOCDとの接続、プログラムの書き込み、ブレークポイント、ステップ実行が正常に動作しました。

---

# まとめ

Windows 11で使用していたGDBでは、UTF-8で記述された日本語コメントが文字化けし、Pythonも使用できないという問題がありました。

そこでGDBの環境を見直し、最終的にMSYS2のGDB Multiarchを採用しました。

今回の環境では、次のことを確認できました。

- UTF-8の日本語コメントを正しく表示できる
- Pythonを使用できる
- 起動時のOS ABI警告を解消できる
- RP2040をOpenOCD経由でデバッグできる
- WindowsのPATHを変更せずに使用できる

既存のArm GNU ToolchainとOpenOCDはそのまま使用し、**GDBのみMSYS2のGDB Multiarchに置き換えています。**

GDBは配布元やビルド構成によって、対応機能や動作が異なります。今回の内容が、Windows環境でGDBの日本語表示やPython対応に困っている方の参考になれば幸いです。
