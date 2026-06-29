# Before Triggers Validation Updates

---

## 1. Introduction

### What Before Triggers Are
In Salesforce, **Before Triggers** are specialized Apex code blocks that execute immediately before a record is inserted into or updated in the database. They act as the first line of defense and automation in the Salesforce transaction lifecycle, intercepting records while they are still in memory.

### Why Salesforce Provides Before Triggers
Salesforce provides Before Triggers primarily for **efficiency** and **data integrity**. Because the records are held in memory (within the `Trigger.new` collection) prior to being committed to the database, developers can modify these records without needing to issue a separate Data Manipulation Language (DML) statement. This saves processing time and preserves critical DML governor limits.

### Difference Between Validation and Automation
* **Validation:** Ensuring data meets strict business rules before it is allowed to enter the database. If rules are violated, the transaction is blocked.
* **Automation:** Modifying data automatically (e.g., calculating fields, standardizing formats, auto-populating relationships) behind the scenes without user intervention.

Before triggers handle *both* exceptionally well, but they should be preferred for automation that requires cross-object lookups or complex programmatic validation that declarative tools cannot handle.

### Real-World Business Use Cases (Automotive CRM)
* **Data Standardization:** Automatically converting a `Vehicle__c` Chassis Number to uppercase before saving.
* **Complex Validation:** Preventing a `Warranty_Claim__c` from being marked "Approved" if the related `Vehicle__c` warranty expired before the claim creation date.

---

## 2. What are Before Triggers?

### Definition
A Before Trigger is an Apex script that fires on `before insert`, `before update`, or `before delete` events. For this guide, we focus on **Before Insert** and **Before Update**.

### Before Insert
Executes when a new record is created, but *before* it is assigned a Salesforce Id and saved to the database. This is the optimal place to set default values or standardize formatting.

### Before Update
Executes when an existing record is modified, but *before* the new values are committed. It allows developers to compare the incoming data (`Trigger.new`) against the existing database data (`Trigger.old`) to track state changes and enforce transition rules.

### Internal Execution Process & Transaction Lifecycle
When a user clicks "Save" (or an API call is made):
1.  Salesforce loads the original record from the database (for updates).
2.  Salesforce overwrites the fields with the new values provided by the user/API.
3.  These in-memory records are passed to the Before Trigger via `Trigger.new`.
4.  Apex modifies `Trigger.new` or flags errors.
5.  Salesforce proceeds with the rest of the save order of execution.

### Why Records Can Be Modified Without Explicit DML
Because `Trigger.new` represents a reference to the exact memory space Salesforce is currently processing for the save operation, any changes you make to the fields of these sObjects are automatically carried forward to the database. Issuing an `update` DML statement on `Trigger.new` inside a Before Trigger will cause a runtime exception (`System.FinalException: Record is read-only`).

---

## 3. Trigger Execution Order

Understanding the **Order of Execution** is the single most important concept for an Enterprise Salesforce Architect. When a record is saved, Salesforce processes events in a very strict order:

1.  **System Validation Rules:** Checks for required fields, field formats (e.g., valid email), and field length.
2.  **Before-Save Flows:** Record-triggered flows optimized for speed.
3.  **Before Triggers:** Executes all `before insert` or `before update` Apex triggers.
4.  **System Validation (Again):** Verifies that Before Triggers didn't break standard validations.
5.  **Custom Validation Rules:** Executes declarative validation rules configured on the object.
6.  **Duplicate Rules:** Executes duplicate matching rules.
7.  **After Triggers:** Executes `after insert` or `after update` Apex triggers (records are now temporarily saved with Ids, but not committed).
8.  **Assignment Rules:** (For Cases/Leads).
9.  **Auto-Response Rules:** (For Cases/Leads).
10. **Workflow Rules:** Executes legacy workflow field updates (which can cause triggers to fire again).
11. **Escalation Rules:** (For Cases).
12. **Processes (Process Builder):** Executes processes.
13. **Flows:** Executes Record-Triggered After-Save Flows.
14. **Entitlement Rules:** Executes entitlement processes.
15. **Roll-up Summary Fields:** Calculates roll-ups on parent records (which triggers the parent's save process).
16. **Commit:** The transaction is permanently committed to the database.

---

## 4. Before Insert Triggers

### Purpose
To manipulate or validate data on brand-new records before they reach the database.

### When They Execute
When an `.insert()` DML operation occurs, or a user creates a record via the UI. At this stage, `Id` fields are `null`.

### Common Use Cases
* **Default Values:** Setting a status based on the user's profile.
* **Data Standardization:** Trimming whitespaces, capitalizing names.
* **Business Validations:** Checking if a related parent object allows new children.

### Production-Quality Example

```apex
trigger VehicleTrigger on Vehicle__c (before insert) {
    for (Vehicle__c veh : Trigger.new) {
        // Data Standardization: Ensure Chassis Number is uppercase
        if (String.isNotBlank(veh.Chassis_Number__c)) {
            veh.Chassis_Number__c = veh.Chassis_Number__c.toUpperCase();
        }
        
        // Default Values: Auto-set Status for new vehicles
        if (veh.Status__c == null) {
            veh.Status__c = 'In Transit';
        }
        
        // Business Validation: Ensure Manufacturing Year is not in the future
        if (veh.Manufacturing_Year__c > Date.today().year()) {
            veh.Manufacturing_Year__c.addError('Manufacturing Year cannot be in the future.');
        }
    }
}
```

#### Line-by-Line Explanation:
* `trigger VehicleTrigger...`: Defines the trigger on the `Vehicle__c` object for the `before insert` event.
* `for (Vehicle__c veh : Trigger.new)`: Iterates through the bulkified list of new records being inserted.
* `if (String.isNotBlank(veh.Chassis_Number__c))`: Checks if the field has a value to avoid Null Pointer Exceptions.
* `veh.Chassis_Number__c.toUpperCase()`: Modifies the field in memory. No DML is required.
* `if (veh.Status__c == null)`: Checks if the user left the status blank.
* `veh.Status__c = 'In Transit'`: Assigns a default enterprise value.
* `if (veh.Manufacturing_Year__c > ...)`: Evaluates business logic.
* `veh.Manufacturing_Year__c.addError(...)`: Blocks the save for this specific record and attaches the error to the specific field.

---

## 5. Before Update Triggers

### Purpose
To validate state changes or update fields based on modifications made to an existing record.

### Existing Record Updates
Because the record already exists, you have access to both the new version (`Trigger.new`) and the old version (`Trigger.old` / `Trigger.oldMap`).

### Comparing Old and New Values
This is the primary function of a Before Update trigger: asking *"Did this specific field change during this transaction?"*

### Production-Quality Example

```apex
trigger WarrantyClaimTrigger on Warranty_Claim__c (before update) {
    for (Warranty_Claim__c claim : Trigger.new) {
        // Fetch the old version of the record using oldMap
        Warranty_Claim__c oldClaim = Trigger.oldMap.get(claim.Id);
        
        // Conditional Update: If status changes to Approved, set Approval Date
        if (claim.Status__c == 'Approved' && oldClaim.Status__c != 'Approved') {
            claim.Approval_Date__c = System.today();
        }
        
        // Status Validation: Prevent reopening closed claims
        if (oldClaim.Status__c == 'Closed' && claim.Status__c != 'Closed') {
            claim.addError('You cannot reopen a Closed Warranty Claim. Please create a new one.');
        }
    }
}
```

#### Line-by-Line Explanation:
* `trigger WarrantyClaimTrigger...`: Defines the trigger for `before update`.
* `for (Warranty_Claim__c claim : Trigger.new)`: Loops through the records being updated.
* `Warranty_Claim__c oldClaim = Trigger.oldMap.get(claim.Id)`: Retrieves the pre-transaction version of the record using its Id.
* `if (claim.Status__c == 'Approved' && oldClaim.Status__c != 'Approved')`: Crucial pattern. Checks if the Status *is currently* 'Approved' AND *was not previously* 'Approved'. This ensures logic only runs when the field actually changes.
* `claim.Approval_Date__c = System.today()`: Auto-populates the date in memory.
* `if (oldClaim.Status__c == 'Closed'...)`: Validates state transition.
* `claim.addError(...)`: Blocks the transaction at the record level.

---

## 6. Trigger Context Variables

Salesforce provides implicit variables in the `System.Trigger` class to help manage execution context.

| Variable | Description | Before Insert | Before Update |
| :--- | :--- | :---: | :---: |
| **Trigger.new** | List of new versions of sObject records. | Available | Available |
| **Trigger.old** | List of old versions of sObject records. | `null` | Available |
| **Trigger.newMap** | Map of Ids to the new versions of sObject records. | `null` (No Ids yet) | Available |
| **Trigger.oldMap** | Map of Ids to the old versions of sObject records. | `null` | Available |
| **Trigger.isBefore** | Returns true if the trigger fired before saving. | `true` | `true` |
| **Trigger.isInsert** | Returns true if the trigger fired on insert. | `true` | `false` |
| **Trigger.isUpdate** | Returns true if the trigger fired on update. | `false` | `true` |

---

## 7. Validation Logic using Before Triggers

### Business & Complex Validations
While Salesforce Declarative Validation Rules are great for simple logic, Before Triggers shine when:
* **Cross-Object Validations:** You need to validate against parent, child, or completely unrelated objects (requires SOQL).
* **Complex Map/Set Logic:** Validations requiring aggregation or complex data structures.

### Why Sometimes Triggers are Preferred over Validation Rules
1.  **State Tracking:** It is easier to write complex "prior value" logic in Apex than using multiple `ISCHANGED()` and `PRIORVALUE()` functions in declarative rules.
2.  **API Callout Simulation:** Preparing data that relies on related integration tables.
3.  **Bypass Mechanisms:** Enterprise frameworks allow you to easily bypass triggers for data migration (e.g., `TriggerBypass.isDisabled('WarrantyClaim')`), which is harder to do dynamically with Validation Rules.

---

## 8. Using addError()

The `addError()` method is used inside triggers to prevent DML operations from committing and to display custom error messages.

### Record-Level Error
**Syntax:** `record.addError('Message');`
**Behavior:** The error appears at the top of the page layout in the UI.

### Field-Level Error
**Syntax:** `record.Field_Name__c.addError('Message');`
**Behavior:** The error appears directly beneath the specific field on the UI, guiding the user exactly to the problem.

### UI vs API Behavior
* **UI:** The user sees a red text error on the standard or custom Lightning page.
* **API:** The external system making the call receives a `SOAP Fault` or `REST 400 Bad Request` with the `FIELD_CUSTOM_VALIDATION_EXCEPTION` error code and your exact message.

---

## 9. Updating Fields in Before Triggers

Modifying fields in Before Triggers is heavily preferred over After Triggers (which would require an explicit DML statement and re-run the transaction).

```apex
// Correct (Before Trigger)
Trigger.new[0].Status__c = 'Approved'; 
// Done. Modifies memory. No DML needed.
```

```apex
// Incorrect (Will throw exception in Before Trigger)
Trigger.new[0].Status__c = 'Approved';
update Trigger.new[0]; // FATAL ERROR: DML statement cannot operate on trigger.new
```

### Performance Benefits
Updating fields in a Before Trigger costs **zero DML statements**. If you update 200 records in a Before Trigger, it counts as 0 against your limit of 150 DML statements per transaction.

---

## 10. Bulkification in Before Triggers

Bulkification means writing code that can handle 1 record or 200 records (the maximum batch size) simultaneously without hitting limits.

### Avoid SOQL and DML in Loops
Never place a `[SELECT ...]` or `update` inside a `for` loop. Instead, use Collections (Lists, Sets, Maps) to gather IDs, perform one SOQL query, and then loop again.

### Production-Quality Example (Automotive)

```apex
trigger WarrantyClaimTrigger on Warranty_Claim__c (before insert) {
    // 1. Initialize a Set to collect parent Ids
    Set<Id> vehicleIds = new Set<Id>();
    
    // 2. First Loop: Collect Ids
    for (Warranty_Claim__c claim : Trigger.new) {
        if (claim.Vehicle__c != null) {
            vehicleIds.add(claim.Vehicle__c);
        }
    }
    
    // 3. Single SOQL Query outside the loop, stored in a Map
    Map<Id, Vehicle__c> vehicleMap = new Map<Id, Vehicle__c>();
    if (!vehicleIds.isEmpty()) {
        vehicleMap = new Map<Id, Vehicle__c>([
            SELECT Id, Warranty_Expiration_Date__c, Dealer__c 
            FROM Vehicle__c 
            WHERE Id IN :vehicleIds
        ]);
    }
    
    // 4. Second Loop: Process logic using the Map
    for (Warranty_Claim__c claim : Trigger.new) {
        if (claim.Vehicle__c != null && vehicleMap.containsKey(claim.Vehicle__c)) {
            Vehicle__c relatedVehicle = vehicleMap.get(claim.Vehicle__c);
            
            // Auto-populate Dealer from Vehicle
            if (claim.Dealer__c == null) {
                claim.Dealer__c = relatedVehicle.Dealer__c;
            }
            
            // Cross-object Validation
            if (relatedVehicle.Warranty_Expiration_Date__c < System.today()) {
                claim.addError('Cannot create a claim. The vehicle warranty has expired.');
            }
        }
    }
}
```

#### Line-by-Line Explanation:
* `Set<Id> vehicleIds = new Set<Id>();`: Prepares a collection to hold unique Vehicle Ids.
* `for (Warranty_Claim__c claim : Trigger.new)`: Loops through incoming claims.
* `vehicleIds.add(claim.Vehicle__c);`: Extracts the parent Vehicle Id from each claim.
* `Map<Id, Vehicle__c> vehicleMap...`: Prepares a map to store the queried Vehicles.
* `if (!vehicleIds.isEmpty())`: Ensures we don't run an empty SOQL query.
* `[SELECT Id... WHERE Id IN :vehicleIds]`: Queries all related vehicles in a single transaction.
* `for (Warranty_Claim__c claim : Trigger.new)`: Loops through the claims a second time to apply logic.
* `if (... && vehicleMap.containsKey(claim.Vehicle__c))`: Verifies the map has the related vehicle data.
* `claim.Dealer__c = relatedVehicle.Dealer__c;`: Performs the field update in memory.
* `claim.addError(...);`: Applies cross-object validation logic based on the queried map.

---

## 11. Governor Limits

Salesforce operates in a multi-tenant environment. To prevent one customer from monopolizing server resources, strict limits are enforced per transaction.

| Limit Type | Limit | Impact on Before Triggers |
| :--- | :--- | :--- |
| **SOQL Queries** | 100 | Bulkify queries. Do not query inside `for` loops. |
| **SOQL Rows** | 50,000 | Be mindful of querying large child relationships. |
| **CPU Time** | 10,000 ms | Optimize map iterations; avoid nested `for` loops. |
| **Heap Size** | 6 MB (Sync) | Clear large lists if no longer needed. Avoid massive data processing. |
| **DML Statements** | 150 | Not usually a risk for Before Triggers (no DML needed), but relevant for cross-object logs. |
| **Trigger Batch Size** | 200 records | Apex processes `Trigger.new` in chunks of 200. |

---

## 12. Trigger Handler Pattern

For Enterprise architecture, **never write logic directly inside the `.trigger` file**. Use the Trigger Handler Pattern.

### Principles
* **One Trigger Per Object:** Avoids order-of-execution unpredictability.
* **Separation of Concerns:** Trigger file manages the *events*, Handler class manages the *logic*.

### Example Implementation

**1. The Trigger File:**
```apex
trigger AccountTrigger on Account (before insert, before update) {
    AccountTriggerHandler handler = new AccountTriggerHandler();
    
    if (Trigger.isBefore) {
        if (Trigger.isInsert) {
            handler.onBeforeInsert(Trigger.new);
        }
        if (Trigger.isUpdate) {
            handler.onBeforeUpdate(Trigger.new, Trigger.oldMap);
        }
    }
}
```

**2. The Handler Class:**
```apex
public with sharing class AccountTriggerHandler {
    
    public void onBeforeInsert(List<Account> newList) {
        setDefaultRegion(newList);
    }
    
    public void onBeforeUpdate(List<Account> newList, Map<Id, Account> oldMap) {
        validateCreditLimit(newList, oldMap);
    }
    
    // Logic isolated in private helper methods
    private void setDefaultRegion(List<Account> newList) {
        for (Account acc : newList) {
            if (acc.BillingCountry == 'USA') {
                acc.Region__c = 'North America';
            }
        }
    }
    
    private void validateCreditLimit(List<Account> newList, Map<Id, Account> oldMap) {
        for (Account acc : newList) {
            Account oldAcc = oldMap.get(acc.Id);
            if (acc.Credit_Limit__c > 100000 && oldAcc.Credit_Limit__c <= 100000) {
                if (acc.Credit_Score__c < 700) {
                    acc.Credit_Limit__c.addError('High credit limits require a score of 700+');
                }
            }
        }
    }
}
```

#### Line-by-Line Explanation (Handler):
* `public with sharing class AccountTriggerHandler`: Defines the handler class, enforcing sharing rules.
* `public void onBeforeInsert(List<Account> newList)`: Public method called by the trigger, accepting `Trigger.new`.
* `setDefaultRegion(newList)`: Delegates to a private helper method for specific business logic.
* `private void validateCreditLimit(...)`: A private method handling update validation.
* `Account oldAcc = oldMap.get(acc.Id);`: Retrieves the previous state of the account.
* `if (acc.Credit_Limit__c > 100000 && oldAcc.Credit_Limit__c <= 100000)`: Checks if the limit *just crossed* the threshold during this transaction.

---

## 13. Validation Rules vs Before Triggers

| Feature | Validation Rule | Before Trigger | Record-Triggered Flow (Before) |
| :--- | :--- | :--- | :--- |
| **Complexity** | Simple, Single-object | Highly Complex, Cross-object | Moderate, Single/Parent-object |
| **Code Required**| No (Declarative) | Yes (Apex) | No (Declarative) |
| **Maintenance** | Easy for Admins | Requires Developers | Medium (Admins/Devs) |
| **Execution Order**| After Before Triggers | Before Validation Rules | Before Before Triggers |
| **Best Used For**| Mandatory field checks, basic data formats. | Bulk complex cross-object validation, Map logic. | Simple same-record field updates. |

---

## 14. Enterprise Design Patterns

In mature Salesforce orgs (like a global Automotive CRM), architecture goes beyond basic handlers:

* **Trigger Framework Pattern (fflib):** A structured framework standardizing bypasses, loop management, and method dispatching.
* **Domain Layer Pattern:** Represents the sObject (e.g., `Vehicles.cls`). Trigger handlers delegate logic to the Domain class to encapsulate object-specific behaviors.
* **Service Layer Pattern:** Cross-object, transactional business logic (e.g., `WarrantyService.cls`). If a trigger needs to invoke SAP integration or calculate multi-object data, the Domain calls the Service layer.
* **Unit of Work Pattern:** Manages database DML operations centrally to optimize limits. (More applicable to After Triggers, but relevant for Enterprise context).

---

## 15. Real Project Scenarios (Automotive CRM)

* **Warranty Claim Validation:** `before insert` trigger checks the `Vehicle__c.Warranty_End_Date__c`. If the date is past, the claim throws an `addError()`.
* **Prevent Closed Claim Updates:** `before update` trigger evaluates `oldMap`. If `Status__c` was 'Closed' and the user modifies the `Claim_Amount__c`, reject the transaction.
* **Auto-populate Dealer Information:** `before insert` trigger queries the associated `Vehicle__c` and copies the `Dealer_Id__c` to the `Warranty_Claim__c` so reporting aligns automatically.
* **Validate Vehicle Chassis Numbers:** `before insert` and `before update` triggers run Regex matching to ensure Chassis Numbers strictly follow a 17-character VIN standard format.
* **Default Service Center Assignment:** Based on the Customer's `BillingPostalCode`, a `before insert` trigger queries a Custom Metadata Type mapping table to stamp the nearest `Service_Center__c` ID.

---

## 16. Performance Optimization

Architect-level tips for optimal Before Triggers:
* **Bulk-safe Coding:** Always assume `Trigger.new` will have 200 records.
* **Efficient SOQL Usage:** Never query fields you don't need. Filter early (`WHERE Id IN :recordIds`).
* **Map-based Lookups:** Avoid nested `for` loops. Query data into Maps and retrieve via `.get(Id)`. This turns an $O(N^2)$ operation into an $O(N)$ operation, massively saving CPU time.
* **Fast Fail Logic:** In loops, put the most restrictive `if` conditions first.
* **Avoid Repeated Variables:** Do not instantiate variables inside a loop if they can be instantiated outside.

---

## 17. Common Mistakes

| Mistake | Consequence | Solution |
| :--- | :--- | :--- |
| **SOQL inside loops** | Limit Exception (101 SOQL) | Gather Ids in a Set, query outside loop, store in Map. |
| **DML inside loops** | Limit Exception (151 DML) | N/A for Before Triggers (No DML needed), but for After Triggers, use a List and DML once outside. |
| **Multiple Triggers per object**| Unpredictable execution order | Implement One Trigger Per Object and a Handler. |
| **Using After for same-object updates** | Hits DML limits, runs transaction twice | Always use Before triggers for modifying `Trigger.new`. |
| **Recursive Triggers** | Maximum stack depth reached | Use a static boolean in an Apex class to track if the trigger has run (e.g., `TriggerUtility.hasRun`). |
| **Hardcoded IDs** | Deployment failure between Sandboxes | Query the ID dynamically or use Custom Labels/Metadata. |

---

## 18. Debugging Before Triggers

* **Developer Console / Debug Logs:** Set the `Apex Code` log level to `FINEST`. Search for `USER_DEBUG` or exception lines.
* **VS Code Replay Debugger:** Download the log and step through the Before Trigger line-by-line to inspect variable state in memory.
* **Trigger Execution Analysis:** Look for `EXECUTION_STARTED` and `EXECUTION_FINISHED` in logs to see exactly how long your trigger took (CPU time profiling).
* **Governor Limit Monitoring:** Use `Limits.getQueries()` and `Limits.getCpuTime()` dynamically in your code via `System.debug()` to track performance in real-time.

---

## 19. Interview Questions & Answers

### Beginner Questions
**Q: Can you perform a DML operation on `Trigger.new` in a Before Trigger?**
*A: No, issuing an update or insert on `Trigger.new` in a Before context will result in a runtime exception because the records are already in the process of being saved.*

### Intermediate Questions
**Q: How do you check if a field's value has changed during an update?**
*A: By comparing the field value in `Trigger.new` with the field value in `Trigger.oldMap` using the record's Id. (e.g., `if(newRecord.Status__c != oldMap.get(newRecord.Id).Status__c)`).*

### Advanced Questions
**Q: Why is `Trigger.newMap` null in a Before Insert trigger?**
*A: Because the records have not yet been committed to the database, they have not generated Salesforce Ids. Maps require Ids for keys.*

### Architect-Level Questions
**Q: You have a Before Trigger failing due to CPU timeouts, but there are no nested loops. What might be causing it, and how do you fix it?**
*A: The trigger might be firing recursively due to workflow rules or other automation updating the record, or it is processing massive collections inefficiently. I would review the Order of Execution, consolidate automation into the trigger, implement a static recursion control mechanism, and check if Before-Save Flows can absorb some of the declarative logic.*

---

## 20. Revision Summary

* **Before Insert:** Modifies/validates brand new records before Id generation.
* **Before Update:** Modifies/validates existing records. Used to track state changes.
* **addError():** Blocks transaction. Can be applied to the Record or a specific Field.
* **Context Variables:** `old` and `oldMap` are null in Insert. `newMap` is null in Before Insert.
* **Field Updates:** Simply assign the value (`Trigger.new[0].Field__c = 'X'`). No DML required.
* **Bulkification:** Always use Sets to collect Ids and Maps to query related data. No queries in loops.
* **Governor Limits:** Max 100 SOQL queries, 10,000ms CPU time per transaction.
* **Trigger Handler Pattern:** One trigger per object. Route logic to a handler class for maintainability.
* **Best Practices:** Only use Before triggers if same-record Before-Save Flows cannot handle the complexity. Keep logic modular.