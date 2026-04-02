# [지능형 국지 방어] 이기종 분산 처리 기반 음향 표적 탐지 및 지향성 감시 체계

**Heterogeneous Distributed Acoustic Target Detection & Surveillance System**

---

## 1. 프로젝트 목적 및 배경 (Background & Objectives)

### 방산 분야 적용

현대 전장 및 주요 보안 시설의 대드론(Anti-Drone) 체계 및 저격수 탐지 시스템에 적용되는 **'음향-시각 융합 복합 센서 체계'** 프로토타입 개발

### 이기종 분산 아키텍처 채택

단일 보드에서 모든 연산을 처리할 경우 발생하는 모터의 역기전력(Back-EMF) 및 고성능 프로세서의 스위칭 노이즈(Ground Bounce)가 초정밀 아날로그 오디오 신호에 치명적인 간섭(EMI)을 일으킴. 이를 원천 차단하기 위해 **고속 음향 처리 전용 FPGA 하드웨어**와 **영상/모터 제어용 GPU 비전 노드(Jetson)**를 물리적으로 분리하는 체계 연동 설계를 적용함.

---

## 2. 시스템 아키텍처 및 체계 연동 조감도 (System Architecture)

### Node 1 — Sensor Node (Custom FPGA Board)

```
Sound Target
  ↓ (Acoustic Wave)
4x Analog MEMS Mic
  ↓
Differential Driver (THS4551)
  ↓
ADS1274 (24-bit ADC, Simultaneous Sampling)
  ↓
Xilinx Artix-7 FPGA
  (Data Capture / Channel Sync / TDOA-DOA / UDP Packetization)
  ↓
Ethernet PHY (LAN8720A) → HR911105A (MagJack)
  ↓
UDP Transmission : "Target at θ°"
```

### 연결 인터페이스

**Standard Ethernet Cable**

### Node 2 — Command & Vision Node (Jetson Orin Nano)

```
Jetson Orin Nano
  (Direction Data Receive + Vision Processing + AI)
  ├── (PWM / GPIO) → 2-Axis Pan-Tilt Motor (Rotate to Target Direction)
  └── (MIPI / USB) → Camera Module (Target Tracking & Imaging)
```

### 보드 구성 상세

| 구분                              | 명칭                                   | 설명                                                                                                                    |
| --------------------------------- | -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **Sensor Deck**             | 마이크로폰 도터보드 (Protruding Array) | 4채널 아날로그 MEMS 마이크와 디커플링 커패시터만 실장, 3D 프린팅 하우징 외부 최상단으로 돌출 배치                       |
| **Processing & Power Core** | 메인보드 (Enclosed Base)               | Artix-7 FPGA, 차동 드라이버(FDA), ADC, 전원부, Ethernet PHY, MagJack 등 노이즈/발열 발생원을 3D 하우징 내부에 밀폐 실장 |
| **Interface**               | Board-to-Board 연결                    | 저레벨 아날로그 마이크 출력 및 전원 라인의 무결성 유지를 위해 쉴딩 처리된 FPC 케이블 체결                               |

---

## 3. 하드웨어 주요 사양 (Hardware Specifications — Sensor Node)

| 항목                        | 사양                                                                                                   |
| --------------------------- | ------------------------------------------------------------------------------------------------------ |
| **Main Processor**    | AMD Xilinx Artix-7 (XC7A35T-1CSG325C) — Bare-metal Chip-down                                          |
| **Sensor Array**      | Infineon IM68A130A Analog MEMS Microphone × 4ea                                                       |
| **Analog Front-End**  | TI THS4551 (Fully Differential Amplifier)                                                              |
| **Data Converter**    | TI ADS1274 (24-bit Simultaneous Sampling ADC)                                                          |
| **Network Interface** | Microchip LAN8720A (10/100M Ethernet PHY, RMII) + HanRun HR911105A (MagJack) + 50MHz Active Oscillator |
| **Configuration**     | Winbond W25Q32 (32Mbit SPI Flash) / JTAG Interface                                                     |
| **PCB 스펙**          | 4-Layer FR-4 (L1: Signal/USER – L2: Solid GND – L3: VCC/Power – L4: Signal/USER), 두께 1.6mm        |
| **EDA Tool**          | KiCad (회로도, PCB Artwork, 3D Rendering, Impedance Calculation)                                       |

### Power Tree

| 레일 | 용도          | 소자                          |
| ---- | ------------- | ----------------------------- |
| 1.0V | FPGA Core     | TI TLV62569 Buck Converter    |
| 3.3V | FPGA I/O      | TI TLV62569 Buck Converter    |
| 3.3V | MEMS Mic      | TI LP5907 Ultra-Low Noise LDO |
| 1.8V | ADC DVDD      | TI LP5907 1.8V LDO            |
| 2.5V | ADC Reference | TI REF5025                    |

### Form Factor

