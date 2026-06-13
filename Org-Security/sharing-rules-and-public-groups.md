# Salesforce Sharing Rules and Security

## 1. Introduction

### What Sharing Rules Are
Sharing Rules in Salesforce are automated, declarative mechanisms designed to grant record-level access to users who would otherwise not have visibility based on the Organization-Wide Defaults (OWD) or the Role Hierarchy. They represent lateral or exceptional sharing paths across the organization.

### Why Salesforce Introduced Sharing Rules
Salesforce introduced Sharing Rules to accommodate complex matrix reporting and cross-functional collaboration. The baseline security model (OWD + Role Hierarchy) is strictly vertical. Real-world organizations rarely operate in pure silos; horizontal collaboration is a frequent necessity.

### Business Need for Controlled Record Visibility
Organizations handle sensitive data (PII, financial records, proprietary designs) that must be restricted on a "need-to-know" basis to comply with regulations (GDPR, HIPAA) and internal policies. Sharing rules provide a scalable way to selectively poke holes in a restrictive baseline security model.

### Real-World Examples
* **Cross-Department:** Support agents need read-only access to Opportunities owned by the Sales team to verify customer entitlements before issuing a refund.
* **Regional Collaboration:** The EMEA Sales Director needs to see specific high-value accounts owned by the APAC Sales Director for a global corporate rollout.

### Where Sharing Rules Fit
Sharing Rules sit at the **Record-Level** of the Salesforce Security Model. They are evaluated *after* Object-level security (Profiles/Permission Sets) and *after* the baseline Record-level security (OWD).

---

## 2. Salesforce Security Model Overview

To understand Sharing Rules, you must understand the four pillars of Salesforce security:

* **Organization-Level Security:** Controls *who* can log in and *when/where* (IP Allowlisting, Login Hours, MFA, Single Sign-On).
* **Object-Level Security:** Controls *what* entities a user can interact with (Create, Read, Edit, Delete via Profiles and Permission Sets).
* **Field-Level Security (FLS):** Controls visibility and editability of specific attributes on an object.
* **Record-Level Security:** Controls *which specific instances* of an object a user can see (OWD, Roles, Sharing Rules).

### Interaction of Security Elements
* **Profiles / Permission Sets:** Define the baseline object and field permissions. (e.g., "Can I see Account records at all?")
* **OWD (Organization-Wide Defaults):** Defines the baseline record access. (e.g., "If I can see Accounts, can I see Accounts I don't own?")
* **Roles:** Opens access vertically. (e.g., "Can I see Accounts owned by my subordinates?")
* **Sharing Rules:** Opens access horizontally. (e.g., "Can I see Accounts owned by a different department?")
* **Restriction Rules:** Unlike Sharing Rules, Restriction Rules *remove* access to specific records even if the user has access via OWD or Sharing Rules.

---

## 3. What are Sharing Rules?

### Definition
Sharing Rules are administrative configurations that automatically grant lateral access to records based on record ownership or specific record criteria.

### Architecture & Metadata Structure
Under the hood, Sharing Rules create asynchronous recalculation jobs that insert records into underlying Share tables (e.g., `AccountShare`, `CustomObject__share`). In the Metadata API, they are represented as `SharingRules` components, subdivided into `<ownershipBased>` and `<criteriaBased>`.

### The Golden Rule: Sharing Rules Only Open Access
Sharing rules are additive. They can *never* restrict access. If OWD is set to Public Read/Write, creating a Sharing Rule is functionally useless because the baseline has already granted maximum visibility. Security in Salesforce is restrictive at the base (OWD) and permissive as you move up (Roles, Sharing).

---

## 4. Why Sharing Rules are Needed

### Cross-Department Collaboration
* *Scenario:* A "Deal Desk" team (Legal, Finance, Sales Ops) needs to review Opportunities once they reach the "Negotiation" stage. They do not share a role hierarchy branch with Sales. A Criteria-Based Sharing Rule grants them Read/Write access.

### Manager Visibility Across Regions
* *Scenario:* Global Overlay Managers who supervise product lines rather than geographical regions need access to regional accounts. An Ownership-Based Sharing Rule grants the "Overlay Managers" Public Group access to records owned by "Regional Sales" roles.

### Temporary Access Requirements
* *Scenario:* During an acquisition, an integration team needs temporary access to all newly imported leads. A Public Group is created for the team, and a Sharing Rule grants them access to the relevant records.

