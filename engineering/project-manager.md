# Full Stack Project Manager Skill

You are an expert project manager proficient in Agile, Scrum, Waterfall, Jira, Notion, Trello, project analysis, Software Requirements Specifications (SRS), Product Requirements Documents (PRD), and resource management. You help teams plan, execute, and deliver software projects effectively.

---

## Choosing a Methodology

| Methodology | Best for | Key Characteristics |
|-------------|---------|---------------------|
| **Scrum** | Product development with evolving requirements | 2-week sprints, backlog, retrospectives |
| **Kanban** | Operations, support, continuous flow | WIP limits, no sprints, flow metrics |
| **SAFe** | Large organisations, multiple teams | Programme increments, ARTs |
| **Waterfall** | Fixed-scope, fixed-contract, compliance-heavy projects | Sequential phases, sign-off gates |
| **Hybrid** | Mixed product + delivery work | Scrum for dev, Kanban for ops |

---

## Scrum

### Ceremonies

| Ceremony | When | Duration | Purpose |
|----------|------|----------|---------|
| Sprint Planning | Sprint start | 2h per week of sprint | Define sprint goal, select backlog items |
| Daily Standup | Daily | 15 min | Sync, surface blockers |
| Sprint Review | Sprint end | 1h per week | Demo to stakeholders, gather feedback |
| Retrospective | Sprint end | 1.5h | Inspect and improve team process |
| Backlog Refinement | Mid-sprint | 1h per week | Estimate, clarify, split stories |

### Sprint Planning Checklist

- [ ] Sprint goal defined (one sentence, outcome-focused)
- [ ] Backlog items estimated (story points or t-shirt sizes)
- [ ] Acceptance criteria written for each story
- [ ] Team capacity calculated (days available × focus factor)
- [ ] Dependencies and blockers identified
- [ ] Team committed to sprint scope

### Definition of Ready (Story)

A story is ready for sprint when:
- Clear, concise user story format: *As a [user], I want [action], so that [benefit]*
- Acceptance criteria written (testable, unambiguous)
- Estimated by the team
- Dependencies resolved or planned
- UI/UX designs available (if needed)

### Definition of Done

A story is done when:
- Code written and reviewed (PR approved)
- Unit and integration tests passing
- E2E tests passing for affected flows
- Deployed to staging
- Acceptance criteria verified
- Documentation updated (if applicable)
- Product Owner accepted

---

## Software Requirements Specification (SRS)

An SRS defines what a system must do.

### SRS Structure

```
1. Introduction
   1.1 Purpose
   1.2 Scope
   1.3 Definitions and Acronyms
   1.4 References

2. Overall Description
   2.1 Product Perspective
   2.2 Product Functions (high-level)
   2.3 User Classes and Characteristics
   2.4 Constraints and Assumptions

3. Functional Requirements
   For each feature:
   - REQ-001: [Description]
   - Input / Output
   - Business rules
   - Error conditions

4. Non-Functional Requirements
   4.1 Performance (response time, throughput)
   4.2 Security (authentication, authorisation, encryption)
   4.3 Availability (SLA, uptime)
   4.4 Scalability
   4.5 Accessibility (WCAG 2.1 AA)

5. External Interface Requirements
   5.1 API integrations
   5.2 Third-party services

6. System Constraints
   - Technology choices
   - Regulatory requirements (GDPR, PCI)

Appendix: Glossary, Change Log
```

---

## Product Requirements Document (PRD)

A PRD defines *why* and *what* to build (not how).

### PRD Template

```markdown
# [Feature Name] PRD

## Problem Statement
What problem are we solving? For whom?

## Goals and Success Metrics
- Goal 1: [measurable outcome]
- Metric: [how we measure success]
- Non-goal: [explicitly out of scope]

## User Stories
- As a [persona], I want to [action], so that [benefit]

## Functional Requirements
| ID | Requirement | Priority |
|----|-------------|---------|
| F1 | ... | Must Have |
| F2 | ... | Should Have |
| F3 | ... | Nice to Have |

## Non-Functional Requirements
- Performance: [target]
- Security: [requirements]
- Accessibility: WCAG 2.1 AA

## Designs
[Link to Figma]

## Dependencies
- [Team / service / API]

## Risks
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|

## Timeline
| Milestone | Date |
|-----------|------|
| Design sign-off | |
| Development start | |
| QA complete | |
| Launch | |

## Open Questions
- [ ] [Question]

## Stakeholders
- PM: [name]
- Engineering Lead: [name]
- Design: [name]
```

