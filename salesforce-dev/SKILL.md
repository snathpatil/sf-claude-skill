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

## 2. Automated Code Review

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

## 3. SOQL Performance Verification

**Do not tell the user to open Developer Console.** Instead, check query performance
by running the REST API explain endpoint yourself:

```bash
# Get query explain plan via sf CLI
sf api request rest "/services/data/v64.0/query/?explain=SELECT+Id,Name+FROM+Account+WHERE+Industry='Technology'" --method GET
```

Parse the JSON response and check:
- `leadingOperationType` — should be `Index`, not `TableScan`
- `cost` — should be `< 1.0` for selective queries
- `cardinality` — estimated records the operation touches

If the query is non-selective (cost >= 1.0 or TableScan), suggest index-friendly
WHERE clause changes. See [soql-sosl-best-practices.md](soql-sosl-best-practices.md) for details.

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

## 5. LWC Best Practices (summary)

For the full reference, see [lwc-best-practices.md](lwc-best-practices.md).

### Architecture

- Prefer LWC over Aura for all new development
- Use `@wire` for reactive reads; imperative Apex only for mutations
- Small, composable, single-responsibility components
- Use Lightning Data Service (`lightning-record-form`, `getRecord`) before custom Apex

### SLDS compliance (mandatory)

1. **Lightning base components first** — `lightning-button`, `lightning-datatable`, `lightning-card`, etc.
2. **SLDS blueprint classes** — `slds-grid`, `slds-col`, `slds-card`, `slds-text-heading_medium`
3. **SLDS styling hooks** (CSS custom properties) — never override SLDS classes directly
4. Never style based on internal rendered markup of base components

### HTML template rules

- Use `lwc:if` / `lwc:elseif` / `lwc:else` — **not** deprecated `if:true` / `if:false`
- `for:each` requires a `key` attribute with unique value (e.g., `Id`)
- Avoid complex expressions in templates — use JS getters

### JavaScript rules

- `async/await` for imperative Apex
- Debounce search/filter inputs
- Clean up listeners/intervals in `disconnectedCallback()`
- `CustomEvent` for child-to-parent; Lightning Message Service for cross-component

### Testing (Jest) — run these yourself

```bash
# Setup (first time only)
sf force lightning lwc test setup

# Run all LWC tests
npm test

# Run with coverage
npm test -- --coverage
```

---

## 6. Aura Best Practices (summary)

> Aura is in maintenance mode. Use LWC for all new development.

- `$A.enqueueAction()` for server calls — never call Apex directly
- Avoid naming JS controller methods the same as `@AuraEnabled` Apex methods (causes silent infinite loops)
- `$A.getCallback()` inside async callbacks (setTimeout, Promises)
- Component Events for parent-child; Application Events sparingly
- Apply SLDS classes for consistent UI
- Migrate to LWC when significantly modifying Aura components

---

## 7. SOQL / SOSL Best Practices (summary)

For the full reference, see [soql-sosl-best-practices.md](soql-sosl-best-practices.md).

- Use selective filters on **indexed fields** — `Id`, `Name`, `CreatedDate`, lookups, `ExternalId`
- Always include `WHERE` clause and/or `LIMIT`
- Never use leading wildcards (`LIKE '%value%'`)
- Use `WITH USER_MODE` or `WITH SECURITY_ENFORCED` for FLS enforcement
- Explicitly list fields — never `SELECT *` equivalent
- Use relationship queries (parent-to-child, child-to-parent) to reduce query count
- Use `IN` clauses with collections — never SOQL inside loops
- Use `FOR UPDATE` only when row-locking is truly needed
- Verify selectivity by running the REST explain endpoint (see section 3)

---

## 8. Security Checklist

For the full reference, see [security-best-practices.md](security-best-practices.md).

