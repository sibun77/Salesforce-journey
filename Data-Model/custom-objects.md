# Custom Objects

## 1. Introduction

### What are Custom Objects
In Salesforce, a Custom Object is a foundational database table created by a system administrator or developer to store information specific to a company's unique business processes. While Salesforce provides standard objects out-of-the-box, custom objects allow orgs to extend the data model limitlessly to capture industry-specific or proprietary data. 

### Why Salesforce Provides Custom Objects
Salesforce operates as a multi-tenant Platform as a Service (PaaS). No two businesses operate exactly the same way. Salesforce provides custom objects to act as the primary extension mechanism of its relational database, enabling organizations to build highly customized applications (apps) natively on the Einstein 1 Platform without managing underlying database infrastructure.

### Business Problems Solved by Custom Objects
* **Data Silos:** Replacing disconnected legacy systems and spreadsheets by bringing proprietary data into the CRM ecosystem.
* **Process Automation:** Allowing unique business processes (e.g., managing a fleet of vehicles) to trigger automations, approvals, and notifications natively.
* **Contextual 360-Degree View:** Linking custom transactional or operational data directly to standard customer records (Accounts and Contacts).

### Benefits of Custom Objects
* **Auto-generated UI:** Creating a custom object automatically generates standard page layouts, related lists, and mobile interfaces.
* **Native Security Integration:** Custom objects instantly inherit Salesforce’s robust, granular security model (OWD, FLS, Profiles/Permission Sets).
* **API Readiness:** Salesforce automatically generates REST, SOAP, and Bulk API endpoints for every custom object (`__c`), requiring zero backend API development.

---

## 2. Standard Objects vs Custom Objects

### Comparison: Standard vs. Custom

| Feature | Standard Objects | Custom Objects |
| :--- | :--- | :--- |
| **Purpose** | Out-of-the-box CRM features (Sales, Service, Marketing). | Unique, company-specific or industry-specific data. |
| **Ownership** | Provided and maintained by Salesforce. | Created and maintained by the Org Admins/Developers. |
| **Flexibility** | Highly structured; some fields/behaviors cannot be altered. | Completely flexible within platform limits. |
| **Customization** | Can add custom fields, but cannot delete standard fields. | Can add, modify, and delete any custom field. |
| **Reporting** | Pre-built standard report types included. | "Allow Reports" must be enabled to auto-generate report types. |
| **Security** | OWD, Profiles, and Sharing Rules apply. | OWD, Profiles, and Sharing Rules apply identically. |
| **API Name** | Standard name (e.g., `Account`, `Contact`). | Appended with `__c` suffix (e.g., `Vehicle__c`). |

### When to Use Standard vs. Creating Custom
**Best Practice:** *Always use Standard Objects first if they loosely fit the business requirement.*
* **Account:** Use for B2B companies, partners, competitors, or B2C households (Person Accounts). *Do not create a "Company__c" object.*
* **Contact:** Use for individual human beings you communicate with. *Do not create an "Employee__c" object unless dealing with deep internal HR data that shouldn't mix with external contacts.*
* **Lead:** Use for unqualified prospects.
* **Opportunity:** Use for revenue-generating deals or pipeline tracking.

**Create a Custom Object when:** The data represents a completely distinct physical or conceptual entity that does not map to standard CRM definitions (e.g., `Property__c` for a Real Estate firm, `Bug__c` for software development).

---

## 3. Why Organizations Create Custom Objects

Organizations create custom objects to model real-world entities that drive their daily operations.

