# Future Methods – Asynchronous Processing

## 1. Introduction

Asynchronous Apex allows developers to execute code in the background, separate from the main execution thread. This prevents long-running operations from blocking the user interface and provides higher governor limits.

Future Methods are the simplest form of Asynchronous Apex. They are defined with the `@future` annotation and are placed in an execution queue to run when system resources become available. Salesforce provides Future Methods to handle processing that does not need to complete immediately, ensuring that the primary synchronous transaction remains fast and responsive.

Benefits of asynchronous processing include enhanced user experience, higher governor limits (like CPU time and heap size), and the ability to integrate with external systems without holding up the database lock. In an Automotive CRM context, a real-world use case is processing thousands of daily Warranty Claims or syncing Dealer data with an external SAP ERP system in the background.

---

## 2. What are Future Methods?

Future Methods are Apex methods that execute asynchronously. They are designed to run in their own separate thread and transaction context. 

When a Future Method is invoked, Salesforce does not execute it immediately. Instead, it places a request into an internal asynchronous queue. The method executes in the background when the Salesforce platform determines that sufficient resources are available. This non-blocking execution model ensures that the synchronous thread (e.g., a user saving a record) completes without waiting for the future method to finish.

Because Future Methods run in a separate transaction, they get their own fresh set of governor limits independent of the transaction that invoked them.

---

## 3. Why Do We Need Future Methods?

Future Methods are essential in Salesforce development for several enterprise scenarios:

* **Long-running operations:** Offloading complex calculations or data processing that would otherwise cause CPU timeouts.
* **External API integrations:** Making web service callouts (REST/SOAP) from triggers, which is strictly prohibited in synchronous Apex.
* **Sending notifications:** Dispatching mass emails or SMS alerts without slowing down record creation.
* **Processing large datasets:** Updating related records across the database asynchronously to avoid locking issues.
* **Improving user experience:** Returning control to the user immediately while heavy processing completes in the background.

In an enterprise Automotive CRM, when a Service Technician logs a `Work_Order__c`, you might use a Future Method to recalculate the inventory levels of spare parts at the Service Center without making the technician wait for the calculation to finish.

---

## 4. Understanding @future Annotation

The `@future` annotation is used to identify a method that runs asynchronously.

```apex
/**
 * @future: Signals the compiler to place the method in the async queue.
 * public static: Future methods MUST be static (no instance state).
 * void: Future methods MUST return void (no synchronous thread to catch a return).
 */
@future
public static void processData() {
    // Background execution logic
}
```

---

## 5. How Future Methods Work Internally

Salesforce handles Future Methods using a sophisticated queueing and worker-thread architecture. 

When a Future Method is called, it does not immediately queue. The platform waits for the current synchronous transaction to commit successfully. This is the **commit dependency** rule. If the synchronous transaction rolls back due to an error, the future method is never queued. 

Once the transaction commits, a message containing the method reference and its parameters is placed into the asynchronous message queue. Background worker threads continuously poll this queue. When a thread becomes available, it dequeues the message and executes the method in a completely new, separate transaction context.

---

## 6. Future Method Parameters

Future methods have strict parameter requirements to ensure data integrity.

```apex
/**
 * ALLOWED: Primitive types (String, Id, Integer) or collections of primitives.
 * NOT ALLOWED: sObjects (like Account or Vehicle__c).
 * WHY: Execution time is not guaranteed. Passing primitive IDs ensures the 
 * method queries the freshest, most up-to-date data when it finally runs.
 */
@future
public static void updateAccounts(List<Id> accountIds) {
    List<Account> accountsToUpdate = [SELECT Id, Name FROM Account WHERE Id IN :accountIds];
    // Processing logic
}
```

---

## 7. Future Callouts

By default, Future Methods are not allowed to make external API callouts. You must explicitly declare this intent.

```apex
/**
 * @description Future Methods should live in a top-level Service or Integration class.
 * This adheres to Separation of Concerns, keeping Triggers clean.
 */
public class WarrantyIntegrationService {

    /**
     * @future flags this method to run asynchronously in a separate thread.
     * (callout=true) is explicitly required by Salesforce to permit outbound HTTP requests.
     */
    @future(callout=true)
    public static void sendWarrantyToSAP(Set<Id> claimIds) {
        
        // 1. QUERY THE LATEST DATA using the passed-in primitive IDs
        List<Warranty_Claim__c> claims = [
            SELECT Id, Claim_Amount__c, Status__c 
            FROM Warranty_Claim__c 
            WHERE Id IN :claimIds
        ];
        
        // 2. PREPARE THE HTTP CALLOUT
        Http http = new Http();
        HttpRequest request = new HttpRequest();
        request.setEndpoint('callout:SAP_Credentials/api/warranty'); // Use Named Credentials
        request.setMethod('POST');
        request.setHeader('Content-Type', 'application/json');
        request.setBody(JSON.serialize(claims));
        
        // 3. EXECUTE AND HANDLE ERRORS
        // Wrapping the callout in a try-catch block is an architectural requirement.
        try {
            HttpResponse response = http.send(request);
            if (response.getStatusCode() == 200) {
                System.debug('Successfully synced claims to SAP.');
            }
        } catch (Exception e) {
            // Log to a Custom Object or Platform Event in production
            System.debug('Exception occurred during SAP Callout: ' + e.getMessage());
        }
    }
}
```

