---
title: "Windows 11にZephyr RTOS開発環境を構築してRaspberry Pi Pico 2で動かす"
emoji: "⚙️"
type: "tech"
topics: ["zephyr", "raspberrypi", "pico", "rtos", "windows"]
published: false
---

## はじめに

Raspberry Pi Pico 2でZephyr RTOSを動かすため、Windows 11に開発環境を構築します。

Zephyrは、組込み機器向けのリアルタイムOS（RTOS）です。Devicetreeや共通のデバイスドライバモデルなど、ハードウェアを扱うための仕組みが充実しています。

今回はZephyrの開発環境を構築し、公式サンプルのBlinkyでLEDを点滅させるところまでを目標とします。

また、今後はGPIO、I2C、SPI、センサ制御など、複数の実験用アプリケーションを開発する予定です。そのため、**Zephyrの開発環境と自作アプリケーションのソースコードを分離する**構成にします。

自作アプリケーションは、まず1つのGitリポジトリにまとめて管理し、独立したプロジェクトとして発展させるものは、必要に応じて別リポジトリに分離する方針です。

## 開発環境

今回使用する環境は次のとおりです。

| 項目 | バージョン・構成 |
|---|---|
| OS | Windows 11 |
| シェル | PowerShell 7 |
| Python | 3.12.0 |
| pip | 26.0 |
| Git | 2.43.0.windows.1 |
| CMake | 4.4.4 |
| Ninja | 1.13.0 |
| west | 1.5.0 |
| Zephyr SDK | 1.0.1 |
| Zephyr RTOS | 4.5.0 |
| 開発ボード | Raspberry Pi Pico 2（RP2350） |

今回は既存の開発環境を利用していますが、初めて環境を構築する場合にも参考になるよう、必要なツールのインストール方法を記載します。

## ディレクトリ構成

Zephyr公式のGetting Startedでは、`zephyrproject` というワークスペースを作成します。

このワークスペースには、Zephyr本体や関連モジュールなどが配置されます。

今回は、自作アプリケーションをワークスペースの外で管理します。

```text
C:\home\<ユーザ名>\
├── env\
│   └── zephyrproject\
│       ├── .venv\
│       ├── .west\
│       ├── zephyr\
│       ├── modules\
│       └── bootloader\
│
└── repos\
    ├── zephyr-sensor-lab\
    │   ├── CMakeLists.txt
    │   ├── prj.conf
    │   ├── env.ps1
    │   └── src\
    │       └── main.c
    │
    └── zenn-contents\
```

`env/zephyrproject`はZephyrの開発環境、`repos/zephyr-sensor-lab`は今回の動作確認用アプリケーションです。

Zephyr本体はwestによって取得・管理するため、自分のGitHubリポジトリにコピーする必要はありません。

今後は`repos/zephyr`を実験用のGitリポジトリとし、複数のアプリケーションをその中で管理する予定です。コンテスト作品など、独立したプロジェクトとして開発するものは、必要に応じて別リポジトリに分離します。

このようなZephyrソースツリーの外にアプリケーションを配置する方式は、Out-of-tree Applicationと呼ばれます。

ディレクトリ名や配置先は任意です。今回使用した`C:\home\<ユーザ名>`はWindows標準のユーザディレクトリではないため、実際に構築する際は自分の環境に合わせてパスを変更してください。

---

## 1. 必要なツールのインストール

### Python

Zephyrのビルド環境ではPythonを使用します。今回はPython 3.12を使用します。

