# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**PlemolKR** is a Korean programming font that merges IBM Plex Mono (English monospace) with IBM Plex Sans KR (Korean) to create a comprehensive programming font. This is a fork of the PlemolJP project, adapted for Korean language support.

**Repository**: soomtong/PlemolKR
**Current Branch**: kr
**Version**: v3.0.0
**License**: SIL Open Font License 1.1 (fonts), MIT License (build scripts)

### Font Composition
- **English/ASCII**: IBM Plex Mono (monospace, primary programming glyphs)
- **Korean**: IBM Plex Sans KR (Korean characters - primary target)
- **Japanese**: IBM Plex Sans JP (legacy support, may be removed)
- **Additional Glyphs**: Hack font (supplementary symbols)
- **Optional Icons**: Nerd Fonts (Powerline symbols, devicons)

### Migration Status: PlemolJP → PlemolKR
The codebase is in transition from PlemolJP to PlemolKR:
- `build.ini` defines both `FONT_NAME = PlemolJP` (legacy) and `NEW_FONT_NAME = PlemolKR`
- `fonttools_script.py` uses `NEW_FONT_NAME` to generate PlemolKR output files
- Some internal references still use PlemolJP naming - this is intentional for build compatibility
- **Original scripts contain Japanese comments** (inherited from PlemolJP upstream)

## Architecture

### Two-Stage Build Process

#### Stage 1: FontForge Script (fontforge_script.py)
Handles font merging and glyph manipulation:
1. Opens source fonts from `/source` directory
2. Unlinking font references for direct glyph manipulation
3. Adjusts EM squares (880 ascent + 120 descent = 1000 total)
4. Merges Hack font for supplementary glyphs
5. Deletes duplicate glyphs (prioritizes appropriate font for each range)
6. Applies custom glyph adjustments (punctuation, brackets, etc.)
7. Handles width transformations (1:2 or 3:5 ratios)
8. Generates italic variants via skew transformation (9 degrees)
9. Optionally adds Nerd Fonts glyphs
10. Outputs intermediate TTF files with `fontforge_` prefix

#### Stage 2: FontTools Script (fonttools_script.py)
Post-processes and finalizes fonts:
1. Applies ttfautohint with style-specific control files
2. Removes vhea/vmtx tables from Korean/Japanese font
3. Merges hinted English and Korean/Japanese portions
4. Extracts and modifies font tables (OS/2, post, name)
5. Fixes metadata for proper font recognition
6. Outputs final TTF files (removes temporary prefixes)

## Build System

### Prerequisites
- Python 3.x
- FontForge (with Python bindings)
- Python packages: fontTools, ttfautohint
- Task (taskfile.dev) - optional but recommended

### Common Commands

```bash
# Show available tasks
task

# Quick builds (debug mode - Regular weight only, fastest iteration)
task quick              # 1:2 ratio, Regular only + polish
task quick:35           # 3:5 ratio, Regular only
task quick:nerd         # Nerd Fonts, Regular+Bold only

# Standard builds (stage 1: fontforge only)
task build              # Default 1:2 ratio, all weights
task build:35           # 3:5 ratio, all weights
task build:console      # Console mode, all weights
task build:console35    # Console + 3:5 ratio
task build:nf           # 1:2 + Nerd Fonts
task build:nf35         # 3:5 + Nerd Fonts
task build:console-nf   # Console + Nerd Fonts
task build:console-nf35 # Console + 3:5 + Nerd Fonts

# Post-processing (stage 2: fonttools) - must run after stage 1
task polish             # Process all built fonts
task polish:variant VARIANT=35Console  # Process specific variant

# Complete workflows (stage 1 + stage 2)
task full               # Build + polish: default + 3:5
task full:all           # Build + polish: all variants
task full:nerd          # Build + polish: Nerd Fonts variants

# Nerd Fonts patching (using external FontPatcher, more complete coverage)
task patch:nerd         # Patch PlemolKRConsole fonts
task patch:nerd:wide    # Patch PlemolKR35Console fonts
task patch:nerd:all     # Patch all Console variants

# Verification and utilities
task clean              # Remove build directory
task check              # List generated fonts and glyph count
task verify             # Check Korean/Japanese character presence
task verify:nerd        # Check Nerd Fonts icon presence
task install            # Install fonts to ~/Library/Fonts/ (macOS)
task info               # Show build environment info
```

**Direct script usage:**
```bash
# fontforge_script.py flags
python fontforge_script.py --debug        # Only Regular weight (fast)
python fontforge_script.py --minimal      # Only Regular + Bold
python fontforge_script.py --35           # 3:5 width ratio
python fontforge_script.py --console      # Console mode
python fontforge_script.py --nerd-font    # Include Nerd Fonts
python fontforge_script.py --hidden-zenkaku-space  # Hide full-width space
python fontforge_script.py --do-not-delete-build-dir  # Preserve existing builds

# Combine flags
python fontforge_script.py --35 --console --nerd-font --do-not-delete-build-dir

# fonttools_script.py (optional variant filter as first arg)
python fonttools_script.py                # Process all fonts in build/
python fonttools_script.py 35Console      # Process only 35Console variant
```

### Configuration (build.ini)

