# jetson-danger-sos

Real-time dangerous object detection on NVIDIA Jetson. When a **knife** or **scissors** shows up in the camera feed, the system flashes an LED and sounds a buzzer in **Morse code SOS** (`··· ——— ···`) and shows a warning on an I2C LCD.

Final project for the **Networked Embedded Systems Applications** (網路嵌入式系統應用) course at **National Cheng Kung University (NCKU)**.

> **中文簡介**
> 本專案為成功大學「網路嵌入式系統應用」課程期末專案。系統在 NVIDIA Jetson 上以 PyTorch YOLOv5s 即時偵測攝影機畫面中的刀具與剪刀，一旦偵測到危險物品，便透過蜂鳴器與 LED 發出摩斯電碼 SOS 警報，並在 I2C LCD 上顯示警告訊息。

---

## Tech Stack

| Component | Purpose |
|---|---|
| [PyTorch](https://pytorch.org/) | Deep learning runtime (CUDA on Jetson) |
| [YOLOv5](https://github.com/ultralytics/yolov5) (`yolov5s`, v7.0) | COCO-pretrained object detector, loaded via `torch.hub` |
| [OpenCV](https://opencv.org/) | Camera capture, drawing bounding boxes, JPEG encoding |
| [Jetson.GPIO](https://github.com/NVIDIA/jetson-gpio) | Buzzer and LED control |
| [RPLCD](https://github.com/dbrgn/RPLCD) + smbus2 | 16x2 character LCD over I2C |
| ipywidgets | Live video preview inside Jupyter |

Target classes (COCO indices): `43` = knife, `76` = scissors.

---

## Hardware

| Part | Notes |
|---|---|
| NVIDIA Jetson board | Jetson Nano or similar, with JetPack installed |
| USB camera | `/dev/video0`, captured at 640×480 |
| Active buzzer | Driven directly from GPIO |
| LED + current-limiting resistor | e.g. 220–330 Ω |
| 16x2 LCD with PCF8574 I2C backpack | I2C address `0x27` |

### Wiring

GPIO numbering uses **`GPIO.BOARD`** (physical pin numbers on the 40-pin header).

| Device | Signal | Jetson pin (BOARD) |
|---|---|---|
| Buzzer | + (signal) | **Pin 33** |
| Buzzer | − | GND (e.g. pin 34) |
| LED | Anode (via resistor) | **Pin 7** |
| LED | Cathode | GND (e.g. pin 6) |
| LCD (PCF8574) | SDA | Pin 3 (I2C bus 1 SDA) |
| LCD (PCF8574) | SCL | Pin 5 (I2C bus 1 SCL) |
| LCD (PCF8574) | VCC | Pin 2 or 4 (5V) |
| LCD (PCF8574) | GND | Pin 6 / 9 / 14 (GND) |

To confirm the LCD is detected on I2C bus 1 (you should see `27`):

```bash
sudo i2cdetect -y -r 1
```

---

## Installation

1. **Install PyTorch for Jetson.** Use the NVIDIA wheels that match your JetPack version, not PyPI:
   <https://forums.developer.nvidia.com/t/pytorch-for-jetson/72048>

2. **Clone the repository and install the dependencies:**

   ```bash
   git clone https://github.com/<your-username>/jetson-danger-sos.git
   cd jetson-danger-sos
   pip install -r requirements.txt
   ```

3. **Give your user GPIO / I2C access** (or run with `sudo`):

   ```bash
   sudo groupadd -f -r gpio
   sudo usermod -aG gpio,i2c $USER
   # log out and back in for this to take effect
   ```

The YOLOv5s weights (`yolov5s.pt`) are downloaded automatically by `torch.hub` on first run. They are excluded from git by `.gitignore`.

---

## Usage

1. Wire up the hardware as shown above and plug in the USB camera.
2. Start Jupyter on the Jetson:

   ```bash
   jupyter notebook
   ```

3. Open `danger_detection.ipynb` and run the cells in order:
   1. Install the extra packages (`ipywidgets`, `seaborn`, `RPLCD`, `smbus2`)
   2. Load the YOLOv5s model on `cuda:0`
   3. Set up the LCD and GPIO, then start the detection loop
4. The live camera feed shows up in the notebook. Hold up a knife or scissors to trigger the alarm.
5. To stop, use **Kernel → Interrupt**. The `finally` block releases the camera, sets the GPIO pins low, runs `GPIO.cleanup()`, and shows `System Off` on the LCD.

---

## How It Works

```
┌────────────┐   frame    ┌──────────────┐  detections  ┌──────────────────────┐
│ USB Camera │ ─────────▶ │ YOLOv5s (GPU)│ ───────────▶ │ class ∈ {knife,      │
└────────────┘  640×480   └──────────────┘              │ scissors} & conf>0.1?│
      ▲                                                 └─────────┬────────────┘
      │                                              no ◀─────────┤ yes
      │                                    (show frame)           ▼
      │                                         ┌───────────────────────────────┐
      │                                         │ 1. Draw red box + "SOS !!!"   │
      │                                         │ 2. LCD: "Danger Detected!"    │
      │                                         │         "Status: SOS"         │
      │                                         │ 3. Buzzer + LED: ··· ——— ···  │
      │                                         │ 4. Cool down 3 s, clear LCD   │
      │                                         │ 5. Flush 5 stale frames       │
      └──────────────────────────────────────── └───────────────────────────────┘
```

1. **Capture.** OpenCV reads frames from camera 0 at 640×480 and converts them from BGR to RGB.
2. **Detect.** YOLOv5s runs inference on the GPU. Each detection is checked against `TARGET_CLASSES = [43, 76]` with a confidence threshold of `0.1`.
3. **Alert.** On a hit, the frame is annotated and pushed to the notebook widget, the LCD shows the warning, and `sos()` drives the buzzer and LED together:
   - Time unit `UNIT = 0.2 s`
   - Dot = 1 unit on, dash = 3 units on, 1 unit off between symbols
   - 2 extra units of gap between letters (S · O · S)
4. **Recover.** After a 3-second cooldown the LCD is cleared. Five buffered frames are thrown away so the next inference runs on a fresh image, and then the loop continues.
5. **Shutdown.** On interrupt, all hardware resources are released cleanly.

---

## Project Structure

```
jetson-danger-sos/
├── danger_detection.ipynb   # Main notebook: model loading, hardware setup, detection loop
├── requirements.txt         # Python dependencies
├── .gitignore
└── README.md
```

---

## Acknowledgements

- [Ultralytics YOLOv5](https://github.com/ultralytics/yolov5)
- NCKU course: Networked Embedded Systems Applications (網路嵌入式系統應用)
