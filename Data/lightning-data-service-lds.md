# Lightning Data Service (LDS)

# 1. Introduction

Lightning Data Service (LDS) is the standard Salesforce mechanism for Lightning Web Components (LWC) and Aura components to read, create, update, and delete Salesforce records without requiring server-side Apex code. 

Salesforce introduced LDS to streamline component development, reduce boilerplate Apex, and optimize performance. Before LDS, every database interaction required custom Apex controllers, which meant more code to write, test, and maintain, alongside redundant server trips. LDS solves this by providing a unified, client-side data layer.

Modern Salesforce development relies heavily on LDS because it automatically handles data caching, reactive updates, and security (CRUD/FLS). In an Automotive CRM context, whether you are querying a **Warranty Claim**, creating a **Work Order**, or updating a **Vehicle** record's status, LDS allows you to interact with Salesforce data efficiently and securely.

# 2. What is Lightning Data Service?

Lightning Data Service is a centralized data-caching and synchronization framework that sits between your LWC and the Salesforce UI API. It acts as the client-side data layer for Lightning components.

*   **Lightning Data Service (LDS):** The client-side framework managing the data lifecycle.
*   **Salesforce UI API:** The underlying REST API that LDS uses to fetch data, metadata, and layouts from Salesforce.
*   **LWC:** The frontend framework consuming the data.
*   **Client-side cache:** A shared local repository in the browser where LDS stores record data to prevent redundant server calls.
*   **Record data:** The actual field values and metadata of Salesforce records.
*   **Security:** LDS automatically enforces Field-Level Security (FLS) and sharing rules.
*   **Reactive updates:** If a record changes in the cache, all LWCs listening to that record automatically re-render.

### LDS Architecture Diagram

```text
       [ LWC ]
          ↓
[ Lightning Data Service ]
          ↓
      [ UI API ]
          ↓
   [ Salesforce Data ]
          ↓
     [ LDS Cache ]
          ↓
      [ LWC UI ]
```

# 3. Why Use LDS?

LDS should be preferred over Apex whenever you are performing standard CRUD operations on single records or retrieving standard metadata.

**Advantages:**
*   **No Apex Required:** Reduces server-side code, testing, and maintenance.
*   **Automatic Caching:** Minimizes network requests and speeds up rendering.
*   **Reactive Data:** Components update instantly when the underlying data changes.
*   **Built-in Security:** Automatically enforces user permissions, CRUD, and FLS.
*   **Standardized Data Access:** Uses consistent, reliable platform APIs.
*   **Easier Maintenance:** Less custom logic means fewer bugs.
*   **Better Performance:** Reduces server load by leveraging client-side shared memory.

**When to prefer LDS over Apex:**
Use LDS for single-record CRUD, fetching picklist values, related lists, and object metadata. Use Apex only for complex SOQL, bulk DML, transactions, callouts, or heavy business logic.

# 4. LDS Architecture

LDS sits at the client level and communicates with the Salesforce database via the UI API.

```text
       [ LWC ] (Component Layer)
          ↓
       [ LDS ] (Client-Side Cache & Logic)
          ↓
      [ UI API ] (REST API Endpoint)
          ↓
[ Salesforce Platform ] (App Server / Security)
          ↓
      [ Database ] (Storage)
```

*   **Client-side cache:** Stores records uniquely identified by their Record ID.
*   **Shared record data:** If Component A and Component B both request Warranty Claim `WC-1001`, LDS fetches it once and shares the cached copy with both.
*   **UI API:** Translates standard database rows into UI-friendly JSON containing data, display values, and metadata.
*   **Security enforcement:** Strips out fields the user lacks FLS for before it even hits the component.
*   **Data synchronization:** Ensures all components displaying the same record stay in sync.

# 5. Lightning Data Service vs UI API

The UI API is the actual Salesforce API endpoint. LDS is the client-side wrapper that uses the UI API.

| Feature | Lightning Data Service (LDS) | Salesforce UI API |
| :--- | :--- | :--- |
| **Location** | Client-side (Browser) | Server-side (Salesforce Platform) |
| **Usage** | Imported in LWC via `@wire` or imperative functions | Consumed via HTTP REST calls or by LDS |
| **Caching** | Built-in client-side caching | None (stateless API) |
| **Reactivity** | Highly reactive (updates UI automatically) | Not applicable |
| **Role** | LWC Data framework | Platform API for Lightning experiences |

UI API provides the platform APIs used by Lightning experiences natively, and LDS provides client-side data access capabilities mapping to those endpoints for custom Lightning components.

# 6. LDS and Wire Service

The Wire Service (`@wire`) is a reactive LWC feature. LDS provides the *adapters* that the Wire Service uses to fetch Salesforce data. LDS commonly uses wire adapters in LWC.

