# Technical Documentation Guide

Comprehensive guide for writing, researching, deploying, and measuring technical documentation quality.

## 1. Documentation Writing Principles

### Pre-Writing Checklist

- [ ] Is there additional context or setup info the reader needs?
- [ ] Are there skipped or incompletely explained steps?
- [ ] Do the steps flow logically when read in sequence?

### Procedure Step Principles

| Principle | Description |
| --------- | ----------- |
| **One action per step** | Include only a single action in each step |
| **State system prerequisites** | Specify login requirements, execution environment, etc. |
| **Minimize context switching** | Reduce transitions between document and UI/CLI |
| **Provide completion verification** | State how to verify successful completion |

### Document Structuring (F-Shaped Pattern)

Readers scan content in an 'F' pattern:

1. Scan horizontally across the top two lines
2. Scan vertically downward looking for headings
3. Do not read every word on the page

Design documents accordingly:

- Write for scanning — help readers find information quickly
- Present most important information first
- Use consistent, predictable structure

### Sample Code Guidelines

Always explain when writing sample code:

- Required library installations
- Required environment variables
- Language/version constraints

Code explanation must cover:

| Element | Description |
| ------- | ----------- |
| **What it does** | Describe the code's functionality |
| **Why it does it** | Provide context and background |
| **Notable aspects** | Unusual naming conventions, unique methods, etc. |

### Editing Checklist

#### Technical Accuracy

- [ ] Does the code/command actually work?
- [ ] Are version, environment, platform differences specified?
- [ ] Are warning notices (outage risk, data loss, etc.) included?
- [ ] Is terminology used consistently?

#### Completeness

- [ ] Does the content include all information users need to succeed?
- [ ] Are all `[TODO]` or `[TBD]` items resolved?
- [ ] Are prerequisites, required permissions, dependencies stated?
- [ ] Are next steps/additional resources provided?

#### Structure

- [ ] Does the title clearly express the purpose?
- [ ] Are section headings arranged logically and consistently?
- [ ] Is the document's purpose explained in the first paragraph?
- [ ] Does the document follow the template structure (if applicable)?

#### Clarity and Conciseness

- [ ] Do all links work?
- [ ] Have you run spell and grammar check?
- [ ] Are graphics and images clear and useful?
- [ ] Has unnecessary content been removed?

### Writing Process

```text
1. Plan: Define audience, purpose, content patterns
2. Draft: Outline → Write body
3. Edit: Technical Accuracy → Completeness → Structure → Clarity
4. Review: Peer review → Friction log testing
5. Publish: Set timeline → Release
```

---

## 2. User Research Methods

### User Story Format

```text
As a [user type], I want to [action] so that I can [goal].
```

- Keep user needs in mind during planning, writing, editing, publishing, and maintenance
- Select key areas and write multiple user stories for each

### Friction Log

A friction log systematically records the friction (confusion, frustration, obstacles) users experience.

#### How to Write

1. **Record basic information**: Scenario, environment info (OS, browser, SDK version), test date
2. **Record step-by-step experience**: For each step, record feeling, time taken, issues found, and suggestions
3. **Self-check**: After each step, evaluate intuitiveness, confidence, uncertainty, navigation, and frustration

#### Self-Check Questions

| Question | Recording Points |
| -------- | ---------------- |
| Did it seem easy? | Intuitiveness, clarity |
| Did I feel confident I was on the right path? | Direction, trust |
| Was I uncertain? | Ambiguity, uncertainty |
| Did I get lost? | Navigation, structure issues |
| Was I frustrated? | Usability issues, bugs |

### User Personas

| Element | Description |
| ------- | ----------- |
| **User type** | Developer, data scientist, DevOps engineer, etc. |
| **Technical level** | Beginner, intermediate, advanced |
| **Primary goals** | What they want to achieve |
| **Pain points** | Difficulties or problems they face |
| **Preferred formats** | Tutorials, API reference, sample code, etc. |

### User Journey Map Components

1. **Stages**: Major phases users go through
2. **Touchpoints**: Points of contact at each stage
3. **Actions**: Activities users perform
4. **Thoughts/Feelings**: User experience at each stage
5. **Opportunities**: Areas for improvement

### Survey Design Principles

| Characteristic | Description |
| -------------- | ----------- |
| **One thing per question** | Avoid compound questions |
| **Closed questions** | Limit possible answers (multiple choice, checkboxes) |
| **Include optional questions** | Do not force answers to all questions |
| **Neutrality** | Avoid leading questions; use unbiased language |

Tactics to improve response rate:

- Be clear about who you are and the research purpose
- State data collection purpose explicitly
- Write easy-to-answer, concise questions
- Avoid excessive demands on respondents

---

## 3. Documentation Deployment Process

### Timeline Coordination

- Integrate documentation timeline into product release timeline
- Set timelines for all documentation releases including minor releases
- Developer documentation must be released alongside the software it describes

#### Timeline Template

| Stage | Task | Owner | Deadline |
| ----- | ---- | ----- | -------- |
| T-14 | Finalize documentation requirements | PM | - |
| T-10 | Complete draft | Tech Writer | - |
| T-7 | Complete peer review | Dev Team | - |
| T-5 | Revisions and final draft | Tech Writer | - |
| T-3 | Final approval | Tech Lead | - |
| T-0 | Documentation deploy | DevOps | - |

