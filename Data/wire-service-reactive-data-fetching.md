# Salesforce LWC Wire Service & Reactive Data Fetching 

## 1. Introduction

### What is the Wire Service?
The Wire Service is the primary way Lightning Web Components (LWC) read data from Salesforce. It is a reactive, declarative data-fetching mechanism built on the Lightning Data Service (LDS) and Apex. It automatically provisions a continuous stream of data to a component without requiring manual imperative calls.

### Why Salesforce provides the Wire Service
Salesforce provides the Wire Service to standardize and optimize client-to-server communication. It handles caching, server round-trips, and component lifecycle events internally, ensuring high performance and a unified data cache across all components on a page.

### How LWC retrieves Salesforce data
LWC retrieves data primarily via:
1. **Wire Service (Declarative):** Using the `@wire` decorator to read data (LDS or Apex).
2. **Imperative Apex (Manual):** Explicitly calling an Apex method via JavaScript promises (usually for DML or complex one-off operations).

### Why reactive data fetching is important
Reactive data fetching means that the UI automatically updates when the underlying parameters or data change. If a user changes a **Warranty Claim** status filter, the wire service detects the change, fetches the new claims, and re-renders the UI—all without writing manual refresh logic. 

### Difference between automatic and manual data retrieval
*   **Automatic (Wire):** Component loads -> Parameters are ready -> Wire fetches data -> UI updates. If parameters change, the process repeats automatically.
*   **Manual (Imperative):** Component loads -> User clicks "Search" -> Developer invokes a JavaScript function -> Promise resolves -> Developer manually updates state -> UI updates.

**Automotive Context Examples:**
*   **Warranty Claims:** Automatically loading claim details when a user opens the record page.
*   **Spare Parts:** Reactively fetching inventory levels as a user types into a search box.
*   **Dealers:** Fetching related dealer info for a given Vehicle record seamlessly.

---

## 2. What is the Wire Service?

### Architecture & Reactive Data Provisioning
The Wire Service acts as an intelligent intermediary between your component and Salesforce. It sits on top of Lightning Data Service (LDS). When you use `@wire`, you are subscribing to a data source. The wire service provisions data to the component asynchronously. If the data is already in the LDS cache, it is delivered instantly. Otherwise, the wire service requests it from the server.

### Data Sources
*   **Lightning Data Service (LDS) Wire Adapters:** Pre-built adapters (e.g., `getRecord`, `getPicklistValues`) that fetch data without Apex.
*   **Apex Methods:** Custom SOQL/logic exposed to LWC via `@AuraEnabled(cacheable=true)`.

### Component Lifecycle Integration
The wire service runs right after the component is created but before it is rendered. It continuously monitors reactive parameters during the component's lifecycle.

### Flow Diagram

```text
[ Reactive Parameter Changes (e.g., searchKey) ]
                         ↓
[ @wire detects change & triggers execution ]
                         ↓
[ Wire Adapter / Cacheable Apex Method ]
                         ↓
  (Checks Client Cache) ──→ (If Missing: Queries Salesforce Server)
                         ↓
[ Returns Data to LWC (data / error objects) ]
                         ↓
[ Reactive UI Update (Component Re-renders) ]
```

---

## 3. Why Use Wire Service?

The Wire Service should be your default choice for reading data in LWC.

### Advantages
*   **Reactive Data:** Automatically provisions new data when parameters change.
*   **Automatic Execution:** No need to write manual initialization logic in `connectedCallback`.
*   **Simplified Data Fetching:** Reduces boilerplate code (no `.then()` and `.catch()` blocks for basic reads).
*   **Salesforce Caching:** Shares a client-side cache with other components using LDS, massively reducing server round-trips.
*   **Reduced Manual State Management:** The framework handles the loading and error states natively through the provisioned object.

### Wire Service vs Imperative Apex
Use **Wire Service** for reading data (Read-only operations). It provides caching and reactivity.
Use **Imperative Apex** for data manipulation (Insert, Update, Delete) or when you need strict control over when the call executes (e.g., clicking a button to fetch an external API).

---

## 4. @wire Decorator

The `@wire` decorator tells LWC to provision data to a property or function.

### Example
```javascript
import { LightningElement, wire } from 'lwc';
import getAccounts from '@salesforce/apex/AccountController.getAccounts';

export default class WireExample extends LightningElement {
    @wire(getAccounts) accounts;
}
```

### Explanation
*   `import { wire } from 'lwc';`: Imports the wire decorator from the core LWC module.
*   `import getAccounts...`: Imports the Apex method.
*   `@wire(getAccounts)`: Decorates the property below it. Instructs the framework to call `getAccounts`.
*   `accounts;`: The property that receives the provisioned object, which contains `data` and `error`.

### Correct Syntax & Common Mistakes
*   **Correct:** `@wire(methodName, { param: '$value' })`
*   **Mistake:** Forgetting to import `wire` from `lwc`.
*   **Mistake:** Putting a semicolon after the `@wire` statement (e.g., `@wire(getAccounts); accounts;` - this will fail).

---

## 5. Wired Property

Wiring to a property is the simplest way to fetch data. The framework provisions an object to the property containing `data` and `error`.

### Example
```html
<!-- claimList.html -->
<template>
    <template lwc:if={claims.data}>
        <template for:each={claims.data} for:item="claim">
            <p key={claim.Id}>{claim.Name}</p>
        </template>
    </template>
    <template lwc:elseif={claims.error}>
        <p>Error loading claims!</p>
    </template>
</template>
```

```javascript
// claimList.js
import { LightningElement, wire } from 'lwc';
import getClaims from '@salesforce/apex/ClaimController.getClaims';

export default class ClaimList extends LightningElement {
    @wire(getClaims) 
    claims; 
}
```

