# Controller TC 결과서 생성 Command

Controller 파일 또는 디렉토리를 분석하여 자체시험 결과서 형식의 TC(Test Case) 문서를 생성한다.

## 사용법
```
/generate-tc <controller-path>
```
- `<controller-path>`: Controller 파일 경로 또는 디렉토리 경로
  - 파일 예: `app-mod/backoffice-app/src/main/java/com/sks/wpm/controller/PushLogController.java`
  - 디렉토리 예: `app-mod/backoffice-app/src/main/java/com/sks/wpm/controller`
- 필수 입력

## 입력 경로
$ARGUMENTS

## 실행 절차

### 1. 입력 경로 분석
입력된 경로가 파일인지 디렉토리인지 확인한다.
- 파일인 경우: 해당 Controller 파일만 분석
- 디렉토리인 경우: 디렉토리 내 모든 `*Controller.java` 파일 분석

### 2. Controller 파일 분석
각 Controller 파일에서 다음 정보를 추출한다:
- 클래스 레벨 `@RequestMapping` 경로 (base path)
- 각 메서드의 HTTP 메서드 타입 (`@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`)
- 각 메서드의 경로
- 각 메서드의 파라미터 (RequestParam, PathVariable, RequestBody 등)
- 각 메서드의 이름과 목적 (JavaDoc 또는 메서드명에서 유추)

### 3. TC 케이스 생성 규칙
각 API endpoint에 대해 다음 TC 케이스들을 생성한다:

#### 성공 케이스
- 필수 파라미터만 사용한 기본 조회/요청
- 각 선택적 검색 조건별 성공 케이스
- 데이터가 존재하지 않는 경우 (빈 목록 반환)

#### 실패 케이스
- 필수 파라미터 누락 (각 필수 파라미터별로)
- 유효하지 않은 데이터 포맷 (날짜, 숫자 등)
- 유효하지 않은 파라미터 값 (범위 초과, 잘못된 타입 등)
- 잘못된 날짜 범위 (startDate > endDate)
- 존재하지 않는 리소스 조회/수정/삭제

### 4. TC ID 생성 규칙
- 형식: `TC_{분류번호}_{일련번호}`
- 일련번호: 001부터 시작하여 3자리 숫자로 패딩

### 5. 출력 파일 생성

#### 출력 파일 경로
- 형식: `docs/tc/{controller-name}_controller.md`
- controller-name: Controller 클래스명에서 'Controller' 제외한 소문자 스네이크 케이스
- 예: `PushLogController.java` → `docs/tc/push_log_controller.md`

#### 출력 파일 형식

```markdown
# {Controller명} 자체시험 결과서

- 생성일: {현재날짜}
- 대상 파일: `{controller 파일 경로}`

## 표지

| 항목 | TOTAL | PASS | NOT TESTED | FAIL | 진행율 |
| --- | --- | --- | --- | --- | --- |
| TC_{분류번호}_{중분류명} | {총개수} | 0 | {총개수} | 0 | 0 |
| TOTAL | {총개수} | 0 | {총개수} | 0 | 0 |

## TC_{분류번호}_{중분류명}

| ID | 대분류 | 중분류 | 소분류 | 상세분류 | Precondition | Test Procedure | Expected Result | result | API Test Collection | API Test ID | API Test Parameter |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| TC_{분류번호}_{일련번호} | {대분류} | {중분류} | {소분류} | {상세분류} | {사전조건} | {테스트절차} | {예상결과} | Not Test | {API URL} | {API Test ID} | {파라미터목록} |
```

### 6. 필드 작성 규칙

#### 대분류
- 백오피스 Controller: "백오피스"
- 그 외: Controller의 도메인에 따라 결정

#### 중분류
- Controller의 주요 기능명 (예: "Push발송 이력", "펌웨어 이력", "삭제 기기")

#### 소분류
- API의 기능명 (예: "Push발송 이력 조회", "펌웨어 이력 조회")

#### 상세분류
- TC 케이스 유형 (예: "성공 - 필수 파라미터만 사용한 기본 조회", "실패 - startDate 누락")

#### Precondition
- 테스트 수행을 위한 사전 조건
- 필요한 데이터베이스 레코드 존재 여부
- 인증 정보 필요 여부

#### Test Procedure
1. API 요청 방법
2. 확인할 항목

#### Expected Result
- 성공 케이스: JSON 응답 예시 (resultCode: 0, resultMsg: "성공", data: {...})
- 실패 케이스: JSON 응답 예시 (resultCode: 400, resultMsg: "옳지 않은 파라미터 : {파라미터명}")

#### result
- 초기값: "Not Test"
- 테스트 후 "PASS" 또는 "FAIL"로 변경

#### API Test Collection
- HTTP 메서드와 전체 URL 경로
- 예: `GET {{WPM_BACKOFFICE_APP_URL}}/api/v1/backoffice/push/log?startDate={{startDate}}&endDate={{endDate}}`

