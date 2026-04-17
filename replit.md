# Workspace

## Overview

pnpm workspace monorepo using TypeScript. Each package manages its own dependencies.

## Stack

- **Monorepo tool**: pnpm workspaces
- **Node.js version**: 24
- **Package manager**: pnpm
- **TypeScript version**: 5.9
- **API framework**: Express 5
- **Database**: PostgreSQL + Drizzle ORM
- **Validation**: Zod (`zod/v4`), `drizzle-zod`
- **API codegen**: Orval (from OpenAPI spec)
- **Build**: esbuild (CJS bundle)

## Structure

```text
artifacts-monorepo/
├── artifacts/              # Deployable applications
│   ├── api-server/         # Express API server
│   └── mobile/             # FocusGuard Expo mobile app
├── lib/                    # Shared libraries
│   ├── api-spec/           # OpenAPI spec + Orval codegen config
│   ├── api-client-react/   # Generated React Query hooks
│   ├── api-zod/            # Generated Zod schemas from OpenAPI
│   └── db/                 # Drizzle ORM schema + DB connection
├── scripts/                # Utility scripts (single workspace package)
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── tsconfig.json
└── package.json
```

## FocusGuard Mobile App (`artifacts/mobile`)

A full-featured screen-time blocker app built with Expo + Supabase.

