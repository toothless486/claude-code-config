# GoogleTest 테스트 코드 템플릿

## 기본 테스트 파일 구조

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>
#include <memory>
#include <string>
#include <vector>

// 테스트 대상 헤더
#include "TargetClass.h"

// Mock 헤더 (필요시)
#include "MockDependency.h"

using ::testing::Return;
using ::testing::_;
using ::testing::AllOf;
using ::testing::HasSubstr;
using ::testing::DoAll;
using ::testing::SetArgReferee;

// ============================================================
// Test Constants
// ============================================================
namespace test_constants {
    constexpr int kValidId = 12345;
    constexpr int kInvalidId = -1;
    constexpr char kValidName[] = "test_name";
    constexpr char kEmptyString[] = "";
}

// ============================================================
// Test Fixture
// ============================================================
class TargetClassTest : public ::testing::Test {
protected:
    void SetUp() override {
        mock_dep_ = std::make_shared<MockDependency>();
        target_ = std::make_unique<TargetClass>(mock_dep_);
    }

    void TearDown() override {
        target_.reset();
        mock_dep_.reset();
    }

    // Helper functions
    TargetClass::Input CreateValidInput() {
        return TargetClass::Input{
            .id = test_constants::kValidId,
            .name = test_constants::kValidName
        };
    }

    std::shared_ptr<MockDependency> mock_dep_;
    std::unique_ptr<TargetClass> target_;
};

// ============================================================
// Test Cases
// ============================================================

// --- 정상 케이스 ---

TEST_F(TargetClassTest, Method_ValidInput_ReturnsSuccess) {
    // Arrange
    auto input = CreateValidInput();
    EXPECT_CALL(*mock_dep_, Process(_))
        .WillOnce(Return(true));

    // Act
    auto result = target_->Method(input);

    // Assert
    EXPECT_TRUE(result.success);
    EXPECT_EQ(result.value, expected_value);
}

// --- 에러 케이스 ---

TEST_F(TargetClassTest, Method_InvalidInput_ReturnsFalse) {
    // Arrange
    TargetClass::Input invalid_input{.id = test_constants::kInvalidId};

    // Act
    auto result = target_->Method(invalid_input);

    // Assert
    EXPECT_FALSE(result.success);
}

// --- 경계값 케이스 ---

TEST_F(TargetClassTest, Method_EmptyInput_ReturnsEmpty) {
    // Arrange
    TargetClass::Input empty_input{};

    // Act
    auto result = target_->Method(empty_input);

    // Assert
    EXPECT_TRUE(result.value.empty());
}

// --- 예외 케이스 ---

TEST_F(TargetClassTest, Method_NullPointer_ThrowsException) {
    // Arrange & Act & Assert
    EXPECT_THROW(target_->Method(nullptr), std::invalid_argument);
}
```

## TEST (Fixture 불필요) 템플릿

```cpp
// ============================================================
// 순수 함수 테스트 (Fixture 불필요)
// ============================================================

TEST(UtilsTest, ParseInt_ValidString_ReturnsInteger) {
    // Arrange
    std::string input = "12345";

    // Act
    int result = Utils::ParseInt(input);

    // Assert
    EXPECT_EQ(result, 12345);
}

TEST(UtilsTest, ParseInt_InvalidString_ThrowsException) {
    // Arrange
    std::string input = "not_a_number";

    // Act & Assert
    EXPECT_THROW(Utils::ParseInt(input), std::invalid_argument);
}

TEST(UtilsTest, ParseInt_EmptyString_ReturnsZero) {
    EXPECT_EQ(Utils::ParseInt(""), 0);
}
```

## TEST_F (Fixture 사용) 템플릿

```cpp
// ============================================================
// Fixture 기반 테스트
// ============================================================

class StackTest : public ::testing::Test {
protected:
    void SetUp() override {
        // 공통 초기화: 스택에 기본 요소 추가
        stack_.Push(1);
        stack_.Push(2);
        stack_.Push(3);
    }

    void TearDown() override {
        // 필요시 정리 작업
    }

    Stack<int> stack_;
};

TEST_F(StackTest, Push_SingleElement_IncreasesSize) {
    // Arrange
    int initial_size = stack_.Size();

    // Act
    stack_.Push(4);

    // Assert
    EXPECT_EQ(stack_.Size(), initial_size + 1);
}

TEST_F(StackTest, Pop_NonEmpty_ReturnsTopElement) {
    // Act
    int result = stack_.Pop();

    // Assert
    EXPECT_EQ(result, 3);  // 마지막에 Push된 요소
}

TEST_F(StackTest, Pop_Empty_ThrowsException) {
    // Arrange: 모든 요소 제거
    while (!stack_.IsEmpty()) {
        stack_.Pop();
    }

    // Act & Assert
    EXPECT_THROW(stack_.Pop(), std::out_of_range);
}
```

## TEST_P (파라미터화 테스트) 템플릿

```cpp
// ============================================================
// 파라미터화 테스트
// ============================================================

struct ParseTestParam {
    std::string input;
    int expected;
    std::string description;
};

class ParseIntParamTest : public ::testing::TestWithParam<ParseTestParam> {};

TEST_P(ParseIntParamTest, ParseInt_ReturnsExpected) {
    // Arrange
    auto param = GetParam();

    // Act
    int result = Utils::ParseInt(param.input);

    // Assert
    EXPECT_EQ(result, param.expected) << "Input: " << param.input;
}

