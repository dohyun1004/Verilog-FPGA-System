# Verilog FPGA Integrated System

**UART 통신으로 제어하고, Watch·Stopwatch 및 HC-SR04·DHT11 센서 데이터를 FND와 UART로 출력하도록 구성한 FPGA 통합 설계 프로젝트입니다.**

- HDL: **Verilog**
- 개발 환경: **Xilinx Vivado**
- 최상위 모듈: `top_all`
- 주요 기능: 디지털 시계, 스톱워치, 초음파 거리 측정, 온습도 측정, UART 명령/응답, 4자리 FND 표시

> **진행 상태:** 이 README는 제공된 RTL 24개를 바탕으로 작성했습니다. README에 설명된 회로 구조와 설계 기능은 소스 기준이며, 전체 합성·시뮬레이션 PASS나 실제 보드 동작을 새로 검증한 것은 아닙니다. 원본 소스와 구조도는 아직 저장소 업로드를 별도로 진행해야 합니다.

## 1. 프로젝트 목표

서로 독립적으로 만든 UART, 시간 카운터, 초음파 센서, 온습도 센서, FND 표시 회로를 하나의 Top 모듈로 통합하는 것이 핵심입니다. **스위치로 동작 모드를 선택하고 버튼 또는 UART 문자 명령으로 제어**하며, 선택된 기능의 데이터를 디스플레이에 표시하거나 PC로 전송하도록 설계했습니다.

핵심 설계 포인트는 다음과 같습니다.

1. **모듈화:** 각 기능을 독립적인 Verilog 모듈로 구현한 뒤 `top_all`에서 연결
2. **제어/데이터 경로 분리:** 입력 디바운싱, 명령 해석, FSM 제어, 시간·센서 데이터, 출력 경로를 구분
3. **FSM 기반 센서 통신:** HC-SR04와 DHT11의 신호 시퀀스를 상태 전이로 제어
4. **UART + FIFO:** 수신한 ASCII 명령을 버튼 이벤트와 연결하고 요청한 데이터는 문자 형태로 송신
5. **공유 출력:** Watch, Stopwatch, 센서 값을 모드에 따라 선택해 FND/UART 출력에 연결

**문서 바로가기:** [시스템/FSM 구조도](docs/architecture.md) · [검증 현황](docs/verification.md) · [트러블슈팅](docs/troubleshooting.md) · [원본 소스 목록](src/README.md)

## 2. 전체 시스템 구조

```text
                       +-------------------------+
Buttons R/L/U/D ------>| Button Debounce         |
Switches SW[3:0] ----->| Control / Mode Selection|----> Watch
UART RX -> UART + FIFO>| ASCII Decoder           |----> Stopwatch
                       +-------------------------+----> HC-SR04 Controller
                                  |                   -> DHT11 Controller
                                  v
                         Selected Data / Mode
                           |           |
                           v           v
                      FND Display   ASCII Sender
                                      |
                                  TX FIFO -> UART TX -> PC
```

통합 Top `top_all`의 주요 입출력은 `clk`, `rst`, `rx/tx`, `btnR/L/U/D`, `sw[3:0]`, HC-SR04 `echo/trig`, DHT11 양방향 `dht11_io`, `fnd_com[3:0]`, `fnd_data[7:0]`, `led[2:0]`입니다.

자세한 연결 구조는 제공된 `UART_시계_센서_통합구조_요약_상세.drawio` 원본으로 관리합니다.

## 3. 주요 기능

| 기능 | 핵심 내용 | 대표 모듈 |
| --- | --- | --- |
| Watch | 10 ms 기반 시각 카운터, 시각 설정 경로 | `watch.v` |
| Stopwatch | 실행/정지, 초기화, 모드 및 랩 관련 데이터 경로 | `stopwatch.v`, `control_unit_btn.v` |
| HC-SR04 | TRIG 출력, ECHO 펄스 측정, 거리 데이터 저장 | `sr04_controller.v`, `top_sr04_contoller.v` |
| DHT11 | 시작 신호, 응답 동기화, 데이터 비트 수신 | `dht11_controller.v`, `dht11_top.v` |
| UART | 직렬 수신·송신, FIFO 버퍼, ASCII 명령 해석 및 응답 생성 | `uart.v`, `top_uart.v`, `fifo_rx.v`, `fifo_tx.v`, `ascii_decoder.v`, `ascii_sender.v` |
| FND | 모드에 따른 시간·센서 데이터 선택 및 7-Segment 표시 | `top_fnd.v`, `display_unit.v`, `fnd_controller.v`, `sensor_fnd.v` |

### 스위치별 기능

`top_all.v`의 주석과 선택 로직 기준입니다.

| 스위치 | 역할 |
| --- | --- |
| `SW0` | 시간 모드에서 Watch(0) / Stopwatch(1) |
| `SW1` | 시간 표시 형태: 초·센티초(0) / 시·분(1) |
| `SW2` | 시간 모드에서는 시계 설정, 센서 모드에서는 HC-SR04(0) / DHT11(1) 선택 |
| `SW3` | 시간 모드(0) / 센서 모드(1) |

## 4. UART 명령 및 응답 흐름

