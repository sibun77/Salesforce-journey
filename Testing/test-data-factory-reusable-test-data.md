# Test Data Factory – Reusable Test Data

## 1. Introduction

In enterprise Salesforce development, testing is a critical component of the application lifecycle. A **Test Data Factory** is a centralized utility class, annotated with `@isTest`, dedicated exclusively to generating reusable test data for Apex test classes.

Salesforce developers use Test Data Factories to abstract the logic required to instantiate and insert both standard and custom objects. When developers write tests manually without a factory, they often duplicate data creation logic across dozens of test classes. This leads to bloated test code, hard-to-maintain hardcoded values, and fragile tests that break whenever validation rules or required fields change.

By centralizing test data creation, a Test Data Factory provides a single source of truth. If a new required field is added to the `Account` or `Warranty_Claim__c` object, developers only need to update the factory method, not every individual test class. In an enterprise context, such as an Automotive CRM, this means a single `TestDataFactory.createDealer()` method can instantly provide valid, compliant Dealer records for all test suites.

---

## 2. What is a Test Data Factory?

### Definition
A Test Data Factory is an Apex utility class used exclusively within the testing context to generate standardized, compliant, and reusable test records. 

### Purpose
Its primary purpose is to decouple test data *creation* logic from test *execution* logic. Test classes should focus on asserting business logic, not building complex object graphs.

### Reusable Test Data
Instead of creating an Account, Contact, and Vehicle in every test method, developers call reusable factory methods. This ensures consistency across the test suite.

### Factory Design Pattern
The Test Data Factory implements a variation of the software engineering **Factory Design Pattern**, adapting it to Salesforce's testing ecosystem. It abstracts the instantiation of complex objects (SObjects) behind clean, static method signatures.

#### Architecture Diagram

```text
[ Test Classes ]
      |
      v (Method Calls)
+-------------------------+
|    TestDataFactory      |
+-------------------------+
| + createAccount()       |
| + createVehicle()       |
| + createWarrantyClaim() |
+-------------------------+
      |
      v (SObject Records)
[ In-Memory / Database ]
```

---

## 3. Why Test Data Factory is Needed

Without a Test Data Factory, Salesforce projects quickly accumulate technical debt in their test suites.

| Problem | How Test Data Factory Solves It |
|---------|---------------------------------|
| **Duplicate Code** | Data creation logic is written once in the factory and reused everywhere. |
| **Hardcoded Values** | Defaults are managed centrally; overrides are passed via parameters. |
| **Difficult Maintenance** | When a new validation rule is added, you only update the factory method. |
| **Large Test Classes** | Test classes remain concise, focusing only on setup, execution, and assertions. |
| **Inconsistent Test Data** | Ensures all test data meets a baseline standard of quality and completeness. |

---

## 4. Factory Design Pattern

### What is the Factory Pattern?
In object-oriented programming, the Factory Pattern is a creational pattern that provides an interface for creating objects in a superclass, but allows subclasses or methods to alter the type and configuration of objects that will be created.

### Why Salesforce Uses It for Testing
Salesforce testing requires building complex relational data structures (SObjects). The Factory pattern abstracts the complexity of populating required fields, managing standard pricebooks, and linking lookup relationships.

### Advantages
- **Reusability:** Write once, use in hundreds of tests.
- **Maintainability:** Single point of failure/update for data structure changes.

#### Diagram: Factory Pattern in Apex Testing

```text
Test Method A ---> [ Factory Method (createVehicle) ] ---> Returns valid Vehicle
Test Method B ---> [ Factory Method (createVehicle) ] ---> Returns valid Vehicle
```

---

## 5. Creating a Basic Test Data Factory

Here is a production-quality example of a basic Test Data Factory:

```apex
@isTest
public class TestDataFactory {

    public static Account createAccount(Boolean doInsert) {
        Account acc = new Account(
            Name = 'Test Automotive Dealer',
            Industry = 'Automotive'
        );
        
        if (doInsert) {
            insert acc;
        }
        return acc;
    }

}
```

### Line-by-Line Explanation:
*   `@isTest`: Ensures this class does not count against the org's Apex code limit and can only be used in tests.
*   `public class TestDataFactory`: Standard public class declaration.
*   `public static Account createAccount(Boolean doInsert)`: A static method returning an Account. `Boolean doInsert` controls whether DML is executed.
*   `Account acc = new Account(...)`: Instantiates the Account with minimum required and standard fields.
*   `if (doInsert) { insert acc; }`: Checks the flag. If true, inserts the record into the test database.
*   `return acc;`: Returns the generated record to the calling test method.

---

## 6. Factory Methods

