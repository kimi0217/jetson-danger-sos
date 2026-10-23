# jetson-danger-sos

在 NVIDIA Jetson Nano 上以 YOLOv5 即時偵測刀具與剪刀，偵測到危險物品時透過蜂鳴器與 LED 發出摩斯電碼 SOS（`··· ——— ···`），並在 I2C LCD 上顯示警告訊息。

---

## 專案動機

近年持刀攻擊事件頻傳，公共場所安全備受關注。本系統透過即時影像辨識偵測危險物品，並以畫面、蜂鳴器、LED、LCD 四種回饋即時發出警示，適用於公共場所、幼兒園等安全需求較高、且可能吵雜或視線不佳的環境。

---

## Demo

![偵測結果](docs/images/demo.png)

*偵測結果（刀子、剪刀、尖嘴鉗被辨識為剪刀、裝飾短劍）*

▶ Demo 影片：<https://youtu.be/uQD8-AqvvYM>

---

## 我的角色

本專案為成功大學工程科學系「網路嵌入式系統應用」課程期末專題（2025，第 19 組，3 人小組）。

我負責全部軟體的設計與實作，包含 YOLOv5 偵測流程、OpenCV 影像處理、GPIO/LCD 警示邏輯與系統整合，並撰寫專題報告；組員協助硬體接線。

---

## 技術堆疊

| 元件 | 用途 |
|---|---|
| [PyTorch](https://pytorch.org/) | 深度學習執行環境（Jetson 上使用 CUDA） |
| [YOLOv5](https://github.com/ultralytics/yolov5)（`yolov5s`，v7.0） | COCO 預訓練物件偵測模型，透過 `torch.hub` 載入 |
| [OpenCV](https://opencv.org/) | 攝影機擷取、繪製偵測框、JPEG 編碼 |
| [Jetson.GPIO](https://github.com/NVIDIA/jetson-gpio) | 控制蜂鳴器與 LED |
| [RPLCD](https://github.com/dbrgn/RPLCD) + smbus2 | 透過 I2C 控制 16x2 字元 LCD |
| ipywidgets | 在 Jupyter 中即時顯示影像 |

偵測目標類別（COCO 索引）：`43` = knife（刀子）、`76` = scissors（剪刀）。

---

## 硬體

- 開發板：NVIDIA Jetson Nano（遠端 JupyterLab 開發環境）
- Webcam
- 蜂鳴器
- LED
- 16x2 LCD（PCF8574 I2C）

### 接線

GPIO 編號採用 **`GPIO.BOARD`**（40-pin 排針的實體腳位編號）。

| 裝置 | 連接 |
|---|---|
| 蜂鳴器 | **Pin 33** |
| LED | **Pin 7** |
| LCD（PCF8574） | I2C 位址 `0x27`，I2C bus 1 |

---

## 安裝

```bash
pip install ipywidgets
pip install seaborn
pip install RPLCD smbus2
```

---

## 使用方式

1. 依上方接線表接好硬體，並插上 Webcam。
2. 開啟 Jetson Nano 的遠端 JupyterLab。

3. 開啟 `danger_detection.ipynb`，依序執行各 cell：
   1. 安裝額外套件（`ipywidgets`、`seaborn`、`RPLCD`、`smbus2`）
   2. 在 `cuda:0` 上載入 YOLOv5s 模型
   3. 初始化 LCD 與 GPIO，並啟動偵測迴圈
4. 攝影機畫面會即時顯示在 notebook 中，將刀子或剪刀放到鏡頭前即可觸發警報。
5. 要停止程式時，點選 **Kernel → Interrupt Kernel**。`finally` 區塊會釋放攝影機、將 GPIO 腳位設為低電位、執行 `GPIO.cleanup()`，並在 LCD 顯示 `System Off`。

---

## 系統流程

```
┌────────────┐   影像     ┌──────────────┐   偵測結果   ┌──────────────────────┐
│   Webcam   │ ─────────▶ │ YOLOv5s (GPU)│ ───────────▶ │ 類別 ∈ {knife,       │
└────────────┘  640×480   └──────────────┘              │ scissors} 且 conf>0.1│
      ▲                                                 └─────────┬────────────┘
      │                                              否 ◀─────────┤ 是
      │                                    （更新畫面）           ▼
      │                                         ┌───────────────────────────────┐
      │                                         │ 1. 畫紅框並顯示 "SOS !!!"     │
      │                                         │ 2. LCD："Danger Detected!"    │
      │                                         │         "Status: SOS"         │
      │                                         │ 3. 蜂鳴器 + LED：··· ——— ···  │
      │                                         │ 4. 冷卻 3 秒，清除 LCD        │
      │                                         │ 5. 丟棄 5 張緩衝畫面          │
      └──────────────────────────────────────── └───────────────────────────────┘
```

1. **擷取：** OpenCV 從攝影機 0 讀取 640×480 的影像，並由 BGR 轉為 RGB。
2. **偵測：** YOLOv5s 在 GPU 上推論，逐一比對偵測結果是否屬於 `TARGET_CLASSES = [43, 76]`，且信心值大於 `0.1`。
3. **警示：** 偵測到危險物品時，在畫面上標註並更新到 notebook，LCD 顯示警告訊息，`sos()` 同時驅動蜂鳴器與 LED：
   - 時間單位 `UNIT = 0.2 s`
   - 點（dot）亮 1 個單位、劃（dash）亮 3 個單位，符號之間間隔 1 個單位
   - 字母之間（S · O · S）額外間隔 2 個單位
4. **恢復：** 冷卻 3 秒後清除 LCD，並丟棄 5 張緩衝畫面，讓下一次推論使用最新影像，接著回到迴圈。
5. **關閉：** 中斷程式時，所有硬體資源都會被正確釋放。

---

## 專案結構

```
jetson-danger-sos/
├── danger_detection.ipynb   # 主程式：載入模型、硬體初始化、偵測迴圈
├── docs/
│   └── images/
│       └── demo.png         # Demo 偵測結果截圖
├── .gitignore
└── README.md
```

