# Field Types

## 1. Introduction

### What are Fields in Salesforce?
In Salesforce, a **field** represents a specific piece of information stored within an object (which acts as a database table). If an object is a spreadsheet, a field is a column. Fields define the structure of the data you collect, enabling organizations to capture business-critical information systematically.

### Why Fields are Important
Fields are the foundational building blocks of the Salesforce Data Model. They dictate data integrity, drive automation, inform reporting, and enforce security. Proper field architecture ensures the system scales efficiently while remaining user-friendly.

### Role of Fields in the Salesforce Data Model
Fields connect the user interface to the underlying database. They define the schema, enforce data validation rules at the database level, and establish complex relational structures between different business entities (objects) to create a cohesive data architecture.

### How Fields Store Business Data
Salesforce leverages a multi-tenant, metadata-driven architecture. Fields do not just store raw data; their definitions (metadata) determine how the data is rendered, validated, and secured across Web, Mobile, and API boundaries.

---

## 2. Salesforce Field Architecture



### Standard Fields
Salesforce provides pre-built "Standard Fields" on both standard objects (like Account, Contact) and custom objects. Common examples include `Name`, `CreatedDate`, `LastModifiedDate`, and `OwnerId`.

### Custom Fields
Custom fields are created by Administrators or Developers to capture unique business requirements that standard fields do not cover. 

### Metadata-Driven Architecture
Salesforce relies on a metadata-driven architecture. When you create a field, you are creating a metadata configuration rather than running a raw SQL `ALTER TABLE` statement. This metadata dictates the field's behavior, UI rendering, security, and API access instantly across the platform.

### How Field Definitions are Stored
Field definitions are stored as XML in the Salesforce underlying metadata repository. This metadata engine abstracts the physical database layer (Oracle/PostgreSQL) from the logical layer the developer interacts with.

### Field API Names
Every field has an API Name used in code (Apex, SOQL, LWC) and integrations. 
* Standard field API Names match their label (e.g., `Industry`).
* Custom field API Names automatically append `__c` (e.g., `Vehicle_Identification_Number__c`).
* Relationship fields append `__r` when traversing relationships in SOQL.

---

## 3. Standard Fields vs Custom Fields

