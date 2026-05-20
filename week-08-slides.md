---
marp: true
theme: default
paginate: true
header: 'CSC 394 — Week 8'
footer: 'Security, Performance, and Technical Debt'
style: |
  section {
    font-family: "Aptos", "Calibri", sans-serif;
    color: #172033;
    background: #f7f8f3;
  }
  h1 {
    color: #12355b;
    font-size: 42px;
    letter-spacing: -0.03em;
  }
  h2, h3 {
    color: #0b6b6f;
  }
  strong {
    color: #b85042;
  }
  blockquote {
    border-left: 8px solid #b85042;
    background: #fff8ed;
    color: #172033;
    padding: 0.35rem 0.8rem;
    font-size: 0.86em;
  }
  table {
    font-size: 0.78em;
  }
  code {
    font-size: 0.78em;
  }
  section.lead {
    background: #12355b;
    color: #f7f8f3;
  }
  section.lead h1,
  section.lead h2 {
    color: #f7f8f3;
  }
  section.lead header,
  section.lead footer,
  section.lead::after {
    color: #cadcfc;
  }
  section.lead strong {
    color: #f4a261;
  }
  .cards {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 0.6rem;
  }
  .card {
    background: white;
    border-top: 8px solid #0b6b6f;
    border-radius: 12px;
    padding: 0.65rem;
    min-height: 6.2rem;
    box-shadow: 0 5px 18px rgba(18, 53, 91, 0.13);
  }
  .card h3 {
    margin: 0 0 0.35rem 0;
    font-size: 1.05rem;
  }
  .impact {
    background: #12355b;
    color: #f7f8f3;
    border-radius: 14px;
    padding: 0.75rem 1rem;
    font-size: 1.05rem;
    margin-top: 0.8rem;
  }
  .impact strong {
    color: #f4a261;
  }
  .two {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 0.75rem;
    align-items: start;
  }
  .story {
    background: #fff8ed;
    border-radius: 14px;
    padding: 0.75rem;
    border-left: 8px solid #b85042;
  }
  .checklist {
    columns: 2;
    font-size: 0.82em;
    line-height: 1.25;
  }
  .checklist p {
    break-inside: avoid;
    margin: 0 0 0.28rem 0;
  }
---

<!-- _class: lead -->

# Week 8: Security, Performance, and Technical Debt

## Building apps that survive real users, real attackers, and real deadlines

CSC 394 — Software Projects  
DePaul University

---

# Today’s Outcome

By the end of this lecture, each team should know what to change before preview demo week.

<div class="cards">
  <div class="card"><h3>Security</h3><p>Protect user data from common attacks and accidental exposure.</p></div>
  <div class="card"><h3>Performance</h3><p>Find bottlenecks users actually feel, then fix the highest-impact ones.</p></div>
  <div class="card"><h3>Maintainability</h3><p>Pay down debt that makes the next change risky or slow.</p></div>
</div>

---

<!-- _class: lead -->

# Web Application Security

## The goal is not perfect security.  
## The goal is avoiding obvious, costly failures.

---

# “It’s Just a Class Project”

That does **not** make it invisible.

<div class="cards">
  <div class="card"><h3>Public repos</h3><p>Employers, classmates, scanners, and bots can inspect them.</p></div>
  <div class="card"><h3>Real-ish data</h3><p>Names, emails, tasks, passwords, and team notes still matter.</p></div>
  <div class="card"><h3>Habits</h3><p>The defaults you practice now become the defaults you ship later.</p></div>
</div>

> **Story:** Exposed cloud keys in public repos have been abused quickly enough to create real bills before the owner noticed.

---

# SQL Injection: Small Input, Huge Blast Radius

> **TalkTalk, 2015:** SQL injection exposed customer data, damaged trust, triggered regulatory penalties, and showed that old vulnerabilities still create modern business damage.

<div class="impact">
<strong>Project outcome:</strong> user input is always treated as data, never executable database logic.
</div>

```javascript
const user = await prisma.user.findUnique({
  where: { email: req.body.email },
});
```

---

# Authentication Failure Has a Long Memory

> **RockYou, 2009:** 32M plain-text passwords leaked. The list still influences password cracking today.

| Bad default | Better default | Impact |
|---|---|---|
| Store passwords directly | Hash with bcrypt | Leaked DB does not expose original passwords |
| Unlimited login attempts | Rate-limit auth routes | Brute force becomes slower and noisier |
| Tokens never expire | Expiring sessions/JWTs | Stolen tokens lose value |

---

# Sensitive Data Exposure

The frontend is public. API responses are observable. Git history persists.

<div class="two">
<div>

## Never expose

- password hashes
- database URLs
- private API keys
- JWT signing secrets
- internal stack traces

</div>
<div>

## Safer pattern

```javascript
await prisma.user.findUnique({
  select: { id: true, name: true, email: true },
});
```

</div>
</div>

---

# XSS: Trusted Page, Attacker Script

> **British Airways, 2018:** Malicious JavaScript on a trusted payment page stole customer payment data. The page was real; the script was not.

