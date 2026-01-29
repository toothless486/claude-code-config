# GoogleMock 작성 가이드

## Mock 클래스 기본 구조

### 인터페이스 정의

Mock을 만들기 전에 인터페이스(추상 클래스)를 정의:

```cpp
// 인터페이스 정의 (헤더 파일)
class IDatabase {
public:
    virtual ~IDatabase() = default;

    virtual bool Connect(const std::string& url) = 0;
    virtual void Disconnect() = 0;

    virtual std::shared_ptr<IResultSet> Select(
        const std::string& query,
        DatabaseError& error) = 0;

    virtual bool Insert(const std::string& query, DatabaseError& error) = 0;
    virtual bool Update(const std::string& query, DatabaseError& error) = 0;
    virtual bool Remove(const std::string& query, DatabaseError& error) = 0;
};

class IResultSet {
public:
    virtual ~IResultSet() = default;

    virtual bool next() = 0;
    virtual int getItemInt(int index) = 0;
    virtual std::string getItemString(int index, int maxLen = 1024) = 0;
    virtual double getItemDouble(int index) = 0;
};
```

### Mock 클래스 생성

```cpp
// Mock 클래스 (테스트용 헤더)
#include <gmock/gmock.h>
#include "IDatabase.h"

class MockDatabase : public IDatabase {
public:
    // 기본 메서드
    MOCK_METHOD(bool, Connect, (const std::string& url), (override));
    MOCK_METHOD(void, Disconnect, (), (override));

    // 참조 파라미터가 있는 메서드
    MOCK_METHOD(std::shared_ptr<IResultSet>, Select,
                (const std::string& query, DatabaseError& error), (override));
    MOCK_METHOD(bool, Insert,
                (const std::string& query, DatabaseError& error), (override));
    MOCK_METHOD(bool, Update,
                (const std::string& query, DatabaseError& error), (override));
    MOCK_METHOD(bool, Remove,
                (const std::string& query, DatabaseError& error), (override));
};

class MockResultSet : public IResultSet {
public:
    MOCK_METHOD(bool, next, (), (override));
    MOCK_METHOD(int, getItemInt, (int index), (override));
    MOCK_METHOD(std::string, getItemString, (int index, int maxLen), (override));
    MOCK_METHOD(double, getItemDouble, (int index), (override));
};
```

## MOCK_METHOD 문법

### 기본 문법

```cpp
MOCK_METHOD(반환타입, 메서드명, (파라미터들), (한정자들));
```

### 한정자 (Qualifiers)

| 한정자 | 설명 |
|--------|------|
| `override` | 가상 함수 오버라이드 |
| `const` | const 메서드 |
| `noexcept` | noexcept 메서드 |
| `ref(&)` | lvalue 참조 한정자 |
| `ref(&&)` | rvalue 참조 한정자 |

```cpp
// const 메서드
MOCK_METHOD(int, GetValue, (), (const, override));

// noexcept 메서드
MOCK_METHOD(void, SafeMethod, (), (noexcept, override));

// 복합 한정자
MOCK_METHOD(int, ConstNoexceptMethod, (), (const, noexcept, override));
```

## EXPECT_CALL vs ON_CALL

### EXPECT_CALL - 호출 검증 필요할 때

```cpp
TEST_F(ServiceTest, Process_CallsDatabase) {
    // EXPECT_CALL: 호출 여부와 파라미터 검증
    EXPECT_CALL(*mock_db_, Select(HasSubstr("SELECT * FROM users"), _))
        .Times(1)
        .WillOnce(Return(mock_result_));

    service_->Process();

    // 테스트 종료 시 자동으로 호출 여부 검증
}
```

### ON_CALL - 동작만 정의할 때

```cpp
TEST_F(ServiceTest, Process_ReturnsExpectedResult) {
    // ON_CALL: 호출되면 이 동작 수행 (호출 검증 없음)
    ON_CALL(*mock_db_, Select(_, _))
        .WillByDefault(Return(mock_result_));

    auto result = service_->Process();

    // 결과만 검증 (Select 호출 여부는 검증 안 함)
    EXPECT_EQ(result, expected_value_);
}
```

### 언제 무엇을 사용할까?

| 상황 | 권장 |
|------|------|
| 메서드가 반드시 호출되어야 함 | EXPECT_CALL |
| 호출 파라미터 검증 필요 | EXPECT_CALL |
| 호출 순서 검증 필요 | EXPECT_CALL + InSequence |
| 단순히 반환값만 설정 | ON_CALL |
| 테스트의 주요 관심사가 아님 | ON_CALL |

## EXPECT_CALL 상세

### Times() - 호출 횟수

```cpp
EXPECT_CALL(*mock, Method(_))
    .Times(1);           // 정확히 1번

EXPECT_CALL(*mock, Method(_))
    .Times(AtLeast(2));  // 최소 2번

EXPECT_CALL(*mock, Method(_))
    .Times(AtMost(5));   // 최대 5번

EXPECT_CALL(*mock, Method(_))
    .Times(Between(2, 5)); // 2~5번

EXPECT_CALL(*mock, Method(_))
    .Times(AnyNumber());  // 0번 이상
```

