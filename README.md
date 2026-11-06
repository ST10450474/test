# 🌸 Bloom: Every Petal Counts

## Technical Report

**PROG7314: Programming 3D | POE Part 3: Final POE**
**Group: Cipher**


| | |
|---|---|
| **Module** | PROG7314|
| **Group name** | Cipher |
| **Group members** | Jenna Martin - ST10450474 (Group Leader) · Hermalia Naidoo - ST10439427 · Devashni Naidoo - ST10447406 · Priyanka Govender - ST10446793 |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Purpose of the Application](#2-purpose-of-the-application)
3. [Design Considerations](#3-design-considerations)
4. [Tech Stack](#4-tech-stack)
5. [System Architecture](#5-system-architecture)
6. [Data Design](#6-data-design)
7. [Key Technical Implementations](#7-key-technical-implementations)
8. [REST API](#8-rest-api)
9. [Repository Structure](#9-repository-structure)
10. [Getting Started](#10-getting-started)
11. [Testing Strategy and Evidence](#11-testing-strategy-and-evidence)
12. [DevOps and CI/CD Integration](#12-devops-and-cicd-integration)
13. [Security Considerations](#13-security-considerations)
14. [Multi-Language Support](#14-multi-language-support)
15. [Known Limitations and Future Work](#15-known-limitations-and-future-work)
16. [Design Documentation](#16-design-documentation)
17. [YouTube Demonstration Video](#17-youtube-demonstration-video)
18. [AI and Software Disclosures](#18-ai-and-software-disclosures)

---

## 1. Executive Summary

Bloom is an Android application that helps people build healthy routines without the guilt that most habit trackers rely on. Instead of a streak counter that resets to zero after a missed day, Bloom rewards effort with a growing avatar (a panda, bunny or koala). A missed day slows that companion's growth but never erases it.

Each day the user completes **one combined check-in** that records both their habits and their mood. Over time, Bloom connects the two and shows the user how their habits relate to how they feel. Friends and groups can share habits, and push notifications keep the experience social without becoming noisy.

This report describes the finished system: the problem it solves, the design decisions behind it, how the Android client, REST API and cloud services fit together, and how the project is tested, built and delivered automatically through GitHub Actions.

### Core Features Implemented

This is the final submission. It covers all of the core minimum requirements plus our custom, user-defined features.

| Feature | Status | Notes |
|---|---|---|
| SSO Registration & Login | ✅ | Google Sign-In via Firebase Authentication |
| User Settings | ✅ | Notification time, language, appearance, biometric toggle |
| REST API connection | ✅ | Retrofit + OkHttp client talking to the custom Express API |
| Offline mode and sync | ✅ | Room local storage, WorkManager background sync, last-write-wins conflict resolution |
| Real-time Push Notifications | ✅ | Firebase Cloud Messaging for social events, a daily check-in reminder and a repeating habit reminder |
| Biometric Authentication | ✅ | Enforced app lock using the Android Biometric API, re-locks after 30 seconds in the background |
| Multi-language support | ✅ | English + isiZulu string resources, switchable in-app without a restart |
| Growing Avatar System (custom) | ✅ | Panda / Bunny / Koala, XP-based growth stages |
| Avatar Wardrobe (custom) | ✅ | Ten unlockable accessories, unlock rules enforced by the server |
| Combined Mood & Habit Check-in (custom) | ✅ | Single daily screen for habit completions + mood rating |
| Journal with prompts (custom) | ✅ | Local-only journal entries, stored in Room |
| First-aid breathing exercise (custom) | ✅ | Guided breathing offered when mood is logged as 1 or 2 |
| Insights (custom) | ✅ | Weekly mood chart and personalised habit-and-mood findings |
| Friends, shared habits and groups (custom) | ✅ | Friend requests, habits shared with a friend or a group |

✅ = working in this submission


---

## 2. Purpose of the Application

### 2.1 Problem Statement

Mainstream habit trackers tend to rely on streaks and loss-aversion. When a streak breaks, users often feel they have failed and abandon the app entirely. Habit tracking and mood tracking are also usually separate products, so users never see how their routines affect their wellbeing.

### 2.2 Aim

Bloom aims to provide a **positive, low-pressure habit and mood tracker** that:

- rewards progress rather than punishing lapses;
- lets users log habits and mood in a single, fast daily action;
- turns that data into simple, personal insights;
- supports accountability through friends and groups without shaming anyone; and
- keeps working when the phone has no internet connection.

### 2.3 Target Users

Students and young adults who want to build routines (water, exercise, sleep, study) and who respond better to encouragement than to pressure.


---

## 3. Design Considerations

**Shame-free engagement.** Growth is driven by experience points (XP): +3 XP for logging a mood and +5 XP for completing a habit. XP only ever increases, so an avatar can never "die" or reset. The avatar reaches stage 2 at 100 XP. This directly supports the aim of rewarding effort instead of punishing absence.

**Offline-first by default.** Users log habits at inconsistent times and places, often with poor connectivity. The client therefore always reads from and writes to the local Room database, and treats the server as a synchronisation target rather than a dependency. Check-ins made offline are marked `PENDING` and uploaded automatically when the connection returns.

**Server-side authority.** The API calculates XP, avatar stage, streaks, accessory unlock rules and insights. The client submits raw events only (for example "habit X was ticked on this date"). This keeps game logic consistent across devices and stops a modified client from granting itself progress.

**Delegated identity.** Bloom does not run its own password or token system. Firebase Authentication is the identity provider, and the API verifies the Firebase ID token on every protected request.

**Fail-safe notifications.** Push notifications support the experience and must never break core flows. The server's notification helpers never reject: a failed push is logged and ignored, users with notifications switched off or no registered device are skipped, and tokens that Firebase reports as invalid are cleared automatically.

**Calm, non-intrusive reminders.** The repeating habit reminder only runs between 08:00 and 21:00, and each new reminder replaces the previous one instead of stacking, so users are never buzzed overnight or flooded with notifications.

**Privacy of reflective content.** Journal entries are deliberately local-only and are never sent to the server.

**Inclusive and localised.** All user-facing text lives in string resources with full English and isiZulu coverage (296 strings in each). Push notification text is also localised on the server using each user's saved language. A dark mode is available.

**Maintainable layered structure.** The client separates Compose UI, ViewModels, repositories, and local (Room) and remote (Retrofit) data sources. The API separates routes, controllers, middleware and pure utility functions, so business rules such as streak maths and sync conflict rules can be unit tested without a database.

---

## 4. Tech Stack

| Layer | Technology |
|---|---|
| Mobile application | Kotlin + Jetpack Compose (Material 3) |
| Android architecture | Android Architecture Components + ViewModel, StateFlow, Navigation Compose |
| REST API | Node.js (18+) + Express |
| Server database | PostgreSQL (via Prisma ORM) |
| Local database/cache | Room (schema v3, with migrations) |
| NoSQL store | Cloud Firestore (equipped accessory) |
| Object storage | Neon Object Storage (S3-compatible, wardrobe images) |
| Authentication | Firebase Authentication (Google SSO) |
| Networking | Retrofit + OkHttp + Gson |
| Background tasks | WorkManager (Android), node-cron (server) |
| Push notifications | Firebase Cloud Messaging |
| Biometrics | AndroidX Biometric |
| Image loading | Coil |
| API testing | Jest + Supertest |
| Android unit testing | JUnit + MockK + kotlinx-coroutines-test |
| API docs | Swagger / OpenAPI (`/api/docs`) |
| API hardening | helmet, cors |
| CI/CD | GitHub Actions |
| Version control | Git + GitHub |

The Android app targets minSdk 25 and targetSdk 36, built with Kotlin 2.2 and Java 17.

---

## 5. System Architecture

```
Android Client (Kotlin + Jetpack Compose)
  UI Layer (Compose screens, bottom-navigation shell, deep-link handling)
        │
  ViewModel Layer (Android Architecture Components)
        │
   ┌────┴─────────────┐
Room (local DB)     Retrofit + OkHttp ──HTTPS + Firebase ID Token──▶ REST API (Node.js + Express, hosted on Render)
   ▲                                                                   │
   │                                                              Auth Middleware (verifies Firebase ID token)
WorkManager +                                                          │
ConnectivityObserver                                            Route Controllers (/auth /users /habits /moods /sync ...)
(background sync)                                                      │
                                                                Business Logic (XP, streaks, sync rules, insights)
                                                                       │
                                              ┌────────────────────────┼───────────────────────┐
                                        PostgreSQL (Neon,        Neon Object Storage      Firestore
                                        via Prisma)              (wardrobe images)        (equipped accessory)

                                         node-cron jobs: daily check-in reminder + repeating habit reminder

External SDKs: Firebase Authentication (Google SSO) · Firebase Cloud Messaging · Android Biometric API
```

A full UML diagram of this architecture is included in the Part 1 Planning & Design document.

### 5.1 Request lifecycle

1. The user signs in with Google and Firebase returns an ID token.
2. `AuthInterceptor` attaches `Authorization: Bearer <token>` to every outgoing request.
3. The API's auth middleware verifies the token with the Firebase Admin SDK and makes sure a user row exists.
4. The controller applies validation and business rules, reads or writes PostgreSQL through Prisma, and returns JSON.
5. Where relevant, the controller fires a push notification without waiting for it to finish.

### 5.2 Client navigation

The app has an authentication and onboarding flow (Launch, Login, Biometric setup, Avatar choice, Habit preferences, Display name), followed by a main shell with five bottom-navigation tabs: **Home, Insights, Avatar, Friends and Profile**. Settings is reachable from the main shell.

---

## 6. Data Design

### 6.1 Server database (PostgreSQL via Prisma)

| Entity | Purpose |
|---|---|
| `User` | Profile and settings. `id` is the Firebase UID. Also stores XP, avatar stage, language, dark mode, FCM token, notification switch and reminder hour. |
| `Habit` / `HabitCompletion` | A user's habits, and one completion row per habit per calendar day (unique on habit and date). |
| `HabitCompletionTombstone` | Records when a tick was undone, so offline sync can tell "never ticked" from "ticked then un-ticked". |
| `Mood` | A 1-5 mood entry with an optional short note and an `updatedAt` clock used for conflict resolution. |
| `Friendship` | A request that moves from `PENDING` to `ACCEPTED`. Declining deletes the row. |
| `SharedHabit` / `SharedHabitCompletion` | A habit shared between two friends, with one completion row per member per day. |
| `Group` / `GroupMember` / `GroupHabit` / `GroupHabitCompletion` | Named groups of friends sharing habits. |
| `Accessory` | Wardrobe catalogue (name, required stage, image key). |

Schema changes are tracked as Prisma migrations, so the database can be rebuilt reproducibly from the repository.

### 6.2 Local database (Room, schema version 3)

| Table | Purpose |
|---|---|
| `journal_entries` | Private journal entries (device-only). |
| `cached_habits` | Habit list and streaks so the check-in screen opens offline. |
| `local_moods` | Mood entries with a client-generated UUID, `updatedAt` clock and `syncStatus`. |
| `local_completions` | Habit ticks (including un-ticks) keyed by user, habit and date, with `syncStatus`. |
| `cached_moods` / `cached_completions` | Read-only copies that keep the Insights screen working offline. |

Room migrations (1 to 2 and 2 to 3) preserve existing user data between app versions.

### 6.3 Why three stores?

| Store | Used for | Reason |
|---|---|---|
| PostgreSQL | Relational, authoritative data | Strong relationships and constraints (friends, groups, completions). |
| Firestore | The equipped accessory (`avatarOutfits/{uid}`) | Small per-user document that suits a NoSQL store. |
| Neon Object Storage | Accessory images | Binary assets served by public URL instead of living in the database. |

---

## 7. Key Technical Implementations

### 7.1 Offline synchronisation

**Client** (`SyncManager`, `SyncWorker`, `SyncRepository`, `ConnectivityObserver`)

- `ConnectivityObserver` watches for validated internet access and triggers a sync each time the phone reconnects.
- `SyncManager` enqueues a one-time WorkManager job on reconnect, plus a **15-minute periodic job** as a safety net. Both require a network connection.
- `SyncWorker` runs `SyncRepository.sync()`, which **pushes** pending rows to `POST /api/sync` and then **pulls** the latest habits and 30 days of moods and ticks from `GET /api/sync/pull`.
- Failures are retried with exponential backoff (30 seconds), up to five attempts. Only retryable errors (network, HTTP 5xx, 408 and 429) are retried.
- A top-bar **sync status chip** shows whether the user is online, syncing, or has changes waiting.

**Server** (`sync.controller.js`, `syncRules.js`)

- **Idempotent:** mood IDs are UUIDs created on the phone and completions are keyed by habit and date, so resending a batch never creates duplicates and never awards XP twice.
- **Conflict resolution is last-write-wins** using the device's `updatedAt`. If two changes carry exactly the same time, the server's copy wins, so a retried request can never flip data.
- **Clock-skew protection:** client timestamps more than five minutes in the future are replaced with the server time.
- **Tombstones** record un-ticks so a later tick on another device is not wrongly overwritten.
- Batches are limited (200 moods, 500 completions) and fully validated.



### 7.2 Push notifications (Firebase Cloud Messaging)

- The app registers its FCM token with the API (`PUT /users/me/fcm-token`) on login and whenever the token refreshes. Registration retries up to three times with back-off, and the token is deleted on sign-out so a shared phone never notifies the previous user.
- The server sends notifications for: friend requests and acceptances, shared habit created / completed / updated / deleted, and group habit created / completed / updated / deleted.
- **Daily check-in reminder:** an hourly `node-cron` job notifies users, at their chosen reminder hour (default 18:00, `Africa/Johannesburg`), who have notifications on and have **not yet logged a mood today**.
- **Repeating habit reminder:** a second job runs every 20 minutes (configurable) between 08:00 and 21:00 and notifies users who still have an active daily habit they have not ticked today. The message lists up to three habits (for example "Drink water, Exercise +2 more"). Each reminder carries a tag so it **replaces** the previous one rather than stacking.
- Message text is built in the recipient's saved language (English or isiZulu).
- Every notification carries a `screen` value. `BloomMessagingService` and `DeepLinkManager` route a tap to the right screen (friends, requests, shared habits, group habits or check-in) whether the app was in the foreground, background or closed.
- When a push arrives while the app is open, `DeepLinkManager` broadcasts it so the Friends and Friend Requests screens refresh themselves automatically.



### 7.3 Biometric authentication and app lock

- `BiometricHelper` wraps `androidx.biometric` (strong biometrics only) and reports every outcome as a typed result (success, unavailable, cancelled, too many failures, locked out, error) rather than throwing, so a sensor problem cannot crash the app.
- `AppLockManager` holds process-wide lock state. On a cold start the app starts **locked** if the user is signed in and has enabled the lock, so the main screen is never visible before authentication.
- Returning from the background after **30 seconds or more** re-locks the app. The lock screen is drawn over the app, so the user does not lose their place.
- Logging out clears the device-level setting so the next account on that phone is not locked by someone else's choice.



### 7.4 Avatar growth and wardrobe

- XP is awarded server-side and the stage is derived from it (stage 2 at 100 XP).
- The wardrobe catalogue is returned by `GET /api/accessories`, with each item marked locked or unlocked for the user's stage. Equipping an item (`PUT /api/accessories/equip`) is **rejected with 403** if the user has not reached the required stage.
- If Firestore is unreachable, the catalogue still loads and nothing is shown as equipped.




### 7.5 Insights

The Insights screen shows a weekly mood chart that the user can page through week by week. The server also compares the user's average mood on days a habit **was** completed against days it **was not**, and returns the top three differences. To avoid misleading conclusions, a finding is only shown when there are **at least two days on each side**. Cached copies of moods and completions keep this screen usable offline.




### 7.6 Friends, shared habits and groups

Friend requests move from `PENDING` to `ACCEPTED`, and the API prevents a reverse request being created while one already exists. Shared habits (between two friends) and group habits (within a named group) define a target (times per day, week or month) and a date range. Each member ticks their own completions and the other members are notified.




### 7.7 First-aid breathing exercise

When a user logs a mood of **1 or 2**, a guided exercise (breathe in 4 seconds, hold 4 seconds, breathe out 6 seconds) is offered. Afterwards the user can restart it, move to the journal, or skip. This supports wellbeing at the moment it is most relevant, without forcing it.


---

## 8. REST API

The API is built with **Node.js and Express**, uses **Prisma** for database access, and is documented with **Swagger / OpenAPI** at `/api/docs` (raw spec at `/api/openapi.json`). `helmet` provides secure HTTP headers and CORS is configurable through `CORS_ORIGIN`.

Authentication is delegated entirely to Firebase: the client signs in with Google SSO, then sends its Firebase ID token as a Bearer token on every request. The API verifies it with the Firebase Admin SDK rather than maintaining its own password or JWT system. All routes except `/health` and `/api/docs` require this token. Habit, shared-habit and group routes check ownership or membership, and return 404 rather than 403 where revealing that a record exists would leak information.

| Area | Method and route | Purpose |
|---|---|---|
| Health | `GET /health` | Liveness check (no auth). |
| Auth | `POST /auth/login` | Verify the Firebase token and create the profile on first login. |
| Users | `GET / PATCH / DELETE /users/me` | Read, update or delete the profile and settings. |
| Users | `PUT /users/me/fcm-token` | Register the device's push token. |
| Habits | `GET / POST /api/habits` | List (with streaks) and create habits. |
| Habits | `GET / PATCH / DELETE /api/habits/:id` | Read, update or delete a habit. |
| Completions | `GET / POST /api/habits/:id/completions` | List completions or mark one complete (triggers streak and XP). |
| Completions | `DELETE /api/habits/:id/completions/:date` | Un-tick a day. |
| Streaks | `GET /api/habits/:id/streak` | Current and longest streak. |
| Moods | `GET / POST /api/moods`, `PATCH / DELETE /api/moods/:id` | Mood entries. |
| Insights | `GET /api/insights` | Weekly mood data and personalised findings. |
| Sync | `POST /api/sync`, `GET /api/sync/pull` | Push offline changes and pull the latest state. |
| Friends | `GET /api/friends`, `GET /api/friends/search` | List friends and search users. |
| Friends | `GET / POST /api/friends/requests`, `POST .../:id/accept`, `POST .../:id/decline` | Friend request workflow. |
| Shared habits | `GET / POST /api/friends/:friendId/shared-habits` | List and create shared habits. |
| Shared habits | `PATCH / DELETE /api/shared-habits/:id`, `POST / DELETE .../completions` | Manage and tick shared habits. |
| Groups | `GET / POST /api/groups` | List and create groups. |
| Group habits | `GET / POST /api/groups/:groupId/habits`, `PATCH / DELETE .../:habitId`, `POST / DELETE .../completions` | Manage and tick group habits. |
| Wardrobe | `GET /api/accessories`, `PUT /api/accessories/equip` | Catalogue and equipping. |


A Postman collection for manual testing is included at `server/bloom-api/postman/bloom-api.postman_collection.json`.

---

## 9. Repository Structure

```
.
├── app/                              # Android client
│   ├── src/main/java/com/emkndn/bloom/
│   │   ├── data/
│   │   │   ├── local/                # Room database, DAOs, entities
│   │   │   ├── model/                # API and domain models
│   │   │   ├── network/              # Retrofit client, API interfaces, auth interceptor
│   │   │   ├── repository/           # Repositories (habits, moods, auth, friends, groups, ...)
│   │   │   └── sync/                 # SyncManager, SyncWorker, SyncRepository, ConnectivityObserver
│   │   ├── notifications/            # FCM service, token registrar, deep-link manager
│   │   ├── security/                 # AppLockManager, BiometricHelper
│   │   ├── ui/                       # Compose screens grouped by feature
│   │   └── viewmodel/                # ViewModels
│   ├── src/main/res/                 # values/ (English), values-zu/ (isiZulu), drawables, fonts
│   ├── src/test/                     # JVM unit tests (JUnit + MockK)
│   └── src/androidTest/              # Instrumented tests
├── server/bloom-api/                 # REST API
│   ├── src/
│   │   ├── config/                   # Prisma client, Firebase Admin, Swagger, object storage
│   │   ├── controllers/              # auth, users, habits, completions, moods, insights, friends, groups, sync, accessories
│   │   ├── routes/
│   │   ├── middleware/               # Firebase auth guard, error handler
│   │   ├── jobs/                     # dailyReminder.js, habitReminder.js (node-cron)
│   │   └── utils/                    # streak, XP, sync rules, insights, notifications, validators
│   ├── prisma/                       # Prisma schema + migrations
│   ├── seed/accessories/             # Wardrobe images
│   ├── tests/                        # Jest + Supertest tests
│   └── postman/                      # Postman collection
└── .github/workflows/                # GitHub Actions CI/CD
```

---

## 10. Getting Started

### Android App

1. Open the project root in Android Studio (JDK 17).
2. Ensure `app/google-services.json` is present (already included, configured against the shared Firebase project).
3. Set the API address in `data/network/ApiConfig.kt`: `http://10.0.2.2:3000/` for the emulator, your computer's LAN address for a physical phone (also listed in `res/xml/network_security_config.xml`), or the hosted HTTPS URL.
4. Sync Gradle, then run on a physical device or emulator (min SDK 25).

### REST API

```bash
cd server/bloom-api
npm install
cp .env.example .env        # fill in the values below
npx prisma migrate dev
npx prisma generate
npm run seed:accessories    # uploads wardrobe images and seeds the catalogue
npm run dev
```

- API runs at `http://localhost:3000`
- Swagger docs: `http://localhost:3000/api/docs`

| Variable | Purpose |
|---|---|
| `DATABASE_URL`, `DIRECT_URL` | PostgreSQL connection strings (Prisma). |
| `FIREBASE_SERVICE_ACCOUNT_JSON` or `FIREBASE_SERVICE_ACCOUNT_PATH` | Firebase Admin credentials. |
| `NEON_STORAGE_*` | Object storage endpoint, keys, bucket and public base URL. |
| `REMINDER_TIMEZONE` | Time zone for reminders (default `Africa/Johannesburg`). |
| `DISABLE_REMINDERS` | `true` turns the daily check-in reminder off. |
| `HABIT_REMINDER_INTERVAL_MINUTES` | How often the habit reminder runs, 1-59 (default 20). |
| `HABIT_REMINDER_START_HOUR`, `HABIT_REMINDER_END_HOUR` | Quiet-hours window (default 8 and 21). |
| `DISABLE_HABIT_REMINDERS` | `true` turns the habit reminder off. |
| `CORS_ORIGIN`, `PORT` | Allowed origins (default `*`) and server port (default `3000`). |

Full setup notes, environment variables and the data model are documented in `server/bloom-api/README.md`.

---

## 11. Testing Strategy and Evidence

Both halves of the codebase are covered by automated unit tests, and both are run automatically through GitHub Actions. External services (Firebase, Prisma, Retrofit) are mocked, so the suites run without a database, a network connection or real credentials.

**Android (JUnit + MockK):**

```bash
./gradlew test
```

Located in `app/src/test/`. Around 57 test cases cover the habit, mood, avatar, journal and profile repositories, the Insights ViewModel, data models and date logic, with the Retrofit layer mocked so no live network calls are made.

**REST API (Jest + Supertest):**

```bash
cd server/bloom-api
npm test
```

Located in `server/bloom-api/tests/`. Around 227 test cases across 18 files cover:

| Area | Test files |
|---|---|
| Core | `auth`, `users`, `health`, `validators` |
| Habits and moods | `habits`, `moods`, `streak` |
| Offline sync | `sync` (idempotency, last-write-wins, tombstones) |
| Social | `friends`, `sharedHabits`, `sharedStatus`, `friendHabitLogic`, `groups` |
| Notifications | `notify`, `pushTriggers`, `habitReminder` |
| Other | `insights`, `accessories` |

> The test counts above come from the test source files. Re-run both suites before submission and update the numbers if they differ.

### Test Evidence

Screenshots showing the unit test suites passing locally.

#### Android unit tests (`./gradlew test`)

**UserProfileTests**
<img width="1457" height="227" alt="image" src="https://github.com/user-attachments/assets/b0e3322a-4fc3-43c8-809f-04cfe00877cf" />

**DateTest**
<img width="1376" height="232" alt="image" src="https://github.com/user-attachments/assets/dbee1df2-e396-45ba-ba80-8b1bca3b9c6a" />

**HabitRepositoryTest**
<img width="1919" height="243" alt="HabitRepositoryTest kt Image" src="https://github.com/user-attachments/assets/da5d00e8-dd30-4062-9586-4cc91a916bc1" />

**ProfileRepositoryTest**
<img width="1919" height="237" alt="ProfileRepositoryTest kt Image" src="https://github.com/user-attachments/assets/4c239522-8146-459f-b589-edc334ade3e9" />

**AvatarRepositoryTest**
<img width="1774" height="170" alt="AvatarRepositoryTest kt Image" src="https://github.com/user-attachments/assets/980158fd-e3bd-456f-a19d-1ba377d2d30f" />

**MoodRepositoryTest**
<img width="1791" height="200" alt="MoodRepositoryTest kt Image" src="https://github.com/user-attachments/assets/66b478c0-60c0-485b-9dcb-71fefd62d9dc" />

**JournalRepositoryTest**
<img width="1800" height="218" alt="JournalRepositoryTest kt Image" src="https://github.com/user-attachments/assets/bab62e8a-a417-4976-928f-4c595c7e6f4b" />

**All Android unit tests**
<img width="763" height="130" alt="image" src="https://github.com/user-attachments/assets/45c9a525-13e9-4689-8903-19bb21e308bf" />



#### REST API unit tests (`npm test`)
<img width="431" height="129" alt="image" src="https://github.com/user-attachments/assets/73060bc1-163e-4855-8c78-7828bbb0207f" />


<img width="458" height="363" alt="image" src="https://github.com/user-attachments/assets/3b8d8bb8-5599-48e4-99bd-105b2d9d866c" />

---

## 12. DevOps and CI/CD Integration

The workflow at `.github/workflows/generate-apk-aab-debug-release.yml` runs on every push to the `release` branch and can also be triggered manually.

| Property | Value |
|---|---|
| Triggers | Push to `release`, and manual run (`workflow_dispatch`) |
| Runner | `ubuntu-latest` |
| Jobs | `test-api`, then `build` (`build` declares `needs: test-api`) |
| Outputs | Debug APK, release APK and release AAB uploaded as workflow artefacts |

**Job 1: `test-api` (backend quality gate)**

1. Check out the repository.
2. Set up Node.js 18 (matching the `engines` field in `package.json`) with npm caching.
3. Install dependencies with `npm ci`, which uses the lockfile for reproducible installs.
4. Run `npm test` (Jest). Firebase and Prisma are mocked, so no live database is required.

**Job 2: `build` (Android build and packaging)**, which only runs once `test-api` has passed.

1. Check out the repository and record the build date and repository name.
2. Set up JDK 17 (Zulu) with Gradle caching.
3. Run Android unit tests with `./gradlew test`.
4. Run `./gradlew build`.
5. Build the debug APK (`assembleDebug`) and the release APK (`assemble`).
6. Build the release App Bundle (`app:bundleRelease`).
7. Upload the debug APKs, release APKs and release AAB as separately named, date-stamped artefacts.

This means neither Android nor API code is built or packaged unless both test suites are green.

| Practice | How it is applied |
|---|---|
| Quality gates | The Android build only starts if the API tests pass, and Android unit tests run before packaging. |
| Reproducible builds | `npm ci` with a lockfile, pinned Node (18) and JDK (17), and the Gradle wrapper. |
| Build caching | Gradle and npm dependency caching shorten pipeline runs. |
| Isolated tests | External services are mocked, so tests are fast, deterministic and need no secrets. |
| Versioned artefacts | Date-stamped APK and AAB artefacts are kept on every run. |
| Pipeline as code | The workflow lives in version control, and database changes are captured as Prisma migrations. |
| Self-documenting API | OpenAPI documentation is generated from the source and served by the API itself. |
| Environment configuration | Secrets and endpoints come from environment variables, not hard-coded in the server. |
| Release branching | The `release` branch is the build trigger, so only release-ready code is packaged. |

### Proof of Workflow Success (see Actions tab)



<img width="948" height="338" alt="image" src="https://github.com/user-attachments/assets/6ed585cf-54e0-493d-83fb-3e10174a1f9e" />

---

## 13. Security Considerations

| Concern | Approach |
|---|---|
| Authentication | Firebase Authentication with Google Sign-In. The API stores no passwords and issues no tokens of its own. |
| Authorisation | Every protected route verifies the Firebase ID token. Ownership and membership checks stop users reading or changing other people's data. |
| Information leakage | Ownership failures return 404 rather than 403 so IDs cannot be probed. |
| Input validation | Server-side validators (names, mood range, UUIDs, dates, batch sizes) mirror the client rules. |
| Anti-cheat | XP, stage, streaks and unlocks are computed on the server from raw events. |
| Device security | Enforced biometric app lock with a background timeout, cleared on sign-out. |
| Private reflections | Journal entries never leave the device. |
| Transport and headers | `helmet` secure headers on the API. Cleartext HTTP is permitted only for local development addresses through the network security config. |
| Token hygiene | The FCM token is deleted on sign-out and cleared server-side when Firebase marks it invalid. |
| Secrets | Firebase service account and storage keys are read from environment variables. |

---

## 14. Multi-Language Support

The app ships with English and isiZulu string resources (`res/values/strings.xml` and `res/values-zu/strings.xml`), with all 296 strings present in both. The language can be selected from the in-app Settings screen and is applied app-wide without requiring a restart. During registration on the Login page there is also a language selector, so isiZulu speakers can register in their own language. The chosen language is saved to the user's profile so that server-sent push notifications arrive in the same language.

| English | isiZulu |
|---|---|
| *(keep your existing English Home screenshot here)* | *(keep your existing isiZulu Home screenshot here)* |

*NB: habit names remain in English because they are user input data read directly from the database.

---

## 15. Known Limitations and Future Work

- **API address:** `ApiConfig.BASE_URL` currently targets the local development server (`10.0.2.2`). It must be set to the hosted HTTPS address for a production build, after which the cleartext entries in `network_security_config.xml` can be removed.
- **Release build hardening:** code optimisation (minification) is disabled for release builds, and release packages are not signed in CI. Signing keys would need to be supplied as repository secrets for Play Store distribution.
- **CI scope:** the pipeline runs on pushes to `release` and manual runs only. It does not run on pull requests, and instrumented (on-device) tests are not part of the pipeline.
- **Offline coverage:** offline sync covers mood entries and habit ticks. Creating or editing habits, and the social features, need a connection.
- **Avatar stages:** the avatar currently has two growth stages. More stages and accessories are a natural extension.
- **Translation review:** isiZulu strings should be reviewed by a native speaker.
- **Possible enhancements:** an iOS client, richer analytics and more reminder options.

---

## 16. Design Documentation

The full Part 1 research and Planning & Design submission, including the competitive analysis, detailed requirements specification, UI/navigation mockups, endpoint specification, UML architecture diagrams, data models and project Gantt chart, is available separately as part of the POE submission.

---

## 17. YouTube Demonstration Video

Part 2: https://youtu.be/NYBgxsrIn6M

Final POE: 

---

## 18. AI and Software Disclosures

AI tool usage and third-party software usage for this project are disclosed in full in the Part 1 Planning & Design document (AI Disclosures and Software Disclosures sections). AI assistance used specifically during development is summarised below, per the POE brief.

### Disclosures

**Title:** Emojis not displaying
- **Section:** PROG7314 Part 2
- **AI Tool Used:** Claude
- **Purpose:** Figure out why emojis don't display
- **Date:** 19 September 2026
- **Link:** https://claude.ai/share/94acac09-f81e-4ccd-b406-0a3489cefe2d

**Title:** Retrofit
- **Section:** PROG7314 Part 2
- **AI Tool Used:** Claude
- **Purpose:** Understanding Retrofit client for Android
- **Date:** 18 September 2026
- **Link:** https://claude.ai/share/d05128ae-df2a-461f-94ad-c639a1ed9c1d

**Title:** Initial Setup
- **Section:** PROG7314 Part 2
- **AI Tool Used:** Claude
- **Purpose:** Initial Application Setup
- **Date:** 21 September 2026
- **Link:** https://claude.ai/share/a13669c2-04c9-4631-a817-ea1c757a1d4e


