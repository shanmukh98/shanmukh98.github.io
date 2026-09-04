---
layout: post
title: Clean up an Android TV safely with ADB
---

Android TV and Google TV devices often ship with recommendation feeds, demo
apps, telemetry, and services you may never use. ADB can disable those packages
without uninstalling them, rooting the TV, or changing its firmware.

> This workflow was inspired by
> [CobaNoV's Android TV cleanup guide](https://tv.cobanov.dev/). The launcher
> recommendation below uses Projectivy instead of FLauncher because Projectivy
> provided reliable, built-in input and HDMI shortcuts on my Xiaomi TV.

## Before you start

Enable **Developer options** and **USB/Wireless debugging** on the TV, find its
IP address, and keep the computer and TV on the same network.

Install ADB:

```sh
# macOS
brew install android-platform-tools

# Debian/Ubuntu
sudo apt install adb
```

Windows users can download Google's Android SDK Platform Tools. Connect with:

```sh
adb connect TV_IP:5555
```

Approve the debugging prompt on the TV. Android 11 and newer may require
`adb pair TV_IP:PAIRING_PORT` first.

## Measure and record everything

Take a baseline before changing the TV:

```sh
mkdir -p tv-debloat
adb shell dumpsys meminfo > tv-debloat/before-meminfo.txt
adb shell pm list packages -s > tv-debloat/before-system-packages.txt
adb shell pm list packages -d > tv-debloat/before-disabled-packages.txt
```

Never uninstall factory packages. Disable only identified packages:

```sh
adb shell pm disable-user --user 0 PACKAGE_NAME
```

Undo one change with:

```sh
adb shell pm enable PACKAGE_NAME
```

Keep every disabled package in `tv-debloat/disabled-packages.txt`, along with a
short explanation and its undo command.

## Work in small batches

Classify packages as:

1. Clearly unused: retail demos, telemetry, unused streaming apps, or
   recommendation providers.
2. Personal choice: voice search, screensavers, casting, accessibility, local
   media players, and live-TV setup.
3. Untouchable: input/HDMI services, remote and Bluetooth services, the
   keyboard, audio, System UI, Play Services, the Play Store, fused location,
   and the current launcher.

Disable no more than ten packages at once. After each batch, test Home and Back,
directional controls, input switching, HDMI, sound, the keyboard, Netflix,
YouTube, voice search, and casting. If something breaks, re-enable the complete
last batch before narrowing it down one package at a time.

Package names vary by manufacturer. Do not disable an unfamiliar package just
because its name looks unimportant; quick-panel and source-menu packages often
look like bloat.

## Replace the ad-heavy launcher

Install **Projectivy Launcher** from the Play Store and open it once. Set it as
the Home app in Android's default-app settings, or use:

```sh
adb shell cmd package set-home-activity --user 0 com.spocky.projengmenu
```

Confirm Android resolves Home to Projectivy:

```sh
adb shell cmd package resolve-activity --brief \
  -a android.intent.action.MAIN \
  -c android.intent.category.HOME
```

Only after Projectivy works should the Google TV launcher be disabled:

```sh
adb shell pm disable-user --user 0 com.google.android.apps.tv.launcherx
```

Some Google TV builds have a setup/recovery package that also claims the Home
button. Do not disable it blindly: it may contain account, replacement-remote,
or Cast setup screens. Identify it, confirm initial setup is complete, record
the recovery command, and test it as a separate batch.

Projectivy was preferable in this case because FLauncher offered a clean grid
but no practical native route to the TV's input selector. Projectivy exposed
source and HDMI actions directly, avoiding button-remapping hacks.

## Apply the low-risk performance changes

Shorten UI animations and trim disposable caches:

```sh
adb shell settings put global window_animation_scale 0.5
adb shell settings put global transition_animation_scale 0.5
adb shell settings put global animator_duration_scale 0.5
adb shell pm trim-caches 999G
```

Reboot, confirm the launcher and essential features still work, and repeat the
three baseline commands using `after-` filenames.

Compare memory only when the TV is doing the same thing in both snapshots.
Streaming video can add app memory plus codec, DRM, GPU, and frame buffers, so
an idle-before versus playback-after comparison is misleading.

Nothing in this process requires root, a bootloader unlock, or a custom ROM.
Keeping a package list and testing small batches makes every persistent change
reversible.
