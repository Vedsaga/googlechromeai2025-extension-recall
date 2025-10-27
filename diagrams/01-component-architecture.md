# Component Architecture Diagram

This diagram shows the detailed component structure of the Recall Chrome Extension with communication patterns across all four architectural layers.

```mermaid
graph TB
    subgraph UI["LAYER 1: USER INTERFACE"]
        CS[Content Script<br/>capture-ui.js<br/>• Selection overlay<br/>• Mic/text buttons<br/>• Screenshot capture]
        SP[Side Panel<br/>sidepanel.html/js<br/>• Scrap list view<br/>• Search interface<br/>• Magic buttons]
    end
    
    subgraph BL["LAYER 2: BUSINESS LOGIC"]
        SW[Service Worker<br/>background.js<br/>• Message orchestration<br/>• Workflow coordination<br/>• State management]
    end
    
    subgraph SVC["LAYER 3: SERVICES"]
        AIH[AI Handler<br/>ai-handler.js<br/>• API abstraction<br/>• Parallel execution<br/>• Error handling]
        STH[Storage Handler<br/>storage.js<br/>• IndexedDB wrapper<br/>• Source grouping<br/>• Query optimization]
    end
    
    subgraph EXT["LAYER 4: EXTERNAL SYSTEMS"]
        NANO[Gemini Nano APIs<br/>• Prompt API<br/>• Summarizer API<br/>• Rewriter API<br/>• Proofreader API<br/>• Writer API]
        IDB[(IndexedDB<br/>• scraps store<br/>• sourceGroups store)]
        CHROME[Chrome APIs<br/>• runtime<br/>• tabs<br/>• sidePanel<br/>• storage]
    end
    
    CS -->|chrome.runtime.sendMessage<br/>request/response| SW
    SP -->|chrome.runtime.sendMessage<br/>+ Message Ports<br/>request/response| SW
    SW -->|function calls| AIH
    SW -->|function calls| STH
    AIH -->|async calls| NANO
    STH -->|async calls| IDB
    SW -->|async calls| CHROME
    
    style UI fill:#e3f2fd
    style BL fill:#c8e6c9
    style SVC fill:#fff9c4
    style EXT fill:#f5f5f5
```

## Component Details

### Layer 1: User Interface (Blue)
- **Content Script**: Injected into web pages for content capture
- **Side Panel**: Chrome side panel UI for viewing and managing scraps

### Layer 2: Business Logic (Green)
- **Service Worker**: Central orchestrator managing all extension workflows

### Layer 3: Services (Yellow)
- **AI Handler**: Abstraction layer for Gemini Nano API interactions
- **Storage Handler**: IndexedDB wrapper with source grouping logic

### Layer 4: External Systems (Gray)
- **Gemini Nano APIs**: Chrome's built-in AI capabilities
- **IndexedDB**: Browser-based local storage
- **Chrome APIs**: Extension platform APIs

## Communication Patterns

1. **chrome.runtime.sendMessage**: Request/response pattern between Content Script/Side Panel and Service Worker
2. **Message Ports**: Real-time bidirectional communication for Side Panel updates
3. **Function Calls**: Synchronous service invocation within Service Worker
4. **Async Calls**: Asynchronous operations to external systems

## Requirements Coverage

This diagram addresses:
- **Requirement 7.1**: Content Script to Service Worker communication using chrome.runtime.sendMessage
- **Requirement 7.2**: Service Worker to Side Panel communication using chrome.runtime.sendMessage and message ports
- **Requirement 7.5**: Structured message types with action and payload properties for inter-component communication
