# Project 3215 – SPI-Based Dual-Node Embedded Locker Security System

**Group 2B:** Pratham Khatri, Harsh Khandelwal, Ali Khan, Jaevana Marryshow

---

## 1. What's in the ZIP

| File | Type | Size | Purpose |
|---|---|---|---|
| `Project 3215.docx` | Report | ~1.1 MB | Full write-up: abstract, architecture, protocol spec, code walkthroughs, challenges, and test/validation results |
| `Code Text files/Master Code.txt` | HCS12 C source | 218 lines | Firmware for the **Master** DragonBoard (admin console) |
| `Code Text files/Slave Code.txt` | HCS12 C source | 423 lines | Firmware for the **Slave** DragonBoard (locker/user terminal) |
| `project3215 video.mp4` | Video | ~68 MB | Demo recording of the working hardware |

---

## 2. What the project is

This is an embedded-systems (HCS12 microcontroller) project that simulates a **two-stage, two-node banking/storage locker security system**, built on two Axiom "DragonBoard" HCS12 boards talking to each other over **SPI (Serial Peripheral Interface)**.

The idea: a person requests access at a physical locker (the **Slave** node). That request has to be remotely approved by an administrator at a separate console (the **Master** node) before the person can even attempt to enter a PIN. So opening the locker requires:

1. **Remote authorization** – Admin approves/denies the request on the Master.
2. **Local authentication** – User enters a PIN on the Slave's keypad.

Only when both checks pass does a servo motor "unlock" the locker and a green LED lights up.

### Hardware used
- 2× HCS12 DragonBoards (Master + Slave)
- 4×4 matrix keypad (input, on the Slave)
- HD44780 character LCD (status display, on both — LCD driven via Port K)
- Multiplexed 7-segment display (on the Slave)
- Servo motor (the "lock" actuator, PWM-driven, Port P)
- Red/Green LEDs (lock-state indicator, PP4/PP5, common cathode to ground)
- SPI wiring between boards (Port S) + a shared/common ground (important — see Challenges below)

---

## 3. System architecture

**Master–Slave topology**, with the Master as SPI bus controller (generates the clock) and the Slave as the peripheral:

- **Master (Admin console):** Runs the LCD status display, receives the admin's approve/deny input, and displays the transmitted PIN for verification. Initiates every SPI transaction (it polls the Slave).
- **Slave (Locker terminal):** Owns the physical hardware — keypad scanning, 7-segment refresh, the PIN database, and the servo/LED lock mechanism.

This split is deliberate: the Slave's keypad scan and 7-segment multiplexing need to run very frequently (else the display flickers), while the Master is free to handle LCD text and high-level decisions. Splitting the work across two CPUs avoids one board being overloaded.

### Why SPI + polling (not interrupts)
SPI's Master/Slave hardware maps directly onto the "admin/locker" relationship, and the HCS12's dedicated hardware SPI module avoids CPU-heavy bit-banging. The team chose **polling** over interrupts specifically because the PIN handshake needs deterministic timing — they need to know exactly when each PIN digit byte has arrived so the receive buffer doesn't get out of sync.

---

## 4. Communication protocol (the interesting part)

Everything rides on top of raw SPI as a tiny single-byte command protocol. The Master continuously polls; the Slave replies with a status byte that drives a state machine on both ends:

| Byte | Sent by | Meaning | Leads to |
|---|---|---|---|
| `'R'` (0x52) | Slave | Access requested by user | Master prompts admin |
| `'Y'` (0x59) | Master | Admin approves | Slave starts locker selection |
| `'N'` (0x4E) | Master | Admin denies | System resets |
| `'S'` (0x53) | Slave | User is selecting a locker | Master resets its timeout |
| `'K'` (0x4B) | Slave | "Key" header — PIN digits follow | Master enters a 4-byte read loop |
| `'O'` (0x4F) | Slave | Correct PIN entered | Master shows "Unlocked" |
| `'D'` (0x44) | Slave | Transaction complete | Master returns to idle |