### WillOnce() / WillRepeatedly()

```cpp
// 한 번만 특정 값 반환
EXPECT_CALL(*mock, GetValue())
    .WillOnce(Return(42));

// 여러 번 호출, 각각 다른 값 반환
EXPECT_CALL(*mock, GetValue())
    .WillOnce(Return(1))
    .WillOnce(Return(2))
    .WillOnce(Return(3));

// 기본값 반복 반환
EXPECT_CALL(*mock, GetValue())
    .WillRepeatedly(Return(100));

// 조합: 처음 2번은 특정 값, 이후는 기본값
EXPECT_CALL(*mock, GetValue())
    .WillOnce(Return(1))
    .WillOnce(Return(2))
    .WillRepeatedly(Return(99));
```

### 파라미터 Matcher

```cpp
// 어떤 값이든 허용
EXPECT_CALL(*mock, Method(_));

// 특정 값
EXPECT_CALL(*mock, Method(42));
EXPECT_CALL(*mock, Method(Eq(42)));

// 비교
EXPECT_CALL(*mock, Method(Gt(10)));   // > 10
EXPECT_CALL(*mock, Method(Le(100)));  // <= 100

// 문자열
EXPECT_CALL(*mock, Query(HasSubstr("SELECT")));
EXPECT_CALL(*mock, Query(StartsWith("INSERT")));
EXPECT_CALL(*mock, Query(MatchesRegex(".*WHERE.*")));

// 조합
EXPECT_CALL(*mock, Method(AllOf(Gt(0), Lt(100))));  // AND
EXPECT_CALL(*mock, Method(AnyOf(1, 2, 3)));         // OR
EXPECT_CALL(*mock, Method(Not(0)));                 // NOT

// 포인터
EXPECT_CALL(*mock, Process(NotNull()));
EXPECT_CALL(*mock, Process(IsNull()));
```

## Action (동작 정의)

### 기본 Action

```cpp
// 값 반환
EXPECT_CALL(*mock, GetValue())
    .WillOnce(Return(42));

// void 반환 (아무것도 안 함)
EXPECT_CALL(*mock, DoSomething())
    .WillOnce(Return());

// 참조로 반환
EXPECT_CALL(*mock, GetRef())
    .WillOnce(ReturnRef(some_object_));

// 참조 파라미터 설정
EXPECT_CALL(*mock, GetOutput(_, _))
    .WillOnce(DoAll(
        SetArgReferee<0>(output_value),  // 첫 번째 파라미터 설정
        SetArgReferee<1>(DatabaseError()),  // 두 번째 파라미터 설정
        Return(true)
    ));

// 포인터 파라미터 설정
EXPECT_CALL(*mock, GetOutput(_))
    .WillOnce(DoAll(
        SetArgPointee<0>(value),
        Return(true)
    ));
```

### 예외 던지기

```cpp
EXPECT_CALL(*mock, RiskyMethod())
    .WillOnce(Throw(std::runtime_error("error")));
```

### Lambda 사용

```cpp
EXPECT_CALL(*mock, Process(_))
    .WillOnce([](const Input& input) {
        return Result{input.id * 2, "processed"};
    });

// 참조 파라미터 수정
EXPECT_CALL(*mock, FillData(_, _))
    .WillOnce([](std::string& out, DatabaseError& err) {
        out = "filled_data";
        err.set(0, "");
        return true;
    });
```

### 여러 Action 조합 (DoAll)

```cpp
EXPECT_CALL(*mock, Query(_, _))
    .WillOnce(DoAll(
        SetArgReferee<1>(DatabaseError()),  // error 파라미터 설정
        Return(mock_result_)                // 값 반환
    ));
```

## 호출 순서 검증

### InSequence

```cpp
TEST_F(TransactionTest, ExecuteInOrder) {
    {
        InSequence seq;
        EXPECT_CALL(*mock_db_, BeginTransaction());
        EXPECT_CALL(*mock_db_, Execute(_));
        EXPECT_CALL(*mock_db_, Commit());
    }

    service_->ExecuteTransaction(query);
}
```

### 복수 시퀀스

```cpp
TEST_F(ServiceTest, ParallelOperations) {
    Sequence s1, s2;

    // 시퀀스 1: A -> B
    EXPECT_CALL(*mock_, A()).InSequence(s1);
    EXPECT_CALL(*mock_, B()).InSequence(s1);

    // 시퀀스 2: C -> D
    EXPECT_CALL(*mock_, C()).InSequence(s2);
    EXPECT_CALL(*mock_, D()).InSequence(s2);

    // A,B와 C,D는 순서 상관없음, 단 A->B, C->D 순서는 보장
}
```

## Mock 타입 선택

### StrictMock

