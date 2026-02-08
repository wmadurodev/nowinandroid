# Notifications Implementation in Now in Android

This document explains the full notification implementation in the Now in Android (NiA) app, specifically focusing on how foreground sync notifications and news notifications are handled.

## 1. Foreground Sync Notifications

When the app performs a data sync in the background using `WorkManager`, it may need to run as a foreground service to ensure it isn't killed by the system. This requires displaying a persistent notification.

### `SyncWorkHelpers.kt`
Located at: `sync/work/src/main/kotlin/com/google/samples/apps/nowinandroid/sync/initializers/SyncWorkHelpers.kt`

This file contains helper functions to create the notification required for foreground work:

- **`syncForegroundInfo()`**: An extension function on `Context` that returns a `ForegroundInfo` object. This object bundles a notification ID (`SYNC_NOTIFICATION_ID`) and the `Notification` itself.
- **`syncWorkNotification()`**: A private function that constructs the actual `Notification`.
    - It ensures a `NotificationChannel` exists (required for API 26+).
    - It uses `NotificationCompat.Builder` to set the small icon, title, and priority.
    - The icon is pulled from the `:core:notifications` module.

### `SyncWorker.kt`
The `SyncWorker` (in `sync/work/.../workers/SyncWorker.kt`) overrides `getForegroundInfo()` to call `appContext.syncForegroundInfo()`. This allows `WorkManager` to promote the worker to a foreground service when necessary (e.g., for expedited jobs).

### Execution Flow on App Start
1. **`NiaApplication.onCreate()`** calls `Sync.initialize(this)`.
2. **`Sync.initialize()`** (in `SyncInitializer.kt`) enqueues a unique `SyncWorker`.
3. **`SyncWorker`** runs and, if expedited, displays the sync notification via **`SyncWorkHelpers`**.

---

## 2. News Notifications

News notifications are user-facing alerts sent when new news resources are synchronized that match the user's followed topics.

### `Notifier` Interface and `SystemTrayNotifier`
- **`Notifier`**: A core interface defined in `:core:notifications` that defines the contract for posting notifications.
- **`SystemTrayNotifier`**: The Android-specific implementation of `Notifier`.
    - It handles creating notification channels.
    - It builds notifications for individual news items and a summary notification for groups.
    - It handles deep linking to specific news resources in the app.
    - It respects the `POST_NOTIFICATIONS` permission (API 33+).

### Integration with Data Layer
The `OfflineFirstNewsRepository` (in `:core:data`) is responsible for triggering these notifications during the sync process:

1. During `syncWith`, it fetches new news items.
2. It checks if the user has completed onboarding and which topics they follow.
3. If new news items match the user's interests, it calls `notifier.postNewsNotifications(addedNewsResources)`.

---

## Summary of Components

| Component | Responsibility | Location |
| :--- | :--- | :--- |
| `SyncWorkHelpers` | Foreground sync notification construction | `:sync:work` |
| `SyncWorker` | Triggering foreground sync info | `:sync:work` |
| `Notifier` | Interface for posting user notifications | `:core:notifications` |
| `SystemTrayNotifier` | Android implementation for system tray alerts | `:core:notifications` |
| `OfflineFirstNewsRepository` | Logic for when to send news notifications | `:core:data` |

## Impact of Removing `:core:notifications`

If the `:core:notifications` module is removed, the app's **offline-first data synchronization would stop working** unless the following changes are made:

1. **Dependency Breaks**: `:core:data` and `:sync:work` both depend on `:core:notifications`. Removing it would cause compilation errors in these modules.
2. **`OfflineFirstNewsRepository`**: This class has a hard dependency on the `Notifier` interface. It uses it to post notifications when new data arrives. To keep it working, you would need to remove the `notifier` parameter from the constructor and the call to `notifier.postNewsNotifications`.
3. **`SyncWorkHelpers`**: This helper uses resources (like the notification icon) from `:core:notifications`. You would need to move these resources elsewhere or update the references.

In short, "online-first" (or rather, the synchronization logic) is tightly coupled with the notification system in the current architecture because the repository uses the `Notifier` to alert the user of new data found during sync. Removing the module requires refactoring the repository to decouple these concerns.
