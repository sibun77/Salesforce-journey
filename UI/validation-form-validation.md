# Validation – Form Validation

# 1. Introduction

Form validation in Salesforce Lightning Web Components (LWC) is the process of ensuring that user input meets specific business rules and data integrity requirements before it is saved to the database.

Form validation is critical because it prevents invalid, incomplete, or malicious data from entering your Salesforce org. When dealing with enterprise applications, data quality directly impacts business operations. 

There is a distinct difference between **validation** and **error handling**:
*   **Validation** proactively checks data correctness (e.g., ensuring a "Warranty Claim Amount" is greater than zero).
*   **Error Handling** gracefully manages exceptions when something goes wrong (e.g., catching a server timeout when submitting the claim).

A good validation User Experience (UX) guides the user to correct their input without frustration. Poor validation UX (like hidden errors or confusing technical jargon) leads to high abandonment rates and data entry mistakes.

**Enterprise Salesforce Examples:**
*   **Warranty Claim Form:** Claim Amount cannot exceed the approved limit.
*   **Work Order Form:** End Date must be after the Start Date.
*   **Vehicle Registration:** VIN must be exactly 17 characters.
*   **Customer Information:** Phone number or Email must be provided.
*   **Spare Parts Order:** Quantity must be an integer greater than 0.
*   **Invoice Form:** Discount cannot exceed 15%.

---

# 2. What is Form Validation?

**Definition:** Form validation is a series of checks performed on data entered into a web form to ensure it conforms to expected formats, types, and business rules.

**Validation Process:**
1.  User enters data.
2.  Client-side logic evaluates the input.
3.  If valid, data is submitted to the server.
4.  Server-side logic re-evaluates the input (for security and complex business rules).
5.  If valid, data is saved.

**Valid vs Invalid Input:**
*   **Valid:** Matches all predefined rules (e.g., `email@example.com` for an email field).
*   **Invalid:** Breaks one or more rules (e.g., text in a number field, missing required fields).

**Client-Side Validation:** Happens in the user's browser using HTML attributes or JavaScript. It provides immediate feedback.
**Server-Side Validation:** Happens on the Salesforce server using Apex, Validation Rules, or database constraints. It acts as the final gatekeeper.
**Salesforce Platform Validation:** Built-in platform checks (e.g., checking if a required field is populated before a DML operation).

**Flow Diagram:**
```text
User Input
   ↓
Client-Side Validation
   ↓
Valid? ── No ──> Show Error on UI
   |
  Yes
   ↓
Submit Data
   ↓
Server-Side Validation (Validation Rules / Apex)
   ↓
Valid? ── No ──> Return Error to UI
   |
  Yes
   ↓
Save Record
```

---

# 3. Client-Side vs Server-Side Validation

| Feature | Client-Side Validation | Server-Side Validation |
| :--- | :--- | :--- |
| **Purpose** | Immediate UX feedback, guide the user, reduce server calls. | Ensure data integrity, enforce complex business logic, security. |
| **Execution Location** | User's Web Browser (JavaScript/HTML5). | Salesforce Server (Apex, Database, Flow). |
| **Speed** | Instantaneous. | Requires network round-trip. |
| **User Experience** | Excellent (real-time feedback). | Slower (feedback appears after submission attempt). |
| **Security** | Low (can be bypassed by malicious users or browser dev tools). | High (cannot be bypassed). |
| **Salesforce Validation Rules** | N/A (Server-side concept). | Fully enforced automatically during DML. |
| **Apex Validation** | N/A. | Enforced in Triggers or Apex Controllers via `addError()`. |
| **LWC Validation** | Uses `lightning-input`, `checkValidity()`, custom JS. | Handled by catching `AuraHandledException` from server. |

**Important Distinction:**
Client-side validation improves user experience but must **NOT** be treated as a security mechanism. Server-side validation is **required** to protect data integrity because client-side checks can easily be bypassed by API calls or browser manipulation.

---

# 4. Required Fields

Required field validation ensures a user cannot submit a form without providing essential information.

*   **HTML `required` attribute:** Standard HTML5 validation.
*   **`lightning-input required`:** Base component property that enforces a value and adds a red asterisk (*) to the label.
*   **`lightning-combobox required`:** Ensures an option is selected from the dropdown.
*   **`lightning-textarea required`:** Ensures the text area is not empty.
*   **Salesforce field-level requiredness:** Fields marked required at the object level will inherently throw errors if saved empty, regardless of UI setup.

