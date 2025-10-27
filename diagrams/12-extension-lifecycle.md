# Extension Lifecycle Diagram

This diagram illustrates the complete lifecycle of the Recall Chrome Extension, including installation, initialization, service worker lifecycle management, state restoration, and side panel interactions.

## Overview

The extension follows Chrome's Manifest V3 lifecycle model where the service worker can be terminated and reactivated by Chrome to conserve resources. The architecture ensures state persistence and seamless restoration across service worker terminations.

## Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> INSTALLING : User installs extension
    
    INSTALLING --> INSTALLED : chrome.runtime.onInstalled
    
    INSTALLED --> INITIALIZING : Service Worker starts
    
    INITIALIZING --> INIT_STORAGE : Initialize IndexedDB
    INIT_STORAGE --> INIT_AI : IndexedDB ready
    INIT_STORAGE --> INIT_RETRY : Connection failed
    INIT_RETRY --> INIT_STORAGE : Retry (exponential backoff)
    INIT_RETRY --> INIT_FALLBACK : Max retries exceeded
    INIT_FALLBACK --> INIT_AI : Use chrome.storage.local
    
    INIT_AI --> CHECKING_API : Check AI availability
    CHECKING_API --> DOWNLOADING_MODEL : Model needs download
    CHECKING_API --> READY : Model available
    DOWNLOADING_MODEL --> READY : Download complete
    CHECKING_API --> READY : API unavailable (degraded mode)
    
    READY --> IDLE : Initialization complete
    
    IDLE --> PROCESSING : Receive message
    PROCESSING --> IDLE : Message handled
    
    IDLE --> TERMINATED : Chrome terminates SW<br/>(30s inactivity)
    
    TERMINATED --> REACTIVATING : New message/event
    
    REACTIVATING --> RESTORE_STATE : Service Worker restarts
    RESTORE_STATE --> RESTORE_STORAGE : Load from chrome.storage.local
    RESTORE_STORAGE --> RESTORE_DB : Reconnect IndexedDB
    RESTORE_DB --> RESTORE_AI : Restore AI sessions
    RESTORE_AI --> IDLE : State restored
    
    IDLE --> SIDEPANEL_OPEN : User clicks extension icon
    SIDEPANEL_OPEN --> SIDEPANEL_INIT : Side Panel loads
    SIDEPANEL_INIT --> SIDEPANEL_READY : Fetch scraps
    SIDEPANEL_READY --> SIDEPANEL_ACTIVE : Display UI
    
    SIDEPANEL_ACTIVE --> SIDEPANEL_CLOSED : User closes panel
    SIDEPANEL_CLOSED --> IDLE : Cleanup listeners
    
    IDLE --> UNINSTALLING : User uninstalls
    UNINSTALLING --> [*] : Cleanup complete
    
    note right of INSTALLING
        Extension installation
        - Manifest validation
        - Permission requests
        - Icon registration
    end note
    
    note right of INITIALIZING
        Service Worker startup
        - Register message handlers
        - Set up event listeners
        - Initialize state
    end note
    
    note right of INIT_STORAGE
        IndexedDB initialization
        - Open VocalContextDB
        - Create object stores
        - Create indexes
        - Retry on failure (3x)
    end note
    
    note right of INIT_AI
        AI API initialization
        - Check capabilities
        - Download model if needed
        - Create initial sessions
        - Cache for 5 minutes
    end note
    
    note right of READY
        Extension ready
        - All systems initialized
        - Listening for messages
        - Can process captures
    end note
    
    note right of TERMINATED
        Service Worker terminated
        - Chrome conserves resources
        - After 30s inactivity
        - State persisted
        - No active processing
    end note
    
    note right of REACTIVATING
        Service Worker reactivation
        - Triggered by message
        - Triggered by alarm
        - Triggered by user action
        - Fast restart (<500ms)
    end note
    
    note right of RESTORE_STATE
        State restoration
        - Reconnect IndexedDB
        - Restore AI sessions
        - Resume message handling
        - No data loss
    end note
    
    note right of SIDEPANEL_ACTIVE
        Side Panel active
        - Displays scraps
        - Handles user interactions
        - Receives broadcasts
        - Virtual scrolling active
    end note
