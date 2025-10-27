# Architecture Diagram Creation Plan

## Overview
This plan outlines the tasks for creating detailed architecture diagrams for the Recall Chrome extension using Mermaid syntax in markdown files.

**Purpose**: Create comprehensive visual documentation that can be used to:
1. Communicate architecture to team/judges
2. Validate design decisions
3. Guide future implementation

---

## Diagram Creation Tasks

- [x] 1. Create Component Architecture Diagram
  - Show all 4 layers: UI, Business Logic, Services, External Systems
  - Include all components: Content Script, Side Panel, Service Worker, AI Handler, Storage Handler
  - Show communication patterns: chrome.runtime.sendMessage, Message Ports, function calls, async calls
  - Add color coding: Blue (UI), Green (Business Logic), Yellow (Services), Gray (External)
  - Save as Mermaid diagram in diagrams/01-component-architecture.md
  - _Requirements: 7.1, 7.2, 7.5_

- [x] 2. Create Capture Flow Sequence Diagram
  - Show complete user interaction flow from Alt+Drag to storage
  - Include all participants: User, Content Script, Service Worker, AI Handler, Storage Handler, Gemini Nano, IndexedDB
  - Show parallel AI processing (audio transcription, text summarization, multi-perspective image analysis)
  - Mark timing annotations (50ms, 200ms, 3s, etc.)
  - Show error handling paths
  - Highlight automatic screenshot and text extraction
  - Save as Mermaid diagram in diagrams/02-capture-flow-sequence.md
  - _Requirements: 1.1-1.6, 2.1-2.5, 11.1-11.7_

- [x] 3. Create Data Model Schema (ERD)
  - Design scraps object store with all fields
  - Design sourceGroups object store with all fields
  - Show relationship: scraps.sourceGroupId → sourceGroups.id (many-to-one)
  - Include all indexes: timestamp, url, domain, sourceGroupId for scraps; url, lastCaptured for sourceGroups
  - Show data types for each field
  - Highlight imageAnalysis nested structure (contentTypes, codeAnalysis, chartAnalysis, tableAnalysis, generalDescription)
  - Use crow's foot notation for relationships
  - Save as Mermaid diagram in diagrams/03-data-model-schema.md
  - _Requirements: 3.1-3.7, 11.1-11.6_

- [x] 4. Create Capture UI State Machine Diagram
  - Show all 8 states: IDLE, SELECTING, READY, RECORDING, TYPING, PROCESSING, COMPLETE, ERROR
  - Show all transitions with trigger events
  - Include timeout transitions (5 min recording, 10s processing)
  - Add state descriptions/notes for each state
  - Mark initial state (IDLE) and final states (COMPLETE)
  - Show error recovery paths
  - Save as Mermaid diagram in diagrams/04-capture-ui-state-machine.md
  - _Requirements: 1.1-1.6, 9.1_

- [x] 5. Create AI Processing Pipeline Flowchart
  - Show decision points: Has audio? Has text >100? Has screenshot?
  - Show parallel execution branches (Branch A: audio, Branch C: text, Branch D: image)
  - Detail multi-perspective image analysis flow:
    - Step 1: Content type classification
    - Step 2: Conditional specialized analyses (code, chart, table)
    - Step 3: General description (always)
  - Show Promise.all synchronization point
  - Show error handling for each branch (continue with null)
  - Show thumbnail generation step
  - Show source grouping logic (exists? add : create)
  - Show final save to IndexedDB
  - Use diamond shapes for decisions, rectangles for processes
  - Highlight parallel branches with color coding
  - Save as Mermaid diagram in diagrams/05-ai-processing-pipeline.md
  - _Requirements: 2.1-2.5, 11.1-11.7_

- [x] 6. Create Message Flow Diagram
  - Show all 9 message types with swimlanes:
    - CAPTURE_COMPLETE (Content Script → Service Worker)
    - SCRAP_SAVED (Service Worker → Side Panel broadcast)
    - GET_ALL_SCRAPS (Side Panel → Service Worker)
    - CLEANUP_TEXT (Side Panel → Service Worker)
    - REFINE_TEXT (Side Panel → Service Worker)
    - DRAFT_EMAIL (Side Panel → Service Worker)
    - SEARCH_SCRAPS (Side Panel → Service Worker)
    - DELETE_SCRAP (Side Panel → Service Worker)
    - GET_SOURCE_GROUP (Side Panel → Service Worker)
  - Show request-response pairs with dotted return arrows
  - Show broadcast messages fanning out to multiple Side Panel instances
  - Add timeout annotations (10s, 5s)
  - Include payload structure for each message
  - Save as Mermaid diagram in diagrams/06-message-flow.md
  - _Requirements: 7.1-7.5_

