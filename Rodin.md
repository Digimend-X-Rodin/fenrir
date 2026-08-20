# Fenrir – Bypass Lock Control Patch (Pattern Update Guide)

This document explains how to **manually update the `bypass_lock_control` patch pattern** when fenrir reports:

```
Warning: Pattern not found for patch 'bypass_lock_control'
```

This happens when your `lk.img` firmware differs from the one fenrir was originally designed for.  
The steps below show how to locate the new pattern using `radare2`, update the device definition, and successfully apply the patch.

---

## Tools Required

- [radare2](https://rada.re/n/) – for disassembly and binary analysis
- fenrir (the injector tool) – available at [R0rt1z2/fenrir](https://github.com/R0rt1z2/fenrir)
- A hex editor or `dd` for manual patching (optional)

---

## Understanding the Patch

The `bypass_lock_control` patch targets a **conditional branch** that checks the lock state.  
If the device is locked, it jumps to an error routine and blocks `fastboot flash`.  
The patch changes the conditional branch (`b.ne`) into an unconditional branch (`b`), so the error is always skipped.

The original pattern (for the old `lk.img`) was:

```
00 74 3d 91 c3 00 00 14 e8 0f 40 b9 1f 05 00 71 21 01 00 54
```

The replacement changes the last 4 bytes from `21 01 00 54` to `09 00 00 14`.

When the pattern is not found, you need to find the **equivalent bytes** in your new firmware.

---

## Step‑by‑Step Procedure

### 1. Determine the correct base load address

The `lk.img` is a raw ARM binary. For MediaTek devices, the common base address is `0x8E000000` or `0x81E00000`.  
You can verify the correct base by loading the binary and checking if a known string (like `"Flashing is not allowed in Lock state"`) shows readable ASCII.

```bash
r2 -a arm -b 32 -m 0x8E000000 lk.img
```

If you see garbage, try other common bases (`0x81E00000`, `0x80000000`, `0x88000000`).  
In our case, `0x8E000000` was correct.

### 2. Locate the load‑compare‑branch pattern

Search for the **core** part of the old pattern that is unlikely to change:

```
e8 0f 40 b9 1f 05 00 71
```

This is the load (`ldr`) and compare (`cmp`) that precede the conditional branch.

```bash
[0x8e000000]> /x e80f40b91f050071
```

This will list several hits. The first hit often corresponds to the lock‑check function (especially if it appears at a low offset, e.g., `0x8e00ca38`).

### 3. Examine the hit and locate the branch

Seek to the found address and disassemble a few instructions:

```bash
[0x8e000000]> s 0x8e00ca38
[0x8e00ca38]> pd 10
```

You should see the load, compare, and then a conditional branch.  
In our case, the bytes at `0x8e00ca40` were `21 01 00 54` (the branch).  
Confirm that this is indeed the lock check – the error string `"Flashing is not allowed in Lock state"` will be referenced nearby.

### 4. Get the full 20‑byte pattern

The pattern fenrir expects is the **20 bytes starting from the instruction before the load**.  
In our case, the load starts at `0x8e00ca38`, so we need the 20 bytes from `0x8e00ca30`:

```bash
[0x8e000000]> s 0x8e00ca30
[0x8e00ca30]> px 20
```

This gives the new pattern. Write it down.

### 5. Update the device definition

Open `devices.py` (or the appropriate device file) and locate the `rodin` entry.  
Replace the `pattern` of `bypass_lock_control` with the **new** 20 bytes you obtained.  
The `replacement` should be the same bytes, but with the last 4 bytes changed from `21 01 00 54` to `09 00 00 14`.

Example update:

```python
'bypass_lock_control': PatchStage(
    'bypass_lock_control',
    pattern='00 d0 3d 91 c3 00 00 14 e8 0f 40 b9 1f 05 00 71 21 01 00 54',
    replacement='00 d0 3d 91 c3 00 00 14 e8 0f 40 b9 1f 05 00 71 09 00 00 14',
    match_mode=MatchMode.ALL,
    description='Allow fastboot flashing regardless of lock state',
),
```

### 6. Re‑run fenrir

Now the pattern will be found and applied automatically:

```bash
./inject.sh rodin lk.img
```

You should see:

```
Successfully applied patch 'bypass_lock_control'
```

### 7. Flash the patched image

The output file (e.g., `rodin-fenrir.bin`) can be flashed to the device:

```bash
fastboot flash lk rodin-fenrir.bin
```

---

## Alternative: Manual Patching (if you don't want to update the device file)

If you prefer to patch the binary once and skip the patch in fenrir:

1. Get the **file offset** of the branch instruction:
   ```bash
   [0x8e000000]> ?v 0x8e00ca40
   ```
   This gives the physical offset (e.g., `0xca40`).

2. Patch the file directly with `dd`:
   ```bash
   printf "\x09\x00\x00\x14" | dd of=lk.img bs=1 seek=$((0xca40)) conv=notrunc
   ```

3. Run fenrir skipping the patch:
   ```bash
   ./inject.sh rodin lk.img --skip-patch bypass_lock_control
   ```

The resulting image will still have all other patches applied.

---

## Why This Works

- The lock state is read by a function and compared with a value.  
- The conditional branch (`b.ne`) jumps to an error when the lock state is not “unlocked”.  
- Changing it to an unconditional branch (`b`) always skips the error, so flashing is always allowed.

This is a safe and reliable way to bypass the lock check without needing to understand the entire bootloader.

---

## Troubleshooting

- If the search for `e80f40b91f050071` gives no hits, the load/compare may have changed.  
  In that case, search for the **conditional branch** opcode (`21 01 00 54`) and look for a preceding load/compare.  
  Or, search for the error string and find cross‑references to locate the function.

- If you are unsure about the correct base address, use the **verbose output** of fenrir (some versions print offsets) to compute the base from a successfully applied patch.

- Always keep a backup of the original `lk.img` before modifying.

---

## Credits

- fenrir by [R0rt1z2](https://github.com/R0rt1z2)
- radare2 for analysis

This guide was written based on real analysis of the `rodin` (Redmi Turbo 4 / POCO X7 Pro) bootloader.
