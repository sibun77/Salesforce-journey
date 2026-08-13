# Conditional Rendering – if:true and if:false

# 1. Introduction

Conditional rendering is a fundamental concept in Salesforce Lightning Web Components (LWC). It allows developers to control exactly what HTML elements, base components, or custom components are displayed on the user interface based on the component's underlying JavaScript state. 

Instead of manually manipulating the DOM to show or hide elements, LWC relies on declarative template directives. When a JavaScript property changes, the framework automatically updates the DOM to reflect the new state. 

In a real-world Automotive CRM, conditional rendering allows you to display a "Submit Warranty Claim" button only to authorized service managers, show a loading spinner while fetching SAP invoice data, or display an empty state when a vehicle has no service history.

---

# 2. What is Conditional Rendering?

Conditional rendering is the process of dynamically adding or removing elements from the Document Object Model (DOM) based on boolean conditions. 

Unlike traditional DOM manipulation (where you might use `document.getElementById('myDiv').style.display = 'none'`), LWC uses a reactive data-binding approach. You define the condition in your HTML template, and the LWC engine handles the creation and destruction of the DOM nodes automatically.

**How it works:**
1. A boolean property or getter is evaluated in JavaScript.
2. The HTML template uses a directive to check this condition.
3. If `true`, the LWC engine creates and inserts the HTML into the DOM.
4. If `false`, the LWC engine removes the HTML from the DOM entirely.

```text
JavaScript State
      ↓
  Condition
      ↓
 True / False
      ↓
 Rendered UI
```

---

# 3. Conditional Rendering in LWC

LWC evaluates conditions strictly based on properties or getters defined in the component's JavaScript class. 

Because LWC templates do not support inline JavaScript expressions (e.g., `<template lwc:if={count > 5}>` is invalid), all evaluation logic must live in the JavaScript file.

The framework continuously monitors reactive properties. When a property used in a conditional directive changes, LWC triggers a rerender, efficiently updating only the parts of the DOM that are affected.

---

# 4. if:true Directive

> ⚠️ **LEGACY SYNTAX:** `if:true` is the older, legacy approach to conditional rendering in LWC. While it is still supported for backward compatibility, it is **no longer recommended for new development**.

The `if:true` directive renders the enclosed elements if the bound JavaScript property evaluates to a truthy value.

### Example:
```html
<template if:true={isVisible}>
    <p>Content is visible</p>
</template>
```

**Line-by-Line Explanation:**
* `<template if:true={isVisible}>`: The legacy directive checks the `isVisible` boolean property in the JS file. The `<template>` tag acts as an invisible wrapper; it doesn't render as a DOM element itself.
* `<p>Content is visible</p>`: This paragraph is inserted into the DOM only when `isVisible` is true.
* `</template>`: Closes the wrapper block.

---

# 5. if:false Directive

> ⚠️ **LEGACY SYNTAX:** Like `if:true`, `if:false` is legacy syntax and **should be avoided in new LWC development**.

The `if:false` directive renders the enclosed elements if the bound JavaScript property evaluates to a falsy value.

### Example:
```html
<template if:false={isVisible}>
    <p>Content is hidden</p>
</template>
```

**Line-by-Line Explanation:**
* `<template if:false={isVisible}>`: Evaluates `isVisible`. If it is `false` (or undefined/null), the block executes.
* `<p>Content is hidden</p>`: Inserted into the DOM only when `isVisible` is false.
* `</template>`: Closes the block.

*Use Case (Legacy):* Previously, developers used a combination of `if:true` and `if:false` to mimic `if-else` logic, which was verbose and prone to errors.

---

# 6. Modern Conditional Rendering

With the Spring '23 release, Salesforce introduced a modern suite of conditional directives: `lwc:if`, `lwc:elseif`, and `lwc:else`. **This is the current recommended syntax.**

### Example:
```html
<template lwc:if={isVisible}>
    <p>Visible Content</p>
</template>
```

**Why it replaced `if:true` and `if:false`:**
* **Readability:** It closely mirrors standard programming `if/else` structures.
* **Performance:** The LWC compiler optimizes these directives more effectively. Instead of evaluating multiple disjointed `<template if:true>` statements, the compiler understands the relationship between `lwc:if`, `lwc:elseif`, and `lwc:else`, evaluating only until a true condition is met.
* **Modern Standards:** It aligns LWC closer to other modern web frameworks and reduces template boilerplate.

