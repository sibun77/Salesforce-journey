# Field-Level Security (FLS) in Salesforce

---

## 1. Introduction

### What is Field-Level Security?
Field-Level Security (FLS) is a core component of the Salesforce security model that controls whether a user can view, edit, or see the value of a specific field on an object, regardless of their access to the object or the record itself.

### Why Salesforce Introduced FLS
In enterprise architectures, multiple departments often use the same objects (like `Account` or `Contact`) but require different levels of data visibility. FLS was introduced to provide granular, attribute-level access control, eliminating the need to create duplicate custom objects just to hide specific data points from certain users.

### Importance of Protecting Sensitive Data
Protecting sensitive data is critical for regulatory compliance (GDPR, HIPAA, SOX, PCI-DSS) and preventing internal data leaks. FLS enforces the **Principle of Least Privilege**, ensuring users only see the data essential for their specific job functions.

### Real-World Examples
* **Customer PAN Number:** Visible only to Finance and Compliance; hidden from Sales.
* **Aadhaar Number:** Encrypted and hidden from all users except verified HR/Background Check profiles.
* **Salary Information:** Visible only to HR and the employee's direct manager.
* **Dealer Commission:** Visible to regional managers, hidden from external portal users.
* **Warranty Settlement Amount:** Visible to claims adjudicators, read-only for field service agents.
* **Bank Account Details:** Hidden from support agents; visible to payroll/finance.

---

## 2. Salesforce Security Architecture Overview

Salesforce evaluates security in a strict hierarchy, from the broadest access down to the most granular.

1.  **Organization-Level Security:** Controls *who* can log in and *when/where* (IP Ranges, Login Hours, MFA).
2.  **Object-Level Security (CRUD):** Controls whether a user can Create, Read, Update, or Delete records of a specific object type (e.g., "Can I see the Account object?").
3.  **Record-Level Security (Sharing):** Controls *which specific records* a user can access (OWD, Roles, Sharing Rules).
4.  **Field-Level Security (FLS):** Controls *which specific fields* on those accessible records the user can see or edit.

### Architecture Diagram: The Security Funnel

```text
[ Login / Org Security ]  <-- Can the user log in?
          |
          v
[ Object Security (CRUD)] <-- Can they see the Object table?
          |
          v
[ Record Security (OWD) ] <-- Which specific rows (records) can they see?
          |
          v
[ Field Security (FLS)  ] <-- WHICH COLUMNS (fields) on those rows can they view/edit?
```

---

## 3. What is Field-Level Security?

### Definition
FLS is a metadata-driven security layer that explicitly defines the visibility (`Read`) and editability (`Edit`) of individual fields across the entire Salesforce platform, including the UI, API, Apex, SOQL, and Reports.

### Purpose
To compartmentalize data within a record so that a single record can securely serve multiple business units without exposing unauthorized data.

### Metadata Structure
In the Salesforce Metadata API, FLS is represented within Profiles and Permission Sets under the `FieldPermissions` array. It consists of two boolean flags:
* `readable` (Visible)
* `editable` (Read-Only vs. Read/Write)

### Access Evaluation
FLS is strictly enforced at the database and API routing layer for all declarative features. However, Apex runs in **System Mode** by default, meaning custom code bypasses FLS unless explicitly enforced by the developer.

### Independence from OWD and Roles
FLS is completely independent of Organization-Wide Defaults (OWD) and the Role Hierarchy.
* *Example:* If OWD for Account is "Public Read/Write", a user can edit any Account record. However, if their FLS for `AnnualRevenue` is set to "Read-Only", they still cannot edit that specific field on any Account.

---

## 4. Why FLS is Important

* **Data Privacy:** Prevents unauthorized internal employees from viewing Personally Identifiable Information (PII).
* **Compliance Requirements:** Essential for adhering to laws that mandate strict access controls on financial and health data.
* **Principle of Least Privilege:** Users are granted only the minimum data access necessary to perform their duties, reducing the blast radius of compromised accounts.
* **Protection of Sensitive Business Data:** Secures proprietary formulas, internal scoring mechanisms, and financial margins from competitive leakage.

