---
title: "Bringing a Degraded mdadm RAID1 Back From the Brink"
date: 2026-06-08T10:58:28Z
draft: false
categories: ["Technology"]
tags: [ "Linux", "RAID" ]
---

A war story about a failed RAID1 mirror, a stubborn superblock, a colon in an array name, a cable knocked loose mid-rebuild, and five bad sectors hiding in — of all things — a rescue image. If you've ever stared at `add new device failed ... Invalid argument` and wondered what you broke, this one's for you.

Claude helped a lot, both during the diagnostic, and to write down this post. At this stage, hallucinations are shared and unvoluntary.

## The starting point

A two-disk mdadm RAID1 (`/dev/md0`, metadata 1.2) had dropped to a single active member. `mdadm --detail` showed one drive removed and the former second drive listed as a spare. The standard recovery dance had already been attempted: remove the drive, zero the superblock, recreate the partition, and re-add it.

That last step failed:

```
mdadm /dev/md0 --add /dev/sdc2
mdadm: add new device failed for /dev/sdc2 as 3: Invalid argument
```

And `dmesg` insisted the partition had no valid v1.2 superblock — even on a freshly wiped partition. That contradiction (`Invalid argument` from mdadm, `no valid superblock` from the kernel) turned out to be the thread that unravelled the whole mystery.

## Lesson 1: a fresh partition has no superblock for `--add` to trust

`mdadm --add` to a degraded array is supposed to write a superblock onto a clean member and start a resync. When it kept refusing, the early suspects were the usual ones:

- **Size mismatch** — the replacement must be the same size or larger than the surviving member, down to the sector.
- **Stale signatures** — leftover filesystem or partition signatures that confuse mdadm's detection.

The fix attempts:

```bash
wipefs -a /dev/sdc2          # clear signatures on the PARTITION only
mdadm --zero-superblock /dev/sdc2
mdadm --manage /dev/md0 --add /dev/sdc2
```

**Important caveat:** never run `wipefs -a` on the whole disk (`/dev/sdc`) if it holds other partitions you care about — that nukes partition-table signatures. Always target the specific partition. To preview what's there without removing anything, run `wipefs` with no `-a`.

That preview revealed the partition still carried a `linux_raid_member` signature from a *previous* array — a different UUID and a different array name (`danu:0`). So it wasn't blank at all.

## Lesson 2: matching UUID isn't enough — the event count matters

After confirming the partition's superblock UUID actually matched the array, the next clue surfaced:

```
Events : 0      (array and active drive were at 226898)
Device Role : spare
```

An events counter of `0` against an array at `226898` means mdadm sees the device as catastrophically out of sync — effectively a stranger wearing the right UUID. Recreating the partition had reset its superblock from scratch.

The intended fix — wipe and let `--add` write a fresh superblock — kept failing, including `mdadm --zero-superblock` itself bailing with *"unrecognised md component device"* because mdadm still associated the partition with a known array.

The blunt instrument that finally cleared it:

```bash
dd if=/dev/zero of=/dev/sdc2 bs=512 count=2048
```

Zeroing the first megabyte destroys any superblock regardless of what mdadm or `wipefs` believe about it.

## Lesson 3: data offset and metadata must actually line up

With the surviving member identified as `/dev/sdb1`, comparing the two superblocks revealed the partitions started at different sectors, producing different data offsets. For some reasons, while this worked before, it seems that for a v1.2 mirror, the data offset recorded in the superblock has to be consistent, so the replacement partition was rebuilt to start at exactly the same sector as the survivor (the swap volume that was the first partition gave its existence for the cause and will be recreated as /dev/sdc2).

On GPT disks with `sgdisk`/`sfdisk`, the recipe is: wipe the target disk's partition table, then create a single RAID partition with the **same start sector and size** as the survivor. The surviving member's start and size came straight from `sfdisk --dump`:

```
start 2048   size 5858578432   (matching the survivor)
```

A useful gotcha here: the partition **type GUID** on the survivor was the generic Linux filesystem type (`0FC63DAF-...`), not the Linux RAID type. Match what's actually there rather than assuming `fd00`.

## Lesson 4: the colon-in-name POSIX trap

Several attempts to force-write a superblock with `mdadm --create` died on:

```
Value "danu:0" cannot be set as name. Reason: Not POSIX compatible.
```

