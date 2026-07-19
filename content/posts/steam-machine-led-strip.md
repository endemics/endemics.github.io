---
title: "Reverse-Engineering the Steam Machine's LED Strip: From a Product Photo to a Kernel Module"
date: 2026-07-19T10:52:46Z
draft: false
categories: ["Technology"]
tags: [ "Linux", "WLED", "Steam", "Valve" ]
---
(Claude helped with both the project and this post)

It started with a spec sheet and a dumb question: *how tightly packed are the LEDs in the Steam Machine's front light bar?* It ended with a cloned kernel driver, a strings dump of the Steam client itself, and a from-scratch Linux kernel module driving a WLED replica. This is the story of everything in between.

## Act 1: What can you learn from one product photo?

The Steam Machine's spec sheet gives you two numbers: **17 individually addressable RGB LEDs**, and a case that's **156mm wide**. Divide one by the other and you get a density - about 109 LEDs per meter, roughly one LED every 9mm. Simple enough, if you're willing to assume the strip runs edge-to-edge with zero bezel. It doesn't, of course. Nothing does. So the real question became: how much margin is there, actually, and can we figure it out without a teardown?

Turns out you can get surprisingly far with one promotional photo and Claude doing some pixel math, provided you have something in frame with a *known* real-world size to calibrate against. The photo had two USB-A ports on it. USB-A connectors are standardized at 12mm × 4.5mm. Find their pixel width, divide, and you have a scale factor for the whole image - no assumptions, no guessing.

![USB-A ports used as a size reference, cross-validated against the case-width scale](/images/01-usb-calibration.png)

Both ports came out **exactly** 49×19 pixels, and the resulting scale (0.2449 mm/px) matched the one derived independently from the case's overall width to four decimal places. That's not a coincidence - it meant the photo had no meaningful lens distortion across its frame, which mattered a lot for trusting every measurement that came after it.

Then things got weird. Tracking the lit portion of the strip gave a *sharp* cutoff on one side and a long, soft, gradual fade on the other - not remotely symmetric, which makes no sense for a centered product feature.

![Brightness profile across the strip: sharp on the right, a long soft ramp on the left](/images/02-brightness-profile.png)

Claude's hypothesis - a diffuser panel unevenly lit - turned out to be wrong. The real explanation: the promotional photo had caught the strip **mid-animation**, a "swoop" effect sweeping across it, photographed with a comet-tail trail. The sharp edge was the leading edge of that animation; the soft fade was its trailing tail. Neither edge was the *strip's actual physical boundary*.

