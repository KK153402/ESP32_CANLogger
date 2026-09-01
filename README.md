# CanLoggerJig

PCとCANインタフェースを持ち込めない現場で、単独でCANログを取得するための治具。
**CAN1(500kbps) / CAN2(250kbps) の2バスを同時に**SDカードへ記録し、
PC側でBLF形式へ変換してCANalyzerで解析する。

```
測定対象のCANバス ──▶ 治具（SDカードへ記録）──▶ PCでBLF変換 ──▶ CANalyzerで解析
```

ファームウェア **v2.5** / ハードウェア **Rev.5**

---

## ドキュメント

| 目的 | ファイル |
|---|---|
| **使い方**（現場での操作手順） | [`docs/USAGE.md`](docs/USAGE.md) |
| **技術解説**（プログラムの中身、初心者向け） | [`docs/TECHNICAL.md`](docs/TECHNICAL.md) |
| **設計仕様**（試算・ファイル形式・実測結果） | [`docs/DESIGN.md`](docs/DESIGN.md) |
| **ハードウェア仕様**（BOM・ピンアサイン・査読結果） | [`docs/HARDWARE.md`](docs/HARDWARE.md) |

---

## 実績

実機での測定結果。

| 項目 | 結果 |
|---|---|
| CH1（500kbps） | 2,747フレーム / **取りこぼし0** |
| CH2（250kbps） | 54,263フレーム / **取りこぼし0** |
| 2ch同時記録 | 57,010フレーム / 59.16秒 |
| BLF変換 → CANalyzer再生 | **動作確認済み** |

---

## ハードウェア構成

```
                    ┌──── 機能絶縁境界（CH1）────┐
CN1 (CAN1 500k) ────┤ RESD1CANY → ISO1042DWV     ├─── ESP32 内蔵TWAI
                    │  NME1S0505SC + 100Ωダミー   │
                    └─────────────────────────────┘

                    ┌──── 機能絶縁境界（CH2）────┐
CN2 (CAN2 250k) ────┤ RESD1CANY → ISO1042DWV     ├─── MCP2515 (HSPI)
                    │  NME1S0505SC + 100Ωダミー   │
                    └─────────────────────────────┘

USB-C ─ TPD2E2U06 ─ MF-FSML100 ─ TPS562201 ─ 3.3V ─ ESP32-WROOM-32E-N16
                                                     ├── microSD (VSPI)
                                                     ├── MCP7940N (I2C) + CR2032
                                                     ├── 開始/停止SW、状態LED
                                                     └── CH340E (USB-UART)
```

| 主要部品 | 型番 |
|---|---|
| MCU | ESP32-WROOM-32E-N16 |
| CANトランシーバ | ISO1042DWV（絶縁）×2 |
| CANコントローラ（CH2） | MCP2515T-I/SO |
| 絶縁DC/DC | NME1S0505SC ×2 |
| RTC | MCP7940N-I/SN |
| 降圧コンバータ | TPS562201DDCR |
| USB-UART | CH340E |

詳細は [`docs/HARDWARE.md`](docs/HARDWARE.md)。

---

## ディレクトリ構成

```
CanLoggerJig/
├── platformio.ini          ビルド設定
├── src/
│   ├── main.cpp            状態機械、受信タスク、書き込みタスク、シリアルコマンド
│   ├── config.h            ピン配置・ビットレート・バッファサイズ
│   ├── ring_buffer.h       24byte固定長レコードとSPSCリングバッファ
│   ├── mcp2515.h           MCP2515ドライバ（Listen-Only、外部ライブラリ不要）
│   ├── mcp7940.h           MCP7940N RTCドライバ
│   └── asc_format.h        ASC形式の生成（ASCモード時のみ）
├── tools/
│   ├── bin_to_blf.py       BIN → BLF 変換。ファイル形式の仕様書も兼ねる
│   └── bin_to_blf.py.txt   同上（拡張子制限がある環境向け）
└── docs/
    ├── USAGE.md            操作手順書
    ├── TECHNICAL.md        技術解説
    ├── DESIGN.md           設計仕様
    └── HARDWARE.md         ハードウェア仕様
```

外部ライブラリへの依存はない（`SD` / `SPI` / `Wire` はフレームワーク同梱）。

---

## ビルドと書き込み

### 必要なもの

