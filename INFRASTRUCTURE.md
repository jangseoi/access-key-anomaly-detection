# Infrastructure

## 아키텍처

![아키텍처 다이어그램](./access-key-anomaly-detection.png)

> AWS Control Tower (Organization Trail) 환경 기준으로 설계되었으며, 단일 계정 멀티 리전 Trail 환경에서도 동일하게 구성 가능합니다.

<br>

## Reference Table

모든 테이블은 공통 키 구조를 사용하여 상관분석이 가능합니다.

| 테이블명 | 설명 | 주요 필드 |
|---------|------|----------|
| `ref_aws_api` | Access Key가 호출한 AWS API 정보 | eventName, eventSource |
| `ref_ip_country` | 호출 Source IP의 지리 정보 | sourceIPAddress, countryCode, city |
| `ref_region` | 호출된 대상 AWS 리전 정보 | awsRegion |
| `ref_user_agent` | 호출 클라이언트 UserAgent 정보 | userAgent, userAgentType |
| `ref_error_event` | errorCode가 발생한 이벤트 정보 | eventName, errorCode, errorMessage |

**공통 키 구조**
- PK: `accessKeyId`
- SK: `eventTime#eventId`
- TTL: 속성명 `ttl`, 7일 (`ref-table-processor`가 적재 시점에 계산해서 기록)

<br>

## 쿨다운 테이블 (ref_alert_cooldown)

시나리오 3(짧은 시간 내 다수 AccessDenied)의 중복 알람을 억제하기 위한 테이블입니다.

| 속성 | 설명 |
|------|------|
| `alertKey` (PK) | Access Key ID |
| `lastIPs` | 쿨다운 기간 동안 이미 알람을 발송한 출발지 IP 목록 |
| `ttl` | 쿨다운 만료 시각 (기본 알람 발송 후 30분) |

정렬 키(Sort Key) 없이 파티션 키(`alertKey`) 단독으로 생성해야 합니다.

<br>

## Lambda 함수

### ref-table-processor
- **역할**: CloudTrail 로그 파일을 파싱하여 AKIA Access Key 이벤트만 필터링 후 Reference Table에 적재
- **트리거**: Log Archive 계정 S3 Event Notification (cross-account)
- **Layer**: `geoip-mmdb` (MaxMind GeoLite2-City.mmdb + geoip2 라이브러리)

### ref-suspicious-detector
- **역할**: DynamoDB Streams를 통해 INSERT 이벤트 수신 후 탐지 시나리오 평가 및 Slack 알림 발송
- **트리거**: DynamoDB Streams (`ref_aws_api`, `ref_error_event`)

### geoip-layer-builder
- **역할**: MaxMind GeoLite2-City DB의 최신 버전을 주기적으로 확인하여, 새 버전이 있으면 `geoip-mmdb` Lambda Layer로 발행하고 `ref-table-processor`에 자동 적용
- **트리거**: 스케줄(EventBridge) 권장

<br>

## 쿨다운 로직

동일 Access Key에서 반복적으로 알람이 발송되는 것을 방지하기 위해 IP 기반 쿨다운 로직을 적용합니다 (현재 시나리오 3에만 적용).

- 임계값을 넘긴 시점의 실제 이벤트(해당 error 이벤트의 SK) 출발지 IP를 기준으로 비교
- 동일 IP면 `ALERT_COOLDOWN_MIN`(기본 30분) 동안 재발송 차단
- 새로운 IP가 감지되면 즉시 재발송하고, 쿨다운 기간 동안 확인된 IP를 누적 저장

쿨다운 이력은 `ref_alert_cooldown` DynamoDB 테이블에 저장하며 TTL로 자동 만료됩니다.

<br>

## 기술 스택

- **AWS**: CloudTrail, DynamoDB (+ Streams), Lambda, Secrets Manager, S3
- **Python**: boto3, geoip2, requests
- **외부**: MaxMind GeoLite2-City (IP 지리 정보), Slack API
