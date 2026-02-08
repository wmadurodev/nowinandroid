# Offline-First Synchronization in Now in Android

This document explains the offline-first synchronization mechanism implemented in the Now in Android application. The process ensures that the app's data is kept up-to-date with a remote source while providing a seamless offline experience.

The synchronization process is initiated when the application starts.

## 1. Initialization (`NiaApplication`)

The entire process begins in the `onCreate()` method of the main `NiaApplication` class.

```kotlin
// app/src/main/kotlin/com/google/samples/apps/nowinandroid/NiaApplication.kt

override fun onCreate() {
    super.onCreate()
    // ...
    // Initialize Sync; the system responsible for keeping data in the app up to date.
    Sync.initialize(context = this)
    // ...
}
```

## 2. Enqueuing the Sync Work (`SyncInitializer`)

The `Sync.initialize()` method is defined in the `:sync:work` module. It uses Android's `WorkManager` to schedule the synchronization task.

```kotlin
// sync/work/src/main/kotlin/com/google/samples/apps/nowinandroid/sync/initializers/SyncInitializer.kt

object Sync {
    fun initialize(context: Context) {
        WorkManager.getInstance(context).apply {
            // Run sync on app startup and ensure only one sync worker runs at any time
            enqueueUniqueWork(
                SYNC_WORK_NAME,
                ExistingWorkPolicy.KEEP,
                SyncWorker.startUpSyncWork(),
            )
        }
    }
}
```

Key points:
* **`WorkManager`**: A standard Android library for deferrable and guaranteed background work.
* **`enqueueUniqueWork`**: This ensures that only one synchronization task with the name `SYNC_WORK_NAME` is active at any given time, preventing redundant sync operations.
* **`ExistingWorkPolicy.KEEP`**: If a sync work request is already pending, the new request is ignored.
* **`SyncWorker.startUpSyncWork()`**: This static method creates the actual `WorkRequest` to be executed.

## 3. Creating the Work Request (`SyncWorker`)

The `startUpSyncWork()` method configures the background task.

```kotlin
// sync/work/src/main/kotlin/com/google/samples/apps/nowinandroid/sync/workers/SyncWorker.kt

companion object {
    fun startUpSyncWork() = OneTimeWorkRequestBuilder<DelegatingWorker>()
        .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
        .setConstraints(SyncConstraints)
        .setInputData(SyncWorker::class.delegatedData())
        .build()
}
```

Instead of directly scheduling `SyncWorker`, it schedules a `DelegatingWorker`. This is a crucial architectural choice that allows the `:sync:work` library module to use Hilt for dependency injection without forcing the main `:app` module to manage custom `WorkManager` configurations.

The `setInputData(SyncWorker::class.delegatedData())` call passes the fully qualified name of `SyncWorker` to the `DelegatingWorker`, telling it which worker to delegate the actual work to.

## 4. The `DelegatingWorker`

The `DelegatingWorker` acts as a proxy. It receives the work request from `WorkManager`, and its job is to create and run the *actual* Hilt-injected worker.

```kotlin
// sync/work/src/main/kotlin/com/google/samples/apps/nowinandroid/sync/workers/DelegatingWorker.kt

class DelegatingWorker(
    appContext: Context,
    workerParams: WorkerParameters,
) : CoroutineWorker(appContext, workerParams) {

    private val workerClassName =
        workerParams.inputData.getString(WORKER_CLASS_NAME) ?: ""

    private val delegateWorker =
        EntryPointAccessors.fromApplication<HiltWorkerFactoryEntryPoint>(appContext)
            .hiltWorkerFactory()
            .createWorker(appContext, workerClassName, workerParams)
            as? CoroutineWorker
            ?: throw IllegalArgumentException("Unable to find appropriate worker")

    override suspend fun doWork(): Result =
        delegateWorker.doWork()
}
```

It retrieves the target worker's class name, uses a Hilt `EntryPoint` to get a `HiltWorkerFactory`, and then creates an instance of the `SyncWorker` with all its dependencies correctly injected. Finally, it calls `doWork()` on this new instance.

## 5. The Core Synchronization Logic (`SyncWorker.doWork`)

This is where the main synchronization happens. `SyncWorker` injects the repositories (`TopicsRepository`, `NewsRepository`) that it needs to sync.

```kotlin
// sync/work/src/main/kotlin/com/google/samples/apps/nowinandroid/sync/workers/SyncWorker.kt

override suspend fun doWork(): Result = withContext(ioDispatcher) {
    // ...
    // First sync the repositories in parallel
    val syncedSuccessfully = awaitAll(
        async { topicRepository.sync() },
        async { newsRepository.sync() },
    ).all { it }

    if (syncedSuccessfully) {
        // ...
        Result.success()
    } else {
        Result.retry()
    }
}
```

The `doWork` method calls the `sync()` method on each repository concurrently using `async` and `awaitAll`. The `SyncWorker` itself implements the `Synchronizer` interface, which it passes to the repositories.

## 6. Repository-Level Synchronization (`OfflineFirstTopicsRepository`)

Each "offline-first" repository is responsible for its own data synchronization logic. The `syncWith()` method calls a generic `changeListSync` helper function.

```kotlin
// core/data/src/main/kotlin/com/google/samples/apps/nowinandroid/core/data/repository/OfflineFirstTopicsRepository.kt

override suspend fun syncWith(synchronizer: Synchronizer): Boolean =
    synchronizer.changeListSync(
        versionReader = ChangeListVersions::topicVersion,
        changeListFetcher = { currentVersion ->
            network.getTopicChangeList(after = currentVersion)
        },
        versionUpdater = { latestVersion ->
            copy(topicVersion = latestVersion)
        },
        modelDeleter = topicDao::deleteTopics,
        modelUpdater = { changedIds ->
            val networkTopics = network.getTopics(ids = changedIds)
            topicDao.upsertTopics(
                entities = networkTopics.map(NetworkTopic::asEntity),
            )
        },
    )
```

The `changeListSync` function orchestrates the sync process for a specific data type (e.g., Topics) by following these steps:

1.  **`versionReader`**: Reads the current local version of the data from `NiaPreferencesDataSource` (a `DataStore`). This version represents the last time a successful sync occurred.
2.  **`changeListFetcher`**: Calls the `NiaNetworkDataSource` (Retrofit) with the current version. The backend API returns a "change list," which contains only the IDs of topics that have been created, modified, or deleted since that version.
3.  **`modelDeleter`**: The IDs of deleted items from the change list are passed to this function, which deletes the corresponding entities from the local Room database (`topicDao::deleteTopics`).
4.  **`modelUpdater`**: The IDs of new or updated items are passed to this function. It makes another network request to fetch the full data for these specific topics and then "upserts" (inserts or updates) them into the local Room database.
5.  **`versionUpdater`**: If all steps are successful, this function is called to update the local version in `DataStore` to the new version received from the server's change list. This ensures the next sync will only fetch changes from this point forward.

This entire process provides a robust and efficient offline-first strategy, minimizing data transfer by only fetching what has changed and ensuring the local database is a reliable source of truth for the UI, whether the device is online or offline.
