# LWC Best Practices — Full Reference

## 1. Architecture Principles

### Component Design

- **Single responsibility** — each component does one thing well
- **Composable** — build complex UIs by nesting smaller components
- **Prefer LWC over Aura** for all new development (LWC loads ~3x faster)
- **Use Lightning Data Service** before writing custom Apex controllers
- **Stateless where possible** — let data flow down via `@api`, events flow up

### Data Loading Priority

1. **`lightning-record-form`** — simplest, auto handles view/edit, built-in Save/Cancel
2. **`lightning-record-edit-form`** + `lightning-input-field` — more layout control
3. **`lightning-record-view-form`** + `lightning-output-field` — read-only display
4. **`@wire` with `getRecord`** — reactive data binding for custom UIs
5. **Imperative Apex** — only when LDS is insufficient (complex queries, mutations)

### Decorator Usage

- `@api` — public properties and methods exposed to parent components
- `@track` — rarely needed (all properties are reactive since API 40+; only needed for reassignment tracking on objects/arrays)
- `@wire` — reactive data provisioning; auto-reruns when reactive (`$`) params change

---

## 2. SLDS Compliance (Mandatory)

### Layered Approach (in order of preference)

1. **Lightning Base Components** — `lightning-button`, `lightning-datatable`, `lightning-card`, `lightning-input`, `lightning-combobox`, `lightning-modal`, `lightning-tree-grid`, `lightning-accordion`, `lightning-tab`, `lightning-tabset`, etc.

2. **SLDS Blueprint Classes** — when no base component fits:
   - Layout: `slds-grid`, `slds-col`, `slds-wrap`, `slds-gutters`
   - Sizing: `slds-size_1-of-2`, `slds-size_1-of-3`, `slds-large-size_1-of-4`
   - Spacing: `slds-m-top_medium`, `slds-p-around_small`, `slds-m-bottom_large`
   - Text: `slds-text-heading_medium`, `slds-text-body_regular`, `slds-text-color_error`
   - Cards: `slds-card`, `slds-card__header`, `slds-card__body`
   - Tables: `slds-table`, `slds-table_bordered`, `slds-table_striped`
   - Badges: `slds-badge`, `slds-badge_inverse`
   - Modals: `slds-modal`, `slds-backdrop` (prefer `lightning-modal` base class)
   - Utilities: `slds-truncate`, `slds-hide`, `slds-show`, `slds-align_absolute-center`

3. **SLDS Styling Hooks** (CSS custom properties) — for visual customization only:
   - Use `var(--slds-g-color-brand-base-50)` instead of hardcoded `#0070d2`
   - Use `var(--slds-g-spacing-4)` instead of hardcoded `16px`
   - Use `var(--slds-g-font-size-4)` instead of hardcoded `14px`

### Anti-Patterns (never do)

- Never style based on rendered internal markup of base components
- Never override SLDS classes directly (`.slds-button { background: red }`)
- Never use `!important` unless absolutely unavoidable
- Never hardcode colors, fonts, or spacing — use SLDS hooks
- Never mix SLDS 1 tokens with SLDS 2 hooks in the same component

### Responsive Design

```html
<div class="slds-grid slds-wrap">
    <div class="slds-col slds-size_1-of-1 slds-medium-size_1-of-2 slds-large-size_1-of-3">
        <!-- responsive column -->
    </div>
</div>
```

### Icons

Always use `lightning-icon` with correct categories:
- `standard:account` — standard object icons
- `utility:search` — utility/action icons
- `action:new_task` — action icons
- `custom:custom1` — custom icons
- `doctype:pdf` — document type icons

---

## 3. HTML Template Rules

### Conditional Rendering

```html
<!-- CORRECT — lwc:if (current standard) -->
<template lwc:if={isLoading}>
    <lightning-spinner alternative-text="Loading"></lightning-spinner>
</template>
<template lwc:elseif={hasError}>
    <p class="slds-text-color_error">{errorMessage}</p>
</template>
<template lwc:else>
    <c-my-content data={records}></c-my-content>
</template>

<!-- WRONG — deprecated directive -->
<template if:true={isLoading}>...</template>
```

### List Rendering

