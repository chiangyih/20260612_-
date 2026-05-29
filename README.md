# ESP32 血氧濃度偵測計詳細教學（MAX30105 + OLED + 無源蜂鳴器）
-<img width="960" height="540" alt="image" src="https://chatgpt.com/s/m_6a18fc9aac048191a47b4af6da134b19" />
本教學使用以下元件製作血氧濃度（SpO2）與心跳（BPM）顯示器，適合國中生到校試探課程：

- ESP32 開發板
- <img width="292" height="642" alt="image" src="https://github.com/user-attachments/assets/32dbcb40-8df6-4702-9ae6-f06a0496e881" />

- MAX30105 感測器
- <img width="349" height="533" alt="image" src="https://github.com/user-attachments/assets/05533de5-9eef-434a-85d4-ec18f47e1dbc" />

- 0.96 吋 128x64 I2C OLED（位址 `0x3C`）
- <img width="470" height="482" alt="image" src="https://github.com/user-attachments/assets/f64dc1ef-f2ab-4fcc-b0ec-9d9b145b6452" />

- 小型無源蜂鳴器（buzzer）

專案內現有檔案：

- 程式碼：[main.cpp](D:/OneDrive/00_待辦工作與計畫/20260612-國中生到校試探-血氧偵測計_esp32/main.cpp)
- 電路圖（Fritzing）：[Oxygen_Saturation.fzz](D:/OneDrive/00_待辦工作與計畫/20260612-國中生到校試探-血氧偵測計_esp32/Oxygen_Saturation.fzz)
<img width="814" height="760" alt="image" src="https://github.com/user-attachments/assets/78a6fb26-babd-47d7-b7d1-a85cf856c0e5" />

- ESP32 針腳圖：[esp32_pin_out.png](D:/OneDrive/00_待辦工作與計畫/20260612-國中生到校試探-血氧偵測計_esp32/esp32_pin_out.png)
<img width="930" height="447" alt="image" src="https://github.com/user-attachments/assets/0c66577f-082f-42a8-a827-15bc3fa19764" />

---

## 1. 課程目標

1. 認識血氧與心跳量測的基本概念。  
2. 學會用 I2C 匯流排同時連接 MAX30105 與 OLED。  
3. 能讀懂並修改 ESP32 程式，完成即時顯示與蜂鳴提示。  
4. 能說明為何量測時要「手指穩定、減少晃動」。

---

## 2. 元件簡介

## 2.1 ESP32

- 一顆常用的 Wi-Fi/Bluetooth 微控制器。  
- 本專案主要使用它的：
  - `I2C` 通訊（連接 MAX30105、OLED）
  - `GPIO` 輸出（驅動蜂鳴器）
  - 足夠的運算能力做即時訊號處理

## 2.2 MAX30105 感測器

- 內含紅光/紅外光 LED 與光感測器。  
- 可讀取手指血液容積變化，推估心跳。  
- 透過紅光與紅外光訊號比例（R 值）估算 SpO2。  
- 程式中使用 `getFIFORed()`、`getFIFOIR()` 取得原始資料。

## 2.3 0.96 吋 128x64 I2C OLED

- 用來顯示 SpO2、BPM 與提示文字。  
- 優點：不需背光、對比高、接線少（SDA/SCL 兩條資料線）。

## 2.4 小型無源蜂鳴器

- 每次偵測到有效心跳時短鳴。  
- 程式使用 `tone(Tonepin, 1000, 10)` 產生 1kHz、10ms 音。

---

## 3. 接線說明（依電路圖）

請以你的 Fritzing 檔為主：  
[Oxygen_Saturation.fzz](D:/OneDrive/00_待辦工作與計畫/20260612-國中生到校試探-血氧偵測計_esp32/Oxygen_Saturation.fzz)

本程式可確定的關鍵腳位如下（依 `esp32_實體.png` 板上絲印）：

- 蜂鳴器訊號腳：`GPIO 4`（`Tonepin = 4`）
- I2C 匯流排：使用板上的 `SDA`、`SCL` 腳位
- OLED 位址：`0x3C`

