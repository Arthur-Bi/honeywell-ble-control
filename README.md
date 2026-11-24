# Honeywell BLE light control — public framework

[简体中文](README.zh-CN.md) · English

A local web controller and ESP32-C3 firmware framework for a Honeywell
**HWX-SICS-07 (learning edition / 学习版)** lighting panel.
The panel label specifies **BLE / LeMesh**, powered by two CR3032 batteries.
The tested lamp was sold as a Honeywell **02BPlus** ceiling light. Compatibility
with other lamps, panel revisions or firmware versions has not been established.
This is an independent project, not an official Honeywell implementation.

<img src="docs/images/panel-front.png" width="360" alt="Front of the Honeywell HWX-SICS-07 lighting panel">

Only the panel's front photograph is distributed. Image file metadata has been removed.

## What is public, and what is deliberately private?

Included:

- The original simple Chinese web UI: scenes, favorite recall, off, manual/timed
  button holds, and bounded experimental absolute brightness/CCT controls.
- A loopback-only Python HTTP server and USB serial bridge.
- ESP-IDF 5.4.2 / ESP32-C3 NimBLE scanning and advertising framework.
- Semantic command codes and meaningful parameter mappings below.
- Read-only capture and candidate sender-ID inspection tools.

**No usable complete lamp-frame encoder is included.** Device-specific identity,
binding/authentication material, complete original advertisements, captured
fixtures, payload transformation/checksum logic and the original private encoder
are omitted. The shipped adapter always refuses to transmit.

The public firmware builds and can passively capture advertisements; the web
demo works without hardware. **Actual lamp control requires your own local
private adapter.** A candidate sender ID alone does not make this project a
working transmitter.

## Architecture

~~~text
Web browser -> local Python server -> USB serial -> ESP32-C3
                                                  |
                            semantic request + independent sequence counter
                                                  |
                                      LOCAL PRIVATE ADAPTER
                                                  |
                                       BLE advertisements -> lamp
~~~

The Python server runs on the connected computer, not on the ESP32. The existing
web workflow requires that computer to remain running. Standalone Wi-Fi, MQTT,
Home Assistant discovery, IR input and Mijia-module GPIO input are integration
ideas, not implemented features in this repository.

## Semantic command reference

The following are **operation meanings and parameters, not complete BLE frames**.
Lengths refer to semantic values here; a private adapter supplies any wire
padding, identity, authentication and framing required by its target.

| Operation | Opcode | Meaningful parameter(s) | Serial command |
| --- | --- | --- | --- |
| Off | 0x01 | No semantic parameter | off |
| Rest / 休息模式 | 0x0E | Scene 0; mode selector 1 | scene 0 |
| Homework / 作业模式 | 0x0E | Scene 1; mode selector 1 | scene 1 |
| Online class / 网课模式 | 0x0E | Scene 2; mode selector 1 | scene 2 |
| Morning wake / 早餐唤醒 | 0x0E | Scene 3; mode selector 1 | scene 3 |
| Soothing sleep / 舒缓助眠 | 0x0E | Scene 4; mode selector 1 | scene 4 |
| Nightlight / 伴读夜灯 | 0x0E | Scene 5; mode selector 1 | scene 5 |
| Recall “My favorite” / 我的最爱 | 0x0E | Scene 128 (0x80); mode selector 1 | scene 128 |
| Brightness button down / up | 0x28 | 1 = down; 2 = up | brightness_press / brightness_release |
| CCT button down / up | 0x46 | 1 = down; 2 = up | temperature_press / temperature_release |
| Absolute brightness candidate | 0x08 | Raw value: high byte, low byte | abs_b RAW LAYOUT |
| Absolute CCT candidate | 0x0D | Two relative channel parameters | abs_t POSITION LAYOUT |

The serial alias **on** selects the homework scene; it is not a claim that a
separate generic power-on opcode was verified. Favorite recall does not save
or overwrite a favorite. Long-press favorite/save and additional built-in favorite
slots are intentionally not implemented.

### Absolute values and their limits

- Web brightness candidate 5–60 maps to raw 50–600 by multiplying by ten.
  The two semantic bytes are raw / 256 and raw % 256.
- CCT position is restricted to 25–75. At positions 25 / 50 / 75, the semantic
  channel pairs are respectively (128, 64) / (128, 128) / (64, 128).
- For position p <= 50, the pair is (128, round(256*p/100)).
  For p > 50, it is (round(256*(100-p)/100), 128).
- Layout A uses seven wire-parameter positions; layout B uses two.
  The private adapter handles the full layout; the public request contains only
  meaningful values. Both layouts produced a visible effect on the tested lamp.

Observed: brightness candidate 20 was brighter than 10; CCT position 25 was cooler
than 75. These values are **not calibrated light-output percentages or Kelvin**.
The input bounds are this project's experimental bounds, not a manufacturer's
published maximum. Different starting states and other lamps require validation.

### Counters and broadcast timing

Each new logical command reserves a new 8-bit sequence number, wrapping 255 -> 0.
All repeated advertisements of that command retain the same number. The
transmitter counter is persisted in NVS. An observed physical-panel counter
never rewinds an already initialized transmitter counter.