Factory methods in Salesforce are typically:
*   **Static:** So they can be called without instantiating the factory class.
*   **Reusable:** Designed to be generic enough for multiple test cases.
*   **Method Naming Conventions:** Typically prefixed with `create` or `build` (e.g., `createVehicle`, `buildWarrantyClaim`).
*   **Return Types:** Always return the exact SObject type (e.g., `Account`, `List<Contact>`) or a wrapper class for complex data graphs.

---

## 7. Parameterized Factory Methods

To make factory methods flexible, you should parameterize fields that frequently change across different test scenarios.

```apex
@isTest
public static Account createAccount(String name, String industry, Boolean doInsert) {
    Account acc = new Account(
        Name = name,
        Industry = industry,
        Type = 'Dealer'
    );
    
    if (doInsert) {
        insert acc;
    }
    return acc;
}
```

### Parameter Explanation:
*   `String name`: Allows the test to specify the Account name to assert against later.
*   `String industry`: Allows testing specific logic that fires only for certain industries (e.g., 'Automotive').
*   `Boolean doInsert`: Controls whether the record consumes a DML statement immediately.

---

## 8. Optional Record Insertion

Almost all enterprise factory methods include a `Boolean doInsert` (or `isInsert`) parameter. 

### Why is this needed?
1.  **Returning unsaved records:** Sometimes a test method needs to modify fields on the returned record *before* it is inserted to test specific triggers or validation rules.
2.  **Bulkification/DML Limits:** If you need to create an Account, Contact, and Vehicle, returning unsaved records allows you to collect them in a List and insert them in bulk, saving DML statements.

**Use Case Example:**
```apex
Account testAcc = TestDataFactory.createAccount('Honda Dealer', false);
testAcc.BillingState = 'CA'; // Modify before insert
insert testAcc;
```

---

## 9. Creating Standard Object Test Data

Below are examples of creating standard objects often used in an Automotive CRM.

```apex
@isTest
public class TestDataFactory {
    
    // Account
    public static Account createAccount(Boolean doInsert) {
        Account acc = new Account(Name = 'Test Dealer', Industry = 'Automotive');
        if (doInsert) insert acc;
        return acc;
    }

    // Contact
    public static Contact createContact(Id accountId, Boolean doInsert) {
        Contact con = new Contact(FirstName = 'Test', LastName = 'Technician', AccountId = accountId);
        if (doInsert) insert con;
        return con;
    }

    // Opportunity
    public static Opportunity createOpportunity(Id accountId, Boolean doInsert) {
        Opportunity opp = new Opportunity(
            Name = 'Fleet Sale', 
            StageName = 'Prospecting', 
            CloseDate = System.today().addDays(30),
            AccountId = accountId
        );
        if (doInsert) insert opp;
        return opp;
    }
}
```

*Explanation:* Each method accepts parent IDs where a relationship is required (like `AccountId` for Contact and Opportunity) and populates mandatory standard fields (like `StageName` and `CloseDate`).

---

## 10. Creating Custom Object Test Data

In an Automotive CRM context, custom objects are heavily utilized.

```apex
@isTest
public class AutomotiveDataFactory {
    
    // Vehicle (Custom Object)
    public static Vehicle__c createVehicle(Id dealerId, Boolean doInsert) {
        Vehicle__c veh = new Vehicle__c(
            VIN__c = '1HGCM82633A000' + Integer.valueof((Math.random() * 100)),
            Make__c = 'Honda',
            Model__c = 'Civic',
            Dealer__c = dealerId
        );
        if (doInsert) insert veh;
        return veh;
    }

    // Warranty Claim (Custom Object)
    public static Warranty_Claim__c createWarrantyClaim(Id vehicleId, Boolean doInsert) {
        Warranty_Claim__c claim = new Warranty_Claim__c(
            Vehicle__c = vehicleId,
            Claim_Date__c = System.today(),
            Status__c = 'Draft'
        );
        if (doInsert) insert claim;
        return claim;
    }
}
```

*Explanation:* `createVehicle` generates a somewhat random VIN to avoid duplicate value validation rules. `createWarrantyClaim` links back to the Vehicle.

---

## 11. Parent-Child Relationships

Testing often requires multi-level object graphs.

```text
[ Account (Dealer) ]
        |
        v
[ Contact (Technician) ]
        |
        v
[ Case (Service Request) ]
```

**Code Implementation:**
```apex
public static Case createServiceCase(Boolean doInsert) {
    // 1. Create Parent
    Account dealer = createAccount(true);
    
    // 2. Create Child
    Contact tech = createContact(dealer.Id, true);
    
    // 3. Create Grandchild
    Case serviceCase = new Case(
        Subject = 'Engine Diagnostics',
        AccountId = dealer.Id,
        ContactId = tech.Id,
        Status = 'New'
    );
    
    if (doInsert) insert serviceCase;
    return serviceCase;
}
```

