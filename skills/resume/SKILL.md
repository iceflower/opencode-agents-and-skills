---
name: resume
description: >-
  Developer resume writing guide including structure, experience description
  with STAR method, technical skills organization, and project description.
  Use when writing or reviewing a developer resume.
---
# Developer Resume Rules

## 1. Resume Structure

### Section Order

| Section            | Required | Purpose                                    |
| ------------------ | -------- | ------------------------------------------ |
| Contact Info       | Yes      | Name, email, phone, GitHub, LinkedIn       |
| Summary            | Yes      | 2-3 sentence career overview               |
| Skills             | Yes      | Technical skill inventory                  |
| Work Experience    | Yes      | Reverse chronological career history       |
| Projects           | Optional | Notable side/open-source projects          |
| Education          | Yes      | Degree, institution, graduation year       |
| Certifications     | Optional | Relevant professional certifications       |

### Formatting Principles

- Keep to 1-2 pages maximum — prioritize relevance over completeness
- Use consistent formatting (fonts, spacing, bullet style) throughout
- Left-align all content — avoid center-aligned body text
- Use clear section headings with visual separation
- Avoid tables, columns, or complex layouts that break ATS parsing
- Save as PDF to preserve formatting across devices
- File name format: `{Name}_Resume_{YYYY}.pdf`

### ATS (Applicant Tracking System) Compatibility

- Use standard section headings (Experience, Skills, Education)
- Avoid headers/footers for critical information
- Do not embed text in images or graphics
- Use standard fonts (sans-serif preferred)
- Include keywords from the job description naturally in context

---

## 2. Experience Description

### STAR Method for Each Bullet Point

| Element   | Description                    | Example                                                  |
| --------- | ------------------------------ | -------------------------------------------------------- |
| Situation | Context or problem background  | Legacy monolith causing 30-min deploy cycles             |
| Task      | Your specific responsibility   | Led migration to microservice architecture               |
| Action    | What you actually did          | Decomposed into 12 domain services with event sourcing   |
| Result    | Measurable outcome             | Reduced deploy time to 5 min, 99.9% uptime achieved      |

### Writing Rules

- Start every bullet with a strong action verb (Designed, Implemented, Optimized, Led, Migrated)
- Quantify results wherever possible (%, time saved, throughput, users, cost reduction)
- Focus on impact and outcomes, not just tasks performed
- Limit to 3-5 bullet points per role — prioritize most impactful contributions
- Use past tense for previous roles, present tense for current role

### Action Verb Categories

| Category      | Verbs                                                           |
| ------------- | --------------------------------------------------------------- |
| Building      | Designed, Developed, Implemented, Built, Created, Engineered    |
| Leading       | Led, Managed, Coordinated, Mentored, Directed, Supervised       |
| Improving     | Optimized, Refactored, Improved, Reduced, Streamlined, Enhanced |
| Analyzing     | Analyzed, Evaluated, Investigated, Diagnosed, Identified        |
| Delivering    | Deployed, Delivered, Launched, Released, Shipped, Migrated      |
| Collaborating | Collaborated, Partnered, Facilitated, Contributed, Supported    |

### Quantification Examples

| Weak (task-focused)                    | Strong (impact-focused)                                          |
| -------------------------------------- | ---------------------------------------------------------------- |
| Wrote unit tests                       | Achieved 85% code coverage, reducing production bugs by 40%      |
| Managed database                       | Optimized slow queries, reducing P99 latency from 3s to 200ms    |
| Worked on API development              | Designed RESTful APIs serving 50K+ daily active users            |
| Participated in code reviews           | Reviewed 200+ PRs, establishing team coding standards            |
| Helped with deployment                 | Automated CI/CD pipeline, reducing deploy time from 2h to 15min  |

---

## 3. Technical Skills Organization

### Skill Categorization

| Category              | Examples                                            |
| --------------------- | --------------------------------------------------- |
| Languages             | Kotlin, Java, TypeScript, Python                    |
| Frameworks            | Spring Boot, React, FastAPI                         |
| Databases             | PostgreSQL, MySQL, Redis, MongoDB                   |
| Infrastructure        | Docker, Kubernetes, Terraform, AWS/GCP              |
| CI/CD                 | GitHub Actions, Jenkins, ArgoCD                     |
| Messaging             | Kafka, RabbitMQ, NATS                               |
| Monitoring            | Prometheus, Grafana, OpenTelemetry                  |
| Tools                 | Git, IntelliJ IDEA, Jira, Confluence                |

### Skill Presentation Rules

- Group skills by category, not by proficiency level
- List only skills you can discuss confidently in an interview
- Prioritize skills mentioned in the target job description
- Include version numbers only for major frameworks (e.g., Spring Boot 3.x)
- Omit outdated or irrelevant skills (e.g., jQuery for a backend role)
- Do not rate skills with stars, bars, or percentage — they are subjective and unhelpful

### Skill Relevance Filtering

- **Always include**: Skills directly mentioned in the job posting
- **Include if relevant**: Skills from the same ecosystem or complementary tools
- **Omit**: Skills unrelated to the target role or tech stack

---

## 4. Project Description

### Project Entry Structure

```text
Project Name | Role | Duration
Brief description (1-2 sentences)

- Technical decisions and rationale
- Your specific contributions
- Measurable outcomes or impact
- Tech stack used
```

### Project Description Rules

- Lead with the business problem or goal, not the technology
- Clearly distinguish your individual contribution from team effort
- Highlight technical decision-making (why you chose X over Y)
- Include scale indicators (users, data volume, request throughput)
- Link to live demo, GitHub repo, or documentation when available

### What to Highlight in Projects

| Aspect              | Why It Matters                                   |
| ------------------- | ------------------------------------------------ |
| Problem definition  | Shows you understand business context            |
| Architecture choice | Demonstrates technical judgment                  |
| Trade-off analysis  | Shows you evaluate alternatives, not just build  |
| Scale and impact    | Provides concrete evidence of capability         |
| Lessons learned     | Shows growth mindset and self-reflection         |

### Project Selection Priority

- Projects that align with the target role's tech stack
- Projects where you made key architectural decisions
- Projects with measurable business impact
- Open-source contributions with community engagement
- Side projects that demonstrate initiative and learning

---

## 5. Anti-Patterns

- Listing every technology ever touched — curate for relevance
- Copy-pasting job descriptions as experience bullets
- Using vague phrases ("various tasks", "responsible for", "helped with")
- Including personal information (age, photo, marital status, religion)
- Lying or exaggerating — interviewers will probe claims in detail
- Using the same resume for every application — tailor per role
- Focusing on duties instead of achievements
- Writing paragraphs instead of concise bullet points
- Including references or "References available upon request"
- Using first person pronouns (I, me, my) in bullet points
