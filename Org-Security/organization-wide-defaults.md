# Salesforce Organization wide defaults

# 1. Introduction

**Organization-Wide Defaults (OWD)** are the foundational building block of record-level security in Salesforce. They determine the baseline level of access that the most restricted user in your organization should have for a specific object's records. 

When Salesforce introduced OWD, the primary purpose was to establish a restrictive foundation upon which developers and administrators could layer granular access. By setting the baseline access to the lowest required denominator, organizations can use subsequent sharing mechanisms (like Role Hierarchies, Sharing Rules, and Apex Sharing) to open up access only where necessary. 

**Real-world Example:** Consider a multinational bank. The OWD for `Account` records is set to **Private**. This ensures that a loan officer in London cannot see the accounts of a loan officer in New York by default. Access is strictly baseline. If cross-region collaboration is needed, a specific Sharing Rule or Account Team is utilized to grant that access.

---

# 2. Salesforce Security Model Overview

Salesforce employs a multi-layered security architecture. OWD fits specifically within the **Record-Level** tier.

### Organization-Level Security
Controls *when* and *from where* users can log in (IP Restrictions, Login Hours, Identity Verification).

### Object-Level Security (CRUD)
Controls *which* objects a user can view, create, edit, or delete. Managed via **Profiles** and **Permission Sets**.

### Field-Level Security (FLS)
Controls *which* fields on an object a user can read or edit. Also managed via **Profiles** and **Permission Sets**.

### Record-Level Security (Sharing)
Controls *which specific records* a user can see, assuming they have Object-level access. Managed via **OWD, Role Hierarchy, Sharing Rules, and Manual/Apex Sharing**.

```text
+-------------------------------------------------------------+
|               SALESFORCE SECURITY ARCHITECTURE              |
+-------------------------------------------------------------+
| 1. ORG LEVEL: Can the user log in?                          |
|    (IP Ranges, Login Hours, Multi-Factor Auth)              |
+-------------------------------------------------------------+
| 2. OBJECT LEVEL: Can the user see the Object?               |
|    (Profiles, Permission Sets - CRED)                       |
+-------------------------------------------------------------+
| 3. FIELD LEVEL: Can the user see the Field?                 |
|    (Field-Level Security)                                   |
+-------------------------------------------------------------+
| 4. RECORD LEVEL: Can the user see THIS specific record?     |
|    +---------------------------------------------------+    |
|    | [ OWD (Baseline) ] <--- WE ARE HERE               |    |
|    |        |                                          |    |
|    |        v                                          |    |
|    | Role Hierarchy (Vertical Access)                  |    |
|    |        |                                          |    |
|    |        v                                          |    |
|    | Sharing Rules (Lateral Access)                    |    |
|    |        |                                          |    |
|    |        v                                          |    |
|    | Manual/Teams/Apex Sharing (Granular Access)       |    |
|    +---------------------------------------------------+    |
+-------------------------------------------------------------+
```

---

# 3. What is OWD?

### Definition
Organization-Wide Defaults (OWD) define the default level of access users have to each other’s records. They represent the "floor" of your security model. 

### Purpose
The purpose is to answer a single question: *"If a user does not own a record, and is not a manager of the owner, what level of access should they have?"*

### Metadata Representation
In the Salesforce Metadata API, OWD settings are stored within the `CustomObject` metadata type. Fields like `sharingModel` dictate the OWD (e.g., `ReadWrite`, `Read`, `Private`, `ControlledByParent`). External OWDs use `externalSharingModel`.

### Access Evaluation
Salesforce evaluates OWD before any other record-level security mechanism. It is impossible to use standard sharing tools to *restrict* access further than what OWD allows (with the modern exception of Restriction Rules, which act as a filter).

---

# 4. Why OWD is Important

* **Data Protection:** Prevents unauthorized viewing of sensitive data (e.g., HR records, financial data).
* **Controlled Visibility:** Ensures users only see information relevant to their roles, reducing screen clutter and confusion.
* **Security Compliance:** Satisfies strict regulatory requirements (like HIPAA, GDPR, or SOX) by enforcing data silos at the database level.
* **Principle of Least Privilege:** OWD enforces the enterprise security mantra: "Grant only the minimum access necessary for users to perform their jobs."

---

# 5. Record-Level Security Architecture

OWD is the starting point. All other sharing mechanisms build upon it. 

