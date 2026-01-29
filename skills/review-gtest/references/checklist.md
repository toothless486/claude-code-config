# 테스트 코드 리뷰 체크리스트

## 구조 (Structure)

- [ ] TEST vs TEST_F vs TEST_P 적절히 선택했는가?
- [ ] Fixture(TEST_F) 사용 시 SetUp/TearDown이 적절한가?
- [ ] 테스트 클래스가 단일 책임을 갖는가?
- [ ] 파라미터화 테스트가 필요한 곳에 TEST_P를 사용했는가?
- [ ] 테스트 파일 구조가 프로덕션 코드 구조와 일치하는가?

## 네이밍 (Naming)

- [ ] 테스트명이 테스트 의도를 명확히 표현하는가?
- [ ] 권장 패턴 사용: `Method_Scenario_Expected` 또는 `Should_Expected_When_Scenario`
- [ ] Fixture 클래스명이 테스트 대상을 명확히 나타내는가?
- [ ] 매직 넘버 대신 의미 있는 상수명을 사용했는가?

```cpp
// ❌ 불명확
TEST(CalcTest, Test1)

// ✅ 명확
TEST(Calculator, Add_TwoPositiveNumbers_ReturnsSum)
TEST(Calculator, Should_ThrowException_When_DivideByZero)
```

## AAA 패턴 (Arrange-Act-Assert)

- [ ] Arrange(준비), Act(실행), Assert(검증) 단계가 명확히 구분되는가?
- [ ] 하나의 테스트에서 하나의 동작만 검증하는가?
- [ ] Arrange 단계가 과도하게 복잡하지 않은가?
- [ ] 주석이나 빈 줄로 단계 구분이 되어 있는가? (선택)

```cpp
TEST_F(UserServiceTest, CreateUser_ValidInput_ReturnsUserId) {
    // Arrange
    UserInput input{"john", "john@example.com"};
    
    // Act
    auto result = service_->CreateUser(input);
    
    // Assert
    EXPECT_TRUE(result.has_value());
    EXPECT_GT(result->id, 0);
}
```

## Assertion

- [ ] EXPECT vs ASSERT를 상황에 맞게 선택했는가?
- [ ] `EXPECT_TRUE(a == b)` 대신 `EXPECT_EQ(a, b)` 사용했는가?
- [ ] 복잡한 조건에 EXPECT_THAT + Matcher를 활용했는가?
- [ ] 부동소수점 비교에 EXPECT_NEAR/EXPECT_FLOAT_EQ 사용했는가?
- [ ] 컬렉션 비교에 적절한 Matcher 사용했는가?
- [ ] 실패 메시지가 명확한가? (필요시 << 연산자로 추가 정보)

```cpp
// ❌ 실패 시 정보 부족
EXPECT_TRUE(result.IsValid());

// ✅ 실패 시 상세 정보 제공
EXPECT_TRUE(result.IsValid()) << "Result: " << result.ToString();

// ✅ Matcher 활용
EXPECT_THAT(items, Contains(expected_item));
EXPECT_THAT(text, HasSubstr("error"));
EXPECT_THAT(values, ElementsAre(1, 2, 3));
```

## Mock

- [ ] EXPECT_CALL vs ON_CALL을 상황에 맞게 선택했는가?
- [ ] Mock 객체가 인터페이스/추상 클래스 기반인가?
- [ ] 과도한 mocking으로 구현 세부사항을 테스트하고 있지 않은가?
- [ ] Strict/Nice/NaggyMock을 적절히 선택했는가?
- [ ] 호출 순서 검증이 필요한 경우 InSequence 사용했는가?
- [ ] Mock 메서드의 기본 반환값이 적절한가?

```cpp
// ❌ 과도한 mocking - 구현 세부사항 테스트
EXPECT_CALL(mock_db, Connect());
EXPECT_CALL(mock_db, BeginTransaction());
EXPECT_CALL(mock_db, Execute(_));
EXPECT_CALL(mock_db, Commit());
EXPECT_CALL(mock_db, Disconnect());

// ✅ 핵심 동작만 검증
EXPECT_CALL(mock_db, Execute(HasSubstr("INSERT")))
    .WillOnce(Return(true));
```

## 독립성 (Independence)

- [ ] 테스트 간 상태 공유가 없는가?
- [ ] 테스트 실행 순서에 의존하지 않는가?
- [ ] 전역 변수/싱글톤 상태에 의존하지 않는가?
- [ ] 외부 리소스(파일, 네트워크, DB) 의존성이 적절히 격리되었는가?
- [ ] 각 테스트가 자체 setup/teardown으로 완결되는가?

## 경계값 및 에러 케이스

- [ ] null/nullptr 케이스 테스트했는가?
- [ ] 빈 컬렉션/문자열 케이스 테스트했는가?
- [ ] 최대/최소 경계값 테스트했는가?
- [ ] 예외 발생 케이스 테스트했는가? (EXPECT_THROW)
- [ ] 타임아웃/실패 케이스 테스트했는가?

```cpp
// 경계값 테스트 예시
TEST_P(BufferTest, HandleEdgeCases) {
    auto size = GetParam();
    Buffer buffer(size);
    EXPECT_EQ(buffer.Size(), size);
}

INSTANTIATE_TEST_SUITE_P(
    EdgeCases,
    BufferTest,
    ::testing::Values(0, 1, MAX_BUFFER_SIZE, MAX_BUFFER_SIZE + 1)
);
```

## 성능 및 리소스

- [ ] 테스트가 합리적인 시간 내에 완료되는가?
- [ ] 리소스(메모리, 파일 핸들 등)가 적절히 해제되는가?
- [ ] 필요한 경우 BENCHMARK 매크로 활용했는가?
- [ ] 대용량 데이터 테스트 시 샘플링을 고려했는가?

## 가독성 및 유지보수

- [ ] 테스트 코드가 문서 역할을 하는가?
- [ ] 중복 코드가 헬퍼 함수나 Fixture로 추출되었는가?
- [ ] 테스트 데이터가 테스트 의도를 명확히 보여주는가?
- [ ] 주석이 꼭 필요한 곳에만 적절히 사용되었는가?
