# Side Panel UI Component Diagram

This diagram shows the component hierarchy, data flow, event handling, and state management for the Side Panel UI.

## Component Hierarchy and Data Flow

```mermaid
graph TB
    subgraph "Side Panel UI Layer"
        SPC[SidePanelController<br/>Root Component<br/>• Manages global state<br/>• Coordinates child components<br/>• Handles message passing]
        
        SB[SearchBar Component<br/>• Input field with debounce<br/>• Clear button<br/>• Loading indicator]
        
        SL["ScrapList Component<br/>• Virtual scrolling container<br/>• Viewport: ~800px<br/>• Renders 8 cards - 4 visible + 4 buffer<br/>• IntersectionObserver for lazy load"]
        
        SC["ScrapCard Component<br/>• Thumbnail display<br/>• Metadata - title, domain, time<br/>• Transcription/text note<br/>• Magic buttons<br/>• Expand/collapse toggle<br/>• Group badge"]
        
        SGV[SourceGroupView Component<br/>• Group header with back button<br/>• Domain and count display<br/>• Chronological scrap list<br/>• Shared layout with main list]
        
        MB[Magic Buttons<br/>• Clean Up button<br/>• Refine button<br/>• Draft Email button<br/>• Loading states<br/>• Disabled states]
        
        CTX[Context Panel<br/>• Summary display<br/>• Image analysis results<br/>• Content type badges<br/>• Selected text<br/>• Collapsible section]
    end
    
    subgraph "Service Worker"
        SW[Service Worker<br/>Message Handler]
    end
    
    subgraph "IndexedDB"
        IDB[(IndexedDB<br/>scraps & sourceGroups)]
    end
    
    %% Component Hierarchy
    SPC --> SB
    SPC --> SL
    SPC --> SGV
    SL --> SC
    SC --> MB
    SC --> CTX
    
    %% Data Flow: Load
    SPC -->|"1. GET_ALL_SCRAPS"| SW
    SW -->|"2. Query"| IDB
    IDB -->|"3. Return scraps[]"| SW
    SW -->|"4. Response"| SPC
    SPC -->|"5. Pass scraps"| SL
    SL -->|"6. Pass individual scrap"| SC
    
    %% Data Flow: Search
    SB -->|"input event (debounced)"| SPC
    SPC -->|"SEARCH_SCRAPS"| SW
    SW -->|"semantic search"| SW
    SW -->|"result indices"| SPC
    SPC -->|"filtered scraps"| SL
    
    %% Data Flow: Magic Buttons
    MB -->|"click event"| SC
    SC -->|"action request"| SPC
    SPC -->|"CLEANUP_TEXT / REFINE_TEXT / DRAFT_EMAIL"| SW
    SW -->|"AI processing"| SW
    SW -->|"processed result"| SPC
    SPC -->|"update scrap"| SC
    
    %% Data Flow: Source Group
    SC -->|"View Group click"| SPC
    SPC -->|"GET_SOURCE_GROUP"| SW
    SW -->|"group + scraps"| SPC
    SPC -->|"render group view"| SGV
    SGV -->|"display scraps"| SC
    
    %% Broadcast Updates
    SW -.->|"SCRAP_CREATED broadcast"| SPC
    SW -.->|"SCRAP_UPDATED broadcast"| SPC
    SW -.->|"SCRAP_DELETED broadcast"| SPC
    SPC -.->|"refresh UI"| SL
    
    style SPC fill:#e3f2fd
    style SB fill:#fff9c4
    style SL fill:#c8e6c9
    style SC fill:#ffecb3
    style SGV fill:#f3e5f5
    style MB fill:#ffe0b2
    style CTX fill:#e0f2f1
    style SW fill:#f5f5f5
    style IDB fill:#f5f5f5
```

## State Management Flow

