# Salesforce Editions: The Architect's Comprehensive Guide

## Introduction

### What are Salesforce Editions?
Salesforce Editions are distinct bundles of features, capabilities, and capacity limits offered by Salesforce to cater to different business sizes, complexities, and use cases. Think of an Edition as the "tier" or "chassis" of your Salesforce instance. It defines the absolute boundaries of what is technically possible within that specific organization (Org).

### Why Salesforce Provides Multiple Editions
Salesforce serves everyone from single-person startups to Fortune 500 global enterprises. A one-size-fits-all approach is impossible due to varying requirements in governance, data volume, automation complexity, and budget. Multiple editions allow Salesforce to scale with a customer's business journey, offering a cost-effective entry point and a seamless upgrade path.

### How Editions Affect Features and Pricing
Editions dictate the baseline capabilities of your Org. Higher-tier editions include advanced features (e.g., full API access, programmatic development, advanced sandbox environments) natively, while lower-tier editions restrict or completely remove these features. Pricing is structured per user, per month, but the base cost of that user license scales dramatically depending on the Org's Edition.

### Difference Between Editions and Licenses
* **Edition:** Determines the overall capacity and capabilities of the *entire environment* (e.g., Enterprise Edition allows Apex code to run). Think of this as the building.
* **License:** Determines what a *specific user* can access within that environment (e.g., a Sales Cloud User License vs. a Platform User License). Think of this as the keycard to specific rooms within the building.

---

## Salesforce Edition Overview

| Edition | Target Audience | Key Differentiator |
| :--- | :--- | :--- |
| **Starter Suite** | Small businesses & startups | Out-of-the-box CRM, simple setup, heavily restricted customization. |
| **Professional** | Mid-sized businesses | Standard CRM functionality, declarative tools, restricted API/Apex. |
| **Enterprise** | Large businesses/Enterprises | Deep customization, full programmatic access (Apex/LWC), standard API access. |
| **Unlimited** | Global Enterprises | Highest limits, Premier Support, advanced sandboxes, massive scalability. |
| **Performance** | High-growth/Specialized | Combines Sales/Service Cloud with advanced analytics and third-party data. |
| **Developer** | Developers & Architects | Free, perpetual org with full Enterprise/Unlimited features but microscopic storage limits. |
| **Platform** | Internal App Users | (License type) Grants access to custom apps without standard CRM objects. |
| **Trial Orgs** | Prospects & Implementers | Time-bound (14-30 days) orgs for Proof of Concepts (POCs). |

---

## Developer Edition

### Purpose
The Developer Edition (DE) is a free, fully featured Salesforce environment designed strictly for building, testing, and learning. It is the sandbox of the individual developer and the foundation of ISV AppExchange package creation.

### Features
* Full programmatic access (Apex, LWC, Aura, Visualforce).
* Full REST/SOAP API access.
* Access to advanced features (e.g., Territory Management, Multiple Currencies) usually reserved for Enterprise+.

### Limitations
* Microscopic data limits (5MB Data Storage, 20MB File Storage).
* Limited user licenses (usually 2 full Salesforce licenses).
* Cannot be used for commercial production purposes.

### Common Use Cases
* Trailhead challenges and personal upskilling.
* Developing and uploading First-Generation (1GP) or Second-Generation (2GP) AppExchange packages.
* Conducting Proof of Concepts (POCs) without impacting a client's production org.

### Why Every Developer Needs One
A DE org is your permanent laboratory. Unlike company sandboxes that get refreshed (wiping your experiments) or Trial orgs that expire, a DE org persists as long as you log in periodically, making it perfect for long-term architectural testing.

---

## Starter Suite

### Intended Customers
Small businesses migrating from spreadsheets or basic email marketing tools.

### Key Capabilities
* Unified Sales, Service, and Marketing (basic).
* Out-of-the-box dashboards and simplified setup wizards.
* Basic email outreach.