### Accessing Data and Errors
When wired to a property (`claims`), the framework sets:
*   `this.claims.data`: Contains the returned records if successful.
*   `this.claims.error`: Contains the error object if the call fails.
*   Initially, both `data` and `error` are `undefined`.

---

## 6. Wired Function

Wiring to a function (or method) allows you to perform custom logic, transform data, or explicitly route data to specific properties when the data is provisioned.

### Example
```javascript
import { LightningElement, wire } from 'lwc';
import getClaims from '@salesforce/apex/ClaimController.getClaims';

export default class WiredFunctionExample extends LightningElement {
    claimsData;
    errorMsg;

    @wire(getClaims)
    wiredClaims({ data, error }) {
        if (data) {
            // Transform or store data
            this.claimsData = data.map(claim => {
                return { ...claim, mappedStatus: claim.Status + ' - Processed' };
            });
            this.errorMsg = undefined;
        } else if (error) {
            this.errorMsg = error.body.message;
            this.claimsData = undefined;
        }
    }
}
```

### Explanation
*   **Function Parameter:** The function `wiredClaims` receives an object containing `{ data, error }`. We destructure it immediately.
*   **Success Handling (`if (data)`):** Executes when data is successfully retrieved. We assign it to a custom variable (`this.claimsData`).
*   **Error Handling (`else if (error)`):** Executes on failure.
*   **Resetting state:** Notice we set `error = undefined` on success, and `data = undefined` on error. This is crucial because a subsequent wire execution could fail after a previous success, and you must clear the old state.

---

## 7. Wired Property vs Wired Function

| Feature | Wired Property | Wired Function |
| :--- | :--- | :--- |
| **Syntax** | `@wire(method) propertyName;` | `@wire(method) functionName({data, error}) {...}` |
| **Data Handling** | Accessed via `this.property.data` | Accessed via the `data` argument. |
| **Error Handling** | Accessed via `this.property.error` | Accessed via the `error` argument. |
| **Transformation**| Cannot transform data directly (requires getter). | Can transform/mutate data easily inside the function. |
| **Custom Logic** | None. Purely declarative assignment. | High. Can trigger other actions, log, or calculate state. |
| **Simplicity** | High (1 line of JS). | Moderate (requires `if/else` boilerplate). |
| **Recommended Use**| Simply displaying data in the UI without changes. | Data needs mapping, filtering, or triggers side-effects. |

---

## 8. Wire Service with Apex

To use Apex with `@wire`, the method must be imported, and it must be annotated properly in Salesforce.

### Apex Controller
```apex
public with sharing class DealerController {
    @AuraEnabled(cacheable=true)
    public static List<Account> getActiveDealers() {
        return [
            SELECT Id, Name, Dealer_Region__c
            FROM Account 
            WHERE RecordType.DeveloperName = 'Dealer' 
            AND IsActive = true
            LIMIT 50
        ];
    }
}
```

### LWC JavaScript
```javascript
import { LightningElement, wire } from 'lwc';
import getActiveDealers from '@salesforce/apex/DealerController.getActiveDealers';

export default class DealerList extends LightningElement {
    @wire(getActiveDealers)
    dealers;
}
```

### Explanation
1.  **Apex Import:** `import getActiveDealers...` creates a reference to the Apex method.
2.  **Execution:** `@wire(getActiveDealers)` executes the SOQL query via the LDS network layer.
3.  **Result:** Result is bound to the `dealers` property (`dealers.data` and `dealers.error`).

---

## 9. @AuraEnabled(cacheable=true)

For an Apex method to be used with `@wire`, it **must** be annotated with `@AuraEnabled(cacheable=true)`.

### Cover Points:
*   **Meaning:** Tells the Lightning infrastructure to cache the method's results on the client side.
*   **Read-only requirement:** Cacheable methods cannot perform DML (Insert, Update, Delete, Undelete). If you attempt a DML, a fatal exception occurs.
*   **Client-side caching:** Subsequent requests for the same data (with the same parameters) resolve instantly from the browser cache without hitting the server.
*   **Performance:** Drastically reduces server round-trips and governor limit consumption.
*   **Cacheable vs Non-cacheable:** Cacheable is for reads (`@wire`), non-cacheable is for writes (Imperative Apex).

---

## 10. Reactive Data Fetching

"Reactive" in LWC means the system reacts to changes automatically.

### Example
```javascript
import { LightningElement, wire } from 'lwc';
import getClaims from '@salesforce/apex/ClaimController.getClaims';

export default class ClaimSearch extends LightningElement {
    searchKey = '';

    @wire(getClaims, { searchTerm: '$searchKey' })
    claims;

    handleInput(event) {
        this.searchKey = event.target.value;
    }
}
```

### Flow Diagram
```text
User types in lightning-input
       ↓
handleInput updates this.searchKey
       ↓
'$searchKey' tells Wire to watch searchKey
       ↓
Wire detects mutation
       ↓
Apex executes again with new searchTerm
       ↓
this.claims.data is updated
       ↓
UI re-renders automatically
```

---

## 11. Reactive Parameters

Parameters passed to the wire adapter can be reactive. By prefixing a property string with `$`, you tell the wire service to watch that property.

### Example
```javascript
@wire(getClaims, {
    status: '$selectedStatus',
    dealerId: '$recordId'
})
claims;
```

### Explanation
*   `getClaims`: The data source.
*   `status: '$selectedStatus'`: The Apex parameter `status` receives the value of `this.selectedStatus`. If `this.selectedStatus` changes, the wire re-runs.
*   `dealerId: '$recordId'`: The Apex parameter `dealerId` receives `this.recordId`.
*   **Multiple parameters:** If *any* reactive parameter changes, the wire adapter executes again.

---

## 12. $ Syntax

The `$` prefix is exclusively used in the `@wire` configuration object to differentiate between a literal string and a dynamic component property.

