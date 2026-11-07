# W65C816 Studio + ROM-Resident BASIC

ALL TYPING MUST BE WITH CAPS ON FOR BASIC TO FUNCTION.

**Complete hardware, wiring, Windows IDE, BASIC ROM, and Tera Term guide**  
**Target:** Western Design Center W65C816SXB with W65C816EDU expansion board  
**Project documentation revision:** 2026-11-07

> **Important status:** This guide describes the user's reported BASIC installation and the currently packaged Windows Studio source. The proven MIND-side procedure uses `/home/sysop/w65c816-basic/tools/launch_rom.py` and a **59-byte RAM launcher**; the Windows Studio source supplied with this documentation instead contains a **13-byte ROM-switching trampoline**. The Windows **START BASIC** button has **not** been established as equivalent to the working MIND launcher or tested on the physical board. Do not interpret a successful build or a “GO sent” message as proof that BASIC actually started.

## Contents

1. [What the system does](#1-what-the-system-does)
2. [Hardware required](#2-hardware-required)
3. [Wire the SXB and UART](#3-wire-the-sxb-and-uart)
4. [Grounds, jumpers, and pins not to ground](#4-grounds-jumpers-and-pins-not-to-ground)
5. [Two independent PC connections](#5-two-independent-pc-connections)
6. [Tera Term configuration](#6-tera-term-configuration)
7. [Build and use W65C816 Studio](#7-build-and-use-w65c816-studio)
8. [BASIC ROM organization and launching](#8-basic-rom-organization-and-launching)
9. [Verified MIND-side launch procedure](#9-verified-mind-side-launch-procedure)
10. [Using BASIC](#10-using-basic)
11. [Flashing versus launching](#11-flashing-versus-launching)
12. [Troubleshooting](#12-troubleshooting)
13. [IDE examples and shortcuts](#13-ide-examples-and-shortcuts)
14. [Reference documents](#14-reference-documents)

## 1. What the system does

The W65C816SXB uses a W65C816S CPU, SRAM, 128 KB flash, a W65C51 ACIA serial interface, and a factory WDC monitor. The W65C816EDU board adds peripherals such as LEDs, an OLED interface, an RTC, and expansion RAM.

The reported installation has **Microsoft-style BASIC in flash overlay bank 0**, with its entry point at **CPU address `$00:8000`**. The factory **WDC monitor is in flash overlay bank 3**. Running BASIC should give the following text on the **separate UART terminal**:

```text
W65C816 BASIC
29951 BYTES FREE
READY
```

The 29,951-byte figure is the expected banner reported for this particular BASIC build; it is **not** a generic capacity guarantee for all flash images.

After starting BASIC, the computer runs BASIC locally. The PC terminal is only a keyboard and text display.

## 2. Hardware required

- WDC **W65C816SXB** single-board computer.
- WDC **W65C816EDU** board, correctly seated on the SXB's J1–J4 headers.
- Original ROM chip containing the WDC monitor plus the separately installed BASIC flash image.
- SXB **micro-USB programming/monitor cable**, connected to the computer running the IDE (or to MIND for the known Python procedure).
- A separate **USB-to-TTL UART serial adapter** compatible with the board's logic voltage. **Verify that the adapter's I/O levels are compatible with the SXB's 5 V logic before wiring.** A 5 V power output on an adapter does *not* necessarily mean its serial signal pins are 5 V tolerant.
- Three signal wires: TX, RX, ground; plus a short jumper for RTS-to-CTS on the SXB ACIA header if the installed BASIC expects it.
- Windows computer with **Tera Term** installed; Windows + **.NET 10 SDK** to build Studio from source.

**Do not connect a true RS-232 ± voltage serial output directly to the W65C51 header.** Use a TTL-level USB-UART adapter or an appropriate RS-232 line driver/receiver. Do **not** connect the adapter's +5 V/VCC supply wire to the powered SXB just to obtain serial communication.

## 3. Wire the SXB and UART

### SXB ACIA connector J4: exact documented pinout

This table comes from the official **W65C816SXB datasheet, page 8, J4 – ACIA Connector**. It is the physical 2×5 ACIA header on the SXB, not a generic 4-pin module header.

| J4 pin | WDC signal | Purpose for terminal hookup |
| ---: | --- | --- |
| **1** | VDD | **+5 V — leave disconnected from USB-UART VCC** |
| **2** | VSS | **Ground; connect to USB-UART GND** |
| **3** | RTSB | **Jumper to J4 pin 9 (CTSB)** for local RTS/CTS loopback |
| **4** | DTRB | Not required for three-wire terminal setup |
| **5** | RXD | **Connect USB-UART TXD → SXB RXD** |
| **6** | DSRB | Not part of the confirmed minimum connection; do not ground without a verified need |
| **7** | TXD | **Connect SXB TXD → USB-UART RXD** |
| **8** | DCDB | Not part of the confirmed minimum connection; do not ground without a verified need |
| **9** | CTSB | **Jumper to J4 pin 3 (RTSB)** |
| **10** | RXC | Receive clock; leave unconnected for the built-in asynchronous clock arrangement unless the firmware explicitly requires something else |

**Connection summary:**

```text
W65C816SXB J4                      USB-to-TTL UART adapter
--------------------------------------------------------------
J4 pin 2   VSS/GND  --------------> GND
J4 pin 5   RXD      <-------------- TXD
J4 pin 7   TXD      --------------> RXD

J4 pin 3   RTSB     --- jumper --- J4 pin 9  CTSB

J4 pin 1   +5 V     -- NOT CONNECTED to adapter VCC
J4 pins 4, 6, 8, 10  -- leave alone unless separately verified
```

**Physical orientation:** J4 is a dual-row header, and its **pin-1 location must be established using the PCB marking, square pad, or an official board drawing for your exact revision** before connecting anything. The table gives pin *numbers*, not an unverified left/right view. If the EDU covers J4, use the appropriately labeled **EDU UART breakout** and confirm signal-to-signal continuity against the schematic instead of trying to push a connector into the stacked header.

**EDU UART header:** The EDU schematic shows TXD/RXD and ground routed to UART-related connectors, but this document does not assume that a particular four-pin breakout has the same physical order as J4. **Follow the silk-screen labels on the exact EDU revision** and meter-check connections before power-up; never assume `GND TX RX VCC` ordering.

## 4. Grounds, jumpers, and pins not to ground

For the **normal BASIC UART hookup**:

- **Connect J4 pin 2 (VSS) to the UART adapter's GND**: this is the required common signal reference.
- **Connect J4 pin 3 (RTSB) to J4 pin 9 (CTSB)** if using the WDC-documented UART local loopback setup. This is **not a connection to ground**.
- **Do not ground J4 pin 1 (+5 V)**; that would short the board's supply.
- **Do not ground J4 pins 5 (RXD) or 7 (TXD)**; wire them to the opposite TX/RX signals of the adapter.
- **Do not manually ground the ROM bank-control lines** or short CA2/CB2 to ground. ROM selection is performed by **software** through the system VIA.
- **Do not ground DSRB or DCDB just because their names end in B** (active-low names). Their need depends on the firmware and hardware circuit, and the cited WDC terminal hookup calls for TXD, RXD, GND, and RTS/CTS loopback only.

Always power down before changing wires. Use a meter to confirm ground and the adapter's electrical levels first.

## 5. Two independent PC connections

```text
                           W65C816SXB + EDU
                         +--------------------+
Windows PC / MIND         | micro-USB monitor  |
  W65C816 Studio   <----> | WDC monitor / USB  |  (Studio's monitor protocol)
                         |                    |
Windows PC / Tera Term    | W65C51 ACIA UART   |
  USB-to-TTL UART  <----> | TXD / RXD / GND     |  (BASIC terminal)
                         +--------------------+
```

- **Studio's SXB USB monitor connection:** configured in the current source for **115200 baud, 8N1**, using its monitor protocol; it is **not** the human-readable BASIC console.
- **Tera Term's USB-UART connection:** **19200 baud, 8N1**, the human-readable BASIC console.
- Use **different COM ports**, each opened by only one application. Tera Term should never open Studio's monitor COM port.
- The board gets power from its SXB micro-USB cable. The USB-UART adapter contributes signals and a common ground only.

## 6. Tera Term configuration

1. Connect the separate USB-UART adapter to Windows. In **Device Manager → Ports (COM & LPT)**, identify its COM number.
2. Start **Tera Term**.
3. In **New connection**, choose **Serial**, then choose the **USB-UART COM port** (not the SXB programmer port). If the port was connected after starting Tera Term, reopen the connection dialog.
4. Open **Setup → Serial port** and set:

   | Setting | Value |
   | --- | --- |
   | Speed | **19200** |
   | Data | **8 bit** |
   | Parity | **None** |
   | Stop | **1 bit** |
   | Flow control | **None** |

5. Open **Setup → Terminal** and turn **Local echo OFF**. For line endings, start with **Transmit: CR** and **Receive: CR+LF / auto** depending on your Tera Term version. If the terminal double-spaces or line editing looks odd, adjust receive newline behavior; do not arbitrarily change baud rate.
6. Optionally save the settings through **Setup → Save setup**, so the next launch uses them.
7. Leave Tera Term open, press the SXB **RESET/S2** button to restore the monitor, then start BASIC using a known working launcher.
8. Once BASIC launches, you should see:

   ```text
   W65C816 BASIC
   29951 BYTES FREE
   READY
   ```

9. If the screen is blank, first verify TX/RX crossing, ground, the RTS/CTS jumper, correct COM port, and actual BASIC launch. **A blank terminal is not evidence that flash needs to be erased or reprogrammed.**

## 7. Build and use W65C816 Studio

This is a **Windows WPF / .NET 10** C# IDE, distributed as **source code**. It is not guaranteed to have been compiled or tested on Windows by this documentation package.

### Build

1. Extract the **entire** Studio ZIP to a new writable folder, keeping `Core`, `Studio`, `assets`, and `BUILD.cmd` together.
2. Install the Windows **.NET 10 SDK** (not just the desktop runtime).
3. Double-click `BUILD.cmd` from the extracted project root.
4. The script runs separate **restore → build → publish** stages and writes logs under `build-logs/`.
5. If all stages succeed, open `publish/W65C816Studio.exe`. Keep the published `tools/` and `examples/` assets alongside it.

You may also open `W65C816-Studio.sln` in Visual Studio, or run:

```bat
dotnet restore Studio\Studio.csproj -r win-x64
dotnet build Studio\Studio.csproj -c Release -r win-x64
dotnet publish Studio\Studio.csproj -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -o publish
```

**If BUILD.cmd fails:** read `build-logs/restore.log`, `build-logs/build.log`, or `build-logs/publish.log` and fix the actual first compiler/restore error. The current publish package is designed to report errors, but a successful Windows publish has **not been independently verified here**.

### IDE controls

| Control | Purpose |
| --- | --- |
| **Build / F6** | Save, assemble (`ca65`), and link (`ld65`); no board needed |
| **Build & Run / F5** | Assemble, link, upload to application RAM, verify, send monitor GO |
| **Upload only** | Build and transfer to RAM without immediately running |
| **Find / Ctrl+F** | Search the ASM editor |
| **Check board** | Query the SXB monitor over its USB connection |
| **START BASIC** | Attempt to launch the **already-flashed** ROM-resident BASIC; **does not flash BASIC** |

The IDE's normal assembly output is linked for application RAM at **`$00:2000`**, not directly into flash. The current Studio source's START BASIC implementation sends a **13-byte SRAM trampoline** to switch to **bank 0** and jump to **`$8000`**. This **has not been validated against the known working 59-byte launcher** and may fail even when BASIC is properly installed. **Do not use ordinary Build & Run as a flash-programming function.**

## 8. BASIC ROM organization and launching

The SXB contains 128 KB of flash, configured as four **32 KB overlays** in CPU address range **`$00:8000`–`$00:FFFF`**. This is not the same thing as the CPU's 24-bit memory bank number.

| Flash overlay | FAMS / CB2 | FA15 / CA2 | Purpose |
| --- | ---: | ---: | --- |
| **0** | 0 | 0 | **User-reported installed BASIC image; entry `$8000`** |
| 1 | 0 | 1 | Not established; do not assume empty |
| 2 | 1 | 0 | Not established; do not assume empty |
| **3** | 1 | 1 | **Factory WDCMON; power-on default** |

Source: WDCMON v2.0 manual, page 7. The **system VIA** drives FAMS/CB2 and FA15/CA2. A switching routine must run from **RAM**, because after a flash overlay changes, the monitor code at the old ROM address is no longer available. **Do not jumper these signals to ground.**

### Intended Windows START BASIC workflow

1. Connect the SXB's programmer USB to the **Windows** computer running Studio.
2. Connect the UART to a separate Windows USB-UART adapter and open Tera Term using the steps above.
3. Press the SXB **RESET/S2** button and select the **USB monitor COM port** in Studio.
4. Click **Check board**.
5. Click **START BASIC**.
6. Watch Tera Term for the BASIC banner and `READY`.

**Limitation of the present IDE:** The source does **not yet reproduce** the project's known 59-byte launcher and monitor/ROM verification helpers. If START BASIC fails, **do not reflash BASIC** merely to address this button. Compare the C# launcher in `Core/Monitor.cs` with `/home/sysop/w65c816-basic/tools/launch_rom.py` before making a verified replacement.

## 9. Verified MIND-side launch procedure

The user's MIND project supplies a launch routine reported to verify the WDC monitor and BASIC ROM, upload RAM verification helpers, load a **59-byte launcher**, and execute BASIC from **overlay 0 at `$8000`**. It **does not** write or erase flash.

If the SXB programmer micro-USB cable is connected to MIND:

1. Keep the UART connected to the Windows computer running Tera Term at 19200 8N1.
2. Press **RESET/S2** on the SXB.
3. On MIND run:

```bash
python3 /home/sysop/w65c816-basic/tools/launch_rom.py
```

4. Look for the BASIC banner in Tera Term.

Refer to these local MIND files for the exact verified launcher details and monitor commands:

```text
/home/sysop/w65c816-basic/README.md
/home/sysop/w65c816-basic/docs/MONITOR.md
/home/sysop/w65c816-basic/tools/launch_rom.py
```

The MIND project itself was **not provided here** for direct source review, so this section documents the launch procedure as reported, not a newly independently tested result. **MIND is not a desired permanent dependency of the Windows IDE**; its successful launch implementation is the reference needed to finish the Windows button correctly.

## 10. Using BASIC

Type commands in **UPPERCASE** for this BASIC build. Example direct commands:

```basic
PRINT "HELLO WORLD"
PRINT 2+2
PRINT 100*100
```

Example numbered program:

```basic
10 FOR I=1 TO 10
20 PRINT I,I*I
30 NEXT I
LIST
RUN
```

Use `NEW` to discard the current BASIC program (do not use it if you want to preserve unsaved work). `WAIT 1000` pauses approximately one second in the reported ROM build. `Ctrl+C` interrupts BASIC. **Ctrl+]** is reported as the escape back to the original monitor. These behaviors are **specific to the installed BASIC build**, not universal Microsoft BASIC commands. Test with a disposable program before relying on them.

A reset/power cycle should select default WDC monitor flash bank 3. BASIC program text stored only in volatile RAM will normally be lost at power-off unless a separate save facility is implemented.

## 11. Flashing versus launching

**START BASIC and `launch_rom.py` are launch operations, not flash updates.** Running them should not rewrite the flash ROM chip. The BASIC installation reportedly already exists in overlay 0.

The MIND project also contains flash-related scripts such as:

```text
/home/sysop/w65c816-basic/tools/program_flash.py
/home/sysop/w65c816-basic/tools/update_basic.py
```

**Do not run either just to start BASIC.** Flash modification can corrupt the monitor or other banks if address mapping, chip erase sectors, or the target image are wrong. Before any future flash update, make and verify a full flash backup, hash the image, check erase-sector boundaries, and preserve a tested restoration path.

## 12. Troubleshooting

| Symptom | Check / action |
| --- | --- |
| Tera Term is blank | Check separate UART COM port, 19200 8N1, TX/RX crossing, common ground, RTSB–CTSB jumper, and that BASIC was actually launched |
| Gibberish characters | Verify baud/8N1, UART voltage levels, and matching firmware UART initialization |
| Cannot type or duplicated characters | Local echo OFF; check TX/RX, Tera Term line endings, and input-handling in BASIC |
| Studio cannot connect | Check the **other** COM port (SXB programmer USB), reset S2, close any app holding that port |
| START BASIC reports GO but no `READY` | Current IDE's 13-byte trampoline may not match the working 59-byte launcher; use the reported MIND launcher for a control test, not `program_flash.py` |
| Power cycle enters WDC monitor, not BASIC | Expected: factory flash bank 3 is the default overlay |
| Build / publish fails | Open the first relevant file under `build-logs`; confirm .NET 10 SDK, complete ZIP extraction, and required assets |
| `ca65` reports syntax errors | Check ca65 syntax and the current source; build errors are separate from UART/ROM behavior |
| No reliable return from BASIC | Try the build-specific Ctrl+] escape; otherwise reset S2 to restore the monitor; save any volatile program first |

### Safety checklist

- [ ] SXB powered from its micro-USB; UART adapter **VCC disconnected**.
- [ ] Adapter serial signal levels verified for the 5 V board.
- [ ] SXB **GND ↔ adapter GND** confirmed.
- [ ] SXB **RXD ↔ adapter TXD**, SXB **TXD ↔ adapter RXD**.
- [ ] **J4 pin 3 ↔ J4 pin 9** if implementing the WDC RTS/CTS local loopback.
- [ ] No pins shorted to ground to change ROM banks.
- [ ] Tera Term on **UART COM** at **19200 8N1**, no flow control, no local echo.
- [ ] Studio on separate **monitor COM**.
- [ ] Flash write scripts **not used** for routine BASIC startup.

## 13. IDE examples and shortcuts

The latest Studio project includes examples in `assets/examples/` for the following:

| Example | File |
| --- | --- |
| EDU 8 LED chase | `EDU-LEDs.s` |
| EDU buzzer | `EDU-Buzzer.s` |
| EDU banked RAM check | `EDU-RAM.s` |
| EDU I²C scanner | `EDU-I2C-Scan.s` |
| EDU SPI RAM | `EDU-SPI-RAM.s` |
| EDU OLED Hello World | `EDU-OLED-Text.s` |
| EDU OLED 512 KB RAM test | `EDU-OLED-512K-RAM-Test.s` |
| EDU OLED RTC clock | `EDU-OLED-RTC-Clock.s` |
| EDU OLED Fahrenheit temperature | `EDU-OLED-Temperature-F.s` |
| EDU OLED primes up to 1 million | `EDU-OLED-Primes-1Million.s` |
| SXB LED sweep | `LED-Sweep.s` |

Examples that write RAM can destroy existing data. Example selection uses verified copies of the bundled examples; modified copies are backed up, rather than blindly overwriting personal ASM work. For personal projects use **Save As**.

Keyboard shortcuts in the supplied project: **F6 Build**, **F5 Build & Run**, **Ctrl+F Find**, **Ctrl+S Save**.

## 14. Reference documents

**Authoritative board and pin references** (consult the document for the exact board revision in hand):

- [WDC W65C816SXB datasheet (PDF)](https://www.westerndesigncenter.com/wdc/documentation/W65C816SXB.pdf) — particularly **page 8, J4 ACIA connector pinout** and page 9 connector description.
- [WDCMON v2.0 manual (PDF)](https://www.westerndesigncenter.com/wdc/documentation/WDCMON_User_Manual_v20.pdf) — particularly **page 7, flash overlay control**.
- [WDC W65C816SXB-Hacker UART connection guide](https://wdc65xx.com/w65c816sxb-hacker/) — explicitly describes TX, RX, GND, RTS/CTS jumper, and 19200 8N1 terminal configuration.
- [WDC W65C816EDU getting started](https://wdc65xx.com/getting-started/w65c816edu-getting-started/) — board stacking and EDU external RAM banking notes.
- [W65C816EDU schematic Rev D](https://www.wdc65xx.com/wdc/Schematics/Schematic_W65C816EDU_REVD.pdf) — EDU signals and UART breakout identification.

**Local project references:**

- Windows Studio: `README.md`, `Core/Monitor.cs`, `Studio/MainWindow.xaml.cs`, `BUILD.cmd`.
- MIND BASIC: `/home/sysop/w65c816-basic/README.md`, `docs/MONITOR.md`, `tools/launch_rom.py`.

---

**Summary:** For terminal operation, connect **ground, crossed TX/RX, and (if needed) RTSB↔CTSB**. Configure Tera Term at **19200 8N1**. The **installed BASIC is overlay bank 0 at `$8000`**, while the **WDC monitor is bank 3 on reset**. **You do not ground ROM bank pins or reflash anything to launch BASIC.** The Windows START BASIC button is a separate software implementation that still needs equivalence testing against the 59-byte launcher used by MIND.
