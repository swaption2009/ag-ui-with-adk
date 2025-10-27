# Complete Tutorial: DOM Understanding & Manipulation with ADK Agent

## 📚 Table of Contents

1. [Introduction](#introduction)
2. [How the System Works](#how-the-system-works)
3. [Prerequisites](#prerequisites)
4. [Part 1: Understanding the Architecture](#part-1-understanding-the-architecture)
5. [Part 2: DOM Understanding (Reading the Screen)](#part-2-dom-understanding-reading-the-screen)
6. [Part 3: DOM Manipulation (Controlling the Screen)](#part-3-dom-manipulation-controlling-the-screen)
7. [Part 4: Advanced Features](#part-4-advanced-features)
8. [Part 5: Real-World Examples](#part-5-real-world-examples)
9. [Security Best Practices](#security-best-practices)
10. [Troubleshooting](#troubleshooting)

---

## Introduction

This tutorial will teach you how to use **Google's ADK (Agent Development Kit)** with **CopilotKit** to create AI agents that can:
- 🔍 **Understand** web page content (DOM reading)
- 🎮 **Manipulate** web page elements (DOM manipulation)
- 🤖 **Interact** with users through natural language
- 🔄 **Maintain state** across conversations

By the end of this tutorial, you'll be able to build AI agents that can interact with any React application or even wrap external websites.

---

## How the System Works

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     USER INTERACTION                        │
│              "Click the submit button"                      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  FRONTEND (React/Next.js)                   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  CopilotSidebar - User Interface                     │   │
│  │  useCopilotAction - DOM Manipulation Handlers        │   │
│  │  useCoAgent - Shared State Management                │   │
│  │  useCopilotReadable - Screen Context Provider        │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              API GATEWAY (Next.js API Route)                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  CopilotRuntime - Message Routing                    │   │
│  │  HttpAgent - Backend Communication                   │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              BACKEND (Python/FastAPI + ADK)                 │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  ADKAgent Wrapper - CopilotKit Integration           │   │
│  │  LlmAgent - Core Agent Logic                         │   │
│  │  Tools - Custom Functions (set_proverbs, etc)        │   │
│  │  Callbacks - State & Prompt Management               │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   GOOGLE GEMINI 2.5 FLASH                   │
│              (Natural Language Understanding)               │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow Example

**User Request:** "Add a proverb about AI"

1. **Frontend**: User types in `CopilotSidebar`
2. **API Gateway**: Routes message to ADK agent via `HttpAgent`
3. **ADK Agent**: Receives message, injects current state into prompt
4. **Gemini**: Processes request, decides to call `set_proverbs` tool
5. **Tool Execution**: Updates agent state with new proverb
6. **Response Chain**: Success message flows back through the stack
7. **Frontend Update**: `useCoAgent` hook detects state change, React re-renders
8. **UI Update**: New proverb appears on screen

---

## Prerequisites

Before starting, ensure you have:

- ✅ Node.js 18+
- ✅ Python 3.12+
- ✅ Google Makersuite API Key ([Get one here](https://makersuite.google.com/app/apikey))
- ✅ Package manager (pnpm, npm, yarn, or bun)
- ✅ Basic knowledge of React and Python

### Initial Setup

1. **Clone the repository**
```bash
git clone <your-repo-url>
cd ag-ui-with-adk
```

2. **Install dependencies**
```bash
# Install Node.js dependencies
npm install

# Install Python dependencies
npm run install:agent
```

3. **Set up environment variables**
```bash
export GOOGLE_API_KEY="your-google-api-key-here"
```

4. **Start the development servers**
```bash
npm run dev
```

This will start:
- Next.js UI server on `http://localhost:3000`
- Python ADK agent server on `http://localhost:8000`

---

## Part 1: Understanding the Architecture

### 1.1 Frontend Components

#### **CopilotKit Provider** (`src/app/layout.tsx`)

```typescript
import { CopilotKit } from "@copilotkit/react-core";

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <CopilotKit 
          runtimeUrl="/api/copilotkit"  // API endpoint
          agent="my_agent"                // Agent name
        >
          {children}
        </CopilotKit>
      </body>
    </html>
  );
}
```

**What it does:**
- Wraps your entire app with CopilotKit context
- Connects to the backend agent via API route
- Manages WebSocket/HTTP communication

#### **API Gateway** (`src/app/api/copilotkit/route.ts`)

```typescript
import { CopilotRuntime, ExperimentalEmptyAdapter, copilotRuntimeNextJSAppRouterEndpoint } from "@copilotkit/runtime";
import { HttpAgent } from "@ag-ui/client";

const runtime = new CopilotRuntime({
  agents: {
    "my_agent": new HttpAgent({url: "http://localhost:8000/"})
  }   
});

export const POST = async (req: NextRequest) => {
  const { handleRequest } = copilotRuntimeNextJSAppRouterEndpoint({
    runtime,
    serviceAdapter: new ExperimentalEmptyAdapter(),
    endpoint: "/api/copilotkit",
  });
  return handleRequest(req);
};
```

**What it does:**
- Creates a bridge between frontend and backend
- Routes messages to the Python ADK agent
- Handles session management

### 1.2 Backend Components

#### **ADK Agent Setup** (`agent/agent.py`)

```python
from fastapi import FastAPI
from ag_ui_adk import ADKAgent, add_adk_fastapi_endpoint
from google.adk.agents import LlmAgent

# Define the agent
proverbs_agent = LlmAgent(
    name="ProverbsAgent",
    model="gemini-2.5-flash",
    instruction="You are a helpful assistant...",
    tools=[set_proverbs, get_weather],
    before_agent_callback=on_before_agent,
    before_model_callback=before_model_modifier,
    after_model_callback=simple_after_model_modifier
)

# Wrap for CopilotKit compatibility
adk_proverbs_agent = ADKAgent(
    adk_agent=proverbs_agent,
    app_name="proverbs_app",
    user_id="demo_user",
    session_timeout_seconds=3600,
    use_in_memory_services=True
)

# Create FastAPI app
app = FastAPI(title="ADK Middleware Proverbs Agent")
add_adk_fastapi_endpoint(app, adk_proverbs_agent, path="/")
```

**What it does:**
- Creates an LLM-powered agent using Gemini
- Wraps it for CopilotKit integration
- Exposes it as a FastAPI endpoint

---

## Part 2: DOM Understanding (Reading the Screen)

### 2.1 Basic Screen Context

Create a new file to provide screen context to the agent:

```typescript
// src/app/components/ScreenContext.tsx
"use client";

import { useCopilotReadable } from "@copilotkit/react-core";
import { useEffect, useState } from "react";

export function ScreenContextProvider({ children }: { children: React.ReactNode }) {
  const [pageContent, setPageContent] = useState("");
  
  useEffect(() => {
    // Capture page content
    const captureContent = () => {
      const content = {
        title: document.title,
        url: window.location.href,
        text: document.body.innerText.slice(0, 5000), // Limit to 5000 chars
        forms: document.querySelectorAll('form').length,
        buttons: document.querySelectorAll('button').length,
        links: document.querySelectorAll('a').length,
        inputs: document.querySelectorAll('input').length
      };
      setPageContent(JSON.stringify(content, null, 2));
    };
    
    captureContent();
    
    // Update on DOM changes
    const observer = new MutationObserver(captureContent);
    observer.observe(document.body, { 
      childList: true, 
      subtree: true 
    });
    
    return () => observer.disconnect();
  }, []);
  
  // Make content readable to agent
  useCopilotReadable({
    description: "Current page content and structure",
    value: pageContent
  });
  
  return <>{children}</>;
}
```

**Update your layout to include it:**

```typescript
// src/app/layout.tsx
import { ScreenContextProvider } from "./components/ScreenContext";

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <CopilotKit runtimeUrl="/api/copilotkit" agent="my_agent">
          <ScreenContextProvider>
            {children}
          </ScreenContextProvider>
        </CopilotKit>
      </body>
    </html>
  );
}
```

### 2.2 Advanced Screen Understanding

For more detailed screen analysis:

```typescript
// src/app/components/AdvancedScreenContext.tsx
"use client";

import { useCopilotReadable, useCopilotAction } from "@copilotkit/react-core";

export function AdvancedScreenContext() {
  // Provide structured page information
  useCopilotReadable({
    description: "List of all interactive elements on the page",
    value: JSON.stringify({
      buttons: Array.from(document.querySelectorAll('button')).map((btn, i) => ({
        index: i,
        text: btn.textContent?.trim(),
        id: btn.id,
        classes: btn.className,
        selector: btn.id ? `#${btn.id}` : `.${btn.className.split(' ')[0]}`
      })),
      inputs: Array.from(document.querySelectorAll('input')).map((input, i) => ({
        index: i,
        type: input.type,
        name: input.name,
        id: input.id,
        placeholder: input.placeholder,
        selector: input.id ? `#${input.id}` : `input[name="${input.name}"]`
      })),
      links: Array.from(document.querySelectorAll('a')).map((link, i) => ({
        index: i,
        text: link.textContent?.trim(),
        href: link.href,
        selector: link.id ? `#${link.id}` : `a[href="${link.getAttribute('href')}"]`
      }))
    })
  });

  // Action to get specific element information
  useCopilotAction({
    name: "getElementInfo",
    description: "Get detailed information about a specific element on the page",
    parameters: [
      {
        name: "selector",
        type: "string",
        description: "CSS selector of the element to inspect",
        required: true
      }
    ],
    handler({ selector }) {
      const element = document.querySelector(selector);
      if (!element) {
        return { success: false, message: `Element not found: ${selector}` };
      }
      
      return {
        success: true,
        tagName: element.tagName,
        text: element.textContent?.trim(),
        html: element.innerHTML,
        attributes: Array.from(element.attributes).map(attr => ({
          name: attr.name,
          value: attr.value
        })),
        styles: window.getComputedStyle(element).cssText,
        boundingBox: element.getBoundingClientRect(),
        isVisible: element.checkVisibility?.() ?? true
      };
    }
  });

  return null;
}
```

### 2.3 Update Backend Agent to Use Screen Context

```python
# agent/agent.py - Update the agent instruction
proverbs_agent = LlmAgent(
    name="ProverbsAgent",
    model="gemini-2.5-flash",
    instruction=f"""
    You are a helpful assistant that can understand and interact with web pages.
    
    You have access to the current page's content through the screen context.
    This includes:
    - Page title and URL
    - All text content on the page
    - List of buttons, inputs, links, and forms
    - Element selectors for interaction
    
    When users ask about the page, analyze the screen context first.
    When users ask you to interact with elements, use the appropriate tools.
    
    Examples:
    - "What's on this page?" → Analyze the screen context and summarize
    - "How many buttons are there?" → Count buttons from screen context
    - "What forms are available?" → List forms from screen context
    """,
    tools=[set_proverbs, get_weather],
)
```

---

## Part 3: DOM Manipulation (Controlling the Screen)

### 3.1 Basic DOM Manipulation

Create a comprehensive DOM manipulation component:

```typescript
// src/app/components/DOMManipulator.tsx
"use client";

import { useCopilotAction } from "@copilotkit/react-core";

export function DOMManipulator() {
  
  // 1. CLICK ELEMENTS
  useCopilotAction({
    name: "clickElement",
    description: "Click any element on the page by CSS selector (e.g., '#submit-btn', '.login-button', 'button[type=\"submit\"]')",
    parameters: [
      {
        name: "selector",
        type: "string",
        description: "CSS selector of the element to click",
        required: true,
      }
    ],
    handler({ selector }) {
      try {
        const element = document.querySelector(selector);
        if (element instanceof HTMLElement) {
          element.click();
          return { 
            success: true, 
            message: `Successfully clicked element: ${selector}` 
          };
        }
        return { 
          success: false, 
          message: `Element not found: ${selector}` 
        };
      } catch (error) {
        return { 
          success: false, 
          message: `Error: ${error.message}` 
        };
      }
    },
  });

  // 2. FILL FORM FIELDS
  useCopilotAction({
    name: "fillInput",
    description: "Fill an input field or textarea with a value",
    parameters: [
      {
        name: "selector",
        type: "string",
        description: "CSS selector of the input field",
        required: true,
      },
      {
        name: "value",
        type: "string",
        description: "Value to fill in the field",
        required: true,
      }
    ],
    handler({ selector, value }) {
      try {
        const element = document.querySelector(selector);
        if (element instanceof HTMLInputElement || element instanceof HTMLTextAreaElement) {
          element.value = value;
          // Trigger events for React and other frameworks
          element.dispatchEvent(new Event('input', { bubbles: true }));
          element.dispatchEvent(new Event('change', { bubbles: true }));
          return { 
            success: true, 
            message: `Filled ${selector} with: ${value}` 
          };
        }
        return { 
          success: false, 
          message: `Input field not found: ${selector}` 
        };
      } catch (error) {
        return { 
          success: false, 
          message: `Error: ${error.message}` 
        };
      }
    },
  });

  // 3. SCROLL TO ELEMENT
  useCopilotAction({
    name: "scrollToElement",
    description: "Scroll to a specific element on the page",
    parameters: [
      {
        name: "selector",
        type: "string",
        description: "CSS selector of the element to scroll to",
        required: true,
      }
    ],
    handler({ selector }) {
      try {
        const element = document.querySelector(selector);
        if (element) {
          element.scrollIntoView({ 
            behavior: 'smooth', 
            block: 'center' 
          });
          return { 
            success: true, 
            message: `Scrolled to: ${selector}` 
          };
        }
        return { 
          success: false, 
          message: `Element not found: ${selector}` 
        };
      } catch (error) {
        return { 
          success: false, 
          message: `Error: ${error.message}` 
        };
      }
    },
  });

  // 4. MODIFY ELEMENT STYLES
  useCopilotAction({
    name: "modifyStyle",
    description: "Change the CSS style of any element",
    parameters: [
      {
        name: "selector",
        type: "string",
        description: "CSS selector of the element",
        required: true,
      },
      {
        name: "property",
        type: "string",
        description: "CSS property to change (e.g., 'backgroundColor', 'display', 'fontSize')",
        required: true,
      },
      {
        name: "value",
        type: "string",
        description: "New value for the CSS property",
        required: true,
      }
    ],
    handler({ selector, property, value }) {
      try {
        const element = document.querySelector(selector);
        if (element instanceof HTMLElement) {
          element.style[property as any] = value;
          return { 
            success: true, 
            message: `Set ${property} to ${value} on ${selector}` 
          };
        }
        return { 
          success: false, 
          message: `Element not found: ${selector}` 
        };
      } catch (error) {
        return { 
          success: false, 
          message: `Error: ${error.message}` 
        };
      }
    },
  });

  // 5. ADD/REMOVE CSS CLASSES
  useCopilotAction({
    name: "toggleClass",
    description: "Add, remove, or toggle a CSS class on an element",
    parameters: [
      {
        name: "selector",
        type: "string",
        description: "CSS selector of the element",
        required: true,
      },
      {
        name: "className",
        type: "string",
        description: "CSS class name to manipulate",
        required: true,
      },
      {
        name: "action",
        type: "string",
        description: "Action to perform: 'add', 'remove', or 'toggle'",
        required: true,
      }
    ],
    handler({ selector, className, action }) {
      try {
        const element = document.querySelector(selector);
        if (element) {
          if (action === 'add') {
            element.classList.add(className);
          } else if (action === 'remove') {
            element.classList.remove(className);
          } else if (action === 'toggle') {
            element.classList.toggle(className);
          } else {
            return { 
              success: false, 
              message: `Invalid action: ${action}. Use 'add', 'remove', or 'toggle'` 
            };
          }
          return { 
            success: true, 
            message: `${action} class '${className}' on ${selector}` 
          };
        }
        return { 
          success: false, 
          message: `Element not found: ${selector}` 
        };
      } catch (error) {
        return { 
          success: false, 
          message: `Error: ${error.message}` 
        };
      }
    },
  });

  // 6. UPDATE TEXT CONTENT
  useCopilotAction({
    name: "updateText",
    description: "Update the text content of an element",
    parameters: [
      {
        name: "selector",
        type: "string",
        description: "CSS selector of the element",
        required: true,
      },
      {
        name: "text",
        type: "string",
        description: "New text content",
        required: true,
      }
    ],
    handler({ selector, text }) {
      try {
        const element = document.querySelector(selector);
        if (element) {
          element.textContent = text;
          return { 
            success: true, 
            message: `Updated text in ${selector}` 
          };
        }
        return { 
          success: false, 
          message: `Element not found: ${selector}` 
        };
      } catch (error) {
        return { 
          success: false, 
          message: `Error: ${error.message}` 
        };
      }
    },
  });

  // 7. SHOW/HIDE ELEMENTS
  useCopilotAction({
    name: "toggleVisibility",
    description: "Show or hide an element",
    parameters: [
      {
        name: "selector",
        type: "string",
        description: "CSS selector of the element",
        required: true,
      },
      {
        name: "visible",
        type: "boolean",
        description: "true to show, false to hide",
        required: true,
      }
    ],
    handler({ selector, visible }) {
      try {
        const element = document.querySelector(selector);
        if (element instanceof HTMLElement) {
          element.style.display = visible ? '' : 'none';
          return { 
            success: true, 
            message: `${visible ? 'Showed' : 'Hidden'} ${selector}` 
          };
        }
        return { 
          success: false, 
          message: `Element not found: ${selector}` 
        };
      } catch (error) {
        return { 
          success: false, 
          message: `Error: ${error.message}` 
        };
      }
    },
  });

  // 8. NAVIGATE TO URL
  useCopilotAction({
    name: "navigateToUrl",
    description: "Navigate to a different URL",
    parameters: [
      {
        name: "url",
        type: "string",
        description: "URL to navigate to (can be relative or absolute)",
        required: true,
      }
    ],
    handler({ url }) {
      try {
        window.location.href = url;
        return { 
          success: true, 
          message: `Navigating to: ${url}` 
        };
      } catch (error) {
        return { 
          success: false, 
          message: `Error: ${error.message}` 
        };
      }
    },
  });

  return null; // This component only registers actions
}
```

### 3.2 Add to Your Layout

```typescript
// src/app/layout.tsx
import { DOMManipulator } from "./components/DOMManipulator";
import { ScreenContextProvider } from "./components/ScreenContext";
import { AdvancedScreenContext } from "./components/AdvancedScreenContext";

export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <CopilotKit runtimeUrl="/api/copilotkit" agent="my_agent">
          <ScreenContextProvider>
            <AdvancedScreenContext />
            <DOMManipulator />
            {children}
          </ScreenContextProvider>
        </CopilotKit>
      </body>
    </html>
  );
}
```

### 3.3 Update Backend Agent Instructions

```python
# agent/agent.py - Enhanced agent with DOM manipulation
proverbs_agent = LlmAgent(
    name="ProverbsAgent",
    model="gemini-2.5-flash",
    instruction=f"""
    You are a helpful assistant that can understand and interact with web pages.
    
    AVAILABLE DOM MANIPULATION TOOLS:
    
    1. clickElement(selector) - Click any button, link, or element
       Example: clickElement("#submit-btn")
    
    2. fillInput(selector, value) - Fill form fields with values
       Example: fillInput("input[name='email']", "user@example.com")
    
    3. scrollToElement(selector) - Scroll to specific elements
       Example: scrollToElement("#pricing-section")
    
    4. modifyStyle(selector, property, value) - Change element styles
       Example: modifyStyle("body", "backgroundColor", "lightblue")
    
    5. toggleClass(selector, className, action) - Add/remove CSS classes
       Example: toggleClass(".modal", "hidden", "remove")
    
    6. updateText(selector, text) - Change element text content
       Example: updateText("h1", "New Title")
    
    7. toggleVisibility(selector, visible) - Show or hide elements
       Example: toggleVisibility(".popup", false)
    
    8. navigateToUrl(url) - Navigate to different pages
       Example: navigateToUrl("/about")
    
    9. getElementInfo(selector) - Get detailed element information
       Example: getElementInfo("#main-form")
    
    SCREEN UNDERSTANDING:
    - You have access to the current page's content and structure
    - Use the screen context to understand what's on the page
    - Always check element selectors before manipulating them
    
    BEST PRACTICES:
    - Use specific CSS selectors (IDs are best: #element-id)
    - Verify elements exist before manipulating them
    - Provide clear feedback to users about what you're doing
    - Handle errors gracefully
    
    EXAMPLE INTERACTIONS:
    
    User: "Fill out the contact form"
    You: 
      1. Analyze screen context to find form fields
      2. fillInput("#name", "John Doe")
      3. fillInput("#email", "john@example.com")
      4. fillInput("#message", "Hello!")
      5. clickElement("#submit-button")
    
    User: "Make the text bigger"
    You: modifyStyle("body", "fontSize", "18px")
    
    User: "Hide the popup"
    You: toggleVisibility(".popup-modal", false)
    
    User: "What buttons are on this page?"
    You: Analyze screen context and list all buttons with their text and selectors
    
    PROVERBS MANAGEMENT:
    {proverbs_json}
    When managing proverbs, use the set_proverbs tool.
    
    WEATHER INFORMATION:
    When asked about weather, use the get_weather tool.
    """,
    tools=[set_proverbs, get_weather],
    before_agent_callback=on_before_agent,
    before_model_callback=before_model_modifier,
    after_model_callback=simple_after_model_modifier
)
```

---

## Part 4: Advanced Features

### 4.1 Complex Multi-Step Workflows

```typescript
// src/app/components/AdvancedWorkflows.tsx
"use client";

import { useCopilotAction } from "@copilotkit/react-core";

export function AdvancedWorkflows() {
  
  // Complete form automation
  useCopilotAction({
    name: "completeForm",
    description: "Automatically complete a form with provided data",
    parameters: [
      {
        name: "formSelector",
        type: "string",
        description: "CSS selector of the form",
        required: true,
      },
      {
        name: "formData",
        type: "object",
        description: "Object with field names as keys and values to fill",
        required: true,
      }
    ],
    async handler({ formSelector, formData }) {
      try {
        const form = document.querySelector(formSelector);
        if (!form) {
          return { success: false, message: `Form not found: ${formSelector}` };
        }

        const results = [];
        
        // Fill each field
        for (const [fieldName, value] of Object.entries(formData)) {
          const field = form.querySelector(`[name="${fieldName}"]`) as HTMLInputElement;
          if (field) {
            field.value = value as string;
            field.dispatchEvent(new Event('input', { bubbles: true }));
            field.dispatchEvent(new Event('change', { bubbles: true }));
            results.push(`Filled ${fieldName}`);
          } else {
            results.push(`Field not found: ${fieldName}`);
          }
        }

        return { 
          success: true, 
          message: `Form completed: ${results.join(', ')}`,
          details: results
        };
      } catch (error) {
        return { success: false, message: `Error: ${error.message}` };
      }
    },
  });

  // Extract all data from a table
  useCopilotAction({
    name: "extractTableData",
    description: "Extract all data from a table on the page",
    parameters: [
      {
        name: "tableSelector",
        type: "string",
        description: "CSS selector of the table",
        required: true,
      }
    ],
    handler({ tableSelector }) {
      try {
        const table = document.querySelector(tableSelector);
        if (!table) {
          return { success: false, message: `Table not found: ${tableSelector}` };
        }

        const headers = Array.from(table.querySelectorAll('th')).map(th => th.textContent?.trim());
        const rows = Array.from(table.querySelectorAll('tbody tr')).map(tr => 
          Array.from(tr.querySelectorAll('td')).map(td => td.textContent?.trim())
        );

        return {
          success: true,
          headers,
          rows,
          rowCount: rows.length
        };
      } catch (error) {
        return { success: false, message: `Error: ${error.message}` };
      }
    },
  });

  // Automated testing workflow
  useCopilotAction({
    name: "testWorkflow",
    description: "Execute a series of test steps on the page",
    parameters: [
      {
        name: "steps",
        type: "array",
        description: "Array of test steps to execute",
        required: true,
      }
    ],
    async handler({ steps }) {
      const results = [];
      
      for (const step of steps as any[]) {
        try {
          await new Promise(resolve => setTimeout(resolve, 500)); // Wait between steps
          
          if (step.action === 'click') {
            document.querySelector(step.selector)?.click();
            results.push({ step: step.name, success: true });
          } else if (step.action === 'fill') {
            const input = document.querySelector(step.selector) as HTMLInputElement;
            if (input) {
              input.value = step.value;
              input.dispatchEvent(new Event('input', { bubbles: true }));
              results.push({ step: step.name, success: true });
            }
          } else if (step.action === 'verify') {
            const element = document.querySelector(step.selector);
            const exists = !!element;
            results.push({ step: step.name, success: exists, found: exists });
          }
        } catch (error) {
          results.push({ step: step.name, success: false, error: error.message });
        }
      }

      return {
        success: true,
        totalSteps: steps.length,
        passed: results.filter(r => r.success).length,
        failed: results.filter(r => !r.success).length,
        results
      };
    },
  });

  return null;
}
```

### 4.2 Screenshot Capture (Optional)

For visual understanding, you can add screenshot capabilities:

```typescript
// src/app/components/ScreenshotCapture.tsx
"use client";

import { useCopilotAction } from "@copilotkit/react-core";

export function ScreenshotCapture() {
  useCopilotAction({
    name: "captureScreenshot",
    description: "Capture a screenshot of the current page or a specific element",
    parameters: [
      {
        name: "selector",
        type: "string",
        description: "CSS selector of element to capture (leave empty for full page)",
        required: false,
      }
    ],
    async handler({ selector }) {
      try {
        // Note: You'll need to install html2canvas
        // npm install html2canvas
        const html2canvas = (await import('html2canvas')).default;
        
        const element = selector ? document.querySelector(selector) : document.body;
        if (!element) {
          return { success: false, message: `Element not found: ${selector}` };
        }

        const canvas = await html2canvas(element as HTMLElement);
        const imageData = canvas.toDataURL('image/png');
        
        // You can send this to your backend for analysis
        return {
          success: true,
          message: "Screenshot captured",
          imageData: imageData.substring(0, 100) + "...", // Truncate for response
          width: canvas.width,
          height: canvas.height
        };
      } catch (error) {
        return { success: false, message: `Error: ${error.message}` };
      }
    },
  });

  return null;
}
```

---

## Part 5: Real-World Examples

### 5.1 E-commerce Assistant

```typescript
// src/app/components/EcommerceAssistant.tsx
"use client";

import { useCopilotAction } from "@copilotkit/react-core";

export function EcommerceAssistant() {
  
  useCopilotAction({
    name: "addToCart",
    description: "Add a product to the shopping cart",
    parameters: [
      {
        name: "productSelector",
        type: "string",
        description: "CSS selector of the product",
        required: true,
      }
    ],
    handler({ productSelector }) {
      const addButton = document.querySelector(`${productSelector} .add-to-cart-btn`);
      if (addButton instanceof HTMLElement) {
        addButton.click();
        return { success: true, message: "Product added to cart" };
      }
      return { success: false, message: "Add to cart button not found" };
    },
  });

  useCopilotAction({
    name: "searchProducts",
    description: "Search for products on the page",
    parameters: [
      {
        name: "query",
        type: "string",
        description: "Search query",
        required: true,
      }
    ],
    handler({ query }) {
      const searchInput = document.querySelector('input[type="search"]') as HTMLInputElement;
      const searchButton = document.querySelector('button[type="submit"]') as HTMLButtonElement;
      
      if (searchInput && searchButton) {
        searchInput.value = query;
        searchInput.dispatchEvent(new Event('input', { bubbles: true }));
        searchButton.click();
        return { success: true, message: `Searching for: ${query}` };
      }
      return { success: false, message: "Search form not found" };
    },
  });

  useCopilotAction({
    name: "filterProducts",
    description: "Apply filters to product listings",
    parameters: [
      {
        name: "filterType",
        type: "string",
        description: "Type of filter (e.g., 'price', 'category', 'brand')",
        required: true,
      },
      {
        name: "filterValue",
        type: "string",
        description: "Filter value to apply",
        required: true,
      }
    ],
    handler({ filterType, filterValue }) {
      const checkbox = document.querySelector(`input[data-filter="${filterType}"][value="${filterValue}"]`) as HTMLInputElement;
      if (checkbox) {
        checkbox.checked = true;
        checkbox.dispatchEvent(new Event('change', { bubbles: true }));
        return { success: true, message: `Applied filter: ${filterType} = ${filterValue}` };
      }
      return { success: false, message: "Filter not found" };
    },
  });

  return null;
}
```

### 5.2 Form Assistant

```typescript
// src/app/components/FormAssistant.tsx
"use client";

import { useCopilotAction } from "@copilotkit/react-core";

export function FormAssistant() {
  
  useCopilotAction({
    name: "fillContactForm",
    description: "Fill out a contact form with user details",
    parameters: [
      { name: "name", type: "string", required: true },
      { name: "email", type: "string", required: true },
      { name: "phone", type: "string", required: false },
      { name: "message", type: "string", required: true }
    ],
    handler({ name, email, phone, message }) {
      const fields = {
        'input[name="name"]': name,
        'input[name="email"]': email,
        'input[name="phone"]': phone,
        'textarea[name="message"]': message
      };

      const results = [];
      for (const [selector, value] of Object.entries(fields)) {
        if (value) {
          const field = document.querySelector(selector) as HTMLInputElement;
          if (field) {
            field.value = value;
            field.dispatchEvent(new Event('input', { bubbles: true }));
            results.push(`Filled ${selector}`);
          }
        }
      }

      return { 
        success: true, 
        message: `Contact form filled: ${results.join(', ')}` 
      };
    },
  });

  useCopilotAction({
    name: "validateForm",
    description: "Check if a form is valid and ready to submit",
    parameters: [
      {
        name: "formSelector",
        type: "string",
        description: "CSS selector of the form",
        required: true,
      }
    ],
    handler({ formSelector }) {
      const form = document.querySelector(formSelector) as HTMLFormElement;
      if (!form) {
        return { success: false, message: "Form not found" };
      }

      const isValid = form.checkValidity();
      const invalidFields = Array.from(form.querySelectorAll(':invalid')).map(
        field => (field as HTMLInputElement).name
      );

      return {
        success: true,
        isValid,
        invalidFields,
        message: isValid ? "Form is valid" : `Invalid fields: ${invalidFields.join(', ')}`
      };
    },
  });

  return null;
}
```

### 5.3 Testing Assistant

```typescript
// src/app/components/TestingAssistant.tsx
"use client";

import { useCopilotAction } from "@copilotkit/react-core";

export function TestingAssistant() {
  
  useCopilotAction({
    name: "testLoginFlow",
    description: "Test the login functionality",
    parameters: [
      { name: "username", type: "string", required: true },
      { name: "password", type: "string", required: true }
    ],
    async handler({ username, password }) {
      const steps = [];
      
      try {
        // Step 1: Fill username
        const usernameField = document.querySelector('input[name="username"]') as HTMLInputElement;
        if (usernameField) {
          usernameField.value = username;
          steps.push({ step: "Fill username", success: true });
        } else {
          steps.push({ step: "Fill username", success: false, error: "Field not found" });
        }

        // Step 2: Fill password
        const passwordField = document.querySelector('input[name="password"]') as HTMLInputElement;
        if (passwordField) {
          passwordField.value = password;
          steps.push({ step: "Fill password", success: true });
        } else {
          steps.push({ step: "Fill password", success: false, error: "Field not found" });
        }

        // Step 3: Click login button
        await new Promise(resolve => setTimeout(resolve, 500));
        const loginButton = document.querySelector('button[type="submit"]') as HTMLButtonElement;
        if (loginButton) {
          loginButton.click();
          steps.push({ step: "Click login", success: true });
        } else {
          steps.push({ step: "Click login", success: false, error: "Button not found" });
        }

        // Step 4: Check for success/error messages
        await new Promise(resolve => setTimeout(resolve, 2000));
        const errorMessage = document.querySelector('.error-message');
        const successMessage = document.querySelector('.success-message');
        
        if (errorMessage) {
          steps.push({ 
            step: "Login result", 
            success: false, 
            error: errorMessage.textContent 
          });
        } else if (successMessage) {
          steps.push({ 
            step: "Login result", 
            success: true, 
            message: successMessage.textContent 
          });
        }

        return {
          success: true,
          testSteps: steps,
          passed: steps.filter(s => s.success).length,
          failed: steps.filter(s => !s.success).length
        };
      } catch (error) {
        return { success: false, message: `Error: ${error.message}`, steps };
      }
    },
  });

  return null;
}
```

---

## Security Best Practices

### 6.1 Whitelist Allowed Selectors

```typescript
// src/app/components/SecureDOMManipulator.tsx
"use client";

import { useCopilotAction } from "@copilotkit/react-core";

const ALLOWED_SELECTORS = [
  '#submit-button',
  '.safe-button',
  'input[name="email"]',
  'input[name="name"]',
  // Add your allowed selectors here
];

const BLOCKED_TAGS = ['SCRIPT', 'IFRAME', 'OBJECT', 'EMBED'];

function isAllowedSelector(selector: string): boolean {
  return ALLOWED_SELECTORS.some(allowed => selector.includes(allowed));
}

function isBlockedElement(element: Element): boolean {
  return BLOCKED_TAGS.includes(element.tagName);
}

export function SecureDOMManipulator() {
  
  useCopilotAction({
    name: "secureClickElement",
    description: "Securely click an element (only allowed selectors)",
    parameters: [
      {
        name: "selector",
        type: "string",
        description: "CSS selector of the element to click",
        required: true,
      }
    ],
    handler({ selector }) {
      // Security check 1: Whitelist
      if (!isAllowedSelector(selector)) {
        console.warn(`Blocked selector: ${selector}`);
        return { 
          success: false, 
          message: "Selector not allowed for security reasons" 
        };
      }

      const element = document.querySelector(selector);
      if (!element) {
        return { success: false, message: "Element not found" };
      }

      // Security check 2: Block dangerous elements
      if (isBlockedElement(element)) {
        console.warn(`Blocked element type: ${element.tagName}`);
        return { 
          success: false, 
          message: "Cannot interact with this element type" 
        };
      }

      // Log action for audit
      console.log(`Agent clicked: ${selector} at ${new Date().toISOString()}`);

      if (element instanceof HTMLElement) {
        element.click();
        return { success: true, message: `Clicked: ${selector}` };
      }

      return { success: false, message: "Element is not clickable" };
    },
  });

  return null;
}
```

### 6.2 Rate Limiting

```typescript
// src/app/components/RateLimitedActions.tsx
"use client";

import { useCopilotAction } from "@copilotkit/react-core";
import { useState, useRef } from "react";

export function RateLimitedActions() {
  const actionCounts = useRef<Map<string, number>>(new Map());
  const MAX_ACTIONS_PER_MINUTE = 10;

  function checkRateLimit(actionName: string): boolean {
    const now = Date.now();
    const key = `${actionName}-${Math.floor(now / 60000)}`;
    const count = actionCounts.current.get(key) || 0;
    
    if (count >= MAX_ACTIONS_PER_MINUTE) {
      return false;
    }
    
    actionCounts.current.set(key, count + 1);
    return true;
  }

  useCopilotAction({
    name: "rateLimitedClick",
    description: "Click an element (rate limited)",
    parameters: [
      { name: "selector", type: "string", required: true }
    ],
    handler({ selector }) {
      if (!checkRateLimit("click")) {
        return { 
          success: false, 
          message: "Rate limit exceeded. Please wait a moment." 
        };
      }

      const element = document.querySelector(selector);
      if (element instanceof HTMLElement) {
        element.click();
        return { success: true };
      }
      return { success: false };
    },
  });

  return null;
}
```

### 6.3 User Confirmation for Sensitive Actions

```typescript
// src/app/components/ConfirmableActions.tsx
"use client";

import { useCopilotAction } from "@copilotkit/react-core";

export function ConfirmableActions() {
  
  useCopilotAction({
    name: "deleteItem",
    description: "Delete an item (requires user confirmation)",
    parameters: [
      { name: "itemId", type: "string", required: true }
    ],
    async handler({ itemId }) {
      // Request user confirmation
      const confirmed = window.confirm(
        `Are you sure you want to delete item ${itemId}?`
      );

      if (!confirmed) {
        return { 
          success: false, 
          message: "Action cancelled by user" 
        };
      }

      // Proceed with deletion
      const deleteButton = document.querySelector(`[data-delete-id="${itemId}"]`);
      if (deleteButton instanceof HTMLElement) {
        deleteButton.click();
        return { success: true, message: `Deleted item ${itemId}` };
      }

      return { success: false, message: "Delete button not found" };
    },
  });

  return null;
}
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. Agent Not Responding

**Problem:** The agent doesn't respond to messages.

**Solutions:**
```bash
# Check if both servers are running
ps aux | grep "next dev"
ps aux | grep "python agent.py"

# Restart the agent server
cd agent
source .venv/bin/activate
python agent.py

# Check API key
echo $GOOGLE_API_KEY
```

#### 2. DOM Manipulation Not Working

**Problem:** Agent can't click or modify elements.

**Solutions:**
```typescript
// Add debugging to your handlers
handler({ selector }) {
  console.log("Attempting to click:", selector);
  const element = document.querySelector(selector);
  console.log("Element found:", element);
  
  if (element instanceof HTMLElement) {
    console.log("Element is clickable");
    element.click();
    return { success: true };
  }
  
  console.error("Element not found or not clickable");
  return { success: false };
}
```

#### 3. Cross-Origin Issues

**Problem:** Can't access iframe content.

**Solution:**
```typescript
// Use postMessage for cross-origin communication
const iframe = document.querySelector('iframe');
iframe?.contentWindow?.postMessage({
  action: 'click',
  selector: '#button'
}, '*');
```

#### 4. State Not Syncing

**Problem:** Frontend and backend state out of sync.

**Solution:**
```typescript
// Force state refresh
const { state, setState } = useCoAgent({
  name: "my_agent",
  initialState: { proverbs: [] }
});

// Manually sync
useEffect(() => {
  // Fetch latest state from backend
  fetch('/api/copilotkit/state')
    .then(res => res.json())
    .then(data => setState(data));
}, []);
```

#### 5. Performance Issues

**Problem:** Page becomes slow with many actions.

**Solutions:**
```typescript
// Debounce screen context updates
import { debounce } from 'lodash';

const updateContext = debounce(() => {
  const content = capturePageContent();
  setPageContent(content);
}, 1000);

// Limit context size
const content = document.body.innerText.slice(0, 5000); // Only first 5000 chars

// Use MutationObserver with throttling
const observer = new MutationObserver(
  debounce(updateContext, 500)
);
```

---

## Testing Your Implementation

### Test Checklist

#### ✅ DOM Understanding Tests

1. **Test screen context reading:**
   - Ask: "What's on this page?"
   - Ask: "How many buttons are there?"
   - Ask: "List all the input fields"

2. **Test element inspection:**
   - Ask: "What's in the main heading?"
   - Ask: "Show me information about the submit button"

#### ✅ DOM Manipulation Tests

1. **Test clicking:**
   - Ask: "Click the submit button"
   - Ask: "Click the first link on the page"

2. **Test form filling:**
   - Ask: "Fill the email field with test@example.com"
   - Ask: "Fill out the contact form"

3. **Test styling:**
   - Ask: "Make the background blue"
   - Ask: "Make the text bigger"
   - Ask: "Hide the popup"

4. **Test navigation:**
   - Ask: "Scroll to the footer"
   - Ask: "Go to the about page"

#### ✅ Complex Workflow Tests

1. **Test multi-step actions:**
   - Ask: "Complete the registration form with my details"
   - Ask: "Search for 'laptop' and filter by price"

2. **Test error handling:**
   - Ask: "Click a button that doesn't exist"
   - Ask: "Fill a field that's not on the page"

---

## Next Steps

### Extend Your Agent

1. **Add More Tools:**
   - File upload handling
   - Drag and drop interactions
   - Keyboard shortcuts
   - Mouse hover effects

2. **Integrate External APIs:**
   - Weather data
   - Stock prices
   - News feeds
   - Database queries

3. **Add Vision Capabilities:**
   - Screenshot analysis
   - Image recognition
   - Visual element detection

4. **Build Browser Extension:**
   - Convert to Chrome extension
   - Work on any website
   - Persistent across pages

### Resources

- [CopilotKit Documentation](https://docs.copilotkit.ai)
- [Google ADK Documentation](https://google.github.io/adk-docs/)
- [AG-UI Protocol](https://developers.googleblog.com/en/delight-users-by-combining-adk-agents-with-fancy-frontends-using-ag-ui/)
- [Gemini API Documentation](https://ai.google.dev/docs)

---

## Conclusion

You now have a complete understanding of how to:
- ✅ Set up ADK agents with CopilotKit
- ✅ Enable agents to understand web page content
- ✅ Allow agents to manipulate the DOM
- ✅ Build complex multi-step workflows
- ✅ Implement security best practices

The combination of Google's ADK and CopilotKit provides a powerful framework for building AI agents that can truly interact with web applications. Experiment with the examples, extend the functionality, and build amazing AI-powered experiences!

---

**Happy Building! 🚀**

For questions or issues, refer to the [Troubleshooting](#troubleshooting) section or check the official documentation.

