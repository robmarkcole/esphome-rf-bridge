# esphome-rf-bridge

An ESPHome RF bridge for Home Assistant, starting with passive detection of an
existing AVANTEK doorbell and expanding to ceiling fans and other RF devices.

## Status

Planning and hardware bring-up. No doorbell or fan signals have been captured or
verified with this setup yet. The YAML below is a starting configuration, not a
hardware-tested integration.

The first milestone is to detect a doorbell press in Home Assistant while keeping
the original button and chime working normally.

## Hardware

| Component | Status and purpose |
| --- | --- |
| AVANTEK wireless doorbell, ASIN B07KG9VXF6 | Existing button and chime; believed to use 433.92 MHz, pending verification |
| ESP32-WROOM-32 development board | Available; runs ESPHome and connects over Wi-Fi |
| RXB6 433 MHz superheterodyne receiver | Reported available in the initial inventory; verify the actual board before wiring |
| DollaTek receiver, ASIN B07DK5ZC77 | Ordered; exact model, frequency, pinout and voltage requirements remain unverified |
| Jumper wires and USB power | Available |
| Ceiling fan and physical remote | Planned second integration; make, model, frequency and protocol still needed |
| RF transmitter or transceiver | Future addition if needed for fan control; not yet selected or ordered |

The [ordered receiver](https://www.amazon.co.uk/dp/B07DK5ZC77) appears to be a
DollaTek superheterodyne receiver based on a matching ASIN reference. Confirm its
actual specifications when it arrives; do not assume it is an RXB6.

## Phase 1: capture the doorbell

### 1. Verify and wire the receiver

Check the receiver's markings and datasheet for its frequency, pinout, supply
voltage and DATA output level. Pin order varies between modules.

| Receiver connection | ESP32 connection |
| --- | --- |
| VCC | 3V3 only if the receiver is documented to operate at 3.3 V; otherwise use its specified supply |
| GND | GND, shared with any external receiver supply |
| DATA | GPIO27, directly only if the signal is safe for a 3.3 V GPIO |

Do not feed a 5 V DATA signal directly into the ESP32. If the receiver needs a
5 V supply, verify its output level and add suitable level shifting if required.
The supply voltage alone does not establish the DATA voltage.

Use the receiver's supplied antenna, if present. For a verified 433.92 MHz module,
a straight wire of approximately 17.3 cm is a possible quarter-wave antenna;
connect it only to the identified antenna point. Keep wiring short and place the
receiver away from noisy USB supplies and other interference sources.

This is passive reception: leave the AVANTEK button and chime unchanged.

### 2. Flash an ESPHome capture configuration

Start with this configuration, assuming a typical ESP32-WROOM-32 development
board supported by `esp32dev`. Confirm the board selection for the actual board.

```yaml
esphome:
  name: rf-bridge
  friendly_name: RF Bridge

esp32:
  board: esp32dev
  framework:
    type: esp-idf

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

logger:
  level: DEBUG

api:
  encryption:
    key: !secret rf_bridge_api_key

ota:
  - platform: esphome
    password: !secret rf_bridge_ota_password

remote_receiver:
  id: rf_receiver
  pin:
    number: GPIO27
    mode:
      input: true
      pullup: false
  dump: all
  filter: 50us
  idle: 10ms
  tolerance: 25%
```

Create a private `secrets.yaml` next to the device YAML with these keys:

```yaml
wifi_ssid: "YOUR_WIFI_SSID"
wifi_password: "YOUR_WIFI_PASSWORD"
rf_bridge_api_key: "REPLACE_WITH_A_GENERATED_32_BYTE_BASE64_KEY"
rf_bridge_ota_password: "REPLACE_WITH_A_STRONG_PASSWORD"
```

The API key placeholder is not a valid key. Generate one through ESPHome's setup
workflow or its [API documentation](https://esphome.io/components/api/).
Before creating configuration files, add `secrets.yaml`, `.esphome/` and local log
files to `.gitignore`. Commit only a `secrets.example.yaml` with placeholders.

Validate the device YAML with ESPHome, flash over USB for the first installation,
then open the logs. Add the device through Home Assistant's ESPHome integration
using the configured API encryption key.

### 3. Capture and identify the signal

1. Record a short idle baseline to understand background RF noise.
2. Press the doorbell at least five times, several seconds apart. Confirm the
   original chime still sounds and save the log output for each press.
3. Look for a consistent decoded protocol and code. Preserve leading zeroes in
   binary codes and record the protocol number as well as the data.
4. If no decoder recognises it, compare complete raw pulse sequences across
   presses. A raw matcher or custom decoder may be needed.
5. If there is no correlated signal, check power, DATA wiring, antenna and actual
   doorbell frequency/modulation before assuming a decoding problem.

The receiver timings are initial settings. `filter` discards short pulses,
`idle` determines frame boundaries, and `tolerance` controls allowed timing
variation. Tune them from captures. Increasing tolerance is not automatically
better at rejecting false matches.

### 4. Match the doorbell and suppress repeats

If captures confirm an RCSwitch-compatible signal, append a sensor like this to
the capture configuration. **Both the code and protocol below are illustrative
placeholders, not verified AVANTEK values.**

```yaml
binary_sensor:
  - platform: remote_receiver
    receiver_id: rf_receiver
    name: "Doorbell"
    rc_switch_raw:
      code: "100010000000000010111110"  # Replace with captured binary data
      protocol: 1                     # Replace with captured protocol
    filters:
      - delayed_off: 1s
```

The sensor turns on immediately after a match and stays on until one second has
passed without another matching packet. This groups repeats from one press;
separate presses less than a second apart will also merge. Adjust after testing.
For another decoded protocol, use its corresponding matcher. For an unsupported
protocol, investigate `raw` pulse matching rather than inventing an RCSwitch code.

Once detection works reliably, reduce or disable capture dumping to limit log
noise. Keep debug captures available for troubleshooting.

## Phase 2: Home Assistant automations

Trigger on the doorbell binary sensor changing to `on`. Confirm its actual entity
ID in Home Assistant; it may differ from the example below.

```yaml
alias: RF doorbell notification
triggers:
  - trigger: state
    entity_id: binary_sensor.rf_bridge_doorbell
    to: "on"
actions:
  - action: notify.mobile_app_your_phone
    data:
      title: "Doorbell"
      message: "Someone is at the door."
mode: single
```

Replace the phone notification action with the actual action for your device.
Later automations can announce on smart speakers, update a Pimoroni Presto display
or switch lights. Choose the Presto integration path once its existing software
and connection to Home Assistant are known.

Acceptance checks:

- Each normal press produces one Home Assistant trigger and the original chime.
- Repeated RF packets do not produce duplicate notifications.
- Unrelated RF traffic does not trigger the doorbell during an extended idle test.
- Detection works from the installed button location and after an ESP32 reboot.
- The physical doorbell continues working when the ESP32 or Home Assistant is off.

## Phase 3: ceiling fan integration

First identify the fan make/model and any remote model number, frequency or
regulatory identifier printed on the remote or inside its battery compartment.
Do not assume a fan uses 433 MHz or the same modulation as the doorbell.

For a compatible signal, use the receiver to capture each remote button several
times and record the button, code, pulse timings, repeats and any address or
checksum fields. Compare short presses and held buttons where relevant.

The current receiver cannot transmit. If Home Assistant is to control the fan,
select a compatible transmitter or transceiver after identifying the protocol.
A 433 MHz CC1101 module is one possible route for supported sub-GHz signals;
retain the existing ESP32 and map SPI/data pins for the WROOM-32 board.

Keep doorbell reception working during this expansion. A CC1101 switches between
receive and transmit, so a shared radio has a reception gap while transmitting.
A separate doorbell receiver may be useful, though a nearby transmitter can still
temporarily interfere with reception.

Start fan transmission with individual command buttons and verify the physical
response. Add fan/light entities only after command behaviour is understood.
Distinguish absolute commands from toggles and relative adjustments. Without
feedback from the fan, displayed state is assumed; hearing the physical remote
does not prove the fan executed its command.

### Reference project: Adroz/esphome-rf-bridge

This [reference project](https://github.com/Adroz/esphome-rf-bridge) documents
Claro Whisper DC fans using 433.92 MHz OOK PWM, a 33-bit frame, a per-remote address
and an XOR checksum. Its captures and
[protocol notes](https://github.com/Adroz/esphome-rf-bridge/blob/main/PROTOCOL.md)
are useful if our fan proves compatible. Its addresses and command bytes are not
universal fan codes.

Review findings from the source inspected on 7 November 2026
(`0baa4441a220fd449e0157f77fa3d8e7a72f9a87`):

- The latest configuration claims hardware verification, but the runbook says
  physical fan control is unconfirmed and reception/state synchronisation fails.
- The receive decoder accumulates 33 bits in a `uint32_t`, losing the first address
  bit. It needs a wider accumulator or separate stop-bit handling.
- Receive handling uses entity control calls that can invoke transmit actions.
  State updates must avoid unintended retransmission.
- Repeated received packets are not deduplicated before light toggles.
- The older and newer files disagree on SPI wiring; do not copy their C3 pin map
  onto the WROOM-32.

Treat this as protocol research and an implementation reference requiring fixes
and hardware validation, rather than a ready-to-flash configuration.

## Repository organisation as the project grows

```text
esphome/                   # Device YAML and reusable per-device packages
captures/doorbell/         # Labelled captures and decoding notes
captures/ceiling-fan/      # Labelled captures and decoding notes
docs/                     # Wiring, protocol details and troubleshooting
home-assistant/            # Example automations
secrets.example.yaml       # Placeholder keys only
```

This structure is planned; the implementation files have not been created yet.
Keep device-specific matching and debounce rules separate so adding a fan does
not change the doorbell's behaviour. RF addresses in captures can identify devices;
review capture contents before publishing them.

## References

- [ESPHome remote receiver](https://esphome.io/components/remote_receiver/)
- [ESPHome ESP32 platform](https://esphome.io/components/esp32/)
- [ESPHome binary sensor filters](https://esphome.io/components/binary_sensor/#binary-sensor-filters)
- [ESPHome CC1101 transceiver](https://esphome.io/components/cc1101/)
- [YGG RF doorbell guide](https://www.ygg.cz/en/guide/esp32-rf433-doorbell/)
- [Adroz fan integration runbook](https://github.com/Adroz/esphome-rf-bridge/blob/main/RUNBOOK.md)

## Immediate next step

Verify the receiver board and safe DATA voltage, wire it to GPIO27, validate and
flash the capture configuration, then save the first set of doorbell captures.
Identify the fan and remote model before selecting transmission hardware.
