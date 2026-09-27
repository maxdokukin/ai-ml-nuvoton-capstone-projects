# Nuvoton Edge-AI Capstone Projects — three student teams building on-device AI on the M55M1

Nuvoton · 2026 · Three student teams: SJSU AI/ML Club (Bence Danko); UCSB EchoTrack (Rohan Koshy, Jonah Cheyette); UCSB AI Fitness Mirror (Edward Ding, Austin, Justin Fan, Jiesheng He) · Max Dokukin: problem designer, biweekly mentor, leadership liaison · Completed

## Overview

As AI/ML Engineer at Nuvoton, Max Dokukin took on three roles in this program:

- **Feasibility and project list.** He showed senior leadership what the M55M1 microcontroller can run (Arm Cortex-M55
  CPU plus Ethos-U55 NPU), then drew up the list of candidate projects that leadership chose from.
- **Problem statements.** He wrote the problem statements for two UCSB capstone projects (September 2025).
- **Mentoring.** He met the student teams every two weeks for technical guidance.

Between April and June 2026, three student teams built INT8, Vela-compiled neural networks into bare-metal and
FreeRTOS firmware on the NuMaker M55M1 board:

- **Elevator people counter.** The SJSU AI/ML Club built an overhead counter that runs a person-only YOLOv8n at
  25 FPS and pushes counts over Wi-Fi to a web dashboard.
- **EchoTrack active-speaker camera.** A UCSB team combined two-microphone direction-of-arrival (DoA) estimation with
  face detection and a mouth-open/closed model.
- **AI Fitness Mirror.** A UCSB team chains YOLOv8n-pose with two small multilayer perceptrons (MLPs), one for rep
  counting and one for form-error classification, across five exercises.

The students wrote all of the code. This entry documents Max's scoping and mentoring role and what each team
delivered, compared with the proposal targets.

## Highlights

- **Elevator.** YOLOv8n (ReLU6, INT8, 192×192 input) detects people at 25 FPS on the M55M1. The camera image is shown
  at 640×480 on the LCD with boxes, FPS and a people count. Counts are pushed every 3 s over an ESP8266 to a FastAPI
  dashboard (elevator `README.md`, `main.cpp`, `board_config.h`).
- **Elevator training pipeline.** The detector was retrained on person-only COCO on a Modal A10G GPU. The pipeline
  exports it through ONNX → INT8 TFLite (per-tensor, coco128 calibration) → Vela (`ethos-u55-256`) in one command
  (`modal_export.py`, `compile_best.bat`).
- **Mirror error classifier.** It reaches 0.80 frame accuracy and 1.00 clip accuracy on 22 held-out clips (2,343
  frames, 11 classes). Push-up classes score F1 0.99 and squats 0.85–0.91 (`training_summary.json`).
- **Mirror on-device stack.** Three models share the M55M1: YOLOv8n-pose, a 23 KB rep-phase MLP and a 22 KB
  error-class MLP. Vela estimates about 38 µs per MLP inference. The build adds a BLE wearable (accelerometer and heart
  rate) and a Streamlit PC dashboard (`sd_card_root/MANIFEST.md`, Vela summary CSVs, `wearable_code.ino`,
  `dashboard.py`).
- **EchoTrack.** GCC-PHAT DoA (48 kHz, 256-sample frames, 150–3500 Hz band) runs between camera frames next to a
  per-face "speaking" detector with 3-on/4-off frame hysteresis. A servo-pan driver and a four-microphone DoA function
  were written but are not yet wired into the main loop (`DoA.c`, `SpeakingDetector.cpp`, `motor.c`, `main.cpp`).

## How it works

```
Max: M55M1 feasibility demo → candidate project list → leadership pick → problem statements → biweekly mentoring
                                                                                    │
Elevator:  HM1055 camera 320×240 → letterbox 192×192 INT8 → YOLOv8n (Ethos-U55) → DFL decode + NMS → count → LCD + ESP8266 → FastAPI dashboard
EchoTrack: I2S mics → GCC-PHAT DoA ┐
           camera → face detector → per-face mouth YOLOv8n → hysteresis → "Speaking" overlay   (servo pan: driver written, not wired)
Mirror:    camera → YOLOv8n-pose → 51-value keypoint vector → rep-phase MLP → rep state machine
                                                            └→ error-class MLP → "ERR"/feedback → LCD + serial → Streamlit dashboard
           XIAO nRF52840 (BMI270 + MAX30102) → BLE UART → board UART1 → AX/AY/AZ/HR on the LCD
```