```text
       [ LWC ]
          ↓
       [ @wire ] (Reactive engine)
          ↓
 [ LDS Wire Adapter ] (e.g., getRecord)
          ↓
      [ UI API ]
          ↓
    [ Salesforce ]
```

# 7. LDS Wire Adapters

LDS exposes several wire adapters imported from `lightning/ui*Api` modules.

| Adapter | Module | Purpose | Use Case |
| :--- | :--- | :--- | :--- |
| `getRecord` | `lightning/uiRecordApi` | Retrieves a single record's data. | Load Warranty Claim details. |
| `getObjectInfo` | `lightning/uiObjectInfoApi` | Retrieves object metadata. | Check if User can create a Vehicle. |
| `getPicklistValues` | `lightning/uiObjectInfoApi` | Retrieves picklist values for a field. | Load "Claim Status" options. |
| `getPicklistValuesByRecordType` | `lightning/uiObjectInfoApi` | Gets all picklist values for a specific Record Type. | Load all picklists for a "Standard" Claim. |
| `getRelatedListRecords` | `lightning/uiRelatedListApi` | Retrieves records from a related list. | Get Work Order Line Items. |
| `getRelatedListInfo` | `lightning/uiRelatedListApi` | Retrieves metadata for a related list. | Get columns for a related list. |
| `getRecords` | `lightning/uiRecordApi` | Retrieves multiple records at once. | Load specific records in bulk. |

# 8. getRecord

`getRecord` fetches a single Salesforce record securely.

```javascript
import { LightningElement, api, wire } from 'lwc';
import { getRecord } from 'lightning/uiRecordApi';
import ID_FIELD from '@salesforce/schema/Account.Id';
import NAME_FIELD from '@salesforce/schema/Account.Name';
import STATUS_FIELD from '@salesforce/schema/Warranty_Claim__c.Status__c';
import CUSTOM_FIELD from '@salesforce/schema/Account.Custom_Field__c';

const FIELDS = [ID_FIELD, NAME_FIELD, STATUS_FIELD, CUSTOM_FIELD];

export default class RecordDetail extends LightningElement {
    @api recordId;

    @wire(getRecord, { recordId: '$recordId', fields: FIELDS, optionalFields: [] })
    record;
}
```

*   **FIELD imports:** Ensures referential integrity. If a field is deleted in Setup, deployment fails.
*   **recordId:** The ID of the record to fetch.
*   **fields:** Array of imported schema fields to strictly retrieve. Will error if user lacks access.
*   **optionalFields:** Fields to fetch only if the user has access (doesn't throw an error if they don't).
*   **data / error:** The `@wire` provisions a generic object containing `data` (the record payload) and `error` (any permissions/network issues).

# 9. getFieldValue

`getFieldValue` safely extracts a field's value from a record returned by `getRecord`.

```javascript
import { getFieldValue } from 'lightning/uiRecordApi';
import ACCOUNT_NAME from '@salesforce/schema/Account.Name';
// ...
get accountName() {
    return getFieldValue(this.record.data, ACCOUNT_NAME);
}
```

*   **Why getFieldValue is useful:** It handles complex nested JSON structures (`data.fields.Account.value.fields.Name.value`) elegantly.
*   **Relationship fields:** Easily traverse relationships safely.
*   **Avoiding complicated nested property access:** Keeps getters clean.
*   **Handling undefined data:** Automatically handles cases where data is not yet loaded, returning `undefined` instead of throwing a JavaScript type error.

# 10. getObjectInfo

Retrieves metadata about an object, such as record type IDs, field definitions, and CRUD permissions.

```javascript
import { LightningElement, wire } from 'lwc';
import { getObjectInfo } from 'lightning/uiObjectInfoApi';
import VEHICLE_OBJECT from '@salesforce/schema/Vehicle__c';

export default class VehicleMetadata extends LightningElement {
    @wire(getObjectInfo, { objectApiName: VEHICLE_OBJECT })
    objectInfo;

    get objectLabel() { return this.objectInfo.data?.label; }
    get defaultRecordTypeId() { return this.objectInfo.data?.defaultRecordTypeId; }
    get isCreateable() { return this.objectInfo.data?.createable; }
    get isUpdateable() { return this.objectInfo.data?.updateable; }
    get isDeletable() { return this.objectInfo.data?.deletable; }
}
```

# 11. getPicklistValues

Retrieves valid picklist values for a specific field and record type.

```javascript
import { LightningElement, wire } from 'lwc';
import { getPicklistValues } from 'lightning/uiObjectInfoApi';
import STATUS_FIELD from '@salesforce/schema/WorkOrder.Status';

export default class WorkOrderStatus extends LightningElement {
    @wire(getPicklistValues, { 
        recordTypeId: '012000000000000AAA', // Often dynamic
        fieldApiName: STATUS_FIELD 
    })
    statusPicklistValues;
    
    // Often mapped for lightning-combobox options
    get options() {
        return this.statusPicklistValues.data ? this.statusPicklistValues.data.values : [];
    }
}
```

