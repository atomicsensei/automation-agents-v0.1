# AutomationAgents

> AI-powered agent pipeline that scaffolds, writes, reviews, and runs test automation — driven entirely through GitHub Copilot Chat.

---

## What Is This?

AutomationAgents is a collection of specialised GitHub Copilot agents that collaborate as a pipeline to build and maintain a test automation framework from scratch. You describe what needs to be tested; the agents handle the rest — from project scaffolding and page object creation to writing specs, reviewing code, and executing tests.

---

## Agent Roster

| Agent | Invoke as | Role |
|-------|-----------|------|
| **QA Chief Planner** | `@QA Chief Planner` | Entry point. Runs the setup wizard, saves project config, then orchestrates all other agents for every test request. |
| **Page Analyst** | `@Page Analyst` | Navigates to UI pages or mobile screens, captures every interactive element, and produces a full selector report plus ready-to-use Page Object / Screen Object code. |
| **API Analyst** | `@API Analyst` | Monitors network traffic, extracts request/response shapes, and produces typed factory functions, endpoint constants, and helper methods. |
| **QA Test Engineer** | `@QA Test Engineer` | Writes complete, runnable test spec files from the task spec and analyst outputs, following the project's framework and conventions. |
| **QA Code Implementer** | `@QA Code Implementer` | Creates and updates Page Objects, factories, API helpers, and fixture registrations so the Test Engineer's specs compile and run. |
| **code-reviewer-agent** | `@code-reviewer-agent` | Runs the linter, audits specs and support files against project conventions, and returns precise file-and-line feedback — approve or reject. |
| **test-runner-agent** | `@test-runner-agent` | Executes the newly created test files, parses results, and reports pass/fail with full error context so the Chief Planner can trigger a fix cycle if needed. |

---

## How to Get Started

### First-Time Setup

Open GitHub Copilot Chat and invoke the Chief Planner:

```
@QA Chief Planner
```

It will detect that no `docs/qa-config.json` exists and launch the **Setup Wizard** — a short, guided conversation covering:

1. **Testing type** — UI, Mobile, API, Mixed, or Full Stack
2. **Framework & language** — Playwright/TS, Selenium/Java, Cypress, Appium, pytest, and more; or bring your own
3. **API testing scope** — built-in or dedicated layer, test categories
4. **CI/CD platform** — GitHub Actions, GitLab CI, Jenkins, Azure DevOps, CircleCI, Bitbucket
5. **Issue tracker** — JIRA, GitHub Issues, Azure DevOps Boards, Linear, or none
6. **Application access** — base URL, auth method, proxy, certificates
7. **Design pattern & folder structure** — POM + Fixtures, Page Factory, Screen Object Model, and others
8. **Linting & code quality** — ESLint, Prettier, Checkstyle, flake8, ruff, ktlint, SwiftLint, and more
9. **Orchestration mode** — choose how the pipeline runs:
   - **Automatic** — agents chain together without pausing; fastest option
   - **Gated** — the Planner stops at each stage for your explicit approval before continuing

At the end, the wizard:
- Writes `docs/qa-config.json` with all your choices
- Scaffolds the full folder structure with starter files
- Generates a CI/CD config file with `# REPLACE` placeholders
- Creates `.env.template` with all required secret variable names (never real values)

---

### Writing Tests for a Feature (Orchestrator Mode)

Once `docs/qa-config.json` exists, give the Chief Planner a ticket or description:

```
@QA Chief Planner  PROJ-123
```

```
@QA Chief Planner  Users should be able to reset their password via email link
```

The Planner will:

1. Fetch ticket details from your configured issue tracker (if a ticket ID is given)
2. Clarify any ambiguities, then write a task spec at `docs/tasks/task-XXX-spec.md`
3. **[Gated mode]** Show you the spec and wait for `Approved` before continuing
4. Dispatch **Page Analyst** and **API Analyst** in parallel (if both are needed)
5. Hand the analyst outputs to the **QA Test Engineer** to write spec files
6. Hand the spec files to the **QA Code Implementer** to build supporting code
7. Send everything to the **code-reviewer-agent** — up to 3 rejection/fix cycles
8. Run tests with the **test-runner-agent** — up to 2 fix cycles before escalating
9. Mark all acceptance criteria complete in the task spec and report results

---

## Pipeline Diagram

```
[Ticket / User Story]
        │
        ▼
  QA Chief Planner
  (creates task spec)
        │
  [Gated: awaits approval]
        │
        ├──► Page Analyst ──────────────┐
        ├──► API Analyst  ──────────────┤  (parallel)
        │                               │
        ▼                               ▼
       Test Engineer  ◄─────── analyst outputs
        │
        ▼
  Code Implementer
        │
        ▼
  Code Reviewer ──► REJECT (max 3) ──► Code Implementer
        │
     APPROVE
        │
        ▼
  Test Runner ──► FAIL (max 2) ──► Code Implementer / Test Engineer
        │
      PASS
        │
        ▼
  [Task spec ACs marked complete]
```

---

## Key Files & Folders

| Path | Purpose |
|------|---------|
| `.github/agents/` | Agent definition files — each agent's instructions live here |
| `docs/qa-config.json` | Project configuration written by the setup wizard; read by every agent |
| `docs/tasks/` | Auto-generated task spec files (`task-001-spec.md`, etc.) |
| `.env.template` | Secret variable name placeholders — copy to `.env` and fill in real values |
| `.env` | **Git-ignored.** Your actual credentials and environment URLs go here |

---

## Orchestration Modes

| Mode | Behaviour | Best For |
|------|-----------|----------|
| `auto` | Agents chain together without any human approval gates | Trusted pipelines, CI runs, high-volume test generation |
| `gated` | Planner pauses at each major stage and waits for explicit approval | Teams that want full visibility and control over each step |

The mode is set during setup and stored in `docs/qa-config.json` → `"orchestrationMode"`. You can edit it at any time.

---

## Supported Frameworks

**UI / Web:** Playwright (TS/JS), Cypress (TS/JS), Selenium (Java, Python), WebdriverIO (TS)

**Mobile:** Appium (Java, TS, Python), Detox (JS), XCUITest (Swift), Espresso (Kotlin/Java)

**API:** Playwright native, Rest Assured, Supertest, pytest + httpx, Karate DSL, RestSharp

**Custom:** Any framework — describe it in plain language and all agents adapt accordingly

---

## Security Notes

- Real credentials must **only** go in `.env` — never in any version-controlled file.
- `.env.template` contains variable names only, no values.
- The CI/CD config uses secret references (`${{ secrets.MY_VAR }}`) — never hardcoded values.
- JIRA integration mode is a **customer choice** set during setup and stored in `docs/qa-config.json` → `issueTracker.jiraIntegration`:
  - `read-only` (default) — agents only call `get_jira_issue` to fetch ticket data. No writes, no status changes, ever.
  - `read-write` — agents can use JIRA MCP to post comments, transition ticket status, and update fields (e.g. comment when tests pass, transition to "In Testing"). Requires JIRA MCP to be configured.

