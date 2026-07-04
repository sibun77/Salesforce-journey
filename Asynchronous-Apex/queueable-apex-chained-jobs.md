# Queueable Apex Chained Jobs

## 1. Introduction

**Asynchronous Apex** is a core Salesforce feature that allows processes to run in a separate thread, at a later time, without blocking the user's current transaction. This is critical for maintaining platform responsiveness, executing long-running operations, and scaling enterprise applications.

**Queueable Apex**, introduced in Winter '15, represents a significant evolution in Asynchronous Apex. It combines the simplicity of `@future` methods with the robust power of Batch Apex. By implementing the `System.Queueable` interface, developers can run complex background processes while passing complex objects (like sObjects or custom Apex types), something `@future` methods cannot do. 

**Why Salesforce Introduced Queueable Apex:**
Historically, developers relied heavily on `@future` methods for quick asynchronous tasks. However, `@future` methods only accept primitive data types, do not return a Job ID for monitoring, and cannot be chained together. Queueable Apex was introduced to bridge this gap, providing a modern, object-oriented approach to async processing.

**Real-world Business Use Case (Automotive CRM):**
Consider an Automotive CRM where a dealership submits a **Warranty Claim**. Upon approval, several distinct, heavy processes must occur:
1. Update the Vehicle's Service History.
2. Generate an Invoice for the dealership.
3. Sync the approved claim to an external SAP ERP system.

Using Queueable Apex, these steps can be separated into distinct asynchronous transactions and chained together sequentially, ensuring high performance, proper governor limit allocation, and fault isolation.

---

## 2. What is Queueable Apex?

Queueable Apex is an interface provided by Salesforce that allows you to submit jobs for asynchronous processing. 

### Deep Dive:
* **Definition:** An Apex class that implements the `Queueable` interface and defines an `execute` method.
* **Purpose:** To offload heavy or non-urgent processing to the background, freeing up the primary transaction (e.g., a trigger or a Lightning Web Component call).
* **Background Processing:** The Salesforce asynchronous queue manager handles Queueable jobs. They execute when system resources become available, which is usually almost instantaneous but can be delayed during peak platform loads.
* **Transaction Separation:** Every Queueable job runs in its own distinct transaction. This means it receives a fresh set of governor limits (e.g., 12MB heap size, 60 seconds CPU time) separate from the thread that enqueued it.
* **Non-blocking Execution:** The user experience remains uninterrupted. A user can click "Approve Claim," the heavy processing is queued, and the user immediately sees a success message on the UI.

---

## 3. Why Do We Need Queueable Apex?

Queueable Apex is the preferred async pattern in modern Salesforce architecture for several reasons:

1.  **Complex Async Processing:** Unlike `@future`, Queueable classes can accept instance variables, including lists of sObjects or custom wrapper classes.
2.  **External API Integrations:** Making HTTP callouts from triggers is prohibited. Queueable Apex allows callouts asynchronously while holding complex state.
3.  **Chained Business Processes:** A Queueable job can enqueue another Queueable job, allowing complex workflows to run sequentially with fresh limits.
4.  **Large Record Processing:** While Batch Apex is best for massive data volumes (millions of records), Queueable is perfect for medium volumes (e.g., processing 1,000 claim lines) that exceed synchronous limits.
5.  **Better Monitoring than Future Methods:** Submitting a Queueable job returns an `AsyncApexJob` ID, allowing developers and admins to track its status natively or programmatically.

---

## 4. Queueable Interface

To utilize Queueable Apex, a class must implement the `System.Queueable` interface. 

### Example:

```apex
public class WarrantyClaimProcessor implements Queueable {
    
    // Instance variable to hold complex data
    private List<Case> approvedClaims;
    
    // Constructor to pass state into the Queueable job
    public WarrantyClaimProcessor(List<Case> claims) {
        this.approvedClaims = claims;
    }

    // Mandatory execute method
    public void execute(QueueableContext context) {
        // Business logic to process claims
        for(Case claim : approvedClaims) {
            claim.Status = 'Processed';
        }
        update approvedClaims;
    }
}
```

