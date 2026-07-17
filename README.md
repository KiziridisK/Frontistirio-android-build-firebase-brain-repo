# Frontistirio — Android Build + Firebase Push Runbook

> **Procedural reference** for wrapping the Ionic/Angular web app (`Frontistirio`) as a **native Android app** with **Capacitor**: running it on an **emulator** and a **physical device**, wiring **Firebase Cloud Messaging (FCM)** push, **signing a release APK**, and **versioning** the app. Written while doing it for the [announcements feature](https://github.com/KiziridisK/Frontistirio-announcements-brain-repo) (2026-07-02). Use it whenever repeating similar native-build / push / release work. Every gotcha here is one we actually hit.

**Stack:** Ionic 8 / Angular 18 / Capacitor 6 / Firebase Admin 13. **Host:** Windows. App id: `com.frontistirio.app`.

---

## TL;DR — the order that works

```
# one-time
npm i @capacitor/core @capacitor/cli @capacitor/android @capacitor/ios @capacitor/push-notifications
# create capacitor.config.ts (appId, webDir: 'www', server.androidScheme)
npm run build:ionic                 # or: ng build --configuration <cfg>   → produces www/
npx cap add android                 # generates android/

# Firebase (console): create project → add Android app (package = appId) → download google-services.json
cp google-services.json android/app/
npx cap sync android                # copies web assets + plugins; auto-applies google-services plugin

# backend send side
cd Frontistirio-API && npm i firebase-admin
# console: Project Settings → Service accounts → Generate private key → save as serviceAccountKey.json (gitignore!)
# .env: FIREBASE_SERVICE_ACCOUNT_PATH=serviceAccountKey.json  → restart backend

# run
npx cap open android                # Android Studio → wait for Gradle sync → create AVD (Google Play image) → ▶ Run
```

**After ANY web-code change:** `ng build …` → `npx cap sync android` → ▶ Run. (A native rebuild alone does **not** pick up web changes.)

---

## Prerequisites
- **Android Studio** (bundles the Android SDK + a JDK + emulator + Device Manager). Needed to build/run the APK; the CLI-only SDK route is possible but painful.
- **Node** + the existing Ionic/Angular project.
- For **iOS**: a **Mac** + Apple Developer account ($99/yr) — not possible on Windows.

---

## 1. Capacitor setup (one-time)
1. Install: `@capacitor/{core,cli,android,ios,push-notifications}@^6` (Capacitor 6 pairs with Ionic 8).
2. `capacitor.config.ts`:
   ```ts
   const config: CapacitorConfig = {
     appId: 'com.frontistirio.app',      // = Play Store identity; hard to change later
     appName: 'Frontistirio',
     webDir: 'www',                      // Ionic/Angular build output
     server: { androidScheme: 'http' },  // dev only — see Mixed content gotcha; use 'https' for prod
     plugins: { PushNotifications: { presentationOptions: ['badge','sound','alert'] } },
   };
   ```
3. Build the web once (`www/` must exist before `cap add`): `npm run build:ionic` (prod) or `ng build --configuration development`.
4. `npx cap add android` → generates the `android/` Gradle project.

> In **this** project `android/` (and `ios/`) are **gitignored** (large + regenerable). After a fresh `cap add` you must re-apply the native customizations: drop in `google-services.json` (§2), add `usesCleartextTraffic` if you need the http emulator (§5.3), and the package.json-driven version logic in `android/app/build.gradle` (§10). *(Alternative: commit `android/` — its nested `.gitignore` already excludes build junk. Your call.)*

---

## 2. Firebase project + Android app
1. `console.firebase.google.com` → **Create project** (ours: `frontistirio-dev`).
2. **Add app → Android**. The **"Android package name"** field = the Capacitor `appId` **exactly** (`com.frontistirio.app`). SHA-1 not needed for push.
3. Download **`google-services.json`** → put in **`android/app/`**.
4. **Gradle wiring — Capacitor 6 does it automatically**, no manual edits:
   - root `android/build.gradle` already has `classpath 'com.google.gms:google-services:4.4.x'`.
   - `android/app/build.gradle` has a conditional that **applies the plugin only if `google-services.json` exists**:
     ```gradle
     try { def f = file('google-services.json'); if (f.text) { apply plugin: 'com.google.gms.google-services' } } catch(e){ … }
     ```
   *(The old manual "add classpath + apply plugin" instructions are for pre-6 Capacitor.)*
5. `npx cap sync android` — copies web assets + updates plugins (should list `@capacitor/push-notifications`).

---

## 3. Backend send side (`firebase-admin`)
1. `cd Frontistirio-API && npm i firebase-admin`.
2. Firebase Console → ⚙️ **Project Settings → Service accounts → Generate new private key** → downloads a JSON. **It's a secret** — save as `serviceAccountKey.json` and `.gitignore` it.
3. `.env`: `FIREBASE_SERVICE_ACCOUNT_PATH=serviceAccountKey.json` (our loader also accepts `FIREBASE_SERVICE_ACCOUNT` = full JSON, or the individual `FIREBASE_*` fields, or `GOOGLE_APPLICATION_CREDENTIALS`).
4. **Restart the backend manually** (nodemon does **not** watch `.env`). Success log: `🔔 Firebase Admin initialized — push notifications enabled`.
5. Sanity-check without a device: `node scripts/sendTestPush.js <fcmToken>` sends one push straight through FCM (isolates "are the creds right?").

---

## 4. Build & run on the emulator
1. `npx cap open android` → opens Android Studio. **Wait for the first Gradle sync** (slow; accept SDK-component prompts).
2. **Create an emulator**: Device Manager → Create Device → pick a phone → **system image with "Google Play"** (required — FCM needs Google Play services) → download → Finish → ▶ start it.
3. Select the running emulator in the top dropdown → **▶ Run**.
4. `"No target device found"` = no AVD running / no device connected → create or start one first.
5. **Windows**: if the emulator won't boot, enable **Windows Hypervisor Platform** + **Virtual Machine Platform** in Windows Features, reboot.

---

## 5. ⚠️ The gotchas (why this runbook exists)

### 5.1 Which environment config the native build bakes in
The native app bundles **whatever `www/` you last built**. This project has **three** envs so you never hand-edit URLs:
- `environment.ts` → `localhost:3000` — **browser dev** (`ng serve`).
- `environment.emulator.ts` → `10.0.2.2:3000` — **emulator** builds (`ng build --configuration emulator`).
- `environment.prod.ts` → deployed API — **production** (`npm run build:ionic`).

**Shortcut:** `npm run cap:emulator` = `ng build --configuration emulator && cap sync android` (build with the emulator URL + sync in one step) → then ▶ Run in Android Studio. So `environment.ts` stays on `localhost` permanently.

### 5.2 `localhost` from the emulator ≠ your PC
Inside the emulator, `localhost` is the emulator itself. Reach the host at **`http://10.0.2.2:3000`** (standard AVD alias). Physical device → the PC's **LAN IP** (same network, firewall open).

### 5.3 Cleartext HTTP is blocked
Android (API 28+) blocks plain `http://` by default. Add to `android/app/src/main/AndroidManifest.xml` `<application>`:
```xml
android:usesCleartextTraffic="true"
```
(Fine for dev; the release build talks https anyway.)

### 5.4 **Mixed content — the one that cost us the most**
Capacitor 6 default `androidScheme: 'https'` serves the app from `https://localhost`. An `http://` API call is then **mixed content** → the WebView **blocks it silently**: the request never leaves the device, so **nothing appears in the server logs**.
- **Symptom:** the emulator's *browser* CAN reach `http://10.0.2.2:3000/` ("Cannot GET /"), but the *app* "login failed" with **zero** backend logs.
- **Fix:** `server.androidScheme: 'http'` in `capacitor.config.ts` → `npx cap sync android` → rebuild. (Both app and API are now http → no mixed content.) **Revert to `https` for production** (with an https API).

### 5.5 CORS for the native WebView origin
With `androidScheme: 'http'` the WebView origin is `http://localhost` (https scheme → `https://localhost`; iOS → `capacitor://localhost`). Add all three to the backend CORS allowlist (both the express `allowedOrigins` and the socket.io `cors.origin`):
```
"http://localhost", "https://localhost", "capacitor://localhost"
```
*(Note: a CORS rejection still reaches the server — you'd see the OPTIONS in logs. If there are NO logs at all, it's mixed content / cleartext / wrong host, not CORS.)*

### 5.6 Rebuild + sync after web changes
`ng build …` → `npx cap sync android` → ▶ Run. cap sync copies `www/` into `android/app/src/main/assets/public`; the APK bundles it at build time.

### 5.7 Diagnostics when it "just fails"
- **chrome://inspect** often opens a **blank** DevTools (proxy / Chromium-version mismatch). Don't rabbit-hole.
- **Bisect network vs app:** open `http://10.0.2.2:3000/` in the **emulator's browser**. A response (e.g. `Cannot GET /`) = network + backend fine → the problem is *inside the app* (mixed content / cleartext / env URL). "Can't reach" = networking.
- **Logcat** (Android Studio, bottom): filter `package:com.frontistirio.app`, press the action, look for `Cleartext …`, `net::ERR_…`.
- **Host checks:** `netstat -ano | grep :3000` (listening?), `curl -i http://localhost:3000/` (up?). Backend must bind `0.0.0.0` for `10.0.2.2` to reach it (ours does).

### 5.8 Emulator keyboard not typing
Common glitch. Fixes: click the field first then type on the **PC keyboard**; **Cold Boot Now** (Device Manager ⋮); or AVD → Edit → Advanced → **Enable keyboard input**.

### 5.9 Other
- Emulator **must use a Google Play (or Google APIs) system image** — FCM needs Google Play services.
- **nodemon doesn't watch `.env`** → restart manually after editing it.
- Android 13+ needs the **`POST_NOTIFICATIONS`** runtime permission — the push plugin declares it and `PushNotifications.requestPermissions()` prompts for it (handled in `push.service.ts`).

### 5.10 The **PROD** backend must allow the native origin too — "login works on web, fails on the installed APK"
Same class as 5.5 but on **production** — the one that bit us on the real device. The native WebView origin is `https://localhost`; if the **deployed** backend's express `allowedOrigins` doesn't include it, the request is rejected — even though the web app (CloudFront origin) logs in fine against the **same backend + DB**.
- **Symptom:** web-prod login OK; the installed release APK shows the **generic** "Login failed".
- 🔑 **Which error message = which layer** (login handler is `this.loginError = err.error?.message || 'Login failed'`):
  - You see a **backend message** ("Invalid credentials") → the request **reached** the server → real 401 (wrong creds — or the **mobile keyboard auto-capitalized the email** / added a space).
  - You see the **generic "Login failed"** → **network/CORS** failure, no JSON body: blocked origin, mixed content, wrong host, or timeout.
- **Bisect with curl** — reproduce exactly what the WebView sends, comparing origins:
  ```bash
  API=https://…execute-api…/Prod
  curl -i -X POST $API/users/login -H "Origin: https://d3tf4mtxs2b8zy.cloudfront.net" -H "Content-Type: application/json" -d '{}'
  #   → 401 + Access-Control-Allow-Origin: https://…cloudfront.net   (works — web)
  curl -i -X POST $API/users/login -H "Origin: https://localhost" -H "Content-Type: application/json" -d '{}'
  #   → 500, NO Access-Control-Allow-Origin header                   (blocked — device ← the bug)
  ```
- ⚠️ The **OPTIONS preflight** may be answered by **API Gateway** with `Access-Control-Allow-Origin: *` and *look* fine — always test the **actual POST**, whose CORS header is set by **express on EC2**. (Our express cors `callback(new Error("Not allowed by CORS"))` → **500 with no CORS header** for a disallowed origin.)
- **Fix:** add `http://localhost`, `https://localhost`, `capacitor://localhost` to the **DEPLOYED** `index.js` `allowedOrigins` (+ socket.io `cors.origin`), then `pm2 restart <app>`. **No APK rebuild needed** — only the backend.

---

## 6. Diagnostic flow (the decision tree we used)
```
"login failed" on the app
        │
        ▼
WHICH message?  (handler = err.error?.message || 'Login failed')
   ├─ backend text ("Invalid credentials")  → request REACHED the server → real 401
   │        → wrong creds, or the MOBILE KEYBOARD auto-capitalized the email / added a space
   │
   └─ GENERIC "Login failed"  → network/CORS, no response body:
            │
            ▼
        does the request show in the backend logs?
          │
   ┌──────┴───────────────────────────────────────────┐
   NO (never reaches)                                  reaches but 500/blocked
   │  EMULATOR + local backend:                        │  DEVICE/PROD:
   │   • backend up + reachable from host?             │   • curl the POST with the two Origins (5.10):
   │     (netstat :3000 / curl localhost:3000)         │     cloudfront → ACAO header (ok) ; localhost → 500/no header (bug)
   │   • emulator browser → http://10.0.2.2:3000/ ?    │   • fix: add http/https/capacitor://localhost to the
   │     "Cannot GET /" = reachable → issue is IN-APP: │     DEPLOYED index.js allowedOrigins (+ socket cors) → pm2 restart
   │       - env URL baked in (10.0.2.2 not prod)  → cap:emulator
   │       - mixed content (androidScheme https→http) → androidScheme 'http'   ← emulator culprit
   │       - cleartext (usesCleartextTraffic)        → manifest
   └───────────────────────────────────────────────────┘
```
> The two culprits that actually bit us: **mixed content** on the emulator (5.4) and the **prod backend not allowing `https://localhost`** on the device (5.10).

---

## 7. End-to-end push test
1. Backend restarted with creds → `🔔 Firebase Admin initialized`.
2. **Device (emulator)** logged in (as a student, built with `10.0.2.2`) → on login it registers its FCM token → row in `DeviceToken` (confirm in DB / logs). Allow the notifications prompt.
3. **Sender (browser, env=localhost)** logged in as a **store-user** → compose an announcement targeting that student → send.
4. Emulator receives the push: **foreground** → in-app toast; **background/killed** (press Home first) → status-bar notification. Backend: `dispatchAnnouncement → pushDispatcher → FCM`, recipient `sent`.

> Tip: the **sender uses `localhost`** (browser) and the **device uses `10.0.2.2`** (baked into its build) — both hit the same backend. Keep `environment.ts` on `localhost` for browser dev; only flip to `10.0.2.2` when producing an emulator build.

---

## 8. Emulator vs production build (automated)
`androidScheme` differs (http for the local-backend emulator, https for prod) but it's a single Capacitor value — so it's driven by an env var instead of hand-editing:
```ts
// capacitor.config.ts
androidScheme: process.env.CAP_ANDROID_SCHEME === 'http' ? 'http' : 'https'   // default https
```
```jsonc
// package.json scripts (needs cross-env)
"cap:emulator": "ng build --configuration emulator && cross-env CAP_ANDROID_SCHEME=http cap sync android",
"cap:prod":     "ng build --configuration production && cap sync android"      // https + prod API
```
- **Device hitting the prod API Gateway** (`https://…execute-api…/Prod`, already in `environment.prod.ts`): `npm run cap:prod` → ▶ Run with the phone connected (USB debugging), or Build → Build APK(s) → sideload. https everywhere → **no cleartext/mixed-content issues** (they were only needed for the local http backend).
- **Deploy the backend** to EC2/prod too: the CORS Capacitor origins + `firebase-admin` + service account must be on the server, or prod push/requests fail.
- Cleartext (`usesCleartextTraffic`) can be dropped for release (or kept debug-only). Never commit `serviceAccountKey.json` (`.gitignore`d).

---

## 9. Production release (signed) APK

**Prereq — deploy the backend first:** the prod server must have the native CORS origins (§5.10) + `firebase-admin` + `serviceAccountKey.json`, or login/push fail on the device.

### 9.0 CLI flow (current — use this)

The GUI wizard below (9.1) is kept for reference only; signing is automated via Gradle now.
**`npm run cap:prod` does NOT build an APK** — it only does `ng build` + `cap sync android`. The APK
comes from `assembleRelease`. **Bump the version BEFORE `cap:prod`** (it runs `sync:version`; bumping
after leaves the android project on the old version).

```powershell
npm run version:minor          # or :patch — BEFORE cap:prod
npm run cap:prod
cd android
$env:JAVA_HOME = "C:\Program Files\Android\Android Studio\jbr"   # no java on PATH; unset by default
.\gradlew.bat assembleRelease
```

Output: **`android/app/build/outputs/apk/release/app-release.apk`** (note: NOT `android/app/release/`,
which is where the old GUI wizard put it). Verify before installing — a stale APK from a previous
build sits at that path and is easy to grab by mistake:

```bash
aapt2 dump badging app-release.apk | grep ^package:     # versionCode/versionName
apksigner verify --print-certs app-release.apk          # expect CN=Konstantinos Kiziridis, O=Logeion
unzip -p app-release.apk assets/capacitor.config.json   # plugins actually compiled in
```
Plugin classes live inside `classes*.dex` — `unzip -l | grep <plugin>` always returns 0 and proves
nothing. Check `capacitor.config.json`, or grep the dex bytes for the class path.

### 9.1 GUI wizard (legacy)

1. **Build prod web + sync:** `npm run cap:prod` (→ https scheme + prod API URL + version synced).
2. **Bump the version** if this is a new release (§10).
3. `npx cap open android` → **Build → Generate Signed Bundle / APK…** → **APK** → Next.
4. **Create the keystore** (once) via **"Key store path → Create new…"**. You *choose* these values now — they are NOT any Google/app-store login:
   - **Keystore** = a `.jks` file that holds your signing key. Save it **outside the repo**.
   - **Keystore password** / **key password** = you pick them (can be the same).
   - **Key alias** = a label for the key, e.g. `frontistirio`.
   - 🔑 **Keep the `.jks` + passwords FOREVER** (password manager + backup). The **same keystore is required for every future update**; lose it → you can't update the app on Play Store. (`*.jks` / `*.keystore` are gitignored.)
5. Build Variant **release** → Finish → output **`android/app/release/app-release.apk`**.
6. Install: `adb install app-release.apk` (USB debugging), or sideload the file (allow "unknown sources"). Verify the login footer shows `v<version>`.
7. **Google Play** wants an **AAB**, not an APK → same wizard, pick **Android App Bundle**.
8. Signing keytool (if you prefer CLI) is bundled with Android Studio: `C:\Program Files\Android\Android Studio\jbr\bin\keytool.exe` (not on PATH by default).

### 9.2 SDK levels — required by the Capgo updater (2026-07-16)

`android/variables.gradle` must be **`compileSdkVersion = 35`** and **`minSdkVersion = 23`**, up from
Capacitor 6's defaults of 34 / 22. `@capgo/capacitor-updater` pulls `androidx.work:work-runtime:2.10.5`
(hardcoded in the plugin's build.gradle — *not* overridable via `rootProject.ext`), which refuses to
build against compileSdk 34; the transitive `androidx.savedstate:1.4.0` then requires minSdk 23.

- **`targetSdkVersion` stays 34 on purpose.** The AAR check only demands *compiling* against 35 —
  raising `targetSdk` would opt into Android 15 runtime behavior (edge-to-edge) and risk the UI for
  no benefit here.
- AGP 8.2.1 prints `WARNING: ... tested up to compileSdk = 34` and builds fine. Only an AGP upgrade
  silences it; not needed.
- minSdk 23 drops Android 5.1 — negligible in 2026, but it *is* a device-reach change.
- ⚠️ `variables.gradle` is inside the gitignored `android/` → **re-apply both bumps after any fresh
  `npx cap add android`**, alongside the signingConfig (§9), `google-services.json`, and the
  versionCode logic (§10).

---

## 10. App versioning (single source of truth)

- **`package.json` `version`** = the one version for web + native.
- `scripts/set-version.js` writes it into `src/environments/version.ts` → shown as **`v1.0.0`** on the login screen. It runs automatically inside `start`, `build`, `build:ionic`, `cap:emulator`, `cap:prod`, and the `version:*` scripts — no manual sync.
- **Android** `android/app/build.gradle` reads `package.json`: `versionName` = the version, `versionCode` derived from semver (`MAJOR*10000 + MINOR*100 + PATCH`; keep MINOR & PATCH **< 100** so it always increases).
- **Bump:**
  - `npm run version:patch|minor|major` — bump only (`--no-git-tag-version`; works with a dirty tree).
  - `npm run release[:minor|:major]` — **bump + deploy** (`build:ionic` once, then S3 + CloudFront **and** the OTA bundle). This is the normal command; web and the phones stay on the same version by construction.
- ⚠️ The build.gradle version logic lives in the gitignored `android/` → re-apply after `cap add` (§1).

### 10.1 The three deploy targets

`release`/`deploy` cover the first two. The third is the only one that needs a store release.

| Target | Command | Ships |
|---|---|---|
| Web (browser) | `npm run deploy:web` | `www/` → S3 + CloudFront |
| Phones, OTA | `npm run deploy:ota` | `www/` → `/ota/` bundle + manifest (installed APKs pick it up) |
| Phones, native | `npm run cap:prod` → `gradlew assembleRelease` (§9) | a new APK |

`deploy` = build once → both `publish:web` and `publish:ota`. `deploy:web` / `deploy:ota` are escape
hatches for one target only; `deploy:ota:dry` builds the artifacts without uploading.

**OTA ships `www/` only** (Angular code, styles, i18n, assets). A new/upgraded Capacitor plugin,
`capacitor.config.ts`, permissions or icons are native → new APK. `scripts/deploy-ota.js` enforces
this rather than trusting memory: it records the plugin list + config hash in `package.json`
`ota.native` and **refuses to publish** when that surface changed unless `ota.minNative` was moved to
the current version (= "the APK for this version carries the new native code"). Override with
`npm run deploy:ota -- --accept-native-change` only when the change is genuinely OTA-safe, e.g. a
plugin was merely removed. See the `ota-live-updates` memory.

---

## 11. Command cheat-sheet
```bash
# web → native
npm run cap:emulator                       # emulator: ng build (10.0.2.2) + http scheme + cap sync
npm run cap:prod                           # device/prod: ng build (API Gateway) + https scheme + cap sync
npx cap sync android                       # copy web + plugins (manual)
npx cap open android                       # open in Android Studio
npx cap run android                        # build+install to a running device/emulator (needs SDK on PATH)

# device / adb
adb devices                                # list targets
adb uninstall com.frontistirio.app         # clean reinstall
adb reverse tcp:3000 tcp:3000              # alt to 10.0.2.2: map device localhost:3000 → host

# versioning + release
npm run version:patch                      # bump 1.0.0 → 1.0.1 (package.json → web + android + login footer)
npm run release                            # bump patch + deploy web AND OTA (build once → S3 + /ota/)
npm run deploy:web                         # web only
npm run deploy:ota                         # OTA only (refuses if a plugin moved w/o a minNative bump)
npm run deploy:ota:dry                     # build bundle + manifest, upload nothing

# firebase send-side sanity
node scripts/sendTestPush.js <fcmToken>    # push straight through FCM (backend)

# prod CORS bisect (device login fails but web works — §5.10)
curl -i -X POST <API>/users/login -H "Origin: https://localhost" -H "Content-Type: application/json" -d '{}'
#   500 / no Access-Control-Allow-Origin → add native origins to the DEPLOYED index.js → pm2 restart
```
