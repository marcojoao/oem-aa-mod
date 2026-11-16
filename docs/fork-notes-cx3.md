# Fork notes: goals for a Mazda CX-3 (2026-11-16)

Owner goals: (1) Bluetooth: when the phone reaches the car, prompt/auto-connect Android Auto; (2) after the reverse camera, resume the Android Auto state; (3) better sound control.
Desk research only. Nothing below has been run on a car.

## What upstream already covers

| Goal | Upstream status |
| --- | --- |
| BT / wireless connect | `bt16pair` activates the AA session over a wireless dongle at GAL 1.6 (skips the in-band BT pairing the phone omits). Wired AA uses Mazda's own pairing in `com.jci.aapa`. |
| Resume after camera | Not touched by the mod. Stock firmware handles video focus around the reverse camera. Needs on-car observation before deciding if anything is wrong. |
| Sound | `aa_audio_low_latency` (prompt cut-off fixes), `mute_pauses_phone`, `unmute_starts_playback`, `block_headunit_media_play`. No volume ducking or per-stream volume. |

## Open bugs in our area

- **#51 call reverts to stock Mazda UI (wireless + GAL 1.6).** Analysis by patrikguth: `libjciuiaphone.so` sets its "AA active" flag only in `UIA_PHONE_AAPairingSeqResultCb` (signal `BCA_AAPairingSeqResult`). `bt16pair` bypasses pairing and calls `ActivateAapSession` directly, so that signal never fires and calls go to the stock phone UI. Wired works (confirmed by a user). Candidate fix: after the bypass, emit or synthesize the pairing-result signal. Best first contribution.
- **#45 "audio bluetooth disconnected" after ~30 s on a dongle.** Likely the same missing pairing state; check after #51.
- **#46 "Enable Android Auto" prompt on start.** Regressed in 0.10.0.

## Build (verified on Apple Silicon)

The toolchain submodule is x86-64 Linux. Build in an amd64 container:

```bash
git submodule update --init --depth 1
docker run --rm --platform linux/amd64 -v "$PWD":/src -w /src debian:bookworm-slim \
  bash -c 'apt-get update -qq && apt-get install -y -qq make && make -C mazda -j4'
```

Output: `mazda/build/release/libpatch-{blmjciaapa,aap_service,svcjcinavi}.so`, ARM EABI5, built from `f8ac5b0`.

## Prerequisites on the car

Firmware 74.00.324 or 74.00.324A and the CarPlay/Android Auto USB hub. Wireless also needs a dongle (AAWireless or similar).
