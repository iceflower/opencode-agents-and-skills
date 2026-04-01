# Language Violation Exceptions

> Special cases that cannot be handled by pattern rules.
> Reference: `language-violation-patterns.md` for pattern-based rules.

## Allowed CJK in Korean Sentences

### Technical Terms (No Korean Equivalent)

These terms are allowed in Korean sentences as-is:

| Term | Reason |
|------|--------|
| JavaScript | Programming language name |
| Python | Programming language name |
| Rust | Programming language name |
| React | Open source project name |
| Vue | Open source project name |
| Kubernetes | Open source project name |
| API | Widely recognized acronym |
| HTTP | Widely recognized acronym |
| SQL | Widely recognized acronym |
| JSON | Widely recognized acronym |
| YAML | Widely recognized acronym |
| npm | Package manager name |
| pip | Package manager name |
| cargo | Package manager name |
| git | Version control system |
| GitHub | Platform name |
| npm | Package registry |

### Proper Nouns

| Category | Examples | Handling |
|----------|----------|----------|
| Company names | OpenAI, Anthropic, Google | Keep as-is |
| Product names | Claude, GPT, Gemini | Keep as-is |
| Person names | (Chinese/Japanese names) | Transliterate or keep as-is |
| Place names | Tokyo, Beijing | Use Korean equivalent if exists |
| Brand names | Zhipu AI (智谱) | Use brand's preferred form |

### Code and Technical Content

| Type | Example | Handling |
|------|---------|----------|
| Variable names | `userName`, `totalCount` | Keep as-is |
| Function names | `getUser()`, `calculate()` | Keep as-is |
| Class names | `UserService`, `OrderController` | Keep as-is |
| File paths | `/src/main.ts` | Keep as-is |
| URLs | `https://example.com` | Keep as-is |
| Commands | `npm install`, `git commit` | Keep as-is |
| Error messages | (original error text) | Keep as-is or translate |

---

## Complex Violations (Manual Review Required)

### Mixed Language Patterns

These violations require context to determine the correct Korean expression:

| Violation | Possible Corrections | Context Needed |
|-----------|---------------------|----------------|
| "智谱" | "지푸" (Zhipu AI brand) or transliteration | Is it referring to the company? |
| "开放平台" | "개방형 플랫폼" or "오픈 플랫폼" | Formal vs casual context |
| "最佳" | "최적의" or "가장 좋은" | Technical vs general context |
| "实践" | "실천" or "실무" or "실습" | Action vs work context |

### Context-Dependent Translations

| Chinese | Korean Options | When to Use |
|---------|---------------|-------------|
| 问题 | 문제 (problem) / 질문 (question) | Context determines meaning |
| 行 | 줄 (line) / 행 (row) / 실행 (execute) | Technical context |
| 列 | 열 (column) / 리스트 (list) | Data structure context |
| 表 | 표 (table) / 나타내다 (express) | Noun vs verb |
| 实 | 실제 (actual) / 실 (reality) | Context determines meaning |

---

## Special Cases

### Compound Words

| Violation | Correction | Notes |
|-----------|------------|-------|
| "共青단 계열" | "공산주의 청년단 계열" | Historical/political term |
| "紧密한 관계" | "밀접한 관계" | Chinese character in Korean word |
| "external 的" | "external" (as-is) | Chinese suffix on English word |
| "最佳的 컨벤션" | "최선의 컨벤션" or "가장 적합한 컨벤션" | Mixed Chinese-English-Korean |

### Sentence Fragments

| Violation | Correction | Notes |
|-----------|------------|-------|
| "这次真的是深刻的教训になりました" | "이번엔 정말 깊은 교훈이 되었습니다" | Chinese-Japanese mixed |
| "테마를 고를까요? 私が設定してもいい?" | "테마를 고를까요? 제가 설정해도 될까요?" | Korean-Japanese mixed |
| "OWASP 표는 Prevention 컬럼이 너무 깽니다" | "OWASP 표의 Prevention 컬럼 내용이 너무 깁니다" | Translation error (깽니다 → 깁니다) |

---

## Reasoning Violations

These are violations in AI's internal reasoning (should be in Korean):

| Violation | Correction |
|-----------|------------|
| "Let me check the module structure..." | "모듈 구조를 확인해보겠습니다..." |
| "This means the variable is not wired correctly" | "이는 변수가 올바르게 연결되지 않았다는 의미입니다" |
| "First, I'll read the file, then..." | "먼저 파일을 읽고, 그 다음..." |
| "The error occurs because..." | "이 오류는 ~때문에 발생합니다" |
| "Now let me continue" | "지금 계속하겠습니다" |
| "Thinking:" | "생각:" |

---

## Detection Exclusions

The CJK detector should skip these patterns:

### Code Blocks

```
```typescript
const message = "你好世界";  // Skip this
```
```

### Import Statements

```typescript
import { something } from "中文路径";  // Skip this
```

### Comments

```typescript
// 这是一个注释  // Skip this
/* 日本語コメント */  // Skip this
```

### Package Declarations

```json
{
  "name": "中文包名",  // Skip this
  "version": "1.0.0"
}
```

### URLs and Paths

```
https://example.com/中文路径  // Skip this
/用户/文档/文件.txt  // Skip this
```

---

## Adding New Exceptions

When adding a new exception:

1. **Check if it fits a pattern** in `language-violation-patterns.md`
2. **If not**, add to appropriate section in this file
3. **Include context** explaining why it's an exception
4. **Provide Korean equivalent** if applicable

### Template

```markdown
| Violation | Correction | Notes |
|-----------|------------|-------|
| "{violation}" | "{correction}" | {why it's an exception} |
```