```mermaid
stateDiagram-v2
    [*] --> Initializing
    
    Initializing : Load all scraps from Service Worker
    Initializing : Set up message listeners
    Initializing : Initialize search input
    
    Initializing --> ListView : Initial load complete
    
    state ListView {
        [*] --> RenderingList
        RenderingList : Display all scraps
        RenderingList : Virtual scrolling active
        RenderingList : Sort by timestamp (newest first)
        
        RenderingList --> ScrollingList : User scrolls
        ScrollingList --> RenderingList : Update visible items
        
        RenderingList --> ExpandingCard : Click expand toggle
        ExpandingCard --> RenderingList : Show/hide context
        
        RenderingList --> ProcessingMagic : Click magic button
        ProcessingMagic : Disable button
        ProcessingMagic : Show loading text
        ProcessingMagic : Send message to SW
        ProcessingMagic --> RenderingList : Update with result
        
        RenderingList --> ViewingGroup : Click "View Source Group"
    }
    
    ListView --> SearchView : User types in search
    
    state SearchView {
        [*] --> Debouncing
        Debouncing : Wait 500ms
        Debouncing --> Searching : Timer expires
        
        Searching : Show loading indicator
        Searching : Send SEARCH_SCRAPS message
        Searching : Wait for semantic results
        
        Searching --> ShowingResults : Results received
        ShowingResults : Display filtered scraps
        ShowingResults : Highlight matches
        
        Searching --> NoResults : Empty results
        NoResults : Show "No results" message
        NoResults : Offer "Show All" button
        
        ShowingResults --> ExpandingCard : Click expand
        ExpandingCard --> ShowingResults : Toggle context
    }
    
    SearchView --> ListView : Clear search input
    
    ListView --> GroupView : Click "View Source Group"
    
    state GroupView {
        [*] --> LoadingGroup
        LoadingGroup : Send GET_SOURCE_GROUP
        LoadingGroup : Wait for response
        
        LoadingGroup --> ShowingGroup : Group data received
        ShowingGroup : Display group header
        ShowingGroup : Show domain and count
        ShowingGroup : List all scraps chronologically
        
        ShowingGroup --> ExpandingCard : Click expand
        ExpandingCard --> ShowingGroup : Toggle context
        
        ShowingGroup --> ProcessingMagic : Click magic button
        ProcessingMagic --> ShowingGroup : Update with result
    }
    
    GroupView --> ListView : Click back button
    
    state "Broadcast Handling" as BH {
        [*] --> ListeningForBroadcasts
        ListeningForBroadcasts --> HandleScrapCreated : SCRAP_CREATED
        ListeningForBroadcasts --> HandleScrapUpdated : SCRAP_UPDATED
        ListeningForBroadcasts --> HandleScrapDeleted : SCRAP_DELETED
        
        HandleScrapCreated : Fetch new scrap
        HandleScrapCreated : Add to scraps array
        HandleScrapCreated : Re-render list
        HandleScrapCreated --> ListeningForBroadcasts
        
        HandleScrapUpdated : Fetch updated scrap
        HandleScrapUpdated : Update in scraps array
        HandleScrapUpdated : Re-render card
        HandleScrapUpdated --> ListeningForBroadcasts
        
        HandleScrapDeleted : Remove from scraps array
        HandleScrapDeleted : Re-render list
        HandleScrapDeleted --> ListeningForBroadcasts
    }
    
    note right of ListView
        State: currentView = 'list'
        Data: scraps[] (all scraps)
        Scroll: track visible range
    end note
    
    note right of SearchView
        State: currentView = 'search'
        Data: searchQuery, filteredScraps[]
        Debounce: 500ms timer
    end note
    
    note right of GroupView
        State: currentView = 'group'
        Data: currentGroupId, groupScraps[]
        Navigation: back to list
    end note
```

## Event Handling Flow

