# Handoff: full DualSense support with an Artemide client and a Vibepollo host

Written 2026-11-19 at the end of a cloud session. No code has been changed in any repository yet.
Everything below is study and plan, checked against the source code of every repository named.

## Goal
Make a physical DualSense connected to an Android device running **Artemide** behave on the
Windows PC running **Vibepollo** as if it were plugged into the PC. That covers adaptive triggers,
HD haptics, speaker, rumble, lightbar/player LEDs, touchpad, gyro and battery. It only matters for
games that use these features.

## Constraints (from the user)
1. Vibepollo already has DualSense features. **Do not rewrite them**; only fill the gaps so the
   client's DualSense is picked up with all its features.
2. Target client: **Artemide**, `a-elkaim/moonlight-android`, branch `settings-import-export`
   (head `52fdf51`, 2026-11-13).
3. Add **as few dependencies as possible**. Reuse code from Vibepollo and Artemide, plus the
   Extended fork.
4. It must work over **USB** (controller cabled to the phone) and over **native Android Bluetooth**
   (Android 12+). **No USB Bluetooth dongle.**
5. **Out of scope for now:** microphone forwarding, and reporting headset-jack state to the host.
   Headset audio only needs to play from the controller's jack on the phone.

## Repositories and versions studied
| Role | Repository | Branch / commit |
|---|---|---|
| Host | `a-elkaim/vibepollo`, a fork of `Nonary/Vibepollo` | `master` @ `1f9ea16` (this session's branch: `ccr-19454a8c-xibsgw`) |
| Host's protocol library | `Nonary/moonlight-common-c` (Vibepollo's submodule `third-party/moonlight-common-c`) | `master` @ `6f05ee8` |
| Target client | `a-elkaim/moonlight-android` ("Artemide", based on Artemis 20.2.6) | `settings-import-export` @ `52fdf51` |
| Client's protocol library | `derflacco/moonlight-common-c` (Artemide's submodule) | `c999436` ("Add send empty payload method"); shares base `5f22801` with Nonary's copy |
| Source of the DualSense code to port | `a-elkaim/Artemis-Extended-Native-Wireless-DualSense-Features`, the user's fork of Taveszfito's | `dualsense-debug-and-native-touch` @ `b6fd417` (adds debug screen, direct-Bluetooth overlay, native touchpad capture) |
| Reference only | `Taveszfito/Apollo-Extended` | Not ported. The user has it working as a reference setup. |

## Key decision: the client learns Vibepollo's protocol
Apollo-Extended and Vibepollo both create a virtual USB DualSense on Windows, with a HID interface
plus 4-channel audio for speaker and haptics, using usbip-win2. Vibepollo's version is the
Microsoft-signed `usbip_ds5` backend (`src/platform/windows/dualsense_usbip_*`). Their *network*
protocols differ, so **Artemide is adapted to Vibepollo's protocol** and the host needs no change.

| Feature | Vibepollo protocol (used) | Apollo-Extended protocol (not used) |
|---|---|---|
| Adaptive triggers | `0x5503`, standard Sunshine message | same |
| HD haptics | `0x5601`: 2 channels (DualSense channels 3/4), 240 frames = 5 ms, 32-bit sequence. Needs session flag `ML_FF_HAPTICS_PCM` (0x04) and controller capability `LI_CCAP_HAPTICS_PCM` (0x8000) | `0x5504`: one 4-channel packet, always for controller 0 |
| Speaker | `0x5602`: 2 channels (channels 1/2), 240 frames, plus generation, sequence, sample position and owner. Needs session flag `ML_FF_CONTROLLER_SPEAKER_PCM` (0x08), capability `LI_CCAP_PS_SPEAKER_PCM` (0x4000, PlayStation type only), and an owner binding input packet (magic `0x55000009`). Spec: `docs/controller_speaker_pcm.md` | inside `0x5504` |
| Controller type request | none: Automatic + `LI_CTYPE_PS` → DualSense | `0xEC00` pattern in capability bits. **Do not port**: it clashes with 0x8000/0x4000 above. |

Don't port `0x5504`, `0xEC00` or Extended's `moonlight-common-c` commit `5fd8827`.