### Practical Scenarios & Real-World Examples
* **Vehicle Management:** A logistics company needs to track `Vehicle__c`, `Maintenance_Log__c`, and `Route__c`.
* **Warranty Claims:** A manufacturer uses Standard Cases for support, but uses `Warranty_Claim__c` and `Claim_Line_Item__c` to manage financial reimbursement and parts.
* **Dealer Networks:** A motor company tracks standard Accounts as Dealerships, but uses `Dealer_Inventory__c` to see exactly which cars are on which lot.
* **Student Management:** Universities extend Standard Contacts to represent Students, creating `Course__c`, `Semester__c`, and `Enrollment__c` objects.
* **Hospital Systems:** `Patient_Record__c`, `Prescription__c`, and `Lab_Result__c`.
* **Inventory Tracking:** `Warehouse__c`, `Inventory_Item__c`, and `Stock_Transfer__c`.
* **Loan Processing:** Banks use `Loan_Application__c`, `Credit_Check__c`, and `Disbursement__c`.
* **Insurance Claims:** `Policy__c`, `Claim__c`, and `Adjuster_Assessment__c`.

---

## 4. Custom Object Architecture

At the database level, custom objects are represented as tables, but they are wrapped in Salesforce's metadata architecture.

### Architecture Components
* **Object Structure:** The table itself (Metadata XML definition).
* **Records:** The rows in the table (Data).
* **Fields:** The columns in the table (Data Types).
* **Relationships:** Foreign keys linking tables together (Lookups, Master-Detail).
* **Metadata:** The configuration data describing the object (Page Layouts, Validation Rules, UI settings).

### Architecture Diagram

    +-------------------------------------------------------------+
    |                      SALESFORCE PLATFORM                    |
    |                                                             |
    |  +-------------------+          +------------------------+  |
    |  | STANDARD OBJECT   |          | CUSTOM OBJECT          |  |
    |  | (e.g., Account)   |<---------+ (e.g., Invoice__c)     |  |
    |  +-------------------+  Lookup  +------------------------+  |
    |  | Id                |          | Id                     |  |
    |  | Name              |          | Name                   |  |
    |  | Industry          |          | Account__c (FK)        |  |
    |  | Type              |          | Total_Amount__c        |  |
    |  +-------------------+          | Status__c              |  |
    |                                 +------------------------+  |
    |                                             ^               |
    |                                             | Master-Detail |
    |                                 +------------------------+  |
    |                                 | CUSTOM CHILD OBJECT    |  |
    |                                 | (e.g., Line_Item__c)   |  |
    |                                 +------------------------+  |
    |                                 | Id                     |  |
    |                                 | Invoice__c (FK)        |  |
    |                                 | Product_Name__c        |  |
    |                                 | Price__c               |  |
    |                                 +------------------------+  |
    +-------------------------------------------------------------+

---

## 5. Creating a Custom Object

When creating a custom object via Setup > Object Manager, architects must configure foundational settings.

### Step-by-Step Breakdown
1. **Object Label:** The human-readable name in the UI (e.g., "Property").
2. **Plural Label:** Used in tabs and lists (e.g., "Properties").
3. **API Name:** Auto-generated but customizable. Will automatically append `__c` upon save (e.g., `Property__c`). Used in Apex, SOQL, and integrations.
4. **Record Name:** The identifier field. Can be **Text** (user-entered, e.g., "123 Main St") or **Auto-Number** (system-generated, e.g., `PROP-{0000}`).
5. **Data Type Selection:** Text or Auto-Number for the Record Name.
6. **Allow Reports:** Generates standard report types (e.g., "Properties", "Properties with Accounts") and makes the object available in the Report Builder.
7. **Allow Activities:** Allows Tasks and Events to be related to the object (adds the Open Activities and Activity History related lists).
8. **Track Field History:** Enables Field History Tracking (up to 20 fields can be tracked for changes).
9. **Launch New Tab Wizard:** Prompts the admin to create a Custom Tab immediately after saving, choosing an icon to make the object visible in apps.

---

## 6. Important Custom Object Settings

### Deployment Status
* **In Development:** The object is completely invisible to all users except those with the "Customize Application" permission (System Admins). Used while building fields and logic.
* **Deployed:** The object is visible to end-users (dependent on Profile permissions and Tab visibility).

### Optional Features & Business Impact
* **Track Activities:** Business Impact: Critical if users need to log calls, emails, or schedule meetings related to this specific record.
* **Track History:** Business Impact: Essential for compliance and auditing. Creates a `[ObjectName]__History` table.
* **Allow Reports:** Business Impact: Required for BI. If disabled, users cannot build native Salesforce reports on this data.
* **Allow Search:** Business Impact: If unchecked, users cannot find records of this object using Global Search, severely impacting user adoption and data accessibility.
* **Allow Sharing:** Allows you to use sharing rules to share records with specific users/roles.