---

# 7. lwc:elseif

The `lwc:elseif` directive allows you to chain multiple conditions. It must immediately follow a `<template lwc:if>` or another `<template lwc:elseif>`.

### Example:
```html
<template lwc:if={isAdmin}>
    <p>Admin Control Panel</p>
</template>
<template lwc:elseif={isManager}>
    <p>Manager Dashboard</p>
</template>
```

**Explanation:**
* **Syntax:** `lwc:elseif={property}`.
* **Evaluation Order:** The framework evaluates `isAdmin` first. If true, the Admin block renders, and `isManager` is skipped entirely. If `isAdmin` is false, it evaluates `isManager`.
* **Multiple Conditions:** You can chain as many `lwc:elseif` blocks as needed.

---

# 8. lwc:else

The `lwc:else` directive acts as a catch-all fallback when preceding `lwc:if` and `lwc:elseif` conditions are false. It does not take an expression.

### Example:
```html
<template lwc:if={isLoggedIn}>
    <p>Welcome, User!</p>
</template>
<template lwc:else>
    <p>Please log in to continue.</p>
</template>
```

**Explanation:**
* **Default Condition:** Renders only if all immediately preceding connected directives (`lwc:if` / `lwc:elseif`) evaluate to false.
* **Relationship:** It must be a direct sibling immediately following an `lwc:if` or `lwc:elseif`.

---

# 9. if:true / if:false vs lwc:if

| Feature | Legacy (`if:true` / `if:false`) | Modern (`lwc:if` / `elseif` / `else`) |
| :--- | :--- | :--- |
| **Current Recommendation** | ❌ Deprecated for new development | ✅ Standard for new development |
| **Syntax** | `<template if:true={condition}>` | `<template lwc:if={condition}>` |
| **Else/Fallback Support** | Requires a separate `if:false` block. | Native `lwc:else` support. |
| **Multiple Conditions** | Requires multiple independent `if:true` blocks. | Native `lwc:elseif` chaining. |
| **Performance** | Evaluates all `<template>` tags individually. | Short-circuits evaluation once a true condition is found. |
| **Readability** | Verbose, disconnected logic. | Clean, unified logical block. |

**Verdict:** Use `lwc:if`, `lwc:elseif`, and `lwc:else` exclusively for all new LWC development. Refactor legacy code when maintaining existing components.

---

# 10. Reactive Properties and Conditional Rendering

In LWC, properties are reactive. When a property bound to a template directive changes, the component automatically rerenders.

### Example:
```javascript
import { LightningElement } from 'lwc';

export default class WarrantyToggle extends LightningElement {
    isVisible = false; // Initial state

    handleClick() {
        this.isVisible = true; // User action changes state
    }
}
```

**The Flow:**
```text
Initial state (isVisible = false)
      ↓
User action (Clicks button)
      ↓
Property changes (isVisible = true)
      ↓
Component rerenders (LWC engine detects change)
      ↓
Conditional UI appears (DOM is updated)
```

---

# 11. Conditional Rendering with Getters

Because LWC templates do not allow inline expressions (e.g., `{amount > 10000}`), you must use JavaScript getter methods to compute derived conditions.

### Example:
```javascript
get showWarning() {
    return this.claimAmount > 10000 && this.status === 'Pending';
}
```

```html
<template lwc:if={showWarning}>
    <p>High-value claim requires Director approval.</p>
</template>
```

**Benefits of Getters:**
* **Derived Conditions:** Easily compute conditions based on multiple properties.
* **Keep Templates Simple:** HTML remains declarative and easy to read.
* **Separation of Concerns:** Business logic stays in JavaScript; UI logic stays in HTML.

---

# 12. Multiple Conditions

In a complex Automotive CRM, a user might have different access levels: Admin, Dealer, or Technician.

### Example:
```html
<template lwc:if={isAdmin}>
    <c-system-config></c-system-config>
</template>
<template lwc:elseif={isDealer}>
    <c-dealer-dashboard></c-dealer-dashboard>
</template>
<template lwc:elseif={isTechnician}>
    <c-work-order-queue></c-work-order-queue>
</template>
<template lwc:else>
    <c-customer-portal></c-customer-portal>
</template>
```

**Evaluation Order:** LWC evaluates from top to bottom. If `isDealer` is true, it renders the dealer dashboard and immediately stops evaluating `isTechnician` or the `else` block.

