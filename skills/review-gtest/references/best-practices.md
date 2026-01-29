# GoogleTest/GoogleMock 베스트 프랙티스

## 테스트 구조 선택

### TEST vs TEST_F vs TEST_P

| 유형 | 사용 시점 | 예시 |
|------|----------|------|
| TEST | 독립적, setup 불필요 | 순수 함수 테스트 |
| TEST_F | 공통 setup 필요 | 클래스 메서드 테스트 |
| TEST_P | 여러 입력값 검증 | 경계값, 다양한 케이스 |
| TYPED_TEST | 여러 타입 동일 로직 | 템플릿 클래스 테스트 |

```cpp
// TEST: 단순 함수
TEST(MathUtils, Abs_NegativeInput_ReturnsPositive) {
    EXPECT_EQ(Abs(-5), 5);
}

// TEST_F: 객체 상태 필요
class StackTest : public ::testing::Test {
protected:
    void SetUp() override {
        stack_.Push(1);
        stack_.Push(2);
    }
    Stack<int> stack_;
};

TEST_F(StackTest, Pop_NonEmpty_ReturnsTopElement) {
    EXPECT_EQ(stack_.Pop(), 2);
}

// TEST_P: 여러 입력값
class PrimeTest : public ::testing::TestWithParam<std::pair<int, bool>> {};

TEST_P(PrimeTest, IsPrime_ReturnsExpected) {
    auto [input, expected] = GetParam();
    EXPECT_EQ(IsPrime(input), expected);
}

INSTANTIATE_TEST_SUITE_P(
    Primes, PrimeTest,
    ::testing::Values(
        std::make_pair(2, true),
        std::make_pair(4, false),
        std::make_pair(17, true)
    )
);
```

## Matcher 활용

### 기본 Matcher

```cpp
// 문자열
EXPECT_THAT(str, StartsWith("Hello"));
EXPECT_THAT(str, EndsWith("World"));
EXPECT_THAT(str, HasSubstr("middle"));
EXPECT_THAT(str, MatchesRegex("^[a-z]+$"));

// 숫자
EXPECT_THAT(value, Gt(10));        // >
EXPECT_THAT(value, Ge(10));        // >=
EXPECT_THAT(value, Lt(100));       // <
EXPECT_THAT(value, Le(100));       // <=
EXPECT_THAT(value, AllOf(Gt(0), Lt(100)));  // AND
EXPECT_THAT(value, AnyOf(Eq(1), Eq(2)));    // OR

// 포인터
EXPECT_THAT(ptr, NotNull());
EXPECT_THAT(ptr, IsNull());
EXPECT_THAT(ptr, Pointee(Gt(0)));  // *ptr > 0
```

### 컨테이너 Matcher

```cpp
std::vector<int> v = {1, 2, 3};

// 정확한 매칭
EXPECT_THAT(v, ElementsAre(1, 2, 3));         // 순서+내용 일치
EXPECT_THAT(v, UnorderedElementsAre(3, 1, 2)); // 순서 무관

// 부분 매칭
EXPECT_THAT(v, Contains(2));
EXPECT_THAT(v, Each(Gt(0)));          // 모든 요소가 > 0
EXPECT_THAT(v, IsEmpty());            // 비어있음
EXPECT_THAT(v, SizeIs(3));            // 크기 확인

// 조합
EXPECT_THAT(v, AllOf(Contains(1), SizeIs(Gt(2))));
```

### 사용자 정의 Matcher

```cpp
MATCHER_P(IsEvenAndGreaterThan, threshold, "") {
    return (arg % 2 == 0) && (arg > threshold);
}

TEST(Numbers, CustomMatcher) {
    EXPECT_THAT(10, IsEvenAndGreaterThan(5));
}
```

## Mock 설계

### 인터페이스 기반 설계

```cpp
// 인터페이스 정의
class IDatabase {
public:
    virtual ~IDatabase() = default;
    virtual bool Connect(const std::string& url) = 0;
    virtual std::optional<std::string> Query(const std::string& sql) = 0;
};

// Mock 생성
class MockDatabase : public IDatabase {
public:
    MOCK_METHOD(bool, Connect, (const std::string& url), (override));
    MOCK_METHOD(std::optional<std::string>, Query, (const std::string& sql), (override));
};
```

### EXPECT_CALL vs ON_CALL

```cpp
// EXPECT_CALL: 호출 검증 필요
TEST(Service, Process_CallsDatabase) {
    MockDatabase db;
    EXPECT_CALL(db, Query(HasSubstr("SELECT")))
        .Times(1)
        .WillOnce(Return("result"));
    
    Service service(&db);
    service.Process();
    // 테스트 종료 시 Query 호출 여부 자동 검증
}

// ON_CALL: 동작만 정의 (호출 검증 불필요)
TEST(Service, Process_ReturnsExpectedResult) {
    MockDatabase db;
    ON_CALL(db, Query(_))
        .WillByDefault(Return("default_result"));
    
    Service service(&db);
    auto result = service.Process();
    
    EXPECT_EQ(result, "processed: default_result");
}
```

### 호출 순서 검증