| 보드          | 크기                                               |
| ------------- | -------------------------------------------------- |
| Mainboard     | 100mm × 100mm (DFM 및 양산 단가 최적화 규격)      |
| Daughterboard | 50mm × 50mm 이내 십자형 구조 (마이크 어레이 전용) |

---

## 4. 하드웨어 설계 핵심 기술 (Core Hardware Design Technologies)

### ① 물리적 노이즈 격리 및 전원 무결성 (Power Integrity)

- **시스템 분리** — 모터 구동부와 고성능 GPU(Jetson)를 물리적으로 분리하여 아날로그 센서부의 GND 흔들림 및 전원 강하 현상 방지
- **초저노이즈 전원 트리(Power Tree)**
  - FPGA Core(1.0V) 및 I/O(3.3V) : 스위칭 강하 효율을 위한 고성능 Buck Converter 적용
  - MEMS Mic(3.3V), ADC Analog Rail, ADC DVDD(1.8V) : 디지털 스위칭 노이즈 차단을 위해 별도의 초저노이즈 LDO 및 기준 전압 레퍼런스 적용
  - Ethernet PHY 아날로그 전원은 Ferrite Bead 후단의 ETH_3V3_A로 분리 공급

### ② 고속 신호 무결성 (Signal Integrity) 및 4-Layer 임피던스 제어

- **4-Layer Stackup 최적화** — `L1(Top Signal) - L2(Solid GND) - L3(Power/VCC) - L4(Bottom Signal)`의 정석적인 4층 구조 채택. 2층(L2)에 끊어짐 없는 완벽한 GND 플레인을 깔아 기판 내 모든 고속 신호의 최단 리턴 패스(Return Path)를 보장하고 고주파 방사 노이즈(EMI)를 원천 차단함.
- **차동 배선(Differential Pair)** — Ethernet PHY ↔ MagJack 간 TX/RX 라인에 대한 100Ω 차동 임피던스 매칭을 수행하며, L2 GND 레퍼런스 플레인을 기준으로 KiCad 임피던스 계산기를 통해 산출된 배선폭/간격을 Artwork에 적용함.
- **Mixed-Signal 프런트엔드** — 4채널 아날로그 마이크 출력을 FDA로 차동 변환한 뒤 ADS1274에 입력하여 공통모드 노이즈 내성 확보
- **동시 샘플링(Simultaneous Sampling)** — ADS1274의 4채널 동시 샘플링을 이용하여 TDOA 연산에 필요한 채널 간 시간 정합성 확보
- **클록 무결성(Clock Integrity)** — ADC 전용 저지터 클록과 Ethernet용 50MHz REF_CLK를 분리하여 각 인터페이스의 지터 민감도를 관리

### ③ Chip-down 기반 FPGA 인프라 자력 설계

- 기성품 평가 보드(Eval Board) 배제, FPGA 구동을 위한 최소 시스템(Minimum System) 직접 설계
- 데이터시트 분석을 통한 Power-On Reset(POR), Configuration Flash, JTAG 디버그 회로 독자 구현
- FPGA 내부 로직 : ADS1274 디지털 데이터 수집 → 채널 정렬/버퍼링 → TDOA/DOA 연산 → Ethernet UDP 패킷화

### ④ 기구-전자 통합 설계 및 음향학적 최적화 (Acoustic-Mechanical Integration)

- **음향 그림자(Acoustic Shadowing) 및 공명 차단** — 단일 보드 설계 시 대형/고발열 소자에 의한 소리 회절과 케이스 내부 빈 공간의 헬름홀츠 공명(Helmholtz Resonance) 왜곡 방지. 마이크 어레이 보드를 케이스 외부로 돌출 분리시켜 360° 완벽한 음향 시야(Line-of-Sight) 확보
- **센서 지오메트리 설계** — 공간 앨리어싱(Spatial Aliasing, ≤21.25mm @ 8kHz) 한계와 샘플링 시간 분해능을 고려하여 TDOA 알고리즘 최적의 30~40mm 정사각형 마이크 배치 간격 도출

---

## 5. 기대 효과 및 포트폴리오 소구점 (Expected Impact)

이 프로젝트는 단위 칩 수준의 정밀한 하드웨어 설계 능력(SI/PI, 4-Layer Artwork)을 갖추었을 뿐만 아니라, 전체 무기 체계의 데이터 흐름과 노이즈 간섭(EMC)을 이해하고 최적의 아키텍처를 설계하는 **시스템 엔지니어링 역량**을 겸비했음을 증명함.

특히 방위산업에서 가장 민감하게 다루는 아날로그 센서 신호의 신뢰성, Mixed-Signal 프런트엔드 설계, FPGA 기반 실시간 TDOA/DOA 연산, 체계 연동 통신(Ethernet, UDP)을 실제 구현함으로써, 입사 후 즉각적으로 실무 하드웨어 R&D 프로젝트에 투입될 수 있는 준비된 인재임을 어필함.