<div class="impact">
<strong>Project outcome:</strong> user-generated content displays as text unless you deliberately sanitize and render HTML.
</div>

```jsx
<div>{task.description}</div>
```

---

# CSRF and Chained Attacks

> **Samy worm, 2005:** A profile payload spread across MySpace by combining browser trust, user sessions, and injected behavior.

**Lesson:** vulnerabilities become much more dangerous when they chain together.

<div class="cards">
  <div class="card"><h3>SameSite cookies</h3><p>Reduce cross-site request abuse.</p></div>
  <div class="card"><h3>CSRF tokens</h3><p>Protect state-changing form submissions.</p></div>
  <div class="card"><h3>Minimal trust</h3><p>Do not approve cross-origin access casually.</p></div>
</div>

---

# Security Middleware: Safe Defaults

```javascript
app.use(helmet());
app.use(cors({ origin: process.env.FRONTEND_URL, credentials: true }));
app.use('/api/auth/login', rateLimit({ windowMs: 15 * 60 * 1000, max: 20 }));
app.use(express.json({ limit: '1mb' }));
```

| Middleware | Outcome |
|---|---|
| `helmet` | safer HTTP headers |
| `cors` | only the real frontend can call the API as approved |
| `rateLimit` | brute-force attempts slow down |
| body limit | cheap oversized requests are rejected |

---

<!-- _class: lead -->

# Authentication vs Authorization

## Login is not permission.

---

# AuthN vs AuthZ

> **Strava heat map, 2018:** users were authenticated, but sensitive location patterns became public because data use lacked appropriate authorization and privacy controls.

| | Authentication | Authorization |
|---|---|---|
| **Question** | Who are you? | What can you do? |
| **When** | At login | Every protected request |
| **Failure** | `401` | `403` |
| **TaskBoard** | Brian logged in | Brian can edit this task |

---

# Ownership Checks

```javascript
const canEdit =
  task.createdById === req.user.id ||
  task.assigneeId === req.user.id ||
  task.board.ownerId === req.user.id;
```

<div class="impact">
<strong>Project outcome:</strong> guessing another task ID does not let a logged-in user edit or delete someone else’s data.
</div>

---

# Role-Based Access Control

| Role | View | Create | Edit any | Manage members |
|---|---|---|---|---|
| **Viewer** | Yes | No | No | No |
| **Member** | Yes | Yes | Own only | No |
| **Admin** | Yes | Yes | Yes | Yes |

Permissions become a **product decision**, not scattered `if` statements.

---

<!-- _class: lead -->

# Input Validation

## Client-side validation is UX.  
## Server-side validation is protection.

---

# Validation Stops Surprises

<div class="two">
<div class="story"><strong>GitHub/Rails mass assignment:</strong> the server accepted parameters it should never have allowed.</div>
<div class="story"><strong>NoSQL injection:</strong> unvalidated JSON can turn "find one" into "return everything."</div>
</div>

```javascript
const CreateTaskSchema = z.object({
  title: z.string().min(1).max(200),
  status: z.enum(['todo', 'in_progress', 'done']).default('todo'),
  boardId: z.string().uuid(),
}).strict();
```

**Project outcome:** fields like `isAdmin: true` never reach business logic.

---

# Request Flow

```text
Request
  -> rate limit
  -> parse body
  -> authenticate
  -> validate
  -> authorize
  -> business logic
  -> response
```

**Why this order matters:** cheap rejections happen early, identity is known before permissions, and unsafe data never enters the handler.

---

<!-- _class: lead -->

# Dependency Security

## Your app runs code you did not write.

---

# Supply Chain Risk

<div class="cards">
  <div class="card"><h3>event-stream</h3><p>A maintainer handoff introduced malicious transitive code.</p></div>
  <div class="card"><h3>Log4Shell</h3><p>A common logging library became a global emergency.</p></div>
  <div class="card"><h3>left-pad</h3><p>A tiny package removal broke builds across the ecosystem.</p></div>
</div>

<div class="impact">
<strong>Project outcome:</strong> dependencies are tracked risk, not invisible background noise.
</div>

---

# Dependency Hygiene

```bash
npm audit
npm audit fix --dry-run
npm outdated
```

| Habit | Impact |
|---|---|
| Enable Dependabot | known vulnerabilities become PRs |
| Merge safe security updates | exposure window shrinks |
| Remove unused packages | attack surface gets smaller |
| Avoid unnecessary packages | fewer maintainers to trust |

---

<!-- _class: lead -->

# Performance Basics

## Users do not diagnose slowness.  
## They leave.

---

# Performance Has Product Impact

> **Friendster:** features that worked early became painfully slow as the network grew. Users moved to faster alternatives.

> **Surprise cloud bills:** endpoints that return everything can turn traffic spikes into downtime and expensive infrastructure.

<div class="impact">
<strong>Project outcome:</strong> measure the bottleneck users feel, then fix the highest-impact path first.
</div>

---

# Measure Before Optimizing