```text
[ RESTRICTION RULES ] -> Can filter access regardless of sharing (The Ceiling)
       ^
       |
[ APEX SHARING ] -> Complex, programmatic exceptions (Share table)
       ^
       |
[ TEAMS & MANUAL SHARING ] -> Ad-hoc exceptions (Account/Opp Teams)
       ^
       |
[ SHARING RULES ] -> Rule-based exceptions (Public Groups, Criteria, Ownership)
       ^
       |
[ ROLE HIERARCHY ] -> Vertical exceptions (Managers see Subordinates' data)
       ^
       |
[ ORGANIZATION-WIDE DEFAULTS ] -> The Baseline (The Floor)
```

---

# 6. OWD Access Models

### Private
* **Definition:** Only the record owner and users above them in the role hierarchy (if enabled) can access the record.
* **Behavior:** Users cannot see records owned by other users.
* **Advantages:** Maximum data security.
* **Limitations:** Requires extensive sharing rules and role hierarchy configuration for collaboration.
* **Business Use Case:** HR Employee records, personalized sales pipelines.

### Public Read Only
* **Definition:** All users can view the record, regardless of ownership, but only the owner/managers can edit it.
* **Behavior:** Global read access; restricted write access.
* **Advantages:** High transparency without risking data integrity.
* **Limitations:** Users cannot help update records they don't own.
* **Business Use Case:** Product Catalogs, Reference Data, closed Won Opportunities.

### Public Read/Write
* **Definition:** All users can view and edit all records.
* **Behavior:** Maximum collaboration; ownership only dictates who can delete or change ownership.
* **Advantages:** Zero friction for collaboration.
* **Limitations:** High risk of data overwriting and compliance breaches.
* **Business Use Case:** Public Ideation boards, generic task trackers.

### Controlled By Parent
* **Definition:** Access to the record is strictly dictated by access to its parent record.
* **Behavior:** Inherits the OWD of the Master object.
* **Advantages:** Streamlines security architecture.
* **Limitations:** Removes the ability to independently share the child record.
* **Business Use Case:** Quote Line Items (controlled by Quote), Custom related tracking metrics.

---

# 7. Private Access Model

The **Private** model establishes a strict silo. 

* **Record Ownership:** Ownership is paramount. The `OwnerId` field dictates baseline visibility. 
* **Visibility Rules:** If user A owns Record X, User B cannot even see Record X in lists, global search, or reports.
* **Sharing Requirements:** To open access, you must configure Sharing Rules, Role Hierarchy, or Apex Sharing. The system leverages underlying Object Share tables (e.g., `AccountShare`, `CustomObject__Share`) to store these exceptions.

---

# 8. Public Read Only

**Public Read Only** is a hybrid state promoting transparency.

* **Visibility Behavior:** The `OwnerId` still exists, but the platform overrides visibility restrictions globally. 
* **Editing Restrictions:** While a user can view the record, the system checks Object-Level security (Profiles/Perm Sets) AND Record-Level ownership/sharing before allowing an *Edit*. If the user doesn't own the record or have a sharing rule granting `Edit`, the Save button will result in an "Insufficient Privileges" error.
* **Business Scenarios:** A corporate directory custom object where all employees need to find phone numbers, but only the specific employee (owner) or HR can update the details.

---

# 9. Public Read/Write

**Public Read/Write** is the most permissive standard model.

* **Access Behavior:** Grants `Read` and `Edit` access to the entire organization. 
* **Collaboration Use Cases:** Startups or small teams where everyone does everything, or non-sensitive objects like a generic "Support Feedback" object.
* **Risks:** * No data privacy.
    * Harder to track malicious or accidental edits without Field Audit Tracking.
    * You cannot use Sharing Rules because access is already completely open.

---

# 10. Controlled By Parent

This model is unique to **Master-Detail relationships** (and some standard relationships).

* **Parent-Child Relationships:** The child record has no `OwnerId` field. Ownership is derived from the Master record.
* **Access Inheritance:** If a user has `Read` access to the Parent, they have `Read` access to the Child. 
* **Special Note:** On Custom Master-Detail relationships, you can specify if users need `Read` or `Read/Write` access on the Master to create, edit, or delete related Child records (Sharing Setting: "Read/Write: Allows users with at least Read access to the Master record to create, edit, or delete related Detail records").

---

# 11. OWD for Standard Objects

Standard objects have unique OWD behaviors compared to custom objects.

| Object | Available OWD Settings | Special Considerations |
| :--- | :--- | :--- |
| **Account** | Private, PRO, PRW | Also controls baseline access to Contacts and Opportunities unless specified otherwise. |
| **Contact** | Controlled by Parent, Private, PRO, PRW | If Controlled by Parent, uses Account OWD. |
| **Opportunity** | Private, PRO, PRW | Tied heavily to Account access. Account owners get full access to child Opps. |
| **Case** | Private, PRO, PRW, Public Read/Write/Transfer | Transfer allows users to change ownership of the case. |
| **Lead** | Private, PRO, PRW, Public Read/Write/Transfer | Transfer allows users to reassign leads. |
| **Campaign** | Private, PRO, PRW, Full Access | Full Access allows all users to delete campaigns (if Profile allows). |

