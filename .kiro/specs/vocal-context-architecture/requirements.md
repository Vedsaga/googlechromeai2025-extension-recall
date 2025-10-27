# Requirements Document

## Introduction

Recall is a privacy-first Chrome extension that enables users to capture web content with voice annotations and process them locally using Chrome's built-in AI (Gemini Nano). The architecture must support real-time content capture, on-device AI processing, local storage, and semantic search while maintaining user privacy through 100% client-side processing.

## Glossary

- **Extension**: The Recall Chrome Extension system
- **Content Script**: JavaScript code injected into web pages for content capture
- **Service Worker**: Background script managing AI processing and storage
- **Side Panel**: Chrome's side panel UI for viewing and managing scraps
- **Scrap**: A captured piece of web content with voice annotation and metadata
- **Source Group**: Automatically linked scraps from the same source URL
- **Collection**: User-created manual grouping of scraps across different sources
- **Gemini Nano**: Chrome's built-in on-device AI model
- **Prompt API**: Chrome AI API for text generation and analysis
- **Summarizer API**: Chrome AI API for text summarization
- **Rewriter API**: Chrome AI API for text refinement
- **Proofreader API**: Chrome AI API for grammar and punctuation correction
- **IndexedDB**: Browser-based local storage for scrap persistence
- **Selection Region**: User-defined rectangular area on a webpage to capture
- **Magic Buttons**: AI-powered transformation buttons (Clean Up, Refine, Draft Email)
- **Capturable Content**: Any content viewable in a browser tab including web pages, PDFs, SVGs, PNGs, and other file types
- **Text Note**: User-typed annotation as an alternative to voice recording
- **Remote Model**: External AI service (e.g., Gemini API) for advanced analysis beyond on-device capabilities
- **Hybrid Mode**: Architecture supporting both on-device (Gemini Nano) and cloud-based (Remote Model) AI processing

## Requirements

### Requirement 1: Universal Content Capture Architecture

**User Story:** As a user, I want to capture any content viewable in my browser (web pages, PDFs, images, SVGs) with voice or text annotations so that I can record my thoughts on any material I'm reviewing.

#### Acceptance Criteria

1. WHEN the user presses and holds the Alt key on any browser tab, THE Extension SHALL inject a selection overlay within 50 milliseconds regardless of content type (HTML, PDF, SVG, PNG, or other viewable formats)
2. WHILE the user drags the mouse with Alt key pressed, THE Extension SHALL render a visual selection rectangle that updates at 60 frames per second
3. WHEN the user releases the mouse button, THE Extension SHALL display both microphone and text note buttons within 100 milliseconds at the bottom-right corner of the selected region
4. WHEN the user clicks the microphone button, THE Extension SHALL request microphone permission if not already granted and initiate audio recording within 200 milliseconds
5. WHEN the user clicks the text note button, THE Extension SHALL display an inline text input field for typed annotations within 100 milliseconds
6. WHEN the user stops recording or completes text input, THE Extension SHALL capture a screenshot of the selected region, extract text content if available, and collect page metadata within 500 milliseconds

### Requirement 2: AI Processing Pipeline Architecture

**User Story:** As a user, I want my captured content to be processed by AI locally so that my data remains private and I get enriched, searchable information.

#### Acceptance Criteria

1. WHEN a scrap is captured, THE Service Worker SHALL execute audio transcription using the Prompt API within 3 seconds
2. WHEN selected text exceeds 100 characters, THE Service Worker SHALL generate a summary using the Summarizer API within 2 seconds
3. WHEN a screenshot contains visual elements, THE Service Worker SHALL analyze the image using the Prompt API with multimodal capabilities within 3 seconds
4. THE Service Worker SHALL execute all AI processing tasks in parallel to minimize total processing time
5. IF any AI processing task fails, THEN THE Service Worker SHALL save the scrap with partial data and set an error flag for retry capability

### Requirement 3: Storage and Source Grouping Architecture

**User Story:** As a user, I want my scraps to be stored locally with automatic URL-based linking so that multiple captures from the same source are grouped together for easy recall.

#### Acceptance Criteria

1. THE Extension SHALL store all scrap data in IndexedDB within the user's browser profile
2. WHEN a scrap is saved, THE Extension SHALL generate a unique identifier using timestamp and random string within 10 milliseconds
3. WHEN a scrap is saved, THE Extension SHALL check if other scraps exist with the same source URL and automatically create bidirectional links to form a Source Group
4. THE Extension SHALL create a thumbnail version of each screenshot at 200x150 pixels to optimize list view performance
5. THE Extension SHALL store original audio and screenshots as Blob objects, and text notes as strings in the IndexedDB scraps store
6. WHEN storage quota falls below 100 megabytes, THE Extension SHALL display a warning notification to the user
7. THE Extension SHALL maintain a separate sourceGroups index that organizes scraps by URL for efficient retrieval of related content

