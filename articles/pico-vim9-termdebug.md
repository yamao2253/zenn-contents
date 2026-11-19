---
title: "Vim 9のtermdebugでRaspberry Pi Picoをデバッグする"
emoji: "🔧"
type: "tech"
topics: ["vim", "gdb", "openocd", "raspberrypi"]
published: true
---

# はじめに

前回の記事では、Windows 11環境でOpenOCDとGDBを使用し、Raspberry Pi Pico上で動作するTry Kernelをコマンドラインからデバッグしました。

今回は、Vim 9の標準機能である `termdebug` を使用して、Vim上からGDBを操作してみます。

`termdebug` を使うと、GDBのコマンドライン操作に加えて、ソースコード上でのブレークポイント設定やステップ実行、現在の実行位置の表示などが可能になります。

追加のVimプラグインを導入せず、実機でどこまでデバッグできるか確認します。

# 使用環境

今回使用した環境は次のとおりです。

| 項目 | 環境 |
|---|---|
| OS | Windows 11 |
| エディタ | GVim 9.0（64bit版） |
| デバッグ対象 | Raspberry Pi Pico W（RP2040） |
| デバッグプローブ | Pico Wを使用したCMSIS-DAP Debug Probe |
| GDB | MSYS2 GDB Multiarch 18.1 |
| OpenOCD | xPack OpenOCD 0.12.0-7.1 |
| 対象プログラム | Try Kernel（`part_5`） |

OpenOCDとGDBのインストール、デバッグプローブの準備、配線については、前回までの記事で紹介しています。