**Example:**
```html
<lightning-input
    label="Customer Name"
    required
    value={customerName}>
</lightning-input>
```

**Line-by-line Explanation:**
*   `<lightning-input`: Opens the base LWC component tag for an input field.
*   `label="Customer Name"`: Sets the visible label above the field.
*   `required`: Marks the field as mandatory, adding the red asterisk and enabling default "Complete this field" validation.
*   `value={customerName}`: Binds the input's value to the `customerName` JS variable.
*   `></lightning-input>`: Closes the component tag.

---

# 5. lightning-input Validation

`lightning-input` comes with extensive built-in validation based on the HTML5 specification.

*   **required**: Cannot be blank.
*   **type**: Defines the expected format (e.g., `email`, `number`, `date`, `url`).
*   **min**: Minimum allowed numeric value or date.
*   **max**: Maximum allowed numeric value or date.
*   **min-length**: Minimum number of characters.
*   **max-length**: Maximum number of characters.
*   **pattern**: A Regular Expression (regex) the value must match.
*   **step**: Specifies legal number intervals.

**Example:**
```html
<lightning-input 
    type="number" 
    label="Quantity" 
    min="1" 
    max="100" 
    message-when-range-underflow="Minimum quantity is 1."
    message-when-range-overflow="Maximum quantity is 100.">
</lightning-input>
```

The component performs built-in validation automatically when the user interacts with it (on blur/change), displaying standard or custom messages without needing complex JavaScript.

---

# 6. checkValidity()

`checkValidity()` is a standard HTML5 API method exposed by LWC base components.

*   **What it does:** Evaluates whether the current value of the input meets all defined constraints (required, min, max, pattern, etc.).
*   **Return value:** Returns `true` if the input is valid, or `false` if it is invalid.
*   **When to use it:** Use it when you need to know the state of a field *without* necessarily showing an error message on the screen yet (e.g., enabling/disabling a button silently).
*   **Valid vs invalid state:** Represents the underlying boolean state of the component's data integrity.

**Example:**
```javascript
const input = this.template.querySelector('lightning-input');

if (!input.checkValidity()) {
    // Invalid: Execute logic, maybe disable submit button
}
```

**Line-by-line Explanation:**
*   `const input = this.template.querySelector('lightning-input');`: Finds the first `lightning-input` element in the DOM and assigns it to `input`.
*   `if (!input.checkValidity()) {`: Calls `checkValidity()`. The `!` negates it, meaning "If the input is NOT valid".
*   `// Invalid`: Placeholder for code to run if the validation fails.
*   `}`: Closes the if-block.

---

# 7. reportValidity()

`reportValidity()` goes one step further than `checkValidity()`.

*   **What it does:** Checks if the input is valid. If it is *invalid*, it forces the component to display the appropriate error message on the UI immediately.
*   **Difference from checkValidity():** `checkValidity()` only returns a boolean silently. `reportValidity()` returns the boolean AND updates the UI to show the red text/border.
*   **Showing validation messages:** Essential for highlighting exactly which fields the user missed upon clicking "Submit".
*   **User experience:** Provides immediate visual feedback to the user.

**Example:**
```javascript
input.reportValidity();
```

| Method | Purpose |
|--------|---------|
| `checkValidity()` | Checks validity (returns true/false silently) |
| `reportValidity()` | Checks validity and displays the error on the UI if invalid |

---

# 8. setCustomValidity()

`setCustomValidity()` allows you to define custom, complex business rules that standard HTML attributes cannot handle.

*   **Custom error messages:** Overrides standard validation with a message of your choosing.
*   **Clearing custom errors:** You must pass an empty string `''` to clear the error state. If you don't, the field remains permanently invalid.
*   **Business-specific validation:** e.g., "Claim amount cannot exceed warranty limit".

**Example:**
```javascript
input.setCustomValidity('Claim amount exceeds the allowed limit.');
input.reportValidity();
```

**Clearing the error:**
```javascript
input.setCustomValidity(''); // Resets the custom error state
input.reportValidity();      // Updates the UI to remove the error message
```

---

# 9. Custom Validation

Standard validation (`required`, `max`) isn't enough for dynamic business logic. Custom validation is required when field validity depends on complex calculations, asynchronous checks, or cross-field dependencies.