---

# 13. Nested Conditional Rendering

You can nest conditional blocks to handle hierarchical logic, but it should be done carefully to avoid "spaghetti markup."

### Example:
```html
<template lwc:if={isWarrantyClaim}>
    <!-- Outer condition met -->
    <template lwc:if={isApproved}>
        <!-- Inner condition met -->
        <p>Warranty Claim Approved</p>
    </template>
    <template lwc:else>
        <p>Warranty Claim Under Review</p>
    </template>
</template>
```

**When to use:** Good for checking a parent state (e.g., "Is this a Warranty Claim?") before checking child states ("Is it approved?").
**When to refactor:** If nesting goes more than two levels deep, it becomes hard to read. **Alternative:** Use a JavaScript getter that combines the logic: `get isApprovedWarrantyClaim() { return this.isWarrantyClaim && this.isApproved; }`

---

# 14. Conditional Rendering of Lightning Base Components

You can conditionally render standard Salesforce base components just like standard HTML elements.

### Example:
```html
<template lwc:if={isEditMode}>
    <lightning-input label="Vehicle VIN" value={vin}></lightning-input>
</template>
<template lwc:else>
    <lightning-formatted-text value={vin}></lightning-formatted-text>
</template>

<template lwc:if={isLoading}>
    <lightning-spinner alternative-text="Loading data" size="medium"></lightning-spinner>
</template>
```

---

# 15. Conditional Rendering of Custom Components

Conditionally rendering child LWCs has significant lifecycle implications. 

### Example:
```html
<template lwc:if={showVehicleDetails}>
    <c-vehicle-details record-id={selectedVehicleId}></c-vehicle-details>
</template>
```

**Component Lifecycle Impacts:**
* **Creation:** When `showVehicleDetails` becomes true, the `c-vehicle-details` component is instantiated. Its `constructor()` and `connectedCallback()` fire.
* **Removal:** When `showVehicleDetails` becomes false, the component is completely destroyed. Its `disconnectedCallback()` fires. 
* **Performance:** Frequently toggling heavy child components can degrade performance because the component is constantly built and torn down.

---

# 16. Loading State

Handling asynchronous operations (like Apex callouts to SAP) requires a loading state to provide good user experience.

### Example:
```html
<lightning-card title="SAP Invoice Sync">
    <template lwc:if={isLoading}>
        <lightning-spinner alternative-text="Loading" size="small"></lightning-spinner>
    </template>
    <template lwc:else>
        <p>Data synced successfully.</p>
    </template>
</lightning-card>
```

**Explanation:** By setting `isLoading = true` before an Apex call, and `isLoading = false` in the `finally` block, you provide immediate visual feedback.

---

# 17. Empty State

Empty states are crucial for UI/UX when a database query returns zero records.

### Example:
```html
<template lwc:if={hasClaims}>
    <c-claim-list claims={claimsData}></c-claim-list>
</template>
<template lwc:else>
    <div class="slds-illustration slds-illustration_small">
        <img src="/img/chatter/OpenRoad.svg" alt="No Claims" />
        <h3 class="slds-text-heading_small">No Warranty Claims Found</h3>
        <p class="slds-text-body_regular">This vehicle currently has no active claims.</p>
        <lightning-button label="Create Claim" onclick={handleCreate}></lightning-button>
    </div>
</template>
```

**Why it matters:** An empty screen looks like a bug. An empty state guides the user on what to do next.

---

# 18. Error State

When a server failure occurs, you must gracefully catch it and update the UI.

### Example:
```html
<template lwc:if={error}>
    <div class="slds-notify slds-notify_alert slds-alert_error" role="alert">
        <span class="slds-assistive-text">error</span>
        <h2>Error loading SAP Data: {errorMessage}</h2>
    </div>
</template>
```

**Explanation:** Store the error message in an `errorMessage` property. Set `error = true` to render this block. It prevents the user from staring at a broken UI.

---

# 19. Conditional Rendering with @wire

When using `@wire`, you must handle three distinct states: Loading, Data, and Error.

### Example:
```javascript
@wire(getWarrantyClaims, { vehicleId: '$recordId' })
wiredClaims({ error, data }) {
    if (data) {
        this.claims = data;
        this.error = undefined;
    } else if (error) {
        this.error = error;
        this.claims = undefined;
    }
}

get isLoading() {
    return !this.claims && !this.error;
}
```
### UI Flow:
```text
Loading (isLoading = true)
   ↓
Data Returns / Error Thrown
   ↓
Records Rendered / Error Message Displayed
```