### Major Limitations
* No API access (cannot integrate with external systems via standard APIs).
* No programmatic development (No Apex, No LWC).
* Strict limits on custom objects and fields.

### When to Use It
Use Starter when a business needs a CRM *today*, has zero internal IT staff, and operates on highly standardized, simple sales processes.

---

## Professional Edition (PE)

### Features
* Comprehensive standard CRM features (Lead to Cash).
* Customizable dashboards and reports.
* Declarative automation (Flows).

### Limitations
* **Apex Restriction:** You cannot write or execute custom Apex code (unless it is part of a certified AppExchange package).
* **API Restriction:** Standard API access is disabled by default (can be purchased as an add-on).
* **Sandbox Limits:** Maximum of 10 Developer sandboxes; no Partial or Full sandboxes.

### Security Capabilities
Basic profiles and permission sets. Limited ability to utilize advanced granular sharing architectures compared to Enterprise.

### Automation Capabilities
Supports declarative automation (Flow Builder), but with lower execution limits per transaction.

### Typical Customer Profile
Companies with established sales teams that need robust tracking but rely on manual processes rather than heavy integrations or custom-coded automation.

---

## Enterprise Edition (EE)

### Features
Enterprise Edition is the industry standard for serious Salesforce implementations. It unlocks the true power of the Salesforce platform.

### Development & Customization Capabilities
* **Full Code Access:** Apex, Lightning Web Components (LWC), Triggers.
* **High Limits:** 2,000 custom objects, 500 custom fields per object.
* **API Access:** Full REST/SOAP/Bulk API access natively included.

### Security Features
* Granular security model: Advanced Role Hierarchies, Sharing Rules, Enterprise Territory Management.
* Supports Salesforce Shield (Platform Encryption, Event Monitoring, Field Audit Trail) as an add-on.

### Integration Support
Fully supports complex middleware integrations (MuleSoft, Boomi) via robust API availability and Platform Events.

### Why Most Enterprise Implementations Use EE
Enterprise Edition is the baseline for "Architectural Freedom." If an organization requires custom integrations with an ERP, custom UI components for a unique business process, or complex, multi-layered data security, Enterprise Edition is the minimum requirement.

---

## Unlimited Edition (UE)

### Additional Features & Storage
Unlimited Edition takes Enterprise and removes the ceiling.
* **Higher Limits:** 2,000 custom objects, 800 custom fields per object.
* **Storage:** Significantly higher baseline data storage allocations (120MB per user vs 20MB in EE).

### Premier Support
Includes 24/7 Premier Success (dedicated support, faster routing, configuration assistance) out of the box.

### Sandbox & Governance Benefits
* Includes 1 Full Sandbox and 1 Partial Copy Sandbox natively.
* Provides 100 Developer Sandboxes and 5 Developer Pro Sandboxes.
* Crucial for complex Application Lifecycle Management (ALM) and DevOps pipelines.

---

## Performance Edition (PE)

*Note: Salesforce heavily shifted focus to Unlimited Edition + Add-ons, but Performance Edition still exists in legacy/specific contracts.*

### Advanced Capabilities
Combines Unlimited Edition with Sales Cloud Einstein, Service Cloud, and industry-specific data integrations. Designed for organizations that need absolute maximum transactional volume and advanced AI analytics natively bundled.

---

## Salesforce Platform License (Core Concept)

### What it is
Not an Edition, but a crucial licensing tier applied *within* Enterprise/Unlimited orgs. It provides access to the Lightning Platform (custom objects, custom apps) without the cost of standard CRM functionality.

### How it Differs from CRM Licenses
A Salesforce Platform user **cannot** access standard sales or service objects like Leads, Opportunities, Campaigns, or Cases. They **can** access Accounts, Contacts, Reports, Dashboards, and Custom Objects.

### Common Use Cases
* HR Onboarding applications built entirely on custom objects.
* IT Helpdesk ticketing (if built custom, not using Service Cloud).
* Supply chain tracking applications.

