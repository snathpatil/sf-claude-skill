---
name: salesforce-dev
description: >-
  Develop, lint, and deploy Salesforce metadata (Apex, LWC, Aura, triggers,
  objects, flows) using sf CLI commands. Enforces Apex governor-limit-safe
  patterns, SLDS-compliant UI, one-trigger-per-object design, bulkified code,
  security (CRUD/FLS/sharing), and testing standards. Use when writing,
  reviewing, deploying, or testing Salesforce code, or when the user mentions
  Apex, LWC, Aura, SOQL, triggers, flows, deploy, or sf CLI.
---

# Salesforce Development & Deployment

## Agent Operating Principles

**You are an autonomous agent.** Execute sf CLI commands directly — never ask the user
to run commands manually or open Developer Console / browser. All validation, testing,
deployment, and diagnostics MUST be performed by running shell commands yourself.

When the user asks to deploy, test, review, or validate:
1. Run the appropriate `sf` commands yourself via the Shell tool
2. Read and interpret the command output
3. Fix any issues found, then re-run
4. Report results back to the user

---

## 1. sf CLI Commands — Execute These Directly

Always use **sf** (v2). Never use deprecated `sfdx force:source:*` commands.

### Deploy

```bash
# Full project
sf project deploy start --source-dir force-app

# Specific Apex class
sf project deploy start --metadata ApexClass:MyClassName

# Specific LWC component
sf project deploy start --source-dir force-app/main/default/lwc/myComponent

# Specific Aura bundle
sf project deploy start --source-dir force-app/main/default/aura/myAuraComponent

# Specific trigger
sf project deploy start --metadata ApexTrigger:MyTrigger

# Multiple components
sf project deploy start --metadata ApexClass:ClassA --metadata ApexClass:ClassB --metadata LightningComponentBundle:compA

# By manifest
sf project deploy start --manifest manifest/package.xml

# To non-default org
sf project deploy start --source-dir force-app --target-org myOrgAlias

# With tests
sf project deploy start --source-dir force-app --test-level RunLocalTests
```

### Validate (dry-run)

```bash
sf project deploy preview --source-dir force-app
sf project deploy validate --source-dir force-app --test-level RunLocalTests
sf project deploy quick --job-id <validationJobId>
```

### Retrieve

```bash
sf project retrieve start --metadata ApexClass:MyClassName
sf project retrieve start --source-dir force-app
sf project retrieve start --manifest manifest/package.xml
```

### Test (run these yourself — do not ask the user)

```bash
# Run all local tests
sf apex run test --test-level RunLocalTests --result-format human --wait 10

# Run specific test class
sf apex run test --class-names MyClassTest --result-format human --wait 10

# Run LWC Jest tests
npm test

# Run Jest with coverage
npm test -- --coverage
```

### Query & Diagnostics (run these yourself)

```bash
# Query data
sf data query --query "SELECT Id, Name FROM Account LIMIT 5" --result-format human

# Check SOQL query performance (query explain plan via REST API)
sf api request rest "/services/data/v64.0/query/?explain=SELECT+Id,Name+FROM+Account+WHERE+Industry='Technology'" --method GET

# Run anonymous Apex
sf apex run --file scripts/anon.apex

# List orgs
sf org list

# Open org in browser (only when user explicitly requests)
sf org open --target-org myAlias

# Check org limits
sf limits api display

# View recent deploy status
sf project deploy report
```

### Automated Deploy Workflow

When the user says "deploy", execute this sequence automatically:

1. **Preview** — run `sf project deploy preview --source-dir force-app` and read output
2. **Validate** — run `sf project deploy validate --source-dir force-app --test-level RunLocalTests` and check results
3. **Deploy** — only if validation passes, run `sf project deploy start --source-dir force-app`
4. **Verify** — run `sf project deploy report` to confirm success
5. Report the final status to the user

If any step fails, read the error output, attempt to fix the issue, and retry.

---

## 2. Discovering Recently Modified Files

When the user says "review recent changes", "check modified files", or "what changed" — do NOT
ask which files. Discover them yourself using these commands:

### Find recently modified Apex classes and triggers

```bash
# Files modified in the last N days (adjust -mtime as needed)
find force-app/main/default/classes -name "*.cls" -mtime -1 | sort
find force-app/main/default/triggers -name "*.trigger" -mtime -1 | sort

# Files modified today
find force-app/main/default -name "*.cls" -o -name "*.trigger" -o -name "*.js" -o -name "*.html" -o -name "*.css" | xargs ls -lt 2>/dev/null | head -20

# If it's a git repo, use git for precise diffs
git diff --name-only HEAD~5 -- force-app/
git diff --name-only --cached -- force-app/
git status --short -- force-app/
```

