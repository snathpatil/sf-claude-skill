# Salesforce Security Best Practices — Full Reference

## 1. Sharing Model (Record-Level Security)

### Keywords

| Keyword | Behavior |
|---------|----------|
| `with sharing` | Enforces current user's sharing rules (OWD, role hierarchy, sharing rules, teams) |
| `without sharing` | System mode — bypasses all sharing; use sparingly |
| `inherited sharing` | Inherits caller's sharing context; ideal for reusable utility/service classes |

### Rules

- **Default to `with sharing`** on every class unless there is a documented, specific reason
- Use `without sharing` only for admin-level operations (e.g., logging, audit, platform events)
- Use `inherited sharing` on service/utility classes so they respect the caller's context
- If no keyword is specified, Apex runs in system mode — this is a security risk
- Triggers always run in system mode; delegate to `with sharing` handler classes
- `with sharing` only enforces **record-level** access — it does NOT enforce CRUD/FLS

### Example

```apex
public with sharing class AccountService {
    // Respects current user's record access
    public List<Account> getAccounts() {
        return [SELECT Id, Name FROM Account WITH USER_MODE];
    }
}

public inherited sharing class UtilityService {
    // Inherits sharing from whatever calls it
    public List<SObject> queryRecords(String soql) {
        return Database.query(soql, AccessLevel.USER_MODE);
    }
}

public without sharing class AuditLogger {
    // Documented exception: audit logs must bypass sharing
    public static void log(String message, String context) {
        insert new Audit_Log__c(Message__c = message, Context__c = context);
    }
}
```

---

## 2. CRUD / FLS (Object & Field-Level Security)

### Enforcement Methods (in order of preference)

#### 1. WITH USER_MODE (Spring '23+ — best option)

- Enforces CRUD, FLS, **and** sharing in a single clause
- Works on both static and dynamic SOQL
- Checks FLS on WHERE clause fields (unlike `WITH SECURITY_ENFORCED`)

```apex
// Static
List<Account> accts = [SELECT Id, Name FROM Account WITH USER_MODE];

// Dynamic
List<Account> accts = Database.query(soql, AccessLevel.USER_MODE);

// DML
Database.insert(records, AccessLevel.USER_MODE);
Database.update(records, AccessLevel.USER_MODE);
```

#### 2. Security.stripInaccessible (Spring '20+)

- Strips inaccessible fields instead of throwing exceptions
- Good when you want partial results (graceful degradation)

```apex
// Read — strips fields user can't see
List<Account> accounts = [SELECT Id, Name, AnnualRevenue FROM Account];
SObjectAccessDecision decision = Security.stripInaccessible(AccessType.READABLE, accounts);
List<Account> safe = decision.getRecords();

// Create — strips fields user can't create
SObjectAccessDecision decision = Security.stripInaccessible(AccessType.CREATABLE, newRecords);
insert decision.getRecords();

// Check which fields were removed
Set<String> removedFields = decision.getRemovedFields().get('Account');
```

#### 3. WITH SECURITY_ENFORCED (Spring '20+)

- Enforces FLS on SELECT fields and CRUD on the object
- Does NOT check FLS on WHERE clause fields
- Throws `System.QueryException` on violation

```apex
SELECT Id, Name, AnnualRevenue
FROM Account
WHERE Industry = 'Technology'
WITH SECURITY_ENFORCED
```

#### 4. Schema Describe (legacy — avoid for new code)

```apex
if (Schema.sObjectType.Account.isAccessible() &&
    Schema.sObjectType.Account.fields.Name.isAccessible()) {
    // safe to query
}
if (Schema.sObjectType.Account.isCreateable()) {
    // safe to insert
}
```

---

## 3. SOQL Injection Prevention

### The Risk

Dynamic SOQL with unsanitized user input allows attackers to modify queries.

### Prevention Methods

```apex
// BEST — bind variables
String industry = userInput;
List<Account> accts = Database.query(
    'SELECT Id, Name FROM Account WHERE Industry = :industry'
);

// GOOD — escapeSingleQuotes (when bind variables aren't possible)
String safe = String.escapeSingleQuotes(userInput);
String query = 'SELECT Id FROM Account WHERE Name LIKE \'%' + safe + '%\'';

// ALSO GOOD — type casting for non-string inputs
Integer recordLimit = Integer.valueOf(userInputLimit);
```

