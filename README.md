# 📱 Android AVD in your Browser — GitHub Actions Workflow

Run a **full Android 15 emulator (AVD)** on a GitHub Actions runner and control it
from **any browser** — no local install, no Android Studio, nothing to keep awake.

```
 Your browser
    │  https://xxxx.trycloudflare.com            (random URL, no account needed)
    ▼
 Cloudflare edge ──► GitHub runner :8000   ws-scrcpy (device screen + control)  ─┐
                └──► GitHub runner :8080   Dashboard (APK · clipboard · status) ─┤
                                                                                 │
                          Android 15 emulator (AVD, KVM-accelerated) ◄───────────┘
                                     │ adb screenrecord (180 s chunks, ~4 GB rolling)
                                     ▼
                          session-recording.mp4 → workflow artifact
```

Everything lives in **one workflow file** — no other repo changes needed.

---

## 🚀 Quick start (2 minutes of setup)

1. **Fork or open your (public) GitHub repository.**
   > Public repo = free Actions minutes **and** free KVM hardware acceleration.
   > On a private repo the emulator still runs, but in slow software mode.

2. Copy the workflow into place:

   ```bash
   mkdir -p .github/workflows
   cp android-web-avd.yml .github/workflows/
   git add .github/workflows/android-web-avd.yml
   git commit -m "Add browser-accessible AVD workflow"
   git push
   ```

3. In GitHub: **Actions tab → “Android AVD in browser (ws-scrcpy)” → Run workflow**
   - **Android system image** — `35` (Android 15) by default; also `34 / 33 / 30`
   - **Session length** — `360` minutes (GitHub’s hard max); also `60 / 120 / 240`
   - **Record the device screen** — on by default

4. Wait **~5–8 minutes** for setup (system image download, emulator boot,
   ws-scrcpy build). Open the run as it progresses.

5. Scroll to the run **Summary** — you’ll see:

   | What | Link |
   |---|---|
   | 🎮 Control the device (ws-scrcpy) | `https://xxxx-yyyy.trycloudflare.com` |
   | 🛠 Dashboard (APK upload · clipboard · status) | `https://zzzz-www.trycloudflare.com` |

   The same links are re-printed into the job log every 5 minutes
   (step *“Keep session alive”*), and sent as `::notice::` annotations.

---

## 🎮 Using the device (ws-scrcpy)

1. Open the **ws-scrcpy** link → the device (`emulator-5554`) appears in the list.
2. Pick a **client**:
   - **H264** — smooth, hardware-decoded in the browser. *Recommended.*
   - **MJPEG** — slower but very compatible; try it if H264 stutters.
3. Click **Connect**. You now have full touch + keyboard control.
   The toolbar includes Home / Back / Overview / Rotate / Volume / Power / Keyboard.
4. The UI has been re-themed (dark mode + **Font Awesome** icons) by the workflow —
   the patch is injected automatically into ws-scrcpy’s HTML at startup.

## 🛠 Dashboard features

- **Install APK** — drag & drop an `.apk` onto the upload zone; it’s uploaded to the
  runner and installed with `adb install -r -t`. Progress bar + installer output shown.
  *Any non-APK file is pushed to `/sdcard/Download` instead.*
- **Clipboard** — paste text and **Send to device** (uses `cmd clipboard`, falls back
  to typing into the focused field). **Read from device** is attempted but may be
  blocked by newer Android versions (Android forbids silent clipboard reads).
- **Device card** — live status (model, Android version, API, resolution, online dot),
  refreshed every 5 s, plus a button that jumps straight to device control.

## 🔴 Screen recording

- Recorded in 180-second chunks at 4 Mbps with a **~4 GB rolling window** (oldest
  chunks are dropped first so the runner’s disk never fills during a 6-hour session).
- When the session ends (or you **cancel the run**), chunks are stitched into
  `session-recording.mp4` (ffmpeg) and uploaded as the **`avd-session`** artifact,
  together with a `session_info.txt` containing the tunnel links.