**Examples:**
*   Claim amount must not exceed a dynamically calculated limit.
*   End date must be after start date.
*   Quantity must be greater than zero.
*   Discount cannot exceed allowed percentage for a specific Tier.
*   Vehicle must be selected before submitting.

**Complete Example:**
```javascript
validateDiscount(event) {
    const inputField = event.target;
    const val = inputField.value;
    
    if (val > this.maxAllowedDiscount) {
        inputField.setCustomValidity(`Discount cannot exceed ${this.maxAllowedDiscount}%`);
    } else {
        inputField.setCustomValidity('');
    }
    inputField.reportValidity();
}
```

---

# 10. Validating Multiple Fields

To validate an entire form before submission, you must iterate over all input fields.

**Example:**
```javascript
const inputs = [...this.template.querySelectorAll('lightning-input')];

const allValid = inputs.reduce((valid, input) => {
    input.reportValidity();
    return valid && input.checkValidity();
}, true);
```

**Line-by-line Explanation:**
*   `const inputs = [...this.template.querySelectorAll('lightning-input')];`: Selects all `lightning-input` elements and uses the spread operator `...` to convert the NodeList into a true JavaScript Array.
*   `const allValid = inputs.reduce((valid, input) => {`: Uses the `reduce` array method to iterate through every input, keeping a running boolean tally (`valid`).
*   `input.reportValidity();`: Forces the current input in the loop to show an error on the UI if it is invalid.
*   `return valid && input.checkValidity();`: Returns `true` only if the running tally (`valid`) is true AND the current input is valid.
*   `}, true);`: Initializes the `reduce` loop with a starting value of `true`.

**Simpler Approach for Beginners:**
```javascript
let isFormValid = true;
const inputs = this.template.querySelectorAll('lightning-input');

inputs.forEach(input => {
    if (!input.checkValidity()) {
        input.reportValidity();
        isFormValid = false;
    }
});
```

---

# 11. Form Submission Validation

Before processing data or making Apex calls, you must intercept the submit action and run validations.

*   **Submit button:** Triggers the action.
*   **handleSubmit():** The JS method invoked by the button.
*   **preventDefault():** Stops the default HTML form submission behavior (if inside a `<form>`).
*   **Validate fields:** Run multiple-field validation.
*   **Submit only when valid:** Proceed to Apex or API call only if `isFormValid` is true.

**Example:**
```javascript
handleSubmit(event) {
    // Prevent default browser refresh/submission
    event.preventDefault();

    // Validate fields (using a helper method)
    const isValid = this.validateAllFields();

    if (isValid) {
        // Submit data to Apex
        this.saveRecord();
    }
}
```

**Line-by-line Explanation:**
*   `handleSubmit(event) {`: The event handler triggered by the Submit button.
*   `event.preventDefault();`: Stops the browser from executing default form actions.
*   `const isValid = this.validateAllFields();`: Calls a custom method that runs `checkValidity()`/`reportValidity()` and returns a boolean.
*   `if (isValid) {`: Checks if the validation passed.
*   `this.saveRecord();`: Custom method to actually execute the server save logic.
*   `}`: Closes the block.

---

# 12. lightning-record-edit-form Validation

`lightning-record-edit-form` heavily automates validation by reading object metadata.

*   **lightning-input-field:** Inherits metadata (required status, field type constraints).
*   **required fields:** Automatically enforced by the form.
*   **submit:** Triggers built-in validation before firing the `onsubmit` event.
*   **success / error:** Fires `onsuccess` if save works, or `onerror` if Salesforce server validation fails (e.g., Validation Rules).
*   **validation messages:** Automatically handled and displayed below the respective fields or at the top of the form.

**Complete Example:**
```html
<lightning-record-edit-form 
    object-api-name="WorkOrder" 
    onsubmit={handleSubmit} 
    onsuccess={handleSuccess}
    onerror={handleError}>
    
    <lightning-messages></lightning-messages>
    
    <lightning-input-field field-name="Subject" required></lightning-input-field>
    <lightning-input-field field-name="Status"></lightning-input-field>
    
    <lightning-button type="submit" label="Save"></lightning-button>
</lightning-record-edit-form>
```

