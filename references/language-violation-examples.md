# Language Violation Examples

> This file contains the full list of language violation examples referenced from §1 of AGENTS.md.
> When the user points out a new violation, add that example to the appropriate section below.

## Reasoning Violation Examples

- "Let me check the module structure..." (X) → "모듈 구조를 확인해보겠습니다..." (O)
- "This means the variable is not wired correctly" (X) → "이는 변수가 올바르게 연결되지 않았다는 의미입니다" (O)
- "First, I'll read the file, then..." (X) → "먼저 파일을 읽고, 그 다음..." (O)
- "The error occurs because..." (X) → "이 오류는 ~때문에 발생합니다" (O)

## Communication Violation Examples (NEVER do these)

- Japanese mixed: "한국어로 작성하겠습니다 メイン" (X) → "한국어로 작성하겠습니다" (O)
- English mixed: "좋sounds good" (X) → "좋습니다" (O)
- Chinese characters mixed: "external 的" (X) → "external" (O), "just 那样" (X) → "그냥 그렇게" (O)
- Chinese word mixed: "紧密한 관계" (X) → "밀접한 관계" (O)
- Mixed-language sentence: "테마를 고를까요? 私が設定してもいい?" (X) → "테마를 고를까요? 제가 설정해도 될까요?" (O)
- Mixed non-Korean terms in Korean: "가능한 대안方案" (X) → "가능한 대안 방안" (O)
- Chinese characters in Korean text: "共青단 계열" (X) → "공산주의 청년단 계열" (O)
- Translation-ese (awkward Korean from direct translation): "OWASP 표는 Prevention 컬럼이 너무 깽니다. 내용을 줄여서 정렬하겠습니다" (X) → "OWASP 표의 Prevention 컬럼 내용이 너무 깁니다. 줄여서 정렬하겠습니다" (O)
- Chinese word in Korean text: "KMA API로迁移하고" (X) → "KMA API로 이관하고" 또는 "KMA API로 마이그레이션하고" (O)