- [x] 7. Create Multi-Perspective Image Analysis Detail Diagram (HACKATHON KEY!)
  - Show two-phase approach:
    - Phase 1: Content Type Classification (single API call)
    - Phase 2: Conditional Specialized Analysis (parallel API calls)
  - Show decision tree for conditional execution:
    - If 'code' detected → Code Analysis
    - If 'chart'/'graph' detected → Chart Analysis
    - If 'table' detected → Table Analysis
    - Always → General Description
  - Show ImageAnalysis output structure
  - Show performance metrics for each scenario:
    - Pure text: ~1s
    - Chart: ~3s
    - Code + Chart: ~3s
  - Highlight this as the key hackathon differentiator
  - Save as Mermaid diagram in diagrams/07-multi-perspective-image-analysis.md
  - _Requirements: 11.1-11.7_

- [x] 8. Create Storage Architecture Diagram
  - Show IndexedDB structure with two stores
  - Show index strategy for each store
  - Show source grouping logic flow:
    - Check if URL exists in sourceGroups
    - If yes: add scrapId to existing group
    - If no: create new group
  - Show bidirectional linking between scraps and sourceGroups
  - Show thumbnail generation and storage
  - Show storage quota checking logic
  - Save as Mermaid diagram in diagrams/08-storage-architecture.md
  - _Requirements: 3.1-3.7_

- [x] 9. Create Error Handling Flow Diagram
  - Show error scenarios and recovery paths:
    - Microphone permission denied → Save screenshot + text, allow retry
    - AI API unavailable → Save raw data, show retry option
    - AI processing fails → Save with partial data, mark errors
    - Storage quota exceeded → Offer JSON export
    - IndexedDB connection fails → Fallback to chrome.storage.local
    - Message timeout → Show error, allow retry
  - Show exponential backoff retry logic (1s, 2s, 4s)
  - Show graceful degradation strategy
  - Save as Mermaid diagram in diagrams/09-error-handling-flow.md
  - _Requirements: 9.1-9.5_

- [x] 10. Create Performance Optimization Diagram
  - Show rate limiting queue (max 3 concurrent AI calls)
  - Show session caching with 5-minute timeout
  - Show virtual scrolling strategy (4 visible + 4 buffer = 8 rendered)
  - Show lazy loading for thumbnails with IntersectionObserver
  - Show memory cleanup after capture
  - Show performance budgets for each operation
  - Save as Mermaid diagram in diagrams/10-performance-optimization.md
  - _Requirements: 8.1-8.5_

- [x] 11. Create Side Panel UI Component Diagram
  - Show component hierarchy:
    - SidePanelController (root)
    - SearchBar component
    - ScrapList component (with virtual scrolling)
    - ScrapCard component (with all sub-elements)
    - SourceGroupView component
  - Show data flow between components
  - Show event handling (click, input, scroll)
  - Show state management
  - Save as Mermaid diagram in diagrams/11-sidepanel-ui-components.md
  - _Requirements: 4.1-4.6_

- [x] 12. Create Extension Lifecycle Diagram
  - Show extension installation and initialization
  - Show service worker lifecycle (install, activate, terminate, reactivate)
  - Show state restoration after service worker termination
  - Show IndexedDB connection initialization
  - Show AI API session initialization
  - Show side panel open/close lifecycle
  - Save as Mermaid diagram in diagrams/12-extension-lifecycle.md
  - _Requirements: 12.1-12.5_

---

## Diagram Validation Checklist

Before marking a diagram as complete, verify:
- [x] All components/actors from requirements are included
- [x] All relationships/connections are clearly shown
- [x] Timing/performance annotations are accurate
- [x] Error paths are documented
- [x] Diagram matches the written design document
- [x] Diagram is clear and understandable without additional explanation
- [x] Diagram follows standard Mermaid notation
- [x] Diagram is saved in the correct diagrams/ folder with proper naming

---

## Notes

- Diagrams 1-6 are **CRITICAL** - these cover the core architecture
- Diagram 7 (Multi-Perspective Image Analysis) is the **HACKATHON DIFFERENTIATOR**
- Diagrams 8-12 provide additional detail for implementation
- All diagrams use Mermaid syntax in markdown files
- Diagrams are stored in `.kiro/specs/vocal-context-architecture/diagrams/` folder
- Each diagram has a descriptive filename: `##-diagram-name.md`
