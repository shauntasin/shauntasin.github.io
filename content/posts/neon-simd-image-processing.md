---
title: "NEON SIMD: The Free Speedup Your Image Pipeline Is Ignoring"
date: 2026-07-15
description: "How ARM NEON intrinsics turn scalar pixel loops into 4x-8x measured wins, with real C code, benchmarks on a Pi 4, and the mental model behind it."
---
Every image processing loop you've ever written is embarrassingly parallel.

Grayscale conversion. Brightness. Thresholding. Alpha blending. Same tiny operation, repeated a few million times per frame, on data that sits right next to each other in memory.

And yet most C code processes pixels one at a time, like a cashier scanning groceries item by item while the conveyor belt holds sixteen.

NEON fixes that. And you don't need assembly to use it.

## The mental model

NEON is ARM's SIMD unit: Single Instruction, Multiple Data. Instead of registers holding one value, you get 128-bit vector registers holding **16 x uint8**, **8 x uint16**, or **4 x float32** at once.

One instruction. Sixteen pixels. That's the whole pitch.

The intrinsics in `arm_neon.h` map almost one-to-one to instructions, so you write C, the compiler handles register allocation, and you keep type checking. Best of both worlds.

```c
#include <arm_neon.h>

uint8x16_t a = vld1q_u8(src);      // load 16 bytes
uint8x16_t b = vaddq_u8(a, a);     // 16 adds, one instruction
vst1q_u8(dst, b);                  // store 16 bytes
```

The naming looks like line noise until you learn the grammar: `v` (vector) + operation + `q` (128-bit, "quad") + type suffix. `vaddq_u8` = vector add, full-width, unsigned 8-bit. After a day it reads like plain English.

## A real example: grayscale

The classic luma conversion: `Y = 0.299R + 0.587G + 0.114B`.

Scalar version, the one everybody writes first:

```c
void gray_scalar(const uint8_t *rgb, uint8_t *gray, int n) {
    for (int i = 0; i < n; i++) {
        gray[i] = (77 * rgb[3*i] + 150 * rgb[3*i+1] + 29 * rgb[3*i+2]) >> 8;
    }
}
```

(Fixed-point weights: 77/150/29 are the coefficients scaled by 256. Floats in a pixel loop is a rookie tax.)

NEON version:

```c
void gray_neon(const uint8_t *rgb, uint8_t *gray, int n) {
    for (int i = 0; i + 16 <= n; i += 16) {
        uint8x16x3_t px = vld3q_u8(rgb + 3*i);   // deinterleave R,G,B

        uint16x8_t lo = vmull_u8(vget_low_u8(px.val[0]), vdup_n_u8(77));
        lo = vmlal_u8(lo, vget_low_u8(px.val[1]), vdup_n_u8(150));
        lo = vmlal_u8(lo, vget_low_u8(px.val[2]), vdup_n_u8(29));

        uint16x8_t hi = vmull_u8(vget_high_u8(px.val[0]), vdup_n_u8(77));
        hi = vmlal_u8(hi, vget_high_u8(px.val[1]), vdup_n_u8(150));
        hi = vmlal_u8(hi, vget_high_u8(px.val[2]), vdup_n_u8(29));

        vst1q_u8(gray + i, vcombine_u8(vshrn_n_u16(lo, 8),
                                       vshrn_n_u16(hi, 8)));
    }
    // scalar tail for n % 16 left as the classic exercise
}
```

Two things to appreciate here.

**`vld3q_u8` is doing structural magic.** Interleaved RGB comes in as `RGBRGBRGB...` and this single instruction splits it into three clean registers: all the Rs, all the Gs, all the Bs. Deinterleaving in scalar code is shuffle hell. NEON gives it to you as a load mode. This one instruction is half the reason NEON is pleasant for image work.

**Widening multiplies handle overflow for free.** `vmull_u8` multiplies 8-bit values into 16-bit results, `vmlal_u8` accumulates on top, and `vshrn_n_u16` shifts and narrows back down to 8-bit. No clamping dance, no undefined behavior, no "why is my image psychedelic" debugging session.

Sixteen pixels per iteration. Let's see what that actually buys.

## Numbers or it didn't happen

I benchmarked this on the Raspberry Pi 4 under my desk (Cortex-A72 @ 1.8 GHz, gcc 12, `-O2`), because that's the closest thing I own to the boards this code actually ships on. One 1920x1080 frame per call, median of 200 runs, buffers touched before timing so the page faults don't lie to you:

```c
struct timespec t0, t1;
clock_gettime(CLOCK_MONOTONIC, &t0);
gray_neon(rgb, gray, W * H);
clock_gettime(CLOCK_MONOTONIC, &t1);
double ms = (t1.tv_sec - t0.tv_sec) * 1e3
          + (t1.tv_nsec - t0.tv_nsec) / 1e6;
```

I also threw in two simpler ops, because grayscale isn't the whole story:

```c
// brightness: saturating add, clamps at 255 in hardware
void brighten_neon(const uint8_t *src, uint8_t *dst, int n, uint8_t k) {
    uint8x16_t vk = vdupq_n_u8(k);
    for (int i = 0; i + 16 <= n; i += 16)
        vst1q_u8(dst + i, vqaddq_u8(vld1q_u8(src + i), vk));
}

// threshold: compare produces 0x00/0xFF masks, which IS the answer
void threshold_neon(const uint8_t *src, uint8_t *dst, int n, uint8_t t) {
    uint8x16_t vt = vdupq_n_u8(t);
    for (int i = 0; i + 16 <= n; i += 16)
        vst1q_u8(dst + i, vcgtq_u8(vld1q_u8(src + i), vt));
}
```

(That threshold one is my favorite party trick: `vcgtq_u8` returns all-ones or all-zeros per lane, so the comparison result *is* the output image. Zero extra instructions. The scalar version pays a compare and a conditional per pixel.)

Results, one 1080p frame (2,073,600 pixels):

| Operation | Scalar | NEON | Speedup |
|---|---|---|---|
| Grayscale (RGB → luma) | 6.9 ms | 0.9 ms | **~7.5x** |
| Brightness (saturating +40) | 3.1 ms | 0.8 ms | **~3.9x** |
| Threshold (> 128) | 2.9 ms | 0.7 ms | **~4.1x** |

Two honest observations about that table.

**Grayscale wins biggest because it has real math in it.** Three multiplies and two accumulates per pixel is enough work that collapsing 16 pixels into one instruction stream dominates everything else.

**Brighten and threshold "only" get 4x because they hit the memory wall.** Those loops are so simple that the Pi's DRAM bandwidth becomes the ceiling — the CPU is mostly waiting for bytes either way. 4x for a three-line change is still a trade I'll take every single day, and on chips with better memory subsystems the gap widens again.

Scale it up: scalar grayscale at 6.9 ms is fine for one frame, but at 30 fps that's 21% of your entire frame budget gone on *one* preprocessing step. The NEON version takes 2.7%. That's the difference between "my vision pipeline fits" and "my vision pipeline drops frames".

## Where the speed actually comes from

People assume SIMD wins are just "16 lanes = 16x." It's more layered:

- **Instruction count collapses.** One `vaddq_u8` replaces 16 adds, but also 16 loads and 16 stores become one load and one store.
- **Memory bandwidth gets used properly.** Pixel data is contiguous. Vector loads pull whole cache lines and actually use every byte they fetched.
- **Saturating arithmetic is built in.** `vqaddq_u8` clamps at 255 in hardware. The scalar equivalent is a compare-and-branch per pixel, and branches in a pixel loop are poison for the pipeline.
- **The narrow/widen instructions eliminate glue code.** Half the cost of scalar image code is type juggling. NEON has instructions for exactly that juggling.

Realistic wins for byte-oriented image ops: 4x to 12x. Not marketing numbers. Measured ones.

## "Doesn't the compiler auto-vectorize?"

Sometimes. When the moon is right.

Auto-vectorization bails on anything interesting: interleaved channel access, saturating math, mixed-width arithmetic, loops where it can't prove pointers don't alias. Exactly the stuff image processing is made of. `restrict` helps, `-O3` helps, and then one innocent refactor later the vectorizer silently gives up and your frame time doubles.

Intrinsics are a contract. The compiler can schedule and allocate registers around them, but the vector operations you wrote are the vector operations you get. For hot loops, I want the contract.

Check what you're actually getting either way: `-S` output or [Compiler Explorer](https://godbolt.org/). If you see `ld3` and `umull` in the disassembly, you're in business.

## The gotchas, because there are always gotchas

- **Handle the tail.** Image widths aren't always multiples of 16. Do a scalar cleanup loop, or overlap the last vector if the buffer allows it. Forgetting this is a rite of passage.
- **Don't dance between NEON and scalar per-element.** Moving single lanes in and out of vector registers has latency. Stay in vector land for the whole pipeline stage.
- **Alignment matters less than it used to** (unaligned loads are fine on AArch64) **but cache behavior still rules everything.** Process row by row, keep your working set hot.
- **Profile on the real target.** Your laptop's Apple Silicon and a Cortex-A53 in an embedded board have very different pipelines. `perf stat` on the device, always.

## Why I care

Every phone, every Raspberry Pi, every embedded vision board, every AWS Graviton box has NEON sitting there. Mandatory in AArch64 — it's not an optional extension you have to detect, it's just *there*.

Meanwhile, the default answer to "my image pipeline is slow" has become "add a GPU" or "add another core." That's reaching for a forklift because you never learned the hand truck. For per-pixel ops on moderate resolutions, the DMA/dispatch overhead of a GPU round-trip often eats the win anyway. A tight NEON loop finishes before the GPU driver returns from the submit call.

I don't have a GPU budget. I have a Pi 4, gcc, and `arm_neon.h`. Turns out that's plenty.

The hardware ships sixteen lanes wide. Writing scalar pixel loops is choosing to use one of them.

Use the whole register. It's free.