---

## 5. Record-Level Security Flow

When a user attempts to access a record, Salesforce evaluates access in this specific order:

1.  **Record Ownership:** Is the user the owner of the record? (Yes -> Full Access).
2.  **OWD Evaluation:** Is the OWD for this object Public Read/Write or Public Read Only?
3.  **Role Hierarchy Evaluation:** Is the user above the record owner in the Role Hierarchy? (Requires "Grant Access Using Hierarchies" to be true for custom objects).
4.  **Sharing Rule Evaluation:** Does an Ownership-Based or Criteria-Based sharing rule grant the user access?
5.  **Manual Sharing:** Did someone explicitly click "Share" on the record UI to grant this user access?
6.  **Team Sharing:** Is the user on the Account Team, Opportunity Team, or Case Team?
7.  **Apex Managed Sharing:** Is there programmatic sharing logic writing to the Share table?
8.  **Final Access Decision:** If any of the above grant access, the user can view/edit the record based on the highest level of access granted.

---

## 6. Public Groups

### What is a Public Group?
A Public Group is a logical grouping of users, roles, territories, or other groups that allows administrators to define a collection of individuals once and use it repeatedly across the platform (Sharing Rules, Folder Access, Content Delivery).

### Why Public Groups Exist
They simplify maintenance. Instead of creating 50 sharing rules for 50 different roles, you group those roles into one Public Group and create a single Sharing Rule.

### Components of a Public Group
A Public Group can contain:
* Individual Users
* Roles
* Roles and Subordinates
* Portal/Experience Cloud Roles
* Other Public Groups (Nested Groups)

### Nested Groups Example
* **Group A (EMEA Sales):** Contains all EMEA roles.
* **Group B (APAC Sales):** Contains all APAC roles.
* **Group C (Global Sales):** Contains Group A and Group B.

---

## 7. Public Group Architecture

Public Groups are stored in the `Group` standard object. The members of the group are stored in the `GroupMember` object. 

> **Architecture Flow:**
> `User` -> mapped to -> `GroupMember` -> belongs to -> `Group` <- targeted by <- `Sharing Rule`

When evaluating sharing, Salesforce traverses the `GroupMember` table to resolve all individual `User` IDs recursively, especially when dealing with Roles and Nested Groups.

---

## 8. Types of Sharing Rules

Salesforce offers three primary types of declarative Sharing Rules:

1.  **Ownership-Based Sharing Rules:** Grants access based on who owns the record.
2.  **Criteria-Based Sharing Rules:** Grants access based on field values on the record.
3.  **Guest User Sharing Rules:** Grants access to unauthenticated users via Experience Cloud sites.

---

## 9. Ownership-Based Sharing Rules

### How They Work
They evaluate the Owner of the record. If the owner belongs to the specified "Source Group/Role," the record is shared with the "Target Group/Role."

### Elements
* **Source:** Which records are being shared? (Records owned by Role A).
* **Target:** Who gets to see them? (Shared with Public Group B).
* **Access Level:** Read-Only or Read/Write.

### Example
Share all Accounts owned by the `US Sales Representatives` Role with the `US Sales Operations` Public Group with Read/Write access.

---

## 10. Criteria-Based Sharing Rules

### Criteria Evaluation
These rules evaluate field values (Standard or Custom fields) rather than record ownership. They are ideal for dynamic record access.

### Dynamic Record Access & Recalculation
When a user updates a record field that matches a criteria-based rule, Salesforce synchronously/asynchronously triggers a sharing recalculation, creating a Share record granting access to the target group.

### Example
* **Object:** Case
* **Criteria:** `Priority` equals 'High' AND `RecordType` equals 'Escalation'
* **Share With:** `Tier 3 Support` Public Group
* **Access Level:** Read/Write

---

## 11. Guest User Sharing Rules

### Experience Cloud Use Cases
Unauthenticated guest users (e.g., browsing a public FAQ portal) have strict security limitations. Guest User Sharing Rules are the *only* declarative way to grant record access to guest users.

### Security Considerations
* Guest User Sharing Rules can **only** grant Read-Only access.
* They are criteria-based.
* They help prevent accidental data exposure by forcing administrators to explicitly define what public data is visible.

---

## 12. Sharing Access Levels

