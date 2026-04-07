# SOQL & SOSL Best Practices — Full Reference

## 1. Query Selectivity

A query is **selective** when its WHERE clause uses indexed fields to filter down to a small percentage of total records. Selective queries use an indexed execution plan instead of a full table scan.

### Thresholds

| Scenario | Must return less than |
|----------|----------------------|
| Single object query | 30% of total records |
| Query with relationship (parent/child) | 15% of total records |

### Automatically Indexed Fields

- `Id`, `OwnerId`, `CreatedDate`, `SystemModstamp`, `RecordTypeId`
- Lookup and Master-Detail relationship fields
- Fields marked `ExternalId` or `Unique`
- `Name` on most standard objects

### Index-Friendly Operators

- `=`, `IN`, `<`, `>`, `<=`, `>=`, `LIKE 'prefix%'` (trailing wildcard only)
- Combining multiple indexed fields improves selectivity

### Index-Hostile Patterns (avoid)

- `!=`, `NOT IN`, `NOT LIKE` — cannot use indexes
- `LIKE '%value%'` — leading wildcard prevents index usage
- Functions on fields in WHERE: `CALENDAR_YEAR(CreatedDate)`, `DAY_IN_MONTH(CloseDate)`
- Null checks on non-indexed fields
- Negative operators combined with non-indexed fields

### Verifying Selectivity (agent must run this — never ask user to open Developer Console)

Use the REST API explain endpoint via sf CLI to check query plans automatically:

```bash
# URL-encode the SOQL query and call the explain endpoint
sf api request rest "/services/data/v64.0/query/?explain=SELECT+Id,Name+FROM+Account+WHERE+Industry='Technology'" --method GET
```

Parse the JSON response and check:
- `leadingOperationType` — must be `Index` (not `TableScan`)
- `cost` — must be `< 1.0` for selective queries
- `cardinality` — estimated records touched (lower is better)
- `sObjectCardinality` — total records in the object (provides context)

If cost >= 1.0 or operation is `TableScan`, refactor the WHERE clause to use indexed fields.

---

## 2. Query Optimization

### Select Only Required Fields

```apex
// BAD — fetches all fields (not even valid SOQL, but conceptually)
// SELECT * FROM Account

// GOOD — explicit field list
SELECT Id, Name, Industry, AnnualRevenue
FROM Account
WHERE Industry = 'Technology'
```

### Always Use WHERE and LIMIT

```apex
// BAD — unbounded query
SELECT Id, Name FROM Account

// GOOD — bounded
SELECT Id, Name FROM Account WHERE Industry = 'Technology' LIMIT 100

// GOOD — for batch processing, use QueryLocator (bypasses 50K limit)
Database.getQueryLocator('SELECT Id, Name FROM Account WHERE Industry = :industry')
```

### Use Collections with IN Clause

```apex
// BAD — SOQL in loop (up to 200 queries)
for (Contact c : contacts) {
    Account a = [SELECT Id FROM Account WHERE Id = :c.AccountId];
}

// GOOD — single query with IN clause
Set<Id> accountIds = new Set<Id>();
for (Contact c : contacts) {
    accountIds.add(c.AccountId);
}
List<Account> accounts = [SELECT Id, Name FROM Account WHERE Id IN :accountIds];
```

### Relationship Queries

#### Child-to-Parent (dot notation)

```apex
SELECT Id, Name, Account.Name, Account.Industry
FROM Contact
WHERE Account.Industry = 'Technology'
```

#### Parent-to-Child (subquery)

```apex
SELECT Id, Name, (SELECT Id, LastName FROM Contacts WHERE IsActive__c = true)
FROM Account
WHERE Industry = 'Technology'
```

Relationship queries reduce total SOQL query count. Use them to fetch related data in a single query.

### Aggregate Queries

```apex
// Count
SELECT COUNT() FROM Account WHERE Industry = 'Technology'

// Grouped aggregates
SELECT Industry, COUNT(Id) total, SUM(AnnualRevenue) revenue
FROM Account
GROUP BY Industry
HAVING COUNT(Id) > 10
ORDER BY COUNT(Id) DESC

// Aggregate limit: 50,000 records processed per query
```