## ⏹ Stopping the session

- **Cancel the workflow run** (Run → *Cancel workflow*) — the finalize steps still
  run (`if: always()`), so your recording artifact is uploaded.
- Starting a **new** dispatch automatically cancels the previous session
  (`concurrency` group `android-avd-browser`).
- At the end of the chosen duration the session exits cleanly on its own.

---

## ⚙️ Inputs reference

| Input | Values | Default | Notes |
|---|---|---|---|
| `api` | 35 / 34 / 33 / 30 | `35` | Android 15 / 14 / 13 / 11 system image (google_apis, x86_64) |
| `duration_minutes` | 60 / 120 / 240 / 360 | `360` | Kept ≤ 350 min internally so artifacts always upload |
| `record` | true / false | `true` | Screen recording → `avd-session` artifact |

## 🧰 What the workflow does, step by step

| # | Step | What happens |
|---|---|---|
| 0 | Preflight | Detects `/dev/kvm`, starts the ADB server |
| 1 | Create & boot AVD | Installs the system image, picks the newest Pixel profile (Pixel 8 → 7 → 6 fallback), boots headless with KVM, waits for `sys.boot_completed`, disables animations |
| 2 | ws-scrcpy | Clones NetrisTV/ws-scrcpy, injects the Font Awesome dark theme into its HTML, builds it, starts it on `:8000` |
| 3 | Dashboard | Starts the Flask dashboard on `:8080` (auto-moves to 8081 if busy) |
| 4 | Tunnels | Downloads `cloudflared`, opens two quick-tunnels, publishes links to the run summary |
| 5 | Recorder | Starts the rolling `screenrecord` chunk loop (if enabled) |
| 6 | Keep-alive | Holds the session, health-checks all services, re-prints the links every 5 min |
| 7 | Finalize | Stops recording, concatenates chunks → `session-recording.mp4`, uploads artifact |

## 🔧 Troubleshooting

| Symptom | Fix |
|---|---|
| Emulator feels slow | Make sure the repo is **public** (KVM). If private, `/dev/kvm` is absent → software mode. Also try the **MJPEG** client, or a lower API image (33/30). |
| No `trycloudflare.com` URL in the summary | Quick-tunnels occasionally fail to hand out a URL — check the log of step 4; simply re-run the workflow. |
| “No devices” in ws-scrcpy | Click **Reload** in the device list. If still empty, check the job log: the emulator may have failed to boot (step 1 prints the emulator log on failure). |
| Clipboard *read* fails | Expected on modern Android (silent reads are blocked). *Send to device* still works; on-device copy/paste within Android also works. |
| APK install fails | Some APKs are split (.apks/.xapk) — install a merged/signed single `.apk`. Check the installer output shown after upload. |
| Dashboard won’t open | The dashboard may have moved to port 8081 — the tunnel is created against whatever port the dashboard actually used, so use the link from the summary, not a raw port. |
| Artifact missing | Recording was disabled, the session lasted < 1 chunk, or no `session_info.txt` existed. |

## 🔒 Security notes

- Anyone who obtains the `trycloudflare.com` URLs gets **full control of the
  emulator** (and the upload endpoint). Treat the links like passwords — the random
  subdomains are unguessable, but don’t post them publicly.
- The emulator is disposable and isolated on the runner; nothing on the runner is
  sensitive (no repo secrets are exposed to the AVD).
- For stronger protection you can front the tunnels with **Cloudflare Access**
  (requires a Cloudflare account + domain) — out of scope for this zero-config setup.

## 💸 Cost & limits

- **Public repo:** Actions minutes are **free** (Linux runners, standard tier).
- **Private repo:** billed per-minute (a 6-hour session = 360 min on Linux) and no
  KVM on standard runners → slow software emulation.
- GitHub’s hard per-job limit is **360 minutes** on hosted runners; the workflow
  self-limits to 350 so artifacts always have time to upload.
- Quick tunnels are best-effort (no SLA) — perfect for a remote-control session,
  but the URL changes on every run by design.
