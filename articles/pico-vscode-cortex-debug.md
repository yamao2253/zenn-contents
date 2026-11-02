---
title: "VSCodeとOpenOCD/GDBでRaspberry Pi Picoをデバッグする"
emoji: "🛠️"
type: "tech"
topics: ["raspberrypi", "pico", "vscode", "gdb", "openocd"]
published: false
---

# はじめに

これまでの記事では、Windows 11上にTry Kernelの開発環境を構築し、OpenOCDとGDBを使ってRaspberry Pi Pico Wの実機デバッグを行いました。

また、Vim 9の標準機能である `termdebug` を使い、GVimからソースコードと連動したデバッグも行いました。

今回は、Visual Studio Code（VSCode）の拡張機能 **Cortex-Debug** を使って、同じTry Kernelをデバッグします。

Cortex-Debugを使用すると、OpenOCD・GDBの起動を自動化し、ブレークポイントの設定やステップ実行、変数の監視などをVSCode上で操作できます。

内部ではこれまでと同じOpenOCDとGDBを使用していますが、それらの起動や接続もCortex-Debugに任せることができます。

## 前提となる環境

今回使用する環境は、これまでの記事と同じです。

- Windows 11
- Raspberry Pi Pico W（RP2040）
- CMSIS-DAP対応デバッグプローブ
- OpenOCD
- Arm GNU Toolchain（`arm-none-eabi-gdb`）
- Try Kernel（`part_5`）

Try Kernelはビルド済みで、デバッグ情報を含む `trykernel.elf` が生成されているものとします。

開発環境の構築やOpenOCD/GDBの基本操作については、以下の記事を参照してください。