```

## Detailed Initialization Sequence

```mermaid
sequenceDiagram
    participant Chrome
    participant SW as Service Worker
    participant STH as Storage Handler
    participant AIH as AI Handler
    participant IDB as IndexedDB
    participant NANO as Gemini Nano
    participant STORE as chrome.storage.local
    
    Note over Chrome,STORE: Extension Installation
    
    Chrome->>SW: chrome.runtime.onInstalled
    activate SW
    SW->>SW: Register message handlers
    SW->>SW: Set up event listeners
    
    Note over SW,IDB: Storage Initialization
    
    SW->>STH: init()
    activate STH
    STH->>IDB: indexedDB.open('VocalContextDB', 1)
    
    alt First Install
        IDB->>STH: onupgradeneeded
        STH->>IDB: Create 'scraps' store
        STH->>IDB: Create 'sourceGroups' store
        STH->>IDB: Create indexes
    end
    
    IDB-->>STH: onsuccess
    STH-->>SW: Database ready
    deactivate STH
    
    Note over SW,NANO: AI Initialization
    
    SW->>AIH: checkAPIAvailability()
    activate AIH
    AIH->>NANO: ai.languageModel.capabilities()
    
    alt Model Available
        NANO-->>AIH: { available: 'readily' }
        AIH->>NANO: ai.languageModel.create()
        NANO-->>AIH: session
        AIH->>AIH: Cache session (5 min TTL)
    else Model Needs Download
        NANO-->>AIH: { available: 'after-download' }
        AIH->>Chrome: Show notification
        AIH->>NANO: ai.languageModel.create()
        Note right of NANO: Downloads model<br/>(may take minutes)
        NANO-->>AIH: session
        AIH->>AIH: Cache session
    else Model Unavailable
        NANO-->>AIH: { available: 'no' }
        AIH->>AIH: Set degraded mode flag
    end
    
    AIH-->>SW: API status
    deactivate AIH
    
    Note over SW: Extension Ready
    
    SW->>STORE: Save initialization state
    SW->>Chrome: Ready for messages
    deactivate SW
```

## Service Worker Termination and Reactivation

```mermaid
sequenceDiagram
    participant Chrome
    participant SW as Service Worker
    participant STH as Storage Handler
    participant AIH as AI Handler
    participant STORE as chrome.storage.local
    participant IDB as IndexedDB
    
    Note over Chrome,IDB: Service Worker Active
    
    SW->>SW: Process messages
    SW->>SW: Handle events
    
    Note over Chrome: 30 seconds of inactivity
    
    Chrome->>SW: Terminate service worker
    deactivate SW
    Note right of SW: Service Worker<br/>terminated<br/>(conserve resources)
    
    Note over Chrome,IDB: Service Worker Terminated
    
    Chrome->>Chrome: New message arrives
    
    Note over Chrome,IDB: Service Worker Reactivation
    
    Chrome->>SW: Restart service worker
    activate SW
    SW->>SW: Re-register handlers
    
    Note over SW,IDB: State Restoration
    
    SW->>STORE: Load persisted state
    STORE-->>SW: { lastInitTime, degradedMode }
    
    SW->>STH: reconnect()
    activate STH
    STH->>IDB: Reopen connection
    IDB-->>STH: Connection ready
    STH-->>SW: Storage reconnected
    deactivate STH
    
    SW->>AIH: restoreSessions()
    activate AIH
    
    alt Sessions Still Cached
        AIH->>AIH: Check session timestamps
        AIH->>AIH: Sessions valid (< 5 min)
        AIH-->>SW: Sessions restored
    else Sessions Expired
        AIH->>AIH: Clear stale sessions
        AIH-->>SW: Sessions cleared (will recreate on demand)
    end
    
    deactivate AIH
    
    SW->>Chrome: Ready to process message
    Note right of SW: Total restoration<br/>time: < 500ms
    
    SW->>SW: Handle queued message
    deactivate SW
