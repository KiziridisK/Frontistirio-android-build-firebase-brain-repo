# Frontistirio — Android Build + Firebase Push Runbook

> **Procedural reference** for wrapping the Ionic/Angular web app (`Frontistirio`) as a **native Android app** with **Capacitor**, running it on an **emulator**, and wiring **Firebase Cloud Messaging (FCM)** push end-to-end. Written after doing it for the [announcements feature](https://github.com/KiziridisK/Frontistirio-announcements-brain-repo) (2026-07-02). Use it whenever repeating similar native-build / push work. Every gotcha here is one we actually hit.

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
4. `npx cap add android` → generates the `android/` Gradle project (commit it).

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
The native app bundles **whatever `www/` you last built**. `npm run build:ionic` = **production** (`environment.prod.ts` → the AWS URL). A plain `ng build --configuration development` = `environment.ts`. So to test against the **local backend**, build with an env that points at it, then `cap sync`.

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

---

## 6. Diagnostic flow (the decision tree we used)
```
login fails, NO request in backend logs
        │
        ▼
backend up + reachable from host?  (netstat :3000 / curl localhost:3000)
        │ yes
        ▼
emulator can reach it?  (emulator browser → http://10.0.2.2:3000/)
        │ "Cannot GET /" = YES → problem is INSIDE the app
        ▼
check, in order:
  • env URL baked in?  (10.0.2.2, not AWS prod, not localhost)   → rebuild dev + cap sync
  • mixed content?     (androidScheme https → http API blocked)   → androidScheme 'http' + sync + rebuild   ← the culprit for us
  • cleartext?         (usesCleartextTraffic)                     → add to manifest
  • CORS?              (native origin allowed)                    → add localhost origins
```

---

## 7. End-to-end push test
1. Backend restarted with creds → `🔔 Firebase Admin initialized`.
2. **Device (emulator)** logged in (as a student, built with `10.0.2.2`) → on login it registers its FCM token → row in `DeviceToken` (confirm in DB / logs). Allow the notifications prompt.
3. **Sender (browser, env=localhost)** logged in as a **store-user** → compose an announcement targeting that student → send.
4. Emulator receives the push: **foreground** → in-app toast; **background/killed** (press Home first) → status-bar notification. Backend: `dispatchAnnouncement → pushDispatcher → FCM`, recipient `sent`.

> Tip: the **sender uses `localhost`** (browser) and the **device uses `10.0.2.2`** (baked into its build) — both hit the same backend. Keep `environment.ts` on `localhost` for browser dev; only flip to `10.0.2.2` when producing an emulator build.

---

## 8. Revert for production
- `capacitor.config.ts` → `server.androidScheme: 'https'`.
- Native build should point at the **https** prod API (production config / `environment.prod.ts`).
- Cleartext (`usesCleartextTraffic`) can be dropped for release (or keep it debug-only).
- Never commit `serviceAccountKey.json` (it's `.gitignore`d).

---

## 9. Command cheat-sheet
```bash
# web → native
ng build --configuration development       # or npm run build:ionic (prod)
npx cap sync android                       # copy web + plugins
npx cap open android                       # open in Android Studio
npx cap run android                        # build+install to a running device/emulator (needs SDK on PATH)

# device / adb
adb devices                                # list targets
adb uninstall com.frontistirio.app         # clean reinstall
adb reverse tcp:3000 tcp:3000              # alt to 10.0.2.2: map device localhost:3000 → host

# firebase send-side sanity
node scripts/sendTestPush.js <fcmToken>    # push straight through FCM (backend)
```