### Line-by-Line Explanation:
* `public class WarrantyClaimProcessor implements Queueable {`: Defines the class and implements the `Queueable` interface.
* `private List<Case> approvedClaims;`: Declares a stateful instance variable. Because this is a class instance, it can store complex types like Lists of sObjects.
* `public WarrantyClaimProcessor(List<Case> claims) { ... }`: The constructor. It initializes the class state with the data passed from the calling transaction (e.g., a trigger).
* `public void execute(QueueableContext context) {`: The mandatory method required by the `Queueable` interface. This is the entry point when the asynchronous job runs.
* `QueueableContext context`: An object provided by Salesforce containing the execution context, most notably the Job ID.
* `for(Case claim : approvedClaims) { ... }`: Standard Apex logic executing within the new asynchronous transaction.
* `update approvedClaims;`: Performs DML. If this fails, only this async transaction rolls back.

---

## 5. System.enqueueJob()

To add your Queueable class to the Salesforce execution queue, you use the `System.enqueueJob()` method.

### Example:

```apex
// Assume this is inside an Apex Trigger or Controller
List<Case> claimsToProcess = [SELECT Id, Status FROM Case WHERE Status = 'Approved'];

// Create an instance of the Queueable class
WarrantyClaimProcessor job = new WarrantyClaimProcessor(claimsToProcess);

// Enqueue the job and capture the ID
Id jobId = System.enqueueJob(job);

System.debug('Enqueued Queueable Job ID: ' + jobId);
```

### Deep Dive:
* **Job Creation:** You must instantiate the class using the `new` keyword, passing in any required constructor arguments.
* **System.enqueueJob():** This system method places the job in the async queue.
* **Returned Job ID:** It returns an 18-character `Id` corresponding to an `AsyncApexJob` record.
* **Monitoring Capabilities:** You can store this ID in a custom log object, return it to an LWC to poll for completion, or query it later to check for success or failure.

---

## 6. Queueable Context

The `execute` method always receives a `QueueableContext` parameter. 

```apex
public void execute(QueueableContext context) {
    Id currentJobId = context.getJobId();
    System.debug('Currently executing Job ID: ' + currentJobId);
    
    // Custom logging framework integration
    AppLogger.logAsyncStart(currentJobId, 'WarrantyClaimProcessor');
}
```

### Deep Dive:
* **QueueableContext Interface:** Contains information about the job environment.
* **context.getJobId():** Retrieves the ID of the currently running `AsyncApexJob`.
* **Tracking and Logging:** This is critical for enterprise architectures. If a job fails, logging the `context.getJobId()` alongside the error message allows administrators to correlate the specific error in your custom logs with the Salesforce platform's native Apex Jobs page.

---

## 7. Chained Queueable Jobs

Job Chaining is the process of enqueuing a new Queueable job from within the `execute` method of an actively running Queueable job. 

### Why Chaining is Needed:
* **Sequential Processing:** Ensuring Step B only happens after Step A completes successfully. 
* **Transaction Separation:** If Step A consumes 90% of the CPU limits, executing Step B in the same transaction would cause a `LimitException`. Chaining offloads Step B to a brand-new transaction with 100% fresh limits.
* **Enterprise Use Cases:** 1. Update Warranty Claim (Job 1).
    2. Generate PDF Invoice (Job 2).
    3. Send Invoice to SAP via HTTP Callout (Job 3).

---

## 8. Implementing Chained Jobs

### Production-Quality Example:

```apex
public class ClaimApprovalJob implements Queueable {
    
    private List<Id> claimIds;
    
    public ClaimApprovalJob(List<Id> claimIds) {
        this.claimIds = claimIds;
    }

    public void execute(QueueableContext context) {
        // Step 1: Process Claim logic
        List<Case> claimsToUpdate = new List<Case>();
        for(Id claimId : claimIds) {
            claimsToUpdate.add(new Case(Id = claimId, Status = 'Processed'));
        }
        update claimsToUpdate;
        
        // Step 2: Chain the next job (SAP Integration)
        if(!Test.isRunningTest()) {
            Id chainedJobId = System.enqueueJob(
                new SAPInvoiceSyncJob(this.claimIds)
            );
            System.debug('Chained SAP Sync Job: ' + chainedJobId);
        }
    }
}
```

