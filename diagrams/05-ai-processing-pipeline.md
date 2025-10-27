# AI Processing Pipeline Flowchart

This flowchart details the complete AI processing logic in the Service Worker, showing parallel execution branches, multi-perspective image analysis, error handling, and storage operations.

## Key Features Highlighted

- **Parallel Processing**: Audio, text, and image processing happen simultaneously
- **Multi-Perspective Image Analysis**: Two-phase approach with content classification and conditional specialized analysis
- **Error Resilience**: Each branch handles failures independently without blocking the pipeline
- **Source Grouping**: Automatic linking of scraps from the same URL
- **Performance**: Promise.all synchronization for optimal speed

---

## Complete AI Processing Pipeline

```mermaid
flowchart TD
    START([Receive Capture Data<br/>from Content Script]) --> INIT[Initialize Processing<br/>Create base scrap object]
    
    INIT --> CHECK_AUDIO{Has Audio<br/>Blob?}
    INIT --> CHECK_TEXT{Has Text<br/>length > 100?}
    INIT --> CHECK_IMAGE{Has Screenshot<br/>with visual content?}
    
    %% BRANCH A: Audio Processing
    CHECK_AUDIO -->|Yes| BRANCH_A[Branch A:<br/>Audio Transcription]
    CHECK_AUDIO -->|No| SKIP_A[Skip Audio<br/>transcription = null]
    
    BRANCH_A --> CALL_PROMPT_AUDIO[Call Prompt API<br/>with audio input<br/>Timeout: 3s]
    CALL_PROMPT_AUDIO --> SUCCESS_A{Success?}
    SUCCESS_A -->|Yes| STORE_TRANS[Store transcription<br/>text string]
    SUCCESS_A -->|No| ERROR_A[Log error<br/>transcription = null<br/>Continue pipeline]
    
    %% BRANCH C: Text Processing
    CHECK_TEXT -->|Yes| BRANCH_C[Branch C:<br/>Text Summarization]
    CHECK_TEXT -->|No| SKIP_C[Skip Summarization<br/>summary = null]
    
    BRANCH_C --> CALL_SUMMARIZER[Call Summarizer API<br/>Timeout: 2s]
    CALL_SUMMARIZER --> SUCCESS_C{Success?}
    SUCCESS_C -->|Yes| STORE_SUM[Store summary<br/>text string]
    SUCCESS_C -->|No| ERROR_C[Log error<br/>summary = null<br/>Continue pipeline]
    
    %% BRANCH D: Image Processing (Multi-Perspective)
    CHECK_IMAGE -->|Yes| BRANCH_D[Branch D:<br/>Multi-Perspective<br/>Image Analysis]
    CHECK_IMAGE -->|No| SKIP_D[Skip Image Analysis<br/>imageAnalysis = null]
    
    BRANCH_D --> PHASE1[PHASE 1:<br/>Content Type Classification]
    PHASE1 --> CLASSIFY[Call Prompt API<br/>Classify content types<br/>Timeout: 1s]
    CLASSIFY --> CLASSIFY_SUCCESS{Success?}
    
    CLASSIFY_SUCCESS -->|No| CLASSIFY_ERROR["contentTypes = ['unknown']<br/>Continue with fallback"]
    CLASSIFY_SUCCESS -->|Yes| PARSE_TYPES["Parse JSON array<br/>e.g. ['text', 'code', 'chart']"]
    
    PARSE_TYPES --> PHASE2[PHASE 2:<br/>Conditional Deep Analysis]
    CLASSIFY_ERROR --> PHASE2
    
    PHASE2 --> CHECK_CODE{Contains<br/>'code'?}
    PHASE2 --> CHECK_CHART{Contains<br/>'chart' or<br/>'graph'?}
    PHASE2 --> CHECK_TABLE{Contains<br/>'table'?}
    PHASE2 --> ALWAYS_DESC[Always execute:<br/>General Description]
    
    %% Conditional Analysis Branches
    CHECK_CODE -->|Yes| ANALYZE_CODE[Call Prompt API<br/>Code Analysis<br/>Language, purpose, functions]
    CHECK_CODE -->|No| SKIP_CODE[codeAnalysis = null]
    
    CHECK_CHART -->|Yes| ANALYZE_CHART[Call Prompt API<br/>Chart Analysis<br/>Type, axes, trends, insights]
    CHECK_CHART -->|No| SKIP_CHART[chartAnalysis = null]
    
    CHECK_TABLE -->|Yes| ANALYZE_TABLE[Call Prompt API<br/>Table Analysis<br/>Columns, rows, data points]
    CHECK_TABLE -->|No| SKIP_TABLE[tableAnalysis = null]
    
    ALWAYS_DESC --> GENERAL_DESC[Call Prompt API<br/>General Description<br/>One sentence summary]
    
    %% Convergence of specialized analyses
    ANALYZE_CODE --> CODE_SUCCESS{Success?}
    CODE_SUCCESS -->|Yes| STORE_CODE[Store codeAnalysis]
    CODE_SUCCESS -->|No| ERROR_CODE[codeAnalysis = null]
    
    ANALYZE_CHART --> CHART_SUCCESS{Success?}
    CHART_SUCCESS -->|Yes| STORE_CHART[Store chartAnalysis]
    CHART_SUCCESS -->|No| ERROR_CHART[chartAnalysis = null]
    
    ANALYZE_TABLE --> TABLE_SUCCESS{Success?}
    TABLE_SUCCESS -->|Yes| STORE_TABLE[Store tableAnalysis]
    TABLE_SUCCESS -->|No| ERROR_TABLE[tableAnalysis = null]
    
    GENERAL_DESC --> DESC_SUCCESS{Success?}
    DESC_SUCCESS -->|Yes| STORE_DESC[Store generalDescription]
    DESC_SUCCESS -->|No| ERROR_DESC[generalDescription =<br/>'Image content']
    
    %% Combine all image analysis results
    STORE_CODE --> COMBINE_IMAGE
    ERROR_CODE --> COMBINE_IMAGE
    SKIP_CODE --> COMBINE_IMAGE
    
    STORE_CHART --> COMBINE_IMAGE
    ERROR_CHART --> COMBINE_IMAGE
    SKIP_CHART --> COMBINE_IMAGE
    
    STORE_TABLE --> COMBINE_IMAGE
    ERROR_TABLE --> COMBINE_IMAGE
    SKIP_TABLE --> COMBINE_IMAGE
    
    STORE_DESC --> COMBINE_IMAGE
    ERROR_DESC --> COMBINE_IMAGE
    
    COMBINE_IMAGE[Combine into<br/>ImageAnalysis object]
    
    %% Synchronization Point
    STORE_TRANS --> WAIT
    ERROR_A --> WAIT
    SKIP_A --> WAIT
    
    STORE_SUM --> WAIT
    ERROR_C --> WAIT
    SKIP_C --> WAIT
    
    COMBINE_IMAGE --> WAIT
    SKIP_D --> WAIT
    
    WAIT[SYNCHRONIZATION POINT<br/>Promise.all waits for<br/>all branches to complete<br/>Max wait: 5s total]
    
    WAIT --> CREATE[Create Complete Scrap Object<br/>with all AI results<br/>processed = true]
    
    %% Post-Processing
    CREATE --> THUMBNAIL[Generate Thumbnail<br/>Resize to 200x150px<br/>Canvas API]
    
    THUMBNAIL --> EXTRACT_DOMAIN[Extract domain<br/>from URL]
    
    EXTRACT_DOMAIN --> CHECK_GROUP{Source Group<br/>exists for<br/>this URL?}
    
    CHECK_GROUP -->|Yes| GET_GROUP[Get existing<br/>sourceGroupId]
    CHECK_GROUP -->|No| CREATE_GROUP[Create new<br/>Source Group<br/>group_domain_timestamp]
    
    GET_GROUP --> ADD_TO_GROUP[Add scrap ID to<br/>group.scrapIds array<br/>Update lastCaptured]
    CREATE_GROUP --> ADD_TO_GROUP
    
    ADD_TO_GROUP --> ASSIGN_GROUP[Assign sourceGroupId<br/>to scrap object]
    
    ASSIGN_GROUP --> SAVE[Save to IndexedDB<br/>scraps store]
    
    SAVE --> SAVE_SUCCESS{Save<br/>Successful?}
    
    SAVE_SUCCESS -->|Yes| UPDATE_GROUP[Update sourceGroups<br/>store with new scrapId]
    SAVE_SUCCESS -->|No| SAVE_ERROR[Log error<br/>Show notification<br/>Offer JSON export]
    
    UPDATE_GROUP --> BROADCAST[Broadcast Message<br/>SCRAP_CREATED<br/>to all Side Panel instances]
    
    BROADCAST --> RESPOND[Send Response<br/>to Content Script<br/>success: true, scrapId]
    
    RESPOND --> END_SUCCESS([END: Success<br/>Scrap saved and indexed])
    
    SAVE_ERROR --> RESPOND_ERROR[Send Response<br/>to Content Script<br/>success: false, error]
    
    RESPOND_ERROR --> END_FAILURE([END: Failure<br/>User notified])
    
    %% Styling
    style BRANCH_A fill:#e1f5ff,stroke:#01579b,stroke-width:3px
    style BRANCH_C fill:#e1f5ff,stroke:#01579b,stroke-width:3px
    style BRANCH_D fill:#e1f5ff,stroke:#01579b,stroke-width:3px
    
    style PHASE1 fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style PHASE2 fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    
    style ANALYZE_CODE fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    style ANALYZE_CHART fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    style ANALYZE_TABLE fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    style GENERAL_DESC fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    
    style WAIT fill:#fff9c4,stroke:#f57f17,stroke-width:3px
    style COMBINE_IMAGE fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    
    style END_SUCCESS fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px
    style END_FAILURE fill:#ffcdd2,stroke:#c62828,stroke-width:3px
    
    style ERROR_A fill:#ffebee,stroke:#c62828
    style ERROR_C fill:#ffebee,stroke:#c62828
    style ERROR_CODE fill:#ffebee,stroke:#c62828
    style ERROR_CHART fill:#ffebee,stroke:#c62828
    style ERROR_TABLE fill:#ffebee,stroke:#c62828
    style ERROR_DESC fill:#ffebee,stroke:#c62828
    style SAVE_ERROR fill:#ffebee,stroke:#c62828
```

