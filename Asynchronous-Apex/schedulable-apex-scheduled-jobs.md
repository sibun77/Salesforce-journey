# Schedulable Apex Scheduled Jobs

## 1. Introduction

**Asynchronous Apex** is a suite of features in Salesforce that allows you to run processes in the background, independent of the current transaction. This prevents blocking the user interface, avoids long-running synchronous governor limits, and improves system performance.

**Schedulable Apex** is a specific asynchronous feature that allows developers to execute Apex classes at specified times. Instead of relying on a user clicking a button or a record being updated, Schedulable Apex runs automatically based on a predefined schedule (e.g., daily, weekly, hourly).

### Why Salesforce Provides Scheduled Apex
Salesforce is a multi-tenant environment. To ensure fair resource allocation, background processing must be heavily regulated. Schedulable Apex provides a native, governor-limit-compliant way to automate recurring tasks without external middleware or manual intervention. 

### Manual vs. Scheduled Execution
* **Manual Execution:** Triggered by user interaction (button click, UI action). Executes immediately (synchronous) or is enqueued immediately (async).
* **Scheduled Execution:** Triggered by the system clock based on a CRON expression. Executes at a predictable, recurring time.

**Real-world Business Use Case:** In an Automotive CRM, dealerships submit warranty claims throughout the day. Instead of processing complex validations and SAP integrations instantly (which could slow down the UI and hit limits), a Scheduled Apex job runs nightly at 2:00 AM to process all claims submitted that day in one optimized batch.

---

## 2. What is Schedulable Apex?

Schedulable Apex allows you to delay the execution of an Apex class so that it runs at a specific time. 

* **Definition:** An Apex class that implements the `Schedulable` interface and is added to the Salesforce job scheduler queue.
* **Purpose:** To automate time-based tasks and recurring system maintenance.
* **Background Execution:** It runs completely decoupled from user sessions. The context user is typically the user who scheduled the job.
* **Job Scheduler Architecture:** The Salesforce platform maintains an internal scheduling engine. When you schedule a class, an entry is created in the `CronTrigger` table. The scheduling engine polls this table, and when the system clock matches the `NextFireTime`, the system allocates a thread from the asynchronous resource pool to execute the logic.



---

## 3. Why Do We Need Scheduled Jobs?

Enterprise Salesforce orgs require scheduled jobs for data integrity, integration, and performance optimization. Common scenarios include:

* **Nightly Processing:** Calculating daily totals, aggregating records, or scoring leads after business hours.
* **Weekly Maintenance:** Archiving old records, reassigning stagnant accounts.
* **Monthly Reporting:** Generating automated PDF invoices or compliance reports on the 1st of every month.
* **Daily Data Synchronization:** Pulling exchange rates or inventory levels from an external ERP.
* **Automatic Cleanup Jobs:** Deleting temporary log records or orphaned data.

### Enterprise Example (Automotive CRM)
A nationwide dealership network requires that all pending `Work_Order__c` records older than 30 days without customer approval be marked as "Expired." A Schedulable job runs every night at 1:00 AM to perform this cleanup, ensuring sales reps have an accurate pipeline the next morning.

---

## 4. Schedulable Interface

To make an Apex class schedulable, it must implement the `Schedulable` interface, which mandates the implementation of a single method: `execute()`.

```apex
global class DailyAccountJob implements Schedulable {
    global void execute(SchedulableContext sc) {
        // Business logic or delegation to Batch/Queueable goes here
    }
}
```

### Line-by-Line Explanation:
* `global class DailyAccountJob implements Schedulable {`: Defines the class. It can be `public` or `global` (global is required if scheduling via a managed package). The `implements Schedulable` tells the Salesforce compiler this class adheres to the scheduling contract.
* `global void execute(SchedulableContext sc) {`: This is the mandatory method. It returns `void`. It receives a `SchedulableContext` variable, which provides runtime information about the job.
* `SchedulableContext sc`: An interface provided by Salesforce. Its primary use is calling `sc.getTriggerId()` to get the ID of the `CronTrigger` associated with the job.

**Job Lifecycle:** Once scheduled, the class waits in the queue. At the specified time, Salesforce instantiates the class and invokes the `execute()` method.

---

## 5. System.schedule()

To programmatically queue a Schedulable class, you use the `System.schedule()` method.

```apex
String cronExp = '0 0 2 * * ?'; 
String jobID = System.schedule('Daily Warranty Processing', cronExp, new DailyWarrantyJob());
```