# 12. getPicklistValuesByRecordType

Returns a collection of *all* picklist fields and their values for a specific record type. 

*   `getPicklistValues`: Best for a single field (e.g., just `Status`).
*   `getPicklistValuesByRecordType`: Best for custom forms where you need multiple picklists (e.g., `Status`, `Priority`, `Category`) for a specific record type in one server call, reducing network overhead.

# 13. getRelatedListRecords

Fetches records associated via a related list using LDS.

```javascript
import { LightningElement, api, wire } from 'lwc';
import { getRelatedListRecords } from 'lightning/uiRelatedListApi';

export default class ClaimLineItems extends LightningElement {
    @api recordId; // Warranty Claim ID

    @wire(getRelatedListRecords, {
        parentRecordId: '$recordId',
        relatedListId: 'Claim_Line_Items__r',
        fields: ['Claim_Line_Item__c.Name', 'Claim_Line_Item__c.Amount__c']
    })
    lineItems;
}
```

# 14. recordId

When a component is placed on a Lightning Record Page via Lightning App Builder, Salesforce automatically injects the current record context's ID into an `@api` decorated property named exactly `recordId`.

```javascript
import { LightningElement, api } from 'lwc';

export default class RecordContextComponent extends LightningElement {
    @api recordId; // Automatically populated by standard record pages
}
```

# 15. Reactive Data with LDS

LDS wire adapters are reactive.

```javascript
@wire(getRecord, {
    recordId: '$recordId',
    fields: FIELDS
})
record;
```

*   **$ symbol:** Makes the parameter dynamic/reactive. 
*   **Automatic updates:** If the user navigates from one record page to another, `$recordId` updates, and LDS fetches the new record automatically without writing manual re-fetch logic.

# 16. Client-Side Caching

LDS maintains a local cache of record data. You must understand cache behavior instead of assuming every request always reaches the server.

```text
First Request
  ↓
Salesforce
  ↓
LDS Cache
  ↓
Component

Second Request (Same Record)
  ↓
LDS Cache
  ↓
Component (No Server Trip)
```

*   **Why caching exists:** To dramatically reduce server requests and improve UI responsiveness.
*   **Shared cache:** All components on a page share the same cache.
*   **Data freshness:** LDS attempts to keep data fresh, but external background changes won't reflect until invalidation.
*   **Cache invalidation:** Managed by LDS inherently during LDS-driven updates.

# 17. Shared LDS Cache

Multiple components can benefit from shared record data.

```text
Component A
     ↓
   LDS Cache
     ↑
Component B
```

If Component A (Header) and Component B (Details) both require Warranty Claim `WC-1001`, LDS fetches it once. This reduces duplicate server requests and keeps Lightning experiences perfectly consistent.

# 18. Automatic Record Updates

LDS keeps record data synchronized automatically when modified via LDS.

```text
Component A
    ↓
Updates Account
    ↓
LDS (Cache updated)
    ↓
Component B
    ↓
Updated Data (Re-renders instantly)
```
If Component A updates a record, Component B (wired to the same record) receives the new data automatically without requiring pub-sub or manual refresh mechanisms.

# 19. createRecord

Creates a new record imperatively.

```javascript
import { LightningElement } from 'lwc';
import { createRecord } from 'lightning/uiRecordApi';
import CLAIM_OBJECT from '@salesforce/schema/Warranty_Claim__c';
import NAME_FIELD from '@salesforce/schema/Warranty_Claim__c.Name';

export default class CreateClaim extends LightningElement {
    handleCreate() {
        const fields = {};
        fields[NAME_FIELD.fieldApiName] = 'New Engine Claim';
        
        const recordInput = { apiName: CLAIM_OBJECT.objectApiName, fields };

        createRecord(recordInput)
            .then(claim => {
                console.log('Success. Created ID: ', claim.id);
            })
            .catch(error => {
                console.error('Error creating record: ', error);
            });
    }
}
```

# 20. updateRecord

Updates an existing record.

```javascript
import { LightningElement, api } from 'lwc';
import { updateRecord } from 'lightning/uiRecordApi';
import ID_FIELD from '@salesforce/schema/Warranty_Claim__c.Id';
import STATUS_FIELD from '@salesforce/schema/Warranty_Claim__c.Status__c';

export default class UpdateClaim extends LightningElement {
    @api recordId;

    handleUpdate() {
        const fields = {};
        fields[ID_FIELD.fieldApiName] = this.recordId;
        fields[STATUS_FIELD.fieldApiName] = 'Approved';

        const recordInput = { fields };

        updateRecord(recordInput)
            .then(() => console.log('Updated successfully'))
            .catch(error => console.error('Error updating', error));
    }
}
```

