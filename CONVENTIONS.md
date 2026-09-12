# Waveshare ESP32-S3 e-Paper 開発規約

## 1. ターゲットハードウェア
- **MCU**: ESP32-S3 (Dual-core Xtensa LX7, 240MHz)
- **Framework**: ESP-IDF (v5.x 系 C99/C++17)
- **Target Display**: Waveshare e-Paper HAT (3.94 / 3.97 インチ系)
- **Host Controller**: SPI2_HOST (FSPI)

## 2. GPIO ピンアサイン定義
ボードの標準結線に従い、ピン定義は `main/epd_pins.h` 等で一元管理すること。
ハードコードは禁止。

| 信号名 | GPIO 番号 | 機能説明 |
|---|---|---|
| EPD_MOSI (DIN) | GPIO 11 | SPI Master Out Slave In |
| EPD_SCK (CLK) | GPIO 12 | SPI シリアルクロック (最大 10MHz 推奨) |
| EPD_CS | GPIO 10 | チップセレクト (Low Active) |
| EPD_DC | GPIO 9 | Data / Command 制御 (High: Data, Low: Command) |
| EPD_RST | GPIO 8 | ハードウェアリセット (Low Active) |
| EPD_BUSY | GPIO 7 | ビジー状態検出 (High: Busy, Low: Idle ※IC仕様による) |
| EPD_PWR | GPIO 6 | 電源制御スイッチ (存在する場合。High で電源供給) |

## 3. ドライバ実装ルール

### SPI 通信
- `driver/spi_master.h` の標準 API を使用すること。
- トランスファーは DMA を有効化 (`SPI_DMA_CH_AUTO`) して帯域を確保すること。
- コマンド送信時とデータ送信時で DC ピンを確実に切り替えること。

### FreeRTOS & Busy 待機
- BUSY ピンの待機処理で `while(gpio_get_level(...));` のような単なるビジーウェイトを禁止する（Task Watchdog Timer 発報防止）。
- ポーリング待機時は必ず `vTaskDelay(pdMS_TO_TICKS(10));` を挟むか、GPIO 割り込みと FreeRTOS セマフォ/イベントグループを組み合わせて実装すること。

### メモリ・バッファ管理
- 解像度に応じたフレームバッファ（1 ピクセルあたり 1 ビットまたは 2 ビット）を確保する際、PSRAM（SPIRAM）が有効化されている場合は `heap_caps_malloc(size, MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT)` を優先使用すること。
- PSRAM が無効または小容量の場合は `heap_caps_malloc(size, MALLOC_CAP_DMA | MALLOC_CAP_8BIT)` を使用し、バッファ枯渇を `ESP_ERROR_CHECK` で検知すること。

### 電源・寿命保護
- 描画更新完了後は、必ずスリープコマンド（Deep Sleep モード）を送信し、電子ペーパーパネルに直流電圧がかかり続けないようにすること。

## 4. ビルド & 動作検証コマンド
- **コンパイル検証**: `idf.py build`
- **シンボル更新**: `idf.py reconfigure && ln -sf build/compile_commands.json compile_commands.json`
- **書き込み & ログ**: `idf.py -p /dev/ttyACM0 flash monitor`
