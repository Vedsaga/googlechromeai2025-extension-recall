# Storage Architecture Diagram

This diagram illustrates the IndexedDB storage structure, indexing strategy, source grouping logic, and storage management for the Recall Chrome extension.

## IndexedDB Structure and Source Grouping Flow

```mermaid
flowchart TB
    subgraph IDB["IndexedDB: RecallDB"]
        subgraph SCRAPS_STORE["Object Store: scraps"]
            SCRAPS_DATA["Primary Key: id<br/>---<br/>Fields:<br/>• id - string<br/>• timestamp - number<br/>• url - string<br/>• pageTitle - string<br/>• domain - string<br/>• screenshot - Blob<br/>• screenshotThumb - Blob<br/>• selectedText - string<br/>• selectedRegion - object<br/>• audioBlob - Blob<br/>• audioDuration - number<br/>• textNote - string<br/>• transcription - string<br/>• summary - string<br/>• imageAnalysis - object<br/>• processed - boolean<br/>• processingErrors - object<br/>• cleanedText - string<br/>• refinedText - string<br/>• emailDraft - string<br/>• sourceGroupId - string FK<br/>• userTags - array<br/>• userNotes - string"]
            
            SCRAPS_IDX["Indexes:<br/>• timestamp<br/>• url<br/>• domain<br/>• sourceGroupId"]
        end
        
        subgraph GROUPS_STORE["Object Store: sourceGroups"]
            GROUPS_DATA["Primary Key: id<br/>---<br/>Fields:<br/>• id - string<br/>• url - string<br/>• domain - string<br/>• scrapIds - array<br/>• firstCaptured - number<br/>• lastCaptured - number<br/>• scrapCount - number"]
            
            GROUPS_IDX["Indexes:<br/>• url<br/>• lastCaptured"]
        end
    end
    
    START([New Scrap to Save]) --> GEN_THUMB[Generate Thumbnail<br/>200x150px from screenshot]
    
    GEN_THUMB --> CHECK_QUOTA{Storage Quota<br/>Available > 100MB?}
    
    CHECK_QUOTA -->|No| WARN_USER[Display Warning<br/>Notification]
    CHECK_QUOTA -->|Yes| LOOKUP_GROUP
    
    WARN_USER --> LOOKUP_GROUP
    
    LOOKUP_GROUP[Query sourceGroups<br/>by URL index]
    
    LOOKUP_GROUP --> GROUP_EXISTS{Source Group<br/>Exists?}
    
    GROUP_EXISTS -->|Yes| GET_GROUP[Retrieve Existing<br/>Source Group]
    GROUP_EXISTS -->|No| CREATE_GROUP["Create New Source Group<br/>id = group_domain_urlhash<br/>url = scrap.url<br/>domain = scrap.domain<br/>scrapIds = []<br/>firstCaptured = now<br/>lastCaptured = now<br/>scrapCount = 0"]
    
    GET_GROUP --> UPDATE_GROUP[Update Source Group:<br/>• Add scrapId to scrapIds array<br/>• Update lastCaptured = now<br/>• Increment scrapCount]
    
    CREATE_GROUP --> SAVE_GROUP[Save Source Group<br/>to sourceGroups store]
    UPDATE_GROUP --> SAVE_GROUP
    
    SAVE_GROUP --> LINK_SCRAP[Set scrap.sourceGroupId<br/>to group.id]
    
    LINK_SCRAP --> SAVE_SCRAP[Save Scrap to scraps store<br/>with all data + thumbnail]
    
    SAVE_SCRAP --> VERIFY{Save<br/>Successful?}
    
    VERIFY -->|Yes| BIDIR[Bidirectional Link Established:<br/>scrap.sourceGroupId → group.id<br/>group.scrapIds contains scrap.id]
    VERIFY -->|No| ERROR[Throw Storage Error<br/>Offer JSON Export]
    
    BIDIR --> SUCCESS([Storage Complete])
    ERROR --> FAIL([Storage Failed])
    
    style SCRAPS_STORE fill:#e3f2fd
    style GROUPS_STORE fill:#c8e6c9
    style CHECK_QUOTA fill:#fff9c4
    style GROUP_EXISTS fill:#fff9c4
    style VERIFY fill:#fff9c4
    style SUCCESS fill:#c8e6c9
    style FAIL fill:#ffcdd2
```

## Storage Quota Management Flow