### Comparison
*   `searchKey: '$searchKey'`: **Reactive Value.** Passes the value of `this.searchKey`. Re-evaluates if `this.searchKey` changes.
*   `searchKey: 'searchKey'`: **Static Value.** Passes the literal string `"searchKey"` to Apex. It will never trigger a re-execution.

*Note: You do not need to use `$` if you intentionally want to pass a hardcoded static string (e.g., `status: 'Open'`).*

---

## 13. Wire Service with recordId

When placing a component on a Lightning Record Page, LWC can automatically receive the ID of the current record. You can wire this directly.

### Example
```javascript
import { LightningElement, api, wire } from 'lwc';
import getClaimDetails from '@salesforce/apex/ClaimController.getClaimDetails';

export default class ClaimSummary extends LightningElement {
    @api recordId; // Automatically populated by the record page

    @wire(getClaimDetails, { claimId: '$recordId' })
    claim;
}
```

### Explanation
*   `@api recordId`: Exposes the property. The Lightning framework detects this and injects the 18-character Salesforce ID.
*   `'$recordId'`: Makes the wire service wait until `recordId` is populated (initially it is undefined). Once populated, the wire fetches the data.

---

## 14. Wire Adapters

Wire adapters are pre-packaged modules that define how data is fetched. 

*   **LDS Wire Adapters:** Provided by Salesforce under the `lightning/ui*Api` modules. They do not require Apex. They respect CRUD/FLS natively and utilize the shared LDS cache. Examples: `getRecord`, `getFieldValue`, `getObjectInfo`, `getPicklistValues`.
*   **Apex Wire Methods:** Custom methods written by you, imported from `@salesforce/apex/...`. They execute SOQL and custom logic.

**Difference:** LDS adapters are standardized API endpoints for standard Salesforce objects/operations. Apex wire methods are highly customizable and can aggregate data, perform complex queries, or integrate with external systems.

---

## 15. Lightning Data Service Wire Adapters

LDS allows you to read Salesforce metadata and record data without a single line of Apex.

*   `getRecord`: Retrieves a record's fields.
*   `getObjectInfo`: Retrieves metadata (Record Type IDs, field labels) for an object.
*   `getPicklistValues`: Retrieves valid picklist values for a specific record type.
*   `getRelatedListRecords`: Retrieves a related list (e.g., all Claim Lines for a Claim).

LDS is highly performant because if another component on the page already fetched the Account record, your component gets it from the browser cache instantly.

---

## 16. getRecord

`getRecord` is the most common LDS wire adapter for fetching record data.

### Example
```javascript
import { LightningElement, api, wire } from 'lwc';
import { getRecord } from 'lightning/uiRecordApi';
import CLAIM_NAME from '@salesforce/schema/Warranty_Claim__c.Name';
import CLAIM_STATUS from '@salesforce/schema/Warranty_Claim__c.Status__c';
import DEALER_NAME from '@salesforce/schema/Warranty_Claim__c.Dealer__r.Name';

export default class ClaimHeader extends LightningElement {
    @api recordId;

    @wire(getRecord, { 
        recordId: '$recordId', 
        fields: [CLAIM_NAME, CLAIM_STATUS, DEALER_NAME] 
    })
    claimRecord;
}
```

### Explanation
*   `import { getRecord }...`: Imports the adapter.
*   `import FIELD_NAME...`: Schema imports ensure referential integrity. If a field is deleted in Salesforce, deployment fails, protecting your code.
*   `fields: [...]`: Requests specific custom, standard, and cross-object relationship fields.

---

## 17. getFieldValue

`getFieldValue` is a helper function used to extract field values from the complex JSON object returned by `getRecord`.

### Example
```javascript
import { getFieldValue } from 'lightning/uiRecordApi';

// Inside the component...
get claimStatus() {
    return getFieldValue(this.claimRecord.data, CLAIM_STATUS);
}
```

### Explanation
*   **Why it is useful:** `getRecord` returns a deeply nested JSON structure (`data.fields.Status__c.value`). `getFieldValue` provides a clean, safe way to extract it.
*   **Relationship fields:** It works seamlessly with spanned fields (e.g., `Dealer__r.Name`).
*   **Avoiding nested access:** Prevents `TypeError: Cannot read property 'value' of undefined` if data hasn't loaded yet.

---

## 18. Wire Service with Custom Objects

### Automotive CRM Scenario: Warranty Claim
Fetching a specific warranty claim using Apex to pull custom fields.

### Apex
```apex
public with sharing class WarrantyClaimController {
    @AuraEnabled(cacheable=true)
    public static Warranty_Claim__c getClaimSummary(Id claimId) {
        return [
            SELECT Id, Claim_Number__c, Status__c, Claim_Amount__c, 
                   Dealer__r.Name, Vehicle__r.VIN__c 
            FROM Warranty_Claim__c 
            WHERE Id = :claimId 
            LIMIT 1
        ];
    }
}
```

### LWC JavaScript
```javascript
import { LightningElement, api, wire } from 'lwc';
import getClaimSummary from '@salesforce/apex/WarrantyClaimController.getClaimSummary';

export default class ClaimSummaryCard extends LightningElement {
    @api recordId;

    @wire(getClaimSummary, { claimId: '$recordId' })
    claim;

    get claimNumber() {
        return this.claim.data?.Claim_Number__c;
    }
}
```

### LWC HTML
```html
<template>
    <lightning-card title="Claim Summary" icon-name="custom:custom93">
        <template lwc:if={claim.data}>
            <div class="slds-m-around_medium">
                <p><strong>Claim #:</strong> {claim.data.Claim_Number__c}</p>
                <p><strong>Status:</strong> {claim.data.Status__c}</p>
                <p><strong>Amount:</strong> ${claim.data.Claim_Amount__c}</p>
                <p><strong>Dealer:</strong> {claim.data.Dealer__r.Name}</p>
                <p><strong>Vehicle VIN:</strong> {claim.data.Vehicle__r.VIN__c}</p>
            </div>
        </template>
    </lightning-card>
</template>
```