## Host (Vibepollo): no code changes expected
- Automatic gamepad mode plus a client reporting `LI_CTYPE_PS` already picks the composite USB
  DualSense when the optional usbip-win2 component is installed.
- Setup guide: `docs/windows-dualsense-haptics.md`. Options: `gamepad = usbip_ds5` or the per-app
  `dualsense-haptics` setting. Gain: `dualsense_haptics_gain`.
- Haptics are sent from `src/stream.cpp` (`send_feedback_msg`, about line 1380). The speaker
  packet encoder is `src/controller_speaker_pcm.h`.
- The debug log checkpoints for haptics are listed in `docs/windows-dualsense-haptics.md`
  ("Diagnosing missing waveform feedback").
- Not needed now: the virtual DualSense's mic input (endpoint 0x82 in
  `dualsense_usbip_protocol.cpp`) currently returns silence. If microphone support is added later,
  feed Opus-decoded client audio there (Opus is already linked). That avoids Apollo-Extended's
  Steam Streaming Microphone dependency.

## Client (Artemide): the work
Artemide today has **no DualSense code**: no USB driver, and adaptive triggers aren't passed from
the native layer to Java. It does report PlayStation controllers and the LED callback exists.

### Step 1: protocol library
Apply these 4 commits from `Nonary/moonlight-common-c` onto `derflacco/moonlight-common-c`
`c999436`. Their common base is `5f22801`, so cherry-picking should work.
- `dda8295` feat: add controller haptics protocol support (`ControllerHaptics.h`, `ControlStream.c`, `SdpGenerator.c`, `Limelight.h`)
- `8f3de30` feat: add ABI-safe controller speaker PCM callback (`LiStartConnectionWithSpeakerPcm`, `ControllerSpeakerPcm.h`, `Connection.c`)
- `f05b067` fix(protocol): scope speaker capability alias to PlayStation
- `cb15903` fix(protocol): bind speaker PCM to explicit client owner incarnation (`InputStream.c` owner binding)

Expect conflicts around the Steam controller start function (`LiStartConnectionWithSteamController`).
Artemide doesn't need the Steam callbacks; `LiStartConnectionWithSpeakerPcm` is enough. Keep
derflacco's commit `c999436`. The tests under `tests/` build on Linux and can be compiled to check
the merge.

**Where to put the result is undecided.** Options: (a) a new fork `a-elkaim/moonlight-common-c`
with Artemide's submodule pointed at it (recommended), or (b) a patch applied at build time.

### Step 2: native-to-Java glue
Files: `app/src/main/jni/moonlight-core/callbacks.c`, `simplejni.c`, `MoonBridge.java`,
`NvConnectionListener.java`, `Game.java`.
- Pass on `setAdaptiveTriggers` (the protocol library already parses `0x5503`).
- Pass on `controllerHaptics(controller, sequence, pcm, frames)`.
- Pass on the speaker callback `(controller, owner, generation, sequence, samplePosition, flags, pcm, frames)`.
- Start the connection with `LiStartConnectionWithSpeakerPcm`.

Artemide builds with `Android.mk`, not CMake. `libopus` (prebuilt) and OpenSSL are already there.

### Step 3: copy DualSense code from the Extended fork (`b6fd417`)
Under `app/src/main/java/com/limelight/` unless marked otherwise.

