# Quick Reference Guide: ADK + CopilotKit DOM Manipulation

## 🚀 Quick Start

```bash
# 1. Install dependencies
npm install
npm run install:agent

# 2. Set API key
export GOOGLE_API_KEY="your-key-here"

# 3. Start servers
npm run dev
```

---

## 📦 Essential Imports

### Frontend (TypeScript/React)
```typescript
import { useCopilotAction, useCoAgent, useCopilotReadable } from "@copilotkit/react-core";
import { CopilotKit, CopilotSidebar } from "@copilotkit/react-ui";
```

### Backend (Python)
```python
from ag_ui_adk import ADKAgent, add_adk_fastapi_endpoint
from google.adk.agents import LlmAgent
from google.adk.tools import ToolContext
```

---

## 🎯 Common Patterns

### 1. Click Element
```typescript
useCopilotAction({
  name: "clickElement",
  description: "Click any element by CSS selector",
  parameters: [{ name: "selector", type: "string", required: true }],
  handler({ selector }) {
    document.querySelector(selector)?.click();
    return { success: true };
  }
});
```

### 2. Fill Input
```typescript
useCopilotAction({
  name: "fillInput",
  description: "Fill an input field",
  parameters: [
    { name: "selector", type: "string", required: true },
    { name: "value", type: "string", required: true }
  ],
  handler({ selector, value }) {
    const input = document.querySelector(selector) as HTMLInputElement;
    if (input) {
      input.value = value;
      input.dispatchEvent(new Event('input', { bubbles: true }));
      return { success: true };
    }
    return { success: false };
  }
});
```

### 3. Modify Style
```typescript
useCopilotAction({
  name: "modifyStyle",
  description: "Change element CSS",
  parameters: [
    { name: "selector", type: "string", required: true },
    { name: "property", type: "string", required: true },
    { name: "value", type: "string", required: true }
  ],
  handler({ selector, property, value }) {
    const element = document.querySelector(selector) as HTMLElement;
    if (element) {
      element.style[property as any] = value;
      return { success: true };
    }
    return { success: false };
  }
});
```

### 4. Read Screen Context
```typescript
useCopilotReadable({
  description: "Current page content",
  value: JSON.stringify({
    title: document.title,
    url: window.location.href,
    text: document.body.innerText.slice(0, 5000)
  })
});
```

### 5. Shared State
```typescript
const { state, setState } = useCoAgent<AgentState>({
  name: "my_agent",
  initialState: { items: [] }
});
```

---

## 🔧 CSS Selectors Cheat Sheet

```typescript
// By ID
"#submit-button"

// By Class
".btn-primary"

// By Attribute
"input[name='email']"
"button[type='submit']"

// By Tag
"h1"
"form"

// Combinations
"form input[type='text']"
".container .card:first-child"
"#main-form button.submit"
```

---

## 🤖 Backend Agent Template

```python
from google.adk.agents import LlmAgent
from google.adk.tools import ToolContext
from typing import Dict

def my_tool(tool_context: ToolContext, param: str) -> Dict[str, str]:
    """Tool description for the agent."""
    # Your logic here
    return {"status": "success", "message": "Done"}

agent = LlmAgent(
    name="MyAgent",
    model="gemini-2.5-flash",
    instruction="""
    You are a helpful assistant.
    Use my_tool when the user asks for X.
    """,
    tools=[my_tool]
)
```

---

## 💡 Common Use Cases

### Form Automation
```typescript
// User: "Fill out the contact form"
useCopilotAction({
  name: "fillContactForm",
  parameters: [
    { name: "name", type: "string", required: true },
    { name: "email", type: "string", required: true },
    { name: "message", type: "string", required: true }
  ],
  handler({ name, email, message }) {
    (document.querySelector('input[name="name"]') as HTMLInputElement).value = name;
    (document.querySelector('input[name="email"]') as HTMLInputElement).value = email;
    (document.querySelector('textarea[name="message"]') as HTMLTextAreaElement).value = message;
    return { success: true };
  }
});
```

### UI Testing
```typescript
// User: "Test the login flow"
useCopilotAction({
  name: "testLogin",
  parameters: [
    { name: "username", type: "string", required: true },
    { name: "password", type: "string", required: true }
  ],
  async handler({ username, password }) {
    (document.querySelector('#username') as HTMLInputElement).value = username;
    (document.querySelector('#password') as HTMLInputElement).value = password;
    document.querySelector('#login-btn')?.click();
    
    await new Promise(r => setTimeout(r, 2000));
    
    const error = document.querySelector('.error-message');
    return { 
      success: !error, 
      message: error?.textContent || "Login successful" 
    };
  }
});
```

### Navigation Helper
```typescript
// User: "Go to the pricing section"
useCopilotAction({
  name: "scrollToSection",
  parameters: [{ name: "section", type: "string", required: true }],
  handler({ section }) {
    const element = document.querySelector(`#${section}`);
    element?.scrollIntoView({ behavior: 'smooth', block: 'center' });
    return { success: !!element };
  }
});
```

---

## 🔒 Security Checklist

```typescript
// ✅ Whitelist selectors
const ALLOWED = ['#safe-btn', '.approved-link'];
if (!ALLOWED.some(a => selector.includes(a))) {
  return { success: false, message: "Not allowed" };
}