### Line-by-Line Explanation:
* `String cronExp = '0 0 2 * * ?';`: Defines the schedule format (CRON expression). Here, it translates to "2:00 AM every day."
* `String jobID =`: The method returns a `String` representing the 18-character ID of the created `CronTrigger` record.
* `System.schedule(`: The system method to submit the job.
* `'Daily Warranty Processing',`: The name of the job as it will appear in the Salesforce UI (Setup -> Scheduled Jobs).
* `cronExp,`: The CRON string defining the schedule.
* `new DailyWarrantyJob()`: An instance of the class that implements the `Schedulable` interface.

---

## 6. Understanding CRON Expressions

Salesforce uses a customized version of CRON expressions consisting of 6 or 7 space-separated fields.

**Format:** `Seconds` `Minutes` `Hours` `Day_of_month` `Month` `Day_of_week` `Optional_year`

### Field Breakdown:
| Field | Allowed Values | Special Characters |
| :--- | :--- | :--- |
| Seconds | 0-59 | None |
| Minutes | 0-59 | None |
| Hours | 0-23 | None |
| Day_of_month | 1-31 | * ? - / L W |
| Month | 1-12 (or JAN-DEC) | * - / |
| Day_of_week | 1-7 (or SUN-SAT) | * ? - / L # |
| Year (Optional) | 1970-2099 | * - / |

### Special Characters:
* `*` (All): Every value (e.g., `*` in Month means every month).
* `?` (No specific value): Used in Day_of_month or Day_of_week when you specify one but not the other.
* `-` (Range): e.g., `MON-FRI`.
* `,` (List): e.g., `1,15` for the 1st and 15th.
* `/` (Increments): e.g., `0/15` in minutes (Note: Salesforce only allows scheduling on the hour/half-hour natively through UI, but programmatically you can get closer, though not guaranteed to run precisely at the minute).
* `L` (Last): Last day of month/week.
* `W` (Weekday): Nearest weekday to a specific date.
* `#` (Nth day): e.g., `2#1` means the first Monday of the month.

### Examples:
* **Every day at 2 AM:** `0 0 2 * * ?`
* **Every Monday at 6 AM:** `0 0 6 ? * MON`
* **Every Sunday midnight:** `0 0 0 ? * SUN`
* **Last Friday of the month at 5 PM:** `0 0 17 ? * 6L`

---

## 7. Scheduling Batch Apex

Schedulable Apex is frequently used strictly as an orchestration tool to launch Batch Apex. Batch Apex is meant for Large Data Volumes (LDV), while Schedulable Apex only has standard synchronous limits inside its `execute()` method.

```apex
public class DailyWarrantyScheduler implements Schedulable {
    public void execute(SchedulableContext sc) {
        // Delegate heavy processing to Batch Apex
        Database.executeBatch(new WarrantyClaimBatch(), 200);
    }
}
```

### Why Schedule Batch?
If an Automotive CRM has 50,000 warranty claims to validate nightly, doing this purely in Schedulable Apex would hit the 10,000 DML row limit or CPU limits. By launching a Batch job from the Schedulable class, the platform chunks the 50,000 records into 250 safe, isolated transactions of 200 records each.

---

## 8. Scheduling Queueable Apex

You can also launch Queueable jobs from Schedulable Apex.

```apex
public class SAPSyncScheduler implements Schedulable {
    public void execute(SchedulableContext sc) {
        System.enqueueJob(new SAPInventoryQueueable());
    }
}
```

### Why and When?
Queueable Apex allows for complex API callouts and chaining. If you need to schedule a daily sync with SAP for Spare Parts Inventory, chaining Queueables is a cleaner, more modern architecture than Future methods or lightweight Batch jobs.

---

## 9. Scheduling Future Methods

Future methods (`@future`) cannot be scheduled directly. You cannot pass a Future method to `System.schedule()`.

### The Workaround
You must create a Schedulable class that simply calls the class containing the Future method.
* **Best Practice:** Prefer Queueable over Future when launching from Schedulable, as Queueable allows for object passing and better monitoring. Future methods are primitive fire-and-forget mechanisms.

---

## 10. Job Lifecycle



1.  **Job Creation:** `System.schedule()` is called.
2.  **Job Scheduling:** Salesforce saves a `CronTrigger` record.
3.  **Waiting State:** The job sits in the queue (`WAITING`).
4.  **Execution:** System clock hits `NextFireTime`. Status moves to `ACQUIRED` then `EXECUTING`.
5.  **Completion/Failure:** Logic finishes or throws an unhandled exception.
6.  **Rescheduling:** The scheduler calculates the next run date based on the CRON expression and updates `NextFireTime`.

---