```html
<!-- key MUST be a unique, stable value (like Id) -->
<template for:each={contacts} for:item="contact">
    <div key={contact.Id} class="slds-m-bottom_small">
        {contact.Name}
    </div>
</template>
```

### Slots for Composition

```html
<!-- Parent -->
<c-card>
    <span slot="title">My Card</span>
    <p slot="body">Content here</p>
</c-card>

<!-- c-card template -->
<div class="slds-card">
    <div class="slds-card__header">
        <slot name="title"></slot>
    </div>
    <div class="slds-card__body">
        <slot name="body"></slot>
    </div>
</div>
```

### Avoid Complex Template Expressions

```javascript
// BAD — complex expression in template
// <template lwc:if={record.Status__c === 'Active'}>

// GOOD — use a getter
get isActive() {
    return this.record?.Status__c === 'Active';
}
// <template lwc:if={isActive}>
```

---

## 4. JavaScript Patterns

### Wire Service

```javascript
import { LightningElement, wire } from 'lwc';
import { getRecord, getFieldValue } from 'lightning/uiRecordApi';
import NAME_FIELD from '@salesforce/schema/Account.Name';
import INDUSTRY_FIELD from '@salesforce/schema/Account.Industry';

export default class MyComponent extends LightningElement {
    @api recordId;

    @wire(getRecord, { recordId: '$recordId', fields: [NAME_FIELD, INDUSTRY_FIELD] })
    account;

    get accountName() {
        return getFieldValue(this.account.data, NAME_FIELD);
    }

    get hasError() {
        return !!this.account.error;
    }
}
```

### Imperative Apex

```javascript
import getContacts from '@salesforce/apex/ContactController.getContacts';

export default class MyComponent extends LightningElement {
    contacts = [];
    error;

    async connectedCallback() {
        try {
            this.contacts = await getContacts({ accountId: this.recordId });
        } catch (error) {
            this.error = error?.body?.message || 'Unknown error';
        }
    }
}
```

### Debouncing

```javascript
_searchTimeout;

handleSearchChange(event) {
    const searchTerm = event.target.value;
    clearTimeout(this._searchTimeout);
    this._searchTimeout = setTimeout(() => {
        this.performSearch(searchTerm);
    }, 300);
}
```

### Event Patterns

```javascript
// Child fires CustomEvent
this.dispatchEvent(new CustomEvent('select', {
    detail: { recordId: this.selectedId },
    bubbles: false,
    composed: false
}));

// Parent handles it
// <c-child onselect={handleSelect}></c-child>
handleSelect(event) {
    const selectedId = event.detail.recordId;
}
```

### Lifecycle Hooks

```javascript
export default class MyComponent extends LightningElement {
    _intervalId;

    constructor() {
        super();
        // no DOM access; initialize non-reactive state
    }

    connectedCallback() {
        // DOM inserted; fetch data, subscribe to events
        this._intervalId = setInterval(() => this.poll(), 30000);
    }

    renderedCallback() {
        // after every render; guard against infinite loops
        if (this._hasRendered) return;
        this._hasRendered = true;
        // one-time DOM manipulations
    }

    disconnectedCallback() {
        // cleanup: remove listeners, clear intervals
        clearInterval(this._intervalId);
    }

    errorCallback(error, stack) {
        // handle child component errors gracefully
        this.error = error.message;
    }
}
```

### Toast Notifications

```javascript
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

this.dispatchEvent(new ShowToastEvent({
    title: 'Success',
    message: 'Record saved successfully',
    variant: 'success' // success, error, warning, info
}));
```

### Navigation

```javascript
import { NavigationMixin } from 'lightning/navigation';

export default class MyComponent extends NavigationMixin(LightningElement) {
    navigateToRecord() {
        this[NavigationMixin.Navigate]({
            type: 'standard__recordPage',
            attributes: {
                recordId: this.recordId,
                objectApiName: 'Account',
                actionName: 'view'
            }
        });
    }
}
```

---

## 5. Performance

- Import only needed fields via `@salesforce/schema` — avoid layout-based wire
- Lazy-load heavy child components with `lwc:if`
- Use `lightning-datatable` with pagination for large datasets
- Avoid `querySelectorAll` in loops — cache DOM references
- Use `@wire` (cached by LDS) over imperative calls when possible
- Debounce user inputs (search, filters) — 300ms minimum
- Do not update wire adapter config in `renderedCallback()` (causes infinite loops)