---

## Performance Metrics

### Processing Time Budgets

| Scenario | Expected Time | Components |
|----------|--------------|------------|
| **Audio Only** | ~3s | Audio transcription (3s) |
| **Text Only** | ~2s | Text summarization (2s) |
| **Image Only (Simple)** | ~2s | Classification (1s) + General desc (1s) |
| **Image with Code** | ~3s | Classification (1s) + Code analysis (1s) + General desc (1s) |
| **Image with Chart** | ~3s | Classification (1s) + Chart analysis (1s) + General desc (1s) |
| **Image with Code + Chart** | ~4s | Classification (1s) + Code (1s) + Chart (1s) + General desc (1s) |
| **Full Capture** | ~5s | Audio (3s) + Text (2s) + Image (3s) in parallel = max(3,2,3) = 3s + processing overhead |

### Multi-Perspective Image Analysis Optimization

The two-phase approach minimizes API calls:

1. **Phase 1 (Classification)**: Single API call (~1s)
   - Identifies all content types present
   - Returns JSON array: `["text", "code", "chart"]`

2. **Phase 2 (Conditional Analysis)**: Only analyze what's detected
   - If no code detected → Skip code analysis (saves ~1s)
   - If no chart detected → Skip chart analysis (saves ~1s)
   - If no table detected → Skip table analysis (saves ~1s)
   - Always run general description (fallback)

