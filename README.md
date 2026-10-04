# musicbox-pico

A [musicbox](https://github.com/sysl-lang/musicbox) score played on a Raspberry Pi Pico 2 W (RP2350),
written in sysl. The board parses the score, renders it at 22,050 Hz, and plays it out of **GPIO 18 as
PWM audio**, on a loop. It plays the first eight bars of Bach's Minuet in G (BWV Anh. 114).

How the audio gets out:

- **PWM.** Slice 1, channel A, drives GPIO 18. TOP is 1023, so each sample is a 10-bit duty cycle on a
  carrier of about 146 kHz, well above anything a speaker or an ear follows.
- **DMA.** Channel 0 is paced by DMA timer 0 at about 22,050 Hz. It copies samples from a 4 KiB
  `@align(4096)` ring into the slice's compare register, and wraps the ring on its own, with no IRQ.
- **The main loop** refills the ring behind the DMA read pointer, 64 samples at a time from
  `Synth.render`. Once a second it prints over USB serial how many samples it has rendered, how many
  underruns it has seen, and how many times the score has looped.

## Wiring

```
GPIO 18 ──[ 1 kΩ ]──┬──── to amp / headphone input
                    │
                  [10 nF]
                    │
GND ────────────────┴──── to amp / headphone ground
```

The RC low-pass (corner about 16 kHz) strips the carrier. Use an amplifier, or high-impedance
headphones, because a GPIO cannot drive a speaker. Connect the board's ground to the amplifier's
ground.

## Building and flashing

You need the [Pico SDK](https://github.com/raspberrypi/pico-sdk) (2.x), the Arm GNU toolchain, CMake,
Ninja, and sysl 0.0.162 or later.

```
export PICO_SDK_PATH=/path/to/pico-sdk
cmake -B build -G Ninja
cmake --build build
```

Hold BOOTSEL while plugging the board in, then copy `build/musicbox_pico.uf2` to the drive that
appears, or run `picotool load -x build/musicbox_pico.uf2`. To see the statistics, open the USB serial
port (`/dev/cu.usbmodem*` on macOS) at any baud rate.

`CMakeLists.txt` runs `sysl build-c player` to build the program into an archive, then lets the SDK
link that archive. There is no C in this repository: sysl exports `main`.

## Dependencies

From `player/package.hocon`:

```
dependencies {
  musicbox { git = "github.com/sysl-lang/musicbox", version = "0.1.3" }
  pico2    { git = "github.com/sysl-lang/pico2",    version = "0.0.9" }
  rp2350   { git = "github.com/sysl-lang/rp2350",   version = "0.1.0" }
}
```

- `musicbox` provides the synthesizer and the notation parser.
- `pico2` brings the board up and provides USB serial and the clock.
- `rp2350` provides the register map. The program drives PWM and DMA through these registers, because
  the SDK's own `pwm_*` and `dma_*` helpers are `static inline`.

## The stack: 16 KB, carved from the top of RAM

`boot` builds the synthesizer and holds the result, a `Result[Synth, _]`, in its own frame before
boxing it. Synth state is about 5.6 KB. The SDK gives core 0 a 4 KB stack in SCRATCH_Y, which is too
small for that.

Raising `PICO_STACK_SIZE` alone does not work. The stack stays in SCRATCH_Y, and once the reservation
is bigger than the region, the link fails. So `CMakeLists.txt` moves the region itself, using the
SDK's `pico_set_linker_script_var`:

| region      | default                    | here                                  |
|-------------|----------------------------|---------------------------------------|
| RAM         | `0x20000000`, 512 KB       | `0x20000000`, **496 KB**              |
| SCRATCH_Y   | `0x20081000`, 4 KB         | **`0x2007C000`, 16 KB** (core 0 stack) |
| SCRATCH_X   | `0x20080000`, 4 KB         | unchanged, unused                     |

`PICO_STACK_SIZE=0x4000` sizes the reservation to the whole region, so the linker checks that the
stack fits. Core 1 is never started, so SCRATCH_X and the old SCRATCH_Y (8 KB at `0x20080000`) go
unused. That is the price of keeping the stack and the heap in one contiguous block of main SRAM.
The heap ends where the stack starts, at `__HeapLimit = 0x2007C000`.

**Proof from the linked ELF:**

- `__StackTop = 0x20080000` and `__StackBottom = 0x2007C000`, so 16,384 bytes.
- `.stack_dummy` is placed at `0x2007C000`, size `0x4000`.
- Word 0 of the vector table, the initial SP, is `0x20080000`.

**Measured frames** (thumb, sysl 0.0.162, from the `arm-none-eabi-objdump -d` prologues):

| function                 | frame   |
|--------------------------|---------|
| `boot`                   | 5,904 B |
| `synth`                  | 2,040 B |
| `Synth.seek`             | 40 B    |
| `Synth.begin`            | 352 B   |
| `Synth.render`           | 104 B   |
| `Synth.carry`            | 72 B    |
| `__udivmoddi4` (+ thunk) | ~56 B   |

The deepest chain is `main → boot → synth → Synth.seek → Synth.begin → __aeabi_uldivmod`, about
**8.4 KB**. The steady-state chain, `boot → Synth.render → Synth.begin`, is about 6.4 KB. That leaves
**about 7.8 KB of headroom** in the 16 KB stack. The USB interrupt handler runs on the same stack, so
it takes some of that headroom.

## What was verified

- The program builds with the repository's CMake against Pico SDK 2.x for `thumb-freestanding-softfp`
  (Arm, Cortex-M33), and the stack placement above was read from the ELF.
- In the disassembly, the main loop calls `Synth.render` from two places: the first fill, and the
  refill inside the loop. `render` uses `smull` 12 times on its inner path, so mixing is
  integer-multiply arithmetic rather than soft-float calls.
- **Playback on hardware is unverified for this release.** No board was attached when it was built.

## License

ISC. See `LICENSE`.