一般接線原則：

1. MAX30105 `VIN/3V3` 接 ESP32 `3.3V`、`GND` 接 `GND`。  
2. MAX30105 `SDA`、`SCL` 與 OLED `SDA`、`SCL` 並聯到 ESP32 I2C。  
3. 無源蜂鳴器 `+` 接 `GPIO 4`，`-` 接 `GND`。  

## 3.1 元件與 ESP32 接腳對應表

> 以下對應以目前程式（`Wire` + `Tonepin=4`）與 `esp32_實體.png` 的板上絲印為準。

| 元件 | 元件腳位 | ESP32 腳位 | 說明 |
|---|---|---|---|
| MAX30105 | VIN / 3V3 | 3V3 | 感測器供電 |
| MAX30105 | GND | GND | 地線（需與所有模組共地） |
| MAX30105 | SDA | SDA（板上絲印） | I2C 資料線 |
| MAX30105 | SCL | SCL（板上絲印） | I2C 時脈線 |
| OLED 0.96" | VCC | 3V3 | 螢幕供電 |
| OLED 0.96" | GND | GND | 地線 |
| OLED 0.96" | SDA | SDA（板上絲印） | 與 MAX30105 並聯同一組 I2C |
| OLED 0.96" | SCL | SCL（板上絲印） | 與 MAX30105 並聯同一組 I2C |
| 無源蜂鳴器 | 正極（+） | GPIO 4 | 程式 `tone(Tonepin, ...)` 控制 |
| 無源蜂鳴器 | 負極（-） | GND | 地線 |

補充：

- 你的實體板右側可直接看到 `SDA`、`SCL` 絲印，教學接線請優先對照這兩個腳位。  
- 若日後改用其他 ESP32 板型，可用 `Wire.begin(SDA, SCL)` 指定 I2C 腳位。  
- OLED 與 MAX30105 都是 I2C 裝置，可共用同一組 SDA/SCL。  

---

## 4. 開發環境與函式庫

可用 Arduino IDE 或 PlatformIO，需安裝：

- `Adafruit GFX Library`
- `Adafruit SSD1306`
- `SparkFun MAX3010x Sensor Library`（提供 `MAX30105.h`）

程式主要 include：

```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "MAX30105.h"
#include "heartRate.h"
```

---

## 5. 完整程式碼（與專案相同）

來源檔案：  
[main.cpp](D:/OneDrive/00_待辦工作與計畫/20260612-國中生到校試探-血氧偵測計_esp32/main.cpp)

