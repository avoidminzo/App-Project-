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