### Line-by-Line Breakdown:
* `public class ClaimApprovalJob implements Queueable`: The first job in the sequence.
* `update claimsToUpdate;`: Executes the DML for the first step.
* `if(!Test.isRunningTest())`: **Crucial Architect insight.** Salesforce unit tests do not allow job chaining natively. If a Queueable enqueues another Queueable during a test context, a `System.AsyncException: Maximum stack depth has been reached` occurs. We bypass the chain in test mode (or use a mock framework).
* `System.enqueueJob(new SAPInvoiceSyncJob(this.claimIds));`: Instantiates the second job in the sequence, passing the same context data (`claimIds`), and enqueues it.

---

## 9. Queueable Callouts

To make HTTP REST or SOAP callouts from a Queueable job, the class must explicitly implement the `Database.AllowsCallouts` marker interface.

### Production-Quality Example (SAP Integration):

```apex
public class SAPInvoiceSyncJob implements Queueable, Database.AllowsCallouts {
    
    private List<Id> claimIds;
    
    public SAPInvoiceSyncJob(List<Id> claimIds) {
        this.claimIds = claimIds;
    }

    public void execute(QueueableContext context) {
        // 1. Prepare Callout
        Http http = new Http();
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:SAP_Named_Credential/api/invoices'); // Use Named Credentials for Auth!
        req.setMethod('POST');
        req.setHeader('Content-Type', 'application/json');
        
        // 2. Build Payload
        String jsonBody = JSON.serialize(new Map<String, Object>{
            'claimIds' => this.claimIds,
            'source' => 'Salesforce_CRM'
        });
        req.setBody(jsonBody);
        
        // 3. Execute Callout
        try {
            HttpResponse res = http.send(req);
            if (res.getStatusCode() == 201) {
                System.debug('SAP Sync Successful: ' + res.getBody());
            } else {
                // Log failure via Platform Event
                EventBus.publish(new Error_Log__e(
                    Job_Id__c = context.getJobId(),
                    Message__c = 'SAP Callout Failed: ' + res.getStatus()
                ));
            }
        } catch(Exception e) {
            System.debug('Callout Exception: ' + e.getMessage());
        }
    }
}
```

### Deep Dive:
* **`implements Queueable, Database.AllowsCallouts`**: Without the second interface, the platform will throw an exception if an `Http.send()` is attempted.
* **Named Credentials**: Always use Named Credentials (`callout:SAP_Named_Credential`) for integrations to handle authentication (OAuth, Basic) securely, avoiding hardcoded tokens.
* **Separation**: By placing this in a chained job, the initial DML (Claim Update) is already committed and safe. If the SAP callout fails, it does not roll back the Claim approval.

---

## 10. Queueable vs Future Methods

Why Queueable Apex is preferred in modern development:

| Feature | Future Methods (`@future`) | Queueable Apex (`Queueable`) |
| :--- | :--- | :--- |
| **Parameter Types** | Primitive types, collections of primitives. | Complex objects, sObject Lists, Custom Types. |
| **Job Chaining** | ❌ Not Allowed (Cannot call `@future` from `@future`). | ✅ Supported (Can enqueue a Queueable from a Queueable). |
| **Monitoring** | ❌ Returns `void`. Cannot be directly tracked. | ✅ Returns `AsyncApexJob` ID for native tracking. |
| **Execution Order** | Not guaranteed. | Handled sequentially when chained. |
| **Best Use Case** | Fire-and-forget simple async logic. | Enterprise async orchestration, complex payloads. |

> **Architect Insight:** The only reason to use `@future` today is to temporarily bypass the "Mixed DML" error in a small trigger where setting up a Queueable class feels like overkill. Otherwise, Queueable is standard.

---

## 11. Queueable vs Batch Apex