Critical settings:
```ini
FONT_NAME = PlemolJP      # Legacy internal name (DO NOT change)
NEW_FONT_NAME = PlemolKR  # Output font name
EM_ASCENT = 880           # EM = 1000 total (880 + 120)
EM_DESCENT = 120
OS2_ASCENT = 950          # Vertical metrics for rendering
OS2_DESCENT = 225
HALF_WIDTH_12 = 528       # Half-width for 1:2 ratio (528:1056)
FULL_WIDTH_35 = 1000      # Full-width for 3:5 ratio (600:1000)
ITALIC_ANGLE = 9
```

### Build Options

| Option | Description |
|--------|-------------|
| Default (1:2) | Half-width is exactly 1/2 of full-width, compact |
| `--35` (3:5) | Half-width is 3/5 of full-width, larger ASCII characters |
| `--console` | Prioritizes IBM Plex Mono glyphs, half-width symbols, EA Ambiguous → half-width |
| `--nerd-font` | Adds Powerline symbols, devicons (built-in method) |
| `--hidden-zenkaku-space` | Disables full-width space (U+3000) visualization |

### Font Families Generated

| Family | Description |
|--------|-------------|
| **PlemolKR** | Standard 1:2 width ratio (528:1056) |
| **PlemolKRConsole** | Console-optimized, half-width symbols |
| **PlemolKR35** | 3:5 width ratio (600:1000), larger ASCII |
| **PlemolKR35Console** | 3:5 width + console mode |

Optional suffixes: **NF** (Nerd Fonts), **HS** (Hidden full-width Space)

Each family: 8 weights (Thin–Bold) × 2 styles (Normal, Italic) = 16 files

## Font Development Details

### Glyph Handling Strategy

**Duplicate Resolution:**
- U+00A2, U+00A3, U+00A5 (currency): Use IBM Plex Sans (Korean/Japanese)
- U+00C0-U+0259 (Latin Extended): Use IBM Plex Mono
- U+3000 (full-width space): Custom visualization or IBM Plex Sans
- U+274C (cross mark): Deleted (fallback to system emoji)

**Custom Adjustments:**
- Quotation marks: Enlarged and repositioned
- Punctuation (;:,.) : Scaled up 8%
- 'r' glyph: Custom adjustment via .sfd file (non-italic only, see `note.md` for details)
- Full-width brackets: Widened opening by ±180 units
- Full-width period/comma: Scaled up 40-45%
- Quotation marks (U+2018-201E): Scaled 25%, full-width
- Arrow symbols: Enlarged for better visibility

**Width Normalization:**
- Glyphs < 500 width → 600 (temporary, becomes half-width later)
- Glyphs 500-1000 or Latin U+00C0-U+0192 → 1000 (full-width)
- Final 1:2 ratio: 528 (half) : 1056 (full)
- Final 3:5 ratio: 600 (half) : 1000 (full)

### Italic Generation
- Korean/Japanese fonts don't have italic styles natively
- Generated algorithmically via skew transformation (9° angle, -40 unit horizontal offset)
- English italics use native IBM Plex Mono Italic glyphs

### Hinting
- Uses ttfautohint with style-specific control files in `/hinting_post_process/`
- Parameters: `-l 6 -r 45 -D latn -f none -S -W -X 13-`
- Italics use default hinting (no control file)

### Font Table Modifications

**OS/2 Table:** `xAvgCharWidth` → half-width value, `fsSelection` → style bits, `panose` → monospace/proportional
**post Table:** `isFixedPitch` → 1 for 1:2, 0 for 3:5
**name Table:** Cleaned copyright, family pattern: `{FONT_NAME} {variant} {weight}`

### Platform Compatibility Notes
- **macOS**: Removes horizontal baseline table to fix glyph clipping in terminals
- **VSCode Terminal**: Vertical metrics (950/225/0) for bottom-row visibility
- **Eclipse Pleiades**: Special handling for half-width space symbol (U+1D1C)

## Code Style & Conventions

**FontForge scripting patterns:**
- Glyph selection: `font.selection.select(("unicode", None), 0xXXXX)`
- Always clear selections: `font.selection.none()`
- Reopen fonts after altuni manipulation to fix encoding issues
- Width transformations preserve ligature multiples
- Use uuid for temporary file naming to avoid conflicts

**Important conventions:**
- FONT_NAME in build.ini must remain "PlemolJP" for internal compatibility
- Use NEW_FONT_NAME for actual output file names
- Intermediate files use prefixes: `fontforge_*` and `fonttools_*`
- Final output removes prefixes

## Common Pitfalls

- Don't change FONT_NAME in build.ini (breaks internal logic)
- Don't forget `--do-not-delete-build-dir` when building multiple variants sequentially
- Stage 2 (fonttools) must run after Stage 1 (fontforge)
- Italic variants are generated algorithmically (9° skew), not from source files
- Full build of all weights is slow - always use `--debug` or `--minimal` for iteration

## Key References

- `note.md` - Manual glyph adjustment notes (r glyph coordinates)
- `source/AdjustedGlyphs/` - Custom glyph modifications (.sfd files)
- `hinting_post_process/` - ttfautohint control files (`normal-{Weight}-ctrl.txt`, `35-{Weight}-ctrl.txt`)
- `FontPatcher/font-patcher` - External Nerd Fonts patcher (more complete than built-in)
- Upstream: [PlemolJP](https://github.com/yuru7/PlemolJP), [IBM Plex](https://github.com/IBM/plex), [Hack](https://github.com/source-foundry/Hack)
