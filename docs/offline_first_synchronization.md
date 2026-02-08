# Offline-First Synchronization in Now in Android

This document details the implementation of offline-first synchronization within the Now in Android application. The offline-first approach ensures that the application remains functional and responsive even without an active internet connection, providing a seamless user experience by prioritizing local data storage while intelligently synchronizing with remote sources when connectivity is available.

## Core Concepts

The Now in Android app employs a robust offline-first strategy, built upon several key Android architectural components:

*   **WorkManager:** For scheduling and executing background synchronization tasks reliably.
*   **Repositories:** Act as single sources of truth, abstracting data origins (local or remote) from the rest of the application.
*   **Room Persistence Library:** For local, on-device data storage using an SQLite database.
*   **Proto DataStore:** For persisting user preferences and metadata, including synchronization versions.
*   **Retrofit:** For handling network requests and communicating with the remote API.
*   **Kotlin Flows:** For reactive data streams, enabling the UI to automatically update in response to data changes.

## Architecture Overview

The synchronization process generally follows this flow:

1.  **Initialization:** On application startup, WorkManager enqueues a one-time synchronization task.
2.  **Worker Execution:** A `SyncWorker` executes in the background.
3.  **Delegation to Repositories:** The `SyncWorker` delegates the actual synchronization logic to various data repositories (e.g., `OfflineFirstTopicsRepository`, `OfflineFirstNewsRepository`).
4.  **Incremental Sync:** Repositories use `Proto DataStore` to read the last-synced version for each data type. They then fetch only the changes (new, updated, or deleted items) from the network API.
5.  **Local Database Update:** Fetched changes are then applied to the local `Room` database.
6.  **Reactive Updates:** As the `Room` database updates, `Kotlin Flows` emit new data, which propagates through the application layers (domain, UI) to refresh the user interface.

## Implementation Details

### 1. WorkManager Integration

WorkManager is the backbone of background synchronization, ensuring tasks are executed reliably and efficiently, respecting device constraints like network availability.

#### `NiaApplication`

The synchronization process is initiated when the application starts.

```kotlin
// app/src/main/kotlin/com/google/samples/apps/nowinandroid/NiaApplication.kt
@HiltAndroidApp
class NiaApplication : Application(), ImageLoaderFactory {
    // ...
    override fun onCreate() {
        super.onCreate()
        setStrictModePolicy()

        // Initialize Sync; the system responsible for keeping data in the app up to date.
        Sync.initialize(context = this)
        profileVerifierLogger()
    }
    // ...
}
```

#### `SyncInitializer`

The `Sync` object, defined in `SyncInitializer.kt`, is responsible for enqueuing the WorkManager task.

```kotlin
// sync/work/src/main/kotlin/com/google/samples/apps/nowinandroid/sync/initializers/SyncInitializer.kt
object Sync {
    // This method is initializes sync, the process that keeps the app's data current.
    // It is called from the app module's Application.onCreate() and should be only done once.
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

internal const val SYNC_WORK_NAME = "SyncWorkName"
```

*   `enqueueUniqueWork`: Ensures that only one synchronization work request with `SYNC_WORK_NAME` is active at any time. `ExistingWorkPolicy.KEEP` means if there's existing pending work, the new request is ignored.
*   `SyncWorker.startUpSyncWork()`: Creates the `OneTimeWorkRequest` for the `SyncWorker`.

#### `SyncWorkHelpers`

This file defines constraints for the sync work and contains commented-out code for foreground notifications.

```kotlin
// sync/work/src/main/kotlin/com/google/samples/apps/nowinandroid/sync/initializers/SyncWorkHelpers.kt
// All sync work needs an internet connectionS
val SyncConstraints
    get() = Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .build()

// The functions for foreground notifications are currently commented out in this file.
// fun Context.syncForegroundInfo() = ForegroundInfo(...)
// private fun Context.syncWorkNotification(): Notification { ... }
```

*   `SyncConstraints`: Specifies that an internet connection (`NetworkType.CONNECTED`) is required for the sync work to run.
*   **Foreground Notifications:** Although the `Notifications.md` document mentions foreground sync notifications, the relevant code in `SyncWorkHelpers.kt` (and consequently in `SyncWorker.kt`) is currently commented out. This indicates that foreground notifications for synchronization are not actively implemented in the current codebase, though the infrastructure exists.