```

## Side Panel Lifecycle

```mermaid
sequenceDiagram
    participant User
    participant Chrome
    participant SW as Service Worker
    participant SP as Side Panel
    participant STH as Storage Handler
    
    Note over User,STH: Side Panel Opening
    
    User->>Chrome: Click extension icon
    Chrome->>SW: Ensure SW active
    activate SW
    Chrome->>SP: Open side panel
    activate SP
    
    SP->>SP: Load sidepanel.html
    SP->>SP: Execute sidepanel.js
    SP->>SP: Initialize SidePanelController
    
    SP->>SW: sendMessage(GET_ALL_SCRAPS)
    SW->>STH: getAllScraps()
    activate STH
    STH-->>SW: Array of scraps
    deactivate STH
    SW-->>SP: response(scraps)
    
    SP->>SP: Render scrap list
    SP->>SP: Set up event listeners
    SP->>SP: Set up message listener
    
    Note right of SP: Side Panel ready<br/>Time: < 500ms
    
    Note over User,STH: Side Panel Active
    
    User->>SP: Interact with UI
    SP->>SW: Send messages
    SW-->>SP: Responses
    
    SW->>SP: Broadcast(SCRAP_CREATED)
    SP->>SP: Update UI
    
    Note over User,STH: Side Panel Closing
    
    User->>Chrome: Close side panel
    Chrome->>SP: Unload event
    SP->>SP: Cleanup listeners
    SP->>SP: Clear DOM
    deactivate SP
    
    Note right of SW: Service Worker<br/>remains active<br/>for 30s
    
    deactivate SW
```

## State Persistence Strategy

```mermaid
flowchart TD
    START([Service Worker Event]) --> CHECK{Event Type?}
    
    CHECK -->|Installation| INSTALL[Install Event]
    CHECK -->|Activation| ACTIVATE[Activate Event]
    CHECK -->|Message| MESSAGE[Message Event]
    CHECK -->|Termination| TERMINATE[Before Termination]
    
    INSTALL --> INIT_DB[Initialize IndexedDB]
    INIT_DB --> INIT_AI[Initialize AI APIs]
    INIT_AI --> SAVE_STATE[Save init state to<br/>chrome.storage.local]
    SAVE_STATE --> READY[Mark as Ready]
    
    ACTIVATE --> CHECK_STATE{State Exists?}
    CHECK_STATE -->|Yes| RESTORE[Restore from<br/>chrome.storage.local]
    CHECK_STATE -->|No| INIT_DB
    RESTORE --> RECONNECT_DB[Reconnect IndexedDB]
    RECONNECT_DB --> RESTORE_AI[Restore AI sessions]
    RESTORE_AI --> READY
    
    MESSAGE --> ENSURE_READY{Is Ready?}
    ENSURE_READY -->|Yes| PROCESS[Process Message]
    ENSURE_READY -->|No| ACTIVATE
    PROCESS --> UPDATE_STATE[Update state if needed]
    UPDATE_STATE --> PERSIST[Persist to storage]
    PERSIST --> RESPOND[Send Response]
    
    TERMINATE --> SAVE_FINAL[Save final state]
    SAVE_FINAL --> CLEANUP[Cleanup resources]
    CLEANUP --> END_TERM([Terminated])
    
    READY --> END_READY([Ready for Events])
    RESPOND --> END_MSG([Message Complete])
    
    style INSTALL fill:#e3f2fd
    style ACTIVATE fill:#c8e6c9
    style MESSAGE fill:#fff9c4
    style TERMINATE fill:#ffcdd2
    style READY fill:#a5d6a7
    style END_READY fill:#a5d6a7
