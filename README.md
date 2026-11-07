# W65C816 Studio

A high-contrast Windows assembly IDE and USB uploader for the **WDC W65C816SXB** and **W65C816SXB + EDU** setup. Write 65C816 assembly, assemble and link it with the bundled cc65 toolchain, upload to board RAM through the factory monitor, and start the program—all from one window.

> **Build status:** This is a **source-code distribution**, not a precompiled Windows application. The current project was inspected and packaged, but has **not been built on Windows or validated on physical hardware** in this release. Hardware features require the corresponding devices to be installed and connected.

## Contents

- [Quick start](#quick-start)
- [Requirements](#requirements)
- [Build from source](#build-from-source)
- [Connect your board](#connect-your-board)
- [Editor and controls](#editor-and-controls)
- [Examples](#examples)
- [Using and protecting example files](#using-and-protecting-example-files)
- [Memory and build outputs](#memory-and-build-outputs)
- [OLED and peripheral notes](#oled-and-peripheral-notes)
- [Troubleshooting](#troubleshooting)
- [Project layout](#project-layout)
- [Changes and limitations](#changes-and-limitations)

## Quick start

1. Extract the entire ZIP to a writable folder, such as `C:\W65C816-Studio`. **Keep the folder structure intact.**
2. Install the **.NET 10 SDK** for Windows, if it is not already installed.
3. Run `BUILD.cmd` from the extracted project root.
4. After a successful build, launch `publish\W65C816Studio.exe`. Keep its adjacent `tools`, `examples`, and other published assets alongside the executable.
5. Attach the W65C816 board over USB. In Studio, select the matching **board profile** and **COM port**. Use **Refresh** and **Check board** to confirm communication.
6. Select an example under **EXAMPLES**, click **Open example**, and review its requirements.
7. Press **F6** to build only, **Upload only** to build and transfer without running, or **F5** for **Build & Run**.

**If a previously uploaded program is still executing:** Press the board's **RESET/S2** button (or the EDU reset control, as applicable) to return to the monitor before attempting another upload.

## Requirements

### PC

- Windows x64, for the provided `BUILD.cmd` publish target.
- .NET 10 **SDK** for compiling the project. The publish command creates a self-contained Windows executable.
- Working USB connection and the appropriate board serial/USB driver so that Windows exposes a COM port.
- Permission to open that COM port; close other serial monitor applications using it.

### Hardware

- **W65C816SXB** with a compatible WDC factory monitor. The serial uploader is written for **factory ROM monitor 2.0.4.3**; other firmware revisions may behave differently.
- Optional **EDU** board and the components required by individual EDU examples.
- For OLED examples: **SSD1306 128×64 I²C OLED** at **7-bit address `$3C`**.
- For the Fahrenheit temperature example: populated **ADS1015 ADC** and **MCP9700A analog temperature sensor**, wired as the board example expects (sensor signal to ADC **AIN2**).

> Board profile selection changes example choices/reference text; it is not automatic hardware detection.

## Build from source

From the extracted project root, double-click:

```bat
BUILD.cmd
```

The script runs:

```bat
dotnet publish Studio/Studio.csproj -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true -o publish
```

Launch `publish\W65C816Studio.exe` when publishing succeeds. **Do not move only the `.exe`**: the `examples`, `tools`, and `EDU-GUIDE.html` published assets must remain available next to it.

To compile and run tests from a terminal (separately from the publishing script):

```bat
dotnet build Studio/Studio.csproj -c Release
dotnet run --project Tests/Studio.Tests.csproj
```

The `Tests` project exercises software behavior; it does **not** replace testing with the actual board and attached peripherals.

### Included development dependencies

| Component | Purpose |
| --- | --- |
| WPF / .NET 10 | Windows desktop application |
| AvalonEdit 6.3.1.120 | ASM source editor and search bar |
| Bundled `ca65.exe` | 65C816 assembler |
| Bundled `ld65.exe` | Linker |
| `assets/tools/board.cfg` | Linker memory map for the board |
| `Core/Monitor.cs` | USB serial transport and factory monitor protocol |

The cc65 distribution license is included in `assets/tools/cc65-LICENSE.txt`.

## Connect your board

1. Attach the board by USB and power it as required.
2. Choose **W65C816SXB** for SXB-only examples, or **W65C816SXB + EDU** for EDU peripheral examples.
3. Select its COM port in **USB PORT**. If it is missing, click **Refresh** and check Windows Device Manager.
4. Click **Check board**. You should see a monitor acknowledgement in **Build output**.
5. If no response arrives, reset the board and retry.

The built-in serial connection uses **115200 baud, 8 data bits, no parity, 1 stop bit (8N1), no flow control**. Studio talks to the board's binary factory monitor; it is **not** a general ASCII serial terminal.

## Editor and controls

The light, high-contrast interface uses black text, a white editor, line numbers, and 65C816 syntax highlighting.

| Action | Button or shortcut | What happens |
| --- | --- | --- |
| Build | **Build** / **F6** | Saves, assembles, and links; does **not** upload |
| Build & Run | **Build & Run** / **F5** | Saves, assembles, links, uploads, verifies, then sends GO |
| Upload only | **Upload only** | Builds, uploads, and verifies; leaves execution stopped |
| Find | **Find** / **Ctrl+F** | Opens search with typing focus in the search field |
| Save | **Save** / **Ctrl+S** | Saves the current source file |
| Save As | **Save as…** / **Ctrl+Shift+S** | Creates a separately named source file |
| Open | **Open source…** / **Ctrl+O** | Opens an existing `.s` or `.asm` file |
| New | **New source** / **Ctrl+N** | Starts a small 65C816 ASM skeleton |
| Check board | **Check board** | Probes the factory monitor |
| Read test results | **Read test results** | Reads bank 0 `$3000–$31FF` from the stopped monitor |
| Open output folder | **Open output folder** | Opens the last build folder (or workspace if not yet built) |
| EDU guide | **EDU guide** | Opens the bundled HTML board reference |

Bottom tabs: **Build output**, **Assembly listing**, **Binary / hex**, **EDU / board reference**, and **Board results**. Double-click an error line in Build output to jump to that location **when the error belongs to the currently open source file**.

### Build & Run workflow

The current code orders the operations as follows:

1. Save current source.
2. Assemble it with `ca65` for CPU `65816`.
3. Link it with `ld65` using `board.cfg`.
4. Transfer the produced binary to board application RAM at `$00:2000`.
5. Read back and compare the binary, then write/read the CPU context.
6. Send the monitor's GO command to start executing at `$00:2000`.

**A successful upload verification does not prove the assembled program behaves correctly.** If the board is already executing another program and won't accept monitor commands, reset first.

## Examples

Under **W65C816SXB + EDU**, Studio currently exposes **11 menu entries** (10 EDU examples and one SXB example). The **W65C816SXB** profile shows the SXB LED sweep.

| Menu entry | Bundled ASM | Description / important requirements |
| --- | --- | --- |
| EDU · 8 LED chase | `EDU-LEDs.s` | Runs the on-board 8-LED chase; **not** the OLED text demo |
| EDU · Buzzer beep | `EDU-Buzzer.s` | Exercises the EDU buzzer; enable its physical switch |
| EDU · Banked RAM check | `EDU-RAM.s` | Tests accessible external RAM; review source and data preservation before running |
| EDU · I²C bus scan | `EDU-I2C-Scan.s` | Scans the EDU I²C bus; view results through board-read functionality |
| EDU · SPI RAM check | `EDU-SPI-RAM.s` | Exercises SPI SRAM where populated |
| EDU · OLED text demo | `EDU-OLED-Text.s` | SSD1306 Hello World/text example |
| EDU · OLED 512K RAM test | `EDU-OLED-512K-RAM-Test.s` | OLED status while testing expansion RAM; **destructive** |
| EDU · OLED RTC clock | `EDU-OLED-RTC-Clock.s` | OLED clock; edit start-time constants before assembling |
| EDU · Large Fahrenheit temperature | `EDU-OLED-Temperature-F.s` | Displays temperature in °F from MCP9700A → ADS1015 AIN2 |
| EDU · OLED primes to 1 million | `EDU-OLED-Primes-1Million.s` | Computes/display primes 2–1,000,000, repeats continuously |
| SXB · X LED sweep | `LED-Sweep.s` | LED sweep for the SXB setup |

Some examples use `edu.inc`, `i2c.inc`, or `spi.inc`. Keep the `examples` folder and `.inc` files with the installation; the assembler includes that folder in its include search path.

### Prime-number example

Select **EDU · OLED primes to 1 million**. The assembly uses a **24-bit candidate counter** to reach 1,000,000 (a 16-bit counter cannot). It counts primes, shows the current prime/count/pass on the OLED, then restarts automatically. A full pass has **78,498 primes**, and the largest is **999,983**.

At the top of `EDU-OLED-Primes-1Million.s`, adjust:

```asm
DISPLAY_EVERY = 1
```

`1` requests an OLED update for every prime. Increasing the value (for example to `16`) reduces display traffic and may make the calculation faster. The actual speed depends on the CPU, display/I²C code, and other runtime costs. This particular ASM has not yet been confirmed by a successful build and board run.

### RTC clock example

Set the documented start-time constants near the beginning of `EDU-OLED-RTC-Clock.s`. The example initializes the on-board RTC when the program starts, so running it again can **reset the RTC to those configured values**. This is not an automatic PC-time sync.

### Temperature example

Requires the **ADS1015** and **MCP9700A** to be physically installed and wired correctly. It reads the sensor through **ADS1015 AIN2** over I²C and converts the result to Fahrenheit. If the ADC is missing, incorrectly addressed, or not communicating, troubleshoot the circuit and I²C bus before assuming the reading is valid.

### Destructive RAM test warning

**`EDU-OLED-512K-RAM-Test.s` overwrites all expansion RAM in banks `$08`–`$0F` as part of its test.** Do not execute it while any data in those banks must be preserved. Other RAM tests may also modify test locations: inspect their source and preserve important data before running.

## Using and protecting example files

To prevent an older `EDU-LEDs.s` from opening as the OLED demo, the IDE now uses a verified examples directory:

```text
%USERPROFILE%\Documents\W65C816 Studio\
  Studio Examples - Verified\
    EDU-LEDs.s
    EDU-OLED-Text.s
    EDU-OLED-Primes-1Million.s
    ...
  Example Backups\
    <timestamp>-<example>.s
```

On launch, and again when you click **Open example**, the application copies packaged examples into **Studio Examples - Verified**. When an existing verified example differs, Studio first backs it up to **Example Backups**, then replaces that verified file with the bundled one. **Legacy files directly under `Documents\W65C816 Studio` are left in place**; they are not deleted or automatically trusted as canonical examples.

**If you want to keep changes to an example, use Save As and save it under your own filename outside the managed verified examples folder.** Otherwise your edits can be backed up and replaced the next time examples refresh. The Documents folder may be redirected by OneDrive; use the actual location shown by Windows and Studio.

## Memory and build outputs

- **Application upload location:** bank 0, `$2000` through `$7DFF` (24,064 bytes maximum).
- **CPU entry point:** bank 0, `$2000`; monitor sets up the execution context before GO.
- **Board result read range:** bank 0, `$3000` through `$31FF` (512 bytes), read when the monitor is stopped. This region is **within application RAM**, not a reserved independent output buffer.
- **SXB reference registers shown by the IDE:** PIA `$7FA0`, VIA `$7FC0`, USB VIA `$7FE0`.
- **EDU expansion RAM for the 512K example:** banks `$08` through `$0F`.

Build products are placed beside the source in a hidden-like build folder named `.w65c816-build` (a normal folder whose name begins with a dot):

```text
<your source folder>\.w65c816-build\
  <program>.o
  <program>.lst
  <program>.bin
  <program>.map
  <program>.dbg
  <program>.lbl
```

The **Assembly listing** and **Binary / hex** tabs display data from the most recent successful build. Generated outputs do not need to be hand-edited.

## OLED and peripheral notes

The bundled OLED examples target **SSD1306, 128×64, I²C `$3C`**. For this EDU wiring, the code uses **VIA port A bit 0 for SCL** and **port A bit 7 for SDA** with software-driven I²C. Different OLED controller types, addresses, supply requirements, or wiring will need code/hardware changes.

EDU peripherals described by the built-in reference include the **MCP79411 RTC**, **ADS1015 ADC**, **MCP9700A temperature sensor**, **23LCV1024 serial SRAM**, and **ATECC608A security IC**. **Do not assume every device is factory-populated.** Some may be missing or require installation. Confirm the actual board's schematic and parts before using a relevant example.

## Troubleshooting

### Studio doesn't build

- Ensure the **.NET 10 SDK** is installed (`dotnet --info`).
- Run `BUILD.cmd` from the extracted root, not from the `Studio` subfolder.
- Keep project files, NuGet access/cache, and the bundled `assets` directory available.
- Inspect the full command output; the build is not considered successful until the publish step finishes.

### The `.exe` opens but examples/tools are missing

Keep the whole **`publish`** directory intact. The self-contained executable still uses external, copied assets. Rebuild from an unmodified extracted source tree if necessary.

### No COM port / Check board fails

- Confirm the board is powered and connected with a **data-capable** USB cable.
- Look for its COM port in Windows Device Manager, then click **Refresh**.
- Select the correct COM port; close any other program holding it open.
- Press **RESET/S2** (or applicable EDU reset control) to return to the factory monitor.
- Verify firmware compatibility with the monitor protocol used by Studio.

### Build works but Build & Run fails

A local assembly/link success does not establish USB connectivity. Read **Build output** for whether failure occurred during transfer, verification, context setup, or GO. Reset the board, check the COM port, then try **Check board** followed by **Upload only** to isolate the problem.

### An example opens the wrong code

Use the **Examples** selector and **Open example**, rather than reopening old root-level workspace files. The verified example directory holds the packaged source. For personal changes, use **Save As**.

### Find opens but I can't type

Use **Find** or **Ctrl+F**; the repaired search action leaves focus in the AvalonEdit search box. Make sure you are running a rebuild of this source, not an older executable.

### OLED is blank or garbled

Check OLED power, ground, SDA/SCL connections, the **SSD1306** controller type, the **128×64** format and **`$3C`** I²C address. First test **OLED text demo** before running the more involved examples.

### Assembler syntax error

The included assembler is **ca65**, not a generic 65xx/65816 assembler. Some immediate character expressions accepted elsewhere are not accepted in this project's syntax. For example, use `lda #$30` for ASCII `0` if `lda #"0"` triggers an expression error. Double-click a relevant source error line in the output panel to jump to it.

### Memory test changed my data

RAM tests can overwrite the regions they probe. **The 512K OLED RAM test intentionally destroys the previous contents of banks `$08`–`$0F`.** Recover from your own saved data or backup; the IDE cannot undo hardware memory writes.

## Project layout

```text
W65C816-Studio-High-Contrast/
├── README.md                 This documentation
├── READ_ME_FIRST.txt         Historical brief setup notes
├── BUILD.cmd                 Publish Windows executable
├── Studio/
│   ├── Studio.csproj         .NET 10 WPF application
│   ├── MainWindow.xaml       Main UI
│   └── MainWindow.xaml.cs    Editor, examples, build/upload actions
├── Core/
│   ├── Studio.Core.csproj
│   └── Monitor.cs            Serial/monitor protocol
├── Tests/
│   ├── Studio.Tests.csproj
│   └── Program.cs
├── assets/
│   ├── EDU-GUIDE.html
│   ├── README.txt
│   ├── tools/                 ca65, ld65, board.cfg and license
│   └── examples/              ASM samples and include files
└── EXAMPLE-AUDIT.md          Previous structural example audit
```

## Changes and limitations

The source bundle includes these recent improvements:

- High-contrast **light interface with black text**, clearer controls, and larger editor text.
- **F5 Build & Run** uses separate ordered build, upload/verification, and GO phases.
- **Find/Ctrl+F** keeps focus in the search input.
- **Verified example selection** avoids the previous 8-LED-chase/OLED-demo file confusion.
- Added OLED text, 512K RAM test, RTC clock, Fahrenheit temperature display, and prime generator examples.

**Current limits:** The hardware uploader assumes a particular factory monitor protocol, board examples assume particular connections/components, and this source release has not been confirmed by compiling/running the entire application on Windows or by executing all samples on real hardware. Treat the test and example-audit notes as inspection evidence, not a blanket certification that every ASM example works on every EDU configuration.

---

**Tip:** After confirming the basic LED chase and OLED text demo work, use **Save As** to create your own project source. This avoids losing edits when the managed examples refresh.
