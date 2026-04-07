# Salesforce Development Skill for Cursor IDE

An agent skill that turns Cursor into an autonomous Salesforce development environment. The agent writes code following all Salesforce best practices, runs `sf` CLI commands to deploy/test/validate, and performs code reviews — all without manual intervention.

## What This Skill Does

When active, the Cursor agent will automatically:

- **Discover what changed** — finds recently modified classes, triggers, LWC, and Aura components by scanning file timestamps or git diffs (no need to tell it which files)
- **Review modified code** for governor limit violations, missing CRUD/FLS, hardcoded IDs, SOQL injection, and more
- **Extract SOQL from changed files** and run explain plans via the REST API to check query selectivity (no Developer Console needed)
- **Write tests as it goes** — scans the project for existing test data factories, bypass patterns, and mock conventions, then generates tests that follow the same style
- **Run tests after every change** — finds the corresponding test class, runs it, and fixes failures before moving on
- **Deploy code** using `sf project deploy start` with a full preview → validate → deploy → report workflow
- **Enforce SLDS compliance** in all LWC/Aura components
- **Generate meta.xml files** with the correct API version from `sfdx-project.json`
- **Fix issues** found during validation and re-deploy automatically

## Coverage

| Area | What's Enforced |
|------|----------------|
| **Apex** | Bulkification, governor limits, one-trigger-per-object, handler pattern, Service/Domain/Selector layers, async patterns (Queueable/Batch/Future/Schedulable), error handling, code quality |
| **Apex Security** | `with sharing` by default, CRUD/FLS via `WITH USER_MODE`, SOQL injection prevention, no hardcoded credentials |
| **Apex Testing** | Auto-generates tests by scanning existing factories/patterns, 85%+ coverage, `@TestSetup`, bulk testing, `System.runAs`, callout mocking, continuous run-after-every-change loop |
| **LWC** | SLDS-first UI (base components → blueprints → styling hooks), `lwc:if` (not deprecated `if:true`), reactive `@wire`, lifecycle cleanup, debounce, accessibility |
| **LWC Testing** | Jest setup, `__tests__/` structure, DOM cleanup, async re-render awaits, Apex mocking |
| **Aura** | Maintenance-mode guidance, `$A.enqueueAction` pattern, naming pitfalls, migration to LWC |
| **SOQL/SOSL** | Selective queries, indexed fields, relationship queries, `WITH USER_MODE`, no leading wildcards, explain plans via CLI |
| **Flows** | Before-save preferred, bulkification, no SOQL/DML in loops, subflows, Transform element |
| **Integration** | Named Credentials mandatory, no DML before callouts, async callout patterns, retry logic |
| **Deployment** | sf CLI v2 commands, preview → validate → deploy workflow, manifest/source-dir/metadata modes |

## Installation

### Option A: Personal Skill (available in all your projects)

```bash
# Create the skill directory
mkdir -p ~/.cursor/skills/salesforce-dev

# Copy all files
cp SKILL.md apex-best-practices.md lwc-best-practices.md soql-sosl-best-practices.md security-best-practices.md ~/.cursor/skills/salesforce-dev/
```

### Option B: Project Skill (shared with your team via Git)

```bash
# From your Salesforce project root
mkdir -p .cursor/skills/salesforce-dev

# Copy all files
cp /path/to/this/repo/SKILL.md .cursor/skills/salesforce-dev/
cp /path/to/this/repo/apex-best-practices.md .cursor/skills/salesforce-dev/
cp /path/to/this/repo/lwc-best-practices.md .cursor/skills/salesforce-dev/
cp /path/to/this/repo/soql-sosl-best-practices.md .cursor/skills/salesforce-dev/
cp /path/to/this/repo/security-best-practices.md .cursor/skills/salesforce-dev/

# Commit to share with your team
git add .cursor/skills/
git commit -m "Add Salesforce development skill for Cursor"
```

### Option C: Clone directly into skills

```bash
# Personal
git clone https://github.com/<your-org>/salesforce-cursor-skill.git ~/.cursor/skills/salesforce-dev

# Project
git clone https://github.com/<your-org>/salesforce-cursor-skill.git .cursor/skills/salesforce-dev
```

