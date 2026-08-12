# Salesforce Lightning Design System (SLDS)

## 1. Introduction

The **Salesforce Lightning Design System (SLDS)** is a CSS framework and design system created by Salesforce. It provides a comprehensive library of design patterns, components, and guidelines that enable developers to build pixel-perfect, scalable, and accessible enterprise user interfaces.

### Why SLDS Exists
Before SLDS, developers relied on custom CSS or third-party frameworks like Bootstrap, leading to fragmented, inconsistent user interfaces that didn't look or behave like native Salesforce. SLDS solves this by providing a unified styling language. 

### Importance of Consistent UI
In enterprise systems like an **Automotive CRM**, users transition between standard Salesforce pages (like Accounts) and custom LWC screens (like a Warranty Claim Processor). A consistent UI ensures a seamless user experience, reducing training time and cognitive load. 

### Relationship Between SLDS, Lightning Experience, and LWC
- **Lightning Experience:** The modern Salesforce user interface.
- **SLDS:** The CSS framework and design rules that define the look and feel of Lightning Experience.
- **LWC (Lightning Web Components):** The modern JavaScript framework used to build components. LWC comes with SLDS pre-loaded, meaning developers can use SLDS classes directly in their HTML templates without importing external stylesheets.

---

## 2. What is SLDS?

SLDS is more than just a CSS library; it is a complete **Design System**. 

### Deep Dive:
- **UI Consistency:** Ensures that a button in your custom Work Order component looks exactly like a button on a standard Lead record.
- **Reusable Design Patterns:** Provides blueprints for common UI paradigms (e.g., Modals, Page Headers).
- **Accessibility (a11y):** Built from the ground up to support W3C accessibility guidelines (ARIA, screen readers, keyboard navigation).
- **Responsive Design:** Includes a robust, mobile-first flexbox grid system.
- **Enterprise UI Standards:** Tailored for complex data-dense enterprise applications, not just simple marketing websites.

By using SLDS, developers save hundreds of hours of CSS development, avoid cross-browser compatibility issues, and ensure future-proofing as Salesforce updates its core UI.

---

## 3. SLDS Architecture

SLDS is composed of several interdependent layers that work together to create the final UI.

```text
SLDS Architecture
 |
 +-- Design Tokens / Styling Hooks (Variables for colors, spacing, typography)
 |
 +-- Utility Classes (Single-purpose CSS classes like margin or text alignment)
 |
 +-- Component Blueprints (HTML/CSS structures for complex UI like Modals)
 |
 +-- Icons (SVG icons for actions, standard objects, utilities)
 |
 +-- Typography (Standardized font stacks, weights, sizes)
 |
 +-- Layout / Grid (Flexbox-based structural system)
 |
 +-- Accessibility Guidelines (ARIA states, focus management)
```

---

## 4. SLDS vs Base Lightning Components

While both help you build Salesforce UIs, they serve different purposes. 

| Feature | SLDS | Lightning Base Components (`<lightning-*>`) | Custom LWC |
|---------|------|---------------------------------------------|------------|
| **Definition** | A CSS framework & design system. | Pre-built LWC components provided by Salesforce. | Components built by developers. |
| **Contents** | CSS classes, design tokens, SVGs, blueprints. | Functional JS, HTML, and encapsulated CSS. | Custom business logic and UI. |
| **Logic** | None (CSS only). | Built-in (e.g., date picker logic, data binding). | Fully custom. |
| **Usage** | `<div class="slds-box">` | `<lightning-card title="Vehicles">` | `<c-vehicle-card>` |

### When to use which?
- **Use Base Components** (`lightning-button`, `lightning-input`, `lightning-card`) whenever possible. They are accessible, fully functional, and automatically updated by Salesforce.
- **Use SLDS** when Base Components don't offer the exact layout or structural flexibility you need, or when you are building a custom container (e.g., using `slds-grid` to arrange base components).
- **Use Custom HTML/CSS** only as a last resort when SLDS absolutely cannot meet the requirement.

---

## 5. Using SLDS in LWC

SLDS is automatically injected into LWC. You simply apply the classes to your HTML elements.