#### `SyncWorker`

`SyncWorker` is the actual WorkManager worker that orchestrates the synchronization.

```kotlin
// sync/work/src/main/kotlin/com/google/samples/apps/nowinandroid/sync/workers/SyncWorker.kt
@HiltWorker
internal class SyncWorker @AssistedInject constructor(
    @Assisted private val appContext: Context,
    @Assisted workerParams: WorkerParameters,
    private val niaPreferences: NiaPreferencesDataSource,
    private val topicRepository: TopicsRepository,
    private val newsRepository: NewsRepository,
    private val searchContentsRepository: SearchContentsRepository,
    @Dispatcher(IO) private val ioDispatcher: CoroutineDispatcher,
    private val analyticsHelper: AnalyticsHelper,
    private val syncSubscriber: SyncSubscriber,
) : CoroutineWorker(appContext, workerParams), Synchronizer {

    // The getForegroundInfo() override is commented out
    // override suspend fun getForegroundInfo(): ForegroundInfo = appContext.syncForegroundInfo()

    override suspend fun doWork(): Result = withContext(ioDispatcher) {
        traceAsync("Sync", 0) {
            analyticsHelper.logSyncStarted()
            syncSubscriber.subscribe()

            // First sync the repositories in parallel
            val syncedSuccessfully = awaitAll(
                async { topicRepository.sync() },
                async { newsRepository.sync() },
            ).all { it }

            analyticsHelper.logSyncFinished(syncedSuccessfully)

            if (syncedSuccessfully) {
                searchContentsRepository.populateFtsData() // For full-text search
                Result.success()
            } else {
                Result.retry()
            }
        }
    }

    override suspend fun getChangeListVersions(): ChangeListVersions =
        niaPreferences.getChangeListVersions()

    override suspend fun updateChangeListVersions(
        update: ChangeListVersions.() -> ChangeListVersions,
    ) = niaPreferences.updateChangeListVersion(update)

    companion object {
        fun startUpSyncWork() = OneTimeWorkRequestBuilder<DelegatingWorker>()
            .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
            .setConstraints(SyncConstraints)
            .setInputData(SyncWorker::class.delegatedData())
            .build()
    }
}
```

*   `HiltWorker`: Indicates that Dagger Hilt is used for dependency injection.
*   `Synchronizer` interface: `SyncWorker` implements `Synchronizer`, which provides methods to interact with `ChangeListVersions` (stored in `niaPreferences`).
*   `doWork()`: The main execution block.
    *   It uses `awaitAll` and `async` to run `topicRepository.sync()` and `newsRepository.sync()` in parallel, improving efficiency.
    *   If the sync is successful, it triggers `searchContentsRepository.populateFtsData()` for search indexing.
*   `startUpSyncWork()`: Configures the `OneTimeWorkRequest`, setting it as expedited and applying `SyncConstraints`. It uses a `DelegatingWorker` to properly instantiate `SyncWorker` with Hilt.

### 2. Data Repositories

Repositories are the entry points to the data layer, acting as intermediaries between the application and various data sources (network, local database).

#### `Syncable` and `Synchronizer` Interfaces

These interfaces define the contract for entities involved in synchronization.

```kotlin
// core/data/src/main/kotlin/com/google/samples/apps/nowinandroid/core/data/SyncUtilities.kt
interface Synchronizer {
    suspend fun getChangeListVersions(): ChangeListVersions
    suspend fun updateChangeListVersions(update: ChangeListVersions.() -> ChangeListVersions)
    suspend fun Syncable.sync() = this@sync.syncWith(this@Synchronizer)
}

interface Syncable {
    suspend fun syncWith(synchronizer: Synchronizer): Boolean
}
```

*   `Synchronizer`: Provides access to `ChangeListVersions` and a convenient `sync()` extension function for `Syncable` objects.
*   `Syncable`: Defines the `syncWith` method that repositories implement to perform their specific synchronization logic.

#### `changeListSync` Utility Function

This is a powerful generic utility function that simplifies incremental synchronization for any data type.