### Never Do

```apex
// VULNERABLE
String query = 'SELECT Id FROM Account WHERE Name = \'' + request.getParameter('name') + '\'';
```

---

## 4. Callout Security

### Named Credentials (mandatory)

- **Always use Named Credentials** for external callouts
- Never hardcode endpoints, API keys, tokens, or passwords in Apex
- Named Credentials handle authentication (OAuth, Basic, JWT) automatically
- Deployable across environments without code changes

```apex
// GOOD — Named Credential
HttpRequest req = new HttpRequest();
req.setEndpoint('callout:My_Named_Credential/api/v1/data');
req.setMethod('GET');

// BAD — hardcoded
req.setEndpoint('https://api.example.com/data');
req.setHeader('Authorization', 'Bearer ' + myHardcodedToken);
```

### External Credentials

- Separate authentication protocol from endpoint URL
- Reuse auth config across multiple Named Credentials
- Support OAuth 2.0, JWT Bearer, AWS Signature V4, and custom auth

### Certificate Management

- Use Salesforce certificates for mutual TLS (mTLS)
- Never store private keys in code or custom settings
- Rotate certificates before expiration

---

## 5. XSS Prevention

### In Apex / Visualforce

- Use `HTMLENCODE()`, `JSENCODE()`, `URLENCODE()` for output encoding
- Never render unescaped user input in Visualforce pages
- Use `<apex:outputText escape="true">` (default is true)

### In LWC

- LWC's template engine auto-escapes expressions (`{expression}`)
- Use `lightning-formatted-rich-text` for rendering sanitized HTML
- Never use `innerHTML` directly — use `lwc:dom="manual"` sparingly
- Do not use `eval()` or `Function()` constructors

---

## 6. CSP (Content Security Policy)

- JavaScript must be served from the org (static resources or ES modules)
- External resources require Trusted URL configuration (HTTPS only)
- Inline `<script>` tags and `javascript:` URIs are blocked
- `eval()` and `new Function()` are blocked by CSP

### Third-Party Libraries

- Must be uploaded as Static Resources
- Must support strict mode (`"use strict"`)
- Must not manipulate DOM across namespace boundaries
- Verify LWS/Locker compatibility before use

---

## 7. Sensitive Data Handling

- Never log sensitive data (PII, credentials, tokens) in `System.debug`
- Use Platform Encryption (Shield) for sensitive fields at rest
- Use encrypted custom settings or Named Credentials for secrets
- Remove debug logs containing sensitive data from production
- Use Event Monitoring to track data access patterns
- Mask or truncate sensitive data in error messages shown to users

---

## 8. Permission & Access Patterns

### Principle of Least Privilege

- Use Permission Sets over Profiles for granular access
- Use Permission Set Groups to bundle related permissions
- Restrict `Modify All Data` and `View All Data` to true admins
- Use Custom Permissions for feature-gating in Apex/Flows

### Testing Permissions

```apex
@isTest
static void testRestrictedUser() {
    User restrictedUser = TestDataFactory.createUser('Minimum Access - Salesforce');
    // Assign only required permission sets
    insert new PermissionSetAssignment(
        AssigneeId = restrictedUser.Id,
        PermissionSetId = myPermSetId
    );

    System.runAs(restrictedUser) {
        Test.startTest();
        // verify behavior under restricted access
        Test.stopTest();
    }
}
```

---

## 9. Security Review Checklist

- [ ] Every class has explicit sharing keyword (`with sharing`, `inherited sharing`, or documented `without sharing`)
- [ ] All SOQL enforces FLS (`WITH USER_MODE` or `WITH SECURITY_ENFORCED`)
- [ ] All DML enforces CRUD (`AccessLevel.USER_MODE` or `Security.stripInaccessible`)
- [ ] Dynamic SOQL uses bind variables or `String.escapeSingleQuotes`
- [ ] No hardcoded credentials, IDs, or org-specific URLs
- [ ] External callouts use Named Credentials
- [ ] Third-party JS libraries loaded from Static Resources (CSP compliant)
- [ ] No `eval()` or `Function()` in JavaScript
- [ ] User input validated on both client and server
- [ ] Sensitive data not exposed in debug logs or error messages
- [ ] `System.runAs()` tests cover restricted user scenarios
- [ ] Custom Permissions used for feature-gating (not profile checks)