---

# 20. Conditional Rendering with Apex

Imperative Apex allows precise control over UI state updates.

### Example:
```javascript
fetchInvoices() {
    this.isLoading = true;
    this.error = null;
    
    getInvoicesFromSAP()
        .then(result => {
            this.invoices = result;
        })
        .catch(error => {
            this.error = 'Failed to fetch invoices.';
        })
        .finally(() => {
            this.isLoading = false;
        });
}
```

The HTML will reactively toggle between the spinner, the data table, and the error alert based on these property changes.

---

# 21. Conditional Rendering with User Permissions

You can import permissions to conditionally render UI.

```javascript
import hasApproveClaimPermission from '@salesforce/customPermission/Approve_Warranty_Claims';

export default class ClaimRecord extends LightningElement {
    get canApprove() {
        return hasApproveClaimPermission;
    }
}
```

> 🚨 **SECURITY WARNING:** Hiding a UI element via conditional rendering is **NOT** a security mechanism. 
> 
> * **UI Visibility:** Prevents the user from accidentally clicking a button.
> * **Server-side Security:** Prevents malicious users from bypassing the UI and calling the Apex method directly via the browser console or API. 
> 
> You **must** enforce `with sharing`, Object/Field-Level Security, and Custom Permission checks inside your Apex controllers.

---

# 22. Conditional Rendering in Automotive CRM

Real enterprise examples contextualized:

* **Approve Button:** `lwc:if={canApprove}` - Only renders for Service Managers.
* **Claim Details:** `lwc:if={hasActiveClaim}` - Hides empty spaces if a vehicle has no claims.
* **Shipment Details:** `lwc:if={shipmentDispatched}` - Only visible after parts leave the warehouse.
* **SAP Error Message:** `lwc:if={sapSyncFailed}` - Alerts the user to retry the integration.
* **Spare Parts Section:** `lwc:if={requiresSpareParts}` - Derived from a getter checking if the Work Order type is 'Replacement'.

---

# 23. Conditional Rendering with Lists

When iterating over lists, you often need to render specific data conditionally.
*Note: You cannot put `lwc:if` and `for:each` on the exact same HTML element.*

### Example:
```html
<template for:each={workOrders} for:item="wo">
    <div key={wo.Id}>
        <p>{wo.Name}</p>
        <!-- Conditional rendering INSIDE the list -->
        <template lwc:if={wo.isUrgent}>
            <lightning-badge label="URGENT" class="slds-theme_error"></lightning-badge>
        </template>
    </div>
</template>
```

**Difference:** 
* Conditionally rendering the *entire list* happens *outside* the `for:each` (Empty State).
* Conditionally rendering *content inside an item* happens *inside* the `for:each` (Row-level conditions).

---

# 24. Conditional Rendering and DOM

When a condition is `false`, LWC does not just make the element invisible; it **completely removes it from the DOM**.

* **Creation:** Memory is allocated, DOM elements are built, event listeners are attached.
* **Removal:** The elements are destroyed, memory is garbage-collected, and child component states are lost.

**Implication:** If a user types data into a child component, and you toggle that component off and back on using `lwc:if`, the typed data is lost unless you explicitly saved it to the parent state.

---

# 25. Conditional Rendering vs CSS Hiding

| Feature | Conditional Rendering (`lwc:if`) | CSS Hiding (`class="slds-hide"`) |
| :--- | :--- | :--- |
| **DOM Presence** | Element is removed from DOM. | Element remains in DOM, visually hidden. |
| **Component Lifecycle** | Disconnected/Connected callbacks fire. | No lifecycle changes occur. |
| **Performance (Initial)** | Faster (less DOM to render). | Slower (renders hidden elements). |
| **Performance (Toggling)** | Slower (rebuilds DOM on toggle). | Faster (just changes CSS classes). |
| **State Retention** | State is destroyed when hidden. | State is preserved when hidden. |
| **Best Use Case** | Toggling large components, security context (visually). | Toggling simple elements rapidly (e.g., accordions). |

---

# 26. Performance Considerations

