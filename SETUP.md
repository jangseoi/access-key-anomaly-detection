# Setup Guide

## 사전 요구사항

- AWS Control Tower (Organization Trail) 환경, 또는 단일 계정 멀티 리전 CloudTrail
- Log Archive 계정의 CloudTrail 중앙 S3 버킷에서 Audit 계정으로 S3 Event Notification을 보낼 수 있는 cross-account 권한
- Slack App (Bot Token, `chat:write` 권한)
- MaxMind 계정 및 License Key (GeoLite2-City DB 다운로드용)

<br>

## 1. DynamoDB 테이블 생성

아래 6개 테이블을 Audit 계정에 생성합니다. Reference Table 5종은 공통 키 구조를 사용합니다.
(참고) 'ref_alert_cooldown' 테이블은 Alert 쿨다운 로직에 활용되는 테이블로, Reference 목적의 테이블이 아닙니다.

| 테이블명 | 파티션 키 | 정렬 키 | TTL 속성 |
|---------|----------|--------|---------|
| `ref_aws_api` | `accessKeyId` (String) | `eventTime#eventId` (String) | `ttl` |
| `ref_ip_country` | `accessKeyId` (String) | `eventTime#eventId` (String) | `ttl` |
| `ref_region` | `accessKeyId` (String) | `eventTime#eventId` (String) | `ttl` |
| `ref_user_agent` | `accessKeyId` (String) | `eventTime#eventId` (String) | `ttl` |
| `ref_error_event` | `accessKeyId` (String) | `eventTime#eventId` (String) | `ttl` |
| `ref_alert_cooldown` | `alertKey` (String) | 없음 | `ttl` |

`ref_aws_api`, `ref_error_event`는 DynamoDB Streams(New image)를 활성화해야 합니다. (시나리오 탐지 Lambda 트리거 목적)

<br>

## 2. IAM 정책 연결

각 Lambda 실행 역할에 아래 권한을 연결합니다.

**ref-table-processor**
- `s3:GetObject` (Log Archive CloudTrail 버킷)
- `dynamodb:PutItem` (Reference Table 5종)
- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`

**ref-suspicious-detector**
- `dynamodb:GetItem`, `dynamodb:Query` (Reference Table 5종)
- `dynamodb:GetRecords`, `dynamodb:GetShardIterator`, `dynamodb:DescribeStream`, `dynamodb:ListStreams` (`ref_aws_api`, `ref_error_event` 스트림)
- `dynamodb:GetItem`, `dynamodb:PutItem` (`ref_alert_cooldown`)
- `secretsmanager:GetSecretValue` (Slack Bot Token 시크릿)
- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`

**geoip-layer-builder**
- `lambda:ListLayerVersions`, `lambda:PublishLayerVersion`, `lambda:GetFunctionConfiguration`, `lambda:UpdateFunctionConfiguration` (대상: `geoip-mmdb` 레이어, `ref-table-processor` 함수)
- `secretsmanager:GetSecretValue` (MaxMind License Key 시크릿)

<br>

## 3. Secrets Manager 설정

Slack Bot Token, Maxmind License Key 사용 시 보안성 향상을 위해 Lambda 실행 시마다 Secrets Manager에서 Token을 동적으로 조회하여 사용하도록 구성합니다.

| 시크릿 이름 (예시) | 키 | 사용처 |
|-------------------|-----|--------|
| `security-event-app-token` | `slack_bot_token` | `ref-suspicious-detector` |
| (환경변수 `SECRET_NAME`으로 지정) | `MAXMIND_LICENSE_KEY` | `geoip-layer-builder` |

<br>

## 4. Lambda 환경변수

### ref-table-processor

| 변수명 | 설명 |
|-------|------|
| `ERROR_EVENT_TABLE` | ref_error_event 테이블명 |
| `IP_COUNTRY_TABLE` | ref_ip_country 테이블명 |
| `AWS_API_TABLE` | ref_aws_api 테이블명 |
| `REGION_TABLE` | ref_region 테이블명 |
| `USER_AGENT_TABLE` | ref_user_agent 테이블명 |

### ref-suspicious-detector

| 변수명 | 설명 | 기본값 |
|-------|------|--------|
| `SLACK_CHANNEL_ID` | Slack 알림 채널 또는 DM ID | - |
| `ALLOWED_COUNTRIES` | 허용 국가코드 (콤마 구분) | `KR` |
| `ALLOWED_REGIONS` | 허용 리전 (콤마 구분) | `ap-northeast-2` |
| `ERROR_THRESHOLD` | 시나리오 3 AccessDenied 임계값 | `5` |
| `ERROR_WINDOW_MIN` | 시나리오 3 탐지 시간 윈도우 (분) | `5` |

### geoip-layer-builder

| 변수명 | 설명 |
|-------|------|
| `SECRET_NAME` | MaxMind License Key가 저장된 Secrets Manager 시크릿 이름 |

<br>

## 5. GeoIP Layer 최초 배포

`ref-table-processor`는 `/opt/GeoLite2-City.mmdb` 경로의 Layer를 사용합니다.
`geoip-layer-builder`를 한 번 수동 실행하여 `geoip-mmdb` Layer를 최초 발행하고 `ref-table-processor`에 연결한 뒤, 이후에는 EventBridge 스케줄(예: 매주 1회)로 자동 갱신되도록 설정합니다.

<br>

## 6. S3 Event Notification 연결

Log Archive 계정의 CloudTrail 버킷에 Object Created 이벤트를 Audit 계정의 `ref-table-processor` Lambda로 전달하도록 cross-account 권한과 알림을 설정합니다.

<br>

## 7. 동작 테스트

권한 없는 Role에 대해 `sts:AssumeRole`을 반복 호출하는 등으로 시나리오 3을 의도적으로 유발해 Slack 알림 및 쿨다운 동작을 확인할 수 있습니다.