---

## 19. Wire Service with Relationships

Apex wire methods can return complex relational data, traversing standard and custom relationships.

### Examples
*   **Claim → Dealer:** `Dealer__r.Name` (Parent traversal)
*   **Work Order → Account:** `Account.Name` (Standard parent)
*   **Dealer → Claims:** `(SELECT Id FROM Warranty_Claims__r)` (Child traversal)

### Nested Values & Null Handling
When traversing in JS, safely access nested data using Optional Chaining (`?.`).
```javascript
get dealerName() {
    // Safely handles null references if Dealer__r is null
    return this.claim.data?.Dealer__r?.Name || 'No Dealer Assigned';
}
```

---

## 20. Wire Service with Search

### Scenario: Search Warranty Claims by Claim Number

```html
<template>
    <lightning-input type="search" label="Search Claims" onchange={handleSearch}></lightning-input>
    
    <template lwc:if={claims.data}>
        <ul class="slds-m-top_medium">
            <template for:each={claims.data} for:item="claim">
                <li key={claim.Id}>{claim.Claim_Number__c} - {claim.Status__c}</li>
            </template>
        </ul>
    </template>
</template>
```

```javascript
import { LightningElement, wire } from 'lwc';
import searchClaims from '@salesforce/apex/ClaimController.searchClaims';

export default class ClaimSearch extends LightningElement {
    searchKey = '';

    @wire(searchClaims, { searchTerm: '$searchKey' })
    claims;

    handleSearch(event) {
        // Triggered on every keystroke
        this.searchKey = event.target.value;
    }
}
```

### Explanation
*   `lightning-input` fires the `onchange` event.
*   `handleSearch` updates `this.searchKey`.
*   Because `$searchKey` is reactive, the Wire Service automatically calls `searchClaims` with the new value.
*   Results update natively in the template. *(Note: Debouncing should be used in production to avoid excessive server calls).*

---

## 21. Wire Service with Filters

Filters function similarly to search, often tied to picklists.

### Example
```javascript
import { LightningElement, wire } from 'lwc';
import getFilteredClaims from '@salesforce/apex/ClaimController.getFilteredClaims';

export default class ClaimFilter extends LightningElement {
    selectedStatus = 'Pending'; // Default state

    @wire(getFilteredClaims, { status: '$selectedStatus' })
    claims;

    handleStatusChange(event) {
        this.selectedStatus = event.detail.value;
    }
}
```

### Explanation
When the user selects a new status from a dropdown (calling `handleStatusChange`), `selectedStatus` updates. The wire detects this, hits the server (or cache), and pulls only the records matching the new status.

---

## 22. Wire Service with Multiple Parameters

You can pass multiple reactive parameters. The wire will re-execute if **any** of them change.

### Example
```javascript
@wire(getClaims, {
    dealerId: '$dealerId',
    status: '$status',
    searchKey: '$searchKey'
})
claims;
```

### Explanation
*   **Multiple Dependencies:** The query depends on the dealer, status, and search string.
*   **Execution:** Whenever `dealerId`, `status`, or `searchKey` updates, a new request is generated.
*   **Undefined Parameters:** If an Apex parameter is missing from the wire config, it is passed as `null`.
*   **Performance Considerations:** Changing two parameters simultaneously (e.g., in a single JS method) batches into a single wire execution cycle, optimizing performance.

---

## 23. Conditional Data Fetching

To prevent a wire from executing before prerequisites are met, ensure its reactive parameters resolve to `undefined`. A wire adapter will **not** invoke a server call if any dynamic parameter in its config is strictly `undefined` (Note: `null` or `''` *will* trigger execution).

### Example
```javascript
export default class VehicleDetails extends LightningElement {
    @api recordId;
    vehicleId; // Initially undefined

    // Only executes when vehicleId has a truthy/defined value
    @wire(getVehicleData, { vehicleId: '$vehicleId' })
    vehicleInfo;

    // Suppose we get vehicleId from another wire call first
    @wire(getRecord, { recordId: '$recordId', fields: ['Claim.Vehicle__c'] })
    wiredClaim({ data }) {
        if (data) {
            // This assignment triggers the getVehicleData wire
            this.vehicleId = data.fields.Vehicle__c.value; 
        }
    }
}
```

---

## 24. Handling Wire Data

Always process `data` and `error` gracefully using a wired function if transformation is needed.

### Example
```javascript
@wire(getClaims)
wiredClaims({ data, error }) {
    if (data) {
        this.claims = data;
        this.error = undefined; // Clear previous errors
    } else if (error) {
        this.error = error;
        this.claims = undefined; // Clear previous data
    }
}
```

### Explanation
*   The `if(data)` block executes when records are fetched successfully. We bind data to a local property and nullify `error`.
*   The `else if(error)` block executes on failure (SOQL error, permissions, etc.). We bind the error and nullify `claims`.

---

## 25. Error Handling

Errors from the wire service come in a specific format (`error.body.message` or array of errors).

### Sources of Errors
*   **Apex:** Unhandled exceptions or `AuraHandledException`.
*   **LDS/Wire Adapters:** Invalid IDs, insufficient access, field missing.
*   **Network:** Offline issues.

### Reusable Error Helper Pattern
```javascript
export function getErrorMessage(error) {
    if (!error) return 'Unknown error';
    if (Array.isArray(error.body)) {
        return error.body.map(e => e.message).join(', ');
    }
    if (error.body && error.body.message) {
        return error.body.message;
    }
    return error.message || 'Unknown error';
}
```
*Use this helper in your `else if (error)` block to safely extract human-readable text for the UI.*