```cpp
// 예상치 못한 호출 시 즉시 테스트 실패
StrictMock<MockDatabase> strict_mock;

TEST(StrictTest, UnexpectedCall_Fails) {
    StrictMock<MockDatabase> mock;

    // Query만 기대
    EXPECT_CALL(mock, Query(_)).WillOnce(Return("result"));

    // Connect() 호출 시 테스트 실패!
    mock.Connect("url");  // FAILS
    mock.Query("select");
}
```

### NiceMock

```cpp
// 예상치 못한 호출 허용 (기본값 반환)
NiceMock<MockDatabase> nice_mock;

TEST(NiceTest, UnexpectedCall_Allowed) {
    NiceMock<MockDatabase> mock;

    EXPECT_CALL(mock, Query(_)).WillOnce(Return("result"));

    // Connect() 호출해도 OK (기본값 반환)
    mock.Connect("url");  // OK, returns false (기본값)
    auto result = mock.Query("select");

    EXPECT_EQ(result, "result");
}
```

### 언제 무엇을 사용?

| Mock 타입 | 사용 시점 |
|----------|----------|
| StrictMock | 모든 호출을 명시적으로 검증해야 할 때 |
| NiceMock | 일부 호출만 관심있고 나머지는 무시할 때 |
| 기본 (NaggyMock) | 예상치 못한 호출 시 경고만 출력 |

## 일반적인 Mock 패턴

### ResultSet Mock 패턴 (do-while 루프)

```cpp
// 프로덕션 코드가 do-while 패턴을 사용하는 경우:
// do {
//     value = result->getItemString(0);
// } while (result->next());

std::shared_ptr<MockResultSet> SetupMockResultSet(
    const std::vector<std::string>& values)
{
    auto mock = std::make_shared<MockResultSet>();
    InSequence seq;

    if (values.empty()) {
        EXPECT_CALL(*mock, getItemString(0, _)).WillOnce(Return(""));
        EXPECT_CALL(*mock, next()).WillOnce(Return(false));
        return mock;
    }

    // 첫 번째 행
    EXPECT_CALL(*mock, getItemString(0, _)).WillOnce(Return(values[0]));

    // 나머지 행들
    for (size_t i = 1; i < values.size(); ++i) {
        EXPECT_CALL(*mock, next()).WillOnce(Return(true));
        EXPECT_CALL(*mock, getItemString(0, _)).WillOnce(Return(values[i]));
    }

    // 마지막 next() 호출
    EXPECT_CALL(*mock, next()).WillOnce(Return(false));

    return mock;
}
```

### COUNT 쿼리 Mock 패턴

```cpp
std::shared_ptr<MockResultSet> SetupMockCountResult(int count) {
    auto mock = std::make_shared<MockResultSet>();

    EXPECT_CALL(*mock, getItemInt(0)).WillOnce(Return(count));
    EXPECT_CALL(*mock, next()).WillOnce(Return(false));

    return mock;
}

TEST_F(RepositoryTest, Count_ReturnsRowCount) {
    auto mock_result = SetupMockCountResult(42);

    EXPECT_CALL(*mock_db_, Select(HasSubstr("COUNT(*)"), _))
        .WillOnce(Return(mock_result));

    int count = repo_->Count();

    EXPECT_EQ(count, 42);
}
```

### 실패 시나리오 Mock

```cpp
TEST_F(ServiceTest, Process_DatabaseFails_ReturnsError) {
    // SELECT 실패 시뮬레이션
    EXPECT_CALL(*mock_db_, Select(_, _))
        .WillOnce(DoAll(
            SetArgReferee<1>(DatabaseError(500, "Connection lost")),
            Return(nullptr)
        ));

    auto result = service_->Process(input_);

    EXPECT_FALSE(result.success);
    EXPECT_EQ(result.error_code, ErrorCode::DatabaseError);
}
```

## 주의사항

### 1. Mock 메서드 반환 타입

```cpp
// shared_ptr 반환 시 주의
MOCK_METHOD(std::shared_ptr<IResultSet>, Select, (...), (...));

// Return()에 shared_ptr 전달
EXPECT_CALL(*mock, Select(_, _))
    .WillOnce(Return(std::make_shared<MockResultSet>()));
```

### 2. const 참조 파라미터

```cpp
// const std::string& 파라미터
MOCK_METHOD(bool, Query, (const std::string& sql), (override));

// Matcher 사용 가능
EXPECT_CALL(*mock, Query(HasSubstr("SELECT")));
```

### 3. 기본 반환값

```cpp
// 기본값이 없는 타입은 명시적으로 설정 필요
ON_CALL(*mock, GetCustomObject())
    .WillByDefault(Return(CustomObject{}));
```

### 4. Mock 수명

```cpp
// Mock은 EXPECT_CALL이 검증되기 전에 파괴되면 안 됨
class ServiceTest : public ::testing::Test {
protected:
    // Mock이 테스트 종료까지 유지됨
    std::shared_ptr<MockDatabase> mock_db_;
    std::unique_ptr<Service> service_;
};
```