```text
PC -> UART RX -> RX FIFO -> ASCII Decoder
                                |
                                +--> R / L / U / D : 버튼 대응 명령
                                +--> M : 기능 리셋 신호
                                +--> S : 선택 데이터 전송 요청
                                               |
                                               v
                                     ASCII Sender (FSM)
                                               |
                                          TX FIFO
                                               |
                                           UART TX -> PC
```

`ascii_decoder.v`는 ASCII 대문자 **R, L, U, D, M, S**를 인식합니다. `top_all.v`는 `M` 명령을 기능 리셋 경로에 연결하며, `S` 요청 시 `ascii_sender.v`는 선택한 기능의 데이터를 송신용 문자 시퀀스로 구성합니다.

송신 선택 값은 `00=WATCH`, `01=STOPWATCH`, `10=SR04`, `11=DHT11`입니다. `ascii_sender`는 요청 시작 시 32비트 데이터를 내부 레지스터에 저장해 전송하는 구조이며, `top_uart.v`에서는 깊이 21의 TX FIFO를 인스턴스화합니다.

## 5. 주요 FSM

### DHT11 — 8개 상태

`IDLE -> START -> WAIT -> SYNC_L -> SYNC_H -> DATA_SYNC -> BIT -> STOP`

시작 조건을 받아 데이터 라인의 방향과 응답 타이밍을 제어하고, 비트 수신을 진행하도록 구성되어 있습니다. 상세 타이밍과 데이터 유효성은 `dht11_controller.v` 구현 및 시뮬레이션으로 확인해야 합니다.

### HC-SR04 — 4개 상태

`IDLE -> START -> WAIT -> RESPONSE`

TRIG 시작 신호, ECHO 대기, ECHO 측정 결과 저장을 처리합니다. 제공된 설계 기준으로 별도 응답 대기 timeout 동작은 확인되지 않았습니다.

### ASCII Sender — 5개 상태

`IDLE / WATCH / STOPWATCH / SR04 / DHT11`

송신 요청 시 선택된 모드의 텍스트 형식으로 데이터를 구성하고 FIFO로 내보냅니다. `full` 조건 및 문자 카운터의 실제 동작은 RTL을 참조하세요.

## 6. 소스 파일 구성

```text
Verilog-FPGA-System/
├── README.md
├── src/                  # Verilog RTL 24개 (업로드 예정)
│   ├── top_all.v
│   ├── top_uart.v
│   ├── uart.v
│   ├── ascii_decoder.v
│   ├── ascii_sender.v
│   ├── fifo.v
│   ├── fifo_rx.v
│   ├── fifo_tx.v
│   ├── watch.v
│   ├── stopwatch.v
│   ├── control_unit.v
│   ├── control_unit_btn.v
│   ├── sr04_controller.v
│   ├── top_sr04_contoller.v
│   ├── dht11_controller.v
│   ├── dht11_top.v
│   └── ... (FND, 디바운싱, 타이밍 등)
└── docs/                 # 구조도/설계 자료 (추가 예정)
```

**모듈명 주의:** 파일 `top_fnd.v`는 모듈 `display_unit`을, 파일 `display_unit.v`는 모듈 `top_fnd`를 정의합니다. `control_unit_btn.v` 내부 모듈명은 `controll_unit_btn`입니다. Vivado에서 파일명보다 실제 `module` 선언과 인스턴스 연결을 기준으로 확인하세요.

## 7. Vivado 실행 및 검증

1. 새 Vivado RTL 프로젝트를 생성하고 원본 Verilog 24개를 Design Sources에 추가합니다.
2. Top Module을 **`top_all`**로 설정합니다.
3. 사용하는 FPGA 보드의 `.xdc` 핀 제약을 추가합니다.
4. 시뮬레이션용 테스트벤치를 추가하고 기능별 파형을 검증합니다.
5. 합성(Synthesis) → 구현(Implementation) → Bitstream 생성 → 보드 동작 확인 순서로 검증합니다.

**검증 범위:** 이번에 제공된 ZIP에는 XDC 파일과 테스트벤치가 포함되지 않았습니다. 따라서 README 자체가 통합 기능의 검증 완료를 의미하지 않습니다.

## 8. 현재 저장소 구성

```text
Verilog-FPGA-System/
├── README.md
├── .gitignore
├── src/
│   └── README.md             # 원본 RTL 업로드 대기
└── docs/
    ├── architecture.md       # 시스템 구조 / FSM Mermaid
    ├── verification.md       # 검증 상태 체크리스트
    ├── troubleshooting.md    # 문제 해결 점검 목록
    └── upload-status.md
```

**현재 소스와 원본 draw.io 파일은 아직 GitHub로 전송되지 않았습니다.** 제공받은 원본 ZIP과 구조도는 보존하고 있으나 GitHub 폴더의 문서가 그 파일을 대체하지는 않습니다.

## 9. 팀 구성

| 역할 | 이름 | GitHub |
| --- | --- | --- |
| 프로젝트 정리 | 서도현 | [dohyun1004](https://github.com/dohyun1004) |
| 팀원 1 | | |
| 팀원 2 | | |
| 팀원 3 | | |

---

**프로젝트 핵심:** UART 입력과 물리 버튼을 공통 제어 경로로 연결하고, FPGA 내부에서 시간·거리·온습도 데이터를 생성·선택하여 FND 및 UART로 출력하는 모듈형 통합 시스템.
