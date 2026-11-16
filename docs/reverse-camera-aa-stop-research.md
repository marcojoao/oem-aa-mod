# Research: wired OEM Android Auto stops after the reverse camera (CX-3, CMU 74.00.324)

Status: desk research only, 2026-11-16. Nothing here was run on a car, an emulator or qemu. No firmware binaries (`blmjciaapa.so`, `libaap_interface.so`, MMUI `uia` libraries, stock v74 `systemApp.js`) were available locally, so every statement about OEM internals comes from the oem-aa-mod source comments, the documented D-Bus interfaces, or public bulletins.

## Setup and symptom

- Mazda CX-3, Mazda Connect CMU 74.00.324, CarPlay/Android Auto USB hub retrofit, wired phone, OEM Android Auto (`com.jci.aapa`, `/jci/aapa/blmjciaapa.so`).
- Some MZD-AIO tweaks installed (which ones is unknown). oem-aa-mod is not installed yet.
- Symptom: shift into R (camera shows), shift out of R, and Android Auto does not come back. The owner describes it as "stops". It is not yet known whether the AA session actually ended or only the screen did not return to AA.

The single most important first question is which of those two things happens. Almost every cause below falls into one of two groups:

- **Group A, "screen not restored"**: the AA session is still alive (phone still projecting, audio may still play), but the CMU returns to a Mazda screen instead of AA, or AA video focus is never handed back.
- **Group B, "session really ended"**: the USB/AOA link or the AA protocol session was torn down, by the phone, by the USB hub, or by `blmjciaapa` itself.

## 1. Plausible root causes, ranked

### Rank 1 (Group A): OEM video-focus or screen-restore bug in the camera handoff

What happens normally: the backup camera service `com.jci.bucpsa` takes the screen (signals `ReverseStatusChanged(i)` and `DisplayMode(u)`, where non-zero means the camera owns the display). The AA side must send the phone a video focus LOSS, then, when the camera ends, MMUI must return to the AA context and `aapa` must send video focus GAIN again.