### Find recently modified LWC and Aura components

```bash
find force-app/main/default/lwc -name "*.js" -o -name "*.html" -o -name "*.css" | xargs ls -lt 2>/dev/null | head -20
find force-app/main/default/aura -name "*.cmp" -o -name "*.js" | xargs ls -lt 2>/dev/null | head -20
```

### Workflow when user says "review recent changes"

1. Run the discovery commands above to get a list of modified files
2. Read each modified file
3. Run the Automated Code Review checklist (section 3) against each file
4. For each modified file containing SOQL, extract and verify query plans (section 4)
5. For each modified Apex class, find/generate tests and run them (section 5)
6. Report findings back to the user

---

## 3. Automated Code Review

When reviewing Apex/LWC code, **check these automatically by reading the source files**:

### Apex static analysis (read the file and verify)

- [ ] No SOQL or DML inside `for` / `while` / `do` loops
- [ ] Class has explicit sharing keyword (`with sharing`, `inherited sharing`, or documented `without sharing`)
- [ ] CRUD/FLS enforced: look for `WITH USER_MODE`, `WITH SECURITY_ENFORCED`, or `Security.stripInaccessible`
- [ ] No hardcoded record IDs (18-char or 15-char patterns like `001...`, `a0B...`)
- [ ] No hardcoded credentials, tokens, or endpoints (use Named Credentials)
- [ ] Trigger delegates to handler class (not inline business logic)
- [ ] Only one trigger per object
- [ ] Dynamic SOQL uses bind variables or `String.escapeSingleQuotes`
- [ ] Test class exists with `@isTest`, `@TestSetup`, bulk scenarios

### LWC static analysis (read the files and verify)

- [ ] Templates use `lwc:if` / `lwc:elseif` / `lwc:else` — not deprecated `if:true` / `if:false`
- [ ] `for:each` has `key` attribute with unique value
- [ ] Uses Lightning base components and SLDS classes — no hardcoded colors/spacing
- [ ] CSS does not override SLDS classes or use `!important`
- [ ] `disconnectedCallback()` cleans up listeners/intervals if `connectedCallback()` sets them up
- [ ] Error handling on all `@wire` and imperative Apex calls
- [ ] No `console.log` in production code

### After review, run tests automatically

```bash
sf apex run test --class-names <RelatedTestClass> --result-format human --wait 10
```

---

## 4. Auto-Detect and Verify SOQL from Modified Files

When the user says "check query plans", "check SOQL performance", or "review queries in
recent changes" — do NOT ask which queries. Extract them yourself from modified files.

### Step-by-step (execute all of this autonomously)

1. **Find modified files** (use section 2 commands)
2. **Extract SOQL from those files** — search for patterns like `[SELECT`, `Database.query(`, `Database.getQueryLocator(`

```bash
# Find all SOQL in recently modified Apex files
rg -n "\[SELECT|Database\.query\(|Database\.getQueryLocator\(" force-app/main/default/classes/MyChangedClass.cls
```

3. **For each extracted query**, URL-encode it and run the explain plan:

```bash
# Read the API version from sfdx-project.json first
# Then call the explain endpoint for each query found
sf api request rest "/services/data/v64.0/query/?explain=SELECT+Id,Name+FROM+Account+WHERE+Industry='Technology'" --method GET
```

4. **Parse the JSON response** and check:
   - `leadingOperationType` — must be `Index`, not `TableScan`
   - `cost` — must be `< 1.0` for selective queries
   - `cardinality` — estimated records touched (lower is better)
   - `sObjectCardinality` — total records in the object (context)

5. **Report results** — for each query, report: the query text, the cost, the operation type,
   and whether it is selective. If any query is non-selective (cost >= 1.0 or TableScan),
   suggest specific WHERE clause changes using indexed fields.

### Which queries to skip

- Queries inside `@isTest` classes (test-only queries are not production-critical)
- Queries already using `Id` or `Id IN :collection` (inherently selective)
- Queries with `LIMIT 1` (bounded)

See [soql-sosl-best-practices.md](soql-sosl-best-practices.md) for full selectivity rules.

---

## 4. Apex Best Practices (summary)