```kotlin
// core/data/src/main/kotlin/com/google/samples/apps/nowinandroid/core/data/SyncUtilities.kt
suspend fun Synchronizer.changeListSync(
    versionReader: (ChangeListVersions) -> Int,
    changeListFetcher: suspend (Int) -> List<NetworkChangeList>,
    versionUpdater: ChangeListVersions.(Int) -> ChangeListVersions,
    modelDeleter: suspend (List<String>) -> Unit,
    modelUpdater: suspend (List<String>) -> Unit,
) = suspendRunCatching {
    val currentVersion = versionReader(getChangeListVersions())
    val changeList = changeListFetcher(currentVersion)
    if (changeList.isEmpty()) return@suspendRunCatching true

    val (deleted, updated) = changeList.partition(NetworkChangeList::isDelete)

    modelDeleter(deleted.map(NetworkChangeList::id))
    modelUpdater(updated.map(NetworkChangeList::id))

    val latestVersion = changeList.last().changeListVersion
    updateChangeListVersions { versionUpdater(latestVersion) }
}.isSuccess
```

*   It takes lambdas for reading/updating versions, fetching changes from the network, deleting models, and updating models.
*   It fetches `NetworkChangeList` (containing IDs of deleted/updated items) based on the `currentVersion`.
*   It then applies deletions and updates to the local database via `modelDeleter` and `modelUpdater`.
*   Finally, it updates the `latestVersion` in `ChangeListVersions`.

#### `OfflineFirstTopicsRepository`

This repository is responsible for synchronizing topic data.

```kotlin
// core/data/src/main/kotlin/com/google/samples/apps/nowinandroid/core/data/repository/OfflineFirstTopicsRepository.kt
internal class OfflineFirstTopicsRepository @Inject constructor(
    private val topicDao: TopicDao,
    private val network: NiaNetworkDataSource,
) : TopicsRepository {

    override fun getTopics(): Flow<List<Topic>> =
        topicDao.getTopicEntities().map { it.map(TopicEntity::asExternalModel) }

    override suspend fun syncWith(synchronizer: Synchronizer): Boolean =
        synchronizer.changeListSync(
            versionReader = ChangeListVersions::topicVersion,
            changeListFetcher = { currentVersion ->
                network.getTopicChangeList(after = currentVersion)
            },
            versionUpdater = { latestVersion -> copy(topicVersion = latestVersion) },
            modelDeleter = topicDao::deleteTopics,
            modelUpdater = { changedIds ->
                val networkTopics = network.getTopics(ids = changedIds)
                topicDao.upsertTopics(entities = networkTopics.map(NetworkTopic::asEntity))
            },
        )
}
```

*   **Dependencies:** `TopicDao` (Room) and `NiaNetworkDataSource` (Network).
*   **Reads:** `getTopics()` reads directly from `topicDao`, emitting a `Flow` of `Topic` domain models.
*   **Sync:** Uses `changeListSync` with specific lambdas for topics:
    *   Reads `topicVersion` from `ChangeListVersions`.
    *   Fetches topic `NetworkChangeList` from the network.
    *   Updates `topicVersion` in `ChangeListVersions`.
    *   Deletes topics from `topicDao`.
    *   Fetches changed `NetworkTopic` data from the network and `upsert`s them into `topicDao`.

#### `OfflineFirstNewsRepository`

This repository handles the synchronization of news resources and their associated topics.

