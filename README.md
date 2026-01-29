# Claude Code Config

Claude Code CLI를 위한 커스텀 커맨드, 스킬, 설정 모음입니다.

## 설치

```bash
# 프로젝트 루트에서
git clone https://github.com/toothless486/claude-code-config.git .claude
```

## 구조

```
.claude/
├── commands/          # 슬래시 커맨드 (/commit-push, /review-pr 등)
├── skills/            # 자동 호출 스킬
│   ├── cpp-naming/    # C++ 네이밍 가이드
│   ├── review-gtest/  # GoogleTest 코드 리뷰
│   └── write-gtest/   # GoogleTest 테스트 작성
└── settings.json      # 공유 설정
```

## 커맨드

| 커맨드 | 설명 |
|--------|------|
| `/commit-push` | 커밋 및 푸시 |
| `/review-pr` | PR 리뷰 |
| `/check-duplicates` | 중복 코드 검사 |
| `/generate-tc` | 테스트 케이스 생성 |
| `/speckit.*` | 스펙 문서 관리 |

## 스킬

- **cpp-naming**: C++ 네이밍 컨벤션 제안
- **review-gtest**: GoogleTest 코드 리뷰 및 개선안
- **write-gtest**: GoogleTest 테스트 코드 작성

## 사용법

프로젝트 루트에 `.claude/` 폴더로 복사하거나 심볼릭 링크 생성:

```bash
ln -s /path/to/claude-code-config /your/project/.claude
```

## 설정

`settings.local.json`은 로컬 전용 설정으로 버전 관리에서 제외됩니다.