For the full reference, see [apex-best-practices.md](apex-best-practices.md).

### Governor limits (critical)

| Limit | Sync | Async |
|-------|------|-------|
| SOQL queries | 100 | 200 |
| DML statements | 150 | 150 |
| Records retrieved | 50,000 | 50,000 |
| CPU time (ms) | 10,000 | 60,000 |
| Heap size | 6 MB | 12 MB |
| Callouts | 100 | 100 |

### Core rules

- **Bulkify everything** — never SOQL or DML inside loops
- **One trigger per object** — delegate to handler classes
- **Use `with sharing` by default** — only `without sharing` with documented justification
- **Enforce CRUD/FLS** — `WITH USER_MODE`, `WITH SECURITY_ENFORCED`, or `Security.stripInaccessible()`
- **No hardcoded IDs** — use Custom Metadata, Custom Settings, or Custom Labels
- **Test coverage >= 85%** — test positive, negative, bulk (200+ records), and permission scenarios
- **Use `@TestSetup`** — shared test data; never `seeAllData=true`
- **Async for heavy work** — Queueable > @future; Batch for large datasets; Schedulable for cron

---

## 5. Continuous Test Generation & Regression Detection

**Write tests as you go.** After every code change, find or create the corresponding test
class, run it, and fix failures — all autonomously. Never leave a class without test coverage.

### Step 1: Discover existing test patterns in the project

Before writing any test, **learn how this project writes tests** by scanning the codebase:

```bash
# Find all test classes
rg -l "@isTest|@IsTest" force-app/main/default/classes/ --glob "*.cls"

# Find test data factories / utility classes
rg -l "TestDataFactory|TestData|TestUtility|TestHelper|TestSetup" force-app/main/default/classes/ --glob "*.cls"

# Find the most common test setup patterns (bypass triggers, custom settings, etc.)
rg -n "@TestSetup|testSetup|setup\(\)" force-app/main/default/classes/ --glob "*Test*.cls" -l

# Find bypass/isolation patterns used in tests
rg -n "bypassTrigger|BypassRulesTriggers|TriggerSettingsUtil|ByPass_Validation" force-app/main/default/classes/ --glob "*Test*.cls"

# Find HTTP mock patterns
rg -n "HttpCalloutMock|Test\.setMock|MultiStaticResourceCalloutMock" force-app/main/default/classes/ --glob "*Test*.cls" -l
```

Read 2-3 of the discovered test classes to understand:
- How `@TestSetup` methods are structured
- Which test data factory/utility classes exist and what methods they provide
- What bypass/isolation patterns are used (custom settings, static flags, etc.)
- How HTTP mocks are implemented (inner class vs shared mock)
- Naming conventions: `*Test.cls`, `*_Test.cls`, `Test*.cls`

### Step 2: Find the right test class for a changed file

```bash
# For a class named "MyService", search for its test class
rg -l "MyService" force-app/main/default/classes/ --glob "*Test*.cls"

# Also check by naming convention
ls force-app/main/default/classes/MyServiceTest.cls 2>/dev/null
ls force-app/main/default/classes/MyService_Test.cls 2>/dev/null
```

If no test class exists, **create one** following the patterns discovered in Step 1.

### Step 3: Generate smart tests

When writing or updating tests, follow these rules:

**Data setup — reuse what exists:**
- Search for existing test data factory classes (Step 1) and use their methods
- If the factory doesn't have a method for the SObject you need, add one to the factory
- Always use `@TestSetup` for shared data across test methods
- Replicate the project's bypass/isolation pattern (custom settings, static flags) to
  avoid validation rule and trigger conflicts during test data creation
- Never use `seeAllData=true`
- Never hardcode record IDs — query them or create them

**Test methods — cover all paths:**
- Positive scenario (happy path with valid data)
- Negative scenario (invalid input, null, missing required fields)
- Bulk scenario (200+ records to verify governor limit safety)
- Permission scenario (`System.runAs()` with a restricted user)
- For callout classes: use `Test.setMock()` with the project's mock pattern
- For trigger handlers: test all registered events (before/after insert/update/delete)

**Regression detection — assert on behavior, not just coverage:**
- Use `System.assertEquals` / `System.assertNotEquals` with descriptive messages
- Assert on record field values after DML, not just record counts
- Assert that error messages are correct in negative scenarios
- Assert that bulk operations produce the same results as single-record operations
- When modifying existing logic, add a test for the specific change to prevent regression

### Step 4: Run tests after every change