The standout feature is the **"secret handshake": the Slave streams the correct locker PIN over to the Master in real time**, so the admin can visually confirm what the user should be typing. When the Slave sends the `'K'` header, the Master pauses its polling and does a tight, timed 4-byte read (with a small `MSDelay(20)` between bytes) to pull in each PIN digit before display.

---

## 5. Firmware structure

**Master Code.txt** implements:
- LCD driver (nibble-mode HD44780 routines: init, write, clear, position)
- Keypad reader for admin Y/N input
- `SPI_Init_Master()` — configures Port S (MOSI/SCK/SS as outputs, MISO as input), sets `SPI0CR1 = 0x50` (SPI enabled, Master mode, Mode 0 clock), sets baud rate register for ~375 kHz
- `SPI_Exchange()` — the blocking send/receive primitive: pulls Slave Select low, waits for transmit-buffer-empty, writes the byte, waits for transfer-complete, reads the reply, then raises Slave Select high again
- A polling main loop that interprets the protocol bytes above and drives the LCD

**Slave Code.txt** implements:
- Keypad scanning (4×4 matrix)
- 7-segment multiplexed display driver
- A PIN "database" (`struct` array mapping locker ID → PIN)
- Servo/LED control for lock/unlock states (PWM duty ~5% = locked, ~10% = open)
- `SPI_Init_Slave()` — mirrors the Master's setup but configures Port S the opposite way (MOSI/SCK/SS as inputs, MISO as output) and sets `SPI0CR1 = 0x40` (Slave mode; baud rate register is irrelevant since the Master drives the clock)
- The state logic that emits `'R'`, `'S'`, `'K'`, `'O'`, `'D'` at the right points

---

## 6. Notable engineering decisions & problems solved

- **Race condition in PIN transfer:** the Master was initially reading SPI faster than the Slave could load new digits, causing repeated/garbled digits. Fixed with a 20 ms software delay per byte in the Master's read loop, giving the Slave time to interrupt its 7-segment refresh and load the next digit.
- **Custom header incompatibility:** the lab's `hcs12.h` didn't define convenient bit-field macros (like `PTS_PTS7`), so the team used raw bitwise operations (`PTS &= ~0x80` etc.) throughout — more portable across CodeWarrior/GCC.
- **Floating ground / signal noise:** Master (USB-powered) and Slave (battery-powered) had different reference voltages, causing garbage characters on the LCD. Fixed by adding a shared ground wire between the boards.
- **Dynamic timeout:** the system auto-resets after 20 seconds of Slave inactivity, *except* during PIN entry, where the timeout is disabled so the user isn't rushed.

---

## 7. How to run it (from the report)

1. Open CodeWarrior IDE and create **two** separate projects — one for Master, one for Slave.
2. Paste `Master Code.txt` into the Master project's `main.c`, and `Slave Code.txt` into the Slave project's `main.c`.
3. Connect both DragonBoards' power and USB, and note which COM port is which.
4. Wire the hardware per the report's diagrams (servo on Master's PP7; red/green LEDs on PP4/PP5 with common cathode to ground; keep the SPI pin wiring separate from that).
5. Flash and run the **Slave** program first.
6. Once the Slave is running, flash/run the **Master** program and disconnect the Slave's USB (so it's not still tethered to the IDE).
7. Both boards now run independently and communicate purely over the SPI wiring.

---

## 8. Validation results (from the report's test log)

- Both boards flash `8888` on the 7-segment at boot to confirm the driver works.
- User request → Master prompt → **admin denies** → Slave shows "ADMIN DENIED", system resets. ✅
- User request → **admin approves** → Slave prompts locker selection → Master immediately displays the correct PIN for that locker. ✅ (flagged as the critical deliverable)
- Wrong PIN entered → "ACCESS DENIED", servo stays locked. ✅
- Correct PIN entered → servo rotates to ~90° (open), green LED on, Master shows "Locker Unlocked". ✅
- `*` key re-locks and resets the system. ✅

All functional requirements were met per the report.

---

*Note: I extracted this summary from the report text and both source files directly; I haven't watched the demo video, so let me know if you'd like me to pull anything specific from it.*