---

# 12. OWD for Custom Objects

For custom objects, OWD settings are straightforward: Private, Public Read Only, or Public Read/Write. 

* **Custom Object Security:** By default, when a custom object is created, its OWD is set to **Public Read/Write**. 
* **Master-Detail:** If the custom object is the detail side of a Master-Detail relationship, its OWD is hardcoded to **Controlled by Parent**.

---

# 13. Grant Access Using Hierarchies

This is a critical checkbox next to the OWD setting.

* **What it is:** Determines whether users higher up in the Role Hierarchy automatically gain access to records owned by users below them.
* **Standard Objects:** For most Standard Objects (Account, Contact, Opportunity, Case), this checkbox is checked and **locked**. You cannot disable it. Managers will always see their subordinates' records.
* **Custom Objects:** For Custom Objects, you **can** uncheck this box.
* **When to Disable (Custom Objects):** Use this for highly sensitive internal compliance objects, like anonymous "Whistleblower Complaints" or "Peer Reviews," where even the CEO or an IT Director shouldn't see a lower-level employee's submitted record unless explicitly shared.

---

# 14. Internal Access vs External Access

Salesforce provides separate OWD columns for Internal vs. External users to protect data in Experience Cloud (Communities).

### Internal Users
Employees with standard Salesforce licenses.

### External Users
Partner Users, Customer Users, and Experience Cloud Users.

### Architecture Rule
**External OWD must always be equal to or more restrictive than Internal OWD.**
You cannot have Internal OWD as Private and External OWD as Public Read/Write.

```text
[ INTERNAL OWD ] >= [ EXTERNAL OWD ]
Private          -> Private
Public Read Only -> Private OR Public Read Only
```

---

# 15. OWD Evaluation Process

When a user requests a record, Salesforce's Sharing Architecture evaluates access in the following order. If access is granted at any stage, the system allows it (unless a Restriction Rule blocks it).

1.  **Record Ownership:** Is the user the `OwnerId`? (If Yes -> Full Access).
2.  **Object/Field Security:** Does the user's Profile/Perm Set allow reading this object? (If No -> Block).
3.  **OWD Evaluation:** Is the OWD Public Read/Write or Public Read Only? (If Yes -> Grant Access based on OWD).
4.  **Role Hierarchy:** Is "Grant Access Using Hierarchies" enabled, and is the user above the owner in the Role Hierarchy? (If Yes -> Grant Access).
5.  **Sharing Rules:** Does an Ownership-based or Criteria-based sharing rule grant this user (or a Public Group they are in) access? (If Yes -> Grant Access).
6.  **Team Sharing:** Is the user on the Account/Opportunity/Case Team? (If Yes -> Grant Access).
7.  **Apex/Manual Sharing:** Is there a record in the Object's `__Share` table for this user? (If Yes -> Grant Access).
8.  **Restriction Rule Evaluation:** Does a Restriction Rule apply to this user? (If Yes -> Filter/Block access regardless of steps 1-7).
9.  **Final Access Decision:** Access Granted or "Insufficient Privileges".

---

# 16. OWD and Role Hierarchy

The Role Hierarchy works hand-in-hand with OWD. 
If OWD is Public Read/Write, the Role Hierarchy is practically irrelevant for record visibility (though still used for forecasting and reporting). 
If OWD is Private, the Role Hierarchy is the primary engine for vertical management access. A Regional Manager role will inherit access to all records owned by Sales Reps in the roles below them.

---

# 17. OWD and Sharing Rules

Sharing rules are used to make lateral exceptions to restrictive OWDs.

* **Ownership-Based Sharing:** "If a record is owned by [Role A], share it with [Role B] with [Read/Write] access."
* **Criteria-Based Sharing:** "If `Industry` = 'Technology', share the record with [Public Group: Tech Specialists]."
* **Public Groups:** A collection of users, roles, and subordinates used to simplify sharing rule administration.

*Note: You can only create Sharing Rules to OPEN access. You cannot use them to restrict access.*

---

# 18. OWD and Teams

Teams provide ad-hoc sharing for specific standard objects when OWD is Private.