### Licensing Restrictions
Strict contractual limits apply to what a Platform user can touch; architects must carefully design data models to ensure Platform users do not inadvertently require access to restricted standard objects.

---

## Trial Orgs

### Purpose
Used by Account Executives and Consulting Partners to demonstrate Salesforce capabilities to prospects.

### Limitations
* Strictly time-bound (usually 14 or 30 days).
* Cannot be converted to a Developer Org (though they can sometimes be converted to active Production orgs upon purchase).

### Usage Scenarios
* Client Proof of Concept (POC).
* Evaluating specific add-ons (e.g., CPQ Trial, FSC Trial).

---

## Detailed Comparison Table

| Feature | Starter | Professional | Enterprise | Unlimited | Developer |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Target Audience** | Micro-Biz | Mid-Market | Large / Enterprise | Global Enterprise | Devs / ISVs |
| **Custom Objects** | Limited | 50 | 2,000 | 2,000 | 2,000 |
| **API Access** | No | Add-on only | Yes | Yes | Yes |
| **Apex/LWC Support** | No | No (Executes only) | Yes (Full) | Yes (Full) | Yes (Full) |
| **Flows** | Yes (Basic) | Yes | Yes (Advanced) | Yes (Advanced) | Yes |
| **Record Types** | No | Yes (Limited) | Yes | Yes | Yes |
| **Full Sandbox** | 0 | 0 | Add-on | 1 Included | 0 |
| **Partial Sandbox**| 0 | 0 | 1 Included | 1 Included | 0 |
| **Data Storage (Min)**| 1 GB | 1 GB | 10 GB | 10 GB | 5 MB |
| **Support Level** | Standard | Standard | Standard | Premier Included | None |

---

## Feature Comparison

### Automation Capabilities
* **Professional:** Basic Flows allowed, but no asynchronous Apex or complex system context triggers.
* **Enterprise/Unlimited:** Full suite of Flow Builder, Apex Triggers, Batchable/Schedulable/Queueable Apex.

### Security Capabilities
* **Professional:** Profile-level security. Limited Sharing Rules.
* **Enterprise/Unlimited:** Extensive Sharing Rules, Manual Sharing, Apex Managed Sharing, Field-Level Security (FLS), Shield compatibility.

### Development Capabilities
* **Professional:** UI declarative development only (Lightning App Builder).
* **Enterprise/Unlimited:** Full IDE integration (VS Code), Salesforce CLI, source-driven development (Scratch Orgs).

### Integration & Platform Events
* **Enterprise:** 100,000 daily API calls + 1,000/user. Standard Platform Event allocations.
* **Unlimited:** 100,000 daily API calls + 5,000/user. High-volume Platform Event allocations.

---

## Storage Limits by Edition

| Edition | Minimum Org Data Storage | Data Storage Per User | Minimum Org File Storage | File Storage Per User |
| :--- | :--- | :--- | :--- | :--- |
| **Professional** | 1 GB | 20 MB | 10 GB | 2 GB |
| **Enterprise** | 10 GB | 20 MB | 10 GB | 2 GB |
| **Unlimited** | 10 GB | 120 MB | 10 GB | 2 GB |

*Note: Storage can always be purchased as an a la carte add-on if orgs exceed these limits.*

---

## API Capabilities by Edition

| Edition | Base API Requests / Day | Additional Requests / User | Max Limits / Org |
| :--- | :--- | :--- | :--- |
| **Professional** | N/A (Needs Add-on) | N/A | 100,000 (with add-on) |
| **Enterprise** | 100,000 | 1,000 | Custom (Requires Salesforce negotiation) |
| **Unlimited** | 100,000 | 5,000 | Custom |

*Architect Insight:* Always calculate API limits during integration design. Bulk API should be used for massive data loads to conserve daily REST/SOAP API limits.

---

## Sandbox Availability

