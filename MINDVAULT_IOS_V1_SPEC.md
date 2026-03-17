# MindVault Native iOS v1 Implementation Spec

This document provides a concrete v1 blueprint for building MindVault as a native iOS app using SwiftUI, Core Data/SQLite, and on-device ML.

## 1) Core Data Schema (Concrete)

### Entities

#### `JournalEntry`
- `id: UUID` (required, unique)
- `createdAt: Date` (required, indexed)
- `updatedAt: Date` (required)
- `title: String?` (optional, max 120 chars)
- `body: String` (required)
- `sentimentLabel: String` (required; enum-like: `very_negative`, `negative`, `neutral`, `positive`, `very_positive`)
- `sentimentScore: Double` (required; range -1.0 to +1.0)
- `keywordCSV: String?` (optional; comma-separated top keywords for fast UI)
- `embeddingVectorRef: String?` (optional; reference key for local embedding blob/file)
- `isPinned: Bool` (required, default `false`)
- `wordCount: Int32` (required)
- `deletedAt: Date?` (optional soft delete)

#### `MoodSnapshot`
- `id: UUID` (required, unique)
- `bucketDate: Date` (required, indexed; normalized to local day start)
- `averageSentiment: Double` (required)
- `entryCount: Int32` (required)
- `dominantMood: String` (required; `low`, `mixed`, `good`)

#### `MemoryLink`
- `id: UUID` (required, unique)
- `entryAId: UUID` (required, indexed)
- `entryBId: UUID` (required, indexed)
- `similarity: Double` (required; range 0.0 to 1.0)
- `reason: String?` (optional; e.g., shared keywords/theme)
- `createdAt: Date` (required)

#### `PromptSuggestion`
- `id: UUID` (required, unique)
- `date: Date` (required, indexed)
- `promptText: String` (required)
- `source: String` (required; `mood_rule`, `keyword_rule`, `model`)
- `used: Bool` (required, default `false`)

### Relationships
- `JournalEntry` 1-to-many `MemoryLink` via `entryAId` / `entryBId` (modeled as fetched associations in code for flexibility).
- `MoodSnapshot` is denormalized analytics and does not require hard foreign keys.

### Indexing strategy
- `JournalEntry.createdAt`
- `JournalEntry.sentimentScore`
- `MoodSnapshot.bucketDate`
- `MemoryLink.entryAId`
- `MemoryLink.entryBId`
- Composite uniqueness in app logic for `(entryAId, entryBId)` with ordered UUIDs to avoid duplicate inverse links.

### Derived fields and update rules
- `wordCount` computed on save.
- `sentimentLabel` derived from thresholds of `sentimentScore`.
- `keywordCSV` stores top 3–7 keywords from NLP pipeline.
- `MoodSnapshot` recomputed nightly and after every new entry.

## 2) SwiftUI Screen-by-Screen Wireframe Spec

## App Shell
- `TabView`
  1. Journal
  2. Insights
  3. Settings

## 2.1 Lock Screen (`LockGateView`)
- Purpose: protect app access with Face ID / Touch ID.
- Components:
  - App logo + "MindVault"
  - Privacy copy: "Your journal stays on this device"
  - Primary CTA: `Unlock with Face ID`
  - Fallback CTA: `Try Again`
- States:
  - loading auth
  - success (navigates to app)
  - failure with inline error

## 2.2 Journal List (`JournalListView`)
- Top bar:
  - title: `Journal`
  - plus button to create entry
- Body:
  - search bar
  - filter chips: `All`, `Pinned`, `This Week`, `Low Mood`
  - list cards with:
    - title (or first line fallback)
    - date/time
    - mood pill color + label
    - first 100 chars preview
- Empty state:
  - copy + CTA "Write your first entry"

## 2.3 Entry Editor (`JournalEditorView`)
- Fields:
  - title text field
  - main multiline text editor
- Footer quick actions:
  - save
  - pin/unpin
  - delete
- Post-save behavior:
  - return to detail view
  - asynchronous analysis status badge (`Analyzing…`, `Analyzed`)

## 2.4 Entry Detail (`JournalDetailView`)
- Sections:
  1. Entry body
  2. Mood Analysis card
     - sentiment score meter
     - mood label + keyword chips
  3. Memory Links card
     - up to 3 similar entries with similarity percent
     - tap opens linked entry
  4. Prompt card
     - "Based on today, you might explore…"

## 2.5 Insights Dashboard (`InsightsView`)
- Segmented control: `7D`, `30D`, `90D`
- Charts:
  - Line chart: daily sentiment trend
  - Bar chart: entries per day
- Secondary cards:
  - top keywords this period
  - streak (days written)
  - average mood vs previous period

## 2.6 Settings (`SettingsView`)
- Sections:
  - Security
    - biometric lock toggle
    - auto-lock timeout
  - Privacy
    - "No cloud processing" explainer
    - export local archive
  - Appearance
    - adaptive theme toggle
    - manual theme override
  - About
    - app version

## 3) Starter Code Structure