// ✅ Block dangerous elements
const BLOCKED = ['SCRIPT', 'IFRAME', 'OBJECT'];
if (BLOCKED.includes(element.tagName)) {
  return { success: false, message: "Blocked element" };
}

// ✅ Log actions
console.log(`Agent action: ${actionName} at ${new Date().toISOString()}`);

// ✅ Rate limiting
const MAX_PER_MINUTE = 10;
if (actionCount > MAX_PER_MINUTE) {
  return { success: false, message: "Rate limit exceeded" };
}

// ✅ User confirmation for sensitive actions
const confirmed = window.confirm("Are you sure?");
if (!confirmed) return { success: false };
```

---

## 🐛 Debugging Tips

```typescript
// Add detailed logging
handler({ selector }) {
  console.log("Action:", "clickElement");
  console.log("Selector:", selector);
  
  const element = document.querySelector(selector);
  console.log("Element found:", element);
  console.log("Element type:", element?.tagName);
  console.log("Element visible:", element?.checkVisibility?.());
  
  if (element instanceof HTMLElement) {
    console.log("Clicking element...");
    element.click();
    console.log("Click successful");
    return { success: true };
  }
  
  console.error("Click failed");
  return { success: false, message: "Element not found or not clickable" };
}
```

---

## 📊 Testing Commands

```bash
# Test agent connection
curl http://localhost:8000/

# Check agent health
curl http://localhost:8000/health

# View agent logs
cd agent
tail -f agent.log

# Test frontend
curl http://localhost:3000/api/copilotkit
```

---

## 🎨 Example Conversations

### DOM Understanding
```
User: "What's on this page?"
Agent: Analyzes screen context and describes content

User: "How many buttons are there?"
Agent: Counts buttons from screen context

User: "List all input fields"
Agent: Lists all inputs with their names and types
```

### DOM Manipulation
```
User: "Click the submit button"
Agent: clickElement("#submit-btn")

User: "Fill the email field with test@example.com"
Agent: fillInput("input[name='email']", "test@example.com")

User: "Make the text bigger"
Agent: modifyStyle("body", "fontSize", "20px")

User: "Hide the popup"
Agent: toggleVisibility(".popup", false)
```

### Complex Workflows
```
User: "Complete the registration form"
Agent: 
  1. fillInput("#name", "John Doe")
  2. fillInput("#email", "john@example.com")
  3. fillInput("#password", "secure123")
  4. clickElement("#register-btn")

User: "Test the search feature"
Agent:
  1. fillInput("#search", "laptop")
  2. clickElement("#search-btn")
  3. Wait 2 seconds
  4. Check results count
```

---

## 📁 File Structure

```
ag-ui-with-adk/
├── src/
│   ├── app/
│   │   ├── api/copilotkit/route.ts    # API Gateway
│   │   ├── layout.tsx                  # CopilotKit Provider
│   │   ├── page.tsx                    # Main UI
│   │   └── components/
│   │       ├── ScreenContext.tsx       # Screen reading
│   │       ├── DOMManipulator.tsx      # DOM control
│   │       └── AdvancedWorkflows.tsx   # Complex actions
├── agent/
│   ├── agent.py                        # ADK Agent
│   └── requirements.txt                # Python deps
└── package.json                        # Node deps
```

---

## 🔗 Useful Links

- [Full Tutorial](./TUTORIAL.md)
- [CopilotKit Docs](https://docs.copilotkit.ai)
- [ADK Docs](https://google.github.io/adk-docs/)
- [Gemini API](https://ai.google.dev/docs)
- [AG-UI Protocol](https://developers.googleblog.com/en/delight-users-by-combining-adk-agents-with-fancy-frontends-using-ag-ui/)

---

## ⚡ Performance Tips

```typescript
// Debounce context updates
import { debounce } from 'lodash';
const updateContext = debounce(captureContent, 1000);

// Limit context size
const text = document.body.innerText.slice(0, 5000);

// Use efficient selectors
"#id"           // Fast
".class"        // Medium
"div > p"       // Slower
"*"             // Avoid

// Batch operations
const elements = document.querySelectorAll('.item');
elements.forEach(el => el.style.color = 'red'); // Better than individual queries
```

---

## 🆘 Common Errors

| Error | Solution |
|-------|----------|
| "Element not found" | Check selector syntax, ensure element exists |
| "CORS error" | Use proxy or enable CORS on backend |
| "Agent not responding" | Check GOOGLE_API_KEY, restart agent server |
| "State not syncing" | Verify useCoAgent hook is properly configured |
| "Rate limit exceeded" | Implement rate limiting, reduce request frequency |

---

**Quick Reference v1.0** | Last updated: 2025-10-19

