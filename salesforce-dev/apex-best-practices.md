# Apex Best Practices — Full Reference

## 1. Governor Limits

### Synchronous Limits

| Resource | Limit |
|----------|-------|
| SOQL queries | 100 |
| DML statements | 150 |
| Records retrieved (SOQL) | 50,000 |
| Records retrieved (Database.getQueryLocator) | 10,000 |
| Heap size | 6 MB |
| CPU time | 10,000 ms |
| Callouts | 100 |
| Callout timeout (single) | 120 seconds |
| Total callout timeout | 120 seconds |
| Future method invocations | 50 |
| Queueable jobs added | 50 |
| Email invocations | 10 |
| Push notifications | 10 |
| Event bus publish (Platform Events) | 150 |

### Asynchronous Limits (Batch, Queueable, Future)

| Resource | Limit |
|----------|-------|
| SOQL queries | 200 |
| CPU time | 60,000 ms |
| Heap size | 12 MB |
| Batch start() query rows via QueryLocator | 50,000,000 |
| Records per execute() chunk (default) | 200 |

### Static Limits (per 24-hour period)

| Resource | Limit |
|----------|-------|
| Batch Apex executions | 250,000 or (number of licenses × 200), whichever is greater |
| Async Apex (combined) | 250,000 or (number of licenses × 200) |
| Mass email | 5,000 |

---

## 2. Bulkification

### The #1 Rule: Never SOQL/DML Inside Loops

```apex
// BAD — 200 SOQL queries when processing 200 records
for (Contact c : Trigger.new) {
    Account a = [SELECT Id, Name FROM Account WHERE Id = :c.AccountId];
}

// GOOD — 1 SOQL query regardless of record count
Set<Id> accountIds = new Set<Id>();
for (Contact c : Trigger.new) {
    accountIds.add(c.AccountId);
}
Map<Id, Account> accountMap = new Map<Id, Account>(
    [SELECT Id, Name FROM Account WHERE Id IN :accountIds]
);
for (Contact c : Trigger.new) {
    Account a = accountMap.get(c.AccountId);
}
```

### Collection Patterns

- Use `Map<Id, SObject>` for O(1) lookups instead of nested loops
- Collect records in `List<SObject>` and perform a single DML statement
- Use `Set<Id>` to collect unique IDs for IN-clause queries
- Pre-build maps before iterating through trigger records

### DML Bulkification

```apex
// BAD
for (Account a : accountsToUpdate) {
    update a;
}

// GOOD
update accountsToUpdate;

// GOOD — partial success with error handling
Database.SaveResult[] results = Database.update(accountsToUpdate, false);
for (Database.SaveResult sr : results) {
    if (!sr.isSuccess()) {
        for (Database.Error err : sr.getErrors()) {
            System.debug(LoggingLevel.ERROR, err.getStatusCode() + ': ' + err.getMessage());
        }
    }
}
```

---

## 3. Trigger Design

### One Trigger Per Object

Multiple triggers on the same object execute in **unpredictable order**. Use a single trigger that routes to a handler class.

```apex
trigger AccountTrigger on Account (
    before insert, before update, before delete,
    after insert, after update, after delete, after undelete
) {
    AccountTriggerHandler.dispatch(Trigger.operationType, Trigger.new, Trigger.old, Trigger.newMap, Trigger.oldMap);
}
```

### Handler Pattern

```apex
public class AccountTriggerHandler {
    public static void dispatch(
        System.TriggerOperation operation,
        List<Account> newList,
        List<Account> oldList,
        Map<Id, Account> newMap,
        Map<Id, Account> oldMap
    ) {
        switch on operation {
            when BEFORE_INSERT  { beforeInsert(newList); }
            when BEFORE_UPDATE  { beforeUpdate(newList, oldMap); }
            when AFTER_INSERT   { afterInsert(newList); }
            when AFTER_UPDATE   { afterUpdate(newList, oldMap); }
            when BEFORE_DELETE  { beforeDelete(oldList); }
            when AFTER_DELETE   { afterDelete(oldList); }
            when AFTER_UNDELETE { afterUndelete(newList); }
        }
    }

    private static void beforeInsert(List<Account> newList) {
        // business logic
    }
    // ... other methods
}
```

