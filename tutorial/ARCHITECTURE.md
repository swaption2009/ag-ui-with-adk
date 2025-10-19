# System Architecture: ADK + CopilotKit

## Overview

This document explains the architecture of the ADK + CopilotKit integration and how data flows through the system.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                           USER                                   │
│              Types: "Click the submit button"                    │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    FRONTEND LAYER                                │
│                  (React/Next.js - Port 3000)                     │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  CopilotSidebar                                          │   │
│  │  - Chat interface                                        │   │
│  │  - Message display                                       │   │
│  │  - User input                                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  React Hooks                                             │   │
│  │  - useCopilotAction: DOM manipulation handlers          │   │
│  │  - useCoAgent: Shared state management                  │   │
│  │  - useCopilotReadable: Screen context provider          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  CopilotKit Provider                                     │   │
│  │  - Wraps entire app                                      │   │
│  │  - Manages WebSocket/HTTP communication                 │   │
│  │  - Routes to API endpoint                                │   │
│  └─────────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY LAYER                             │
│              (Next.js API Route - /api/copilotkit)              │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  CopilotRuntime                                          │   │
│  │  - Message routing                                       │   │
│  │  - Session management                                    │   │
│  │  - Protocol translation                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  HttpAgent                                               │   │
│  │  - HTTP client to backend                                │   │
│  │  - Request/response handling                             │   │
│  │  - Error handling                                        │   │
│  └─────────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     BACKEND LAYER                                │
│                (Python/FastAPI - Port 8000)                      │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  FastAPI Application                                     │   │
│  │  - HTTP server                                           │   │
│  │  - Endpoint: POST /                                      │   │
│  │  - CORS handling                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  ADKAgent Wrapper                                        │   │
│  │  - CopilotKit integration                                │   │
│  │  - Session management                                    │   │
│  │  - State persistence                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  LlmAgent (Google ADK)                                   │   │
│  │  - Agent logic                                           │   │
│  │  - Tool management                                       │   │
│  │  - Callback handling                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Custom Tools                                            │   │
│  │  - set_proverbs()                                        │   │
│  │  - get_weather()                                         │   │
│  │  - Your custom tools...                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Callbacks                                               │   │
│  │  - before_agent: Initialize state                        │   │
│  │  - before_model: Inject context                          │   │
│  │  - after_model: Process response                         │   │
│  └─────────────────────────────────────────────────────────┘   │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AI MODEL LAYER                              │
│                  (Google Gemini 2.5 Flash)                       │
│                                                                   │
│  - Natural language understanding                                │
│  - Tool selection & parameter extraction                         │
│  - Response generation                                           │
│  - Context management                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Data Flow: Complete Request Cycle

### Example: User says "Add a proverb about AI"

```
┌──────────────────────────────────────────────────────────────────┐
│ STEP 1: User Input                                               │
│ User types in CopilotSidebar: "Add a proverb about AI"          │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 2: Frontend Processing                                      │
│ - CopilotKit captures message                                    │
│ - Packages with current state                                    │
│ - Sends POST to /api/copilotkit                                  │
│                                                                   │
│ Payload: {                                                       │
│   message: "Add a proverb about AI",                            │
│   state: { proverbs: [...] },                                   │
│   actions: [clickElement, fillInput, ...]                       │
│ }                                                                │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 3: API Gateway                                              │
│ - CopilotRuntime receives request                               │
│ - Routes to "my_agent" via HttpAgent                            │
│ - Forwards to http://localhost:8000/                            │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 4: Backend Receives Request                                 │
│ - FastAPI endpoint receives POST                                 │
│ - ADKAgent wrapper processes request                             │
│ - Extracts message and state                                     │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 5: Before Agent Callback                                    │
│ - on_before_agent() executes                                     │
│ - Initializes state if needed                                    │
│ - Sets default proverbs: []                                      │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 6: Before Model Callback                                    │
│ - before_model_modifier() executes                               │
│ - Injects current proverbs into system prompt                    │
│                                                                   │
│ Enhanced Prompt:                                                 │
│ "You are a helpful assistant for maintaining proverbs.          │
│  Current proverbs: ['CopilotKit may be new...']                 │
│  User: Add a proverb about AI"                                  │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 7: Gemini Processing                                        │
│ - ADK sends enhanced prompt to Gemini 2.5 Flash                 │
│ - Gemini analyzes request                                        │
│ - Decides to call set_proverbs tool                             │
│                                                                   │
│ Tool Call:                                                       │
│ set_proverbs(new_proverbs=[                                     │
│   "CopilotKit may be new...",                                   │
│   "AI is the future of human creativity"                        │
│ ])                                                               │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 8: Tool Execution                                           │
│ - set_proverbs() function executes                              │
│ - Updates tool_context.state["proverbs"]                        │
│ - Returns success message                                        │
│                                                                   │
│ State Update:                                                    │
│ proverbs: [                                                      │
│   "CopilotKit may be new...",                                   │
│   "AI is the future of human creativity"                        │
│ ]                                                                │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 9: After Model Callback                                     │
│ - simple_after_model_modifier() executes                         │
│ - Stops consecutive tool calling                                 │
│ - Prepares final response                                        │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 10: Response Chain                                          │
│ - ADKAgent packages response                                     │
│ - Sends back through FastAPI                                     │
│ - HttpAgent receives response                                    │
│ - CopilotRuntime processes                                       │
│                                                                   │
│ Response: {                                                      │
│   message: "I've added a proverb about AI",                     │
│   state: { proverbs: [...] },                                   │
│   success: true                                                  │
│ }                                                                │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ STEP 11: Frontend Update                                         │
│ - useCoAgent hook detects state change                          │
│ - React triggers re-render                                       │
│ - New proverb appears in UI                                      │
│ - CopilotSidebar shows agent message                            │
└──────────────────────────────────────────────────────────────────┘
```