---

## 7. Custom Object Relationships

Proper relationship design is the core of Salesforce Data Architecture.

### Lookup Relationship
* **Purpose:** A loose, optional coupling between two objects.
* **Use Cases:** Linking a standard object to a custom object where the custom object has its own independent lifecycle.
* **Examples:** Linking a `Vehicle__c` to a `Contact` (Driver). If the Contact leaves the company and is deleted, the Vehicle remains in the system.

### Master-Detail Relationship
* **Purpose:** A tightly coupled, parent-child relationship.
* **Roll-Up Summary Support:** Parent records can aggregate data from child records (COUNT, SUM, MIN, MAX).
* **Ownership Behavior:** The child record does not have an Owner field. It inherits the owner of the Master record.
* **Security Behavior:** Security is cascaded. If a user can read/edit the Parent, they can read/edit the Child. (Controlled by Parent OWD). Deleting the parent permanently deletes the child (Cascade Delete).

### Self Relationship
* **Purpose:** A Lookup relationship pointing back to the same object.
* **Example:** `Employee__c` has a lookup to `Employee__c` called "Manager".

### Hierarchical Relationship
* **Purpose:** A specialized Self Relationship.
* **Note:** *Strictly available ONLY on the standard User object.* Used to define management hierarchies for approval processes and sharing.

### Many-to-Many Relationship (Junction Object)
* **Purpose:** Connects two objects where multiple records on one side relate to multiple on the other.
* **Implementation:** Created using a custom "Junction" object with *two* Master-Detail relationships.
* **Example:** `Student__c` and `Course__c`. The Junction is `Enrollment__c`.
  * `Enrollment__c` has a Master-Detail to `Student__c`.
  * `Enrollment__c` has a Master-Detail to `Course__c`.

---

## 8. Custom Fields in Custom Objects

Data types dictate how data is stored, validated, and interacted with in the UI.

| Field Type | Purpose | Use Cases | Best Practices / Limitations |
| :--- | :--- | :--- | :--- |
| **Text** | Short string (up to 255 chars). | Names, short identifiers. | Can be marked as External ID / Unique. |
| **Number** | Numeric values (no currency symbol). | Quantities, scores, ages. | Define decimal places carefully; affects math. |
| **Currency** | Financial data. | Amounts, budgets. | Respects multi-currency org settings. |
| **Date / DateTime** | Temporal data. | Birthdates, System timestamps. | DateTime respects the viewing user's timezone. |
| **Picklist** | Dropdown list. | Status, Type, Category. | Use Global Value Sets for shared lists across objects. |
| **Multi-Select Picklist** | Select multiple values. | Certifications, Interests. | **Avoid if possible.** Difficult to report on, filter, or use in formulas. |
| **Formula** | Read-only calculated values. | Cross-object references, math. | Evaluated at runtime; does not consume physical storage. |
| **Checkbox** | Boolean (True/False). | Is Active, Needs Review. | Defaults to unchecked; cannot be null. |
| **Email / Phone / URL** | Specific string formats. | Contact info, web links. | Auto-formatted as clickable links in UI. |
| **Geolocation** | Latitude / Longitude. | Mapping, distance calculations. | Counts as 3 custom fields against limits. |
| **Long Text Area** | Up to 131,072 chars. | Descriptions, notes. | Cannot be used in formulas or filtering. |
| **Rich Text Area** | Formatted text & images. | Articles, formatted descriptions. | Similar limits to Long Text; renders HTML. |

---

## 9. Record Types in Custom Objects

### Why Record Types are Used
Record Types allow you to offer different business processes, picklist values, and page layouts to different users based on their profile.

### Core Capabilities
* **Business Process Separation:** A `Support_Ticket__c` object might have "IT Request" and "HR Request" record types.
* **Page Layout Assignment:** Assign different UI layouts per record type per profile.
* **Profile Assignment:** Control which profiles can create which types of records.