Evidence:
- `mazda-cmu-documentation/dbus/com.jci.bucpsa.xml.md`: `GetDisplayMode`, `DisplayMode`, `ReverseStatusChanged`, `GetReverseStatus` are the camera hand-off surface.
- `mazda-cmu-documentation/dbus/com.jci.aapa.xml.md`: `VideoProjectionEventToMD(u)` (method, MMUI to aapa), `VideoProjectionRequestFromMD(u)` (signal), `ClearLastFocus((bb))`, `NotifyNoVideoSinkSetup(b)`, `NotifyStopAndSetup(u)`, `NotifyDeviceNotResponding(b)`, `DisplayReconnectingDialog(b)`. These are the OEM hooks that move AA video in and out.
- Mazda bulletin for v74 (NHTSA MC-10226834-0001, https://static.nhtsa.gov/odi/tsbs/2022/MC-10226834-0001.pdf) lists as fixed in 74.00.324: "If the shift lever is operated quickly while displaying the CarPlay screen, the rear view screen may not return to the CarPlay screen" and "When the gear position is shifted into P from R, the screen may not return to the previous condition before shifting into the R position". Same text quoted at https://mazdas247.com/forum/threads/mazda-connect-firmware-74-00-230a-now-324a.123875753/page-11. So Mazda itself had bugs in exactly this path in v74, the fix text names only CarPlay and R to P, and earlier v70/v59 bulletins also list several "screen goes to Home instead of the previous source after reverse" bugs. A remaining AA-only or R to D variant on 324 is plausible.
- Legacy contrast, `~/Repos/headunit/mazda/callbacks.cpp:242-252`: the legacy binary releases video focus when `DisplayMode != 0` and, on `DisplayMode == 0`, re-requests it after a fixed 1000 ms delay (`callbacks.cpp:219`), with the comment that "100ms is too early" because the CMU has not finished switching surfaces from the camera back to Opera. That shows the race is real on this hardware. The OEM `blmjciaapa` has its own logic for this, which we cannot see.
- oem-aa-mod notes (`docs/fork-notes-cx3.md`, "Resume after camera") already flag this as untouched by the mod and needing on-car observation.

Fits if: the phone still shows Android Auto as connected, audio may continue (or resumes), and long-press of the HOME button (Mazda's AA toggle, see https://www.mazdausa.com/siteassets/global-resources/vehicle-resources/mazda-connect/user-guides/android-auto-user-guide.pdf) or selecting Android Auto in the menu brings it back immediately without unplugging.

### Rank 2 (Group B): USB link drop on the retrofit hub when reverse is engaged

The reverse lamp circuit and the camera switch on when R is engaged. A marginal hub, cable, ground or supply can brown out or glitch at that moment, the phone re-enumerates, and the AOA session dies. After a USB re-enumeration the OEM stack may wait for a fresh connect or fall into its "stop and setup" or "reconnecting" path.

Evidence:
- Mazda's own AA FAQ treats "miscommunication" as "disconnect the cord from the USB port once and then reconnect it" (https://connect.mazda.com/en/smartphone-integration/android-auto/index.html).
- Retrofit owners report reverse-camera related trouble after hub installs (for example https://www.mazda3revolution.com/threads/issues-with-the-retrofit-kit-%E2%80%93-2017-mazda-3.260988/, title "Reverse Camera Blurry or Black Screen, Sensors Not Working in Reverse"; https://www.swedespeed.com/threads/android-auto-carplay-retrofit-prior-to-2015-5-model-years.645022/page-19, "blank screen (repeatedly) after reversing"). These are other cars or other kits, so they only show the class of problem exists. Not verified for a CX-3.
- miatafy notes firmware cannot fix "weak USB hub hardware" (https://miatafy.com/firmware/how-firmware-updates-work/).

Fits if: the phone's charging indicator blinks or restarts when R is engaged, the phone shows AA disconnected, and only unplug and replug brings AA back. It should also happen with a different cable less or more often.

### Rank 3 (Group B): phone side, the AA app stalls or is killed on the focus change

When the head unit takes video focus away, the AA projection on the phone goes to the background. Some phones' battery managers (Samsung, Xiaomi, Huawei, OnePlus) kill or freeze it; some AA app versions freeze on the transition.

Evidence:
- Google AA community thread (Jeep Renegade, same symptom: "when I switch to reverse ... on switching back android auto freezes both on the screen and on the phone", 31 "same question"): https://support.google.com/androidauto/thread/14629317/auto-crashes-when-switching-into-reverse-camera-and-back
- r/XVcrosstrek "Android auto often stops working when shifting": https://www.reddit.com/r/XVcrosstrek/comments/13nff7w/android_auto_often_stops_working_when_shifting/
- Common advice to set AA battery use to "Unrestricted" (XDA summary, https://www.facebook.com/xda.developers/posts/android-auto-kept-disconnecting-until-i-changed-one-hidden-setting/1357373629771812/).

Fits if: it depends on the phone (a second Android phone does not do it), the AA notification on the phone disappears or the phone shows an AA error, and clearing AA app cache, setting battery to Unrestricted, or updating AA changes the behaviour.

### Rank 4 (Group B): `blmjciaapa` tears the session down itself after the camera

If the phone does not answer the video focus change fast enough, or no video sink is available when the camera ends, OEM AA has a "device not responding", "no video sink" and "stop and setup" path (`NotifyDeviceNotResponding`, `NotifyNoVideoSinkSetup`, `NotifyStopAndSetup`, `DisplayReconnectingDialog` in `com.jci.aapa.xml.md`). Any of those can end in `RaceAap::UnInit` and `aap_destroy_session` (the shim hook point at `mazda/patches/blmjciaapa/lifecycle.cpp:303`).

Fits if: a Mazda dialog appears ("Android Auto not responding", "reconnecting", "stop the vehicle to set up", or similar) right after leaving R, and the phone shows the session ended. This overlaps with Ranks 2 and 3 and needs logs to separate (section 3).

### Rank 5 (Group A or B): MZD-AIO tweaks that do not belong on v74

Stock MZD-AIO does not support v74: `compatibility_check` in `MZD-AIO/.../tweaks/00_start.txt:81-118` returns 0 for v74 (only 55 to 70 are mapped, `00_start.txt:107`). The widely used v74 forum guide tells users to edit `run.sh` and replace `-eq 70` with `-eq 74` (https://www.mazda3revolution.com/threads/mzd-aio-tweak-on-fw-74.252435/). With that edit, v74.00.324 is treated as COMPAT_GROUP 6 (v70.00.xxx). Consequences:

- "No More Disclaimer" (`02_disclaimer-i.txt:15`), "Order of Audio Sources" (`09_audioorder-i.txt:14`) and the CASDK app-list patch (`00_59patch.txt:10`) overwrite `/jci/gui/apps/system/js/systemApp.js` with a copy built from v70.00.335 (`config/audio_order_AND_no_More_Disclaimer/.../systemApp.70.js`, `config/jci/gui/apps/system/js/systemApp.70.js`). Any v74 change in that file is lost. I found no reverse-camera or AA-resume code in the v70 copy (grep for camera, reverse, rvc, backup returns nothing), so camera return is most likely native MMUI, and this is a low-probability cause. It still cannot be ruled out without diffing against the stock v74 file (AIO keeps it as `systemApp.js.org` via `backup_org`).
- "Android Auto Headunit App" (`25_androidauto-i.txt`) installs the legacy `headunit` and adds `headunit-wrapper &` to the boot script (`25_androidauto-i.txt:54`). On v74 with OEM AA this is a second AA implementation competing for the same USB phone, and it has its own `bucpsa` camera handling (`headunit/mazda/callbacks.cpp:180-252`). Owners report it does not work on v70/v74 ("unstable connection" loops, same forum thread). If it is installed, a USB re-enumeration after reverse (Rank 2) could let the legacy binary grab the phone instead of `aapa`.
- "Remove reverse camera safety warning" (`03_warning-i.txt`) and "semi-transparent parking sensor graphics" (`04_sensor-i.txt`) only replace PNG files. Not a plausible cause.
- "Disable reconnecting message" (`00_reconnectmsg-i.txt`) only auto-answers the Bluetooth audio reconnect-failed dialog. Unlikely, but it hides a dialog that could be a clue.

Fits if: the problem started after the AIO install, or an "Android Auto" custom app icon (the legacy one) is visible in Applications, or the AIO uninstall or a restore of the original `systemApp.js` makes it go away.

### Rank 6: oem-aa-mod (future only)

Not installed, so not a cause today. If it is installed later, keep in mind for diagnosis: its per-session workers start on `aap_create_session` and stop on `aap_destroy_session` (`lifecycle.cpp:144-301`, `303-356`), and the wireless pairing bypass only acts on dongle device names (`bt16pair/bt16pair.cpp:160-200`). Nothing in it listens to `bucpsa`. Test with and without it once installed. (Minor doc nit found: `lifecycle.cpp:188` says ProjectionStatusCb is cb_list slot 4, while `bt16pair/bt16pair.cpp:26-36` says slot 3, which is the authoritative table.)

### How the owner can tell them apart (no tools)

Do each test 3 to 5 times, engine running, phone unlocked once at the start, Google Maps or music running in AA.

| Observation after leaving R | Points to |
| --- | --- |
| Phone still shows the AA notification as connected, and long-press HOME (or Menu, Android Auto) shows AA immediately | Group A, Rank 1 (screen or focus not restored) |
| Music or podcast from AA keeps playing through the car speakers while the screen shows a Mazda page | Group A, Rank 1 |
| AA icon in the Mazda menu is greyed out or missing, phone says AA disconnected, only unplug and replug helps | Group B (Ranks 2, 3, 4) |
| Phone charging symbol drops or restarts the moment R is engaged | Rank 2 (USB power or hub) |
| A Mazda dialog appears ("not responding", "reconnecting", "stop vehicle to set up") | Rank 4, maybe 2 |
| Phone shows an AA error, AA app restarts, or the phone screen freezes | Rank 3 |
| Only quick R to D shifts fail, slow ones (wait 3 s in R) are fine | Timing race, Rank 1 or 4 |
| R to P behaves differently from R to D | Rank 1 (matches the bulletin's R to P wording) |
| Same with a second Android phone and a short known-good cable | Rules out Rank 3 and the cable part of Rank 2 |
| Same with the other hub port, or with the phone on a battery-only test after the engine is running | Separates hub supply from cable |
| AA battery usage set to Unrestricted, "Start Android Auto automatically" on, issue unchanged | Weakens Rank 3 |
| A legacy "Android Auto" app icon shows in Applications (AIO headunit app) | Rank 5, uninstall it first |
| Started right after AIO, or gone after AIO full uninstall/restore | Rank 5 |

## 2. Can a shim in `blmjciaapa` fix it?

Short answer: for Group A, probably yes, with medium effort and one round of on-car logging first. For Group B caused by USB or phone (Ranks 2 and 3), no: a shim cannot keep a USB link or a phone process alive, it can only shorten recovery. For Rank 4 it might, once logs show which OEM path tears the session down.

### Hook points that already exist in the mod

- Session bracket: `aap_create_session` and `aap_destroy_session` PLT shadows in `mazda/patches/blmjciaapa/lifecycle.cpp:144` and `:303`.
- SDK callback table, 19 slots copied at create time (`bt16pair/bt16pair.cpp:26-36`): slot 0 OnErrorCb, 1 SessionStatusCb, 2 StreamRequestCb, 3 ProjectionStatusCb (events 0x400 NONE, 0x401 ON, 0x402 OFF, 0x403 SETUP, 0x404 CONFIG), 4 AudioStateCb, 5 BtPairingReqCb, 6 BtAuthStatusCb, 8 MediaPlaybackStatusCb, 9 PhoneStatusCb. Wrapping a slot transparently is proven by `bt16pair_pre_aap_create_session` (`bt16pair/bt16pair.cpp:332-356`).
- OEM state readers by anchor offset (`oem/blmjciaapa.h:80-139`, offsets in `oem/blmjciaapa.cpp:25-51`, harvested for 74.00.324A NA): `VideoManager_IsAAVideoInFocus`, `AudioManager_IsAAMediaInFocus`, `AapConnectionManager_connect_mode` (3 = activated), `AapConnectionManager_ActivateAapSession` (runs StartResourcesControl, ResumeLastMode, NotifyBTConnectionComplete).
- Passive D-Bus listener pattern on the HMI bus with raw libdbus: `playpause/playpause.cpp:86-97` and `:484` (match rule plus own connection and thread). `bucpsa` lives on the HMI bus (the legacy binary binds it to `hmiBus`, `headunit/mazda/callbacks.cpp:180`).

### Proposed "camera resume" module (design only)

1. On session up, start a watcher thread with match rules on the HMI bus:
   `type='signal',interface='com.jci.bucpsa',member='DisplayMode'` and `member='ReverseStatusChanged'`.
2. On `DisplayMode` going non-zero, record `was_aa_video = VideoManager_IsAAVideoInFocus(vm)` and `mode = connect_mode(cm)`.
3. On `DisplayMode` going back to 0, wait about 1000 to 1500 ms (the legacy binary's proven delay, `callbacks.cpp:219`), then check:
   - session still up (`connect_mode == 3`, no `aap_destroy_session` seen), and
   - `was_aa_video` is true and AA video is not in focus now.
4. If so, ask the OEM to bring AA back through the same path the user's long-press HOME uses. Options, safest first:
   - a. Replay the MMUI to aapa D-Bus call that the long-press HOME produces (likely `com.jci.aapa.VideoProjectionEventToMD(u)` with a specific value, or an MMUI `SelectAndroidAuto` event). The value must be captured with the logger in section 3 before coding. This uses the OEM's own state machine, so it is the least risky.
   - b. Call the OEM internal that `ActivateAapSession` uses to resume (`ResumeLastMode`). Its offset is not harvested yet and calling it out of sequence may confuse the connection manager. Only with a debug build and a log.
   - c. Do not send key events or fake projection events to the phone: the AA protocol video focus message is owned by `libaap_interface.so`, and its exported API is not mapped in this repo yet.
5. Rate-limit to one retry per camera cycle, and do nothing if the user deliberately left AA before reversing (`was_aa_video == false`).

Feasibility: medium. Everything except step 4a's exact value is already demonstrated in this codebase. Risk is low if limited to 4a. Effort: one logging drive plus about one day of shim work in a debug build. Must be gated by a new `libpatch.conf` key, default off.

### For Group B

- Rank 2 (USB): no shim fix. Fix hardware: different cable, hub connector reseat, hub ground and supply check, compare with the hub's other port. A shim could at most log the USB disconnect time to prove it.
- Rank 3 (phone): no shim fix. Phone settings and AA app version.
- Rank 4 (OEM teardown): first log the return address in `aap_destroy_session` (`__builtin_return_address(0)` minus the `blmjciaapa.so` base) and the OnErrorCb and SessionStatusCb codes. If the teardown comes from an OEM watchdog after the camera, a later shim could delay that decision, but intercepting teardown is risky and should not be done before the logs exist.

## 3. Minimal diagnostic logging

Two levels. Level 1 needs no compiled code and is enough to classify the problem. Level 2 is a debug-only shim addition for after Level 1. Both are listen-only. Nothing below was run.

### Level 1: D-Bus and USB logger script (adapt patrikguth's gist)

Base: https://gist.github.com/patrikguth/ae37a207e0044ef372cbfb7a1e9379a6 (listen-only `dbus-monitor` on `/tmp/dbus_service_socket` and `/tmp/dbus_hmi_socket`, `@uptime` timestamps, md5 self-check, MUTE twice as marker, sync every 5 s, stops when the stick is removed). Its notes say only unfiltered monitoring was reliable, so keep monitoring unfiltered and filter in awk.

Changes for this problem:

- Widen the awk keep-filter to: `bucpsa|ReverseStatus|DisplayMode|CameraOff|VIDEOCTRL|nativeguictrl|SetRequiredSurfaces|aapa|AOASession|VideoProjection|NotifyStopAndSetup|NoVideoSink|DeviceNotResponding|ReconnectingDialog|DisplayStopForSetup|ProjectionStatus|ClearLastFocus|SetRouteState|NotifyAudioFocusStatus|NotifyDeviceConnection|usbm|DeviceConnected|DeviceDisconnected|SelectAndroidAuto|[Mm]ute`.
- Add a third background loop that once a second appends `@uptime` plus any new kernel lines matching `usb` from `dmesg` (dedupe by remembering the last line), so a USB re-enumeration shows up next to the D-Bus timeline.
- Add a once-a-second snapshot of the AA processes (`pidof` for the aapa launcher and `aap_service`) so a crash or restart is visible.
- Owner procedure: start logger, connect phone, start AA music and Maps, then do: slow R to D, quick R to D, R to P, each 3 times, with MUTE twice before each test as a marker. Then long-press HOME once in the failed state and mark it with MUTE twice. That last step captures the exact D-Bus call needed for shim option 4a.

What to read in the logs:
- `DisplayMode 1` then `DisplayMode 0`, then whether `com.jci.aapa` sees `VideoProjectionEventToMD`, and whether the AA surface is requested again (`SetRequiredSurfaces`). Missing restore with a live session means Group A.
- `AOASessionStatus false`, `NotifyDeviceConnection false`, `usbm DeviceDisconnected`, or a dmesg USB disconnect within a second of R means Group B, Rank 2.
- `NotifyDeviceNotResponding`, `NotifyNoVideoSinkSetup`, `NotifyStopAndSetup` or `DisplayReconnectingDialog` before the drop means Rank 4.
- Nothing on the CMU side before the drop, and the phone ends the session, points to Rank 3.

### Level 2: debug-only shim tracing (after Level 1, if needed)

All in a DEBUG build of `libpatch-blmjciaapa`, behind a new config key, logging only:

- Wrap cb_list slots 0, 1, 3 and 4 (OnError, SessionStatus, ProjectionStatus, AudioState) the same way `bt16pair` wraps slot 3, and log event codes with uptime.
- In `aap_destroy_session` (`lifecycle.cpp:303`), log `__builtin_return_address(0)` relative to the `blmjciaapa.so` base so the OEM caller can be found in Ghidra.
- A `bucpsa` watcher thread (playpause pattern) that, on every `DisplayMode` change and at +1 s, +3 s and +5 s after it, logs `DisplayMode`, `IsAAVideoInFocus`, `IsAAMediaInFocus` and `connect_mode`.

This gives the exact sequence needed to decide between shim option 4a, a teardown investigation, or "not fixable in software".

## Recommended order

1. Owner does the no-tool tests in the table (Group A or B, phone, cable, AIO legacy AA app present or not).
2. If AIO's legacy "Android Auto Headunit App" is installed, uninstall it first and retest.
3. Run the Level 1 logger for one drive with the test procedure, including the long-press HOME capture.
4. Only if Level 1 shows Group A with a live session: build the camera-resume module as a debug build behind a default-off config key, and test on the car with the owner's agreement.

## Not verified

- No access to stock v74 `systemApp.js`, `blmjciaapa.so`, `libaap_interface.so` or the MMUI decision tables, so the OEM's actual camera-return logic, the `VideoProjectionEventToMD` values and the teardown triggers are unknown.
- No public report was found of this exact symptom on wired OEM AA with 74.00.324 on a CX-3. The ranking is based on related bulletins, other brands' reports, and code structure.
- reddit and Facebook threads could not be fetched in full (blocked), only their search snippets were used.
