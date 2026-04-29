# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 디렉토리 구조 주의사항

리포지토리 최상위에 동일한 이름의 하위 디렉토리(`architecture-diagram-generator/architecture-diagram-generator/`)가 중첩되어 있습니다. 실제 소스 코드는 모두 **하위 디렉토리** 안에 있습니다. 모든 경로 참조 및 작업은 해당 하위 디렉토리 기준으로 진행해야 합니다.

## 주요 명령어

### Lambda 단위 테스트

각 Lambda 함수에는 자체 `unit_tests.py`가 있습니다. 실행은 해당 함수 디렉토리에서 수행:

```bash
cd architecture-diagram-generator/lambda_function/<function_name>
python -m unittest unit_tests.py
```

`<function_name>`은 `translate_to_pseudocode`, `generate_uml_code`, `generate_diagram` 중 하나입니다.

### 인프라 배포 (Terraform)

```bash
cd architecture-diagram-generator/terraform
terraform init
terraform apply -var="claude_api_key=<KEY>" -var="plantuml_layer_arn=<ARN>"
```

`s3_bucket_name`과 `plantuml_layer_arn`은 `variables.tf`에 `"hidden in public"` 플레이스홀더로 남아 있어 실제 값을 변수로 주입해야 합니다.

### Lambda 패키징

Terraform은 `lambda_function.zip`을 참조하므로 배포 전 각 Lambda 디렉토리에서 zip 패키징이 필요합니다. 또한 Terraform의 zip 경로가 `lambda_functions/`(복수형)을 가리키지만 실제 디렉토리는 `lambda_function/`(단수형)입니다 — 배포 전 이 불일치를 해결해야 합니다.

## 아키텍처 개요

### 처리 파이프라인

사용자가 코드 파일을 업로드하면 다음 순서로 처리됩니다:

```
Frontend → API Gateway → Step Functions → [3개의 Lambda 순차 실행] → S3 → Frontend
```

Step Functions(`step_functions.tf`의 `DiagramGeneratorStateMachine`)가 Lambda 3개를 순차 호출하며, 각 단계는 ServiceException 등에 대해 6회 retry 정책을 가집니다. 단계 간 데이터는 S3 객체 키로만 전달됩니다(코드 본문이나 의사코드 자체를 페이로드에 싣지 않음).

### Lambda 간 데이터 흐름

각 Lambda는 이전 단계의 S3 키를 받아 결과물을 다시 S3에 올리고 다음 키만 반환합니다:

1. **`translate_to_pseudocode`**: `event['code_content']`(base64) → Claude API(`v1/complete`, `claude-2` 모델) 호출 → `pseudocode.txt`로 업로드 → `{'pseudocode_key': ...}` 반환.
2. **`generate_uml_code`**: `event['pseudocode_key']` → S3에서 의사코드 다운로드 → 정규식이 아닌 단순 문자열 파싱(`function`, `calls` 키워드 매칭)으로 PlantUML 텍스트 생성 → `uml_code.puml` 업로드.
3. **`generate_diagram`**: `event['uml_code_key']` → PlantUML jar(`/opt/plantuml.jar`, Lambda Layer로 마운트)로 SVG 생성 → S3 업로드 후 세 결과물의 **presigned URL**을 반환.

### 환경별 설정 로딩

모든 Lambda는 `os.environ['ENV']`(`dev`|`prod`) 값에 따라 `config.dev_config` 또는 `config.prod_config`에서 `CLAUDE_API_KEY`/`S3_BUCKET_NAME`을 import합니다. 그러나 **`config/` 디렉토리는 현재 리포지토리에 존재하지 않습니다** — Lambda 패키지를 만들 때 직접 추가해야 동작합니다(README의 "Project Structure"에는 명시).

### Terraform 리소스 중복

`terraform/` 디렉토리에 동일한 리소스가 여러 파일에 중복 정의되어 있습니다 — `main.tf`가 사실상 모든 것(S3 버킷, 3개 Lambda, API Gateway)을 포함하면서, `lambda.tf`/`s3.tf`/`api_gateway.tf`도 같은 이름의 리소스를 또 정의합니다. 또한 두 API Gateway 정의가 다릅니다:

- `main.tf`: Lambda 3개를 각각 `/translate`, `/generate-uml`, `/generate-diagram` 엔드포인트로 직접 노출 (`AWS_PROXY` 통합).
- `api_gateway.tf`: 단일 `/generate` 엔드포인트가 Step Functions `StartExecution`을 호출 (`AWS` 통합).

`terraform apply` 전에 어느 쪽을 사용할지 결정하고 다른 쪽을 제거해야 합니다. README의 "Workflow"는 후자(Step Functions를 통한 단일 엔드포인트) 모델을 전제로 합니다.

### Frontend ↔ Backend 계약 불일치

- `frontend/script.js`는 `executionArn`을 받고 `/execution-status` 엔드포인트를 폴링하지만, 어떤 Terraform 파일에도 해당 엔드포인트가 정의되어 있지 않습니다.
- `script.js`의 API URL은 `https://your-api-gateway-endpoint/prod/...` 플레이스홀더이므로 배포 후 실제 URL로 교체해야 합니다.
- `unit_tests.py`의 일부 테스트는 `event['body']` / `isBase64Encoded` / `statusCode` 형태(API Gateway proxy 형식)를 가정하지만, 실제 Lambda handler는 Step Functions에서 호출되므로 `event['code_content']`, `event['pseudocode_key']` 등을 직접 읽고 dict를 반환합니다. 테스트 자체가 핸들러와 일치하지 않습니다.

## 작업 시 참고 사항

- Claude API 호출이 구버전 형식(`/v1/complete`, `claude-2`, `prompt`/`max_tokens_to_sample`)을 사용합니다. 신규 작업으로 모델/엔드포인트를 갱신할 때는 Messages API(`/v1/messages`)로 마이그레이션해야 하며, 현재 인증도 `x-api-key`만 보내고 `anthropic-version` 헤더가 누락되어 있습니다.
- `generate_uml_code`의 PlantUML 생성기는 의사코드의 특정 표현(`function name(...)`, `X calls Y(...)`)에만 의존하는 매우 좁은 파서입니다. 의사코드 포맷이 바뀌면 UML이 비어버리므로 두 Lambda의 prompt/파서를 함께 수정해야 합니다.