```html
<template>
    <!-- slds-p-around_medium adds medium padding to all sides of the container -->
    <div class="slds-p-around_medium">
        
        <!-- slds-text-heading_medium applies standard heading styling (size, weight) -->
        <!-- slds-m-bottom_medium adds margin to the bottom to push down the next element -->
        <h1 class="slds-text-heading_medium slds-m-bottom_medium">
            Warranty Claim
        </h1>
        
        <p>Claim status details go here.</p>
    </div>
</template>
```

**Line-by-line explanation:**
- `<div class="slds-p-around_medium">`: Creates a container with standardized Salesforce padding, ensuring the content isn't flushed against the edges.
- `<h1 class="...">`: Uses utility classes to format the text as a heading and adds a bottom margin to separate it from the paragraph below.

---

## 6. SLDS Utility Classes

Utility classes are single-purpose CSS classes used to tweak layout and presentation without writing custom CSS.

| Category | Example | Purpose |
|----------|---------|---------|
| Padding | `slds-p-around_medium` | Adds medium padding on all 4 sides. |
| Margin | `slds-m-bottom_large` | Adds large margin to the bottom. |
| Text | `slds-text-align_center` | Centers text horizontally. |
| Visibility | `slds-hide` | Applies `display: none;` to an element. |
| Sizing | `slds-size_1-of-2` | Sets width to 50%. |
| Grid | `slds-grid` | Initiates a flexbox container. |
| Borders | `slds-border_bottom` | Adds a standard bottom border. |

**Practical Example:**
```html
<div class="slds-box slds-theme_default slds-m-top_small">
    <p class="slds-text-color_error slds-text-title_caps">Error processing Warranty</p>
</div>
```

---

## 7. Spacing

SLDS uses a strictly defined spacing scale (xxx-small to xxx-large) applied via Margin (`m`) and Padding (`p`).

**Syntax:** `slds-[m/p]-[direction]_[size]`

**Directions:**
- `top`, `bottom`, `left`, `right`
- `horizontal` (left and right)
- `vertical` (top and bottom)
- `around` (all four sides)

**Examples:**
- `slds-p-around_small`: Small padding everywhere. Used inside cards.
- `slds-m-top_medium`: Pushes an element down from the one above it.
- `slds-p-horizontal_medium`: Ideal for lists to ensure text doesn't touch the borders.

---

## 8. Typography

SLDS ensures a readable, hierarchical text structure.

- **Headings:** `slds-text-heading_large`, `medium`, `small`.
- **Body Text:** Default text size. Use `slds-text-body_small` for secondary text.
- **Alignment:** `slds-text-align_left`, `center`, `right`.
- **Truncation:** `slds-truncate` (adds `...` when text overflows - highly useful in tables).
- **Color:** `slds-text-color_default`, `weak`, `error`, `success`.

```html
<h2 class="slds-text-heading_small slds-truncate" title="VIN: 1234567890">
    VIN: 1234567890
</h2>
<p class="slds-text-color_weak">Last serviced: 12 Oct 2025</p>
```

---

## 9. Colors

SLDS provides semantic color classes, removing the need to hardcode HEX values.

- **Brand:** `slds-text-color_brand` (Standard Salesforce blue).
- **Success:** `slds-text-color_success` (Green - e.g., Claim Approved).
- **Error:** `slds-text-color_error` (Red - e.g., Validation failed).
- **Backgrounds:** `slds-theme_default` (White box), `slds-theme_shade` (Light gray).

Using these classes ensures that if Salesforce updates its color palette, your app updates automatically.

---

## 10. SLDS Grid System

The SLDS grid is based on CSS Flexbox. It allows you to create complex layouts quickly.

```html
<!-- slds-grid creates the flex container -->
<div class="slds-grid slds-gutters">
    
    <!-- slds-col defines a flexible child column -->
    <div class="slds-col slds-size_1-of-2">
        <p>Dealer Name: AutoCorp</p>
    </div>
    
    <!-- slds-size_1-of-2 forces the column to take up 50% width -->
    <div class="slds-col slds-size_1-of-2">
        <p>Dealer Code: AC-908</p>
    </div>
</div>
```

**Key Concepts:**
- `slds-grid`: The wrapper.
- `slds-col`: The columns.
- `slds-gutters`: Adds standardized spacing between columns.
- `slds-wrap`: Allows columns to wrap to the next line if they exceed 100% width.

