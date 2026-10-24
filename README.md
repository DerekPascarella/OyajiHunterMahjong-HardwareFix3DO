# Oyaji Hunter Mahjong

<img align="right" width="200" src="https://github.com/DerekPascarella/OyajiHunterMahjong-HardwareFix3DO/blob/main/cover.png?raw=true"> A fix for "Oyaji Hunter Mahjong" on the 3DO preventing crashes when played via CD-R or ODE. Without it, the game hangs or resets on the loading screen before Stage 5, and sometimes before other stages. It also fixes the loading screen itself, so all three graphics render as intended in software emulators.

The crash was originally reported in [issue #16 of fixel's 3DO ODE firmware repo](https://github.com/fixelsan/3do-ode-firmware/issues/16), and was reproduced there on an FZ-1 with an internal ODE, on MNEMO's ODE, and on a plain CD-R. It turned out to be a bug in the game itself, not the ODEs. The original pressed disc reportedly gets away with it, but CD-Rs and ODEs don't.

Two patches are available. One can be applied to the [Eight Mansions English translation v1.1.1](https://github.com/Eight-Mansions/Oyaji-Hunter-Mahjong/releases/tag/v1.1.1), and the other goes to the original Japanese version for those wishing to play it in its original language.

**⯈ Download Patch (English v1.1.1):** [Oyaji Hunter Mahjong (Hardware Crash Fix for T-En v1.1.1).xdelta](https://github.com/DerekPascarella/OyajiHunterMahjong-HardwareFix3DO/raw/refs/heads/main/Oyaji%20Hunter%20Mahjong%20(Hardware%20Crash%20Fix%20for%20T-En%20v1.1.1).xdelta)

**⯈ Download Patch (Japanese):** [Oyaji Hunter Mahjong (Hardware Crash Fix).xdelta](https://github.com/DerekPascarella/OyajiHunterMahjong-HardwareFix3DO/raw/refs/heads/main/Oyaji%20Hunter%20Mahjong%20(Hardware%20Crash%20Fix).xdelta)

## Patching Instructions

<img align="right" width="200" src="https://github.com/DerekPascarella/OyajiHunterMahjong-HardwareFix3DO/blob/main/screenshot_fmv.png?raw=true"> The XDelta patch files shipped with this release can be used with any number of Delta utilities, such as such as [Delta Patcher](https://www.romhacking.net/utilities/704/) or the xdelta3 command line tool.

**English**

1. The English v1.1.1 ISO is built first using the [Eight Mansions release](https://github.com/Eight-Mansions/Oyaji-Hunter-Mahjong/releases/tag/v1.1.1) and its instructions. This produces `Oyaji Hunter Mahjong - English v1.1.1.iso`.
2. `Oyaji Hunter Mahjong (Hardware Crash Fix for T-En v1.1.1).xdelta` is then applied to that ISO.

**Japanese**

1. `Oyaji Hunter Mahjong (Hardware Crash Fix).xdelta` is applied to the original Japanese ISO.

From the command line, either one looks like this.

```
xdelta3 -d -s "source.iso" "patch.xdelta" "patched.iso"
```

## Checksums

| Disc Image | MD5 |
|---|---|
| Japanese original | `ACBC487C646C340A2D691D9B5AE95449` |
| Japanese with hardware fix | `2AF27343C6E66B5F1139E5A48522D289` |
| English v1.1.1 | `5BA7610BBBED67F2F2A8BB9DE539C49B` |
| English v1.1.1 with hardware fix | `2267D95E095D521C3931A9C378A5BFC3` |

## About the Bug

<img align="right" width="200" src="https://github.com/DerekPascarella/OyajiHunterMahjong-HardwareFix3DO/blob/main/screenshot_loading.png?raw=true"> Between stages the game shows a NOW LOADING screen with a few images of the next opponent. That screen is its own little program on the disc called `/wait`, and it runs at the same time as the main game loads the next stage. For each stage it loads up to three images, one after another, with the 3DO library's `LoadCel()`, which asks the OS for a fresh block of memory to hold each one.

Going into Stage 5 the main game is loading a lot at once and memory is tight, so the second and third images fail to load and `LoadCel()` hands back a NULL pointer. `/wait` never checks for that. It sets the image's X and Y position anyway, which writes zeros to addresses `0x10` and `0x14`.

On a real 3DO, the memory controller only lets a task write to its own pages. Address `0x10` isn't one of them, so the write causes a data abort, the OS kills the task, and the game hangs or resets. Emulators generally don't enforce this, so the write goes through without complaint. The original report describes what happens on the Phoenix emulator, where the image doesn't load and NOW LOADING gets drawn a second time over the previous image.

## About the Fix

<img align="right" width="200" src="https://github.com/DerekPascarella/OyajiHunterMahjong-HardwareFix3DO/blob/main/screenshot_mahjong.png?raw=true"> `/wait` already has a full screen buffer in VRAM. It uses it to keep a copy of the background so it can redraw it every frame while the NOW LOADING text animates. That buffer is 153,600 bytes, and the biggest loading image on the disc is 148,548 bytes, so every image fits in it. The patch reads each image straight into that buffer instead of asking for new memory, effectively never running out.

Reusing the buffer is safe because of the order in which things already happen. The image is drawn from the buffer onto the screen first, and only after that does the game copy the finished screen back into the buffer, which is the game's original behavior.

There's was no extra free space in `/wait`, so an unused library routine at `wait+2B80` was repurposed for a small new helper routine.

* `LoadFileHere` at `wait+2BF8` works like `LoadFile`, except the caller supplies the buffer. It refuses any file that's bigger than the buffer. It was also linked in and never used.
* `ParseCel` at `wait+16F0` is the second half of `LoadCel()`. It takes a file that's already in memory and sets up the cel's pointers in place, without allocating anything.

Here's the call site in the stage image loop, before and after.

```
             original                  patched
wait+DA8     mov  r1, #0x40000         ldr  r1, =globals     ; VramTemp lives in here
wait+DAC     bl   LoadCel              bl   LoadWaitCel
wait+DB0     mov  r5, r0               movs r5, r0
wait+DB4     mov  r0, #0               beq  skip_image       ; NULL check
...
wait+E38     bl   UnloadCel            nop                   ; buffer isn't heap memory
```

And the new routine.

```
LoadWaitCel:                           ; r0 = file name, r1 = globals
    str   lr, [sp, #-8]!
    ldr   r2, [r1, #0x20]              ; VramTemp, the screen buffer
    mov   r3, #0x25800                 ; its size, 153,600 bytes
    add   r1, sp, #4                   ; LoadFileHere writes the file size here
    bl    LoadFileHere
    cmp   r0, #0
    ldrne r1, [sp, #4]
    blne  ParseCel                     ; returns the cel, or 0
    ldr   pc, [sp], #8
```

The NULL check stays in just in case an image ever fails to load for some reason, like a bad read. In such cases, it's skipped instead of crashing the game.