```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "MAX30105.h"
#include "heartRate.h"


#ifndef IRAM_ATTR
#define IRAM_ATTR
#endif

//螢幕
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

//初始
const int Tonepin = 4; 
MAX30105 particleSensor;
bool oledReady = false;
bool sensorReady = false;

// 血氧
#define FINGER_ON 7000          
#define MINIMUM_SPO2 90.0   

const byte RATE_SIZE = 8;
byte rates[RATE_SIZE];
byte rateSpot = 0;
byte validRateCount = 0;
long lastBeat = 0;
float beatsPerMinute = 0;
int beatAvg = 0;

int sampleCount = 0;
const int numSamples = 30;
double avered = 0, aveir = 0;
double sumirrms = 0, sumredrms = 0;
double SpO2 = 0, ESpO2 = 90.0;
const double FSpO2 = 0.7;
const double frate = 0.95;

unsigned long lastDisplayUpdate = 0;
const unsigned long DISPLAY_INTERVAL = 250; // 刷新250ms

// 工具
void resetReadings() {
  for (byte i = 0; i < RATE_SIZE; i++) rates[i] = 0;
  rateSpot = 0;
  validRateCount = 0;
  lastBeat = 0;
  beatsPerMinute = 0;
  beatAvg = 0;

  sampleCount = 0;
  avered = 0;
  aveir = 0;
  sumirrms = 0;
  sumredrms = 0;
  SpO2 = 0;
  ESpO2 = 90.0;
}

void printCentered(const char *text, int y, int textSize) {
  display.setTextSize(textSize);
  int16_t x1, y1;
  uint16_t w, h;
  display.getTextBounds(text, 0, y, &x1, &y1, &w, &h);
  display.setCursor((SCREEN_WIDTH - w) / 2, y);
  display.print(text);
}

void drawBootScreen(const char *line1, const char *line2) {
  if (!oledReady) return;
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  printCentered(line1, 16, 1);
  printCentered(line2, 34, 1);
  display.display();
}

void drawMainScreen(bool fingerOn) {
  if (!oledReady) return;

  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextWrap(false);

  if (!fingerOn) {
    printCentered("PULSE OXIMETER", 2, 1);
    display.drawFastHLine(0, 13, 128, SSD1306_WHITE);
    printCentered("PLACE FINGER", 26, 1);
    printCentered("ON SENSOR", 42, 1);
    display.display();
    return;
  }

  display.setTextSize(1);
  display.setCursor(0, 0);
  display.print("SpO2");
  display.setCursor(93, 0);
  display.print("BPM");
  display.drawFastHLine(0, 10, 128, SSD1306_WHITE);

  display.setTextSize(3);
  display.setCursor(0, 20);
  if (beatAvg > 30 && ESpO2 >= 90.0) {
    display.print((int)(ESpO2 + 0.5));
  } else {
    display.print("--");
  }

  display.setTextSize(2);
  display.setCursor(48, 28);
  display.print("%");

  display.drawFastVLine(70, 12, 52, SSD1306_WHITE);
  display.setTextSize(3);
  display.setCursor(78, 20);
  if (beatAvg > 0) {
    if (beatAvg < 100) display.print(" ");
    display.print(beatAvg);
  } else {
    display.print("--");
  }


  display.setTextSize(1);
  display.setCursor(0, 56);
  if (beatAvg > 30) {
    display.print("Keep still");
  } else {
    display.print("Measuring...");
  }


  if (millis() - lastBeat < 180) {
    display.fillCircle(122, 58, 3, SSD1306_WHITE);
  } else {
    display.drawCircle(122, 58, 3, SSD1306_WHITE);
  }

  display.display();
}

void setup() {
  Serial.begin(115200); 

  oledReady = display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR);
  if (oledReady) {
    display.clearDisplay();
    display.setTextColor(SSD1306_WHITE);
    drawBootScreen("Hello", "Starting...");
  } else {
    Serial.println("OLED not found. Check I2C address/wiring.");
  }

  if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) {
    Serial.println("MAX30105 was not found. Check wiring/power.");
    drawBootScreen("SENSOR ERROR", "CHECK WIRING");
    return;
  }

  sensorReady = true;

  // ledBrightness, sampleAverage, ledMode, sampleRate, pulseWidth, adcRange
  particleSensor.setup(0x7F, 4, 2, 800, 215, 16384);
  particleSensor.enableDIETEMPRDY();
  particleSensor.setPulseAmplitudeRed(0x24);
  particleSensor.setPulseAmplitudeIR(0x24);
  particleSensor.setPulseAmplitudeGreen(0);

  delay(800);
  drawMainScreen(false);
}

// mainloop
void loop() {
  if (!sensorReady) {
    delay(1000);
    return;
  }

  long irValue = particleSensor.getIR();
  bool fingerOn = (irValue > FINGER_ON);

  if (fingerOn) {
    if (checkForBeat(irValue)) {
      long delta = millis() - lastBeat;
      lastBeat = millis();

      beatsPerMinute = 60.0 / (delta / 1000.0);

      if (beatsPerMinute > 20 && beatsPerMinute < 255) {
        rates[rateSpot++] = (byte)beatsPerMinute;
        rateSpot %= RATE_SIZE;
        if (validRateCount < RATE_SIZE) validRateCount++;

        int total = 0;
        for (byte i = 0; i < validRateCount; i++) total += rates[i];
        beatAvg = total / validRateCount;

        tone(Tonepin, 1000, 10);
      }
    }

    
    particleSensor.check();
    while (particleSensor.available()) {
      uint32_t red = particleSensor.getFIFORed();
      uint32_t ir = particleSensor.getFIFOIR();

      double fred = (double)red;
      double fir = (double)ir;

      avered = avered * frate + fred * (1.0 - frate);
      aveir = aveir * frate + fir * (1.0 - frate);

      sumredrms += (fred - avered) * (fred - avered);
      sumirrms += (fir - aveir) * (fir - aveir);

      sampleCount++;
      if (sampleCount >= numSamples) {
        if (avered > 0 && aveir > 0 && sumirrms > 0) {
          double R = (sqrt(sumredrms) / avered) / (sqrt(sumirrms) / aveir);
          SpO2 = -23.3 * (R - 0.4) + 100.0;
          ESpO2 = FSpO2 * ESpO2 + (1.0 - FSpO2) * SpO2;

          if (ESpO2 <= MINIMUM_SPO2) ESpO2 = MINIMUM_SPO2;
          if (ESpO2 > 100.0) ESpO2 = 99.9;
        }

        sumredrms = 0.0;
        sumirrms = 0.0;
        sampleCount = 0;
      }

      particleSensor.nextSample();
    }
  } else {
    resetReadings();
  }

  if (millis() - lastDisplayUpdate >= DISPLAY_INTERVAL) {
    drawMainScreen(fingerOn);
    lastDisplayUpdate = millis();
  }
}
```

