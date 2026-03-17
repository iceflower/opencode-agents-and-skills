# Agents.md

AI 코딩 에이전트를 위한 글로벌 규칙 파일(`AGENTS.md`)을 관리하는 저장소입니다.

## AGENTS.md란?

`AGENTS.md`는 AI 코딩 에이전트(OpenCode, Codex 등)가 읽어들이는 글로벌 행동 규칙 파일입니다. 에이전트의 언어, 보안, 문서 작성, 작업 완료 기준 등을 정의합니다.

## 포함된 규칙

- **언어 규칙**: 한국어 커뮤니케이션, 존댓말 사용, 추측 금지
- **민감 정보 처리**: API 키, 비밀번호 등 보안 가이드
- **마크다운 규칙**: 문서 포맷팅 표준
- **파일 수정 규칙**: 편집 후 저장 일관성, Lint 검증
- **대화 및 컨텍스트 규칙**: 맥락 파악, 다단계 작업 인식
- **태스크 완료 선언**: 작업 완료 체크리스트
- **사용자 확인 프로토콜**: 위험 작업 시 확인 절차
- **코드 품질 필수 원칙**: 6대 코드 품질 원칙

## 사용 방법

```bash
# AGENTS.md를 글로벌 설정 경로에 복사
cp AGENTS.md ~/.config/opencode/
```

## Skills (스킬)

스킬은 별도 저장소로 분리되었습니다: [iceflower/agent-skills](https://github.com/iceflower/agent-skills)

## 라이선스

MIT License