### Examples
On a `Property__c` object, Record Types: "Commercial" and "Residential".
* "Commercial" shows fields like "Zoning Type" and "Loading Docks".
* "Residential" shows fields like "Bedrooms" and "HOA Fees".

---

## 10. Page Layouts & User Interface

### Layout Design
Page layouts control the positioning of fields, sections, related lists, and buttons on the classic record detail page.

### Dynamic Forms (Modern Best Practice)
Salesforce Architects must prioritize **Dynamic Forms** over traditional page layouts. 
* **Concept:** Instead of monolithic page layouts, fields and sections are placed directly onto Lightning Record Pages as individual components.
* **Benefits:** Conditional visibility (e.g., Show "Spouse Name" only if "Marital Status" = Married) without needing multiple page layouts or record types.

### Mobile and UX Considerations
* Put the most critical fields at the top (Highlights Panel).
* Keep the number of fields per section under 10-15 to avoid scrolling fatigue.
* Optimize Action menus for the Salesforce Mobile App.

---

## 11. Security Considerations

Security in Salesforce is layered. A custom object must navigate all layers.

* **Object Level Security (OLS):** Controlled by **Profiles** and **Permission Sets**. Determines if a user can Create, Read, Edit, or Delete (CRED) the object *as a whole*.
* **Field Level Security (FLS):** Controlled by Profiles and Permission Sets. Determines if a user can see or edit specific fields *within* the object.
* **Record Level Security (RLS) / Sharing:**
    * **OWD (Organization-Wide Defaults):** The baseline (e.g., Private, Public Read Only, Public Read/Write).
    * **Role Hierarchy:** Grants access vertically to managers above the record owner.
    * **Sharing Rules:** Grants access laterally based on criteria or ownership.
    * **Manual Sharing:** One-off sharing by the record owner.

*Architect Note: Security configuration dictates system performance. Heavy use of Sharing Rules on objects with Millions of records will cause lock contentions and slow down bulk data loads.*

---

## 12. Reporting Considerations

### Report Types
When "Allow Reports" is checked, standard report types are generated. 
* E.g., "Properties", "Properties and Maintenance Logs" (if Master-Detail).

### Custom Report Types (CRTs)
If an organization needs to report across Lookups, or needs "WITH or WITHOUT" relationships (e.g., "Properties with or without Maintenance Logs"), a CRT is required.
* **Best Practice:** Use CRTs to rename fields for specific business units or to hide unnecessary system fields from report builders.

### Dashboards and Analytics
Custom objects can feed directly into Salesforce Dashboards. For Large Data Volumes (LDV), architects utilize CRM Analytics (Tableau CRM) to digest millions of custom object records without impacting transactional performance.

---

## 13. Automation on Custom Objects

### Validation Rules
* **Purpose:** Ensure data integrity before a record is saved.
* **Example:** `Discount__c` cannot exceed 20% on `Invoice__c` unless `Manager_Approval__c` is true.

### Flows (Record-Triggered)
* **Purpose:** The modern declarative automation standard. Can create/update records, send emails, or call external APIs when a custom object record is created/updated/deleted.
* **Example:** When `Inventory__c` stock falls below 10, automatically create a `Purchase_Order__c` record.

### Apex Triggers
* **Purpose:** Code-based automation for complex logic that Flows cannot handle (e.g., complex collections manipulation, high-volume bulkifications, dynamic DML).
* **Example:** Complex commission calculations crossing 5 different custom objects upon `Deal__c` closure.

### Approval Processes
* **Purpose:** Route records to users for approval/rejection.
* **Example:** `Expense_Report__c` routing to the Submitter's Manager, then to Finance.

---

## 14. Custom Object Limits

Salesforce limits enforce multi-tenant stability. Limits vary by edition.

### Core Limits (Enterprise vs. Unlimited)