---

## 26. refreshApex

`refreshApex` is a function imported from `@salesforce/apex` used to force the wire service to bypass the cache and query the server for fresh data.

### Why it is required
Because `@AuraEnabled(cacheable=true)` caches data aggressively, a user might update a record (via imperative Apex), but the wire cache won't know about it. `refreshApex` solves this.

### Example
```javascript
import { LightningElement, wire } from 'lwc';
import { refreshApex } from '@salesforce/apex';
import getClaims from '@salesforce/apex/ClaimController.getClaims';

export default class ClaimList extends LightningElement {
    wiredResult; // Store the complete provisioned object
    claims;

    @wire(getClaims)
    wiredClaims(result) {
        this.wiredResult = result; // MUST store the 'result' object
        if (result.data) {
            this.claims = result.data;
        }
    }

    handleManualRefresh() {
        // Forces a server trip, bypassing cache
        return refreshApex(this.wiredResult); 
    }
}
```

### Explanation
*   **Stored wired result:** You **must** pass the entire provisioned object (the `result` argument containing data/error) to `refreshApex`, NOT just the `data`.
*   **Common mistake:** Calling `refreshApex(this.claims)` will fail.

---

## 27. refreshApex After DML

### Complete Example Flow
1. Fetch claims (Wire).
2. User approves a claim (Imperative Apex).
3. Data is updated in Salesforce.
4. Call `refreshApex` to update the list.

```javascript
import approveClaim from '@salesforce/apex/ClaimController.approveClaim';

export default class ClaimList extends LightningElement {
    wiredClaimResult;
    claims;

    @wire(getClaims)
    wiredClaims(result) {
        this.wiredClaimResult = result;
        if(result.data) this.claims = result.data;
    }

    handleApprove(event) {
        const claimId = event.target.dataset.id;
        
        // Imperative Apex for DML
        approveClaim({ claimId: claimId })
            .then(() => {
                // DML Success -> Force cache refresh
                return refreshApex(this.wiredClaimResult);
            })
            .catch(error => {
                console.error(error);
            });
    }
}
```

---

## 28. Wire Service and Imperative Apex Together

The most common enterprise pattern in LWC is combining the two:

*   **Wire for Reading:** Use `@wire` to display lists, dashboards, and read-only forms.
*   **Imperative Apex for Actions:** Use standard Promises (`.then().catch()`) on button clicks to perform DML (Create, Update, Delete).
*   **The Bridge:** `refreshApex()` connects them, ensuring the read-only UI reflects the imperative action's results.

---

## 29. Wire Service and Lightning Data Service Together

If using LDS wire adapters (`getRecord`), you do **not** use `refreshApex` to update the cache. LDS has a unified cache.

*   `getRecord`: Reads data declaratively.
*   `updateRecord` (LDS Imperative): Updates data. Once the promise resolves, LDS *automatically* updates the local cache, and any `@wire(getRecord)` components instantly react and re-render. No `refreshApex` needed!
*   **Rule of thumb:** Use `refreshApex` for Custom Apex wire methods. For LDS adapters, mutations via LDS update the cache automatically.

---

## 30. Wire Service Lifecycle

Developers should think of the wire service as a reactive stream, not a traditional "call-and-wait" function.

### Conceptual Lifecycle
1. **Component Created:** `constructor` runs.
2. **Reactive Parameters Available:** Framework resolves parameters (like `recordId`).
3. **Wire Adapter Executes:** Connects to cache/server.
4. **Data / Error Returned:** Provisioned to the property/function.
5. **Component Renders:** UI updates with data.
6. **Reactive Parameter Changes:** User types in search.
7. **Wire Executes Again:** Automatically loops back to step 3.

---

## 31. Wire Service with Getters

Getters allow you to manipulate wired data cleanly before rendering, keeping complex logic out of the HTML template.

### Example
```javascript
export default class ClaimDashboard extends LightningElement {
    @wire(getClaims) claims;

    get activeClaims() {
        // Use optional chaining safely
        return this.claims.data?.filter(
            claim => claim.Status__c === 'Active'
        ) || [];
    }
}
```

### Explanation
*   **Derived data:** Transforms raw wired data into view-specific models.
*   **Template simplicity:** HTML simply calls `<template for:each={activeClaims}>`.
*   **Reactivity:** If `this.claims.data` changes, the getter re-evaluates automatically.
*   **Performance:** Keep getters computationally lightweight.

---

## 32. Wire Service with Iteration

To display a list of wired records, use `for:each` in the template.

### Example
```html
<template lwc:if={claims.data}>
    <ul>
        <template for:each={claims.data} for:item="claim">
            <!-- key is mandatory on the first element inside loop -->
            <li key={claim.Id}>
                {claim.Claim_Number__c} - {claim.Status__c}
            </li>
        </template>
    </ul>
</template>
```

### Flow
Apex Query -> `@wire` property -> `claims.data` -> Template iterates `for:each={claims.data}` -> UI renders rows.

---

## 33. Wire Service with Conditional Rendering

Modern LWC uses `lwc:if`, `lwc:elseif`, and `lwc:else` to handle UI states natively.

### Complete Example
```html
<template>
    <!-- Loading State -->
    <template lwc:if={isLoading}>
        <lightning-spinner alternative-text="Loading"></lightning-spinner>
    </template>

    <!-- Data State -->
    <template lwc:elseif={hasRecords}>
        <template for:each={claims.data} for:item="claim">
            <p key={claim.Id}>{claim.Name}</p>
        </template>
    </template>

    <!-- Empty State -->
    <template lwc:elseif={isEmpty}>
        <p>No claims found for this dealer.</p>
    </template>

    <!-- Error State -->
    <template lwc:else>
        <p class="slds-text-color_error">Error: {errorMessage}</p>
    </template>
</template>
```