```text
MindVault/
  App/
    MindVaultApp.swift
    RootCoordinator.swift
  Core/
    Extensions/
    DesignSystem/
      Colors.swift
      Typography.swift
      MoodTheme.swift
    Utils/
      DateBucket.swift
      Logger.swift
  Data/
    Persistence/
      PersistenceController.swift
      Entities/
        JournalEntry+CoreDataClass.swift
        MoodSnapshot+CoreDataClass.swift
        MemoryLink+CoreDataClass.swift
        PromptSuggestion+CoreDataClass.swift
      Repositories/
        JournalRepository.swift
        InsightsRepository.swift
        LinkRepository.swift
    Migration/
      ModelVersioning.md
  Services/
    ML/
      NLPService.swift
      SentimentAnalyzer.swift
      KeywordExtractor.swift
      EmbeddingService.swift
      MemoryLinker.swift
    Security/
      BiometricAuthService.swift
      KeychainService.swift
      EncryptionPolicy.swift
    Prompting/
      PromptSuggestionService.swift
  Features/
    Lock/
      LockGateView.swift
      LockGateViewModel.swift
    Journal/
      List/
      Detail/
      Editor/
    Insights/
      InsightsView.swift
      InsightsViewModel.swift
      Components/
    Settings/
      SettingsView.swift
      SettingsViewModel.swift
  Background/
    BackgroundTaskRegistrar.swift
    SnapshotRecomputeTask.swift
  Resources/
    Localizable.strings
    MLModels/
      sentiment.mlmodelc
      embeddings.mlmodelc
  Tests/
    Unit/
      NLPServiceTests.swift
      MemoryLinkerTests.swift
      JournalRepositoryTests.swift
    Snapshot/
      InsightsViewSnapshotTests.swift
```

## 4) Service contracts (minimal)

```swift
protocol NLPService {
    func analyze(text: String) async throws -> NLPAnalysisResult
}

struct NLPAnalysisResult {
    let sentimentScore: Double
    let sentimentLabel: SentimentLabel
    let keywords: [String]
    let embedding: [Float]?
}

protocol MemoryLinking {
    func links(for entryID: UUID, topK: Int) async throws -> [MemoryLinkResult]
}
```

## 5) Operational flow (save pipeline)
1. User taps Save in `JournalEditorView`.
2. `JournalRepository` persists raw entry immediately.
3. Background task invokes `NLPService.analyze(text:)`.
4. Persist sentiment/keywords/embedding reference.
5. `MemoryLinker` recomputes nearest neighbors for new entry.
6. `InsightsRepository` updates daily `MoodSnapshot`.
7. UI observes updates via `@FetchRequest` or view model publisher.

## 6) v1 non-functional targets
- Entry save latency: <150 ms for raw text persistence.
- NLP post-processing: <1.5 s for average entry (200–400 words).
- App lock authentication: <1 s in normal conditions.
- Cold start memory footprint: target <200 MB.
- Fully offline behavior for all core features.

## 7) Sprint-ready backlog (first 20 tickets)

1. Set up SwiftUI app shell with `TabView` and root coordinator.
2. Implement Core Data stack with in-memory preview configuration.
3. Create `JournalEntry` entity + repository CRUD methods.
4. Create `MoodSnapshot`, `MemoryLink`, and `PromptSuggestion` entities.
5. Add migration baseline and model versioning doc.
6. Implement biometric gate (`LocalAuthentication`) and lock-state coordinator.
7. Add Keychain wrapper for app secret material.
8. Enforce file protection policy on app support/documents directories.
9. Build Journal list screen with search and filters.
10. Build Journal editor and detail screens.
11. Implement NLP service adapter (stub -> Core ML backend).
12. Add sentiment threshold mapping utility and tests.
13. Add keyword extraction utility and tests.
14. Add embedding generation + persistence reference strategy.
15. Implement memory-link similarity search for top-K matches.
16. Build insights charts (7D/30D/90D) and aggregation query layer.
17. Implement adaptive mood theme mapping and settings overrides.
18. Add daily background task for snapshot recomputation.
19. Add export flow for local encrypted archive.
20. Add privacy copy/onboarding explainer and QA pass for offline mode.

## 8) Acceptance criteria (v1)

- User can create, edit, pin, and delete entries without network access.
- Entry save is immediate; analysis updates appear asynchronously.
- App can be protected by Face ID / Touch ID, with retry flows.
- Insights show trend charts populated from local data only.
- Memory links show at least one relevant prior entry when similarity exceeds threshold.
- Adaptive theme updates from latest sentiment label when enabled.
- Export generates a local archive without transmitting data externally.

## 9) Test plan

### Unit tests
- Sentiment label threshold mapping edge cases (`-1.0`, `-0.2`, `0.0`, `0.2`, `1.0`).
- Keyword extractor returns deterministic top-N terms.
- Memory link cosine similarity ranking correctness.
- Repository CRUD operations and soft-delete behavior.

### Integration tests
- Save-entry pipeline from editor -> persistence -> analysis -> UI refresh.
- Background snapshot recompute updates `MoodSnapshot` rows.
- Biometric lock state transitions on success/failure.

### UI tests
- Journal list empty state and create-entry flow.
- Entry detail displays mood card + memory links.
- Settings toggles persist across relaunch.

## 10) Privacy and security defaults

- Default mode: strictly offline processing (no remote inference endpoints).
- Crash logs should redact entry body content.
- App analytics disabled by default in v1.
- Clipboard access avoided for entry body unless user explicitly copies text.
- Optional iCloud backup setting surfaced with clear user consent messaging.
