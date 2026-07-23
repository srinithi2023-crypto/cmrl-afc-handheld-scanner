[Handheld Scanner Drive Requirements README.md](https://github.com/user-attachments/files/30290034/Handheld.Scanner.Drive.Requirements.README.md)
# Handheld Scanner Integration for CMRL Automated Fare Gates

A supplementary handheld QR/barcode scanner added to the Raspberry Pi 4 controlling a
CMRL automated fare gate, with a passive scan logger that records every handheld scan
**without modifying or interfering with the existing AFC gate software**.

---

## Table of contents

- [Overview](#overview)
- [The problem](#the-problem)
- [How it works](#how-it-works)
- [Hardware](#hardware)
- [Architecture and design decisions](#architecture-and-design-decisions)
- [Repository contents](#repository-contents)
- [Installation](#installation)
- [Configuring the scanner](#configuring-the-scanner)
- [Usage](#usage)
- [Log format](#log-format)
- [Running as a service](#running-as-a-service)
- [Troubleshooting](#troubleshooting)
- [Project status](#project-status)
- [Safety notes](#safety-notes)

---

## Overview

CMRL automated fare gates use a fixed scanner connected to a Raspberry Pi 4. A passenger
presents a QR ticket, the Pi's AFC (Automatic Fare Collection) reader software sends the
ticket to the AFC backend for validation, and the gate opens on a valid result.

This project adds a **Newland NLS-HR2081 handheld scanner** as a second scanning input for
use during peak crowd periods, together with a logging tool that records handheld scans for
audit and analysis.

The defining constraint of the project: the existing gate software is production
fare-collection infrastructure and must not be altered or destabilised. Every design choice
below follows from that.

---

## The problem

During peak hours a single fixed scanner per gate lane becomes a throughput bottleneck —
passengers queue faster than one lane can clear them. Station staff need a way to scan
additional passengers manually to drain the queue, without waiting on changes to the
certified AFC software stack.

Requirements:

1. Add a handheld scanning input to an existing, working gate.
2. Make no changes to the AFC reader or validation logic.
3. Record every handheld scan with a timestamp, for audit and crowd analysis.
4. Fail safe — if the added software stops, the gate must continue operating normally.

---

## How it works

```
  ┌──────────────────┐
  │ Handheld scanner │   Newland NLS-HR2081
  │   (USB)          │
  └────────┬─────────┘
           │  decoded ticket string
           ▼
  ┌──────────────────┐
  │  Pi device node  │   /dev/ttyACM0        (serial mode)
  │                  │   /dev/input/eventN   (HID keyboard mode)
  └────────┬─────────┘
           │
     ┌─────┴──────┐
     ▼            ▼
┌──────────┐  ┌──────────────┐
│AFC reader│  │ Scan logger  │
│(existing)│  │  (this repo) │
│          │  │              │
│validates │  │ timestamps   │
│opens gate│  │ and records  │
└──────────┘  └──────────────┘
```

A scan travels from the scanner into a Linux device node on the Pi, where it is read by two
independent programs:

- The **AFC reader** — the existing, unmodified software. It validates the ticket against
  the backend and opens the gate.
- The **scan logger** — this project. It reads the same scan and appends a timestamped
  record to a logfile.

The logger is a pure observer. It performs no validation, and has no code path that can
open, close, or block the gate.

---

## Hardware

| Item | Detail |
|---|---|
| Scanner | Newland NLS-HR2081-S0 (HR20 series) handheld barcode/QR scanner |
| Host | Raspberry Pi 4 (Raspberry Pi OS) |
| Connection | USB-A |
| Drivers | **None required** |

### On drivers

The HR2081 is driverless on Linux in both of its USB modes:

- **HID keyboard mode (HID-KBW)** — enumerates as a standard USB keyboard. Fully
  plug-and-play; scans arrive as keystrokes.
- **Serial mode (USB Virtual COM / CDC)** — handled by the `cdc_acm` module, which is built
  into the Raspberry Pi OS kernel and loads automatically on plug-in. Appears as
  `/dev/ttyACM0`.

No driver package needs to be installed on the Pi for either mode. (Vendor drivers are
sometimes needed for Virtual-COM on *Windows* — that does not apply here.)

---

## Architecture and design decisions

### 1. Passive observation — never `grab()` the input device

In HID mode, `evdev` allows a process to call `.grab()` to take exclusive control of an
input device. This logger deliberately **does not**. Grabbing would divert scans away from
the AFC reader and break fare collection. By reading passively, both programs receive every
scan and the logger can run live on a production gate.

### 2. Output-format matching instead of software changes

Rather than adapting the AFC reader to a new device, the handheld scanner is configured to
emit output identical to the existing fixed scanner — same USB mode, same line terminator,
no added prefix or Code-ID. The AFC reader therefore cannot distinguish a handheld scan from
a normal gate scan, and requires no modification.

### 3. Stable `by-id` device paths

The kernel-assigned numbers `ttyACM0` and `eventN` depend on plug order and can change
across re-plug or reboot. The logger resolves the scanner through its persistent aliases
instead:

```
/dev/serial/by-id/usb-Newland_NLS-HR20...-if00
/dev/input/by-id/usb-Newland_...-event-kbd
```

These point to the same physical device regardless of enumeration order. The bare numbered
node is used only as a fallback.

### 4. Handling the single-owner serial constraint

A serial port can be held open by only one process. If the scanner is in serial mode and the
AFC reader already owns `/dev/ttyACM0`, the logger cannot also open it. The code detects
this condition and reports it explicitly rather than failing silently.

**This is why HID keyboard mode is the recommended configuration for live logging** — input
event devices support multiple concurrent readers, so the logger and the AFC reader coexist
cleanly. Serial-mode logging is suitable for bench testing only.

### 5. Fail-safe separation

The logger runs as its own process (or notebook thread) with no coupling to gate control.
If it crashes or is stopped, the gate continues to operate normally.

---

## Repository contents

| File | Purpose |
|---|---|
| `handheld_scan_logger.py` | Standalone logger. Auto-detects mode, reconnects on unplug, designed to run as a systemd service. |
| `handheld_scanner_logger.ipynb` | Jupyter notebook version — device detection, background-threaded logger, live scan view, and pandas/matplotlib log analysis. |
| `README.md` | This document. |

---

## Installation

Run on the Raspberry Pi itself — the scanner's device nodes exist only there.

```bash
# 1. Dependencies
pip3 install pyserial evdev

# 2. Device permissions (dialout = serial ports, input = HID event devices)
sudo usermod -aG dialout,input $USER
```

Log out and back in for the group membership to take effect.

For the notebook, additionally:

```bash
pip3 install jupyter pandas matplotlib
```

---

## Configuring the scanner

Before logging, confirm the scanner is detected and identify which mode it is in.

```bash
lsusb                    # a "Newland" entry should appear
dmesg -w                 # then unplug/replug to watch it attach
```

Interpreting `dmesg`:

| Output contains | Mode | Device node |
|---|---|---|
| `cdc_acm`, `ttyACM0` | Serial (Virtual COM) | `/dev/ttyACM0` |
| `input`, `-kbd`, `event` | HID keyboard | `/dev/input/eventN` |

Then retrieve the stable path:

```bash
ls -l /dev/serial/by-id/     # serial mode
ls -l /dev/input/by-id/      # HID mode
```

### Matching the existing scanner

Using the Newland HR20 user-manual configuration barcodes, set the handheld to match the
existing fixed scanner:

- **USB mode** — HID-KBW or USB Virtual COM, matching the existing unit
  (HID-KBW recommended, see design decision 4)
- **Suffix terminator** — `CR`, `LF`, or `CR+LF`, matching the existing unit
- **Prefix / Code-ID / AIM-ID** — disabled unless the existing unit transmits them

To verify the match, capture raw output from each scanner and compare byte-for-byte:

```bash
python3 -c "import serial; s=serial.Serial('/dev/ttyACM0',9600,timeout=5); print(repr(s.readline()))"
```

Both should produce the same shape, e.g. `b'A1234567890\r\n'`.

---

## Usage

### Standalone script

```bash
python3 handheld_scan_logger.py                          # auto-detect mode
python3 handheld_scan_logger.py --mode hid               # force HID
python3 handheld_scan_logger.py --mode serial            # force serial
python3 handheld_scan_logger.py --device /dev/input/by-id/usb-Newland_...-event-kbd
python3 handheld_scan_logger.py --logfile /var/log/handheld_scans.log
```

### Notebook

Open `handheld_scanner_logger.ipynb` on the Pi and run the cells in order:

| Cells | Purpose |
|---|---|
| 1–2 | Install dependencies, set configuration |
| 3–4 | Device-discovery helpers and HID keymap |
| 5 | **Detect the scanner** — reports HID or serial mode |
| 6–7 | Define and start the background logger |
| 8 | Live view of incoming scans (interrupt to stop watching; logging continues) |
| 9 | Stop logging |
| 10–11 | Load the log into pandas, chart scans per minute |

The logger runs on a daemon thread, so the notebook kernel stays responsive throughout.

---

## Log format

Tab-separated, one record per scan:

```
2026-06-08T14:03:22.481	hid	A1234567890
2026-06-08T14:03:25.107	hid	A1234567891
```

| Field | Description |
|---|---|
| `timestamp` | ISO-8601 local time, millisecond precision |
| `mode` | `hid` or `serial` — how the scan was captured |
| `data` | The decoded ticket string |

The scans-per-minute chart in the notebook is built from this file and is useful for
identifying actual peak-crowd windows at a station.

---

## Running as a service

```bash
sudo tee /etc/systemd/system/handheld-scan-logger.service >/dev/null <<'EOF'
[Unit]
Description=CMRL handheld scanner logger
After=multi-user.target

[Service]
ExecStart=/usr/bin/python3 /home/pi/handheld_scan_logger.py --logfile /var/log/handheld_scans.log
User=pi
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now handheld-scan-logger
journalctl -u handheld-scan-logger -f
```

Log rotation, so a busy station does not fill the disk:

```bash
sudo tee /etc/logrotate.d/handheld_scans >/dev/null <<'EOF'
/var/log/handheld_scans.log {
    daily
    rotate 14
    compress
    missingok
    notifempty
    copytruncate
}
EOF
```

---

## Troubleshooting

| Symptom | Cause | Resolution |
|---|---|---|
| Scanner absent from `lsusb` | Cable or port issue | Reseat the connector; try another USB port |
| `Permission denied` opening the device | User not in `dialout` / `input` | Run the `usermod` step, then log out and back in |
| `Could not open /dev/ttyACM0` — port busy | AFC reader already holds the serial port | Switch the scanner to HID-KBW mode for live logging, or bench-test with the reader stopped |
| Logger finds nothing after a reboot | Device number changed | Use the `/dev/serial/by-id/` or `/dev/input/by-id/` path |
| Scans logged but the gate rejects them | Output format differs from the fixed scanner | Re-scan the configuration barcodes; compare raw `repr()` output between the two scanners |
| Garbled characters in the log | Ticket payload uses characters outside the HID keymap | Extend the keymap dictionaries in the notebook / script |

---

## Project status

**Implemented and working**

- Driverless USB integration of the HR2081 on the Raspberry Pi 4
- Dual-mode (HID and serial) scan capture with automatic detection
- Passive, non-intrusive logging architecture verified against the no-software-change constraint
- Stable `by-id` device resolution and automatic reconnection
- Timestamped logging, systemd service, log rotation
- Notebook with live scan view and scans-per-minute analysis

**Open item**

Confirmation of whether the existing fixed scanner operates in **HID or serial mode**, via
`lsusb` / `dmesg` on the deployed unit. This determines the exact Newland configuration
barcodes required, and whether live parallel logging works directly (HID) or is limited to
bench testing (serial).

---

## Safety notes

- The logger performs **no fare validation**. Ticket validity remains entirely with the AFC
  backend.
- The logger has **no gate-control code path**. It cannot open, close, or block a gate.
- In HID mode the logger never calls `.grab()`, so it cannot divert scans from the AFC
  reader.
- All changes should be verified on a **test gate** before deployment, and never introduced
  on a live gate during peak hours.
- Scan logs contain ticket identifiers and should be handled according to CMRL data-handling
  policy.