### Requirement 4: Side Panel UI Architecture with Source Group Views

**User Story:** As a user, I want to view and manage my scraps in a side panel with clear indication of related captures so that I can see all my thoughts on a particular source.

#### Acceptance Criteria

1. WHEN the user clicks the extension icon, THE Extension SHALL open the side panel within 200 milliseconds with a smooth slide-in animation
2. THE Side Panel SHALL display scraps as cards showing thumbnail, page title, domain, timestamp, transcription preview, and a badge indicating number of related scraps from the same source URL
3. WHEN the side panel loads, THE Extension SHALL retrieve and render all scraps from IndexedDB within 500 milliseconds
4. WHEN a scrap has related captures from the same URL, THE Side Panel SHALL display a "View Source Group" button that expands to show all linked scraps in chronological order
5. THE Side Panel SHALL implement virtual scrolling for lists exceeding 50 scraps to maintain performance
6. WHEN a scrap card is clicked, THE Side Panel SHALL expand to show full details with smooth animation within 300 milliseconds

### Requirement 5: Magic Buttons Processing Architecture

**User Story:** As a user, I want to transform my raw voice transcriptions into polished content so that I can use them in professional contexts.

#### Acceptance Criteria

1. WHEN the user clicks the Clean Up button, THE Extension SHALL process the transcription using the Proofreader API and display results within 3 seconds
2. WHEN the user clicks the Refine button, THE Extension SHALL present tone options and process the text using the Rewriter API within 5 seconds
3. WHEN the user clicks the Draft Email button, THE Extension SHALL generate a professional email using the Writer API including subject line and body within 5 seconds
4. THE Extension SHALL store each processing version separately in the scrap object to maintain version history
5. THE Extension SHALL allow users to switch between original, cleaned, refined, and draft versions without reprocessing

### Requirement 6: Semantic Search Architecture

**User Story:** As a user, I want to search my scraps using natural language so that I can find relevant information even when I don't remember exact keywords.

#### Acceptance Criteria

1. WHEN the user types in the search box, THE Extension SHALL debounce input for 500 milliseconds before triggering search
2. WHEN search is triggered, THE Extension SHALL build a context summary of all scraps including transcription and summary snippets
3. THE Extension SHALL use the Prompt API to perform semantic matching between the query and scrap context within 3 seconds
4. THE Extension SHALL rank results by relevance score and display them with highlighted matching text within 100 milliseconds of search completion
5. WHEN no results match the query, THE Extension SHALL display helpful suggestions and allow the user to view all scraps

### Requirement 7: Inter-Component Communication Architecture

**User Story:** As a developer, I want clear communication patterns between extension components so that the system is maintainable and reliable.

#### Acceptance Criteria

1. THE Content Script SHALL communicate with the Service Worker using chrome.runtime.sendMessage API for all capture operations
2. THE Service Worker SHALL communicate with the Side Panel using chrome.runtime.sendMessage and message ports for real-time updates
3. WHEN a scrap is saved, THE Service Worker SHALL broadcast an update message to all open Side Panel instances within 100 milliseconds
4. THE Extension SHALL implement a request-response pattern with timeout handling of 10 seconds for all async operations
5. THE Extension SHALL use structured message types with action and payload properties for all inter-component communication

### Requirement 8: Performance and Resource Management Architecture

**User Story:** As a user, I want the extension to perform efficiently so that it doesn't slow down my browsing or consume excessive resources.

#### Acceptance Criteria

1. THE Extension SHALL limit memory usage per scrap to 2 megabytes including all assets and metadata
2. THE Extension SHALL implement lazy loading for scrap thumbnails in the side panel list view
3. WHEN AI processing is active, THE Service Worker SHALL limit concurrent API calls to 3 simultaneous requests
4. THE Extension SHALL implement caching for AI model initialization to reduce cold start latency from 5 seconds to 1 second
5. THE Extension SHALL clean up DOM elements and event listeners within 100 milliseconds after content capture completion

### Requirement 9: Error Handling and Resilience Architecture

**User Story:** As a user, I want the extension to handle errors gracefully so that I don't lose my captured content when something goes wrong.

#### Acceptance Criteria

