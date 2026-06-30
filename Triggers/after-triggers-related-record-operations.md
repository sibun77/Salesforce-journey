# After Triggers Related Record Operations

# 1. Introduction

**After Triggers** are Apex scripts that execute immediately after a record has been saved to the Salesforce database but before the transaction is officially committed. 

Salesforce provides After Triggers primarily so developers can access system-generated fields (like the Record `Id`, `CreatedDate`, `SystemModstamp`, and formula fields) and use them to orchestrate complex logic across the system. While *Before* triggers are designed for same-record validation and field manipulation, *After* triggers are the engine for **related record operations** and **cross-object automation**.

**Real-World Automotive Business Use Cases:**
* Creating a related `Warranty_Line_Item__c` only after a `Warranty_Claim__c` is inserted (since the child record requires the parent's `Id`).
* Generating a `Service_History__c` record and notifying the customer when a `Work_Order__c` status changes to "Completed".
* Updating a parent `Dealer__c`'s roll-up statistics when a child `Vehicle_Registration__c` is deleted.

---

# 2. What are After Triggers?

An After Trigger fires during the post-save phase of the execution lifecycle. By this point, Salesforce has generated the primary key (Record ID) and saved the record, but holds the transaction open.

* **After Insert:** Fires after a new record is inserted into the database. The `Id` is now available.
* **After Update:** Fires after an existing record's fields are saved. Both new and old states are available for comparison.
* **After Delete:** Fires after a record is moved to the Recycle Bin (or hard deleted). Used to cascade logic or log data.
* **After Undelete:** Fires after a record is restored from the Recycle Bin. Used to re-establish connections or recalculate totals.

**Transaction Lifecycle & Committed Values:**
In an After Trigger, the data is highly reliable. Because the record has already passed system validation, before triggers, and custom validation rules, the values in `Trigger.new` reflect what will actually be committed to the database. However, because the record is locked for saving, **you cannot directly update fields on `Trigger.new`** in an After Trigger; you must perform explicit DML on related records instead.

---

# 3. Before vs After Triggers

| Feature | Before Trigger | After Trigger |
| :--- | :--- | :--- |
| **Primary Purpose** | Validation, same-record field updates. | Related record operations, cross-object automation. |
| **DML Required?** | No explicit DML required on `Trigger.new`. | Explicit DML required to update any record (including related ones). |
| **Field Updates** | Directly modify `Trigger.new` fields. | Read-only access to `Trigger.new`. |
| **Record IDs** | Not available on Insert. | Always available. |
| **System Fields** | Not yet calculated (e.g., Formulas). | Calculated and available. |

**When to use:** Use *Before* to set the `Status__c` of a Warranty Claim based on its input. Use *After* to create the related `Invoice__c` once that Claim is approved and saved.

---

# 4. Trigger Order of Execution

Understanding the Order of Execution (OoE) is critical for Enterprise Architecture to prevent unexpected recursion and bugs.

1.  **System Validation:** Checks required fields and field formats.
2.  **Before Triggers:** Executes `before insert` / `before update`.
3.  **Validation Rules:** Custom validation rules run.
4.  **Duplicate Rules:** Checks for duplicate records.
5.  **Database Save:** Record is saved to DB (but not committed); ID is generated.
6.  **After Triggers:** Executes `after insert` / `after update`.
7.  **Assignment Rules:** Executes lead/case assignment.
8.  **Auto Response Rules:** Sends automated emails.
9.  **Workflow Rules:** Field updates and email alerts (Field updates will trigger a re-run of triggers).
10. **Flows & Processes:** Record-Triggered Flows and Process Builders execute.
11. **Roll-Up Summary:** Parent roll-ups calculate.
12. **Commit:** Data is permanently saved to the database.

---

# 5. After Insert Triggers

**Purpose:** To perform actions that require the newly generated Record ID.
**Common Use Cases:** Creating child records, sending external system notifications (via future methods), and creating audit logs.

### Example: Create Warranty Line Items after Warranty Claim creation

```apex
// WarrantyClaimTriggerHandler.cls
public static void handleAfterInsert(List<Warranty_Claim__c> newClaims) {
    // 1. Initialize a list to hold the new child records
    List<Warranty_Line_Item__c> linesToInsert = new List<Warranty_Line_Item__c>();
    
    // 2. Iterate through the newly created Claims
    for (Warranty_Claim__c claim : newClaims) {
        // 3. Check business criteria
        if (claim.Type__c == 'Comprehensive') {
            // 4. Create child record and link it using the newly generated Parent ID
            Warranty_Line_Item__c line = new Warranty_Line_Item__c();
            line.Warranty_Claim__c = claim.Id; // ID is available in After Insert
            line.Description__c = 'Default Comprehensive Coverage';
            linesToInsert.add(line);
        }
    }
    
    // 5. Perform bulkified DML outside the loop
    if (!linesToInsert.isEmpty()) {
        insert linesToInsert;
    }
}
```

---

# 6. After Update Triggers

**Purpose:** To detect state changes (e.g., Status changed from 'Pending' to 'Approved') and synchronize related records based on that delta.

### Example: Update Invoice Status when Warranty Claim is Approved

```apex
public static void handleAfterUpdate(Map<Id, Warranty_Claim__c> newMap, Map<Id, Warranty_Claim__c> oldMap) {
    Set<Id> approvedClaimIds = new Set<Id>();
    
    // 1. Identify which records actually changed to 'Approved'
    for (Warranty_Claim__c claim : newMap.values()) {
        Warranty_Claim__c oldClaim = oldMap.get(claim.Id);
        if (claim.Status__c == 'Approved' && oldClaim.Status__c != 'Approved') {
            approvedClaimIds.add(claim.Id);
        }
    }
    
    // 2. Query related records if necessary and update them
    if (!approvedClaimIds.isEmpty()) {
        List<Invoice__c> invoicesToUpdate = [
            SELECT Id, Status__c 
            FROM Invoice__c 
            WHERE Warranty_Claim__c IN :approvedClaimIds
        ];
        
        for (Invoice__c inv : invoicesToUpdate) {
            inv.Status__c = 'Ready for Payment';
        }
        update invoicesToUpdate;
    }
}
```

---

# 7. After Delete Triggers

**Purpose:** To execute logic when a record is deleted, such as cleaning up related orphan records (if not handled by Master-Detail cascades), updating parent summaries, or logging the deletion for audit compliance.

```apex
public static void handleAfterDelete(List<Vehicle_Registration__c> oldRegistrations) {
    List<Audit_Log__c> logs = new List<Audit_Log__c>();
    for(Vehicle_Registration__c reg : oldRegistrations) {
        logs.add(new Audit_Log__c(
            Action__c = 'Deleted',
            Record_Name__c = reg.Name,
            Details__c = 'Registration deleted for VIN: ' + reg.VIN__c
        ));
    }
    insert logs;
}
```

---

# 8. After Undelete Triggers

**Purpose:** To restore custom business logic or relationships when a record is rescued from the Recycle Bin.

```apex
public static void handleAfterUndelete(List<Warranty_Claim__c> restoredClaims) {
    // Example: Recalculate parent Dealer statistics when a claim is restored
    Set<Id> dealerIds = new Set<Id>();
    for(Warranty_Claim__c claim : restoredClaims) {
        dealerIds.add(claim.Dealer__c);
    }
    DealerService.recalculateClaimMetrics(dealerIds);
}
```

---

# 9. Trigger Context Variables

| Variable | Type | Insert | Update | Delete | Undelete | Description |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `Trigger.new` | `List<sObject>` | Yes | Yes | No | Yes | New versions of records. |
| `Trigger.old` | `List<sObject>` | No | Yes | Yes | No | Old versions of records. |
| `Trigger.newMap` | `Map<Id, sObject>` | Yes (After) | Yes | No | Yes | Map of IDs to new record versions. |
| `Trigger.oldMap` | `Map<Id, sObject>` | No | Yes | Yes | No | Map of IDs to old record versions. |

*Booleans:* `Trigger.isAfter`, `Trigger.isInsert`, `Trigger.isUpdate`, `Trigger.isDelete`, `Trigger.isUndelete` are used to route logic dynamically in trigger handler patterns.

---

# 10. Related Record Operations

Related record operations form the core of enterprise Salesforce development. Because Salesforce is a relational database, an event on one object (like a `Work_Order__c`) almost always impacts another (like `Parts_Inventory__c` or `Service_History__c`).

Always follow this pattern:
1. Identify records meeting the criteria.
2. Collect related Ids in a `Set<Id>`.
3. Query the related records.
4. Process updates in memory.
5. Perform a single explicit DML statement.

---

# 11. Parent-Child Relationship Operations

* **Master-Detail:** Deletions cascade automatically. Roll-up summary fields are native.
* **Lookup:** Deletions do not cascade natively. Roll-ups require custom Apex.
* **Manual Roll-Ups:** Used in After Triggers when Roll-Up Summary Fields hit their limit (max 40) or when relationships are Lookups.

---

# 12. Cross-Object Updates

In an Automotive CRM, data spans many objects.
* **Warranty Claim → Invoice:** Approval triggers invoice creation.
* **Dealer → Vehicle:** Changing a dealer's region might update alignment on all child vehicles.
* **Service Center → Work Orders:** Updating a center's operational hours might trigger recalculation of SLAs on pending work orders.

---

# 13. DML Operations in After Triggers

In a Before Trigger, you modify the record in memory, and Salesforce saves it. In an After Trigger, the record is locked. If you want to change a related record, you must use explicit DML:

* `insert`: Adding new child records.
* `update`: Modifying parent or child records.
* `delete`: Removing obsolete related data.
* `undelete`: Restoring related data.
* `upsert`: Inserting or updating based on an External ID (excellent for SAP Integration).

---

# 14. Bulkification in After Triggers

Bulkification is the practice of designing code to handle up to 200 records at once seamlessly.

**Rules:**
1.  **Never put SOQL in a loop.**
2.  **Never put DML in a loop.**
3.  **Use Collections (Sets, Maps, Lists) heavily.**

```apex
// BAD - Anti-pattern
for (Warranty_Claim__c claim : Trigger.new) {
    // SOQL inside loop! Will hit 100 limit quickly.
    List<Invoice__c> inv = [SELECT Id FROM Invoice__c WHERE Warranty_Claim__c = :claim.Id]; 
}

// GOOD - Enterprise Pattern
Set<Id> claimIds = new Set<Id>();
for(Warranty_Claim__c claim : Trigger.new) {
    claimIds.add(claim.Id);
}
// Single query outside the loop
List<Invoice__c> invoices = [SELECT Id, Warranty_Claim__c FROM Invoice__c WHERE Warranty_Claim__c IN :claimIds];
```

---

# 15. Governor Limits

| Resource | Synchronous Limit | Asynchronous Limit |
| :--- | :--- | :--- |
| SOQL Queries | 100 | 200 |
| DML Statements | 150 | 150 |
| Records processed via DML | 10,000 | 10,000 |
| CPU Time | 10,000 ms | 60,000 ms |
| Heap Size | 6 MB | 12 MB |

*Scalable Design:* Move heavy cross-object processing to `@future` or `Queueable` Apex if synchronous CPU limits are a risk during complex After Update operations.

---

# 16. Recursion Prevention

Recursion occurs when Trigger A updates Object B, which triggers Object B's trigger, which updates Object A, firing Trigger A again.

**Static Boolean/Set Pattern:**
```apex
public class RecursionControl {
    public static Set<Id> processedClaimIds = new Set<Id>();
}
```
In your trigger logic:
```apex
List<Warranty_Claim__c> toProcess = new List<Warranty_Claim__c>();
for(Warranty_Claim__c claim : Trigger.new) {
    if(!RecursionControl.processedClaimIds.contains(claim.Id)) {
        toProcess.add(claim);
        RecursionControl.processedClaimIds.add(claim.Id); // Mark processed
    }
}
```

---

# 17. Trigger Handler Pattern

Enterprise architecture demands **One Trigger Per Object**, delegating logic to Handler classes.

**1. The Trigger (`WarrantyClaimTrigger.trigger`)**
```apex
trigger WarrantyClaimTrigger on Warranty_Claim__c (after insert, after update) {
    if (Trigger.isAfter && Trigger.isInsert) {
        WarrantyClaimTriggerHandler.handleAfterInsert(Trigger.new);
    }
    if (Trigger.isAfter && Trigger.isUpdate) {
        WarrantyClaimTriggerHandler.handleAfterUpdate(Trigger.newMap, Trigger.oldMap);
    }
}
```

**2. The Handler (`WarrantyClaimTriggerHandler.cls`)**
*Handles routing and context variable mapping.*

**3. The Service Layer (`WarrantyClaimService.cls`)**
*Handles the actual business logic and DML, completely decoupled from the Trigger context.*

---

# 18. Enterprise Trigger Frameworks

For massive orgs, simple handlers aren't enough. Architects use frameworks like **fflib Apex Enterprise Patterns**.

* **Domain Layer:** Encapsulates object-specific behavior (e.g., `WarrantyClaims.cls`).
* **Service Layer:** Exposes macro business processes (e.g., `ClaimProcessingService.approveClaim()`).
* **Unit of Work (UoW):** Optimizes DML by registering inserts/updates in memory and committing them all at the very end in a single database transaction, vastly reducing DML statements.

---

# 19. Real Project Scenarios (Automotive CRM)

1.  **Update Dealer Statistics:** `After Insert/Update/Delete` on `Vehicle_Registration__c`. Calculate total units sold and update `Dealer__c.Total_Registrations__c`.
2.  **Generate Service History:** `After Update` on `Work_Order__c`. If Status == 'Closed', insert a `Service_History__c` record attached to the `Vehicle__c`.
3.  **Synchronize SAP Records:** `After Update` on `Invoice__c`. If Status changes to 'Paid', enqueue a Queueable job to callout to SAP and update the financial ledger.
4.  **Maintain Spare Parts Inventory:** `After Insert` on `Work_Order_Line_Item__c`. Deduct the consumed quantity from `Parts_Inventory__c`.

---

# 20. Performance Optimization

* **Filter Early:** Iterate over `Trigger.new` and filter records that actually need processing into separate Lists/Maps *before* doing any queries or complex logic.
* **Map-Based Lookups:** Use `Map<Id, sObject>` heavily to associate parent records with child records in memory instead of relying on nested loops (which drain CPU time).
* **Avoid Unnecessary SOQL:** Check if a field value actually changed (`Trigger.new[i].Status__c != Trigger.old[i].Status__c`) before executing a query to fetch related records.

---

# 21. Common Mistakes

1.  **Using After Trigger instead of Before:** Doing `Trigger.new[0].Status = 'New';` in an After Trigger results in a "Record is Read-Only" runtime exception.
2.  **Hardcoded IDs:** Avoid `if(claim.RecordTypeId == '012500000009WXYZ')`. Use `Schema.SObjectType.Warranty_Claim__c.getRecordTypeInfosByName().get('Auto').getRecordTypeId()`.
3.  **Multiple Triggers on One Object:** Leads to unpredictable order of execution. Always use a single trigger dispatcher.

---

# 22. Debugging After Triggers

* **Developer Console & Debug Logs:** Set Apex Profiling to FINEST to trace CPU usage across complex After Trigger cascades.
* **Governor Limit Monitoring:** Use `Limits.getQueries()` and `Limits.getDMLStatements()` in your `System.debug()` statements to ensure loops aren't silently eating resources.
* **VS Code Replay Debugger:** Download a `.log` file and step through the After Trigger logic line-by-line to see exactly where a map lookup failed or a recursion loop started.

---

# 23. Interview Questions & Answers

### Beginner Questions
**Q: Can you update a field on `Trigger.new` in an After Update trigger?**
*A: No. The records are read-only at this stage. You will get an Apex runtime error. You must use Before Update for same-record field modifications.*

### Intermediate Questions
**Q: How do you identify which records had a specific field changed during an After Update trigger?**
*A: By comparing `Trigger.newMap.get(recordId).Field__c` with `Trigger.oldMap.get(recordId).Field__c`. If they differ, the field was changed.*

### Advanced Questions
**Q: Explain how to write a manual roll-up summary via trigger for a Lookup relationship.**
*A: On After Insert, Update, Delete, and Undelete of the child object, gather the Parent Ids. Query the parent records along with an aggregate query `[SELECT ParentId, SUM(Amount) FROM Child GROUP BY ParentId]`. Map the results and issue an update DML on the parents.*

### Architect-Level Questions
**Q: How does the Unit of Work (UoW) pattern solve governor limit issues in deep After Trigger cascades?**
*A: Instead of each Trigger Handler executing its own DML statements, they register the new/dirty records with a singleton UoW instance. At the end of the transaction, UoW executes exactly one `insert`, `update`, or `delete` statement per sObject type, ensuring you never hit the 150 DML limit regardless of trigger depth.*

---

# 24. Revision Summary

* **After Insert/Update/Delete/Undelete:** Used primarily for updating/creating related records and external integrations.
* **Record Access:** Record IDs and System fields (Formulas, Modstamps) are fully available.
* **Read-Only:** `Trigger.new` cannot be modified directly; explicit DML is required for any database changes.
* **Context Variables:** `Trigger.newMap` and `Trigger.oldMap` are essential for detecting field-level changes (deltas).
* **Bulkification:** Mandatory. Always use Sets/Maps to group data, one SOQL query to fetch data, and one DML statement to commit data.
* **Trigger Frameworks:** Always use a single Trigger per object and delegate logic to Handler and Service classes.
* **Recursion:** Use static Sets to track processed records and prevent infinite loops during cross-object updates.