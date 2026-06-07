# Salesforce Flow

A comprehensive, long-term reference for Salesforce Developers, Administrators, Consultants, and Architects.

## 1. Flow Architecture
Salesforce Flow is the declarative automation engine built on the Lightning Platform. At its core, Flow metadata is compiled into Java classes executed by the Salesforce underlying rule engine. It provides a visual interface (Flow Builder) to encapsulate complex business logic, perform CRUD operations, make callouts, and interact with the database without writing Apex code.

## 2. Flow Builder
The Flow Builder is the UI used to assemble Flows. It consists of the Canvas, the Toolbox (Manager and Elements), and the Button Bar. Flow Builder utilizes a drag-and-drop interface mapping directly to metadata components (Flow elements, variables, resources, and connections).

## 3. Flow Elements
Elements are the building blocks of a Flow. They fall into three main categories:
- **Interaction:** Screens (collecting user input), Actions (email alerts, core actions, Apex invocations), and Subflows.
- **Logic:** Assignments (setting variable values), Decisions (conditional branching), Loops (iterating over collections), Collection Sort, and Collection Filter.
- **Data:** Get Records (SOQL), Create Records (Insert), Update Records (Update), Delete Records (Delete), and Rollback Records.

## 4. Flow Types

### Screen Flows
Interactive flows requiring user input. Built with Screen components, they can be embedded in Lightning Pages, Utility Bars, Experience Cloud sites, Quick Actions, or launched via URL.

### Record-Triggered Flows
Fires automatically when a record is created, updated, or deleted.
- **Before-Save Flows (Fast Field Updates):** Run before the record is saved to the database. Equivalent to `before insert` or `before update` Apex triggers. They are up to 10x faster because they do not perform separate DML operations.
- **After-Save Flows (Actions and Related Records):** Run after the record is saved. Used to update related records, send emails, or call external systems. Equivalent to `after insert` or `after update` triggers.

### Scheduled Flows
Run asynchronously at a specified time and frequency (Once, Daily, Weekly) for a batch of records. They execute in batches (default 200) similar to Batch Apex.

### Autolaunched Flows
Run in the background without user interaction. These can be invoked by Apex, REST API, Webhooks, Process Builder, or other Flows (as Subflows).

### Platform Event-Triggered Flows
Triggered by the publishing of a Platform Event. Used for asynchronous, decoupled integrations between Salesforce and external systems.

### Flow Orchestration
Builds multi-user, multi-step, complex business processes. It coordinates multiple flows across different users (Stages, Steps, Work Items).

## 5. Flow Order of Execution
Understanding when Record-Triggered flows execute is critical for architects:
1. System Validation
2. **Before-Save Flows**
3. Before Triggers (Apex)
4. Custom Validation
5. Save to database (but not committed)
6. After Triggers (Apex)
7. Assignment Rules, Auto-Response, Workflow Rules
8. Escalation Rules
9. **After-Save Flows**
10. Entitlement Rules
11. Roll-up Summaries
12. DML Commit

## 6. Flow Governor Limits
- **SOQL Queries:** 100 per transaction.
- **DML Statements:** 150 per transaction.
- **Flow Elements executed at runtime:** 2,000 per flow interview.
- **Iteration Limit:** Loops process sequentially. Hitting 2000 elements across a loop will cause an error.

## 7. Flow Performance Optimization
- **Bulkification:** Flows automatically bulkify DML and SOQL inside loops *only* if designed correctly. However, never put DML or Get Records inside a Loop. Assign to a collection variable and perform DML outside the loop.
- **Filter Logic:** Perform filtering using `Collection Filter` rather than looping and deciding.
- **Choose Before-Save:** Prefer before-save over after-save for same-record updates.
- **Asynchronous Paths:** Use async paths for callouts or heavy processing to avoid locking the UI.

## 8. Flow Debugging
Use the **Debug** button in Flow Builder to test with specific records or inputs. It simulates execution and outputs a path trace. Use **Fault Paths** connected to Data elements to gracefully handle DML/SOQL exceptions, logging errors to a custom Error Log object.

## 9. Flow Design Patterns
- **One Record-Triggered Flow per Object per Context:** (Legacy pattern, evolving). Now with Flow Trigger Explorer, multiple modular flows ordered via entry conditions and order numbers are preferred.
- **Subflow Pattern:** Extract repeated logic into Autolaunched Flows to be called as Subflows.
- **Fault Handling Pattern:** Always attach a fault path to database operations.

