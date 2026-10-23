# KiCad LCSC Import Skill

## Purpose
Safely add/update LCSC part numbers in KiCad schematic files (.kicad_sch) without breaking the S-expression structure.

## When to use
- Need to import LCSC codes from a BOM/CSV (e.g., JLCPCB BOM) into schematic symbols
- Need to add LCSC as a custom property to components
- Multiple hierarchical sheets need updating

## Prerequisites
- CSV with columns: Reference, LCSC (and optionally others)
- References may be comma-separated in a single row
- LCSC may have whitespace
- Some rows may have empty LCSC

## Workflow

### 1. Analyze
```bash
# Find schematics (exclude .history, backups)
find . -name "*.kicad_sch" | grep -v ".history" | grep -v ".bak"
# Check CSV
head -5 file.csv
```

### 2. Create backups
Always backup before modifying:
```bash
cp file.kicad_sch file.kicad_sch.bak.$(date +%Y%m%d_%H%M%S)
```

### 3. Parse CSV safely
Use Python's csv module, not string splitting. Handle:
- Comma-separated references in one cell
- Strip whitespace from references and LCSC
- Skip empty LCSC
- Detect conflicts (same ref different LCSC values)
- Build ref -> lcsc map

### 4. Parse KiCad S-expression structure
KiCad schematics are S-expressions. Don't do naive string replacement.

**Safe approach:**
1. Read entire file as string
2. Parse symbol blocks by counting parentheses (track depth, respect strings)
3. For each symbol block, extract Reference property
4. Check for existing LCSC property within symbol block
5. Modify only that symbol block
6. Reconstruct content

**Or use line-based with symbol tracking:**
- Track current symbol start/end by depth
- Find reference within symbol
- Search for LCSC within symbol bounds only
- Insert/modify only within bounds

### 5. Add LCSC property correctly
LCSC must be a property at symbol level (same level as Reference, Value, Footprint).

Correct format:
```scheme
(property "LCSC" "C123456"
    (at 0 0 0)
    (hide yes)
    (show_name no)
    (do_not_autoplace no)
    (effects
        (font
            (size 1.27 1.27)
        )
    )
)
```

**Placement:** Insert after Value property (or Reference if no Value). Match indentation to surrounding properties.

### 6. Handle conflicts
- If LCSC exists with same value: count as already_same, don't change
- If LCSC exists with different value: report conflict, DON'T overwrite
- If LCSC exists but empty: fill it
- If missing: add it

### 7. Process all sheets
If project has hierarchical sheets, process all .kicad_sch files in project (except .history/backups).

### 8. Verify
After changes, check for syntax issues. KiCad will complain on load if structure broken.

Common errors:
- "Expecting font, justify, hide or href. Got ''property''" - property placed inside wrong block (inside effects)
- "Expecting id, at, hide..." - wrong nesting level
- Parentheses unbalanced

### Python script template
```python
#!/usr/bin/env python3
import csv, re, os
from datetime import datetime

# Backup
sch = 'main.kicad_sch'
os.system(f'cp {sch} {sch}.bak.{datetime.now().strftime("%Y%m%d_%H%M%S")}')

# Parse CSV
ref_to_lcsc = {}
with open('bom.csv') as f:
    for row in csv.DictReader(f):
        lcsc = row['LCSC'].strip()
        if not lcsc: continue
        for ref in row['Reference'].split(','):
            ref = ref.strip()
            if ref and ref not in ref_to_lcsc:
                ref_to_lcsc[ref] = lcsc
            elif ref in ref_to_lcsc and ref_to_lcsc[ref] != lcsc:
                print(f'CONFLICT {ref}')

# Parse and update
with open(sch) as f:
    content = f.read()

i = 0
while True:
    idx = content.find('(symbol', i)
    if idx == -1: break
    # find end
    depth = j = idx
    in_str = False
    while j < len(content):
        c = content[j]
        if c == '"' and (j==0 or content[j-1]!='\\'): in_str = not in_str
        elif not in_str:
            if c == '(': depth += 1
            elif c == ')':
                depth -= 1
                if depth == 0:
                    end = j+1
                    break
        j += 1
    block = content[idx:end]
    rm = re.search(r'\(property "Reference" "([^"]+)"', block)
    if rm and rm.group(1) in ref_to_lcsc:
        ref = rm.group(1)
        lcsc = ref_to_lcsc[ref]
        if not re.search(r'\(property "LCSC"', block):
            vm = re.search(r'(\(property "Value"[^()]*\([^()]*\)[^()]*\))', block)
            if vm:
                lp = f'\n\t\t(property "LCSC" "{lcsc}"\n\t\t\t(at 0 0 0)\n\t\t\t(hide yes)\n\t\t\t(show_name no)\n\t\t\t(do_not_autoplace no)\n\t\t\t(effects\n\t\t\t\t(font\n\t\t\t\t\t(size 1.27 1.27)\n\t\t\t\t)\n\t\t\t)\n\t\t)'
                block = block.replace(vm.group(1), vm.group(1)+lp)
                content = content[:idx]+block+content[end:]
                end = idx+len(block)
    i = end

with open(sch,'w') as f:
    f.write(content)
```

## Tips
- Always work on copies first for testing
- Match indentation style of existing properties in that symbol
- Don't modify other properties
- Count parens carefully - easy to break structure
- If structure breaks, restore from backup
- Use `kicad-cli` or just open in KiCad to validate if possible
