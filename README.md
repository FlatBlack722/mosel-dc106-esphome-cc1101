# MOSEL DC106 Guillotine Windows → Home Assistant via ESPHome + CC1101 (433 MHz)

This project integrates **motorized guillotine windows** (vertical sliding windows) controlled by a **MOSEL DC106** RF remote into **Home Assistant** using an **ESP32-C6** running **ESPHome** with a **CC1101** 433 MHz transceiver.

It provides basic window control:
- Open / Close / Stop
- A position slider using **time-based estimation** (`time_based` cover)

---

## Remote Used (Confirmed)

Remote details (from the label):
- **Brand:** MOSEL
- **Model:** **DC106**
- **Type:** Nine-channel transmitter
- **Printed band:** **433.05–434.79 MHz**
- **Supply:** 3VDC

Even though the remote is branded MOSEL, the RF frames decode successfully using ESPHome’s built-in **Dooya** protocol decoder/transmitter:
- `remote_receiver: dump: dooya`
- `remote_transmitter.transmit_dooya`

---

## Important Note: “Inverted %” (Home Assistant limitation)

Home Assistant’s cover UI currently does not provide a built-in “reverse/invert cover direction” option.

This configuration intentionally uses **inverted percentage semantics** so the slider behaves as:
- **0% = open**
- **100% = closed**

Because of this, the **Open / Close buttons in Home Assistant may feel inverted**.  
This is **expected behavior** in this setup.

---

## Hardware

### Required
- **Seeed Studio XIAO ESP32-C6**
- **CC1101** RF transceiver module (SPI)
- **433 MHz antenna**
  - Quarter-wave length ≈ **17.3 cm**
- Jumper wires / solder / protoboard
- USB cable for flashing and power

### RF settings used
- **Frequency:** 433.92 MHz (within the remote’s printed band)
- **Modulation:** ASK / OOK

---

## Wiring (XIAO ESP32-C6 ↔ CC1101)

This uses the XIAO “Dx” pin labels as used in ESPHome:

| XIAO ESP32-C6 | CC1101 |
|---------------|--------|
| 3V3           | VCC    |
| GND           | GND    |
| D8            | SCK    |
| D10           | MOSI   |
| D9            | MISO   |
| D3            | CS / SS |
| D1            | GDO0 (TX for `remote_transmitter`) |
| D2            | GDO2 (RX for `remote_receiver`) |

> ⚠️ CC1101 boards may label pins differently.  
> This setup assumes:
> - **GDO0 = TX**
> - **GDO2 = RX**

---

## How the Remote Was Decoded

### 1) Enable Dooya RF dump in ESPHome

To decode the remote, configure the receiver dump:

~~~yaml
remote_receiver:
  pin: D2
  dump:
    - dooya
~~~

Flash the firmware and open ESPHome logs, then press the remote buttons.

---

### 2) Record these fields from logs

For each button press, note:
- `id` (remote identifier)
- `channel` (which window/device)
- `button`
- `check`

⚠️ The **check values matter**.  
For this remote, **UP** and **DOWN** each involved **two frames** with different `check` values.

---

### 3) Safe testing tip

Before transmitting to real windows, test on an **unused channel** if possible to avoid accidental movement.

---

## Decoded Values Used in This Setup

### Remote ID

~~~
0x00FF3421
~~~

### Channels (3 windows)
- **Small right:** 33
- **Small left:** 34
- **Large:** 35

### Button + check mapping (implemented in scripts)

**UP (`dooya_up`)**
- `button = 1`
  - frame 1: `check = 1`
  - frame 2: `check = 14`

**DOWN (`dooya_down`)**
- `button = 3`
  - frame 1: `check = 3`
  - frame 2: `check = 12`

**STOP (`dooya_stop`)**
- `button = 5`
  - frame 1: `check = 5`

---

## ESPHome Code

The final working ESPHome YAML is located at:

~~~
esphome/balcony-window-bridge.yaml
~~~

Wi-Fi credentials are intentionally **not stored** in this repository.

Use:

~~~
esphome/secrets.yaml.example
~~~

Copy it to:

~~~
esphome/secrets.yaml
~~~

---

## Troubleshooting

### ESPHome error: `No such file or directory: 'secrets.yaml'`

If compiling with the ESPHome CLI on your PC, you must create:

~~~
esphome/secrets.yaml
~~~

See the example file provided.

---

### Nothing decodes in logs
- Verify wiring (especially **GDO2** for RX)
- Confirm CC1101 is powered at **3.3V**
- Check antenna connection and placement
- Confirm RF settings:
  - 433.92 MHz
  - ASK / OOK

---

### Transmit works but windows don’t respond
- Verify correct `id` and `channel`
- Ensure the same `button` + `check` values are transmitted
- Increase `tx_repeats` or adjust `tx_wait` if needed

---

## Safety / Disclaimer

This project transmits RF commands that may move physical windows.

- Use only with devices you own
- Ensure safe operation around people, pets, and objects
- **Rolling-code or encrypted remotes are not supported**

You assume all responsibility for safe use.
