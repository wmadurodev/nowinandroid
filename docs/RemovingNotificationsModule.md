# Guide: Safely Removing the `:core:notifications` Module

This guide provides a step-by-step process to remove the `:core:notifications` module while maintaining the functionality of the offline-first data synchronization.

## Overview of Dependencies
The `:core:notifications` module is currently a dependency for:
1. **`:core:data`**: Uses `Notifier` in `OfflineFirstNewsRepository`.
2. **`:sync:work`**: Uses notification resources in `SyncWorkHelpers.kt`.
3. **`:app`**: Provides the concrete implementation (`SystemTrayNotifier`) via Hilt.

---

## Step 1: Migrate Shared Resources
The `:sync:work` module uses a notification icon located in `:core:notifications`.

1. **Move Icon**: Copy `core_notifications_ic_nia_notification.xml` from `core/notifications/src/main/res/drawable/` to a shared UI module, such as `core/designsystem/src/main/res/drawable/`.
2. **Update References**: In `sync/work/src/main/kotlin/com/google/samples/apps/nowinandroid/sync/initializers/SyncWorkHelpers.kt`, update the resource reference:
   ```kotlin
   // From:
   com.google.samples.apps.nowinandroid.core.notifications.R.drawable.core_notifications_ic_nia_notification
   // To:
   com.google.samples.apps.nowinandroid.core.designsystem.R.drawable.core_notifications_ic_nia_notification
   ```

## Step 2: Decouple `OfflineFirstNewsRepository`
The repository uses the `Notifier` to send user alerts. We need to remove this dependency.

1. **Modify Constructor**: Open `core/data/src/main/kotlin/com/google/samples/apps/nowinandroid/core/data/repository/OfflineFirstNewsRepository.kt`.
2. **Remove Notifier**: 
    - Remove `private val notifier: Notifier` from the `@Inject constructor`.
    - Remove the `import com.google.samples.apps.nowinandroid.core.notifications.Notifier`.
3. **Remove Notification Call**: Delete the block of code inside `syncWith` that calls `notifier.postNewsNotifications`.
   ```kotlin
   // Remove this:
   if (addedNewsResources.isNotEmpty()) {
       notifier.postNewsNotifications(newsResources = addedNewsResources)
   }
   ```

## Step 3: Remove Module Dependencies
Once the code no longer references `Notifier` or resources from that module, update the Gradle files.

1. **`:core:data`**: In `core/data/build.gradle.kts`, remove `implementation(projects.core.notifications)`.
2. **`:sync:work`**: In `sync/work/build.gradle.kts`, remove any dependency on `:core:notifications` (if present, check both `implementation` and `androidTestImplementation`).
3. **`:app`**: In `app/build.gradle.kts`, remove `implementation(projects.core.notifications)`.

## Step 4: Clean Up Hilt Modules
The `:app` module or a specific DI module likely provides the `Notifier` binding.

1. **Locate Binding**: Search for `@Binds abstract fun bindNotifier` or `@Provides fun provideNotifier`.
2. **Delete Binding**: Remove the provider/binding for `Notifier` and `SystemTrayNotifier`. This is usually found in a `NotificationsModule.kt` file.

## Step 5: Delete the Module
1. Delete the physical directory `/core/notifications`.
2. Remove `:core:notifications` from the `settings.gradle.kts` file.

---

## Verification
1. **Gradle Sync**: Run a Gradle sync to ensure all dependency references are gone.
2. **Build**: Run `./gradlew assembleDebug` to verify the project compiles.
3. **Test**: Run `./gradlew :sync:work:test` and `./gradlew :core:data:test` to ensure synchronization logic still functions correctly without the notification side-effects.