### Tech Stack
- **Framework**: Expo (SDK 54) + Expo Router (file-based routing)
- **Auth**: Supabase (email/password)
- **State**: Zustand + AsyncStorage
- **UI**: React Native (standard Animated API — no reanimated), @expo/vector-icons
- **Theme**: Dark mode — charcoal background (#0E0F11), electric teal accent (#00D4AA)

### Native Module (`modules/app-tracking/` → `@focusguard/app-tracking`)

Local Expo native module (expo-modules-core v3, `expo-module.config.json` for autolinking). Provides:
- `getInstalledApps()` — PackageManager list of all user-launchable apps
- `getForegroundApp()` — UsageStatsManager query for current foreground package
- `startMonitoring(packageNames[])` — starts `ForegroundAppService` with monitored list
- `stopMonitoring()` / `updateMonitoredApps(packageNames[])` — service lifecycle
- `showOverlay(packageName)` / `hideOverlay()` — WindowManager TYPE_APPLICATION_OVERLAY system overlay
- `hasOverlayPermission()` — checks SYSTEM_ALERT_WINDOW permission

**Kotlin files:**
- `AppTrackingModule.kt` — Expo module definition
- `ForegroundAppService.kt` — Android foreground service, polls UsageStatsManager every 1 000 ms, shows/hides overlay
- `AppOverlayManager.kt` — singleton, drives WindowManager overlay on main thread

**Config plugin:** `plugins/withAppTrackingManifest.js` registers the service + `FOREGROUND_SERVICE_DATA_SYNC` permission in AndroidManifest.xml during EAS prebuild.

### App Monitoring JS Layer
- `store/monitoredAppsStore.ts` — Zustand store; persists selected apps to AsyncStorage (`@focusguard_monitored_apps_v1`); starts/stops/syncs native service automatically
- `lib/AppTrackingService.ts` — JS-side 1 000 ms poller via `setInterval` + AppState listener; exposes `addListener(fn)` for in-app overlay reactions
- `components/AppPickerModal.tsx` — searchable modal; calls `getInstalledApps()` on Android (real device apps), falls back to curated list in Expo Go / iOS

### Environment Variables (Secrets)
- `EXPO_PUBLIC_SUPABASE_URL` — Supabase project URL
- `EXPO_PUBLIC_SUPABASE_ANON_KEY` — Supabase anon/public key

### Features Built

**Phase 1 — Auth**
- Sign In / Sign Up with email+password (8+ char validation)
- Mock "Continue with Google" button
- Session persistence via AsyncStorage + Supabase client

**Phase 2 — Onboarding** (`app/onboarding.tsx`)
- 3-screen horizontal scroll: Intro, Challenge preview, Permissions
- Dot pagination indicators
- Skip option; sets `hasCompletedOnboarding` in AsyncStorage

**Phase 3 — Hard-Mode Blocker** (`components/BlockOverlay.tsx`)
- Full-screen modal overlay
- 120-character unlock paragraph with real-time validation
- Any typo resets progress (no copy/paste: `contextMenuHidden`)
- Progress bar + character counter
- On success: increments `wallOfShameCount` in AsyncStorage

**Phase 4 — 4-Tab UI**
- `(tabs)/index.tsx` — Dashboard: stats, focus mode toggle, app usage bars, Wall of Shame
- `(tabs)/settings.tsx` — Managed Apps: expansion tile per app with lock duration, unlock goals, time limit pickers
- `(tabs)/focus-zones.tsx` — Focus Zones CRUD: create/edit/delete time range schedules with day picker and color picker; inline delete confirmation (no Alert.alert)
- `(tabs)/challenges.tsx` — Physical Challenges tab: push-up/squat/custom exercise sessions with 3-2-1 countdown, SVG progress ring, PoseTracker (web: TF.js MoveNet auto-detection; native: tap-to-count), and completion screen that grants a 10-minute unlock window via `unlockFocus()` in the dashboard store

**Phase 5 — Full Challenge Engine + Banked Minutes**
- `store/bankedMinutesStore.ts` — Zustand store: `bankedMinutes`, `addMinutes()`, `grantAccess()` (converts all banked minutes to a timed session), `tick()`, `hasAccess()`, `remainingSeconds()`, `loadFromStorage()`; persisted in AsyncStorage
- `(tabs)/challenges.tsx` — Complete rewrite with 10 phases: select → buildYourOwn → difficulty → customRatio → duration → guideWelcome → guidePosition → countdown → session → complete. Challenge labels: "Pushup to Scroll", "Squat to Scroll", "Build Your Own". Difficulty presets: Easy (1 rep=3 min), Medium (1 rep=1 min), Hard (3 reps=1 min), Athlete (10 reps=1 min), Custom (dual-incrementer). Circuit mode for both exercises. Earned minutes added to banked store on complete.
- `components/BlockOverlay.tsx` — Shows "Use X banked minutes" prominent button; auto-dismisses if session already active; keep "Earn minutes instead" shortcut link to Challenges tab; removed challenge-polling in favour of banked-minutes flow
- `components/PoseTracker.tsx` — Web: TF.js WebGL + getUserMedia + MoveNet Lightning + AudioContext beep; Native: expo-camera CameraView → `takePictureAsync` snapshot every 250ms → `decodeJpeg` (@tensorflow/tfjs-react-native) → MoveNet Lightning inference → SVG skeleton overlay + expo-av beep on every rep; falls back to manual tap-to-count if camera/model fails

**Phase 6 — Managed Apps Overhaul**
- `(tabs)/settings.tsx` — Trash icon + inline cancel/confirm delete per app row; `deleteApp()` handler; `AddAppModal` with 15-app catalogue (Snapchat, Pinterest, LinkedIn, Discord, Twitch, WhatsApp, Netflix, Spotify, Telegram, etc.) filterd to exclude already-managed apps; "+ Add an App" dashed-border button at list bottom

### Navigation Flow
```
index.tsx (router)
  ├── /onboarding   (first launch only, AsyncStorage flag)
  ├── /auth         (no session)
  └── /(tabs)       (authenticated)
        ├── index         (Dashboard)
        ├── settings      (Managed Apps)
        ├── focus-zones   (Focus Zones)
        └── challenges    (Physical Challenges)
```

### Key Files
- `lib/supabase.ts` — Supabase client (AsyncStorage session)
- `context/AuthContext.tsx` — Auth state provider + hooks
- `constants/colors.ts` — Full dark-mode color palette
- `store/dashboardStore.ts` — Zustand persisted store (usage stats, focus mode, challenge unlock state)

---

## Packages

### `artifacts/api-server` (`@workspace/api-server`)
Express 5 API server. Routes live in `src/routes/`.

### `lib/db` (`@workspace/db`)
Database layer using Drizzle ORM with PostgreSQL.

### `lib/api-spec` (`@workspace/api-spec`)
Owns the OpenAPI 3.1 spec and Orval codegen config.
Run codegen: `pnpm --filter @workspace/api-spec run codegen`

### `lib/api-zod` (`@workspace/api-zod`)
Generated Zod schemas from the OpenAPI spec.

### `lib/api-client-react` (`@workspace/api-client-react`)
Generated React Query hooks and fetch client.

### `scripts` (`@workspace/scripts`)
Utility scripts. Run via `pnpm --filter @workspace/scripts run <script>`.