(we also had similar failures with /dev/sdcX or other combinations, we're not narrow-minded)

mdadm 4.3 rejects an array `--name` containing a colon. The original name `danu:0` is really a homehost (`danu`) plus an array index (`0`). The correct way to express it:

```bash
--homehost=danu --name=0
```

This single quirk had been sabotaging every `--create`-based workaround for hours. Worth knowing if you're on a modern mdadm trying to recreate an old array.

## Lesson 5: inconsistent feature flags get rejected by the kernel

Even after a standalone superblock was written to the replacement with the right UUID and data offset, the kernel still refused it with `md_import_device returned -22` (EINVAL). Comparing superblocks turned up the real culprit:

```
sdc1  Feature Map : 0x8   Bad Block Log at offset 264 sectors
sdb1  Feature Map : 0x8   Bad Block Log at offset 24 sectors
```

The replacement, written by a 2014-vintage-array-meets-mdadm-4.3 `--create`, had its bad-block-log at a different offset and was even advertising "bad blocks present" on a brand-new partition. The kernel rejected the inconsistency.

## The fix that worked: recreate the array, preserve the data

When every `--add`/`--re-add`/`--assemble` path is exhausted and the only real inconsistency is metadata, recreating the **array** rebuilds both superblocks with one consistent mdadm version. For RAID1 there's no parity to recompute, so with the right flags the existing data stays untouched:

```bash
mdadm --stop /dev/md0

mdadm --create /dev/md0 --metadata=1.2 --level=1 --raid-devices=2 \
  --data-offset=262144s --assume-clean --homehost=danu --name=0 \
  /dev/sdb1 missing
```

The critical flags:

- **`--data-offset=262144s`** — must match the survivor's offset exactly, or the data region is misread as garbage. The `s` suffix means sectors.
- **`--assume-clean`** — don't resync; trust the existing data in place.
- **`/dev/sdb1 missing`** — survivor first, second slot intentionally empty. Putting the good drive first preserves its data.

mdadm warns that `sdb1` looks like it's part of an existing array and asks to continue — that's expected; answer `yes`.

**Then verify before touching the second drive.** Mount read-only and confirm the files are really there:

```bash
mount -o ro /dev/md0 /mnt/check
ls /mnt/check
umount /mnt/check
```

The directory listing came back intact — every folder present, filesystem recognized, data offset confirmed correct. Only *after* that confirmation:

```bash
mdadm --manage /dev/md0 --add /dev/sdc1
```

This time it took. With both superblocks written by the same mdadm against a freshly consistent array, the kernel accepted the new member and started rebuilding. Note that a `--create` generates a **new array UUID**, so the config has to be updated afterward (see the end).

## Plot twist: a cable knock mid-rebuild

About 85% into the rebuild (e.g. around 5 hours), the second drive got marked failed. Cause: a SATA cable was nudged (while trying to reset a usb cable for an external drive... long story). After reseating it and rebooting, the array reappeared as `md127` (the kernel's fallback name when no `mdadm.conf` entry matches) with the survivor active and degraded — exactly the safe state to recover from.

The survivor's data was never at risk: during a RAID1 rebuild the source is only *read*, while the target is *written*, so a mid-rebuild disconnect only affects the target. Renaming back and resuming:

```bash
mdadm --stop /dev/md127
mdadm --assemble /dev/md0 /dev/sdb1
mdadm --manage /dev/md0 --re-add /dev/sdc1
```

`--re-add` was accepted because the partial member's superblock still matched the array (UUID intact, only a handful of events behind). The rebuild restarted — and this time it finished, reaching `[2/2] [UU]`.

## The real problem underneath: a drive with bad sectors

The rebuild logged the genuinely worrying line:

```
md/raid1:md0: sdb1: unrecoverable I/O read error for block 5719987584
Add. Sense: Unrecovered read error - auto reallocate failed
```

Time to read SMART rather than panic. The data told a more nuanced story than the scary kernel logs:

| Attribute | Value | Read |
|---|---|---|
| Reallocated_Sector_Ct | 13 | Low — not a dying drive |
| Current_Pending_Sector | 5 | Five sectors currently unreadable |
| Offline_Uncorrectable | 5 | Same five |
| UDMA_CRC_Error_Count | 29670 | **Huge — but these are *link* errors** |

The key insight: **CRC errors are interface/cable errors, not platter errors.** Nearly 30,000 of them, combined with repeated `port frozen` / `hard resetting link` events, pointed at a marginal SATA connection — almost certainly inflated by all the cable-handling — rather than a disk about to die. The actual platter defect was tiny: five bad sectors.

## Lesson 6: know your controller before blaming the cable

The link kept negotiating at 3.0 Gbps. Was that a fault? `lspci` identified the controller as an **NVIDIA MCP55** — a SATA II (3 Gbps) chipset from around 2006. So 3 Gbps is the port's genuine ceiling, even though the drive reported `SATA 3.1, 6.0 Gb/s (current: 3.0 Gb/s)`.

The 3 Gbps was a red herring. The MCP55's **SWNCQ** (software NCQ) implementation is also famously cranky and shows up all over the error logs (`ata5: EH in SWNCQ mode`). Mitigations worth knowing:

```bash
# Make the drive give up on a bad sector quickly instead of freezing the port
smartctl -l scterc,70,70 /dev/sdb

# Disable NCQ on a flaky nForce port
echo 1 > /sys/block/sdb/device/queue_depth
# permanent: libata.force=noncq (or scoped to one port) on the kernel cmdline
```

## Lesson 7: healing pending sectors — and what a `repair` can't do

Note: We also ran `smartctl -l scterc,70,70 /dev/sdc` for good measure during this step.

The instinct was to run an md `repair` scrub to remap the bad sectors:

```bash
echo repair > /sys/block/md0/md/sync_action
```

It completed cleanly with no new CRC errors (good — the link held) but **the five pending sectors didn't clear.** Why? A `repair` only rewrites a sector when it can read a good copy from the *other* drive. These particular blocks had been unreadable from *both* members during the rebuild — there was no good copy to write back, so the sectors stayed pending.

Pending sectors only get remapped when they're **written**. So the job was to force a write to exactly those locations. First, find out what lives there. The kernel logs the bad block in 512-byte md sectors; `debugfs` works in 4096-byte filesystem blocks:

```
fs_block = md_sector / (4096 / 512) = 5719987584 / 8 = 714998448
```

```bash
debugfs -R "icheck 714998448" /dev/md0     # block -> inode
debugfs -R "ncheck 105808316" /dev/md0     # inode -> path
```

Both bad blocks mapped to a single inode, and `ncheck` named it: `/backup/blackbox/rescue.img` — a ddrescue capture of a *previously* failed NVMe drive. (The path from `debugfs` is relative to the filesystem root; on the running system it lived under the `/data` mountpoint.) Fittingly, the corruption was probably in a region ddrescue had never recovered from the original drive in the first place. (Plot twist: the NVMe drive was actually fine, one of the DIMM of the source machine wasn't, though).

Because the file was expendable and backed up externally, the cleanest cure was to delete it and overwrite the freed space, forcing the drive to write (and thus remap) the pending sectors:

```bash
rm /data/backup/blackbox/rescue.img
sync

# fill free space so every freed sector gets written
dd if=/dev/zero of=/data/zerofill bs=1M status=progress; sync
rm /data/zerofill
sync

smartctl -a /dev/sdb | grep -iE "pending|reallocated"
```

The result:

```
Reallocated_Sector_Ct   ... 15      (was 13)
Current_Pending_Sector  ... 0       (was 5)
```

Pending dropped to zero. Two of the five sectors were physically bad and got remapped to spares (reallocated 13 → 15); the other three were merely transiently unreadable and the write succeeded outright. Defect healed.

## Finishing up

A few loose ends after any `--create`-based recovery:

**Persist the new array UUID** so it boots as `md0` instead of `md127`:

```bash
mdadm --detail --scan | grep md0      # clean ARRAY line (no spares= once [UU])
# replace the old ARRAY line in /etc/mdadm/mdadm.conf with that output
update-initramfs -u
```

**Watch the CRC baseline.** `UDMA_CRC_Error_Count` is cumulative and can't be reset, so note the current value and recheck in a few days. If it stops climbing, the link is healthy and the old count was just cable handling. If it keeps rising, replace the cable.

## Takeaways

- **`Invalid argument` + `no valid superblock` together** usually means a metadata inconsistency (events count, data offset, or feature flags), not a literally-missing superblock.
- **`wipefs` and `--zero-superblock` can both refuse** a device mdadm thinks belongs to a known array; `dd` over the first MB is the reliable last resort.
- **Recreating a RAID1 array with `--assume-clean` and a matching `--data-offset` is non-destructive** — but verify read-only before adding the second member, and remember the UUID changes.
- **Modern mdadm rejects colons in `--name`;** use `--homehost` + `--name`.
- **CRC errors are cable/link problems, not platter problems.** Don't condemn a drive for a high CRC count.
- **An md `repair` can't heal a sector that's unreadable on both mirrors** — force a write (delete + zerofill, or rewrite the affected file) to remap pending sectors.
- **RAID1 is not a backup.** (Repeat after me Yann: _"Raid is not a backup"_ :stuck_out_tongue_winking_eye:) It survived a single drive degrading; it would not have survived the controller corrupting both. Keep a real off-array copy — especially on twenty-year-old hardware.

_"Trust me, I'm a professional"_... Erm..