* **Account Teams:** Allows the Account Owner to manually add colleagues (e.g., a Solutions Engineer, a Legal Advisor) to specific accounts, granting them Read or Read/Write access to the Account and related Opportunities/Cases.
* **Opportunity Teams:** Specific to individual deals.
* **Case Teams:** Allows collaboration on complex support tickets.

---

# 19. OWD and Apex Managed Sharing

When business logic is too complex for standard Sharing Rules, Architects use **Apex Managed Sharing**.

* **Programmatic Sharing:** Code is written to insert records into the Share object (e.g., `Job_Application__Share`).
* **RowCause:** A critical concept in Apex sharing. Developers define Apex Sharing Reasons (e.g., `Hiring_Manager__c`). If the record owner changes, standard manual sharing is wiped out, but sharing tied to an Apex `RowCause` survives.

```apex
// Example: Creating an Apex Share record
Job_Application__Share jobShare = new Job_Application__Share();
jobShare.ParentId = jobId; // The record being shared
jobShare.UserOrGroupId = hiringManagerId; // The user getting access
jobShare.AccessLevel = 'Edit';
jobShare.RowCause = Schema.Job_Application__Share.RowCause.Hiring_Manager__c;
insert jobShare;
```

---

# 20. OWD and Restriction Rules

Restriction Rules changed the Salesforce security paradigm.

* **Opening vs Restricting:** OWD, Roles, and Sharing Rules *open* access. Restriction Rules *restrict* access.
* **Security Evaluation Order:** Restriction Rules apply *after* all other sharing calculations. If OWD is Public Read/Write, but a Restriction Rule says a specific user can only see records where `Confidential__c = false`, that user will be blocked from viewing confidential records, regardless of the OWD.

---

# 21. Enterprise Security Design Scenarios

### Automotive CRM

| Object | Recommended OWD | Justification |
| :--- | :--- | :--- |
| **Dealers (Account)** | Public Read Only | All internal staff need to see Dealer info, but only Regional Managers edit. |
| **Vehicles** | Public Read Only | Inventory must be visible to all for selling, but edits are restricted to logistics. |
| **Warranty Claims** | Private | Contains sensitive financial/legal data. Share only with specific adjudicators via Sharing Rules. |
| **Spare Orders** | Private | Sales reps should only see their own dealer's orders to prevent poaching/data leaks. |
| **Invoices** | Controlled By Parent | Tied directly to the Order. |

### Banking Organization

| Object | Recommended OWD | Justification |
| :--- | :--- | :--- |
| **Customer Records** | Private | Strict banking regulations (PII, Financial Data). |
| **Loan Applications** | Private | Extremely sensitive. Access granted only via Apex Sharing based on Loan Officer assignment. |

### Insurance Company

| Object | Recommended OWD | Justification |
| :--- | :--- | :--- |
| **Policies** | Private | Agents only see policies they write. Managers see region via Role Hierarchy. |
| **Claims** | Private | Handled by distinct Claims Adjuster public groups via Criteria-based sharing (by Claim Type). |

---

# 22. OWD Design Patterns

### Private First Strategy
**The Gold Standard for Enterprise.** Set everything to Private. Use Public Groups, Roles, and Sharing rules to grant explicit access. Protects against accidental data leaks.

### Collaboration Strategy
Set non-sensitive objects to Public Read/Write. Use for internal ticketing, generic tasks, or company-wide wikis. Reduces administrative overhead.

### Regional Access Strategy
OWD Private. Create Public Groups for "North America", "EMEA", "APAC". Use Criteria-based sharing on the `Region__c` field to share records laterally within the same region.

### Department-Based Security Strategy
OWD Private. Use Ownership-based sharing rules to share records among peers within the same department (e.g., all users in the "Marketing Role" share Read/Write access with each other).

---

# 23. Performance Considerations

* **Large Data Volumes (LDV):** Changing an OWD on an object with millions of records triggers a massive asynchronous sharing recalculation. This can lock tables and take hours or days.
* **Role Hierarchy Impact:** A deep, complex Role Hierarchy combined with Private OWD forces the system to calculate thousands of share records. Keep the hierarchy as flat as possible.
* **Sharing Recalculation:** Always use **Deferred Sharing Recalculation** when doing bulk data loads or major architectural changes to prevent system timeouts.

---

# 24. OWD Limits and Considerations

| Feature | Limitation / Consideration |
| :--- | :--- |
| **Master-Detail** | Child OWD is locked to 'Controlled by Parent'. You cannot grant separate access. |
| **External OWD** | Must be equal to or more restrictive than Internal OWD. |
| **Changing OWD** | Changing from Public to Private triggers a full sharing recalculation (locks records). |
| **Standard Objects** | "Grant Access Using Hierarchies" cannot be disabled for core objects (Account, Opp). |
| **User Object** | The User object has its own OWD. 'Private' prevents users from seeing each other in the directory. |

