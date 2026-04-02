# Atlassian Cloud Automation — AI Skill

A portable knowledge package that turns any major LLM (Claude, ChatGPT, Gemini) into an expert Atlassian Cloud engineer. It generates production-ready Python automation scripts for Jira, Confluence, JSM, and Atlassian Analytics — with safety guardrails built in.

---

## What Is This?

If you use an AI assistant to help automate Atlassian Cloud, this skill teaches it **how to do it correctly**.

Without the skill, an LLM might:
- Use deprecated Jira search endpoints that return `410 Gone`
- Silently drop real users by filtering on the wrong account type
- Skip the dry-run gate and modify live data
- Generate scripts without audit logging

With the skill, the LLM already knows all of this. It generates scripts that follow a battle-tested template: environment selection, dry-run protection, retry logic, concurrent execution, and CSV audit logging — every time.

---

## Who Is This For?

- **Atlassian Admins** managing Cloud instances at scale
- **Atlassian Consultants** who use AI to write automation scripts
- **IT Teams** running Confluence permission migrations, Jira bulk operations, or user audits
- Anyone who's tired of debugging Atlassian API quirks by trial and error

You don't need to be a developer. Describe what you want in plain English — the LLM handles the technical details.

---

## What Can It Do?

| Area | Examples |
|------|---------|
| **Jira** | Search issues with JQL, create/update/transition issues, bulk operations, sprint & epic management, custom field discovery |
| **Confluence** | Manage space permissions, migrate to RBAC roles, create/update pages, audit space members |
| **Users & Groups** | Check active status, filter bots from humans, manage group membership via Organizations API |
| **Analytics** | Write correct Atlassian Analytics SQL queries (with the right table names — yes, they're tricky) |
| **Safety** | Every script has dry-run mode, audit logging, rate-limit protection, and environment selection |

---

## Repository Structure

```
Atlassian-Skill/
├── README.md                        ← You are here
├── config.sample.json               ← Copy this, fill in credentials
├── .gitignore
├── LICENSE
│
├── skill/                           ← The AI skill (LLM-agnostic)
│   ├── SYSTEM_PROMPT.md             ← Paste into any LLM's system prompt
│   ├── SKILL.md                     ← Claude Projects / Claude Code format
│   ├── references/
│   │   ├── api_gotchas.md           ← Battle-tested API quirks & fixes
│   │   ├── jql_guide.md             ← JQL syntax reference
│   │   ├── custom_field_discovery.md
│   │   ├── jira_api_reference.md    ← Jira v3 endpoint catalog
│   │   ├── confluence_api_reference.md
│   │   └── org_api_reference.md
│   └── templates/
│       ├── base_script.py           ← Script template (starting point)
│       └── issue_creation.json      ← Issue field format reference
│
└── llm-setup/                       ← Per-LLM setup guides
    ├── claude.md                    ← Claude Projects / Claude Code / API
    ├── openai.md                    ← Custom GPTs / Assistants API
    └── gemini.md                    ← Gems / AI Studio / Gemini API
```

**The key file is `skill/SYSTEM_PROMPT.md`** — it contains all engineering standards, API gotchas, and code patterns in a format any LLM understands. Everything else is supplementary.

---

## Quick Start

### 1. Pick Your LLM

Follow the setup guide for your AI assistant:

| LLM | Guide |
|-----|-------|
| Claude | [`llm-setup/claude.md`](llm-setup/claude.md) |
| ChatGPT / OpenAI | [`llm-setup/openai.md`](llm-setup/openai.md) |
| Gemini | [`llm-setup/gemini.md`](llm-setup/gemini.md) |

The short version: paste `skill/SYSTEM_PROMPT.md` into the system instructions field of whichever LLM you're using.

### 2. Ask for a Script

```
Write me a script that finds all Jira issues in project MYPROJ
that haven't been updated in 90 days and exports them to CSV.
```

The LLM will generate a complete Python script using the base template.

### 3. Set Up Credentials

The first time you run the script, it will create a `config.json` template next to itself and exit with instructions:

```
⚠️  config.json was not found — created a template at:
   /path/to/your/script/config.json

   Open it and replace the placeholder values with your Atlassian
   credentials, then re-run this script.
```

Fill in your details:

```json
{
  "environments": {
    "Production": {
      "Production": {
        "base_url": "https://your-instance.atlassian.net",
        "email": "you@company.com",
        "api_token": "your-api-token-here"
      }
    }
  }
}
```

**Get an API token:** [id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens)

### 4. Run

```
$ python my_script.py

Select Application:
1. Jira
2. Confluence
Enter 1 or 2: 1

Select Environment for Jira:
1. Production
Enter 1-1: 1

⚠️  Targeting: Production (https://your-instance.atlassian.net)
Enable Dry Run? (Y/n): Y

🚀 Execution Started | Mode: 🛡️ DRY RUN
```

Dry Run is on by default. You must type `n` explicitly to run live.

---

## How Every Generated Script Works

```
┌─────────────────────────────────────────────┐
│  1. Pre-flight    Auto-install dependencies  │
│  2. Config        Select environment         │
│  3. Dry Run Gate  Safe by default            │
│  4. Execute       Concurrent + retries       │
│  5. Audit Log     CSV with every action      │
└─────────────────────────────────────────────┘
```

**Dry Run** prints what would happen without making any changes. Default — you have to actively opt out.

**Audit logs** (`Audit_Log_YYYYMMDD_HHMM.csv`) record every action: timestamp, environment, action type, entity key, status, and HTTP response code. Useful for compliance, rollback, and debugging.

---

## API Gotchas This Skill Knows About

| Gotcha | What Goes Wrong | What the Skill Does |
|--------|----------------|-------------------|
| Old Jira search endpoints | `GET /rest/api/3/search` returns 410 | Uses `POST /rest/api/3/search/jql` |
| Silent `maxResults` cap | Requesting 100 results returns 50 | Uses `nextPageToken` to paginate correctly |
| Wrong account type filter | `accountType == "atlassian"` drops real users | Filters with `!= "app"` instead |
| `accountStatus` not in Confluence v1 | Can't tell if a user is active | Uses the Identity API |
| Analytics table name | `jira_issue_field_history` doesn't exist | Uses `jira_issue_history` |
| Jira 429 rate limits | Bulk operations crash | Reads `Retry-After`, exponential backoff, 5 workers |

See [`skill/references/api_gotchas.md`](skill/references/api_gotchas.md) for the full list with code examples.

---

## Adding Environments

Add as many environments as you need to `config.json` — the script lists them all at startup:

```json
{
  "environments": {
    "Production": {
      "Production": { "base_url": "...", "email": "...", "api_token": "..." }
    },
    "Sandbox": {
      "Sandbox": { "base_url": "...", "email": "...", "api_token": "..." }
    },
    "Client-ABC": {
      "Client-ABC": { "base_url": "...", "email": "...", "api_token": "..." }
    }
  }
}
```

---

## Security

- **Never commit `config.json`** — it contains your API tokens. The `.gitignore` already excludes it.
- **Use a service account** with minimal permissions for automation scripts, not your personal account.
- **Dry Run is on by default.** You must explicitly type `n` to make real changes.
- All scripts generate audit logs so you always have a record of what ran.

---

## Contributing

Found a new API gotcha? Have a better pattern? PRs welcome.

Most valuable contributions:
- New entries in `skill/references/api_gotchas.md`
- Corrections to endpoint references
- New script templates for common operations

---

## License

MIT — use it however you like.