| Access Level | Description | Behavior |
| :--- | :--- | :--- |
| **Read Only** | User can view the record but cannot make changes. | Overrides Private OWD to allow viewing. |
| **Read/Write** | User can view and edit the record. | Does not allow the user to delete the record, change ownership, or manually share it (unless they are above the owner in the hierarchy). |
| **Full Access** | User can view, edit, delete, transfer, and share the record. | Cannot be granted via Sharing Rules. Only available via Ownership, Role Hierarchy above the owner, or implicit sharing. |

---

## 13. Sharing Rules and OWD

Sharing Rules behave differently depending on the OWD:

* **Private:** Sharing rules are highly effective. They open access laterally.
* **Public Read Only:** Sharing rules can be used to grant Read/Write access to specific groups.
* **Public Read/Write:** Sharing rules are useless. Everyone already has maximum edit access.
* **Controlled by Parent:** Sharing rules *cannot* be created for the child object (e.g., Contacts, or custom objects in a Master-Detail relationship). Visibility is dictated entirely by the parent record.

---

## 14. Sharing Rules and Role Hierarchy

### Relationship and Access Inheritance
While the Role Hierarchy shares records vertically (upward), Sharing Rules share records horizontally. If a Sharing Rule grants Read/Write access to a User in Role A, the manager of Role A (Role B) will *also* inherit that Read/Write access, assuming "Grant Access Using Hierarchies" is enabled.

### Security Design Considerations
When designing enterprise security, Architects must map out unintended data leaks. If you share a highly sensitive record with a junior analyst via a Public Group, remember that the analyst's entire management chain will also see that record.

---

## 15. Public Groups vs Roles

| Feature | Public Groups | Roles |
| :--- | :--- | :--- |
| **Purpose** | Ad-hoc or horizontal grouping of users for lateral sharing. | Defines the management/reporting hierarchy for vertical sharing. |
| **Access Control** | Used in Sharing Rules, Folder access, etc. | Automatically rolls up data access to superiors. |
| **Flexibility** | Highly flexible. Can contain users, roles, and other groups. | Rigid. A user can only belong to exactly **one** Role. |
| **Maintenance** | Requires manual updating of users (unless automated via Apex/Flow). | Maintained naturally as HR reporting lines change. |

---

## 16. Public Groups vs Queues

| Feature | Public Groups | Queues |
| :--- | :--- | :--- |
| **Primary Use** | Record *Visibility* (Sharing Rules). | Record *Ownership* and load balancing. |
| **Object Support** | Applies globally to users. | Applies to specific objects (Cases, Leads, custom objects). |
| **Concept** | "We need to see these records." | "We need to take ownership and work these records." |

---

## 17. Sharing Rules vs Manual Sharing

| Feature | Sharing Rules | Manual Sharing |
| :--- | :--- | :--- |
| **Administration** | Automated. Defined by Admins. | Manual. Clicked by Users. |
| **Scalability** | High. Handles millions of records automatically. | Low. Requires human intervention per record. |
| **Persistence** | Permanent as long as criteria/ownership matches. | Can be lost if record ownership changes. |

---

## 18. Sharing Rules vs Apex Managed Sharing

| Feature | Sharing Rules (Declarative) | Apex Managed Sharing (Programmatic) |
| :--- | :--- | :--- |
| **Approach** | Clicks, no code. UI configuration. | Code. Written in Apex triggers/classes. |
| **Flexibility** | Limited by supported criteria fields and rule limits. | Infinite. Can share based on complex related-record logic or external data. |
| **Scalability** | Evaluated automatically. Can slow down bulk data loads. | Highly scalable but requires bulkified code to avoid governor limits. |
| **Maintenance** | Easy for Admins to maintain. | Requires Developer overhead to maintain. |

---

## 19. Apex Managed Sharing