**Example Savings**:
- Pure text image: 1 classification + 1 general = 2 API calls (~2s)
- Code + chart image: 1 classification + 2 specialized + 1 general = 4 API calls (~4s)
- Without optimization: Always 5 API calls regardless of content (~5s)

---

## Error Handling Strategy

### Graceful Degradation

Each processing branch handles errors independently:

1. **Audio Transcription Fails**
   - Store `transcription = null`
   - Continue with text and image processing
   - User can still see screenshot and text

2. **Text Summarization Fails**
   - Store `summary = null`
   - Continue with audio and image processing
   - User can still see original text

3. **Image Analysis Fails**
   - Store minimal ImageAnalysis object:
     ```javascript
     {
       contentTypes: ['unknown'],
       generalDescription: 'Image analysis unavailable'
     }
     ```
   - Continue with audio and text processing
   - User can still see screenshot thumbnail

4. **All AI Processing Fails**
   - Save scrap with raw data only
   - Set `processed = false`
   - Store error details in `processingErrors` object
   - Show retry option in Side Panel UI

### Retry Logic

For transient failures (network timeouts, API rate limits):

```javascript
async function retryWithBackoff(fn, maxAttempts = 3) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) throw error;
      
      const delay = Math.pow(2, attempt - 1) * 1000; // 1s, 2s, 4s
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

---

## Source Grouping Logic

### Automatic URL-Based Linking

When a scrap is saved, the system automatically:

1. **Extract Domain**: Parse URL to get domain (e.g., `github.com`)
2. **Check Existing Group**: Query `sourceGroups` store by URL
3. **Create or Update**:
   - If group exists: Add scrap ID to `scrapIds` array, update `lastCaptured`
   - If new: Create group with `id = group_domain_timestamp`
4. **Bidirectional Link**: Store `sourceGroupId` in scrap object

### Benefits

- **Contextual Recall**: See all thoughts on a particular article/page
- **Temporal Tracking**: Know when you first and last captured from a source
- **Efficient Queries**: Use `sourceGroupId` index for fast retrieval
- **UI Enhancement**: Show "View Source Group" button with count badge

---

## Requirements Coverage

This diagram satisfies the following requirements:

- **2.1**: Audio transcription using Prompt API (Branch A)
- **2.2**: Text summarization using Summarizer API (Branch C)
- **2.3**: Image analysis using Prompt API multimodal (Branch D)
- **2.4**: Parallel execution with Promise.all (Synchronization Point)
- **2.5**: Error handling with partial data save (Error branches)
- **11.1**: Content type classification (Phase 1)
- **11.2**: Conditional specialized analysis (Phase 2 decision tree)
- **11.3**: Code analysis for programming content
- **11.4**: Chart analysis for data visualizations
- **11.5**: Table analysis for tabular data
- **11.6**: General description fallback
- **11.7**: Parallel execution of specialized analyses

---

## Implementation Notes

### Rate Limiting

The AI Handler implements a queue to limit concurrent API calls:

```javascript
private requestQueue: Promise<any>[] = [];
private readonly MAX_CONCURRENT = 3; // Requirement 8.3