---

## 3. Security in Queries

### WITH USER_MODE (Spring '23+ — preferred)

Enforces CRUD, FLS, **and** sharing rules. Also checks FLS on WHERE clause fields.

```apex
// Static SOQL
List<Account> accounts = [
    SELECT Id, Name, AnnualRevenue
    FROM Account
    WHERE Industry = 'Technology'
    WITH USER_MODE
];

// Dynamic SOQL
List<Account> accounts = Database.query(
    'SELECT Id, Name FROM Account WHERE Industry = :industry',
    AccessLevel.USER_MODE
);
```

### WITH SECURITY_ENFORCED (Spring '20+)

Enforces FLS on SELECT fields and object-level CRUD. Does **not** check WHERE clause fields.

```apex
SELECT Id, Name, AnnualRevenue
FROM Account
WHERE Industry = 'Technology'
WITH SECURITY_ENFORCED
```

Throws `System.QueryException` if the user lacks field access.

### Security.stripInaccessible

Use after querying in system mode when you need partial results instead of exceptions.

```apex
List<Account> accounts = [SELECT Id, Name, AnnualRevenue FROM Account];
SObjectAccessDecision decision = Security.stripInaccessible(AccessType.READABLE, accounts);
List<Account> sanitized = decision.getRecords();
// AnnualRevenue is stripped if user lacks FLS
```

---

## 4. SOSL (Salesforce Object Search Language)

Use SOSL for text-based searches across multiple objects.

```apex
List<List<SObject>> results = [
    FIND 'Acme*' IN ALL FIELDS
    RETURNING Account(Id, Name), Contact(Id, Name, Email)
    WITH NETWORK = ALL
    LIMIT 20
];
List<Account> accounts = (List<Account>) results[0];
List<Contact> contacts = (List<Contact>) results[1];
```

### SOSL Limits

| Resource | Limit |
|----------|-------|
| SOSL queries per transaction | 20 |
| Records returned per object | 2,000 (default) |
| Total records returned | 2,000 (across all objects) |

### SOQL vs SOSL Decision

| Use Case | Choose |
|----------|--------|
| Exact ID / field match | SOQL |
| Known object and fields | SOQL |
| Text search across multiple objects | SOSL |
| Fuzzy / partial text match | SOSL |
| Complex relationship queries | SOQL |
| Aggregate calculations | SOQL |

---

## 5. Dynamic SOQL

### When to Use

- Field or object names determined at runtime
- User-configurable queries (search, filters)
- Generic utility methods

### Injection Prevention

```apex
// BAD — SOQL injection vulnerable
String query = 'SELECT Id FROM Account WHERE Name = \'' + userInput + '\'';

// GOOD — bind variables (preferred)
String query = 'SELECT Id FROM Account WHERE Name = :userInput';
List<Account> results = Database.query(query);

// GOOD — String.escapeSingleQuotes (when bind variables aren't possible)
String safe = String.escapeSingleQuotes(userInput);
String query = 'SELECT Id FROM Account WHERE Name = \'' + safe + '\'';
```

### User Mode for Dynamic SOQL

```apex
List<Account> results = Database.query(query, AccessLevel.USER_MODE);
```

---

## 6. Performance Checklist

- [ ] Only select fields that are needed
- [ ] WHERE clause uses indexed fields
- [ ] No leading wildcards in LIKE clauses
- [ ] No SOQL inside loops
- [ ] Use relationship queries to reduce query count
- [ ] Use LIMIT when full result set is not needed
- [ ] Verify query plan via `sf api request rest` explain endpoint shows `Cost < 1.0`
- [ ] Use `WITH USER_MODE` or `WITH SECURITY_ENFORCED` for FLS
- [ ] Use bind variables for dynamic SOQL (prevent injection)
- [ ] Use `FOR UPDATE` only when row-locking is needed
- [ ] Large datasets use Batch Apex with `Database.getQueryLocator`