```mermaid
sequenceDiagram
    actor User
    participant SB as SearchBar
    participant SPC as SidePanelController
    participant SC as ScrapCard
    participant MB as MagicButtons
    participant SW as Service Worker
    
    Note over User,SW: Scenario 1: Search Flow
    User->>SB: Type search query
    SB->>SB: Debounce 500ms
    SB->>SPC: input event (query)
    SPC->>SPC: Set currentView = 'search'
    SPC->>SW: sendMessage(SEARCH_SCRAPS)
    SW->>SW: Semantic search with Prompt API
    SW-->>SPC: response(result indices)
    SPC->>SPC: Filter scraps by indices
    SPC->>SL: renderScrapList(filteredScraps)
    SL->>User: Display results
    
    Note over User,SW: Scenario 2: Magic Button Flow
    User->>MB: Click "Clean Up" button
    MB->>SC: click event
    SC->>SC: Disable button, show "Cleaning..."
    SC->>SPC: handleCleanup(scrap)
    SPC->>SW: sendMessage(CLEANUP_TEXT)
    SW->>SW: Call Proofreader API
    SW->>IDB: updateScrap(cleanedText)
    SW-->>SPC: response(cleanedText)
    SPC->>SC: updateScrapCard(scrap)
    SC->>SC: Enable button, show result
    SC->>User: Display cleaned text
    
    Note over User,SW: Scenario 3: Expand Context Flow
    User->>SC: Click "Show Context" toggle
    SC->>SC: Toggle .hidden class on context panel
    SC->>SC: Update button text
    SC->>User: Reveal/hide context section
    
    Note over User,SW: Scenario 4: View Source Group Flow
    User->>SC: Click "View Source Group"
    SC->>SPC: showSourceGroup(groupId)
    SPC->>SPC: Set currentView = 'group'
    SPC->>SW: sendMessage(GET_SOURCE_GROUP)
    SW->>IDB: getSourceGroup(groupId)
    IDB-->>SW: group + scraps[]
    SW-->>SPC: response(group, scraps)
    SPC->>SGV: renderGroupView(group, scraps)
    SGV->>User: Display group with all scraps
    
    Note over User,SW: Scenario 5: Scroll with Virtual Scrolling
    User->>SL: Scroll down
    SL->>SL: IntersectionObserver detects
    SL->>SL: Calculate visible range
    SL->>SL: Render new cards (buffer)
    SL->>SL: Remove off-screen cards
    SL->>User: Smooth scrolling experience
    
    Note over User,SW: Scenario 6: Broadcast Update
    SW->>SPC: broadcast(SCRAP_CREATED)
    SPC->>SW: sendMessage(GET_SCRAP, scrapId)
    SW-->>SPC: response(newScrap)
    SPC->>SPC: Add to scraps array
    SPC->>SL: renderScrapList()
    SL->>User: New scrap appears at top
```

## Component State Details

```mermaid
classDiagram
    class SidePanelController {
        -scraps: Scrap[]
        -currentView: 'list' | 'search' | 'group'
        -searchQuery: string
        -currentGroupId: string
        +init()
        +renderScrapList()
        +performSearch(query)
        +showSourceGroup(groupId)
        +handleNewScrap(scrapId)
        +handleScrapUpdate(scrapId)
        +handleScrapDelete(scrapId)
        +sendMessage(message)
    }
    
    class SearchBar {
        -inputElement: HTMLInputElement
        -debounceTimer: number
        +setupSearch()
        +handleInput(event)
        +clearSearch()
    }
    
    class ScrapList {
        -container: HTMLElement
        -visibleRange: Object
        -observer: IntersectionObserver
        +render(scraps)
        +getVisibleScraps()
        +handleScroll()
        +updateVisibleRange()
    }
    
    class ScrapCard {
        -scrap: Scrap
        -element: HTMLElement
        -expanded: boolean
        +create(scrap)
        +attachListeners()
        +handleExpand()
        +updateContent(updates)
    }
    
    class MagicButtons {
        -scrap: Scrap
        -cleanupBtn: HTMLButtonElement
        -refineBtn: HTMLButtonElement
        -draftBtn: HTMLButtonElement
        +handleCleanup()
        +handleRefine()
        +handleDraft()
        +setLoading(button, loading)
    }
    
    class SourceGroupView {
        -groupId: string
        -group: SourceGroup
        -scraps: Scrap[]
        +render(group, scraps)
        +handleBack()
    }
    
    class ContextPanel {
        -scrap: Scrap
        -visible: boolean
        +render(scrap)
        +toggle()
        +renderImageAnalysis()
        +renderSummary()
    }
    
    SidePanelController --> SearchBar : manages
    SidePanelController --> ScrapList : manages
    SidePanelController --> SourceGroupView : manages
    ScrapList --> ScrapCard : contains multiple
    ScrapCard --> MagicButtons : contains
    ScrapCard --> ContextPanel : contains
```