```kotlin
// core/data/src/main/kotlin/com/google/samples/apps/nowinandroid/core/data/repository/OfflineFirstNewsRepository.kt
internal class OfflineFirstNewsRepository @Inject constructor(
    private val niaPreferencesDataSource: NiaPreferencesDataSource,
    private val newsResourceDao: NewsResourceDao,
    private val topicDao: TopicDao,
    private val network: NiaNetworkDataSource,
    // private val notifier: Notifier, // Currently commented out
) : NewsRepository {

    override fun getNewsResources(...): Flow<List<NewsResource>> = newsResourceDao.getNewsResources(...)
        .map { it.map(PopulatedNewsResource::asExternalModel) }

    override suspend fun syncWith(synchronizer: Synchronizer): Boolean {
        // ... (simplified for brevity)
        return synchronizer.changeListSync(
            versionReader = ChangeListVersions::newsResourceVersion,
            changeListFetcher = { currentVersion ->
                network.getNewsResourceChangeList(after = currentVersion)
            },
            versionUpdater = { latestVersion -> copy(newsResourceVersion = latestVersion) },
            modelDeleter = newsResourceDao::deleteNewsResources,
            modelUpdater = { changedIds ->
                // ... logic for handling onboarding, viewed status, and notifications (commented out) ...

                // Obtain the news resources which have changed from the network and upsert them locally
                changedIds.chunked(SYNC_BATCH_SIZE).forEach { chunkedIds ->
                    val networkNewsResources = network.getNewsResources(ids = chunkedIds)

                    // Order of invocation matters to satisfy id and foreign key constraints!
                    topicDao.insertOrIgnoreTopics(
                        topicEntities = networkNewsResources.map(NetworkNewsResource::topicEntityShells).flatten().distinctBy(TopicEntity::id),
                    )
                    newsResourceDao.upsertNewsResources(
                        newsResourceEntities = networkNewsResources.map(NetworkNewsResource::asEntity),
                    )
                    newsResourceDao.insertOrIgnoreTopicCrossRefEntities(
                        newsResourceTopicCrossReferences = networkNewsResources.map(NetworkNewsResource::topicCrossReferences).distinct().flatten(),
                    )
                }
                // ... (notification logic commented out) ...
            },
        )
    }
}
```

*   **Dependencies:** `NiaPreferencesDataSource`, `NewsResourceDao`, `TopicDao`, `NiaNetworkDataSource`.
*   **Reads:** `getNewsResources()` reads from `newsResourceDao`, supporting filtering and mapping to `PopulatedNewsResource` (which includes related topics) and then to `NewsResource` domain models.
*   **Sync:** Uses `changeListSync` for news resources.
    *   Reads `newsResourceVersion`.
    *   Fetches news `NetworkChangeList` from the network.
    *   Updates `newsResourceVersion`.
    *   Deletes news resources from `newsResourceDao`.
    *   `modelUpdater` is more complex:
        *   It fetches changed `NetworkNewsResource` data in batches (`chunked`).
        *   Crucially, it first `insertOrIgnoreTopics` from the news resources to ensure all referenced topics exist in the database.
        *   Then, it `upsertNewsResources`.
        *   Finally, it `insertOrIgnoreTopicCrossRefEntities` to establish the many-to-many relationships between news and topics.
        *   It also contains logic for handling onboarding status and viewed news resources. Notification logic is present but commented out.

### 3. Room Database

Room is used for local persistence, providing an abstraction layer over SQLite.

#### Entities (`core/database/model`)

Room entities define the schema of the local database tables.

```kotlin
// core/database/src/main/kotlin/com/google/samples/apps/nowinandroid/core/database/model/TopicEntity.kt
@Entity(tableName = "topics")
data class TopicEntity(
    @PrimaryKey val id: String,
    val name: String,
    val shortDescription: String,
    @ColumnInfo(defaultValue = "") val longDescription: String,
    @ColumnInfo(defaultValue = "") val url: String,
    @ColumnInfo(defaultValue = "") val imageUrl: String,
)

// core/database/src/main/kotlin/com/google/samples/apps/nowinandroid/core/database/model/NewsResourceEntity.kt
@Entity(tableName = "news_resources")
data class NewsResourceEntity(
    @PrimaryKey val id: String,
    val title: String,
    val content: String,
    val url: String,
    @ColumnInfo(name = "header_image_url") val headerImageUrl: String?,
    @ColumnInfo(name = "publish_date") val publishDate: Instant,
    val type: String,
)

// core/database/src/main/kotlin/com/google/samples/apps/nowinandroid/core/database/model/NewsResourceTopicCrossRef.kt
@Entity(
    tableName = "news_resources_topics",
    primaryKeys = ["news_resource_id", "topic_id"],
    foreignKeys = [
        ForeignKey(entity = NewsResourceEntity::class, parentColumns = ["id"], childColumns = ["news_resource_id"], onDelete = ForeignKey.CASCADE),
        ForeignKey(entity = TopicEntity::class, parentColumns = ["id"], childColumns = ["topic_id"], onDelete = ForeignKey.CASCADE),
    ],
    indices = [Index(value = ["news_resource_id"]), Index(value = ["topic_id"])],
)
data class NewsResourceTopicCrossRef(
    @ColumnInfo(name = "news_resource_id") val newsResourceId: String,
    @ColumnInfo(name = "topic_id") val topicId: String,
)

// core/database/src/main/kotlin/com/google/samples/apps/nowinandroid/core/database/model/PopulatedNewsResource.kt
data class PopulatedNewsResource(
    @Embedded val entity: NewsResourceEntity,
    @Relation(
        parentColumn = "id",
        entityColumn = "id",
        associateBy = Junction(
            value = NewsResourceTopicCrossRef::class,
            parentColumn = "news_resource_id",
            entityColumn = "topic_id",
        ),
    )
    val topics: List<TopicEntity>,
)
```

