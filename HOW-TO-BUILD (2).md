# Reader — Build & Install Guide

A clean, offline text-to-speech reader for Android 8.0+ (API 26+).

Built from scratch with original code. No ads, no trackers, no analytics, no
account, no network access. Cloned in spirit and screen-flow from the
@Voice Aloud Reader, with none of the bloatware.

## Deliverables

This folder ships two APKs and the source tree:

| File | Size | Signing | Use |
|------|------|---------|-----|
| `Reader-debug.apk` | ~26 MB | debug keystore | First test — installs as `app.reader.debug` so it can coexist with the release variant |
| `Reader-release.apk` | ~7.8 MB | release keystore (bundled) | Production install — installs as `app.reader`, well under the 15 MB cap |
| `reader-app/` | — | — | Full source tree (Gradle project) |
| `reader-release.keystore` | 2.6 KB | — | Self-signed keystore for the release variant. **Replace with your own before any public release.** |

> Why is the debug APK so much bigger? It skips R8 shrinking, keeps all
> Compose runtime checks, and bundles every PDFBox class. The release APK
> runs R8 + resource shrinking and strips everything unused — landing at
> 7.8 MB.

## Install

```bash
# Debug build (separate app id — safe to install alongside release)
adb install -r /home/z/my-project/download/Reader-debug.apk

# Release build (the "real" install)
adb install -r /home/z/my-project/download/Reader-release.apk
```

On Android 13+ the app will ask for `POST_NOTIFICATIONS` permission the first
time you press Play — that's the only permission prompt. Grant it so the
playback notification with transport controls can show on the lock screen.

## Test on the client device

Target: Samsung SM-A145M, Android 14.

1. **Airplane mode** — turn it on, then open the app, open a file, tap Play.
   Every core feature should work. If anything calls out to the network with
   airplane mode on, that's a bug.
2. **Screen off** — press Play, then power-off the screen. Audio should keep
   playing. Lock screen should show transport buttons.
3. **Resume** — kill the app from Recents, reopen it. The last document
   should reopen at the last sentence.
4. **Rotation / theme change** — rotate the device, switch system theme light
   → dark. App should not crash; the highlighted sentence should stay
   highlighted.
5. **Share-from-another-app** — in any app that has text (browser, email),
   select text → Share → Reader. Reader should land on the Paste screen with
   the text pre-filled.
6. **File picker** — tap the floating + button on the Library screen, pick
   a `.txt` / `.html` / `.epub` / `.pdf` / `.docx` file. It opens in the
   reader.

## Build from source

Prerequisites:
- JDK 21 (`java` and `javac` on PATH)
- Android SDK with platform 34 + build-tools 34.0.0
- ~2 GB free RAM for the build (Gradle + Kotlin daemon)

Steps:

```bash
cd reader-app

# Set up the Android SDK path (already configured in local.properties)
# Edit local.properties if your SDK lives elsewhere.

# Build the debug APK
./gradlew :app:assembleDebug

# Build the release APK (uses the bundled keystore)
./gradlew :app:assembleRelease

# Output paths:
#   app/build/outputs/apk/debug/app-debug.apk
#   app/build/outputs/apk/release/app-release.apk
```

To sign with your own release key instead of the bundled one:

1. Generate a keystore:
   ```bash
   keytool -genkeypair -alias reader -keyalg RSA -keysize 2048 -validity 3650 \
     -keystore my-release.keystore \
     -storepass <your-store-pass> -keypass <your-key-pass> \
     -dname "CN=Reader,O=YourOrg,C=US"
   ```
2. Edit `app/build.gradle.kts` → `signingConfigs.release` → point `storeFile`
   at your keystore and fill in the passwords. Or, better, move them into
   environment variables / `~/.gradle/gradle.properties` so they don't end up
   in version control.

## Architecture notes

- **UI**: Jetpack Compose + Material 3 with dynamic color (Android 12+),
  light/dark/system theme.
- **Storage**: Room 2.6.1 for documents / bookmarks / queue. DataStore for
  user preferences. Documents opened via the system file picker are copied
  into app-private storage so bookmarks survive the source file moving.