```cpp
TEST(TransactionManager, ExecuteInOrder) {
    MockDatabase db;
    
    {
        InSequence seq;
        EXPECT_CALL(db, BeginTransaction());
        EXPECT_CALL(db, Execute(_));
        EXPECT_CALL(db, Commit());
    }
    
    TransactionManager tm(&db);
    tm.ExecuteTransaction("INSERT ...");
}
```

### Mock 타입 선택

```cpp
// StrictMock: 예상치 못한 호출 시 즉시 실패
StrictMock<MockDatabase> strict_db;

// NiceMock: 예상치 못한 호출 허용 (기본값 반환)
NiceMock<MockDatabase> nice_db;

// NaggyMock (기본): 예상치 못한 호출 시 경고
MockDatabase naggy_db;
```

## 테스트 데이터 관리

### 상수 활용

```cpp
namespace test_constants {
    constexpr int kValidUserId = 12345;
    constexpr int kInvalidUserId = -1;
    constexpr char kValidEmail[] = "test@example.com";
    constexpr char kInvalidEmail[] = "not-an-email";
}

TEST(UserValidator, Validate_ValidEmail_ReturnsTrue) {
    using namespace test_constants;
    EXPECT_TRUE(validator.ValidateEmail(kValidEmail));
}
```

### 헬퍼 함수

```cpp
class UserServiceTest : public ::testing::Test {
protected:
    // 테스트용 객체 생성 헬퍼
    User CreateTestUser(const std::string& name = "test_user") {
        return User{
            .id = next_id_++,
            .name = name,
            .email = name + "@example.com",
            .created_at = Clock::Now()
        };
    }
    
    // 검증 헬퍼
    void ExpectUserEquals(const User& actual, const User& expected) {
        EXPECT_EQ(actual.id, expected.id);
        EXPECT_EQ(actual.name, expected.name);
        EXPECT_EQ(actual.email, expected.email);
    }

private:
    int next_id_ = 1;
};
```

## 비동기 테스트

### 조건 기반 대기

```cpp
// 타임아웃과 함께 조건 대기
template<typename Predicate>
bool WaitUntil(Predicate pred, std::chrono::milliseconds timeout) {
    auto deadline = std::chrono::steady_clock::now() + timeout;
    while (std::chrono::steady_clock::now() < deadline) {
        if (pred()) return true;
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
    return pred();
}

TEST(AsyncService, Start_Completes_WithinTimeout) {
    AsyncService service;
    service.Start();
    
    ASSERT_TRUE(WaitUntil(
        [&]{ return service.IsReady(); },
        std::chrono::seconds(5)
    )) << "Service failed to become ready";
}
```

### Future/Promise 활용

```cpp
TEST(AsyncProcessor, Process_ReturnsResult) {
    AsyncProcessor processor;
    auto future = processor.ProcessAsync(input);
    
    ASSERT_EQ(future.wait_for(std::chrono::seconds(5)), 
              std::future_status::ready);
    EXPECT_EQ(future.get(), expected_result);
}
```

## 리소스 관리

### RAII 패턴

```cpp
class TempFileTest : public ::testing::Test {
protected:
    void SetUp() override {
        temp_path_ = CreateTempFile();
    }
    
    void TearDown() override {
        if (!temp_path_.empty()) {
            std::remove(temp_path_.c_str());
        }
    }
    
    std::string temp_path_;
};
```

### 스마트 포인터 활용

```cpp
class ServiceTest : public ::testing::Test {
protected:
    void SetUp() override {
        mock_db_ = std::make_unique<MockDatabase>();
        service_ = std::make_unique<Service>(mock_db_.get());
    }
    
    std::unique_ptr<MockDatabase> mock_db_;
    std::unique_ptr<Service> service_;
};
```

## 예외 테스트

```cpp
// 특정 예외 타입 검증
TEST(Parser, Parse_InvalidInput_ThrowsParseError) {
    EXPECT_THROW(parser.Parse("invalid"), ParseError);
}

// 예외 메시지 검증
TEST(Parser, Parse_InvalidInput_ThrowsWithMessage) {
    try {
        parser.Parse("invalid");
        FAIL() << "Expected ParseError";
    } catch (const ParseError& e) {
        EXPECT_THAT(e.what(), HasSubstr("invalid syntax"));
    }
}

// 예외 없음 검증
TEST(Parser, Parse_ValidInput_DoesNotThrow) {
    EXPECT_NO_THROW(parser.Parse("valid input"));
}
```

## 사망 테스트 (Death Test)

```cpp
// ASSERT 실패, abort() 등 검증
TEST(DebugUtils, CheckFailed_Aborts) {
    EXPECT_DEATH(CHECK(false), "Check failed");
}

// 정규식으로 메시지 매칭
TEST(Validator, NullPointer_Dies) {
    EXPECT_DEATH(validator.Process(nullptr), 
                 ".*null.*pointer.*");
}
```

## 성능 테스트 (Google Benchmark 연동 시)

```cpp
// 벤치마크 예시 (별도 파일)
static void BM_VectorPush(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<int> v;
        for (int i = 0; i < state.range(0); ++i) {
            v.push_back(i);
        }
    }
}
BENCHMARK(BM_VectorPush)->Range(8, 8<<10);
```