| Feature | Enterprise Edition (EE) | Unlimited Edition (UE) |
| :--- | :--- | :--- |
| **Max Custom Objects per Org** | 200 | 2,000 |
| **Max Custom Fields per Object**| 500 | 800 |
| **Max Lookup Relationships** | 40 per object | 40 per object |
| **Max Master-Detail Relationships**| 2 per object | 2 per object |
| **Max Roll-Up Summaries** | 40 per parent object | 40 per parent object |
| **Max Active Validation Rules** | 100 per object | 500 per object |
| **Max Record Types** | No limit | No limit |

*Note: Limits can occasionally be increased by purchasing add-ons or contacting Salesforce Support.*

---

## 15. Custom Metadata vs Custom Objects

Architects must know when to use configurations versus transactional data.

### Comparison
| Feature | Custom Object | Custom Metadata Type (CMDT) |
| :--- | :--- | :--- |
| **Purpose** | Transactional business data (e.g., Invoices). | Application configuration data (e.g., API Keys, Routing rules). |
| **Deployment** | Records are Data. Must use Data Loader. | Records are Metadata. Deployed via Changesets/CI-CD. |
| **SOQL Limits** | Counts against SOQL queries/governor limits. | **Does not** count against SOQL queries (cached). |

* **Use Case:** Creating a mapping matrix that determines lead routing based on Zip Code. Store this in CMDT, not a Custom Object, to save SOQL queries during high-volume data operations.

---

## 16. Custom Settings vs Custom Objects

*(Note: Custom Settings are largely being superseded by Custom Metadata, but Hierarchy settings remain crucial).*

| Type | Purpose & Use Cases | Versus Custom Object |
| :--- | :--- | :--- |
| **Hierarchy Settings** | Allows configuration to vary by Profile or User. (e.g., "Bypass Validation Rules" checkbox for the System Admin profile). | Custom Objects cannot easily vary data contextually by the running user without complex SOQL. |
| **List Settings** | Storing static datasets (e.g., Country Codes). | Mostly deprecated. Use Custom Metadata instead. |

---

## 17. Real Project Scenarios

### 1. Warranty Management System
* **Account (Standard):** The Customer.
* **Dealer__c (Custom):** Lookup to Account. Represents the dealership doing the repair.
* **Warranty_Registration__c (Custom):** Lookup to Account. Tracks the active warranty period.
* **Warranty_Claim__c (Custom):** Master-Detail to Warranty Registration. The actual claim filed.

### 2. University Management System
* **Contact (Standard):** The Student.
* **Course__c (Custom):** The catalog of classes (e.g., BIO-101).
* **Enrollment__c (Custom Junction):** Master-Detail to Contact, Master-Detail to Course. Tracks grade, semester, and attendance.

### 3. Hospital Management System
* **Account/Contact (Standard):** Represents the Patient (Person Accounts highly recommended here).
* **Doctor__c (Custom):** Lookup to User.
* **Appointment__c (Custom Junction):** Master-Detail to Patient, Master-Detail to Doctor.

---

## 18. Common Mistakes

1. **Creating Unnecessary Custom Objects:** Creating a "Company" object instead of using Standard Accounts.
2. **Overusing Master-Detail:** Choosing Master-Detail simply to get a Roll-Up Summary, without realizing it forces restrictive sharing rules and cascade deletions.
3. **Poor Naming Conventions:** Using generic names like `Data__c` or `Table__c`.
4. **Security Misconfiguration:** Leaving OWD as Public Read/Write on sensitive custom objects (like HR or Financial records).
5. **Data Duplication / Lack of External IDs:** Failing to index external ID fields during integration, causing massive duplicate record creation on upserts.

---

## 19. Best Practices

* **Naming Standards:** Use clear, singular nouns (e.g., `Invoice__c`, not `Invoices__c`). Use Help Text on every custom field.
* **API Naming Conventions:** For enterprise orgs, prefix custom objects with a department or project acronym (e.g., `FIN_Invoice__c` or `HR_Review__c`) to prevent namespace collisions.
* **Relationship Design:** Default to Lookup relationships unless the strict parent-child ownership, security, and deletion rules of Master-Detail are explicitly required by the business.
* **Scalability:** Keep the number of fields per object under control. Just because you have 500 fields available doesn't mean you should use them. Overly wide tables degrade database performance.
* **Reporting Optimization:** Use picklists instead of text fields wherever possible to ensure clean grouping in reports.

