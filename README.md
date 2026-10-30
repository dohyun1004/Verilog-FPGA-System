# Verilog FPGA Integrated System

Verilog HDL / Vivado 기반 FPGA 통합 프로젝트: UART 통신, Watch·Stopwatch, HC-SR04, DHT11, FND 표시.

## 구성

- `src/`: Verilog RTL 원본 24개
- `docs/`: 통합 블록 다이어그램(draw.io) 및 검증 기록
- Top module: `top_all`

## FSM

- DHT11: IDLE / START / WAIT / SYNC_L / SYNC_H / DATA_SYNC / BIT / STOP
- SR04: IDLE / START / WAIT / RESPONSE
- ASCII Sender: IDLE / WATCH / STOPWATCH / SR04 / DHT11

## Vivado 설정

`top_all`을 top module로 지정합니다. 보드용 XDC 및 테스트벤치는 제공된 원본 ZIP에 포함되지 않아 별도 추가해야 합니다. 검증 PASS 또는 보드 구동 완료를 새로 주장하지 않습니다.

**모듈 이름 주의:** `top_fnd.v`의 내부 모듈은 `display_unit`이고, `display_unit.v`의 내부 모듈은 `top_fnd`입니다.

## 팀 정보

| 역할 | 이름 | GitHub |
| --- | --- | --- |
| 작성자 | 서도현 | dohyun1004 |
| 팀원 1 | | |
| 팀원 2 | | |
| 팀원 3 | | |