---

## Component Interactions

### Frontend Components

```typescript
// Layout.tsx - Root wrapper
<CopilotKit runtimeUrl="/api/copilotkit" agent="my_agent">
  <ScreenContextProvider>      // Provides page context
    <DOMManipulator />          // Registers DOM actions
    <YourApp />                 // Your application
  </ScreenContextProvider>
</CopilotKit>

// Page.tsx - Main UI
function Page() {
  // Shared state with agent
  const { state, setState } = useCoAgent({
    name: "my_agent",
    initialState: { proverbs: [] }
  });

  // Frontend action
  useCopilotAction({
    name: "setThemeColor",
    handler({ themeColor }) {
      setThemeColor(themeColor);
    }
  });

  return <YourUI state={state} />;
}
```

### Backend Components

```python
# agent.py - Agent setup
def set_proverbs(tool_context: ToolContext, new_proverbs: list[str]):
    """Tool that agent can call"""
    tool_context.state["proverbs"] = new_proverbs
    return {"status": "success"}

# Create agent
proverbs_agent = LlmAgent(
    name="ProverbsAgent",
    model="gemini-2.5-flash",
    instruction="You are a helpful assistant...",
    tools=[set_proverbs, get_weather],
    before_agent_callback=on_before_agent,
    before_model_callback=before_model_modifier,
    after_model_callback=simple_after_model_modifier
)

# Wrap for CopilotKit
adk_proverbs_agent = ADKAgent(
    adk_agent=proverbs_agent,
    app_name="proverbs_app",
    user_id="demo_user",
    session_timeout_seconds=3600,
    use_in_memory_services=True
)

# Expose via FastAPI
app = FastAPI()
add_adk_fastapi_endpoint(app, adk_proverbs_agent, path="/")
```

---

## State Management