INSTANTIATE_TEST_SUITE_P(
    ValidInputs,
    ParseIntParamTest,
    ::testing::Values(
        ParseTestParam{"0", 0, "zero"},
        ParseTestParam{"123", 123, "positive"},
        ParseTestParam{"-456", -456, "negative"},
        ParseTestParam{"2147483647", 2147483647, "max_int"}
    ),
    [](const ::testing::TestParamInfo<ParseTestParam>& info) {
        return info.param.description;
    }
);

// ============================================================
// 간단한 파라미터화 (std::pair 사용)
// ============================================================

class IsPrimeTest : public ::testing::TestWithParam<std::pair<int, bool>> {};

TEST_P(IsPrimeTest, IsPrime_ReturnsExpected) {
    auto [input, expected] = GetParam();
    EXPECT_EQ(IsPrime(input), expected);
}

INSTANTIATE_TEST_SUITE_P(
    Primes,
    IsPrimeTest,
    ::testing::Values(
        std::make_pair(2, true),
        std::make_pair(3, true),
        std::make_pair(4, false),
        std::make_pair(17, true),
        std::make_pair(100, false)
    )
);
```

## Mock 클래스 템플릿

```cpp
// ============================================================
// Mock 클래스 정의
// ============================================================

// 인터페이스 (테스트 대상이 의존하는)
class IDatabase {
public:
    virtual ~IDatabase() = default;
    virtual bool Connect(const std::string& url) = 0;
    virtual std::shared_ptr<IResultSet> Select(const std::string& query, DatabaseError& error) = 0;
    virtual bool Insert(const std::string& query, DatabaseError& error) = 0;
    virtual bool Update(const std::string& query, DatabaseError& error) = 0;
    virtual bool Remove(const std::string& query, DatabaseError& error) = 0;
};

// Mock 클래스
class MockDatabase : public IDatabase {
public:
    MOCK_METHOD(bool, Connect, (const std::string& url), (override));
    MOCK_METHOD(std::shared_ptr<IResultSet>, Select,
                (const std::string& query, DatabaseError& error), (override));
    MOCK_METHOD(bool, Insert, (const std::string& query, DatabaseError& error), (override));
    MOCK_METHOD(bool, Update, (const std::string& query, DatabaseError& error), (override));
    MOCK_METHOD(bool, Remove, (const std::string& query, DatabaseError& error), (override));
};

// ResultSet Mock
class MockResultSet : public IResultSet {
public:
    MOCK_METHOD(bool, next, (), (override));
    MOCK_METHOD(int, getItemInt, (int index), (override));
    MOCK_METHOD(std::string, getItemString, (int index, int maxLen), (override));
};
```

## 헬퍼 함수 패턴

```cpp
// ============================================================
// TestHelpers.h
// ============================================================

namespace TestHelpers {

// 테스트 객체 생성 헬퍼
inline User CreateTestUser(
    int id = 1,
    const std::string& name = "test_user",
    const std::string& email = "test@example.com")
{
    return User{id, name, email};
}

// JSON 객체 생성 헬퍼
inline Json::Value CreateTestMetadata(
    const std::map<std::string, std::string>& fields)
{
    Json::Value metadata(Json::objectValue);
    for (const auto& [key, value] : fields) {
        metadata[key] = value;
    }
    return metadata;
}

// Mock ResultSet 설정 헬퍼
inline std::shared_ptr<MockResultSet> SetupMockResultSet(
    const std::vector<std::vector<std::string>>& rows)
{
    auto mock = std::make_shared<MockResultSet>();

    if (rows.empty()) {
        EXPECT_CALL(*mock, next()).WillOnce(Return(false));
        return mock;
    }

    // 첫 번째 행은 이미 커서가 위치함
    for (size_t col = 0; col < rows[0].size(); ++col) {
        EXPECT_CALL(*mock, getItemString(col, _))
            .WillOnce(Return(rows[0][col]));
    }

    // 나머지 행들
    for (size_t row = 1; row < rows.size(); ++row) {
        EXPECT_CALL(*mock, next()).WillOnce(Return(true));
        for (size_t col = 0; col < rows[row].size(); ++col) {
            EXPECT_CALL(*mock, getItemString(col, _))
                .WillOnce(Return(rows[row][col]));
        }
    }

    EXPECT_CALL(*mock, next()).WillOnce(Return(false));
    return mock;
}

} // namespace TestHelpers
```

## main.cpp 템플릿

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>

int main(int argc, char** argv) {
    ::testing::InitGoogleTest(&argc, argv);
    ::testing::InitGoogleMock(&argc, argv);
    return RUN_ALL_TESTS();
}
```

## Makefile 템플릿

```makefile
# GoogleTest 테스트 빌드 설정
GTEST_DIR = /path/to/googletest
GMOCK_DIR = /path/to/googlemock

CXX = g++
CXXFLAGS = -std=c++14 -Wall -Wextra -g
INCLUDES = -I$(GTEST_DIR)/include -I$(GMOCK_DIR)/include -I../src

LDFLAGS = -L$(GTEST_DIR)/lib -L$(GMOCK_DIR)/lib
LDLIBS = -lgtest -lgmock -lpthread

TEST_SRCS = $(wildcard *_test.cpp)
TEST_OBJS = $(TEST_SRCS:.cpp=.o)
TEST_APP = TestApp

all: $(TEST_APP)

$(TEST_APP): main.o $(TEST_OBJS)
	$(CXX) $(CXXFLAGS) -o $@ $^ $(LDFLAGS) $(LDLIBS)

%.o: %.cpp
	$(CXX) $(CXXFLAGS) $(INCLUDES) -c $< -o $@

clean:
	rm -f *.o $(TEST_APP)

test: $(TEST_APP)
	./$(TEST_APP)

.PHONY: all clean test
```
