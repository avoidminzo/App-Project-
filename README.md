# CineFlow — Local-First Native Android Streaming Client

**CineFlow** is an open, decentralized, native Android movie discovery and streaming client built with modern Android architecture. It runs entirely on the Android device without requiring a proprietary backend, fetching repository manifests and provider catalogs directly over HTTPS.

---

## 🏗 Architecture & Design

CineFlow follows **Clean Architecture** with **MVVM** and unidirectional data flow.

```
┌────────────────────────────────────────────────────────┐
│                   Jetpack Compose UI                   │
│  (Home, Search, Details, Player, Library, Providers)   │
└───────────────────────────▲────────────────────────────┘
                            │ State / Events
┌───────────────────────────┴────────────────────────────┐
│                    ViewModels (Hilt)                   │
└───────────────────────────▲────────────────────────────┘
                            │ Domain Calls
┌───────────────────────────┴────────────────────────────┐
│                      Domain Layer                      │
│   • MediaProvider         • ProviderRegistry           │
│   • RepositoryManager     • PlaybackState              │
└───────────────────────────▲────────────────────────────┘
                            │ Data Sources
        ┌───────────────────┴───────────────────┐
        ▼                                       ▼
┌──────────────────────────────┐     ┌─────────────────────────┐
│     Local Storage Layer      │     │      Network Layer      │
│ • Room (6 Tables + DAOs)     │     │ • Ktor Client (CIO)     │
│ • DataStore (Preferences)    │     │ • HTTPS Security Engine │
└──────────────────────────────┘     └─────────────────────────┘
        ▼                                       ▼
┌──────────────────────────────┐     ┌─────────────────────────┐
│        Playback Engine       │     │   Repository Manager    │
│ • AndroidX Media3 ExoPlayer  │     │ • CineFlow Manifest     │
│ • HLS, DASH, MP4, Subtitles  │     │ • Cloudstream Adapter   │
└──────────────────────────────┘     └─────────────────────────┘
```

---

## 🌟 Core Subsystems

### 1. Repository Manager
- **Decentralized Repository Delivery**: Adds and synchronizes repositories using standard HTTPS Git endpoints and raw JSON manifests.
- **Documented Manifest Formats**:
  - **CineFlow Native Format (`cineflow-repo.json`)**: Modern declarative manifest declaring provider metadata, endpoints, supported media types, and versions.
  - **Cloudstream Compatibility Adapter**: Parses official Cloudstream repository manifests (`repo.json` / `plugins.json`). Providers may declare `catalogUrl`, `catalogUrls`, or an inline `catalog` array to publish real media over HTTPS. Cloudstream `.cs3` bytecode is never executed.
- **Resilience & Caching**: Manifests and provider declarations are cached in Room (`repositories` and `providers` tables), allowing fully functional offline browsing.
- **Background Sync**: Automated background updates powered by `WorkManager` (`RepositorySyncWorker`).

#### Declarative provider catalogs

A provider can publish its full media catalog without shipping executable bytecode. Add any of these to a CineFlow provider object or a CloudStream `plugins.json` entry:

```json
{
  "name": "My Provider",
  "internalName": "my.provider",
  "catalogUrl": "https://example.com/catalog.json",
  "catalogUrls": [
    "https://example.com/more.json"
  ],
  "catalog": [
    {
      "id": "my-provider-1",
      "title": "Example Movie",
      "year": 2024,
      "rating": 8.2,
      "mediaType": "MOVIE",
      "overview": "A real catalog item."
    }
  ]
}
```

Catalog endpoints may return either a JSON array of media items or an object with a `movies`, `items`, or `catalog` array. All URLs must be HTTPS.

### 2. Provider System & Security Sandbox
- **Common Kotlin `MediaProvider` Interface**:
  - `getPopular(page: Int): Result<List<MediaItem>>`
  - `search(query: String): Result<List<MediaItem>>`
  - `getDetails(id: String): Result<MediaDetails>`
  - `getSources(id: String): Result<List<VideoSource>>`
  - `checkHealth(): ProviderStatus`
- **Security & Sandboxing**:
  - **No Arbitrary Bytecode Execution**: External binary plugins (`.dex`, `.cs3`, `.apk`) are recognized by `SecurityValidator` and flagged as untrusted/sandboxed. CineFlow strictly forbids dynamic execution of downloaded bytecode in the application process.
  - **Declarative REST Provider Adapter**: Allows external community providers to connect via safe, structured JSON/REST endpoints without binary dependencies.
- **Provider Registry**:
  - Multi-provider aggregation: parallel search and popular item queries with coroutine concurrency.
  - Provider health monitoring: `HEALTHY`, `DEGRADED`, `OFFLINE`, and `UNVERIFIED`.
  - Individual provider toggle controls in UI.

### 3. Built-In Streaming Provider (`CineFlowOpenMediaProvider`)
Provides verified, high-definition public domain and open-source cinema with live, working streams:
- **Tears of Steel** (Blender Foundation, 4K/1080p DASH & MP4 with WebVTT subtitles)
- **Big Buck Bunny** (Blender Institute, Adaptive Multi-Bitrate HLS & 1080p MP4)
- **Sintel** (Blender Foundation, Akamai Multi-Bitrate HLS & multi-language subtitles: English, German, Spanish, French)
- **Cosmos Laundromat** (Blender Gooseberry, 1080p MP4)
- **Elephants Dream** (Blender Foundation, 1080p MP4)
- **Night of the Living Dead** (George A. Romero, 1968 Public Domain Master, Internet Archive)

