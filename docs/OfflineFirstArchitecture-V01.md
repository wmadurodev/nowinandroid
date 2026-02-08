# Offline-First Architecture in Now in Android

This document details the "Offline-First" implementation approach in the Now in Android application, exploring its core concepts, synchronization mechanisms, and the architectural correlation between various components. The offline-first strategy ensures that the application remains functional and provides a seamless user experience even without a constant internet connection, while efficiently keeping data synchronized with remote sources.

## Core Principles of Offline-First

The Now in Android app follows an offline-first paradigm, where the local data storage (Room database and Proto DataStore) is considered the single source of truth. This means:
*   **Reliable User Experience:** Users can browse content and interact with the app without an internet connection.
*   **Performance:** Data is served quickly from local storage, leading to a snappier UI.
*   **Resilience:** The app is less susceptible to network fluctuations or outages.
*   **Background Synchronization:** Data updates from remote sources are managed in the background and intelligently reconciled with local data.

## Architectural Layers and Offline-First

The offline-first approach is deeply integrated into the app's three architectural layers: Data, Domain, and UI.

### 1. Data Layer

The Data Layer is the cornerstone of the offline-first implementation. It is responsible for providing application data and business logic, abstracting whether the data comes from a network source or local storage. Repositories in this layer are designed to be offline-first.

*   **Source of Truth:** Local storage (Room for relational data, Proto DataStore for user preferences) is the primary source of truth.
*   **Repositories:** Each repository acts as the public API for data access, typically offering data streams (Kotlin Flows) for reading and suspend functions for writing. These streams always emit data from local storage, reacting to any changes.
*   **Data Sources:** Repositories depend on one or more data sources, which can be local (e.g., `TopicsDao` for Room, `NiaPreferencesDataSource` for DataStore) or remote (`NiaNetworkDataSource` for network API calls via Retrofit).

#### Data Synchronization within the Data Layer

Repositories handle the reconciliation of local data with remote sources. When data is fetched from a remote source, it's immediately written to local storage. This triggers the data streams, updating any listening clients in higher layers.

The synchronization process employs an exponential backoff strategy, managed by `WorkManager` via the `SyncWorker`, an implementation of the `Synchronizer` interface.

**Example: `OfflineFirstTopicsRepository.syncWith`**
This method demonstrates how a repository synchronizes its data:
1.  **`versionReader`**: Reads the current local version of the data from `NiaPreferencesDataSource`.
2.  **`changeListFetcher`**: Fetches a "change list" from the network (via `NiaNetworkDataSource`) containing IDs of items (e.g., topics) that have been created, modified, or deleted since the last sync version.
3.  **`modelDeleter`**: Deletes items from the local Room database based on the change list.
4.  **`modelUpdater`**: Fetches full data for new/updated items from the network and upserts them into the local Room database.
5.  **`versionUpdater`**: Updates the local version in `DataStore` to reflect the latest synchronized state.

This `changeListSync` mechanism minimizes data transfer and ensures the local database is consistently up-to-date.

### 2. Domain Layer

The Domain Layer contains use cases that encapsulate business logic. In an offline-first architecture, these use cases typically combine and transform data streams provided by the repositories in the Data Layer. They operate solely on the data exposed by repositories, abstracting away the underlying data sources and the complexities of synchronization.

**Example: `GetUserNewsResourcesUseCase`**
This use case combines `NewsResource` streams from `NewsRepository` with `UserData` streams from `UserDataRepository` to create a stream of `UserNewsResource`s, which include the user's bookmarking status. This ensures that the UI always displays the most current data available locally, which is then updated reactively as synchronization occurs.

### 3. UI Layer

The UI Layer comprises Jetpack Compose UI elements and Android ViewModels. ViewModels consume the data streams provided by use cases and repositories, transforming them into observable UI states. Because the data streams are continuously updated by the Data Layer's synchronization, the UI automatically reflects the latest available data, whether it's from local storage or newly synced from the network.

*   **Reactive UI:** Built entirely with Jetpack Compose, the UI reacts to changes in the UI state, which in turn reflects changes in the underlying data streams.
*   **Unidirectional Data Flow (UDF):** Ensures that data flows predictably from the Data Layer up to the UI, and user interactions flow down to the ViewModel for processing.