Advertising is configured at a 20 ms interval for approximately 300 ms per
command, with an independent controller timeout. A 300 ms burst resembles the
observed panel burst duration. **The configured 20 ms interval is not claimed to
be an exact measurement of the original panel's RF timing.**
Transmit success is not an acknowledgement from the lamp.
## Obtain your own device binding information

**Use your own panel and lamp. Do not copy another user's sender identity.**
The BLE advertiser address, the controller/sender ID inside an advertisement,
and an authentication key are distinct concepts. This experiment identified a
stable controller-ID candidate; it did not establish that the ID is a cryptographic
key or a globally unique lamp ID.

1. Build/flash the public firmware above, close the web server and any serial
   monitor, and install the serial dependency:

   ~~~bash
   python3 -m venv .venv
   .venv/bin/python -m pip install -r requirements.txt
   .venv/bin/python tools/capture.py --serial-port YOUR_USB_PORT --seconds 30
   ~~~

2. Keep your panel near the C3. Leave it idle briefly, then press “off” five times
   about two seconds apart, followed by a few homework/favorite presses. Keep
   track of the timing. Capture is passive and sends no lamp-control command.
   It records nearby BLE advertisements, so keep the result private.
3. List candidate controllers locally:

   ~~~bash
   .venv/bin/python tools/inspect_binding.py .local/captures/panel.jsonl
   ~~~

4. Correlate report bursts with your own presses, repeat the capture if needed,
   and select that controller's observed address:

   ~~~bash
   .venv/bin/python tools/inspect_binding.py .local/captures/panel.jsonl \
     --address YOUR_CONTROLLER_ADDRESS
   ~~~

   The tool writes an ignored **.local/device-profile.json**. In the observed
   manufacturer-data variant, it reads only the four plain sender-ID bytes after
   the version byte. It does not decode the remaining payload, verify binding,
   derive authentication keys, or construct a control frame. An unrelated BLE
   device can resemble that header; the generated ID remains a candidate.
5. Keep that profile and all captures local. Complete and validate your private
   adapter using your own device information and protocol work. If your variant
   needs additional authentication material, it must be obtained legitimately
   for your device; an advertiser address or candidate ID is not a substitute.

The public project does not pair, rebind or overwrite lamp favorites.

## Add your private adapter locally

See [the private adapter interface](private/README.md).
Copy the unavailable example to an ignored local source, implement the interface
yourself, and explicitly select that file:

~~~bash
cp private/adapter.example.c private/adapter.local.c
idf.py -DLAMP_PRIVATE_ADAPTER=private/adapter.local.c build
~~~

An absolute path outside the repository also works. The original private
implementation is not present in this repository, including its untracked files.

With a completed adapter and initialized counter:

~~~bash
.venv/bin/python web/server.py --serial-port YOUR_USB_PORT
~~~

On a fresh NVS namespace, an adapter can recognize your physical panel to
initialize the counter. Alternatively, provide your own last known counter with
the optional **--seed-counter** argument. An initialized counter cannot be
rewound by a subsequent seed command.

## Home Assistant and Mijia integration ideas

These are proposed extensions, not claims that this public build is already
connected to those platforms.

| Route | What to add | Notes |
| --- | --- | --- |
| HA -> existing HTTP bridge | HA REST commands/scripts for /api/action and /api/absolute | The server binds loopback. HA must be able to reach that host; a different host/container needs deliberate network/authentication work. |
| HA -> C3 over MQTT | Wi-Fi/MQTT firmware and command dispatch to the same semantic operations | Can eventually remove the computer/USB dependency. |
| Mijia Mesh module -> C3 | Low-voltage, three-channel momentary dry-contact module and GPIO inputs | Channels map to original favorite, a stored brightness/CCT preset, and off. Native momentary reset avoids brightness encoding. Local/offline execution depends on the gateway and rule. |
| XiaoAi IR -> C3 | Compatible IR receiver and RMT decoding | Reuse three distinct remote commands exposed by the chosen Mijia template. Check automation availability, repeated identical commands and receiver placement. |
| Bemfa -> C3 | Wi-Fi/MQTT and a Bemfa account | Official device synchronization exists. Verify the required Mijia UI/automation actions and end-to-end latency on the current app. This adds a third-party cloud dependency. |

For a second favorite, store absolute brightness and CCT externally; no additional
favorite slot in the lamp has been verified. Validate recall from the off state
and check for an unwanted bright flash before relying on a preset.
The original panel can remain in use, but this one-way broadcast approach does
not provide authoritative lamp state feedback.

## References

- [Espressif ESP-IDF 5.4.2 / ESP32-C3 BLE device discovery](https://docs.espressif.com/projects/esp-idf/en/v5.4.2/esp32c3/api-guides/ble/get-started/ble-device-discovery.html)
- [smartLightConnector](https://github.com/v1ack/smartLightConnector) and [ha-ble-adv LE codec](https://github.com/NicoIIT/ha-ble-adv/blob/main/custom_components/ble_adv/codecs/le.py): community reverse engineering of related LE lighting protocols, **not Honeywell's official protocol specification**.
- [Bemfa device synchronization](https://cloud.bemfa.com/v2/docs/start/mi.html)
- [Xiaomi's Home Assistant integration](https://github.com/XiaoMi/ha_xiaomi_home): imports Xiaomi devices into HA; it does not by itself publish this custom C3 controller into Mijia.

Code is MIT licensed. The front photograph was supplied for this project.
