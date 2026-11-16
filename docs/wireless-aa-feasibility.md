# Wireless Android Auto on a Gen 6 Mazda CMU: feasibility report

Target: Mazda CX-3, Mazda Connect CMU firmware 74.00.324, CarPlay/Android Auto USB hub retrofit, wired Android phone, oem-aa-mod installed.
Question: can the phone connect to Android Auto wirelessly and automatically when entering the car, using the CMU's own Wi-Fi and Bluetooth, without buying a dongle?

Method: desk research only. Local reference repos, the documented kernel config and D-Bus interfaces, reference implementations on GitHub, and community sources. Nothing was run on the car, and no code was changed. Every "confirmed" below means "confirmed in a document or source file", not "tested on this CX-3".

## Short answer

- On-CMU wireless Android Auto is technically plausible, and someone is already building it (Miatafy "AA Wireless", closed beta, not working end to end yet). It is not available today, and it is not something to build casually: it needs reverse engineering of JCI's Bluetooth and network services, and the CX-3's Wi-Fi radio may be the limiting factor.
- The "CMU joins the phone hotspot" route works only through Android Auto's developer "head unit server", which has to be started by hand on the phone. It is not usable as a daily auto-connect solution.
- Recommendation for daily use today: a wireless adapter. Cheapest reliable path is a DIY Raspberry Pi Zero 2 W running aa-proxy-rs (about 25 to 35 EUR all in), or a Carlinkit A2A (about 30 to 40 USD). AAWireless TWO (60 to 100 USD) if you want the best-supported commercial option. oem-aa-mod already supports dongles (`bt16pair`). Watch the Miatafy beta if you want the no-dongle option later.

## Background: how wireless Android Auto bootstraps

Verified from the reference implementations, not assumed:

1. The head unit (or dongle) advertises a Bluetooth RFCOMM service with UUID `4de17a00-52cb-11e6-bdf4-0800200c9a66` (the "AA Wireless" profile). WirelessAndroidAutoDongle registers it with BlueZ on RFCOMM channel 8 and also needs an HSP/HFP profile so the phone treats it as a car.
   Source: https://github.com/nisargjhaveri/WirelessAndroidAutoDongle, `aa_wireless_dongle/package/aawg/src/bluetoothHandler.cpp` (`AAWG_PROFILE_UUID`, `"Channel", 8`); same UUID in https://github.com/aa-proxy/aa-proxy-rs `src/bluetooth.rs`.
2. Over that RFCOMM socket the head unit sends `WifiStartRequest { ip_address, port }`, the phone asks for credentials, and the head unit answers `WifiInfoResponse { ssid, key, bssid, security_mode, access_point_type }`.
   Source: `aa_wireless_dongle/package/aawg/src/proto/WifiStartRequest.proto` and `WifiInfoResponse.proto` in WirelessAndroidAutoDongle.
3. The phone joins the head unit's Wi-Fi network and opens a TCP connection to the given IP and port (5288 by default in the dongle, `common.cpp`: `getenv("AAWG_PROXY_PORT", 5288)`). The normal Android Auto protocol then runs over TCP, with TLS, exactly as over USB.

Direction of the Wi-Fi link: in the native protocol the phone always joins the head unit's network. The credentials message carries the head unit's SSID, PSK and BSSID; there is no message for "head unit joins phone". Open Headunit (an Android app that implements native wireless AA) confirms this: its Native mode offers only "WiFi Direct group this unit creates" or "this unit's own hotspot". "Phone Hotspot" exists only in its deprecated Helper mode, which needs a companion app on the phone.
Source: https://github.com/andreknieriem/open-headunit/wiki/Settings (sections "Wireless Mode: Wireless Helper", "Wireless Mode: Native").

Other requirements: Android 11 or later on the phone for wireless AA (Android 10 on some Pixel and Galaxy models), per https://miatafy.com/mazda-connect/wireless-android-auto/. Google prefers 5 GHz. 2.4 GHz works but aa-proxy-rs warns it is "being deprecated by Google", can interfere with Bluetooth during pairing, and limits resolution on large screens (https://github.com/aa-proxy/aa-proxy-rs README, "Supported Hardware"). Open Headunit says 2.4 GHz is reliable up to 720p, so the CMU's 800x480 screen is within what 2.4 GHz can carry.

