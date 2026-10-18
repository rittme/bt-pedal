# Three-pedal ZMK keyboard

ZMK configuration for a modified PCsensor FS2020U1 three-pedal unit connected to
an nRF52840 SuperMini. It is built against ZMK **v0.3.0** as a `nice_nano_v2`
compatible controller target.

## Wiring and default keymap

| Pedal position | Physical pedal | nRF52840 GPIO | Input mode                   | Default key |
| -------------- | -------------- | ------------- | ---------------------------- | ----------- |
| 1              | center         | P0.31         | active-low, internal pull-up | F16         |
| 2              | left           | P0.29         | active-low, internal pull-up | F17         |
| 3              | right          | P0.02         | active-low, internal pull-up | F18         |

The original FS2020U1 IR break-beam sensor boards have been replaced with plain
switches, because the IR sensors drew roughly 16 mA continuously (whole device
measured at ~0.08 W from USB) and their outputs only fell to ~1.5 V against the
nRF52840's internal pull-ups.

Wire each switch between its GPIO and **GND** (use COM + NO on three-terminal
switches); the grounds may be shared. No external resistors or 3.3 V
connection are needed because the GPIOs use internal pull-ups. Expect ~3.3 V
at rest and ~0 V while pressed.

The keycodes are the three `&kp` bindings in `config/pedal.keymap`. F16–F18 do
not produce visible text; use a key event viewer to test them.

## Bluetooth controls

ZMK supports five BLE profiles by default.

- Hold the two right pedals (center + right, pedals 1 and 3) and tap the left pedal (pedal 2) to clear/reset the selected Bluetooth profile.
- Hold all three pedals down for 5 seconds to enter UF2 bootloader mode (mounts the USB drive for flashing).

An unpaired selected profile advertises for pairing. Select it, then choose
`BT Pedals` from the host's Bluetooth pairing screen. Remove the keyboard
from the host too when clearing a profile, then pair again.

USB remains ZMK's preferred output whenever USB and BLE are both connected.
The profile-management chord is still available while USB is attached; stock
ZMK keymaps cannot conditionally disable a combo based on USB attachment. This
configuration does not broadcast one pedal event to multiple hosts.

## Build

The repository root is a Zephyr module (`zephyr/module.yml`) so that
`boards/shields/pedal` is discovered. The local west workspace therefore lives
in the git-ignored `.zmk/` directory; placing it at the repository root would
put Zephyr's own `zephyr/` checkout where the module metadata lives and break
module discovery.

One-time workspace setup, from this repository root:

```sh
mkdir -p .zmk && ln -s ../config .zmk/config
(cd .zmk && west init -l config && west update)
```

Build using the ZMK build container:

```sh
docker run --rm -v "$PWD":/work -w /work/.zmk zmkfirmware/zmk-build-arm:stable \
  sh -lc 'west zephyr-export && west build -p always -s zmk/app -d build/pedal \
    -b nice_nano_v2 -- -DSHIELD=pedal -DZMK_CONFIG=/work/config \
    -DZMK_EXTRA_MODULES=/work'
```

The expected artifact is `.zmk/build/pedal/zephyr/zmk.uf2`. GitHub Actions builds
the same target and uploads its firmware artifact.

## Flashing and hardware validation

The board target is selected because the hardware owner confirmed SuperMini
compatibility with nice!nano v2. The exact SuperMini revision and its
bootloader/reset procedure have not been verified. Do **not** assume the
nice!nano UF2 flashing procedure applies until the board's bootloader volume
and reset behavior are confirmed. Once verified, flash the generated UF2 using
that board-specific procedure.

The `nice_nano_v2` definition normally assumes VDDH battery sensing. This
configuration disables that inherited battery node because the SuperMini's
battery/sensing circuit is unverified. It makes no battery-life or idle-current
claim.

Physical test checklist:

1. Confirm board markings, bootloader entry method, USB data operation, and
   successful UF2 flashing/recovery.
2. With a meter or logic analyzer, confirm P0.31/P0.29/P0.02 are high at rest
   and low only while the corresponding pedal is pressed.
3. Verify F16, F17, and F18 each generate a key-down on press and key-up on
   release, including simultaneous independent presses.
4. Pair a fresh BLE profile, cycle profiles with the pedal chord, and verify
   profile clearing and re-pairing.
5. Check wake-from-sleep behavior, real idle current, and the actual battery
   sensing topology before enabling any battery reporting.
