# Apps Overview

The Now in Android project contains two distinct Android applications, each serving a specific purpose within the ecosystem.

## 1. Now in Android (Main App)
**Module:** `:app`

### Functionality
The main consumer-facing application. It provides a platform for users to stay up-to-date with the latest news in the Android development world.
*   **Personalized Feed:** Users receive a "For You" feed based on their followed topics.
*   **Topic Following:** Users can browse and follow specific Android development topics (e.g., UI, Performance, Tools).
*   **Bookmarks:** Users can save news resources for later reading.
*   **Search:** Deep search capabilities for news items and topics.
*   **Notifications:** Integration with system notifications for new content.
*   **Settings:** Support for theme switching (Dark/Light) and dynamic color (Material You).

### Architecture
The app follows the **Official Android Architecture** guidance:
*   **Layering:** Divided into UI, Domain, and Data layers.
*   **Reactive UI:** Built entirely with **Jetpack Compose**.
*   **State Management:** Uses **Unidirectional Data Flow (UDF)** with ViewModels and Kotlin Flows.
*   **Offline-First:** Uses **Room** and **DataStore** to ensure the app works without an internet connection, with **WorkManager** handling background synchronization.
*   **Dependency Injection:** Powered by **Hilt**.

---

## 2. NiA Catalog
**Module:** `:app-nia-catalog`

### Functionality
A standalone developer-focused tool used to showcase and interact with the **NiA Design System** components in isolation.
*   **Component Gallery:** Displays all custom UI components defined in `:core:designsystem` (e.g., buttons, chips, cards, navigation bars).
*   **Theme Testing:** Allows developers to quickly verify how components look across different themes and color configurations.
*   **Interactive Demos:** Provides a playground to test component behavior (states, clicks, etc.) without running the full app.

### Architecture
*   **Lightweight:** A single-activity application (`NiaCatalogActivity`) designed for speed and simplicity.
*   **Compose-only:** Built directly with Jetpack Compose, bypassing the complex data and domain layers of the main app.
*   **Design System Consumer:** Its primary dependency is `:core:designsystem`, making it the "source of truth" for UI component standards within the project.