---

## 11. Responsive Design

SLDS supports responsive layouts via sizing utilities targeting specific screen widths: `_small`, `_medium`, `_large`.

**Automotive Example: Dealer Dashboard**
```html
<div class="slds-grid slds-wrap">
    <!-- Full width on mobile, 50% on tablet, 33% on desktop -->
    <div class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2 slds-large-size_1-of-3">
        <c-warranty-metrics></c-warranty-metrics>
    </div>
    <div class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2 slds-large-size_1-of-3">
        <c-recent-work-orders></c-recent-work-orders>
    </div>
</div>
```
This ensures the Dealer Dashboard remains highly usable whether the service advisor is on an iPad (`medium`) or a desktop (`large`).

---

## 12. SLDS Buttons

**SLDS button classes:** `slds-button`, `slds-button_brand`, `slds-button_neutral`, `slds-button_destructive`.

**Best Practice:** Always prefer `<lightning-button>` over writing custom HTML buttons with SLDS classes. `lightning-button` handles accessibility, disabled states, and events seamlessly.

**Base Component vs SLDS markup:**
```html
<!-- DO THIS (Base Component) -->
<lightning-button variant="brand" label="Submit Claim" onclick={handleSubmit}></lightning-button>

<!-- AVOID THIS (unless you need extreme custom logic inside the button) -->
<button class="slds-button slds-button_brand" onclick={handleSubmit}>Submit Claim</button>
```

---

## 13. SLDS Forms

Forms define inputs, labels, and validations. 

While SLDS provides blueprints for form elements (using `slds-form-element`, `slds-label`, `slds-input`), **you should almost exclusively use `<lightning-input>` and `<lightning-combobox>`**. 

Base components automatically wire up the complex DOM structure needed for accessibility, error handling, and data binding.

```html
<!-- DO THIS -->
<lightning-input label="Vehicle Mileage" type="number" required></lightning-input>
```

---

## 14. SLDS Cards

Cards group related information. 

**Base Component (`<lightning-card>`)** is preferred for standard layouts:
```html
<lightning-card title="Vehicle Details" icon-name="standard:service_territory">
    <lightning-button label="Edit" slot="actions"></lightning-button>
    <p class="slds-p-horizontal_small">VIN: 12345...</p>
</lightning-card>
```

**Use SLDS Custom Cards (`slds-card`)** only when you need a radically different layout (e.g., hiding headers, custom complex footers) that the Base Component slots do not support.

---

## 15. SLDS Tables

For displaying data rows (e.g., Shipment Tracking, Spare Parts).

Use `<lightning-datatable>` whenever you need sorting, inline editing, or row selection. It is highly optimized.

Use raw **SLDS Tables** (`slds-table`, `slds-table_cell-buffer`, `slds-table_bordered`) when you need custom cell rendering that `lightning-datatable` struggles with (like complex nested LWC inside a cell, or highly customized row grouping).

---

## 16. SLDS Icons

Icons provide visual context. 

- **Utility:** Standard UI actions (settings, close, check).
- **Standard:** Standard objects (Account, Case).
- **Custom:** Custom objects.
- **Action:** Quick actions.

**Usage:** Always use `<lightning-icon>` rather than manually embedding SLDS SVG sprites.
```html
<lightning-icon icon-name="utility:check" alternative-text="Approved" variant="success"></lightning-icon>
```

---

## 17. SLDS Modals

Modals overlay the main screen to demand user attention (e.g., "Confirm Work Order Cancellation").

**Modern Approach:** Use the `LightningModal` module available in newer LWC releases. It manages z-index, accessibility, and focus trapping perfectly.

If building a custom modal container for older implementations, you must adhere strictly to the SLDS Modal Blueprint (`slds-modal`, `slds-backdrop`), ensuring you manage the `aria-hidden` and focus states via JavaScript.

---

## 18. SLDS Badges and Pills

**Badges:** Used for read-only status indicators.
```html
<span class="slds-badge slds-theme_success">Claim Approved</span>
```

**Pills:** Used for actionable selections (e.g., removing a selected spare part).
Prefer the `<lightning-pill>` base component to handle the close/remove event natively.

