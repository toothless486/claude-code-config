# GoogleTest 테스트 패턴

## 테스트 케이스 설계 패턴

### 1. 정상/에러/경계값 패턴

모든 메서드에 대해 최소 3가지 카테고리의 테스트 작성:

```cpp
// 1. 정상 케이스 (Happy Path)
TEST_F(UserServiceTest, CreateUser_ValidInput_CreatesUser) {
    auto result = service_->CreateUser(valid_input_);
    EXPECT_TRUE(result.success);
    EXPECT_GT(result.user_id, 0);
}

// 2. 에러 케이스 (Error Path)
TEST_F(UserServiceTest, CreateUser_DuplicateEmail_ReturnsError) {
    service_->CreateUser(valid_input_);  // 첫 번째 생성
    auto result = service_->CreateUser(valid_input_);  // 중복
    EXPECT_FALSE(result.success);
    EXPECT_EQ(result.error_code, ErrorCode::DuplicateEmail);
}

// 3. 경계값 (Edge Cases)
TEST_F(UserServiceTest, CreateUser_EmptyName_ReturnsValidationError) {
    valid_input_.name = "";
    auto result = service_->CreateUser(valid_input_);
    EXPECT_FALSE(result.success);
}

TEST_F(UserServiceTest, CreateUser_MaxLengthName_Succeeds) {
    valid_input_.name = std::string(255, 'a');  // 최대 길이
    auto result = service_->CreateUser(valid_input_);
    EXPECT_TRUE(result.success);
}
```

### 2. 입력 조합 패턴 (Input Combination)

여러 입력 조합을 체계적으로 테스트:

```cpp
// 파라미터화로 입력 조합 테스트
struct ValidationTestCase {
    std::string name;
    std::string email;
    bool expected_valid;
    std::string description;
};

class UserValidationTest : public ::testing::TestWithParam<ValidationTestCase> {};

TEST_P(UserValidationTest, Validate_ReturnsExpected) {
    auto param = GetParam();
    UserInput input{param.name, param.email};

    bool result = UserValidator::Validate(input);

    EXPECT_EQ(result, param.expected_valid)
        << "Failed for: " << param.description;
}

INSTANTIATE_TEST_SUITE_P(
    InputCombinations,
    UserValidationTest,
    ::testing::Values(
        ValidationTestCase{"john", "john@example.com", true, "valid_all"},
        ValidationTestCase{"", "john@example.com", false, "empty_name"},
        ValidationTestCase{"john", "", false, "empty_email"},
        ValidationTestCase{"", "", false, "all_empty"},
        ValidationTestCase{"john", "invalid", false, "invalid_email"}
    )
);
```

### 3. 상태 전이 패턴 (State Transition)

객체의 상태 변화를 테스트:

```cpp
class ConnectionTest : public ::testing::Test {
protected:
    Connection conn_;
};

// 초기 상태
TEST_F(ConnectionTest, Initial_State_IsDisconnected) {
    EXPECT_EQ(conn_.GetState(), Connection::State::Disconnected);
}

// 상태 전이: Disconnected -> Connecting -> Connected
TEST_F(ConnectionTest, Connect_FromDisconnected_TransitionsToConnected) {
    EXPECT_EQ(conn_.GetState(), Connection::State::Disconnected);

    conn_.BeginConnect();
    EXPECT_EQ(conn_.GetState(), Connection::State::Connecting);

    conn_.OnConnected();
    EXPECT_EQ(conn_.GetState(), Connection::State::Connected);
}

// 잘못된 상태 전이
TEST_F(ConnectionTest, Disconnect_WhenAlreadyDisconnected_ThrowsException) {
    EXPECT_EQ(conn_.GetState(), Connection::State::Disconnected);
    EXPECT_THROW(conn_.Disconnect(), std::logic_error);
}
```

### 4. 의존성 주입 패턴 (Dependency Injection)

Mock을 활용한 의존성 격리:

```cpp
class OrderServiceTest : public ::testing::Test {
protected:
    void SetUp() override {
        mock_inventory_ = std::make_shared<MockInventoryService>();
        mock_payment_ = std::make_shared<MockPaymentService>();
        mock_notification_ = std::make_shared<MockNotificationService>();

        service_ = std::make_unique<OrderService>(
            mock_inventory_, mock_payment_, mock_notification_);
    }

    std::shared_ptr<MockInventoryService> mock_inventory_;
    std::shared_ptr<MockPaymentService> mock_payment_;
    std::shared_ptr<MockNotificationService> mock_notification_;
    std::unique_ptr<OrderService> service_;
};

TEST_F(OrderServiceTest, PlaceOrder_Success_NotifiesUser) {
    // Arrange
    Order order{.item_id = 1, .quantity = 2};

    EXPECT_CALL(*mock_inventory_, CheckStock(1, 2))
        .WillOnce(Return(true));
    EXPECT_CALL(*mock_payment_, Process(_))
        .WillOnce(Return(PaymentResult::Success));
    EXPECT_CALL(*mock_notification_, SendOrderConfirmation(_))
        .Times(1);

    // Act
    auto result = service_->PlaceOrder(order);

    // Assert
    EXPECT_TRUE(result.success);
}
```

## 데이터베이스 테스트 패턴

### SELECT 결과 Mock

```cpp
TEST_F(UserRepositoryTest, FindById_UserExists_ReturnsUser) {
    // Arrange
    auto mock_result = std::make_shared<MockResultSet>();

    EXPECT_CALL(*mock_db_, Select(HasSubstr("SELECT"), _))
        .WillOnce(DoAll(
            SetArgReferee<1>(DatabaseError()),  // 에러 없음
            Return(mock_result)
        ));

    // do-while 패턴: 첫 행 읽기 -> next() 호출
    EXPECT_CALL(*mock_result, getItemInt(0)).WillOnce(Return(1));
    EXPECT_CALL(*mock_result, getItemString(1, _)).WillOnce(Return("john"));
    EXPECT_CALL(*mock_result, getItemString(2, _)).WillOnce(Return("john@example.com"));
    EXPECT_CALL(*mock_result, next()).WillOnce(Return(false));  // 더 이상 행 없음

    // Act
    auto user = repo_->FindById(1);

    // Assert
    ASSERT_NE(user, nullptr);
    EXPECT_EQ(user->id, 1);
    EXPECT_EQ(user->name, "john");
}

TEST_F(UserRepositoryTest, FindById_UserNotExists_ReturnsNullptr) {
    // Arrange
    EXPECT_CALL(*mock_db_, Select(_, _))
        .WillOnce(Return(nullptr));  // 결과 없음

    // Act
    auto user = repo_->FindById(999);

    // Assert
    EXPECT_EQ(user, nullptr);
}
```

### INSERT/UPDATE/DELETE 테스트

```cpp
TEST_F(UserRepositoryTest, Save_NewUser_ExecutesInsert) {
    // Arrange
    User user{0, "john", "john@example.com"};

    EXPECT_CALL(*mock_db_, Insert(
        AllOf(
            HasSubstr("INSERT INTO users"),
            HasSubstr("john"),
            HasSubstr("john@example.com")
        ), _))
        .WillOnce(Return(true));

    // Act
    bool result = repo_->Save(user);

    // Assert
    EXPECT_TRUE(result);
}

TEST_F(UserRepositoryTest, Delete_ExistingUser_ExecutesDelete) {
    // Arrange
    EXPECT_CALL(*mock_db_, Remove(
        AllOf(
            HasSubstr("DELETE FROM users"),
            HasSubstr("WHERE id = 1")
        ), _))
        .WillOnce(Return(true));

    // Act
    bool result = repo_->Delete(1);

    // Assert
    EXPECT_TRUE(result);
}
```

## 비동기 테스트 패턴

### 타임아웃 기반 대기

```cpp
TEST_F(AsyncServiceTest, Start_CompletesWithinTimeout) {
    // Arrange
    service_->StartAsync();

    // Act & Assert
    auto start_time = std::chrono::steady_clock::now();

    while (!service_->IsReady()) {
        auto elapsed = std::chrono::steady_clock::now() - start_time;
        ASSERT_LT(elapsed, std::chrono::seconds(5))
            << "Service failed to start within timeout";
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    EXPECT_TRUE(service_->IsReady());
}
```

### Future/Promise 패턴

```cpp
TEST_F(AsyncProcessorTest, ProcessAsync_ReturnsResult) {
    // Arrange
    auto future = processor_->ProcessAsync(input_);

    // Act
    auto status = future.wait_for(std::chrono::seconds(5));

    // Assert
    ASSERT_EQ(status, std::future_status::ready)
        << "Operation timed out";
    EXPECT_EQ(future.get(), expected_result_);
}
```

### Callback Mock 패턴