*   `TopicEntity`: Represents a topic in the `topics` table.
*   `NewsResourceEntity`: Represents a news resource in the `news_resources` table.
*   `NewsResourceTopicCrossRef`: A junction table (`news_resources_topics`) to manage the many-to-many relationship between news resources and topics, with `ForeignKey.CASCADE` for automatic deletion.
*   `PopulatedNewsResource`: A data class used by Room to return a `NewsResourceEntity` with its associated `TopicEntity` list, leveraging `@Relation` and `@Junction`.

#### Data Access Objects (DAOs) (`core/database/dao`)

DAOs define methods for interacting with the database.

```kotlin
// core/database/src/main/kotlin/com/google/samples/apps/nowinandroid/core/database/dao/TopicDao.kt
@Dao
interface TopicDao {
    @Query("SELECT * FROM topics") fun getTopicEntities(): Flow<List<TopicEntity>>
    @Upsert suspend fun upsertTopics(entities: List<TopicEntity>)
    @Insert(onConflict = OnConflictStrategy.IGNORE) suspend fun insertOrIgnoreTopics(topicEntities: List<TopicEntity>): List<Long>
    @Query("DELETE FROM topics WHERE id in (:ids)") suspend fun deleteTopics(ids: List<String>)
    // ... other queries
}

// core/database/src/main/kotlin/com/google/samples/apps/nowinandroid/core/database/dao/NewsResourceDao.kt
@Dao
interface NewsResourceDao {
    @Transaction
    @Query(
        value = """
            SELECT * FROM news_resources
            WHERE CASE WHEN :useFilterNewsIds THEN id IN (:filterNewsIds) ELSE 1 END
             AND CASE WHEN :useFilterTopicIds THEN id IN (SELECT news_resource_id FROM news_resources_topics WHERE topic_id IN (:filterTopicIds)) ELSE 1 END
            ORDER BY publish_date DESC
        """,
    )
    fun getNewsResources(...): Flow<List<PopulatedNewsResource>>
    @Upsert suspend fun upsertNewsResources(newsResourceEntities: List<NewsResourceEntity>)
    @Insert(onConflict = OnConflictStrategy.IGNORE) suspend fun insertOrIgnoreTopicCrossRefEntities(newsResourceTopicCrossReferences: List<NewsResourceTopicCrossRef>)
    @Query("DELETE FROM news_resources WHERE id in (:ids)") suspend fun deleteNewsResources(ids: List<String>)
    // ... other queries
}
```

*   `TopicDao`: Provides `Flow`-based read operations and suspend functions for `upserting`, `inserting` (ignoring conflicts), and `deleting` `TopicEntity` objects.
*   `NewsResourceDao`: Offers complex `Flow`-based queries for `PopulatedNewsResource` (including filtering by topics), and suspend functions for `upserting` news, `inserting` cross-references, and `deleting` news resources. The `@Transaction` annotation ensures atomicity for complex queries involving joins.

### 4. Proto DataStore

Proto DataStore is used for storing typed, asynchronous, and consistent data, specifically `UserPreferences` which includes the `ChangeListVersions`.

#### `ChangeListVersions`

A simple data class used to track the version of the last synchronization for each data type.

```kotlin
// core/datastore/src/main/kotlin/com/google/samples/apps/nowinandroid/core/datastore/ChangeListVersions.kt
data class ChangeListVersions(
    val topicVersion: Int = -1,
    val newsResourceVersion: Int = -1,
)
```