| Feature | Standard Fields | Custom Fields |
| :--- | :--- | :--- |
| **Purpose** | Out-of-the-box fields for common business use cases. | Tailored fields for specific, unique organizational requirements. |
| **Creation** | Pre-built by Salesforce. | Created manually by Admins/Devs or via Metadata API. |
| **Modification** | Highly restricted. Cannot change data type or length. Can change Label, Help Text, and Picklist values. | Highly flexible. Can modify Type (with limitations), Length, Label, and Security. |
| **Deletion** | Cannot be deleted. Can only be removed from page layouts. | Can be deleted. (Stored in Recycle Bin for 15 days before hard deletion). |
| **Ownership** | Maintained by Salesforce across releases. | Maintained by the customer's deployment/admin team. |
| **API Name Example**| `AnnualRevenue` | `Tax_Identification_Number__c` |
| **Use Cases** | Core CRM tracking (Account Name, Email, Phone). | Custom business logic (Driver's License Number, VIP Status). |

---

## 4. Complete Salesforce Field Types Overview

| Category | Field Type | Description | Use Case | Advantages | Limitations |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Basic** | **Text** | Up to 255 characters. | Names, short identifiers. | Searchable, indexable. | Size limit. |
| | **Text Area** | Up to 255 chars on separate lines. | Short descriptions. | Multi-line support. | Not searchable in list views. |
| | **Long Text Area** | Up to 131,072 characters. | Notes, extensive details. | Massive storage. | Cannot be filtered in reports or SOQL `WHERE`. |
| | **Rich Text Area** | Formatted text, images, links. | HTML formatted descriptions. | Great UI experience. | Slower to load, un-indexable. |
| | **Number** | Real numbers. | Quantities, scores. | Supports math operations. | Precision limits. |
| | **Currency** | Financial values. | Prices, budgets. | Multi-currency support. | Tied to corporate currency rules. |
| | **Percent** | Percentages (rendered with %). | Discounts, margins. | Automatic UI formatting. | Easy to misconfigure decimal places. |
| | **Auto Number** | System-generated read-only sequence. | Invoice numbers, IDs. | Guarantees uniqueness. | Cannot be edited manually. |
| **Date/Time** | **Date** | Date only. | Birthdays, close dates. | Clean UI, time-zone agnostic. | No time precision. |
| | **Date/Time** | Date and Time. | SLA targets, meeting times. | Precise tracking. | Time zone conversion complexity. |
| | **Time** | Time of day without date. | Shift starts, business hours. | Lightweight scheduling. | Cannot be used in complex date math easily. |
| **Contact** | **Email** | Validates email format. | Customer/User emails. | Native validation, used in alerts. | Only basic regex validation. |
| | **Phone** | Formats phone numbers. | Contact numbers. | CTI Integration ready. | Format depends on locale. |
| | **URL** | Web address. | Websites, external links. | Clickable links. | Validates prefix only. |
| **Selection** | **Picklist** | Dropdown of defined values. | Statuses, Categories. | Standardizes data entry. | Hard to maintain with 1000+ values. |
| | **Multi-Select Picklist** | Select multiple values. | Interests, tags. | Flexible selection. | **Terrible for reporting and formulas.** |
| | **Checkbox** | Boolean (True/False). | Opt-ins, active flags. | Simple UI, fast queries. | Only two states (cannot be 'null'). |
| **Calculated**| **Formula** | Read-only calculation. | Ages, dynamic traffic lights. | Real-time, no code needed. | Cannot trigger automation, compile limits. |
| | **Roll-Up Summary**| Aggregates child records. | Total Invoice Amount. | Persisted to DB, fast. | Only works in Master-Detail. |
| **Relation** | **Lookup** | Loose link to another object. | Case to Contact. | Flexible, optional. | No automatic cascade delete. |
| | **Master-Detail** | Tightly coupled parent-child. | Invoice to Line Items. | Roll-ups, strict security. | Child cannot exist without parent. |
| | **External/Indirect**| Link to external data sources. | ERP ID linking. | Avoids data duplication. | Requires Salesforce Connect. |
| **Advanced** | **Geolocation**| Latitude/Longitude coordinates. | Dispatch tracking. | Native distance calculations. | Complex to query via standard SOQL. |
| | **Encrypted** | 128-bit encryption for text. | SSNs, Credit Cards. | High security (Shield). | Severe limitations in SOQL/Filters. |

---

## 5. Formula Fields

### What are Formula Fields?
Formula fields are read-only fields that automatically calculate a value based on other fields, expressions, or values. 

### Why they are used
They enforce standard calculations across the org without relying on code (Apex) or workflow rules, ensuring data consistency.

### Read-Only and Real-Time Calculation
Formulas do not store data in the underlying database layer. Instead, they calculate their value *at runtime* whenever the record is viewed, queried, or reported on. 

### Cross-Object Formulas
Formulas can span across objects via lookup or master-detail relationships, "reaching up" to 10 levels away (e.g., `Contact.Account.Owner.Name`).

### Formula Operators & Functions
* **Logical:** `IF()`, `AND()`, `OR()`, `CASE()`, `ISBLANK()`
* **Mathematical:** `+`, `-`, `*`, `/`, `ROUND()`, `MOD()`
* **Text:** `TEXT()`, `ISPICKVAL()`, `CONTAINS()`, `LEFT()`, `RIGHT()`
* **Date:** `TODAY()`, `NOW()`, `YEAR()`, `MONTH()`, `ADDMONTHS()`

---

## 6. Formula Field Real Project Scenarios

**1. Warranty Expiry Calculation (Date)**
Calculates if a warranty is expired based on purchase date and term.
```salesforce
Purchase_Date__c + (Warranty_Term_Months__c * 30)
```

**2. SLA Calculation (Logical & Visual)**
Displays a visual indicator based on SLA status using image resources.
```salesforce
IF( Days_Open__c > 5, 
    IMAGE("/img/samples/flag_red.gif", "Red Flag"), 
    IMAGE("/img/samples/flag_green.gif", "Green Flag")
)
```

**3. Discount Calculations (Currency)**
```salesforce
Total_Amount__c * (1 - Discount_Percentage__c)
```

**4. Customer Categorization (Text)**
```salesforce
IF(AnnualRevenue > 1000000, "Tier 1 - Enterprise", 
   IF(AnnualRevenue > 500000, "Tier 2 - Mid-Market", "Tier 3 - SMB"))
```

---

## 7. Roll-Up Summary Fields

### What are Roll-Up Summary Fields?
A Roll-Up Summary (RUS) field calculates values from related records, such as those in a related list. Unlike formulas, RUS fields *store* their data in the database, allowing them to be used in list view filters and triggers.

### Requirements & Relationship Dependency
They can **only** be created on the *Master* object in a Master-Detail relationship. 

### Supported Operations
* **COUNT:** Number of child records.
* **SUM:** Total of a specific numeric/currency field on child records.
* **MIN:** Lowest value of a field on child records.
* **MAX:** Highest value of a field on child records.

---

## 8. Roll-Up Summary Real Project Scenarios

* **Total Invoice Amount:** `SUM` of `Line_Item_Amount__c` on Invoice Line Items rolled up to the Invoice.
* **Total Warranty Claims:** `COUNT` of related Warranty Claims rolled up to the Vehicle object.
* **Earliest Enrollment:** `MIN` of `Enrollment_Date__c` on Student records rolled up to the Course object.
* **Largest Deal Size:** `MAX` of `Amount` on Opportunities rolled up to the Account.

---

## 9. Lookup Relationship



### What is a Lookup Relationship?
A loosely coupled relationship linking two objects together so you can "look up" one object from the related items on another object.

### Architecture & Behavior
* **Ownership:** The child record has its own Owner. It does not inherit the parent's owner.
* **Sharing/Security:** Security is independent. Having access to the parent does not guarantee access to the child, and vice versa.
* **Optional:** The lookup field is not required by default (though it can be made required via validation rules or page layouts).
* **Delete Behavior:** If the parent is deleted, the child remains. The lookup field on the child is simply cleared (default behavior), or you can block the deletion of the parent.

### Lookup Filters
Administrators can apply filters to restrict valid choices (e.g., "Only allow linking a Case to an Active Account").

---

## 10. Lookup Relationship Real Project Scenarios

* **Employee ↔ Vehicle:** An employee might be assigned a company car. If the car is retired (deleted), the Employee record must still exist.
* **Case ↔ Vendor:** A support case might involve a third-party vendor. Ownership and security of the Case and Vendor are entirely separate.
* **Claim ↔ Dealer:** A warranty claim might be processed by a specific dealer. 
* **Student ↔ Mentor:** A student has a mentor. The relationship is loose; mentors and students have independent security access.

---

## 11. Master-Detail Relationship



### What is a Master-Detail Relationship?
A strongly coupled relationship where the master (parent) dictates the existence, security, and ownership of the detail (child).

### Architecture & Behavior
* **Parent-Child Architecture:** The detail record cannot exist without the master. The relationship field is implicitly required.
* **Ownership Inheritance:** Detail records do not have an Owner field. They inherit the owner of the Master.
* **Security Inheritance:** Sharing rules and manual sharing cannot be applied to the detail record. Access to the child is determined entirely by access to the parent.
* **Cascade Delete:** If the Master is deleted, all Detail records are automatically deleted.
* **Reparenting:** By default, once a child is assigned to a parent, it cannot be moved to another parent. (Can be enabled via the "Allow Reparenting" setting).
* **Roll-Up Summaries:** Supported natively on the Master object.

---

## 12. Master-Detail Real Project Scenarios

* **Invoice → Invoice Line Item:** An invoice line item has no logical reason to exist without its parent invoice. If the invoice is deleted, lines must be deleted.
* **Order → Order Item:** Individual items belong exclusively to the parent order.
* **Course → Enrollment:** If a course is permanently deleted from the system, the enrollments for that course should also be removed.
* **Warranty Registration → Claim:** Claims roll up to the parent registration for easy SLA tracking.

---

## 13. Lookup vs Master-Detail

| Feature | Lookup Relationship | Master-Detail Relationship |
| :--- | :--- | :--- |
| **Coupling** | Loose | Tight |
| **Required** | Optional (by default) | Always Required |
| **Ownership** | Child has its own Owner | Child inherits Master's Owner |
| **Security/Sharing** | Independent | Inherited from Master |
| **Deletion Behavior**| Field is cleared, or deletion blocked | Cascade Delete (Child is deleted) |
| **Roll-Up Summaries**| Not supported natively (needs Apex/DLRS) | Natively supported on Master |
| **Reparenting** | Always allowed | Blocked by default (can enable) |
| **Limit per Object** | Up to 40 | Up to 2 |
| **Use Case Idea** | Linking independent entities (Case to Account) | Linking dependent entities (Order to Items) |

---

## 14. Relationship Architecture



### One-to-One (1:1)
Salesforce does not have a native 1:1 relationship type. It is simulated using a Lookup relationship where a unique constraint (or validation rule) ensures a parent can only have one child.

### One-to-Many (1:N)
The standard Lookup or Master-Detail setup. One Account can have many Contacts.

### Many-to-Many (M:N) & Junction Objects
A single record on Object A needs to relate to multiple records on Object B, and vice versa. This requires a **Junction Object** with two Master-Detail relationships.
* *Example:* `Student` and `Course`. A Student takes many Courses. A Course has many Students. 
* *Junction:* `Enrollment` (Master-Detail to Student, Master-Detail to Course).

---

## 15. Field Security

### Field-Level Security (FLS)
FLS controls whether a user can Read or Edit a specific field. It is the most robust way to secure data at the field level, superseding Page Layout configurations.

### Profiles vs. Permission Sets
Historically, FLS was heavily tied to Profiles. The modern Salesforce architectural best practice is to set a baseline of minimum FLS on the Profile, and open up access using **Permission Sets** and **Permission Set Groups**.

### Dynamic Forms
A UI-layer feature that allows Admins to place individual fields dynamically on Lightning Record Pages, setting visibility rules (e.g., "Only show 'Discount %' if the user is a Manager"). *Note: Dynamic Forms control UI visibility, not database FLS.*

---

## 16. Reporting Considerations

* **Reporting on Formula Fields:** Evaluated at runtime. High complexity formulas can cause reports to time out.
* **Reporting on Roll-Up Summaries:** Highly efficient because the data is physically stored and indexed.
* **Relationship Reporting:** Custom Report Types dictate how relationships are queried (e.g., "Accounts WITH Contacts" vs. "Accounts WITH OR WITHOUT Contacts").
* **Cross-Object Reporting:** You can pull parent fields directly into a child's report without a Custom Report Type using Lookups.

---

## 17. Field Limits

### General Limits Table (Enterprise Edition)

| Limit Type | Limit Allocation |
| :--- | :--- |
| Custom Fields per Object | 500 (Enterprise) / 900 (Unlimited) |
| Active Lookup Filters per Object | 5 |
| Master-Detail Relationships per Object | 2 |
| Lookup Relationships per Object | 40 |
| Roll-Up Summaries per Object | 25 (can be increased to 40 via Support) |
| Formula Compile Size | 5,000 bytes |

---

## 18. Performance Considerations

* **Formula Field Performance:** Because they evaluate at runtime, sorting list views or filtering reports by complex formula fields forces full table scans, impacting performance.
* **Large Data Volumes (LDV):** In orgs with millions of records, adding a new Roll-Up Summary field can take hours to calculate. Data skews (e.g., >10,000 children on one Master) cause locking issues.
* **Optimization:** Convert heavily filtered or sorted formulas into standard fields updated via Flow or Apex to enable database indexing.

---

## 19. Real Project Scenarios (Enterprise)

* **Banking System:** * *Requirement:* Track Customer (Account) and Bank Accounts. 
    * *Design:* Master-Detail. A Bank Account cannot exist without a Customer. `Total_Deposits__c` is a Roll-Up Summary on the Account.
* **Automotive Industry:**
    * *Requirement:* Track Dealerships (Accounts) and Vehicles.
    * *Design:* Lookup. A vehicle might be transferred between dealerships. Standard cascade delete is inappropriate.
* **Hospital Management System:**
    * *Requirement:* Track Doctors and Patients.
    * *Design:* Many-to-Many. A junction object `Appointment` connects them.

---

## 20. Common Mistakes

1.  **Overusing Formula Fields:** Leading to compile-size limit errors and slow report rendering.
2.  **Incorrect Relationship Choice:** Choosing Master-Detail just to get Roll-Up Summaries, inadvertently destroying security architecture.
3.  **Excessive Cross-Object Formulas:** Reaching up too many levels creates brittle metadata dependencies.
4.  **Poor Naming Conventions:** Naming a field `Date__c` instead of `Contract_Start_Date__c`.
5.  **Security Mistakes:** Hiding a field on the Page Layout but leaving it accessible via API/SOQL because FLS wasn't restricted.

---

## 21. Best Practices

* **Naming Standards:** Use clear, descriptive labels. Add `_ID__c` for external keys, `_Date__c` for dates.
* **Relationship Design:** Always default to Lookup unless Master-Detail behavior (security, deletion) is strictly required by the business process.
* **Formula Optimization:** Use `CASE()` instead of nested `IF()` statements to reduce compile size.
* **Security Design:** Deny by default. Provide baseline access via Profiles, grant FLS via Permission Sets.
* **Reporting Optimization:** Avoid multi-select picklists at all costs if metrics and grouping are needed.

---

## 22. Enterprise Architecture Considerations

* **Scalability:** When designing relationships, anticipate Large Data Volumes (LDV). Avoid Account Data Skew (attaching >10k child records to a single parent).
* **Data Governance:** Implement strict naming conventions and metadata descriptions. Use a data dictionary.
* **Global Implementations:** Leverage Custom Labels within Formula fields to ensure multi-language support (i18n) instead of hardcoding text strings.
* **Metadata Management:** Ensure managed package field limits are accounted for alongside custom field limits.

---

## 23. Interview Questions & Answers

### Beginner Questions
**Q: What is the difference between a custom field and a standard field?**
**A:** Standard fields are provided by Salesforce out-of-the-box (e.g., Name, Owner) and cannot be deleted. Custom fields are created by admins to meet specific business needs, append `__c` to their API names, and can be modified or deleted.

### Intermediate Questions
**Q: When would you use a Lookup instead of a Master-Detail relationship?**
**A:** Use a Lookup when the child record needs to exist independently of the parent, needs its own security/sharing settings, or needs its own owner. Master-Detail enforces strict security inheritance and cascade deletion.

### Advanced Questions
**Q: You need a Roll-Up Summary but the relationship is a Lookup. How do you solve this?**
**A:** You can use an autolaunched Flow (Record-Triggered) or an Apex Trigger to manually calculate and update the parent record. Alternatively, utilize an open-source tool like Declarative Lookup Rollup Summaries (DLRS).

### Architect-Level Questions
**Q: How do formula fields impact Large Data Volume (LDV) reporting performance, and what is the architectural remediation?**
**A:** Formulas calculate at runtime. If used as report filters or list view sorts, they prevent the database engine from using indexes, leading to full table scans and timeouts. Remediation involves replacing the formula with a standard physical field, populated by before-save trigger Flows or Apex, and requesting Salesforce Support to place a custom database index on that new physical field.

---

## 24. Revision Summary

* **Fields:** Core database columns defined by metadata.
* **Formula Fields:** Real-time, read-only calculations. Watch compile sizes and performance.
* **Roll-Up Summaries:** Database-persisted calculations (SUM, MIN, MAX, COUNT). Requires Master-Detail.
* **Lookup Relationships:** Loose coupling, independent security, optional parent.
* **Master-Detail Relationships:** Tight coupling, inherited security, cascade delete.
* **Security:** Always control data access via FLS (Permission Sets), not just UI page layouts.
* **Limits:** Be mindful of the 2 Master-Detail limit and 40 Lookup limit per object.
* **Best Practice:** Design for scalability; don't use Master-Detail solely for roll-up capabilities if the security model demands independent access.