---

## 5.1 加註解版程式碼（教學對照）

以下為已加上詳細正體中文註解的版本，與專案目前 `main.cpp` 同步，方便學生逐段閱讀：

```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "MAX30105.h"
#include "heartRate.h"

// 某些平台/核心版本可能未定義 IRAM_ATTR，先做相容性處理
#ifndef IRAM_ATTR
#define IRAM_ATTR
#endif

// =========================
// OLED 顯示器參數設定
// =========================
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define OLED_ADDR 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// =========================
// 硬體物件與狀態旗標
// =========================
// 無源蜂鳴器接在 GPIO4（每次偵測到有效心跳時短鳴）
const int Tonepin = 4; 
// MAX30105 感測器物件
MAX30105 particleSensor;
// OLED 是否初始化成功
bool oledReady = false;
// 感測器是否初始化成功
bool sensorReady = false;

// =========================
// 量測參數與演算法變數
// =========================
// IR 值高於此門檻，視為「手指已放上感測器」
#define FINGER_ON 7000          
// 顯示上的最低血氧下限（避免不合理低值造成教學干擾）
#define MINIMUM_SPO2 90.0   

// 心跳移動平均緩衝區大小（取最近 8 次有效 BPM）
const byte RATE_SIZE = 8;
byte rates[RATE_SIZE];
byte rateSpot = 0;        // 寫入緩衝區的位置
byte validRateCount = 0;  // 目前已有多少有效心跳樣本（最多 RATE_SIZE）
long lastBeat = 0;        // 上一次偵測到心跳的時間戳（ms）
float beatsPerMinute = 0; // 單次計算出的 BPM
int beatAvg = 0;          // 平滑後（移動平均）的 BPM

// SpO2 計算使用的取樣與統計變數
int sampleCount = 0;
const int numSamples = 30; // 累積 30 筆再計算一次 SpO2
double avered = 0, aveir = 0;
double sumirrms = 0, sumredrms = 0;
double SpO2 = 0, ESpO2 = 90.0;
const double FSpO2 = 0.7; // SpO2 顯示平滑係數（越大越穩、反應越慢）
const double frate = 0.95; // Red/IR 基線（DC）指數平滑係數

// 螢幕更新節流（避免過度刷新）
unsigned long lastDisplayUpdate = 0;
const unsigned long DISPLAY_INTERVAL = 250; // 刷新250ms

// =========================
// 工具函式
// =========================
// 重置所有量測狀態（手指離開感測器時呼叫）
void resetReadings() {
  for (byte i = 0; i < RATE_SIZE; i++) rates[i] = 0;
  rateSpot = 0;
  validRateCount = 0;
  lastBeat = 0;
  beatsPerMinute = 0;
  beatAvg = 0;

  sampleCount = 0;
  avered = 0;
  aveir = 0;
  sumirrms = 0;
  sumredrms = 0;
  SpO2 = 0;
  ESpO2 = 90.0;
}

// 置中顯示文字（方便畫面排版）
void printCentered(const char *text, int y, int textSize) {
  display.setTextSize(textSize);
  int16_t x1, y1;
  uint16_t w, h;
  display.getTextBounds(text, 0, y, &x1, &y1, &w, &h);
  display.setCursor((SCREEN_WIDTH - w) / 2, y);
  display.print(text);
}

// 開機/錯誤提示畫面
void drawBootScreen(const char *line1, const char *line2) {
  if (!oledReady) return;
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  printCentered(line1, 16, 1);
  printCentered(line2, 34, 1);
  display.display();
}

// 主畫面繪製：依「是否有手指」切換顯示內容
void drawMainScreen(bool fingerOn) {
  if (!oledReady) return;

  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextWrap(false);

  // 未放手指：顯示提示文字
  if (!fingerOn) {
    printCentered("PULSE OXIMETER", 2, 1);
    display.drawFastHLine(0, 13, 128, SSD1306_WHITE);
    printCentered("PLACE FINGER", 26, 1);
    printCentered("ON SENSOR", 42, 1);
    display.display();
    return;
  }

  display.setTextSize(1);
  display.setCursor(0, 0);
  display.print("SpO2");
  display.setCursor(93, 0);
  display.print("BPM");
  display.drawFastHLine(0, 10, 128, SSD1306_WHITE);

  // 左側顯示 SpO2：當心跳平均值足夠且血氧估算有效時才顯示數字
  display.setTextSize(3);
  display.setCursor(0, 20);
  if (beatAvg > 30 && ESpO2 >= 90.0) {
    display.print((int)(ESpO2 + 0.5));
  } else {
    display.print("--");
  }

  display.setTextSize(2);
  display.setCursor(48, 28);
  display.print("%");

  display.drawFastVLine(70, 12, 52, SSD1306_WHITE);
  display.setTextSize(3);
  display.setCursor(78, 20);
  if (beatAvg > 0) {
    if (beatAvg < 100) display.print(" ");
    display.print(beatAvg);
  } else {
    display.print("--");
  }

  // 底部狀態提示：心跳剛開始抓到前顯示 Measuring，穩定後提醒保持不動
  display.setTextSize(1);
  display.setCursor(0, 56);
  if (beatAvg > 30) {
    display.print("Keep still");
  } else {
    display.print("Measuring...");
  }

  // 右下角心跳指示點：在最近一次心跳後 180ms 內填滿，否則只畫空心圓
  if (millis() - lastBeat < 180) {
    display.fillCircle(122, 58, 3, SSD1306_WHITE);
  } else {
    display.drawCircle(122, 58, 3, SSD1306_WHITE);
  }

  display.display();
}

// =========================
// 系統初始化
// =========================
void setup() {
  Serial.begin(115200); 

  // 初始化 OLED，失敗時改用序列埠提示
  oledReady = display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR);
  if (oledReady) {
    display.clearDisplay();
    display.setTextColor(SSD1306_WHITE);
    drawBootScreen("Hello", "Starting...");
  } else {
    Serial.println("OLED not found. Check I2C address/wiring.");
  }

  // 初始化 MAX30105，失敗則顯示錯誤並停止後續量測
  if (!particleSensor.begin(Wire, I2C_SPEED_FAST)) {
    Serial.println("MAX30105 was not found. Check wiring/power.");
    drawBootScreen("SENSOR ERROR", "CHECK WIRING");
    return;
  }

  sensorReady = true;

  // 感測器設定：
  // setup(ledBrightness, sampleAverage, ledMode, sampleRate, pulseWidth, adcRange)
  // ledMode=2 代表使用 Red + IR（血氧估算需要）
  particleSensor.setup(0x7F, 4, 2, 800, 215, 16384);
  particleSensor.enableDIETEMPRDY();
  // Red/IR LED 強度（可依手指反射狀況微調）
  particleSensor.setPulseAmplitudeRed(0x24);
  particleSensor.setPulseAmplitudeIR(0x24);
  // 關閉 Green LED，避免不必要耗電/干擾
  particleSensor.setPulseAmplitudeGreen(0);

  delay(800);
  drawMainScreen(false);
}

// =========================
// 主迴圈
// =========================
void loop() {
  // 若初始化失敗，避免空跑
  if (!sensorReady) {
    delay(1000);
    return;
  }

  // 先用 IR 值判斷是否有手指接觸
  long irValue = particleSensor.getIR();
  bool fingerOn = (irValue > FINGER_ON);

  if (fingerOn) {
    // ---------- 心跳偵測與 BPM 計算 ----------
    if (checkForBeat(irValue)) {
      long delta = millis() - lastBeat;
      lastBeat = millis();

      // BPM = 60 / 心跳週期（秒）
      beatsPerMinute = 60.0 / (delta / 1000.0);

      // 過濾不合理區間，避免雜訊誤判
      if (beatsPerMinute > 20 && beatsPerMinute < 255) {
        // 寫入環狀緩衝區
        rates[rateSpot++] = (byte)beatsPerMinute;
        rateSpot %= RATE_SIZE;
        if (validRateCount < RATE_SIZE) validRateCount++;

        // 計算最近幾次有效心跳平均（平滑 BPM）
        int total = 0;
        for (byte i = 0; i < validRateCount; i++) total += rates[i];
        beatAvg = total / validRateCount;

        // 每次有效心跳短鳴提示
        tone(Tonepin, 1000, 10);
      }
    }

    // ---------- 讀取 FIFO 並估算 SpO2 ----------
    // check() 會更新內部 FIFO 緩衝狀態
    particleSensor.check();
    while (particleSensor.available()) {
      // 讀出 Red / IR 原始光訊號
      uint32_t red = particleSensor.getFIFORed();
      uint32_t ir = particleSensor.getFIFOIR();

      double fred = (double)red;
      double fir = (double)ir;

      avered = avered * frate + fred * (1.0 - frate);
      aveir = aveir * frate + fir * (1.0 - frate);

      // 累積 AC 能量（以偏離基線的平方和近似）
      sumredrms += (fred - avered) * (fred - avered);
      sumirrms += (fir - aveir) * (fir - aveir);

      sampleCount++;
      if (sampleCount >= numSamples) {
        // 防止除以零與異常值
        if (avered > 0 && aveir > 0 && sumirrms > 0) {
          // Ratio-of-ratios：
          // R = (ACred/DCred) / (ACir/DCir)
          double R = (sqrt(sumredrms) / avered) / (sqrt(sumirrms) / aveir);
          // 經驗公式估算 SpO2（教學示範用）
          SpO2 = -23.3 * (R - 0.4) + 100.0;
          // 輸出再做一次平滑，降低跳動
          ESpO2 = FSpO2 * ESpO2 + (1.0 - FSpO2) * SpO2;

          // 限制顯示範圍，避免出現不合理值
          if (ESpO2 <= MINIMUM_SPO2) ESpO2 = MINIMUM_SPO2;
          if (ESpO2 > 100.0) ESpO2 = 99.9;
        }

        // 重置本輪 RMS 累積，開始下一批樣本
        sumredrms = 0.0;
        sumirrms = 0.0;
        sampleCount = 0;
      }

      // 移到 FIFO 下一筆
      particleSensor.nextSample();
    }
  } else {
    // 手指離開後，清空量測狀態，避免舊值殘留
    resetReadings();
  }

  // 控制更新頻率（每 250ms 更新一次畫面）
  if (millis() - lastDisplayUpdate >= DISPLAY_INTERVAL) {
    drawMainScreen(fingerOn);
    lastDisplayUpdate = millis();
  }
}
```

