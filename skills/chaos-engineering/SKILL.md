# Skill: Chaos Engineering

**Source**: Release It! Second Edition - Michael Nygard (Chapter 17)

Principles and practices for building resilient systems through controlled failure injection. Use when designing reliability testing, incident prevention strategies, or system resilience verification.

---

## 1. Chaos Engineering Overview

### Definition

Chaos Engineering is the discipline of experimenting on a system to build confidence in its capability to withstand turbulent conditions in production.

### Core Principle

> "Improvement through destruction"

- Intentionally inject failures to discover weaknesses
- Build confidence in system resilience
- Identify vulnerabilities before users do

---

## 2. Pioneers and Origins

### Netflix Chaos Engineering

Netflix pioneered chaos engineering practices:

- **Chaos Monkey**: Randomly terminates instances in production
- Tests automatic recovery capabilities
- If cluster can't recover, the service team must address the issue

### Evolution

| Generation | Tool | Description |
| ---------- | ---- | ----------- |
| 1st | Chaos Monkey | Random instance termination |
| 2nd | Latency Monkey | Inject delays into calls |
| 3rd | FIT (Failure Injection Testing) | Fine-grained failure injection |

---

## 3. Chaos Monkey

### What It Does

- Randomly "wakes up" and kills an instance in an auto-scaling cluster
- Cluster must automatically recover
- If recovery fails, the owning team must respond

### Purpose

- Verify automatic recovery works
- Ensure monitoring alerts are functioning
- Test runbook effectiveness
- Build muscle memory for incident response

### Implementation

```yaml
# Chaos Monkey Configuration Example
enabled: true
schedule:
  start_hour: 9
  end_hour: 15
  timezone: "America/Los_Angeles"
  weekends: false
  
target:
  app: "my-service"
  region: "us-east-1"
  
termination:
  frequency: "daily"
  max_instances: 1
```

---

## 4. Chaos Injection Types

### 4.1 Instance Termination

**Description**: The most basic and crudest injection type.

**What it tests**:

- Auto-scaling group recovery
- Load balancer health checks
- Connection draining

**Limitations**:

- Finds obvious weaknesses
- Not sufficient alone

### 4.2 Latency Injection (Latency Monkey)

**Description**: Deliberately adds latency to calls.

**What it tests**:

- Timeout configurations
- Retry logic behavior
- Graceful degradation

**Weaknesses discovered**:

| Issue | Description |
| ----- | ----------- |
| **Fallback behavior** | Services should have useful fallbacks, not just timeout or error |
| **Race conditions** | Latency can reveal race conditions that only appear when responses arrive in different orders |

### 4.3 Service Call Failure

**Description**: Fail specific service-to-service calls.

**What it tests**:

- Circuit breaker behavior
- Fallback mechanisms
- Error propagation

**Useful for**:

- Deep call tree architectures
- Microservice dependencies
- Critical path analysis

---

## 5. Failure Injection Testing (FIT)

### What is FIT?

Netflix's FIT (Failure Injection Testing) injects more subtle failures:

- Tag requests with a cookie at the API gateway boundary
- Cookie instructs: "Process normally, but fail when Service G calls Service H"
- At the call point, check the cookie and fail as instructed

### How It Works

```text
Request → API Gateway (tag with cookie)
        → Service A (normal processing)
        → Service B (normal processing)
        → ...
        → Service G (check cookie → fail call to H)
```

### Implementation Requirements

- Common service call framework
- Request context propagation
- Configurable failure injection rules

---

## 6. Adopting Your Own Monkey

### 17.4.1 Prerequisites

Before starting chaos experiments:

| Prerequisite | Description |
| ------------ | ----------- |
| **Monitoring** | Must have visibility into system behavior |
| **Alerting** | Must be notified when things break |
| **Runbooks** | Must know how to respond to failures |
| **Recovery mechanisms** | Must have automated or manual recovery procedures |
| **Stakeholder buy-in** | Leadership must understand and support experiments |

### 17.4.2 Experiment Design

**Key questions to answer**:

1. What hypothesis are we testing?
2. What's the expected system behavior?
3. How will we measure the impact?
4. What's the blast radius?
5. What's the rollback plan?

**Experiment template**:

```markdown
## Chaos Experiment: [Name]

### Hypothesis
[What we believe will happen]

### Expected Behavior
[What should happen when failure is injected]

### Metrics to Observe
- [ ] Error rate
- [ ] Latency
- [ ] Customer impact
- [ ] Recovery time

### Blast Radius
[Which services/users are affected]

### Rollback Plan
[How to stop the experiment]
```

### 17.4.3 Chaos Injection

**System knowledge required**:

- Know which instances to kill
- Know where to add latency
- Know which service calls to fail

### 17.4.4 Target Selection

**Selection criteria**:

| Factor | Consideration |
| ------ | ------------- |
| **Criticality** | Start with non-critical services |
| **Dependency depth** | Test deep call chains |
| **Recent changes** | Focus on newly deployed components |
| **Historical issues** | Target previously problematic areas |

### 17.4.5 Automation and Repetition

**Principles**:

- Automate experiment execution
- Run experiments regularly (daily/weekly)
- Integrate with CI/CD pipelines
- Track results over time

---

## 7. Disaster Simulation

### 17.5 Game Days

**Purpose**: Practice incident response in controlled environment.

**Structure**:

```text
1. Announce game day schedule
2. Select failure scenario
3. Inject failure
4. Observe response
5. Debrief and document
```

### Scenarios to Simulate

| Scenario | Purpose |
| -------- | ------- |
| **AZ failure** | Test multi-AZ resilience |
| **Region failure** | Test DR procedures |
| **Database failover** | Test RTO/RPO |
| **Third-party outage** | Test fallback behavior |
| **Traffic spike** | Test auto-scaling |

---

## 8. Chaos Engineering Checklist

### Before Starting

- [ ] Monitoring and alerting in place?
- [ ] Runbooks documented?
- [ ] Stakeholders informed?
- [ ] Rollback plan ready?

### For Each Experiment

- [ ] Hypothesis defined?
- [ ] Metrics identified?
- [ ] Blast radius understood?
- [ ] Timing appropriate (not during peak traffic)?
- [ ] Team available to respond?

### After Each Experiment

- [ ] Results documented?
- [ ] Weaknesses identified?
- [ ] Improvements planned?
- [ ] Follow-up experiments scheduled?

---

## 9. Common Anti-Patterns

| Anti-Pattern | Problem |
| ------------ | ------- |
| **Chaos without monitoring** | Can't observe impact |
| **Too much chaos at once** | Hard to isolate cause |
| **Chaos only in test environments** | Doesn't prove production resilience |
| **No rollback plan** | Risk of prolonged outage |
| **Surprise chaos** | Undermines team trust |

---

## References

- Release It! Second Edition, Michael Nygard, Chapter 17
- Netflix Tech Blog: FIT (Failure Injection Testing)
- Principles of Chaos Engineering (principlesofchaos.org)

---

## Related Skills

- **system-stability-patterns** — Stability patterns verified by chaos experiments (circuit breaker, timeouts)
- **monitoring** — Monitoring patterns essential for observing chaos experiments
- **k8s-workflow** — Kubernetes health checks and pod management