#### API Test ID
- Postman Collection 테스트 ID 형식
- 예: `01. R1.0.0 > TC_011_Backoffice 관리 > 001_Push발송이력조회_성공_필수파라미터`

#### API Test Parameter
- 테스트에 사용되는 파라미터 목록
- 각 파라미터의 설명 또는 예시값 포함

## 환경 설정 정보

### DB 연결 정보
```yaml
# application.yml 구조
spring:
  profiles:
    active: ${PROFILE}  # 환경변수로 프로파일 지정 (dev, stg, live, local)
    include:
      - "data-jdbc-${spring.profiles.active}"  # DB 설정은 외부 Config 서버에서 주입
```

**프로파일**:
- `local` - 로컬 개발
- `dev` - 개발 서버
- `stg` - 스테이징 서버
- `live` - 운영 서버

### API 인증 정보
```
Header Name: X-Header-Extra-Info
Header Value: AES 암호화된 JSON (userId, userName 포함)
```

**인증이 필요한 API**: `@UserCert` 어노테이션이 붙은 엔드포인트

**인증 처리 위치**: `core-mod/src/main/java/com/sks/wpm/core/aop/UserCertAspect.java`

## 실행 절차 (확장)

### 7. 테이블 매핑 정보 추출
Controller → Service → Repository → Entity 의존성을 추적하여 관련 테이블 정보를 추출한다.

#### 추적 경로
1. Controller에서 주입된 Service 인터페이스 확인
2. Service 구현체에서 주입된 Repository 확인
3. Repository에서 사용하는 Entity 클래스 확인
4. Entity 클래스의 `@Table` 어노테이션에서 테이블명 추출

#### 경로 패턴
- Service: `app-mod/{app}/src/main/java/com/sks/wpm/service/**/*.java`
- Repository: `domain-mod/src/main/java/com/sks/wpm/repository/**/*.java`
- Entity: `domain-mod/src/main/java/com/sks/wpm/entity/**/*.java`

### 8. Setup/Teardown SQL 생성
각 TC 케이스에 필요한 테스트 데이터 SQL을 생성한다.

#### Setup SQL (테스트 전)
```sql
-- 테스트 데이터 식별자: TEST_ prefix 또는 특정 ID 범위 사용
INSERT INTO {테이블명} ({컬럼목록})
VALUES ({테스트값목록});
```

#### Teardown SQL (테스트 후)
```sql
-- 테스트 데이터 정리
DELETE FROM {테이블명} WHERE {조건};
-- 또는 트랜잭션 롤백 사용
```

### 9. API 호출 명령어 생성

#### curl 명령어 형식
```bash
# GET 요청 예시
curl -X GET "{{BASE_URL}}/backoffice/{endpoint}?{params}" \
  -H "Content-Type: application/json" \
  -H "X-Header-Extra-Info: {{AUTH_KEY}}"

# POST 요청 예시
curl -X POST "{{BASE_URL}}/backoffice/{endpoint}" \
  -H "Content-Type: application/json" \
  -H "X-Header-Extra-Info: {{AUTH_KEY}}" \
  -d '{request_body}'
```

#### 환경변수
- `BASE_URL`: API 서버 주소 (예: `http://localhost:8080`)
- `AUTH_KEY`: AES 암호화된 인증 토큰

## 출력 파일 형식 (확장)

생성되는 TC 문서에 다음 섹션을 추가한다:

```markdown
## 테이블 매핑 정보

| Entity | Table | 용도 |
| --- | --- | --- |
| {Entity클래스명} | {테이블명} | {설명} |

## 테스트 데이터 Setup/Teardown

### Setup SQL
\`\`\`sql
-- TC 테스트를 위한 데이터 생성
{INSERT 문}
\`\`\`

### Teardown SQL
\`\`\`sql
-- TC 테스트 후 데이터 정리
{DELETE 문}
\`\`\`

## API 호출 예시

### 인증 토큰 생성
\`\`\`bash
# AUTH_KEY 환경변수 설정 (실제 값은 보안상 별도 관리)
export AUTH_KEY="your_encrypted_auth_key"
export BASE_URL="http://localhost:8080"
\`\`\`

### curl 명령어
\`\`\`bash
{각 TC별 curl 명령어}
\`\`\`
```

## 참고사항

- 생성된 TC 문서는 초기 템플릿이며, 실제 테스트 수행 후 result 필드를 업데이트해야 한다.
- Expected Result의 JSON 응답은 실제 API 스펙에 맞게 수정이 필요할 수 있다.
- 새로운 API가 추가되면 해당 TC 케이스를 추가해야 한다.
- Setup/Teardown SQL은 실제 테이블 스키마에 맞게 수정이 필요할 수 있다.
- AUTH_KEY는 보안상 환경변수로 관리하고, 문서에 직접 노출하지 않는다.
