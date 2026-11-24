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