* **Expensive Getters:** Getters used in conditional templates evaluate frequently. Avoid complex loops or calculations inside getters.
* **Frequent Toggling:** If a complex child component needs to be toggled rapidly, consider using CSS classes (`slds-hide`) instead of `lwc:if` to avoid destroying and recreating the component lifecycle constantly.
* **Large Component Trees:** For massive DOM trees, `lwc:if` is preferred initially because it saves browser memory by not rendering off-screen sections until needed (e.g., lazy loading tabs).

---

# 27. Common Mistakes

### Mistake 1: Using `if:true` in new development
* **Problem:** Clutters code with disjointed logic.
* **Cause:** Following outdated documentation.
* **Solution:** Always use `lwc:if`.

### Mistake 2: Assuming hidden UI provides security
* **Problem:** Malicious users invoke Apex directly.
* **Cause:** Believing `lwc:if={isAdmin}` secures data.
* **Solution:** Validate permissions in Apex.

### Mistake 3: Complex logic in templates
* **Problem:** Attempting to write `<template lwc:if={status === 'Open'}>`.
* **Cause:** Confusing LWC with React/Vue.
* **Solution:** Create a getter: `get isOpen() { return this.status === 'Open'; }`.

### Mistake 4: Losing state on toggle
* **Problem:** User fills a form, minimizes the section, opens it, and data is gone.
* **Cause:** `lwc:if` destroyed the form component.
* **Solution:** Cache input data in the parent JS object before the child unmounts, or use CSS hiding.

---

# 28. Debugging Conditional Rendering

**Debugging Checklist:**
1. **UI not appearing:** Check the casing of the property in the template. `isvisible` vs `isVisible`.
2. **Getter not firing:** Ensure the properties used inside the getter are tracked properly (using `@track` if mutating object properties/arrays).
3. **Unexpected toggling:** Place `console.log()` inside the getter to see how many times it's evaluated and what it returns.
4. **Data not loaded:** If rendering based on `@wire`, ensure the record ID is properly passed and accessible before the wire executes.

---

# 29. Best Practices Checklist

* ✅ **Use `lwc:if` for new development:** Completely abandon `if:true` and `if:false`.
* ✅ **Use `lwc:elseif` for multiple conditions:** Keeps logic grouped and readable.
* ✅ **Use `lwc:else` for fallback UI:** Ensure users never see a blank screen.
* ✅ **Keep conditions simple:** Only reference a single boolean property or getter.
* ✅ **Use getters for derived conditions:** Shift complex logic out of the template.
* ✅ **Avoid deeply nested conditions:** Refactor to single compound getters.
* ✅ **Handle loading, empty, and error states:** The holy trinity of good UI design.
* ✅ **Do not rely on UI hiding for security:** Validate in Apex.
* ✅ **Use CSS hiding for rapid toggling:** Save performance on heavy child components.

---

# 30. Complete End-to-End Example

**Scenario:** A Warranty Claim Details component in an Automotive CRM. 

### `warrantyClaim.html`
```html
<template>
    <lightning-card title="Warranty Claim Overview" icon-name="custom:custom31">
        
        <!-- LOADING STATE -->
        <template lwc:if={isLoading}>
            <lightning-spinner alternative-text="Loading Claim Data"></lightning-spinner>
        </template>
        
        <!-- ERROR STATE -->
        <template lwc:elseif={error}>
            <div class="slds-text-color_error slds-p-around_medium">
                Failed to load claim: {error}
            </div>
        </template>
        
        <!-- EMPTY STATE -->
        <template lwc:elseif={isEmpty}>
            <div class="slds-p-around_medium slds-text-align_center">
                <p>No active claim associated with this vehicle.</p>
                <lightning-button label="Create Claim" class="slds-m-top_small"></lightning-button>
            </div>
        </template>
        
        <!-- DATA STATE -->
        <template lwc:else>
            <div class="slds-p-around_medium">
                <p><strong>Claim Number:</strong> {claimData.ClaimNumber}</p>
                <p><strong>Status:</strong> {claimData.Status}</p>
                
                <!-- CONDITIONAL BUTTONS BASED ON STATUS -->
                <div class="slds-m-top_medium">
                    <template lwc:if={isPending}>
                        <lightning-button label="Approve Claim" variant="brand"></lightning-button>
                        <lightning-button label="Reject Claim" class="slds-m-left_small" variant="destructive"></lightning-button>
                    </template>
                    <template lwc:elseif={isApproved}>
                        <lightning-badge label="APPROVED" class="slds-theme_success"></lightning-badge>
                    </template>
                    <template lwc:else>
                        <lightning-badge label="REJECTED" class="slds-theme_error"></lightning-badge>
                    </template>
                </div>
            </div>
        </template>

    </lightning-card>
</template>
```

