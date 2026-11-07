# SAVER BY SKYPIN

**Version 9.6.9** · Author: **@Skypin969** (Telegram) · BY SKYPIN

A KernelSU module that saves battery while your screen is off, and locks the screen when you forget it on. It works in two stages and puts everything back the moment you unlock your phone.

> Tested on **Redmi Note 15 4G** (Android 16, KernelSU Next) and **POCO X7 5G** (LineageOS 24, Android 17, KernelSU Next). Other devices are not tested yet. Every feature is checked before use and skipped if your device does not support it.

---

## What it does

### Stage 1 - screen off for less than 30 minutes
- Forces **Doze** and turns on **Battery Saver**, then clears background apps (`am kill-all`).
- **Mobile data, Wi-Fi, Bluetooth and hotspot stay on**, so you still get messages and your hotspot clients stay connected.
- If music is playing, Doze is delayed until the music stops.

### Stage 2 - screen off for more than 30 minutes
- Turns off **Wi-Fi, mobile data, Bluetooth and hotspot**, even if music is playing. Only an active phone call postpones it.
- It remembers what was on before. **Only those items are turned on again**: if your hotspot was off, it stays off.
- Works while charging too.

### Unlock at any time
- Everything is restored: Doze off, Battery Saver off, and the radios that were on before come back by themselves.
- If the phone **reboots during Stage 2**, the radios that were on are restored automatically after boot (Android does not bring the hotspot back after a reboot, this module does).

### Auto screen lock
- If the screen is on and you do **not touch the screen or press a volume key for 30 minutes**, the screen locks (even during video).
- Map and navigation apps, and phone calls, are excluded.

The 30-minute values are fixed.

---

## Install

1. Download `SAVER-BY-SKYPIN-v9.6.9.zip` from the **Releases** page.
2. Open **KernelSU / KernelSU Next**, go to **Modules**, then **Install from storage**, and choose the zip.
3. Read the install screen. It lists your device checks (`dumpsys`, `settings`, `svc`, `input`, `getevent`, `cmd`, `am`, `wake_lock`).
4. **Reboot.** The module starts by itself on every boot. No menu, no settings.

## Uninstall

Remove the module in KernelSU and reboot. The uninstall script stops the service, restores your settings and deletes the state, log and saved hotspot password files.

---

## Hotspot password note (some ROMs)

To bring your hotspot back, the module needs its name, security type, band and password.

- On many ROMs (for example Redmi / HyperOS) this is read from the system automatically.
- On some ROMs (for example LineageOS 24 on POCO X7) Android **hides the password** from every readable place. If so, the log says `hotspot password hidden by Android`. Save the password once, as root:

```sh
su -c 'echo "yourpassword" > /data/adb/saver_hotspot_pass; chmod 600 /data/adb/saver_hotspot_pass'
```

The module never restarts the hotspot with a wrong password: if the password is unknown it skips the hotspot and writes the reason to the log. The file is readable by root only and is deleted when you uninstall the module.

The hotspot is restarted with the band your device accepts: the saved band first, then 2.4 GHz, then 5 GHz, then any. Some devices refuse "any".

---

## How it works

| Part | How |
| --- | --- |
| Boot | `service.sh` runs on every boot and loops in the background. |
| Screen state | Read from `dumpsys power` (`mWakefulness`). |
| Doze | `dumpsys deviceidle force-idle` / `unforce`. |
| Battery Saver | `settings put global low_power 1` / `0`. |
| Music / calls | Read from `dumpsys audio` and `dumpsys telephony.registry`. |
| Stage 2 timer | Holds a wake lock (`/sys/power/wake_lock`) with a timeout so the phone cannot sleep through the timer. |
| Radios off / on | `cmd wifi set-wifi-enabled`, `cmd bluetooth_manager enable/disable`, `cmd phone data enable/disable`. `svc` is used as a backup, because `svc` fails when started from the boot service while `cmd` works. |
| Radio restore | After unlock or boot, radios are turned on and **checked again up to 8 times, 10 seconds apart**, because some stay off right after boot. |
| Hotspot off / on | `cmd wifi stop-softap` / `cmd wifi start-softap`. The config is read from `WifiConfigStoreSoftAp.xml`, or from `dumpsys wifi` when that file does not exist. The start is judged by the output (`failed to start`), not by the exit code, which is always 0. |
| Touch detection | `getevent` on the touch and volume devices, to reset the 30-minute auto-lock timer. |
| Auto lock | `input keyevent 223` (sleep). |
| State | `/data/adb/saver_state` is written when Stage 2 starts and deleted after a full restore. If it exists after a reboot, the restore runs. |
| Log | `/data/adb/saver.log` |

Files in the module: `module.prop`, `customize.sh` (install screen and device checks), `service.sh` (all logic), `uninstall.sh`, `README.txt`.

---

## Check that it works

```sh
su -c 'tail -20 /data/adb/saver.log'
```

A normal day looks like this:

```
SAVER BY SKYPIN v9.6.9 started (TEST=0)
screen off
stage1 on
doze forced
state saved
after-off w=0 b=0 d=0
stage2 on
hotspot start rc=0 sec=wpa3_transition band=2
radio retry x1
radios w=1 b=1 d=1
restored
screen on, normal
```

On some devices the `d=` value (mobile data) in the log is not accurate. Check the data icon on the phone instead.

---

## Notes and limits

- Needs root with **KernelSU / KernelSU Next**. Magisk is not tested.
- Hotspot restore needs a ROM that has `cmd wifi start-softap`. If it does not, the module logs `start-softap unsupported, skipped` and leaves the hotspot off.
- Wi-Fi, mobile data and Bluetooth restore after boot uses a short retry loop. Allow up to about 30 seconds after the home screen appears.
- Another battery module or kernel tweak that also forces Doze may fight with this one.
- The module changes system settings. Use it at your own risk.

## Telegram

**@Skypin969**

## License

MIT. See `LICENSE`.
