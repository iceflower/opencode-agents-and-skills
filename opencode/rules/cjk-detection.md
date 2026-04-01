# CJK Character Detection Rules

## Korean Writing Rules (AGENTS.md §1)

### Core Principle
Korean sentences must be written entirely in Korean. Chinese and Japanese characters are prohibited.

### Detection Patterns

| Pattern | Type | Example | Correction |
|---------|------|---------|------------|
| Korean + Chinese | Mixed | "밀접한 关系" | "밀접한 관계" |
| Korean + Japanese | Mixed | "한국어로 작성하겠습니다 メイン" | "한국어로 작성하겠습니다" |
| Chinese only | Violation | "这个问题很严重" | "이 문제는 매우 심각합니다" |
| Japanese only | Violation | "検出して記録する" | "검출해서 기록하다" |

### Unicode Ranges

| Character Type | Unicode Range | Regex |
|----------------|---------------|-------|
| Chinese (Hanzi) | U+4E00–U+9FFF | `[\u4e00-\u9fff]` |
| Japanese Hiragana | U+3040–U+309F | `[\u3040-\u309f]` |
| Japanese Katakana | U+30A0–U+30FF | `[\u30a0-\u30ff]` |
| Korean (Hangul) | U+AC00–U+D7AF | Allowed |

### Allowed Exceptions

1. **Technical terms without Korean equivalent**:
   - Programming language names: JavaScript, Python, Rust
   - Framework names: React, Vue, Kubernetes
   - Acronyms: API, HTTP, SQL, JSON, YAML

2. **User-initiated foreign text**: When user explicitly uses foreign text

3. **Code blocks**: Content inside ```...``` is exempt

### Immediate Correction Rule

When CJK mixing is detected:
1. Correct immediately without asking
2. Do not wait for user to point out the violation
3. Apply correction silently

### Common Violations Reference

> For pattern-based correction rules, single character maps, and exceptions, see:
> - `reference/language-violation-patterns.md`
> - `reference/language-violation-exceptions.md`