# Project Module Structure

This document provides a comprehensive overview of the modules within the Now in Android (NiA) project, categorized by their layer and responsibility.

## Overview

The project follows a modularized architecture to improve build times, maintainability, and reusability. Modules are broadly categorized into `app`, `feature`, and `core`.

## 1. App Modules

*   **`:app`**: The main entry point of the application. It contains the `MainActivity`, `NiaApp` (scaffolding), and the top-level navigation logic (`NiaNavHost`). It ties all features and core modules together.
*   **`:app-nia-catalog`**: A standalone app used to showcase and test the project's design system and UI components in isolation.

## 2. Feature Modules

Located in the `feature/` directory, these modules represent distinct user-facing screens or flows. Most features are split into `:api` (navigation/contracts) and `:impl` (logic/UI).

*   **`:feature:bookmarks`**: Handles the saved/bookmarked news resources.
*   **`:feature:foryou`**: The home screen, providing a personalized feed of news.
*   **`:feature:interests`**: Allows users to follow/unfollow topics of interest.
*   **`:feature:search`**: Provides functionality to search for news and topics.
*   **`:feature:settings`**: App settings, including theme and dynamic color preferences.
*   **`:feature:topic`**: Detailed view for a specific topic and its associated news.

## 3. Core Modules

Located in the `core/` directory, these provide common functionality shared across multiple features.

### Data & Logic
*   **`:core:data`**: The repository layer. Orchestrates data from network and local sources.
*   **`:core:database`**: Local persistence using Room (DAOs, Entities, Migrations).
*   **`:core:datastore`**: User preferences and small key-value data using Proto DataStore.
*   **`:core:network`**: Network communication logic using Retrofit and OkHttp.
*   **`:core:domain`**: Business logic layer containing Use Cases that combine multiple repositories.
*   **`:core:model`**: Pure Kotlin/Java models used across all modules.

### UI & Design
*   **`:core:designsystem`**: The NiA design system (Theme, Icons, Base Components).
*   **`:core:ui`**: Reusable UI components that depend on the data layer (e.g., News Resource cards).
*   **`:core:navigation`**: Shared navigation utilities.

### Utilities & Infrastructure
*   **`:core:common`**: Common utilities, dispatchers, and project-wide constants.
*   **`:core:analytics`**: Logic for logging events and analytics.
*   **`:core:notifications`**: Logic for handling system notifications.

### Testing
*   **`:core:testing`**: Shared testing utilities and rules.
*   **`:core:data-test`**: Fakes and mocks for the data layer.
*   **`:core:datastore-test`**: Utilities for testing DataStore.
*   **`:core:screenshot-testing`**: Infrastructure for Roborazzi-based screenshot tests.

## 4. Other Modules

*   **`:sync`**: Logic for background data synchronization using WorkManager.
*   **`:benchmarks`**: Macrobenchmark tests to measure app performance.
*   **`:lint`**: Custom lint rules for the project.
*   **`:build-logic`**: Shared Gradle build logic and convention plugins.
