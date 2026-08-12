# AWS Access Key Anomaly Detection

> IAM Access Key 유출 및 이상행위를 실시간으로 탐지하여 신속한 대응을 지원하는 보안 모니터링 시스템

<br>

## 개요

AWS CloudTrail 이벤트를 기반으로 IAM Access Key(AKIA)의 이상행위를 실시간으로 탐지하고, Slack으로 알림을 발송하는 보안 모니터링 시스템입니다.

CloudTrail 이벤트를 5개의 Reference Table로 분리 적재하여 다차원 컨텍스트를 수집하고, DynamoDB Streams를 통해 데이터 적재 즉시 탐지 로직을 실행합니다. 공격자 행위 패턴을 기반으로 설계된 5개의 탐지 시나리오를 통해 이상행위를 식별합니다.

<br>

## 아키텍처

```
[멤버 계정] CloudTrail (모든 리전)
        ↓
[Log Archive] S3 (CloudTrail 중앙 버킷)
        ↓ S3 Event Notification (cross-account)
[Audit 계정] Lambda: ref-table-processor
        ↓
[Audit 계정] DynamoDB: Reference Table 5종
        ↓ DynamoDB Streams
[Audit 계정] Lambda: ref-suspicious-detector
        ↓
Slack 알림
```

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
- TTL: 30일

<br>

## 탐지 시나리오

| 시나리오 | 제목 | 트리거 | 조건 |
|---------|------|--------|------|
| 시나리오 1 | 공격자 초기 정찰 의심 | GetCallerIdentity / ListUserPolicies / ListAttachedUserPolicies | 국외 IP에서 호출 |
| 시나리오 2 | 권한 상승 시도 의심 | CreateAccessKey / AttachUserPolicy / CreateUser 등 | 국외 IP에서 호출 |
| 시나리오 3 | 짧은 시간 내 다수 AccessDenied | ref_error_event INSERT | 5분 내 동일 키 N회 이상 AccessDenied |
| 시나리오 4 | 비정상 리전 리소스 생성 의심 | RunInstances / CreateFunction | 허용 리전 외에서 호출 |
| 시나리오 5 | 알려진 공격 도구 사용 의심 | 모든 API 호출 | UserAgent에 공격 도구 시그니처 포함 |

<br>

## Lambda 함수

### ref-table-processor
- **역할**: CloudTrail 로그 파일을 파싱하여 AKIA Access Key 이벤트만 필터링 후 Reference Table에 적재
- **트리거**: Log Archive 계정 S3 Event Notification (cross-account)
- **Layer**: MaxMind GeoLite2-City.mmdb, geoip2 라이브러리

### ref-suspicious-detector
- **역할**: DynamoDB Streams를 통해 INSERT 이벤트 수신 후 탐지 시나리오 평가 및 Slack 알림 발송
- **트리거**: DynamoDB Streams (ref_aws_api, ref_error_event)

<br>

## 환경변수

### ref-table-processor

| 변수명 | 설명 |
|-------|------|
| `ERROR_EVENT_TABLE` | ref_error_event 테이블명 |
| `IP_COUNTRY_TABLE` | ref_ip_country 테이블명 |
| `AWS_API_TABLE` | ref_aws_api 테이블명 |
| `REGION_TABLE` | ref_region 테이블명 |
| `USER_AGENT_TABLE` | ref_user_agent 테이블명 |
| `FIREHOSE_ERROR_EVENT` | Firehose 스트림명 (ref_error_event) |
| `FIREHOSE_IP_COUNTRY` | Firehose 스트림명 (ref_ip_country) |
| `FIREHOSE_AWS_API` | Firehose 스트림명 (ref_aws_api) |
| `FIREHOSE_REGION` | Firehose 스트림명 (ref_region) |
| `FIREHOSE_USER_AGENT` | Firehose 스트림명 (ref_user_agent) |

### ref-suspicious-detector

| 변수명 | 설명 | 기본값 |
|-------|------|--------|
| `SLACK_CHANNEL_ID` | Slack 알림 채널 또는 DM ID | - |
| `ALLOWED_COUNTRIES` | 허용 국가코드 (콤마 구분) | `KR` |
| `ALLOWED_REGIONS` | 허용 리전 (콤마 구분) | `ap-northeast-2` |
| `ERROR_THRESHOLD` | 시나리오 3 AccessDenied 임계값 | `5` |
| `ERROR_WINDOW_MIN` | 시나리오 3 탐지 시간 윈도우 (분) | `5` |

> Slack Bot Token은 AWS Secrets Manager(`security-event-app-token`)에서 런타임 시 동적으로 조회합니다.

<br>

## 쿨다운 로직

동일 Access Key에서 반복적으로 알람이 발송되는 것을 방지하기 위해 IP 기반 쿨다운 로직을 적용합니다.

- 시나리오 1, 4, 5: 동일 IP에서 10분 내 재발송 차단
- 시나리오 2: 동일 IP에서 30분 내 재발송 차단
- 시나리오 3: 동일 IP에서 30분 내 재발송 차단, **새로운 IP 감지 시 즉시 재발송**

쿨다운 이력은 `ref_alert_cooldown` DynamoDB 테이블에 저장하며 TTL로 자동 만료됩니다.

<br>

## 기술 스택

- **AWS**: CloudTrail, DynamoDB, Lambda, Kinesis Firehose, Athena, QuickSight, Secrets Manager, S3
- **Python**: boto3, geoip2
- **외부**: MaxMind GeoLite2-City (IP 지리 정보), Slack API

<br>

## 활용 사례

- 사내 Control Tower 환경에 직접 구축하여 운영 중
- 고객사 2곳 도입 검토 진행 중 (2026 현재)