### State Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend State                            │
│  const { state, setState } = useCoAgent({                   │
│    name: "my_agent",                                        │
│    initialState: { proverbs: [] }                           │
│  });                                                         │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Synchronized via CopilotKit
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                    Backend State                             │
│  tool_context.state = {                                     │
│    "proverbs": [...]                                        │
│  }                                                           │
└─────────────────────────────────────────────────────────────┘
```

### State Updates

**Frontend → Backend:**
```typescript
// User manually updates state
setState({ proverbs: [...state.proverbs, "New proverb"] });
// → Sent to backend via CopilotKit
// → Updates tool_context.state
```

**Backend → Frontend:**
```python
# Agent updates state via tool
tool_context.state["proverbs"] = new_proverbs
# → Sent to frontend via response
# → useCoAgent hook detects change
# → React re-renders
```

---

## DOM Manipulation Flow

```
┌──────────────────────────────────────────────────────────────┐
│ User: "Click the submit button"                              │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│ Agent decides to call clickElement action                     │
│ Parameters: { selector: "#submit-btn" }                      │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│ CopilotKit routes to frontend action handler                 │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│ useCopilotAction handler executes:                           │
│ handler({ selector }) {                                      │
│   document.querySelector(selector)?.click();                 │
│   return { success: true };                                  │
│ }                                                             │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│ DOM updated: Button clicked                                   │
│ Result sent back to agent                                     │
│ Agent responds: "I clicked the submit button"                │
└──────────────────────────────────────────────────────────────┘
```

---

## Screen Understanding Flow

```
┌──────────────────────────────────────────────────────────────┐
│ Page loads / DOM changes                                      │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│ ScreenContextProvider captures content:                      │
│ - document.title                                             │
│ - document.body.innerText                                    │
│ - All buttons, inputs, links                                 │
│ - Element selectors                                          │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│ useCopilotReadable makes it available to agent:              │
│ {                                                             │
│   description: "Current page content",                       │
│   value: JSON.stringify(pageContent)                         │
│ }                                                             │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│ Agent receives context with every request                     │
│ Can analyze and understand page structure                     │
└──────────────────────────────────────────────────────────────┘
```

---

## Security Layers

```
┌──────────────────────────────────────────────────────────────┐
│ Layer 1: Frontend Validation                                  │
│ - Whitelist allowed selectors                                │
│ - Block dangerous element types                              │
│ - Rate limiting                                               │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 2: Action Handlers                                      │
│ - Try-catch error handling                                   │
│ - Type checking                                               │
│ - Logging all actions                                         │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 3: User Confirmation                                    │
│ - Prompt for sensitive actions                               │
│ - Require explicit approval                                  │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│ Layer 4: Backend Validation                                   │
│ - Agent instruction constraints                              │
│ - Tool parameter validation                                  │
│ - State access controls                                       │
└──────────────────────────────────────────────────────────────┘
```

---

## Session Management

```
┌──────────────────────────────────────────────────────────────┐
│ User Session                                                  │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ Session ID: generated by CopilotKit                      │ │
│ │ User ID: "demo_user" (configurable)                      │ │
│ │ Timeout: 3600 seconds                                     │ │
│ │ Storage: In-memory (can use Redis/DB)                    │ │
│ └──────────────────────────────────────────────────────────┘ │
│                                                               │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ Conversation History                                      │ │
│ │ - All messages                                            │ │
│ │ - Tool calls                                              │ │
│ │ - State changes                                           │ │
│ └──────────────────────────────────────────────────────────┘ │
│                                                               │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ Current State                                             │ │
│ │ { proverbs: [...] }                                       │ │
│ └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## Deployment Architecture

### Development
```
Localhost:3000 (Next.js) ←→ Localhost:8000 (FastAPI)
                              ↓
                         Google Gemini API
```

### Production
```
Your Domain (Next.js)
    ↓
Next.js API Route (/api/copilotkit)
    ↓
Backend Service (FastAPI on Cloud Run / Lambda / etc)
    ↓
Google Gemini API
```

---

## Performance Considerations

### Optimization Points

1. **Frontend:**
   - Debounce screen context updates
   - Limit context size (5000 chars)
   - Use efficient selectors
   - Batch DOM operations

2. **API Gateway:**
   - Connection pooling
   - Request caching
   - Compression

3. **Backend:**
   - Session caching
   - Tool result memoization
   - Async operations

4. **AI Model:**
   - Prompt optimization
   - Token management
   - Response streaming

---

## Monitoring & Debugging

### Key Metrics to Track

```
Frontend:
- Action execution time
- State update frequency
- Error rates
- User interaction patterns

Backend:
- Request latency
- Tool execution time
- Gemini API calls
- Error rates
- Session count

Infrastructure:
- CPU/Memory usage
- Network latency
- API rate limits
- Cost per request
```

### Logging Points

```typescript
// Frontend
console.log("Action:", actionName, "Selector:", selector);
console.log("State updated:", newState);
console.error("Action failed:", error);

// Backend
logger.info(f"Tool called: {tool_name}")
logger.info(f"State updated: {state}")
logger.error(f"Tool failed: {error}")
```

---

## Extending the Architecture

### Adding New Capabilities

1. **New Frontend Actions:**
   - Add `useCopilotAction` in component
   - Automatically available to agent

2. **New Backend Tools:**
   - Define Python function
   - Add to agent's `tools` array
   - Update agent instructions

3. **New State Fields:**
   - Update `AgentState` type
   - Initialize in `initialState`
   - Update agent callbacks

4. **External Integrations:**
   - Add API clients
   - Create integration tools
   - Handle authentication

---

## Conclusion

This architecture provides:
- ✅ Clear separation of concerns
- ✅ Bidirectional communication
- ✅ Real-time state synchronization
- ✅ Extensible design
- ✅ Security by default
- ✅ Production-ready patterns

For implementation details, see [TUTORIAL.md](./TUTORIAL.md).

