# Skills Learned - KiCad PCB Project (Last 24 Hours)

## Project Overview
- Repository: `siliconvalleyar-oss/esp32_xiazhi_pcb`
- Branch: `ESP32C6`
- Project: ESP32-C6 Xiaozhi PCB Design

---

## 1. Git & Version Control
- **Navigate to project directory**: `cd esp32_c6_xiaozhi`
- **Check status**: `git status`
- **Stage all changes**: `git add -A` or `git add <file>`
- **Commit with message**: `git commit -m "message"`
- **Push to remote**: `git push`
- **Clean up untracked files**: Remove lock files (`*.lck`), backup directories, history directories

---

## 2. KiCad PCB File Manipulation (esp32_xiaozhi.kicad_pcb)

### Via Management
- **Remove all vias**: Search for `(via` patterns and delete all 17 vias from the PCB
- **Verify removal**: `grep -n "via" esp32_xiaozhi.kicad_pcb`

### Footprint Text Properties (Reference & Value)
- **Target format** (matching D8/SW6/Q13):
  - Font: Arial
  - Size: 0.7mm x 0.7mm
  - Thickness: 0.1mm
- **Apply to all 166 footprint References**: Used Python regex to find and replace font blocks in Reference property effects
- **Apply to all footprint Values**: Same font settings applied to Value properties
- **Verification**: Grep for `face "Arial"`, `size 0.7 0.7`, `thickness 0.1`

### File Structure Understanding
- PCB file is S-expression format (KiCad 10.0)
- Footprints contain properties: Reference, Value, Datasheet
- Each property has effects/font block for text rendering
- Render cache polygons follow text properties

---

## 3. KiCad Schematic File Manipulation (ft232.kicad_sch)

### Net Name Changes
- **Global find/replace** in schematic:
  - `+3V3` → `+3.3VA` (18 occurrences)
  - `GND` → `GND1` (33 occurrences)
- **Scope**: Only ft232.kicad_sch (not project-wide)
- **Verification**: `grep` before/after to confirm counts

---

## 4. PCB Design Rules (esp32_xiaozhi.kicad_pro)

### Board Setup → Design Rules
- **Minimum track width**: 0.382mm (was 0.2mm)
- **Minimum via drill**: 0.3mm (was 0.1mm)
- **Minimum via diameter**: 0.5mm

### Net Classes Configuration
| Net Class | Track Width | Via Diameter | Via Drill | Patterns |
|-----------|-------------|--------------|-----------|----------|
| Default | 0.762mm | 0.8mm | 0.4mm | (catch-all) |
| power3V3 | 0.382mm | 0.9mm | 0.5mm | `+3.3VA*`, `+3.3V*`, `+3V*` |
| power5V | 0.508mm | 1.2mm | 0.7mm | `+5V*`, `V+_USB*`, `V_USB*` |
| power | 1.27mm | 1.2mm | 0.7mm | `+12V*`, `+15V*`, `+16*` |
| signal | 0.382mm | 0.8mm | 0.4mm | (signals) |
| rf | 3.62mm | 1.2mm | 0.7mm | `RF*`, `/RF*` |
| V+_USB | 0.508mm | 1.2mm | 0.7mm | `_V+_USB*` |
| V_USB | 0.508mm | 1.2mm | 0.7mm | `V_USB*` |

### Key Insight: Autorouter Uses Net Class track_width
- The **Design Rules → min_track_width** sets the DRC minimum
- The **Net Class → track_width** is what the autorouter actually uses
- Both must be set to 0.382mm for 3.3V nets

---

## 5. Python Scripting for KiCad Files