- [「ラズパイPicoで1500行 ゼロから作るOS」を2026年に動かす](https://zenn.dev/helowota/articles/pico-trykernel-windows-env)
- [OpenOCDとGDBでRaspberry Pi Picoをデバッグする](https://zenn.dev/helowota/articles/pico-openocd-gdb-debug)

また、Vimでデバッグする場合は、[Vim 9のtermdebugでRaspberry Pi Picoをデバッグする](https://zenn.dev/helowota/articles/pico-vim9-termdebug) を参照してください。同じ環境を使用しているため、VSCodeとの操作の違いも比較できます。

---

# Cortex-Debugを導入する

VSCodeに拡張機能「Cortex-Debug」をインストールします。

Cortex-Debugは、Arm Cortex-M向けのデバッグ拡張機能で、OpenOCDとGDBを使用した実機デバッグをVSCodeから操作できるようにします。

VSCodeの拡張機能画面（`Ctrl+Shift+X`）で `Cortex-Debug` を検索し、インストールします。

# デバッグ構成を設定する

## OpenOCDとGDBの役割

今回のデバッグ環境は、概念的には次の構成です。

```text
VSCode
  └─ Cortex-Debug
       ├─ GDB（arm-none-eabi-gdb）
       │    └─ GDB Remote Protocol
       └─ OpenOCD
            └─ CMSIS-DAP
                 └─ SWD
                      └─ Raspberry Pi Pico W
```

これまで手動で実行していたOpenOCDの起動やGDBの接続を、Cortex-Debugに任せます。

OpenOCDとGDBの役割は変わりません。変わるのは、それらを起動・操作する方法です。

## launch.jsonを作成する

VSCodeでは、デバッグ構成を `.vscode/launch.json` に記述します。

今回はCortex-DebugがOpenOCDを起動する構成を使用します。

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Try Kernel (OpenOCD)",
            "type": "cortex-debug",
            "request": "launch",
            "servertype": "openocd",
            "cwd": "${workspaceFolder}",
            "executable": "${workspaceFolder}/part_5/build/trykernel.elf",
            "device": "RP2040",
            "interface": "swd",
            "configFiles": [
                "interface/cmsis-dap.cfg",
                "target/rp2040.cfg"
            ],
            "serverArgs": [
                "-c",
                "adapter speed 1000"
            ],
            "runToEntryPoint": "main"
        }
    ]
}
```

<!-- TODO: ワークスペースのルート、Cortex-Debugのバージョン、設定項目、起動動作を実機で確認する。上記は検証前の暫定設定。 -->

ここでは、主に次の項目を指定しています。

| 項目 | 内容 |
|---|---|
| `type` | Cortex-Debugを使用する |
| `servertype` | OpenOCDを使用する |
| `executable` | デバッグ情報を含むELFファイル |
| `configFiles` | CMSIS-DAPとRP2040のOpenOCD設定 |
| `serverArgs` | OpenOCDへ渡す追加引数 |
| `runToEntryPoint` | 起動後に停止させる関数 |

なお、OpenOCDをPowerShellから手動起動し、VSCodeから接続する構成も可能です。今回は、Cortex-DebugにOpenOCDの起動も任せる構成を使用します。

# VSCodeからデバッグを開始する

VSCodeでTry Kernelのワークスペースを開き、デバッグ構成「Try Kernel (OpenOCD)」を選択します。

`F5` を押すとデバッグが開始されます。

Cortex-DebugがOpenOCDとGDBを起動し、Raspberry Pi Pico Wへ接続します。

<!-- TODO: 実機で起動確認。mainで停止するか、起動ログ、表示されるソースを確認する。 -->

<!-- TODO: デバッグ開始後のVSCode画面を掲載 -->

ここで、従来のGDB記事との違いを確認できます。

これまではPowerShellでOpenOCDを起動し、GDBから `target remote localhost:3333` を実行していました。

VSCodeでは、これらの操作を拡張機能が担当します。

# ブレークポイントとステップ実行

## ブレークポイントを設定する

Try Kernelの `kernel/inittsk.c` を開き、`main` 関数内の31行目にブレークポイントを設定します。

VSCodeでは、ソースコード左端の余白をクリックするか、`F9` でブレークポイントを設定・解除できます。

<!-- TODO: 実機で設定と停止を確認し、画像を掲載する。 -->

## ステップ実行する

VSCodeでは、標準のファンクションキーでデバッグ操作ができます。

| キー | 操作 | GDBの相当コマンド |
|---|---|---|
| `F5` | 実行再開 | `continue` |
| `F10` | ステップオーバ | `next` |
| `F11` | ステップイン | `step` |
| `Shift+F11` | ステップアウト | `finish` |

termdebugの記事では、これらの操作をVimのキーマッピングに登録しました。

VSCodeでは標準のデバッグ操作として用意されているため、追加設定なしで利用できます。

<!-- TODO: 各操作を実機で確認する。 -->

# 変数の値を確認する

## 変数の値を確認する

デバッグ中にプログラムが停止すると、VSCodeのデバッグ画面から変数の値を確認できます。

ソースコード上で変数にマウスカーソルを合わせる方法もあります。

<!-- TODO: inittsk.cの実在する変数で確認する。 -->

## ウォッチ式を登録する

継続して値を確認したい変数や式は、VSCodeの「ウォッチ」に登録できます。

これはGDBターミナルで `display` コマンドを使う場合と目的は似ていますが、VSCodeではGUI上で登録した式を確認できます。

<!-- TODO: ウォッチ式の登録・更新・削除を確認し、画像を掲載する。 -->

# メモリの内容を確認する

組込み開発では、変数だけでなくメモリの内容を直接確認したいこともあります。

GDBでは、例えば次のコマンドを使用します。

```gdb
x/16wx 0x20000000
```

`x` コマンドの `/` 以降は表示方法の指定で、`16` は表示個数、`w` は32bit（word）、最後の `x` は16進数表示を指定します。

この例では、アドレス `0x20000000` から64バイトのメモリ内容を表示します。

<!-- TODO: VSCodeでメモリ表示を行う方法を確認する。Cortex-Debugの機能、VSCodeのメモリ表示、GDBコンソールのどれを記事の中心にするか判断する。 -->

# デバッグを終了する

VSCodeのデバッグ操作から停止を実行します。

<!-- TODO: 停止時にGDBとOpenOCDが終了すること、再度F5で開始できることを確認する。 -->

# おわりに

今回はVSCodeのCortex-Debugを使用し、Windows 11上からRaspberry Pi Pico WのTry Kernelをデバッグする環境を構築します。

これまでの記事で手動操作していたOpenOCDやGDBを、VSCodeから利用できるようにするのが今回の目的です。

便利な開発環境を使うことと、その内部の仕組みを理解することは矛盾しません。

普段はVSCodeに操作を任せ、問題が発生したときにはOpenOCDやGDBの知識を使って原因を調べられる。そんな開発環境を目指します。

<!-- TODO: 実際に使った感想を反映して、公開前に書き直す。 -->
