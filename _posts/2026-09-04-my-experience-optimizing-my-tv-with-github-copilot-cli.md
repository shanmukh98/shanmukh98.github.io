---
layout: post
title: My Experience Optimizing My TV with GitHub Copilot CLI
description: A reversible Android TV cleanup using wireless ADB, small test batches, and Projectivy Launcher.
accent: "#65728f"
tags:
  - android-tv
  - adb
  - github-copilot
---

I came across
[CobaNoV's Android TV cleanup guide](https://tv.cobanov.dev/) while looking for
a way to fix my laggy Xiaomi TV. The home screen was full of recommendations,
and moving between apps had become noticeably slow.

The guide showed that Android TV apps can be disabled safely over ADB without
rooting the TV or uninstalling system files. That sounded like exactly what I
wanted.

## I tried it with GitHub Copilot CLI

I already had access to
[GitHub Copilot CLI](https://docs.github.com/copilot/how-tos/use-copilot-agents/use-copilot-cli),
so I used it to run the process for me. This was useful because Copilot could
inspect the packages on my specific TV, explain what they did, run the ADB
commands, and pause after every batch while I tested the remote and apps.

## The steps I followed

1. I installed Copilot CLI. It requires an active GitHub Copilot subscription.
   On macOS I used `brew install --cask copilot-cli`; Windows users can use
   `winget install GitHub.Copilot`.
2. On the TV, I opened **Settings -> System -> About** and pressed **Build**
   seven times. I then enabled USB/Wireless debugging and found the TV's IP
   address under its network settings.
3. I opened an empty folder, ran `copilot`, signed in with `/login`, and pasted
   the prompt below. I reviewed and approved commands before Copilot ran them.
4. Copilot installed ADB, connected to the TV, and asked me to approve the
   debugging dialog shown on the TV.
5. It measured memory and saved the installed and disabled package lists before
   changing anything.
6. It grouped packages into safe, optional, and untouchable categories. I
   approved small batches and tested the TV after each one.
7. When prompted, I installed Projectivy from the Play Store on the TV and
   opened it once before changing the default Home app.

Android 11 and newer may display a pairing code under **Wireless debugging**.
In that case, Copilot can pair first and then connect to the ADB port displayed
on that screen; it is not always port `5555`.

<details markdown="1">
<summary><strong>The prompt I used, updated with what I learned</strong></summary>

```text
Clean up my Android TV over ADB. Run the commands yourself and explain each
step in plain language.

MY TV
- Make/model: REPLACE_ME
- IP address: REPLACE_ME
- Developer options and USB/Wireless debugging are enabled.
- My computer and TV are on the same network.

RULES
1. Never uninstall anything. Only disable packages with:
     adb shell pm disable-user --user 0 PACKAGE
   Every change must be recoverable with:
     adb shell pm enable --user 0 PACKAGE
2. Do not root the TV, unlock its bootloader, or flash firmware.
3. Before changing anything, save these to files:
     dumpsys meminfo
     pm list packages -s
     pm list packages -d
4. Disable no more than 10 packages per batch. After every batch, stop and ask
   me to test Home, Back, directional controls, input/source switching, HDMI,
   Netflix, YouTube, sound, the keyboard, voice search, and casting.
5. If something breaks, re-enable the complete last batch first. Then isolate
   the package one at a time.
6. Keep a text file containing every disabled package and a Markdown log with
   the reason, exact command, and exact undo command.
7. Never disable Android/System UI, settings, input or HDMI services, remote or
   Bluetooth services, audio, Play Services, Play Store, fused location, input
   methods, or the active launcher.
8. Package names differ by manufacturer. Identify this TV's quick panel,
   source selector, TV-input stack, remote services, and keyboard. If a package
   is uncertain, ask me before disabling it.

WORKFLOW
1. Install ADB if needed:
   - macOS: brew install android-platform-tools
   - Debian/Ubuntu: sudo apt install adb
   - Windows: install Google's Android SDK Platform Tools
2. If the TV requests wireless pairing, use adb pair TV_IP:PAIRING_PORT first.
   Then connect with adb connect TV_IP:ADB_PORT using the port shown on the TV.
   Use port 5555 only when the TV uses the older ADB-over-network setup.
3. Capture the baseline and identify the manufacturer, Android version,
   current Home activity, input method, and TV-input services.
4. Sort packages into clearly unused, depends on me, and untouchable. Show me
   each proposed batch and wait for approval.
5. Ask me to install and open Projectivy Launcher. Set
   com.spocky.projengmenu as Home and confirm the resolved Home component
   belongs to com.spocky.projengmenu.
   Only then consider disabling:
     com.google.android.apps.tv.launcherx
6. Some Google TV builds have com.google.android.tungsten.setupwraith claiming
   Home after launcherx is disabled. Do not disable it automatically. Inspect
   it, confirm initial setup is complete, explain that account,
   replacement-remote, or Cast setup may require it later, and get my
   approval. Treat the launcher switch as a separate batch.
7. Preserve any OEM package containing the input/source popup, even if it also
   contains recommendations. Test Projectivy's source and HDMI shortcuts.
8. After package testing, set all three animation scales to 0.5 and trim only
   disposable caches with:
     adb shell pm trim-caches 999G
9. Reboot, confirm Projectivy is still Home, and repeat the measurements. Take
   both memory snapshots under the same workload, preferably idle on Home with
   no video playing.
10. When everything is complete, run adb disconnect and ask me to turn off
    Wireless debugging unless I still need it.

WHEN FINISHED
- Give me a before/after memory table.
- List every disabled package with a one-line explanation.
- Give me one command that restores all persistent changes.
- Explain that deleted cache files cannot be restored but regenerate normally.
```

</details>

## Issues I faced

The first problem was simple: ADB could see the TV, but the connection remained
unauthorized until I accepted the debugging prompt on the TV.

The package names were the harder part. On my Xiaomi TV,
`com.mitv.tvhome.atv` looked like another content app, but it also contained
the input-source popup. Disabling it could have removed my ability to switch to
HDMI, so I left it enabled.

Disabling only Google's launcher exposed a separate Google setup/recovery Home
screen. Home, Back, and the directional buttons then stopped working on the
home screen.

Copilot followed the safety rule and immediately re-enabled the last package.
That returned the TV to normal before I tried anything else. After inspecting
the setup fallback and getting my approval, Copilot disabled both Google Home
packages together and FLauncher became the working Home app.

## Why I chose Projectivy

[Projectivy Launcher](https://play.google.com/store/apps/details?id=com.spocky.projengmenu)
was a better fit for my TV because it included native source and HDMI actions.
I could keep a clean home screen and still reach the input selector without
button-remapping tools or other workarounds.

FLauncher itself displayed apps and worked with the remote, but it did not
offer a practical way to open Xiaomi's input selector. The Google launcher and
its completed setup fallback had already been disabled while FLauncher was
Home. After confirming Projectivy worked, I made it the default and disabled
FLauncher too.

The setup fallback also contains account, remote-pairing, and Cast setup
screens. If I need one of those later, I can temporarily re-enable it.

## What Copilot changed

Android's `adb shell pm disable-user --user 0 PACKAGE` command disables a
package only for the main user. The original APK remains on the TV, and
`adb shell pm enable --user 0 PACKAGE` restores it. Nothing was uninstalled,
the TV was never rooted, and its firmware was never touched.

Copilot disabled 17 packages, including telemetry, recommendation providers,
the Google launcher, Ambient Mode, and apps I did not use. It deliberately kept
the HDMI/input stack, remote and Bluetooth services, sound, keyboard, voice
search, casting, Play Services, Netflix, YouTube, Prime Video, and local media
features.

It also:

- kept a file containing every disabled package and its undo command;
- set all three Android animation scales to `0.5`;
- trimmed disposable app caches;
- rebooted the TV and confirmed Projectivy was still Home; and
- repeated the original package and memory measurements.

## Results

- The ads and recommendation rows were gone.
- Projectivy and its input shortcuts still worked after reboot.
- The TV felt noticeably smoother during normal use.
- 17 packages were disabled, including FLauncher after Projectivy
  replaced it, but nothing was uninstalled.

Cache trimming temporarily recovered about **496 MiB** of storage; apps will
rebuild some of it. Disabled-package processes had used about **170 MiB PSS**,
and the targeted reduction after accounting for Projectivy was about
**95.5 MiB**.

I never captured a clean idle-before and idle-after comparison because I
started Prime Video after reboot. Whole-system Used RAM rose while the app,
codec, DRM, GPU, and frame buffers were active, so the PSS figure above is a
package-level comparison rather than a claim about total idle RAM.

The exact package list will differ by TV brand. The reusable lesson is to
measure first, identify packages instead of guessing, make reversible changes
in small batches, test the real hardware after every batch, keep a complete
undo log, and turn wireless debugging off when finished.