# 21. deleteRecord

Deletes an existing record.

```javascript
import { LightningElement } from 'lwc';
import { deleteRecord } from 'lightning/uiRecordApi';

export default class DeletePart extends LightningElement {
    handleDelete(partId) {
        if(confirm('Are you sure?')) {
            deleteRecord(partId)
                .then(() => {
                    // UI automatically reflects the deletion if wired
                    console.log('Record deleted successfully');
                })
                .catch(error => console.error('Error deleting', error));
        }
    }
}
```

# 22. CRUD with LDS

| Operation | LDS API | Supported Base Components |
| :--- | :--- | :--- |
| **Create** | `createRecord` | `lightning-record-form`, `lightning-record-edit-form` |
| **Read** | `getRecord` | `lightning-record-form`, `lightning-record-view-form` |
| **Update** | `updateRecord` | `lightning-record-form`, `lightning-record-edit-form` |
| **Delete** | `deleteRecord` | None (Imperative only) |

# 23. lightning-record-form

A high-level component that displays a read, edit, or create form based on its configuration, directly tied to page layouts.

```html
<lightning-record-form
    record-id={recordId}
    object-api-name="Warranty_Claim__c"
    layout-type="Full"
    columns="2"
    mode="view">
</lightning-record-form>
```

**Use when:** You need a standard form quickly without custom layout control. It automatically switches modes and handles save actions.

# 24. lightning-record-edit-form

Provides a customizable form for creating or editing records, allowing manual placement of fields.

```html
<lightning-record-edit-form record-id={recordId} object-api-name="Warranty_Claim__c" onsuccess={handleSuccess} onerror={handleError}>
    <lightning-messages></lightning-messages>
    <lightning-input-field field-name="Name"></lightning-input-field>
    <lightning-input-field field-name="Status__c"></lightning-input-field>
    <lightning-button type="submit" label="Save Claim"></lightning-button>
</lightning-record-edit-form>
```

# 25. lightning-record-view-form

Provides a customizable read-only view of a record.

```html
<lightning-record-view-form record-id={recordId} object-api-name="Dealer__c">
    <div class="slds-box">
        <lightning-output-field field-name="Name"></lightning-output-field>
        <lightning-output-field field-name="Region__c"></lightning-output-field>
    </div>
</lightning-record-view-form>
```

# 26. lightning-input-field

Used inside `lightning-record-edit-form`. It automatically renders the correct UI control based on the field type (e.g., a lookup component for lookups, a combobox for picklists) and enforces required fields natively based on schema validation.

# 27. LDS Forms Comparison

| Feature | `lightning-record-form` | `lightning-record-edit-form` | `lightning-record-view-form` | Imperative API |
| :--- | :--- | :--- | :--- | :--- |
| **Flexibility** | Low | Medium | Medium | High |
| **Custom UI** | No (uses Page Layout) | Yes (place fields anywhere) | Yes | Yes (custom HTML entirely) |
| **Validation** | Automatic | Automatic | N/A | Manual / API handled |
| **Events** | Load, Submit, Success, Error | Load, Submit, Success, Error | Load | Promise then/catch |
| **CRUD** | C, R, U | C, U | R | C, R, U, D |
| **Use Case** | Quick standard forms | Custom layout data entry | Read-only custom layout | Custom wizards, bulk updates |

# 28. LDS Validation

LDS automatically respects:
*   Required fields at the schema level.
*   Salesforce Validation Rules (server-side).
*   Field-level type validation (client-side).
Errors are thrown during save attempts, and standard components (`lightning-messages`, `lightning-input-field`) render these error events automatically.

# 29. LDS Error Handling

Errors can stem from `getRecord`, mutations, UI API failures, Validation Rules, or Permission/FLS errors. 

```javascript
.catch(error => {
    // Example extracting UI API error structure
    const message = error.body ? error.body.message : 'Unknown error';
    this.showToast('Error', message, 'error');
})
```

# 30. LDS Error Object

Create a reusable utility to extract messages safely.

```javascript
// errorUtils.js
export function getErrorMessage(error) {
    if (error) {
        if (Array.isArray(error.body)) {
            return error.body.map(e => e.message).join(', ');
        }
        if (error.body && error.body.message) {
            return error.body.message;
        }
    }
    return 'An unknown error occurred';
}
```

# 31. notifyRecordUpdateAvailable

Notifies the LDS cache that a record has been modified *outside* of standard LDS paths (e.g., via Imperative Apex, Visualforce, or external integrations) so the cache can synchronize.