- **TTS**: Android platform `TextToSpeech`. Sentence splitting uses
  `java.text.BreakIterator` (locale-aware). No cloud TTS — fully offline.
- **Playback**: `PlaybackService` is a foreground service of type
  `mediaPlayback` with a `MediaSessionCompat` for lock-screen transport
  controls. Notification uses `MediaStyle` for compact-transport layout.
- **Parsers**: plain text (built-in), HTML (jsoup), EPUB (in-house zip+xml,
  no external dep), PDF (TomRoush PDFBox-Android 2.0.27.0 — native ARM64),
  DOCX (in-house OOXML walker via platform `XmlPullParser`, no Apache POI).
- **Package**: `app.reader` (release) / `app.reader.debug` (debug).
- **Min SDK**: 26 (Android 8.0). **Target SDK**: 34 (Android 14).
- **Permissions**: only `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_MEDIA_PLAYBACK`,
  `WAKE_LOCK`, `POST_NOTIFICATIONS`. Nothing else.

## What this app does NOT do

(Per the README brief — these are deliberate omissions, not missing features.)

- No ads, no analytics, no crash reporters.
- No network access. The `AndroidManifest.xml` does not declare
  `INTERNET` permission. Airplane-mode safe by construction.
- No account, sign-in, or onboarding wizard.
- No "rate us", "what's new", EULA, ToS, privacy-policy, cookie-notice, or
  newsletter prompts.
- No premium / subscribe / paywall / restore-purchase flows.
- No anti-tamper / root / debugger / signature-verification checks. The app
  does not defend itself against its own user.
- No bundled third-party SDKs for things the platform already provides.
- No background work beyond the playback service. No periodic network pings,
  no usage analytics, no install-referrer hooks.

## What's not yet built (v2 roadmap, per README)

The README's v1 scope is what's shipped. Later items, deferred until the
client asks for them:

- Cloud TTS providers (each behind an explicit opt-in in settings).
- Offline dictionary lookup.
- OCR for image-only PDFs.
- Per-voice defaults and a per-document voice override.
- A speech editor for hand-editing a passage before reading.

## Source tree

```
reader-app/
├── build.gradle.kts              # Root Gradle config
├── settings.gradle.kts
├── gradle.properties            # Memory + Java home settings
├── local.properties             # SDK path — edit for your machine
├── gradlew, gradlew.bat         # Wrapper
├── reader-release.keystore      # Bundled release keystore (REPLACE!)
└── app/
    ├── build.gradle.kts         # Dependencies + signing configs
    ├── proguard-rules.pro
    └── src/main/
        ├── AndroidManifest.xml  # No INTERNET permission, no extra perms
        ├── res/
        │   ├── values/strings.xml
        │   ├── values/colors.xml, themes.xml
        │   ├── values-night/themes.xml
        │   ├── drawable/         # Vector icons (notification + app launcher)
        │   ├── mipmap-anydpi-v26/ # Adaptive launcher icon
        │   └── xml/              # file_paths, locales, backup, data extraction
        └── java/app/reader/
            ├── ReaderApp.kt       # Application — inits PDFBox + channel
            ├── MainActivity.kt    # Single-activity host, intent router
            ├── data/
            │   ├── db/             # Room entities, DAOs, database
            │   └── repo/           # DocumentRepository, SettingsRepository, IntakeBus
            ├── parser/             # txt, html, epub, pdf, docx, sentence splitter
            ├── tts/                # TtsController, PlaybackState, LoadedDoc
            ├── service/
            │   └── PlaybackService.kt   # Foreground + MediaSession
            └── ui/
                ├── theme/          # Material 3 dynamic color + typography
                ├── nav/            # Destinations + NavGraph
                ├── library/        # Library screen + VM
                ├── reader/         # Reader screen + VM + PlaybackBar
                ├── bookmarks/      # (sheet content lives inside reader/)
                ├── queue/          # Reading list screen + VM
                ├── settings/       # Settings screen + VM
                └── paste/          # Paste-text screen
```

## Contact / License

Personal-use build. No license granted — this is for the client's own device.
Replace the bundled keystore before any redistribution.