**Salesforce handling:** It automatically parses Validation Rule errors from the server and injects them into the `<lightning-messages>` component or attaches them to specific `lightning-input-field` elements.

---

# 13. lightning-record-form Validation

`lightning-record-form` is the most abstracted UI component.

*   **Create / Edit / View:** Handles all modes automatically.
*   **Built-in validation:** Reads all metadata. If a field is required in Salesforce setup, it is automatically required on the UI.
*   **Standard Salesforce validation:** Handles Validation Rules seamlessly without custom JS.

**Comparison:**
| Feature | `lightning-record-form` | `lightning-record-edit-form` |
| :--- | :--- | :--- |
| **Custom Layout** | Minimal (Uses page layout or lists fields). | High (Can place fields anywhere, use Grid). |
| **Validation Handling** | Completely automatic. | Mostly automatic, allows interception in `onsubmit`. |

---

# 14. lightning-input-field Validation

Used exclusively inside `lightning-record-edit-form`.

*   **Required fields:** Can be forced via `required` attribute, or inherited from database schema.
*   **Field metadata:** Automatically renders the correct UI (e.g., date picker for Dates, toggle for Checkboxes).
*   **Picklists / Lookups:** Automatically pulls active picklist values and provides standard lookup search capabilities.
*   **Automatic Validation:** It restricts text length, ensures valid dates, validates number formats, and enforces currency decimals automatically based on the Salesforce Field Definition.

---

# 15. Salesforce Validation Rules

Validation Rules live on the server and evaluate formulas during the save process. 

*   **Formula-based validation:** e.g., `Claim_Amount__c > Approved_Limit__c`.
*   **Record save prevention:** Rolls back the transaction if the formula returns True.
*   **Error messages:** Defined by the admin.
*   **Field-level / Object-level:** Can display next to a specific field or at the top of the page.

**Example Scenario:**
Claim amount cannot exceed approved warranty limit.

**The Relationship Flow:**
```text
LWC (User clicks Save)
 ↓
Save (API Call to server)
 ↓
Validation Rule (Evaluates Server-Side)
 ↓
Pass / Error (If error, throws AuraHandledException to LWC)
```

---

# 16. Apex Validation

Apex validation is needed for rules too complex for formula fields (e.g., querying related records, cross-object logic).

*   **addError():** Used in Triggers to prevent DML.
*   **AuraHandledException:** Used in `@AuraEnabled` methods to return clean, catchable errors to the LWC.

**Realistic Example:**
```java
@AuraEnabled
public static void processClaim(Id vehicleId, Decimal claimAmount) {
    Vehicle__c veh = [SELECT Warranty_Limit__c FROM Vehicle__c WHERE Id = :vehicleId LIMIT 1];
    if (claimAmount > veh.Warranty_Limit__c) {
        throw new AuraHandledException('Claim amount exceeds vehicle warranty limit.');
    }
    // Proceed with save
}
```

Apex validation is crucial because API integrations (MuleSoft, Postman) bypass UI components. Client-side validation is just for UX; Apex ensures the data is strictly correct at the database level.

---

# 17. Cross-Field Validation

Validating dependencies between two or more fields.

**Examples:**
*   Start Date < End Date
*   Quantity × Price must be valid

**Complete LWC Example:**
```javascript
validateDates() {
    const startDateField = this.template.querySelector('.start-date');
    const endDateField = this.template.querySelector('.end-date');
    
    const start = new Date(startDateField.value);
    const end = new Date(endDateField.value);
    
    if (start && end && end <= start) {
        endDateField.setCustomValidity('End Date must be after Start Date.');
    } else {
        endDateField.setCustomValidity(''); // Clear error
    }
    endDateField.reportValidity();
}
```

---

# 18. Conditional Validation

Fields that are required only based on the value of another field.

**Example:**
*   If Claim Type = "Warranty", Warranty Number is required.

**LWC Implementation:**
```javascript
handleTypeChange(event) {
    this.claimType = event.detail.value;
    const warrantyInput = this.template.querySelector('.warranty-number');
    
    if (this.claimType === 'Warranty') {
        warrantyInput.required = true;
    } else {
        warrantyInput.required = false;
        warrantyInput.setCustomValidity(''); // Clear lingering errors
        warrantyInput.reportValidity();
    }
}
```

*Server-Side Enforcement:* A Salesforce Validation Rule should also exist:
`AND(ISPICKVAL(Claim_Type__c, 'Warranty'), ISBLANK(Warranty_Number__c))`