```mermaid
flowchart LR
    CHECK[Check Storage Quota] --> CALC[Calculate:<br/>navigator.storage.estimate]
    
    CALC --> AVAILABLE{Available Space<br/>> 100MB?}
    
    AVAILABLE -->|Yes| PROCEED[Proceed with Save]
    AVAILABLE -->|No| WARNING[Show Warning Notification:<br/>'Storage running low']
    
    WARNING --> OFFER[Offer Options:<br/>1. Continue anyway<br/>2. Export old scraps<br/>3. Delete scraps]
    
    OFFER --> USER_CHOICE{User Choice?}
    
    USER_CHOICE -->|Continue| PROCEED
    USER_CHOICE -->|Export| EXPORT[Generate JSON Export<br/>of selected scraps]
    USER_CHOICE -->|Delete| DELETE[Delete selected scraps<br/>+ update sourceGroups]
    
    EXPORT --> PROCEED
    DELETE --> PROCEED
    
    PROCEED --> SAVE[Save Scrap]
    
    style AVAILABLE fill:#fff9c4
    style WARNING fill:#ffeb3b
    style PROCEED fill:#c8e6c9
```

## Thumbnail Generation Process

```mermaid
flowchart LR
    SCREENSHOT[Original Screenshot<br/>Blob PNG] --> LOAD[Load into Image Object]
    
    LOAD --> CANVAS[Create Canvas<br/>200x150px]
    
    CANVAS --> CALC_ASPECT{Calculate<br/>Aspect Ratio}
    
    CALC_ASPECT --> DRAW[Draw Image<br/>with aspect fit<br/>centered]
    
    DRAW --> COMPRESS[Convert to Blob<br/>JPEG quality: 0.8]
    
    COMPRESS --> THUMB[Thumbnail Blob<br/>~10-20KB]
    
    THUMB --> STORE[Store in<br/>scrap.screenshotThumb]
    
    style SCREENSHOT fill:#e3f2fd
    style THUMB fill:#c8e6c9
```

## Index Strategy and Query Patterns

```mermaid
graph TB
    subgraph QUERIES["Common Query Patterns"]
        Q1["Get All Scraps<br/>Chronologically"]
        Q2["Get Scraps by Domain"]
        Q3["Get Source Group<br/>by URL"]
        Q4["Get All Scraps in<br/>Source Group"]
        Q5["Get Recent Activity"]
    end
    
    subgraph INDEXES["Index Usage"]
        I1["scraps.timestamp<br/>(descending)"]
        I2["scraps.domain<br/>(equality)"]
        I3["sourceGroups.url<br/>(equality)"]
        I4["scraps.sourceGroupId<br/>(equality)"]
        I5["sourceGroups.lastCaptured<br/>(descending)"]
    end
    
    Q1 --> I1
    Q2 --> I2
    Q3 --> I3
    Q4 --> I4
    Q5 --> I5
    
    I1 --> PERF1["O(log n) + scan<br/>Fast for pagination"]
    I2 --> PERF2["O(log n) + scan<br/>Fast domain filtering"]
    I3 --> PERF3["O(log n)<br/>Instant group lookup"]
    I4 --> PERF4["O(log n) + scan<br/>Fast group expansion"]
    I5 --> PERF5["O(log n) + scan<br/>Fast recent sources"]
    
    style QUERIES fill:#e3f2fd
    style INDEXES fill:#fff9c4
    style PERF1 fill:#c8e6c9
    style PERF2 fill:#c8e6c9
    style PERF3 fill:#c8e6c9
    style PERF4 fill:#c8e6c9
    style PERF5 fill:#c8e6c9
```

## Bidirectional Linking Example

```mermaid
graph LR
    subgraph SG["Source Group: group_example_abc123"]
        SG_ID["id: group_example_abc123"]
        SG_URL["url: https://example.com/article"]
        SG_IDS["scrapIds: [<br/>  'scrap_001',<br/>  'scrap_002',<br/>  'scrap_003'<br/>]"]
    end
    
    subgraph S1["Scrap: scrap_001"]
        S1_ID["id: scrap_001"]
        S1_URL["url: https://example.com/article"]
        S1_GRP["sourceGroupId:<br/>group_example_abc123"]
    end
    
    subgraph S2["Scrap: scrap_002"]
        S2_ID["id: scrap_002"]
        S2_URL["url: https://example.com/article"]
        S2_GRP["sourceGroupId:<br/>group_example_abc123"]
    end
    
    subgraph S3["Scrap: scrap_003"]
        S3_ID["id: scrap_003"]
        S3_URL["url: https://example.com/article"]
        S3_GRP["sourceGroupId:<br/>group_example_abc123"]
    end
    
    SG_IDS -.->|contains| S1_ID
    SG_IDS -.->|contains| S2_ID
    SG_IDS -.->|contains| S3_ID
    
    S1_GRP -.->|references| SG_ID
    S2_GRP -.->|references| SG_ID
    S3_GRP -.->|references| SG_ID
    
    style SG fill:#c8e6c9
    style S1 fill:#e3f2fd
    style S2 fill:#e3f2fd
    style S3 fill:#e3f2fd
```

