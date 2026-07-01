# Trigger Framework – Best Practices
---

## 1. Introduction

### What a Trigger Framework is
A Trigger Framework is an architectural pattern used to manage trigger execution logic in Salesforce. Instead of writing business logic directly inside the trigger body, a framework acts as a dispatcher, routing context variables (`Trigger.new`, `Trigger.old`, etc.) to dedicated Apex classes.

### Why Trigger Frameworks are important
As a Salesforce org grows, managing triggers becomes exponentially difficult. Without a framework, you risk hitting governor limits, creating recursive loops, and building tightly coupled code that is impossible to test or maintain. Enterprise trigger architectures enforce separation of concerns, making code scalable, predictable, and bulk-safe.

### Problems with Traditional Triggers
* **Logic in Triggers:** Hard to unit test and reuse.
* **Order of Execution:** Multiple triggers on the same object fire in an unpredictable order.
* **Performance:** Redundant queries and DML operations.

---

## 2. Why We Need Trigger Frameworks

Without a framework, developers often create "spaghetti code."

| Problem | Explanation | Example |
| :--- | :--- | :--- |
| **Multiple Triggers** | Creates unpredictable execution order. | `AccountTrigger1`, `AccountTrigger2` running simultaneously. |
| **SOQL in Loops** | Quickly exceeds the 100 SOQL limit. | Querying related Contacts inside a `for(Account a : Trigger.new)` loop. |
| **DML in Loops** | Exceeds the 150 DML limit. | Updating related records inside the trigger loop. |
| **Recursive Triggers** | Causes "Maximum trigger depth exceeded" error. | A workflow updates the record, firing the trigger again. |
| **Poor Testability** | Logic tied to the database context cannot be tested in isolation. | Mocking data becomes impossible. |

---

## 3. One Trigger Per Object Principle

The **One Trigger Per Object** principle is a foundational rule of Salesforce development. 

### Why only one trigger should exist
Salesforce does not guarantee the order of execution if multiple triggers exist on the same object. Having a single entry point ensures that developers explicitly control the flow of execution.

### Enterprise Standards Example
```java
// Automotive CRM: WarrantyClaimTrigger.trigger
trigger WarrantyClaimTrigger on Warranty_Claim__c (before insert, before update, before delete, after insert, after update, after delete, after undelete) {
    // Single entry point. All logic is delegated.
    TriggerDispatcher.run(new WarrantyClaimTriggerHandler());
}
```
**Explanation:**
This trigger listens for all 7 trigger contexts but contains zero business logic. It simply instantiates the handler and passes it to a central dispatcher.

---

## 4. Trigger Handler Pattern

The Trigger Handler pattern separates the trigger routing from the business logic.

```java
public class WarrantyClaimTriggerHandler extends TriggerHandler {
    
    // Override context methods provided by the TriggerHandler virtual class
    public override void beforeInsert() {
        WarrantyClaimService.validateClaims(Trigger.new);
    }
    
    public override void afterInsert() {
        WarrantyClaimService.createClaimLines(Trigger.new);
        SAPIntegrationService.syncClaimsToSAP(Trigger.new);
    }
}
```
**Line-by-line Explanation:**
* `extends TriggerHandler`: Inherits from a base virtual class that handles loop execution and recursion checks.
* `override void beforeInsert()`: Routes the `before insert` context.
* `WarrantyClaimService...`: Delegates the actual business logic to a Service layer.

---

## 5. Handler Class Structure

A standard handler class maps directly to the 7 DML contexts:

* `beforeInsert()`: Validation, defaulting fields.
* `beforeUpdate()`: Comparing `Trigger.new` and `Trigger.old`.
* `afterInsert()`: Creating related records (requires the new record's ID).
* `afterUpdate()`: Updating related records based on field changes.
* `beforeDelete()`: Preventing deletion based on rules.
* `afterDelete()`: Cleaning up related data.
* `afterUndelete()`: Restoring related data state.

---

## 6. Service Layer Pattern

The Service Layer encapsulates business logic. Handlers call Services, and Services execute the logic.

### Purpose
* **Business Logic Separation:** Keeps handlers lightweight.
* **Reusability:** Visualforce, LWC controllers, and Batch Apex can call the Service Layer without faking a trigger context.

### Example: Automotive CRM Service
```java
public class WarrantyClaimService {
    
    public static void validateClaims(List<Warranty_Claim__c> newClaims) {
        // 1. Bulkified validation logic
        for(Warranty_Claim__c claim : newClaims) {
            if(claim.Mileage__c > 100000 && claim.Type__c == 'Standard') {
                claim.addError('Standard warranty expired due to high mileage.');
            }
        }
    }
}
```
**Line-by-line Explanation:**
* `public static void`: Standard signature for stateless service methods.
* `List<Warranty_Claim__c> newClaims`: Accepts a collection, enforcing bulkification.
* `claim.addError`: Uses standard platform error handling on the specific record.

---

## 7. Domain Layer Pattern

The Domain layer is responsible for object-specific logic, defaults, and validation. In the **fflib** architecture, it represents the collection of records being processed.

### fflib Domain Pattern
```java
public class WarrantyClaims extends fflib_SObjectDomain {
    
    public WarrantyClaims(List<Warranty_Claim__c> sObjectList) {
        super(sObjectList);
    }

    public class Constructor implements fflib_SObjectDomain.IConstructable {
        public fflib_SObjectDomain construct(List<SObject> sObjectList) {
            return new WarrantyClaims(sObjectList);
        }
    }

    public override void onBeforeInsert() {
        // Domain-specific logic, defaults, and validation
    }
}
```

---

## 8. Separation of Concerns

* **Trigger Layer:** The entry point.
* **Handler Layer:** Routes the context (before/after).
* **Service Layer:** Executes complex business processes (cross-object).
* **Domain Layer:** Encapsulates logic for a specific SObject.
* **Repository / Selector Layer:** Handles all SOQL queries.
* **Unit of Work Layer:** Handles all DML operations.

---

## 9. Apex Enterprise Patterns (fflib)

The `fflib` framework is the gold standard for enterprise Salesforce architecture.

* `fflib_SObjectDomain`: Trigger handlers and object-specific logic.
* `fflib_UnitOfWork`: Orchestrates DML to ensure transactional integrity and prevent DML inside loops.
* `fflib_Service`: Business logic accessible via API, LWC, or Triggers.
* `fflib_Selector`: Centralizes SOQL queries.
* `fflib_Application`: The factory class for Dependency Injection.

---

## 10. Bulkification Best Practices

Always design for collections (max 200 records per trigger chunk).

```java
public static void linkDealers(List<Vehicle__c> vehicles) {
    // 1. Gather all Dealer Codes (Using Sets to ensure uniqueness)
    Set<String> dealerCodes = new Set<String>();
    for(Vehicle__c veh : vehicles) {
        if(veh.Dealer_Code__c != null) {
            dealerCodes.add(veh.Dealer_Code__c);
        }
    }
    
    // 2. Query once (Avoiding SOQL in loop)
    Map<String, Dealer__c> dealerMap = new Map<String, Dealer__c>();
    for(Dealer__c d : [SELECT Id, Dealer_Code__c FROM Dealer__c WHERE Dealer_Code__c IN :dealerCodes]) {
        dealerMap.put(d.Dealer_Code__c, d);
    }
    
    // 3. Process records (Using Maps for O(1) lookups)
    for(Vehicle__c veh : vehicles) {
        if(dealerMap.containsKey(veh.Dealer_Code__c)) {
            veh.Dealer__c = dealerMap.get(veh.Dealer_Code__c).Id;
        }
    }
}
```

---

## 11. Recursion Prevention

Recursion happens when a trigger fires, updates a record, and causes itself to fire again.

### Static Boolean Pattern (Simple but flawed for bulk)
`public static boolean hasRun = false;` (Fails if trigger processes > 200 records).

### Static Set Pattern (Enterprise Standard)
```java
public class TriggerRecursionDefense {
    public static Set<Id> processedIds = new Set<Id>();
}

// In Handler:
List<Account> toProcess = new List<Account>();
for(Account a : Trigger.new) {
    if(!TriggerRecursionDefense.processedIds.contains(a.Id)) {
        toProcess.add(a);
        TriggerRecursionDefense.processedIds.add(a.Id);
    }
}
```

---

## 12. Error Handling Strategies

Enterprise frameworks should not just fail; they should log errors gracefully.

```java
try {
    // DML operation
    insert newInvoices;
} catch (DmlException e) {
    // Log to custom object using an Error Logging Framework
    ErrorLogger.logException(e, 'InvoiceService', 'createInvoices');
    
    // Surface error to UI safely
    for(Integer i = 0; i < e.getNumDml(); i++) {
        Trigger.new[e.getDmlIndex(i)].addError(e.getDmlMessage(i));
    }
}
```

---

## 13. Governor Limit Considerations

| Limit | Description | Enterprise Mitigation |
| :--- | :--- | :--- |
| **100 SOQL** | Max queries per transaction | Use Selector classes; pass sets/lists to avoid queries in loops. |
| **150 DML** | Max DML statements per transaction | Use Unit Of Work (`fflib`) to consolidate DML. |
| **10s CPU** | Max CPU time limits | Use maps instead of nested loops (`O(1)` vs `O(n^2)`). |
| **6MB Heap** | Max memory usage | Use SOQL for loops (`for(Account a : [SELECT...])`) for large datasets. |

---

## 14. Dependency Injection in Apex

Dependency Injection (DI) allows us to inject mock implementations during testing.

```java
// Service uses an interface
public interface IDiscountCalculator {
    Decimal calculate(Decimal amount);
}

// Handler injects the dependency
public void beforeInsert() {
    IDiscountCalculator calc = (IDiscountCalculator) Application.DI.newInstance(IDiscountCalculator.class);
    InvoiceService.applyDiscounts(Trigger.new, calc);
}
```

---

## 15. Unit Testing Trigger Frameworks

Test the Service layer directly, bypassing the trigger to ensure modularity.

```java
@IsTest
private class WarrantyClaimServiceTest {
    @IsTest
    static void testHighMileageValidation() {
        // 1. Setup Mock Data (No DML needed if designed well)
        Warranty_Claim__c claim = new Warranty_Claim__c(Mileage__c = 150000, Type__c = 'Standard');
        
        // 2. Execute directly
        Test.startTest();
        WarrantyClaimService.validateClaims(new List<Warranty_Claim__c>{claim});
        Test.stopTest();
        
        // 3. Assert
        System.assertEquals(true, claim.hasErrors(), 'High mileage should trigger an error');
    }
}
```

---

## 16. Test Data Factory Pattern

Centralize test data creation to ensure consistent setup and reduce boilerplate code.

```java
@IsTest
public class TestDataFactory {
    public static Vehicle__c createVehicle(Boolean doInsert) {
        Vehicle__c v = new Vehicle__c(VIN__c = '12345ABCDE', Make__c = 'Toyota');
        if(doInsert) insert v;
        return v;
    }
}
```

---

## 17. Mocking Strategies

Using the Apex Stub API or `fflib_ApexMocks`, you can mock Selectors and Services to test logic without querying the database.

```java
// Using fflib_ApexMocks
fflib_ApexMocks mocks = new fflib_ApexMocks();
ISapIntegrationService mockSap = (ISapIntegrationService) mocks.mock(SAPIntegrationService.class);

// Inject mock
Application.Service.setMock(ISapIntegrationService.class, mockSap);
```

---

## 18. Enterprise Trigger Architecture

* **Database Layer:** SObjects (e.g., Warranty_Claim__c, Vehicle__c).
* **Trigger Layer:** The raw `trigger` file. Purely an entry point.
* **Handler Layer:** Translates the execution context (`before insert`) to actions.
* **Selector Layer:** Handles all database reads (SOQL). 
* **Service Layer:** Houses the proprietary business logic.
* **UnitOfWork Layer:** Handles all database writes (DML).
* **External Systems:** Connected via HTTP Callouts executed from asynchronous contexts (Future/Queueable) triggered by the Service Layer.

---

## 19. Real Project Scenarios (Automotive CRM)

### Warranty Claim Trigger Framework
* **Requirement:** When a `Warranty_Claim__c` is approved, create a `Work_Order__c` and notify the `Dealer__c`.
* **Implementation:**
    * `WarrantyClaimTrigger` routes to `WarrantyClaimHandler.afterUpdate()`.
    * `WarrantyClaimService.processApprovedClaims()` evaluates the data.
    * Uses `DealerSelector` to fetch Dealer Emails efficiently.
    * Uses `fflib_UnitOfWork` to queue the insertion of Work Orders and commit them at the end of the transaction.

---

## 20. Performance Optimization

* **Bulk-safe design:** Always assume `Trigger.new` contains 200 records.
* **Map-based lookups:** Use `Map<Id, SObject>` instead of nested `for` loops to match related records.
* **CPU optimization:** Avoid calling heavy utility methods repeatedly inside loops. Cache results when possible.
* **Lazy loading strategies:** Only instantiate classes and query data if the trigger context genuinely requires it (e.g., only query Dealer data if a Vehicle's Dealer field actually changed).

---

## 21. Common Mistakes

| Mistake | Solution |
| :--- | :--- |
| **Business logic inside triggers** | Move all logic to an Apex Service class. |
| **No recursion prevention** | Implement a Static Set to track processed IDs. |
| **Nested DML/SOQL** | Extract to collections, query once, process via Maps. |
| **Hardcoded IDs** | Query Record Types by `DeveloperName` or use Custom Metadata. |

---

## 22. Best Practices Checklist

* **One Trigger Per Object:** No exceptions.
* **Handler Classes:** Use a virtual/abstract base class to standardize routing.
* **Service Layer:** Keep handlers strictly for routing; put logic here.
* **Bulkification:** Utilize Maps, Sets, and Lists. Absolutely no queries inside loops.
* **Recursion Prevention:** Use ID sets rather than static booleans.
* **Unit Tests:** Test the service layer directly, not just by inserting records.
* **Error Handling:** Use `addError()` gracefully to surface issues to users.
* **Dependency Injection:** Code to interfaces to allow for mocking.
* **No Hardcoded IDs:** Abstract away specific configuration data.
* **Governor Limit Monitoring:** Watch CPU time and Heap size during bulk loads.

---

## 23. Debugging Trigger Frameworks

* **Developer Console:** Check the *Execution Overview* > *Save Order* to see exactly how limits are consumed across complex save operations.
* **Debug Logs:** Set `Apex Code` to `DEBUG` and `System` to `NONE` to reduce noise. Filter for your Handler/Service classes.
* **VS Code Replay Debugger:** Use this to step backward through the trigger context to find exactly where variables mutate.
* **Transaction Analysis:** Ensure your UnitOfWork `commitWork()` is only called once per transaction.

---

## 24. Interview Questions & Answers

### Beginner Questions
**Q: Why shouldn't you put logic inside a trigger?**
A: It makes the code difficult to reuse (e.g., from a Batch class), increases the likelihood of breaking governor limits, and is nearly impossible to unit test cleanly without complex database setup.

### Intermediate Questions
**Q: How do you prevent a trigger from firing twice due to workflow rules?**
A: Implement a static variable check (preferably a `Set<Id>` of processed records rather than a boolean) inside the Trigger Handler to track which records have already passed through the logic.

### Advanced Questions
**Q: Explain the difference between `Trigger.new` and `Trigger.oldMap` in an `after update` context.**
A: `Trigger.new` contains the new versions of the SObject records, while `Trigger.oldMap` contains a map of IDs to the old versions of the SObject records before the update. This is heavily used to check if a specific field's value has changed by comparing `Trigger.new[i].Field__c` against `Trigger.oldMap.get(Trigger.new[i].Id).Field__c`.

### Architect-Level Questions
**Q: How does the fflib Unit of Work pattern improve trigger performance?**
A: It registers records for insertion, updating, or deletion in memory across multiple service layer calls, and then commits them to the database in a single, optimized set of DML statements at the very end of the transaction. This prevents DML inside loops, minimizes database locking issues, and ensures transactional integrity.

---

## 25. Revision Summary

* **Trigger Frameworks** centralize execution flow and enforce clean architecture.
* **Handler Pattern** dictates *when* things happen (routing).
* **Service Layer** dictates *what* happens (business logic).
* **fflib** is the enterprise standard for implementing separation of concerns (Selector, Domain, Service, UoW).
* **Bulkification and Recursion Prevention** are mandatory for scalable orgs to stay within Governor Limits.
* **Dependency Injection & Mocking** are critical for writing fast, isolated, and reliable unit tests.