### Pattern Matching Approach
```python
import re

# Read file
with open('file.kicad_pcb', 'r') as f:
    content = f.read()

# Regex with DOTALL flag for multi-line matching
pattern = r'(\(property "Reference" "[^"]+"[\s\S]*?\(effects\s+\(font\s+)([\s\S]*?)(\n\s+\)\s+\)\s+\))'

def replace_font(match):
    before = match.group(1)
    font_content = match.group(2)
    after = match.group(3)
    if 'face "Arial"' in font_content and 'size 0.7 0.7' in font_content and 'thickness 0.1' in font_content:
        return match.group(0)
    return before + '\n(face "Arial")\n(size 0.7 0.7)\n(thickness 0.1)\n' + after

new_content = re.sub(pattern, replace_font, content, flags=re.DOTALL)
```

### Debugging Tips
- Use `sed -n 'line_start,line_endp' file` to inspect specific lines
- Use `grep -n "pattern" file` to find line numbers
- Check raw content with `cat -A` to see tabs/spaces
- Test regex on small samples first

---

## 6. File Cleanup
### Removed Unnecessary Files
- Unused schematics: `Battery.kicad_sch`, `esp32_S3.kicad_sch`, `osc_ft232.kicad_sch`, `power_ft232.kicad_sch`, `tp_ft232.kicad_sch`, `usb_conector_Ft232.kicad_sch`, `untitled.kicad_sch`, `oled.kicad_sch`
- Lock files: `~*.lck`
- Backup directory: `esp32_xiaozhi-backups/`
- History directory: `.history/`
- Updated `.gitignore` to exclude these patterns

---

## 7. KiCad File Formats
- **.kicad_pcb**: PCB layout (S-expressions)
- **.kicad_sch**: Schematic (S-expressions)
- **.kicad_pro**: Project settings (JSON)
- **.kicad_prl**: Local project settings (ignored)
- **.lck**: Lock files (ignored)

---

## 8. Common Commands Reference

```bash
# Check PCB file size
ls -la esp32_xiaozhi.kicad_pcb

# Count occurrences
grep -c "pattern" file

# View specific lines
sed -n '100,200p' file

# Find in all project files
grep -r "pattern" /path/to/project --include="*.kicad_*"

# Python one-liner for quick edits
python3 -c "
import re
with open('file') as f: c=f.read()
c=re.sub(r'old', 'new', c)
with open('file', 'w') as f: f.write(c)
"
```

---

## 9. Troubleshooting

### PCB Load Error
```
Error: Expecting at, descr, locked... Got 'footprint'
```
**Cause**: Malformed S-expression (missing closing parentheses)
**Fix**: Check the line mentioned, ensure all `(` have matching `)`

### Autorouter Using Wrong Track Width
**Cause**: Net Class `track_width` not updated (only Design Rules `min_track_width` was)
**Fix**: Update both Design Rules AND Net Classes

### Font Changes Not Applying
**Cause**: Regex pattern not matching due to whitespace/formatting differences
**Fix**: Use `re.DOTALL` flag, inspect raw content with `cat -A`

---

## 10. Voltage-Specific Track Width Standards
| Voltage | Min Track Width | Typical Use |
|---------|-----------------|-------------|
| 3.3V | 0.382mm (15 mil) | Logic, MCU IO, peripherals |
| 5V | 0.508mm (20 mil) | USB power, 5V regulators |
| 12V+ | 1.27mm (50 mil) | High current power rails |
| RF | 3.62mm | Impedance controlled traces |

---

## Summary of Changes Made
1. ✅ Removed all 17 vias from PCB
2. ✅ Renamed `+3V3` → `+3.3VA` in ft232 schematic
3. ✅ Renamed `GND` → `GND1` in ft232 schematic
4. ✅ Set all 166 footprint References to Arial 0.7x0.7 0.1
5. ✅ Set all footprint Values to Arial 0.7x0.7 0.1
6. ✅ Board Setup: min_track_width = 0.382mm
7. ✅ Board Setup: min_via_drill = 0.3mm
8. ✅ Net Classes: Default=0.762, 3.3V=0.382, 5V=0.508, signal=0.382
9. ✅ Added `+3.3VA` pattern to power3V3 net class
10. ✅ Cleaned up unused files and directories
11. ✅ All changes committed and pushed to GitHub