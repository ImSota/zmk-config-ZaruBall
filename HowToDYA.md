# ZaruBall v3 DYA Studio 対応手順

ZaruBall v3 を DYA Studioに対応させるための手順です。
本手順では、ZMKのフォーク版である [cormoran](https://github.com/cormoran/zmk) 版を使用し、必要なモジュールを組み込む方法を解説します。

## DYA Studioとは？
そもそもDYA Studioってなに？という方はDYA Studioのデモモードでどのような設定ができるか確認してみてください。
 [DYA Studio](https://studio.dya.cormoran.works/)\
特に素晴らしい点はファームウェアのビルド、書き込みを不要とせず、直接DYA Studioからトラックボール・ロータリーエンコーダーの設定変更ができる点です。その他にもバッテリー残量の履歴の確認、スリープの設定を簡単に変更できます。

## 1. `config/west.yml` の編集

`config/west.yml` を編集し、ZMKの取得元を変更するとともに、DYA Studio 用のモジュールを追加します。

```yaml
manifest:
  remotes:
    # cormoranリモートを追加
    - name: cormoran
      url-base: https://github.com/cormoran
    # (既存のzmkfirmwareリモート設定は削除)
    # - name: zmkfirmware
    #   url-base: https://github.com/zmkfirmware

  projects:
    # ZMK 本体を cormoran フォークに変更
    - name: zmk
      remote: cormoran
      revision: v0.3+custom-studio-protocol
      import: app/west.yml

    # 以下の DYA Studio 用モジュールを追加
    - name: zmk-module-ble-management
      remote: cormoran
      revision: main
    - name: zmk-module-battery-history
      remote: cormoran
      revision: main
    - name: zmk-module-settings-rpc
      remote: cormoran
      revision: main
    - name: zmk-module-runtime-input-processor
      remote: cormoran
      revision: main
    - name: zmk-behavior-runtime-sensor-rotate
      remote: cormoran
      revision: main
```

## 2. キーマップ (`config/ZaruBall.keymap`) の編集

DYA Studio の機能（動的なキーマップ変更、マウス設定、エンコーダー設定など）を利用するために、必要なヘッダのインクルードと設定を追加します。

### ヘッダファイルのインクルード
ファイルの冒頭（`#include <behaviors.dtsi>` のあたり）に以下を追加します。

```c
#include <input/processors/runtime-input-processor.dtsi>
#include <behaviors/battery_history_request.dtsi>
#include <behaviors/runtime-sensor-rotate.dtsi>
```

### エンコーダーの動作設定 (Behavior)
エンコーダー（ロータリーエンコーダー）の動作を Studio から変更可能にするため、標準の `sensor-bindings` ではなく、`runtime-sensor-rotate` ビヘイビアを使用するように定義します。

```c
/ {
    behaviors {
        // スクロール用
        EC11_scroll: behavior_ec11_scroll {
            compatible = "zmk,behavior-runtime-sensor-rotate";
            #sensor-binding-cells = <0>;
            cw-binding = <&msc SCRL_DOWN>;  // デフォルト動作
            ccw-binding = <&msc SCRL_UP>;    // デフォルト動作
            tap-ms = <5>;
        };

        // 音量用
        EC11_vol: behavior_ec11_vol {
            compatible = "zmk,behavior-runtime-sensor-rotate";
            #sensor-binding-cells = <0>;
            cw-binding = <&kp C_VOL_UP>;
            ccw-binding = <&kp C_VOL_DN>;
            tap-ms = <5>;
        };
    };
};
```

キーマップの `sensor-bindings` では、上記で定義した `&EC11_scroll` や `&EC11_vol` を使用します。

```c
    sensor-bindings = <&EC11_scroll>;
```

### マウス/トラックボール設定 (`runtime-input-processor`)
トラックボールやマウスカーソルの挙動（速度やスクロール量など）を動的に変更する場合、`trackball_listener` に `&mouse_runtime_input_processor` を追加します。

```c
&trackball_listener {
    input-processors = <
        // 他のプロセッサ (例: &zip_xy_transform ...)
        &mouse_runtime_input_processor
    >;
    scroller {
        input-processors = <
            // 他のプロセッサ
            &scroll_runtime_input_processor
        >;
    };
};
```

### Studio Unlock キーの配置
ZaruBall v3 のデフォルトキーマップでは既に `&studio_unlock` が設定されています。
もしカスタマイズの過程で削除してしまった場合は、Adjustレイヤーなどに再度追加してください。
DYA Studio で設定変更のロックを解除するために必要です。

```c
&studio_unlock
```

## 3. 設定ファイル (`config/*.conf`) の編集

従来共通だった `ZaruBall.conf` を左右別々の設定ファイル (`ZaruBall_left.conf`, `ZaruBall_right.conf`) に分割し、それぞれの役割に応じた設定を記述します。
DYA Studioを使用するために必要な設定が含まれています。

### `config/ZaruBall.conf` (共通設定)
左右共通の基本的な設定ファイルです。

```properties
CONFIG_SPI=y
CONFIG_NFCT_PINS_AS_GPIOS=y

CONFIG_ZMK_IDLE_TIMEOUT=30000
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=3600000
```

### `config/ZaruBall_right.conf` (Central / トラックボール側)
主にこちらでStudio機能やBLE管理、マウス設定を有効化します。以下は左右分割時の右側用設定ファイルの全内容です。

```properties
CONFIG_ZMK_POINTING=y
CONFIG_PMW3610=y
CONFIG_INPUT=y

CONFIG_RGBLED_WIDGET=y
CONFIG_RGBLED_WIDGET_INTERVAL_MS=500    

CONFIG_RGBLED_WIDGET_BATTERY_LEVEL_HIGH=60
CONFIG_RGBLED_WIDGET_BATTERY_LEVEL_LOW=30
CONFIG_RGBLED_WIDGET_BATTERY_LEVEL_CRITICAL=10

CONFIG_RGBLED_WIDGET_SHOW_LAYER_COLORS=y
CONFIG_RGBLED_WIDGET_LAYER_0_COLOR=0
CONFIG_RGBLED_WIDGET_LAYER_1_COLOR=6
CONFIG_RGBLED_WIDGET_LAYER_2_COLOR=1
CONFIG_RGBLED_WIDGET_LAYER_3_COLOR=4
CONFIG_RGBLED_WIDGET_LAYER_4_COLOR=5
CONFIG_RGBLED_WIDGET_LAYER_5_COLOR=2
CONFIG_RGBLED_WIDGET_LAYER_6_COLOR=3

# ========================================
# DYA Studio / Central 設定
# ========================================

# ZMK Studio 有効化
CONFIG_ZMK_STUDIO=y
CONFIG_ZMK_STUDIO_LOCKING=y
# 注意: CONFIG_ZMK_STUDIO_LOCKING=n を設定しないでください。
# Studio Locking は BLE 接続に必要なため、デフォルトの y を維持する必要があります。
# 詳細は「BLE 接続」セクションを参照。

# BLE 接続スロット（プロファイル + Peripheral + Studio）
CONFIG_BT_MAX_CONN=6
CONFIG_BT_MAX_PAIRED=6

# DYA Studio モジュール設定
CONFIG_ZMK_BLE_MANAGEMENT=y
CONFIG_ZMK_BLE_MANAGEMENT_STUDIO_RPC=y

CONFIG_ZMK_BATTERY_HISTORY=y
CONFIG_ZMK_BATTERY_HISTORY_STUDIO_RPC=y
# デフォルトでは USB 給電中はバッテリー履歴の記録がスキップされます。
# USB 接続中もバッテリー履歴を記録したい場合は n に設定してください。
CONFIG_ZMK_BATTERY_SKIP_IF_USB_POWERED=n

CONFIG_ZMK_SETTINGS_RPC=y
CONFIG_ZMK_SETTINGS_RPC_STUDIO=y

# 左右間通信 (Split Relay)
CONFIG_ZMK_SPLIT_RELAY_EVENT=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_SPLIT_RUN_STACK_SIZE=768

# Runtime Input Processor（トラックボール設定を DYA Studio から変更）
# トラックボール/ポインティングデバイスがある場合のみ必要
CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR=y
CONFIG_ZMK_RUNTIME_INPUT_PROCESSOR_STUDIO_RPC=y

# 設定の永続化
CONFIG_ZMK_SETTINGS_SAVE_DEBOUNCE=10000

# Enable the runtime sensor rotate module
CONFIG_ZMK_RUNTIME_SENSOR_ROTATE=y
CONFIG_ZMK_RUNTIME_SENSOR_ROTATE_STUDIO_RPC=y

# Enable settings storage (if not already enabled)
CONFIG_SETTINGS=y

CONFIG_ZMK_BATTERY_REPORTING=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=y
CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y
```

### `config/ZaruBall_left.conf` (Peripheral側)
Peripheral側でもバッテリー履歴や設定同期のためにいくつかの設定を有効にする必要があります。以下は左側用設定ファイルの全内容です。

```properties
CONFIG_EC11=y
CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y

# バッテリー履歴（Peripheral 側）
CONFIG_ZMK_BATTERY_HISTORY=y

# Settings RPC（Peripheral 側）
CONFIG_ZMK_SETTINGS_RPC=y

# Split イベントリレー（Peripheral 側）
CONFIG_ZMK_SPLIT_RELAY_EVENT=y

# 設定の永続化
CONFIG_ZMK_SETTINGS_SAVE_DEBOUNCE=10000
```

---
以上の変更を行い、ファームウェアをビルド・書き込みすることで、DYA Studio などの対応ツールから設定変更が可能になります。