---

## 8. Future Methods vs Synchronous Processing

| Feature | Synchronous Apex | Future Methods |
| :--- | :--- | :--- |
| **Execution Timing** | Immediate | Background (Delayed) |
| **User Experience** | User waits for completion | Non-blocking (Immediate UI return) |
| **Transaction Context** | Same transaction | Separate transaction |
| **Governor Limits** | Lower (e.g., 10s CPU, 6MB Heap) | Higher (e.g., 60s CPU, 12MB Heap) |
| **Callouts from Triggers** | Not Allowed | Allowed with `callout=true` |

---

## 9. Future Methods vs Queueable Apex

Modern Salesforce architecture heavily favors Queueable Apex.

| Feature | Future Methods | Queueable Apex |
| :--- | :--- | :--- |
| **Parameters** | Primitives only | Complex objects and sObjects allowed |
| **Chaining** | Not Allowed | Allowed (Can enqueue another job) |
| **Monitoring** | Basic (AsyncApexJob) | Advanced (Returns a queryable Job ID) |
| **Method Signature** | `@future` static void | Implements `Queueable` interface |

Queueable Apex is generally preferred because it provides the simplicity of Future Methods combined with the power of Batch Apex. It allows chaining jobs for sequential processing, passing non-primitive types, and tracking the job via the returned `AsyncApexJob` ID. Future Methods remain relevant for simple trigger callouts and legacy systems.

---

## 10. Future Methods vs Batch Apex

| Feature | Future Methods | Batch Apex |
| :--- | :--- | :--- |
| **Data Volume** | Small to Medium | Large Data Volumes (up to 50 million) |
| **Statefulness** | Stateless | Can maintain state with `Database.Stateful` |
| **Scheduling** | Cannot be scheduled | Can be scheduled via `Schedulable` |
| **Execution Size** | Executes all at once | Executes in chunks (default 200 records) |

---

## 11. Governor Limits

Future Methods provide extended asynchronous governor limits, but are governed by strict platform rules.

| Limit Type | Synchronous Limit | Asynchronous Limit |
| :--- | :--- | :--- |
| **Total Heap Size** | 6 MB | 12 MB |
| **Maximum CPU Time** | 10,000 ms (10s) | 60,000 ms (60s) |
| **Future Calls per Transaction** | N/A | 50 calls |
| **Callouts** | 100 | 100 |

**Scalability Considerations:** The maximum number of asynchronous Apex executions per 24-hour period is 250,000 or (User Licenses × 200), whichever is greater. Future Methods must be highly bulkified to avoid burning through the 24-hour async limit.

---

## 12. Limitations of Future Methods

* **No sObject parameters:** Forces a SOQL query inside the method.
* **No chaining:** Cannot call a Future Method from another Future Method.
* **No Job ID:** Difficult to programmatically track completion or failure.
* **No return values:** You cannot directly pass results back to the calling process.
* **Execution order not guaranteed:** Queued methods execute when resources are free, not strictly in order.

---

## 13. Bulkification Best Practices

Never design a Future Method to process a single record.

```apex
// ANTI-PATTERN: Fails if a trigger passes multiple records (hits 50 call limit)
@future
public static void processSingleVehicle(Id vehicleId) { ... }

// BEST PRACTICE: Bulkified approach
@future
public static void processMultipleVehicles(Set<Id> vehicleIds) {
    List<Vehicle__c> vehicles = [SELECT Id, VIN__c FROM Vehicle__c WHERE Id IN :vehicleIds];
    for (Vehicle__c v : vehicles) {
        // Process each vehicle
    }
}
```

---

## 14. Error Handling

Because Future Methods run in the background, robust error handling is required to avoid silent failures.

```apex
@future
public static void syncDealers(Set<Id> dealerIds) {
    try {
        // Sync logic
    } catch (Exception e) {
        // Publish a Platform Event so admins are alerted to background failures
        Error_Log__e errorEvent = new Error_Log__e(
            Error_Message__c = e.getMessage(),
            Context__c = 'FutureMethod: syncDealers'
        );
        EventBus.publish(errorEvent);
    }
}
```

---

## 15. Testing Future Methods

Testing asynchronous code requires the use of `Test.startTest()` and `Test.stopTest()`.

