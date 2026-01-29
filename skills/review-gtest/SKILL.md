---
name: gtest-review
description: GoogleTest/GoogleMock 테스트 코드 리뷰 skill. 테스트 코드 리뷰, 테스트 품질 개선, gtest/gmock 코드 검토, 단위 테스트 피드백 요청 시 사용. 테스트 구조, assertion 사용, mock 설계, 테스트 독립성, 네이밍 컨벤션, AAA 패턴, 안티패턴을 검토하고 개선안 제시.
---

# GoogleTest/GoogleMock 테스트 코드 리뷰

## 리뷰 워크플로우

1. **구조 분석**: TEST vs TEST_F vs TEST_P 선택 적절성 확인
2. **네이밍 검토**: 테스트명이 의도를 명확히 표현하는지 확인
3. **AAA 패턴**: Arrange-Act-Assert 구조 명확성 검토
4. **Assertion 검토**: EXPECT vs ASSERT 선택, matcher 활용
5. **Mock 설계**: 과도한 mocking 여부, Mock 타입 적절성
6. **안티패턴 탐지**: 체크리스트 기반 문제점 식별
7. **개선안 제시**: 구체적 코드 수정 제안

## 심각도 분류

| 레벨 | 설명 | 예시 |
|------|------|------|
| 🔴 Critical | 테스트 신뢰성 훼손 | 테스트 간 의존성, 미검증 assertion, 항상 통과하는 테스트 |
| 🟡 Major | 유지보수성 저하 | 불명확한 네이밍, 중복 코드, 과도한 mocking |
| 🟢 Minor | 개선 권장 | 스타일, 가독성, 매직 넘버 |

## 리뷰 출력 형식

```
## 테스트 코드 리뷰 결과

### 요약
- 검토 파일: [파일명]
- Critical: N건 / Major: N건 / Minor: N건

### 🔴 Critical Issues
[번호]. [이슈 제목]
- 위치: [파일:라인]
- 문제: [설명]
- 개선안: [코드 또는 설명]

### 🟡 Major Issues
...

### 🟢 Minor Issues
...

### 👍 잘된 점
...
```

## 핵심 체크포인트

### 테스트 구조
- TEST: 독립적 단순 테스트
- TEST_F: 공통 setup 필요 시 (fixture)
- TEST_P: 파라미터화 테스트 (여러 입력값)
- TYPED_TEST: 여러 타입에 동일 로직

### Assertion 선택
| 상황 | 권장 |
|------|------|
| 실패해도 계속 실행 | EXPECT_* |
| 실패 시 즉시 중단 | ASSERT_* |
| 복잡한 조건 검증 | EXPECT_THAT + Matcher |
| 부동소수점 비교 | EXPECT_NEAR, EXPECT_FLOAT_EQ |

### Mock 체크포인트
- EXPECT_CALL: 호출 여부 검증 필요 시
- ON_CALL: 동작만 정의, 호출 검증 불필요 시
- StrictMock: 모든 호출 명시 필요
- NiceMock: 미지정 호출 허용 (기본값 반환)

## 상세 참조

- **체크리스트**: references/checklist.md
- **안티패턴**: references/anti-patterns.md
- **베스트 프랙티스**: references/best-practices.md