That distinction mattered enough to re-derive everything. The fix: instead of tracking the *light*, track the **physical seam** in the case - the recessed channel is visibly darker than the surrounding material regardless of whether an LED underneath happens to be lit that frame. Using a reference point unrelated to the animation (the case's power button, whose exact center a circle-detection pass pinned to pixel-level precision) helped triangulate where the animation's visible edge sat relative to the strip's true physical end.

![The power button's detected center (magenta) sitting between the animation's visible cutoff (yellow) and the true physical groove edge (green)](/images/03-swoop-vs-button.png)

The corrected numbers came out almost perfectly symmetric - a ~2mm margin on each side, a channel running **~152mm** out of the 156mm case width. Recompute the density with *that* number instead of the naive full-width guess, and you land at **~112-116 LEDs/m** - reassuringly close to the very first back-of-envelope estimate, despite a much shakier hypothesis (144 LEDs/m, then 120 LEDs/m) sitting in between.

![The corrected measurement: physical groove edges (green) vs. case edges (red), essentially symmetric](/images/04-physical-groove.png)

## Act 2: What kind of LEDs, physically?

Here's a detail worth pausing on: the strip's error states render as sharp, crisp red text-like patterns. If the LEDs sat behind a single shared diffuser panel - the kind of thing that blends neighboring points into a soft glow - that sharpness wouldn't be possible. Diffusion is, definitionally, blending. A ~2mm-wide sharp transition on an ~8.6mm LED pitch (about a quarter of the spacing between LEDs) is strong physical evidence for **individually diffused packages** - each LED with its own small dome or frosted cap, not a continuous light-guide running the length of the strip.

That single observation - "the error text is too sharp for a shared diffuser" - turned out to be a surprisingly strong constraint. It ruled out an entire category of construction before we'd bought a single component.

Shopping for a physical match produced its own lesson. The first LED strip that matched the target density (120 LEDs/m) used an **SMD2020** chip - a 2.0mm×2.0mm package - not the SMD5050 (5.0mm×5.0mm) package assumed going in. A second listing had the right SMD5050 package but turned out to be a classic 12V/24V analog RGB strip, wired in fixed 3-LED groups with one shared color - **not individually addressable at all**, and therefore physically incapable of reproducing the crisp per-pixel animations we'd just used to rule out the shared-diffuser theory in the first place. The part that actually fits the bill is the overlap of both: an individually-addressable WS2812B/SK6812-class strip built on the SMD5050 package, at 120 LEDs/m. No SMD5050 strip at 120 LEDs/m ever turned up, so the mockup will run on an SMD2020 one at that density instead - narrower package than the real thing, but the closest match available while the search for the right part continues.

## Act 3: Is any of this actually controllable from software?

The Steam Machine's light bar clearly does more than glow one color - it tracks download progress, runs a boot animation, and is user-customizable. That means somewhere, there's a real software interface. Finding it took a few wrong turns before the right one.

**First wrong turn: assuming it's `steamos-manager`.** Valve's own cross-device system daemon is open source, and it seemed like the obvious place to look. Cloning it and grepping the entire D-Bus interface definition for anything LED-related came up completely empty.

**A real clue, from actual news:** a well-publicized "Red Line of Death" incident (a firmware bug that made the red error indicator trigger at the wrong temperature threshold) confirmed that error-state handling genuinely lives in **BIOS/EC firmware** - RAM-detection failures and CMOS-reset menus happen before any OS loads, so that part of the logic has to be pre-boot. Meanwhile the customizable, download-progress-tracking behavior is clearly OS-level. Two owners, one physical LED strip, depending on system state.

**The actual find:** Valve open-sources its kernel patches. A quick search of their GitHub org turned up `fremont-hw-support` ("Fremont" being the Steam Machine's internal codename) - which turned out to be just a firmware-update distribution config, not the driver itself. But cloning the full kernel integration tree and grepping for the codename led straight to it:

```c
drivers/leds/rgb/leds-valve.c
```

A real, GPL-licensed, in-tree-style Linux kernel driver, written by a Collabora engineer under contract to Valve. It confirmed, in one `#define`, something we'd been estimating for an entire blog post's worth of photo forensics:

```c
#define VALVE_NUM_LEDS 17
```

Better still, it's a standard `led-class-multicolor` driver - meaning on real hardware, you'd see `/sys/class/leds/valve-leds[0]` through `valve-leds[16]`, each independently controllable via the normal Linux LED-class sysfs interface. It talks to the actual hardware via I/O port registers shared with the embedded controller, with a literal "commit to EC non-volatile storage" command - explaining exactly why a user-customized color survives a reboot, before SteamOS has even loaded. And it exposes a genuinely rich set of custom sysfs attributes: an `effect` selector (`patrol`, `breath`, `factory`, `normal`, `off`, `rainbow`, `demo`, `manual`), plus tunable parameters like `delay`, `patrol_num`, `breath_offset`, `breath_level`, and `color_shift`.

One dead end worth mentioning: the corresponding firmware update file (`F7F0106.cab`) turned out to be a full 33.5MB UEFI capsule image - the entire system BIOS, not an isolated EC blob. A `strings` scan for the register/effect names found nothing, which makes sense in retrospect: those names only exist as documentation in the kernel driver's source, not as literal symbols in compiled EC machine code. We didn't want to decompile a proprietary firmware capsule for various reasons, so that's where the firmware archaeology stopped.

## Act 4: Building a replica

With the real interface identified, the natural next question: could you build a Linux kernel module that presents the *exact same sysfs shape*, but backed by a DIY WS2812B strip instead of real Valve hardware?

Short answer: yes, but not entirely from userspace. `/sys/class/leds/` entries only exist because a kernel driver registers them - there's no way to fake one with a udev rule. Linux does ship a generic mechanism for exactly this kind of "userspace-backed LED" use case (`uleds`, `/dev/uleds`), but it only supports the simple monochrome LED class, not the multicolor one Valve's driver uses. So a small custom kernel module was unavoidable - though the design could stay minimal: the kernel module's *only* job is to present the interface (register the classdevs, expose the sysfs attributes) and hand off every state change to userspace via a companion misc character device. All the actual work - rendering effects, talking to real hardware - happens in an ordinary userspace daemon, which is both safer and dramatically easier to iterate on.

That daemon speaks to a physical strip via an ESP32 running [WLED](https://kno.wled.ge/) - if you've read my [earlier audio-reactive lighting setup post](/posts/wled-audioreactive/), you will understand why: I already have experience with sending stuff to WLED and have hardware lying around, making it the obvious default option rather than something requiring more research. It talks over WLED's **DNRGB** realtime UDP protocol - verified directly against WLED's own source code, rather than the most generic DDP (Distributed Display Protocol). DNRGB's per-pixel offset field turned out to be exactly right for this: no total-strip-length assumption baked into the protocol, no multi-packet fragmentation logic needed at these tiny LED counts, just a start-index and a run of RGB triples.

```c
// The classdev registration loop - adapted directly from leds-valve.c,
// but backed by an in-memory frame buffer instead of real hardware:
for (i = 0; i < num_leds; i++) {
    led->mcdev.led_cdev.name =
        devm_kasprintf(&pdev->dev, GFP_KERNEL, "valve-leds[%d]", i);
    led->mcdev.led_cdev.brightness_set_blocking = sw_set_brightness;
    devm_led_classdev_multicolor_register(&pdev->dev, &led->mcdev);
}
```

One naming decision mattered more than it looked like it would. The classdevs are named `valve-leds[N]` - matching Valve's real driver exactly, rather than something more clearly "ours." There's no actual risk in doing this: the real driver only binds via a DMI hardware check that non-Valve hardware will never satisfy, so the two can never collide on the same machine.

## Act 5: Closing the loop

The naming decision paid off in a way that turned into the best confirmation of the whole project. A strings dump of `steamui.so` - the actual Steam client UI binary, shipped identically across every Linux distro running Steam, not something SteamOS-exclusive - turned up a hardcoded reference:

```c
#LED_FrontBar    /sys/class/leds/valve-leds
```

Alongside a whole native `CLEDController`/`CLEDDeviceSysfs` implementation, exposing a `LEDManager` RPC surface (`GetState`, `SetEnabled`, `SetColor`, `SetSpeed`, `SetEffect`, `SetBrightness`) to the client's own settings UI. That settled an open question from earlier in the project: LED control isn't handled by a system daemon at all - it's baked directly into the Steam client binary. And critically, no DMI/hardware-identity strings turned up anywhere near it - a real (if not airtight) signal that the client just checks the sysfs path, without a separate hardware gate blocking a well-named replica from being picked up.

The way to validate all of this didn't even require real Steam Machine hardware. Instead I tried this on Bazzite, running the identical generic Linux Steam client: launched with undocumented flags straight out of the strings table (`-led-enumerate-sysfs-all` and `-testled`), it did show LED options under Big Picture settings/Customizations. Unfortunately, these weren't the LED options I expected (only test ones, indeed) and my `/sys/class/leds/*` devices seemed ignored.

A few device permissions fixes later as well as adding the option to run the code in debug mode, it was clear that nothing was touching those. So it seemed that there was some gating somewhere. A strings search turned up an interesting-looking `Plat_IsSteamOS()` in `steamui.so`, and luckily it turned out that spoofing steamos in `/etc/os-release` was all that was needed to make the feature appear.

## A road not taken: OpenRGB and the motherboard ARGB header

Worth a mention, because the idea was genuinely interesting even though it didn't pan out. "ARGB" - the term motherboard vendors (Asus Aura, MSI Mystic Light, and others) use for their addressable headers - is just marketing for the same WS2812B wire protocol at 5V. In principle, that meant skipping the ESP32 entirely: wire the strip straight into a motherboard's ARGB header, and drive it locally with [OpenRGB](https://openrgb.org), which has a solid documented SDK and an existing Python client.

The blocker was Linux-specific and very concrete: OpenRGB's own documentation states that ASUS and ASRock motherboards put their RGB controller behind an SMBus interface that isn't reachable from an unmodified Linux kernel at all - requiring yet another out-of-tree kernel patch, on top of the one this project already needed, with real user reports of continued flakiness even after patching on similar-generation boards. Filed away as a "remove the ESP32" stretch goal, not a reliability upgrade.

## Where this stands

What exists right now: a kernel module that registers `valve-leds[0..N-1]` as real multicolor LED classdevs, a companion misc device for a userspace bridge to read frame updates from, and a Python daemon that renders effects and forwards them to a physical strip over WLED's DNRGB protocol.

All of it was compiled and hardware-tested on a Bazzite Linux machine and a D1 mini running WLED. The full source is up at [github.com/endemics/steamos-wled](https://github.com/endemics/steamos-wled), for anyone who wants to follow along or try it themselves. Sure, there are still a few rough edges in terms of installation (it requires compiling, loading a kernel module, starting a python script, mimicking steamos), and the effects are perfectible, but it essentially works as a proof-of-concept.

None of this needed a teardown, a debugger, or even a single line of Valve's proprietary code disassembled. Just a product photo, a public kernel patch archive, and a willingness to cross-check every hypothesis against the most primary source available - even when a shortcut was tempting, and even when that discipline occasionally caught the writer's own mistakes along the way.

---

*Code for the kernel module and WLED bridge daemon adapts `drivers/leds/rgb/leds-valve.c` (Copyright © 2025 Valve Corporation, written by Robert Beckett at Collabora, GPL-2.0+) - see the project README for the full attribution and license details.*
