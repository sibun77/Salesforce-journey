# Salesforce Workflow Rules

## 1. Introduction

### What Workflow Rules Are
Workflow Rules are Salesforce’s legacy declarative automation tool, designed to evaluate records as they are created or edited and automatically execute predefined actions if specific criteria are met. They form the foundational layer of Salesforce's classic automation capabilities, operating primarily on a single-object context.

### Why Salesforce Introduced Workflow Rules
Salesforce introduced Workflow Rules to empower administrators to automate standard business processes without writing Apex code. Before Process Builder and Flow, Workflow Rules were the primary mechanism to enforce business logic, standardize data entry, and notify users of critical updates, reducing manual effort and data inconsistency.

### Historical Evolution of Salesforce Automation
* **Era 1: Workflow Rules:** Simple IF/THEN statements. Highly reliable but limited to four specific actions (Field Update, Email Alert, Task Creation, Outbound Message).
* **Era 2: Process Builder:** Introduced visual layouts and cross-object updates, capable of evaluating multiple criteria nodes in a single transaction.
* **Era 3: Lightning Flow (Now Salesforce Flow):** The ultimate declarative automation tool, replacing both Workflow and Process Builder, offering Turing-complete logic, UI screens, and bulkified processing.

### Current Status of Workflow Rules
As of Winter '23, Salesforce has disabled the creation of *new* Workflow Rules (with some exceptions for managed packages). Existing rules continue to function, but Salesforce strongly advises migrating them to Flow.

### Why Developers Still Need to Understand Them
Enterprise environments contain years of accumulated technical debt. Architects and Developers must understand Workflow Rules to:
1. Troubleshoot legacy system behavior and order-of-execution bugs.
2. Safely migrate complex logic to Salesforce Flow.
3. Understand the historical context of existing architecture.

---

## 2. Workflow Rule Architecture

### Workflow Engine
The Workflow Engine is an internal Salesforce service that listens to DML operations. It triggers during the "Save Order of Execution" after database triggers and assignment rules, evaluating records against active workflow criteria.

### Architecture Diagram

```text
[ Record DML Operation ] --> [ Database Save ] --> [ Before/After Triggers ]
                                                           |
                                                           v
[ Workflow Engine Intercept ] <-------------------- [ Assignment & Auto-Response Rules ]
        |
        +-- [ Rule Evaluation ]
                |
                +-- Criteria NOT Met ---> [ Ignore & Continue ]
                |
                +-- Criteria Met -------> [ Action Execution Queue ]
                                                |
                                                +-- [ Immediate Actions ] -> Execute Now
                                                |
                                                +-- [ Time-Dependent Actions ] -> Send to Workflow Queue
```

