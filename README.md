# Sony PS Link USB adapter kills Windows audio: root cause found (USB protocol violation), with fix

*Read this in [Português (Brasil)](README.pt-BR.md).*

> [!NOTE]
> **AI disclosure:** I am not a specialist in USB protocol, audio stacks or firmware analysis. This work is well beyond my personal technical depth. Both the investigation and this write-up were done with help from LLMs (Anthropic's Claude Opus 4.8 and Claude Fable 5). They drove the methodology, the tooling and the analysis, while I operated the hardware and made the decisions. Don't take any of us on faith: every claim here is backed by captures and reproducible experiments, so please verify.

> [!IMPORTANT]
> ### The human part of this story
> My real contribution here was curiosity and stubbornness. PlayStation support told me it was "a Windows issue" and closed the case. Later I learned that the LLMs would also give up if I let them: more than once they concluded that unless Sony updates the firmware there was nothing to be done, and suggested I settle for an auto-recovery watchdog as the final answer. I had to insist quite a bit for the investigation to keep going after that, and the root cause in this document was found because it did.
>
> The evidence was sitting on the wire, available to anyone with the right tools. It just needed someone willing to keep asking.

**TL;DR:** If your PULSE Elite / PULSE Explore headset (via the PS Link USB adapter) randomly kills ALL Windows audio — especially when changing volume — it is not your drivers, not Realtek, not Windows. The adapter's firmware violates the USB spec: it transmits **256-byte bursts on an interrupt endpoint it declared as 64-byte max**. Windows detects this as **babble** (`USBD_STATUS_BABBLE_DETECTED`), resets the pipe, and when audio is streaming, that error recovery takes the audio stream down with it — cascading into a full `audiodg.exe` lockup. **Workaround below (no software needed) — audio becomes stable and the volume buttons keep working.**

- Device: PlayStation Link USB adapter, `VID_054C PID_0ECC`, firmware/bcdDevice **1.43** (`REV_0143`, latest at time of writing). **Every finding in this document was captured on firmware 1.43 specifically** — other versions may or may not behave the same. To check yours: Device Manager → the adapter's "PlayStation Link" entry → Properties → Details → Hardware Ids → the `REV_xxxx` suffix is the firmware (e.g. `USB\VID_054C&PID_0ECC&REV_0143`). If you have a different version, reports either way are very welcome.
- Host: Windows 11 (build 26200), Microsoft inbox drivers only (`usbaudio.sys` + `HidUsb`) — Sony's PC app is NOT in the failure path (verified: the freeze reproduces with it killed)
- Reproduced on **two different Windows laptops** (different hardware), which rules out a single flaky PC or USB controller as the cause. The detailed bus-level captures in this document were taken on one of them.
- Evidence: USB bus captures (USBPcap), ETW audio traces, an `audiodg.exe` hang dump, and controlled A/B experiments. Raw captures available on request (device serial redacted).

> **Note on PlayStation support:** I contacted PlayStation support about these freezes long before any of this analysis existed. The case was dismissed as *"a Windows issue"* and support was denied — no escalation path was offered. Everything below exists because that door was closed. As the captures show, it is not a Windows issue: Windows is the side detecting and correctly recovering from the device's protocol violation.

---

## Symptom

While audio plays through the PS Link adapter, Windows audio dies system-wide: every app goes silent, the volume UI still moves but nothing plays, and only killing `audiodg.exe` (or restarting the audio services / toggling any audio effect that rebuilds the audio graph) brings sound back. Triggers: changing headset volume, muting the mic, moving any slider in Sony's app — but it also happens "spontaneously" during plain playback. In my case: median ~7 minutes between freezes during active use (48 freezes in one ~7 h session, all logged).

Searches show scattered reports of "Pulse Elite audio cutting out every few seconds on PC" that match the milder form of this failure. Most people blame Windows or their drivers and move on. The mechanism below explains both the mild and the severe form.

## Root cause (captured on the wire)

The adapter is a USB Full-Speed composite device: an audio function (interfaces 0–2, standard USB Audio Class 1.0, driven by Microsoft's inbox `usbaudio.sys`) and one HID function (interface 3, driven by inbox `HidUsb`). The HID interface declares an interrupt IN endpoint 0x81 with `wMaxPacketSize = 64`, and its largest input report (ID 0xB0) as 64 bytes.

**Every time the headset state changes** — volume button pressed, mic muted, sidetone slider moved, EQ preset changed (each of these updates the device's 64-byte state report 0xB0) — **the firmware pushes a 256-byte burst (4 identical concatenated copies of the 0xB0 report) through that 64-byte endpoint.**

That is a hard USB protocol violation, and the host flags it as exactly that:

```
interrupt IN completion, EP 0x81, length=256, USBD_STATUS = 0xC0000012 (BABBLE_DETECTED)
→ pending URB canceled (0xC0010000)
→ URB_FUNCTION_ABORT_PIPE
→ URB_FUNCTION_SYNC_RESET_PIPE_AND_CLEAR_STALL
```

I captured this identical sequence **9 times in one session** — once per button press / slider move, deterministically.

A note for anyone about to say "babble is usually an electrical/cabling/host-controller fault, not the device": here it is not. It is **deterministic** (one per state-change event, never at random), the reported length is **exactly 4× the device's own declared report size** (256 = 4 × 64) rather than a garbled or truncated length, and the payload is **four clean, aligned copies of report 0xB0**, not corrupted data. Electrical babble does not produce four byte-perfect repetitions of a specific HID report on cue. This is the firmware deliberately emitting an oversized transfer.

**Why it kills audio:** when no audio is streaming, the abort/reset recovery is harmless (all 9 instances above had no audible effect — the audio stream was closed at the time). But when the isochronous audio stream **is** active on the same Full-Speed device, the babble recovery disrupts the device's USB service and the audio stream stops completing:

- In WASAPI **exclusive** mode (audio engine bypassed): the player dies with *"Unrecoverable playback error: Waiting for hardware timed out"* — I captured a babble event at the **exact second** of the button press that killed the stream, followed ~2 s later by the stream teardown (`SET_INTERFACE alt 0`). The Windows glitch-event channel logged **zero** events — proving the failure happens below the audio engine.
- In normal **shared** mode: the stalled stream surfaces as `KS "BASE Output Unexpected Buffer Completed"`, then a ~130 events/s glitch storm (`Microsoft-Windows-Audio/GlitchDetection`), then `audiodg.exe` worker threads deadlock — the system-wide freeze.

This also explains the "spontaneous" freezes (the device occasionally updates its state report on its own — e.g. battery level — emitting the same malformed burst) and why the buttons feel like "reliable triggers" (every press emits one).

**Confirmation by elimination (A/B):** with the HID interface disabled on the host (see workaround) — so the interrupt pipe is never polled and the device physically cannot transmit the burst — the freeze became **unreproducible** under the exact button-hammering that reliably killed audio within 1–3 minutes before. Tested with continuous exclusive-mode playback.

Secondary spec violation, for completeness: the adapter's isochronous audio OUT endpoint is declared **asynchronous** but provides **no feedback endpoint whatsoever** (neither explicit nor implicit; `bSynchAddress = 0`), which USB 2.0 §5.12.4.2 requires for async sinks. Not the trigger here, but indicative of the firmware's USB compliance level.

**Fault attribution:** the device violates the spec; Windows detects the violation and recovers the pipe by the book. The only thing arguably on Microsoft's side is severity — the glitch storm escalating to an `audiodg` deadlock instead of a graceful stream restart. The firmware was presumably designed against Sony's own host stack (PS5 / PlayStation Portal), where nobody enforces the HID declaration — on Windows, the inbox class driver does.

## Limits of this analysis (what I did NOT prove)

In the interest of honesty, and so nobody has to point these out for me:

- **The internal causal link is inferred, not captured.** I proved that the babble and the audio death are tightly correlated in time, and that removing the HID interface removes the freeze. I did **not** capture the exact mechanism *inside* the drivers by which the interrupt-pipe error recovery disrupts the isochronous audio pipe — that step is a reasoned inference from the timing, the error sequence, and the A/B result. It is also possible (though I consider it less likely) that the babble and the stall are two symptoms of one deeper device fault rather than one causing the other. Either way, the practical conclusion holds: no HID polling, no freeze.
- **USBPcap captured no isochronous packets on this system.** So I could not directly watch the audio stream's timing degrade on the wire; the audio-side evidence is from ETW (the KS glitch signature) and the player-level failure, correlated by timestamp with the babble. A hardware USB analyzer would settle the internal mechanism definitively.
- **Fault is shared, not 100% Sony.** The device violates the spec and is the root trigger, but a case can be made that Windows' class drivers could isolate a HID-endpoint error from an unrelated audio function on the same composite device more gracefully. I attribute the *trigger* to Sony's firmware and the *severity* (full audiodg deadlock vs. a brief stream restart) partly to how Windows handles it.
- **Sample scope.** One headset unit, firmware 1.43, two laptops, one investigator. The spec violations are inherent to the device's descriptors (independent of my machines), but I have not tested multiple headset units or firmware versions. Independent reproduction — especially on other firmware — is exactly what would strengthen or bound this.

None of these weaken the headline finding or the workaround. They mark the boundary between what is captured fact and what is reasoned from it.

## Why the buttons still work without any of this

A bonus finding: the volume buttons are processed **entirely inside the headset** (its DSP applies the level locally; the state report merely informs the host). Sony's PC app only mirrors that state to the Windows volume slider. This means the workaround below does NOT sacrifice the volume buttons.

## Workaround (no software, reversible, tested)

Disable the adapter's HID interface so Windows never polls the malformed endpoint. The adapter has exactly one HID interface (interface 3, "MI_03" in its hardware ID), so there is only one correct target. Close Sony's "PlayStation Link" app first if it is running (it holds the device open and makes the disable fail).

**Option A — Device Manager (easiest to get right):**

1. Open Device Manager (`devmgmt.msc`) → menu **View → Devices by container**.
2. Expand the **"PlayStation Link Adapter"** container. You'll see the audio entries plus one **"USB Input Device"**.
3. Right-click that "USB Input Device" → **Disable device**.

(If you prefer the default view: it's under Human Interface Devices, among possibly many identical "USB Input Device" entries. Pick the one whose Properties → Details → **Hardware Ids** shows `USB\VID_054C&PID_0ECC&MI_03`. The "MI_03" never appears in the device's display name, only in that property.)

**Option B — PowerShell as Administrator (finds it by ID, no ambiguity):**

```powershell
Stop-Process -Name 'PlayStation Link' -Force -ErrorAction SilentlyContinue
Get-PnpDevice | Where-Object InstanceId -like 'USB\VID_054C&PID_0ECC&MI_03*' |
    Disable-PnpDevice -Confirm:$false
```

To undo either option later, right-click → Enable device, or run the same PowerShell with `Enable-PnpDevice`.

Results and costs:

- Audio keeps working (different interface, untouched). Volume buttons keep working (handled in-headset). The freeze mechanism is physically eliminated.
- While disabled, Sony's PC app can't see the device (no sidetone/EQ changes, no battery readout, no firmware updates). Re-enable temporarily to change those, then disable again.
- Survives reboots. A different adapter unit (different serial) needs the disable applied once again.

## Reproduction (for anyone who wants to verify)

0. Check your firmware version first (see the `REV_xxxx` note at the top). Everything below was verified on **1.43**; a different version might not reproduce, and knowing which versions do or don't is valuable data in itself.
1. Play continuous audio through the PS Link adapter (any mode; WASAPI exclusive makes the death legible as a clean player error).
2. Press volume +/− on the headset repeatedly (every ~10 s). Typical time-to-freeze here: under 3 minutes.
3. For wire-level proof: capture with USBPcap on the adapter's root hub while doing (2); filter the adapter's device address; observe `0xC0000012` completions (256 bytes) on EP 0x81 at each press, and correlate the audio death with one of them. (Note: USBPcap did not capture isochronous packets on my system — not needed; the babble + pipe-reset sequence and the timing correlation are visible regardless.)
4. Then disable the MI_03 HID child and repeat (2) — the freeze should not reproduce.

## What Sony should fix

Frame the interrupt IN transfers correctly: deliver the 0xB0 input report as a single ≤64-byte transfer per poll (or declare a larger `wMaxPacketSize`/report size consistently). One-line description; firmware-only; no hardware change needed. Fixing the missing async feedback endpoint would be nice too.

## Corroborating signs this was a low-priority PC port

None of these are the bug — the bug is the babble above. But taken together they suggest the Windows side did not get much attention, which is context for how something this reproducible shipped:

- **The firmware violates the USB spec in two independent places.** The 64-byte interrupt endpoint that transmits 256 bytes (the freeze), and the asynchronous audio OUT endpoint that ships with no feedback endpoint at all (`bSynchAddress = 0`, no explicit or implicit feedback), which USB 2.0 §5.12.4.2 requires for async sinks. Both are visible in the device's own descriptors.
- **The shipped, WHQL-signed driver package still carries a developer test comment.** In Sony's Extension INF (`PlayStationLink.inf`), right above the line that sets the device's friendly name:
  ```
  [Dev_AddReg]
  ; Just to test that it installed
  HKR,,FriendlyName,, "PlayStation Link"
  ```
  A leftover "just to test" note is harmless, but it is not something you expect to survive into a signed production release that ships to every PC user via Windows Update.

To be fair, the same package does some things right (`DriverIsolation`, `PnpLockDown`). The point is not that Sony is incompetent; it is that the PC port looks like it was validated lightly — consistent with a device designed for the PS5, where Sony controls the host and none of the above would ever surface.

---

## The investigation journey (how we got here)

This was not found in an afternoon. The trail, in order — including the wrong turns, because they are part of the evidence:

1. **May 2026 — first deep dive.** After months of freezes: ETW audio tracing + a memory dump of the hung `audiodg.exe`. Found the failure signature (`BASE Output Unexpected Buffer Completed` → glitch storm → deadlocked engine threads). First suspect — Realtek/third-party audio effects (APOs) — **exonerated**: removing them changed nothing (disabling effects only "fixed" things because it rebuilds the audio graph, which is what any recovery does). PlayStation support contacted; case dismissed as "a Windows issue"; support denied.
2. **May–July 2026 — the band-aid era.** Built a watchdog that detects the glitch storm and auto-restarts the audio engine (~1 s recovery). It went through four generations (event-channel polling; immunity to system clock jumps; a global-hotkey path for freezes the channel never logs; a service-restart fallback for when `audiosrv` itself wedges). It made life bearable — median time between freezes in active use was ~7 minutes — but it is recovery, not cure.
3. **July 21 — full reconnaissance of the adapter.** Fresh USB descriptor dump (all interfaces/endpoints, both configurations), the complete 951-byte HID report descriptor, and a live decode of the vendor HID protocol: state report `0xB0` (buttons bitfield, headset volume level 0–13, mic-mute state), config report `0xD0` (sidetone level, EQ presets), confirmed byte-for-byte on the bus. Discovered Sony's PC app never receives button *input reports* — it polls `GET_REPORT(0xB0)` ~4×/s. First sighting of the malformed 256-byte interrupt bursts — noted as a curiosity, not yet understood as the murder weapon.
4. **Key negative result:** killing Sony's PC app entirely did **not** stop the freezes. That exonerated Sony's *software* and every "sanitize the traffic" mitigation idea — the problem had to live in the device or below.
5. **Clock-drift hypothesis.** The descriptor dump showed the async audio OUT endpoint has **no feedback endpoint at all** (spec violation) — a structurally guaranteed clock-drift scenario, and for a while the leading theory. It explained the passive freezes and the post-recovery "grace window". It was wrong about the main mechanism, but it motivated the decisive experiment.
6. **Night of July 21 into the early hours of the 22nd — the decisive night.** WASAPI-exclusive playback (audio engine bypassed) died on a volume press within a minute — with the Windows glitch channel **completely silent**. Bus capture of the next kill: `BABBLE_DETECTED` on EP 0x81 at the exact second of the fatal press. Re-analysis of the previous day's capture: the same babble + abort/reset sequence at **every** button/slider event — harmless then, because no audio was streaming. Final confirmation: with the HID interface disabled (interrupt pipe never polled → babble physically impossible), button-hammering could no longer kill audio — and the volume buttons kept working, revealing they are processed inside the headset all along.
7. **Grace-window explained:** after each crash the babbled pipe stays down until the HID driver reopens it; while it is down, no IN tokens → no babble possible → buttons "safe" for a few seconds. Every observed behavior now has one mechanism.

Tools: USBPcap + Wireshark, UsbTreeView, Windows ETW (`Microsoft-Windows-Audio/GlitchDetection`), WinDbg (dump analysis), Python (custom pcap/USBPcap parsers, HID enumeration and access-testing via ctypes, Core Audio monitoring via pycaw).

## Appendix — key captured evidence

Babble sequence at a state change (repeated identically at every button/slider event; timestamps relative to capture start):

```
48.736  EP 0x81 interrupt IN  cpl  status=C0000012 (BABBLE_DETECTED)  len=256
        payload: b0 50 01 4e b4 be 1a bf ... (4x the 64-byte report 0xB0)
48.737  EP 0x81               cpl  status=C0010000 (CANCELED)         len=0
48.737  URB_FUNCTION_ABORT_PIPE (EP 0x81)
48.934  URB_FUNCTION_SYNC_RESET_PIPE_AND_CLEAR_STALL (EP 0x81)
```

The kill, captured live (WASAPI exclusive playback; second volume press of the round):

```
189.897  EP 0x81 interrupt IN  cpl  status=C0000012  len=256   ← the babble (= the button press)
189.897  EP 0x81               cpl  status=C0010000  len=0
189.897  ABORT_PIPE / 190.087 RESET_PIPE_AND_CLEAR_STALL
191.942  SET_INTERFACE(if=2, alt=0)                            ← audio stream torn down (~player timeout)
194.902  EP 0x81 interrupt re-submitted … never completes again
Player error: "Unrecoverable playback error: Waiting for hardware timed out"
Windows glitch channel: zero events (audio engine not involved)
```

Endpoint declarations (from the device's own descriptors, active configuration):

```
EP 0x81  interrupt IN   wMaxPacketSize=64   ← the endpoint being babbled
EP 0x0C  iso OUT  async wMaxPacketSize=196  bSynchAddress=0, no feedback EP anywhere
EP 0x8C  iso IN   sync  wMaxPacketSize=96
HID report 0xB0 (input): declared 64 bytes  ← delivered as 256
```

Shared-mode failure signature (ETW, from the May diagnosis of the same freeze):

```
KS Endpoint Glitch: BASE Output Unexpected Buffer Completed
→ ~130 glitch events/s (Microsoft-Windows-Audio/GlitchDetection, IDs 38/40/41)
→ audiodg.exe worker threads parked (WaitForMultipleObjectsEx / EventPairLow) — system-wide audio dead
```

*Full pcap captures, ETW traces, the audiodg dump, and the HID report-descriptor dump exist and can be shared (device serial redacted). Happy to answer questions or help anyone reproduce.*