```

## Key Lifecycle Events

### 1. Installation (chrome.runtime.onInstalled)

**Triggers:**
- User installs extension from Chrome Web Store
- Developer loads unpacked extension
- Extension updates to new version

**Actions:**
- Create IndexedDB database and object stores
- Initialize default settings in chrome.storage.local
- Check AI API availability
- Register content scripts
- Set up alarm for periodic cleanup (optional)

**Timing:** ~1-2 seconds (first install), ~5-10 seconds if model download needed

### 2. Service Worker Activation

**Triggers:**
- Extension installation
- Browser startup
- Service worker restart after termination

**Actions:**
- Re-register message handlers
- Reconnect to IndexedDB
- Restore AI session cache (if within 5-minute TTL)
- Load persisted state from chrome.storage.local
- Resume message processing

**Timing:** <500ms (from Requirements 12.3)

### 3. Service Worker Termination

**Triggers:**
- 30 seconds of inactivity (Chrome's default)
- Browser resource constraints
- Extension update

**Actions:**
- Save current state to chrome.storage.local
- Close IndexedDB connections gracefully
- Clear AI session cache
- Remove event listeners

**State Preserved:**
- Initialization status
- Degraded mode flag
- Last activity timestamp
- Pending operation flags

**State NOT Preserved:**
- Active AI sessions (recreated on demand)
- In-memory caches
- Temporary processing data

### 4. Side Panel Open/Close

**Open Actions:**
- Ensure service worker is active
- Load side panel HTML/JS
- Fetch all scraps from IndexedDB
- Render initial UI
- Set up broadcast message listener

**Close Actions:**
- Remove event listeners
- Clear DOM elements
- Stop virtual scrolling observers
- Service worker remains active for 30s

**Timing:** <200ms to open (Requirements 4.1), <100ms to close

### 5. IndexedDB Connection Management

**Initial Connection:**
- Open database with version 1
- Create object stores if first install
- Create indexes for efficient queries
- Retry up to 3 times on failure
- Fallback to chrome.storage.local if all retries fail

**Reconnection After Termination:**
- Reopen existing database
- Verify object stores exist
- Restore transaction handlers
- No data migration needed

**Connection Failure Handling:**
- Exponential backoff: 1s, 2s, 4s
- Fallback to chrome.storage.local (limited capacity)
- User notification if fallback activated

### 6. AI API Session Management

**Session Creation:**
- Check capabilities before creating
- Create session on first use (lazy initialization)
- Cache session with 5-minute TTL
- Store timestamp for expiration check

**Session Restoration:**
- Check timestamp on service worker reactivation
- If < 5 minutes: reuse existing session
- If > 5 minutes: clear and recreate on demand
- No persistent storage of sessions

**Session Cleanup:**
- Automatic cleanup on 5-minute timeout
- Manual cleanup on service worker termination
- Destroy session objects to free memory

## Performance Targets

| Lifecycle Event | Target Time | Requirement |
|----------------|-------------|-------------|
| Initial Installation | 1-2s | 12.1 |
| Service Worker Activation | <500ms | 12.3 |
| IndexedDB Connection | <1s | 12.1 |
| AI Session Creation | 1-5s | 12.2 |
| State Restoration | <500ms | 12.3 |
| Side Panel Open | <200ms | 4.1 |
| Side Panel Close | <100ms | 12.5 |

## Error Recovery

### IndexedDB Failure
1. Retry connection 3 times with exponential backoff
2. If all retries fail, activate fallback mode
3. Use chrome.storage.local (limited to 10MB)
4. Notify user of degraded functionality
5. Offer data export option

### AI API Unavailable
1. Check capabilities on initialization
2. If unavailable, set degraded mode flag
3. Save scraps without AI processing
4. Allow manual retry from UI
5. Continue with core capture functionality

### Service Worker Crash
1. Chrome automatically restarts service worker
2. State restoration from chrome.storage.local
3. Reconnect to IndexedDB
4. Resume message processing
5. No user intervention required

### Side Panel Crash
1. User can reopen side panel
2. Fresh initialization from storage
3. No data loss (stored in IndexedDB)
4. Service worker unaffected

## Validation Checklist

- [x] All lifecycle events from Requirements 12.1-12.5 are documented
- [x] Service worker install, activate, terminate, and reactivate flows are shown
- [x] State restoration after termination is detailed
- [x] IndexedDB connection initialization and reconnection are illustrated
- [x] AI API session initialization and caching strategy are explained
- [x] Side panel open/close lifecycle is documented
- [x] Performance targets match requirements
- [x] Error recovery paths are defined
- [x] Diagram uses standard Mermaid notation
- [x] Timing annotations are accurate
- [x] All components from architecture are included

