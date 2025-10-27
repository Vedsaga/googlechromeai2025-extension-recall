# Multi-Perspective Image Analysis Detail Diagram

## Overview

This diagram illustrates the **key hackathon differentiator**: a sophisticated two-phase multimodal AI approach that intelligently analyzes images based on their content type, avoiding unnecessary API calls while providing deep, specialized insights.

## Architecture Diagram

```mermaid
flowchart TD
    START([Screenshot Captured]) --> PHASE1[<b>PHASE 1: Content Classification</b><br/>Single API Call<br/>~500ms]
    
    PHASE1 --> CLASSIFY[Prompt API:<br/>Identify ALL content types<br/>text, code, chart, graph,<br/>table, diagram, photo, UI]
    
    CLASSIFY --> PARSE{Parse JSON<br/>Response}
    PARSE -->|Success| TYPES[contentTypes Array]
    PARSE -->|Failure| FALLBACK["contentTypes = ['unknown']"]
    
    TYPES --> PHASE2[<b>PHASE 2: Conditional Analysis</b><br/>Parallel API Calls]
    FALLBACK --> PHASE2
    
    PHASE2 --> DECISION_CODE{Contains<br/>'code'?}
    PHASE2 --> DECISION_CHART{Contains<br/>'chart' or<br/>'graph'?}
    PHASE2 --> DECISION_TABLE{Contains<br/>'table'?}
    PHASE2 --> ALWAYS[Always Execute]
    
    DECISION_CODE -->|Yes| CODE_ANALYSIS[<b>Code Analysis</b><br/>Prompt API<br/>~1-2s<br/><br/>Identify:<br/>• Programming language<br/>• Purpose<br/>• Key functions/methods]
    DECISION_CODE -->|No| SKIP_CODE[Skip<br/>codeAnalysis = null]
    
    DECISION_CHART -->|Yes| CHART_ANALYSIS[<b>Chart Analysis</b><br/>Prompt API<br/>~1-2s<br/><br/>Describe:<br/>• Chart type<br/>• Axes labels<br/>• Data trends<br/>• Key insights]
    DECISION_CHART -->|No| SKIP_CHART[Skip<br/>chartAnalysis = null]
    
    DECISION_TABLE -->|Yes| TABLE_ANALYSIS[<b>Table Analysis</b><br/>Prompt API<br/>~1-2s<br/><br/>Describe:<br/>• Column headers<br/>• Approximate rows<br/>• Notable data points]
    DECISION_TABLE -->|No| SKIP_TABLE[Skip<br/>tableAnalysis = null]
    
    ALWAYS --> GENERAL[<b>General Description</b><br/>Prompt API<br/>~500ms<br/><br/>Fallback:<br/>• One-sentence summary<br/>• Always executed]
    
    CODE_ANALYSIS --> SYNC
    SKIP_CODE --> SYNC
    CHART_ANALYSIS --> SYNC
    SKIP_CHART --> SYNC
    TABLE_ANALYSIS --> SYNC
    SKIP_TABLE --> SYNC
    GENERAL --> SYNC
    
    SYNC[<b>Promise.all Synchronization</b><br/>Wait for all parallel calls]
    
    SYNC --> OUTPUT[<b>ImageAnalysis Object</b>]
    
    OUTPUT --> STRUCT{{"<b>Output Structure:</b><br/>{<br/>  contentTypes: string[],<br/>  codeAnalysis?: string,<br/>  chartAnalysis?: string,<br/>  tableAnalysis?: string,<br/>  generalDescription: string<br/>}"}}
    
    STRUCT --> END([Return to Service Worker])
    
    style PHASE1 fill:#e1f5ff,stroke:#01579b,stroke-width:3px
    style PHASE2 fill:#fff9c4,stroke:#f57f17,stroke-width:3px
    style CODE_ANALYSIS fill:#c8e6c9
    style CHART_ANALYSIS fill:#c8e6c9
    style TABLE_ANALYSIS fill:#c8e6c9
    style GENERAL fill:#c8e6c9
    style SYNC fill:#ffccbc
    style OUTPUT fill:#d1c4e9
    style STRUCT fill:#f8bbd0
```

## Performance Scenarios

```mermaid
gantt
    title Performance Metrics by Content Type
    dateFormat X
    axisFormat %Ls
    
    section Pure Text
    Classification    :0, 500ms
    General Desc      :500ms, 500ms
    Total: ~1s        :milestone, 1000ms, 0ms
    
    section Chart Only
    Classification    :0, 500ms
    Chart Analysis    :500ms, 1500ms
    General Desc      :500ms, 1500ms
    Total: ~2.5s      :milestone, 2500ms, 0ms
    
    section Code + Chart
    Classification    :0, 500ms
    Code Analysis     :500ms, 1500ms
    Chart Analysis    :500ms, 1500ms
    General Desc      :500ms, 1500ms
    Total: ~2.5s      :milestone, 2500ms, 0ms
    
    section Complex (All)
    Classification    :0, 500ms
    Code Analysis     :500ms, 2000ms
    Chart Analysis    :500ms, 2000ms
    Table Analysis    :500ms, 2000ms
    General Desc      :500ms, 2000ms
    Total: ~3s        :milestone, 3000ms, 0ms
```

## Decision Tree