<div class="cards">
  <div class="card"><h3>Network tab</h3><p>Find slow, large, or blocking API calls.</p></div>
  <div class="card"><h3>Server logs</h3><p>See which route or query costs the most.</p></div>
  <div class="card"><h3>User workflow</h3><p>Time what the user actually tries to do.</p></div>
</div>

Do not spend hours optimizing a function that was never the bottleneck.

---

# N+1 Queries

<div class="two">
<div>

## Slow pattern

```javascript
const tasks = await prisma.task.findMany();
for (const task of tasks) {
  task.assignee = await prisma.user.findUnique({
    where: { id: task.assigneeId },
  });
}
```

</div>
<div>

## Better pattern

```javascript
const tasks = await prisma.task.findMany({
  include: { assignee: true },
});
```

</div>
</div>

<div class="impact">
<strong>Impact:</strong> one board load does not multiply into dozens of database calls.
</div>

---

# Practical Performance Wins

| Practice | Outcome |
|---|---|
| Index filtered columns | Faster lookups as data grows |
| Select needed fields only | Smaller payloads, less exposure |
| Paginate long lists | Stable page loads |
| Avoid request waterfalls | Faster first usable screen |
| Cache stable data carefully | Less repeated expensive work |

---

<!-- _class: lead -->

# Technical Debt

## Debt is not a moral failure.  
## Untracked debt is risk.

---

# Technical Debt Has Real Consequences

<div class="cards">
  <div class="card"><h3>Healthcare.gov</h3><p>Complexity and integration risk became public launch failure.</p></div>
  <div class="card"><h3>Toyota litigation</h3><p>Maintainability concerns appeared in safety-critical software analysis.</p></div>
  <div class="card"><h3>Knight Capital</h3><p>Dead code and deployment risk created massive trading losses.</p></div>
</div>

---

# The Credit Card Analogy

<div class="cards">
  <div class="card"><h3>Take on debt</h3><p>"Skip tests now so we can demo."</p></div>
  <div class="card"><h3>Pay interest</h3><p>Every future change is slower and riskier.</p></div>
  <div class="card"><h3>Pay it down</h3><p>Refactor, test, simplify, and delete dead code.</p></div>
</div>

<div class="impact">
<strong>Project outcome:</strong> debt becomes an explicit tradeoff instead of an invisible drag.
</div>

---

# Common Student-Project Debt

| Debt | Impact |
|---|---|
| No error handling | blank screens and no diagnosis |
| Copy-pasted route logic | one bug fixed repeatedly |
| Giant route files | teammates avoid important code |
| No critical-path tests | every change is a gamble |
| TODOs with no issue | "later" disappears |

---

# Refactor Toward Safe Change

```javascript
router.post('/api/tasks',
  authenticate,
  validate(CreateTaskSchema),
  authorizeBoard('member'),
  asyncHandler(createTask)
);
```

<div class="impact">
<strong>Project outcome:</strong> auth, validation, authorization, and business logic can be tested and changed independently.
</div>

---

# Managing Debt

<div class="cards">
  <div class="card"><h3>Track it</h3><p>Create GitHub issues labeled <strong>tech-debt</strong>.</p></div>
  <div class="card"><h3>Prioritize it</h3><p>Start with debt that blocks features, causes bugs, or threatens demo quality.</p></div>
  <div class="card"><h3>Slice it</h3><p>Improve one route, module, or critical flow at a time.</p></div>
</div>

**Question:** What part of your project are you afraid to touch?

---

<!-- _class: lead -->

# Demo-Ready Checklist

## The deliverable is a safer, faster, more maintainable app.

---

# Security and Quality Checklist

<div class="checklist">

<p>☐ `.env` is ignored and not tracked</p>
<p>☐ no secrets in frontend code</p>
<p>☐ passwords are hashed</p>
<p>☐ sessions or JWTs expire</p>
<p>☐ auth routes are rate-limited</p>
<p>☐ protected routes require login</p>
<p>☐ ownership or roles checked</p>
<p>☐ server validates request bodies</p>
<p>☐ API responses hide internal fields</p>
<p>☐ CORS is restricted</p>
<p>☐ `helmet` is enabled</p>
<p>☐ `npm audit` has no critical/high issues</p>
<p>☐ slow pages have been measured</p>
<p>☐ long lists are paginated</p>
<p>☐ N+1 patterns are removed</p>
<p>☐ critical flows have tests or a manual test path</p>

</div>

---

# Week 8 Summary

| Topic | Outcome |
|---|---|
| Security | safer defaults and fewer obvious attack paths |
| AuthN/AuthZ | users can only access what they should |
| Validation | bad data is rejected before business logic |
| Dependencies | supply-chain risk becomes visible work |
| Performance | bottlenecks are measured, not guessed |
| Technical debt | risky code becomes tracked and prioritized |

---

# Next Week

## Preview Demo and Operational Quality

- Demo your working application.
- Get feedback on functionality, UX, and technical quality.
- Focus on logging, error handling, deployment reliability, and recovery.
- Leave with a prioritized fix list for the final sprint.

<div class="impact">
Prepare by deploying, running the checklist, and rehearsing the demo flow as a team.
</div>