## Storage Operations API

```mermaid
classDiagram
    class StorageHandler {
        -db: IDBDatabase
        +initialize() Promise~void~
        +saveScrap(scrap) Promise~string~
        +getScrap(id) Promise~Scrap~
        +getAllScraps(filters) Promise~Scrap[]~
        +updateScrap(id, updates) Promise~void~
        +deleteScrap(id) Promise~void~
        +getOrCreateSourceGroup(url) Promise~string~
        +getSourceGroup(id) Promise~SourceGroup~
        +updateSourceGroup(id, updates) Promise~void~
        +searchScraps(query) Promise~Scrap[]~
        +checkStorageQuota() Promise~QuotaInfo~
        +exportScraps(scrapIds) Promise~Blob~
        -generateThumbnail(blob) Promise~Blob~
        -generateScrapId() string
        -generateGroupId(url) string
    }
    
    class Scrap {
        +id: string
        +timestamp: number
        +url: string
        +pageTitle: string
        +domain: string
        +screenshot: Blob
        +screenshotThumb: Blob
        +selectedText: string
        +selectedRegion: object
        +audioBlob: Blob
        +textNote: string
        +transcription: string
        +summary: string
        +imageAnalysis: object
        +sourceGroupId: string
    }
    
    class SourceGroup {
        +id: string
        +url: string
        +domain: string
        +scrapIds: string[]
        +firstCaptured: number
        +lastCaptured: number
        +scrapCount: number
    }
    
    class QuotaInfo {
        +usage: number
        +quota: number
        +available: number
        +percentUsed: number
    }
    
    StorageHandler --> Scrap : manages
    StorageHandler --> SourceGroup : manages
    StorageHandler --> QuotaInfo : returns
    Scrap --> SourceGroup : references via sourceGroupId
    SourceGroup --> Scrap : contains scrapIds array
```

## Key Design Decisions

### 1. Two-Store Architecture
- **scraps store**: Contains all capture data with full fidelity
- **sourceGroups store**: Lightweight aggregation layer for URL-based grouping
- Separation allows efficient queries without loading full scrap data

### 2. Bidirectional Linking
- Scraps reference their group via `sourceGroupId` (foreign key)
- Groups maintain array of `scrapIds` for reverse lookup
- Enables both "find group for scrap" and "find all scraps in group" queries

### 3. Thumbnail Strategy
- Generate 200x150px thumbnails immediately on save
- Store as separate field to avoid loading full screenshots in list views
- JPEG compression (quality 0.8) reduces size by ~90%
- Typical thumbnail: 10-20KB vs 200-500KB for full screenshot

### 4. Index Selection
- **timestamp**: Primary sorting for chronological views
- **url**: Fast lookup for source grouping logic
- **domain**: Enables domain-based filtering
- **sourceGroupId**: Efficient group expansion queries
- **lastCaptured**: Recent activity sorting for groups

### 5. Storage Quota Management
- Check quota before every save operation
- Warn at 100MB threshold (conservative for typical usage)
- Offer export/delete options to free space
- Never block saves unless quota truly exceeded

### 6. Source Group ID Generation
- Format: `group_{domain}_{urlhash}`
- Deterministic: same URL always generates same ID
- Enables idempotent group creation
- Example: `group_example.com_a3f2b9c1`

## Performance Characteristics

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Save scrap | O(log n) | Index updates |
| Get scrap by ID | O(1) | Primary key lookup |
| Get all scraps | O(n) | Full scan with index ordering |
| Get scraps by domain | O(k log n) | Index scan where k = matches |
| Get source group | O(log n) | URL index lookup |
| Expand source group | O(m log n) | m = scraps in group |
| Check storage quota | O(1) | Browser API call |
| Generate thumbnail | O(1) | Fixed size operation |

## Requirements Coverage

This diagram addresses the following requirements:

- **3.1**: IndexedDB storage in browser profile
- **3.2**: Unique ID generation (timestamp + random)
- **3.3**: Automatic URL-based source grouping with bidirectional links
- **3.4**: Thumbnail generation (200x150px)
- **3.5**: Blob storage for audio/screenshots, strings for text
- **3.6**: Storage quota warning at 100MB threshold
- **3.7**: Separate sourceGroups index for efficient retrieval