---

## 19. SLDS Notifications

Notifications inform users of system events.
- **Toast:** Pop-up at the top for temporary success/error messages. Use `ShowToastEvent` in LWC JavaScript. Do not build this manually with SLDS HTML.
- **Alerts/Notices:** Banners inside components (e.g., "This vehicle's warranty is expired"). Use SLDS scoped notification blueprints (`slds-scoped-notification`) or `lightning-helptext`.

---

## 20. SLDS Styling Hooks

Styling hooks are CSS Custom Properties (variables) provided by Salesforce to safely customize the look of Base Components without relying on DOM structure (which might change).

Instead of overriding a class like `.slds-button`, you override its variable:

```css
/* In your LWC .css file */
:host {
    --sds-c-button-brand-color-background: #004487; /* Custom Automotive Blue */
    --sds-c-button-brand-color-border: #004487;
}
```
This is the **only supported way** to style Base Lightning Components.

---

## 21. Custom CSS with SLDS

- **When to use:** When SLDS lacks a specific utility (e.g., a highly specific background image or animation).
- **When to avoid:** Never write CSS for margins, padding, colors, or basic typography. Use utility classes instead.

**Bad:**
```css
.my-title { font-size: 18px; margin-bottom: 10px; font-weight: bold; }
```
**Good (No CSS needed):**
```html
<h2 class="slds-text-heading_small slds-m-bottom_small">Title</h2>
```

---

## 22. LWC Shadow DOM and SLDS

LWC uses **Shadow DOM** to encapsulate components. 

```text
Parent Component (c-dealer-dashboard)
 |
 +-- Child Component (c-warranty-card)
      |
      +-- Shadow Boundary (CSS cannot cross this line)
```
- CSS written in `dealerDashboard.css` **will not** leak into or style elements inside `warrantyCard.html`.
- SLDS utility classes applied inside the HTML template of any component work normally because Salesforce globally imports SLDS stylesheets into the shadow roots automatically.

---

## 23. Accessibility (a11y)

Enterprise apps must be usable by everyone. SLDS incorporates web accessibility standards:
- **Semantic HTML:** Using `<button>` instead of `<div>` for clickable items.
- **ARIA Attributes:** `aria-expanded`, `aria-hidden` are predefined in SLDS blueprints.
- **Color Contrast:** SLDS standard colors pass WCAG 2.1 AA contrast requirements.
- **Base Components:** By using `<lightning-input>`, screen readers automatically associate the label with the input field.

---

## 24. SLDS Design Patterns

Salesforce provides documented patterns for common UI scenarios:
- **Page Header:** The standard top block containing a title, record icon, and primary actions.
- **Empty State:** An illustration and text showing no data exists (e.g., "No Spare Parts found").
- **Two-Column Form:** The standard detail page layout for records.

Adhering to these patterns ensures your Automotive CRM feels natively integrated with standard Sales/Service Cloud.

---

## 25. SLDS with Base Lightning Components

Base components are essentially wrappers around SLDS HTML blueprints.

Always prefer:
- `<lightning-input>` over SLDS form classes.
- `<lightning-datatable>` over SLDS table classes.
- `<lightning-spinner>` over manual SLDS spinner SVGs.

Use SLDS utility classes on the outer wrapper of base components to control spacing and layout (e.g., wrapping a `lightning-button` in an `slds-m-left_small` div).

---

## 26. SLDS in Enterprise Salesforce Applications

In a large Automotive CRM, maintaining thousands of lines of custom CSS is a technical debt nightmare.

By enforcing SLDS:
- **Warranty Claim App:** Uses standard grids, meaning it renders perfectly on service technicians' mobile devices.
- **Dealer Dashboard:** Uses standard cards and typography, ensuring visual cohesion with standard Salesforce reports.
- **Maintenance:** When Salesforce updates its UI theme, all LWC components using SLDS automatically inherit the new look without code changes.

---

## 27. Common Mistakes

