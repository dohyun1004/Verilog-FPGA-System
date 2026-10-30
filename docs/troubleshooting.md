# Troubleshooting

기록된 이슈와 실제 재현·해결이 확인된 이슈를 구분합니다.

| 점검 항목 | 확인할 사항 |
|---|---|
| DHT11 valid 미출력 | start, 데이터선 방향, 응답 동기화, 40비트 비트 카운터, 체크섬 |
| UART 미송신 | baud tick, tx_start, busy, FIFO empty/full, POP 타이밍 |
| FND 표시 불일치 | 모듈명과 파일명 불일치, SW 선택, 자리 스캔 |
| SR04 측정 멈춤 | ECHO 입력, 측정 상태 전이, timeout 부재 여부 |
| Vivado 합성 오류 | 누락 모듈, Top 지정, pin/XDC, 복수 드라이버 |

**주의:** 위 내용은 점검 목록이지, 이번 커밋에서 재현·해결했다는 검증 기록이 아닙니다.