| Feature | Batch Apex (`Database.Batchable`) | Queueable Apex (`Queueable`) |
| :--- | :--- | :--- |
| **Primary Purpose** | Massive data processing (Millions of records). | Complex async logic, medium data sets, integrations. |
| **Execution Limits** | Up to 50 million records via `Database.QueryLocator`. | Standard async limits (e.g., query up to 50,000 records). |
| **Interface Complexity** | High (`start`, `execute`, `finish` methods required). | Low (Only `execute` method required). |
| **Chaining** | Batch can call Batch (from `finish`), but slow. | Native, rapid chaining. |
| **Scheduling** | Easily schedulable via `System.scheduleBatch`. | Can be scheduled, but requires implementing `Schedulable`. |

> **Architect Insight:** Use Batch when you need to process large volumes of data overnight. Use Queueable for user-initiated asynchronous actions that need to happen quickly in the background.

---

## 12. Governor Limits

Queueable Apex operates under Asynchronous Governor Limits, which are generally double the synchronous limits.

| Limit Type | Limit Allocation |
| :--- | :--- |
| **Jobs Added to Queue (per transaction)** | 50 (Sync) / 1 (Async/Chained) |
| **Chaining Depth** | Unlimited (Production) / 5 (Developer/Trial orgs) |
| **Total SOQL Queries** | 100 (Sync) / 200 (Async) |
| **Total DML Statements** | 150 |
| **Maximum CPU Time** | 10,000 ms (10 seconds sync) / 60,000 ms (60 seconds async) |
| **Maximum Heap Size** | 6 MB (Sync) / 12 MB (Async) |
| **Total HTTP Callouts** | 100 (max 120 seconds total timeout) |
| **Daily Async Apex Limit** | 250,000 or (Number of User Licenses x 200), whichever is greater. |

### Scalability Considerations:
While chaining is unlimited in production, an infinite loop of chained jobs will rapidly consume your 24-hour Daily Async Apex Limit, bringing down the org. Always include termination logic.

---

## 13. Transaction Separation

A fundamental concept for architects is understanding that **each Queueable job is an entirely separate database transaction.**

1.  **Independent Limits:** Job A queries 199 times. It chains Job B. Job B starts with 0 queries used.
2.  **Failure Isolation:** If Job A executes DML successfully and chains Job B, but Job B throws a `NullPointerException`, Job B rolls back. **Job A's database changes remain committed.**
3.  **Flow Design:** Because of this separation, state must be explicitly passed from Job A to Job B via the constructor. Job B cannot access variables from Job A's memory.

---

## 14. Error Handling

Because Queueable jobs run silently in the background, robust error handling is critical. If a job fails silently, business processes halt.

### Architect-Level Pattern: Platform Events
Do not just use `System.debug()`. Create a custom object or use Platform Events to log errors for IT monitoring.

```apex
public void execute(QueueableContext context) {
    Savepoint sp = Database.setSavepoint();
    try {
        // Risky DML or Callout
        insert warrantyClaims;
    } catch (Exception e) {
        Database.rollback(sp);
        
        // Publish error to Event Bus for monitoring
        App_Error__e errorEvent = new App_Error__e(
            Job_Id__c = context.getJobId(),
            Error_Message__c = e.getMessage(),
            Stack_Trace__c = e.getStackTraceString(),
            Class_Name__c = 'WarrantyClaimProcessor'
        );
        EventBus.publish(errorEvent);
    }
}
```

---

## 15. Monitoring Queueable Jobs

### Via SOQL:
You can query the `AsyncApexJob` object programmatically to check the status.

```apex
List<AsyncApexJob> jobs = [
    SELECT Id, Status, NumberOfErrors, ExtendedStatus 
    FROM AsyncApexJob 
    WHERE JobType = 'Queueable' 
    ORDER BY CreatedDate DESC LIMIT 10
];
```

### Via UI:
Navigate to **Setup → Environments → Jobs → Apex Jobs**. Here, admins can see "Completed", "Queued", "Preparing", or "Failed" statuses, along with any unhandled exception messages.

---

## 16. Testing Queueable Apex

Testing async code in Salesforce requires strict adherence to `Test.startTest()` and `Test.stopTest()`.

### Complete Example:

```apex
@isTest
private class WarrantyClaimProcessorTest {
    
    @testSetup
    static void setupData() {
        insert new Case(Subject = 'Engine Failure', Status = 'Approved');
    }

    @isTest
    static void testQueueableExecution() {
        List<Case> testClaims = [SELECT Id, Status FROM Case];
        
        Test.startTest();
        // The Queueable job is enqueued
        System.enqueueJob(new WarrantyClaimProcessor(testClaims));
        // Test.stopTest() forces all asynchronous processes to execute synchronously
        Test.stopTest(); 
        
        // Assertions MUST happen after Test.stopTest()
        Case updatedClaim = [SELECT Status FROM Case WHERE Id = :testClaims[0].Id];
        System.assertEquals('Processed', updatedClaim.Status, 'The Queueable job should have updated the status.');
    }
}
```

### Best Practices:
* `Test.stopTest()` forces execution.
* **Chaining Limits in Tests:** You can only enqueue *one* asynchronous job inside a test context. If `Job1` enqueues `Job2`, the test will fail. You must use `Test.isRunningTest()` to block the chain in `Job1` and test `Job2` in a separate test method.

---

## 17. Bulkification Best Practices

Just like triggers, Queueable classes must be bulkified. Although a Queueable job might be initiated by a single user action, it might process data generated by a Bulk API upload.

* **Process Collections:** Always pass `List<sObject>`, `Map<Id, sObject>`, or `Set<Id>` into the constructor. Do not pass single Ids.
* **Efficient SOQL:** Query everything needed outside of loops.
* **Efficient DML:** Use lists to execute DML once at the end of the `execute` method.

---

## 18. Enterprise Design Patterns

### The Async Service Layer
Do not put business logic directly inside the Queueable `execute` method. Use the Queueable class simply as an asynchronous router that calls a centralized Service class.

```apex
public class WarrantyQueueable implements Queueable {
    private Set<Id> claimIds;
    
    public WarrantyQueueable(Set<Id> claimIds) {
        this.claimIds = claimIds;
    }

    public void execute(QueueableContext context) {
        // Delegate to Service Layer
        WarrantyService.processClaims(this.claimIds);
    }
}
```
*Benefits:* The same `WarrantyService.processClaims()` logic can now be called synchronously from a Batch job, an LWC controller, or asynchronously from a Queueable without duplicating code.

---

## 19. Real Project Scenarios (Automotive CRM)

### Scenario 1: Warranty Claim → Invoice Generation → SAP Sync
**Requirement:** When a Warranty Claim is approved, the dealer must receive a PDF invoice, and the financial data must sync to SAP.
**Solution:**
1. Trigger calls `Queueable 1` (Invoice Data Generation - heavy DML).
2. `Queueable 1` chains `Queueable 2` (PDF Generation - heavy CPU).
3. `Queueable 2` chains `Queueable 3` (SAP HTTP Callout - network wait time).
**Why:** Isolates CPU, DML, and Callout limits entirely.

### Scenario 2: Spare Parts Inventory Synchronization
**Requirement:** Nightly sync of 5,000 spare parts from a warehouse API. 
**Solution:** A Schedulable class enqueues a Queueable job. The Queueable pulls data from the API, upserts the inventory records, and chains itself (re-enqueues) if pagination tokens exist indicating more data to fetch, ensuring no single transaction hits the 120-second callout timeout.

---

## 20. Performance Optimization

* **Reducing Heap Usage:** If passing a List of 10,000 sObjects exceeds heap limits, pass a `Set<Id>` instead and query the records inside the `execute` method (which gives you the larger 12MB async heap).
* **Efficient Chaining:** Do not chain a job for a single record update. Aggregate records into a list and chain one bulkified job.
* **Callout Performance:** If making multiple callouts in a Queueable, parallelize them if possible using `Continuation` (if via UI) or strictly monitor the 120-second total callout limit.

---

## 21. Common Mistakes

