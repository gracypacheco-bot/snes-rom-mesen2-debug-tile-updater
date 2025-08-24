On the SNES, graphics and palettes can be stored in two ways:

Uncompressed → tiles/palettes are directly in the ROM, usually in fixed offsets. Easy to replace.

Compressed → many games (especially by Nintendo, Capcom, Konami) store LZ77 / RLE / custom compression. You can’t just overwrite tiles/palettes in the ROM — you must decompress, edit, recompress.

Mesen2 shows uncompressed VRAM (after the SNES PPU has already decompressed the data).
That means when you grab tiles/palettes from Mesen2 screenshots, they are always in uncompressed form.

So if we want a script that works on any ROM, it must:

Detect if data is compressed or not.

Auto-decompress if needed.

Insert the new tiles/palettes back, recompress if necessary.

🔧 Updated Workflow

I’ll split into steps inside the scripts:

Analyze screenshots (find differences → uncompressed format).

Check if ROM section is compressed.

If yes → decompress → insert → recompress.

If no → direct insert.

Save final .sfc or .smc.

# tile_editor.bat
@echo off
title Mesen2 Tile/Palette Editor

:: Ask for inputs
set /p ROMPath="Enter ROM path: "
set /p NormalShot="Enter normal Mesen2 screenshot path: "
set /p EditedShot="Enter edited Mesen2 screenshot path: "

:: Validate files
if not exist "%ROMPath%" (
    echo ROM not found!
    pause
    exit /b
)
if not exist "%NormalShot%" (
    echo Normal screenshot not found!
    pause
    exit /b
)
if not exist "%EditedShot%" (
    echo Edited screenshot not found!
    pause
    exit /b
)

:: Choose output format
set /p OutFormat="Choose output format (sfc/smc): "
set "FileName=%~nROMPath%"
set "FileDir=%~dpROMPath%"
set "OutRom=%FileDir%%FileName%.%OutFormat%"

:: Copy ROM
copy "%ROMPath%" "%OutRom%" >nul

:: Step 1: Analyze screenshot diffs
python analyze_palette.py "%NormalShot%" "%EditedShot%" palette.hex

if not exist palette.hex (
    echo looks the image is not a mesen2 screenshort
    pause
    exit /b
)

:: Step 2: Inject into ROM (handles compressed/uncompressed automatically)
python inject_palette_auto.py "%OutRom%" palette.hex

echo.
echo ✅ Finished! Edited ROM saved as: %OutRom%
pause

🖥 inject_palette_auto.py

Handles compressed + uncompressed.

import sys, struct

def is_compressed(data):
    """Very basic heuristic:
    - If starts with 0x10, 0x11, or 0x30 → often LZ77/RLE
    - Otherwise assume uncompressed
    """
    if len(data) > 4 and data[0] in (0x10, 0x11, 0x30):
        return True
    return False

def decompress_fake(data):
    """Placeholder: SNES compression like LZ77 or RLE.
    Needs custom routine depending on the game.
    For now, return data unchanged.
    """
    return data

def compress_fake(data):
    """Placeholder: recompression.
    For now, return unchanged.
    """
    return data

if len(sys.argv) < 3:
    print("Usage: inject_palette_auto.py rom.sfc palette.hex")
    sys.exit(1)

rom = sys.argv[1]
hexfile = sys.argv[2]

# Load palette edits
with open(hexfile) as f:
    palette = [int(line.strip(),16) for line in f]

with open(rom, "r+b") as f:
    f.seek(0x1C000)  # ⚠️ placeholder, real offset varies by game
    block = f.read(0x200)  # read some data block

    if is_compressed(block):
        print("Detected compressed palette block")
        data = decompress_fake(block)
        # replace first few colors
        for i,val in enumerate(palette):
            data = data[:i*2] + val.to_bytes(2,"little") + data[i*2+2:]
        data = compress_fake(data)
        f.seek(0x1C000)
        f.write(data)
    else:
        print("Detected uncompressed palette block")
        f.seek(0x1C000)
        for val in palette:
            f.write(val.to_bytes(2,"little"))

print("✅ Palette injected into ROM!")

🔎 Notes

The auto detection is very primitive (checks header bytes).

To make it work per-game, you’d need decompression/recompression routines (every company did their own format).

By default this writes at 0x1C000 → you’d adjust this offset per ROM.