---

## 5. FLS Access Types

| Access Type | Behavior | Advantages | Limitations | Business Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Visible** | User can view and edit the field. | Maximum utility for data entry. | Highest risk for data corruption/unauthorized changes. | Sales reps updating "Opportunity Stage". |
| **Read Only** | User can see the field but cannot edit it. | Prevents accidental modifications while maintaining visibility. | User cannot fix incorrect data even if they spot an error. | Support reps viewing "Total Lifetime Value". |
| **Hidden** | User cannot see the field anywhere (UI, API, Reports). | Maximum security; completely removes the field from the user's schema. | Can cause integration or Apex errors if not handled properly. | Standard users restricted from seeing `Salary__c`. |

---

## 6. FLS Evaluation Process

Salesforce calculates a user's effective field access dynamically at runtime:

1.  **User Login:** The system identifies the user and their base Profile.
2.  **Profile Evaluation:** Evaluates the baseline FLS assigned via the Profile.
3.  **Permission Set Evaluation:** Aggregates any additional FLS granted by assigned Permission Sets (additive).
4.  **Permission Set Group Evaluation:** Evaluates aggregated PSGs, specifically looking for **Muting Permission Sets** (subtractive).
5.  **Effective FLS Calculation:** Base Profile + Permission Sets - Muted Permissions.
6.  **Final Field Access Decision:** The field is rendered as Editable, Read-Only, or Hidden.

```text
[Base Profile FLS] (+) [Permission Sets FLS] (-) [Muting Permission Sets] = Effective Field Access
```

---

## 7. FLS and Profiles

### Field Permissions in Profiles
Historically, Profiles were the primary container for FLS. A profile establishes the absolute *baseline* field access for a group of users.

### Security Implications
Relying solely on Profiles for FLS leads to "Profile Explosion"—creating hundreds of profiles just to handle minor variations in field visibility. This creates an unmaintainable administrative burden.

---

## 8. FLS and Permission Sets

### Extending Field Access
Permission Sets *add* permissions. They cannot restrict permissions already granted by a Profile. If a Profile grants "Edit" access to a field, a Permission Set cannot make it "Read-Only".

### Modern Salesforce Security Strategy
The modern best practice is:
1.  **Minimum Access Profile:** Grants baseline login access and hides almost all sensitive fields.
2.  **Task-Based Permission Sets:** Deployed to specific users to incrementally grant "Visible" or "Read-Only" access to specific fields based on their current job requirements.

---

## 9. FLS and Permission Set Groups

### Aggregated Permissions
Permission Set Groups (PSGs) bundle multiple Permission Sets together for specific job personas (e.g., "Tier 1 Support Agent").

### Muting Implications
PSGs include **Muting Permission Sets**. This is the *only* declarative way to remove FLS mathematically without altering the base profile.
* *Example:* A user gets "Edit" access to "Discount %" from a general "Sales Tools" Permission Set. A Muting PS inside the "Junior Sales" PSG can mute that edit access specifically for junior users, making it Read-Only for them.

---

## 10. FLS vs Object Permissions

| Feature | Field-Level Security (FLS) | Object Permissions (CRUD) |
| :--- | :--- | :--- |
| **Purpose** | Controls access to individual attributes/columns. | Controls access to the table/entity itself. |
| **Scope** | Granular (Field Level). | Broad (Object Level). |
| **Security Layer** | Data attribute layer. | Entity layer. |
| **Access Control** | Visible, Read-Only, Hidden. | Create, Read, Edit, Delete, View All, Modify All. |

---

## 11. FLS vs Record-Level Security