```mermaid
graph TD
    START[Image Captured] --> CLASS[Phase 1: Classification]
    
    CLASS --> RESULTS{Content Types<br/>Detected}
    
    RESULTS -->|text only| SCENARIO1[<b>Scenario 1: Pure Text</b><br/>API Calls: 2<br/>Time: ~1s<br/><br/>✓ Classification<br/>✓ General Description]
    
    RESULTS -->|chart/graph| SCENARIO2[<b>Scenario 2: Chart</b><br/>API Calls: 3<br/>Time: ~2.5s<br/><br/>✓ Classification<br/>✓ Chart Analysis<br/>✓ General Description]
    
    RESULTS -->|code| SCENARIO3[<b>Scenario 3: Code</b><br/>API Calls: 3<br/>Time: ~2.5s<br/><br/>✓ Classification<br/>✓ Code Analysis<br/>✓ General Description]
    
    RESULTS -->|code + chart| SCENARIO4[<b>Scenario 4: Code + Chart</b><br/>API Calls: 4<br/>Time: ~2.5s<br/><br/>✓ Classification<br/>✓ Code Analysis ║<br/>✓ Chart Analysis ║<br/>✓ General Description<br/><br/>║ = Parallel]
    
    RESULTS -->|code + chart + table| SCENARIO5[<b>Scenario 5: Complex</b><br/>API Calls: 5<br/>Time: ~3s<br/><br/>✓ Classification<br/>✓ Code Analysis ║<br/>✓ Chart Analysis ║<br/>✓ Table Analysis ║<br/>✓ General Description<br/><br/>║ = Parallel]
    
    RESULTS -->|unknown/error| SCENARIO6[<b>Scenario 6: Fallback</b><br/>API Calls: 2<br/>Time: ~1s<br/><br/>✓ Classification<br/>✓ General Description]
    
    style SCENARIO1 fill:#e8f5e9
    style SCENARIO2 fill:#fff3e0
    style SCENARIO3 fill:#fff3e0
    style SCENARIO4 fill:#ffe0b2
    style SCENARIO5 fill:#ffccbc
    style SCENARIO6 fill:#f5f5f5
```

## Key Hackathon Differentiators

### 1. Intelligent Resource Management
- **Conditional Execution**: Only runs specialized analyses when relevant content is detected
- **Parallel Processing**: All Phase 2 analyses run simultaneously via `Promise.all`
- **Performance Budget**: Stays within 3-second total processing time even for complex images

### 2. Multi-Perspective Understanding
- **Code Perspective**: Understands programming languages, identifies functions, explains purpose
- **Data Perspective**: Interprets charts, extracts trends, identifies insights
- **Structural Perspective**: Analyzes tables, understands data organization
- **General Perspective**: Always provides fallback description

### 3. Sophisticated Content Classification
- **Single Classification Call**: Identifies ALL content types in one API request
- **JSON-Structured Response**: Enables programmatic decision-making
- **Graceful Degradation**: Falls back to general description if classification fails

### 4. Real-World Use Cases

**Use Case 1: Developer Documentation**
- Screenshot of API documentation with code examples
- Detects: `["text", "code"]`
- Provides: Language identification + code explanation + general context
- Time: ~2.5s

**Use Case 2: Data Analysis**
- Screenshot of dashboard with multiple charts
- Detects: `["chart", "graph", "table"]`
- Provides: Chart interpretation + table summary + general overview
- Time: ~3s

**Use Case 3: Research Paper**
- Screenshot of academic paper with equations and diagrams
- Detects: `["text", "diagram"]`
- Provides: General description (no specialized analysis needed)
- Time: ~1s

### 5. Integration with Other AI Features

The multi-perspective image analysis enriches other extension features:

- **Email Drafts**: Uses `codeAnalysis` and `chartAnalysis` to write more accurate, context-aware emails
- **Semantic Search**: Indexes all analysis fields for comprehensive search coverage
- **Source Grouping**: Links related captures with similar content types
- **Magic Buttons**: Refine and Clean Up operations leverage image context

## Implementation Reference

See `design.md` Section 3 (AI Handler) for the complete implementation:

```javascript
async analyzeImageContent(imageBlob: Blob): Promise<ImageAnalysis> {
  // Phase 1: Classification (single call)
  const contentTypes = await classifyContent(imageBlob);
  
  // Phase 2: Conditional parallel analysis
  const [code, chart, table, general] = await Promise.all([
    contentTypes.includes('code') ? analyzeCode(imageBlob) : null,
    contentTypes.some(t => t.includes('chart') || t.includes('graph')) 
      ? analyzeChart(imageBlob) : null,
    contentTypes.includes('table') ? analyzeTable(imageBlob) : null,
    getGeneralDescription(imageBlob), // Always executed
  ]);
  
  return { contentTypes, codeAnalysis: code, chartAnalysis: chart, 
           tableAnalysis: table, generalDescription: general };
}
```

## Requirements Coverage

This diagram addresses the following requirements from `requirements.md`:

- **11.1**: Content type classification within 1 second
- **11.2**: Conditional specialized analysis based on detected types
- **11.3**: Code analysis for programming content (language, purpose, functions)
- **11.4**: Chart analysis for data visualizations (type, trends, insights)
- **11.5**: Table analysis for tabular data (headers, rows, data points)
- **11.6**: General description as fallback
- **11.7**: Parallel execution within 5-second budget (achieves 3s)
- **11.8**: Unified scrap context combining audio, text, and image
- **11.9**: Enhanced email drafts using multi-perspective analysis
- **11.10**: Demo video showcasing 3+ multimodal combinations

---

**Status**: ✅ Complete - Ready for hackathon demonstration