---

## 20. Enterprise Architecture Considerations

### Large Data Volume (LDV)
* When custom objects exceed 5-10 million records, standard queries and list views will slow down.
* **Data Skew:** Avoid having more than 10,000 child records linked to a single parent record. This causes severe lock contention during record updates.
* **Skinny Tables:** For extreme performance needs, Architects can request Salesforce to create Skinny Tables (which combine custom and standard fields into a single, un-indexed database view).

### Archiving & Governance
* Set up a data archiving strategy early. Use Heroku, Snowflake, or AWS to offload older `Invoice__c` or `Log__c` records. Use Salesforce Connect (External Objects, `__x`) to view archived data in Salesforce without consuming native storage.

### Global Implementations (Multi-Org vs Single-Org)
* In single-org architectures spanning multiple global business units, use Record Types, Profiles, and strict OWD (Private) to ensure the EMEA team does not see the AMER team's custom object data.

---

## 21. Interview Questions & Answers

### Beginner
**Q: What is the difference between an Object and a Record?**
**A:** An object is the database table definition (e.g., `Invoice__c`). A record is a single row of data within that table (e.g., Invoice #1005 for $500).

**Q: What suffix is added to a custom object's API name?**
**A:** `__c` (underscore underscore c).

### Intermediate
**Q: When would you choose a Lookup relationship over a Master-Detail relationship?**
**A:** You choose a Lookup when the child record needs to exist independently of the parent, when the child needs to have its own owner, or when the child needs its own independent security/sharing settings.

**Q: What happens to a child record in a Master-Detail relationship if the parent is deleted?**
**A:** The child record is permanently deleted (Cascade Delete) and sent to the Recycle Bin along with the parent.

### Advanced
**Q: Can you convert a Lookup relationship to a Master-Detail relationship?**
**A:** Yes, but ONLY if every single child record in the system already contains a value in the lookup field. If even one record has a blank value, the platform will block the conversion.

**Q: How do you bypass the 2 Master-Detail relationship limit on a custom object if you need a 3rd?**
**A:** You cannot technically bypass it. However, you can use a Lookup relationship and simulate Master-Detail behavior using Flow or Apex (to enforce deletion/sharing) and Declarative Lookup Rollup Summaries (DLRS) or Flow for roll-up calculations.

### Architect-Level
**Q: You are designing an integration where millions of external `Transaction__c` records will be loaded nightly. What are the architectural concerns regarding relationships and automation?**
**A:** 1. **Data Skew:** Ensure no single Account receives more than 10k transactions to avoid record locking.
2. **Automation:** Disable or highly optimize record-triggered Flows and Apex Triggers to prevent governor limit exceptions during the bulk load.
3. **Indexing:** Ensure the external ID field used for matching is indexed.
4. **LDV (Large Data Volume):** If records are purely for historical viewing and not processed natively, consider an External Object (`__x`) instead of a Custom Object to save storage costs.

---

## 22. Revision Summary

* **Creation:** Managed in Object Manager. Defines data structure for unique business entities. Always appends `__c`.
* **Relationships:** Lookup (Loose, independent), Master-Detail (Tight, cascade delete, inherited security), Junction (Many-to-Many via two M-Ds).
* **Security:** OLS (Object level), FLS (Field level), OWD/Sharing (Record level). Master-Detail ignores OWD/Sharing and inherits from Parent.
* **Reporting:** Must check "Allow Reports". Standard vs Custom Report Types depending on lookup traversals needed.
* **Limits:** 200/2000 objects (EE/UE), 2 Master-Details per object, 40 Lookups per object, 40 Roll-ups per parent.
* **Best Practices:** Prefer Standard objects first. Design for LDV. Use Dynamic Forms instead of bloated page layouts. Use Custom Metadata for configuration data, not Custom Objects.