**Keep:**
- **Bluetooth:** `dualsense/DirectDualSenseBt.kt`, `DirectDualSenseBtHidBridge.kt`, `DirectDualSenseBtReportBuilder.kt`, `DirectDualSenseBtAudioHapticsBuilder.kt`, `DirectDualSenseBtCrc.kt`.
- **USB:** `dualsense/DualSenseWiredOutput.kt` (rumble, triggers, LEDs) and `binding/input/driver/DualSenseController.java` (only if raw USB input is needed; Android's own input handling may be enough).
- **Audio:** `dualsense/DualSenseAudioBridge.kt` (includes `DualSenseIsoNative` and `DualSenseBtAudioNative`) and `jni/moonlight-core/dualsense_iso.cpp`. The latter sends 4-channel audio to the controller's USB audio interface and Opus-encodes speaker audio for Bluetooth.
- **Testing:** `DualSenseDebugActivity.java`, plus its manifest entry and strings.
- **Related edits** to `ControllerHandler.java`, `Game.java`, `PcView.java` and `AndroidNativePointerCaptureProvider.java`, including the Android 17 native touchpad fix from `b6fd417`.

**Leave out:**
- the USB Bluetooth dongle bridge: `com/example/usbbtonandroid/**`, `dualsense/DualSenseBridge.kt`, `DualSenseBridgeActivity.java` (about 4k lines)
- the microphone files: `DualSenseMicrophone*.kt`, `DualSenseBluetoothMicrophoneTest.kt`, `microphone_capture.cpp`, `control_stream_extended.c`
- the keyboard/mouse mapper, Extended's controller-type request code, `0x5504`

**One new dependency:** `org.lsposed.hiddenapibypass:hiddenapibypass`. Native Bluetooth uses a
hidden part of Android's Bluetooth system (`BluetoothHidHost` raw reports). It can break on some
vendor Android versions or after updates; when it does, the controller falls back to a plain
controller with no triggers or HD haptics.

### Step 4: the one new piece of code, an audio adapter
`DualSenseAudioBridge` expects one 4-channel stream (`receive(controller, sequence, frameCount, channels, flags, pcm)`).
Vibepollo sends two 2-channel streams of 240 frames each:
- haptics (`0x5601`) → channels 3/4
- speaker (`0x5602`) → channels 1/2

The adapter joins them into 4-channel blocks and fills silence when one stream is missing, so
haptics-only must work. It uses Vibepollo's sequence numbers and the speaker's
generation/discontinuity flag, in place of Extended's 4-packet reordering. It also drops speaker
blocks whose owner doesn't match.

### Step 5: controller arrival (`ControllerHandler.java`)
Report `LI_CTYPE_PS` with touchpad, accel, gyro, RGB LED and battery. Add `LI_CCAP_HAPTICS_PCM`
and `LI_CCAP_PS_SPEAKER_PCM` **only when playback is actually ready**: the USB audio interface is
claimed, or direct Bluetooth is connected (`DirectDualSenseBt.isConnected`). Otherwise announce a
plain controller. Use the protocol library's arrival function, which sends the speaker owner
binding.

### Step 6: a setting
Add one toggle, "Native DualSense features", default off, so Artemide's latency work is unchanged
when it's off.

## Order of work and checkpoints
1. Step 1 + Step 2, then **adaptive triggers** over USB and Bluetooth. That's the smallest piece
   that tests the whole chain. Use `DualSenseDebugActivity` to test the controller on its own.
2. HD haptics through the adapter: USB first, then Bluetooth. Check the Vibepollo debug log lines.
3. Speaker, and speaker plus haptics at the same time.
4. **Hardware checklist:**
   - left and right haptics are distinct
   - triggers and input work while haptics play
   - lightbar and player LEDs, touchpad, gyro, battery
   - controller disconnect and reconnect; Moonlight's "keep controllers attached"; stream teardown
   - two controllers
   - stock Moonlight and other clients still work on Vibepollo
   - with the setting off, the controller behaves as a plain controller

## Environment needed (the cloud session had none of this)
- Android SDK + NDK and a JDK. The cloud container couldn't reach `dl.google.com`.
- Push access to `a-elkaim/moonlight-android`, ideally on a new branch off `settings-import-export`,
  and a home for the merged `moonlight-common-c` (see Step 1).
- A Windows host with Vibepollo and the optional "DualSense USB audio and haptics" component, an
  Android 12+ device and a DualSense.

## Other notes
- Installing the Extended APK was blocked by Play Protect, because it asks for accessibility,
  microphone and Bluetooth permissions. Fix: `adb install`, or turn off Play Protect scanning
  temporarily.
- Extended, official Artemis and Artemide all use the release package name `com.limelight.noir`,
  signed with different keys. Change `applicationIdSuffix` in `app/build.gradle` to install them
  side by side.
- Licensing: everything involved is GPL-3.0. Keep the original copyright and credit lines in ported
  files; Extended credits CloudPad-Android-Dualsense-Wireless for the Bluetooth method.