## 11. Monitoring Scheduled Jobs

You can monitor jobs natively without code:
1.  **Setup → Scheduled Jobs:** View currently scheduled recurring jobs (Can delete them here).
2.  **Setup → Apex Jobs:** View the execution history (Success, Failure, Aborted).
3.  **Developer Console:** View logs generated during execution.

---

## 12. CronTrigger Object

The `CronTrigger` object holds the schedule and execution details of a scheduled job.

### Key Fields:
* `Id`: Unique identifier for the schedule.
* `CronExpression`: The CRON string used to schedule it.
* `NextFireTime`: The exact DateTime it will run next.
* `PreviousFireTime`: The last time it executed.
* `State`: Current status (e.g., `WAITING`, `ACQUIRED`, `EXECUTING`, `COMPLETE`, `ERROR`, `DELETED`).
* `TimesTriggered`: Count of how many times it has run.

### SOQL Example:
```apex
SELECT Id, CronJobDetail.Name, CronExpression, NextFireTime, State 
FROM CronTrigger 
WHERE State = 'WAITING'
```

---

## 13. CronJobDetail Object

`CronJobDetail` holds the metadata (like the Name and Job Type) for a scheduled job. It is a parent object to `CronTrigger`.

* **Purpose:** Separates the definition/name of the job from the actual time-based trigger.
* **Relationship:** `CronTrigger` has a lookup to `CronJobDetail` via `CronJobDetailId`.

### SOQL Example:
```apex
SELECT Id, Name, JobType 
FROM CronJobDetail 
WHERE Name = 'Daily Warranty Processing'
```

---

## 14. Governor Limits

Schedulable Apex is bound by specific limits to protect the multi-tenant environment.

| Limit Type | Limit |
| :--- | :--- |
| **Max Scheduled Apex Jobs** | 100 active jobs at a time. |
| **Max Concurrent Jobs** | Handled by platform queuing; no strict concurrent limit, but resources throttle. |
| **SOQL Queries** | 100 (Synchronous limit applies inside `execute()`). |
| **DML Statements** | 150. |
| **DML Rows** | 10,000. |
| **CPU Time** | 10,000 milliseconds (Synchronous limit). |
| **Heap Size** | 6 MB. |

**Scalability Note:** Because the limits inside the `execute()` method are standard synchronous limits (not async), you should almost *never* process data directly in a Schedulable class. Always hand off to Batch or Queueable.

---

## 15. Transaction Lifecycle



* **Separate Transactions:** The act of *scheduling* the job is one transaction. The *execution* of the job happens in a completely separate transaction hours or days later.
* **Commit Behavior:** Database changes made inside the `execute()` method are committed at the end of the `execute()` block.
* **Rollback:** If an unhandled exception occurs inside `execute()`, all DML operations in that specific transaction roll back.
* **Governor Limit Reset:** Every time the job fires, it gets a fresh set of governor limits.

---

## 16. Error Handling

Because scheduled jobs run in the background, a silent failure means business processes halt without users knowing.

### Architect Recommendation: Custom Logging Object
```apex
public class DealerSyncScheduler implements Schedulable {
    public void execute(SchedulableContext sc) {
        try {
            // Complex logic or Batch kickoff
            Database.executeBatch(new DealerPerformanceBatch());
        } catch (Exception e) {
            // Log to custom object
            Error_Log__c log = new Error_Log__c();
            log.Class_Name__c = 'DealerSyncScheduler';
            log.Error_Message__c = e.getMessage();
            log.Stack_Trace__c = e.getStackTraceString();
            insert log;
            
            // Or send alert email
            Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();
            mail.setToAddresses(new String[] {'admin@automotivecrm.com'});
            mail.setSubject('Scheduled Job Failed: Dealer Sync');
            mail.setPlainTextBody(e.getMessage());
            Messaging.sendEmail(new Messaging.SingleEmailMessage[] { mail });
        }
    }
}
```

---

## 17. Testing Scheduled Apex

Testing Scheduled Apex requires verifying that the job can be successfully scheduled. It does *not* actually wait for the CRON time in a test context.