---

## 34. Loading State

By default, the wire service is completely asynchronous. `this.claims.data` is `undefined` while loading.

### Handling Spinners
Because wired properties don't have explicit "start" and "end" events, manual loading states using wired functions are preferred.
```javascript
@wire(getClaims)
wiredClaims({ data, error }) {
    this.isLoading = false; // Turn off spinner once provisioned
    if (data) { ... }
}
```
*Note: Set `this.isLoading = true` initially, and optionally set it to `true` when a reactive parameter changes (e.g., inside the `handleSearch` method).*

---

## 35. Empty Data Handling

An empty dataset (`[]`) is a successful server call (data is not undefined, and error is undefined). Distinguishing between states:

*   **Loading:** `data === undefined` && `error === undefined`
*   **No records:** `data !== undefined` && `data.length === 0`
*   **Records found:** `data !== undefined` && `data.length > 0`
*   **Error:** `error !== undefined`

Provide a clean UI state for users when no records match their search or filters.

---

## 36. Wire Service and Pagination

Loading 10,000 Warranty Claims into the browser will crash the component or hit Apex heap limits. 

### Server-Side Pagination
Use `LIMIT` and `OFFSET` in Apex, driven by reactive parameters.
```javascript
pageNumber = 1;

@wire(getClaims, { pageNum: '$pageNumber' })
claims;

handleNext() {
    this.pageNumber += 1;
}
```
*Note: `OFFSET` has a maximum limit of 2,000 in SOQL. For massive datasets, use **Keyset Pagination** (passing the ID or date of the last viewed record).*

---

## 37. Wire Service and Sorting

### Server-Side Sorting (Recommended)
Add `ORDER BY` to the Apex query. Pass the sort field and direction as reactive parameters.
```apex
String query = 'SELECT Id, Name FROM Account ORDER BY ' + sortField + ' ' + sortDir;
```
### Client-Side Sorting
If the dataset is small (e.g., < 100 rows), use a getter or wired function to sort `data` in JavaScript using `Array.prototype.sort()`. Remember to clone the array first `[...data].sort()` because wired data is read-only.

---

## 38. Wire Service and Caching

*   **Wire Caching:** Managed automatically. If `getClaims({ status: 'Open' })` executes, the result is cached. If the user changes status to 'Closed', then back to 'Open', the second 'Open' fetch hits the cache—instant render, zero server load.
*   **Data Freshness:** Caches can go stale. Use `refreshApex` to forcefully invalidate the cache and fetch fresh data if business logic demands real-time accuracy.
*   **Assumption Warning:** Do not assume a wire call executes an Apex query every time a component renders. It queries only if the data isn't cached.

---

## 39. Wire Service vs Imperative Apex

| Feature | Wire Service (`@wire`) | Imperative Apex |
| :--- | :--- | :--- |
| **Execution** | Automatic / Declarative | Manual / Procedural |
| **Reactivity** | Yes (via `$param`) | No (must call manually) |
| **Parameters** | Passed as an object block | Passed as object to function |
| **Caching** | Yes (requires `cacheable=true`) | Optional (usually no cache) |
| **DML Support** | **NO** (Read-only) | **YES** (Insert/Update/Delete) |
| **User Actions** | Not for button clicks | Ideal for button clicks |
| **Error Handling** | Provisioned via `error` obj | Caught via `.catch()` block |
| **Refresh** | Use `refreshApex` | Call function again |
| **Best Use Case**| Loading data on component mount | Saving data, complex orchestrations |

---

## 40. Wire Service vs Lightning Data Service

| Feature | Wire Service (Apex) | Lightning Data Service (Adapters) |
| :--- | :--- | :--- |
| **Purpose** | Complex queries, aggregations, wrappers. | Standard single-record retrieval/updates. |
| **Data Source** | Custom Apex Methods. | Salesforce UI API. |
| **Apex Required** | Yes. | No. |
| **Caching** | Method-specific cache (`cacheable=true`). | Shared global cache across all components. |
| **CRUD** | Read-only (via Wire). | Supports Read/Create/Update/Delete. |
| **Security** | Must enforce `WITH USER_MODE` manually. | Automatically enforces CRUD and FLS. |

---

## 41. Common Mistakes

*   **Problem:** Apex not firing.
    **Cause:** Missing `@AuraEnabled(cacheable=true)`.
    **Solution:** Add the annotation to the Apex method.
*   **Problem:** Passing static parameters accidentally.
    **Cause:** `status: 'selectedStatus'` instead of `'$selectedStatus'`.
    **Solution:** Add the `$` prefix for reactivity.
*   **Problem:** Cannot mutate `data`.
    **Cause:** Wired data is read-only.
    **Solution:** Deep clone it first: `let mutableData = JSON.parse(JSON.stringify(data));`
*   **Problem:** `refreshApex(this.claims.data)` fails.
    **Cause:** Did not store the full provisioned `result`.
    **Solution:** Store `result` in a variable, call `refreshApex(this.result)`.
*   **Problem:** Performing DML inside a wired method.
    **Cause:** `cacheable=true` strictly blocks DML.
    **Solution:** Use imperative Apex for DML operations.

---

## 42. Debugging Wire Service

**Debugging Checklist:**
1.  **Is it executing?** Put a `System.debug` in Apex. If it doesn't print, LWC is using the cache or parameters are undefined.
2.  **Are parameters defined?** `console.log` your reactive variables right before the wire should trigger.
3.  **Are you handling errors?** Log `error` inside the wired function. Often, FLS or SOQL errors fail silently if unhandled.
4.  **Is Cache stale?** Use Chrome DevTools. Check if `refreshApex` is returning a resolved promise with new data.
5.  **Record Access:** Does the running user have access to the records/fields being queried?

---

## 43. Performance Considerations