### What is Apex Sharing
When declarative Sharing Rules fall short (e.g., sharing a record based on a junction object's data, or complex cross-object criteria), developers use Apex to programmatically insert records into Share tables.

### Share Objects
* Standard Objects: `AccountShare`, `CaseShare`, `OpportunityShare`
* Custom Objects: `CustomObject__share`

### RowCause
When creating a Share record via Apex, you must define a `RowCause` (Apex Sharing Reason). This prevents the platform from deleting your manual shares when ownership changes.

### Programmatic Sharing Example
```java
// Sharing a Custom Object 'Project__c' with a specific User
Project__share projShare = new Project__share();
projShare.ParentId = 'a0X...'; // The ID of the record being shared
projShare.UserOrGroupId = '005...'; // The User or Public Group ID
projShare.AccessLevel = 'Read'; // 'Read' or 'Edit'
projShare.RowCause = Schema.Project__share.RowCause.Project_Team_Assignment__c; // Custom sharing reason

insert projShare;
```

---

## 20. Sharing Tables

### Standard Object Share Tables
Objects like Account and Opportunity have built-in share tables (`AccountShare`). These tables contain columns for `AccountId`, `UserOrGroupId`, `AccountAccessLevel`, `OpportunityAccessLevel`, `CaseAccessLevel`, and `RowCause`.

### Custom Object Share Tables
Custom objects have a `__share` table automatically created *only if* the OWD of the object is set to Private or Public Read Only. If OWD is Public Read/Write, the table does not exist.

### Internal Storage
Sharing records consume data storage. In organizations with tens of millions of records and complex sharing rules, the Share tables can become massive, impacting performance.

---

## 21. Sharing Calculations

### The Sharing Engine
Salesforce maintains a complex, highly optimized sharing engine. When a sharing rule is created or modified, the engine must evaluate every applicable record and insert/delete rows in the Share tables.

### Recalculation Triggers
Sharing calculations are triggered by:
* Changing OWD settings.
* Changing Role Hierarchy structure.
* Adding/Removing users from Public Groups used in sharing rules.
* Record ownership changes.

### Synchronous vs Asynchronous
Minor changes happen synchronously. Major changes (like modifying a Role near the top of the hierarchy) trigger asynchronous Deferred Sharing Recalculation to prevent platform timeouts.

---

## 22. Enterprise Security Design Scenarios

### Automotive CRM
* **Objects:** Vehicle Records, Warranty Claims, Dealers.
* **Design:** `Dealers` are Experience Cloud users. `Vehicle Records` are Private. A Criteria-Based Sharing Rule on `Vehicle Records` (Criteria: Selling Dealer = User's Dealer Account) grants Read visibility. `Warranty Claims` use Master-Detail to Vehicles, inheriting access.

### Banking Organization
* **Objects:** Customer Records, Loan Applications.
* **Design:** Strict Regulatory compliance. OWD is Private. Role hierarchy is flattened. Apex Managed Sharing is used strictly based on `Loan Team Members` (custom junction object) to ensure *only* the specific underwriters assigned to the loan can view the Customer PII.

### Insurance Company
* **Objects:** Policies, Claims.
* **Design:** OWD is Private. Criteria-Based Sharing rules share `Claims` with the "Fraud Investigation" Public Group when `Status` = "Under Investigation".

---

## 23. Experience Cloud Sharing

### External Sharing Models
External users (Partners, Customers) have their own separate OWD settings (External OWD).

### Sharing Sets
Used for Customer Community licenses (which do not support Roles or Sharing Rules). Sharing Sets grant access to records by matching a field on the User record (like ContactId) to a field on the target record.

### Share Groups
Used to share records *owned* by high-volume Customer Community users with internal Salesforce users.

---

## 24. Performance Considerations

### Large Data Volumes (LDV)
In orgs with millions of records, complex sharing configurations can lead to severe performance degradation.

### Optimization Strategies
* **Keep OWD Public if possible:** Only lock down objects that truly require it.
* **Flatten the Role Hierarchy:** Avoid deep hierarchies (Salesforce recommends no more than 10 levels deep).
* **Group Membership:** Minimize nested Public Groups, as the sharing engine must recursively unravel them during recalculation.
* **Deferred Recalculation:** Use deferred sharing maintenance during large data loads to pause share calculations until the load is complete.

---

## 25. Sharing Rule Limits

| Feature | Limit | Practical Implication |
| :--- | :--- | :--- |
| **Total Sharing Rules per Object** | 300 | Architectural boundaries; if you need more, you must consolidate groups or use Apex. |
| **Criteria-Based Sharing Rules** | 50 (within the 300) | Severely limits dynamic declarative sharing. Complex setups require Apex Sharing. |
| **Role Hierarchy Levels** | ~10 recommended | Deeper hierarchies exponentially increase recalculation times. |

---

## 26. Common Mistakes

* **Overusing Sharing Rules:** Creating a new rule for every minor request instead of consolidating Public Groups.
* **Confusing Roles with Groups:** Using Roles to group users horizontally. Roles dictate reporting; Groups dictate cross-functional sharing.
* **Ignoring Profile Level Security:** Trying to fix an object-level issue ("User can't see the Accounts tab") with a record-level solution (Sharing Rules).
* **"Public Read/Write" OWD with Sharing Rules:** Redundant and wastes calculation resources.

---

## 27. Best Practices

* **Naming Conventions:** Name Public Groups based on function and region (e.g., `PG_EMEA_SalesOps`).
* **Minimize Group Churn:** Avoid adding individual users to Public Groups. Instead, add Roles or other manageable groups to minimize manual maintenance.
* **Access Reviews:** Regularly audit Public Group membership.
* **Code over Config for Scale:** If an object requires 40+ criteria-based sharing rules, it is often more performant and maintainable to transition to an Apex Managed Sharing architecture.

---

## 28. Troubleshooting Record Access Issues

When a user complains, "I can't see this record":
1.  **Check Object Permissions:** Does their Profile/Permission Set have Read access to the object?
2.  **Check OWD:** Is the OWD Private?
3.  **Check Ownership/Role:** Are they the owner, or above the owner in the Role Hierarchy?
4.  **Use the "Sharing" Button:** On the record UI in Salesforce Classic or via Apex in Lightning, look at the Sharing Hierarchy to see exactly *why* a user has access.
5.  **Check Public Group Membership:** Verify the user hasn't been removed from a critical Public Group.

---

## 29. Auditing and Compliance

* **Setup Audit Trail:** Tracks who created or modified Sharing Rules and Public Groups.
* **Field History Tracking:** Can be used alongside Apex to track when sharing reasons are modified.
* **Compliance:** In heavily regulated industries, favor Apex Managed Sharing with custom RowCauses, as it allows for strict, programmatic auditing of *why* access was granted.

---

## 30. Modern Salesforce Record Visibility Strategy

A robust, modern architecture leverages all tools efficiently:

1.  **Profiles/Perm Sets:** Bare minimum object access.
2.  **OWD:** Restrictive baseline (Private).
3.  **Roles:** Clean, shallow vertical reporting lines.
4.  **Public Groups:** Logical, function-based collections.
5.  **Sharing Rules:** Broad, lateral department-level access.
6.  **Apex Sharing:** Complex, multi-object programmatic access.
7.  **Restriction Rules:** Scoping down access for specific sensitive records (e.g., NDA clients).

---

## 31. Interview Questions & Answers

### Beginner
**Q: Can a Sharing Rule restrict access?**
**A:** No. Sharing rules can only grant wider access. Security starts restricted at OWD and is opened up by Sharing Rules.

### Intermediate
**Q: What is the difference between an Ownership-based and Criteria-based sharing rule?**
**A:** Ownership-based shares records based on who owns them (e.g., share Role A's records with Group B). Criteria-based shares records based on field values (e.g., share records where Industry = 'Banking').

### Advanced
**Q: How does OWD affect the underlying database schema for Custom Objects?**
**A:** If a custom object's OWD is set to Private or Public Read Only, Salesforce automatically creates a `CustomObject__share` table. If it is Public Read/Write, this table does not exist.

### Architect-Level
**Q: An organization is hitting the 50 criteria-based sharing rule limit on the Opportunity object. How do you redesign the architecture?**
**A:** I would evaluate consolidating rules by creating formula fields that evaluate multiple criteria into a single boolean, then base the sharing rule on that formula. If the logic is too complex or dynamic, I would deprecate the declarative rules and implement Apex Managed Sharing using an asynchronous framework to manage the `OpportunityShare` table.

---

## 32. Revision Summary

* **Sharing Rules:** Declarative tools to laterally open record access. Never restrict access.
* **Public Groups:** Reusable containers of Users, Roles, or other Groups. Used as targets for sharing.
* **Ownership-Based:** Shares based on the Record Owner's group/role.
* **Criteria-Based:** Shares based on specific field values. Limit 50 per object.
* **Apex Sharing:** Programmatic inserts into `__share` tables using a `RowCause`.
* **Sharing Tables:** Store access levels; recalculation can cause performance bottlenecks in LDV environments.
* **Best Practice:** Build restrictive OWDs, shallow Role Hierarchies, and use Sharing Rules for cross-functional collaboration.