- [「ラズパイPicoで1500行 ゼロから作るOS」を2026年に動かす](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)
- [OpenOCDとGDBでRaspberry Pi Picoをデバッグする](https://zenn.dev/helowota/articles/pico-openocd-gdb-debug)
- [Windows 11でGDBの日本語文字化けを解消 ― MSYS2 GDB Multiarchを使う](https://zenn.dev/helowota/articles/pico-gdb-multiarch-windows-env)

今回は、これらの環境が構築済みであることを前提とします。

# Vimのtermdebugとは

`termdebug` はVim 9に標準で付属するプラグインで、追加のインストールは不要です。

`termdebug` はGDBを置き換えるものではなく、**GDBのデバッグ機能とVimのソースコード表示・操作機能を連携させる仕組み**です。

Vim内にGDBのターミナルを開くだけでなく、GDBの停止位置に合わせてソースコードを表示したり、ソースコード上からブレークポイントを操作したりできます。

デバッグ操作は、**GDBのターミナルからコマンドを入力する方法**と、**Vimのソースコード表示画面から `:Break` や `:Step` などのコマンドを実行する方法**の両方が使えます。

以降では、操作する場所を区別して説明します。

# コマンドラインのGDBとの違いは？

前回の記事では、GDBのコマンドラインからブレークポイントの設定やステップ実行を行いました。これだけでも実機デバッグは可能ですが、ソースコードを確認しながら操作するには、GDBとエディタを行き来する必要があります。

Vimの `termdebug` を使うと、GDBのデバッグ機能をそのまま利用しながら、ソースコードと連動した操作が可能になります。

| 操作 | コマンドラインのGDB | Vimのtermdebug |
|---|---|---|
| ソースコードの確認 | GDBのコマンドや別のエディタで確認 | 停止位置のソースを自動表示 |
| 実行位置の確認 | ファイル名と行番号で確認 | 該当行を強調表示 |
| ブレークポイント設定 | `break main`、`break ファイル名:行番号` | ソースコード上で `:Break` |
| ブレークポイント解除 | `delete 番号` | ソースコード上で `:Clear` |
| ステップ実行 | `step`、`next`、`finish` | ソースコード表示画面で `:Step`、`:Over`、`:Finish` |
| 変数の確認 | `print`、`display` | ソースコード表示画面で `:Evaluate`（継続的な監視にはGDBターミナルで `display` を実行） |

今回、特に便利だと感じたのは、ブレークポイントで停止すると、その場所のソースコードが自動的に表示されることです。

また、ブレークポイントを設定する際に、関数名や行番号をGDBへ入力する必要がなく、ソースコード上のブレークポイントを設定したい行にカーソルを置いて `:Break` を実行できます。

---

# termdebugを使うための事前設定

Vimの `termdebug` は標準プラグインですが、使用するには `packadd` による読み込みが必要です。

また、今回はMSYS2のGDB Multiarchを使用するため、`g:termdebug_config` にGDBの実行ファイルと起動オプションを設定します。

GVimで次のコマンドを実行します。

```vim
:packadd termdebug
:let g:termdebug_config = {'command': ['C:/msys64/ucrt64/bin/gdb-multiarch.exe', '-iex', 'set osabi none']}
```

`g:termdebug_config` の `command` には、使用するGDBの実行ファイルと起動引数をリスト形式で指定します。

今回指定している内容は次のとおりです。

| 指定 | 内容 |
|---|---|
| `C:/msys64/ucrt64/bin/gdb-multiarch.exe` | MSYS2 GDB Multiarchの実行ファイル |
| `-iex` | GDBの初期化処理より前にコマンドを実行するオプション |
| `set osabi none` | OS ABIをベアメタル環境向けに設定 |

GDB MultiarchはWindows向けにビルドされているため、そのまま起動するとOS ABIがWindowsになります。

今回はRP2040上で動作するTry Kernelをデバッグするため、起動時に `set osabi none` を実行します。

この設定により、GDB起動後に毎回OS ABIを変更する必要がなくなります。

## 設定の自動化

上記の設定はGVimを起動するたびに必要ですが、Vimの設定ファイルに記述しておけば自動的に設定されます。

Windowsの場合は、通常ホームディレクトリにある `_vimrc` に以下を追加します。

```vim
packadd termdebug

let g:termdebug_config = {
    \ 'command': [
    \   'C:/msys64/ucrt64/bin/gdb-multiarch.exe',
    \   '-iex',
    \   'set osabi none'
    \ ]
    \ }
```

なお、`_vimrc` に記述する場合は、先頭の `:` は不要です。

設定ファイルの場所は、GVimで `:echo $MYVIMRC` と入力すると確認できます。

## 設定を確認する

GVimを再起動し、次のコマンドでtermdebugを起動します。

```vim
:Termdebug
```

GDBターミナルで、OS ABIを確認します。

```gdb
show osabi
```

実行結果は次のとおりです。

```text
The current OS ABI is "none".
The default OS ABI is "Windows".
```

OS ABIが `none` になっていることを確認できました。

これで、GDB Multiarchを使用するための設定は完了です。

---

# termdebugとOpenOCD/GDBでTry Kernelをデバッグする

ここからは、実際にRaspberry Pi Pico上で動作するプログラムを使って、`termdebug` によるデバッグを試してみます。

今回のデバッグ対象は、Interface誌の特集「ラズパイPicoで1500行 ゼロから作るOS」で紹介されている自作OS「Try Kernel」です。

Try Kernelの開発環境の構築方法や、OpenOCDとGDBを使ったコマンドラインでのデバッグ方法については、以下の記事で紹介しています。

- [Windows 11でRaspberry Pi PicoのTry Kernel開発環境を構築する](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)
- [Raspberry Pi PicoをOpenOCDとGDBでデバッグする](https://zenn.dev/helowota/articles/pico-openocd-gdb-debug)

今回は、この環境を利用して、Vimのソースコード表示画面とGDBを連携させたデバッグ操作を確認します。

## OpenOCDを起動する

まず、PowerShellでOpenOCDを起動します。

```powershell
openocd -f interface/cmsis-dap.cfg -c "adapter speed 1000" -f target/rp2040.cfg
```

ここでは、デバッグプローブ（CMSIS-DAP）とデバッグ対象（RP2040）の設定ファイルを指定しています。また、`adapter speed 1000` でSWDの通信速度を1000kHzに設定しています。

OpenOCDはGDBサーバとして動作し、GDBからの接続を待ち受けます。

OpenOCDを起動したPowerShellは、そのまま開いておきます。

## termdebugからGDBを起動する

GVimで次のコマンドを実行します。

```vim
:Termdebug
```

GVimの画面が分割され、上側にGDBターミナル、下側にソースコード表示画面が開きます。

<img src="/images/pico-vim9-termdebug/termdebug-start.png"
     alt="termdebug起動直後のGVim画面"
     width="700">

上側のGDBターミナルには `No symbol table is loaded. Use the "file" command.` と表示されています。これは、まだデバッグ対象のELFファイルを読み込んでいないためです。

次に、GDBターミナルからELFファイルを読み込み、OpenOCDへ接続します。

---

## ELFファイルを読み込む

ここからは、GVim内に開いた**GDBターミナル**で操作します。

`gdb>` プロンプトで、Try KernelのELFファイルを読み込みます。

```gdb
file C:/home/<ユーザ名>/repos/src/pico/interface_trykernel/part_5/build/trykernel.elf
```

実際のELFファイルの保存場所に合わせてパスを指定してください。

今回の環境では、次のようにシンボル情報が読み込まれました。

```text
Reading symbols from C:/home/.../part_5/build/trykernel.elf...
gdb>
```

## OpenOCDに接続する

引き続き、GDBターミナルでOpenOCDに接続します。

```gdb
target remote localhost:3333
```

接続に成功すると、GVim側に現在の実行位置のソースコードが表示されます。

表示されるファイルや行番号は、接続時点のRP2040の実行状態によって異なります。

今回の検証でも、GDB Multiarchの停止位置に合わせてソースコードが自動的に表示されることを確認できました。

<img src="/images/pico-vim9-termdebug/termdebug-connected.png" alt="OpenOCD接続後、現在の実行位置が表示されたtermdebug画面" width="700">

このように、**GDBの実行位置とVimのソースコード表示が連動している**ことが確認できます。

---

# ブレークポイントとステップ実行

OpenOCDへの接続が完了し、GVimにソースコードが表示されました。

ここからは、**GVimのソースコード表示画面を中心にデバッグを進めます。**

## ブレークポイントを設定する

まず、Try Kernelの `main` 関数にブレークポイントを設定します。

OpenOCDへの接続後、Vimの通常操作で `main` 関数が定義されている `kernel/inittsk.c` を開きます。

```vim
:e C:/home/<ユーザ名>/repos/src/pico/interface_trykernel/part_5/kernel/inittsk.c
```

`termdebug` では、GDBが自動表示したソースだけでなく、Vimで任意のソースファイルを開いてデバッグ操作ができます。

`main` 関数内の31行目にカーソルを移動し、ブレークポイントを設定します。

```vim
:Break
```

<img src="/images/pico-vim9-termdebug/termdebug-breakpoint.png" alt="GVimのソースコード上からブレークポイントを設定した画面" width="700">

左端に赤いブレークポイント番号 `01` が表示され、設定できたことが確認できます。

今回は `main` 関数の先頭付近で停止させるため、GDBターミナルでRP2040をリセットします。

```gdb
monitor reset halt
```

その後、GVimのソースコード表示画面で実行を再開します。

```vim
:Continue
```

`main` 関数で停止すると、現在の実行位置が強調表示されます。

ブレークポイントを解除するには、設定した行で次を実行します。

```vim
:Clear
```

## ステップ実行する

ブレークポイントで停止したら、GVimのソースコード表示画面からステップ実行できます。

```vim
:Step
```

`:Step` はステップイン、`:Over` はステップオーバ、`:Finish` は現在の関数から戻るまで実行するコマンドです。

これらを実行すると、GVimのソースコード表示が実行位置に追従します。

また、GVimの画面上部にあるボタンからも、ステップ実行や実行再開などを操作できます。

## GDBターミナルとの併用

ここまでは、ソースコード表示画面を中心にデバッグ操作を行いました。

一方、`termdebug` ではGDBターミナルも引き続き利用できます。

例えば、ソースコード上でブレークポイントを設定し、レジスタやメモリの内容を詳しく調べたいときにはGDBコマンドを使う、といった使い分けができます。

**普段はVimのソースコード上から操作し、必要に応じてGDB本来の機能も使える。** この柔軟さが `termdebug` の利点です。

## VimとGDBのコマンド比較

ここまで紹介した操作を、VimとGDBのコマンドで比較すると次のようになります。

| 操作 | Vim/termdebug | GDB |
|---|---|---|
| ブレークポイント設定 | `:Break` | `break` |
| ブレークポイント解除 | `:Clear` | `clear` |
| ステップイン | `:Step` | `step` |
| ステップオーバ | `:Over` | `next` |
| 関数から戻る | `:Finish` | `finish` |
| 実行再開 | `:Continue` | `continue` |

Vimコマンドはソースコード表示画面で、GDBコマンドはGDBターミナルで実行します。

どちらも同じGDBを操作しているため、実行位置やブレークポイントの情報がGVimの画面に反映されます。

---

# 変数の値を確認する

デバッグでは、ブレークポイントやステップ実行だけでなく、変数の値を確認する操作も重要です。

## ソースコード上で変数を評価する

`termdebug` では、ソースコード上の変数にカーソルを置き、次のコマンドで値を確認できます。

```vim
:Evaluate
```

## 変数の値を継続して確認する

GDBには、プログラムが停止するたびに登録した式の値を表示する `display` コマンドがあります。

例えば、`tskid_ini` を登録する場合は、GDBターミナルで次を実行します。

```gdb
display tskid_ini
```

これはVimのコマンドではなく、GDBの機能です。

なお、GDBの `watch` は値を表示し続ける機能ではなく、値が変更されたときに実行を停止させるウォッチポイントです。

## メモリの内容を確認する

メモリの内容を直接確認したい場合は、GDBターミナルで `x` コマンドを使用します。

```gdb
x/16wx 0x20000000
```

`x` コマンドの `/` 以降は表示方法の指定で、`16` は表示個数、`w` は32bit（word）、最後の `x` は16進数表示を指定します。

この例では、アドレス `0x20000000` から64バイトのメモリ内容を表示します。

このように、termdebugから直接操作できない機能も、GDBターミナルを併用することで利用できます。

---

## ファンクションキーでデバッグする

termdebugのコマンドはVimのキーマッピングに登録できます。

例えば、VSCodeのデバッグ操作に合わせて、次のように設定できます。

```vim
nnoremap <F5> :Continue<CR>
nnoremap <F10> :Over<CR>
nnoremap <F11> :Step<CR>
nnoremap <S-F11> :Finish<CR>
```

これにより、ソースコード画面からファンクションキーで実行再開やステップ実行ができます。

| キー | 操作 |
|---|---|
| `F5` | 実行再開 |
| `F10` | ステップオーバ |
| `F11` | ステップイン |
| `Shift+F11` | ステップアウト |

これらはVimのノーマルモードで有効な設定です。GDBターミナルでは、従来どおりGDBのコマンドを使用します。

---

## デバッグを終了する

デバッグを終了するには、GDBターミナルで次のコマンドを実行します。

```gdb
quit
```

GDBが終了すると、termdebugのデバッグセッションも終了します。

GVim自体は終了しないため、そのままソースコードの編集を続けたり、`:Termdebug` でデバッグを再開したりできます。

OpenOCDも終了する場合は、起動したPowerShellで `Ctrl+C` を押します。

---

# おわりに

Vim 9の標準機能である `termdebug` を使用し、Windows 11上のGVimからRaspberry Pi Pico Wの実機デバッグを行えました。

OpenOCDへの接続、ソースコードへの実行位置の追従、ブレークポイントの設定と解除、ステップ実行、変数やメモリの確認など、組込みデバッグに必要な基本機能を利用できます。

特に印象的だったのは、**追加のVimプラグインをインストールすることなく、使い慣れたVimを中心にデバッグできる**ことです。ファンクションキーに操作を割り当てれば、IDEに近い感覚でステップ実行もできます。

もちろん、変数の継続監視やメモリダンプなど、GDBターミナルで操作する機能もあります。しかし、ソースコード上での操作とGDBの機能を必要に応じて使い分けられるため、思っていた以上に実用的な環境になりました。

VSCodeを使用したデバッグ環境についても、別の記事で紹介しています。