| Feature | Field-Level Security (FLS) | Record-Level Security (Sharing) |
| :--- | :--- | :--- |
| **OWD** | N/A (Does not apply to fields). | Sets the baseline access for records. |
| **Roles** | N/A (Does not affect FLS). | Opens up record access vertically (managers see reports' records). |
| **Sharing Rules** | N/A (Does not affect FLS). | Opens up record access laterally (group A sees group B's records). |
| **FLS** | Determines visibility of *columns* on the accessible records. | Determines which *rows* the user can access. |

---

## 12. FLS in Salesforce UI

* **Page Layouts:** If FLS hides a field, it is automatically removed from the Page Layout, regardless of the admin's layout configuration.
* **Dynamic Forms:** Respects FLS natively. If a user lacks FLS, the dynamic field component simply does not render.
* **Related Lists:** Columns in related lists drop hidden fields automatically.
* **Standard UI Behavior:** The standard Salesforce UI *always* strictly enforces FLS.

---

## 13. FLS in Lightning Experience

* **Lightning Record Pages:** Fields omitted by FLS will leave no awkward blank spaces; the standard Lightning UI collapses gracefully.
* **Standard Components:** Standard base components (`lightning-record-form`, `lightning-record-view-form`) automatically enforce FLS without custom code.

---

## 14. FLS in Apex

### System Mode vs User Mode
By default, Apex classes execute in **System Mode**. This means Apex ignores user CRUD and FLS settings. It can query, view, and modify all fields, even those hidden from the running user.

### Security Risks
If developers do not explicitly enforce FLS in their Apex controllers, they risk exposing sensitive data via custom LWC/Aura components or APIs, leading to critical data leaks.

### Why Apex Doesn't Auto-Enforce
Apex is often used for system operations (e.g., automated lead routing, complex backend financial calculations) where the business logic *must* read and write fields that the user shouldn't manipulate directly.

---

## 15. Security.stripInaccessible()

### Purpose
An Apex method that sanitizes sObject lists by stripping out fields the running user does not have permission to access, preventing FLS bypasses during data operations.

### AccessTypes
* `AccessType.READABLE`: Removes fields the user cannot view.
* `AccessType.CREATABLE`: Removes fields the user cannot insert.
* `AccessType.UPDATABLE`: Removes fields the user cannot update.
* `AccessType.UPSERTABLE`: Combines Create/Update checks.

### Apex Example
```java
List<Account> accountsToUpdate = new List<Account>{
    new Account(Id = '001...', Name = 'Acme', Secret_Margin__c = 50)
};

// Strips Secret_Margin__c if the user lacks Edit FLS
SObjectAccessDecision decision = Security.stripInaccessible(
    AccessType.UPDATABLE, 
    accountsToUpdate
);

// Safe update: Secret_Margin__c is ignored if user doesn't have access
update decision.getRecords(); 
```

---

## 16. WITH SECURITY_ENFORCED

### Purpose
An SOQL clause that automatically enforces field-level and object-level security during query execution.

### Limitations
It is a "hard fail" mechanism. If the user lacks access to even *one* field in the SELECT statement, the entire query throws a `System.QueryException`.

### Example
```java
// Throws System.QueryException if the user cannot read Secret_Margin__c
List<Account> secureAccounts = [
    SELECT Id, Name, Secret_Margin__c 
    FROM Account 
    WITH SECURITY_ENFORCED
];
```

*Comparison:* `stripInaccessible()` degrades gracefully (removes the field and continues); `WITH SECURITY_ENFORCED` fails the entire transaction.

---

## 17. User Mode Database Operations

### WITH USER_MODE
Introduced by Salesforce to simplify secure database operations. It natively enforces FLS, CRUD, and sharing rules within Apex.

### USER_MODE DML & Queries
```java
// Secure Query
List<Contact> contacts = [SELECT Id, LastName, SSN__c FROM Contact WITH USER_MODE];

// Secure DML
Account acc = new Account(Name = 'Safe Account', Rating = 'Hot');
insert as user acc; // Fails immediately if user lacks Create on Account or Edit FLS on Rating
```

---

## 18. FLS in SOQL

If FLS is not enforced in SOQL (via `WITH SECURITY_ENFORCED`, `WITH USER_MODE`, or `stripInaccessible`), a user with access to an Apex-backed UI can extract hidden fields.

*Secure Query Example:*
```java
public static List<Opportunity> getOpps() {
    return [SELECT Amount, ExpectedRevenue FROM Opportunity WITH USER_MODE];
}
```

---

## 19. FLS in DML Operations

Always enforce FLS before performing Insert, Update, Upsert, or Delete operations in Apex.

```java
// Traditional explicit check (Legacy/Verbose)
if (Schema.sObjectType.Contact.fields.Phone.isUpdateable()) {
    update contactList;
}

// Modern Approach (Cleaner and Safer)
update as user contactList;
```

---

## 20. FLS in LWC

* **Lightning Data Service (LDS):** Components like `lightning-record-form` and standard wire adapters (`@wire(getRecord)`) automatically enforce FLS. No extra Apex is needed.
* **UI API:** Automatically respects FLS.
* **Apex Calls:** If an LWC calls an `@AuraEnabled` Apex method, that method runs in System Mode by default. You **must** secure the Apex method using `WITH USER_MODE` or `stripInaccessible()`.

### Example LWC Apex Controller
```java
@AuraEnabled(cacheable=true)
public static List<Account> getAccounts() {
    // MUST use WITH USER_MODE so the LWC doesn't expose hidden fields
    return [SELECT Id, Name, AnnualRevenue FROM Account WITH USER_MODE];
}
```

---

## 21. FLS in Aura Components

Similar to LWC, Aura respects FLS natively when using `force:recordData`. However, any data fetched via Server-side controllers (Apex) requires manual FLS enforcement by the developer.

---

## 22. FLS in Flow

* **Screen Flow:** Runs in the context of the user by default. Respects FLS natively.
* **Record-Triggered Flow:** Runs in System Context by default. It can evaluate and update fields the user cannot normally see, which is useful for automated backend logic.
* **Autolaunched Flow:** Depends on how it is invoked (System or User context).

---

## 23. FLS in APIs

* **REST, SOAP, Bulk API:** All standard APIs authenticate as a specific user and **strictly enforce** that user's FLS automatically. An external system cannot bypass FLS using standard APIs.
* **Composite API:** Follows the exact same strict FLS enforcement rules.

---

## 24. Sensitive Data Protection

* **PII Data:** Fields like Birthdate, Personal Email, or SSN should be hidden from general users via FLS and exposed only to HR/Support via specific Permission Sets.
* **Financial Data:** Credit card tokens, bank details, or internal profitability margins.
* **Medical Data:** HIPAA requires strict FLS auditing for fields containing diagnosis codes or treatment history.

---

## 25. Enterprise Security Design Scenarios

### Automotive CRM
* **Dealer Commission:** Read-Only for Sales Managers; Hidden from Dealers (Partner Community Users).
* **Warranty Settlement Amount:** Visible to Finance; Read-Only for Service Agents.

### Banking Organization
* **Account Balance:** Read-Only for Tellers; Hidden from Marketing.
* **Credit Score:** Visible to Underwriters; Hidden from front-line support.

### Insurance Company
* **Claim Approval Amount:** Visible to Claims Adjusters; Hidden from external Brokers.
* **Policy Premium:** Read-Only for Support; Visible to Underwriters.

---

## 26. Common Mistakes

* **Assuming Apex Enforces FLS:** Believing custom UI naturally respects FLS. *Solution:* Always use `WITH USER_MODE` in `@AuraEnabled` methods.
* **Ignoring `stripInaccessible()`:** Passing raw user input to DML operations. *Solution:* Sanitize lists prior to DML.
* **Hardcoding Access Logic:** Checking Profile names in code (`if(profileName == 'Admin')`) instead of checking FLS. *Solution:* Use Schema methods or Custom Permissions.
* **Overexposing Fields:** Adding sensitive fields to standard profiles blindly during deployment. *Solution:* Implement a Minimum Access Profile strategy.

---

## 27. Best Practices

1.  **Least Privilege Principle:** Default all new custom fields to hidden for all profiles except System Administrator.
2.  **Permission Set Strategy:** Group FLS by business function/task rather than job title (e.g., "Manage Employee Salary Data" vs "HR Manager").
3.  **Secure Coding Standards:** Mandate `WITH USER_MODE` or `stripInaccessible` in CI/CD pipeline scanners (like PMD or Checkmarx).
4.  **Data Classification:** Use Salesforce Data Classification metadata to tag FLS-sensitive fields for easier auditing.

---

## 28. Auditing and Compliance

* **GDPR / Data Privacy:** FLS proves to external auditors that customer data is segmented internally.
* **Field Audit Trail:** Tracks *changes* to fields, but FLS configuration tracks *access*. Both are required for SOX compliance.
* **Security Health Check:** Use Salesforce's built-in tools to measure baseline FLS compliance.

---

## 29. Troubleshooting FLS Issues

| Issue | Root Cause & Troubleshooting Steps |
| :--- | :--- |
| **User Cannot See Field** | Check Page Layout -> Check Profile FLS -> Check Permission Sets. |
| **User Cannot Edit Field** | FLS is Read-Only OR the Record is Locked (Approval Process) OR Page Layout is set to Read-Only. |
| **Apex Security Errors** | `System.QueryException` due to `WITH SECURITY_ENFORCED`. User lacks FLS on queried field. |
| **LWC Data Visibility Issues** | Component uses LDS but field is returning null. Verify FLS for the specific user/community profile. |

---

## 30. Modern Salesforce Security Architecture

In modern implementations, field access is built like a pyramid:

```text
  [ Effective FLS / Access ]            <-- Final Result User Experiences
             ^
[ Permission Set Group (Muting) ]       <-- Restrict Specific FLS exceptions
             ^
   [ Permission Sets ]                  <-- Grant specific FLS/CRUD based on Tasks
             ^
[ Minimum Access Profile ]              <-- Base Auth, Zero FLS/CRUD
```

---

## 31. Interview Questions & Answers

### Beginner
**Q: What happens if a field is hidden via FLS but added to a Page Layout?**
**A:** The field will not be visible to the user on the page layout. FLS is absolute and overrides Page Layout configurations.

### Intermediate
**Q: Can a user with "Public Read/Write" OWD edit a field if FLS is set to Read-Only?**
**A:** No. OWD controls access to the *record* (the row); FLS controls access to the *field* (the column). FLS restricts the edit action on that specific data point.

### Advanced
**Q: What is the difference between `WITH SECURITY_ENFORCED` and `Security.stripInaccessible()`?**
**A:** `WITH SECURITY_ENFORCED` throws an exception and fails the entire query if any requested field is inaccessible. `stripInaccessible()` gracefully removes the unauthorized fields from the sObject list and allows processing to continue with the remaining data.

### Architect-Level
**Q: How do you design an FLS strategy for a global deployment with varying regional compliance laws?**
**A:** Implement a Minimum Access Profile strategy. Keep all PII/Financial fields hidden at the base level. Create regional Permission Set Groups (e.g., `EU_Sales_PSG`, `US_Sales_PSG`). Apply regional FLS via Permission Sets within those groups, and use Muting Permission Sets to handle country-specific redactions within the region without needing new profiles.

---

## 32. Revision Summary

* **FLS:** Controls column/attribute visibility independently of record sharing.
* **Visible:** User can Read and Write.
* **Read Only:** User can View, but not edit.
* **Hidden:** Removed from UI, API, and Reports for the user.
* **`stripInaccessible()`:** Gracefully removes inaccessible fields from sObjects before DML or returning to UI.
* **`WITH SECURITY_ENFORCED`:** Fails SOQL queries entirely if FLS is breached.
* **User Mode (`WITH USER_MODE` / `as user`):** Enforces FLS natively in modern Apex DB operations.
* **Secure Apex:** Apex runs in System Mode. Developers MUST manually enforce FLS using the tools above.
* **Best Practices:** Default to Minimum Access Profiles, rely on Permission Sets/Groups, and enforce User Mode in all customer-facing Apex controllers.