### 4. Playback Engine (AndroidX Media3 ExoPlayer)
- **Streaming Formats**:
  - **HLS** (`.m3u8` playlists, adaptive bitrates, multi-audio)
  - **DASH** (`.mpd` manifests, multi-period, adaptive streams)
  - **MP4** (Direct progressive video streams)
- **Subtitles & Closed Captions**:
  - External WebVTT (`.vtt`) and SubRip (`.srt`) track injection
  - In-stream CEA-608 / CEA-708 captions
  - Dynamic subtitle track picker dialog
- **Audio Tracks**:
  - Multi-language audio track selection via Media3 `TrackSelectionParameters`
- **Playback Controls & Resume**:
  - Sub-second resume position tracking synced to Room `playback_progress` table
  - Double-tap / 10-second seek forward & rewind
  - Aspect ratio switching: Fit, Fill, Zoom
  - Playback speed selection (0.5x, 0.75x, 1.0x, 1.25x, 1.5x, 2.0x)
  - Immersive fullscreen controls overlay with auto-fade timeout

### 5. Local Storage (Room & DataStore)
- **Room Database (`cineflow.db`)**:
  - `media_items`: Cached media catalog items for quick offline search
  - `favorites`: User-bookmarked movies with reactive Flow observers
  - `watch_history`: Timestamps, last watched episode/movie
  - `playback_progress`: Current position, total duration, completion status
  - `repositories`: Remote repository URLs, sync timestamps, status
  - `providers`: Provider metadata, health status, and enable/disable state
- **Preferences DataStore**:
  - Theme mode (AMOLED Black / Dark)
  - Stream quality preference (1080p, 720p, Auto)
  - Subtitle default language and toggle
  - Repository auto-synchronization settings

### 6. Premium Cinematic Dark UI (Jetpack Compose)
- **Home Screen**: Featured hero banner with gradient overlay, "Continue Watching" progress row, "Trending & Popular", "Open Source Cinema", and "Public Domain Classics".
- **Search Screen**: Real-time 300ms debounced search across active providers, category filter chips, and responsive adaptive grid.
- **Movie Details Screen**: Backdrop header, metadata badges, synopsis, stream source selector (showing format, quality, subtitle count), cast row, and favorite toggle.
- **Player Screen**: Immersive fullscreen video playback, scrubber bar, custom overlay controls, subtitle & audio selector dialogs.
- **Library Screen**: Dedicated tabs for Favorites and Watch History, with progress bars and clear history action.
- **Provider Manager**: View registered providers, inspect sandbox and health status, toggle providers on/off.
- **Repository Settings**: Add HTTPS repositories via URL, trigger repository sync, inspect sync timestamps, delete repositories.

---

## 🛠 Tech Stack & Dependencies

| Category | Component / Library | Version | Purpose |
|---|---|---|---|
| **Language** | Kotlin | 1.9.23 | Modern, null-safe language |
| **UI** | Jetpack Compose + Material 3 | BOM 2024.04.01 | Declarative reactive UI |
| **Media Player** | AndroidX Media3 ExoPlayer | 1.3.1 | Core streaming engine (HLS/DASH/MP4/Subs) |
| **Network** | Ktor Client | 2.3.10 | High-performance asynchronous HTTP client |
| **Database** | AndroidX Room | 2.6.1 | Local-first SQLite persistence & reactive flows |
| **Preferences** | AndroidX DataStore | 1.0.0 | Asynchronous key-value settings storage |
| **DI** | Google Hilt | 2.51.1 | Dependency injection |
| **Images** | Coil Compose | 2.6.0 | Fast, coroutine-based asynchronous image loading |
| **Background** | AndroidX WorkManager | 2.9.0 | Periodic repository update checks |
| **Serialization**| Kotlinx Serialization | 1.6.3 | JSON manifest & provider schema parsing |
| **Testing** | JUnit 4, MockK, Coroutines Test | 4.13.2 / 1.13.10 | Unit testing & repository validation |

---

## 🔒 Security Architecture

1. **Strict HTTPS Validation**:
   - `SecurityValidator.validateHttpsUrl()` rejects plain HTTP, `file://`, `content://`, `javascript:`, or malformed URLs.
   - `android:usesCleartextTraffic="false"` enforced in `AndroidManifest.xml`.
2. **Bytecode Isolation & Sandbox**:
   - Executable binary extensions (`.dex`, `.cs3`, `.apk`, `.so`) are detected and blocked from execution in the main application process.
3. **Integrity Checksums**:
   - Manifest parser supports SHA-256 integrity verification (`verifySha256`).
4. **Credential & Token Protection**:
   - Ktor network logger sanitizes authorization headers and access tokens before outputting to Android logs.
   - CineFlow does not bypass DRM systems or private authentication endpoints.

---

## 🚀 Building & Testing

### Prerequisites
- JDK 17 (e.g. Eclipse Temurin 17)
- Android SDK Platform 34 & Build Tools 34.0.0

### Command-Line Build
```powershell
# Set environment
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-17.0.21.10002-hotspot"
$env:ANDROID_HOME = "C:\Android\sdk"

# Run all unit tests
.\gradlew.bat test

# Assemble Debug APK
.\gradlew.bat assembleDebug
```
