# MVT GUI Wrapper (Windows)

Simple desktop GUI wrapper for [Mobile Verification Toolkit (MVT)](https://docs.mvt.re/en/latest/) built with Python + Tkinter.

The goal is to let users run common MVT workflows without using terminal commands manually.

## Tech Stack

- Python 3.10+ (tested with modern Python on Windows)
- Tkinter (built into Python)
- `mvt` Python package

Why this stack:
- Fast to implement and easy to maintain
- No heavy frontend/runtime dependencies
- Easy local run on Windows with `pip` and `python`

## Features Implemented

- Platform selection:
  - iOS
  - Android
- Workflow selection:
  - iOS: `check-backup`, `check-fs`
  - Android: `check-backup`, `check-androidqf`, `check-adb`
- Input selection via GUI:
  - backup / filesystem folder
  - IOC file (optional)
  - output directory
- Background execution:
  - MVT runs in a worker thread (UI stays responsive)
- Runtime visibility:
  - command queue
  - currently running command
  - progress bar (best-effort from output parsing)
  - live logs/console output
  - success/error summary
  - stop/cancel running job
- Output handling:
  - timestamped result folder per run
  - button to open results folder in Explorer
  - parsed JSON report tab (table view across output JSON files)
  - export parsed report to CSV and HTML
- UX helpers:
  - remembers paths/platform/workflow if enabled
  - ANSI log formatting support
  - clickable links in logs with confirmation popup

## Not Implemented (Current Scope)

- Full coverage of every MVT command and advanced options
- Built-in packaging to `.exe` (can be added with PyInstaller)
- Exporting parsed report tables to PDF

## Requirements

- Windows 10/11
- Python installed and available in PATH
- MVT dependencies installed (via `requirements.txt`)

Install dependencies:

```bash
pip install -r requirements.txt
```

## Run on Windows

Option A:

```bash
python app.py
```

Option B (if you use the included batch script):

```bat
run.bat
```

## Project Structure

- `app.py` - main Tkinter GUI application
- `requirements.txt` - Python dependencies
- `run.bat` - convenience launcher
- `Sample/` - optional sample files/output folders for local testing

## Usage

1. Launch the app.
2. Select platform (`iOS` / `Android`).
3. Select workflow.
4. Choose required paths (input, optional IOC, output directory).
5. Click **Run MVT**.
6. Watch command/log/progress panels.
7. Open results folder when done.

## Known Limitations

- `check-adb` requires a connected Android device with ADB available.
- Progress is estimated from log output patterns; not every module reports percentages.
- Some MVT modules may report "no data to extract" depending on acquisition type/content.
- GUI behavior depends on MVT package behavior/version.
- Parsed report is schema-agnostic and best-effort (different MVT modules produce different JSON formats).

## Security / Forensics Note

MVT and this wrapper are triage/analysis tools and do not guarantee detection of advanced spyware by themselves.
If compromise is strongly suspected, consult professional incident response/forensic support.

## Next Steps (Suggested)

- Add per-workflow advanced options in GUI (module filters, verbose mode, etc.)
- Package as standalone Windows executable
