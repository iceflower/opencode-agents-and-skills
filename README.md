# OpenCode Agents & Skills

OpenCode AI 에이전트를 위한 글로벌 설정 파일 모음입니다.

## 파일 구조

```
├── AGENTS.md          # 글로벌 규칙
└── skills/            # 스킬 정의 파일들
    ├── git-workflow/SKILL.md
    ├── spring-framework/SKILL.md
    └── ...
```

## AGENTS.md

글로벌 규칙 파일로, 다음을 포함합니다:

- **언어 규칙**: 한국어 커뮤니케이션, 추측 금지
- **마크다운 규칙**: 문서 포맷팅 표준
- **민감 정보 처리**: API 키, 비밀번호 등 보안 가이드
- **태스크 완료 선언**: 작업 완료 체크리스트

## Skills 목록

### Spring & Spring Boot

| Skill | 설명 |
|-------|------|
| spring-framework | IoC/DI, AOP, 트랜잭션 관리 |
| spring-boot-migration | Spring Boot 버전 마이그레이션 |
| spring-security | 보안 설정 및 인증 |
| spring-webflux | WebFlux 및 Kotlin Coroutines |
| spring-jpa | JPA 패턴 및 N+1 방지 |
| spring-config | 설정 관리 패턴 |

### Kotlin & Java

| Skill | 설명 |
|-------|------|
| kotlin-convention | Kotlin 코딩 컨벤션 |
| java-convention | Java 코딩 컨벤션 |
| java-kotlin-interop | Java-Kotlin 상호 운용 |
| kotlin-migration | Kotlin 버전 마이그레이션 |
| java-migration | Java 버전 마이그레이션 |

### Database

| Skill | 설명 |
|-------|------|
| database | 일반 데이터베이스 패턴 |
| mysql | MySQL 특화 규칙 |
| postgresql | PostgreSQL 특화 규칙 |
| exposed | JetBrains Exposed ORM |
| spring-jpa | Spring Data JPA |

### Infrastructure

| Skill | 설명 |
|-------|------|
| terraform-workflow | Terraform 핵심 워크플로우 |
| terraform-aws-provider | AWS Terraform 프로바이더 |
| terraform-azure-provider | Azure Terraform 프로바이더 |
| k8s-workflow | Kubernetes 매니페스트 |
| karpenter-workflow | Karpenter 오토스케일링 |

### 코드 품질 & 테스트

| Skill | 설명 |
|-------|------|
| code-quality | 코드 품질 원칙 |
| code-review | 코드 리뷰 체크리스트 |
| testing-unit | BDD 스타일 단위 테스트 |
| testing-integration | 통합 테스트 패턴 |
| error-handling | 에러 처리 패턴 |

### 기타

| Skill | 설명 |
|-------|------|
| git-workflow | Git 커밋 컨벤션 및 브랜치 전략 |
| ci-cd | GitHub Actions CI/CD |
| security | 보안 규칙 |
| caching | 캐싱 전략 |
| logging | 로깅 표준 |
| monitoring | 관측 가능성 패턴 |

## 사용 방법

이 파일들을 OpenCode 설정 디렉토리에 배치하세요:

```bash
# AGENTS.md 복사
cp AGENTS.md ~/.config/opencode/

# Skills 복사
cp -r skills/* ~/.config/opencode/skills/
```

## 라이선스

MIT License