## Virtual Scrolling Implementation

```mermaid
flowchart TD
    START([User Opens Side Panel]) --> LOAD[Load All Scraps<br/>scraps.length = N]
    
    LOAD --> CALC[Calculate Viewport<br/>• Viewport height: 800px<br/>• Card height: 200px<br/>• Visible cards: 4<br/>• Buffer: 2 above + 2 below<br/>• Total rendered: 8]
    
    CALC --> RENDER[Render Initial 8 Cards<br/>indices 0-7]
    
    RENDER --> OBSERVE[Set up IntersectionObserver<br/>on all rendered cards]
    
    OBSERVE --> WAIT{User Action?}
    
    WAIT -->|Scroll Down| DETECT_DOWN[Detect bottom card<br/>entering viewport]
    DETECT_DOWN --> CALC_NEW_DOWN[Calculate new range<br/>e.g., indices 2-9]
    CALC_NEW_DOWN --> UPDATE_DOWN[• Remove cards 0-1<br/>• Add cards 8-9]
    UPDATE_DOWN --> OBSERVE
    
    WAIT -->|Scroll Up| DETECT_UP[Detect top card<br/>entering viewport]
    DETECT_UP --> CALC_NEW_UP[Calculate new range<br/>e.g., indices 0-7]
    CALC_NEW_UP --> UPDATE_UP[• Remove cards 8-9<br/>• Add cards 0-1]
    UPDATE_UP --> OBSERVE
    
    WAIT -->|Search| FILTER[Filter scraps<br/>filteredScraps.length = M]
    FILTER --> CALC
    
    WAIT -->|View Group| LOAD_GROUP[Load group scraps<br/>groupScraps.length = K]
    LOAD_GROUP --> CALC
    
    style CALC fill:#fff9c4
    style RENDER fill:#c8e6c9
    style OBSERVE fill:#e3f2fd
    style UPDATE_DOWN fill:#ffecb3
    style UPDATE_UP fill:#ffecb3
```

## Performance Optimizations

### 1. Virtual Scrolling
- Only render 8 cards at a time (4 visible + 4 buffer)
- Use IntersectionObserver for efficient scroll detection
- Dynamically add/remove DOM elements as user scrolls
- Reduces memory footprint for large scrap collections (>50 items)

### 2. Lazy Loading
- Thumbnails loaded only when cards enter viewport
- Use `loading="lazy"` attribute on img tags
- Defer loading of context panels until expanded

### 3. Debouncing
- Search input debounced to 500ms
- Prevents excessive API calls during typing
- Cancels pending searches when new input arrives

### 4. Event Delegation
- Attach listeners to parent container, not individual cards
- Use event.target to identify clicked element
- Reduces memory usage and improves performance

### 5. State Caching
- Cache scraps array in SidePanelController
- Only re-fetch when broadcast update received
- Avoid redundant IndexedDB queries

---

## Requirements Coverage

This diagram addresses the following requirements:

- **Requirement 4.1**: Side panel opens within 200ms with smooth animation
- **Requirement 4.2**: Scrap cards display thumbnail, metadata, transcription, and group badge
- **Requirement 4.3**: Scraps retrieved and rendered within 500ms
- **Requirement 4.4**: "View Source Group" button expands to show linked scraps
- **Requirement 4.5**: Virtual scrolling for lists >50 scraps
- **Requirement 4.6**: Scrap cards expand with smooth animation within 300ms

## Key Interactions

1. **Search**: User types → debounce → semantic search → filter results → render
2. **Magic Buttons**: Click → disable → send message → AI processing → update → enable
3. **Expand Context**: Click toggle → toggle class → update button text
4. **View Group**: Click → load group → render group view → display scraps
5. **Scroll**: Scroll event → IntersectionObserver → calculate range → update DOM
6. **Broadcast**: Service Worker sends update → fetch new data → update state → re-render