## 10. Flow vs Apex vs Workflow vs Process Builder
| Feature | Flow | Apex | Workflow Rules (Retiring) | Process Builder (Retiring) |
| :--- | :--- | :--- | :--- | :--- |
| **Complexity** | High (Declarative) | Very High (Programmatic) | Low | Medium |
| **UI / Screens** | Yes | No (requires LWC/Aura) | No | No |
| **Delete Records** | Yes | Yes | No | No |
| **Execution Speed** | Fast (Especially Before-Save) | Fastest | Fast | Slow |
| **Maintanability** | High | Medium (needs dev skills) | Low | Low |

## 11. Enterprise Flow Governance
- Implement strict naming conventions (`Type_Object_Action`).
- Use Flow Trigger Explorer to manage order of execution.
- Bypass Toggles: Use Custom Hierarchy Settings or Custom Permissions to bypass flows during data loads.

---

# DEEP DIVE: Custom Labels in Flow

## 1. What are Custom Labels
**Definition:** Custom Labels are custom text values, up to 1,000 characters, accessible from Apex classes, Visualforce pages, Lightning components, and Flows.
**Purpose:** To present data to users in their native language and to centralize text string management.
**Metadata Architecture:** Stored as `CustomLabel` metadata type. They are highly cacheable and do not consume SOQL limits.
**Translation Support:** Fully integrated with the Translation Workbench, allowing a single label to dynamically render in the user's localized language.

## 2. Why Custom Labels are Important in Flow
- **Avoiding Hardcoded Text:** Hardcoding error messages or instructions in a Flow makes it brittle. Changes require deploying a new Flow version.
- **Maintainability:** Admins can change a Custom Label value in Production without creating a new Flow version.
- **Reusability:** A single label (e.g., `Error_ContactAdmin`) can be used across 50 different flows.
- **Localization & Multi-Language Implementations:** For global rollouts, Custom Labels automatically render the correct language based on the viewing user's Locale/Language settings.

## 3. Creating Custom Labels
Created via **Setup > User Interface > Custom Labels**.
- **Label Name:** The UI display name.
- **API Name:** e.g., `Flow_Welcome_Message`.
- **Value:** The actual text string.
- **Categories:** Text fields to group labels (e.g., `Flows`, `Errors`, `Sales`).
- **Protected Labels:** In managed packages, protected labels cannot be seen or edited by the subscriber.
- **Translation Workbench Integration:** Once created, you add translations via Translation Workbench -> Translate.

## 4. Using Custom Labels in Flow
To reference a Custom Label in Flow Builder, use the syntax: `{!$Label.Your_Label_API_Name}`.
They can be used in:
- **Display Text Components:** Directly insert `{!$Label.Greeting}`.
- **Formula Resources:** E.g., `IF(IsNew, {!$Label.New_Record}, {!$Label.Existing_Record})`.
- **Decision Elements:** Checking against a translated string (though checking against API names is safer).
- **Assignment Elements:** Assigning a label to a text variable.
- **Error Messages & Notifications:** Custom Fault screens.

## 5. Screen Flow Examples
- **Welcome Messages:** Instead of typing 'Welcome', use `{!$Label.Portal_Welcome_Msg}` in a Display Text component.
- **Instructions:** `{!$Label.Provide_Details_Instruction}`.
- **Validation Errors:** When configuring Input Validation, set the Error Message to `{!$Label.Val_Error_DateInPast}`.
- **Help Text:** Use labels for standard help bubbles to ensure translations.

## 6. Record-Triggered Flow Examples
- **Email Notifications:** Populate the Subject and Body using Custom Labels mapped to Text Templates.
- **Chatter Messages:** Use an Action element to Post to Chatter, setting the Message parameter to `{!$Label.Chatter_DealWon}`.
- **Task Descriptions:** When a Flow creates a Follow-up Task, set the Description to `{!$Label.Task_Desc_FollowUp}`.

## 7. Multi-Language Flow Design
With Translation Workbench active, you define translations for your Custom Labels. When the Flow executes, Salesforce automatically detects the running user's context.
**Example Label: `Case_Submission_Success`**
- **English (Default):** 'Your case has been submitted.'
- **French (`fr`):** 'Votre demande a été soumise.'
- **German (`de`):** 'Ihr Fall wurde eingereicht.'
- **Japanese (`ja`):** 'ケースが送信されました。'
- **Hindi (`hi`):** 'आपका केस सबमिट कर दिया गया है।'
*No flow branching is needed! A single Display Text component `{!$Label.Case_Submission_Success}` dynamically swaps the content based on the user.*