```apex
@isTest
private class WarrantyServiceTest {
    
    @isTest
    static void testWarrantyCallout() {
        Warranty_Claim__c claim = new Warranty_Claim__c(Claim_Amount__c = 500);
        insert claim;
        Set<Id> claimIds = new Set<Id>{claim.Id};
        
        Test.setMock(HttpCalloutMock.class, new SAPMockHttpResponseGenerator());
        
        Test.startTest();
        // Invoke Future Method
        WarrantyIntegrationService.sendWarrantyToSAP(claimIds);
        // Test.stopTest() forces queued asynchronous code to execute synchronously
        Test.stopTest();
        
        Warranty_Claim__c updatedClaim = [SELECT Status__c FROM Warranty_Claim__c WHERE Id = :claim.Id];
        System.assertEquals('Sent to SAP', updatedClaim.Status__c);
    }
}
```

---

## 16. Debugging Future Methods

* **Developer Console:** Set Trace Flags for the `Automated Process` entity.
* **Apex Jobs:** Navigate to Setup -> Apex Jobs to view statuses (Queued, Completed, Failed).
* **Debug Logs:** Future methods generate their own distinct debug log, separate from the transaction that initiated them.

---

## 17. Enterprise Design Patterns

Use the **Service Layer Pattern**. Do not put `@future` annotations directly on trigger handlers.

```apex
public class WarrantyIntegrationService {
    
    @future(callout=true)
    public static void executeSAPIntegrationAsync(Set<Id> claimIds) {
        // Delegates to a synchronous worker method for reusability
        executeSAPIntegration(claimIds);
    }
    
    public static void executeSAPIntegration(Set<Id> claimIds) {
        // Actual HTTP logic here. Can be called by Batch or Future.
    }
}
```

---

## 18. Real Project Scenarios (Automotive CRM)

* **Sending Warranty Claim Data to SAP:** Approving a claim triggers a future method that formats a JSON payload and makes an HTTP callout to SAP.
* **Customer SMS Notifications:** A future method makes an API callout to Twilio when a vehicle is "Ready for Pickup", ensuring the service advisor's UI isn't slowed by API latency.
* **Dealer Notification Emails:** Calculating complex dealer allocation matrices and sending summary emails in the background.

---

## 19. Performance Optimization

* **Reduce Future Calls:** Combine multiple background tasks into a single Queueable job.
* **Efficient Database Operations:** Filter SOQL strictly by the provided `List<Id>`.
* **Avoid Over-Queuing:** If an Automotive data-load creates 50,000 records, triggering 50,000 future methods will crash the org. Use Batch Apex.

---

## 20. Common Mistakes

* **Calling future methods inside loops:** Causes `System.LimitException: Too many future calls: 51`.
* **Passing sObjects instead of IDs:** Causes `System.AsyncException`.
* **Ignoring Bulkification:** Fails during mass data loader operations.
* **No Error Handling:** Causes integrations to fail silently.

---

## 21. Best Practices Checklist

* ✅ Use Sets/Lists of IDs instead of passing entire sObjects.
* ✅ Bulkify future methods to handle mass records.
* ✅ Use `@future(callout=true)` exclusively for API integrations.
* ✅ Handle exceptions properly and log them to custom objects/Platform Events.
* ✅ Prefer Queueable Apex for new enterprise implementations.
* ✅ Avoid invoking future methods from loops inside triggers.
* ✅ Write proper unit tests utilizing `Test.startTest()` and `Test.stopTest()`.

---

## 22. Interview Questions & Answers

### Beginner Questions
**Q: What is a Future Method?** A: An Apex method that runs asynchronously in the background in a separate transaction. Declared with `@future`.

### Intermediate Questions
**Q: Why can't we pass sObjects to a Future Method?** A: Execution time is not guaranteed. Passing an sObject risks stale data. Passing IDs ensures the method queries the freshest data.

### Advanced Questions
**Q: How do you make an external HTTP callout from an Apex Trigger?** A: Triggers do not support synchronous callouts. Gather record IDs, pass them to a `@future(callout=true)` method, and perform the callout there.

### Architect-Level Questions
**Q: If you need to chain multiple background processes sequentially, would you use Future Methods?** A: No. Future Methods cannot invoke other Future Methods. Queueable Apex is the architectural standard for sequential processing.

---

## 23. Revision Summary

* **Future Methods:** Run asynchronously, separate transaction, own limits.
* **@future Annotation:** Must be static, return void, accept only primitives.
* **Future Callouts:** Requires `@future(callout=true)`.
* **Governor Limits:** Higher limits (60s CPU, 12MB Heap), limit of 50 future calls per transaction.
* **Limitations:** No sObjects, no chaining, no Job ID tracking.
* **Testing:** Use `Test.startTest()` and `Test.stopTest()` to force synchronous execution.
* **Bulkification:** Always pass `List<Id>` or `Set<Id>`.
* **Queueable Comparison:** Queueable is the modern alternative allowing complex objects and chaining.