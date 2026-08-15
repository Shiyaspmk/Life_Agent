# Life Agent — MVP (Android home-screen widget)

On-device "life agent" widget: shows your next calendar event and a digest of
recent notifications, with a couple of suggested quick replies for the most
important one. Everything runs locally — no backend, no network calls, no
API keys.

This is idea #1 from the trending-app list: an on-device AI agent widget,
built as a working MVP skeleton you can open directly in Android Studio.

## What's actually "AI" here vs. what's a stub

To keep the MVP zero-setup and runnable on any Android 8+ phone, the
"agent brain" (`agent/SimpleAgentEngine.kt`) is a deterministic, rule-based
implementation — keyword matching and recency scoring, not a model. It sits
behind the `AgentEngine` interface (`agent/AgentEngine.kt`) specifically so
you can swap in real generative AI later without touching the widget,
notification pipeline, or calendar code:

- **ML Kit GenAI APIs** (Summarization / Rewriting) — on-device, Gemini
  Nano-backed, no internet required. Easiest official upgrade path.
- **AICore** directly, for lower-level Gemini Nano prompt access.
- **MediaPipe LLM Inference API** with a bundled small model (e.g. Gemma 2B)
  if you need it to run on devices without AICore/Gemini Nano support
  (currently Pixel 8+/recent Galaxy S-series for on-device Gemini Nano).

To swap: implement `AgentEngine` with your real model calls in a new class,
then point `AgentEngineProvider.engine` at it.

## How it's built

- **Kotlin + Jetpack Compose** for the companion app screen (permissions +
  live preview).
- **Jetpack Glance** for the actual home-screen widget (`widget/`).
- **NotificationListenerService** (`notifications/`) captures notification
  title/text on-device and stores the last 30 in SharedPreferences as JSON
  (swap for Room if you outgrow it — nothing else needs to change).
- **CalendarContract** query (`calendar/`) for the next event in the next 24h,
  read-only.

```
app/src/main/java/com/example/lifeagent/
  MainActivity.kt              companion screen: grant permissions, preview digest
  agent/AgentEngine.kt          pluggable interface for the "brain"
  agent/SimpleAgentEngine.kt    MVP rule-based implementation
  notifications/                capture + store notifications
  calendar/CalendarRepository.kt   read next upcoming event
  widget/LifeAgentWidget.kt     the actual home-screen widget UI (Glance)
  widget/LifeAgentWidgetReceiver.kt
  widget/WidgetUpdater.kt       triggers a refresh after new data arrives
```

## Running it

1. Open the `LifeAgentWidget/` folder in Android Studio (Koala/2024.1+
   recommended). This project doesn't ship a Gradle wrapper (it was built in
   a sandbox without network access to Gradle's distribution server) —
   Android Studio will offer to generate one automatically on first open; say
   yes, then let it sync.
2. Run the app on a device or emulator (API 26+).
3. In the app: tap **"Open notification access settings"** and enable
   *Life Agent* — this can't be requested as a normal runtime permission,
   only granted manually in system settings.
4. Tap **"Grant calendar permission"** for the normal runtime prompt.
5. Long-press your home screen → **Widgets** → **Life Agent** → drag it on.
6. Trigger a notification (e.g. send yourself a message) and watch the
   widget's digest update, or tap **Refresh** in the app to force it.

## Known MVP limitations (by design, not oversights)

- Reply "drafting" is template-based, not generative — see above for the
  real-AI upgrade path.
- Tapping a suggested reply currently just refreshes the widget rather than
  opening the source app's reply intent — wiring that up is app-specific
  (each messaging app exposes reply actions differently via
  `Notification.Action.RemoteInput`, which is the next thing to build).
- No onboarding polish, no icon, no dark/light theming beyond the fixed
  widget palette — cosmetic, fast to add once the core flow is validated.
- Notification data lives only in SharedPreferences on-device; nothing is
  synced or backed up, which is intentional for privacy but means it clears
  on uninstall/app-data-clear.

## Permissions this app requests, and why

| Permission | Why | How it's granted |
|---|---|---|
| Notification access | Read notification title/text to build the digest and reply drafts | Manual toggle in system settings (deep-linked from the app) |
| `READ_CALENDAR` | Look up your next event | Standard runtime permission prompt |
