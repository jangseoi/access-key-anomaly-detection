# 체인지로그

## 2026-08-19

- `ref-suspicious-detector.py` 주석 정리
- README에 최종 수정일 표기

## 2026-08-12

- **fix**: 쿨다운 테이블 키 이름을 실제 스키마(`alertKey`)에 맞춤 — `ref_alert_cooldown` 테이블의 파티션 키가 `accessKeyId`가 아닌 `alertKey`라서 `ValidationException`으로 쿨다운 기록이 매번 실패하던 문제 수정
- **fix**: 시나리오 3 쿨다운이 매번 새 IP로 오판되는 문제 수정 — 쿨다운 비교 기준을 "최근 5분 윈도우 내 API 호출 5건"에서 "임계값을 넘긴 시점의 실제 이벤트 출발지 IP"로 변경
- **feat**: `geoip-layer-builder` 추가 — MaxMind GeoLite2-City DB 최신 버전을 주기적으로 확인해 Lambda Layer로 자동 발행
- **refactor**: `ref-table-processor`의 Firehose 적재 제거, Reference Table TTL을 30일에서 7일로 변경
- **feat**: 초기 구성 — Reference Table 5종, `ref-table-processor` / `ref-suspicious-detector` Lambda, 탐지 시나리오 5종