async queueRequest<T>(fn: () => Promise<T>): Promise<T> {
  while (this.requestQueue.length >= this.MAX_CONCURRENT) {
    await Promise.race(this.requestQueue);
  }
  
  const promise = fn();
  this.requestQueue.push(promise);
  
  promise.finally(() => {
    const index = this.requestQueue.indexOf(promise);
    if (index > -1) this.requestQueue.splice(index, 1);
  });
  
  return promise;
}
```

### Session Caching

AI sessions are cached for 5 minutes to reduce initialization overhead:

```javascript
private sessions: Map<string, any> = new Map();
private sessionTimestamps: Map<string, number> = new Map();
private readonly SESSION_TIMEOUT = 300000; // 5 minutes

async getSession(type: string, creator: () => Promise<any>) {
  this.cleanupStaleSessions();
  
  if (!this.sessions.has(type)) {
    const session = await creator();
    this.sessions.set(type, session);
    this.sessionTimestamps.set(type, Date.now());
  }
  
  this.sessionTimestamps.set(type, Date.now());
  return this.sessions.get(type);
}
```

### Memory Cleanup

After processing completes, the Service Worker cleans up:

```javascript
async function cleanupAfterCapture() {
  // Release blob URLs
  URL.revokeObjectURL(screenshotUrl);
  URL.revokeObjectURL(audioUrl);
  
  // Clear temporary data
  tempData = null;
  
  // Force garbage collection hint
  if (global.gc) global.gc();
}
```

---

## Hackathon Differentiator

The **Multi-Perspective Image Analysis** (Branch D, Phase 2) is the key innovation:

- **Smart**: Only analyzes what's actually in the image
- **Fast**: Parallel execution of specialized analyses
- **Comprehensive**: Covers code, charts, tables, and general content
- **Efficient**: Saves API calls by skipping irrelevant analyses
- **Robust**: Always provides general description as fallback

This demonstrates sophisticated use of Chrome's multimodal AI capabilities beyond simple image captioning.