#### `UserPreferencesSerializer`

This class handles the serialization and deserialization of the `UserPreferences` Protobuf object.

```kotlin
// core/datastore/src/main/kotlin/com/google/samples/apps/nowinandroid/core/datastore/UserPreferencesSerializer.kt
class UserPreferencesSerializer @Inject constructor() : Serializer<UserPreferences> {
    override val defaultValue: UserPreferences = UserPreferences.getDefaultInstance()
    override suspend fun readFrom(input: InputStream): UserPreferences =
        try { UserPreferences.parseFrom(input) } catch (exception: InvalidProtocolBufferException) { throw CorruptionException("Cannot read proto.", exception) }
    override suspend fun writeTo(t: UserPreferences, output: OutputStream) { t.writeTo(output) }
}
```

*   It uses `UserPreferences.parseFrom()` and `t.writeTo()` to manage the binary serialization of the Protobuf data.

#### `NiaPreferencesDataSource`

The main class that interacts with the `DataStore<UserPreferences>`, providing higher-level methods for managing user data and synchronization versions.

```kotlin
// core/datastore/src/main/kotlin/com/google/samples/apps/nowinandroid/core/datastore/NiaPreferencesDataSource.kt
class NiaPreferencesDataSource @Inject constructor(
    private val userPreferences: DataStore<UserPreferences>,
) {
    val userData = userPreferences.data.map { /* maps UserPreferences proto to UserData domain model */ }

    // ... suspend functions for setting various user preferences ...

    suspend fun getChangeListVersions() = userPreferences.data
        .map {
            ChangeListVersions(
                topicVersion = it.topicChangeListVersion, // Assuming topicChangeListVersion is in UserPreferences proto
                newsResourceVersion = it.newsResourceChangeListVersion, // Assuming newsResourceChangeListVersion is in UserPreferences proto
            )
        }
        .firstOrNull() ?: ChangeListVersions()

    suspend fun updateChangeListVersion(update: ChangeListVersions.() -> ChangeListVersions) {
        userPreferences.updateData { currentPreferences ->
            val updatedChangeListVersions = update(
                ChangeListVersions(
                    topicVersion = currentPreferences.topicChangeListVersion,
                    newsResourceVersion = currentPreferences.newsResourceChangeListVersion,
                ),
            )
            currentPreferences.copy {
                topicChangeListVersion = updatedChangeListVersions.topicVersion
                newsResourceChangeListVersion = updatedChangeListVersions.newsResourceVersion
            }
        }
    }
}
```

*   `userData`: Provides a `Flow` that transforms the raw `UserPreferences` Protobuf object into a `UserData` domain model, used by the UI layer.
*   `getChangeListVersions()`: Extracts the `topicChangeListVersion` and `newsResourceChangeListVersion` from the `UserPreferences` proto and returns them as a `ChangeListVersions` object. This is how the `Synchronizer` reads the last synced version.
*   `updateChangeListVersion()`: Updates the `topicChangeListVersion` and `newsResourceChangeListVersion` within the `UserPreferences` proto based on the result of the synchronization. This is how the `Synchronizer` persists the new synced version.

### 5. Network Layer

The network layer is responsible for fetching data from the remote API.

#### `NiaNetworkDataSource` Interface

Defines the contract for fetching data from the network.

```kotlin
// core/network/src/main/kotlin/com/google/samples/apps/nowinandroid/core/network/NiaNetworkDataSource.kt
interface NiaNetworkDataSource {
    suspend fun getTopics(ids: List<String>? = null): List<NetworkTopic>
    suspend fun getNewsResources(ids: List<String>? = null): List<NetworkNewsResource>
    suspend fun getTopicChangeList(after: Int? = null): List<NetworkChangeList>
    suspend fun getNewsResourceChangeList(after: Int? = null): List<NetworkChangeList>
}
```

#### `RetrofitNiaNetwork`

The Retrofit implementation of `NiaNetworkDataSource`.