---

# 19. Validation with Picklists and Comboboxes

Validation for `<lightning-combobox>`.

*   **required:** Enforces selection.
*   **value:** Binds to selection.
*   **empty selection:** Usually represented by lack of value.

**Complete Example:**
```html
<lightning-combobox
    name="vehicleType"
    label="Vehicle Type"
    value={selectedVehicle}
    placeholder="Select a Vehicle"
    options={vehicleOptions}
    required
    onchange={handleVehicleChange}
    message-when-value-missing="Please select a vehicle type to proceed.">
</lightning-combobox>
```

---

# 20. Validation with Textarea

Validation for `<lightning-textarea>`.

*   **minlength / maxlength:** Restricts character count.

**Example:**
```html
<lightning-textarea 
    label="Damage Description" 
    required 
    minlength="10" 
    maxlength="500" 
    message-when-too-short="Please provide at least 10 characters detailing the damage.">
</lightning-textarea>
```

---

# 21. Date and Time Validation

Validating dates.

**LWC Example (No past dates):**
```html
<lightning-input 
    type="date" 
    label="Service Date" 
    min={todayDate} 
    message-when-range-underflow="Service date cannot be in the past.">
</lightning-input>
```

Combining JS and Validation Rules ensures that if a user bypasses the UI, the database still blocks past dates via a rule like `Service_Date__c < TODAY()`.

---

# 22. Number and Currency Validation

Controlling numerical inputs.

**Examples:**
*   Quantity > 0 (`min="1" step="1"`)
*   Discount between 0 and 100 (`min="0" max="100"`)

**Example:**
```html
<lightning-input 
    type="currency" 
    label="Claim Amount" 
    min="0.01" 
    message-when-range-underflow="Claim amount must be greater than zero.">
</lightning-input>
```

---

# 23. Email and Phone Validation

Using standard inputs vs custom regex.

*   **Standard:** `type="email"` or `type="tel"` utilizes browser standards.
*   **Pattern (Regex):** `<lightning-input type="tel" pattern="[0-9]{10}">` enforces exact formats.

*Warning:* Use regex carefully. Overly strict regex can frustrate users (e.g., blocking valid international numbers).

---

# 24. Validation with Iteration

Validating dynamically generated fields in lists, like Claim Line Items.

**Approach:**
1.  Use `for:each` to generate rows.
2.  Assign a `data-id` or `data-index` to identify the row.
3.  Query all inputs, validate collectively.

```html
<template for:each={lineItems} for:item="item" for:index="index">
    <div key={item.id}>
        <lightning-input data-index={index} value={item.quantity} required></lightning-input>
    </div>
</template>
```

```javascript
validateRows() {
    const inputs = this.template.querySelectorAll('lightning-input');
    let isValid = true;
    inputs.forEach(input => {
        if(!input.checkValidity()) {
            input.reportValidity();
            isValid = false;
        }
    });
    return isValid;
}
```

---

# 25. Validation in Custom LWC Components

When building complex UIs, parent components often hold multiple custom child form components.

*   **@api methods:** Child exposes a `@api validate()` method.
*   **Parent calling child:** Parent calls `child.validate()` before submitting.

**Example:**
*Child Component (childForm.js)*
```javascript
@api
validate() {
    const input = this.template.querySelector('lightning-input');
    input.reportValidity();
    return input.checkValidity();
}
```

*Parent Component (parentForm.js)*
```javascript
handleSave() {
    const child = this.template.querySelector('c-child-form');
    if (child.validate()) {
        // Proceed with save
    }
}
```

---

# 26. Async Validation

Sometimes validation requires querying the server (e.g., Check duplicate VIN).

*   **Apex Callout:** Call an `@AuraEnabled` method.
*   **Loading state:** Show a spinner while checking.
*   **Use sparingly:** Network calls delay user experience.

```javascript
async handleVINBlur(event) {
    this.isLoading = true;
    const vin = event.target.value;
    try {
        const isDuplicate = await checkDuplicateVIN({ vinCode: vin });
        if (isDuplicate) {
            event.target.setCustomValidity('This VIN is already registered.');
        } else {
            event.target.setCustomValidity('');
        }
        event.target.reportValidity();
    } catch(error) {
        // Handle error
    } finally {
        this.isLoading = false;
    }
}
```