1. IF microphone permission is denied, THEN THE Extension SHALL save the screenshot and text content and allow audio retry
2. IF an AI API is unavailable, THEN THE Extension SHALL save raw data and display a clear error message with retry option
3. IF IndexedDB storage fails, THEN THE Extension SHALL offer to export the scrap as JSON file for manual backup
4. THE Extension SHALL implement exponential backoff retry logic with maximum 3 attempts for failed AI processing tasks
5. WHEN any component encounters an error, THE Extension SHALL log detailed error information to console without exposing sensitive user data

### Requirement 10: Privacy and Security Architecture

**User Story:** As a user, I want my data to remain completely private so that I can trust the extension with sensitive information.

#### Acceptance Criteria

1. THE Extension SHALL process all data locally using Chrome's built-in AI without any network requests to external servers
2. THE Extension SHALL store all data in the user's local browser profile without synchronization to cloud services
3. THE Extension SHALL request only necessary permissions: activeTab, storage, sidePanel, and microphone
4. THE Extension SHALL not include any analytics, tracking, or telemetry code in the production build
5. THE Extension SHALL provide clear documentation explaining that all processing occurs on-device and no data leaves the user's computer

### Requirement 11: Multi-Perspective Multimodal AI Architecture

**User Story:** As a hackathon judge, I want to see sophisticated multi-perspective image analysis that demonstrates deep understanding of different content types so that the extension showcases advanced Chrome AI capabilities.

#### Acceptance Criteria

1. WHEN a screenshot is captured, THE Extension SHALL perform content type classification to identify all present content types (text, code, chart, graph, table, diagram, photo, UI) within 1 second
2. THE Extension SHALL conditionally execute specialized analysis based on detected content types: code analysis for programming content, chart analysis for data visualizations, and table analysis for tabular data
3. WHEN code is detected in a screenshot, THE Extension SHALL identify the programming language, purpose, and key functions using the Prompt API within 2 seconds
4. WHEN charts or graphs are detected, THE Extension SHALL describe the chart type, axis labels, data trends, and key insights using the Prompt API within 2 seconds
5. WHEN tables are detected, THE Extension SHALL describe column headers, approximate row count, and notable data points using the Prompt API within 2 seconds
6. THE Extension SHALL always generate a general description as a fallback regardless of content type detection results
7. THE Extension SHALL execute specialized analyses in parallel to minimize total processing time while staying within the 5-second total budget
8. THE Extension SHALL combine audio transcription, text extraction, and multi-perspective image analysis into a unified scrap context
9. THE Extension SHALL use multi-perspective image analysis results when generating email drafts to provide richer, more accurate content
10. THE Extension SHALL demonstrate at least 3 different multimodal combinations in the demo video: (1) audio + image, (2) text + image, and (3) audio + text + image processing with content type detection

### Requirement 12: Extension Lifecycle and State Management Architecture

**User Story:** As a user, I want the extension to maintain consistent state across browser sessions so that my scraps are always available.

#### Acceptance Criteria

1. WHEN the browser starts, THE Service Worker SHALL initialize and verify IndexedDB connection within 1 second
2. THE Extension SHALL persist all user preferences and settings in chrome.storage.local
3. WHEN the Service Worker is terminated by Chrome, THE Extension SHALL restore state from storage when reactivated within 500 milliseconds
4. THE Extension SHALL implement a state synchronization mechanism to keep Side Panel UI consistent with storage changes
5. WHEN the user closes and reopens the Side Panel, THE Extension SHALL restore the previous view state including scroll position and expanded cards

---

## Lower Priority Requirements (Future Enhancements)

### Requirement 13: Hybrid AI Architecture with Remote Model Support (MEDIUM PRIORITY)

**User Story:** As a power user, I want the option to send my scraps to a remote AI model for more advanced analysis so that I can get deeper insights beyond on-device capabilities.

#### Acceptance Criteria

1. THE Extension SHALL provide a settings toggle to enable hybrid mode that allows communication with remote AI services (UI may be stubbed for MVP demonstration)
2. WHEN hybrid mode is enabled, THE Extension SHALL display a "Deep Analysis" button on scrap cards that sends data to Gemini Developer API or Firebase AI Logic
3. THE Extension SHALL implement secure API key storage using chrome.storage.local with encryption
4. WHEN the user requests deep analysis, THE Extension SHALL send scrap context (transcription, summary, image description) to the remote model and display results within 10 seconds
5. THE Extension SHALL clearly indicate which processing occurred on-device versus cloud-based with visual badges (on-device: green "🔒 Private", cloud: blue "☁️ Enhanced")
6. THE Extension SHALL allow users to chat with the remote model about their scraps, maintaining conversation context across multiple queries
7. THE Extension SHALL cache remote analysis results locally to avoid redundant API calls
8. THE Extension architecture SHALL support hybrid mode integration with minimal code changes, even if full implementation is deferred post-MVP