---

# 25. Common Mistakes

1.  **Using Public Read/Write as a shortcut:** Admins often set OWD to Public Read/Write to fix immediate "I can't see this record" errors, creating massive security vulnerabilities.
2.  **Over-reliance on Sharing Rules:** Creating hundreds of sharing rules instead of designing a proper Role Hierarchy impacts performance.
3.  **Ignoring External OWD:** Deploying Communities without hardening External OWD, leading to customer data leaks.
4.  **Poor Role Design:** Designing the Role Hierarchy based on HR reporting lines rather than *Data Access* requirements.

---

# 26. Best Practices

* **Private by Default:** Always start with a Private OWD during architecture design. Justify any relaxation to Public.
* **Least Privilege Principle:** Only grant the access absolutely required.
* **Security Reviews:** Run the Salesforce Health Check regularly to audit baseline security.
* **Governance Standards:** Document every Sharing Rule and its business justification. If the justification expires, delete the rule.
* **Use Public Groups:** Never assign Sharing Rules to specific users. Always use Public Groups, even if there is only one user in the group today.

---

# 27. Troubleshooting Record Access Issues

### User Cannot See Record
1.  Check Object-level access (Profile/Perm Set).
2.  Check OWD. If Private, check ownership.
3.  Use the **Sharing Button** on the record to view exactly *why* a user has access (or doesn't).

### User Can See Too Many Records
1.  Check OWD (Is it Public?).
2.  Check if they have "View All Data" on their Profile.
3.  Check if they are high in the Role Hierarchy.

### Sharing Not Working
Ensure Sharing Recalculation isn't suspended or currently processing in the background (check Apex Jobs).

---

# 28. Auditing and Compliance

### Security Audits
Use Salesforce Shield Event Monitoring to track who is querying records they shouldn't be.

### Access Reviews
Regularly review the "Sharing" button on sensitive records.

### GDPR / SOX Considerations
For GDPR (Right to be Forgotten, Data Privacy), a strict **Private** OWD with granular consent-based Apex sharing is often required. For SOX, financial records must strictly adhere to separation of duties (Private OWD + strict Profiles).

---

# 29. Modern Salesforce Security Architecture

How everything works together to form the "Access Equation":

```text
( Profile/Perm Set [CRUD] ) + ( OWD [Baseline] + Roles + Sharing Rules + Teams/Apex [Additions] ) - ( Restriction Rules [Filters] ) = FINAL RECORD VISIBILITY
```

If Profile = `Read/Edit`
AND OWD = `Private`
AND Sharing Rule = `Read`
AND Restriction Rule = `None`
Result -> User can **View** the record, but cannot Edit it.

---

# 30. Interview Questions & Answers

### Beginner Questions
**Q: What is the difference between Profile and OWD?**
*A: A Profile controls what you can do with an Object (e.g., Read Account). OWD controls which specific records you can see within that Object (e.g., Only your Accounts).*

### Intermediate Questions
**Q: Can I use a Sharing Rule to hide a record?**
*A: No. Standard sharing tools (OWD, Sharing Rules, Roles) only open up access. To hide or filter access, you must use Restriction Rules or Scoping Rules.*

### Advanced Questions
**Q: A custom object has OWD set to Private. "Grant Access Using Hierarchies" is disabled. How can a manager see their subordinate's record?**
*A: The manager would not see it automatically. Access must be granted via a Sharing Rule, Manual Share, or Apex Share.*

### Architect-Level Questions
**Q: You are changing the OWD of `Account` from Public Read/Write to Private in an org with 50 million records. What is your deployment strategy?**
*A: This is an LDV (Large Data Volume) operation. It will trigger a massive synchronous lock and background recalculation. I would defer sharing calculations, perform the OWD metadata update during a weekend maintenance window, and then manually trigger or monitor the recalculation batch to ensure it completes before Monday morning, communicating downtime to stakeholders.*

---

# 31. Revision Summary

* **OWD:** The baseline/floor of record-level security.
* **Private:** Only owner + hierarchy (if enabled) can see the record.
* **Public Read Only:** Everyone can see; only owner/hierarchy can edit.
* **Public Read/Write:** Everyone can see and edit.
* **Controlled By Parent:** Used in Master-Detail; child inherits parent's security.
* **Role Hierarchy:** Opens vertical access.
* **Sharing Rules:** Opens lateral access.
* **Restriction Rules:** Filters/blocks access.
* **Best Practice:** Always default to Private and grant least privilege.