---

# 27. Error Handling

Present errors clearly based on origin:

*   **Field-level errors:** Use `reportValidity()` (best for UI validation).
*   **Form-level errors:** Use `<lightning-messages>` (best for Validation Rules).
*   **Toast messages:** Use `ShowToastEvent` for system errors, Apex exceptions, or network failures.

---

# 28. Validation UX Best Practices

*   **Validate at the right time:** On `blur` (leaving field) or `submit`. Not aggressively on every keystroke (`keyup`) which frustrates users.
*   **Show errors near the relevant field:** Context matters.
*   **Use meaningful error messages:** "Invalid format" is bad. "Please enter a 17-character VIN" is good.
*   **Preserve entered values:** Never clear a form just because validation failed.
*   **Clear errors when corrected:** Re-evaluate dynamically.

---

# 29. Accessibility

Accessibility (a11y) ensures users with disabilities can interact with validation.

*   **Labels & Required indicators:** Base components handle this for Screen Readers.
*   **Color contrast:** Do not rely *only* on red text. The error message text itself must explain the problem explicitly.
*   **Focus management:** If a form is long and submission fails, programmatically set focus to the first invalid field so screen readers announce it.

---

# 30. Validation vs Security

*   **Validation:** "Is this data formatted correctly for my business rules?" (UX focus).
*   **Security:** "Is this user authorized to modify this data, and is the data safe?" (Integrity/Access focus).

Client-side JavaScript can easily be bypassed via Chrome DevTools or direct API calls. Apex logic, FLS (Field Level Security), and Validation Rules enforce actual data security.

---

# 31. Validation Order in Salesforce

Simplified conceptual flow when saving from LWC:

1.  **LWC Client-Side:** JS validation (`checkValidity`).
2.  **Lightning Data Service (if used):** Basic type/required checks.
3.  **System Validation:** Required fields, field formats.
4.  **Before Triggers:** Apex validation / overrides.
5.  **Custom Validation Rules:** Evaluated.
6.  **After Triggers:** Post-save logic.
7.  **Database Commit:** Record saved.

---

# 32. Common Mistakes

| Mistake | Cause | Solution |
| :--- | :--- | :--- |
| **Only validating on client** | Believing UI is secure. | Enforce critical rules in Apex/Validation Rules. |
| **Not clearing custom validity** | Forgetting `setCustomValidity('')`. | Always clear the string if conditions are met. |
| **Incorrect `querySelector`** | Grabbing wrong element. | Use specific classes or `data-id`. |
| **Duplicating business rules** | Coding exact same complex logic in JS and Apex. | Rely on LDS/Record forms to bubble server rules to UI where possible to reduce maintenance. |

---

# 33. Debugging Validation

