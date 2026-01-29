# Commit & Push Command

변경 사항을 커밋하고 원격 저장소에 푸시한다.

## 사용법
```
/commit [commit message]
```
- `[commit message]`: 선택적 커밋 메시지. 생략 시 변경 사항을 분석하여 자동 생성

## 실행 절차

### 1. 현재 상태 확인
```bash
git status
git diff --staged
git diff
git log --oneline -5
```

### 2. 사용자 확인 (필수)
**반드시 AskUserQuestion 도구를 사용하여 다음 사항을 확인받는다:**

#### 2-1. 브랜치 확인
- 현재 브랜치명을 표시하고 이 브랜치에 커밋할 것인지 확인
- 예: "현재 브랜치는 `feature/LGUVSAAS-1234` 입니다. 이 브랜치에 커밋하시겠습니까?"

#### 2-2. 커밋 대상 파일 확인
- 변경된 파일 목록을 보여주고 커밋할 파일을 선택받음
- staged 파일과 unstaged 파일을 구분하여 표시
- 사용자가 커밋할 파일을 명시적으로 승인해야 진행

**사용자 승인 없이는 절대 커밋을 진행하지 않는다.**

### 3. 커밋 메시지 작성 규칙
- 변경 사항의 **목적(why)**을 중심으로 작성
- 형식: `<type>: <description>`
  - `feat`: 새로운 기능
  - `fix`: 버그 수정
  - `refactor`: 코드 리팩토링
  - `docs`: 문서 수정
  - `test`: 테스트 추가/수정
  - `chore`: 빌드, 설정 등 기타 변경

### 4. 커밋 수행
- 스테이징되지 않은 변경 사항이 있으면 확인 후 추가
- 민감 정보(`.env`, 인증키 등) 포함 여부 확인
- 커밋 메시지 끝에 서명 추가:
```
🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>
```

### 5. 푸시 수행
- 원격 브랜치 추적 여부 확인
- 필요시 `-u` 플래그로 업스트림 설정
```bash
git push -u origin <current-branch>
```

### 6. PR용 메시지 생성
- 푸시 완료 후 PR 메시지 생성 여부를 사용자에게 확인
- 아래 템플릿 사용:

```markdown
## 해결하려는 문제가 무엇인가요?
<변경 사항의 배경과 목적을 설명>

## 어떻게 해결했나요?
<구현 방법과 주요 변경 내용을 설명>

```

## 안전 규칙
- `--force` 푸시 금지 (명시적 요청 없이)
- `main`/`master`/`develop` 브랜치 직접 푸시 금지
- pre-commit hook 실패 시 `--no-verify` 사용 금지

## 입력
$ARGUMENTS

