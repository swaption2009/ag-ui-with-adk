# Practical Examples: DOM Manipulation with ADK

This file contains ready-to-use code examples for common DOM manipulation scenarios.

---

## Table of Contents

1. [Form Automation](#form-automation)
2. [UI Testing](#ui-testing)
3. [E-commerce Automation](#e-commerce-automation)
4. [Content Extraction](#content-extraction)
5. [Accessibility Helpers](#accessibility-helpers)
6. [Navigation Assistants](#navigation-assistants)
7. [Data Entry Automation](#data-entry-automation)
8. [Visual Modifications](#visual-modifications)

---

## Form Automation

### Example 1: Contact Form Filler

```typescript
// src/app/components/ContactFormHelper.tsx
"use client";

import { useCopilotAction } from "@copilotkit/react-core";

export function ContactFormHelper() {
  useCopilotAction({
    name: "fillContactForm",
    description: "Automatically fill out a contact form with provided details",
    parameters: [
      { name: "name", type: "string", required: true },
      { name: "email", type: "string", required: true },
      { name: "phone", type: "string", required: false },
      { name: "subject", type: "string", required: false },
      { name: "message", type: "string", required: true }
    ],
    handler({ name, email, phone, subject, message }) {
      const fields = [
        { selector: 'input[name="name"]', value: name },
        { selector: 'input[name="email"]', value: email },
        { selector: 'input[name="phone"]', value: phone },
        { selector: 'input[name="subject"]', value: subject },
        { selector: 'textarea[name="message"]', value: message }
      ];

      const results = [];
      for (const field of fields) {
        if (field.value) {
          const element = document.querySelector(field.selector) as HTMLInputElement;
          if (element) {
            element.value = field.value;
            element.dispatchEvent(new Event('input', { bubbles: true }));
            element.dispatchEvent(new Event('change', { bubbles: true }));
            results.push(`✓ ${field.selector}`);
          } else {
            results.push(`✗ ${field.selector} not found`);
          }
        }
      }

      return {
        success: true,
        message: `Form filled: ${results.join(', ')}`,
        details: results
      };
    }
  });

  return null;
}
```

**Usage:**
```
User: "Fill the contact form with my details: John Doe, john@example.com, 555-1234, and message: I'd like to learn more about your services"
```

### Example 2: Multi-Step Form Wizard

```typescript
useCopilotAction({
  name: "completeFormWizard",
  description: "Complete a multi-step form wizard",
  parameters: [
    { name: "step1Data", type: "object", required: true },
    { name: "step2Data", type: "object", required: true },
    { name: "step3Data", type: "object", required: true }
  ],
  async handler({ step1Data, step2Data, step3Data }) {
    const steps = [
      { data: step1Data, nextButton: '#step1-next' },
      { data: step2Data, nextButton: '#step2-next' },
      { data: step3Data, nextButton: '#submit' }
    ];

    for (let i = 0; i < steps.length; i++) {
      const step = steps[i];
      
      // Fill current step
      for (const [field, value] of Object.entries(step.data)) {
        const input = document.querySelector(`[name="${field}"]`) as HTMLInputElement;
        if (input) {
          input.value = value as string;
          input.dispatchEvent(new Event('input', { bubbles: true }));
        }
      }

      // Wait a bit
      await new Promise(resolve => setTimeout(resolve, 500));

      // Click next/submit
      const button = document.querySelector(step.nextButton) as HTMLButtonElement;
      button?.click();

      // Wait for next step to load
      await new Promise(resolve => setTimeout(resolve, 1000));
    }

    return { success: true, message: "Form wizard completed!" };
  }
});
```

---

## UI Testing

### Example 3: Login Flow Tester

```typescript
useCopilotAction({
  name: "testLoginFlow",
  description: "Test the login functionality with various scenarios",
  parameters: [
    { name: "username", type: "string", required: true },
    { name: "password", type: "string", required: true },
    { name: "expectedResult", type: "string", required: true } // "success" or "error"
  ],
  async handler({ username, password, expectedResult }) {
    const testResults = [];

    try {
      // Step 1: Navigate to login page
      const loginLink = document.querySelector('a[href*="login"]') as HTMLAnchorElement;
      if (loginLink) {
        loginLink.click();
        testResults.push({ step: "Navigate to login", passed: true });
        await new Promise(r => setTimeout(r, 1000));
      }

      // Step 2: Fill username
      const usernameField = document.querySelector('#username, input[name="username"]') as HTMLInputElement;
      if (usernameField) {
        usernameField.value = username;
        usernameField.dispatchEvent(new Event('input', { bubbles: true }));
        testResults.push({ step: "Fill username", passed: true });
      } else {
        testResults.push({ step: "Fill username", passed: false, error: "Field not found" });
      }

      // Step 3: Fill password
      const passwordField = document.querySelector('#password, input[name="password"]') as HTMLInputElement;
      if (passwordField) {
        passwordField.value = password;
        passwordField.dispatchEvent(new Event('input', { bubbles: true }));
        testResults.push({ step: "Fill password", passed: true });
      } else {
        testResults.push({ step: "Fill password", passed: false, error: "Field not found" });
      }

      // Step 4: Submit form
      await new Promise(r => setTimeout(r, 500));
      const submitButton = document.querySelector('button[type="submit"], #login-btn') as HTMLButtonElement;
      if (submitButton) {
        submitButton.click();
        testResults.push({ step: "Submit form", passed: true });
      } else {
        testResults.push({ step: "Submit form", passed: false, error: "Button not found" });
      }

      // Step 5: Check result
      await new Promise(r => setTimeout(r, 2000));
      const errorMessage = document.querySelector('.error-message, .alert-error');
      const successMessage = document.querySelector('.success-message, .alert-success');
      const dashboardElement = document.querySelector('#dashboard, .dashboard');

      if (expectedResult === "success") {
        const passed = !!(successMessage || dashboardElement);
        testResults.push({
          step: "Verify success",
          passed,
          message: passed ? "Login successful" : "Login failed"
        });
      } else {
        const passed = !!errorMessage;
        testResults.push({
          step: "Verify error",
          passed,
          message: passed ? `Error shown: ${errorMessage?.textContent}` : "No error shown"
        });
      }

      const totalPassed = testResults.filter(r => r.passed).length;
      const totalFailed = testResults.filter(r => !r.passed).length;

      return {
        success: totalFailed === 0,
        message: `Test completed: ${totalPassed} passed, ${totalFailed} failed`,
        results: testResults
      };
    } catch (error) {
      return {
        success: false,
        message: `Test error: ${error.message}`,
        results: testResults
      };
    }
  }
});
```

### Example 4: Comprehensive Page Validator

```typescript
useCopilotAction({
  name: "validatePage",
  description: "Validate that all expected elements are present on the page",
  parameters: [
    {
      name: "requiredElements",
      type: "array",
      description: "Array of CSS selectors that must be present",
      required: true
    }
  ],
  handler({ requiredElements }) {
    const results = (requiredElements as string[]).map(selector => {
      const element = document.querySelector(selector);
      return {
        selector,
        found: !!element,
        visible: element ? (element as HTMLElement).checkVisibility?.() ?? true : false,
        text: element?.textContent?.trim().substring(0, 50)
      };
    });

    const allFound = results.every(r => r.found);
    const allVisible = results.every(r => r.visible);

    return {
      success: allFound && allVisible,
      message: `Validation: ${results.filter(r => r.found).length}/${results.length} found, ${results.filter(r => r.visible).length}/${results.length} visible`,
      results
    };
  }
});
```

---

## E-commerce Automation

### Example 5: Product Search & Filter

```typescript
useCopilotAction({
  name: "searchAndFilterProducts",
  description: "Search for products and apply filters",
  parameters: [
    { name: "searchQuery", type: "string", required: true },
    { name: "priceRange", type: "string", required: false },
    { name: "category", type: "string", required: false },
    { name: "sortBy", type: "string", required: false }
  ],
  async handler({ searchQuery, priceRange, category, sortBy }) {
    const actions = [];

    // Step 1: Search
    const searchInput = document.querySelector('input[type="search"], #search-input') as HTMLInputElement;
    if (searchInput) {
      searchInput.value = searchQuery;
      searchInput.dispatchEvent(new Event('input', { bubbles: true }));
      actions.push("Entered search query");
      
      const searchButton = document.querySelector('button[type="submit"], .search-button') as HTMLButtonElement;
      searchButton?.click();
      actions.push("Clicked search");
      
      await new Promise(r => setTimeout(r, 1500));
    }

    // Step 2: Apply price filter
    if (priceRange) {
      const priceFilter = document.querySelector(`input[value="${priceRange}"]`) as HTMLInputElement;
      if (priceFilter) {
        priceFilter.checked = true;
        priceFilter.dispatchEvent(new Event('change', { bubbles: true }));
        actions.push(`Applied price filter: ${priceRange}`);
        await new Promise(r => setTimeout(r, 500));
      }
    }

    // Step 3: Apply category filter
    if (category) {
      const categoryFilter = document.querySelector(`input[value="${category}"]`) as HTMLInputElement;
      if (categoryFilter) {
        categoryFilter.checked = true;
        categoryFilter.dispatchEvent(new Event('change', { bubbles: true }));
        actions.push(`Applied category filter: ${category}`);
        await new Promise(r => setTimeout(r, 500));
      }
    }

    // Step 4: Sort results
    if (sortBy) {
      const sortSelect = document.querySelector('select[name="sort"], #sort-select') as HTMLSelectElement;
      if (sortSelect) {
        sortSelect.value = sortBy;
        sortSelect.dispatchEvent(new Event('change', { bubbles: true }));
        actions.push(`Sorted by: ${sortBy}`);
      }
    }

    return {
      success: true,
      message: `Search and filter completed: ${actions.join(', ')}`,
      actions
    };
  }
});
```

### Example 6: Add Multiple Items to Cart

```typescript
useCopilotAction({
  name: "addMultipleToCart",
  description: "Add multiple products to shopping cart",
  parameters: [
    {
      name: "productSelectors",
      type: "array",
      description: "Array of CSS selectors for products to add",
      required: true
    }
  ],
  async handler({ productSelectors }) {
    const results = [];

    for (const selector of productSelectors as string[]) {
      try {
        const product = document.querySelector(selector);
        if (product) {
          const addButton = product.querySelector('.add-to-cart, button[data-action="add-to-cart"]') as HTMLButtonElement;
          if (addButton) {
            addButton.click();
            results.push({ selector, success: true });
            await new Promise(r => setTimeout(r, 500)); // Wait between additions
          } else {
            results.push({ selector, success: false, error: "Add button not found" });
          }
        } else {
          results.push({ selector, success: false, error: "Product not found" });
        }
      } catch (error) {
        results.push({ selector, success: false, error: error.message });
      }
    }

    const successCount = results.filter(r => r.success).length;
    return {
      success: successCount > 0,
      message: `Added ${successCount}/${results.length} items to cart`,
      results
    };
  }
});
```

---

## Content Extraction

### Example 7: Extract Table Data

```typescript
useCopilotAction({
  name: "extractTableData",
  description: "Extract all data from a table into structured format",
  parameters: [
    { name: "tableSelector", type: "string", required: true }
  ],
  handler({ tableSelector }) {
    const table = document.querySelector(tableSelector);
    if (!table) {
      return { success: false, message: "Table not found" };
    }

    // Extract headers
    const headers = Array.from(table.querySelectorAll('thead th, thead td')).map(
      th => th.textContent?.trim() || ''
    );

    // Extract rows
    const rows = Array.from(table.querySelectorAll('tbody tr')).map(tr => {
      const cells = Array.from(tr.querySelectorAll('td')).map(
        td => td.textContent?.trim() || ''
      );
      
      // Create object with headers as keys
      const rowObject: Record<string, string> = {};
      headers.forEach((header, i) => {
        rowObject[header] = cells[i] || '';
      });
      
      return rowObject;
    });

    return {
      success: true,
      message: `Extracted ${rows.length} rows with ${headers.length} columns`,
      headers,
      rows,
      rowCount: rows.length,
      columnCount: headers.length
    };
  }
});
```

### Example 8: Extract All Links

```typescript
useCopilotAction({
  name: "extractAllLinks",
  description: "Extract all links from the page with their text and URLs",
  parameters: [
    { name: "filterPattern", type: "string", required: false }
  ],
  handler({ filterPattern }) {
    const links = Array.from(document.querySelectorAll('a[href]')).map((link, index) => ({
      index,
      text: link.textContent?.trim() || '',
      href: (link as HTMLAnchorElement).href,
      target: (link as HTMLAnchorElement).target || '_self',
      isExternal: (link as HTMLAnchorElement).hostname !== window.location.hostname
    }));

    // Filter if pattern provided
    const filtered = filterPattern
      ? links.filter(link => 
          link.href.includes(filterPattern) || link.text.includes(filterPattern)
        )
      : links;

    return {
      success: true,
      message: `Found ${filtered.length} links${filterPattern ? ` matching "${filterPattern}"` : ''}`,
      links: filtered,
      totalLinks: links.length,
      filteredLinks: filtered.length
    };
  }
});
```

---

## Accessibility Helpers

### Example 9: Increase Text Size

```typescript
useCopilotAction({
  name: "increaseTextSize",
  description: "Increase the text size across the entire page for better readability",
  parameters: [
    { name: "percentage", type: "number", required: false }
  ],
  handler({ percentage = 120 }) {
    const currentSize = parseFloat(
      window.getComputedStyle(document.body).fontSize
    );
    const newSize = (currentSize * (percentage as number)) / 100;
    
    document.body.style.fontSize = `${newSize}px`;
    
    return {
      success: true,
      message: `Text size increased to ${percentage}% (${newSize}px)`,
      oldSize: currentSize,
      newSize
    };
  }
});
```

### Example 10: High Contrast Mode

```typescript
useCopilotAction({
  name: "toggleHighContrast",
  description: "Toggle high contrast mode for better visibility",
  parameters: [
    { name: "enable", type: "boolean", required: true }
  ],
  handler({ enable }) {
    if (enable) {
      document.body.style.filter = 'contrast(1.5) brightness(1.1)';
      document.body.style.backgroundColor = '#000';
      document.body.style.color = '#fff';
      
      // Invert images for better visibility
      const images = document.querySelectorAll('img');
      images.forEach(img => {
        (img as HTMLElement).style.filter = 'invert(1)';
      });
      
      return { success: true, message: "High contrast mode enabled" };
    } else {
      document.body.style.filter = '';
      document.body.style.backgroundColor = '';
      document.body.style.color = '';
      
      const images = document.querySelectorAll('img');
      images.forEach(img => {
        (img as HTMLElement).style.filter = '';
      });
      
      return { success: true, message: "High contrast mode disabled" };
    }
  }
});
```

---

## Navigation Assistants

### Example 11: Smart Scroll to Section

```typescript
useCopilotAction({
  name: "scrollToSection",
  description: "Intelligently scroll to a section by name or keyword",
  parameters: [
    { name: "sectionName", type: "string", required: true }
  ],
  handler({ sectionName }) {
    // Try multiple strategies to find the section
    const strategies = [
      // Strategy 1: ID match
      () => document.querySelector(`#${sectionName.toLowerCase().replace(/\s+/g, '-')}`),
      // Strategy 2: Heading text match
      () => Array.from(document.querySelectorAll('h1, h2, h3, h4, h5, h6')).find(
        h => h.textContent?.toLowerCase().includes(sectionName.toLowerCase())
      ),
      // Strategy 3: Section with matching aria-label
      () => document.querySelector(`section[aria-label*="${sectionName}" i]`),
      // Strategy 4: Any element with matching text
      () => Array.from(document.querySelectorAll('section, article, div[id]')).find(
        el => el.textContent?.toLowerCase().includes(sectionName.toLowerCase())
      )
    ];

    for (const strategy of strategies) {
      const element = strategy();
      if (element) {
        element.scrollIntoView({ behavior: 'smooth', block: 'start' });
        return {
          success: true,
          message: `Scrolled to section: ${sectionName}`,
          foundBy: strategies.indexOf(strategy) + 1
        };
      }
    }

    return {
      success: false,
      message: `Section not found: ${sectionName}`
    };
  }
});
```

### Example 12: Page Navigator

```typescript
useCopilotAction({
  name: "navigatePages",
  description: "Navigate through paginated content",
  parameters: [
    { name: "direction", type: "string", required: true } // "next", "previous", "first", "last"
  ],
  handler({ direction }) {
    let button: HTMLElement | null = null;

    switch (direction) {
      case "next":
        button = document.querySelector('.next-page, .pagination-next, a[rel="next"]') as HTMLElement;
        break;
      case "previous":
        button = document.querySelector('.prev-page, .pagination-prev, a[rel="prev"]') as HTMLElement;
        break;
      case "first":
        button = document.querySelector('.first-page, .pagination-first') as HTMLElement;
        break;
      case "last":
        button = document.querySelector('.last-page, .pagination-last') as HTMLElement;
        break;
    }

    if (button) {
      button.click();
      return {
        success: true,
        message: `Navigated to ${direction} page`
      };
    }

    return {
      success: false,
      message: `${direction} page button not found`
    };
  }
});
```

---

## Data Entry Automation

### Example 13: Auto-fill Address

```typescript
useCopilotAction({
  name: "fillAddress",
  description: "Automatically fill address fields",
  parameters: [
    { name: "street", type: "string", required: true },
    { name: "city", type: "string", required: true },
    { name: "state", type: "string", required: true },
    { name: "zip", type: "string", required: true },
    { name: "country", type: "string", required: false }
  ],
  handler({ street, city, state, zip, country }) {
    const fieldMappings = [
      { value: street, selectors: ['input[name*="street" i]', '#street', '#address1'] },
      { value: city, selectors: ['input[name*="city" i]', '#city'] },
      { value: state, selectors: ['input[name*="state" i]', 'select[name*="state" i]', '#state'] },
      { value: zip, selectors: ['input[name*="zip" i]', 'input[name*="postal" i]', '#zip', '#postal-code'] },
      { value: country, selectors: ['select[name*="country" i]', '#country'] }
    ];

    const results = [];
    for (const mapping of fieldMappings) {
      if (!mapping.value) continue;

      for (const selector of mapping.selectors) {
        const field = document.querySelector(selector) as HTMLInputElement | HTMLSelectElement;
        if (field) {
          field.value = mapping.value;
          field.dispatchEvent(new Event('input', { bubbles: true }));
          field.dispatchEvent(new Event('change', { bubbles: true }));
          results.push(`✓ ${selector}`);
          break;
        }
      }
    }

    return {
      success: results.length > 0,
      message: `Filled ${results.length} address fields`,
      details: results
    };
  }
});
```

---

## Visual Modifications

### Example 14: Theme Switcher

```typescript
useCopilotAction({
  name: "switchTheme",
  description: "Switch between light and dark themes",
  parameters: [
    { name: "theme", type: "string", required: true } // "light", "dark", "auto"
  ],
  handler({ theme }) {
    if (theme === "dark") {
      document.documentElement.style.setProperty('--background', '#1a1a1a');
      document.documentElement.style.setProperty('--foreground', '#ffffff');
      document.body.style.backgroundColor = '#1a1a1a';
      document.body.style.color = '#ffffff';
    } else if (theme === "light") {
      document.documentElement.style.setProperty('--background', '#ffffff');
      document.documentElement.style.setProperty('--foreground', '#000000');
      document.body.style.backgroundColor = '#ffffff';
      document.body.style.color = '#000000';
    } else if (theme === "auto") {
      const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
      return handler({ theme: prefersDark ? "dark" : "light" });
    }

    return {
      success: true,
      message: `Switched to ${theme} theme`
    };
  }
});
```

### Example 15: Focus Mode

```typescript
useCopilotAction({
  name: "enableFocusMode",
  description: "Hide distractions and focus on main content",
  parameters: [
    { name: "enable", type: "boolean", required: true }
  ],
  handler({ enable }) {
    const distractionSelectors = [
      'aside',
      'nav',
      '.sidebar',
      '.advertisement',
      '.ad',
      '.popup',
      '.modal',
      'header',
      'footer'
    ];

    if (enable) {
      distractionSelectors.forEach(selector => {
        document.querySelectorAll(selector).forEach(el => {
          (el as HTMLElement).style.display = 'none';
        });
      });

      // Highlight main content
      const main = document.querySelector('main, article, .content');
      if (main) {
        (main as HTMLElement).style.maxWidth = '800px';
        (main as HTMLElement).style.margin = '0 auto';
        (main as HTMLElement).style.padding = '2rem';
      }

      return { success: true, message: "Focus mode enabled" };
    } else {
      distractionSelectors.forEach(selector => {
        document.querySelectorAll(selector).forEach(el => {
          (el as HTMLElement).style.display = '';
        });
      });

      const main = document.querySelector('main, article, .content');
      if (main) {
        (main as HTMLElement).style.maxWidth = '';
        (main as HTMLElement).style.margin = '';
        (main as HTMLElement).style.padding = '';
      }

      return { success: true, message: "Focus mode disabled" };
    }
  }
});
```

---

## Combining Examples

### Example 16: Complete E-commerce Checkout

```typescript
useCopilotAction({
  name: "completeCheckout",
  description: "Complete the entire checkout process",
  parameters: [
    { name: "shippingInfo", type: "object", required: true },
    { name: "paymentInfo", type: "object", required: true }
  ],
  async handler({ shippingInfo, paymentInfo }) {
    const steps = [];

    try {
      // Step 1: Fill shipping info
      const shipping = shippingInfo as any;
      (document.querySelector('#shipping-name') as HTMLInputElement).value = shipping.name;
      (document.querySelector('#shipping-email') as HTMLInputElement).value = shipping.email;
      (document.querySelector('#shipping-address') as HTMLInputElement).value = shipping.address;
      (document.querySelector('#shipping-city') as HTMLInputElement).value = shipping.city;
      (document.querySelector('#shipping-zip') as HTMLInputElement).value = shipping.zip;
      steps.push("Filled shipping info");

      // Step 2: Continue to payment
      await new Promise(r => setTimeout(r, 500));
      (document.querySelector('#continue-to-payment') as HTMLButtonElement)?.click();
      await new Promise(r => setTimeout(r, 1500));
      steps.push("Navigated to payment");

      // Step 3: Fill payment info
      const payment = paymentInfo as any;
      (document.querySelector('#card-number') as HTMLInputElement).value = payment.cardNumber;
      (document.querySelector('#card-expiry') as HTMLInputElement).value = payment.expiry;
      (document.querySelector('#card-cvv') as HTMLInputElement).value = payment.cvv;
      steps.push("Filled payment info");

      // Step 4: Review and confirm
      await new Promise(r => setTimeout(r, 500));
      (document.querySelector('#place-order') as HTMLButtonElement)?.click();
      steps.push("Placed order");

      return {
        success: true,
        message: "Checkout completed successfully!",
        steps
      };
    } catch (error) {
      return {
        success: false,
        message: `Checkout failed: ${error.message}`,
        steps
      };
    }
  }
});
```

---

## Usage Examples

### In Your Agent Instructions

Update your agent's instructions to use these examples:

```python
# agent/agent.py
proverbs_agent = LlmAgent(
    name="ProverbsAgent",
    model="gemini-2.5-flash",
    instruction=f"""
    You are a helpful web automation assistant.
    
    AVAILABLE ACTIONS:
    - fillContactForm: Fill contact forms
    - testLoginFlow: Test login functionality
    - searchAndFilterProducts: Search and filter e-commerce products
    - extractTableData: Extract data from tables
    - increaseTextSize: Make text more readable
    - scrollToSection: Navigate to page sections
    - fillAddress: Auto-fill address forms
    - switchTheme: Change page theme
    - enableFocusMode: Hide distractions
    
    When users ask you to interact with the page, use these actions.
    Always confirm what you're about to do before taking action.
    """,
    tools=[set_proverbs, get_weather]
)
```

### Example Conversations

```
User: "Fill out the contact form with my details"
Agent: fillContactForm(name="John Doe", email="john@example.com", ...)

User: "Test if login works with username 'test' and password 'pass123'"
Agent: testLoginFlow(username="test", password="pass123", expectedResult="success")

User: "Search for laptops under $1000"
Agent: searchAndFilterProducts(searchQuery="laptops", priceRange="0-1000")

User: "Make the text bigger, I can't read it"
Agent: increaseTextSize(percentage=150)

User: "Hide all the ads and sidebars"
Agent: enableFocusMode(enable=true)
```

---

## Next Steps

1. Copy the examples you need into your `src/app/components/` directory
2. Import and add them to your layout
3. Update your agent instructions
4. Test with natural language commands

For more information, see [TUTORIAL.md](./TUTORIAL.md).