**Checklist:**
1.  Did `checkValidity()` return false? (Check HTML constraints).
2.  Did you call `reportValidity()`? (Message won't show otherwise).
3.  Are you clearing `setCustomValidity()`?
4.  Is a Validation Rule blocking it silently? (Check Network Tab for API response).
5.  Is the DOM rendering properly? (Ensure `lightning-input` elements aren't hidden).

---

# 34. Best Practices Checklist

*   ✅ **Use Base Component validation whenever possible:** Less code, highly accessible.
*   ✅ **Use checkValidity():** To programmatically gate logic.
*   ✅ **Use reportValidity():** To show errors.
*   ✅ **Use setCustomValidity():** For custom business logic.
*   ✅ **Clear custom validity:** Crucial to unblock users.
*   ✅ **Enforce important rules server-side:** Always protect the database.

---

# 35. Real Project Scenarios (Automotive CRM)

1.  **Warranty Claim Form:** Requires custom validation to ensure `Claim Amount` doesn't exceed `Approved_Limit__c` (Server-side + UI).
2.  **Spare Parts Order:** Uses `type="number" min="1"` to prevent ordering 0 parts.
3.  **Vehicle Registration:** Custom RegEx for exactly 17 alphanumeric VIN characters.

---

# 36. Complete End-to-End Example

**Scenario:** Warranty Claim Form

**HTML:**
```html
<template>
    <lightning-card title="Warranty Claim Form">
        <div class="slds-p-around_medium">
            <lightning-input class="claim-input" label="Customer Name" value={customerName} required></lightning-input>
            
            <lightning-input class="claim-input" type="number" label="Claim Amount" value={claimAmount} min="1" required></lightning-input>
            
            <lightning-input class="claim-input claim-date" type="date" label="Claim Date" value={claimDate} required></lightning-input>
            
            <lightning-button label="Submit Claim" variant="brand" onclick={handleSubmit}></lightning-button>
        </div>
    </lightning-card>
</template>
```

**JavaScript:**
```javascript
import { LightningElement, track } from 'lwc';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

export default class WarrantyClaimForm extends LightningElement {
    @track customerName;
    @track claimAmount;
    @track claimDate;

    handleSubmit(event) {
        // 1. Cross-field / Custom Validation checks before standard validity
        this.validateDate();

        // 2. Collect all inputs
        const inputs = [...this.template.querySelectorAll('.claim-input')];

        // 3. Check and Report Validity
        const isValid = inputs.reduce((valid, input) => {
            input.reportValidity();
            return valid && input.checkValidity();
        }, true);

        // 4. Submit if valid
        if (isValid) {
            this.processSubmit();
        } else {
            this.dispatchEvent(new ShowToastEvent({
                title: 'Error',
                message: 'Please review the errors on the form.',
                variant: 'error'
            }));
        }
    }

    validateDate() {
        const dateInput = this.template.querySelector('.claim-date');
        const selectedDate = new Date(dateInput.value);
        const today = new Date();
        
        // Cannot claim in the future
        if (selectedDate > today) {
            dateInput.setCustomValidity('Claim date cannot be in the future.');
        } else {
            dateInput.setCustomValidity(''); // Must clear
        }
    }

    processSubmit() {
        // Apex callout here
        this.dispatchEvent(new ShowToastEvent({
            title: 'Success',
            message: 'Claim Submitted.',
            variant: 'success'
        }));
    }
}
```

**Explanation:**
The `handleSubmit` function halts standard actions, runs a custom date validation, iterates over all fields to check and report validity, and only proceeds to `processSubmit` if the entire form resolves as valid.

---

# 37. Interview Questions & Answers

### Beginner Questions
**Q: What is form validation in LWC?**
A: It is the process of checking user input against specific rules (like required fields or formats) before submitting data to the server.

### Intermediate Questions
**Q: What is the difference between `checkValidity()` and `reportValidity()`?**
A: `checkValidity()` checks if the field is valid and returns a boolean silently. `reportValidity()` checks validity, returns a boolean, AND displays the error message on the UI.

**Q: How do you validate multiple `lightning-input` components?**
A: By using `querySelectorAll('lightning-input')`, converting it to an array, and using `reduce` or `forEach` to call `checkValidity()` and `reportValidity()` on each.

### Advanced Questions
**Q: What is `setCustomValidity()` and what is the most common mistake when using it?**
A: It allows setting a custom validation error message. The most common mistake is forgetting to clear it by passing an empty string (`''`) once the data is corrected, leaving the form permanently invalid.

**Q: How do you handle asynchronous validation?**
A: By calling an Apex method imperatively on `blur` or before submit, awaiting the response, and using `setCustomValidity()` based on the server response (e.g., checking if a VIN is a duplicate).

### Architect-Level Questions
**Q: Can client-side validation be considered security?**
A: Absolutely not. Client-side validation exists purely for User Experience. Malicious actors or poorly written integrations can bypass UI components entirely via the API. True security and data integrity must be enforced at the server level via Validation Rules, Triggers, and FLS.

---

# 38. Revision Summary

*   **Form Validation:** Crucial for UX and data integrity.
*   **Client vs Server:** UI validation (JS/HTML) for fast feedback; Server validation (Apex/Rules) for absolute data integrity.
*   **Validity APIs:** `checkValidity()` (silently check), `reportValidity()` (check and show error), `setCustomValidity()` (custom error logic).
*   **Record Forms:** `lightning-record-form` and `lightning-record-edit-form` automate validation based on schema metadata.
*   **Advanced Validations:** Handled in JS for Conditional (if X then Y) and Cross-field (Start Date < End Date) requirements.
*   **Apex & Async:** Imperative Apex calls handle heavy server-side checks. Always show loaders to maintain UX.
*   **Best Practices:** Clear custom validity, rely on HTML attributes when possible, use Toast events for form-level feedback, and always prioritize accessibility.