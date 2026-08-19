# Notifications – Toast Messages

# 1. Introduction

In modern Salesforce Lightning Web Components (LWC) development, providing timely, clear, and non-intrusive feedback to users is a critical aspect of User Experience (UX). **Notifications** are UI elements designed to inform users about the status of the system, background processes, or the results of their interactions. 

**Toast messages** are a specific, standardized type of notification in the Salesforce Lightning Design System (SLDS). They appear temporarily at the top of the screen (or container) to communicate a brief message about an action's outcome. 

Notifications and toast messages are important because they close the interaction loop. When a user performs an action, the system must respond. Without immediate feedback, users may click a button multiple times, assume the system is broken, or navigate away before a process completes.

**When Toast Messages Should Be Used:**
Toast messages should be used for transient, high-level feedback that does not require the user to stop their current workflow.

**Automotive CRM Examples:**
*   ✅ **Warranty Claim submitted successfully** (Success Toast)
*   ✅ **Work Order updated successfully** (Success Toast)
*   ✅ **Invoice generated successfully** (Success Toast)
*   ✅ **Spare Part order created** (Success Toast)
*   ❌ **SAP integration failed** (Error Toast - alerting the user the sync failed)
*   ❌ **Invalid user input** (Error Toast - when field-level validation isn't enough)
*   ✅ **Record deleted successfully** (Success Toast)

---

# 2. What is a Toast Message?

### Definition
A Toast Message is a lightweight, temporary, and non-blocking notification element that slides in at the top of the user's screen to provide feedback on an action or system event.

### Purpose
The primary purpose of a toast is to acknowledge user interactions (like a save, delete, or system error) without interrupting their current task or requiring them to dismiss a heavy modal window. 

### Non-blocking UI Feedback vs. Modal
*   **Toast:** Non-blocking. The user can continue reading the page or typing in a form while the toast is visible. It disappears automatically (in most modes).
*   **Modal:** Blocking. The user *must* interact with the modal (click OK, Cancel, etc.) before they can return to the underlying page.

### Notification Flow Diagram

```text
[ User Action ] (e.g., Clicks "Submit Warranty Claim")
       ↓
[ Processing ] (e.g., Apex Controller creates record)
       ↓
[ Success / Error / Warning / Info ] (Outcome determined)
       ↓
[ Toast Notification ] (Fires on the UI)
       ↓
[ User Feedback ] (User reads the result and continues working)
```

---

# 3. Why Use Toast Messages?

Using toast messages is an architectural and UX best practice in Salesforce for several reasons:

*   **Immediate feedback:** Users instantly know if their action succeeded or failed.
*   **Better user experience:** Prevents user frustration and double-clicking.
*   **Non-blocking interaction:** Users don't have to break their flow to acknowledge the message.
*   **Clear status communication:** Standardized colors (green, red, yellow, grey) instantly convey meaning.
*   **Consistent Salesforce UI:** Using standard toast APIs ensures your custom components look and behave exactly like standard Salesforce components.
*   **Reduced need for custom notification components:** You don't need to build and maintain custom banners or popups for simple feedback.

**Appropriate Salesforce Use Cases:**
*   Confirming a CRUD operation (Create, Update, Delete).
*   Alerting the user that an asynchronous process (like a batch job or callout) has been successfully queued.
*   Displaying high-level page errors (e.g., "Unable to load Vehicle Data").

---

# 4. Toast Variants

Salesforce provides four standard variants for toast messages, mapping to different colors, icons, and SLDS utility classes.

| Variant | Purpose | Example |
|---------|---------|---------|
| `success` | Indicates a successful operation (Green). | Warranty Claim submitted successfully. |
| `error` | Indicates an operation failed (Red). | SAP callout failed. |
| `warning` | Indicates a potential problem or cautionary state (Yellow). | Spare Part stock is low. |
| `info` | Provides general system information (Grey). | Background synchronization started. |

**When to use each:**
*   **Use `success`** strictly for confirmed positive outcomes.
*   **Use `error`** for hard stops, failed DML, or broken integrations.
*   **Use `warning`** when a user can proceed, but they need to be aware of a risk (e.g., "This Dealer's contract expires in 5 days").
*   **Use `info`** for neutral updates (e.g., "Refreshing grid data").

---

# 5. Lightning Toast API

The **Lightning Toast API** (`lightning/toast`) is the modern, recommended approach for triggering toast messages in LWC. Introduced to decouple toast generation from standard DOM events, it acts as a standalone utility module.

### Configuration
*   **`label`**: (String) The bold heading of the toast (previously `title` in older APIs).
*   **`message`**: (String) The detailed text of the toast.
*   **`variant`**: (String) `success`, `error`, `warning`, or `info`.
*   **`mode`**: (String) `dismissable`, `pester`, or `sticky`.
*   **`messageLinks`**: (Object) Used to insert clickable URLs into the message.

### Modern LWC Example

```javascript
// 1. Import the modern Toast utility
import Toast from 'lightning/toast';
import { LightningElement } from 'lwc';

export default class ModernToastExample extends LightningElement {
    
    showSuccess() {
        // 2. Call Toast.show() and pass the configuration object
        Toast.show({
            label: 'Success',
            message: 'Warranty Claim created successfully.',
            variant: 'success',
            mode: 'dismissable'
        });
    }
}
```

**Line-by-Line Explanation:**
1.  `import Toast from 'lightning/toast';` — Imports the `Toast` utility module from the standard library.
2.  `Toast.show({ ... });` — Invokes the `show` method on the `Toast` object.
3.  `label: 'Success'` — Sets the bold header text of the toast.
4.  `message: 'Warranty Claim created...'` — Sets the body text providing the specific detail.
5.  `variant: 'success'` — Renders the toast in green with a checkmark icon.
6.  `mode: 'dismissable'` — The toast will auto-close after 3 seconds, but the user can click the 'X' to close it sooner.

**IMPORTANT:** This is the *current recommended approach* for modern LWC development. It is cleaner, does not rely on DOM event bubbling, and is easier to mock in Jest tests.

---

# 6. ShowToastEvent

The `ShowToastEvent` is the older, yet extremely widely used, event-based approach. Because the modern `lightning/toast` API is relatively new, 90%+ of existing Salesforce projects still utilize `ShowToastEvent`.

### Configuration
*   **`title`**: (String) Equivalent to `label` in the new API.
*   **`message`**: (String) The body text.
*   **`variant`**: (String) `success`, `error`, `warning`, `info`.
*   **`mode`**: (String) `dismissable`, `pester`, `sticky`.

### Example

```javascript
// 1. Import the specific event class
import { ShowToastEvent } from 'lightning/platformShowToastEvent';
import { LightningElement } from 'lwc';

export default class LegacyToastExample extends LightningElement {
    
    showSuccess() {
        // 2. Instantiate the event with configuration
        const event = new ShowToastEvent({
            title: 'Success',
            message: 'Record created successfully.',
            variant: 'success',
            mode: 'dismissable'
        });
        
        // 3. Dispatch the event up the DOM tree
        this.dispatchEvent(event);
    }
}
```

**Line-by-Line Explanation:**
1.  `import { ShowToastEvent }...` — Imports the custom event class designed to bubble up to the Lightning framework.
2.  `const event = new ShowToastEvent({...})` — Creates a new instance of the event with the provided settings. Note it uses `title`, not `label`.
3.  `this.dispatchEvent(event);` — Fires the event. The underlying Salesforce application (Lightning Experience) listens for this event at the top level and renders the toast.

**IMPORTANT:** This approach is crucial for understanding and maintaining existing codebases. It is also sometimes required in certain Experience Cloud contexts or older Aura wrappers where the modern API might not yet be fully supported.

---

# 7. Lightning Toast API vs ShowToastEvent

| Feature | `lightning/toast` (Modern) | `ShowToastEvent` (Older/Legacy) |
|---------|----------------------------|---------------------------------|
| **API Architecture** | Utility Module | Custom DOM Event |
| **Import Statement** | `import Toast from 'lightning/toast';` | `import { ShowToastEvent } from 'lightning/platformShowToastEvent';` |
| **Execution** | `Toast.show({...})` | `this.dispatchEvent(new ShowToastEvent({...}))` |
| **Header Property** | `label` | `title` |
| **Recommended Usage**| **New LWC Development** | Maintaining existing projects |
| **Dependency** | Independent of DOM location | Requires component to be attached to DOM to bubble events |
| **Flexibility** | Highly flexible, callable from external JS files easily | Tied to `this.dispatchEvent`, harder to abstract into pure JS helpers |
| **Experience Cloud** | Supported in modern LWR sites | Supported in Aura sites, historically the standard |

**Which to prefer?**
For *new development*, always prefer `lightning/toast` (`Toast.show()`). It avoids the overhead of DOM event bubbling, makes utility classes easier to write (since you don't need to pass `this` context to dispatch the event), and aligns with modern modular JavaScript practices.

---

# 8. Toast Title

The **Title** (or `label` in the modern API) is the bolded headline of the toast. 

### Importance
The title is the first thing a user scans. It should instantly categorize the notification before the user reads the detailed message.

### Good vs. Bad Titles

*   **Success:**
    *   *Bad:* "Success" (Too generic)
    *   *Good:* "Claim Submitted" (Specific to the action)
*   **Error:**
    *   *Bad:* "System Exception" (Scary, technical)
    *   *Good:* "Integration Failed" or "Update Failed" (Clear, action-oriented)
*   **Warning:**
    *   *Bad:* "Warning"
    *   *Good:* "Low Stock Warning"
*   **Information:**
    *   *Bad:* "Info"
    *   *Good:* "Processing Report"

---

# 9. Toast Message

The **Message** is the body text of the toast. It provides the specific detail of what happened.

### Guidelines
*   **Dynamic Values:** Use template literals to include record names or IDs (e.g., "Claim #1002 updated.")
*   **User-friendly Language:** Speak to the business user, not the developer.
*   **Avoiding Technical Errors:** Never expose raw SQL, SOQL, or NullPointerExceptions.
*   **Meaningful Feedback:** Tell them exactly what failed or succeeded.

**Bad:**
> "Exception occurred: System.DmlException: Update failed. First exception on row 0; first error: FIELD_CUSTOM_VALIDATION_EXCEPTION."

**Better:**
> "Unable to submit the warranty claim. Please ensure all mandatory dealer fields are populated and try again."

**Why?** Users cannot fix a "DmlException". They *can* fix a missing dealer field. Translate technical errors into actionable business steps.

---

# 10. Toast Mode

The `mode` property dictates how long the toast stays on the screen and how the user interacts with it.

| Mode | Behavior | When to Use | Example |
|------|----------|-------------|---------|
| `dismissable` (Default) | Stays for 3 seconds. Has a close ('X') button. | Standard success or informational messages. | "Invoice generated." |
| `pester` | Stays for 3 seconds. **No** close button. | Warnings or info the user doesn't need to interact with but should see. | "Data refreshing..." |
| `sticky` | Stays indefinitely until the user clicks the close ('X') button. | Critical errors or warnings the user *must* acknowledge. | "SAP connection failed. Please contact IT." |

---

# 11. Success Toast

Success toasts validate that the user's intended action completed securely on the backend.

### Example: Record Creation (Modern API)

```javascript
import Toast from 'lightning/toast';
import { LightningElement } from 'lwc';

export default class SuccessToastExample extends LightningElement {
    
    handleSuccess() {
        // 1. Invoke Toast.show()
        Toast.show({
            // 2. Clear, action-specific label
            label: 'Warranty Claim Created',
            // 3. Detailed message
            message: 'Warranty Claim was created and routed for approval.',
            // 4. Variant triggers the green UI
            variant: 'success',
            // 5. Normal timeout behavior
            mode: 'dismissable'
        });
    }
}
```

*Explanation:* This clearly informs the user not only that it was created, but what the immediate next system step is ("routed for approval").

---

# 12. Error Toast

Error notifications inform the user that their request could not be fulfilled.

### Scenarios
*   **Apex Errors:** Backend validation or DML failures.
*   **Validation Errors:** Complex multi-field validations.
*   **Callout Errors:** Remote system (e.g., SAP) timeouts.
*   **Network Errors:** User went offline.

### Example: Apex Error (Older API for variety)

```javascript
import { ShowToastEvent } from 'lightning/platformShowToastEvent';
import { LightningElement } from 'lwc';

export default class ErrorToastExample extends LightningElement {
    
    handleError(errorMsg) {
        this.dispatchEvent(new ShowToastEvent({
            title: 'Submission Failed',
            message: errorMsg,
            variant: 'error',
            mode: 'sticky' // Keep visible until user dismisses
        }));
    }
}
```

**Why Raw Exceptions Should Not Be Displayed:**
Raw exceptions (like `System.NullPointerException` or stack traces) cause user panic, provide no actionable steps for resolution, and can expose internal database schema (security risk).

---

# 13. Warning Toast

Warnings are for cautionary states. The operation might have succeeded, or the user is about to do something risky, but it is not a hard error.

### Examples:
*   **Low stock:** "Only 2 brake pads remaining in inventory."
*   **Claim SLA:** "This claim is 1 day away from breaching SLA."
*   **Duplicate record:** "A similar Warranty Claim exists for this VIN."

**Difference between Warning and Error:**
An error means the system *prevented* an action. A warning means the system *allows* the action (or is just reporting state), but the user should proceed with caution.

---

# 14. Information Toast

Information toasts provide neutral context.

### Examples:
*   **Processing started:** "Compiling monthly dealer report..."
*   **Background job submitted:** "Syncing vehicles with SAP. This may take a few minutes."
*   **Data refresh:** "Updating pricing data."

**When Informational Feedback is Useful:**
Use `info` when an action is asynchronous. If a user clicks a button and nothing happens for 10 seconds, they will click it again. An `info` toast immediately acknowledges the click while the backend works.

---

# 15. Dynamic Toast Messages

Toasts are most useful when they are contextual. Use JavaScript Template Literals (`\``) to inject dynamic variables.

### Examples:
*   Record Name: "Account Acme Corp updated."
*   Claim Number: "Warranty Claim CL-99882 created."

### Example

```javascript
import Toast from 'lightning/toast';
import { LightningElement, api } from 'lwc';

export default class DynamicToastExample extends LightningElement {
    @api claimNumber = 'WC-10025';

    showDynamicToast() {
        Toast.show({
            label: 'Claim Created',
            // Using template literals to inject the claimNumber variable
            message: `Warranty Claim ${this.claimNumber} was created successfully.`,
            variant: 'success'
        });
    }
}
```

**Line-by-Line Explanation:**
1.  `@api claimNumber` — A public property holding the claim string.
2.  `` message: `Warranty Claim ${this.claimNumber}...` `` — The backticks allow variable interpolation. `${this.claimNumber}` dynamically resolves to `WC-10025` at runtime.

---

# 16. Toast Messages with Apex

This is the most common use case: handling the result of an Imperative Apex call.

### Flow Diagram
```text
LWC Button Click -> JS imperative call -> Apex Method 
-> Success (Try) -> Return Data -> Success Toast
-> Error (Catch) -> Return Exception -> Error Toast
```

### Complete Imperative Example

```javascript
import { LightningElement } from 'lwc';
import submitClaim from '@salesforce/apex/ClaimController.submitClaim';
import Toast from 'lightning/toast';

export default class ApexToastExample extends LightningElement {
    
    async handleSave() {
        try {
            // 1. Call Apex imperatively and wait for result
            const result = await submitClaim({ vehicleId: 'xxx' });
            
            // 2. On success, show success toast
            Toast.show({
                label: 'Success',
                message: `Claim ${result.ClaimNumber} submitted.`,
                variant: 'success'
            });

        } catch (error) {
            // 3. On error, extract message and show error toast
            const errorMessage = error.body ? error.body.message : 'Unknown error';
            
            Toast.show({
                label: 'Apex Error',
                message: errorMessage,
                variant: 'error',
                mode: 'sticky'
            });
        }
    }
}
```

---

# 17. Toast Messages with @wire

Wired properties and functions are reactive. You must be very careful when using toasts in `@wire` handlers to avoid infinite notification loops.

```javascript
import { LightningElement, wire } from 'lwc';
import getVehicleData from '@salesforce/apex/VehicleController.getVehicleData';
import Toast from 'lightning/toast';

export default class WireToastExample extends LightningElement {
    
    @wire(getVehicleData, { dealerId: '123' })
    wiredVehicles({ error, data }) {
        if (data) {
            // Usually NO toast here. We don't want a toast every time data loads!
        } else if (error) {
            // Show error toast on initial load failure
            Toast.show({
                label: 'Data Load Failed',
                message: 'Could not retrieve Vehicle information.',
                variant: 'error'
            });
        }
    }
}
```

**Toast vs. Inline Error for @wire:**
*   **Appropriate:** An inline error message (e.g., a custom illustration component in the UI) is usually *better* for `@wire` errors than a toast, because if the user navigates away and back, the toast might pop up annoyingly every time the component renders.
*   **Inappropriate:** Never show a `success` toast inside a `@wire` when data is retrieved. It will fire on every page load, annoying the user.

---

# 18. Toast Messages with Forms

Standard LWC forms (`lightning-record-edit-form`) emit events you can catch to show toasts.

```html
<template>
    <lightning-record-edit-form 
        object-api-name="Warranty_Claim__c"
        onsuccess={handleSuccess}
        onerror={handleError}>
        
        <lightning-messages></lightning-messages>
        <lightning-input-field field-name="Name"></lightning-input-field>
        <lightning-button type="submit" label="Save"></lightning-button>
    </lightning-record-edit-form>
</template>
```

```javascript
import { LightningElement } from 'lwc';
import Toast from 'lightning/toast';

export default class FormToastExample extends LightningElement {
    
    handleSuccess(event) {
        // Form submitted and saved successfully
        Toast.show({
            label: 'Record Saved',
            message: `Record ID: ${event.detail.id} saved successfully.`,
            variant: 'success'
        });
    }

    handleError(event) {
        // Form failed to save (lightning-messages usually handles inline UI, 
        // but we can add a toast for severe errors)
        Toast.show({
            label: 'Save Failed',
            message: event.detail.detail, // Extracts standard form error
            variant: 'error'
        });
    }
}
```

---

# 19. Toast Messages After CRUD Operations

| Operation | Recommended Feedback | Why? |
|-----------|----------------------|------|
| **Create** | Success toast | Confirms a new ID was generated and saved to the database. |
| **Update** | Success toast | Reassures the user their edits were committed. |
| **Delete** | Success toast | Reassures the user the data is permanently gone. |
| **Read** | Usually NO toast | Reading data is a passive expectation. Users don't need a toast to tell them the page loaded; they can see the data on the page. |

---

# 20. Toast Messages and Salesforce Errors

Salesforce error objects are deeply nested depending on where they come from (Apex, LDS, Callouts). You need a utility to parse them so you don't show `[object Object]` in your toast.

### Reusable Error Handling Function

```javascript
/**
 * Extracts a user-friendly error message from various Salesforce error responses
 */
export function getErrorMessage(error) {
    if (!error) return 'Unknown error occurred.';

    // 1. Array of errors (often from LDS)
    if (Array.isArray(error.body)) {
        return error.body.map(e => e.message).join(', ');
    }
    
    // 2. Single object error body (often from Apex)
    if (error.body && typeof error.body.message === 'string') {
        return error.body.message;
    }
    
    // 3. Page/Field level errors (often from standard forms)
    if (error.detail && typeof error.detail === 'string') {
        return error.detail;
    }

    // 4. JS standard error
    if (error.message) {
        return error.message;
    }

    return 'An unexpected error occurred.';
}
```

**Usage:**
```javascript
catch (error) {
    Toast.show({
        label: 'Error',
        message: getErrorMessage(error),
        variant: 'error'
    });
}
```
*Why this helper?* Different APIs return different error shapes. This function sanitizes them into a readable string.

---

# 21. Toast Messages and Validation

Do not abuse toast messages for basic field validation. 

### Validation Strategy:
*   **Field Error (Inline):** User forgets an email format? Use `setCustomValidity()` on the field. Show it in red text *under* the field.
*   **Form Error (Inline Banner):** Multiple fields failed? Use `<lightning-messages>` at the top of the form.
*   **Operation Error (Toast):** System failure, backend integration failure, or generic record save failure? Use a Toast.

**Why?** If a user leaves 5 required fields blank, firing 5 error toasts (or one massive error toast) forces the user to memorize the errors before closing the toast. Inline errors stay next to the fields that need fixing.

---

# 22. Toast Messages with Lightning Data Service

When using `lightning/uiRecordApi` methods, toast usage is identical to Apex.

```javascript
import { LightningElement } from 'lwc';
import { updateRecord } from 'lightning/uiRecordApi';
import Toast from 'lightning/toast';

export default class LdsToastExample extends LightningElement {
    
    handleUpdate(recordInput) {
        updateRecord(recordInput)
            .then(() => {
                // Success -> Toast
                Toast.show({
                    label: 'Success',
                    message: 'Vehicle Record updated via LDS.',
                    variant: 'success'
                });
            })
            .catch(error => {
                // Error -> Toast
                Toast.show({
                    label: 'Update Failed',
                    message: error.body.message,
                    variant: 'error'
                });
            });
    }
}
```
LDS simplifies operations because it manages the cache automatically; the toast is the final step to confirm the UI is now in sync with the database.

---

# 23. Toast Messages with Callouts and Integrations

**Enterprise Scenario:** An agent clicks "Send Claim to SAP".

```text
Salesforce (LWC)
       ↓ (Apex Callout)
SAP Integration
       ↓ (Response: 500 Internal Error)
Success / Error logic in Apex
       ↓ (Throws AuraHandledException)
Error Toast in LWC
```

**Handling SAP Errors:**
If SAP returns: `{"errorCode": "E_001", "errorText": "JDBC Connection Timeout to SAP DB node 4"}`, **DO NOT** show this exact string.

*Instead, parse it in Apex and send to LWC:*
"The SAP system is currently unavailable. Please try syncing the Warranty Claim again in 5 minutes."

This provides meaningful feedback without exposing sensitive technical SAP architecture details to the end-user.

---

# 24. Toast Messages with Asynchronous Apex

Toasts are strictly **client-side, current-session UI constructs**.

### The Limitation
If you call a `@future` method, Queueable, or Batch job from LWC:
1.  LWC receives a success response immediately ("Job Queued").
2.  You show an `info` toast: "Processing started."
3.  **The problem:** When the job finishes 2 minutes later, the LWC *cannot* easily show a "Finished" toast. The user might have closed the browser, changed tabs, or navigated away.

### Better Approaches for Async Completion:
*   Use **Platform Events**. Have an LWC use `lightning/empApi` to subscribe to a Platform Event fired by the Batch job. When the event arrives, *then* show the success Toast (if the user is still on the page).
*   Use standard bell notifications (Custom Notifications API) for persistent alerts that survive page navigation.

---

# 25. Toast Message Links

The modern `lightning/toast` API supports message links to improve navigation.

```javascript
import Toast from 'lightning/toast';
import { LightningElement } from 'lwc';

export default class LinkToastExample extends LightningElement {
    showToastWithLink() {
        Toast.show({
            label: 'Claim Created',
            message: 'Warranty Claim {0} created successfully. {1}',
            messageLinks: {
                0: {
                    url: '/lightning/r/Warranty_Claim__c/a0A.../view',
                    label: 'WC-10025'
                },
                1: {
                    url: '/lightning/o/Warranty_Claim__c/list',
                    label: 'View all claims'
                }
            },
            variant: 'success'
        });
    }
}
```
*Note: In the older `ShowToastEvent`, the syntax relies on passing a `messageData` array containing objects with `url` and `label` properties.*
Providing links in a success toast is excellent UX—it immediately answers the user's implicit question: *"It's created, now take me to it."*

---

# 26. Toast vs Modal vs Inline Message

| Feature | Toast (`lightning/toast`) | Modal (`lightning/modal`) | Inline Message |
|---------|---------------------------|---------------------------|----------------|
| **Blocking Behavior** | Non-blocking | **Blocking** (requires interaction) | Non-blocking |
| **User Attention** | High (slides in, colored) | Highest (dims background) | Medium (embedded in page) |
| **Persistence** | Temporary (usually 3s) | Until dismissed explicitly | Permanent while condition exists |
| **Best Use Case** | Operation outcomes (Save success) | Confirmation dialogs ("Are you sure?") | Form validation, missing data alerts |
| **Salesforce Example** | "Record Saved" | "Delete this record?" | "Email format invalid" |

---

# 27. Toast vs Lightning Alert

*   **Toast:** Best for positive confirmations or non-critical errors. Disappears automatically (in `dismissable` mode).
*   **Lightning Alert (`lightning/alert`):** A modern LWC replacement for `window.alert()`. It is a small modal that dims the background and forces the user to click "OK".
*   **When to use Alert:** When an error is so critical that the user *must* acknowledge it before continuing (e.g., "Session expired, you must log in again").

---

# 28. Toast Accessibility

Salesforce designs standard toasts to be WCAG compliant.

*   **Screen Readers:** Toasts use `role="alert"` or `role="status"`. Screen readers (like NVDA or VoiceOver) automatically announce the toast text when it appears.
*   **Color Not the Only Indicator:** Standard standard variants include icons (check for success, exclamation for warning). This ensures color-blind users can still differentiate severity.
*   **Avoiding Excessive Notifications:** Firing 5 toasts in a row creates chaos for a screen reader user. Combine them into one summary message.

---

# 29. Toast UX Best Practices

✅ **Keep messages concise:** "Record Updated." not "The record you were looking at has been successfully updated in the database."
✅ **Tell users what to do next:** "Failed. Try checking your network connection."
✅ **Avoid technical jargon:** No "DML Exception on row 0".
✅ **Avoid duplicate notifications:** Disable buttons immediately on click to prevent double-firing.
❌ **Do not use toast for every event:** Clicking a tab shouldn't fire a toast.
❌ **Do not use sticky mode for success:** Users hate having to manually close success banners.

---

# 30. Toast Notification Design Patterns

### Pattern 1: Success After Save
`User submits form -> Disable Button -> Save to Server -> Success Toast -> Re-enable Button / Navigate`

### Pattern 2: Error After Apex
`User Action -> Apex -> Try/Catch -> Error Toast -> Reset UI state`

### Pattern 3: Warning Before Action
*(Generally, modals are better for this, but a toast can be used to notify state)*:
`User opens Claim -> Detect missing SLA data -> Warning Toast ("SLA data missing")`

### Pattern 4: Background Processing
`Action -> Queueable Job -> "Processing started" Info Toast` (Does not notify on finish unless using Platform Events).

---

# 31. Preventing Duplicate Toasts

### Common Reasons for Duplicates
*   **Multiple event handlers:** An `onsuccess` event bubbles up, and both the child and parent component handle it and fire a toast.
*   **Wire function looping:** Putting a toast directly inside a `@wire` that provisions multiple times.
*   **Double Clicks:** User clicks "Save" twice before the server responds.

### Strategies
1.  **Disable the Save button** immediately upon click (`this.isSaving = true`).
2.  Use `event.stopPropagation()` on component boundaries if using older events.
3.  Manage boolean flags in `@wire` to only fire a toast on the *first* error, not subsequent cache refreshes.

---

# 32. Toasts in Parent and Child Components

If a child component performs a DML operation, who shows the toast?

**Approach 1: Child Handles It (Encapsulated)**
The child imports `lightning/toast` and fires it. The parent knows nothing. Good for independent widgets.

**Approach 2: Centralized Notification (Delegated)**
The child dispatches a custom event (`this.dispatchEvent(new CustomEvent('save'))`). The parent catches it and fires the toast. 
*Why?* Useful if the parent needs to orchestrate multiple child saves and wants to show *one* combined success toast ("All 3 records saved") instead of 3 separate toasts.

---

# 33. Reusable Toast Utility

For enterprise apps, create a lightweight utility JS file (e.g., `c/toastUtility`).

**`toastUtility.js`**
```javascript
import Toast from 'lightning/toast';

export function showSuccess(label, message) {
    Toast.show({ label, message, variant: 'success' });
}

export function showError(label, message) {
    Toast.show({ label, message, variant: 'error', mode: 'sticky' });
}
```

**Why do this?**
*   **Consistency:** Forces all developers to use the same configuration.
*   **Maintainability:** If Salesforce changes the Toast API again, you only update one file.
*   **Reuse:** Drastically reduces boilerplate code in every LWC.

---

# 34. Common Mistakes

| Problem | Cause | Solution | Example |
|---------|-------|----------|---------|
| **Toast not appearing** | Using `ShowToastEvent` but not connected to DOM | Use `lightning/toast` API | Background utility JS throwing event |
| **Showing raw exceptions** | Catching Apex error and passing `.message` blindly | Parse the error and provide user-friendly text | "Validation Formula Failed" instead of `FIELD_CUSTOM_VALIDATION` |
| **Sticky success toasts** | Setting `mode: 'sticky'` on `success` variant | Change to `dismissable` | Success stays on screen forever |
| **Toast behind modal** | z-index conflicts in custom modals | Use standard `lightning/modal` which manages z-index for toasts correctly | Custom `<div>` modal overlays standard toast |

---

# 35. Debugging Toast Messages

**Checklist:**
1.  [ ] Did you import the correct API? (`lightning/toast` vs `platformShowToastEvent`)
2.  [ ] Are the properties named correctly? (`label` for new API, `title` for old API).
3.  [ ] Is the LWC actually executing the code block? (Add a `console.log` right before the Toast execution).
4.  [ ] If passing an error, is `error.body.message` defined? (If undefined, the toast might fail silently or show blanks).
5.  [ ] Did a page navigation event happen *before* the toast could render? (Toasts belong to the current page. If you navigate away via `NavigationMixin` simultaneously, the toast gets destroyed with the origin page).

---

# 36. Performance Considerations

*   **Avoid excessive toast creation:** Don't loop over 50 records and fire 50 toasts. Consolidate into one: "50 records updated."
*   **Memory Leaks:** If you create custom DOM listeners relying on toast lifecycles (rare, but possible), ensure you clean them up. Standard APIs manage memory well.
*   **Avoid heavy logic in toast constructors:** Keep the string construction lightweight.

---

# 37. Best Practices Checklist

*   ✅ **Use the modern Lightning Toast API** (`lightning/toast`) for new development where appropriate to decouple from DOM events.
*   ✅ **Understand ShowToastEvent** for maintaining existing projects.
*   ✅ **Use success** for successful operations.
*   ✅ **Use error** for failures (DML, integrations).
*   ✅ **Use warning** for potential problems (SLA risks).
*   ✅ **Use info** for general information (Async processing).
*   ✅ **Keep messages concise** and scannable.
*   ✅ **Use meaningful titles/labels** that define the action context.
*   ✅ **Avoid exposing technical errors** (Parse Apex exceptions).
*   ✅ **Use inline validation** for field-specific errors, reserve toasts for form/operation level.
*   ✅ **Avoid duplicate toasts** by disabling action buttons during processing.
*   ✅ **Choose toast mode appropriately** (dismissable for success, sticky for critical errors).
*   ✅ **Do not use toast as a security mechanism** (It's UI layer only; data must be secured in Apex).

---

# 38. Real Project Scenarios (Automotive CRM)

| # | User Action | Backend Operation | Result | Toast Variant | Appropriate Mode |
|---|-------------|-------------------|--------|---------------|------------------|
| 1 | Submits Warranty Claim | Apex inserts `Warranty_Claim__c` | Success | **Success** | `dismissable` |
| 2 | Edits Claim Details | LDS `updateRecord` | Success | **Success** | `dismissable` |
| 3 | Clicks "Send to SAP" | Queues Async Callout | Pending | **Info** | `dismissable` ("Sync Started") |
| 4 | (SAP Sync Fails)* | Platform Event -> LWC | Error | **Error** | `sticky` ("SAP Sync Failed") |
| 5 | Generates Invoice PDF | Visualforce rendering / Attachment | Success | **Success** | `dismissable` |
| 6 | Orders Spare Part | External OData API Callout | Success | **Success** | `dismissable` |
| 7 | Shipment Synchronized | Webhook update | Success | **Info/Success**| `dismissable` |
| 8 | Closes Work Order | Apex updates status | Success | **Success** | `dismissable` |
| 9 | Add Vehicle VIN | Inline form validation fails | Error | **None** (Use inline) | N/A |
| 10| Dealer Contract Expires | Loads Dealer Record page | Warning | **Warning** | `pester` |

*(Note for #4: Real-time async error reporting requires streaming/platform events, otherwise the toast only shows if the callout is synchronous).*

---

# 39. Complete End-to-End Example

**Scenario:** Submitting a Warranty Claim.

### 1. Apex Controller (`ClaimController.cls`)
```java
public with sharing class ClaimController {
    @AuraEnabled
    public static Warranty_Claim__c submitClaim(Id vehicleId, String issueDescription) {
        try {
            Warranty_Claim__c claim = new Warranty_Claim__c(
                Vehicle__c = vehicleId,
                Issue_Description__c = issueDescription,
                Status__c = 'Submitted'
            );
            insert claim;
            
            // Query to return the generated Name/Number
            return [SELECT Id, Name FROM Warranty_Claim__c WHERE Id = :claim.Id LIMIT 1];
        } catch (Exception e) {
            // Throw AuraHandledException to send cleanly to LWC
            throw new AuraHandledException('Unable to create Claim: ' + e.getMessage());
        }
    }
}
```

### 2. LWC HTML (`warrantyClaimForm.html`)
```html
<template>
    <lightning-card title="New Warranty Claim" icon-name="custom:custom31">
        <div class="slds-p-around_medium">
            <lightning-textarea 
                label="Issue Description" 
                value={description} 
                onchange={handleDescChange}
                required>
            </lightning-textarea>
            
            <div class="slds-m-top_medium">
                <lightning-button 
                    label="Submit Claim" 
                    variant="brand" 
                    onclick={saveClaim}
                    disabled={isSaving}>
                </lightning-button>
            </div>
            
            <template if:true={isSaving}>
                <lightning-spinner alternative-text="Loading" size="small"></lightning-spinner>
            </template>
        </div>
    </lightning-card>
</template>
```

### 3. LWC JavaScript (`warrantyClaimForm.js`)
```javascript
import { LightningElement, api } from 'lwc';
import submitClaim from '@salesforce/apex/ClaimController.submitClaim';
// 1. Import Modern API
import Toast from 'lightning/toast'; 
// 2. Import Legacy API (Included purely for comparison/existing code)
import { ShowToastEvent } from 'lightning/platformShowToastEvent'; 

export default class WarrantyClaimForm extends LightningElement {
    @api recordId; // Assume this is placed on a Vehicle__c record page
    description = '';
    isSaving = false;

    handleDescChange(event) {
        this.description = event.target.value;
    }

    async saveClaim() {
        if (!this.description) {
            // Inline validation preferred here, but omitted for brevity
            return;
        }

        this.isSaving = true; // Prevents duplicate clicks

        try {
            // 3. Call Apex
            const claim = await submitClaim({ 
                vehicleId: this.recordId, 
                issueDescription: this.description 
            });

            // 4. Modern API Implementation
            Toast.show({
                label: 'Claim Submitted',
                message: `Warranty Claim ${claim.Name} successfully routed to Dealer.`,
                variant: 'success',
                mode: 'dismissable'
            });

            /* 5. Legacy API Equivalent (Commented out)
            this.dispatchEvent(
                new ShowToastEvent({
                    title: 'Claim Submitted',
                    message: `Warranty Claim ${claim.Name} successfully routed to Dealer.`,
                    variant: 'success'
                })
            );
            */
           
            this.description = ''; // Reset form

        } catch (error) {
            // 6. Handle Apex Error
            const errorMsg = error.body ? error.body.message : 'Unknown error';
            
            Toast.show({
                label: 'Submission Failed',
                message: errorMsg,
                variant: 'error',
                mode: 'sticky'
            });
        } finally {
            this.isSaving = false; // Re-enable button
        }
    }
}
```

### 4. LWC Meta (`warrantyClaimForm.js-meta.xml`)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="[http://soap.sforce.com/2006/04/metadata](http://soap.sforce.com/2006/04/metadata)">
    <apiVersion>60.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__RecordPage</target>
    </targets>
</LightningComponentBundle>
```

---

# 40. Interview Questions & Answers

### Beginner Questions

**Q: What is a toast message?**
**A:** A non-blocking, temporary UI notification that pops up to provide the user with feedback about an action they just performed, such as saving or deleting a record.

**Q: What are the different toast variants?**
**A:** `success` (green), `error` (red), `warning` (yellow), and `info` (grey).

### Intermediate Questions

**Q: What is the Lightning Toast API?**
**A:** Introduced as `lightning/toast`, it is the modern utility module to invoke toast messages via `Toast.show()`. It decouples toast creation from DOM event bubbling, making it highly flexible.

**Q: What is the difference between Toast.show() and ShowToastEvent?**
**A:** `Toast.show()` is a modern JavaScript utility module execution. `ShowToastEvent` relies on creating a custom event and using `this.dispatchEvent()` to bubble the event up the DOM hierarchy until the framework catches it. 

**Q: Which toast approach should be preferred for new LWC development?**
**A:** `lightning/toast` should be preferred for new development because it is cleaner, doesn't require DOM attachment, and is easier to abstract into JS helper libraries.

**Q: What is the difference between `dismissable`, `pester`, and `sticky` modes?**
**A:** 
*   `dismissable`: Disappears in 3s, has a close button.
*   `pester`: Disappears in 3s, has NO close button.
*   `sticky`: Stays indefinitely until the user clicks the close button.

### Advanced Questions

**Q: How do you handle Apex errors in a toast?**
**A:** When calling imperative Apex, catch the exception in a `catch(error)` block. Extract the specific message (usually `error.body.message`), and pass that into the `message` parameter of an `error` configured toast. Never display the raw Exception object.

**Q: Should validation errors always be displayed as toast messages?**
**A:** No. Form-level or field-level validation (like an invalid email) should use inline validation (red text under the field) using `setCustomValidity`. Toasts should be reserved for system errors, backend DML failures, or successful completion.

**Q: What is the difference between toast and modal?**
**A:** A toast is non-blocking and temporary. A modal dims the background, blocks user interaction with the page, and requires explicit user action (clicking a button) to dismiss.

### Architect-Level Questions

**Q: Can toast messages be used as a security mechanism?**
**A:** Absolutely not. Toasts operate entirely on the client-side UI layer. Showing an error toast does not prevent database changes if the backend Apex is not properly secured with `WITH USER_MODE` or `isAccessible()` checks.

**Q: How do toast messages work with asynchronous Apex (like Queueable)?**
**A:** Poorly. A toast only lives in the active browser tab session. If a Queueable job takes 5 minutes, the user may have left the page. You should use a toast to notify "Job Started", and rely on Platform Events + Bell Notifications to alert the user when the job actually finishes.

**Q: How do you create a reusable toast utility for an enterprise project?**
**A:** Create an independent LWC JavaScript module (e.g., `toastUtils.js`) that imports `lightning/toast` and exports standard wrapper functions like `showSuccess(msg)` and `showError(msg)`. Other components import these functions, enforcing standard styling and reducing duplicate code.

---

# 41. Revision Summary

*   **Notifications:** Crucial for closing the UX interaction loop.
*   **Toast Messages:** Temporary, non-blocking SLDS UI elements.
*   **Lightning Toast API:** The modern approach (`import Toast from 'lightning/toast'`; `Toast.show()`).
*   **ShowToastEvent:** The older DOM-bubbling approach (`dispatchEvent(new ShowToastEvent())`).
*   **Variants:** `success`, `error`, `warning`, `info`.
*   **Mode:** `dismissable` (auto-close with X), `pester` (auto-close no X), `sticky` (manual close).
*   **Dynamic Messages:** Use JS template literals (`` `Record ${id} saved` ``).
*   **Message Links:** Supported natively to allow quick navigation to newly created records.
*   **Apex:** Use in `then/catch` or `async/await try/catch` blocks. Parse `error.body.message` before displaying.
*   **@wire:** Be extremely cautious. Only show error toasts, never success toasts, to avoid infinite loops.
*   **CRUD/LDS:** Show success for Create/Update/Delete. Do not show toasts for Read.
*   **Integrations:** Catch callout errors and display business-friendly messages, omitting raw JSON/XML errors.
*   **Async Apex limitations:** Toasts cannot natively wait for background jobs to finish if the user navigates away.
*   **Accessibility:** Ensure messages are meaningful; standard APIs handle screen reader ARIA roles.
*   **Duplicate Prevention:** Disable buttons (`isSaving = true`) on click to prevent multi-fire.
*   **Reusable Utility:** Abstract toast logic into a shared JS file for enterprise consistency.