```bash
# Run the specific test class for the changed file
sf apex run test --class-names MyServiceTest --result-format human --wait 10

# If test fails, read the output, fix the issue, and re-run
# Repeat until green
```

### Step 5: Check coverage

```bash
# Run with code coverage to check the changed class
sf apex run test --class-names MyServiceTest --code-coverage --result-format human --wait 10
```

If coverage is below 85%, identify uncovered lines and add targeted tests.

### Continuous loop during development

Every time you modify an Apex class or trigger:
1. Find its test class (or create one)
2. Scan the project's existing test patterns (factory, bypass, mock conventions)
3. Update/add tests to cover the change
4. Deploy the class AND its test together
5. Run the test and verify green
6. If red: read the failure, fix, re-run — do not leave broken tests

---

## 6. Best Practices Quick Reference

Full details in the reference files — read them when deeper guidance is needed.

| Area | Reference File | Key Rules |
|------|---------------|-----------|
| **Apex** | [apex-best-practices.md](apex-best-practices.md) | No SOQL/DML in loops; `with sharing` default; CRUD/FLS enforced; one trigger per object; 85%+ test coverage; Queueable > @future |
| **LWC** | [lwc-best-practices.md](lwc-best-practices.md) | SLDS base components first; `lwc:if` not `if:true`; `@wire` for reads; cleanup in `disconnectedCallback`; Jest in `__tests__/` |
| **Aura** | *(maintenance mode)* | Use LWC for new work; `$A.enqueueAction()`; no same-name JS/Apex methods; SLDS classes |
| **SOQL/SOSL** | [soql-sosl-best-practices.md](soql-sosl-best-practices.md) | Indexed fields in WHERE; no leading wildcards; `WITH USER_MODE`; relationship queries; bind variables |
| **Security** | [security-best-practices.md](security-best-practices.md) | `with sharing` everywhere; Named Credentials for callouts; no hardcoded IDs/credentials; CSP-compliant JS |
| **Integration** | *(in security ref)* | Named Credentials mandatory; no DML before callouts; `HttpCalloutMock` in tests; 100 callouts/txn |
| **Flows** | *(below)* | Before-save > After-save (~10x faster, zero DML); no SOQL/DML in loops; test with 200+ records |

### Governor limits

| Limit | Sync | Async |
|-------|------|-------|
| SOQL queries | 100 | 200 |
| DML statements | 150 | 150 |
| Records retrieved | 50,000 | 50,000 |
| CPU time (ms) | 10,000 | 60,000 |
| Heap size | 6 MB | 12 MB |
| Callouts | 100 | 100 |

### Metadata rules

- Read `sfdx-project.json` for `sourceApiVersion` before generating any `-meta.xml`
- Apex meta: `<apiVersion>NN.0</apiVersion>` + `<status>Active</status>`
- LWC meta: `<isExposed>true</isExposed>` + `<targets>` for placement
- Flows: Process Builder/Workflow Rules are EOL — use Flow Builder; before-save preferred

---

## 7. Pre-Deploy Automation

Before deploying, **run these checks automatically** (do not ask the user):

1. **Read `sfdx-project.json`** — get `sourceApiVersion` and `packageDirectories`
2. **Discover changed files** — use section 2 commands (filesystem timestamps or git diff)
3. **Static analysis** — read changed files and verify best practices (section 3)
4. **SOQL explain plans** — extract queries from changed files and verify selectivity (section 4)
5. **Test generation** — for each changed class, find or create tests following project patterns (section 5)
6. **Run Apex tests** — `sf apex run test --test-level RunLocalTests --result-format human --wait 10`
7. **Run LWC Jest tests** — `npm test` (if `__tests__/` directories exist)
8. **Preview deployment** — `sf project deploy preview --source-dir force-app`
9. **Validate with tests** — `sf project deploy validate --source-dir force-app --test-level RunLocalTests`
10. **Deploy** — `sf project deploy start --source-dir force-app` (only after green validation)
11. **Report** — `sf project deploy report` to confirm and share status with user

If any step fails, read the error, fix it if possible, and retry. Only escalate to the user if you cannot resolve the issue.

---

## Additional Resources

- [Apex best practices (full reference)](apex-best-practices.md)
- [LWC best practices (full reference)](lwc-best-practices.md)
- [SOQL/SOSL best practices (full reference)](soql-sosl-best-practices.md)
- [Security best practices (full reference)](security-best-practices.md)
