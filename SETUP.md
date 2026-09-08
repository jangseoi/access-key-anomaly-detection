# Setup Guide

이 저장소는 AWS SAM CLI로 배포하도록 패키징되어 있습니다 (`template.yaml`). 아래 "SAM CLI로 배포하기" 섹션을 먼저 참고하시고,
수동으로 하나씩 구성하고 싶다면 그 아래 단계별 가이드를 참고하세요. SAM으로 배포해도 Log Archive 계정 S3 알림 연결,
Secrets Manager 시크릿 생성, geoip Layer 최초 발행 등 일부는 여전히 수동/별도 절차가 필요합니다.

<br>

## SAM CLI로 배포하기

### 사전 준비물

- [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html), Python 3.12
  (로컬에 Python 3.12가 없다면 `sam build --use-container` 사용 — Docker 필요)
- Slack Bot Token을 담은 Secrets Manager 시크릿 (기본 이름: `security-event-app-token`, 키: `slack_bot_token`)
- MaxMind License Key를 담은 Secrets Manager 시크릿 (키: `MAXMIND_LICENSE_KEY`)
- Log Archive 계정 CloudTrail 버킷의 ARN 및 계정 ID (cross-account Lambda invoke 권한 부여용)

시크릿은 템플릿이 생성하지 않습니다. 값 자체(토큰/라이선스 키)를 IaC에 넣지 않기 위해 사전에 직접 생성해야 합니다.

```bash
aws secretsmanager create-secret \
  --name security-event-app-token \
  --secret-string '{"slack_bot_token":"xoxb-..."}'

aws secretsmanager create-secret \
  --name maxmind-license-key \
  --secret-string '{"MAXMIND_LICENSE_KEY":"..."}'
```

### 빌드 및 배포

```bash
sam build
sam deploy --guided
```

`--guided` 진행 중 아래 파라미터를 입력합니다 (`samconfig.toml`에 스캐폴딩되어 있으니 값만 채워도 됩니다).

| 파라미터 | 설명 |
|---------|------|
| `SlackChannelId` | 알림 발송 대상 Slack 채널/DM ID |
| `SlackSecretName` | 위에서 생성한 Slack 시크릿 이름 |
| `MaxmindSecretName` | 위에서 생성한 MaxMind 시크릿 이름 |
| `AllowedCountries` | 정상 국가코드 (기본 `KR`) |
| `AllowedRegions` | 정상 리전 (기본 `ap-northeast-2`) |
| `ErrorThreshold` / `ErrorWindowMin` | 시나리오 3 임계값/윈도우 |
| `AlertCooldownMinutes` | 알림 쿨다운 (기본 30분) |
| `CloudTrailBucketArn` / `CloudTrailBucketAccountId` | Log Archive 계정 CloudTrail 버킷 ARN / 계정 ID |

배포가 완료되면 6개 DynamoDB 테이블과 3개 Lambda 함수가 생성됩니다. `sam deploy` 출력의 Outputs에서
`RefTableProcessorFunctionArn` 등을 확인할 수 있습니다.

### 배포 후 수동 단계 (템플릿만으로는 완결되지 않는 부분)

1. **geoip Layer 최초 발행** — `ref-table-processor`는 배포 직후에는 GeoIP Layer가 연결되어 있지 않습니다
   (Layer 발행/연결은 CloudFormation이 아니라 `geoip-layer-builder` 함수가 Lambda API로 직접 수행하도록 설계되어 있어,
   재배포 시 값이 초기화되지 않게 하기 위함입니다). 아래처럼 한 번 수동 호출하세요.

   ```bash
   aws lambda invoke --function-name <stack-name>-geoip-layer-builder /tmp/out.json && cat /tmp/out.json
   ```

   이후에는 템플릿에 포함된 EventBridge 스케줄(매주 1회, `rate(7 days)`)이 자동으로 갱신합니다.

2. **S3 Event Notification 연결 (cross-account)** — 템플릿은 Log Archive 계정 버킷이 `ref-table-processor`를
   invoke할 수 있도록 `AWS::Lambda::Permission`만 생성합니다. 실제 "Object Created 발생 시 이 Lambda로 알림 전송"
   설정은 Log Archive 계정의 S3 버킷 쪽에서 별도로 구성해야 합니다 (대상 Lambda ARN은 배포 Output 참고).

> 기존에 수동으로 구성해 운영 중인 환경을 이 스택으로 완전히 대체(마이그레이션)하려면 리소스 임포트 등 별도 절차가
> 필요합니다. 이 템플릿은 새 환경(신규 고객사 등)에 동일 시스템을 재현하는 용도로 우선 사용하는 것을 권장합니다.

<br>

## 수동 구성 가이드

아래는 SAM 없이 각 리소스를 직접 구성할 때의 참고 절차입니다.

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
| `SLACK_SECRET_NAME` | Slack Bot Token이 저장된 Secrets Manager 시크릿 이름 | - |
| `ALLOWED_COUNTRIES` | 허용 국가코드 (콤마 구분) | `KR` |
| `ALLOWED_REGIONS` | 허용 리전 (콤마 구분) | `ap-northeast-2` |
| `ERROR_THRESHOLD` | 시나리오 3 AccessDenied 임계값 | `5` |
| `ERROR_WINDOW_MIN` | 시나리오 3 탐지 시간 윈도우 (분) | `5` |
| `ALERT_COOLDOWN_MIN` | 알림 쿨다운 시간 (분) | `30` |
| `AWS_API_TABLE` | ref_aws_api 테이블명 | - |
| `IP_COUNTRY_TABLE` | ref_ip_country 테이블명 | - |
| `REGION_TABLE` | ref_region 테이블명 | - |
| `USER_AGENT_TABLE` | ref_user_agent 테이블명 | - |
| `ERROR_EVENT_TABLE` | ref_error_event 테이블명 | - |
| `ALERT_COOLDOWN_TABLE` | ref_alert_cooldown 테이블명 | - |

> 과거 버전에서는 테이블명/Secret 이름/리전이 코드에 하드코딩되어 있었으나, SAM 패키징 과정에서
> 위 환경변수로 전환했습니다 (환경별 재사용을 위함). Lambda 실행 리전은 별도 환경변수 없이
> Lambda 예약 환경변수 `AWS_REGION`을 boto3가 자동으로 사용합니다.

### geoip-layer-builder

| 변수명 | 설명 | 기본값 |
|-------|------|--------|
| `SECRET_NAME` | MaxMind License Key가 저장된 Secrets Manager 시크릿 이름 | - |
| `LAYER_NAME` | 발행할 Lambda Layer 이름 | `geoip-mmdb` |
| `TARGET_FUNCTION_NAME` | Layer를 연결할 대상 함수 이름 (`ref-table-processor`) | - |

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