| Mistake | Consequence | Solution |
| :--- | :--- | :--- |
| **`enqueueJob` inside a `for` loop** | `LimitException: Too many queueable jobs added to the queue: 51` | Collect data in a List/Map, then enqueue *one* job passing the collection outside the loop. |
| **Infinite Job Chaining** | Consumes Daily Async limits, causes Org disruption. | Implement a recursion counter or strict boolean flags in the database before chaining. |
| **Uncaught Exceptions** | Job fails silently, business logic stops, user is unaware. | Implement robust `try/catch` and log errors to a custom object/Platform Event. |
| **Hardcoded API Endpoints** | Deployments fail, sandbox test data goes to production endpoints. | Always use **Named Credentials**. |

---

## 22. Best Practices Checklist

* [x] **Use Queueable Apex instead of Future Methods** for new implementations requiring async processing.
* [x] **Use Job IDs for monitoring** and store them in custom log records for easier debugging.
* [x] **Chain jobs responsibly:** Ensure chaining has a definitive end condition to avoid infinite loops.
* [x] **Handle exceptions properly:** Wrap critical logic in try/catch and log to custom objects/Platform Events.
* [x] **Bulkify processing:** Accept Lists/Sets in constructors, not single records.
* [x] **Write proper unit tests:** Use `Test.isRunningTest()` to bypass chaining limits in tests.
* [x] **Separate business logic:** Delegate logic from the `execute` method to an Enterprise Service Layer class.
* [x] **Avoid enqueueing jobs inside loops:** Ensure `System.enqueueJob()` is called only once per trigger/transaction context.

---

## 23. Interview Questions & Answers

### Beginner Questions
**Q: What is the main difference between `@future` and Queueable Apex?**
**A:** Queueable Apex can take non-primitive data types (like sObjects or custom classes) as parameters, returns a Job ID for monitoring, and allows chaining. `@future` methods only take primitives and cannot be monitored or chained natively.

### Intermediate Questions
**Q: How do you make an HTTP callout from a Queueable job?**
**A:** The Queueable class must implement both the `Queueable` interface and the `Database.AllowsCallouts` marker interface.

**Q: Can you chain a Queueable job from a Batch job?**
**A:** Yes, you can enqueue a Queueable job from the `finish` method of a Batch class, or even from the `execute` method (though subject to the 1 async job per transaction limit).

### Advanced Questions
**Q: How do you test chained Queueable jobs?**
**A:** Salesforce restricts asynchronous testing to a single execution depth. If Job A chains Job B, you will hit a maximum stack depth limit in tests. You must use `!Test.isRunningTest()` in Job A to prevent chaining, and test Job B independently in its own `@isTest` method.

### Architect-Level Questions
**Q: You have a requirement to process 2,000 records, perform heavy calculations, update them, and send the results to an external ERP. Explain your architecture.**
**A:** I would use Chained Queueable Apex. 
1. Job A accepts a `Set<Id>` of the 2,000 records. It queries them (leveraging the 12MB async heap), performs calculations, executes the DML, and chains Job B.
2. Job B accepts the same `Set<Id>`, implements `Database.AllowsCallouts`, formats the JSON payload, and executes the REST callout using a Named Credential. 
This architecture separates the DML limits from the Callout limits and isolates the transaction boundaries, ensuring data integrity if the ERP is offline.

---

## 24. Revision Summary

* **Queueable Apex:** Object-oriented async processing (Winter '15). Combines the best of `@future` and Batch.
* **System.enqueueJob():** Adds the class to the async queue and returns an `AsyncApexJob` Id.
* **QueueableContext:** Interface providing the `getJobId()` method inside the execution context.
* **Chained Jobs:** Enqueueing a new job from the `execute` method of an existing job. Infinite in production, limits apply in Dev orgs.
* **Queueable Callouts:** Requires `Database.AllowsCallouts`.
* **Governor Limits:** Async limits apply (higher Heap and CPU). Only 1 chained job per transaction (50 in sync transactions).
* **Testing:** Use `Test.startTest()` and `stopTest()`. Bypass chaining in tests.
* **Monitoring:** Use `AsyncApexJob` object or Apex Jobs UI.
* **Error Handling:** Use `try/catch` and log to custom objects/Platform Events to prevent silent failures.
* **Best Practices:** Pass Collections, delegate logic to Service layers, avoid `enqueueJob` in loops.