---

## 6. 程式碼重點解說

## 6.1 顯示器與硬體初始化

- `display.begin(..., 0x3C)`：初始化 OLED。  
- `particleSensor.begin(Wire, I2C_SPEED_FAST)`：初始化 MAX30105。  
- 若感測器初始化失敗，螢幕顯示 `SENSOR ERROR`，並停止正常量測流程。

## 6.2 心跳偵測（BPM）

- 先讀取 `irValue = particleSensor.getIR()`。  
- `checkForBeat(irValue)` 偵測到心搏時，計算兩次心搏時間差 `delta`。  
- `BPM = 60 / 秒數`。  
- 以 `rates[8]` 做移動平均，降低單次抖動。  
- 每次有效心跳都觸發蜂鳴器短鳴。

## 6.3 血氧估算（SpO2）

程式流程：

1. 讀取 Red/IR FIFO 原始值。  
2. 用 `frate=0.95` 做指數移動平均，得到 DC 成分（`avered`、`aveir`）。  
3. 累積訊號偏移平方和（近似 AC 能量）成為 `sumredrms`、`sumirrms`。  
4. 每 30 筆樣本 (`numSamples`) 計算一次：  
   - `R = (ACred/DCred) / (ACir/DCir)`  
   - `SpO2 = -23.3 * (R - 0.4) + 100`  