```cpp
class MockCallback {
public:
    MOCK_METHOD(void, OnComplete, (const Result& result));
    MOCK_METHOD(void, OnError, (const Error& error));
};

TEST_F(AsyncServiceTest, Process_Success_CallsOnComplete) {
    // Arrange
    MockCallback callback;
    EXPECT_CALL(callback, OnComplete(_)).Times(1);
    EXPECT_CALL(callback, OnError(_)).Times(0);

    // Act
    service_->Process(input_,
        [&](const Result& r) { callback.OnComplete(r); },
        [&](const Error& e) { callback.OnError(e); }
    );

    // 비동기 완료 대기
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
}
```

## 예외 테스트 패턴

```cpp
// 예외 타입 검증
TEST_F(ParserTest, Parse_InvalidSyntax_ThrowsParseError) {
    EXPECT_THROW(parser_->Parse("invalid{{{"), ParseError);
}

// 예외 메시지 검증
TEST_F(ParserTest, Parse_InvalidSyntax_ThrowsWithDetailedMessage) {
    try {
        parser_->Parse("invalid{{{");
        FAIL() << "Expected ParseError to be thrown";
    } catch (const ParseError& e) {
        EXPECT_THAT(e.what(), HasSubstr("syntax error"));
        EXPECT_THAT(e.what(), HasSubstr("line 1"));
    }
}

// 예외 없음 검증
TEST_F(ParserTest, Parse_ValidSyntax_DoesNotThrow) {
    EXPECT_NO_THROW(parser_->Parse("valid syntax"));
}
```

## 리소스 관리 패턴

### 임시 파일 패턴

```cpp
class FileProcessorTest : public ::testing::Test {
protected:
    void SetUp() override {
        temp_file_ = CreateTempFile("test_content");
    }

    void TearDown() override {
        if (!temp_file_.empty()) {
            std::remove(temp_file_.c_str());
        }
    }

    std::string CreateTempFile(const std::string& content) {
        std::string path = "/tmp/test_" + std::to_string(getpid()) + ".txt";
        std::ofstream file(path);
        file << content;
        return path;
    }

    std::string temp_file_;
};

TEST_F(FileProcessorTest, Process_ValidFile_ReturnsContent) {
    auto result = processor_->Process(temp_file_);
    EXPECT_EQ(result, "test_content");
}
```

### RAII Guard 패턴

```cpp
class ScopedEnvironmentVariable {
public:
    ScopedEnvironmentVariable(const std::string& name, const std::string& value)
        : name_(name), original_value_(GetEnvSafe(name)) {
        setenv(name.c_str(), value.c_str(), 1);
    }

    ~ScopedEnvironmentVariable() {
        if (original_value_.has_value()) {
            setenv(name_.c_str(), original_value_->c_str(), 1);
        } else {
            unsetenv(name_.c_str());
        }
    }

private:
    std::string name_;
    std::optional<std::string> original_value_;
};

TEST(ConfigTest, LoadConfig_UsesEnvironmentVariable) {
    ScopedEnvironmentVariable env("APP_CONFIG_PATH", "/custom/path");

    Config config;
    config.Load();

    EXPECT_EQ(config.GetPath(), "/custom/path");
}
```

## 복잡한 검증 패턴

### 커스텀 Matcher

```cpp
// 특정 필드 검증 Matcher
MATCHER_P2(HasIdAndName, expected_id, expected_name, "") {
    return arg.id == expected_id && arg.name == expected_name;
}

TEST_F(UserServiceTest, GetUser_ReturnsMatchingUser) {
    auto user = service_->GetUser(1);
    EXPECT_THAT(user, HasIdAndName(1, "john"));
}

// 범위 검증 Matcher
MATCHER_P2(IsBetween, low, high, "") {
    return arg >= low && arg <= high;
}

TEST(RangeTest, GenerateRandom_WithinBounds) {
    int result = GenerateRandom(1, 100);
    EXPECT_THAT(result, IsBetween(1, 100));
}
```

### 검증 헬퍼 함수

```cpp
class OrderTest : public ::testing::Test {
protected:
    void ExpectOrderEquals(const Order& actual, const Order& expected) {
        EXPECT_EQ(actual.id, expected.id) << "Order ID mismatch";
        EXPECT_EQ(actual.status, expected.status) << "Status mismatch";
        EXPECT_NEAR(actual.total, expected.total, 0.01) << "Total mismatch";
        EXPECT_THAT(actual.items, UnorderedElementsAreArray(expected.items))
            << "Items mismatch";
    }
};

TEST_F(OrderTest, CreateOrder_ReturnsCorrectOrder) {
    Order expected{.id = 1, .status = Status::Pending, .total = 99.99};

    auto actual = service_->CreateOrder(input_);

    ExpectOrderEquals(actual, expected);
}
```
