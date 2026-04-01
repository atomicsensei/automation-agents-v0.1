# Using AutomationAgents via the `gh aw` CLI

This guide is for users who do not use VS Code. The same QA agent pipeline that runs in GitHub Copilot Chat can also be driven entirely from the terminal using [GitHub Agentic Workflows (`gh aw`)](https://github.github.com/gh-aw/setup/cli/).

---

## How It Works

The `gh aw` CLI compiles agent markdown files into GitHub Actions workflows and dispatches them. Each major step of the QA pipeline (setup, planning, analysis, implementation, review, test run) is exposed as a dispatchable workflow. You trigger them from the terminal; they run in GitHub Actions using your configured AI engine (GitHub Copilot, Claude, etc.).

```
Your terminal
    │
    │  gh aw run qa-chief-planner --raw-field ticket=PROJ-123
    │
    ▼
GitHub Actions (AI engine runs the agent)
    │
    ▼
Results posted back as PR comments / workflow summary
```

---

## Prerequisites

| Requirement | Install |
|-------------|---------|
| GitHub CLI (`gh`) | https://cli.github.com |
| `gh aw` extension | `gh extension install github/gh-aw` |
| Repository write access | Must be able to push to `.github/workflows/` |
| AI engine token | GitHub Copilot token, Anthropic API key, or OpenAI key — depends on your chosen engine |

Verify the setup:

```bash
gh --version
gh aw version
```

---

## One-Time Repository Initialisation

Run this once inside your clone of this repository:

```bash
# 1. Authenticate with GitHub
gh auth login

# 2. Initialise agentic workflows (creates the dispatcher agent file)
gh aw init

# 3. Bootstrap required secrets (engine API key, JIRA token, etc.)
gh aw secrets bootstrap

# 4. Compile all agent markdown files to GitHub Actions YAML
gh aw compile

# 5. (Optional) Verify compilation with full linting
gh aw validate
```

`gh aw init` will interactively ask you which AI engine to use (Copilot, Claude, Codex) and store the selection in workflow frontmatter.

---

## Configuring Secrets

All credential names are listed in `.env.template`. Upload them to GitHub Actions secrets with:

```bash
# Application access
gh aw secrets set BASE_URL_DEV
gh aw secrets set BASE_URL_STAGING
gh aw secrets set TEST_USERNAME
gh aw secrets set TEST_PASSWORD

# Issue tracker (if using JIRA)
gh aw secrets set JIRA_BASE_URL
gh aw secrets set JIRA_EMAIL
gh aw secrets set JIRA_API_TOKEN
gh aw secrets set JIRA_PROJECT_KEY

# Issue tracker (if using GitHub Issues — already available via GITHUB_TOKEN)
# No extra secret needed.

# Proxy (if required)
gh aw secrets set PROXY_URL
```

Or run the interactive bootstrap to let `gh aw` detect what is missing:

```bash
gh aw secrets bootstrap
```

---

## Running the Setup Wizard (First Time)

If `docs/qa-config.json` does not exist in your repository, trigger the setup workflow:

```bash
gh aw run qa-chief-planner --raw-field mode=setup --push
```

The agent will:
1. Walk through the same guided wizard as in VS Code
2. Write `docs/qa-config.json` with all your framework choices
3. Scaffold the folder structure
4. Generate the CI/CD config file
5. Commit and open a pull request for your review

Monitor the run:

```bash
gh aw status --ref main
gh aw logs qa-chief-planner
```

---

## Generating Tests for a Ticket

Once `docs/qa-config.json` exists, dispatch the orchestrator with a ticket or description:

```bash
# From a JIRA ticket ID
gh aw run qa-chief-planner --raw-field ticket=PROJ-123 --push

# From a pasted user story
gh aw run qa-chief-planner \
  --raw-field description="Users should be able to reset their password via email link" \
  --push

# Targeting a specific environment (defaults to the first env in qa-config.json)
gh aw run qa-chief-planner \
  --raw-field ticket=PROJ-123 \
  --raw-field environment=staging \
  --push
```

The `--push` flag commits and pushes any local changes before dispatching, ensuring the agent runs on the latest code.

---

## Pipeline Execution and Parallelism

The `qa-chief-planner` workflow internally coordinates the full agent pipeline:

```
qa-chief-planner  (creates task spec)
        │
        ├──► page-analyst      (parallel, if UI tests needed)
        ├──► api-analyst       (parallel, if API tests needed)
        │
        ▼
  test-engineer
        │
        ▼
  code-implementer
        │
        ▼
  code-reviewer  ──► loop (max 3 rejections)
        │
      PASS
        │
        ▼
  test-runner    ──► loop (max 2 fix cycles)
        │
      PASS → PR opened with results
```

In **gated** orchestration mode (set during setup), the workflow pauses after the task spec is created and posts a comment on the auto-created PR asking for your approval before continuing. Reply with `Approved` on the PR and the pipeline resumes automatically.

In **auto** mode, the entire pipeline runs end-to-end without pausing.

---

## Running Individual Agents

You can trigger any single agent directly if you want to run only part of the pipeline:

```bash
# Re-analyse pages for an existing task spec
gh aw run page-analyst \
  --raw-field task=docs/tasks/task-001-spec.md \
  --push

# Re-run only the reviewer on existing code
gh aw run code-reviewer-agent \
  --raw-field task=docs/tasks/task-001-spec.md \
  --push

# Re-run tests after a manual fix
gh aw run test-runner-agent \
  --raw-field task=docs/tasks/task-001-spec.md \
  --push
```

---

## Monitoring and Observability

```bash
# Quick status of all workflows
gh aw status

# Status with latest run info on main
gh aw status --ref main

# Stream logs for a specific workflow run
gh aw logs qa-chief-planner

# Download and analyse logs from the last week
gh aw logs qa-chief-planner -c 10 --start-date -1w

# Detailed audit of a specific run (by run ID or URL)
gh aw audit 12345678
gh aw audit https://github.com/owner/repo/actions/runs/12345678

# Health metrics over time
gh aw health qa-chief-planner --days 30
gh aw health --threshold 90   # alert if success rate drops below 90%
```

---

## Updating the Agents

When new versions of the agent files are released, compile and push the updates:

```bash
# Pull latest agent changes
git pull

# Recompile all workflows
gh aw compile

# Fix any deprecated fields automatically
gh aw fix --write

# Push updated lock files
git add .github/workflows/*.lock.yml
git commit -m "chore: recompile agent workflows"
git push
```

Or let `gh aw` do everything:

```bash
gh aw upgrade --create-pull-request
```

---

## Shell Completions (Recommended)

Enable tab completion for workflow names, engines, and flags:

```bash
# Bash
gh aw completion bash > ~/.bash_completion.d/gh-aw && source ~/.bash_completion.d/gh-aw

# Zsh
gh aw completion zsh > "${fpath[1]}/_gh-aw" && compinit

# Fish
gh aw completion fish > ~/.config/fish/completions/gh-aw.fish

# PowerShell
gh aw completion powershell | Out-String | Invoke-Expression
```

---

## GitHub Enterprise Server

If your organisation uses GitHub Enterprise Server, configure the host before running any commands:

```bash
export GH_HOST="github.your-company.com"
gh auth login --hostname github.your-company.com
gh aw init
```

The compiled agent workflows automatically detect and configure `gh` CLI for the correct GHES instance at runtime.

---

## Differences from VS Code

| Feature | VS Code (Copilot Chat) | CLI (`gh aw`) |
|---------|----------------------|---------------|
| Conversation style | Interactive chat, one question at a time | Workflow dispatched with `--raw-field` inputs |
| Setup wizard | Guided Q&A in chat | `gh aw run qa-chief-planner --raw-field mode=setup` |
| Spec approval (gated mode) | Reply in chat | Comment `Approved` on the auto-created PR |
| Results | Displayed in chat | Posted as PR comments and workflow summaries |
| Logs | Shown inline | `gh aw logs` / `gh aw audit` |
| File access | Direct workspace reads | Agent clones the repo in the Actions runner |

---

## Quick Reference

```bash
# Install
gh extension install github/gh-aw

# One-time setup
gh aw init
gh aw secrets bootstrap
gh aw compile

# Run setup wizard (first time, no qa-config.json)
gh aw run qa-chief-planner --raw-field mode=setup --push

# Generate tests for a ticket
gh aw run qa-chief-planner --raw-field ticket=PROJ-123 --push

# Monitor
gh aw status --ref main
gh aw logs qa-chief-planner
gh aw health

# Update agents
gh aw upgrade --create-pull-request
```

---

## Further Reading

- [gh aw CLI reference](https://github.github.com/gh-aw/setup/cli/)
- [Quick Start](https://github.github.com/gh-aw/setup/quick-start/)
- [Frontmatter reference](https://github.github.com/gh-aw/reference/frontmatter/)
- [Authentication guide](https://github.github.com/gh-aw/reference/auth/)
- [MCP server configuration](https://github.github.com/gh-aw/reference/gh-aw-as-mcp-server/)
- [Troubleshooting common issues](https://github.github.com/gh-aw/troubleshooting/common-issues/)
