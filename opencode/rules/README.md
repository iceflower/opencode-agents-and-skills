# OpenCode Rules 디렉토리

OpenCode 시작 시 **자동 로드**되는 규칙 파일들이 위치한 디렉토리입니다.

## 목적

OpenCode는 이 디렉토리의 모든 파일을 에이전트 컨텍스트에 자동 주입합니다. 중요한 규칙이 항상 적용되도록 보장합니다.

## 파일 목록

| 파일 | 설명 |
|------|------|
| `cjk-detection.md` | 한국어 문장에서 한자/일본어 문자 감지 규칙 |
| `information-accuracy.md` | 사실 검증 및 소스 확인 규칙 |

## 자동 로드 방식

`opencode.jsonc` 설정에서 로드됩니다:

```jsonc
"instructions": [
  "~/.config/opencode/rules/cjk-detection.md",
  "~/.config/opencode/rules/information-accuracy.md"
]
```

## 새 규칙 추가 방법

1. 이 디렉토리에 `.md` 파일 생성
2. `opencode.jsonc`의 `instructions` 배열에 경로 추가
3. OpenCode 재시작

## 관련 디렉토리

- `../reference/` - 참조 파일 (자동 로드되지 않음, 필요시 조회)
- `../AGENTS.md` - 전역 에이전트 지침