# CASIO ClassWiz fx-580VN X — Standalone Desktop Emulator

[![Platform: Windows](https://img.shields.io/badge/Platform-Windows%20(x86%20%2F%20x64)-blue.svg)](https://microsoft.com/windows)
[![Reconstructed with: REA](https://img.shields.io/badge/Reverse--Engineered%20with-morluto%2Frea-8A2BE2.svg)](https://github.com/morluto/rea)
[![Firmware: CY-298A](https://img.shields.io/badge/Firmware-CY--298A%20Ver.2.01-green.svg)](#firmware--hardware-architecture)
[![Format: Portable EXE](https://img.shields.io/badge/Delivery-Single--File%20Standalone%20(3.9%20MB)-orange.svg)](#key-features)
[![License: MIT / Educational](https://img.shields.io/badge/License-MIT%20%2F%20Educational-lightgrey.svg)](LICENSE)

A standalone, portable, pixel-perfect Windows desktop emulator for the **CASIO ClassWiz fx-580VN X** scientific calculator, reverse engineered and reconstructed using the [**REA** (Reverse Engineer Anything)](https://github.com/morluto/rea) toolkit.

Runs the authentic 256 KB **CY-298A** firmware on top of the native LAPIS/ROHM nX-U8/100 16-bit RISC core simulator (`SimU8.dll`), completely decoupled from subscription/activation checks (`ActivationFx.dll`). Packaged as a **single, zero-install 3.9 MB `.exe`** with high-DPI rendering, a clean white UI, full PC keyboard support, and a 1-click desktop shortcut creator.

---

## Table of Contents

- [Key Features](#key-features)
- [How It Was Reconstructed Using REA](#how-it-was-reconstructed-using-rea)
  - [1. De-packaging the InstallShield Archive](#1-de-packaging-the-installshield-archive)
  - [2. Static PE & Module Decomposition](#2-static-pe--module-decomposition)
  - [3. Firmware Extraction via x86 Emulation](#3-firmware-extraction-via-x86-emulation)
  - [4. Hardware Interrupt & I/O Table Mapping](#4-hardware-interrupt--io-table-mapping)
  - [5. Paravirtualized Display & Keypad Bus](#5-paravirtualized-display--keypad-bus)
- [Hardware & Memory Architecture](#hardware--memory-architecture)
- [User Guide & Shortcuts](#user-guide--shortcuts)
  - [Window Controls](#window-controls)
  - [PC Keyboard Bindings](#pc-keyboard-bindings)
  - [Context Menu & Quick Actions](#context-menu--quick-actions)
- [Building from Source](#building-from-source)
- [Disclaimer & License](#disclaimer--license)

---

## Key Features

- **100% Standalone Single Binary**: Only one file (`CasioFx580VNX.exe`, ~3.9 MB). All firmware images, hardware interrupt vector tables, high-resolution textures, icons, and native simulator libraries are embedded and automatically unpacked into memory/temp at runtime. No installer, no administrator privileges required.
- **Authentic Firmware Simulation**: Runs the genuine **CY-298A** firmware executing all scientific, algebraic, statistical, calculus, matrix, vector, and polynomial equation calculations identical to the real hardware.
- **Ultra-Sharp High-DPI UI**:
  - Full Windows Per-Monitor DPI awareness via `SetProcessDPIAware()`.
  - 4× super-sampled dot-matrix LCD renderer (`768×252` rendering surface for the `192×63` physical LCD).
  - High-resolution 3× status annunciator bar (`576×24`) with all 20 hardware status icons (`[S]`, `[A]`, `M`, `STO`, `D`, `Math`, etc.).
  - Unsharp-masked physical faceplate with smooth anti-aliased rounded borders.
- **Clean White Aesthetics**: Pure white background (`#FFFFFF`) with a customizable LCD palette (switchable between Modern Paper White `#F8FAFC` and Classic Casio Green `#DAE6DA` with a single click).
- **1-Click Desktop Shortcut**: Create a desktop shortcut directly from the right-click menu or by launching with `--create-shortcut`.
- **Vietnamese & English Modes**: One-click `VN` button on the titlebar automates the SETUP sequence to switch the system language to Vietnamese (`1:Phép tính thường`).
- **Complete Offline Privacy**: Bypasses the cloud subscription daemon (`ActivationFx.dll`) — zero network telemetry, zero periodic license checks.

---

## How It Was Reconstructed Using REA

This project was analyzed, extracted, and rebuilt using [**REA (`morluto/rea`)**](https://github.com/morluto/rea), an advanced reverse engineering MCP and static analysis framework.

### 1. De-packaging the InstallShield Archive

The original subscription distribution executable (`ClassWiz Emulator Subscription for fx-580VN X Ver.2.01_c.exe`) was inspected using REA's binary layout tools. Analysis revealed an appended `ISSetupStream` v4 payload located at file offset `1056256`:

- **Archive Layout**: 49 compressed streams containing `.msi`, `.mst`, and configuration files.
- **Decryption Cipher**: Reverse engineering of the routine at virtual address `0x40f478` in `.text` showed a byte transformation algorithm:
  $$\text{Decoded}[i] = \sim\left(\left(\left(\text{Raw}[i] \gg 4\right) \mid \left(\left(\text{Raw}[i] \ \& \ 0x0F\right) \ll 4\right)\right) \oplus \text{Key}[i \pmod L]\right)$$
  where $\text{Key}$ is derived by XOR-ing the ASCII filename with the seed `\x13\x35\x86\x07`.
- **Decompression**: Streams with flag `0x04` were decompressed using standard zlib inflate to yield the clean administrative `.msi` package.

### 2. Static PE & Module Decomposition

Using REA's `PeImportsProvider`, `PeResourcesProvider`, and `NativeExportParser`, the core components were mapped:

| Binary | Subsystem | REA Inspection Findings | Role in Reconstructed App |
| :--- | :--- | :--- | :--- |
| `fx-580VN X Emulator.exe` | Win32 GUI | 54 PE resources (16 bitmaps `ID_130..ID_156`), 14 imported DLLs | Contains the compiled ROM loading tables and LCD drawing logic |
| `CLASSWIZ_P20.dll` | Win32 DLL | 17 exports (`GetFaceSize`, `GetDisplayPosition`, `GetKeyInfo`, `GetKeyCount`) | Provided physical faceplate bitmap (`ID_103`, `644×1250`) and the 50-key matrix at `0x10004010` |
| `SimU8.dll` | Win32 DLL | 23 `__stdcall` exports controlling memory banking, interrupts, and engine scheduling | Core execution engine wrapper |
| `SimU8engine.dll` | Win32 DLL | 46 exported C++ methods of `CSimU8core` simulating the 16-bit LAPIS nX-U8/100 RISC core | Direct CPU opcode decoding and execution |
| `ActivationFx.dll` | Win32 DLL | 15 exports handling web activation licenses | **Bypassed completely** — decoupled from execution loop |

### 3. Firmware Extraction via x86 Emulation

Static disassembly of `fx-580VN X Emulator.exe` showed that the calculator firmware was not stored as a plain file on disk, but was synthesized dynamically into code memory across 10 initialization procedures (`0x401e90` through `0x4a81a0`).

A lightweight x86 execution tracer was written to emulate the 10 routines:
- Parsed 14,515 invocations of `WriteCodeMemory(addr, size, buffer)`.
- Reconstructed the complete, contiguous **256 KB ROM image (`CY-298A`, 262,144 bytes)** with correct interrupt vectors at `0x00000` and international/Vietnamese localized string tables.

### 4. Hardware Interrupt & I/O Table Mapping

Tracing `SetInterruptTable` at VA `0x4bb960` revealed the exact configuration required by `SimU8.dll` for the LAPIS micro-controller:
- 18 hardware interrupt descriptors (40 bytes each, 720 bytes total).
- Associated the timer interrupt `TM0INT` (Vector `0x12`, SFR `0x0F014` bit 5) for real-time clock ticking and cursor blinking.
- Associated the keyboard matrix interrupt `XI0INT` (Vector `0x0a`, SFR `0x0F014` bit 1) for tactile keypad input.

### 5. Paravirtualized Display & Keypad Bus

Reverse engineering the event callback at `0x4bc670` revealed the hardware handshake between the emulator and the nX-U8 core:
- **`0x48E00` (Command Register)**:
  - `1`: Firmware requests keypad input (`KI` from `0x48E01`, `KO` from `0x48E02`) and signals display refresh.
  - `2` / `8`: Break / AC key query.
  - `3`: Power-down state (`[SHIFT] [AC]` OFF).
  - `4`: Unconditional LCD repaint.
  - `5` / `7`: QR code URL generation buffer ready at `0x4A800`.
- **`0x0F800 – 0x0FFFF` (LCD Video RAM)**:
  - Row 0 (32 bytes): 20 annunciator status flags (`0x0F800` to `0x0F816`).
  - Rows 1–63 (32 bytes stride, 24 bytes active): `192 × 63` monochrome dot-matrix pixels.

---

## Hardware & Memory Architecture

```
0x00000 +------------------------------------------+
        | Code Memory: 256 KB CY-298A Firmware    |
        | (Mapped via SimU8::WriteCodeMemory)      |
0x3FFFF +------------------------------------------+

0x00000 +------------------------------------------+
        | ROM Window: 52 KB (SetRomWindowSize)    |
0x0CFFF +------------------------------------------+

0x0D000 +------------------------------------------+
        | Data Memory / RAM & Work Registers       |
        |                                          |
0x0F000 | Special Function Registers (SFRs)        |
        |   - 0x0F010..0x0F012 : Interrupt Enable  |
        |   - 0x0F014..0x0F016 : Interrupt Request |
        |                                          |
0x0F800 | LCD VRAM: 2048 Bytes                     |
        |   - Row 0: 20 Status Symbols             |
        |   - Rows 1..63: 192x63 Pixel Matrix      |
        |                                          |
0x48E00 | Paravirtualized I/O Communication Bus    |
        |   - 0x48E00 : Command Dispatch           |
        |   - 0x48E01 : Keypad Input Strobe (KI)   |
        |   - 0x48E02 : Keypad Output Line (KO)    |
        |                                          |
0x4A800 | QR Code String Buffer (4096 Bytes)       |
0x8FFFF +------------------------------------------+
```

---

## User Guide & Shortcuts

### Window Controls

The compact header bar at the top of the calculator provides quick toggles:
- **`VN`**: Automatically switches calculator language to Vietnamese.
- **`LCD`**: Toggles LCD background between *Paper White* (`#F8FAFC`) and *Casio Green* (`#DAE6DA`).
- **`68%`**: Cycles window scaling (`56%` → `68%` → `80%` → `92%` → `100%`).
- **`Ghim`**: Toggles Always-on-Top mode.
- **`—`**: Minimizes the window.
- **`×`**: Exits the application cleanly (safely stopping CPU simulation).
- **Draggable Header**: Click and drag anywhere on the title bar or faceplate margin to move the window.

### PC Keyboard Bindings

You can use either your mouse on the physical faceplate buttons or your computer keyboard:

| Physical Key | PC Keyboard | Function |
| :---: | :---: | :--- |
| `0` .. `9` | `0` .. `9` (or NumPad) | Number digits |
| `+`, `-`, `×`, `÷` | `+`, `-`, `*`, `/` | Basic arithmetic operators |
| `=` | `Enter` or `=` | Calculate result |
| `AC` | `Esc` | All Clear |
| `DEL` | `Backspace` or `Delete` | Delete character |
| `.` | `.` (Period or NumPad Decimal) | Decimal point |
| `(` and `)` | `(` and `)` | Parentheses |
| `^` | `^` | Power $x^y$ |
| `SHIFT` | `S` | Shift mode |
| `ALPHA` | `A` | Alpha mode |
| `MENU` | `M` | Function / Setup Menu |
| `OPTN` | `O` | Options menu |
| `CALC` | `C` | Calculate / Solve |
| `x` | `X` | Variable $x$ |
| `↑` `↓` `←` `→` | Arrow Keys | Directional navigation |
| `Ctrl + C` | `Ctrl + C` | Copy current LCD screen to Windows Clipboard |
| `Ctrl + 1..5`| `Ctrl + 1..5` | Quick-switch scale (56%, 68%, 80%, 92%, 100%) |

### Context Menu & Quick Actions

Right-click anywhere on the calculator body to access:
- **Kích thước**: Choose from 5 preset window zoom levels.
- **Đổi nền màn hình LCD**: Toggle between White and Green display.
- **Luôn nổi trên cùng**: Toggle Always-on-Top.
- **Chuyển ngôn ngữ sang Tiếng Việt**: Run the automated Vietnamese localization sequence.
- **Tạo lối tắt ra màn hình Desktop**: Instantly place a `CASIO fx-580VN X` shortcut on your Windows Desktop.
- **Sao chép ảnh màn hình LCD**: Export a high-resolution snapshot of the current LCD display to the clipboard.
- **Reset máy [ON]**: Perform a hardware cold restart.

---

## Building from Source

The project can be built using Microsoft's native C# compiler (`csc.exe`) bundled with the .NET Framework 4.0 or later (installed on every Windows 8, 10, and 11 system out of the box).

```bat
:: Compile standalone single-file binary with all embedded assets
C:\Windows\Microsoft.NET\Framework\v4.0.30319\csc.exe /nologo /target:winexe /platform:x86 ^
  /win32icon:build_assets\casio.ico ^
  /resource:build_assets\fx580vnx_rom.bin,fx580vnx_rom.bin ^
  /resource:build_assets\interrupt_table.bin,interrupt_table.bin ^
  /resource:build_assets\faceplate.png,faceplate.png ^
  /resource:build_assets\symbols_hires.png,symbols_hires.png ^
  /resource:build_assets\SimU8.dll.deflate,SimU8.dll.deflate ^
  /resource:build_assets\SimU8engine.dll.deflate,SimU8engine.dll.deflate ^
  /resource:build_assets\mfc100.dll.deflate,mfc100.dll.deflate ^
  /resource:build_assets\msvcr100.dll.deflate,msvcr100.dll.deflate ^
  /resource:build_assets\msvcp100.dll.deflate,msvcp100.dll.deflate ^
  /out:CasioFx580VNX.exe ^
  CasioFx580Desktop.cs
```

### Self-Test & Diagnostic Mode

To verify the embedded hardware simulation headless in CI or terminal:

```cmd
CasioFx580VNX.exe --self-test
```

Exit code `0` confirms the ROM was loaded, execution cycles advanced, and arithmetic computation ($7 \times 8 = 56$) rendered correctly into the LCD display buffer.

---

## Disclaimer & License

- This software was developed for **interoperability, educational, and digital preservation** purposes through clean-room static reverse engineering using [**REA**](https://github.com/morluto/rea).
- **CASIO** and **ClassWiz** are registered trademarks of CASIO Computer Co., Ltd.
- The emulator wrapper source code is licensed under the [MIT License](LICENSE).