### Recursion Guards

```apex
public class TriggerRecursionGuard {
    private static Set<Id> processedIds = new Set<Id>();

    public static Boolean hasBeenProcessed(Id recordId) {
        return processedIds.contains(recordId);
    }

    public static void markProcessed(Id recordId) {
        processedIds.add(recordId);
    }

    public static void markProcessed(Set<Id> recordIds) {
        processedIds.addAll(recordIds);
    }
}
```

---

## 4. Architecture Layers

### Service Layer

- Encapsulates business operations callable from triggers, UI, APIs, and batch jobs
- Methods accept and return bulk collections (never single records)
- Remains platform-agnostic — no UI or trigger context dependencies
- Coordinates Domain and Selector layers

### Domain Layer

- Contains object-specific business logic (validation, defaulting, calculation)
- Called from triggers and Service layer
- Each Domain class maps to one SObject

### Selector Layer

- Centralizes all SOQL queries for an SObject
- Prevents query inconsistencies and ensures field consistency
- Enforces security (CRUD/FLS) at the query level
- Returns strongly-typed lists or maps

```apex
public with sharing class AccountSelector {
    public static List<Account> selectByIds(Set<Id> accountIds) {
        return [
            SELECT Id, Name, Industry, AnnualRevenue
            FROM Account
            WHERE Id IN :accountIds
            WITH USER_MODE
        ];
    }

    public static Map<Id, Account> selectMapByIds(Set<Id> accountIds) {
        return new Map<Id, Account>(selectByIds(accountIds));
    }
}
```

---

## 5. Asynchronous Apex

### When to Use What

| Tool | Use Case | Max Records | Chaining | State | Monitoring |
|------|----------|-------------|----------|-------|------------|
| `@future` | Fire-and-forget, callouts | N/A (primitives only) | No | No | No |
| `Queueable` | Complex async with object params | N/A | Yes (chain via `System.enqueueJob`) | Yes | `AsyncApexJob` |
| `Batch` | Large data volumes (thousands–millions) | 50M via QueryLocator | Yes (via `finish()`) | Yes (`Database.Stateful`) | `AsyncApexJob` |
| `Schedulable` | Cron-based execution | N/A | N/A (launches Batch/Queueable) | No | `CronTrigger` |

### Queueable (preferred over @future)

```apex
public class MyQueueable implements Queueable, Database.AllowsCallouts {
    private List<Id> recordIds;

    public MyQueueable(List<Id> recordIds) {
        this.recordIds = recordIds;
    }

    public void execute(QueueableContext context) {
        List<Account> accounts = [SELECT Id, Name FROM Account WHERE Id IN :recordIds WITH USER_MODE];
        // process
    }
}

// Enqueue
System.enqueueJob(new MyQueueable(accountIds));
```

### Batch Apex

```apex
public class MyBatch implements Database.Batchable<SObject>, Database.Stateful {
    private Integer successCount = 0;

    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator('SELECT Id, Name FROM Account WHERE Industry = \'Technology\'');
    }

    public void execute(Database.BatchableContext bc, List<Account> scope) {
        for (Account a : scope) {
            // process
            successCount++;
        }
    }

    public void finish(Database.BatchableContext bc) {
        System.debug('Processed: ' + successCount);
    }
}

// Execute with custom batch size
Database.executeBatch(new MyBatch(), 50);
```

### Best Practices

- Never enqueue async jobs inside loops
- Pass collections of IDs, not single IDs
- Adjust batch size based on logic complexity: heavy = 10–50, light = 200
- Use `Database.Stateful` only when you need cross-chunk state (adds overhead)
- Implement `Finalizer` interface for retry logic on Queueable failures

---

## 6. Error Handling

### Try/Catch Patterns

```apex
try {
    update records;
} catch (DmlException e) {
    for (Integer i = 0; i < e.getNumDml(); i++) {
        System.debug(LoggingLevel.ERROR,
            'Field: ' + e.getDmlFieldNames(i) +
            ' Message: ' + e.getDmlMessage(i)
        );
    }
}
```

### Partial DML

