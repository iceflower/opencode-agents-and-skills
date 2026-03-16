---
name: ci-cd
description: CI/CD pipeline patterns with GitHub Actions. Use when writing or
  reviewing workflow files, deployment pipelines, or branch protection rules.
---

# CI/CD Pipeline Rules

## 1. GitHub Actions Workflow Structure

### Directory Layout

```text
.github/
├── workflows/
│   ├── ci.yml              # Build + test on PR
│   ├── cd-dev.yml          # Deploy to dev
│   ├── cd-staging.yml      # Deploy to staging
│   └── cd-prod.yml         # Deploy to production
└── actions/
    └── gradle-setup/       # Reusable composite action
        └── action.yml
```

### Naming Convention

| Workflow         | Trigger              | Purpose              |
| ---------------- | -------------------- | -------------------- |
| `ci.yml`         | PR, push to main     | Build, test, lint    |
| `cd-dev.yml`     | Push to develop      | Auto-deploy to dev   |
| `cd-staging.yml` | Manual or tag        | Deploy to staging    |
| `cd-prod.yml`    | Manual with approval | Deploy to production |

---

## 2. CI Workflow Template

### Kotlin/Spring Boot

```yaml
name: CI

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main, develop]

permissions:
  contents: read
  checks: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'

      - uses: gradle/actions/setup-gradle@v4

      - name: Build and test
        run: ./gradlew build

      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: '**/build/test-results/**/*.xml'
```

---

## 3. CD Workflow Template

### Build and Push Container Image

```yaml
name: CD - Dev

on:
  push:
    branches: [develop]

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'

      - uses: gradle/actions/setup-gradle@v4

      - name: Build JAR
        run: ./gradlew bootJar -x test

      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ vars.REGISTRY }}/${{ vars.IMAGE_NAME }}:${{ github.sha }}

      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/${{ vars.APP_NAME }} \
            app=${{ vars.REGISTRY }}/${{ vars.IMAGE_NAME }}:${{ github.sha }} \
            -n ${{ vars.NAMESPACE }}
```

---

## 4. Environment and Secret Management

### GitHub Environments

| Environment | Protection Rules              | Secrets Scope     |
| ----------- | ----------------------------- | ----------------- |
| dev         | None                          | Dev credentials   |
| staging     | Required reviewers (optional) | Staging creds     |
| production  | Required reviewers + wait     | Prod credentials  |

### Secret Naming Convention

```text
DB_URL              # Database connection URL
DB_USERNAME         # Database username
DB_PASSWORD         # Database password
REGISTRY_USERNAME   # Container registry username
REGISTRY_PASSWORD   # Container registry password
KUBECONFIG          # Kubernetes config (base64 encoded)
```

### Secret Management Rules

- Use GitHub Environments to scope secrets per deployment target
- Never echo or print secrets in workflow steps
- Use `${{ secrets.NAME }}` — never hardcode values
- Rotate secrets periodically and after team member changes
- Use OIDC (`id-token: write`) over long-lived credentials when possible

---

## 5. Caching Strategy

### Gradle Cache

```yaml
- uses: gradle/actions/setup-gradle@v4
  # Gradle action handles caching automatically
```

### Docker Layer Cache

```yaml
- uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Caching Rules

- Always cache dependency downloads (Gradle, npm, pip)
- Use GitHub Actions cache or setup actions that handle caching
- Set appropriate cache keys to avoid stale caches
- Monitor cache hit rates — low hit rate means wasted storage

---

## 6. Deployment Safety

### Production Deployment Checklist

- [ ] All CI checks passed on the commit being deployed
- [ ] Staging deployment verified (manual or automated smoke test)
- [ ] Required reviewers approved the deployment
- [ ] Database migrations tested against production-like data
- [ ] Rollback plan documented and tested

### Rollback Strategy

```yaml
# Quick rollback via kubectl
- name: Rollback on failure
  if: failure()
  run: |
    kubectl rollout undo deployment/${{ vars.APP_NAME }} \
      -n ${{ vars.NAMESPACE }}
```

### Progressive Deployment

| Strategy       | Risk  | Speed  | Use Case                  |
| -------------- | ----- | ------ | ------------------------- |
| Rolling update | Low   | Medium | Default for most services |
| Blue/Green     | Low   | Fast   | Zero-downtime required    |
| Canary         | Lower | Slow   | High-traffic services     |

---

## 7. Workflow Best Practices

### Do

- Pin action versions with full SHA or major version (`@v4`)
- Use `permissions` to limit GITHUB_TOKEN scope
- Separate CI (test) and CD (deploy) workflows
- Use `environment` for deployment protection rules
- Run tests in parallel when possible (`strategy.matrix`)
- Fail fast — stop remaining jobs on first failure

### Do Not

- Use `pull_request_target` without careful security review
- Store secrets in workflow files or repository code
- Skip tests in CD pipeline ("it passed in CI")
- Deploy directly from feature branches to production
- Use `latest` tags for production container images
- Run `sudo` or install packages without pinned versions

---

## 8. Branch Protection Rules

### Recommended Settings for Main Branch

| Rule                            | Setting |
| ------------------------------- | ------- |
| Require PR before merging       | Yes     |
| Require status checks to pass   | Yes     |
| Require branches up to date     | Yes     |
| Required approvals              | 1+      |
| Dismiss stale reviews           | Yes     |
| Restrict force push             | Yes     |
| Restrict deletions              | Yes     |