### Requirement 14: Advanced Collection Management (LOW PRIORITY)

**User Story:** As a researcher, I want to manually organize scraps into custom collections so that I can group related thoughts across different sources.

#### Acceptance Criteria

1. THE Extension SHALL allow users to create named collections with custom colors and icons
2. WHEN viewing a scrap, THE Extension SHALL provide an "Add to Collection" button that displays a collection picker
3. THE Extension SHALL support adding a single scrap to multiple collections simultaneously
4. THE Side Panel SHALL provide a collections view that shows all custom collections with scrap counts
5. WHEN a collection is selected, THE Side Panel SHALL display only scraps belonging to that collection

### Requirement 15: Export and Sharing Architecture (LOW PRIORITY)

**User Story:** As a user, I want to export my scraps in various formats so that I can use them in other applications.

#### Acceptance Criteria

1. THE Extension SHALL provide export functionality for individual scraps in JSON, Markdown, and PDF formats
2. WHEN exporting a collection, THE Extension SHALL generate a single document containing all scraps with proper formatting and metadata
3. THE Extension SHALL support bulk export of all scraps with folder structure organized by source domain
4. THE Extension SHALL generate shareable links that encode scrap data in URL parameters for sharing with other Recall users
5. THE Extension SHALL implement import functionality to restore scraps from exported JSON files

### Requirement 16: Advanced Search with Filters (MEDIUM PRIORITY)

**User Story:** As a user with many scraps, I want advanced filtering options so that I can narrow down search results more precisely.

#### Acceptance Criteria

1. THE Extension SHALL provide filter options for date ranges (today, this week, this month, custom range)
2. THE Extension SHALL allow filtering by source domain with autocomplete suggestions
3. THE Extension SHALL support filtering by content type (has audio, has text note, has image analysis)
4. THE Extension SHALL allow filtering by processing status (cleaned, refined, drafted)
5. THE Extension SHALL support combining multiple filters with AND/OR logic

### Requirement 17: Annotation and Tagging System (LOW PRIORITY)

**User Story:** As a user, I want to add tags and additional notes to my scraps so that I can organize them with my own categorization system.

#### Acceptance Criteria

1. THE Extension SHALL allow users to add custom tags to scraps with autocomplete from existing tags
2. WHEN viewing a scrap, THE Extension SHALL provide an "Add Note" button for appending additional text annotations
3. THE Extension SHALL support tag-based filtering in search and list views
4. THE Extension SHALL display tag clouds showing most frequently used tags
5. THE Extension SHALL allow bulk tagging of multiple selected scraps

### Requirement 18: Keyboard Shortcuts and Accessibility (MEDIUM PRIORITY)

**User Story:** As a power user, I want comprehensive keyboard shortcuts so that I can use the extension without relying on mouse interactions.

#### Acceptance Criteria

1. THE Extension SHALL support customizable keyboard shortcuts for capture initiation (default: Alt+C)
2. THE Extension SHALL provide keyboard navigation for side panel (arrow keys, Enter to expand, Escape to close)
3. THE Extension SHALL support keyboard shortcuts for magic buttons (Ctrl+1 for Clean, Ctrl+2 for Refine, Ctrl+3 for Draft)
4. THE Extension SHALL implement ARIA labels and roles for all interactive elements to support screen readers
5. THE Extension SHALL provide high contrast mode for users with visual impairments

### Requirement 19: Analytics and Insights Dashboard (LOW PRIORITY)

**User Story:** As a user, I want to see statistics about my capture habits so that I can understand how I'm using the extension.

#### Acceptance Criteria

1. THE Extension SHALL track local statistics including total scraps, captures per day, most captured domains, and average processing time
2. THE Extension SHALL provide a dashboard view showing capture trends over time with charts
3. THE Extension SHALL identify most frequently used magic buttons and suggest workflow optimizations
4. THE Extension SHALL calculate total time saved compared to manual note-taking
5. THE Extension SHALL ensure all analytics are computed locally without any data transmission

### Requirement 20: Collaborative Features with Privacy (LOW PRIORITY)

**User Story:** As a team member, I want to share specific scraps with colleagues while maintaining privacy for my other captures.

#### Acceptance Criteria

1. THE Extension SHALL generate encrypted export packages for individual scraps that can be imported by other users
2. THE Extension SHALL support creating "public" scraps that generate shareable view-only links
3. THE Extension SHALL implement optional peer-to-peer sharing using WebRTC for direct browser-to-browser transfer
4. THE Extension SHALL allow users to create team collections that sync via user-controlled storage (not extension servers)
5. THE Extension SHALL clearly indicate which scraps are private versus shared with visual indicators