```text
Apex Update
    ↓
Salesforce Record
    ↓
notifyRecordUpdateAvailable()
    ↓
LDS Clears Cache
    ↓
UI Refresh
```

```javascript
import { notifyRecordUpdateAvailable } from 'lightning/uiRecordApi';

async handleApexProcess() {
    await invokeApexLogic({ recordId: this.recordId });
    await notifyRecordUpdateAvailable([{recordId: this.recordId}]);
}
```

# 32. refreshApex vs notifyRecordUpdateAvailable

| Feature | `refreshApex` | `notifyRecordUpdateAvailable` |
| :--- | :--- | :--- |
| **Purpose** | Refreshes data provisioned by an `@wire(ApexMethod)` | Refreshes data cached by LDS (`@wire(getRecord)`) |
| **Data Source** | Custom Apex | Standard UI API / LDS |
| **Usage** | Pass the exact variable holding the wire provision | Pass an array of recordIds |
| **Example** | `refreshApex(this.wiredApexResult)` | `notifyRecordUpdateAvailable([{recordId: id}])` |

*Use `refreshApex` strictly for wired Apex results. Use `notifyRecordUpdateAvailable` to notify LDS/UI API that specific records changed.*

# 33. RefreshView API

The modern RefreshView API approach synchronizes component data natively inside standard Lightning Pages without hardcoding communication between sibling components.

```javascript
import { RefreshEvent } from 'lightning/refresh';
// ...
// Tell the view hierarchy something changed, prompting a refresh of view-level data
this.dispatchEvent(new RefreshEvent());
```
Unlike `refreshApex` (which is highly localized), `RefreshView API` coordinates updates across multiple standard and custom components on a page.

# 34. LDS with Imperative Apex

Use LDS for standard display, and Imperative Apex for complex business operations. Once Apex completes the custom business operation and updates the record, synchronize the UI.

```text
LDS -> Display Record -> User clicks button -> Imperative Apex -> Update Record -> notifyRecordUpdateAvailable -> Data Consistency
```

# 35. LDS vs Imperative Apex

| Feature | LDS | Imperative Apex |
| :--- | :--- | :--- |
| **Purpose** | Simple CRUD & metadata | Custom logic, callouts, bulk |
| **Apex Requirement** | None | Required |
| **CRUD** | Supported natively | DML Required |
| **Security** | Automatic (CRUD/FLS) | Developer enforced (`WITH SECURITY_ENFORCED`) |
| **Caching** | Automatic client-side | Manual (`@AuraEnabled(cacheable=true)`) |
| **SOQL** | Cannot run complex queries | Full SOQL/SOSL capability |
| **Complexity** | Low | High |
| **Performance** | High (Client-Side) | Lower (Server-Side trips) |

*Always prefer LDS unless complex SOQL, callouts, or heavy business logic dictate Apex.*

# 36. LDS vs Wire Service

*   **Wire Service (`@wire`):** A reactive framework mechanism to provision streams of data to components.
*   **LDS:** A Salesforce data service. LDS simply *provides* the wire adapters (`getRecord`) that the Wire Service uses to operate.

# 37. LDS vs SOQL

*   **LDS:** Perfect for simple record retrieval. Respects UI API, enforces security automatically, heavily caches.
*   **Apex SOQL:** Required for complex queries, deep joins/relationship queries, aggregations (`GROUP BY`), and custom business logic. Harder to maintain and requires explicit security handling.

# 38. LDS Security

LDS operates strictly within the context of the running user.
*   **CRUD/FLS:** Automatically enforced. Fields missing FLS are stripped.
*   **Record-level security (Sharing):** Enforced natively.
*   **User permissions / Permission Sets:** Fully respected by UI API security.

LDS is safer than writing custom Apex because the platform guarantees security enforcement without the risk of a developer forgetting `WITH USER_MODE`.

# 39. LDS and Governor Limits

LDS requests do not consume standard Apex governor limits (like 100 SOQL/150 DML) directly, as they run via the UI API. However, UI API calls still make server requests (if not cached) and count towards general org API limits. LDS does not mean limits disappear.

# 40. LDS Performance

*   **Client-side caching:** Prevents repetitive server requests.
*   **Field selection:** Requesting only necessary fields speeds up payload delivery.
*   **Reusing LDS data:** Multiple components on the same page using LDS for the same record only hit the server once.
*   **Avoiding unnecessary Apex:** Reduces server-side execution time.

# 41. LDS with Conditional Rendering

```html
<template>
    <div lwc:if={claim.data}>
        <p>Claim Data Loaded.</p>
    </div>
    <div lwc:elseif={claim.error}>
        <p>Error loading data.</p>
    </div>
    <div lwc:else>
        <lightning-spinner alternative-text="Loading"></lightning-spinner>
    </div>
</template>
```

