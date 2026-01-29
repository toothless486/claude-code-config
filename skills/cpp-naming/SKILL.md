---
name: cpp-naming
description: C/C++ 코드의 함수, 변수, 클래스 이름을 효과적으로 작성하기 위한 네이밍 가이드. 코드 리뷰, 리팩토링, 새 코드 작성 시 의도를 드러내는 이름(Intention-Revealing Names)을 제안할 때 사용. "이름 추천", "네이밍", "변수명", "함수명", "클래스명" 관련 요청 시 활용.
---

# C/C++ Naming Skill

코드의 이름은 **왜 존재하는지**, **무엇을 하는지**, **어떻게 사용되는지**를 답해야 한다.

## 핵심 원칙: Intention-Revealing Names

### "How"가 아닌 "What/Why"를 드러내라

```cpp
// ❌ 구현 방법(How)을 드러냄
int linearSearchPosition(const std::vector<int>& arr, int target);
void quickSortArray(std::vector<int>& arr);
char* mallocString(size_t size);

// ✅ 의도(What/Why)를 드러냄
int indexOf(const std::vector<int>& arr, int target);
void sort(std::vector<int>& arr);
char* allocateString(size_t size);
```

### 테스트: 다른 구현으로 바꿔도 이름이 유효한가?

`linearSearchPosition` → 이진 탐색으로 바꾸면 이름이 거짓이 됨
`indexOf` → 어떤 구현이든 의미가 유효함

## C/C++ 네이밍 컨벤션

| 대상 | 스타일 | 예시 |
|------|--------|------|
| 지역 변수 | snake_case | `frame_count`, `user_input` |
| 전역 변수 | g_ 접두사 + snake_case | `g_config`, `g_instance_count` |
| 함수 | camelCase | `calculateChecksum()`, `getFrameBuffer()` |
| 클래스/구조체 | PascalCase | `VideoEncoder`, `FrameBuffer` |
| 상수/매크로 | SCREAMING_SNAKE_CASE | `MAX_BUFFER_SIZE`, `DEFAULT_TIMEOUT_MS` |
| 멤버 변수 | _ 접미사 | `frame_count_`, `buffer_size_` |
| 포인터 | p 접두사 (선택) | `pBuffer`, `pContext` |
| 열거형 값 | k 접두사 또는 대문자 | `kStateIdle`, `STATE_RUNNING` |

## 품사 규칙

### 변수: 명사/명사구
```cpp
// ✅ Good
int connection_count;
std::string user_name;
VideoFrame* current_frame;
bool is_connected;      // bool은 is_, has_, can_, should_ 접두사

// ❌ Bad
int count;              // 무엇의 count?
std::string str;        // 의미 없음
bool flag;              // 무슨 flag?
```

### 함수: 동사/동사구
```cpp
// ✅ Good
void sendPacket(const Packet& pkt);
int calculateChecksum(const uint8_t* data, size_t len);
bool validateInput(const std::string& input);
VideoFrame* getNextFrame();

// ❌ Bad
void packet(const Packet& pkt);     // 동사 없음
int checksum();                      // 동작 불명확
bool input();                        // 무엇을 하는지 불명확
```

### 클래스: 명사 (역할을 나타내는)
```cpp
// ✅ Good
class VideoEncoder;
class ConnectionPool;
class FrameBuffer;
class EventHandler;

// ❌ Bad
class VideoEncoderManager;    // Manager, Handler, Processor, Data, Info는 피하라
class DoEncoding;             // 동사 사용 금지
class Utility;                // 너무 모호함
```

## 피해야 할 패턴

### 1. 약어와 축약 (팀 내 합의된 것만 허용)
```cpp
// ❌ Bad
int nCnt;
char* pszBuf;
void calcChkSum();

// ✅ Good
int connection_count;
char* string_buffer;
void calculateChecksum();

// ✅ 허용되는 약어 (도메인에서 표준)
int fps;              // frames per second
size_t num_bytes;     // number of bytes
void* ctx;            // context (매우 짧은 스코프에서만)
```

### 2. 헝가리안 표기법 (타입 인코딩)
```cpp
// ❌ Bad - 타입을 이름에 포함
int iCount;
std::string strName;
float fTemperature;
int* pnArray;

// ✅ Good - 의미만 포함
int count;
std::string name;
float temperature;
int* values;
```

### 3. 단일 문자 (루프 인덱스 외 금지)
```cpp
// ✅ 허용: 짧은 루프
for (int i = 0; i < n; i++) { ... }
for (size_t j = 0; j < cols; j++) { ... }

// ❌ 금지: 그 외 모든 경우
int n = getUserCount();    // → int user_count
void* p = allocate(sz);    // → void* buffer
```

### 4. 모호한 단어
```cpp
// ❌ 피해야 할 단어: data, info, temp, tmp, result, value, item, object, thing
int data;           // → int sensor_reading;
void* temp;         // → void* swap_buffer;
std::string info;   // → std::string error_message;

// ❌ 클래스에서 피할 접미사: Manager, Processor, Handler, Helper, Utility
class DataManager;  // → class DataStore; 또는 class DataCache;
```

### 5. 부정형보다 긍정형
```cpp
// ❌ Bad - 이중 부정 유발
bool isNotConnected;
bool disableLogging;
if (!isNotConnected) { ... }

// ✅ Good
bool isConnected;
bool enableLogging;
if (isConnected) { ... }
```

## 스코프와 길이

| 스코프 | 권장 길이 | 예시 |
|--------|----------|------|
| 루프 인덱스 | 1자 | `i`, `j`, `k` |
| 람다 파라미터 | 1-2자 | `[](auto& e) { ... }` |
| 지역 변수 (좁은 스코프) | 짧게 | `frame`, `buf` |
| 지역 변수 (넓은 스코프) | 설명적 | `decoded_video_frame` |
| 멤버 변수 | 설명적 | `connection_timeout_` |
| 전역/상수 | 매우 설명적 | `MAX_CONCURRENT_CONNECTIONS` |

## 일관성 체크리스트

- [ ] 같은 개념에 같은 단어 사용 (`get`/`fetch`/`retrieve` 중 하나만)
- [ ] 반대 개념에 대칭적 이름 (`open`/`close`, `begin`/`end`, `create`/`destroy`)
- [ ] 단위가 있으면 이름에 포함 (`timeout_ms`, `distance_km`, `angle_deg`)
- [ ] 컬렉션은 복수형 (`frames`, `connections`, `users`)
- [ ] bool은 질문 형태 (`isValid`, `hasData`, `canRetry`, `shouldStop`)

## 리팩토링 예시

```cpp
// Before
class VM {
    int n;
    void* p;
    bool f;
    void proc();
    int calc(int x);
};

// After
class VideoManager {
    int active_stream_count_;
    void* frame_buffer_;
    bool is_recording_;
    void processNextFrame();
    int calculateBitrate(int qualityLevel);
};
```

## 네이밍 제안 시 출력 형식

```
제안: [새 이름]
이유: [왜 이 이름이 의도를 더 잘 드러내는지]
대안: [다른 후보들]
주의: [팀 컨벤션 확인 필요 시]
```