| Sandbox Type | Storage Limit | Refresh Interval | Included in EE | Included in UE |
| :--- | :--- | :--- | :--- | :--- |
| **Developer** | 200 MB | 1 Day | 25 | 100 |
| **Developer Pro**| 1 GB | 1 Day | 0 | 5 |
| **Partial Copy** | 5 GB | 5 Days | 1 | 1 |
| **Full Copy** | Prod Size | 29 Days | 0 (Add-on) | 1 |

---

## Choosing the Right Edition

### Example Scenarios

1.  **Startup (Pre-revenue, 5 employees):** *Starter Suite.* Lowest cost, immediate CRM capabilities without overhead.
2.  **Small Business (Local B2B, 30 employees):** *Professional Edition.* Good reporting and process tracking, no budget for custom code or external integrations.
3.  **Mid-Sized Company (Fast growth, utilizing an ERP):** *Enterprise Edition.* Required to build standard integrations with the ERP via REST API and write custom Apex triggers for complex routing.
4.  **Large Enterprise / Global Org:** *Unlimited Edition.* Requires Premier support, global data distribution, high data storage allocations, and Full Sandboxes for rigorous DevOps testing.
5.  **ISV Development:** *Developer Edition* (for packaging) and *Enterprise Edition Partner Orgs* for testing.

---

## Real Project Scenarios

* **Manufacturing Company:** Needs to integrate Salesforce with SAP for inventory data. **Decision:** Enterprise Edition. Professional edition cannot support the standard API integrations required for SAP middleware.
* **Banking Organization:** Highly regulated, requires encryption at rest, tracking of all user clicks, and custom Aura/LWC components for teller UIs. **Decision:** Unlimited Edition + Salesforce Shield. UE provides the Full Sandbox needed for strict UAT compliance testing.
* **Startup App Developer:** Wants to build a new AppExchange product. **Decision:** Developer Edition. It is free and allows the creation of a managed package namespace.

---

## Common Misconceptions

* **"Unlimited Edition has no limits."** FALSE. Unlimited Edition simply has the *highest* governor limits, storage, and API allocations. You can still hit CPU time limits, SOQL limits, and data storage caps in UE.
* **"Developer Edition is the same as a Developer Sandbox."** FALSE. A Developer Sandbox is tied to a paid Production org and is used for ALM. A Developer Edition is a standalone, free org not tied to any company.
* **"We can just use Platform Licenses for our Sales Team to save money."** FALSE. This is a contractual violation and technically impossible, as Platform licenses cannot view the standard Opportunity object.

---

## Interview Questions & Answers

### Beginner
**Q: What is the primary difference between Professional and Enterprise Edition?**
A: Enterprise Edition includes full programmatic development (Apex, LWC) and API access natively, whereas Professional relies on declarative tools and lacks API access out-of-the-box.

### Intermediate
**Q: A client has Enterprise Edition and wants to test a massive data migration of 50GB. What Sandbox should they use, and do they have it natively?**
A: They need a Full Copy Sandbox because it matches Production data sizing. However, Enterprise Edition does *not* include a Full Sandbox by default; it must be purchased as an add-on (or they must upgrade to Unlimited).

### Advanced (Architect Level)
**Q: Explain how Edition impacts your Application Lifecycle Management (ALM) strategy.**
A: Edition directly dictates sandbox availability. If a client is on Professional Edition, ALM is severely restricted (change sets between Developer sandboxes and Prod only). If on Enterprise, we have a Partial Copy sandbox for integration testing. For Unlimited, we can design a mature CI/CD pipeline using Developer Pro sandboxes for integration, Partial for UAT, and Full for Staging/Performance testing.

---

## Revision Summary
* **Current State:** Document established with core edition definitions, storage, API, Sandbox limits, and architectural scenarios.
* **Next Review:** Check for Salesforce Spring/Summer/Winter release updates regarding Storage minimums or Starter Suite rebranding.
