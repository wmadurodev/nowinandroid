# Notifications Implementation in Now in Android

This document explains the full notification implementation in the Now in Android (NiA) app,
specifically focusing on how foreground sync notifications and news notifications are handled.

## 1. Foreground Sync Notifications

When the app performs a data sync in the background using `WorkManager`, it may need to run as a
foreground service to ensure it isn't killed by the system. This requires displaying a persistent
notification.

### `SyncWorkHelpers.kt`

Located at:
`sync/work/src/main/kotlin/com/google/samples/apps/nowinandroid/sync/initializers/SyncWorkHelpers.kt`

This file contains helper functions to create the notification required for foreground work:

- **`syncForegroundInfo()`**: An extension function on `Context` that returns a `ForegroundInfo`
  object. This object bundles a notification ID (`SYNC_NOTIFICATION_ID`) and the `Notification`
  itself.
- **`syncWorkNotification()`**: A private function that constructs the actual `Notification`.
    - It ensures a `NotificationChannel` exists (required for API 26+).
    - It uses `NotificationCompat.Builder` to set the small icon, title, and priority.
    - The icon is pulled from the `:core:notifications` module.

### `SyncWorker.kt`

The `SyncWorker` (in `sync/work/.../workers/SyncWorker.kt`) overrides `getForegroundInfo()` to call
`appContext.syncForegroundInfo()`. This allows `WorkManager` to promote the worker to a foreground
service when necessary (e.g., for expedited jobs).

---

## 2. News Notifications

News notifications are user-facing alerts sent when new news resources are synchronized that match
the user's followed topics.

### `Notifier` Interface and `SystemTrayNotifier`

- **`Notifier`**: A core interface defined in `:core:notifications` that defines the contract for
  posting notifications.
- **`SystemTrayNotifier`**: The Android-specific implementation of `Notifier`.
    - It handles creating notification channels.
    - It builds notifications for individual news items and a summary notification for groups.
    - It handles deep linking to specific news resources in the app.
    - It respects the `POST_NOTIFICATIONS` permission (API 33+).

### Integration with Data Layer

The `OfflineFirstNewsRepository` (in `:core:data`) is responsible for triggering these notifications
during the sync process:

1. During `syncWith`, it fetches new news items.
2. It checks if the user has completed onboarding and which topics they follow.
3. If new news items match the user's interests, it calls
   `notifier.postNewsNotifications(addedNewsResources)`.

---

## Summary of Components

| Component                    | Responsibility                                | Location              |
|:-----------------------------|:----------------------------------------------|:----------------------|
| `SyncWorkHelpers`            | Foreground sync notification construction     | `:sync:work`          |
| `SyncWorker`                 | Triggering foreground sync info               | `:sync:work`          |
| `Notifier`                   | Interface for posting user notifications      | `:core:notifications` |
| `SystemTrayNotifier`         | Android implementation for system tray alerts | `:core:notifications` |
| `OfflineFirstNewsRepository` | Logic for when to send news notifications     | `:core:data`          |

# Notifications Near App Startup in Now in Android

In the **Now in Android** app, there are two main types of notifications that can occur near app
startup.  
The one most likely visible *“when the app starts”* is the **Foreground Sync Notification**.

---

## 1. Foreground Sync Notification

This notification informs the user that the app is syncing data in the background. It is triggered
during the initial synchronization process when the app is launched.

- **Trigger**  
  `NiaApplication.onCreate()` calls `Sync.initialize(this)`, which enqueues a `SyncWorker`.

- **Worker**  
  `SyncWorker.kt` (in `:sync:work`) is an *expedited worker*.  
  Because it’s expedited, it can run as a foreground service, which requires a notification.

- **Notification Creation**  
  The `SyncWorker` calls `appContext.syncForegroundInfo()` defined in `SyncWorkHelpers.kt`.

    - **Location**  
      `sync/work/src/main/kotlin/com/google/samples/apps/nowinandroid/sync/initializers/SyncWorkHelpers.kt`

    - The `syncWorkNotification()` function in this file constructs the actual `Notification` object
      using `NotificationCompat.Builder`.

---

## 2. News Notifications

If the sync process identifies new news items that match the user’s followed topics, it may also
post a user-facing news notification.

- **Logic**  
  Handled in `OfflineFirstNewsRepository.kt` within the `syncWith` method.

- **Implementation**  
  Uses the `Notifier` interface (implemented by `SystemTrayNotifier` in `:core:notifications`) to
  post notifications to the system tray.

    - **Location**  
      `core/data/src/main/kotlin/com/google/samples/apps/nowinandroid/core/data/repository/OfflineFirstNewsRepository.kt`

---

## Summary of Key Files

- **SyncInitializer.kt**  
  Enqueues the sync work on startup.

- **SyncWorker.kt**  
  The background task that triggers the foreground state.

- **SyncWorkHelpers.kt**  
  Contains the UI code / builder for the *“Syncing…”* notification.

- **SystemTrayNotifier.kt**  
  Contains the implementation for showing news-specific alerts.


