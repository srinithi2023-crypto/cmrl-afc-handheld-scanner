
# Handheld QR Scanner Integration for Automated Fare Collection (CMRL)

## Overview

This project was developed during my internship at **Chennai Metro Rail Limited (CMRL)** under the Automated Fare Collection (AFC) Department.

The objective was to integrate a handheld QR scanner with the existing AFC infrastructure to improve passenger handling during peak-hour congestion without modifying the existing fare collection software.

---

## Features

- Raspberry Pi 4 based implementation
- USB HID & Serial scanner integration
- Automatic device detection
- Real-time scan logging
- Timestamped scan records
- Scan analytics using Pandas

---

## Technologies

- Python
- Raspberry Pi 4
- Linux
- USB HID
- Serial Communication
- Pandas

---
## System Architecture

+------------------+
| Passenger QR Code|
+------------------+
          |
          v
+------------------+
| Handheld Scanner |
+------------------+
          |
       USB HID
          |
          v
+------------------+
| Raspberry Pi 4   |
+------------------+
          |
          v
+------------------+
| Python Logger    |
+------------------+
     |          |
     v          v
+---------+  +----------------+
| Logs    |  | Pandas Analysis|
+---------+  +----------------+

 ## Key Contributions
 • Integrated Newland NLS-HR2081 scanner with Raspberry Pi 4.

• Developed a Python-based scan logger with automatic device detection.

• Implemented timestamped logging and scan analytics.

• Assisted in deployment and testing within the AFC department.

## Skills Demonstrated

Python
Linux
Raspberry Pi
USB Communication
Industrial Automation
Embedded Systems
Data Logging