```apex
@isTest
private class DailyAccountJobTest {
    @isTest
    static void testScheduledJob() {
        // 1. Setup Test Data (Test Data Factory)
        Account acc = new Account(Name = 'Test Dealer');
        insert acc;
        
        // 2. Define a dummy CRON (time doesn't matter in test)
        String cron = '0 0 0 1 1 ? 2099';
        
        Test.startTest();
        // 3. Schedule the job
        String jobId = System.schedule('Test Job', cron, new DailyAccountJob());
        Test.stopTest(); // <--- THIS is when the scheduled job executes synchronously!
        
        // 4. Assertions
        CronTrigger ct = [SELECT Id, CronExpression, TimesTriggered, NextFireTime 
                          FROM CronTrigger WHERE Id = :jobId];
        
        System.assertEquals(cron, ct.CronExpression, 'CRON should match');
        System.assertEquals(0, ct.TimesTriggered, 'Job should not trigger before stopTest completes internally');
        
        // Verify business logic actually ran
        Account updatedAcc = [SELECT Id, Status__c FROM Account WHERE Id = :acc.Id];
        System.assertEquals('Processed', updatedAcc.Status__c, 'Account status should be updated');
    }
}
```

### Key Concept:
Calling `Test.stopTest()` forces all asynchronous code enqueued during `Test.startTest()` to execute immediately and synchronously. This is how you assert the results.

---

## 18. Schedulable Apex vs Batch Apex

| Feature | Schedulable Apex | Batch Apex |
| :--- | :--- | :--- |
| **Primary Purpose** | Time-based execution (Scheduling). | Processing Large Data Volumes (LDV). |
| **Execution Model** | Runs once per schedule. | Breaks data into chunks (200 records). |
| **Data Limits** | Normal synchronous limits (10K rows). | Up to 50 million records. |
| **Use Case** | Run "Report Generator" every Friday. | Update 2 million `Vehicle__c` records. |

**Recommendation:** Combine them. Use Schedulable to trigger the Batch.

---

## 19. Schedulable Apex vs Queueable Apex

| Feature | Schedulable Apex | Queueable Apex |
| :--- | :--- | :--- |
| **Trigger Mechanism** | Time-based (CRON). | Event-based (System.enqueueJob). |
| **Chaining** | No direct chaining built-in. | Can chain another job in `execute()`. |
| **Complex Types** | Cannot easily pass complex objects in constructor. | Can pass sObjects and complex collections. |
| **Use Case** | Daily tasks. | API Callouts after a Trigger fires. |

**Recommendation:** If a time-based task requires callouts or complex chaining, use Schedulable to enqueue a Queueable job.

---

## 20. Schedulable Apex vs Future Methods

| Feature | Schedulable Apex | Future Methods |
| :--- | :--- | :--- |
| **Monitoring** | UI (Scheduled Jobs page), `CronTrigger`. | Limited (Apex Jobs page only). |
| **Simplicity** | Requires an interface and class. | Easy `@future` annotation on a method. |
| **Execution** | Predictable schedule. | Whenever resources are available. |

**Recommendation:** Avoid Future methods for complex architectures. Use Queueable instead. 

---

## 21. Enterprise Design Patterns

### The Orchestrator Pattern (Separation of Concerns)
Never put business logic inside the Schedulable class. It should act solely as an orchestrator or router.

```apex
// POOR DESIGN
public class BadScheduler implements Schedulable {
    public void execute(SchedulableContext sc) {
        List<Work_Order__c> wos = [SELECT Id FROM Work_Order__c WHERE Status = 'Pending'];
        // ... 50 lines of logic ...
        update wos;
    }
}

// ENTERPRISE DESIGN
public class GoodScheduler implements Schedulable {
    public void execute(SchedulableContext sc) {
        // Scheduler only knows HOW to start the process, not WHAT the process does.
        WorkOrderService.processPendingOrders(); 
    }
}
```

---

## 22. Real Project Scenarios (Automotive CRM)

1.  **Nightly Warranty Claim Processing:** Dealerships enter claims all day. Schedulable Apex runs at 2 AM, querying all `Claim__c` records with `Status = 'Submitted'`. It launches a Batch job to run complex validation rules and approve/reject claims.
2.  **Daily SAP Synchronization:** At 5 AM, a Schedulable job runs. It enqueues a Queueable Apex job that makes an HTTP REST callout to SAP to fetch updated pricing for `Spare_Part__c` records.
3.  **Customer Reminder Notifications:** Every morning at 8 AM, Schedulable Apex launches a Batch job checking `Vehicle__c.Next_Service_Date__c`. If it equals Today + 7 days, it generates a Service Reminder email.

---

## 23. Performance Optimization

* **Avoid Overlapping Jobs:** Do not schedule a 2-hour long Batch job to run every hour. Overlapping jobs lock tables and cause `UNABLE_TO_LOCK_ROW` exceptions.
* **Off-Peak Execution:** Schedule heavy data processing for 1:00 AM - 4:00 AM to minimize impact on active users.
* **Delegate LDV:** Never do SOQL/DML in the Schedulable class directly if volume can exceed a few hundred records. 