- [ ] `with sharing` or `inherited sharing` on every class (unless documented exception)
- [ ] CRUD/FLS enforced on all data operations (`WITH USER_MODE`, `Security.stripInaccessible()`)
- [ ] No hardcoded credentials, IDs, or org-specific values
- [ ] Bind variables in SOQL — never string concatenation (SOQL injection)
- [ ] Named Credentials for all external callouts — never hardcode endpoints/secrets
- [ ] CSP-compliant third-party libraries (no `eval`, no `Function()` constructors)
- [ ] Sensitive data not exposed in debug logs or error messages
- [ ] User input validated on both client (LWC) and server (Apex)

---

## 9. Integration & Callout Best Practices

- **Always use Named Credentials** — never hardcode endpoints or credentials in code
- Use External Credentials for auth protocol reuse across callouts
- Limits: 100 callouts/transaction, 10s sync timeout, 120s async timeout, 6MB heap
- **No DML before callouts** — DML locks rows and uncommitted work causes `CalloutException`
- Use `@future(callout=true)` or `Queueable` for non-blocking callout patterns
- Use `Continuation` class for long-running callouts from LWC/Visualforce
- Implement retry logic with exponential backoff for transient failures
- Use `HttpCalloutMock` / `MultiStaticResourceCalloutMock` in tests

---

## 10. Flow Best Practices

- Process Builder and Workflow Rules are EOL (Dec 2025) — use Flow Builder exclusively
- **Before-save > After-save** — before-save runs ~10x faster and uses zero DML
- Never place Get/Update/Create elements inside loops — bulkify with collection variables
- Use one record-triggered Flow per object per timing (before/after)
- Use Subflows for reusable logic and to stay within per-element limits
- Screen elements and Wait elements break transaction boundaries (new governor limits)
- Use the Transform element instead of loop + assignment for object mapping
- Test Flows with bulk data (200+ records) to verify bulkification

---

## 11. Project Structure & Metadata

```
force-app/
├── main/default/
│   ├── classes/          # Apex classes + *.cls-meta.xml
│   ├── triggers/         # Apex triggers + *.trigger-meta.xml
│   ├── lwc/              # LWC bundles (js, html, css, *.js-meta.xml)
│   ├── aura/             # Aura bundles
│   ├── objects/          # Custom objects & fields
│   ├── flows/            # Flows
│   ├── permissionsets/   # Permission sets
│   ├── layouts/          # Page layouts
│   ├── pages/            # Visualforce pages
│   ├── customMetadata/   # Custom Metadata Type records
│   └── staticresources/  # Static resources
```

- Every component requires a `-meta.xml` with `apiVersion` matching `sfdx-project.json`
- Read `sfdx-project.json` to determine the correct `sourceApiVersion` before generating meta.xml
- Apex meta.xml: `<apiVersion>NN.0</apiVersion>` + `<status>Active</status>`
- LWC meta.xml must have `<isExposed>true</isExposed>` and `<targets>` for placement

---

## 12. Pre-Deploy Automation

Before deploying, **run these checks automatically** (do not ask the user):

1. **Read `sfdx-project.json`** — get `sourceApiVersion` and `packageDirectories`
2. **Static analysis** — read changed files and verify best practices (section 2)
3. **Run Apex tests** — `sf apex run test --test-level RunLocalTests --result-format human --wait 10`
4. **Run LWC Jest tests** — `npm test` (if `__tests__/` directories exist)
5. **Preview deployment** — `sf project deploy preview --source-dir force-app`
6. **Validate with tests** — `sf project deploy validate --source-dir force-app --test-level RunLocalTests`
7. **Deploy** — `sf project deploy start --source-dir force-app` (only after green validation)
8. **Report** — `sf project deploy report` to confirm and share status with user

If any step fails, read the error, fix it if possible, and retry. Only escalate to the user if you cannot resolve the issue.

---

## Additional Resources

- [Apex best practices (full reference)](apex-best-practices.md)
- [LWC best practices (full reference)](lwc-best-practices.md)
- [SOQL/SOSL best practices (full reference)](soql-sosl-best-practices.md)
- [Security best practices (full reference)](security-best-practices.md)