| Problem | Cause | Solution | Example |
|---------|-------|----------|---------|
| **Excessive CSS** | Developers reverting to standard web dev habits. | Use SLDS Utility Classes. | Use `slds-p-around_medium` instead of `padding: 1rem;` |
| **Styling Base Components directly** | Trying to target `.slds-button` in a custom CSS file. | Use Styling Hooks. | `--sds-c-button-brand-color-background: red;` |
| **Rebuilding the wheel** | Creating custom tables or paginations. | Prefer Base Components. | Use `lightning-datatable`. |
| **Hardcoded Colors** | Using `#0070d2` in CSS. | Use standard classes. | Use `slds-text-color_brand`. |

---

## 28. Performance Considerations

- **Excessive DOM Elements:** Deeply nested grids (`slds-grid` inside `slds-grid` inside `slds-grid`) degrade rendering performance. Flatten your layouts where possible.
- **Large CSS Files:** If you are using SLDS utilities correctly, your component's `.css` file should be nearly empty. This reduces the component payload.
- **Unnecessary Wrappers:** Don't wrap a component in a `div` just to add margin; apply the margin class directly to the component if supported, or use CSS `host` styling.

---

## 29. Best Practices Checklist

- ✅ **Prefer Base Lightning Components when available:** Faster to build, accessible, and maintained by Salesforce.
- ✅ **Use SLDS utility classes for common styling:** Eliminates technical debt and CSS bloat.
- ✅ **Follow SLDS spacing conventions:** Never use random pixel values. Use `small`, `medium`, `large`.
- ✅ **Build responsive layouts:** Use `slds-size`, `slds-medium-size` to support desktop and mobile.
- ✅ **Avoid unnecessary custom CSS:** 95% of styling needs are met by SLDS.
- ✅ **Use Styling Hooks where appropriate:** The only safe way to tweak Base Component styling.
- ✅ **Follow accessibility standards:** Ensure screen reader text and proper ARIA roles.
- ✅ **Avoid hardcoded colors:** Rely on design tokens or SLDS color utilities.

---

## 30. Real Project Scenarios (Automotive CRM)

1. **Warranty Claim Dashboard:** Uses `slds-grid` with `slds-wrap` to show KPIs at the top (mobile responsive) and a `lightning-datatable` below for active claims.
2. **Vehicle Information Card:** Uses `lightning-card` housing a 2-column `slds-grid`. Fields use `slds-form-element` patterns for standard read-only data presentation.
3. **Spare Parts Selection:** Uses an SLDS Modal (`LightningModal`) to allow a technician to search SAP inventory without leaving the work order page.

---

## 31. Complete End-to-End Example

**Scenario:** A Warranty Claim Summary card displayed on a Work Order record.

### `warrantyClaimSummary.html`
```html
<template>
    <!-- Card container using Base Component -->
    <lightning-card title="Warranty Claim Summary" icon-name="custom:custom31">
        
        <!-- Slot for top-right actions -->
        <lightning-button label="Approve Claim" slot="actions" variant="brand" onclick={handleApprove}></lightning-button>

        <!-- Body wrapped in standard padding -->
        <div class="slds-p-around_medium">
            
            <!-- Grid for 2-column layout -->
            <div class="slds-grid slds-wrap slds-gutters">
                
                <!-- Column 1: Customer Details -->
                <div class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2 slds-m-bottom_medium">
                    <h3 class="slds-text-heading_small slds-text-color_weak slds-m-bottom_x-small">Customer</h3>
                    <p class="slds-text-body_regular">John Doe</p>
                    <p class="slds-text-body_regular">VIP Tier</p>
                </div>

                <!-- Column 2: Vehicle Details -->
                <div class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2 slds-m-bottom_medium">
                    <h3 class="slds-text-heading_small slds-text-color_weak slds-m-bottom_x-small">Vehicle</h3>
                    <p class="slds-text-body_regular">2024 Model X</p>
                    <p class="slds-text-body_regular slds-truncate" title="VIN: 1HGCM82633A">VIN: 1HGCM82633A</p>
                </div>
            </div>

            <!-- Distinct visual separation -->
            <div class="slds-border_top slds-m-top_small slds-p-top_small">
                <div class="slds-grid slds-grid_align-spread">
                    <span class="slds-text-heading_small">Total Claim Amount</span>
                    <span class="slds-text-heading_small slds-text-color_error">$2,450.00</span>
                </div>
                <div class="slds-m-top_small">
                    <!-- Base component badge for status -->
                    <lightning-badge label="Pending SAP Approval" class="slds-theme_warning"></lightning-badge>
                </div>
            </div>

        </div>
    </lightning-card>
</template>
```

