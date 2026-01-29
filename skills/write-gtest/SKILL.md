---
name: write-gtest
description: GoogleTest/GoogleMock 테스트 코드 작성 skill. 단위 테스트 작성, gtest/gmock 코드 생성, Mock 클래스 생성, 테스트 케이스 추가 요청 시 사용. AAA 패턴, fixture 설계, assertion 선택, mock 설계를 반영한 테스트 코드 생성.
---

# GoogleTest/GoogleMock 테스트 코드 작성

## 작성 워크플로우

1. **대상 분석**: 테스트 대상 코드의 공개 인터페이스 파악
2. **의존성 파악**: Mock이 필요한 외부 의존성 식별
3. **테스트 구조 결정**: TEST vs TEST_F vs TEST_P 선택
4. **테스트 케이스 설계**: 정상/에러/경계값 시나리오 도출
5. **Mock 클래스 작성**: 필요시 인터페이스 기반 Mock 생성
6. **테스트 코드 작성**: AAA 패턴으로 각 테스트 구현
7. **검증**: 테스트가 의도한 동작을 검증하는지 확인

## 테스트 구조 선택 가이드

| 상황 | 권장 구조 |
|------|----------|
| 순수 함수, 독립적 테스트 | TEST |
| 공통 setup 필요, 객체 상태 테스트 | TEST_F |
| 여러 입력값으로 동일 로직 검증 | TEST_P |
| 여러 타입에 동일 로직 검증 | TYPED_TEST |

## 네이밍 규칙

### 테스트명 패턴
```
Method_Scenario_ExpectedResult
```

예시:
- `Add_TwoPositiveNumbers_ReturnsSum`
- `Parse_EmptyInput_ThrowsException`
- `Connect_InvalidUrl_ReturnsFalse`

### Fixture 클래스명
```
[대상클래스명]Test
```

예시: `UserServiceTest`, `DatabaseConnectionTest`

## AAA 패턴 (Arrange-Act-Assert)

```cpp
TEST_F(ServiceTest, Method_Scenario_Expected) {
    // Arrange - 테스트 준비
    auto input = CreateTestInput();
    EXPECT_CALL(*mock_db_, Query(_)).WillOnce(Return("result"));

    // Act - 동작 실행
    auto result = service_->Process(input);

    // Assert - 결과 검증
    EXPECT_EQ(result.status, Status::Success);
    EXPECT_EQ(result.data, "processed");
}
```

## 테스트 케이스 설계 원칙

### 필수 커버리지
1. **정상 케이스**: 유효한 입력에 대한 정상 동작
2. **에러 케이스**: 유효하지 않은 입력에 대한 에러 처리
3. **경계값**: 빈 입력, null, 최대/최소값
4. **예외 상황**: 예외 발생 및 예외 메시지 검증

### 테스트 독립성
- 테스트 간 상태 공유 금지
- 실행 순서에 의존하지 않음
- 각 테스트는 자체 setup/teardown으로 완결

## Assertion 선택 가이드

| 상황 | 권장 Assertion |
|------|---------------|
| 실패해도 계속 실행 | EXPECT_* |
| 실패 시 즉시 중단 (후속 검증 의미 없음) | ASSERT_* |
| 동등 비교 | EXPECT_EQ, EXPECT_NE |
| 비교 연산 | EXPECT_LT, EXPECT_GT, EXPECT_LE, EXPECT_GE |
| 불리언 | EXPECT_TRUE, EXPECT_FALSE |
| 포인터 null 검사 | EXPECT_NE(ptr, nullptr), ASSERT_NE(ptr, nullptr) |
| 문자열 포함 | EXPECT_THAT(str, HasSubstr("text")) |
| 컬렉션 검증 | EXPECT_THAT(vec, ElementsAre(1, 2, 3)) |
| 부동소수점 | EXPECT_NEAR(a, b, tolerance), EXPECT_FLOAT_EQ |
| 예외 발생 | EXPECT_THROW(expr, ExceptionType) |
| 예외 없음 | EXPECT_NO_THROW(expr) |

## Mock 작성 원칙

### EXPECT_CALL vs ON_CALL
- **EXPECT_CALL**: 호출 여부를 검증해야 할 때
- **ON_CALL**: 동작만 정의하고 호출 검증 불필요할 때

### Mock 타입 선택
- **StrictMock**: 모든 호출을 명시적으로 설정 (예상치 못한 호출 시 실패)
- **NiceMock**: 미설정 호출 허용 (기본값 반환)
- **NaggyMock**: 기본값, 미설정 호출 시 경고 출력

## 출력 형식

작성된 테스트 코드는 다음 구조를 따름:

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>
// 필요한 헤더

using ::testing::Return;
using ::testing::_;
// 필요한 using 선언

// Mock 클래스 (필요시)
class MockDependency : public IDependency {
public:
    MOCK_METHOD(ReturnType, MethodName, (Args), (Modifiers));
};

// Test Fixture (필요시)
class TargetClassTest : public ::testing::Test {
protected:
    void SetUp() override {
        // 공통 초기화
    }

    void TearDown() override {
        // 정리
    }

    // 헬퍼 함수
    // 멤버 변수
};

// 테스트 케이스들
TEST_F(TargetClassTest, Method_Scenario_Expected) {
    // Arrange
    // Act
    // Assert
}
```

## 상세 참조

- **템플릿**: references/templates.md
- **패턴**: references/patterns.md
- **Mock 가이드**: references/mock-guide.md