After installing, **reload Cursor** (`Cmd+Shift+P` → `Developer: Reload Window` on macOS, or `Ctrl+Shift+P` on Windows/Linux) for the skill to activate.

## Prerequisites

1. **Salesforce CLI (sf v2)** installed:

```bash
npm install -g @salesforce/cli
sf --version
```

2. **Authenticated org** with an alias:

```bash
sf org login web --alias myOrg --set-default
```

3. **sfdx-project.json** in your project root (standard Salesforce DX project structure)

4. **Node.js + npm** (for LWC Jest tests):

```bash
# One-time Jest setup for your project
sf force lightning lwc test setup
```

## Usage Examples

Once the skill is active, use natural language in Cursor:

### Review recently modified files

> "Review recent changes"

The agent finds all recently modified `.cls`, `.trigger`, `.js`, `.html` files itself, reads each one, checks for best practice violations, extracts SOQL and runs explain plans, and reports findings.

> "Check query plans for anything I changed today"

The agent finds modified files, extracts SOQL queries from them, runs REST API explain plans on each, and reports cost/selectivity.

### Deploying

> "Deploy the GleanService class"

The agent runs: `sf project deploy start --metadata ApexClass:GleanService`

> "Deploy the enrollmentTimelineModal component"

The agent runs: `sf project deploy start --source-dir force-app/main/default/lwc/enrollmentTimelineModal`

> "Deploy everything and run tests"

The agent runs the full preview → validate → deploy → report workflow automatically.

### Testing and test generation

> "Write tests for MyNewService"

The agent first scans the project for existing test data factories, bypass patterns, and mock conventions. Then it generates a test class following those exact patterns — reusing the same factory methods, same bypass custom settings, same mock style.

> "Run tests for my recent changes"

The agent discovers which files changed, finds their test classes, and runs them. If a test class is missing, it creates one.

> "Make sure I haven't broken anything"

The agent finds all modified classes, runs their associated tests, and reports pass/fail with details.

### Code Review

> "Review GleanService.cls for best practices"

The agent reads the file and checks for: SOQL/DML in loops, missing sharing keywords, CRUD/FLS enforcement, hardcoded IDs, test coverage, and more.

### Diagnostics

> "Check org limits"

The agent runs: `sf limits api display`

> "What's deployed differently from local?"

The agent runs: `sf project deploy preview --source-dir force-app`

## File Structure

```
salesforce-dev/
├── README.md                   # This file — setup and usage guide
├── SKILL.md                    # Main skill entry point (agent reads this first)
├── apex-best-practices.md      # Full Apex reference (governor limits, patterns, testing)
├── lwc-best-practices.md       # Full LWC reference (SLDS, lifecycle, Jest, accessibility)
├── soql-sosl-best-practices.md # Full SOQL/SOSL reference (selectivity, security, performance)
└── security-best-practices.md  # Full security reference (sharing, CRUD/FLS, XSS, CSP)
```

The `SKILL.md` is the primary file the agent loads. It contains summaries and links to the detailed reference files, which the agent reads on-demand when deeper guidance is needed (progressive disclosure to optimize context window usage).

## Customization

You can tailor the skill to your team's conventions:

- **API version**: The agent reads `sfdx-project.json` automatically, but you can update version references in the files if needed
- **Test level**: Change `RunLocalTests` to `RunSpecifiedTests` or `RunAllTestsInOrg` in deploy commands
- **Target org**: Add your org alias to the deploy commands if you use a non-default org
- **Additional rules**: Add team-specific coding standards to any of the reference files

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Skill not appearing | Reload Cursor window (`Cmd/Ctrl+Shift+P` → `Developer: Reload Window`) |
| `sf: command not found` | Install Salesforce CLI: `npm install -g @salesforce/cli` |
| Deploy fails with auth error | Re-authenticate: `sf org login web --alias myOrg --set-default` |
| Jest tests not found | Run setup: `sf force lightning lwc test setup` |
| "No default org" error | Set default: `sf config set target-org myOrg --global` |

## License

MIT