*Explanation:* This compound factory method handles the creation of the entire hierarchy, returning the lowest-level child with all relationships properly established.

---

## 12. Complex Test Data

For deep hierarchies (e.g., Quote to Invoice), use a specialized method that returns a wrapper class or a `Map<String, SObject>` containing the whole graph, or rely on `@TestSetup` combined with the factory.

```text
Account -> Opportunity -> Quote -> Order -> Invoice
```

*Tip:* Avoid generating a 5-level hierarchy synchronously inside a single `for` loop, as it will rapidly consume DML limits. Use bulkified factory methods for complex graphs.

---

## 13. Bulk Test Data Generation

Governor limits necessitate bulk testing. Your factory must support generating Lists of records.

```apex
public static List<Account> createAccounts(Integer count, Boolean doInsert) {
    List<Account> accounts = new List<Account>();
    for(Integer i = 0; i < count; i++) {
        accounts.add(new Account(
            Name = 'Bulk Dealer ' + i,
            Industry = 'Automotive'
        ));
    }
    
    if (doInsert) {
        insert accounts;
    }
    return accounts;
}
```

### Usage:
*   `10 Accounts`: Minor trigger testing.
*   `100 Accounts`: General limits testing.
*   `200 Accounts`: Absolute threshold for Salesforce trigger batch sizes (`Trigger.new` contains up to 200 records).

---

## 14. Test Data Isolation

### SeeAllData=false
By default, Apex test classes run in a silo and cannot see production data. This is enforced by `@isTest(SeeAllData=false)`.

### Why Isolation Matters:
*   **Independent Tests:** Test A running in parallel with Test B should not fail because Test B modified a shared record.
*   **Repeatable Tests:** Tests shouldn't pass in Sandboxes and fail in Production because the underlying data is different.
*   **Reliable Testing:** Ensures tests evaluate code logic, not data presence. Tests *must* insert their own data via the Test Data Factory.

---

## 15. Using Test Data Factory in Test Classes

```apex
@isTest
private class WarrantyClaimTriggerTest {
    
    @isTest
    static void testClaimApproval() {
        // 1. Arrange (Setup Data)
        Account dealer = TestDataFactory.createAccount(true);
        Vehicle__c vehicle = AutomotiveDataFactory.createVehicle(dealer.Id, true);
        Warranty_Claim__c claim = AutomotiveDataFactory.createWarrantyClaim(vehicle.Id, false);
        
        claim.Status__c = 'Submitted';
        
        // 2. Act (Execute Logic)
        Test.startTest();
        insert claim;
        Test.stopTest();
        
        // 3. Assert (Verify Output)
        Warranty_Claim__c insertedClaim = [SELECT Id, Approval_Date__c FROM Warranty_Claim__c WHERE Id = :claim.Id];
        System.assertNotEquals(null, insertedClaim.Approval_Date__c, 'Approval Date should be stamped');
    }
}
```

*Explanation:* 
*   We use the factory to get dependencies (`dealer`, `vehicle`).
*   We get the `claim` *without* inserting (`false`).
*   We change the status to 'Submitted'.
*   We insert it within `Test.startTest()` to isolate governor limits.

---

## 16. Combining with @TestSetup

A Test Data Factory is a set of tools (methods). `@testSetup` is a lifecycle hook in a test class that runs *once* before any test methods execute.

| Feature | Test Data Factory | @TestSetup |
| :--- | :--- | :--- |
| **Nature** | Utility Class | Method Annotation |
| **Reusability** | Across the entire Salesforce Org / all test classes | Scoped to the specific Test Class |
| **Purpose** | Abstraction of record creation logic | Execution of data creation before tests run |

**How they work together:**
Use the Test Data Factory *inside* the `@testSetup` method to create the baseline data for the test class.

---

## 17. Governor Limits

Test data creation consumes governor limits. Testing 200 records requires careful factory design.

| Limit | Consideration during Test Data Creation |
| :--- | :--- |
| **SOQL Queries (100)** | Avoid querying records inside factory loops. Create relationships in memory. |
| **DML Statements (150)** | Use `doInsert=false` in loops, collect in a `List`, and insert once outside the loop. |
| **Heap Size (6MB)** | Don't retain unnecessary variables or massive lists if not required by assertions. |
| **CPU Time (10,000ms)**| Complex trigger frameworks running on 200 inserted test records can breach CPU limits. |

---

## 18. Enterprise Design Patterns

In large orgs, testing architecture evolves beyond a simple factory:

*   **Factory Pattern:** Basic record instantiation.
*   **Builder Pattern:** A fluent interface for building objects (`new AccountBuilder().withName('Honda').build();`). Useful for highly complex objects with optional parameters.
*   **Service Layer Integration:** Factories that can mock Service Layer responses rather than hitting the database.