- **Shared platform.** The NuMaker M55M1 board (Cortex-M55 CPU plus Ethos-U55 NPU with 256 multiply-accumulates per
  cycle), HyperRAM for model storage, and SD-card model loading. Every model is an INT8 TFLite file compiled with Arm
  Vela and run with TensorFlow Lite Micro. The firmware is built in Keil µVision on Nuvoton's BSP and NuEdgeWise/ML
  samples.
- **Elevator firmware.** FreeRTOS runs three tasks (inference, display blit and Wi-Fi push). Two camera buffers let
  capture and NPU inference overlap, and a lookup-table quantizer converts camera pixels to model input. Memory
  protection unit (MPU) cache regions cover the tensor arena and the frame buffers.
- **EchoTrack firmware.** A single loop runs in this order: trigger camera capture, then DoA on one audio frame, then
  face + mouth inference, then draw.
- **Mirror firmware.** Three models are loaded back-to-back into HyperRAM. The same 51-feature pose vector feeds both
  MLPs. The firmware prints `DATA:` serial lines for the dashboard.

## Results

| Project | Metric | Value | Note / source |
|---|---|---|---|
| Elevator | Detection throughput | 25 FPS | 640×480 on the LCD, YOLOv8n INT8 192×192 (README); 20 FPS in the prior version |
| Mirror | Error classifier frame accuracy | 0.800 | 2,343 validation frames, 11 classes, clip-grouped split (`training_summary.json`) |
| Mirror | Error classifier clip accuracy | 1.00 | 22 held-out clips, mean of frame probabilities |
| Mirror | Error classifier macro F1 | 0.787 | per-class 0.58 (JJ good) to 0.99 (push-ups) |
| Mirror | Rep-phase classifier validation accuracy | 1.00 | 66 images, 8 phase classes (`Rep_Count.ipynb`) |
| Mirror | Vela estimate per MLP | ~37.5–37.8 µs, ~26.5k inf/s | compiler estimate, not a board measurement |
| EchoTrack | Performance targets | not measured | no FPS, latency or focus numbers in the repo |

The proposal's latency, accuracy and stability targets were not measured in either UCSB repository: ≥15/≥20 FPS,
≤100 ms latency, PCK@0.2 ≥ 0.85, rep F1 ≥ 0.90, ≤3 s retarget and ≥90% session focus. The elevator README's 25 FPS
is the only on-device throughput figure.

## Getting started

Nothing here is runnable on its own. Each team repository has its own build steps:

```bash
# Elevator: flash PeopleCounting.uvprojx with Keil, put MODEL.TFL on the SD card, then
python web_server.py
# Mirror: copy sd_card_root/*.tflite to the SD card, flash KEIL/PoseLandmark.uvprojx, then
pip install streamlit pyserial && streamlit run dashboard.py
# EchoTrack: build SampleCode/NuEdgeWise/PoseLandmark_YOLOv8n/KEIL/PoseLandmark.uvprojx; mouth model on SD as best_full_integer_quant_vela.tflite
```

Requirements: a NuMaker M55M1 board, Keil µVision 5 with the M55M1 device pack, and a FAT32 SD card. Training needs
Python 3.10+, Ultralytics, TensorFlow 2.15–2.16 and `ethos-u-vela` (3.10.0 for the elevator, 4.0.0 for the Mirror).

## Documents

- Problem statement: *AI Camera: Meeting Room Active Speaker Tracking* (`deliverables/proposal_ai_camera_echotrack.docx`)
- Problem statement: *AI Smart Mirror: an Intelligent Workout Coach* (`deliverables/proposal_ai_fitness_mirror.docx`)
- Team repositories: [ai-ml-nuvoton-sjsu-elevator](https://github.com/maxdokukin/ai-ml-nuvoton-sjsu-elevator), [ai-ml-nuvoton-ucsb-echotrack](https://github.com/maxdokukin/ai-ml-nuvoton-ucsb-echotrack), [ai-ml-nuvoton-ucsb-mirror](https://github.com/maxdokukin/ai-ml-nuvoton-ucsb-mirror)
- Elevator team report and slides: Google Docs/Slides links in the elevator README history (not reviewed here)