### Behind the Scenes
When a rule triggers:
1. **Rule Evaluation:** The engine checks the evaluation criteria (e.g., *created*, or *created, and every time it's edited*).
2. **Criteria Processing:** The engine evaluates the specific rule criteria or formula against the current record's state.
3. **Action Execution:** Actions are fired. If a Field Update occurs, it may re-trigger the workflow engine and Apex triggers (up to 5 additional times) depending on the "Re-evaluate Workflow Rules after Field Change" setting.

---

## 3. Components of a Workflow Rule

1. **Rule Name & Description:** Identifies the rule. (e.g., `Escalate High Priority Case`).
2. **Evaluation Criteria:** Dictates *when* the engine should check the record.
3. **Rule Criteria:** The specific IF condition (e.g., `Case.Priority == 'High'`).
4. **Workflow Actions:** The THEN response (Immediate or Time-Dependent).
5. **Active / Inactive State:** A boolean flag determining if the engine will process the rule.

---

## 4. Workflow Evaluation Criteria

The Evaluation Criteria is the first gatekeeper.

1. **Created:** Evaluates only once upon record creation.
   * *Real-world usage:* Welcome emails, initial default field values.
2. **Created, and every time it's edited:** Evaluates on insert and every update.
   * *Internal behavior:* Cannot be used with time-dependent actions.
   * *Common mistake:* Causing infinite loops if a field update triggers an edit that meets the criteria again.
3. **Created, and any time it's edited to subsequently meet criteria:** Evaluates on insert if it meets criteria, OR on update if the record previously did *not* meet the criteria but now *does*.
   * *Real-world usage:* Sending a "Closed Won" email. You only want this sent once when the stage changes to Closed Won, not every time a user edits a Closed Won opportunity.

---

## 5. Workflow Rule Criteria

### Criteria-Based Rules
Uses a standard UI filter (Field, Operator, Value).
* *Example:* `Lead Status` [EQUALS] `Open - Not Contacted` AND `Rating` [EQUALS] `Hot`.

### Formula-Based Rules
Uses Salesforce formula syntax. Must evaluate to True or False.
* *Example:* `AND(ISPICKVAL(StageName, "Closed Won"), Amount > 50000, NOT(ISNEW()))`
* *Best Practices:* Use formulas for cross-object references (e.g., traversing up a lookup to check a parent account's status).
* *Common Pitfalls:* Exceeding compile size limits, or failing to handle null values properly (e.g., using `ISBLANK()`).

---

## 6. Workflow Actions Overview

| Action Type | Purpose | Use Case | Limitations |
| :--- | :--- | :--- | :--- |
| **Field Update** | Updates a field on the record or its master record. | Changing Stage to "Closed" when approved. | Cannot update cross-object lookups (except Master-Detail parent). |
| **Email Alert** | Sends an email via a predefined template. | Notifying user of a new Case. | Daily limits apply; complex dynamic recipients are hard to manage. |
| **Task Creation** | Creates a standard Salesforce Task. | Assigning a follow-up call. | Cannot create generic custom objects or Events. |
| **Outbound Message** | Sends XML via SOAP to an external endpoint. | Syncing record data to an ERP. | SOAP only (No REST/JSON). |

---

## 7. Field Update Action

### How It Works
Modifies a field value on the exact record that triggered the workflow, or a related Master record in a Master-Detail relationship.

### Re-evaluation of Workflow Rules
A crucial checkbox exists on Field Updates: **"Re-evaluate Workflow Rules after Field Change"**.
* If checked, Salesforce re-runs all workflow rules on the object.
* It also re-runs `before update` and `after update` Apex triggers.
* **Recursive Behavior:** Salesforce limits this recursive loop to 5 iterations. If it hits the 6th, the transaction fails with a runtime exception.

---

## 8. Email Alerts

### Components
* **Email Templates:** Defines the content (Text, HTML, Custom, Visualforce).
* **Recipients:** Can be static users, roles, public groups, or dynamic based on fields (e.g., `Owner`, `Creator`, or specific Email fields on the record).

### Practical Example
* *Use Case:* Contract Expiration Warning.
* *Dynamic Recipient:* `Opportunity.OwnerId` and `Account.Manager__c`.

---

## 9. Task Creation

Generates an actionable item assigned to a specific user, role, or record owner.

### Key Considerations
* **Assignment:** Tasks can only be assigned to a single user.
* **Due Dates:** Can be dynamic (e.g., `Rule Trigger Date + 3 Days`).
* *Real-world usage:* When a Lead is created with "Hot" rating, a Task is generated for the Lead Owner due in 24 hours to "Call Lead".

---

## 10. Outbound Messages

### What They Are
A legacy, SOAP-based mechanism to send a synchronous payload out of Salesforce to an external system without writing Apex callouts.

### Architecture

```text
[ Workflow Rule Met ] --> [ Outbound Message Action ] --> [ SOAP XML Payload Generated ]
                                                                      |
[ External ERP/System ] <--- HTTPS POST <-----------------------------+
        |
        +-- Responds with <Ack>true</Ack>
```

### Reliability and Retry Behavior
* If the external system does not respond with a positive Acknowledgment, Salesforce will retry.
* It retries for up to **24 hours**, with increasing intervals (exponential backoff).
* Messages are delivered "at least once" (can result in duplicate deliveries; idempotency must be handled by the receiving system).

---

## 11. Immediate Actions vs Time-Dependent Actions

* **Immediate Actions:** Fire instantly in the same transaction as the DML statement.
* **Time-Based Actions:** Scheduled to fire at a specific time relative to a date/time field on the record or the rule trigger date.
  * *Example:* Send an email 30 days before `Contract_End_Date__c`.
  * *Constraint:* Cannot be used if the Evaluation Criteria is set to "Every time it's edited".

---

## 12. Time-Based Workflow

### Workflow Queue
When a time-dependent criteria is met, the action is placed in the **Time-Based Workflow Queue**.
* **Monitoring:** Admins can view this queue via Setup -> Time-Based Workflow to see pending actions.
* **Dynamic Removal:** If a record is edited and *no longer meets* the workflow criteria, its pending actions are automatically removed from the queue.

---

## 13. Workflow Rules and Order of Execution

This is crucial for Salesforce Architects.

1. System Validation Rules
2. `before insert` / `before update` Apex Triggers
3. Custom Validation Rules
4. `after insert` / `after update` Apex Triggers
5. Assignment Rules
6. Auto-Response Rules
7. **WORKFLOW RULES**
8. **If Workflow Field Updates occur:**
   * System Validation (Custom validations are *not* run again)
   * `before update` triggers
   * `after update` triggers
   * *(Note: Execution does not cascade further into Assignment/Auto-Response again)*
9. Escalation Rules
10. Roll-Up Summary Fields
11. Database Commit

*Architect Note:* Workflow field updates run very late in the transaction. If a workflow updates a field, the `before update` and `after update` triggers run *again*, which can severely impact transaction performance and cause limits to be breached (e.g., SOQL 101).

---

## 14. Workflow Rule Limitations

* **No Record Creation:** Cannot create child records (except Tasks).
* **No Record Deletion:** Cannot delete records.
* **No Complex Branching:** No IF/ELSE IF logic. Each rule is a single boolean check.
* **No Loops:** Cannot iterate over child records.
* **Limited Logic:** Cannot perform complex math or queries across disparate objects.
* **Order of Execution:** You cannot control the order in which multiple Workflow Rules on the same object execute.

---

## 15. Workflow Rules vs Process Builder

| Feature | Workflow Rules | Process Builder |
| :--- | :--- | :--- |
| **Logic** | Single IF statement | Multiple IF/ELSE IF nodes |
| **Cross-Object Updates** | Master-Detail only | Any related record (Lookups & M-D) |
| **Record Creation** | Tasks only | Any object |
| **Apex Invocation** | No | Yes (`@InvocableMethod`) |
| **Performance** | Very Fast (Native C++ under the hood) | Slower (Metadata driven) |
| **Current Status** | Deprecated for new creation | Deprecated for new creation |

---

## 16. Workflow Rules vs Flow

| Feature | Workflow Rules | Salesforce Flow (Record-Triggered) |
| :--- | :--- | :--- |
| **Functionality** | Extremely limited (4 actions) | Turing complete (Create, Update, Delete, Get, Loops) |
| **Performance** | Highly optimized | Highly optimized (Before-save flows are 10x faster than PB/Workflow) |
| **Scalability** | Poor (leads to rule sprawl) | Excellent (Subflows, entry conditions) |
| **Order Control** | None | Explicitly controllable via Trigger Order (Flow Explorer) |
| **Future Roadmap** | End of Life planned | Salesforce's flagship automation platform |

**Salesforce Recommendation:** Migrate ALL declarative automation to Flow.

---

## 17. Workflow Rule Migration to Flow

### Why Migration is Required
Salesforce is focusing all R&D on Flow. Maintaining Workflow Rules splits the declarative automation layer, making order of execution impossible to predict and debug.

### Migration Strategy
1. **Analyze:** Document existing rules. Identify redundancies.
2. **Optimize:** Do not do a 1:1 migration. Consolidate multiple workflow rules on the same object into a single Record-Triggered Flow using Decision nodes.
3. **Choose Flow Type:**
   * *Same-Record Field Update?* Use **Before-Save Flow** (Fast Field Updates).
   * *Email, Task, Outbound Message?* Use **After-Save Flow** (Actions and Related Records).

### Migration Tools
Salesforce provides the **"Migrate to Flow"** tool in Setup, which automates 1:1 translation. However, enterprise architects recommend *manual rebuilds* to consolidate logic.

---

## 18. Real Project Scenarios

* **Lead Qualification:** When `Rating` = 'Hot', create a Task for the owner and send an Email Alert to the regional manager.
* **Opportunity Stage Updates:** When an Opportunity is closed, update a custom `Close_Date_Timestamp__c` field. (Migrated to Before-Save Flow).
* **Outbound ERP Integration:** When an Account's `ERP_Sync_Status__c` is 'Ready', trigger an Outbound Message to send the XML payload to SAP.

---

## 19. Common Workflow Patterns

* **Notification Workflows:** Purely designed to send Email Alerts.
* **Assignment Workflows:** Changing record ownership via Field Updates.
* **Status Update Workflows:** E.g., If all checklist fields are true, change `Status` to 'Complete'.

---

## 20. Common Mistakes

* **Recursive Updates:** Checking "Re-evaluate Workflow Rules" without tight exit criteria, causing trigger loops and hitting the 5-iteration limit.
* **Incorrect Evaluation Criteria:** Using "Created and every time it's edited" when you only want an email sent ONCE when a status changes.
* **Time-Based Action Issues:** Assuming a time-based workflow will fire even if the record's criteria changes (it won't; the queue auto-clears).
* **Conflicting Automations:** Having Workflow Rules, Process Builder, and Triggers all updating the same object simultaneously.

---

## 21. Best Practices

* **Migration Planning:** Stop creating new Workflows immediately. Map out a phased migration to Flow.
* **Documentation Standards:** Document the purpose, criteria, and exact actions of every legacy rule before disabling it.
* **Naming Conventions:** Prefix legacy rules with `[Deprecating]` or `[Migrated]` once disabled, before final deletion.

---

## 22. Enterprise Architecture Considerations

### Technical Debt Management
Large enterprises often have hundreds of Workflow Rules. The architect's job is to prevent "lift-and-shift" migrations. Moving 100 Workflow Rules into 100 Flows creates a Flow maintenance nightmare.
* **Consolidation:** Implement an enterprise Flow architecture (e.g., One Before-Save Flow and One After-Save Flow per object, using Subflows).
* **Governance:** Enforce strict peer review. Any modified legacy workflow must be migrated to Flow rather than updated in place.

---

## 23. Debugging and Troubleshooting

* **Debug Logs:** Set `Workflow` to `FINEST` in Developer Console to see rule evaluation true/false states.
* **Workflow Queue Monitoring:** Always check `Time-Based Workflow` in Setup when users complain that "Emails aren't sending later."
* **Order of Execution Failures:** If a Validation Rule suddenly breaks a process, check if a Workflow Field Update is causing triggers to run a second time, modifying data unexpectedly.

---

## 24. Interview Questions & Answers

**Beginner:**
* *Q: What are the 4 actions a workflow rule can perform?*
  *A: Field Update, Email Alert, Task Creation, Outbound Message.*

**Intermediate:**
* *Q: What happens to pending time-based actions if a record is edited to no longer meet the workflow criteria?*
  *A: They are automatically removed from the Time-Based Workflow Queue.*

**Advanced:**
* *Q: Explain the impact of "Re-evaluate Workflow Rules after Field Change".*
  *A: It causes the Workflow Engine to run again, and critically, causes `before update` and `after update` triggers to execute an additional time (up to 5 recursions), which can consume CPU time and SOQL limits.*

**Architect:**
* *Q: How would you design a migration strategy for an org with 300 active workflow rules?*
  *A: I would not use the 1:1 auto-migration tool. I would extract the metadata, group rules by Object, separate same-record updates from actions, and build a unified Flow architecture (Before-Save for field updates, After-Save for actions). I'd implement bypass toggles and migrate object by object, deploying with heavy automated regression testing.*

---

## 25. Revision Summary

* **Architecture:** Declarative, DML-driven automation engine executing late in the transaction cycle.
* **Actions:** Limited to Field Updates, Emails, Tasks, Outbound Messages.
* **Order of Execution:** Runs after assignment rules; field updates re-trigger execution.
* **Limitations:** No loops, no complex logic, cannot control execution order.
* **Migration:** Essential. Flow replaces Workflow. Consolidate logic during migration to optimize performance.