## 8. Custom Labels vs Flow Constants vs Custom Metadata
### Custom Labels vs Flow Constants
| Feature | Custom Labels | Flow Constants |
| :--- | :--- | :--- |
| **Purpose** | UI Text, user-facing messaging | Fixed values for internal logic (IDs, limits) |
| **Scope** | Global across the entire Org | Local to the specific Flow |
| **Reusability** | Extremely High | None (unless duplicated) |
| **Translation** | Supported (Translation Workbench) | Not supported |
| **Maintenance** | Edited outside Flow (Admin friendly) | Requires editing Flow and saving new version |

### Custom Labels vs Custom Metadata
| Feature | Custom Labels | Custom Metadata Types |
| :--- | :--- | :--- |
| **Purpose** | Simple text, UI messaging | Complex configurations, mapped fields, thresholds |
| **Data Storage** | Single Text String | Multiple fields (Text, Number, Checkbox, etc.) |
| **Translation** | Out-of-the-box support | Limited/Complex to translate field values |
| **Querying** | `$Label.X` (No limits) | Required `Get Records` or formula limits |

## 9. Real Project Scenarios
- **Warranty Management System:** An Experience Cloud portal for global dealers. When a dealer submits a claim, the Screen Flow utilizes `{!$Label.Claim_Submitted}`. A Japanese dealer sees Japanese; a US dealer sees English. All managed via one Flow.
- **Banking Application:** A Record-Triggered flow on Loan Status update sends a Chatter Post and Email. Using `{!$Label.Loan_Approved}` ensures the automated email adopts the language of the Loan Officer.
- **Insurance Application:** Complex fault handling in Flows uses a standardized `{!$Label.System_Error_Please_Contact_IT}`.

## 10. Common Mistakes & Best Practices
**Mistakes:**
- *Hardcoding:* Hardcoding screen text. Requires a full deployment lifecycle to change a typo.
- *Config Data:* Using Custom Labels for Configuration Data (e.g., storing an Integration API Key). Use Custom Metadata instead.
- *Duplicate Labels:* Creating `Error1`, `Error2` without centralized strategy.

**Best Practices:**
- **Naming Convention:** `[Domain]_[Component]_[Purpose]`. E.g., `Flow_Screen_Welcome` or `Comm_Email_Subject`.
- **Categorization:** Diligently use the 'Category' field on Custom Labels to make them searchable in Setup.
- **Fault Screens:** Always map generic fault screens to a standardized Label.

---

# Interview Questions

### Beginner
**Q: What is the difference between a Screen Flow and an Autolaunched Flow?**
A: Screen Flows require user interaction and display UI components. Autolaunched flows run in the background automatically and have no UI.

### Intermediate
**Q: Why would you use a Custom Label in a Flow instead of just typing the text?**
A: To support multi-language translations automatically, to allow Admins to change the text without versioning the Flow, and to reuse the same text across multiple flows and Apex classes.

**Q: How do you bulkify a Flow?**
A: By keeping DML elements (Create, Update, Delete) and SOQL (Get Records) outside of Loops. You assign records to a Collection Variable inside the loop, and perform the DML on the Collection outside the loop.

### Advanced
**Q: Explain the difference between Before-Save and After-Save Flows in execution context.**
A: Before-Save flows run before the database commit, updating the `$Record` directly without a separate DML operation, making them 10x faster. After-Save flows run after the record is saved, allowing access to system fields like the newly created Id, and are used to query or perform DML on related records.

### Architect-Level
**Q: You are designing a globally distributed Experience Cloud site spanning 15 languages. Complex flow screens are required. How do you architect the localization to minimize technical debt?**
A: I would build a single set of Screen Flows. Every user-facing text element (Display Text, Field Labels, Error Messages, Choices) would reference a Custom Label (e.g., `{!$Label.Portal_X}`). I would utilize the Translation Workbench to maintain translations for all 15 languages. The Flow will dynamically render the correct language based on the community user's locale setting. No routing or language-specific Flow clones are created, minimizing maintenance to zero for flow logic when language changes occur.

---

# Revision Summary
- **Flow Architecture:** Declarative logic, compiles to Java, executes via engine.
- **Flow Types:** Screen, Record-Triggered (Before/After), Scheduled, Autolaunched, Orchestration.
- **Performance:** Bulkify collections, avoid DML in loops, prefer Before-Save.
- **Custom Labels:** Crucial for UI text, multi-language support (Translation Workbench), reusability, and separating configuration from logic.
- **Enterprise Design:** Centralize messaging, leverage subflows, use Flow Trigger Explorer.