# 42. LDS with Iteration

```html
<template lwc:if={lineItems.data}>
    <ul>
        <template for:each={lineItems.data.records} for:item="item">
            <li key={item.id}>
                {item.fields.Name.value} - {item.fields.Amount__c.value}
            </li>
        </template>
    </ul>
</template>
```

# 43. LDS with Datatable

```javascript
get datatableData() {
    if (this.lineItems.data) {
        return this.lineItems.data.records.map(record => ({
            Id: record.id,
            Name: record.fields.Name.value,
            Amount: record.fields.Amount__c.value
        }));
    }
    return [];
}
```
You map the nested LDS `getRelatedListRecords` format into a flat array suitable for `lightning-datatable` columns.

# 44. LDS with Record Pages

Using Dynamic Forms and Lightning App Builder, components can rely entirely on `@api recordId` context.
*   Drop component on Warranty Claim page.
*   `recordId` is injected.
*   LDS wire fires instantly with current context.

# 45. LDS and Custom Objects

Using Automotive CRM Custom Objects:

```javascript
import CLAIM_OBJECT from '@salesforce/schema/Warranty_Claim__c';
import DEALER_FIELD from '@salesforce/schema/Warranty_Claim__c.Dealer__c';
import AMOUNT_FIELD from '@salesforce/schema/Warranty_Claim__c.Claim_Amount__c';

// getRecord safely retrieves these specific fields
```

# 46. LDS and Picklists

Loading Warranty Claim Status Picklist:

```javascript
@wire(getObjectInfo, { objectApiName: CLAIM_OBJECT })
objectInfo;

@wire(getPicklistValues, { 
    recordTypeId: '$objectInfo.data.defaultRecordTypeId', 
    fieldApiName: STATUS_FIELD 
})
statusPicklist;
```

# 47. LDS and Related Records

Use `getRelatedListRecords` for:
*   Work Order → Work Order Line Items
*   Warranty Claim → Claim Line Items
*   Vehicle → Service History

Useful when you want to display a subset of related records on a parent record page without Apex SOQL.

# 48. LDS Lifecycle

1.  **Component Created**
2.  **Record ID Available** (Injected by Lightning page)
3.  **LDS Wire Adapter** (triggers reactively)
4.  **UI API** (called by LDS)
5.  **Cache / Server** (checked for data)
6.  **Data / Error** (provisioned to JS property)
7.  **UI Rendering** (automatically updates)
8.  **Record Changes** (LDS mutation or external change)
9.  **LDS Synchronization** (cache updates)
10. **UI Update** (re-renders all wired components)

# 49. Common Mistakes

*   **Problem:** Using Apex when LDS is sufficient.
    *   **Cause:** Old Visualforce/Aura habits.
    *   **Solution:** Default to LDS forms/adapters.
*   **Problem:** Requesting unnecessary fields.
    *   **Cause:** "Select *" mentality.
    *   **Solution:** Specify only required fields in `getRecord`.
*   **Problem:** Not handling undefined data.
    *   **Cause:** Trying to read properties before wire resolves.
    *   **Solution:** Use `getFieldValue` and `lwc:if`.
*   **Problem:** Using `refreshApex` for LDS data incorrectly.
    *   **Cause:** Confusing wire provisioning methods.
    *   **Solution:** Use `notifyRecordUpdateAvailable`.
*   **Problem:** Using LDS for complex SOQL requirements.
    *   **Cause:** Pushing LDS beyond its single-record/related-list design.
    *   **Solution:** Switch to custom Apex.

# 50. Debugging LDS

*   **getRecord not returning data:** Is `@api recordId` correctly populated?
*   **Field not available:** Check Profile FLS settings.
*   **Permission errors:** Does the user have CRUD on the object?
*   **createRecord failure:** Check for missing required fields or Validation Rules.
*   **UI not refreshing:** Did you forget `notifyRecordUpdateAvailable` after an Apex update?
*   **Cache confusion:** Check network tab to see if a UI API call is actually being made.

# 51. Best Practices Checklist