### Platform Selection Criteria

| Criterion | Question |
| --------- | -------- |
| **User needs** | What format do users prefer? |
| **Search capability** | Is effective search available? |
| **Version management** | Can multiple doc versions be managed? |
| **Collaboration** | Can team members easily contribute? |
| **Integration** | Does it integrate with existing dev workflows? |
| **Cost** | Can it operate within budget? |

### API Reference Automation Checklist

- [ ] Is the API spec file included in the code repository?
- [ ] Is doc generation automated in CI/CD pipeline?
- [ ] Are generated docs automatically deployed?
- [ ] Are changes version-controlled?

### Rollback Procedure

1. **Identify problem**: User report or internal discovery
2. **Assess impact**: Number of affected users and severity
3. **Decide**: Determine rollback necessity
4. **Execute**: Restore previous version
5. **Notify**: Apologize and inform users
6. **Rework**: Fix issue and redeploy

### Post-Deployment Monitoring

| Metric | Measurement Method | Target |
| ------ | ------------------ | ------ |
| **Pageviews** | Analytics tools | Understand traffic trends |
| **Bounce rate** | Analytics tools | < 50% |
| **Search queries** | Internal search logs | Understand user needs |
| **404 errors** | Log analysis | 0 occurrences |
| **Doc sentiment** | Survey tools | > 80% positive |

### Documentation Maintenance Checklist

#### Product Updates

- [ ] Add documentation for new features
- [ ] Update documentation for changed features
- [ ] Remove or mark documentation for deprecated features
- [ ] Update screenshots/diagrams

#### Regular Reviews

- [ ] Validate links
- [ ] Verify code examples work
- [ ] Update version information
- [ ] Incorporate user feedback

---

## 4. Quality Measurement and Improvement

### Sentiment Measurement

Place a simple survey tool on pages:

```text
Was this page helpful?    [Yes]    [No]
```

- Collect large numbers of responses for useful data
- For low-rated pages: analyze causes, apply improvements, measure impact
- For high-rated pages: analyze success factors, extract patterns, replicate success

### Feedback Classification

Classify collected feedback using three questions:

| Question | Description |
| -------- | ----------- |
| **1. Is this issue valid?** | Confirm the problem actually exists |
| **2. Can this issue be resolved?** | Assess technical/resource feasibility |
| **3. How important is this issue?** | Determine user impact and priority |

### Issue Priority System (P0-P3)

| Level | Name | Description | Response Target |
| ----- | ---- | ----------- | --------------- |
| **P0** | Critical | Core content wrong, security info missing, code causing fatal errors | Within 24 hours |
| **P1** | High | Frequently used feature docs incomplete, recurring user-reported issues | Current sprint |
| **P2** | Medium | Less-used feature improvement, readability enhancement | Next sprint |
| **P3** | Low | Minor typos, low-traffic page improvements | Backlog |

### Quality Metrics Dashboard

| Metric | Formula | Target |
| ------ | ------- | ------ |
| **Sentiment Score** | (Yes responses / Total responses) x 100 | > 80% |
| **Response Rate** | (Responses / Pageviews) x 100 | > 5% |
| **Feedback Resolution Rate** | (Resolved feedback / Total feedback) x 100 | > 90% |
| **P0 Resolution Time** | Average time to resolve P0 issues | < 24 hours |

### PDCA Improvement Cycle

1. **Plan**: Set goals, create improvement plan
2. **Do**: Apply improvements
3. **Check**: Review sentiment scores, feedback
4. **Act**: Further improvements, standardization

---

## 5. Deployment Strategies

### Immutable Infrastructure

| Management Style | Characteristics | Failure Response |
| ---------------- | --------------- | ---------------- |
| **Manual (mutable)** | Individually configure and manage servers | Manual recovery, root cause analysis needed |
| **Automated (immutable)** | Code-based infrastructure, servers are replaceable | Auto-replacement, fast recovery |

### Deployment Phase Structure

```text
Preparation → Application → Cleanup
```

- **Preparation**: Upload new content, prepare DB schema changes (must not affect current version)
- **Application**: Version switch, traffic switch, health check
- **Cleanup**: Clean up old resources, collect logs

### Rolling Deployment Procedure

1. Set group to stop accepting new requests
2. Wait for in-progress work to complete
3. Apply code and configuration updates
4. Verify all instances are healthy
5. Re-enable group for new requests
6. Repeat for next group

### Canary Deployment Monitoring

| Item | Check |
| ---- | ----- |
| Error logs | Any increase? |
| Latency | Any increase? |
| RAM usage | Any increase? |
| Response time | SLA compliance? |

### Health Check Requirements

- Implement non-blocking health check endpoint for all applications
- Return meaningful status for downstream dependencies
- Include version and uptime information for debugging
- Use state changes to reject new requests while completing in-progress work

---

## Further Reading

- Docs for Developers (Jared Bhatti et al.)
- Docs Like Code (Anne Gentle)
- Release It! Second Edition (Michael Nygard)
- Google Technical Writing Course