### `warrantyClaimSummary.js`
```javascript
import { LightningElement, api } from 'lwc';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

export default class WarrantyClaimSummary extends LightningElement {
    @api recordId;

    handleApprove() {
        // Logic to approve claim
        const evt = new ShowToastEvent({
            title: 'Success',
            message: 'Claim Approved and sent to SAP.',
            variant: 'success',
        });
        this.dispatchEvent(evt);
    }
}
```

### `warrantyClaimSummary.js-meta.xml`
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

**Explanation:**
- `lightning-card` establishes the native UI shell.
- `slds-p-around_medium` gives breathing room to the contents.
- `slds-grid slds-wrap slds-gutters` creates a responsive 2-column layout that falls back to 1 column on mobile (`slds-size_1-of-1`).
- `slds-text-heading_small` and `slds-text-color_weak` format the section headers consistently.
- `slds-grid_align-spread` pushes the 'Total Claim Amount' text to the left and the price to the far right.

---

## 32. Common Interview Questions & Answers

### Beginner Questions
**Q: What is SLDS?**
A: Salesforce Lightning Design System is a CSS framework and set of UI guidelines that allow developers to build consistent, responsive, and accessible user interfaces that match standard Salesforce styling.

**Q: What are SLDS utility classes?**
A: Single-purpose CSS classes (like `slds-m-top_small` or `slds-text-align_center`) used to modify layout, typography, or spacing without writing custom CSS.

### Intermediate Questions
**Q: What is the difference between SLDS and Base Lightning Components?**
A: SLDS provides the CSS and design blueprints. Base Lightning Components (like `lightning-button`) are fully functional, pre-built LWC elements that incorporate SLDS under the hood, along with JavaScript logic and accessibility features.

**Q: How do you create a responsive layout using SLDS?**
A: By using the `slds-grid` system combined with responsive sizing classes like `slds-size_1-of-1 slds-medium-size_1-of-2`. This makes columns stack on mobile but sit side-by-side on tablets/desktops.

### Advanced Questions
**Q: How does SLDS work with Shadow DOM in LWC?**
A: LWC uses Shadow DOM to encapsulate CSS, preventing styles from leaking in or out. However, Salesforce globally injects SLDS stylesheets into the shadow roots of LWC, allowing you to use SLDS utility classes directly in your templates. Custom CSS in your component file remains scoped.

**Q: When should you use custom CSS instead of SLDS?**
A: Custom CSS should be avoided. It is only appropriate when a highly specific design requirement cannot be met by SLDS (e.g., custom animations, specific brand backgrounds not supported by hooks, or 3rd party library integrations). 

### Architect-Level Questions
**Q: How do you customize the styling of standard Base Components if standard CSS cannot pierce the Shadow DOM?**
A: You must use **Styling Hooks**. These are CSS Custom Properties (variables) provided by Salesforce (e.g., `--sds-c-button-brand-color-background`). You define these variables in your component's `.css` file, and the base component inherits them, safely customizing the UI without relying on internal DOM structure.

---

## 33. Revision Summary

- **SLDS:** The core design language of Salesforce.
- **Base Components > SLDS > Custom CSS:** Always prefer pre-built base components. If you must build custom UI, use SLDS classes. Write custom CSS only as a last resort.
- **Utility Classes:** Use `m`, `p`, `text`, and `align` classes for quick layout adjustments.
- **Grid:** Flexbox-based (`slds-grid`, `slds-col`, `slds-wrap`) for responsive data layouts.
- **Styling Hooks:** The only supported way to override colors and spacing inside Base Components (`--sds-c-*`).
- **Accessibility:** SLDS provides built-in focus management, ARIA patterns, and high-contrast color tokens to meet W3C standards.
- **Shadow DOM:** Ensures component isolation; SLDS is globally available within this boundary but custom CSS is strictly scoped. 

*This concludes the SLDS & LWC UI Development Handbook chapter. Keep your components accessible, responsive, and native.*