## (A) CMU Wi-Fi: hardware, driver, AP mode

What the docs say:

- Chip: "wl1285q-bt possibly made by Texas Instruments, uses wl12xx driver" (`~/Repos/mazda-reference/mazda-cmu-documentation/hardware.md`, section "Wireless"). This is the TI WiLink 7 family (Wi-Fi plus Bluetooth on one module). I could not find a public datasheet for "WL1285", so its band support is not verified. Most WiLink 7 parts are 2.4 GHz only; assume 2.4 GHz until proven otherwise.
- Kernel: Linux 3.0.35 (`kernel-config.md` header). `CONFIG_CFG80211=m`, `CONFIG_MAC80211=m`, `CONFIG_CFG80211_WEXT=y`, `CONFIG_NL80211_TESTMODE=y`, `CONFIG_WL12XX=m`, `CONFIG_WL12XX_SDIO=m`, `CONFIG_WL12XX_HT=y`. So the stack is the standard mac80211 one with nl80211, which is what hostapd needs. The upstream wl12xx driver of that era supports AP mode (with a separate AP firmware blob). `CONFIG_HOSTAP` is not set, but that is an old Prism2 driver and is unrelated to the hostapd daemon.
- Routing: `CONFIG_IP_ADVANCED_ROUTER=y`, `CONFIG_NF_NAT=m`, `CONFIG_IP_NF_TARGET_MASQUERADE=m`, `CONFIG_TUN=m`; `CONFIG_BRIDGE` is not set. Not needed for AA, which only needs a local IP link.
- JCI already has AP mode in its network manager. `com.jci.nms` (`dbus/com.jci.nms.xml.md`) exposes `WIFI_SetMode` / `WIFI_GetMode`, `AP_SetOpenProfile`, `AP_SetWepProfile`, `AP_SetWpaProfile ((suii)(is))`, `AP_SelectProfile`, `AP_GetInfo`, `AP_GetClients`, `AP_SetHiddenNetwork`, AP blacklist methods, `DHCP_ServerSetConfig` / `DHCP_ServerStart` / `DHCP_ServerGetClients`, `ActivateNAT`, `AdvancedRouting`, and signals `InterfaceWiFiModeChanged`, `WiFiAPClientConnected`, `WiFiAPClientDisconnected`, `APWpaInfo`. This is a complete soft-AP plus DHCP server API. Whether this firmware build actually ships the AP firmware blob and a hostapd-equivalent for this radio is not verified (needs a firmware dump: look for `wl1271-fw-ap.bin` or similar, and for a `hostapd` binary).
- Wireless CarPlay exists on some Gen 6 CMUs on firmware 74. Examples: MZD-AIO issue "Wireless CarPlay stop working on 74.00.324A ... CMU150" on a 2022 Mazda 2 (https://github.com/Trevelopment/MZD-AIO/issues/229); a warning that AIO tweaks on 74.00.331 "may disable wireless CarPlay" (https://www.mazda3revolution.com/threads/mzd-aio-tweak-on-fw-74.252435/page-16); oem-aa-mod issue #46 quoting that some AIO tweaks on 74.00.324 "remove CarPlay WiFi" (https://github.com/VitaliyKurokhtin/oem-aa-mod/issues/46). Wireless CarPlay requires the head unit to be the Wi-Fi AP, so the firmware 74 software stack does run the CMU as an AP on those cars.
- But the hardware differs. Mazdas247 lists part numbers "DEVB-66-9C0 ... WITH Carplay Wireless" and "DEVC-66-9C0 ... WITHOUT Carplay Wireless", and a regular there says wireless needs "a new CMU with correct Wireless internals" (https://mazdas247.com/forum/threads/mazda-connect-firmware-74-00-230a-now-324a.123875753/page-11). Only some late cars (2021 Mazda 6, MX-5, CX-5 Signature, 2022 Mazda 2) had it. A CX-3 with a retrofitted USB hub almost certainly has the non-wireless-CarPlay CMU, probably because the newer CMUs have a 5 GHz-capable radio. Miatafy also notes "Mazda used more than one Wi-Fi setup across these cars".
- Field reports on the older radio are poor: a Mazda statement that "electromagnetic noise generated in the CMU hardware may disturb the Wi-Fi connection" (https://www.mazda3revolution.com/threads/wifi-hotspot-problem.118594/page-3), users needing channel 11, and North American CMUs with weak Wi-Fi reception (comments in https://github.com/gartnera/headunit/issues/121).

Verdict (A): AP mode is very likely possible in software (driver class, nl80211, and JCI's own `com.jci.nms` AP and DHCP API). The risk is the radio: probably 2.4 GHz only, shared with Bluetooth on the same module, with documented noise problems. That is enough for 800x480 video in principle, but connection stability is unproven on this hardware.

## (B) Bluetooth: can a WPP RFCOMM server run next to JCI's stack?

What the docs say:

- Bluetooth sits on UART4, `/dev/ttymxc3` (`platform_data.md`), with an RF-kill GPIO (`GPIO4_IO16`, `DO_BT_EN`). The kernel has `CONFIG_BT=m`, `CONFIG_BT_RFCOMM=m`, `CONFIG_BT_HCIUART=m` with `CONFIG_BT_HCIUART_LL=y` (TI's HCILL protocol) (`kernel-config.md`).
- Userspace is JCI's own stack, controlled through `com.jci.bds` (device/service management, signals only in the docs) and `com.jci.bca` (connection agent: `ConnectRequest`, `SupportedProfiles`, `ReqStartAndroidAutoPairing`, `AAPairingSeqResult`). There is no BlueZ D-Bus API, and none of the documented methods registers a custom RFCOMM/SPP service UUID.
- It is not documented whether JCI's stack uses the kernel HCI layer (in which case a raw kernel RFCOMM listener could in theory coexist) or drives the UART itself in userspace (in which case it owns the controller exclusively). Either way, a second stack such as BlueZ would fight it for the controller, and the AA Wireless UUID also needs an SDP record, which only the owning stack can publish.
- OEM AA has no Wi-Fi bootstrap. oem-aa-mod's `bt16pair` exists because Mazda's `blmjciaapa` "has no such bootstrap" and gates session activation on its own in-band BT pairing result (`~/Repos/mazda-reference/oem-aa-mod/mazda/patches/blmjciaapa/bt16pair/bt16pair.h`). `com.jci.aapa` (`dbus/com.jci.aapa.xml.md`) has only BT pairing and AOA session methods, nothing about Wi-Fi.
- Prior art says it can be done through the factory stack: Miatafy reports that "the factory Bluetooth service advertises Android Auto and completes the opening setup with the phone", tested with Android Auto 17.3, while "Mazda keeps ownership of Bluetooth and Wi-Fi" (https://miatafy.com/aa-wireless/). That means someone has found a way to make JCI's stack publish the AA Wireless RFCOMM service. How (a hook inside the JCI BT daemon, an undocumented API) is not public.

Verdict (B): running a separate RFCOMM server next to JCI's stack is not a safe plan. The workable approach is the one Miatafy describes: make JCI's own Bluetooth service publish the AA Wireless UUID and hand the socket to a WPP handler, most likely with an LD_PRELOAD hook in the style of oem-aa-mod. That is reverse engineering work on an undocumented daemon.

## (C) Options ranked

### Option 1: CMU-native wireless AA (CMU as AP, WPP over JCI Bluetooth)

What it needs:
1. Publish the AA Wireless RFCOMM service through JCI's Bluetooth stack and run the WPP exchange (WifiStartRequest, WifiInfoResponse, and so on; message flow in aa-proxy-rs `src/bluetooth.rs`).
2. Put the radio in AP mode with WPA2 via `com.jci.nms` (`AP_SetWpaProfile`, `AP_SelectProfile`, `WIFI_SetMode`, `DHCP_ServerStart`), and report the real BSSID.
3. Feed the phone's TCP connection into the factory AA receiver (`aap_service` / `blmjciaapa`). Miatafy says the factory receiver is Google's wireless-capable receiver and "can accept a wireless connection" (confirmed on their bench). With a wired setup, oem-aa-mod's `bt16pair`-style gating fixes would also be needed because OEM `blmjciaapa` expects its own BT pairing step.
4. Keep wired AA working as a fallback, and handle reconnects, calls, and two-phone cases.

- Effort: very high. Months of reverse engineering on two undocumented JCI daemons, plus the AA receiver's wireless entry point. Miatafy has been at it for a while and is still at "turn on projection Wi-Fi and pass its details to the phone: in progress; phone joins the car's Wi-Fi and opens the first full wireless session: up next".
- Risk to the car: medium. Software-only and reversible if done as LD_PRELOAD shims, but it touches the Bluetooth and network daemons that also handle phone calls. A crash loop in those services is the main brick-adjacent risk. Opening a Wi-Fi AP also exposes the CMU's SSH and other services to anyone in range unless they are firewalled. Stock CMUs are known to use a well-known default root password, so WPA2 with a strong PSK plus firewall rules are mandatory.
- UX if it works: best possible. Phone connects on its own when you start the car, nothing plugged in.
- Hardware caveat: the CX-3 probably has the older 2.4 GHz-only radio with documented noise problems. Stability could be the real limit even with perfect software.

### Option 2: CMU joins phone hotspot, AA over TCP via "head unit server"

How it works: the phone turns on its hotspot; the CMU joins it as a client (it already does this for SSH); on the phone, Android Auto developer mode is enabled and "Start head unit server" is tapped; the head unit then opens a TCP connection to the phone on port 5277. The legacy headunit supports this: `common/config.cpp` (`wifiTransport`, `phoneIpAddress = "192.168.43.1"`) and `hu/hu_tcp.cpp` (`htons(5277)`) in `~/Repos/headunit`, added by gartnera/headunit PR #134 (https://github.com/gartnera/headunit/issues/121).

- Still supported? Yes, as a developer tool. Google's DHU guide still documents "Start head unit server" in Android Auto developer settings (https://developer.android.com/training/cars/testing/dhu, last updated 2026-07-13).
- Daily auto-start: no. The server has to be started by hand on the phone every time. A contributor in headunit issue #121 put it plainly: "more for development than every day use, because you have to start the HU server on the phone every time you want to connect". Open Headunit's "Headunit Server, Auto" mode only searches for a phone that is already running the server. Mazda users in that thread also reported the session getting sluggish after a few minutes of music, and dropping within 5 minutes on North American CMUs.
- Which receiver: on firmware 74 with the AA hub, the OEM receiver (`aap_service`) has no documented TCP client mode for this, so you would be back on the legacy `headunit` binary, which MZD-AIO lists as untested ("?") on firmware 74 (mazda3revolution FW 74 thread above) and which lacks oem-aa-mod's audio, HUD and touch work.
- Effort: low to medium (configuration plus getting legacy headunit running on 74 next to OEM AA).
- Risk to the car: low to medium (legacy headunit is installed via `stage_wifi.sh` edits, see `~/Repos/headunit/mazda/installer/tweaks.sh`).
- UX: poor. Hotspot on, developer menu, start server, every drive. Phone hotspot also uses mobile data for anything the CMU does on the network. Fine for a demo, not for daily use.

### Option 3: wireless adapter on the USB port (works today)

All of these present themselves to the CMU as a wired phone, so they inherit the working wired path. oem-aa-mod already handles them: `bt16pair` recognises "AAWireless", "smartBox", "carlink", "motorolama1", "AutoKit" and others for GAL 1.6 activation (`mazda/patches/blmjciaapa/bt16pair/bt16pair.cpp`, `kDongleDevNames`). Known open item: oem-aa-mod issue #51 (calls revert to the stock phone UI over wireless plus GAL 1.6), see `docs/fork-notes-cx3.md`.

| Adapter | Approx. price | Notes |
| --- | --- | --- |
| DIY Raspberry Pi Zero 2 W + aa-proxy-rs or WirelessAndroidAutoDongle | Board about 15 to 20 EUR, plus SD card and a short USB data cable: about 25 to 35 EUR total | Open source, active (aa-proxy-rs v1.2.0, Oct 2026). 2.4 GHz only on the Zero 2 W, fine at 800x480 but Bluetooth/Wi-Fi interference during pairing is a known issue on this board. Pi 3 A+ or Pi 4 add 5 GHz. Image from https://github.com/aa-proxy/aa-proxy-rs releases or https://github.com/nisargjhaveri/WirelessAndroidAutoDongle releases. Bonus: aa-proxy-rs MITM features (DPI change, video in motion, etc.). |
| Carlinkit A2A | 28 to 40 USD | Cheapest commercial, AA only. Avoid Carlinkit 4.0 (widescreen cut-off bug on Mazda). |
| Motorola MA1 | about 55 USD | Google-authorised, but Mazda owner reports are negative (dropouts needing replug, no firmware updates). MA2 announced at about 40 USD. |
| AAWireless TWO / TWO+ | 60 to 100 USD | Community favourite on Mazda, companion app with DPI override and tuning, OTA updates. aa-proxy-rs can also run on AAWireless hardware. |

Prices and Mazda-specific notes from https://miatafy.com/mazda-connect/wireless-android-auto/ and the project READMEs; check current prices before buying.

- Effort: minimal (flash an SD card, or plug in a commercial unit, pair once).
- Risk to the car: none beyond what oem-aa-mod already does. Fully reversible by unplugging.
- UX: good. Phone connects automatically on start, typically within 20 to 30 seconds (WirelessAndroidAutoDongle README: "connection under 30 seconds"). Downsides: something stays plugged in, phone battery drains on long drives, and some users see the occasional "Enable Android Auto" prompt (oem-aa-mod issue #46).

## (D) Prior art for on-CMU wireless AA

- Miatafy "AA Wireless" (https://miatafy.com/aa-wireless/): software for Gen 6 Mazda Connect on 74.00.324A, closed beta, free to join. Uses the factory Bluetooth service, factory Wi-Fi, and the factory AA receiver. Status as published: the phone finds the AA service over Mazda Bluetooth and completes the opening setup (confirmed with AA 17.3); wired AA still works with the layer loaded; turning on the car's projection Wi-Fi and handing details to the phone is in progress; the first full wireless session is "up next". Vehicle coverage will depend on which Wi-Fi hardware each car has. No public code.
- TikTok video linked from oem-aa-mod issue #52 (https://www.tiktok.com/@kid_mixer/video/7703591136882904340, via `gh issue view 52 -R VitaliyKurokhtin/oem-aa-mod`): a Vietnamese Mazda installer (tags CX-5, CX-8) advertising "stock, full-function wireless Android Auto, used alongside wireless CarPlay". No transcript or technical detail. The issue was closed with no comments. The CX-5 and CX-8 in that market have wireless-CarPlay CMUs, so even if real, it is on the hardware variant that the CX-3 probably lacks. Unverified.
- gartnera/headunit "AA over Wi-Fi" (https://github.com/gartnera/headunit/issues/121, PR #134): the developer-mode TCP route from Option 2. Worked, but manual and laggy. Not true wireless AA.
- I found no public, working, end-to-end implementation of native wireless AA on a Gen 6 CMU.

## (E) Recommendation

1. For wireless AA now: build a Raspberry Pi Zero 2 W adapter with aa-proxy-rs (cheapest, open source, tweakable), or buy a Carlinkit A2A or AAWireless TWO if you prefer plug and play. Keep oem-aa-mod's `bt16pair` enabled for GAL 1.6. If you hit 2.4 GHz pairing trouble on the Zero 2 W, move to a Pi 3 A+ or an AAWireless unit.
2. Do not pursue Option 2 for daily use. Manual server start every drive defeats the goal.
3. Treat Option 1 as a research project, not a near-term fix. If you want to explore it without touching the car, the cheap first steps are firmware-dump only: (a) confirm the exact Wi-Fi module and band in this CX-3's CMU (part number on the board or the CMU label, DEVB vs DEVC style suffix); (b) check the firmware dump for an AP firmware blob for wl12xx and for the binary behind `com.jci.nms` AP mode; (c) look for how `aap_service` accepts a non-USB transport. Before any car-side test, decide on AP security (WPA2, strong PSK, firewall SSH on the AP interface). Alternatively, sign up for the Miatafy beta and let them carry the reverse-engineering risk; check whether your CX-3's Wi-Fi hardware is on their supported list once they publish one.

## What was and was not verified

- Verified in documents or source: kernel Wi-Fi and Bluetooth config, `com.jci.nms` AP/DHCP API, absence of Wi-Fi in `com.jci.aapa`, the WPP UUID and proto messages, the head-unit-server TCP path in legacy headunit, adapter compatibility in oem-aa-mod, the published status of Miatafy AA Wireless.
- Not verified: the WL1285 band support; whether firmware 74 on a non-wireless-CarPlay CMU ships the AP firmware and can actually enter AP mode; whether JCI's Bluetooth stack drives the controller through the kernel HCI layer; the content of the TikTok video; current prices. No vehicle access was used.