### `warrantyClaim.js`
```javascript
import { LightningElement, api, wire } from 'lwc';
import getClaimData from '@salesforce/apex/WarrantyController.getClaimData';

export default class WarrantyClaim extends LightningElement {
    @api recordId; // Vehicle Id
    
    claimData;
    error;
    isLoading = true;

    @wire(getClaimData, { vehicleId: '$recordId' })
    wiredClaim({ error, data }) {
        if (data) {
            this.claimData = data;
            this.error = undefined;
        } else if (error) {
            this.error = error.body.message;
            this.claimData = undefined;
        }
        this.isLoading = false;
    }

    // Derived Getters for Conditional Rendering
    get isEmpty() {
        return !this.claimData && !this.error;
    }

    get isPending() {
        return this.claimData?.Status === 'Pending';
    }

    get isApproved() {
        return this.claimData?.Status === 'Approved';
    }
}
```

---

# 31. Interview Questions & Answers

### Beginner Questions
**Q: What is conditional rendering in LWC?**
A: It is the ability to dynamically show or hide HTML elements in the DOM based on JavaScript properties using template directives.

**Q: What is `if:true` and why is it legacy?**
A: `if:true` is an older directive that evaluates a single condition. It is legacy because Salesforce introduced `lwc:if`, which provides a more unified, readable, and performant way to handle logic, including native `elseif` and `else` support.

### Intermediate Questions
**Q: What is the difference between `lwc:if` and `if:true`?**
A: `lwc:if` is the modern compiler directive that supports chaining with `lwc:elseif` and `lwc:else`, allowing short-circuit evaluation. `if:true` is isolated, requiring manual `if:false` blocks for fallback logic, which is less performant and harder to read.

**Q: How do you handle complex conditions like `(Amount > 100 && Status == 'Open')` in the HTML?**
A: You cannot write inline expressions in LWC templates. You must create a getter in JavaScript (e.g., `get isHighValueOpen()`) that returns the evaluated boolean, and bind that getter to the `lwc:if` directive.

### Advanced Questions
**Q: What happens to a child component when its parent `lwc:if` condition turns from true to false?**
A: The child component is completely unmounted and removed from the DOM. Its `disconnectedCallback()` fires, and all localized state within that component is destroyed and garbage-collected.

**Q: What is the difference between conditional rendering and hiding elements using CSS?**
A: Conditional rendering adds/removes elements from the DOM, saving memory initially but costing performance on rapid toggles. CSS hiding (`display: none`) keeps elements in the DOM, maintaining component state and allowing for faster toggling at the cost of heavier initial DOM weight.

### Architect-Level Questions
**Q: A developer has hidden a "Delete Invoice" button using `lwc:if={isAdmin}`. What is the security flaw in this design?**
A: Hiding UI is not security; it is only a UX mechanism. A malicious user could still inspect the page, modify the DOM, or invoke the Apex controller method directly. An architect must ensure that the underlying Apex method checks Custom Permissions and `Schema.sObjectType.Invoice__c.isDeletable()` before executing the DML operation.

---

# 32. Revision Summary

* **Conditional Rendering:** Controls what is dynamically inserted or removed from the DOM.
* **Legacy Syntax:** `if:true` and `if:false`. Avoid using these in new development.
* **Modern Syntax:** `lwc:if`, `lwc:elseif`, `lwc:else`. Use these exclusively for clean, performant code.
* **Reactive Properties:** Changes to tracked variables or component state automatically trigger conditional rerenders.
* **Getters:** Essential for handling multi-variable conditions since HTML expressions are prohibited.
* **Nested vs Chained:** Prefer `lwc:elseif` over deeply nested `lwc:if` blocks for better readability.
* **Crucial States:** Always design components to handle Loading (spinner), Empty (illustrations), and Error (alerts/toasts) states.
* **DOM Behavior:** `lwc:if` destroys elements. Use CSS hiding if you need to retain state or toggle rapidly.
* **Security:** UI visibility is not authorization. Always enforce rules on the server via Apex.