```kotlin
// core/network/src/main/kotlin/com/google/samples/apps/nowinandroid/core/network/retrofit/RetrofitNiaNetwork.kt
@Singleton
internal class RetrofitNiaNetwork @Inject constructor(
    networkJson: Json,
    okhttpCallFactory: dagger.Lazy<Call.Factory>,
) : NiaNetworkDataSource {

    private val networkApi = trace("RetrofitNiaNetwork") {
        Retrofit.Builder()
            .baseUrl(NIA_BASE_URL)
            .callFactory { okhttpCallFactory.get().newCall(it) } // Lazy OkHttp initialization
            .addConverterFactory(networkJson.asConverterFactory("application/json".toMediaType()))
            .build()
            .create(RetrofitNiaNetworkApi::class.java)
    }

    override suspend fun getTopics(ids: List<String>?): List<NetworkTopic> = networkApi.getTopics(ids = ids).data
    override suspend fun getNewsResources(ids: List<String>?): List<NetworkNewsResource> = networkApi.getNewsResources(ids = ids).data
    override suspend fun getTopicChangeList(after: Int?): List<NetworkChangeList> = networkApi.getTopicChangeList(after = after)
    override suspend fun getNewsResourceChangeList(after: Int?): List<NetworkChangeList> = networkApi.getNewsResourcesChangeList(after = after)
}

// Private Retrofit API declaration
private interface RetrofitNiaNetworkApi {
    @GET(value = "topics") suspend fun getTopics(@Query("id") ids: List<String>?): NetworkResponse<List<NetworkTopic>>
    @GET(value = "newsresources") suspend fun getNewsResources(@Query("id") ids: List<String>?): NetworkResponse<List<NetworkNewsResource>>
    @GET(value = "changelists/topics") suspend fun getTopicChangeList(@Query("after") after: Int?): List<NetworkChangeList>
    @GET(value = "changelists/newsresources") suspend fun getNewsResourcesChangeList(@Query("after") after: Int?): List<NetworkChangeList>
}
```

*   Uses `Retrofit` with `kotlinx.serialization` for JSON parsing.
*   `RetrofitNiaNetworkApi` defines the specific HTTP endpoints (`@GET`) and their parameters.
*   Provides implementations for all methods in `NiaNetworkDataSource`, fetching data and change lists from the backend.

## Synchronization Flow Summary

1.  **App Launch:** `NiaApplication.onCreate()` calls `Sync.initialize()`.
2.  **WorkManager Enqueue:** `Sync.initialize()` enqueues a `OneTimeWorkRequest` for `SyncWorker` (via `DelegatingWorker`) with network connectivity constraints.
3.  **`SyncWorker` Execution:** When network conditions are met, `SyncWorker.doWork()` starts.
4.  **Parallel Repository Sync:** `SyncWorker` concurrently calls `sync()` on `OfflineFirstTopicsRepository` and `OfflineFirstNewsRepository`.
5.  **Incremental Data Fetch:**
    *   Each repository (e.g., `OfflineFirstTopicsRepository`) uses `niaPreferencesDataSource.getChangeListVersions()` to get its last known `topicVersion` (or `newsResourceVersion`).
    *   It then calls `network.getTopicChangeList(after = currentVersion)` to fetch only the changes from the remote API since that version.
6.  **Local Data Update:**
    *   `changeListSync` processes the fetched `NetworkChangeList`.
    *   Deleted items are removed from the local Room database (`topicDao.deleteTopics()`, `newsResourceDao.deleteNewsResources()`).
    *   Updated/new items are fetched in full from the network (`network.getTopics(ids = changedIds)`, `network.getNewsResources(ids = chunkedIds)`) and then `upsert`ed into the local Room database (`topicDao.upsertTopics()`, `newsResourceDao.upsertNewsResources()`, `newsResourceDao.insertOrIgnoreTopicCrossRefEntities()`).
7.  **Version Update:** After successful updates, `niaPreferencesDataSource.updateChangeListVersion()` is called to save the `latestVersion` in the `UserPreferences` Proto DataStore.
8.  **Reactive UI:** As data in the Room database changes, `Flow`s emitted by the DAOs trigger updates in the repositories, then in the Use Cases (domain layer), and finally in the ViewModels and UI, ensuring the user always sees the most up-to-date information, whether from cache or network.

This comprehensive offline-first synchronization strategy ensures data consistency, provides a resilient user experience, and optimizes network usage by only fetching necessary changes.
