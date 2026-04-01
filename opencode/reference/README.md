# OpenCode Reference 디렉토리

OpenCode 시작 시 **자동 로드되지 않는** 참조 파일들이 위치한 디렉토리입니다.

## 목적

참조 파일은 상세 데이터, 패턴, 예시를 제공합니다. 토큰 절약을 위해 컨텍스트에 자동 주입되지 않으며, 필요시 에이전트가 조회합니다.

## 파일 목록

| 파일 | 설명 | 크기 |
|------|------|------|
| `language-violation-patterns.md` | 패턴 기반 CJK 수정 규칙, 단일 문자 맵 | ~17KB |
| `language-violation-exceptions.md` | CJK 감지 예외 케이스 | ~5KB |

## 사용 방식

규칙 파일(`rules/`)에서 참조 포인터로 연결됩니다:

```markdown
> 패턴 기반 수정 규칙은 `reference/language-violation-patterns.md`를 참조하세요
```

## Rules vs Reference 차이

| 구분 | 위치 | 자동 로드 | 용도 |
|------|------|----------|------|
| 핵심 규칙 | `rules/` | Yes | 항상 적용 필요 |
| 상세 데이터 | `reference/` | No | 필요시 조회 |

## 새 참조 파일 추가 방법

1. 이 디렉토리에 `.md` 파일 생성
2. 관련 `rules/` 파일에 참조 포인터 추가
3. 설정 변경 불필요 - 상대 경로로 접근 가능

## 관련 디렉토리

- `../rules/` - 자동 로드 규칙 파일
- `../AGENTS.md` - 전역 에이전트 지침