*   **Selective SOQL:** Limit rows and specify only needed fields in Apex.
*   **LDS over Apex:** Prefer `getRecord` when querying a single record. It saves Apex execution time and shares the global UI API cache.
*   **Avoid Over-fetching:** Do not use `SELECT *` style queries or pull giant datasets. Use Pagination.
*   **Debounce Input:** If wiring a search bar, debounce the keystrokes so you don't fire 10 Apex queries for a 10-letter word.
*   **Governor Limits:** Cached `@wire` calls do not count against Apex governor limits, providing massive scale.

---

## 44. Security Considerations

**Wire Service does NOT magically bypass Salesforce security.** 
*   **LDS Adapters:** Automatically respect Object permissions (CRUD), Field-Level Security (FLS), and Sharing Rules.
*   **Apex Wire Methods:** Default to system context. You **must** enforce security manually:
    *   Use `public with sharing class` to enforce record-level sharing.
    *   Use `WITH USER_MODE` in SOQL (Spring '23 feature) to enforce CRUD and FLS.
    ```apex
    SELECT Id, Name FROM Warranty_Claim__c WITH USER_MODE
    ```

---

## 45. Best Practices Checklist

*   ✅ **Use Wire Service for reactive read operations:** It optimizes caching and rendering natively.
*   ✅ **Use `@AuraEnabled(cacheable=true)`:** Required for Apex wires.
*   ✅ **Use reactive parameters with `$`:** Ensures UI updates immediately when state variables change.
*   ✅ **Handle both data and error:** Never leave a wire property without template conditionals for failure.
*   ✅ **Store the wired result for `refreshApex`:** Specifically capture the full `{ data, error }` wrapper.
*   ✅ **Prefer LDS wire adapters:** If standard fields on a single record are needed, use `getRecord`.
*   ✅ **Avoid expensive transformations in getters:** Getters fire on every render cycle.
*   ✅ **Respect Salesforce security:** Enforce `WITH USER_MODE` in cacheable Apex queries.

---

## 46. Real Project Scenarios (Automotive CRM)

1.  **Warranty Claim List:** Wire Apex fetching `Warranty_Claim__c` filtering by a `$dealerId` reactive parameter.
2.  **Dealer Claim Filtering:** Use multiple reactive params (`$status`, `$dateRange`) to re-query the Claims List.
3.  **Vehicle Service History:** `getRelatedListRecords` to fetch Work Orders related to a `Vehicle__c`.
4.  **Spare Parts Availability:** Wire to Apex hitting an external SAP system callout (if cacheable).
5.  **Invoice List:** Display invoices using `@wire` and `for:each`.
6.  **Shipment Tracking:** Use polling or imperative calls mixed with wire state to show real-time shipment nodes.
7.  **Customer Vehicle List:** Wire querying all `Vehicle__c` where `OwnerId = $userId`.
8.  **Dealer-wise Dashboard:** Aggregate queries via Apex wire to return Wrapper objects with KPI data.

---

## 47. Complete End-to-End Example

**Scenario:** A Warranty Claim Management interface that searches, filters, lists records, handles errors/loading, and uses imperative Apex for approvals followed by a wire refresh.

### 1. Apex Controller (`ClaimController.cls`)
```apex
public with sharing class ClaimController {
    
    @AuraEnabled(cacheable=true)
    public static List<Warranty_Claim__c> getClaims(String statusFilter, String searchKey) {
        String key = '%' + searchKey + '%';
        
        return [SELECT Id, Claim_Number__c, Status__c, Claim_Amount__c, Dealer__r.Name
                FROM Warranty_Claim__c 
                WHERE Status__c = :statusFilter 
                AND Claim_Number__c LIKE :key
                WITH USER_MODE 
                LIMIT 50];
    }

    @AuraEnabled
    public static void approveClaim(Id claimId) {
        Warranty_Claim__c claim = new Warranty_Claim__c(Id = claimId, Status__c = 'Approved');
        update as user claim;
    }
}
```

### 2. LWC JavaScript (`claimManager.js`)
```javascript
import { LightningElement, wire } from 'lwc';
import { refreshApex } from '@salesforce/apex';
import getClaims from '@salesforce/apex/ClaimController.getClaims';
import approveClaim from '@salesforce/apex/ClaimController.approveClaim';

export default class ClaimManager extends LightningElement {
    status = 'Pending';
    searchKey = '';
    
    wiredResult;
    claims = [];
    error;
    isLoading = true;

    // 1. Wire Service Execution
    @wire(getClaims, { statusFilter: '$status', searchKey: '$searchKey' })
    processClaims(result) {
        this.isLoading = false;
        this.wiredResult = result; // Store for refresh
        
        if (result.data) {
            this.claims = result.data;
            this.error = undefined;
        } else if (result.error) {
            this.error = result.error.body ? result.error.body.message : 'Error fetching claims';
            this.claims = [];
        }
    }

    // 2. Reactive Parameter updates trigger automatic Wire rerun
    handleSearch(event) {
        this.isLoading = true;
        this.searchKey = event.target.value;
    }

    handleStatusChange(event) {
        this.isLoading = true;
        this.status = event.target.value;
    }

    // 3. Imperative Apex + refreshApex
    handleApprove(event) {
        this.isLoading = true;
        const claimId = event.target.dataset.id;
        
        approveClaim({ claimId })
            .then(() => {
                // Refresh cache to show new Approved status
                return refreshApex(this.wiredResult);
            })
            .catch(err => {
                this.error = 'Failed to approve: ' + err.body.message;
            })
            .finally(() => {
                this.isLoading = false;
            });
    }

    // 4. Getter for Empty State
    get isEmpty() {
        return this.claims.length === 0;
    }
}
```

### 3. LWC HTML (`claimManager.html`)
```html
<template>
    <lightning-card title="Warranty Claim Manager" icon-name="standard:case">
        
        <!-- Filters -->
        <div class="slds-m-around_medium slds-grid slds-gutters">
            <lightning-input type="search" label="Search by Claim #" onchange={handleSearch} class="slds-col"></lightning-input>
            <lightning-combobox label="Status Filter" value={status} options={statusOptions} onchange={handleStatusChange} class="slds-col"></lightning-combobox>
        </div>

        <!-- Spinner -->
        <template lwc:if={isLoading}>
            <lightning-spinner alternative-text="Loading"></lightning-spinner>
        </template>

        <!-- Error State -->
        <template lwc:elseif={error}>
            <div class="slds-text-color_error slds-m-around_medium">{error}</div>
        </template>

        <!-- Empty State -->
        <template lwc:elseif={isEmpty}>
            <div class="slds-m-around_medium">No claims match your filters.</div>
        </template>

        <!-- Data State -->
        <template lwc:else>
            <div class="slds-m-around_medium">
                <template for:each={claims} for:item="claim">
                    <div key={claim.Id} class="slds-box slds-m-bottom_small">
                        <p><strong>{claim.Claim_Number__c}</strong> - {claim.Status__c}</p>
                        <p>Amount: ${claim.Claim_Amount__c} | Dealer: {claim.Dealer__r.Name}</p>
                        <lightning-button label="Approve" data-id={claim.Id} onclick={handleApprove}></lightning-button>
                    </div>
                </template>
            </div>
        </template>
    </lightning-card>
</template>
```

### 4. LWC XML (`claimManager.js-meta.xml`)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="[http://soap.sforce.com/2006/04/metadata](http://soap.sforce.com/2006/04/metadata)">
    <apiVersion>58.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__AppPage</target>
        <target>lightning__RecordPage</target>
    </targets>
</LightningComponentBundle>
```

---

## 48. Interview Questions & Answers

### Beginner Questions
**Q: What is Wire Service in LWC?**
A: A declarative mechanism to fetch data reactively. It connects a component to Salesforce data (LDS or Apex) and automatically updates the component when parameters or cache changes.

**Q: What is the @wire decorator?**
A: A JavaScript decorator used to annotate a property or function, instructing the LWC framework to provision data from a specified data adapter.

**Q: Can a wired Apex method perform DML?**
A: No. It must be annotated with `@AuraEnabled(cacheable=true)`, which strictly enforces read-only operations. Attempting DML causes an exception.

### Intermediate Questions
**Q: What is the difference between wired property and wired function?**
A: A wired property assigns the data/error directly to a variable for template rendering. A wired function allows execution of custom JavaScript logic (filtering, modifying state) every time the data is provisioned.

**Q: What does the $ symbol mean in @wire?**
A: It marks a parameter as *reactive*. It binds the parameter to a component property. If the property changes, the wire service automatically re-evaluates and executes the query again.

**Q: How do you handle errors from a wired method?**
A: By checking `this.property.error` (for properties) or the `error` argument (for functions). Errors should be logged and a user-friendly message mapped to a UI property for display.

### Advanced Questions
**Q: What is refreshApex and when should it be used?**
A: `refreshApex` forces a server-side request, bypassing the client cache. It is used exclusively for custom Apex wired methods, typically after an imperative DML operation modifies the data that the wire is currently displaying.

**Q: How do you refresh wired Apex data after DML?**
A: Store the complete provisioned `result` object (not just `result.data`) in a variable. Execute your imperative DML. In the `.then()` block, return `refreshApex(this.storedResult)`.

**Q: What is the difference between Wire Service and Imperative Apex?**
A: Wire service is declarative, reactive, cached, and read-only. Imperative Apex is manual, promise-based, usually uncached, and used for write operations (DML) or logic that requires explicit button-click triggers.

### Architect-Level Questions
**Q: How does Wire Service respect Salesforce security?**
A: LDS adapters inherently respect CRUD, FLS, and sharing. Custom Apex wires do *not* automatically enforce CRUD/FLS. The Architect must ensure the Apex class uses `with sharing` and SOQL includes `WITH USER_MODE` or explicit schema checks.

**Q: How do you optimize Wire Service for large datasets?**
A: Wire service should not be used to pull thousands of rows. Implement server-side SOQL pagination (LIMIT/OFFSET), utilize reactive `$pageNumber` parameters, ensure fields in the SELECT statement are minimized, and consider lazy loading strategies.

---

## 49. Revision Summary

*   **Wire Service:** Declarative, reactive, cache-driven read operations.
*   **@wire:** Decorator syntax to bind adapters/Apex.
*   **Wired Property:** Direct assignment (`this.data`, `this.error`).
*   **Wired Function:** Intercept data provisioning for logic (`({ data, error }) => {}`).
*   **@AuraEnabled(cacheable=true):** Mandatory for Apex `@wire`. Blocks DML. Enables LDS caching.
*   **Reactive Data & $ Syntax:** `'$param'` triggers automatic re-execution when `this.param` updates.
*   **recordId:** Use `@api recordId` and `'$recordId'` to fetch context-specific data.
*   **LDS Wire Adapters:** e.g., `getRecord`. No Apex needed. Inherently secure. Shares UI API cache.
*   **Data Handling:** Always check `if (data)` and `else if (error)`. Clear old state.
*   **refreshApex:** Bypasses Apex cache. Requires the raw provisioned object. Used after imperative DML.
*   **UI States:** Explicitly manage Loading, Empty, and Error states using `lwc:if/elseif/else`.
*   **Wire + Imperative:** Wire reads data; Imperative mutates data; `refreshApex` syncs them.
*   **Security:** Apex `@wire` defaults to system mode FLS. Enforce with `USER_MODE`.
*   **Best Practice:** Always use Wire for fetching read-only data to leverage Salesforce's client-side caching architecture.