---

## 6. Security (LWC-specific)

### Lightning Web Security (LWS)

- LWS replaces Locker Service (default for orgs created Winter '23+)
- Provides namespace-isolated JavaScript sandboxes
- Third-party libraries must comply: no `eval()`, no `Function()` constructors, support strict mode

### CSP Compliance

- All JS must be loaded from org (static resources or modules) — never CDN
- Add external domains to Trusted URLs for images/fonts/frames
- HTTPS required for all external resources
- No inline event handlers in markup

### Data Security

- Import fields from `@salesforce/schema` for compile-time validation
- Use Lightning Data Service (auto-enforces FLS and sharing)
- When using imperative Apex, ensure the Apex method enforces CRUD/FLS
- Never expose sensitive data in `console.log` in production

---

## 7. Jest Testing

### Setup

```bash
sf force lightning lwc test setup   # one-time project setup
npm test                            # run all tests
npm test -- --coverage              # with coverage report
npm test -- --watch                 # continuous testing
```

### Test File Location

```
force-app/main/default/lwc/myComponent/
├── myComponent.html
├── myComponent.js
├── myComponent.css
├── myComponent.js-meta.xml
└── __tests__/
    └── myComponent.test.js
```

### Test Template

```javascript
import { createElement } from 'lwc';
import MyComponent from 'c/myComponent';

describe('c-my-component', () => {
    afterEach(() => {
        while (document.body.firstChild) {
            document.body.removeChild(document.body.firstChild);
        }
    });

    it('renders the component', () => {
        const element = createElement('c-my-component', { is: MyComponent });
        document.body.appendChild(element);

        const heading = element.shadowRoot.querySelector('h2');
        expect(heading.textContent).toBe('Expected Title');
    });

    it('handles reactive property changes', async () => {
        const element = createElement('c-my-component', { is: MyComponent });
        element.recordId = '001xx000003ABCDEF';
        document.body.appendChild(element);

        await Promise.resolve(); // wait for rerender

        const detail = element.shadowRoot.querySelector('.detail');
        expect(detail).not.toBeNull();
    });

    it('fires custom event on button click', () => {
        const element = createElement('c-my-component', { is: MyComponent });
        document.body.appendChild(element);

        const handler = jest.fn();
        element.addEventListener('select', handler);

        const button = element.shadowRoot.querySelector('lightning-button');
        button.click();

        expect(handler).toHaveBeenCalledTimes(1);
    });
});
```

### Mocking Apex

```javascript
import { createElement } from 'lwc';
import MyComponent from 'c/myComponent';
import getAccounts from '@salesforce/apex/AccountController.getAccounts';

jest.mock('@salesforce/apex/AccountController.getAccounts', () => ({
    default: jest.fn()
}), { virtual: true });

const MOCK_ACCOUNTS = [
    { Id: '001xx000001', Name: 'Test Account 1' },
    { Id: '001xx000002', Name: 'Test Account 2' }
];

describe('c-my-component', () => {
    it('renders accounts from Apex', async () => {
        getAccounts.mockResolvedValue(MOCK_ACCOUNTS);
        const element = createElement('c-my-component', { is: MyComponent });
        document.body.appendChild(element);

        await Promise.resolve();
        await Promise.resolve();

        const items = element.shadowRoot.querySelectorAll('.account-item');
        expect(items.length).toBe(2);
    });
});
```

---

## 8. Accessibility

- Include `alternative-text` on all `lightning-icon` and `lightning-spinner`
- Use `aria-label` on interactive elements without visible labels
- Use `aria-live="polite"` for dynamic content regions
- Ensure all interactive elements are keyboard-navigable (Tab, Enter, Escape)
- Use `lightning-helptext` for contextual tooltips
- Use semantic HTML (`<h1>`–`<h6>`, `<nav>`, `<main>`, `<section>`)
- Maintain WCAG 2.1 AA color contrast (4.5:1 for normal text, 3:1 for large)
- Test with screen readers and keyboard-only navigation