*   ✅ **Prefer LDS for standard record CRUD:** Only use Apex when absolutely necessary.
*   ✅ **Use getRecord for record retrieval.**
*   ✅ **Use getFieldValue for field access:** Safest way to navigate the JSON.
*   ✅ **Use getObjectInfo for metadata.**
*   ✅ **Use getPicklistValues for picklists.**
*   ✅ **Use getRelatedListRecords for related lists when appropriate.**
*   ✅ **Use createRecord/updateRecord/deleteRecord for standard CRUD operations.**
*   ✅ **Use lightning-record-form for simple record forms.**
*   ✅ **Use lightning-record-edit-form for customized forms.**
*   ✅ **Use lightning-record-view-form for read-only forms.**
*   ✅ **Handle loading, success, empty, and error states:** Never leave the user guessing.
*   ✅ **Respect CRUD and FLS:** Handled natively by LDS.
*   ✅ **Use LDS caching effectively:** Don't force server refreshes unnecessarily.
*   ✅ **Request only the fields that are required.**
*   ✅ **Use Apex when complex SOQL, business logic, aggregation, or callouts are required.**
*   ✅ **Use notifyRecordUpdateAvailable when appropriate:** Bridging Apex changes to LDS.
*   ✅ **Use refreshApex only for wired Apex data.**
*   ✅ **Avoid loading unnecessarily large datasets.**

# 52. Real Project Scenarios

1.  **Display Warranty Claim Details:** Use `getRecord` or `lightning-record-view-form`. Secure, cached, performant.
2.  **Create Warranty Claim:** Use `lightning-record-edit-form` for UI, or `createRecord` for custom wizard logic.
3.  **Edit Warranty Claim:** Use `updateRecord` to update status on a quick action button.
4.  **Delete Claim Line:** Imperatively call `deleteRecord` from a datatable row action.
5.  **Display Shipment Related Records:** `getRelatedListRecords` to show tracking updates under an Order.
6.  **Load Warranty Claim Status Picklist:** `getPicklistValues` mapped to a combo-box for inline editing.

# 53. Complete End-to-End Example

*Warranty Claim Management Component*

**HTML (warrantyClaimManager.html)**
```html
<template>
    <lightning-card title="Warranty Claim Overview">
        <template lwc:if={claim.data}>
            <div class="slds-m-around_medium">
                <p><strong>Claim Number:</strong> {claimNumber}</p>
                <p><strong>Status:</strong> {claimStatus}</p>
                <p><strong>Amount:</strong> {claimAmount}</p>
                
                <div class="slds-m-top_medium">
                    <lightning-button label="Approve Claim" variant="brand" onclick={handleApprove}></lightning-button>
                </div>
            </div>
            
            <div class="slds-m-top_large">
                <template lwc:if={lineItemsData}>
                    <lightning-datatable
                        key-field="Id"
                        data={lineItemsData}
                        columns={columns}
                        onrowaction={handleRowAction}>
                    </lightning-datatable>
                </template>
            </div>
        </template>
        
        <template lwc:elseif={claim.error}>
            <p class="slds-text-color_error">Error loading claim data.</p>
        </template>
        
        <template lwc:else>
            <lightning-spinner alternative-text="Loading"></lightning-spinner>
        </template>
    </lightning-card>
</template>
```

**JavaScript (warrantyClaimManager.js)**
```javascript
import { LightningElement, api, wire } from 'lwc';
import { getRecord, getFieldValue, updateRecord, deleteRecord } from 'lightning/uiRecordApi';
import { getRelatedListRecords } from 'lightning/uiRelatedListApi';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

import ID_FIELD from '@salesforce/schema/Warranty_Claim__c.Id';
import NAME_FIELD from '@salesforce/schema/Warranty_Claim__c.Name';
import STATUS_FIELD from '@salesforce/schema/Warranty_Claim__c.Status__c';
import AMOUNT_FIELD from '@salesforce/schema/Warranty_Claim__c.Claim_Amount__c';

const FIELDS = [NAME_FIELD, STATUS_FIELD, AMOUNT_FIELD];
const COLUMNS = [
    { label: 'Line Item', fieldName: 'Name' },
    { label: 'Amount', fieldName: 'Amount', type: 'currency' },
    { type: 'action', typeAttributes: { rowActions: [{ label: 'Delete', name: 'delete' }] } }
];

export default class WarrantyClaimManager extends LightningElement {
    @api recordId;
    columns = COLUMNS;

    @wire(getRecord, { recordId: '$recordId', fields: FIELDS })
    claim;

    @wire(getRelatedListRecords, {
        parentRecordId: '$recordId',
        relatedListId: 'Claim_Line_Items__r',
        fields: ['Claim_Line_Item__c.Id', 'Claim_Line_Item__c.Name', 'Claim_Line_Item__c.Amount__c']
    })
    lineItems;

    get claimNumber() { return getFieldValue(this.claim.data, NAME_FIELD); }
    get claimStatus() { return getFieldValue(this.claim.data, STATUS_FIELD); }
    get claimAmount() { return getFieldValue(this.claim.data, AMOUNT_FIELD); }

    get lineItemsData() {
        if (this.lineItems.data) {
            return this.lineItems.data.records.map(record => ({
                Id: record.id,
                Name: record.fields.Name.value,
                Amount: record.fields.Amount__c.value
            }));
        }
        return null;
    }

    handleApprove() {
        const fields = {};
        fields[ID_FIELD.fieldApiName] = this.recordId;
        fields[STATUS_FIELD.fieldApiName] = 'Approved';
        
        updateRecord({ fields })
            .then(() => {
                this.showToast('Success', 'Claim Approved', 'success');
            })
            .catch(error => {
                this.showToast('Error updating record', error.body.message, 'error');
            });
    }

    handleRowAction(event) {
        if (event.detail.action.name === 'delete') {
            const rowId = event.detail.row.Id;
            deleteRecord(rowId)
                .then(() => {
                    this.showToast('Success', 'Line item deleted', 'success');
                    // UI API automatically removes it from related list cache
                })
                .catch(error => {
                    this.showToast('Error deleting record', error.body.message, 'error');
                });
        }
    }

    showToast(title, message, variant) {
        this.dispatchEvent(new ShowToastEvent({ title, message, variant }));
    }
}
```