- [PlatformIO Core](https://docs.platformio.org/en/latest/core/installation/)
  または VSCode + PlatformIO IDE拡張

### 手順

```bash
cd CanLoggerJig

pio run                  # ビルド
pio run -t upload        # 書き込み
pio device monitor       # シリアルモニタ（115200）
```

初回は arduino-esp32 2.0.17 のツールチェーンを自動取得するため数分かかる。

> **書き込みは手動操作が必要。** USB-UARTにCH340E（DTR#なし）を使っているため自動リセットができない。
>
> **BOOTボタンを押しながらENボタンを押して離す → BOOTを離す → `pio run -t upload`**

### ビルド環境（env）

| env | 用途 | コマンド |
|---|---|---|
| `esp32dev` | 通常構成（CH1 + CH2 + RTC）。既定 | `pio run` |
| `ch1_only` | CH1のみ。MCP2515未実装基板の検証用 | `pio run -e ch1_only -t upload` |
| `no_rtc` | RTC無し。MCP7940N未実装基板の検証用 | `pio run -e no_rtc -t upload` |
| `asc_format` | ASCテキスト保存 | `pio run -e asc_format -t upload` |

`config.h` の主要な設定値には `#ifndef` ガードがあるので、
ソースを書き換えずに `build_flags` の `-D` で上書きできる。

```ini
build_flags = -DCH2_ENABLE=0 -DCH1_BITRATE=250000UL
```

### バージョン固定について

`platformio.ini` で `espressif32@6.8.1`（= arduino-esp32 2.0.17）を固定している。
本ファームは `driver/twai.h` を直接使っており、arduino-esp32 3.x では
API・ヘッダ構成が変わるためビルドが通らない。

### Arduino IDE で使う場合

`src/main.cpp` は `.ino` ではないためArduino IDEでは開けない。
使う場合は `main.cpp` を `CanLoggerJig/CanLoggerJig.ino` にリネームし、ヘッダ5つを同じフォルダへ置く。
**全ての関数を呼び出し箇所より前に定義してある**ので、どちらの形式でもビルドできる。
関数を追加する際はこの順序を崩さないこと。

---

## シリアルコマンド

```bash
pio device monitor
```

| コマンド | 内容 |
|---|---|
| `SETTIME YYYY-MM-DD HH:MM:SS` | RTCに現地時刻を設定（記録中は不可） |
| `GETTIME` | 現在のRTC時刻を表示 |
| `TRIM [-127..127]` | 発振周波数の補正値を表示／設定 |
| `STATUS` | 状態・fault・ヒープ残量・RTC診断フラグ |
| `HELP` | コマンド一覧 |

製作時に一度 `SETTIME` すれば、CR2032が切れるまで保持される。

```
> STATUS
state=IDLE  rtc=2026-09-01 15:01:50  fault=none  freeheap=216312
rtc flags: OSCRUN=1 PWRFAIL=0 VBATEN=1 (RTCWKDAY=0x2B)  trim=+0 (+0.0 ppm)
```

---

## PC側変換ツール

```bash
cd tools
pip install python-can

python bin_to_blf.py LOG0001.BIN -o LOG0001.blf
```

| オプション | 内容 |
|---|---|
| `-o` | 出力先。省略時は入力名 + `.blf` |
| `--start` | 計測開始の実時刻。省略時は「ヘッダの `unix_time` → それが0なら入力ファイルの更新時刻」の順 |
| `--tz-offset` | CANalyzer表示用の補正量[時間]。省略時はPCのタイムゾーン（日本なら+9） |
| `--utc` | 補正せずUTCのまま出力 |

> **タイムゾーン**: python-canはBLFヘッダへUTCを書くが、CANalyzerはそれを現地時刻として
> 表示するため、補正しないと9時間ずれる。既定で補正している（相対時刻は不変）。

BLF内のチャネル番号は 1 = CAN1 / 2 = CAN2 になり、CANalyzerの設定とそのまま一致する。

ASCモードで取得した場合は python-can 同梱のコンバータでも変換できる。

```bash
python -m can.logconvert LOG0001.ASC LOG0001.blf
```

---

## 設計上の要点

| 項目 | 内容 |
|---|---|
| **Listen-Only** | 両chともACKを返さず、バスに一切影響を与えない |
| **受信と書き込みの分離** | 受信タスクは打刻とリングへの積み込みのみ。SD書き込みは別コア |
| **リングバッファ** | SDカードの数百msの停止を吸収（実測では1024段中12段しか使用せず） |
| **バイナリ保存** | ASCテキストの約1/3のデータ量。SD速度の余裕を確保 |
| **縮退保存** | 異常時もそこまでのログと統計を必ず残す |
| **統計ファイル** | ログの信頼性を現地で判定できるようにする |

詳細な理由は [`docs/TECHNICAL.md`](docs/TECHNICAL.md)。

---

## 未了項目

**ファームウェア**

- [ ] RTCの校正（1号機は約+30ppm進む。`TRIM` で補正。[`docs/HARDWARE.md`](docs/HARDWARE.md) §11.6）
- [ ] RTC電池での時刻保持確認（電源断→復帰で `PWRFAIL=1` になること）
- [ ] ファイル自動分割（FAT32 4GB上限対策）

**ハードウェア（次版）**

- [ ] RTC水晶を **CL=6〜7pF品** へ変更（MCP7940Nは6〜9pFに最適化）
- [ ] 3.3Vバイパスコンデンサの配置割当確定（SDソケット直近を最優先）

**運用**

- [ ] 筐体設計
- [ ] 治具ラベル（「制御系CAN専用。高電圧バスに接続しないこと」）