---

## 19. Real Project Scenarios (Automotive CRM)

*   **Warranty Claim Test Data:** Requires Account (Dealer), Contact (Customer), Vehicle__c, and Claim_Line_Item__c. The factory method `createCompleteClaimGraph()` handles this to ensure triggers expecting rolled-up line item amounts don't fail.
*   **SAP Integration Records:** Factory methods create mock Integration_Log__c records to simulate SAP sync states (e.g., `Status = 'Pending_Sync'`).
*   **Shipment Processing:** Factory generates `Shipment__c` and related `Dealer_Inventory__c` records to test receiving logic.

---

## 20. Common Mistakes

| Mistake | Solution |
| :--- | :--- |
| **Creating data in every test class** | Refactor to use the centralized Test Data Factory. |
| **Hardcoded specific values** | Use parameters or random generators (e.g., random VINs) to prevent uniqueness constraint failures. |
| **No optional insert parameter** | Always implement `Boolean doInsert` to save DML statements when bulkifying. |
| **Too much logic in factory** | Keep factory methods dumb. They instantiate data; they do not test logic. |
| **Using SeeAllData=true** | Strictly forbidden unless testing Pricebooks (in older API versions) or specific standard metadata. Use `SeeAllData=false`. |

---

## 21. Best Practices Checklist

*   [x] **Keep factory methods reusable:** Methods should serve 90% of use cases.
*   [x] **Use meaningful method names:** e.g., `createAccountWithBillingAddress()`.
*   [x] **Parameterize methods:** Allow tests to inject specific states.
*   [x] **Support optional insert:** Always include `Boolean doInsert`.
*   [x] **Create related records when needed:** Pass parent Ids as parameters.
*   [x] **Avoid duplicate logic:** If an Opportunity needs an Account, have `createOpportunity` call `createAccount` if a null Id is passed.
*   [x] **Keep test data independent:** Use randomizers for unique fields (Emails, Usernames, VINs).
*   [x] **Use SeeAllData=false:** Never rely on prod/sandbox data.
*   [x] **Use @TestSetup where appropriate:** Combine factory methods with test setup blocks for efficiency.

---

## 22. Interview Questions & Answers

### Beginner Questions
**Q: What is a Test Data Factory?**
A: An Apex utility class (`@isTest`) used to centralize and standardize the creation of test data (SObjects) for use across multiple test classes.

### Intermediate Questions
**Q: Why do we pass a `Boolean doInsert` parameter to factory methods?**
A: To allow test classes to decide whether the record should be committed to the database immediately or held in memory. This is crucial for bulkifying test data creation (saving DML limits) and modifying fields before triggers run.

### Advanced Questions
**Q: How do you handle deep parent-child hierarchies in a Test Data Factory without hitting DML limits?**
A: By utilizing `doInsert=false` for children, returning lists of records, and inserting them sequentially in bulk. Alternatively, utilizing the `System.Test.loadData` method for static resources, or implementing a specialized graph creation method that executes minimal DML.

### Architect-Level Questions
**Q: Compare the Builder Pattern with the Factory Pattern for Salesforce Test Data. Which do you recommend for an Enterprise org?**
A: The Factory Pattern is simpler and excellent for standard scenarios (e.g., `createVehicle(dealerId)`). However, as objects grow to have dozens of validation rules, the Builder Pattern (`new VehicleBuilder().withDealer(dealerId).withStatus('In Transit').build()`) provides superior flexibility and readability, preventing "parameter pollution" where factory methods require 10+ arguments. In Enterprise orgs, a hybrid is best: a Test Data Factory wrapping Builder classes.

---

## 23. Revision Summary

*   **Test Data Factory:** Centralized `@isTest` utility class for generating mock records.
*   **Factory Pattern:** Software design pattern abstracting object creation.
*   **Reusable Test Data:** Eliminates duplicate code and reduces technical debt.
*   **Static Factory Methods:** Accessible without instantiating the class.
*   **Parameterized Methods:** Allow dynamic input for flexible testing.
*   **Optional Insert (`doInsert`):** Essential for DML limit management and in-memory field updates.
*   **Parent-Child Records:** Handled by passing Parent Ids to Child factory methods.
*   **Bulk Test Data:** Generate lists of 200 records to test trigger bulkification.
*   **Test Isolation:** Ensure tests are independent and agnostic of org data (`SeeAllData=false`).
*   **@TestSetup Integration:** Use factory methods inside `@testSetup` to establish baseline context efficiently.
*   **Governor Limits:** Respect 150 DML and 100 SOQL limits by batching factory inserts.