**XML (warrantyClaimManager.js-meta.xml)**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="[http://soap.sforce.com/2006/04/metadata](http://soap.sforce.com/2006/04/metadata)">
    <apiVersion>59.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__RecordPage</target>
    </targets>
</LightningComponentBundle>
```

**When Apex Becomes Necessary:** If approving the claim required an HTTP callout to an SAP ERP system to validate part numbers before setting the status to "Approved," `updateRecord` would be insufficient. We would transition `handleApprove` to call Imperative Apex, process the callout, and then invoke `notifyRecordUpdateAvailable` upon success.

# 54. LDS Decision-Making Guide

```text
Need Salesforce Data?
        ↓
Can LDS/UI API handle it? (Single record CRUD, Metadata, Related Lists)
        ↓
      YES
        ↓
       Use LDS

      NO (Needs custom SOQL, Aggregations, Callouts, Bulk processing)
        ↓
Need custom SOQL/business logic?
        ↓
       YES
        ↓
     Use Apex
```

# 55. Interview Questions & Answers

### Beginner Questions
*   **What is Lightning Data Service?**
    It's the client-side data layer for LWC that caches and provisions Salesforce data securely via the UI API without requiring Apex.
*   **Why is LDS used in LWC?**
    To eliminate boilerplate Apex code, improve performance through client-side caching, and ensure FLS/CRUD are strictly enforced.

### Intermediate Questions
*   **What is the relationship between LDS and UI API?**
    UI API is the server-side REST endpoint; LDS is the client-side framework that consumes it.
*   **What is getFieldValue?**
    A utility function to safely traverse the complex JSON structure returned by LDS `getRecord` without throwing undefined errors.
*   **How does LDS caching work?**
    LDS stores fetched records in the browser. If another component requests the same record ID, it is served from the cache, preventing duplicate API calls.

### Advanced Questions
*   **What is the difference between refreshApex and notifyRecordUpdateAvailable?**
    `refreshApex` refreshes a specific variable populated by an Apex wire adapter. `notifyRecordUpdateAvailable` clears the LDS cache for a specific record ID, forcing LDS wired properties to fetch fresh data from the server.
*   **What is RefreshView API?**
    A modern standard to trigger a refresh of the entire view hierarchy/component tree in a Lightning Page, rather than targeting specific wired variables.
*   **Does LDS bypass CRUD/FLS?**
    No, it strictly enforces it based on the running user. Fields without FLS visibility are stripped from the response.

### Architect-Level Questions
*   **When should Apex be used instead of LDS?**
    For complex SOQL (deep relationships, aggregates), complex business logic, transactional control across multiple objects, or external system callouts.
*   **What happens when multiple components access the same record?**
    LDS consolidates the request, hits the UI API once, and provisions the shared cached data to all components. Updates from one component automatically re-render the others.

# 56. Revision Summary

*   **Lightning Data Service:** Client-side cache framework for Salesforce data.
*   **UI API:** The underlying REST API.
*   **Wire Service:** The LWC reactivity mechanism.
*   **LDS Wire Adapters:** `getRecord`, `getObjectInfo`, `getPicklistValues`, `getRelatedListRecords`.
*   **CRUD:** `createRecord`, `updateRecord`, `deleteRecord`.
*   **Forms:** `lightning-record-form` (basic), `lightning-record-edit-form` (custom edit), `lightning-record-view-form` (custom read-only).
*   **Cache:** Shared across the page; vastly improves performance.
*   **Security:** CRUD and FLS are native and guaranteed.
*   **Refresh Methods:** Use `notifyRecordUpdateAvailable` to sync external/Apex changes back to LDS. Use `RefreshView` for page-level sync. Use `refreshApex` strictly for wired Apex.
*   **Best Practice:** Prefer LDS for single-record UI logic; reserve Custom Apex for complex queries, transactions, and integrations.