```apex
Database.SaveResult[] results = Database.insert(records, false);
List<ErrorLog__c> errorLogs = new List<ErrorLog__c>();
for (Integer i = 0; i < results.size(); i++) {
    if (!results[i].isSuccess()) {
        for (Database.Error err : results[i].getErrors()) {
            errorLogs.add(new ErrorLog__c(
                Message__c = err.getMessage(),
                Context__c = 'MyClass.myMethod'
            ));
        }
    }
}
if (!errorLogs.isEmpty()) {
    insert errorLogs;
}
```

### Trigger Validation Errors

```apex
for (Account a : Trigger.new) {
    if (String.isBlank(a.Industry)) {
        a.addError('Industry is required.');
        // OR field-specific:
        a.Industry.addError('This field is required.');
    }
}
```

---

## 7. Testing

### Test Class Structure

```apex
@isTest
private class AccountServiceTest {

    @TestSetup
    static void setupData() {
        List<Account> accounts = TestDataFactory.createAccounts(200);
        insert accounts;
    }

    @isTest
    static void testPositiveScenario() {
        List<Account> accounts = [SELECT Id FROM Account];
        Test.startTest();
        AccountService.process(accounts);
        Test.stopTest();

        List<Account> results = [SELECT Id, Status__c FROM Account];
        System.assertEquals(200, results.size(), 'All accounts should be processed');
        for (Account a : results) {
            System.assertEquals('Processed', a.Status__c, 'Status should be Processed');
        }
    }

    @isTest
    static void testNegativeScenario() {
        Test.startTest();
        try {
            AccountService.process(null);
            System.assert(false, 'Should have thrown an exception');
        } catch (AccountService.ServiceException e) {
            System.assert(e.getMessage().contains('cannot be null'));
        }
        Test.stopTest();
    }

    @isTest
    static void testBulk() {
        // @TestSetup already created 200 records — verifies bulk behavior
        List<Account> accounts = [SELECT Id FROM Account];
        System.assertEquals(200, accounts.size());
        Test.startTest();
        AccountService.process(accounts);
        Test.stopTest();
    }

    @isTest
    static void testWithDifferentProfile() {
        User limitedUser = TestDataFactory.createUser('Standard User');
        System.runAs(limitedUser) {
            Test.startTest();
            // test behavior under restricted permissions
            Test.stopTest();
        }
    }
}
```

### Testing Rules

- Never use `seeAllData=true` — create all test data explicitly
- Never hardcode record IDs (they differ across orgs)
- Use `Test.startTest()` / `Test.stopTest()` to reset governor limits
- Use `System.runAs()` to test sharing and permission enforcement
- Test positive, negative, single-record, and bulk (200+) scenarios
- Mock callouts with `Test.setMock(HttpCalloutMock.class, new MyMock())`
- Use `@TestVisible` sparingly to expose private members for testing
- Target 85%+ coverage with meaningful assertions (not just line coverage)

### Test Data Factory

```apex
@isTest
public class TestDataFactory {
    public static List<Account> createAccounts(Integer count) {
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < count; i++) {
            accounts.add(new Account(Name = 'Test Account ' + i));
        }
        return accounts;
    }

    public static User createUser(String profileName) {
        Profile p = [SELECT Id FROM Profile WHERE Name = :profileName LIMIT 1];
        return new User(
            FirstName = 'Test', LastName = 'User',
            Email = 'test' + Crypto.getRandomInteger() + '@test.com',
            Username = 'test' + Crypto.getRandomInteger() + '@test.com.test',
            Alias = 'tuser', TimeZoneSidKey = 'America/Los_Angeles',
            LocaleSidKey = 'en_US', EmailEncodingKey = 'UTF-8',
            LanguageLocaleKey = 'en_US', ProfileId = p.Id
        );
    }
}
```

---

## 8. Code Quality Standards

- Use meaningful names: classes = `PascalCase`, methods/variables = `camelCase`, constants = `UPPER_SNAKE`
- Add ApexDoc (`/** */`) on all `public` and `global` methods
- Limit methods to ~40 lines — extract helper methods for readability
- Avoid nested conditionals deeper than 3 levels
- Never use `@SuppressWarnings` — fix the root issue
- Prefer `switch on` over long `if/else if` chains
- Use custom exceptions extending `Exception` for domain-specific errors
- Never catch generic `Exception` silently — always log or rethrow