## Key Components in Offline-First Implementation

### WorkManager

`WorkManager` is crucial for scheduling and executing background synchronization tasks, especially for long-running operations or when specific constraints (like network availability) need to be met.

*   **Initialization:** On app startup, `Sync.initialize()` enqueues a `SyncWorker` using `WorkManager.enqueueUniqueWork` to ensure only one sync task runs at a time.
*   **`SyncWorker`:** This `CoroutineWorker` orchestrates the synchronization of all necessary repositories (e.g., `topicRepository.sync()`, `newsRepository.sync()`). It uses `async` and `awaitAll` for parallel synchronization.
*   **`DelegatingWorker`:** To facilitate Hilt dependency injection into `SyncWorker` without complex `WorkManager` configurations in the main app module, a `DelegatingWorker` acts as a proxy. It receives the work request, creates an instance of the Hilt-injected `SyncWorker`, and delegates the `doWork()` call.

### Local Persistence (Room and DataStore)

*   **Room Database (`core:database`):** Used for storing structured, relational data (e.g., `NewsResource`, `Topic` entities). DAOs (Data Access Objects) provide the interface for reading and writing this data. Changes in Room tables trigger updates in corresponding Kotlin Flows, which are consumed by repositories.
*   **Proto DataStore (`core:datastore`):** Used for storing unstructured data, such as user preferences (e.g., followed topics) and synchronization versions. `NiaPreferencesDataSource` manages user preferences, and `UserPreferencesSerializer` handles the serialization/deserialization of the Protobuf model.

### Network Communication (`core:network`)

*   **Retrofit:** Used to define the API interface and make network requests to the remote backend.
*   **`NiaNetworkDataSource`:** An abstraction over the network API, providing methods to fetch data like `getTopicChangeList` and `getTopics`.

### Notifications

Notifications are an integral part of the synchronization process, especially for providing user feedback.

*   **Foreground Sync Notifications:** When `SyncWorker` runs as an expedited job (e.g., during app startup), it requires a foreground service notification to inform the user that background work is in progress.
    *   `SyncWorkHelpers.kt` (in `:sync:work`) contains `syncForegroundInfo()` and `syncWorkNotification()` to construct this notification.
    *   `SyncWorker.kt` calls `appContext.syncForegroundInfo()` to provide the `ForegroundInfo` object to `WorkManager`.
*   **News Notifications:** After a successful synchronization, if new news items matching the user's followed topics are found, the app can post user-facing news notifications.
    *   `OfflineFirstNewsRepository` (in `:core:data`) is responsible for triggering these.
    *   It uses the `Notifier` interface, implemented by `SystemTrayNotifier` (in `:core:notifications`), to build and post specific news alerts, including deep links.

## Correlation Between Elements

The offline-first architecture in Now in Android is a tightly integrated system where each component plays a specific role to achieve robust data synchronization and a smooth user experience:

*   **`NiaApplication` -> `Sync.initialize()` -> `WorkManager`:** The app's lifecycle initiates the synchronization process.
*   **`WorkManager` -> `DelegatingWorker` -> `SyncWorker`:** `WorkManager` delegates to a proxy worker for Hilt injection, which then executes the core sync logic.
*   **`SyncWorker` -> Repositories (`OfflineFirstTopicsRepository`, `OfflineFirstNewsRepository`):** The worker coordinates synchronization across different data types.
*   **Repositories -> Data Sources (`Room`, `DataStore`, `NiaNetworkDataSource`):** Repositories abstract data access, deciding whether to fetch from local cache or network, and persisting network data locally.
*   **Data Sources -> Kotlin Flows:** Changes in local data sources (Room, DataStore) automatically emit updates via Kotlin Flows.
*   **Repositories/Use Cases -> ViewModels -> UI:** Data flows reactively from the data layer through use cases to ViewModels, which transform it into UI state, ensuring the UI always reflects the latest local (and synced) information.
*   **`SyncWorker`/`OfflineFirstNewsRepository` -> Notifications:** Specific events during synchronization trigger user-facing notifications for foreground work or new content.

This interconnected system ensures data consistency, optimal performance, and a resilient application that prioritizes the user's experience by providing immediate access to data, regardless of network availability.