5. 再用 `FSpO2=0.7` 平滑成 `ESpO2`，讓顯示更穩定。  

## 6.4 手指偵測與畫面狀態

- 判斷條件：`irValue > FINGER_ON`（門檻為 `7000`）。  
- 無手指時顯示 `PLACE FINGER ON SENSOR`，並重置統計值。  
- 有手指時顯示 SpO2、BPM，右下角小圓點以心跳節奏閃爍。

## 6.5 重要參數可調整建議

- `FINGER_ON`：手指是否接觸的門檻值。  
- `numSamples`：每次血氧計算使用樣本數。  
- `DISPLAY_INTERVAL`：畫面更新頻率（目前 250ms）。  
- `particleSensor.setup(...)`：感測器取樣率、脈衝寬度、ADC 範圍等設定。

---

## 7. 課堂操作步驟（建議）

1. 完成接線，確認 `GND` 共地。  
2. 上傳程式到 ESP32。  
3. 開啟序列埠監控（115200）觀察錯誤訊息。  
4. 手指輕放 MAX30105（勿過度施壓），保持 10~20 秒穩定。  
5. 觀察 OLED 的 SpO2 與 BPM 是否逐漸穩定。  
6. 試著調整 `FINGER_ON` 與 `numSamples` 比較反應速度與穩定性。

---

## 8. 常見問題排除

## 8.1 OLED 無顯示

- 檢查位址是否為 `0x3C`。  
- 檢查 SDA/SCL 是否接對。  
- 檢查電壓是否符合模組規格。

## 8.2 MAX30105 找不到

- 確認 `VIN/3V3`、`GND`、`SDA`、`SCL`。  
- 若線太長或接觸不良，I2C 會不穩。  
- 先只接 MAX30105 測試，再加入 OLED。

## 8.3 數值亂跳

- 手指需固定，避免晃動。  
- 感測器與手指接觸要穩定。  
- 可提高 `numSamples` 或調整平滑係數，但反應會變慢。

---

## 9. 教學延伸活動

1. 加入「過低血氧警報」邏輯（例如 SpO2 < 94 時連續蜂鳴）。  
2. 把資料透過 Wi-Fi 上傳到網頁儀表板。  
3. 增加按鈕切換顯示模式（即時值/平均值）。

---

## 10. 安全與教學提醒

- 這是教學型專題，不可作為醫療診斷。  
- 不要以單次讀值作結論，需多次量測與交叉比對。  
- 課堂上應強調「訊號品質」對量測結果的影響。
