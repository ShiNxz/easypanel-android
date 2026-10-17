# Android emulator for Easypanel

The official Android emulator with a Google Play system image, running in Docker as one Easypanel
Compose service, with its screen in your browser:

- **emulator**: [HQarroum/docker-android](https://github.com/HQarroum/docker-android), Android 13
  with Google Play. A full virtual machine with its own kernel and a simulated modem and SIM, so
  apps see a phone rather than a container.
- **web**: [ws-scrcpy-web](https://github.com/bilbospocketses/ws-scrcpy-web), scrcpy 5 in the
  browser, with a UHID mouse and keyboard (Android draws a real pointer), an adb shell and a file
  manager.
- **proxy**: Caddy in front of the viewer, so any domain works without configuring it.

Both images are pinned by digest in the Environment tab.

## Requirements

- A Linux host with KVM: `ls -l /dev/kvm` must exist. A VM works only with nested virtualization.
- About 6 GB of RAM and 4 cores per emulator, plus about 15 GB of disk.
- Easypanel.

## Deploy

1. Make the template JSON. Start from [`template.example.json`](template.example.json) and replace
   the basic auth password with a long random one (`openssl rand -base64 18`).
2. In Easypanel, open a project: **Templates**, **Custom**, **Create From Schema**, paste, **Create**.
   Easypanel pulls this repo and deploys. The first start pulls about 3 GB of images, and the
   phone's first boot takes 2-5 minutes (`ANDROID_READY` in the emulator's logs).
3. Open the service's domain and sign in with the basic auth user and password (also in the
   service's **Security** tab).
4. On the device card, click **connect**, tick **UHID Input (keyboard + mouse)** and click the
   screen. Esc gives the mouse back.

The template uses a git source on purpose. Easypanel 2.32's Create From Schema with an inline
Compose file writes into the service's folder before creating it, restarts the panel and deploys
before the Compose file exists. A git source creates the folder first.

To change the domain later, edit it in the **Domains** tab and keep service `proxy`, port `80`.

### An existing Compose service

In its **Source** tab choose **Git**: repository `https://github.com/ShiNxz/easypanel-android.git`,
branch `v1`, build path `/`, Compose file `docker-compose.yml`. Put the Environment lines below in
its Environment tab with **Create .env file** on, point its domain at service `proxy`, port `80`,
and deploy. Its `files` folder stays, and with it the phone.

## Environment

| Variable | Meaning |
| --- | --- |
| `EMULATOR_IMAGE` | The emulator image, pinned by digest. Changing it under an existing phone can change the device: back up first. |
| `VIEWER_IMAGE` | The ws-scrcpy-web image, pinned by digest. |
| `MEMORY`, `CORES` | RAM in MB and CPU cores for the emulator. 4096 and 2 is the minimum; 6144 and 4 is smooth. |
| `EXTRA_FLAGS` | Emulator flags. Default `-no-audio -no-metrics -no-boot-anim`. Add `-partition-size=8192` for 8 GB of storage on a new phone only. |
| `SCREEN_SIZE`, `SCREEN_DENSITY` | Set on the phone after each boot. Default `720x1280` and `320`: fewer pixels for the software renderer and the stream. Empty leaves the phone's own. |
| `WS_SCRCPY_ALLOW_REMOTE_ADMIN` | `1` keeps the viewer's Settings usable without its own login. Set `0` only after turning that login on. |

## Desktop scrcpy

adb listens on the host's `127.0.0.1` only, on a free port Docker picks. On the host:

```bash
docker port PROJECT_SERVICE-emulator-1 5555
```

Then tunnel to it from your computer and connect (use a local port other than 5555):

```bash
ssh -N -L 5556:127.0.0.1:HOST_PORT root@HOST
```

```bash
adb connect 127.0.0.1:5556
```

The emulator accepts only the key in `files/adb/adbkey`: approve your computer's key on the phone's
screen, or load that key with `ADB_VENDOR_KEYS`.

## Security

- The domain is protected by Easypanel's basic auth. Never remove it while the domain is attached.
- Easypanel puts `proxy` on its shared `easypanel` Docker network. Any other app on the same server
  reaches the viewer there without the basic auth, and the viewer controls the phone (screen,
  shell, files). If anything else runs on the server, turn on the viewer's own login first:
  **Settings**, **Users**, **manage users** (an admin with a password), **enable login**, then set
  `WS_SCRCPY_ALLOW_REMOTE_ADMIN=0` and deploy.
- The emulator stays on the service's own network, and adb on the host's loopback.
- `files/adb/adbkey` is a key to the phone. Treat it like a password.

## Back up and move

Stop the service, then copy its `files` folder (`emulator-data`, `adb`, `ws-scrcpy-web`). To move
the phone: create the service on the new host and stop it once it is up (its first boot makes a
blank phone), stop the old one, copy the folder over with `rsync -aHS --delete` (which replaces the
blank phone), then start the new one. Never run two copies of the same phone at once.

## Updating

Releases are branches (`v1`, `v2`, ...). An existing service moves only when you change the branch
in its Source tab and deploy.