インストールは[Python公式サイト](https://www.python.org/downloads/windows/)の手順を参考にしてください。

インストール後、WindowsのPython Launcher（`py`）を使用してバージョンを表示し、実行できることを確認します。`py` は複数のPythonがインストールされている場合でも、バージョンを指定して実行できます。

```powershell
py -3.12 --version
```

実行結果：

```text
Python 3.12.0
```

### Git

GitはZephyr本体や関連モジュールの取得に使用します。

インストールは[Git公式サイト](https://git-scm.com/downloads/win)の手順を参考にしてください。

インストール後、バージョンを表示して実行できることを確認します。

```powershell
git --version
```

実行結果：

```text
git version 2.43.0.windows.1
```

### CMakeとNinja

Zephyrのビルドには、CMakeとNinjaを使用します。

- **CMake**：ビルド構成を生成するツール
- **Ninja**：CMakeが生成したビルド定義に従ってコンパイルなどを実行するツール

Windowsでは、標準で用意されているパッケージ管理ツール`winget`を使用してインストールできます。

```powershell
winget install --id Kitware.CMake -e --source winget
winget install --id Ninja-build.Ninja -e --source winget
```

インストール後、PowerShellを開き直し、それぞれのバージョンを確認します。

```powershell
cmake --version
ninja --version
```

実行結果：

```text
cmake version 4.4.4

1.13.0.git.kitware.jobserver-pipe-1
```

両方のツールが正常に実行できることを確認しました。

---

## 2. Python仮想環境の構築

Zephyr専用のPython仮想環境を作成します。

仮想環境を利用することで、ほかのPythonプロジェクトとパッケージの依存関係を分離できます。

今回は、前述のディレクトリ構成に従い、`env\zephyrproject` の下に `.venv` を作成します。

PowerShellで次のコマンドを実行します。

```powershell
New-Item -ItemType Directory -Force "C:\home\<ユーザ名>\env\zephyrproject"
cd "C:\home\<ユーザ名>\env\zephyrproject"

py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

仮想環境が有効になると、PowerShellのプロンプトに `(.venv)` が表示されます。

Pythonのバージョンと実行ファイルの場所を確認します。

```powershell
python --version
(Get-Command python).Source
```

実行結果：

```text
Python 3.12.0
C:\home\<ユーザ名>\env\zephyrproject\.venv\Scripts\python.exe
```

仮想環境内のPythonが使用されていることを確認できました。

以降のPythonパッケージは、この仮想環境にインストールします。

---

## 3. westのインストール

westはZephyrの開発を支援するコマンドラインツールで、Pythonパッケージとして提供されています。

Zephyr本体や関連リポジトリの取得、ビルド、書込み、デバッグなどを統一したコマンドで操作できます。

前項で作成したPython仮想環境を有効にした状態で、`pip`を使用してインストールします。

```powershell
python -m pip install west
```

インストール後、バージョンを表示して実行できることを確認します。

```powershell
west --version
```

実行結果：

```text
West version: v1.5.0
```

---

## 4. Zephyr RTOSの取得

westを使用してZephyrのワークスペースを初期化し、Zephyr RTOSと関連モジュールを取得します。

バージョンを指定しない場合、通常は開発中の`main`ブランチが取得されます。今回は安定版を使用するため、バージョンを指定して取得します。

安定版のリリース情報は、[Zephyr公式GitHubのReleases](https://github.com/zephyrproject-rtos/zephyr/releases)で確認できます。

今回は、**執筆時点で最新の正式リリース (Latest) であるZephyr v4.5.0**を使用します。

PowerShellで次のコマンドを実行します。--mr オプションで取得するZephyrのバージョンを v4.5.0 に指定します。

```powershell
cd "C:\home\<ユーザ名>\env\zephyrproject"

west init -m https://github.com/zephyrproject-rtos/zephyr --mr v4.5.0 .
west update
```

`west init` はワークスペースを初期化し、指定したバージョンのZephyrをマニフェストリポジトリとして取得します。

`west update` はマニフェストに従って、必要な関連リポジトリを取得します。

Zephyrの開発環境は単一のGitリポジトリではなく、複数のリポジトリで構成されています。

`west update`では多数のGitリポジトリを取得するため、初回の実行には時間がかかります。また、今回の環境ではZephyr本体と関連モジュールで約8.7GBのディスク容量を使用しました。十分な空き容量を確保してください。

取得が完了すると、`zephyr` や `modules` などのディレクトリが作成されます。

Zephyrのバージョンを確認します。

```powershell
git -C zephyr describe --tags --always
Get-Content .\zephyr\VERSION
```

実行結果：

```text
v4.5.0
VERSION_MAJOR = 4
VERSION_MINOR = 5
PATCHLEVEL = 0
VERSION_TWEAK = 0
EXTRAVERSION =
```

---

## 5. Python依存パッケージのインストール

Zephyrのビルドや各種ツールの実行に必要なPythonパッケージをインストールします。

Zephyrでは、必要なPythonパッケージが複数のrequirementsファイルで管理されています。westを使用すると、これらをまとめて取得できます。

Windowsでは、`west packages pip --install`を直接実行すると、west自身が使用しているパッケージの更新時に権限エラーが発生する可能性があります。

そのため、westが案内するPowerShell用の方法でインストールします。

```powershell
python -m pip install @((west packages pip) -split ' ')
```

このコマンドは、`west packages pip`で必要なパッケージの一覧を取得し、`pip`に渡してインストールするものです。

**初回は多数のPythonパッケージをダウンロード・インストールするため、時間がかかります。** 今回構築したPython仮想環境の容量は約0.63GB（630MB）でした。

インストールが完了したら、パッケージの依存関係を確認します。

```powershell
python -m pip check
```

実行結果：

```text
No broken requirements found.
```

Pythonパッケージの依存関係に問題がないことを確認できました。

---

## 6. Zephyr SDKのインストール

Zephyr SDKには、Zephyrのビルドに必要なコンパイラや関連ツールが含まれています。

今回は既存のArm GCCではなく、Zephyr SDKを使用します。

インストールは次の手順で進めます。

1. **7-Zipの準備**：SDKの展開・セットアップに必要な7-Zipを使用できるようにする
2. **Zephyr SDKのインストール**：SDK本体（minimal版）をインストールする
3. **Arm用ツールチェインの追加**：Raspberry Pi Pico 2向けのコンパイラをインストールする
4. **コンパイラの確認**：Arm用GCCが正常に実行できることを確認する

### 6.1 7-ZipのインストールとPATH設定

Windowsでは、Zephyr SDKのセットアップに7-Zipが必要です。

7-Zipがインストールされていない場合は、PowerShellで次のコマンドを実行します。

```powershell
winget install --id 7zip.7zip -e --source winget
```

7-Zipがインストールされていても、`7z.exe` がPATHに登録されていない場合はZephyr SDKのセットアップに失敗します。

今回は、PowerShellの環境変数PATHに7-Zipのインストール先を追加します。

```powershell
$env:PATH = "C:\Program Files\7-Zip;$env:PATH"
```

この設定は現在のPowerShellセッションでのみ有効です。以降のSDKインストールも同じPowerShellで実行してください。

次のコマンドで7-Zipが実行できることを確認します。

```powershell
7z.exe
```

実行結果（抜粋）：

```text
7-Zip 24.09 (x64)
Usage: 7z <command> [<switches>...] ...
```

これでZephyr SDKのセットアップに必要な7-Zipの準備ができました。

### 6.2 SDKのインストール

次のコマンドでZephyr SDKをインストールします。

```powershell
west sdk install
```

今回使用したZephyr v4.5.0では、Zephyr SDK 1.0.1が選択されました。

インストール先：

```text
C:\Users\<ユーザ名>\zephyr-sdk-1.0.1
```

なお、今回取得されたSDKはminimal版です。Arm用コンパイラは含まれていないため、次の項で追加します。

### 6.3 Arm用ツールチェインのインストール

`west sdk install`で取得されたSDKはminimal版で、コンパイラは含まれていません。

Raspberry Pi Pico 2（RP2350）のArm Cortex-M33向けにビルドするため、Arm用GNUツールチェインを追加します。

```powershell
& "$env:USERPROFILE\zephyr-sdk-1.0.1\setup.cmd" /t arm-zephyr-eabi
```

実行結果（抜粋）：

```text
Installing 'arm-zephyr-eabi' GNU toolchain ...
Everything is Ok
All done.
```

これでArm用ツールチェインのインストールが完了しました。

### 6.4 コンパイラの確認

Arm用GCCのバージョンを確認します。

```powershell
& "$env:USERPROFILE\zephyr-sdk-1.0.1\gnu\arm-zephyr-eabi\bin\arm-zephyr-eabi-gcc.exe" --version
```

実行結果（抜粋）：

```text
arm-zephyr-eabi-gcc.exe (Zephyr SDK 1.0.1) 14.3.0
```

Zephyr SDK 1.0.1に含まれるArm用GCC 14.3.0が正常に実行できることを確認しました。

これでRaspberry Pi Pico 2向けのビルドに必要なコンパイラの準備ができました。

---

## 7. Raspberry Pi Pico 2向けにビルド

最初の動作確認には、Zephyr公式のBlinkyサンプルを使用します。

BlinkyはLEDを周期的に点滅させるサンプルで、Zephyr本体と一緒に取得されているため、別途ダウンロードする必要はありません。

今回はRaspberry Pi Pico 2のオンボードLEDを点滅させます。

### 7.1 ビルド

PowerShellでZephyrのワークスペースに移動し、Pythonの仮想環境を有効にします。

```powershell
cd C:\home\<ユーザ名>\env\zephyrproject
.\.venv\Scripts\Activate.ps1
```

次のコマンドでBlinkyをビルドします。

```powershell
west build -b rpi_pico2/rp2350a/m33_0 zephyr/samples/basic/blinky
```

`rpi_pico2/rp2350a/m33_0`は、Raspberry Pi Pico 2のArm Cortex-M33コア0を指定しています。

なお、ビルドディレクトリに以前の設定が残っている場合は、`--pristine`を指定して初期化できます。

```powershell
west build --pristine -b rpi_pico2/rp2350a/m33_0 zephyr/samples/basic/blinky
```

### 7.2 ビルド結果

今回の環境では、Zephyr SDK 1.0.1が自動的に検出され、ビルドに成功しました。

実行結果（抜粋）：

```text
-- Zephyr version: 4.5.0
-- Board: rpi_pico2, qualifiers: rp2350a/m33_0
-- Found toolchain: zephyr 1.0.1
-- The C compiler identification is GNU 14.3.0
...
[154/154] Linking C executable zephyr\zephyr.elf
Memory region         Used Size  Region Size  %age Used
           FLASH:       18332 B         4 MB      0.44%
             RAM:        4072 B       520 KB      0.76%
        IDT_LIST:           0 B        32 KB      0.00%
...
Converted to uf2, output size: 36864, start address: 0x10000000
Wrote 36864 bytes to zephyr.uf2
```

FLASH使用量は18,332バイト、RAM使用量は4,072バイトでした。

ビルドによって、次のUF2ファイルが生成されます。

```text
build\zephyr\zephyr.uf2
```

## 8. Raspberry Pi Pico 2への書込み

Pico 2はBOOTSELモードを利用してUF2ファイルを書き込めます。

1. Pico 2のUSBケーブルを取り外します。
2. BOOTSELボタンを押したままUSBケーブルを接続します。
3. Windowsに認識された`RP2350`ドライブを確認します。
4. `build\zephyr\zephyr.uf2`を`RP2350`ドライブにコピーします。
5. コピー後、Pico 2が自動的に再起動します。

今回はOpenOCDやデバッグプローブを使用せず、USB経由で書き込みました。

SWDデバッグについては、別の機会に扱います。

## 9. 動作確認

UF2ファイルを書き込んだところ、Raspberry Pi Pico 2のオンボードLEDが周期的に点滅することを確認できました。

これにより、Windows 11上でZephyrの開発環境を構築し、公式サンプルをビルドして実機で動作させるまでの一連の手順を確認できました。

ここまでで、Zephyr RTOSを使用した開発の出発点ができました。

## 10. 自作アプリケーションを別リポジトリに配置する

ここまでの手順では、Zephyr公式のBlinkyサンプルを使用しました。

今後のセンサ制御開発に向けて、自作アプリケーションをZephyr本体とは別のGitリポジトリで管理します。

今回は公式Blinkyを参考に、LEDの点滅速度を変更したアプリケーションを作成し、ワークスペース外からのビルドと実機動作を確認します。

### 10.1 アプリケーションの作成

PowerShellで次のディレクトリを作成します。

```powershell
cd C:\home\<ユーザ名>\repos
mkdir zephyr-sensor-lab
cd zephyr-sensor-lab
mkdir src
```

ディレクトリ構成は次のとおりです。

```text
zephyr-sensor-lab/
├── CMakeLists.txt
├── prj.conf
├── env.ps1
└── src/
    └── main.c
```

`CMakeLists.txt`を作成します。

```cmake
cmake_minimum_required(VERSION 3.20.0)

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(zephyr_sensor_lab)

target_sources(app PRIVATE src/main.c)
```

`prj.conf`を作成します。

```ini
CONFIG_GPIO=y
```

`src/main.c`を作成します。

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

#define LED0_NODE DT_ALIAS(led0)

static const struct gpio_dt_spec led =
    GPIO_DT_SPEC_GET(LED0_NODE, gpios);

int main(void)
{
    int ret;

    if (!gpio_is_ready_dt(&led)) {
        return 0;
    }

    ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT_INACTIVE);
    if (ret < 0) {
        return 0;
    }

    while (1) {
        ret = gpio_pin_toggle_dt(&led);
        if (ret < 0) {
            return 0;
        }

        k_msleep(100);
    }

    return 0;
}
```

公式Blinkyとの違いを確認しやすいように、LEDの状態を100msごとに反転させます。

これにより、点灯100ms、消灯100ms、約200ms周期で点滅します。

### 10.2 開発環境の設定

Zephyrのワークスペース外で`west build`を実行したところ、最初は次のエラーが発生しました。

```text
west: unknown command "build"; do you need to run this inside a workspace?
```

`west`がZephyrのワークスペースを見つけられなかったことが原因です。

環境変数`ZEPHYR_BASE`を設定することで解決しました。

また、Python仮想環境の有効化も毎回必要になるため、これらをまとめた`env.ps1`を作成します。

```powershell
# env.ps1

& "C:\home\<ユーザ名>\env\zephyrproject\.venv\Scripts\Activate.ps1"

$env:ZEPHYR_BASE = "C:\home\<ユーザ名>\env\zephyrproject\zephyr"
```

`<ユーザ名>`は、自分の環境に合わせて変更してください。

PowerShellで次のように実行します。

```powershell
.\env.ps1
```

これで、現在のPowerShellセッションでZephyrの開発環境が使用できるようになります。

### 10.3 自作アプリケーションのビルド

`zephyr-sensor-lab`ディレクトリで、次のコマンドを実行します。

```powershell
west build -b rpi_pico2/rp2350a/m33_0 .
```

ビルド結果（抜粋）：

```text
-- Application: C:/home/<ユーザ名>/repos/zephyr-sensor-lab
-- Zephyr version: 4.5.0
-- Board: rpi_pico2, qualifiers: rp2350a/m33_0
-- Found toolchain: zephyr 1.0.1
...
[154/154] Linking C executable zephyr\zephyr.elf
Memory region         Used Size  Region Size  %age Used
           FLASH:       18228 B         4 MB      0.43%
             RAM:        4072 B       520 KB      0.76%
        IDT_LIST:           0 B        32 KB      0.00%
...
Wrote 36864 bytes to zephyr.uf2
```

Zephyr本体とは別のディレクトリに配置したアプリケーションを、正常にビルドできました。

生成されたUF2ファイルは次の場所にあります。

```text
zephyr-sensor-lab\build\zephyr\zephyr.uf2
```

ソースコードだけを変更した場合は、次のコマンドで再ビルドできます。

```powershell
west build
```

### 10.4 実機動作の確認

第8項と同じ手順で、生成した`zephyr.uf2`をRaspberry Pi Pico 2に書き込みます。

書き込み後、オンボードLEDが公式Blinkyよりも高速に点滅することを確認できました。

これで、Zephyr本体と自作アプリケーションを別のディレクトリで管理し、ビルドから実機動作まで行えることを確認できました。

### 10.5 今後のGit管理方針

今回の動作確認では、`zephyr-sensor-lab`をZephyrのワークスペース外に配置しました。

今後は、`repos/zephyr`という実験用リポジトリを作成し、GPIO、Devicetree、I2C、SPI、センサ制御などのアプリケーションをまとめて管理する予定です。

各アプリケーションは独立してビルドできる構成とし、Zephyr本体は共通の開発環境を使用します。

また、ビルド生成物はGit管理の対象外とします。例えば、`.gitignore`には次のように記載します。

```gitignore
build/
**/build/
__pycache__/
*.pyc
```

コンテスト作品など、独立したプロジェクトとして開発する段階になったら、必要なソースコードを別リポジトリに分離する方針です。

これにより、Zephyr本体を変更せず、複数の実験を進められる環境を整えていきます。

## おわりに

Windows 11にZephyrの開発環境を構築し、Raspberry Pi Pico 2で公式サンプルを動かすまでを確認しました。

今回の構成では、Zephyrの開発環境と自作アプリケーションを分離しています。

これにより、Zephyr本体を更新しながら、複数の実験用アプリケーションを共通の開発環境で管理できます。

次はDevicetreeやDevice Modelを確認し、実際のセンサをZephyrから制御するところへ進みます。

## 参考資料

- [Zephyr Project - Getting Started](https://docs.zephyrproject.org/latest/develop/getting_started/)
- [Zephyr Project - Application Development](https://docs.zephyrproject.org/latest/develop/application/index.html)
- [Zephyr Project - Raspberry Pi Pico 2](https://docs.zephyrproject.org/latest/boards/raspberrypi/rpi_pico2/doc/index.html)
- [Raspberry Pi Documentation](https://www.raspberrypi.com/documentation/microcontrollers/)