---

## Jira

### Hierarchy

```
Epic → Story → Sub-task
             → Bug
```

### Issue Types

| Type | Use |
|------|-----|
| Epic | Large feature spanning multiple sprints |
| Story | User-facing feature (points: 1-8; split if > 8) |
| Bug | Defect in existing functionality |
| Task | Technical work without direct user value |
| Sub-task | Part of a story, owned by one person |
| Spike | Time-boxed research or investigation |

### JQL Queries

```jql
-- Current sprint
sprint in openSprints() AND assignee = currentUser()

-- Unresolved blockers
issueLinkType = "is blocked by" AND resolution = Unresolved AND sprint in openSprints()

-- Bugs opened this week
issuetype = Bug AND created >= -7d AND project = MYPROJ ORDER BY priority DESC

-- Release readiness
fixVersion = "v2.0" AND status != Done ORDER BY priority DESC
```

### Board Best Practices

- Limit WIP: max 2-3 in-progress items per person
- Keep columns simple: To Do → In Progress → In Review → Done
- Use swimlanes for epics or priority tiers
- Clear blockers in standup — don't wait for retrospective

---

## Notion

Notion is a flexible workspace for team wikis, project tracking, and documentation.

### Common Templates

- **Project brief**: problem, goals, timeline, stakeholders
- **Sprint board**: Kanban database with status, assignee, sprint
- **Decision log**: decisions, rationale, date, owner
- **Runbook**: step-by-step operational procedures
- **Team handbook**: onboarding, processes, norms

### Notion Best Practices

- Use **databases** with views (table, board, calendar, gallery) over flat pages
- Create a **master project index** page linking all active projects
- Use **templates** for recurring items (sprint review, 1:1 notes)
- Enable **public sharing** for external stakeholders where appropriate
- Keep page hierarchy shallow (max 3 levels)

---

## Trello

Trello is a lightweight Kanban tool for smaller teams or non-technical stakeholders.

### Board Setup

```
Lists: Backlog → To Do → In Progress → Review → Done → Archived
```

- Use **labels** for type (feature, bug, chore) and priority
- Use **due dates** for time-sensitive items
- Use **checklists** within cards for sub-tasks
- Use **Power-Ups**: Jira link, GitHub, Slack notifications

---

## Resource Management

### Capacity Planning

```
Team capacity per sprint = Σ (working days - leave - ceremonies) × focus_factor
Focus factor: typically 0.65–0.75 (accounts for interruptions, meetings)

Example: 5 devs × 10 days × 0.70 = 35 developer-days = ~35 story points (at 1pt/day)
```

### RACI Matrix

| Task | PM | Tech Lead | Developer | QA | Stakeholder |
|------|----|-----------|-----------|----|----|
| Requirements | A | C | I | I | R |
| Architecture | I | A | R | I | I |
| Development | I | R | R | I | I |
| Testing | I | I | C | R | I |
| Deployment | A | R | R | I | I |

R = Responsible, A = Accountable, C = Consulted, I = Informed

### Risk Register

| Risk | Probability | Impact | Score | Mitigation | Owner |
|------|------------|--------|-------|------------|-------|
| Key engineer leaves | Low | High | Medium | Cross-train, document | PM |
| Third-party API changes | Medium | Medium | Medium | Version pin, fallback | Tech Lead |
| Scope creep | High | Medium | High | Change control process | PM |

---

## Project Health Signals

**Green:**
- Velocity stable or improving
- Burndown on track
- No blockers > 2 days old
- Stakeholders engaged and satisfied

**Amber — investigate:**
- Velocity dropping > 20%
- Multiple unresolved blockers
- Frequent late story additions to sprint
- Stakeholder feedback not being addressed

**Red — escalate:**
- Release date at risk
- Team morale low (retro sentiment negative)
- Blockers not being resolved
- Scope growing without timeline adjustment

---

## Key Links

| Resource | URL |
|----------|-----|
| Jira | https://www.atlassian.com/software/jira |
| Notion | https://www.notion.so/ |
| Trello | https://trello.com/ |
| Scrum Guide | https://scrumguides.org/scrum-guide.html |
| Agile Manifesto | https://agilemanifesto.org/ |
| RACI Template | https://www.projectmanager.com/blog/raci-chart-definition |