---

## 24. Common Mistakes

| Mistake | Consequence | Solution |
| :--- | :--- | :--- |
| **Hardcoding Schedules** | If requirements change, code must be deployed. | Use Custom Metadata to store the CRON string, or schedule via UI. |
| **Processing LDV directly** | `LimitException: Too many DML rows: 10001`. | Always route to Batch Apex. |
| **Scheduling Duplicate Jobs** | Hitting the 100 scheduled jobs limit. | Query `CronTrigger` before scheduling via code to ensure it doesn't exist. |
| **No Exception Handling** | Silent failures. | Implement try-catch and log to a Custom Error object. |

---

## 25. Best Practices Checklist

* ✅ **Use Scheduler only for orchestration:** Keep the `execute` method thin.
* ✅ **Schedule Batch Apex for large datasets:** Rely on Batch for robust data handling.
* ✅ **Use Queueable Apex for API callouts:** Schedulable -> Queueable -> HTTP Callout.
* ✅ **Use meaningful job names:** `Daily Warranty Sync` is better than `Job1`.
* ✅ **Monitor CronTrigger regularly:** Build dashboards or reports on error logs.
* ✅ **Write comprehensive unit tests:** Assert state changes, not just coverage.
* ✅ **Avoid overlapping jobs:** Give adequate buffer time between runs.
* ✅ **Check for existing jobs programmatically:** `if([SELECT count() FROM CronTrigger WHERE CronJobDetail.Name = 'MyJob'] == 0) { System.schedule(...); }`

---

## 26. Debugging Scheduled Apex

* **Developer Console:** Open the console, wait for the schedule to trigger, and look for logs with the `System` or context user.
* **Scheduled Jobs Page (`Setup -> Scheduled Jobs`):** Shows if the job is active and when it runs next. If it's missing, it completed or failed completely.
* **Apex Jobs Page (`Setup -> Apex Jobs`):** Check here for `Status = Failed` and view the Status Detail column for exception messages.
* **SOQL `AsyncApexJob`:** Run `SELECT Id, Status, NumberOfErrors, ExtendedStatus FROM AsyncApexJob WHERE JobType = 'ScheduledApex'` for deep programmatic debugging.

---

## 27. Interview Questions & Answers

### Beginner Questions
**Q: Can you execute callouts directly from Schedulable Apex?**
A: No. Schedulable Apex does not support synchronous callouts directly. You must call an `@future(callout=true)` method or enqueue a Queueable class that implements `Database.AllowsCallouts`.

### Intermediate Questions
**Q: What happens if a Scheduled Job hits a governor limit?**
A: The entire transaction is rolled back, no data is saved, the job is marked as failed for that specific run, but it remains in the `CronTrigger` queue to execute again at its next scheduled interval.

### Advanced Questions
**Q: How do you prevent users from scheduling the same job multiple times and hitting the 100-job limit?**
A: In an installer script or post-install script, query the `CronJobDetail` object by name. If it exists, abort the previous job or skip scheduling the new one. 

### Architect-Level Questions
**Q: A client requires a job to run every 5 minutes to sync High-Priority Leads. Standard Schedulable Apex only allows hourly schedules natively. How do you design this?**
A: Because standard Schedulable Apex UI is hourly, you can write a script to schedule the same class 12 times (using 12 distinct CRON strings offset by 5 minutes: `0 0 * * * ?`, `0 5 * * * ?`, etc.). However, as an Architect, I would push back. Near real-time syncs should be event-driven (Platform Events, Outbound Messaging, or CDC) rather than polling via aggressive scheduling, which wastes platform resources.

---

## 28. Revision Summary

* **Schedulable Apex:** Automates time-based execution of background logic.
* **Schedulable Interface:** Must implement `global void execute(SchedulableContext sc)`.
* **System.schedule():** Queues the job returning a `CronTrigger` ID. Takes Job Name, CRON, and instance.
* **CRON Expressions:** 7 parts (Seconds, Minutes, Hours, DayOfMonth, Month, DayOfWeek, Year).
* **Best Partner:** Batch Apex for LDV and Queueable for callouts/chaining.
* **CronTrigger / CronJobDetail:** System objects tracking scheduled states and metadata.
* **Governor Limits:** 100 active jobs max. Standard synchronous limits apply inside `execute()`.
* **Testing:** Triggered synchronously by `Test.stopTest()`.
* **Best Practices:** Keep the class purely as an orchestrator. Handle exceptions and log to custom objects to avoid silent failures.